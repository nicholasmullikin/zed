# Editing previously sent messages for external (ACP) agents

**Date:** 2026-05-29
**Status:** Design approved; pending implementation plan
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
- The ACP protocol (Zed's own `agent-client-protocol` crate, pinned at `=0.12.1` with
  `features = ["unstable"]` in `Cargo.toml`) has **no truncate / rewind / replay verb**.
  - `session/fork` (unstable, `unstable_session_fork`) duplicates the *whole* session
    context — its docs say it creates a branch "without affecting the original session's
    history," for side tasks like summaries. There is no "fork at message N." Not usable
    for rewinding.
  - `session/resume` reconnects to a paused session; `session/load` makes the agent replay
    its stored history back as `session/update` notifications. Neither edits history.
  - The only extensibility point on existing calls is the `_meta` field.

## What already exists in Zed (and can be reused)

The UI and thread plumbing for rewinding an ACP thread are already built; only the
connection method and the protocol verb behind it are missing.

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

The new work is therefore concentrated in (a) the protocol crate, (b) the Claude adapter,
and (c) a small wiring change in `AcpConnection` plus threading message IDs end to end.

## Key foundational gap: message IDs

`PromptRequest` already has an (unstable, `unstable_message_id`) `message_id: Option<String>`
field, and the spec says the agent SHOULD echo it back as `userMessageId` in
`PromptResponse`. **Zed does not currently populate it** — `acp_thread.rs` builds the request
with `acp::PromptRequest::new(session_id, message)` and tracks its own `UserMessageId`
separately, never putting it on the wire or correlating the echo.

Any rewind must reference "truncate to *this* message," so populating `PromptRequest.message_id`
from Zed's `UserMessageId` and consuming the echoed `userMessageId` is foundational and must
land regardless of the rewind mechanism.

## Design decisions

