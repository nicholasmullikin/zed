# ACP truncate feasibility — spike findings (Task 1)

**Date:** 2026-05-29
**Spike for:** [2026-05-29-acp-edit-sent-messages-design.md](./2026-05-29-acp-edit-sent-messages-design.md), Task 1 of [the plan](../plans/2026-05-29-acp-edit-sent-messages.md)
**Method:** Inspected the live ACP registry and a shallow clone of the Claude adapter source (`github.com/agentclientprotocol/claude-agent-acp`).

## TL;DR

**Recommendation: GO on the dedicated `session/truncate` protocol verb (Plan A), but Plan B (adapter) is the real risk and is partially blocked on the Claude Agent SDK.**

- The adapter does **not** echo `message_id` today, and it does **not** correlate the ACP `message_id` with anything. Task 2 (thread `message_id` from Zed) is necessary but **not sufficient** — the adapter must additionally record an `acpMessageId → SDK message uuid` mapping.
- The underlying `@anthropic-ai/claude-agent-sdk` exposes **no rewind/truncate/edit/checkpoint API**. Its only session-control primitives are `resume` (continue full session), `forkSession` (branch the *whole* session), and `deleteSession` (drop the whole session). Reads are via `getSessionMessages` / `listSessions`.
- Therefore a clean "forget everything after message N" is **not achievable with the current SDK surface**. It requires either (a) a new rewind primitive in `@anthropic-ai/claude-agent-sdk`, or (b) a supported "trim the on-disk transcript, then resume" path. Both need Anthropic/Claude-Agent-SDK involvement. This should be confirmed with the SDK owners before committing to Plan B.
- Additional complication: the SDK performs **automatic context compaction** mid-turn (the adapter tracks `compactionInProgress`). Once a conversation has been compacted, a precise message-boundary truncation is harder still, because earlier turns may no longer exist as discrete, replayable messages.

## Step 1 — Exact adapter Zed launches

- Agent id: `claude-acp` (`crates/agent_servers/src/custom.rs:18`, `CLAUDE_AGENT_ID`).
- Registry: `https://cdn.agentclientprotocol.com/registry/v1/latest/registry.json` (`crates/project/src/agent_registry_store.rs:20`).
- Live registry entry for `claude-acp`:
  - name: "Claude Agent", version `0.39.0`
  - npx package: `@agentclientprotocol/claude-agent-acp@0.39.0`
  - repository: `https://github.com/agentclientprotocol/claude-agent-acp`
  - authors: Anthropic, Zed Industries, JetBrains
- A test in `crates/agent_servers/src/acp.rs:3234` references `0.32.0`; Zed bounds the npm version by build date via `bounded_npm_package_spec`, but the package and repo are the same.
- Adapter dependencies (from `package.json`): `@agentclientprotocol/sdk@0.22.1`, `@anthropic-ai/claude-agent-sdk@0.3.156`, `zod`. The adapter is a thin ACP↔SDK bridge; the Claude Agent SDK owns all session/transcript state.

## Step 2 — Does the adapter echo `message_id`?

**No.** Confirmed by source inspection of the cloned adapter at `0.39.0`:

