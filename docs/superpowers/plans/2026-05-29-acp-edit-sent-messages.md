# Editing previously sent messages for external (ACP) agents — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let users edit a previously sent user message and regenerate from that point when talking to external ACP agents (Claude Code first), by making the agent's server-side session forget everything after the edited message.

**Architecture:** Reuse Zed's existing inline-edit + Regenerate UI and `AcpThread::rewind()` path, which already call `AgentConnection::truncate()`. The missing pieces are: (a) a new `session/truncate` verb in the `agent-client-protocol` crate, (b) its implementation in the Claude adapter, and (c) an `AcpConnection::truncate()` wiring that calls it, gated on an advertised capability. A foundational prerequisite — sending the client's `message_id` on each prompt — is buildable in this repo today and lands first.

**Tech Stack:** Rust, GPUI, the `agent-client-protocol` crate (Zed-published), the `@agentclientprotocol/claude-agent-acp` npm adapter (Zed-published, outside this tree).

---

## Scope note: this plan is gated and spans repos

The full feature has a hard dependency chain:

```
Task 1 spike  →  protocol crate verb  →  Claude adapter  →  Zed truncate-wiring (UI lights up)
                                                              ▲
Task 2 message-ID threading (this repo, no upstream dep) ────┘ (prerequisite)
```

Only **Task 1** (a research spike) and **Task 2** (message-ID threading) can be written as concrete, compiling work against the current tree. The protocol verb requires bumping the pinned `agent-client-protocol = "=0.12.1"` to a version that does not yet exist; the adapter lives in a separate npm repo. Those phases are captured as the **Gated follow-on roadmap** at the end and become their own plans once Task 1 resolves and the protocol release lands. This split follows the writing-plans scope-check guidance: each plan must produce working, testable software on its own.

---

## File Structure (this plan)

- `docs/superpowers/specs/2026-05-29-acp-truncate-feasibility-findings.md` — **Create.** Written deliverable of the Task 1 spike.
- `crates/acp_thread/src/connection.rs` — **Modify.** Add a `Display` impl for `UserMessageId` so its UUID string can be put on the wire.
- `crates/acp_thread/src/acp_thread.rs` — **Modify.** In `AcpThread::send`, populate `PromptRequest.message_id` from the client `UserMessageId`. Add a regression test in the existing `mod tests`.

---

## Task 1: Feasibility spike — probe the running Claude adapter

This is a **research task**, not TDD. Its output is a written findings note plus a go/no-go on the dedicated-`session/truncate`-verb approach. Do not write production code in this task.

**Files:**
- Create: `docs/superpowers/specs/2026-05-29-acp-truncate-feasibility-findings.md`

- [ ] **Step 1: Identify the exact adapter version Zed launches**

Find the pinned/registry version of `@agentclientprotocol/claude-agent-acp`.

Run:
```bash
cd /home/nick-mullikin/src/zed
grep -rn "claude-agent-acp\|claude-acp\|CLAUDE_AGENT_ID" crates/agent_servers/src crates/project/src
curl -s https://cdn.agentclientprotocol.com/registry/v1/latest/registry.json | grep -i claude
```
Expected: the npm package spec and a resolved version (e.g. `@agentclientprotocol/claude-agent-acp@0.32.0`). Record the version and the adapter's source repository URL in the findings note.

- [ ] **Step 2: Confirm whether the adapter echoes `message_id`**

The protocol says the agent SHOULD echo a client-sent `message_id` as `userMessageId` in `PromptResponse`. Determine empirically whether the current adapter does. Easiest path: read the adapter source at the version from Step 1 (clone its repo) and grep for `messageId` / `userMessageId` / `message_id` handling in its prompt path. If source inspection is inconclusive, run the adapter locally over stdio and send a `session/prompt` with a `messageId`, observing the response.

Record: does it echo? does it assign its own id when none is sent?

- [ ] **Step 3: Determine whether the adapter can trim its context to a message**

