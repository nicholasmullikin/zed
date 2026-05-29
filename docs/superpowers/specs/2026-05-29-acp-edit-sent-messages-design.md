# Editing previously sent messages for external (ACP) agents

**Date:** 2026-05-29
**Status:** Design approved; realigned to upstream RFD [`session-rewind` (PR #1214)](https://github.com/agentclientprotocol/agent-client-protocol/pull/1214).
**Scope:** Capability-gated, generic across ACP agents. Claude Code (`@agentclientprotocol/claude-agent-acp`) is the first target.

## Problem

Zed's native agent lets a user re-focus a previously sent user message, edit it, and
hit **Regenerate**, which truncates every message after that point and re-sends. For
external agents reached over the Agent Client Protocol (ACP) — most notably Claude Code —
the edit affordance is deliberately disabled, showing a `PencilUnavailable` icon with the
tooltip *"Editing previous messages is not available for Claude Code yet."*

We want to enable the same edit-and-regenerate flow for ACP agents.

## Why it is disabled today

The blocker is architectural, not cosmetic:

- `AgentConnection::truncate()` (`crates/acp_thread/src/connection.rs`) returns
  `Option<Rc<dyn AgentSessionTruncate>>` and defaults to `None`. `AcpConnection`
  (`crates/agent_servers/src/acp.rs`, `impl AgentConnection for AcpConnection`) does not
  override it, so `supports_truncate(cx)` is `false` for every external agent.
- Each `acp::PromptRequest` carries **only the new user turn** — the agent (the Claude
  adapter) maintains its own conversation context server-side. Zed can truncate its own UI
  entries, but it has no way to tell Claude "forget everything after message N."
- The ACP protocol (the `agent-client-protocol` crate, pinned at `=0.12.1` with
  `features = ["unstable"]` in `Cargo.toml`) has **no rewind / edit / truncate verb** — true
  even on latest `main` (schema 0.13.4 / rust crate 0.12.1). The session verbs are
  `new / load / resume / fork / list / delete / close / prompt / cancel / set_* / update`.
  - `session/fork` (unstable) duplicates the *whole* session context for non-destructive
    side queries — there is no "fork at message N". Not a rewind.
  - `session/resume` reconnects to a session; `session/load` replays stored history as
    `session/update` notifications. Neither edits history.

## Upstream design already exists: RFD #1214 (Session Rewind)

The spike (see the feasibility findings doc) found an **open draft RFD** —
[`docs/rfds/session-rewind.mdx`, PR #1214](https://github.com/agentclientprotocol/agent-client-protocol/pull/1214) —
that proposes exactly this feature. We adopt its design rather than inventing our own verb.
The RFD proposes two methods, both gated behind a new `unstable_session_rewind` feature and
both building on `unstable_message_id`:

- **`session/rewind { sessionId, toMessageId }`** → `{ remainingMessageCount, lastMessageId }`:
  "treat history up to and including `toMessageId` as canonical, discard everything after."
  The next `session/prompt` continues from the rewound state. No implicit re-prompt. If a turn
  is active, the agent cancels it first. Filesystem rollback is explicitly out of scope.
- **`session/edit_prompt { sessionId, messageId, newContent }`** → same shape as
  `session/prompt` (streams via `session/update`): replace a **user** message's content and
  re-run its turn. Defined as an implicit `session/rewind` to the message before `messageId`,
  then replace + re-run with the original model/MCP/tools. **This is the direct mapping of
  "edit a previously sent message."**
- Capability discovery: `agentCapabilities.rewindSession { supported, supportsEditPrompt }`.

RFD status: **draft, "open for champion from the core team"** — designed, not yet accepted or
implemented. It is motivated by concrete Zed issues
([zed#52153](https://github.com/zed-industries/zed/issues/52153),
[zed#39997](https://github.com/zed-industries/zed/issues/39997),
[zed#28676](https://github.com/zed-industries/zed/issues/28676),
[zed#55888](https://github.com/zed-industries/zed/issues/55888)).
There is also an open `/rewind` issue on the adapter
([claude-agent-acp#460](https://github.com/agentclientprotocol/claude-agent-acp/issues/460)).

## What already exists in Zed (and can be reused)

The UI and thread plumbing for rewinding an ACP thread are already built; only the
connection method and the protocol verbs behind it are missing.

- `AcpThread::rewind(id: UserMessageId)` (`crates/acp_thread/src/acp_thread.rs`) already:
  calls `self.connection.truncate(&self.session_id, cx)`, truncates `self.entries` from the
  message index, emits `AcpThreadEvent::EntriesRemoved`, kills orphaned terminals from the
  removed entries, and calls `action_log().reject_all_edits(...)` to undo agent file edits
  made after the edit point. It already returns `Err("not supported")` when `truncate()`
  yields `None`.
- The inline message editor, `editing_message: Option<usize>` state, and the **Regenerate**
  button in `crates/agent_ui/src/conversation_view/thread_view.rs` are all gated on
  `self.thread.read(cx).supports_truncate(cx)` and `user_message.id.is_some()`. They will
  light up automatically once `supports_truncate` is `true`.
- The native agent is the model to follow: `NativeAgentConnection::truncate()` and
  `NativeAgentSessionTruncate` (`crates/agent/src/agent.rs`) return a working
  `AgentSessionTruncate` whose `run()` calls `Thread::truncate()` (`crates/agent/src/thread.rs`).

So Zed's existing `AgentConnection::truncate()` / `AcpThread::rewind()` hook stays — for ACP
agents it simply dispatches `session/rewind` (or `session/edit_prompt`) under the hood.

## Key foundational gap: message IDs (done)

`PromptRequest` already has an (unstable, `unstable_message_id`) `message_id: Option<String>`
field, and the agent SHOULD echo it as `userMessageId` in `PromptResponse`. **Zed did not
populate it.** Both `session/rewind` and `session/edit_prompt` reference messages by id, and
the RFD itself notes that *"without operations that consume message IDs, the flag has no
user-facing leverage."*

**Status: implemented.** `AcpThread::send` now sets `PromptRequest.message_id` from Zed's
`UserMessageId` (see the implementation plan, Task 2; landed on the fork). The Claude adapter
must still record an `acpMessageId → SDK transcript uuid` mapping to honor it.

## The native mechanism exists in the Claude Agent SDK

The spike confirmed `@anthropic-ai/claude-agent-sdk@0.3.156` already provides the rewind
primitive the adapter would call:

- `Options.resumeSessionAt` — resume a session "only ... up to and including the message with
  this UUID" (same session id).
- `forkSession(sessionId, { upToMessageId })` — slice the transcript up to a uuid into a new
  session.
- `rewindFiles(userMessageId)` + `enableFileCheckpointing` — native file restore (Zed keeps
  its own git checkpoint; this is optional).

So no new Anthropic primitive is required — the adapter wires existing SDK calls behind the
RFD methods.

## Design decisions

1. **Mechanism: adopt RFD #1214** — implement `session/rewind` (the primitive) and
   `session/edit_prompt` (the edit-message affordance), gated behind `unstable_session_rewind`
   and `unstable_message_id`. We do **not** invent a separate `session/truncate` verb;
   converging on the upstream proposal avoids ecosystem divergence and is the path to landing.
2. **Scope: capability-gated and generic.** Implement on the shared `AcpConnection`, gated on
   `agent_capabilities.rewindSession.supported` (and `.supportsEditPrompt`), mirroring how
   `load` / `resume` / `close` are already handled (`supports_load_session`,
   `supports_resume_session`, `supports_close_session`). Any ACP agent advertising the
   capability enables automatically; Claude is simply first.
3. **Deliverable spans repos.** The protocol crate (`agentclientprotocol/agent-client-protocol`
   + `agentclientprotocol/rust-sdk`) and the Claude adapter
   (`agentclientprotocol/claude-agent-acp`) are explicitly part of the deliverable, and the
   first dependency is the RFD being championed/accepted upstream.
4. **Filesystem rollback stays out of the protocol** (per the RFD). Zed keeps its existing git
   checkpoint / **Restore Checkpoint** behavior, layered separately.

## Phases

### Phase 0 — Feasibility spike (done)

Outcome: the SDK has a native rewind primitive (`resumeSessionAt` / `forkSession({upToMessageId})`),
the adapter ignores `message_id` today, and upstream RFD #1214 already specifies the protocol
surface. See the feasibility findings doc. Recommendation: align to the RFD.

### Phase 1 — Protocol (`agent-client-protocol`)

- Land RFD #1214 (champion + acceptance), then implement `unstable_session_rewind`:
  `RewindSessionRequest/Response`, `EditPromptRequest` (response aliases `PromptResponse`), the
  `session/rewind` and `session/edit_prompt` methods, and the `rewindSession` agent capability —
  following `AGENTS.md` conventions and the existing `session/fork` / `session/resume` shapes.
- Release the crate and bump Zed's pinned `agent-client-protocol = "=0.12.1"` to the new version.
- **Done in Zed already:** `PromptRequest.message_id` is populated (Task 2).

### Phase 2 — Claude adapter (`claude-agent-acp`)

- In `prompt()` (`src/acp-agent.ts:732`), record an `acpMessageId → SDK transcript uuid`
  mapping and echo `message_id` as `userMessageId` (the adapter currently ignores it).
- Implement `session/rewind` via the SDK's `resumeSessionAt` (preferred — keeps the session id)
  or `forkSession({ upToMessageId })`; implement `session/edit_prompt` as rewind-to-before +
  replace + re-run.
- Advertise `agentCapabilities.rewindSession` in the initialize handshake (sibling of the
  existing `resume`/`fork`/`close`/`delete`/`list` flags at `src/acp-agent.ts:629-637`).
- Validate behavior across an auto-compaction boundary (refuse, or fall back to the nearest
  surviving boundary).

### Phase 3 — Zed integration (this repo, small)

- Wire `AcpConnection::truncate()` to return an `AgentSessionTruncate` whose `run()` issues
  `session/rewind` (or `session/edit_prompt` for the edit case), gated on
  `agent_capabilities.rewindSession.supported`. Add the capability check alongside
  `supports_load_session` / `supports_resume_session` / `supports_close_session`.
- This flips `AcpThread::supports_truncate(cx)` to `true` for capable agents, so the existing
  inline-edit + **Regenerate** UI and `AcpThread::rewind()` path engage with no UI rewrite.
- Update the disabled-state tooltip in `crates/agent_ui/src/conversation_view/thread_view.rs`
  so the "not available yet" message only shows for agents that genuinely lack the capability.

### Phase 4 — UX and edge cases

- Per the RFD, the agent cancels any active turn before rewinding; confirm Zed's path matches.
- Keep the edit affordance disabled for any message whose `id` is `None` (e.g. older sessions
  loaded before ids were threaded).
- Surface `session/rewind` / `session/edit_prompt` RPC failures to the UI, not a silent drop.
- Leave git-checkpoint behavior matching the native agent: the separate **Restore Checkpoint**
  button (driven by `Checkpoint.show`) is unaffected (filesystem rollback is out of the
  protocol per the RFD).

### Phase 5 — Testing

- Extend the fake/test ACP agent in `crates/agent_servers/src/acp.rs` to advertise
  `rewindSession` and honor `session/rewind` / `session/edit_prompt`.
- Tests for: the rewind path end to end, capability gating (button hidden when unsupported),
  the `message_id` round-trip, and the no-`message_id` edge case.
- Per CLAUDE.md, use GPUI executor timers (`cx.background_executor().timer`) rather than
  `smol::Timer::after` in any test that drives `run_until_parked()`.

## Non-goals

- The lossy "Zed-only session rebuild" (new session + replay) fallback is explicitly **not**
  pursued.
- Filesystem rollback in the protocol — out of scope per the RFD; Zed's git checkpoint is
  unchanged.
- No new UI design — this reuses the existing inline editor and Regenerate button.

## Open questions / risks

- **Gating dependency is upstream, not technical.** RFD #1214 must be championed and accepted,
  then implemented in the protocol crate and the adapter, before Zed's Phase 3 can compile.
- Auto-compaction: rewinding across an SDK compaction boundary needs validation.
- Version coordination: Zed pins `agent-client-protocol = "=0.12.1"`; the new methods require a
  coordinated release of the protocol crate, the adapter, and the registry entry.

## Key references

- RFD: [`session-rewind.mdx` / PR #1214](https://github.com/agentclientprotocol/agent-client-protocol/pull/1214);
  related [`session-fork`](https://github.com/agentclientprotocol/agent-client-protocol/blob/main/docs/rfds/session-fork.mdx)
  and [`message-id`](https://github.com/agentclientprotocol/agent-client-protocol/blob/main/docs/rfds/message-id.mdx) RFDs.
- Motivating Zed issues: zed#52153, zed#39997, zed#28676, zed#55888.
- `crates/acp_thread/src/connection.rs` — `AgentConnection` trait, `AgentSessionTruncate` trait, `UserMessageId`.
- `crates/acp_thread/src/acp_thread.rs` — `AcpThread::rewind`, `supports_truncate`, `handle_session_update`, prompt/send flow, `Checkpoint`/`restore_checkpoint`.
- `crates/agent_servers/src/acp.rs` — `AcpConnection`, capability storage and `supports_load_session` / `supports_resume_session` / `supports_close_session`, prompt flow, test harness.
- `crates/agent/src/agent.rs` — `NativeAgentConnection::truncate`, `NativeAgentSessionTruncate` (the model to follow); `crates/agent/src/thread.rs` — `Thread::truncate`.
- `crates/agent_ui/src/conversation_view/thread_view.rs` — `editing_message`, `regenerate`, Regenerate button, disabled tooltip.
- SDK: `@anthropic-ai/claude-agent-sdk@0.3.156` `resumeSessionAt` / `forkSession({upToMessageId})` / `rewindFiles`.
