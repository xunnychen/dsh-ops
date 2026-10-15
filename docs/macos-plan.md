# macOS 适配：剩余工作计划

分支：`macos-support`（fork `xunnychen/dsh-ops`）。在 Apple Silicon Mac 本机执行。

## 已完成（已提交）

- `package.json` / `dsh-plugin.json`：`os` 加 `darwin`，`cpu` 加 `arm64`
- `lib/ptc/runtime.js`：`PLATFORMS` 加 `darwin-arm64`、`darwin-x64`
- `lib/shells.js`：新增 `osOwnedBash()`，macOS 上无打包 bash 时用 `/bin/bash`（source `'os'`）；`resolveShells` 接受 `options.osBash`（`null` 关闭）
- `lib/profile-install.js`：允许 macOS，桌面版 CLI 探测 `/Applications/DeepSeek Harness.app/Contents/Resources/runtime/cli/bin/dsh`
- `scripts/validate-manifest.mjs`、`bin/check-release.mjs`：跟随新平台声明
- `.github/workflows/macos.yml`：macOS arm64/x64 构建 + `npm run verify` + 打包
- `docs/macos.md`

CI（run 38686475642）结果：**构建全部成功**，Mach-O 校验、`/bin/bash` 识别通过；`npm run verify` 有 13 处失败，见下。

## 第 0 步：准备

```bash
git clone -b macos-support https://github.com/xunnychen/dsh-ops && cd dsh-ops
npm ci
cargo build --release --locked --manifest-path vendor/fastctx/Cargo.toml
npm run build:ptc
DSH_OPS_REQUIRE_RUNTIME=1 npm run verify    # 复现失败
```

单独跑某个套件：`node test/run.mjs shells`（按文件名过滤）。

## 第 1 步：修测试失败（按根因分组）

### A. macOS 临时目录是符号链接（优先，可能一次修掉 4 处）

`os.tmpdir()` 在 macOS 返回 `/var/folders/...`，而 `/var` → `/private/var` 是符号链接。
插件和 Rust worker 都刻意拒绝穿过符号链接的路径，所以测试工作区一放在临时目录就报
`Links, reparse points and special objects are not traversed` 或
`Workspace root must not resolve through a junction or symlink`。

受影响：`mount.test.mjs`（2 处：`real FastCtx calls…`、`native file paths resolve…`）、
`permissions.test.mjs`（2 处）。

修法：
1. `test/lib/harness.mjs` 的 `withTempDir`（及 `run.mjs` 里给每个套件建 `DSH_HOME` 的地方）
   对 `mkdtemp` 结果做 `fs.realpathSync.native()`，让测试拿到 `/private/var/...`。
2. 检查生产代码：`lib/file/worker.js:44` 的 `writeRoots` 已做 `canonical()`；确认
   `lib/ptc/worker.js` 等传给 Rust worker 的根路径也都先 realpath。**不要**放宽 Rust 端
   的符号链接拒绝逻辑——那是安全边界。
3. `permissions.test.mjs` 里 `preparation coalesces…` 用 `platform:'win32'` 跑真实临时目录，
   修完第 1 点应自动通过；若仍失败，说明它本就该只在 Windows 跑，加 `process.platform !== 'win32'` 时跳过。

### B. 测试写死了 Windows 包布局（7 处）

- `binary.test.mjs` 5 处（`published platform package is preferred over PATH` 等，第 190–333 行）：
  `FORGED` fixture 固定 `platform:'win32'` 和 `fastctx.exe`，但 `resolveBinary` 内部某处仍用
  `process.platform` 推导包名或可执行名。先看 `+ actual - expected` 的具体差异，再决定：
  - 若是生产代码里漏传 `platform` 参数 → 修 `lib/binary.js`（这是真 bug）
  - 若是测试本身假设 Windows → 让 fixture 跟随传入的 platform
- `shells.test.mjs` 4 处（第 306、371 行附近，及 pin table 那条）：`stagePlatformPackage` 写死
  `bash.exe` / `pwsh.exe`，但用 `process.platform` 调 `bundledShell`。shell 平台包只发布
  `win32-x64`，这些测试应传 `platform:'win32', arch:'x64'`，而不是 `process.platform`。

### C. 错误信息文案（1 处）

`test/ptc.test.mjs:211-212`：正则 `/Windows x64 only/` 改为匹配新文案
`/supports Windows x64 and macOS/`。

## 第 2 步：本机通过后

```bash
DSH_OPS_REQUIRE_RUNTIME=1 npm run verify   # 应全部通过
node bin/dsh-ops.mjs ladder --json          # bash.source 应为 "os"
git commit -am "macOS: fix gate failures" && git push
```

确认 GitHub Actions 的 macOS 工作流两个架构都变绿。

## 第 3 步：装进本机 DSH 桌面版实测（CI 测不了的部分）

```bash
npm pack
npx --yes ./dsh-ops-0.3.0.tgz provision --from "$PWD/vendor/fastctx/target/release/fastctx"
"/Applications/DeepSeek Harness.app/Contents/Resources/runtime/cli/bin/dsh" plugin --profile desktop add "$PWD/dsh-ops-0.3.0.tgz"
```

若 CLI 不在该路径：`find "/Applications/DeepSeek Harness.app" -name dsh -type f`，找到后同步修
`lib/profile-install.js` 里的 `desktopCliCandidates()`。

重启桌面版，在一个测试工作区里验证：

1. `ops_bash` 可用，`echo $BASH_VERSION` 输出 3.2.x
2. file 组件四个工具能搜索/读取工作区文件
3. 开启 ptc 组件后 `ops_batch` 能执行
4. **沙箱拒绝**：在 workspace-write 模式下让 `ops_bash` 和 file 工具写工作区**外**的路径（如 `~/Desktop/x`），
   必须被拒绝；read-only 模式下写工作区内也必须被拒绝。记录 DSH 返回的 `enforcement` 值。
5. 工作区放在 `/tmp` 下（符号链接到 `/private/tmp`）时的行为——确认是清晰报错而不是静默失败。

把结果写进 `docs/macos.md` 的"状态"一节。

## 不要做的事

- 不要放宽 Rust worker 的符号链接 / 特殊文件拒绝
- 不要把 `allowSystemShellFallback` 默认改成 `true`
- 不要打包 bash / PowerShell 的 macOS 版本（不需要）
- 不要 `npm publish`
