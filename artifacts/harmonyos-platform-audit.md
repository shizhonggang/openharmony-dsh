# HarmonyOS 平台审查报告（供 dsh 团队理解运行环境）

> 生成：2026-08-14 · 审阅方：Claude Code（外部工具）
> 数据来源：本机长期实测积累 + 本次会话 2026-08-14 复核。所有"✅ 今日实测"为本次会话验证；其余标注实测时的内核版本。
> 姊妹文档：[`deepseek-harness-issues.md`](./deepseek-harness-issues.md)、[`deepseek-harness-platform-findings.md`](./deepseek-harness-platform-findings.md)

---

## 1. 系统总览

| 项 | 值 | 备注 |
|---|---|---|
| 系统 | HarmonyOS PC | 非 Android、非 Linux 发行版 |
| 内核 | **HongMeng Kernel 1.13.0**（今日实测） | 曾为 1.12.0，近期系统更新；`/proc/sys/kernel/ostype` = `HarmonyOS` |
| 架构 | aarch64 (ARMv8)，20 核 Kirin X90 | Cortex-A510/A715；支持 SVE2/BTI/MTE/国密 SM3-SM4 |
| 内存 | 32GB，当前可用 ~16GB | |
| libc | **musl**（`/lib/ld-musl-aarch64.so.1`） | 无 glibc loader（无 `/lib/ld-linux-*`） |
| Node 平台标识 | `process.platform === 'openharmony'`，`arch === 'arm64'` | 几乎所有 npm 平台分支都不认识 |
| 进程沙箱 | `Seccomp: 2`（filter 模式）+ 1 filter，**今日实测激活** | 每个进程被 appspawn 注入 seccomp |
| 包管理 | hnp（系统域）/ ohpm / harmonybrew（用户域） | 无 apt/yum/dpkg |

**一句话定性**：自研宏内核（非 Linux fork）+ 内核内 Linux syscall 兼容层 + 用户态直接借用 Linux 生态（musl+ELF）。与 macOS 的关系对称：macOS = 自研 XNU + 借 BSD 用户态；鸿蒙 = 自研 HongMeng 内核 + 借 Linux 用户态。

---

## 2. 内核架构（分层模块化宏内核）

```
L4 扩展层: file_info | fileguard | kstate | iaware_notify（各配 _ext 热扩展同伴）
L3 驱动层: hmtpp_dal → tty → kconsole；hmtpp_freq_dal_kirin（Kirin 专属频率驱动）
L2 服务层: net_socket (HTP/MTP) | transfs | of (设备树) | sd.proxy | tracefs
L1 宿主层: devhost（设备驱动框架，替代 Linux bus/driver/device 模型）
L0 兼容层: libdh-lnxbase.so —— Linux syscall ABI 兼容库，被 8 个内核模块共享
```

关键差异 vs Linux：
- **调度器自研**：`ices_rq[0-19]` per-CPU 队列，无 CFS 的 vruntime/红黑树；RT 队列 `rt_rq[0]`。
- **驱动模型自研**：devhost 替代 Linux driver model；`/sys` 基本为空；无 pcieport/xhci_hcd 等标准驱动（PCIe 硬件存在，KirinX90 hi_pcie）。
- **内存管理**：`hyperhold`（内存压缩，macOS 同级）、purgeable 可清除内存（iOS 同级）；缺 `swappiness`/THP/`vfs_cache_pressure`；`/proc/slabinfo`、`/proc/zoneinfo` 不可读。
- **内核模块 23 个全 `[permanent]`**：无动态模块加载机制（无 .ko）。
- **网络**：TCP/UDP + 华为自研 HTP/MTP 协议。
- **syscall 编号空间**：0–283 Linux 标准；284–386 未用；**387–511 seccomp SIGSYS 封杀**；512+ 鸿蒙自留地（系统进程专用）。

---

## 3. Linux 兼容性全景（量化）

| 维度 | 相似度 | 说明 |
|---|---|---|
| 用户态工具链 | **95%** | 25/25 常见命令齐，musl+GNU coreutils |
| syscall 接口 | **83%** | ≤283 全兼容；≥387 几乎全 SIGSYS |
| /proc | **65%** | 327 条目，可读但部分自研字段 |
| 内核子系统 | **~35%** | 调度/驱动/MM 全部自研 |
| **加权综合** | **~61%** | 用户态够用，内核态差异巨大 |

### syscall 封杀清单（实测于 1.12.0，1.13.0 建议复核）

