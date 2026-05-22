# `compaction` — Summarizer, Eviction, Trigger

> **Depends on:** [`architecture.md`](architecture.md), [`protocol.md`](protocol.md), [`storage.md`](storage.md), [`harness.md`](harness.md), [`subagents.md`](subagents.md)
> **Owns:** when compaction fires, how summaries are produced, how the transcript records compaction, how active context is reshaped.

---

## Layered context model

Argos uses **two stacked mechanisms** to keep prompt size bounded:

1. **Sliding window** (every turn): a `transformContext` hook in PiProcessor walks `agent.state.messages` newest→oldest, dropping the prefix older than `~70%` of the context window. Cuts at user-message boundaries so tool_call/tool_result pairs stay together. Runs cheaply per turn — no LLM call, no event. `<conversation_summary>` markers (synthetic user messages) are always kept regardless of position. Source: `packages/harness/src/processors/sliding-window.ts`. Default ratio overridable via `ARGOS_SLIDING_WINDOW_RATIO`.

2. **Compaction** (occasional): when the *full* transcript on disk grows large enough that the sliding window starts dropping non-trivial slices, compaction summarizes the dropped prefix and writes a `compaction` entry to the transcript. On replay, that entry becomes a `<conversation_summary>` synthetic message — which the sliding window then preserves indefinitely. So compaction's job is to convert verbatim-but-truncated history into compressed-but-permanent memory.

In normal companion-style usage (frequent short turns), most days the agent never hits compaction — sliding window alone is enough. Compaction fires when long single-task agents accumulate enough verbatim history that summarization buys real space.

---

## When compaction fires

Two triggers, both evaluated by the harness in its `agent_event` handler at `agent_end`:

**1. Self-trigger on token threshold.**

- Harness tracks token usage from each LLM response (pi-agent-core surfaces this in usage events).
- After each `agent_end`, computes `usage = (last response's prompt_tokens) / context_window`.
- If `usage >= compaction_threshold` (default 0.85, configurable per agent in `init.compaction_threshold`), fires compaction.

**2. Force-trigger from CP.**

- CP sends `{type: "compact", conversation_id}` frame on stdin.
- Triggered by:
  - CLI: `argos compact <agent> [--conversation default]` (ops/debug command).
  - Future: PWA "compact now" button.
  - Sleeptime tooling that wants to rotate context proactively.

Both triggers run the same compaction algorithm. Only the entry point differs.

---

## Compaction algorithm

For a given `conversation_id`:

1. **Emit** `compaction.start` frame on stdout: `{conversation_id, tokens_before}`.

2. **Load active context** (already in memory; harness keeps it cached).

