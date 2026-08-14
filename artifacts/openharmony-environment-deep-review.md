# 本机 HarmonyOS 运行环境深度剖析（Agent 工作基础设施手册）

日期：2026-08-14 · 作者：dsh Web UI 内运行的 coding agent（deepseek-v4-flash）
定位：为后续在此平台上的所有开发工作打底的基础设施知识库。**修正并深化了 [`deepseek-harness-issues.md`](./deepseek-harness-issues.md) 中的两处猜测。**

---

## 0. 一句话画像

> 这是一台 **HarmonyOS NEXT（PC 版）**，Linux 底座（HongMeng Kernel 1.13.0）+ 鸿蒙服务层。我作为 agent 运行在 **HiShell 应用沙箱**里（SELinux domain `hishell_hap` + seccomp mode-2 过滤器 + 鸿蒙自研文件系统），uid 20020075，20 核麒麟 CPU、32GB 内存、928GB f2fs 持久盘，网络完全开放。

---

## 1. 系统身份与硬件

| 项 | 值 | 说明 |
|---|---|---|
| 内核 | `HongMeng Kernel 1.13.0`（`uname` 报 `GNU/Linux`） | 鸿蒙自研内核，Linux ABI 兼容层 |
| 架构 | aarch64 | ARM64 |
| CPU | 20 核，`CPU implementer 0x48 part 0xd42` | 麒麟系（Hisilicon），支持 SVE2/PAuth/BTI（ACLE 全特性） |
| 内存 | 32GB 总量 / 16GB 可用 | 大内存，编译能力强 |
| 持久盘 | 928GB f2fs（userdata 分区） | 已用 137GB |
| libc | musl（无 `ldd`，`ldd` 命令不存在） | — |

---

## 2. 安全机制：三层 MAC 沙箱（本文档核心澄清）

我运行在一个**真实存在的、三层叠加的沙箱**里——这与旧文档"沙箱不可用"的表述需要澄清：**不可用的是 dsh 自己的用户态沙箱（bwrap/landlock/seatbelt），但鸿蒙平台的进程沙箱一直是开着的。**

### 第 1 层：SELinux（进程级 MAC）
- 当前 domain：**`u:r:hishell_hap:s0`**（`id -Z` 实测）
- 所有挂载点带 `seclabel`，文件带 SELinux 标签
- 效果：限制我这个 HAP 进程能访问的路径（如 `/data/local/tmp`、`/proc` 部分内容 `Permission denied`）

### 第 2 层：seccomp（系统调用级过滤）
- 实测 `/proc/self/status`：`Seccomp: 2`（SECCOMP_MODE_FILTER）+ `Seccomp_filters: 1`（appspawn 在 exec 前注入）
- **拦的是现代 syscall（编号 ≥387）**：rseq/clone3/landlock/openat2/mount_setattr 等被 SIGSYS 处死；`linkat`（syscall 37）在 ≤283 兼容范围内，**并非 seccomp 拦截目标**
- 能力集进一步佐证非标准环境：`CapEff/Prm/Amb = 0x2a`（仅 CAP_DAC_OVERRIDE + CAP_FOWNER + CAP_KILL），`CapBnd` 全开但运行中被沙箱收紧

### 第 3 层：文件系统（存储级限制）
- `hmdfs`（鸿蒙分布式文件系统）**自身不支持硬链接** → `Operation not permitted`（EOPNOTSUPP）
- `/tmp` 挂载在 **erofs 只读系统镜像** → 只读的根因（不是配置，是镜像分区只读）

### 硬链接禁令的精确双机制（修正旧文档 #12 的"疑似"）
| 场景 | 报错 | 真凶 |
|---|---|---|
| `$HOME` / `Documents` 下 `ln` | `Operation not permitted` | **hmdfs 不支持硬链接**（文件系统特性） |
| `DSH_HOME`（hmfs/f2fs）下会话保存 `link()` | `EACCES: permission denied` | **hmfs 文件系统层拒绝硬链接**（与 hmdfs 的 EPERM 错误码不同 → 非 seccomp，见下） |

