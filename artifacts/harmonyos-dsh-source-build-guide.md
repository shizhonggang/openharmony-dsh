# 鸿蒙平台 dsh 源码 build 复活指南（可复现）

日期：2026-08-14 · 三方协作（DSH + Claude Code + Kimi Code）联合产出并复核
状态：**已跑通**（`pnpm run build` 在鸿蒙 exit 0，连续两次通过）
定位：未来升级 dsh、换机器、或新 agent 接手时，照着走通源码 build 的可复现手册。

---

## 0. 一句话结论

鸿蒙（openharmony aarch64, musl, Node 26）上 dsh 源码 build 卡住的两道坎，解法是：
1. **预编译 `.node` 缺 `.codesign` → dlopen 被拒** → 用 SDK 的 `binary-sign-tool` 批量补 self-sign。
2. **lightningcss 官方预编译版 `NEEDED libgcc_s.so.1` → 段错误（musl ABI 不匹配）** → 推荐换 `lightningcss-wasm`（纯 WASM）；备选本机 ohos Rust 源码编译（compiler-rt）。

---

## 1. 关键环境事实

| 项 | 值 |
|---|---|
| 平台 | `process.platform === 'openharmony'`，aarch64，musl |
| Node | v26.5.0（harmonybrew） |
| 编译器 | 仅 clang 15.0.4（`/data/service/hnp/bin`），无 gcc |
| Rust | 1.95.0，`aarch64-unknown-linux-ohos` target 已内置（`~/usr/rust-1.95.0-aarch64-unknown-linux-ohos/bin`） |
| 签名工具 | `/data/service/hnp/bin/binary-sign-tool`（ohos-sdk 26 自带） |
| binutils | `/data/storage/el2/base/.harmonybrew/opt/binutils/bin/readelf` |
| 源码树 | `/data/storage/el2/base/deepseek-harness` |

**两条铁律**（本平台，反复验证）：
- 硬链接（linkat）全盘禁止（文件系统层：hmdfs 报 EPERM、hmfs 报 EACCES），原子写一律 `rename()`。
- `/tmp` 是 erofs 只读镜像；临时文件用 `$TMPDIR` 或 `/data/storage/el2/base/tmp`。

---

## 2. 坎一：预编译 `.node` 缺签名 → 批量补 self-sign

### 根因
上游 `*-openharmony-arm64` 预编译产物带 `.note.ohos.ident` 但**无 `.codesign`**；鸿蒙内核要求动态加载的库（`mmap(PROT_EXEC)`）必须带 `.codesign`（Merkle 自洽哈希），否则 `dlopen` 报 `Permission denied`。本机 clang 现编的产物由工具链自动 self-sign，故能加载。

### 关键结论（CC 实测）
- **self-sign 与正式签名在 dlopen 层等价**（内核只查 Merkle 自洽，不验证书链）。
- **签名不绑定路径/机器**，补签产物可拷贝分发。
- `-selfSign 1` 模式只需 **3 个参数**：`-inFile -outFile -signAlg`，无需 keystore/cert/profile。

### 幂等批量补签（已签则跳过）
```sh
BT=/data/service/hnp/bin/binary-sign-tool
READELF=/data/storage/el2/base/.harmonybrew/opt/binutils/bin/readelf
find /data/storage/el2/base/deepseek-harness/node_modules/.pnpm -name "*.node" | while read -r f; do
  "$READELF" -S "$f" 2>/dev/null | grep -q ".codesign" && continue
  cp "$f" "$f.orig-bak"
  "$BT" sign -selfSign 1 -inFile "$f" -outFile "$f" -signAlg SHA256withECDSA
done
```
（deepseek-harness 树内共 9 个 .node，缺签名的正好 5 个：rolldown@1.0.3、oxc-parser、oxlint、rollup@4.62、oxc-resolver。）

> 也可用已固化的 `sign_native` 动态工具插件（本会话 `nsign-1`），一键 `sign_native(path)`。

---

## 3. 坎二：lightningcss → 官方预编译不可用（推荐 wasm，Rust 为备选）