1. **Mechanism: a dedicated protocol method**, `session/truncate`, taking
   `{ session_id, message_id }`, gated behind a new `unstable_session_truncate` feature and a
   `SessionTruncateCapabilities` flag on `SessionCapabilities`. Rejected alternative: smuggling
   truncate through `_meta` (non-standard, avoids a schema bump but contradicts the "do it
   right" goal).
2. **Scope: capability-gated and generic.** Implement on the shared `AcpConnection`, gated on
   `agent_capabilities.session_capabilities.truncate.is_some()`, mirroring how `load`/`resume`/
   `close` are already handled (`supports_load_session`, `supports_resume_session`,
   `supports_close_session`). Any ACP agent advertising the capability enables automatically;
   Claude is simply first.
3. **Deliverable spans repos.** The protocol crate (`agent-client-protocol`) and the Claude
   adapter (`@agentclientprotocol/claude-agent-acp`, a Zed-published npm package outside this
   tree) are explicitly part of the deliverable. Approved by the requester.
4. **Lead with a feasibility spike** before committing to the build, since the adapter's
   current behavior is only knowable by probing the running process.

## Phases

### Phase 0 — Feasibility spike (decision gate)

Probe the **running** `@agentclientprotocol/claude-agent-acp` adapter (it lives outside this
repo). Answer:

- Does it echo `userMessageId` in `PromptResponse` when Zed sends `message_id`?
- Does it already expose any rewind/truncate today (via `_meta` or an undocumented method)?
- What does `session/load` replay look like, and is its transcript store something a truncate
  could trim?
- Confirm the adapter version (from the ACP registry,
  `https://cdn.agentclientprotocol.com/registry/v1/latest/registry.json`) and where its source
  lives, so Phase 2 is actionable.

**Output:** a short findings note and a go/no-go on the dedicated-method path. If the adapter
already exposes something usable, adapt the later phases to it; otherwise proceed to build
`session/truncate`.

### Phase 1 — Protocol (`agent-client-protocol` crate)

- Add the `unstable_session_truncate` feature, `SessionTruncateCapabilities`,
  `TruncateSessionRequest { session_id, message_id }`, `TruncateSessionResponse`, and the
  `session/truncate` method, following the shape of the existing `session/fork` and
  `session/resume` definitions in the schema.
- Bump Zed's pinned `agent-client-protocol` dependency to the version carrying the new verb.
- **Separable, foundational:** populate `PromptRequest.message_id` from Zed's `UserMessageId`
  and consume the echoed `userMessageId` in the `PromptResponse` handling
  (`crates/agent_servers/src/acp.rs` prompt flow + `crates/acp_thread/src/acp_thread.rs` send
  flow). This can land and be tested ahead of the truncate verb.

### Phase 2 — Claude adapter (`@agentclientprotocol/claude-agent-acp`)

- Implement `session/truncate`: drop all turns after `message_id` from the adapter's maintained
  context/transcript.
- Advertise `session_capabilities.truncate` in the initialize handshake.
- Ensure `message_id` is echoed as `userMessageId` (if Phase 0 found it missing).

### Phase 3 — Zed integration (this repo, small)

- Implement `AcpConnection::truncate()` to return an `AgentSessionTruncate` whose `run()` sends
  a `session/truncate` request and resolves when it completes — gated on
  `agent_capabilities.session_capabilities.truncate.is_some()`. Add a `supports_truncate`-style
  capability check alongside the existing `supports_load_session` / `supports_resume_session` /
  `supports_close_session` methods.
- This flips `AcpThread::supports_truncate(cx)` to `true` for capable agents, so the existing
  inline-edit + **Regenerate** UI and the existing `AcpThread::rewind()` path engage with no UI
  rewrite.
- Update the disabled-state tooltip copy in
  `crates/agent_ui/src/conversation_view/thread_view.rs` so the "not available yet" message
  only shows for agents that genuinely lack the capability.

### Phase 4 — UX and edge cases

- Cancel any in-flight prompt for the session before issuing `session/truncate`
  (`AcpThread::rewind` already cancels via the truncate path on the native side; confirm parity).
- When a message has no `message_id` (e.g. an older session loaded via `session/load` before
  IDs were threaded), keep the edit affordance disabled for that message rather than failing.
- Surface `session/truncate` RPC failures to the UI as a user-visible error, not a silent drop.
- Leave git-checkpoint behavior matching the native agent: the separate **Restore Checkpoint**
  button (driven by `Checkpoint.show`) is unaffected by this change.

### Phase 5 — Testing

- Extend the fake/test ACP agent in `crates/agent_servers/src/acp.rs` to advertise and honor
  `session/truncate`.
- Tests for: the rewind path end to end, capability gating (button hidden when unsupported),
  the `message_id` round-trip (sent on `PromptRequest`, echoed on `PromptResponse`), and the
  no-`message_id` edge case.
- Per CLAUDE.md, use GPUI executor timers (`cx.background_executor().timer`) rather than
  `smol::Timer::after` in any test that drives `run_until_parked()`.

## Non-goals

- The lossy "Zed-only session rebuild" (new session + replay) fallback is explicitly **not**
  pursued; the requester chose the protocol + adapter extension.
- No change to git-checkpoint capture/restore behavior beyond what already exists.
- No new UI design — this reuses the existing inline editor and Regenerate button.

## Open questions / risks

- Phase 0 may reveal the adapter cannot trim its transcript cleanly (e.g. context is compacted
  or summarized server-side); if so, the adapter work in Phase 2 grows and should be re-scoped.
- Version coordination: Zed pins `agent-client-protocol = "=0.12.1"`; the new verb requires a
  coordinated release of the protocol crate, the adapter, and the registry entry.

## Key references

- `crates/acp_thread/src/connection.rs` — `AgentConnection` trait, `AgentSessionTruncate` trait,
  `UserMessageId`.
- `crates/acp_thread/src/acp_thread.rs` — `AcpThread::rewind`, `supports_truncate`,
  `handle_session_update`, prompt/send flow, `Checkpoint`/`restore_checkpoint`.
- `crates/agent_servers/src/acp.rs` — `AcpConnection`, capability storage and
  `supports_load_session` / `supports_resume_session` / `supports_close_session`, prompt flow,
  test harness.
- `crates/agent/src/agent.rs` — `NativeAgentConnection::truncate`, `NativeAgentSessionTruncate`
  (the model to follow).
- `crates/agent/src/thread.rs` — `Thread::truncate` (native, in-process reference).
- `crates/agent_ui/src/conversation_view/thread_view.rs` — `editing_message`, `regenerate`,
  Regenerate button, disabled tooltip.
- `Cargo.toml` — `agent-client-protocol = { version = "=0.12.1", features = ["unstable"] }`.