| syscall 范围 | 状态 | 说明 |
|---|---|---|
| ≤283 (membarrier) | ✅ 全可用 | 基础 syscall 100% |
| 387 (rseq) | ❌ SIGSYS | 现代运行时的 per-CPU 重启序列 |
| 434 (pidfd_open) | ✅ | 少数例外 |
| 435 (clone3) | ❌ SIGSYS | 现代线程创建首选 |
| 436 (close_range) | ❌ SIGSYS | musl closefrom 用 |
| 437 (openat2) | ❌ SIGSYS | 扩展 openat |
| 441 (epoll_pwait2) | ❌ SIGSYS | |
| 442–449 | ❌ 全部 SIGSYS | mount_setattr/landlock/memfd_secret/futex_waitv 等 |

**含义**：任何依赖现代 syscall（rseq/clone3/landlock/openat2）的运行时都会撞墙。Bun（内联 SVC 调 rseq）、Go 1.21+、某些新版 Node 原生路径均受影响。Node 26.5.0（harmonybrew）能跑说明其主路径避开了被禁 syscall。

---

## 4. C 运行时与二进制兼容

### 4.1 musl 而非 glibc（今日实测确认）
- 系统二进制全为 musl 动态链接，interpreter `/lib/ld-musl-aarch64.so.1`。
- **无 glibc loader** → glibc 动态链接 ELF 直接 `permission denied`。
- 第三方二进制**必须选 musl 或静态版**；安装脚本的 `isMusl()` 检测在鸿蒙上失灵（无 `/etc/alpine-release`），常误选 glibc 版。

### 4.2 ELF 执行/加载验证链（本平台最独特之处）
鸿蒙对 ELF 有**来源/特征白名单**式的加载验证，三层：

1. **hmdfs MAC（执行放行，按"出身"判定）**：
   - 用户存储（`/storage/Users`）上的二进制，**只有本机 clang 现编的能执行**；cp/cat 复制、SDK 预编译、外来源一律内核层拦（execve EPERM/EACCES）。
   - 放行规则：主二进制带 `.note.ohos.ident` + `.codesign` → 放行；只带 `.codesign` 无 `.note.ohos.ident` → 静态放行/动态拒绝；系统路径（`$OHOS_SDK/bin`，hnp_file:s0 可信域）不受限。
   - **规避**：hnp 打包安装（SELinux 域变为 hnp_file:s0）或注入 `.note.ohos.ident` + Merkle 签名。

2. **ELF 签名（.codesign，Merkle 树格式）**：
   - 内核用 **Merkle 树 + fs-verity 自洽哈希**验证：4KB 分页 SHA256 → Merkle 根 → descriptor → signature = SHA256(descriptor)。非传统 RSA/ECDSA。
   - **共享库（.so）也要签名**——mmap(PROT_EXEC) 的库同样验证，未签名 dlopen 报 `Permission denied`（今日已验证，见 4.3）。

3. **dlopen 差异（今日实测，关键！）**：
   - **本机 node-gyp + clang 现编的 `.node` → 可加载**（node-pty 编译成功并加载）。
   - **npm 预编译的 `.node` → dlopen 报 `Permission denied`**（rolldown binding 案例；chmod +x、复制到其他目录均无效）。
   - 机制未明：疑与"来源白名单"（现编 vs 预编译）或 ELF 元数据/签名约定有关，而非单纯权限位。
   - **这是 dsh 源码 build 不可行的根因**（tsdown→rolldown 的预编译 binding 加载失败）。

### 4.3 硬链接禁令（今日实测）
- **系统级禁止 linkat**：hmfs 报 EACCES，homefs 报 EPERM——**错误码因文件系统而异**，证明是文件系统层各自拒绝，不是 syscall 级 seccomp（per-syscall 过滤器与路径无关，错误必一致）。
- 影响：一切 `link()` 原子写模式（会话持久化、附件存储）必须改 `rename()`。

---

## 5. 沙箱与安全模型

### 5.1 appspawn"浇筑式"沙箱
沙箱不是进程自锁，而是特权启动器 **appspawn** 在 exec 前搭好笼子：
- 私有 mount namespace + `sandbox.json` 白名单视图
- SELinux/hmmac domain + UID 隔离 + cgroup/rlimit
- **seccomp filter 强注入**（`Seccomp: 2`，今日确认）
- 运行中进程**没有自沙箱原语**

