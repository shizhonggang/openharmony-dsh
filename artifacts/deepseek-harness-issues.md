# DeepSeek Harness 鸿蒙安装问题全记录

日期：2026-08-13 · 状态：**已跑通**（Web UI 运行于 http://127.0.0.1:3080）

## 环境
- HarmonyOS PC，aarch64，**musl libc**，`process.platform === 'openharmony'`
- Node v26.5.0（harmonybrew）、npm 11.17.0、pnpm 11.7.0（npm 全局装）
- 无 gcc/g++，只有 clang 15.0.4；`cc`→clang 软链；无独立 `node-gyp`（用 npm 自带的）
- `/tmp` 只读

## 核心结论
1. **源码 build 不可行**，必须用 npm 包：`npm install -g @deepseek-ai/dsh@0.1.0-rc.6 --ignore-scripts`
2. 拦路虎是 **rolldown（tsdown）的预编译 `.node` 在此平台 dlopen 失败**（Permission denied），而本机 node-gyp 现编的 `.node` 能加载
3. 启动必须 `node --expose-internals` 直调 bin.js + `DSH_PERMISSION_MODE=danger-full-access`
4. 沙箱在此平台**只能 full-access**（无 bwrap/landlock 后端）

## 全部问题清单

### 一、源码安装阶段（走不通，仅记录）
| # | 症状 | 根因 | 解法 |
|---|---|---|---|
| 1 | esbuild@0.21.5 `Unsupported platform: openharmony arm64 LE` | 老 esbuild（vite@5/website 用）无 openharmony 支持 | 新 esbuild 0.25/0.28 有 `@esbuild/openharmony-arm64`，正常；0.21.5 仅是 website 的 devDep，非核心 |
| 2 | node-pty 无 openharmony prebuild，源码编译 | prebuild-install 找不到 openharmony 平台包 | node-gyp + clang 现编（见下） |
| 3 | koffi cnoke 编译报 x86-64 汇编错误（`movq $8191,%rax` 在 arm64 上） | koffi 生成 `gnu.inc`（x86-64 trampoline）而非 aarch64，ABI 检测 bug | 无法编译，只能 stub |
| 4 | lefthook `Cannot find module 'lefthook-openharmony-arm64'` | 无鸿蒙 git-hook 二进制 | 忽略（仅开发期 git hook） |
| 5 | **rolldown/tsdown** `Error loading shared library rolldown-binding.openharmony-arm64.node: Permission denied` | 预编译 `.node` 在此平台 dlopen 被拒（chmod +x、复制到别处均无效；本机现编的 pty.node 却能加载） | **判定源码 build 不可行，转 npm 包** |

### 二、npm 包运行阶段
| # | 症状 | 根因 | 解法 |
|---|---|---|---|
| 6 | `dsh web` 启动即崩：node-pty require MODULE_NOT_FOUND | `--ignore-scripts` 跳过了 node-pty 编译 | 手动 `node-gyp rebuild`（CC=clang CXX=clang++） |
| 7 | 启动崩：`dsh-sandbox-local: Cannot find the native Koffi module` | koffi 原生 binding 缺失；sandbox-local **静态 import** windows-acl→koffi（不是只在 Windows 加载！） | 给 koffi 打 stub（见下） |
| 8 | 启动崩：sharp `Could not load using openharmony-arm64 runtime` | `--ignore-scripts` 没装平台二进制 | 装 `@img/sharp-wasm32`，sharp 0.35.3 自动走 WASM 兜底 |
| 9 | 启动崩：HMR 需要 `--expose-internals` | cordis-plugin-hmr 需要该 Node 标志；且 NODE_OPTIONS 不允许带它 | 直接 `node --expose-internals <bin.js> web` |
| 10 | 尝试禁用 sandbox 插件 → `2 entries did not activate`（bash-sandbox 等 sandbox 服务、permission-presets 等 shell 服务） | 禁用引发依赖级联 | **不要禁插件**，改用 stub + danger-full-access |
| 11 | 裸 `dsh web` 再启动报 `EADDRINUSE 127.0.0.1:3080` | 端口被已运行实例占用 | 用启动脚本，先停旧实例 |

## 关键修复细节

### node-pty（真实编译，非 stub）
```sh
cd <dsh>/node_modules/node-pty
export CC=clang CXX=clang++
node $EL2_BASE/.harmonybrew/Cellar/node/26.5.0/libexec/lib/node_modules/npm/node_modules/node-gyp/bin/node-gyp.js rebuild
```

### koffi stub（编辑 `koffi/src/koffi/index.js`）
- 触发点：dsh-sandbox-windows-acl 在**模块加载时**调 `koffi.pointer()`/`koffi.struct()`，并断言：
  - `STARTUPINFOW.size === 104`
  - `PROCESS_INFORMATION.size === 24`