Read the adapter source for how it stores conversation/context per session and whether anything (an `_meta` hook, an undocumented method, or `session/load` rebuild logic) could drop turns after a given message. Specifically answer:
- Does it keep a per-session transcript it could truncate, or is context compacted/summarized server-side (which would make clean truncation hard)?
- What does `session/load` replay? Could a truncated transcript be re-loaded to achieve a rewind without a new verb?

- [ ] **Step 4: Write the findings note and a recommendation**

Create `docs/superpowers/specs/2026-05-29-acp-truncate-feasibility-findings.md` answering Steps 1–3, plus an explicit recommendation:
- **Go (dedicated verb):** proceed to design `session/truncate` in the protocol crate (the spec's primary path).
- **Adapt:** the adapter already exposes something usable — describe how to wire to it instead.
- **Blocked:** the adapter cannot trim context cleanly — re-scope the adapter work and flag it to the requester.

- [ ] **Step 5: Commit the findings note**

```bash
cd /home/nick-mullikin/src/zed
git add docs/superpowers/specs/2026-05-29-acp-truncate-feasibility-findings.md
git commit -m "acp_thread: Record ACP truncate feasibility findings

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

**Decision gate:** the recommendation in Step 4 determines the shape of the Gated follow-on roadmap. Task 2 below is independent of the outcome and can proceed in parallel.

---

## Task 2: Send the client `message_id` on every prompt (foundation)

Zed creates a `UserMessageId` for each user message and stores it on the thread entry, but never puts it on the `PromptRequest`. Every rewind references "truncate to *this* message," so the id must travel to the agent. The `message_id` field already exists on `PromptRequest` (gated by the `unstable_message_id` feature, which Zed enables workspace-wide via `agent-client-protocol = { features = ["unstable"] }`). This task only needs to populate it.

This change is safe to merge ahead of the rest: an agent that ignores `message_id` is unaffected.

**Files:**
- Modify: `crates/acp_thread/src/connection.rs` (add `Display` for `UserMessageId`, near the `impl UserMessageId` at lines 19-23)
- Modify: `crates/acp_thread/src/acp_thread.rs:2324-2327` (`AcpThread::send`)
- Test: `crates/acp_thread/src/acp_thread.rs` (existing `mod tests`, alongside `test_send_assigns_message_id_without_truncate_support` at line 5425)

- [ ] **Step 1: Write the failing test**

Add this test in `crates/acp_thread/src/acp_thread.rs` inside `mod tests`, immediately after `test_send_assigns_message_id_without_truncate_support`:

```rust
    #[gpui::test]
    async fn test_send_populates_message_id_on_prompt_request(cx: &mut TestAppContext) {
        init_test(cx);

        let fs = FakeFs::new(cx.executor());
        let project = Project::test(fs, [], cx).await;

        let captured: Rc<RefCell<Option<String>>> = Rc::new(RefCell::new(None));
        let connection = Rc::new(FakeAgentConnection::new().on_user_message({
            let captured = captured.clone();
            move |params, _thread, _cx| {
                *captured.borrow_mut() = params.message_id.clone();
                async move { Ok(acp::PromptResponse::new(acp::StopReason::EndTurn)) }
                    .boxed_local()
            }
        }));

        let thread = cx
            .update(|cx| {
                connection.new_session(project, PathList::new(&[Path::new(path!("/test"))]), cx)
            })
            .await
            .unwrap();

        thread
            .update(cx, |thread, cx| thread.send_raw("hello", cx))
            .await
            .unwrap();

        let stored_id = thread.read_with(cx, |thread, _| {
            let AgentThreadEntry::UserMessage(message) = &thread.entries[0] else {
                panic!("expected first entry to be a user message")
            };
            message.id.clone().expect("user message should have an id")
        });

        let sent = captured.borrow().clone();
        assert_eq!(
            sent.as_deref(),
            Some(stored_id.to_string().as_str()),
            "prompt request should carry the same message_id stored on the entry"
        );
    }
```

- [ ] **Step 2: Run the test to verify it fails**

Run:
```bash
cd /home/nick-mullikin/src/zed
cargo test -p acp_thread test_send_populates_message_id_on_prompt_request
```
Expected: FAIL. Two possible failures, both acceptable as "red": a compile error on `stored_id.to_string()` (no `Display` for `UserMessageId` yet), or, if it compiles, an assertion failure showing `sent == None` because `send` never sets `message_id`.

- [ ] **Step 3: Add `Display` for `UserMessageId`**

In `crates/acp_thread/src/connection.rs`, directly after the `impl UserMessageId { pub fn new() ... }` block (ends at line 23), add:

```rust
impl std::fmt::Display for UserMessageId {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        std::fmt::Display::fmt(&self.0, f)
    }
}
```

- [ ] **Step 4: Populate `message_id` in `AcpThread::send`**

In `crates/acp_thread/src/acp_thread.rs`, replace these three lines (currently 2324-2327):

```rust
        let request = acp::PromptRequest::new(self.session_id.clone(), message.clone());
        let git_store = self.project.read(cx).git_store().clone();

        let message_id = UserMessageId::new();