3. **Decide eviction range:**
   - Keep system message (index 0) — pass-through; not part of active messages anyway.
   - Keep the most recent `retain_tail_fraction` of messages (default 70%, matching Letta's sliding_window default) — these stay verbatim.
   - Keep messages back to the nearest "user message" boundary at or after the tail cutoff (don't slice mid-tool-call).
   - Everything between (oldest user message after system, up to retain boundary) is the eviction range.

4. **Build summarizer input:**
   - Messages in eviction range, formatted as a transcript (compact JSON-ish or plain text).
   - Optional context: any prior compaction summaries that are still in the active context.

5. **Call summarizer LLM:**
   - Same model + provider as the agent itself by default (configurable via `summarizer_model` setting per agent if needed).
   - System prompt: see § Summarizer prompt below.
   - User message: the formatted eviction-range transcript.
   - One synchronous call. Block here ~5-15s for typical compactions.

6. **Receive summary text** (a structured-but-prose digest). Hard-capped at 50,000 chars (matching Letta's `clip_chars` default) — if the summarizer returns more, the excess is truncated with a marker. Prevents a very large eviction range from producing a summary that itself consumes most of the context budget.

7. **Append `compaction` entry to today's transcript:**
   ```jsonc
   { "id": "msg_01H...", "ts": "...", "kind": "compaction",
     "evict_range": ["msg_first_id", "msg_last_id"],
     "summary": "<summarizer output>",
     "tokens_before": <int>, "tokens_after": <int>,
     "model_used": "claude-sonnet-4-6" }
   ```

8. **Reshape active context cache:**
   - Replace evicted messages with a single synthetic `assistant` message containing the summary, wrapped in `<conversation_summary>` tags so the agent recognizes it on next turn.
   - Active context is now `[system, summary_as_user_message, retained_tail]`.

9. **Update `agent.state.messages`** with the new shape. Next LLM call uses this.

10. **Emit `compaction.complete` frame** with `{tokens_before, tokens_after, evicted_count, summary_id}`.

11. **Sleeptime "on-compaction" trigger:** if enabled, emit `agent.spawn` for sleeptime subagent, passing the evicted content + summary as task input. Wait for sleeptime to complete before accepting next user message (small delay; acceptable).

12. **Resume.** Next inbound message is processed against the compacted context.

---

## Summarizer prompt

A constant in the harness package, version-locked. Sketch:

```
You are a conversation summarizer. Your job is to produce a faithful, compact digest
of an excerpt of a conversation between a personal companion AI and a user (and
sometimes other parties).

The digest will replace the full excerpt in the AI's working memory. Include
everything the AI would need to:

- recall what the user has said about themselves, their preferences, or their
  context;
- recall decisions, plans, or commitments made;
- recall the resolution of any open threads from the excerpt;
- recall references to external resources (URLs, file paths, names) that may matter
  later.

Exclude:

- chitchat that didn't change anything;
- intermediate steps in a problem-solving sequence whose final answer is captured;
- exact tool-call arguments (capture intent, not raw args);
- repetitive content.

Format: a structured prose digest with subsections as needed. No bullet salad —
write it as a coherent narrative the AI can scan in one read.

Length target: ~10-20% of the excerpt's length. Quality over brevity; if the
excerpt is information-dense, the digest can be longer.

Return only the digest. Do not preamble or explain.

EXCERPT TO SUMMARIZE:

<excerpt>
```

This prompt is **lifted in spirit** from `letta/services/summarizer/summarizer.py`'s `simple_summary` and `letta/prompts/summarizer_prompt.py`. We don't directly copy the wording (Letta's is also tuned for their model behavior); we evolve our own based on what the agent actually needs to remember.

---

## Summary placement in active context

After compaction:

```
agent.state.messages = [
  /* system message — managed separately by pi-agent-core via agent.state.systemPrompt */
  { role: "user", content: "<conversation_summary>\n<summary text>\n</conversation_summary>" },
  /* retained tail messages (user, assistant, tool calls) verbatim */
  { role: "user", content: "..." },
  { role: "assistant", content: "..." },
  ...
]
```

Wrapping in `<conversation_summary>` tags makes it identifiable to the agent (its base prompt teaches it what these tags mean). The agent treats the summary as the prefix of the conversation it can rely on, with the retained tail being the recent context it can engage with directly.

**Why a `user` role for the summary:** `assistant` role would imply the agent itself produced it (confusing); `system` role would conflict with pi-agent-core's separate system prompt. `user` with explicit tag wrapping is the clearest signal.

---

## Recovery & robustness

**Summarizer LLM failure:**

- Network error / timeout / rate limit:
  - Retry up to 2 times with exponential backoff (1s, 5s).
  - On final failure: emit `compaction.complete` with `ok: false`; **do not** evict messages; log error; continue.
  - Active context grows; threshold trigger will retry on next agent_end if usage still elevated.
  - Mitigation if persistent: agent's responses may eventually 4xx (provider rejects oversized prompt). Abort run, emit error message to user, prompt to retry later.

**Partial summary:**

- If summarizer returns < 100 chars, treat as failure (model gave up). Log + skip eviction.

**Corrupted active-context cache after compaction:**

- The transcript file always has both pre- and post-compaction content. Active context can be rebuilt from the transcript via the boot replay algorithm. If runtime cache is suspect, force rebuild via `compact` frame trigger or harness restart.

**Agent in middle of a turn when compaction fires:**

- It can't be — compaction triggers on `agent_end`, after the turn completes. Force-trigger from CP also waits for any in-flight turn to end before proceeding.

---

## Cache implications

Compaction invalidates the message-array prefix entirely (every cached message after the system block is replaced). Next LLM call after compaction is a full cache miss for the message portion (system + tools cache breakpoints still hit).

This is the cost. Compaction trades a one-time cache write for sustained smaller-context turns going forward. Net cost-positive over many subsequent turns.

---

## Compaction's relationship with sleeptime

Sleeptime "on-compaction" trigger fires AFTER step 10 (compaction.complete emitted) and BEFORE step 12 (next message accepted). The compaction has already happened; sleeptime sees the evicted content as task input and can write to `system/*` memory blocks based on it.

Sleeptime's writes invalidate the *next* turn's system-prompt cache (memory blocks changed). Net: compaction + sleeptime together pay one cache write to the message array AND one to the system block. Worth it for the durable knowledge captured.

If sleeptime is disabled (per agent setting), step 11 is skipped; only compaction happens.

---

## Per-conversation compaction

Compaction is **scoped to a single conversation**. The harness tracks token usage and active context separately per conversation. Compacting `default` doesn't affect the active context of `matrix-!Wabc:matrix.org`.

This is correct because each conversation has its own context budget — a long-running Matrix room doesn't blow out the PWA chat's working memory.

If multiple conversations need compacting concurrently, harness compacts them serially (one summarizer LLM call at a time per harness; concurrent summarizers across agents are fine because they're in different processes).

---

## Configuration knobs

Per-agent (in `agents` table or `init` frame):

- `compaction_threshold` (float, 0.0-1.0, default 0.85)
- `retain_tail_fraction` (float, default 0.70 — retain 70%, evict 30%, matching Letta's sliding_window default)
- `summarizer_model` (string, default = parent's model)
- `sleeptime_on_compaction` (bool, default true)
- `sleeptime_every_n_turns` (int or null, default null)

Globally (in `settings`):

- (none for v1; per-agent suffices)

---

## Open considerations

- **Tail-boundary heuristic.** v1 uses "retain back to nearest user-message boundary at or after cutoff." Edge case: if the conversation has no user messages in the tail (e.g., agent-only multi-step run after a single user prompt), the tail boundary collapses. Solution: also accept assistant/tool-result boundaries; never split mid-tool-call. To be tightened in implementation.
- **Quality measurement.** How do we know the summarizer is producing good summaries? v1: log + manual review. v2: optional A/B with longer history retention to detect summary-induced regressions in agent behavior.
- **Summarizer in a subagent vs inline.** Currently inline (one harness-internal LLM call). Alternative: spawn a `summarizer` subagent per Q11. Trade-off: subagent has separate context window, more overhead per compaction (~1-2s); inline blocks the harness for ~5-15s. Inline is simpler and matches Letta's design. Stick with inline; revisit if blocking becomes a UX issue.
- **Multiple compactions before evicted summary itself gets evicted.** Over a long-running conversation, summaries-of-summaries-of-summaries could lose fidelity. v1 doesn't address this. v2 idea: keep the *original* evicted content in cold storage (transcript file already does this), and on N-th compaction, re-summarize from the original transcript rather than from previous summary.
- **Multimodal content in evicted messages.** Image / audio attachments in the eviction range — what happens? v1: drop from summary (we describe their *existence* in prose only, not the content). Re-fetching original media would require keeping URLs. Acceptable for v1.

---

## PR sequence (preview)

1. Token usage tracking in harness (subscribe to pi-agent-core usage events, maintain per-conversation counter).
2. Compaction trigger evaluator (`agent_end` handler check).
3. Eviction range computation (tail fraction, user-message boundary).
4. Summarizer prompt + LLM call (uses pi-ai directly with the agent's provider).
5. Compaction transcript entry write.
6. Active context cache reshape.
7. `compact` frame handler (force trigger).
8. Sleeptime hookup (emit agent.spawn after compaction.complete).
9. Retry / failure handling.
10. Tests: synthetic conversations, summarizer mock, replay verification.