### 根因（DSH 实测排除过捷径）
上游 `lightningcss.linux-arm64-musl.node` 改名+补签后，`NEEDED libgcc_s.so.1`；本机 python manylinux 轮子 vendored 的 `libgcc_s` 软链后 dlopen 依赖关能过，但 **transform 一执行即段错误（exit 139）** —— Rust unwinder 的 `libgcc_s`/`libc` ABI 与鸿蒙 musl 不匹配。**官方预编译版无解。**

### 方案①（推荐，CC 验证）：换 `lightningcss-wasm`
纯 WASM、同 API、同步、无 libgcc_s 依赖；cssModules 哈希/导出/minify 输出与原生完全一致。改动最小、可进 `pnpm patch`。
```sh
# 1. host 侧 stub 降级：tsdown 加载 config 时 lightningcss 急切 import 会崩，
#    编辑 .pnpm/lightningcss@1.32.0/node_modules/lightningcss/node/index.js 为：
#    native 双失败 → native=null → 惰性 throw 的 4 函数 stub
#    （transform / transformStyleAttribute / bundle / bundleAsync），vite 因此回退 postcss。

# 2. client 侧换 wasm
pnpm add -w lightningcss-wasm@1.32.0 --ignore-scripts
#    改 packages/client/tsdown.client.ts 第 16 行：
#    import { transform } from 'lightningcss'  →  from 'lightningcss-wasm'

# 3. 全量构建（host + client + web 全 EXIT=0）
pnpm run build
```

### 方案②（可选，Kimi 验证）：本机 ohos Rust 源码编译
原生产物性能更优，但需约 4.5 分钟编译 + 手改 node_modules，工程化成本高；**仅在 client 侧 CSS 处理成为性能瓶颈时启用**。
```sh
export LC=/data/storage/el2/base/kimi-tmp/lightningcss-src
git clone --depth 1 --branch v1.32.0 https://github.com/parcel-bundler/lightningcss.git "$LC"
cd "$LC"
mv rust-toolchain.toml rust-toolchain.toml.bak
export PATH="/storage/Users/currentUser/usr/rust-1.95.0-aarch64-unknown-linux-ohos/bin:$PATH"
cargo build -p lightningcss_node --release
SRC="$LC/target/release/liblightningcss_node.so"
DST=/data/storage/el2/base/deepseek-harness/node_modules/.pnpm/lightningcss@1.32.0/node_modules/lightningcss/lightningcss.openharmony-arm64.node
rm -f "$DST"; cp -a "$SRC" "$DST"; chmod +x "$DST"
# 验证：readelf -d 应仅 NEEDED [libc.so]；display-sign 应 self-sign
```

## 4. 跑通 build

```sh
cd /data/storage/el2/base/deepseek-harness
pnpm run build        # 期望 exit 0；产物在 packages/*/lib/index.js、apps/cli/lib/bin.js
```

---

## 5. 踩坑清单（防再踩）

1. 预编译 `.node` dlopen `Permission denied` = 缺 `.codesign`，不是权限位问题（chmod +x 无效）。
2. lightningcss 预编译版段错误 = `libgcc_s` ABI 不匹配，**别浪费时间去补 libgcc_s**，直接源码编译。
3. `/tmp` 只读、硬链接全禁 → 所有脚本/工具避开这两个平台特性。
4. 裸 `dsh web` 需 `node --expose-internals`；HMR 依赖该标志（不能放 NODE_OPTIONS）。

---

## 6. 遗留与上游建议（非阻塞）

1. **工程化**：推荐方案（stub + lightningcss-wasm）目前仍是手改 node_modules + 未提交的源码 diff，建议 `pnpm patch` 固化；上游提 issue 求官方 `openharmony-arm64` 预编译或 wasm 一等支持。
2. **补签流程**：已封装为 `sign_native` 工具 + 幂等 shell 脚本，可供 CI 复用。
3. **route c patch**（`artifacts/route-c-lightningcss-lazy.patch`）已被推荐方案的 stub 取代，可归档。

---

## 关联

- 完整问题记录：`deepseek-harness-issues.md`（安装排错）
- 平台深度剖析：`harmonyos-platform-audit.md`、`openharmony-environment-deep-review.md`
- 协作协议与三方言谈：`../PROTOCOL.md`、`../outbox-*.md`