```

with (note `message_id` is created first so it can be attached to the request):

```rust
        let message_id = UserMessageId::new();
        let request = acp::PromptRequest::new(self.session_id.clone(), message.clone())
            .message_id(message_id.to_string());
        let git_store = self.project.read(cx).git_store().clone();
```

The rest of `send` already clones `message_id` into the entry and passes it to `connection.prompt(message_id, request, cx)`, so no other change is needed.

- [ ] **Step 5: Run the new test and the existing message-id test to verify they pass**

Run:
```bash
cd /home/nick-mullikin/src/zed
cargo test -p acp_thread test_send_populates_message_id_on_prompt_request test_send_assigns_message_id_without_truncate_support
```
Expected: PASS (2 tests). The existing test confirms entries still always get an id; the new test confirms the id is now on the wire.

- [ ] **Step 6: Lint**

Run:
```bash
cd /home/nick-mullikin/src/zed
./script/clippy -p acp_thread
```
Expected: no warnings introduced by the change.

- [ ] **Step 7: Commit**

```bash
cd /home/nick-mullikin/src/zed
git add crates/acp_thread/src/connection.rs crates/acp_thread/src/acp_thread.rs
git commit -m "acp_thread: Send client message_id on prompt requests

Threads the per-message UserMessageId through PromptRequest.message_id so
agents can echo it and, in a follow-up, reference it when truncating a
session. No behavior change for agents that ignore the field.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Gated follow-on roadmap (separate plans, written after Task 1)

These phases cannot be expressed as compiling, file-exact tasks today: the protocol types do not exist at the pinned version, and the adapter lives in a different repo. Each becomes its own plan once Task 1 resolves and the protocol release lands. They are listed here for sequencing and scope only — intentionally deferred, not placeholders.

### Plan A — Protocol: add `session/truncate` (`agent-client-protocol` crate)
- Add an `unstable_session_truncate` feature, `SessionTruncateCapabilities` on `SessionCapabilities`, `TruncateSessionRequest { session_id, message_id }`, `TruncateSessionResponse`, and the `session/truncate` method — mirroring the existing `session/fork` / `session/resume` definitions (`src/v1/agent.rs`).
- Release the crate and bump Zed's `Cargo.toml:508` pin (`agent-client-protocol = "=0.12.1"`) to the new version.
- **Done when:** Zed compiles against the new version and the types are referenceable.

