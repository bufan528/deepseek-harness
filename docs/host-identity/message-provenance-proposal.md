# DSH Host Identity Feasibility: Message-level Provenance (`via`) — Proposal A2

> Purpose: Host API feasibility review for third-party plugin memory authorization.  
> Baseline: DSH desktop **0.2.1-rc.5** (installed).  
> Status: Design proposal only — **not implemented**.  
> Channel: Repository Issues disabled (410); this PR is the formal review vehicle.

## Three questions for maintainers

1. Can a **trusted source handoff** be established between `Commands.prompt` and plugin message entry (Host-private context, not message-authored fields)?
2. Should unified injection live in **`Session.append`** or **`AgentLoop.send`**? How is forgery prevented?
3. How is **message-level provenance** persisted in the session event log and read **read-only** after fork/restore/restart?

## Why not Session-only (verified 0.2.1-rc.5)

- Desktop and QQ both use shared `agent.followup/steer`; followup does not identify the constructor.
- `session.append` only checks `source.kind`.
- Plugins also call `createUserMessage` — author-written `via` is forgeable.
- `rpcId` / `principalOf` do not prove desktop origin.
- `SessionHeader.origin` only allows `'subagent'`.

## Minimal contract

```ts
SessionHeader.sourceChannel?: 'session-controller' | 'plugin' | 'unknown'  // host create only
user message via?: 'host-desktop' | 'plugin' | 'unknown'                   // host intake only
```

Host stamps `via` at the **intake boundary** from a **private trusted context**; strips caller-supplied provenance. Plugins: **read-only**. Non-user host injections are **not** blanket-labeled `plugin`. Missing historical `via` → fail-closed deny for PC personal memory.

## Lifecycle

| Stage | Rule |
|---|---|
| create | host writes `sourceChannel` |
| intake | host stamps `via` from trusted context; strips forgeries |
| fork | historical via kept; new messages re-stamped; no escalation |
| restore/restart | read log only; never derive message via from Session header |
| migration | no via → missing; never bulk-label host-desktop |

## Plugin vs host

Plugins own QQ live context + owner binding + memory Owner Gate. Host provides only trusted provenance facts.

## Acceptance (minimal)

F1 desktop entry → host-desktop; F2 plugin entry → plugin; F3 forge → stripped; F4 append forge → no override; F5–F7 cross-source/group no escalation; F8–F9 fork/restore/restart; F10 missing → deny; F11 plugin without live QQ → deny; F12 after unbind → deny.

## Compatibility

Old messages without via: PC personal-memory stays disabled. New plugins + old host: PC memory off; QQ live path OK.

## Non-goals

No owner-binding / QQ / memory business logic in host. No A1 write-isolation ACL in first pass.

---

# 完整提案（中文 · FINAL）

基线 0.2.1-rc.5 · 未实施 · 三问见上。

现状：SessionHeader 无通道来源；桌面 create 仅 cwd/agentPreset；消息无权威 via；Desktop/QQ 共用 followup 不识别构造者；append 只校验 kind；rpcId/principalOf 不可作 PC 证明。

必要性：Session 创建 ≠ 当前消息；必须在可信上下文 → 入库边界钉 via，不信作者字段；QQ 也调用 createUserMessage，constructor 参数里的 via 不可当凭证。

注入：Commands.prompt 建立 Host 私有 Desktop 上下文（机制维护者选型，不要求 ALS）；入库边界（append 或 AgentLoop.send，PENDING_HOST）写 via 并剥离伪造；无上下文 → plugin/unknown。非 user 内部注入不机械标 plugin。

防伪：via 仅 Host 入库写；插件 via/sourceChannel 忽略剥离；Desktop 不靠消息字段自证；已写消息不可普通追加修改。

生命周期：via 随 user 消息持久化；fork 历史保留、新消息重签；restore/restart 只读日志；migration 无 via=missing。

结构：SessionHeader.sourceChannel + user source.via（可选 issuer/version）。改现有结构；不建双服务；不引入签名；插件只读。

模块：dsh-session；session-controller；append/AgentLoop.send（PENDING）；UI 可选。

插件职责：qqbot-merge 管 live QQ 与 binding；longterm-memory 管 Owner Gate 与记忆。宿主只供来源事实。

测试：F1–F12。兼容：旧消息 missing → PC DENY；新插件旧宿主 PC 仍关。非职责：Owner Binding/QQ/记忆业务；A1 隔离可选二期。

PENDING_HOST：append vs send 与可信上下文传递；freeze 消息并列写 provenance；非 user 是否需要 via（建议不需要）。

状态：可提交宿主评审；宿主实现等待评审；WP-2B 冻结。
