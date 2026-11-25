# HyperCeiler-HYOS4（补丁仓库）

本仓库**不保存 HyperCeiler 源码**，只保存补丁和构建用 workflow：

1. GitHub Actions 从 [`ReChronoRain/HyperCeiler`](https://github.com/ReChronoRain/HyperCeiler) 拉取指定提交的源码；
2. 依次套用 `patches/*.patch`；
3. 用仓库密钥签名并 `assembleCanary`，APK 作为 artifact 上传。

## 目录

```
.github/workflows/build.yml   拉上游 → 打补丁 → 构建 → 上传 APK
patches/                      四个补丁，按文件名顺序套用
docs/                         功能说明与验证记录（HyperCeiler-r4801-三功能说明.md）
```

## 补丁内容

| 补丁 | 作用 |
| --- | --- |
| `0001-feat-home-port-HomeTweaks-folder-auto-close.patch` | 桌面：移植 HomeTweaks 81「自动关闭文件夹」（两处 NOP + 关闭例程重滚门改 `b`） |
| `0002-feat-systemui-let-dual-row-signal-stack-with-non-WiFi-hide-mode.patch` | SystemUI：双排信号与「非 WiFi 下始终显示」可叠加，去掉互斥 |
| `0003-fix-module-stop-flagging-self-built-packages-as-bad-signatures.patch` | 模块：去掉自编译包的「签名校验失败」首页横幅 |
| `0004-fix-home-keep-the-folder-open-guard-in-the-auto-close-patch.patch` | 桌面：保留 `isFolderClingOpen` 保护门，桌面点击不再误入关闭流程 |
| `0005-fix-module-declare-static-scope-so-LSPosed-loads-int.patch` | 模块：`staticScope=true` / `minApiVersion=102`，否则 LSPosed 不会注入由 hyos_spawner 拉起的桌面进程，桌面类功能不生效 |

细节与真机验证方法见 [docs/HyperCeiler-r4801-三功能说明.md](docs/HyperCeiler-r4801-三功能说明.md)。

## 怎么跑

- **自动**：往 `main` 推送 `patches/`、`README.md` 或 workflow 的改动会自动触发。
- **手动**：Actions → `Build patched HyperCeiler` → Run workflow。
  - `upstream_ref` 留空 = 使用 `build.yml` 里固定的 `UPSTREAM_BASE`；
  - 也可以填上游的 commit / tag / branch（补丁对不上会直接失败并打印冲突文件）。
  - `task` 可选 `assembleCanary`（默认）或 `assembleDebug`。

构建完成后在该次 run 的 Artifacts 里下载 APK。

## 需要的仓库 secrets

| Secret | 用途 |
| --- | --- |
| `KEY_STORE_PASSWORD` / `ALIAS` / `SIGNING_KEY` | 签名（`SIGNING_KEY` 是 keystore 的 base64）。不配则回退 debug 签名，覆盖安装会签名冲突 |
| `GIT_ACTOR` / `GIT_TOKEN` | 拉取 `com.sevtinge.hyperceiler` 的 GitHub Packages 依赖（`read:packages`） |

## 上游更新后怎么维护补丁

```bash
git clone https://github.com/ReChronoRain/HyperCeiler.git hc
cd hc
git checkout <new-upstream-commit>
git am /path/to/patches/*.patch          # 冲突就手工解决后 git am --continue
git format-patch <new-upstream-commit>..HEAD -o /path/to/this-repo/patches --no-signature
```

然后把 `patches/` 里的旧文件替换成新导出的四个补丁（保持文件名顺序），并更新
`.github/workflows/build.yml` 里的 `UPSTREAM_BASE`。

## 说明

- 补丁按 8.11.02.7755 的桌面 `libapp.so` 指令形状定位；桌面版本不符时自动关闭功能只写日志、不改内存。
- 本仓库与 HyperCeiler 上游、LSPosed 无隶属关系，仅用于个人构建。