### Plan B — Claude adapter: implement `session/truncate` (`@agentclientprotocol/claude-agent-acp`)
Task 1 confirmed the native mechanism exists in `@anthropic-ai/claude-agent-sdk@0.3.156` — no new Anthropic primitive is required.
- In `prompt()` (`src/acp-agent.ts:732`), record an `acpMessageId → SDK transcript uuid` mapping (and echo `message_id` as `userMessageId`), since the adapter currently ignores the client id.
- Implement the `session/truncate` handler using the SDK's **`resumeSessionAt`** query option (preferred — keeps the session id) or **`forkSession(sessionId, { upToMessageId })`** (new session id). Resume up to the assistant turn preceding the edited user message, then accept the edited prompt.
- Advertise `sessionCapabilities.truncate` in the initialize handshake (sibling of the existing `resume`/`fork`/`close`/`delete`/`list` flags at `src/acp-agent.ts:629-637`).
- Validate behavior across an auto-compaction boundary (refuse, or fall back to the nearest surviving boundary).
- **Done when:** a manual `session/truncate` against a running adapter trims context via `resumeSessionAt`/`forkSession` and the capability is advertised.

### Plan C — Zed truncate-wiring (this repo; small, depends on Plan A)
- Implement `AcpConnection::truncate()` in `crates/agent_servers/src/acp.rs` to return an `AgentSessionTruncate` whose `run()` sends a `session/truncate` request; gate on `self.agent_capabilities.session_capabilities.truncate.is_some()`. Place the capability check alongside `supports_load_session` / `supports_resume_session` / `supports_close_session` (acp.rs:1721-1821); the prompt-sending pattern to follow is the existing `prompt()` at acp.rs:1958.
- This flips `AcpThread::supports_truncate(cx)` (acp_thread.rs:1435) true for capable agents, so the existing inline editor, `editing_message` state, and **Regenerate** button in `crates/agent_ui/src/conversation_view/thread_view.rs` and the existing `AcpThread::rewind()` (acp_thread.rs:2596) engage with no UI rewrite.
- Update the disabled-state tooltip in `thread_view.rs` ("Editing previous messages is not available for … yet") so it only shows for agents that genuinely lack the capability.

### Plan D — UX, edge cases, tests (this repo; depends on Plan C)
- Cancel any in-flight prompt before issuing `session/truncate`; surface RPC failures to the UI rather than dropping them.
- Keep the edit affordance disabled for any message whose `id` is `None` (e.g. older sessions loaded before ids were threaded).
- Leave git-checkpoint behavior unchanged (the separate **Restore Checkpoint** button, driven by `Checkpoint.show`, is independent). Optionally evaluate the SDK's native `rewindFiles(userMessageId)` + `enableFileCheckpointing` as an alternative/complement to the git checkpoint for restoring the working tree at the rewind point.
- Extend `StubAgentConnection` / `FakeAgentConnection` (in `crates/agent_servers/src/acp.rs` and `crates/acp_thread/src/connection.rs`) to advertise and honor truncate; test the rewind path end to end, capability gating (button hidden when unsupported), the `message_id` round-trip, and the no-`message_id` edge case. Per CLAUDE.md, use `cx.background_executor().timer(...)` (not `smol::Timer::after`) in any test driving `run_until_parked()`.

---

## Self-Review

**Spec coverage:** Spec Phase 0 → Task 1. Spec Phase 1 (message-ID threading, the separable foundational half) → Task 2; the new `session/truncate` half of Phase 1 → Plan A. Spec Phase 2 → Plan B. Spec Phase 3 → Plan C. Spec Phases 4–5 → Plan D. All spec phases are accounted for.

**Placeholder scan:** Tasks 1 and 2 contain concrete commands, real code, and exact expected output. Plans A–D are explicitly marked as deferred sub-plans gated on Task 1 / the protocol release, with concrete scope and file references — they are sequencing entries, not in-task placeholders.

**Type consistency:** `UserMessageId` gains `Display`; `message_id.to_string()` relies on it. `PromptRequest::message_id(impl IntoOption<String>)` accepts a `String` via `impl<T> IntoOption<T> for T`. `params.message_id` on the test's captured `PromptRequest` is `Option<String>` (feature `unstable_message_id`, enabled workspace-wide). The `on_user_message` closure signature and `.boxed_local()` match the existing `FakeAgentConnection` API. Names are consistent across tasks.
