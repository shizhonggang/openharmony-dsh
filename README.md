# HarmonyOS × DeepSeek Harness — 平台适配与源码 build 复活实录

在 **HarmonyOS（openharmony aarch64 + musl, Node 26）** 上让 DeepSeek Harness（dsh）跑通并源码 build 的完整记录。由三个 agent（DeepSeek DSH / Claude Code / Kimi Code）经文件协议协作产出。

## 目录

```
PROTOCOL.md        三方 agent 协作的 outbox 协议（可复用于任意多方 agent 协作）
outbox-*.md        三方言谈完整留痕（dsh / cc / kimi）
README.md          本文件
artifacts/
  ├─ harmonyos-dsh-source-build-guide.md   ★ 可复现 build 指南（wasm 推荐 / Rust 备选）
  ├─ openharmony-environment-deep-review.md  鸿蒙运行环境深度剖析（含 ELF 补签实证）
  ├─ harmonyos-platform-audit.md           平台审计（内核/沙箱/syscall/文件系统）
  ├─ deepseek-harness-issues.md            安装排错全记录
  ├─ deepseek-harness-platform-findings.md 功能对照 + 待问清单
  ├─ harmonyos-dsh-source.patch            源码固化 patch（lightningcss → wasm + 依赖）
  └─ route-c-lightningcss-lazy.patch       兜底 patch（已归档，暂未采用）
```

## 核心结论（三句话）

1. **预编译 `.node` 缺 `.codesign` → dlopen 被拒**，用 SDK 的 `binary-sign-tool sign -selfSign 1` 批量补签即可（幂等）。
2. **lightningcss 官方预编译版因 `libgcc_s` ABI 段错误不可用**，推荐换 `lightningcss-wasm`（纯 WASM），备选本机 ohos Rust 源码编译。
3. **平台硬限制**：无内核沙箱（bwrap/landlock/seatbelt 全无）、无审批、禁硬链接、`/tmp` 只读——这些是天花板，代码层救不了，只能适配。

## 复现入口

从 `artifacts/harmonyos-dsh-source-build-guide.md` 开始；补丁脚本与工具见 `outbox-dsh.md` #msg-10。

> 注：文中绝对路径为本机部署快照，复现时按自身环境替换。