- `grep -rn "messageId|userMessageId|message_id"` over `src/` returns **zero** matches. The adapter never reads `params.message_id` from `PromptRequest` and never sets `user_message_id` on `PromptResponse`.
- In `prompt()` (`src/acp-agent.ts:732`), the adapter generates its **own** per-message id and ignores the client's:
  ```ts
  const userMessage = promptToClaude(params);
  const promptUuid = randomUUID();
  userMessage.uuid = promptUuid;
  ```
  `promptUuid` is used internally only (a `pendingMessages` queue, and `message.uuid === promptUuid` at `src/acp-agent.ts:1162` to detect the prompt's own echo). It is persisted into the SDK transcript as the user message's uuid, but is never linked to the ACP `message_id`.

**Implication:** Task 2 (Zed sends `message_id`) lands cleanly and harmlessly, but on its own buys nothing until the adapter is changed to (a) honor/echo it and (b) remember which SDK transcript uuid it maps to.

## Step 3 — Can the adapter trim its context to a message?

**Not with the current SDK surface; only via SDK changes or fragile transcript editing.**

What the adapter has to work with (full `@anthropic-ai/claude-agent-sdk` import surface in `src/acp-agent.ts:47-67`):
`CanUseTool, deleteSession, getSessionMessages, listSessions, McpServerConfig, ModelInfo, ModelUsage, Options, PermissionMode, PermissionUpdate, Query, query, Settings, SDKAssistantMessageError, SDKMessageOrigin, SDKPartialAssistantMessage, SDKUserMessage, SlashCommand`.

- **No** `rewind`, `truncate`, `editMessage`, `revert`, `undo`, `setMessages`, `removeMessages`, or `checkpoint` is imported or used. Grep over `src/` confirms none exist.
- Session control primitives actually used:
  - `query(...)` with `resume: <sessionId>` — continue an existing session with its full transcript (`createSession`, `src/acp-agent.ts:1921`, `2150`).
  - `query(...)` with `resume: <sessionId>, forkSession: true` — `unstable_forkSession` (`src/acp-agent.ts:663-680`) branches the **entire** session; there is no "fork at message N".
  - `deleteSession` — removes a whole session.
  - `getSessionMessages(sessionId)` / `listSessions({dir})` — read the SDK-managed, file-based transcript. `replaySessionHistory` (`src/acp-agent.ts:1556`) iterates `getSessionMessages` and emits ACP `session/update` notifications, so `session/load` is pure agent-side replay.
- Advertised capabilities (`src/acp-agent.ts:615-637`): `loadSession`, and `sessionCapabilities: { additionalDirectories, close, delete, fork, list, resume }`. **No `truncate`.**

**Viable implementation paths for a real rewind, in order of preference:**

1. **New SDK rewind primitive.** Ask the `@anthropic-ai/claude-agent-sdk` owners (Anthropic) for a supported "resume session truncated at message uuid" (or equivalent). This is the clean path and the one Plan B should target. Requires SDK work outside both Zed and the adapter.
2. **Transcript trim + resume.** Because the transcript is file-based and uuid-keyed, the adapter could locate the session transcript, drop entries at/after the boundary uuid, then `resume`/`forkSession`. This is **unsupported and fragile**: it depends on the SDK tolerating externally-edited transcripts, and is undermined by automatic compaction (below).
3. **Fork-as-rewind is not viable** on its own: `forkSession` duplicates the full context, with no message boundary.

**Compaction caveat:** `prompt()` tracks `compactionInProgress` (`src/acp-agent.ts` ~760) because the SDK auto-compacts context within long turns. After compaction, earlier turns may not survive as discrete messages, so even a perfect id mapping cannot guarantee a precise truncation. Any rewind design must define behavior across a compaction boundary (e.g. refuse, or fall back to nearest surviving boundary).

## Consequences for the plan

- **Plan A (protocol verb):** unchanged and correct. Proceed when prioritized.
- **Task 2 (message_id threading):** still the right first step, but document that it is inert until the adapter records the id mapping. Worth landing regardless (spec-compliant, harmless).
- **Plan B (adapter):** **re-scope and gate on the Claude Agent SDK.** Before committing, raise with the SDK owners whether a rewind/resume-at-message primitive exists or can be added. If not, the only Zed-side-only alternative is the spec's explicitly-rejected lossy "new session + replay" rebuild — which the requester ruled out. Surface this to the requester as a decision point.
- **Capability gating (Plan C):** the generic, capability-gated approach is validated — the adapter already advertises `sessionCapabilities` flags (`close`, `resume`, `fork`, etc.), so adding a `truncate` flag fits the existing pattern exactly, and Zed branching on `session_capabilities.truncate.is_some()` mirrors `resume`/`close` handling.

## Artifacts

- Adapter source inspected: shallow clone of `github.com/agentclientprotocol/claude-agent-acp` @ `0.39.0` (removed after inspection).
- Key files: `src/acp-agent.ts` (capabilities `615-637`, `newSession`/`forkSession`/`resumeSession`/`loadSession` `651-705`, `prompt` `732`, `replaySessionHistory` `1556`, `createSession` `1933`), `package.json`.