> 结论：无论文件系统支不支持，硬链接在此平台**全局不可用**，原子发布只能 `rename()`。

---

## 3. 文件系统地图（对工作最实用）

### 挂载点分类

| 路径 | 文件系统 | 可写 | 持久性 | 用途 |
|---|---|---|---|---|
| `/`（根） | tmpfs | ✅ | ❌ 内存 | 运行时根 |
| `/usr/bin` `/usr/lib` `/usr/share` | erofs | ❌ | ✅ 只读镜像 | 系统程序 |
| `/tmp` | **erofs** | ❌ | — | 只读（勿用！） |
| `/system` `/vendor` `/cust` `/preload` `/version` | erofs | ❌ | ✅ 只读镜像 | 系统/厂商分区 |
| `/data/...（el0–el4）` | **f2fs** | ✅ | ✅ 持久 | 应用数据（加密分级） |
| `/data/service/...` | hmfs | ✅（部分） | ✅ | 系统服务数据 |
| **`$HOME`（=$HOME）** | **hmdfs** | ✅ | ✅（分布式，可同步） | 用户可见文件 |
| `$HOME/appdata` | sharefs | ❌（`Operation not permitted`） | ✅ | 应用私有数据 |
| `/mnt/hmdfs/100/cloud` | hmdfs | — | 分布式 | 云端目录（`df` 报 Function not implemented） |

### 关键路径定位（旧文档的速查表更新）

```text
用户主目录 $HOME      $HOME           ← hmdfs（分布式，重启不丢，可跨设备）
                    ⚠️ 主目录在 hmdfs 上，硬链接不可用

DSH_HOME             $EL2_BASE/dsh-home       ← f2fs（持久）
                    ⚠️ 会话持久化在此，但 hmfs 拒绝硬链接（报 EACCES）

临时文件              $TMPDIR = $HOME/tmp   ← 系统已约定（hmdfs）
                    另：$EL2_BASE/tmp（f2fs，可写）

dsh 源码仓库          $EL2_BASE/deepseek-harness   ← f2fs
harmonybrew 根        $EL2_BASE/.harmonybrew      ← f2fs
Claude Code 临时      $EL2_BASE/claude-tmp         ← f2fs
```

### 临时文件的正确姿势
- ✅ 环境变量 **`TMPDIR=$HOME/tmp`** 已默认指向可写目录，直接用它
- ✅ `$EL2_BASE/tmp`（f2fs）也可写，适合编译类临时产物
- ❌ 不要碰 `/tmp`（erofs 只读）、`/data/local/tmp`（SELinux 拒绝）

---

## 4. 工具链与包管理

### 编译器与构建
| 工具 | 位置 | 备注 |
|---|---|---|
| clang / clang++ | `$OHOS_SDK/bin` | 版本 15.0.4，鸿蒙官方工具链（含 ohos target：`aarch64-unknown-linux-ohos-clang`） |
| cmake / ninja | `$OHOS_SDK/bin` | 纯 clang 工具链，**无 gcc/g++** |
| make | `$HOME/usr/local/bin` | — |
| binutils | `$DATA/el2/base/.harmonybrew/opt/binutils` | objdump/readelf 在此 |

### 语言运行时
| 运行时 | 位置 | 版本 |
|---|---|---|
| Node.js | `.harmonybrew/bin/node` | v26.5.0 |
| Python3 | `.harmonybrew/bin/python3` | — |
| Rust | `$HOME/usr/rust-1.95.0-aarch64-unknown-linux-ohos` | 1.95.0（**含 ohos target！**） |
| Go | ❌ 未安装 | 可考虑补装 |

### 包管理：harmonybrew（Homebrew 移植）
- 根目录 `$EL2_BASE/.harmonybrew`，Cellar 约 80+ 包（binutils、cairo、glib、curl、croc、kimi-code、tailscale 等）
- 是**本机唯一成体系的包来源**，比裸 npm 更适合装 C 库
- `busybox` 在 `$OHOS_SDK/bin`，toybox 无

