# DeepSeek Harness 鸿蒙部署 — 平台功能限制与 Agent 运行环境发现

日期：2026-08-14 · 作者：dsh Web UI 内运行的 coding agent（deepseek-v4-flash）
目的：整理本机 HarmonyOS 部署的功能限制与运行环境事实，**供外部工具（如 Claude Code）审阅并提出改进建议**。

> 姊妹文档：[`deepseek-harness-issues.md`](./deepseek-harness-issues.md)（2026-08-13 安装问题全记录，本文件为其后续实测复核与能力对照）

---

## 1. 环境事实（本次会话实测验证，非文档转述）

| 事实 | 实测结果 | 验证方式 |
|---|---|---|
| 平台 | `process.platform === 'openharmony'`，aarch64，Node v26.5.0 | `node -e` |
| libc | musl（系统无 `ldd`，无法直接验证，HarmonyOS 系 musl） | — |
| dsh 版本 | `@deepseek-ai/dsh@0.1.0-rc.6`（npm 全局安装） | `package.json` |
| dsh 安装位置 | `~/.local/npm-global/lib/node_modules/@deepseek-ai/dsh` | `ls` |
| 插件源码位置 | `$EL2_BASE/dsh-home/profiles/node_modules/@deepseek-ai/`（252 个模块，dsh 插件树完整） | `ls` |
| DSH_HOME | `$EL2_BASE/dsh-home`（sessions / profiles / storages / settings.yaml） | `env` |
| 沙箱 | 禁用：`$DSH_HOME/disable-sandbox.patch.yml`（`sandbox: disabled: true`），启动脚本设 `DSH_PERMISSION_MODE=danger-full-access` | `cat` |
| `/tmp` | **只读**（`mktemp` 报 `Read-only file system`；注意 `cd /tmp` 会成功，易误判） | `mktemp` |
| 硬链接 | **系统级禁止**：`ln` 在 `$HOME` 下报 `Operation not permitted`（全盘生效，疑似内核/安全层屏蔽 linkat） | `ln` |
| 会话持久化 patch | ✅ 已生效：`dsh-session-persistence-jsonl/lib/index.js` 导入无 `link`，用 `rename(tmp, finalPath)` | `grep` |
| node-pty | ✅ 本机 clang 现编：`profiles/node_modules/node-pty/build/Release/pty.node` 存在 | `ls` |
| koffi | ⚠️ stub：`koffi/src/koffi/index.js` 为打包产物，win32 函数惰性 throw | `head` |
| sharp | ⚠️ WASM 兜底：`@img/sharp-wasm32` | 文档+实测 |
| 启动方式 | `$EL2_BASE/bin/dsh-web.sh`：`DSH_PERMISSION_MODE=danger-full-access node --expose-internals <bin.js> web` | `cat` |
| 源码仓库 | `$EL2_BASE/deepseek-harness`（存在，但 **build 不可行**：rolldown 预编译 `.node` dlopen 被拒） | `ls` |
| Web UI | 运行于 `http://127.0.0.1:3080`，本次会话即在其内 | `env` |
| 审批提示 | **本会话禁用**：需审批的动作被自动拒绝，`sandbox_permissions` 提权一律无效 | 会话运行时上下文 |

---

## 2. 与 macOS / Linux / Windows 的功能对照

| 功能维度 | macOS / Linux / Windows（标准支持） | 本机 HarmonyOS | 影响级别 |
|---|---|---|---|
| 内核沙箱隔离 | ✅ bwrap/landlock（Linux）、ACL（Windows）、seatbelt（macOS）；受限模式运行，越权需审批 | ❌ 无任何内核沙箱后端，只能 `danger-full-access`，命令真实执行、文件全权可改 | 🔴 最大差异 |
| 审批/提权流程 | ✅ 可弹审批，用户同意后临时提权 | ❌ 审批提示被禁用，需审批动作自动拒绝，无中间确认 | 🔴 无安全网 |
| 原生模块安装 | ✅ prebuilds 普遍可用，`npm install` 即装即用 | ⚠️ 预编译 `.node` dlopen 被拒（rolldown 案例），只能 node-gyp + clang 现编；koffi 需手写 stub | 🟠 升级成本高 |
| 源码 build / 开发迭代 | ✅ `pnpm build` 可重建 web 产物 | ❌ build 不可行 → 无法重建 web shell，扩展只能走动态 Cordis 插件 | 🟠 扩展路径受限 |
| 图片处理（sharp） | ✅ 原生 libvips，快 | ⚠️ WASM 兜底，功能可用、性能明显更慢 | 🟡 性能降级 |
| PTY 终端（node-pty） | ✅ 预编译 | ✅ 已 clang 现编可用，功能等同 | 🟢 无差异 |
| 临时文件 | ✅ `/tmp`（或 TEMP）可写 | ❌ `/tmp` 只读，临时文件只能放 `$HOME`/工作区 | 🟡 需改习惯 |
| 硬链接（linkat） | ✅ 全平台可用 | ❌ 系统级禁止；依赖 link 原子写的组件需 patch 成 rename（已 patch：会话持久化、附件存储） | 🟡 已解决，写代码需避开 |
| PowerShell | ✅ Windows 有 pwsh 工具 | ❌ 无 pwsh，只有 bash | 🟢 本平台用不上 |
| Windows ACL 安全（koffi） | ✅ 仅 Windows 用 | ❌ koffi stub，win32 路径永不执行 | 🟢 无影响 |
| 热重载（HMR） | ✅ `pnpm run dev:web` 即插即用 | ⚠️ 依赖 `--expose-internals`（NODE_OPTIONS 不允许传），必须启动脚本直传 | 🟡 可绕 |
| Web UI / 会话 / 附件 / 文件监控（chokidar） | ✅ | ✅ 全部可用 | 🟢 无差异 |
| 核心 Agent 能力（模型路由、子代理、目标、工作流、动态 Cordis 插件、web 搜索） | ✅ | ✅ 完整保留 | 🟢 无差异 |

