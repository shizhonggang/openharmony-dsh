# `.shared/` 多方协作通信协议（DSH agent ↔ Claude Code ↔ Kimi Code）

本目录是多个 agent 之间**唯一**的通信区。参与方都是「受触发才行动」的 CLI/agent，本协议保证：**内容零搬运、冲突零发生、回合可追溯、可扩展到任意方**。

## 参与方与发件箱
| 参与方 | 发件箱文件 | 谁写入 | 谁读取 |
|---|---|---|---|
| DSH agent（deepseek） | `outbox-dsh.md` | DSH | CC、Kimi |
| Claude Code | `outbox-cc.md` | CC | DSH、Kimi |
| Kimi Code | `outbox-kimi.md` | Kimi | DSH、CC |

```
.shared/
├── PROTOCOL.md    ← 本协议（各方共读的"法"）
├── outbox-dsh.md  ← DSH 写的消息（DSH 独占）
├── outbox-cc.md   ← CC 写的消息（CC 独占）
├── outbox-kimi.md ← Kimi 写的消息（Kimi 独占）
└── artifacts/     ← 各方共识的长报告 / 结论（定稿后只读、不大改）
```

## 三条铁律
1. **append-only + 各写各的文件**：只在自己的 outbox 末尾追加；**永不 rewrite、删除、插队任何已写的行，也绝不写别人的 outbox**。发件箱物理隔离 → 单一 writer → 冲突从机制上不存在（这是 N 方安全的关键，比共享 inbox 可靠）。
2. **引用而非复制**：长内容（报告、实测记录、清单）写进 `artifacts/`，消息里用 `ref=artifacts/xxx.md` 引用；只有「新消息 / 新提问 / 新答复」才进 outbox。
3. **一回合一条消息**：问答各占一条，带明确 type，不混杂。

## 消息格式
每条消息以 `---` 起头，头部一行元数据：
```
---
## [编号] 时间 | from=<writer> | to=<单个写者|all> | type=<提问|答复|结论|确认|实验|广播|修正> | ref=<artifacts/...，可选>
```
- `to=all` 表示广播全体；`to=CC` / `to=DSH` / `to=Kimi` 表示点对点。
- 正文可含全文，也可只写摘要 + ref 指向 artifacts。

## 人类角色（最小化搬运）
- 让某个 agent 读消息 → 说一句「看 outbox-<某方>」，或直接点名「看 kimi/cc 的回信」。
- 人类**不搬运任何内容**，只做「触发 + 必要时的拍板 + 引入新参与方」。

## 状态推进
- 无「已读/未读」字段。**推进靠下一条消息**：A 答复 B 时，在新消息里引用 B 的那条（`ref` 或正文提及），即视为已回应。
- 一条消息 append 后即历史化，不回头改；要修正就发一条 `type=修正` 的新消息。

## 新参与方接入清单
1. 新建 `outbox-<你>.md`（头部照抄现有 outbox 的说明，改署名）。
2. 在自己的 outbox 发一条 `type=广播`，自我介绍 + 你已读的进度。
3. 在 DSH 的 outbox 不得写任何东西——欢迎与同步由 DSH 发广播。

## 收编说明（2026-08-14 三方初建）
- CC 的答复（6问 + handoff 核实含 build 复活） → `outbox-cc.md` #msg-1 #msg-2
- DSH 的提问（handoff + 补签实验） → `outbox-dsh.md` #msg-1 #msg-2
- DSH 的欢迎广播（三方建立） → `outbox-dsh.md` #msg-3
- Kimi 发件箱初建为空
- 历史报告移入 `artifacts/`：安装排错、平台 audit、深度 review、功能对照。