---

## 5. 网络

| 项 | 值 |
|---|---|
| 接口 | wlan0（活跃），lo，Hisilicon0 |
| 外网 | ✅ 完全开放（`https://www.baidu.com` HTTP 200，0.095s） |
| DNS | 114.114.114.114 / 8.8.8.8 |
| 特殊 | 有 tailscale 痕迹（`$DATA/el2/base/tailscale`）、croc（文件互传）、分布式云目录挂载 |

> 网络无障碍 → web 检索、`git clone` 外网、`npm install`、`brew install` 都能正常进行。

---

## 6. 修正记录（相对旧文档的深化）

| 旧文档表述 | 本次实测修正 |
|---|---|
| "硬链接…疑似 seccomp 或内核屏蔽 linkat" | **实锤为文件系统层拒绝（非 seccomp）**：hmdfs 报 EPERM、hmfs 报 EACCES，错误码因文件系统而异——若 seccomp 过滤则错误必一致；且 linkat=37 在 ≤283 兼容范围。正解仍是 `rename()` |
| "`/tmp` 只读"（未明原因） | 根因是 `/tmp` 挂载在 **erofs 只读系统镜像**（`/dev/block/dm-0`，4.3G 用满 100%） |
| "沙箱在此平台只能 full-access" | 澄清：**平台沙箱一直开着**（SELinux hishell_hap + seccomp），不可用的是 dsh 自身 userland 沙箱（bwrap/landlock 无后端） |
| "临时文件只能放 $HOME" | 精确化：`$TMPDIR` 已指向 `$HOME/tmp`（hmdfs），另有 f2fs 的 `base/tmp` 可选 |

---

## 7. 对后续 Agent 工作的操作原则（清单）

1. **临时文件**：用 `$TMPDIR`（默认已指向可写目录）或 `$DATA/el2/base/tmp`；禁 `/tmp`、`/data/local/tmp`。
2. **原子写文件**：一律 `rename()`（同目录）；**任何用到 `link()`/硬链接的工具在此平台都会失败**（包括 dsh 自带的 write 工具——已实测 EPERM）。
3. **编译**：只有 clang（`CC=clang CXX=clang++`），无 gcc；原生 .node 模块用 node-gyp 现编，预编译 .node dlopen 大概率被拒。
4. **装依赖**：优先 harmonybrew（C 库）→ pnpm/npm（JS）→ cargo（Rust，含 ohos target）；评估原生模块风险。
5. **持久性认知**：`$HOME`（hmdfs）与 `DSH_HOME`、`base`（f2fs）都持久；不要把重要数据放根 tmpfs。
6. **进程沙箱认知**：我是受限进程（SELinux domain + seccomp + 受限 capabilities），能否访问某路径、某 syscall 都可能被 OS 层拒，遇到 `Permission denied` 先想"是不是沙箱拦的"而非重试。
7. **性能资源**：20 核 / 32GB，可放心跑并行编译、多 subagent。

---

## 8. 可开拓方向（后续工作建议）

- **Go toolchain**：当前缺失，Go 对 aarch64-musl 支持良好，装一个可补全工具链。
- **ohos 原生 target 开发**：rust 1.95 已带 `aarch64-unknown-linux-ohos` target，clang 也有 ohos triple → 有机会写鸿蒙原生应用/库。
- **harmonybrew 扩充**：Cellar 已有 80+ 包，可作为稳定依赖来源，沉淀关键编译依赖（如 libvips 原生版替代 WASM sharp 提升图片性能）。
- **分布式能力利用**：hmdfs/云目录/tailscale/croc 提示这台机器有跨设备协同意图，未来可能做"多设备工作流"。
- **seccomp 边界测绘**：系统性地测绘 ≥387 现代 syscall 的封杀清单（rseq/clone3/landlock 等已确认），输出"可用/禁用 syscall 表"，为运行时选型（Bun/Go/新版 Node）提供硬约束。
