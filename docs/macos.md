# macOS（实验性）

状态：fork 分支 `macos-support`，arm64 与 x64 均在 GitHub Actions 的 macOS runner 上构建并跑 `npm run verify`。
**真实 DSH 沙箱（Seatbelt）下的拒绝行为已在真机实测**（macOS 26.5.2 / arm64 / Node v26.4.0，桌面版 profile 内），见「状态」。

## 状态

### 1. 闸门

```console
DSH_OPS_REQUIRE_RUNTIME=1 npm run verify   # 13 suite(s) passed
node bin/dsh-ops.mjs ladder --json         # shells.bash.source === "os", executable === "/bin/bash"
```

GitHub Actions `macOS` 工作流双架构绿（run 38691638696：darwin-arm64 5m52s、darwin-x64 11m47s）。

初跑 13 项红、分布 5 个套件。三组根因**全部在测试侧**，产品行为一行未改（`git diff --name-only 0b57f32..105e5a4` 只有 `test/` 下 6 个文件）：

| 组 | 根因 | 改动 |
| --- | --- | --- |
| A | macOS 的临时目录本身是符号链接（`os.tmpdir()` = `/var/folders/…`，`/var` → `/private/var`）。原生 worker 会把收到的**工作区根** canonical 化（`native/src/lib.rs` 的 `std::fs::canonicalize`），所以根上有链接没问题；它拒绝的是**被操作路径的组件**里有链接（`native/src/file_path.rs` 逐组件 `open_dir_nofollow` 检查），于是套件用未解析路径拼出的绝对目标全被拒 | `test/run.mjs` 与 `test/lib/harness.mjs` 把一次性 `DSH_HOME`／临时工作区 canonical 化；harness 的临时目录守卫两侧都按 canonical 比较（并按分隔符比较，避免兄弟目录误判） |
| B | 断言钉死 Windows 包布局 | `test/binary.test.mjs` 用被伪造目标的可执行名播种受管副本（原先播种宿主名，搜索永远落空）；`test/shells.test.mjs` 的 store 路径断言改为随宿主形态 |
| C | 错误文案已改 | `test/ptc.test.mjs` 的 `/Windows x64 only/` → `/supports Windows x64 and macOS/` |

另有两处「工作区外」夹具其实落在沙箱授权内（原因见 3.1），已移到仓库根的 `.tmp-*` 临时树，拒绝断言才真正成立。

### 2. 真机沙箱实测

装进桌面版 profile：`npm pack` → `npx --yes ./dsh-ops-0.3.0.tgz provision --from "$PWD/vendor/fastctx/target/release/fastctx"` → `dsh plugin --profile desktop add "$PWD/dsh-ops-0.3.0.tgz"`。
装进去的包（`~/.dsh/profiles/desktop/node_modules/dsh-ops`）为 0.3.0，`PLATFORMS` 含 `darwin-arm64`/`darwin-x64`，`lib/shells.js` 带 `osOwnedBash`；桌面 profile 四个 row（shell / file / background / ptc）全部开启。

复现脚本 `test/.tmp-macos-verify.mjs`（临时文件，`.tmp-*` 已被 gitignore，不在闸门内）：按桌面 profile 的 row 配置挂载**已安装的那个包**，跑真实 Cordis、真实工具注册表、真实 `@deepseek-ai/dsh-sandbox-local` / `dsh-subprocess-local` / `dsh-fs-local`。

| 检查 | 结果 | `enforcement` |
| --- | --- | --- |
| `ops_bash` 可用；`echo $BASH_VERSION` | `3.2.57(1)-release`；`command -v bash` → `/bin/bash` | `"full"` |
| file 四个工具搜索/读取工作区（glob、grep、inspect_local_file、replace） | 全部 `completed` | `"policy"` + `boundary: "trusted-native-file-operations"` |
| ptc 组件开启后 `run_code` 里的 `tools.ops_batch` | `status: "completed"` | `"full"` |
| workspace-write：`ops_bash` 写工作区外（`~/Desktop/x`） | 拒绝：exit 1、`permission.code = POLICY_DENIED`（`command file effects`，需 `danger-full-access`），stderr `/bin/bash: …: Operation not permitted`，文件未创建 | 拒绝结果不带 `sandbox` 字段 |
| workspace-write：file 工具写工作区外 | 拒绝：`status: "rejected"`、`POLICY_DENIED`（preflight，`replace outside the workspace or DSH temporary areas`），文件未改动 | `{"mode":"workspace-write"}` |
| read-only：`ops_bash` 写工作区内 | 拒绝：exit 1、`POLICY_DENIED`（`command file effects`，需 `workspace-write`），文件未创建 | 拒绝结果不带 `sandbox` 字段 |
| read-only：file 工具写工作区内 | 拒绝：`status: "rejected"`、`POLICY_DENIED`（preflight，`replace`），文件未改动 | `{"mode":"read-only"}` |
| 工作区放在 `/tmp`（→ `/private/tmp`） | `ops_glob` 与 `ops_inspect_local_file` 均 `POLICY_DENIED`：`Links, reparse points and special objects are not traversed; inspect the target path for the offending entry` —— 清晰报错，不是静默失败 | 拒绝结果不带 `sandbox` 字段 |

### 3. 三条需要知道的 macOS 行为

**3.1 整个平台临时目录都在 `workspace-write` 的授权内。**
`writableRoots()`（`node_modules/@deepseek-ai/dsh-sandbox/lib/index.js`）在 `workspace-write` 下把 `policy.workspaceRoot`、`/tmp`、`os.tmpdir()` 去重授权；Windows 走的是每工作区 SID 加随机私有临时目录，所以「工作区的兄弟目录」只在 Windows 落在授权外。⇒ 在 macOS 上，证明边界的夹具必须落在授权之外（仓库根的 `.tmp-*` 临时树），否则断言恒真。

**3.2 被操作路径里出现链接会明确拒绝，不是缺陷。**
`/tmp` 与 `os.tmpdir()` 都是链接（`/private/tmp`、`/private/var/…`）。工作区根会被 worker canonical 化（`native/src/lib.rs`），但被操作的目标路径由 `native/src/file_path.rs` 逐组件检查，链接一律拒绝并给出可读理由（那句 `Links, reparse points and special objects are not traversed` 出自本仓库的 `native/src/file_path.rs`，不在任何依赖里），而不是静默改写路径或静默失败——这是安全边界。会话工作区放在 `/tmp` 下会得到第 2 节表格里那条报错；用 `realpath` 后的路径即可。

**3.3 桌面版 CLI 启动器可能没有执行位。**
本机 `/Applications/DeepSeek Harness.app/Contents/Resources/runtime/cli/bin/dsh` 是 mode 644（同目录留有 `dsh.dsh-purge.bak`），直接执行报 `Permission denied`；`sh "<该路径>" …` 正常运行。`lib/profile-install.js` 的 `desktopCliCandidates()` 只按「文件存在」挑选命令，随后直接 `spawnSync`，在同样情况下会拿到 `EACCES`。本机实测即用 `sh` 绕过，未改产品代码。

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
   （若 CLI 不在该路径，自行定位 app 包内的 `dsh`；该启动器若丢了执行位，改用 `sh "<该路径>" …`，见 3.3。）
4. 重启桌面版，新构建才会被加载。

## 从源码

```console
npm ci
cargo build --release --locked --manifest-path vendor/fastctx/Cargo.toml
npm run build:ptc
npm run verify
```
