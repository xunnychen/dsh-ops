# macOS（实验性）

状态：fork 分支 `macos-support`，arm64 与 x64 均在 GitHub Actions 的 macOS runner 上构建并跑 `npm run verify`。
**真实 DSH 沙箱（Seatbelt）下的拒绝行为尚未在真机验证。**

## 各组件在 macOS 上的情况

| 组件 | macOS |
| --- | --- |
| `dsh-ops/shell` | 不打包 bash / PowerShell。`ops_bash` 默认使用系统 `/bin/bash`（位于 SIP 保护、已签名的系统卷，无需 `allowSystemShellFallback`）。显式开启 `allowSystemShellFallback` 时优先用 PATH 上的 bash（如 Homebrew bash 5）。PowerShell 层在 macOS 上不存在。|
| `dsh-ops/file` | 需要 FastCtx 可执行文件，见下文。|
| `dsh-ops/ptc` | 原生 Rust worker，包内携带 `runtime/darwin-<arch>/`。|
| `dsh-ops/background` | 纯 JS，无平台差异。|

## 安装（从 CI 产物）

1. 在 Actions 的 `macOS` 工作流下载 `dsh-ops-darwin-arm64`（Intel Mac 选 `x64`），得到 `dsh-ops-<version>.tgz` 和 `fastctx`。
2. 安装 FastCtx 到受管目录：`npx --yes ./dsh-ops-<version>.tgz provision --from "$PWD/fastctx"`
3. 用桌面版自带 CLI 把插件装进 profile：
   `"/Applications/DeepSeek Harness.app/Contents/Resources/runtime/cli/bin/dsh" plugin --profile desktop add "$PWD/dsh-ops-<version>.tgz"`
   （若 CLI 不在该路径，自行定位 app 包内的 `dsh`。）

## 从源码

```console
npm ci
cargo build --release --locked --manifest-path vendor/fastctx/Cargo.toml
npm run build:ptc
npm run verify
```