### 5.2 非特权自沙箱原语全灭（实测于 1.12.0）
`landlock_create_ruleset`、`unshare(CLONE_NEWUSER/CLONE_NEWNS)`、`openat2` 全部 SIGSYS 处死。`/proc/self/ns/` 只有 ipc/mnt/net/pid/uts，**无 user namespace**。

**对 agent harness 的结论**：Codex 的 Landlock/bwrap 路线、任何依赖 user namespace 的沙箱（bwrap/docker/unshare）在鸿蒙**必然失效**。这就是 dsh 的 bwrap/landlock 后端都不可用的根本原因——不是没装，是内核不提供。

### 5.3 SELinux
- 用户态域 `u:r:hishell_hap:s0` 无 `process:ptrace` 权限 → **gdb/strace/lldb 全挂**。
- 文件 SELinux 上下文随存储层变化：hmdfs=`u:object_r:hmdfs:s0`、hnp 系统=`u:object_r:hnp_file:s0`。

---

## 6. 文件系统景观

### 6.1 三层结构
```
$HOME/    ← hmdfs（FUSE 分布式覆盖层，faked uid 20001006）
    ↓ 底层
$EL2_BASE/        ← hmfs（物理文件系统，标准 Unix 权限）★ 工具安装首选
$EL2/database/    ← hmfs
/data/service/el2/100/hmdfs/   ← hmdfs 非账户数据
```

| 问题 | hmdfs (FUSE) | hmfs (物理) |
|---|---|---|
| Fake uid | 🔴 是 | 🟢 正常 |
| `user.hmdfs.perm` xattr | 🔴 必需 | 🟢 标准位 |
| 权限模型 | 抽象层，坑多 | 基本正常 |

### 6.2 EL2 存储分级（7.0 起）
`$EL2` 从「整个 tmpfs」改为「**真数据 hmfs** + media 留 tmpfs」：el2/base、el2/database、el2/log → userdata 磁盘（**持久**）；el2/media → tmpfs（重启即失）；el2/share → sharefs。

### 6.3 平台级路径怪癖（今日/近期实测）
- **`/tmp` 只读**（mktemp 报 `Read-only file system`；但 `cd /tmp` 成功，易误判）→ 需 `TMPDIR` 指向可写目录。
- 无 `/var/tmp`、`/var/log`、`/run`。
- **硬链接全盘禁止**（见 4.3）。
- hmfs 挂载 `noflush_merge`（写不合并→锁竞争）、`fsync_mode=nobarrier`（断电可能丢数据）、`errors=panic`。
- hmdfs 每文件 2 个 dentry → Slab 膨胀；内存 31GB 中 Slab 占 ~3.8GB。

### 6.4 权限/路径坑（对 Node 应用的直接含义）
- npm 全局包装到 `~/.local/npm-global` 没问题，但**任何要 mmap MAP_SHARED 写输出的编译（go build）在 hmdfs 上失败**。
- 工具装 `$EL2_BASE/`（hmfs，无 hmdfs 坑）最稳。

---

## 7. 进程与性能特征（对服务型应用重要）

| 操作 | 鸿蒙 | Linux 同级 | 倍数 |
|---|---|---|---|
| getpid | 187 ns | ~150 ns | 1.25x |
| pipe 往返 | 1375 ns | ~800 ns | 1.7x |
| mmap+munmap | 1638 ns | ~500 ns | 3.3x |
| **fork+exit** | **2817 µs** | ~200 µs | **14x** |

fork 慢 14 倍原因：每次 fork 做 SELinux context 分配 + hmdfs MAC 标签 + 进程记账同步检查。

**对 dsh 的含义**：避免高频 fork/exec（如每命令 spawn 新 shell）会有明显成本；长驻进程 + 复用连接更划算。服务端 Web UI 场景影响小。

---

## 8. 开发与包管理生态

