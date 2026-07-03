# Recon: Enterprise HITL Approval Console on AG-UI

Scope: `copilotkit/copilotkit` monorepo (cloned for this recon; the `.github` org-profile repo has no product code). Findings from direct code inspection, July 2026.

## Existing Primitives

**AG-UI interrupt mechanism (undocumented but real).** There is no dedicated `INTERRUPT` event type in the canonical event list (`skills/copilotkit-agui/references/protocol-spec.md:21-587`), but the actual runtime implements a native pause/resume contract that the skill docs don't describe: `RunFinishedEvent.outcome = {type:"interrupt", interrupts: Interrupt[]}` (`packages/runtime/src/agent/index.ts:1686-1693`), a `RunAgentInput.resume` field matched by `interruptId` on the next run (`:1039-1080`), and a server-side `ctx.interrupt(interrupts) → Promise<ResumeEntry[]>` (`:717-724`). Tool configs can declare `interrupt`/`interruptReason`/`interruptMessage` (`:277-281`). `Interrupt`/`ResumeEntry` types are imported straight from `@ag-ui/client` (`:20-23`), i.e. genuine upstream protocol surface, not a CopilotKit-only extension.

**LangGraph-native interrupt path.** `sdk-python/copilotkit/langgraph.py:23,456-489` wraps LangGraph's own `interrupt()`. Runtime-side, dedicated event types exist: `packages/runtime/src/agents/langgraph/events.ts:20-28,352-393` (`OnInterrupt`/`OnCopilotKitInterrupt`, `LangGraphInterruptEvent`). `copilotkit_lg_middleware.py:519-566` reconciles interrupted vs. completed tool calls in message history — production-grade edge-case handling.

**Frontend HITL hooks.** `useHumanInTheLoop` (`packages/react-core/src/hooks/use-human-in-the-loop.ts:64-138`, v2 core at `packages/react-core/src/v2/hooks/use-interrupt.tsx`) exposes a `respond(result)` callback keyed on `toolCallId`/`interruptId` (`packages/react-core/src/v2/types/interrupt.ts:1-58`, `human-in-the-loop.ts:1-55`). Legacy `renderAndWait`/`renderAndWaitForResponse` (`packages/react-core/src/types/frontend-action.ts:109-169`) are deprecated but still shimmed (`use-copilot-action.ts:161-169`). Worked approval-UI example: `showcase/shell-docs/.../useHumanInTheLoop.mdx:20-76`.

**Observability hooks already shipped.** `CopilotObservabilityConfig` (`packages/runtime/src/lib/observability.ts:39-76`) provides `handleRequest`/`handleResponse`/`handleError` hooks, wired into the resolver stream (`copilot.resolver.ts:685,728`) — currently scoped to LLM request/response logging, gated behind a CopilotKit Cloud API key (`observability.ts:96-99,158`), but the hook shape is directly reusable for interaction events. `DebugEventBus` (`packages/runtime/src/v2/runtime/core/debug-event-bus.ts:6-45`) is a pub/sub that already broadcasts every `BaseEvent` with `{agentId,threadId,runId}`, used today by the VS Code inspector.

**Central event chokepoint.** `packages/runtime/src/v2/runtime/handlers/shared/sse-response.ts` (`createSseEventResponse()`, `next:` handler lines 82-144) sees every AG-UI event before SSE encoding — the cleanest single insertion point for both audit logging and an export feed. The legacy v1 equivalent is `copilot.resolver.ts:296-298,458`.

**Telemetry (product usage, not interaction data).** Segment-based, anonymous, opt-out, 5% sampled: `packages/shared/src/telemetry/telemetry-client.ts:1,25-84`, event schema of 5 lifecycle-only events (`events.ts:1-16`) — no message/tool-call content.

## Gaps

1. **No structured multi-approver/escalation/delegation semantics.** `Interrupt[]` is a flat list of independent pending interrupts, not a quorum or approval-chain construct. No `approverId`, tier, or reassignment concept anywhere in the interrupt code path (confirmed via grep across `packages/` and `sdk-python`).
2. **No timeout/TTL on interrupts.** A paused run stays paused indefinitely until a `resume` is posted — no `expiresAt` field, no auto-escalation-on-timeout.
3. **Rejection carries no structured feedback.** `ResumeEntry.status` is `"resolved"|"cancelled"` only; a rejection reason would have to be smuggled into the free-form `payload`.
4. **State persistence is in-memory only in the OSS runtime.** `InMemoryAgentRunner` (`packages/runtime/src/v2/runtime/runner/in-memory.ts:36-49,79-81`) stores runs in a process-local `Map` — lost on restart. Durable persistence exists only via CopilotKit's hosted Intelligence Platform (`intelligence.ts:48`, external/opaque) or whatever LangGraph checkpointer a developer's own graph configures (not shipped by `sdk-python`).
5. **Multi-device/reconnect sync is hosted-platform-only.** `thread_run_activity` reconnect (tested in `CopilotChat.runActivityReconnect.test.tsx`) and the realtime thread WebSocket (`use-threads.tsx:71-72,188-192`) both depend on the Intelligence Platform's `wsUrl`. No equivalent for a self-hosted `InMemoryAgentRunner` deployment — a page refresh against a plain OSS runtime risks losing a pending approval's context entirely.
6. **No RBAC anywhere.** All `role`/`scope`/`permission` hits are unrelated (message role, action visibility, LLM error strings) — confirmed noise, not access control.
7. **No local authN/authZ.** Identity is delegated entirely to a developer-supplied `identifyUser` callback (`packages/runtime/src/v2/runtime/core/runtime.ts:151,158,171`), whose return value is shape-validated only (`resolve-intelligence-user.ts:15-23`) — no verification of the caller.
8. **No local multi-tenancy enforcement.** `organizationId` is passed through to the hosted platform but hardcoded empty in the OSS in-memory runner (`in-memory.ts:44,385`); real isolation lives in Cloud, not this repo.
9. **No audit logging exists anywhere** — a net-new build, though the hook point is clear (`packages/runtime/src/v2/runtime/handlers/intelligence/run.ts:249-262`, already switching on `event.type` with `userId`/`threadId`/`runId` in scope).
10. **No PII redaction.** Pino logger config redacts only `pid`/`hostname` (`packages/runtime/src/lib/logger.ts:18-21`); `copilot.resolver.ts:186` logs full request input including message content. **INFERENCE:** any audit/export write built on the current event stream will carry unredacted PII unless a redaction layer is added first.
11. **No LangSmith/OTel/Datadog export today.** LangSmith is present only as a hashed key attached to telemetry (`telemetry-agent-runner.ts:28-52`) — no tracing integration. OTel appears only as a transitive Python dependency, unused by CopilotKit code. Datadog is mentioned only in troubleshooting docs as a self-integration suggestion.
12. **Framework HITL maturity is uneven.** Only **LangGraph** has checkpoint-level native interrupt/resume (`langgraph.py:456-489`, dedicated runtime events, two working example apps). Microsoft Agent Framework (Python) has a native `approval_mode="always_require"` tool flag (**Partial**). Mastra, PydanticAI, MS Agent Framework .NET, Strands only demonstrate HITL via the generic frontend `useHumanInTheLoop` hook (**Demo-only** — no framework-native pause). CrewAI, Google ADK, LlamaIndex, Agno have **zero** HITL references in code, docs, or examples (**Unsupported**).

