> 本文档来自补丁作者本地的构建记录（当时直接在 fork 的 `main` 上构建）。本仓库改为「上游源码 + `patches/`」的方式重建：`0001`–`0004` 即文中三项功能与自动关闭修复，`0005` 是让 LSPosed 注入 HyperOS 4 桌面进程的 `staticScope` 声明。

# HyperCeiler HYOS4 三功能构建说明（r4801，含自动关闭修复）

## 产物

| 项 | 值 |
| --- | --- |
| APK | `deliverables/HyperCeiler-2.11.170-6317c1b0a-r4801-canary.apk` |
| SHA-256 | `69097b08ee864e6f720f4c04ea4c92eb14693edc73011988fb74fa011bdf464b` |
| versionName / versionCode | `2.11.170-6317c1b0a-r4801` / `4801` |
| targetSdk / minSdk | 37 (Android 17) / 35 |
| 签名 | APK Signature Scheme v2，证书 `CN=Android Debug`（CI 未配置自有密钥，见「已知限制」） |
| CI 运行 | `keweiya/HyperCeiler` run `45609855198`（workflow_dispatch → `ci_build.yml`，Build App 成功） |

**r4800（`79ce79835`）已废弃**：它的自动关闭补丁把「文件夹确实已打开」这道保护门也 NOP 了，导致桌面图标点击偶尔走进关闭流程、应用起不来。r4801 修正了补丁点。

## 修订记录：为什么 r4800 会「偶尔打不开应用」

`AppIconRemoteClick.handleRemoteClick` 里连续有三道门，任何一道跳走都会跳过 `handleFolderClingClose`，落到普通启动路径：

| 门 | 指令位置（8.11.02.7755） | 判定内容 | r4800 | r4801（HomeTweaks 81 原样） |
| --- | --- | --- | --- | --- |
| 1 | `0x1451218` `tbnz w0,#4` | 点击模式（fp 溢出 → `RxObjectMixin.value`） | NOP | **NOP** |
| 2 | `0x145124c` `tbnz w0,#4` | `Inst|isRegistered`（控制器是否已注册） | 原样 | **NOP** |
| 3 | `0x1451280` `tbnz w0,#4` | `[controller+0xef]`，即 `FolderClingGetxController.isFolderClingOpen` | **NOP（错）** | **保持原样** |

r4800 把第 3 道「文件夹是否真的打开」的守卫拆掉了，于是在桌面点应用时也会调用 `handleFolderClingClose`；控制器不存在/已关闭时这条 Dart 调用会失败，启动续接被中断，表现就是「偶尔图标点了没反应」。r4801 把补丁点改回 HomeTweaks 的第 1、2 道门，保留第 3 道保护门：桌面点击走到守卫处必然为 false → 直接启动；文件夹内点击守卫生效 → 先关文件夹再启动。

交叉验证：从 `libhometweaks.so`（HomeTweaks 81 补丁计划）里取出的 56 字模式窗口，正好覆盖从第 1 道门到 `bl handleFolderClingClose`；其匹配器对下标 2 和 15（即第 1、2 道门）允许「原指令或 NOP」（它自己写 NOP 的位置），对下标 28（第 3 道门）要求**精确等于** `0x37200380`。本仓库解析器现在与这三条完全一致，真实 `libapp.so` 审计输出：

```
plan mode=0x1451218 registered=0x145124c guard=0x1451280 scroll=0x1933dc8 branch=0x14000019
PASS auto-close plan: 3 sites, mutations accepted=735 refused=18
```

## 源码基线

- 全新克隆 `keweiya/HyperCeiler` 的 `main`（`f914ede03`），独立工作区 `/root/dsh-workspace/hyperos4/hc2`，分支 `feat/os4-folder-signal`。
- 已推送到 fork `main`（`6317c1b0a`）：

```
6317c1b0a fix(home): keep the folder-open guard in the auto-close patch
79ce79835 fix(module): stop flagging self-built packages as bad signatures
ae1442403 feat(systemui): let dual-row signal stack with non-WiFi hide mode
025330013 feat(home): port HomeTweaks folder auto-close
```

## 功能一：打开文件夹内应用后自动关闭文件夹