| 通道 | 覆盖 | 说明 |
|---|---|---|
| hnp（系统域） | 系统级二进制 | `$OHOS_SDK/`，SELinux 可信域，可执行；不可写 |
| ohpm（OpenHarmony 包管理器） | @ohos/* | 需华为云 registry |
| **harmonybrew**（Homebrew 移植） | 3000+ 预编译 | `~/.harmonybrew/Cellar/`，Node v26.5.0 由此而来 |
| cmd-pkgs（OpenHarmony 预编译仓库） | 1247 工具 | `~/usr/local/` |
| cargo（Rust ohos target） | Rust 工具 | 自动注入 `.note.ohos.ident` + `.codesign` |
| npm/pnpm | JS 生态 | 原生模块需现编（clang）；预编译 .node 大概率 dlopen 失败 |

**工具链关键点**：
- **只有 clang 15.0.4**，无 gcc/g++；node-gyp 需 `CC=clang CXX=clang++`。
- Rust 1.95.0 有官方 `aarch64-unknown-linux-ohos` target，编译自动签名。
- Go 现编（go build）在 hmdfs 上 mmap 失败；交叉编译（Mac）最稳。
- HarmonyOS SDK（DevEco Studio 配套）提供 clang/binutils/签名工具。

---

## 9. 对 dsh 的直接影响与建议

### 9.1 本会话验证的"必踩坑"汇总
1. **预编译 `.node` dlopen 被拒** → node-pty 必须现编；koffi/rolldown 预编译不可用。
2. **koffi 被静态 import**（sandbox-local→windows-acl）→ 所有平台启动都要加载 win32 依赖，建议改平台条件动态 import。
3. **硬链接禁止** → `link()` 原子写全部改 `rename()`（dsh 自己的 write 工具也中招）。
4. **`/tmp` 只读** → TMPDIR 需指向可写目录（如 `$DSH_HOME/tmp`）。
5. **无沙箱原语** → bwrap/landlock/userns 全部失效，只能 `danger-full-access` + 无审批；这是平台上限，不是 dsh 缺陷。
6. **HMR 需 `--expose-internals`** → 裸 `dsh web` 启动失败，需启动脚本直传。
7. **`process.platform='openharmony'`** → 所有 npm 平台分支（esbuild 0.21.5、@img/sharp-*、node-pty prebuild、@esbuild 等）都要按 openharmony-arm64 单独处理。

### 9.2 建议 dsh 代码层的平台兼容策略
- **原生模块加载统一走"先试预编译、失败回退现编"**，并在 openharmony 上文档明确建议现编。
- **原子写统一 `rename()`**（同目录原子，全平台兼容），消除 link 依赖。
- **win32-only 依赖改动态/平台条件 import**，避免拖垮非 Windows 启动。
- **HMR 设可降级/可关**，避免成为启动硬依赖。
- 可选的 `openHarmony` 平台识别：在 package.json 加 `os: ["openharmony"]` 或用 `libc` 字段区分 musl。

### 9.3 已知可工作的模式（供参考）
- 本机 clang 现编 C/C++/Rust → 可执行可加载 ✅
- musl 静态二进制 + `.note.ohos.ident` + Merkle `.codesign` → 可执行 ✅
- hnp 打包安装 → 进入可信域可执行 ✅
- 纯 JS 应用（Node 26.5.0 harmonybrew）→ 完全可用 ✅（dsh Web UI 已跑通）
- sharp WASM 兜底、`@img/sharp-wasm32` → 功能可用 ✅

---

## 10. 平台能力矩阵（开发速查）

| 能力 | 状态 | 备注 |
|---|---|---|
| Linux syscall ≤283 | ✅ | 基础齐全 |
| 现代 syscall ≥387 | ❌ SIGSYS | rseq/clone3/landlock/openat2 |
| user namespace / mount ns 自沙箱 | ❌ | appspawn 独占 |
| hardlink (linkat) | ❌ | 全盘禁止 |
| `/tmp` 可写 | ❌ | 需 TMPDIR 覆盖 |
| 预编译 `.node` dlopen | ❌ | 现编才行 |
| 本机 clang 现编 ELF | ✅ | 内核信任"本地出身" |
| glibc 二进制 | ❌ | musl only |
| go build (MAP_SHARED) | ❌ hmdfs | 交叉编译替代 |
| ptrace (gdb/strace) | ❌ | SELinux 禁 |
| 沙箱（landlock/bwrap/userns） | ❌ | 无此原语 |
| fork 性能 | ⚠️ 14x 慢 | 避免高频 fork/exec |
| Node.js (harmonybrew) | ✅ | v26.5.0 |
| 文件系统 hmfs | ✅ | 标准权限，工具首选 |
| ELF 签名（Merkle） | 需 | 外来二进制要 .note.ohos.ident+.codesign |

---

**总结**：鸿蒙对 dsh 是"用户态 95% 熟悉、内核态 60% 陌生"的平台。纯 JS 服务（Web UI）完全可跑；真正的坎全在**原生模块加载、沙箱原语缺失、文件系统怪癖**三处。dsh 若做 openharmony 一等公民支持，优先修：原子写用 rename、win32 依赖动态 import、原生模块现编回退、HMR 可关。
