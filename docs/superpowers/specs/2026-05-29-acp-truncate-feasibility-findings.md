# ACP truncate feasibility — spike findings (Task 1)

**Date:** 2026-05-29
**Spike for:** [2026-05-29-acp-edit-sent-messages-design.md](./2026-05-29-acp-edit-sent-messages-design.md), Task 1 of [the plan](../plans/2026-05-29-acp-edit-sent-messages.md)
**Method:** Inspected the live ACP registry; a shallow clone of the Claude adapter source (`github.com/agentclientprotocol/claude-agent-acp@0.39.0`); the published `@anthropic-ai/claude-agent-sdk@0.3.156` type definitions (`sdk.d.ts`); and the latest `main` branches of `agentclientprotocol/agent-client-protocol` + `agentclientprotocol/rust-sdk` plus their open issues/PRs.

> **Two revisions, both after deeper review:** (1) an earlier version concluded the SDK had "no rewind primitive" — **wrong**, it was based only on what the *adapter imports*; the SDK's `sdk.d.ts` has native primitives. (2) a second pass of the protocol/adapter repos found an **open draft RFD that already specifies the protocol surface** for this feature, so the design should align to it rather than invent a verb.

## TL;DR

**Recommendation: GO, and align to upstream [RFD #1214 "Session Rewind"](https://github.com/agentclientprotocol/agent-client-protocol/pull/1214).** Both halves of the stack are further along than first thought: the protocol has a drafted design, and the SDK already has the implementing primitive.

- **Upstream protocol design exists:** RFD #1214 (open, **draft, "open for champion from the core team"**) proposes `session/rewind { sessionId, toMessageId }` and `session/edit_prompt { sessionId, messageId, newContent }`, gated `unstable_session_rewind`, built on `unstable_message_id`, with an `agentCapabilities.rewindSession { supported, supportsEditPrompt }` flag. It is motivated by Zed issues #52153 / #39997 / #28676 / #55888. The adapter has an open `/rewind` issue (#460) too. **We adopt this design** instead of inventing `session/truncate`.
- **SDK primitive exists:** `resumeSessionAt` (resume truncated to a message uuid, same session id) and `forkSession(sessionId, { upToMessageId })` (slice to a uuid into a new session). Plus `rewindFiles(userMessageId)` + `enableFileCheckpointing` for native file restore.
- The adapter surfaces none of these today and does **not** echo/map the ACP `message_id`. Remaining work is bounded: land RFD #1214 in the protocol crate, have the adapter map `acpMessageId → SDK transcript uuid` and implement the methods via `resumeSessionAt`, then wire Zed (Plan C).
- One caveat to validate, not assume: the SDK auto-compacts context mid-turn (the adapter tracks `compactionInProgress`). Rewinding across a compaction boundary needs testing, though these are first-class SDK features.

## Step 1 — Exact adapter Zed launches

- Agent id: `claude-acp` (`crates/agent_servers/src/custom.rs:18`, `CLAUDE_AGENT_ID`).
- Registry: `https://cdn.agentclientprotocol.com/registry/v1/latest/registry.json` (`crates/project/src/agent_registry_store.rs:20`).
- Live registry entry for `claude-acp`: name "Claude Agent", version `0.39.0`, npx `@agentclientprotocol/claude-agent-acp@0.39.0`, repo `github.com/agentclientprotocol/claude-agent-acp`, authors Anthropic / Zed Industries / JetBrains. (A test at `crates/agent_servers/src/acp.rs:3234` references `0.32.0`; Zed bounds the npm version by build date, same package/repo.)
- Adapter deps: `@agentclientprotocol/sdk@0.22.1`, `@anthropic-ai/claude-agent-sdk@0.3.156`, `zod`. The adapter is a thin ACP↔SDK bridge; the Claude Agent SDK owns all session/transcript state.

## Step 2 — Does the adapter echo `message_id`?

**No.** `grep -rn "messageId|userMessageId|message_id"` over the adapter `src/` returns zero matches. In `prompt()` (`src/acp-agent.ts:732`) it ignores `params.message_id` and generates its own:
```ts
const userMessage = promptToClaude(params);
const promptUuid = randomUUID();
userMessage.uuid = promptUuid;          // persisted as the transcript user-message uuid
```
`promptUuid` is used only internally (a pending-message queue; `message.uuid === promptUuid` at `src/acp-agent.ts:1162`). It is never linked to the ACP `message_id`.

**Implication:** Task 2 (Zed sends `message_id`) is correct and harmless but inert until the adapter records the id↔uuid mapping needed to pick a rewind point.

## Step 3 — Native rewind mechanism in the SDK

The full `@anthropic-ai/claude-agent-sdk@0.3.156` surface (`sdk.d.ts`) contains, verbatim:

- **`Options.resumeSessionAt?: string`** (`sdk.d.ts:1703-1707`):
  > "When resuming, only resume messages up to and including the message with this UUID. Use with `resume`. This allows you to resume from a specific point in the conversation. The message ID should be from `SDKAssistantMessage.uuid`."
- **`forkSession(sessionId, options)`** (`sdk.d.ts:654-684`):
  > "Fork a session into a new branch with fresh UUIDs... Supports `upToMessageId` for branching from a specific point in the conversation."
  with `ForkSessionOptions.upToMessageId?: string` — "Slice transcript up to this message UUID (inclusive). If omitted, full copy." Returns a new `sessionId` resumable via `query({ options: { resume } })`.
- **`Query.rewindFiles(userMessageId, { dryRun? })`** (`sdk.d.ts:2291-2300`) + **`Options.enableFileCheckpointing`** (`sdk.d.ts:1406-1413`):
  > "Rewind tracked files to their state at a specific user message. Requires file checkpointing."

Transcript model: messages are uuid-keyed with a `parentUuid` chain; the SDK persists per-session transcripts read via `getSessionMessages` / `listSessions` (used by the adapter's `replaySessionHistory`, `src/acp-agent.ts:1556`).

**What the adapter uses today (and the gap):** it uses `query` with `resume: sessionId` and the whole-session `Options.forkSession: boolean` flag (`unstable_forkSession`, `src/acp-agent.ts:663`). It does **not** use `resumeSessionAt`, the standalone `forkSession({upToMessageId})` function, or `rewindFiles`. So neither point-in-time primitive is exposed over ACP yet.

## Step 4 — Upstream protocol design: RFD #1214

Searching the protocol/adapter repos surfaced an open draft RFD that specifies exactly this feature, so the design should converge on it rather than invent a parallel verb.

- **`agent-client-protocol` PR #1214 — "docs(rfd): Session Rewind"** (open; author `htahaozlu`; `docs/rfds/session-rewind.mdx`). Proposes two methods, both gated `unstable_session_rewind`, both depending on `unstable_message_id`:
  - **`session/rewind { sessionId, toMessageId }` → `{ remainingMessageCount, lastMessageId }`** — "treat history up to and including `toMessageId` as canonical, discard everything after." Next `session/prompt` continues from there; no implicit re-prompt; agent cancels an active turn first; filesystem rollback explicitly out of scope.
  - **`session/edit_prompt { sessionId, messageId, newContent }` → same shape as `session/prompt`** — replace a **user** message's content and re-run its turn (= rewind-to-before + replace + re-run with the original model/MCP/tools). This is the direct mapping of "edit a previously sent message."
  - Capability: `agentCapabilities.rewindSession { supported, supportsEditPrompt }`.
- **Status:** draft, explicitly "open for champion from the core team" — designed, not accepted or implemented. Consolidates discussions #239 (checkpoint restore) and #329 (`session/undo`/`redo`). Motivated by Zed issues #52153, #39997, #28676, #55888.
- **Released code, latest `main`:** no rewind/edit/truncate verb in the protocol (schema 0.13.4 / rust crate 0.12.1) or the adapter (0.39.0). Adapter issue #460 "/rewind" is an empty placeholder. So nothing is *shipped*, but the design is drafted and the SDK primitive is ready.

## Recommended mechanism

Adopt RFD #1214's two methods (do not invent `session/truncate`), backed by the SDK primitive:

1. **`session/rewind` via `resumeSessionAt` (preferred).** Adapter handles `session/rewind { sessionId, toMessageId }` by starting the next `query` with `resume: <sdkSessionId>, resumeSessionAt: <transcript uuid for toMessageId>`. Keeps the ACP session id stable.
   - Requires the adapter to map the ACP `message_id` (now sent by Zed) to the right transcript uuid. `resumeSessionAt` wants an `SDKAssistantMessage.uuid`; to rewind to "before user message N" the adapter resumes up to the assistant message preceding N. It already observes SDK messages during `prompt()` and can record, per ACP user message, the surrounding transcript uuids.
2. **`session/edit_prompt`** = the above rewind to the message before `messageId`, then replace its content and re-run the turn (streaming via `session/update`).
3. **Fork-at-message (alternative for rewind).** `forkSession(sdkSessionId, { upToMessageId })` slices to a uuid and yields a new session id — but the ACP session id changes, so Zed's bookkeeping must re-key. `resumeSessionAt` avoids that.
4. **File restore (optional, native).** `enableFileCheckpointing` + `rewindFiles(userMessageId)` could restore the working tree natively at the rewind point; the RFD keeps filesystem rollback out of the protocol, so Zed's git checkpoint stays as-is and this is purely optional.

## Consequences for the plan

- **Plan A (protocol):** champion/accept RFD #1214, then implement `session/rewind` + `session/edit_prompt` + the `rewindSession` capability behind `unstable_session_rewind`. No new verb invented.
- **Task 2 (message_id threading):** correct first step; the RFD itself names it as the prerequisite ("without operations that consume message IDs, the flag has no user-facing leverage"). Inert until the adapter records the id↔uuid mapping. Landed.
- **Plan B (adapter):** **feasible now, not blocked.** Scope: record the id↔uuid mapping in `prompt()`, advertise `agentCapabilities.rewindSession`, and implement the methods via `resumeSessionAt` (or `forkSession({upToMessageId})`). Validate behavior across a compaction boundary.
- **Plan C (Zed wiring):** the generic capability-gated approach is validated — the adapter already advertises `sessionCapabilities` flags (`close`, `resume`, `fork`, `delete`, `list`), so a `rewindSession` flag and Zed branching on it mirror the existing `resume`/`close` handling exactly.

## Cross-repo reality (separate from feasibility)

Feasibility is confirmed and a design exists, but the work still lives outside this checkout, and the first dependency is **social, not technical**: RFD #1214 must be championed/accepted upstream. Then `session/rewind` + `session/edit_prompt` land in `agentclientprotocol/agent-client-protocol` (+ `rust-sdk`, the `agent-client-protocol` crate Zed pins at `=0.12.1`), the handler lands in the `claude-agent-acp` npm repo, and only then can Zed's Plan C compile. None of these can be built from the Zed monorepo working tree.

## Artifacts

- Adapter source: shallow clone of `github.com/agentclientprotocol/claude-agent-acp@0.39.0` (removed after inspection).
- SDK types: `@anthropic-ai/claude-agent-sdk@0.3.156` `sdk.d.ts` (fetched from jsDelivr). Key lines: `forkSession` 654-684, `resumeSessionAt` 1703-1707, `forkSession` boolean flag 1429, `rewindFiles` 2291-2300, `enableFileCheckpointing` 1406-1413.
- Adapter key files: `src/acp-agent.ts` (capabilities 615-637, session methods 651-705, `prompt` 732, `replaySessionHistory` 1556, `createSession` 1933), `package.json`.
- Protocol repos (latest `main`): `agentclientprotocol/agent-client-protocol` (schema 0.13.4; session verbs `new/load/resume/fork/list/delete/close/prompt/cancel/set_*/update`; no rewind in v1 or v2 unstable) and `agentclientprotocol/rust-sdk` (`agent-client-protocol` crate 0.12.1). RFD: PR #1214 `docs/rfds/session-rewind.mdx`; related `session-fork` / `message-id` RFDs; v2 overview lists "Truncate/Edit support" as a goal.