- 两处源字 NOP（第 1、2 道门）+ 关闭例程里的重滚门改成 `b`，其余一律不动：启动续接、保护门、被拦截启动的保护都保持原样。
- 关闭动画仍由原生 `handleFolderClingClose` 播放（x2 传动画标志，未改成立即关闭）。
- 三处源字走原有 journal/apply/settle 通道，关掉开关时逐字节还原；解析器对任何歧义（门不唯一、顺序不对、目标不是 `close` 之后）直接放弃。
- 设置位置：**桌面 → 文件夹 → 「自动关闭」**（摘要「启动应用后关闭所在的文件夹」）。
- 生效日志（`adb logcat -s HyperCeiler.HomeLayout`）：

```
folder auto-close HomeTweaks plan mode=0x… registered=0x… guard=0x… scroll=0x… target=0x…
folder auto-close source-word requested=1 applied=1 sites=3 nop=mode,registered scroll=skip open-guard=kept animated-close=stock
```

## 功能二：WiFi 下隐藏信号 + 双排信号不再互斥

1. `MobileNetworkSettings`：只有勾选「隐藏 SIM 卡 1/2 信号图标」时才禁用双排入口，`signalMode == 1` 不再禁用。
2. `SystemUIB` / `SystemUIV`：双排 Hook 装载条件由 `signal_mode != 0` 改为 `signal_mode >= 2`。
3. `MobilePublicHookV`：双排与信号模式统一走 `refreshVisibility()`，WiFi 连接时整组隐藏；`dualRowHidesSlot()` 强制隐藏 slot 1+ 的 root，避免叠出第二套双排。

设置：**系统界面 → 状态栏 → 移动网络 → 移动信号显示逻辑 → 非 WiFi 下始终显示**，再开 **系统界面 → 状态栏 → 双排移动网络图标 → 启用**（两页均带重启 SystemUI 提示）。

## 功能三：关闭「签名校验失败」告警

- 移除 `HomePageBannerManager.createWarningBannerIfNeeded()` 里 `SignUtils.isSignCheckPass()` 失败时返回的 `warning_sign` 横幅；非官方 ROM、框架异常、系统版本不支持等其它告警不变。
- 产物核验：反编译后 `classes.dex` 中已无 `warning_sign` 常量（分支被 R8 消除），`warning_rom`、`warning_framework` 仍在。

## 构建与产物核验

- `:library:libhook` 与 `:app:compileCanaryJavaWithJavac` 本地编译通过；NDK `clang++ -Wall -Wextra -Werror -fsyntax-only` 通过。
- CI：`native_host_tests`、`Build App` 均成功；产物解包核对：
  - `lib/arm64-v8a/libHyperCeilerNative.so` 含 `… nop=mode,registered … open-guard=kept …` 与 `… plan mode=… registered=… guard=… …`；
  - `MobilePublicHookV` 反编译可见 `signalShowMode >= 1 || dualRowEnabled()` 与 `dualRowHidesSlot()`；
  - `SystemUIB` 双排装载条件为 `< 2`；`MobileNetworkSettings` 中双排禁用条件只剩 `card1 || card2`。

## 安装

```bash
adb install -r HyperCeiler-2.11.170-6317c1b0a-r4801-canary.apk
```

## 已知限制

- 没有真机：自动关闭的点击路径只做了真实 `libapp.so` 审计与静态推演，仍需你在设备上确认；出问题时 `logcat -s HyperCeiler.HomeLayout` 能看到是否 `applied=1`、三处地址是否命中。
- 自动关闭依赖桌面 `libapp.so` 的指令形状（8.11.02.7755 已验证）；版本不符时只写日志、不改内存，不会破坏原生行为。
- 双排 + 非 WiFi 模式下，WiFi 连接会让整组双排两行一起隐藏；单卡仍按原逻辑。
- CI 未配 `KEY_STORE_PASSWORD / ALIAS / SIGNING_KEY`，每次构建都是 runner 随机 debug 证书：覆盖安装需先卸载旧版；要长期免卸载升级需给 fork 配上这三个 secret。
- 推送 `main` 触发的 push 版 run 在 `Push to Canary Channel`（Telegram）步骤必然失败，是 fork 没配该 secret，与 APK 无关。