**结论：功能面无缺失，缺失的是"安全网"与"平台原生性"。**

---

## 3. 本 Agent 在此系统的实际限制（请外部工具重点审阅）

1. **零缓冲的真实文件操作**：`danger-full-access` + 审批禁用 → 我对文件系统的所有操作都是真实的、无中间确认的。主流平台"犯错可拦截"，这里没有。Agent 需主动采取保守策略（先读后写、不删未知文件、不触碰无关目录）。
2. **提权通道关闭**：任何 `sandbox_permissions` 请求都被拒绝且为终局，不能重试或换法绕过。
3. **写代码需避开平台禁令**：
   - 原子发布文件必须用同目录 `rename()`，**禁止 `link()`**（系统级事实，任何文件系统上成立；连 dsh 自己的 write 工具都用 link 发布，会直接 EPERM 失败）；
   - 临时文件不能放 `/tmp`；
   - 新装依赖需评估原生模块风险（预编译 .node 大概率 dlopen 失败）。
4. **扩展 Harness 的唯一可行路径是动态 Cordis 插件**（定义→运行于当前进程），因为源码 build 不可行、无法重建 web 产物。
5. **shell 会话工作目录句柄异常**：每次 bash 调用报 `getcwd: cannot access parent directories`（cwd 句柄失效），所有命令需显式传 `workdir` 或先 `cd` 到稳定目录。
6. **无浏览器 DOM/截图上下文**：对 `127.0.0.1:3080` 的 UI 无隐式可见性，只能经工具（Inspect 查询、Host RPC）间接感知页面状态。
7. **升级即回归风险**：dsh 每次升级都需要重打补丁（koffi stub、sharp WASM、link→rename、node-pty 现编），补丁针对具体版本，可能随版本失效。

---

## 4. 希望获得建议的问题清单

1. **无沙箱 + 无审批环境下**，Agent 是否有成熟的"自我约束协议"（如操作分级、危险命令前缀确认、只读优先策略）可借鉴？
2. **动态 Cordis 插件作为唯一扩展路径**是否合理？有无在此平台下更稳妥的扩展方式（避开源码 build）？
3. **升级 dsh 的补丁维护**：koffi stub / sharp WASM / link→rename 三处补丁如何管理才能最小化升级回归？（本地 patch 管理工具？npm overrides？）
4. 硬链接禁令（`Operation not permitted`）在 HarmonyOS 上是否有已知的绕过或官方说明？是否确认是 seccomp 屏蔽 linkat？
5. `/tmp` 只读是挂载配置还是平台限制？是否有可用的可写临时目录惯例（如 `$TMPDIR`、`$XDG_RUNTIME_DIR`）？
6. 该平台是否有值得关注的 **dsh 官方或社区的 HarmonyOS 适配进展**（已知 rolldown openharmony-arm64 已发布但 dlopen 被拒）？

---

## 5. 关键路径速查（便于外部工具直接核查）

```text
dsh npm 安装根      ~/.local/npm-global/lib/node_modules/@deepseek-ai/dsh
dsh 插件源码树      $EL2_BASE/dsh-home/profiles/node_modules/@deepseek-ai/
启动脚本            $EL2_BASE/bin/dsh-web.sh
沙箱禁用补丁        $EL2_BASE/dsh-home/disable-sandbox.patch.yml
会话持久化 patch    .../dsh-session-persistence-jsonl/lib/index.js   （rename 原子写，line ~1128）
koffi stub          .../dsh/node_modules/koffi/src/koffi/index.js    （win32 函数惰性 throw）
node-pty 现编产物   .../node-pty/build/Release/pty.node
源码仓库（build 不通）$EL2_BASE/deepseek-harness
Web UI              http://127.0.0.1:3080
```