- stub 需实现 win32 ABI 布局计算（uint32=4/uint16=2/str16=8/指针=8，按对齐排布），两个 size 手算可对上
- 其余函数（load/decode/encode/alloc/address…）全部返回惰性 throw——它们只在 win32 运行时被调用，本平台永不执行

### sharp（真实 WASM 实现，非 stub）
```sh
cd <dsh> && npm install @img/sharp-wasm32 --no-save
```
`@img/sharp-wasm32` 的 exports 映射 `./sharp.node → index.cjs → lib/sharp-wasm32-*.node.js`（WASM 加载器），sharp 自动兜底。

### 启动脚本 `$EL2_BASE/bin/dsh-web.sh`
```sh
DSH_HOME=$EL2_BASE/dsh-home DSH_PERMISSION_MODE=danger-full-access \
  node --expose-internals ~/.local/npm-global/lib/node_modules/@deepseek-ai/dsh/lib/bin.js web
```
- `--expose-internals` 不可放 NODE_OPTIONS（Node 拒绝），只能命令行传参
- `DSH_PERMISSION_MODE=danger-full-access`：避开受限模式（沙箱不可用时 confined 模式抛 SANDBOX_UNAVAILABLE）

## 功能降级澄清（纠正早前汇总）
| 项 | 早前说法 | 实际 | 验证 |
|---|---|---|---|
| node-pty / PTY 终端 | "stub，不可用" | **真实编译可用** | `spawn` 返回 function，build/Release/pty.node 存在 |
| sharp / 图片附件 | "stub，不可用" | **真实 WASM 可用**（慢于原生） | 成功渲染 PNG（97 bytes） |
| chokidar / 热重载 | "no-op" | **真实可用** | watch 事件实测触发 |
| 沙箱隔离 | 关闭 | 关闭（danger-full-access），鸿蒙本不支持 | — |
| DEEPSEEK_API_KEY | 需要 | **需要** | Web UI Settings→Models 填入，存 `$DSH_HOME/.credentials.yaml` |

真正的降级仅两项：**沙箱内核隔离关闭**、**sharp 用 WASM 而非原生（性能略低）**。

## 追加问题（运行期 #12）
| # | 症状 | 根因 | 解法 |
|---|---|---|---|
| 12 | 会话保存报 `EACCES: permission denied, link '.../session.jsonl.zstd.tmp' -> '.../session.jsonl.zstd'` | **本系统全盘禁止硬链接**（hmfs 报 Permission denied，Documents 报 Operation not permitted），而 dsh 的会话持久化和附件存储用 `link()`（硬链接）做原子发布 | 把两处 `link` 换成 `rename`（同目录 rename 同样原子，hmfs 上实测可用）：`dsh-session-persistence-jsonl/lib/index.js`（materializePosix）和 `dsh-attachment-local/lib/index.js`（存附件）。`dsh-atomic-write` 本来就用的 rename，无需改 |

**硬链接禁令是系统级事实**：任意文件系统（hmfs / Documents）上 `ln` 都失败——疑似 seccomp 或内核屏蔽 linkat。以后凡见 `link()` 原子写模式，一律换 `rename()`。

## 给 dsh 团队的可行动建议
按对普通用户的影响排序，以下几项建议在代码层修复（不依赖用户本地 patch）：
1. **#5 rolldown 预编译 `.node` dlopen 失败**：`@rolldown/binding-openharmony-arm64` 已发布，但 `dlopen` 报 Permission denied（本机现编的 `.node` 却可加载）。请确认 openharmony 预编译产物是否有问题，或提示用户用源码编译回退。
2. **#7 koffi 被静态 import**：`dsh-sandbox-local` 静态 import `dsh-sandbox-windows-acl` → `koffi`，导致**所有平台**启动时都要加载 koffi 原生模块。建议改成平台条件动态 import，避免非 Windows 平台被 win32 依赖拖垮。
3. **#12 `link()` 原子写**：会话持久化/附件存储用硬链接发布。部分系统（如 HarmonyOS）禁止 linkat。建议原子写统一用 `rename`（同目录原子），或提供 fallback。
4. **#3 koffi cnoke 在 openharmony_arm64 生成 x86-64 trampoline**：ABI 检测 bug，arm64 上编译不过。
5. **#9 HMR 依赖 `--expose-internals`**：该标志无法经 NODE_OPTIONS 传入，导致裸 `dsh web` 启动失败。建议 HMR 改为可选或降级提示。
6. **#1 esbuild@0.21.5 不支持 openharmony**：仅 website 的 vite@5 依赖，建议升级或标注为非核心。

## 关联
- 源码仓库：`$EL2_BASE/deepseek-harness`（本机研究用，build 不通）
- npm 安装位置：`~/.local/npm-global/lib/node_modules/@deepseek-ai/dsh`
- 本文档为 HarmonyOS（openharmony aarch64）单机实测报告