## Architecture Risks

- **Doc/reality drift on the protocol itself.** The AG-UI skill's `protocol-spec.md` was generated from an external `ag-ui/` checkout not present in this repo and predates the `Interrupt`/`ResumeEntry`/`outcome`/`resume` fields actually used by `packages/runtime` at the pinned `@ag-ui/*` version (0.0.57). Building an RFC on the documented spec alone would miss the real interrupt contract.
- **Version skew across the ecosystem.** JS packages pin `@ag-ui/core`/`client` at `0.0.57` (pre-1.0); Python pins `ag-ui-protocol >= 0.1.15` — a different numbering scheme entirely, and example apps show an inconsistent spread from `^0.0.35` to `0.0.57`. A protocol extension for approval semantics needs a versioning/compat story this repo doesn't currently enforce (one documented precedent: the `THINKING_*`→`REASONING_*` rename required client-side `BackwardCompatibility_0_0_45` shimming).
- **Durability and multi-device support are coupled to the hosted Intelligence Platform.** If the enterprise console must be self-hostable/on-prem (a plausible requirement given governance/PII sensitivity), the reconnect and persistence primitives that exist today won't transfer — they'd need to be rebuilt against a durable store (Postgres, Redis, or a LangGraph checkpointer) rather than reused.
- **The one existing observability hook is cloud-gated.** `CopilotObservabilityConfig` requires a CopilotKit Cloud API key (`observability.ts:96-99`) — reusing its shape for an export feed is fine, but the gating logic itself may need to be decoupled for a governance product that must work self-hosted.
- **Governance has literally nothing to build on.** Auth, RBAC, tenancy enforcement, and audit logging are all greenfield. This is a large, unbounded portion of the "governance features bundled in" scope and should be sized accordingly in the RFC.

## Open Questions for RFC

1. Does the Approval Console target the OSS self-hosted runtime, CopilotKit Cloud/Intelligence Platform, or both? This single decision determines whether persistence/multi-device sync is inherited or built from scratch.
2. Should multi-approver/escalation/delegation/timeout be proposed as new AG-UI protocol event types (upstream contribution) or implemented entirely at the application layer via `STATE_DELTA`/`CUSTOM` events on top of the existing single-interrupt primitive?
3. Given the framework maturity gap, is LangGraph the only v1 target, with other frameworks explicitly out of scope until they gain native interrupt support?
4. Where does audit-log data live — the same export feed as observability, or a separate, compliance-grade, access-controlled store with different retention/redaction rules?
5. Who owns PII redaction — a protocol-level concern (redact before emitting the event) or a console-side concern (redact on ingest)? This affects whether the fix belongs upstream in `packages/runtime` or downstream in the console.
6. Given the `@ag-ui/*` 0.0.x/0.1.x version skew, what's the compatibility contract for a product that must keep working as CopilotKit ships protocol changes underneath it?

## Suggested Build Sequence

1. **Durable state layer** — replace/augment `InMemoryAgentRunner` with a persistent thread/interrupt store; without this, even single-approver flows lose state on restart or refresh, which undermines the product's core premise.
2. **Structured rejection + timeout on the interrupt contract** — extend `ResumeEntry`/`Interrupt` usage with a reason field and TTL, validated end-to-end on the one mature framework (LangGraph) before generalizing.
3. **Audit log hook** at the `run.ts:249-262` subscription point (or its v2 equivalent), writing approval/rejection/edit events with `userId`/`threadId`/`runId` already in scope — plus a redaction pass before persistence.
4. **Policy layer for multi-approver/escalation/delegation** as an application-level state machine on top of the existing single-interrupt primitive (via `STATE_DELTA`/`CUSTOM`), since the protocol has no native quorum concept.
5. **Export feed** off the `sse-response.ts` chokepoint or `DebugEventBus` pattern, redacted, targeting OTel first (framework-neutral), then LangSmith/Datadog sinks.
6. **RBAC/tenancy** layered on top of `identifyUser`, since no access-control primitive exists today — likely the largest net-new component and worth scoping as its own RFC track.
