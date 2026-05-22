# `harness` — Per-Agent Runtime

> **Depends on:** [`architecture.md`](architecture.md), [`protocol.md`](protocol.md), [`storage.md`](storage.md)
> **Owns:** the per-agent process. Wraps `pi-agent-core`, manages tool registration, drives the agent loop, handles JSONL stdio I/O with CP, instantiates channel adapters, dispatches to subagents.

---

## What the harness is

A Bun process that runs one agent. It is the heir to letta-code, but built on `pi-agent-core` and shaped for our protocol and storage choices.

**One harness == one agent.** Multiple agents = multiple harnesses, each supervised by CP.

**The harness is the per-agent runtime.** `pi-agent-core` owns the LLM loop; the harness owns everything else: multi-conversation state machine and dispatch queue, transcript persistence, compaction lifecycle (self-trigger and force), channel adapter lifecycle (start, reload, teardown), subagent event correlation, sleeptime triggers, and MemFS git. The class implementing this is `Harness` in `packages/harness/src/runtime.ts`.

### Core interfaces (seams)

The harness exposes three explicit seams that keep it testable and let the implementation evolve without callers breaking:

- **`MessageProcessor`** (`packages/harness/src/types.ts`) — what the harness calls to run the agent loop. `PiProcessor` wraps `pi-agent-core`; `EchoProcessor` is the test/probe path. Swap the processor to test the harness shell without an LLM.
- **`ChannelOps`** — what tools call to interact with channels (dispatch action to owning adapter, notify operator, run an adapter-registered tool). Implemented by `Harness`; injected into the processor and through to tools. Tests inject a mock to verify tool behaviour without real adapters.
- **`RuntimeOps`** — what channel adapters call for operator `!commands` (cancel, compact, recompile, reset, ctx, model). Implemented by `Harness`; passed to each adapter at start. Keeps adapter code channel-agnostic — it calls `runtimeOps.cancel()`, not `processor.abort()` directly.

---

## Boot sequence

1. **Process starts** with stdio piped (CP spawns `bun harness.ts <agent_id>`).
2. **Read `init` frame** from stdin (must be first frame).
3. **Validate `init`** — required fields present, paths exist, model resolvable.
4. **Initialize storage:**
   a. Verify `data_dir` exists; create if missing.
   b. If `is_subagent: false`, run MemFS git pull (skip if no remote configured or pull fails — log warning, continue).
   c. List subagent type definitions from `subagents_dirs` for later resolution.
5. **Build initial system prompt** — see § System prompt assembly below.
6. **Construct `pi-agent-core` Agent instance:**
   a. Pass `systemPrompt` (the assembled string).
   b. Register tools via Pi's tool registration (file tools + memory-aware variants + skills + agent + cron + inter-agent + channel send).
   c. Configure model + provider + API keys.
   d. Configure cache breakpoint policy (system + tools + last message; default — see § Prompt caching).
7. **Instantiate channel adapters** for each `init.channels[*]` (skip if `is_subagent: true`).
8. **Replay transcript** for each conversation directory found under `data_dir/conversations/`:
   a. For each conversation, load active context per the algorithm in `storage.md`.
   b. Scan replayed transcript for orphan tool_use blocks (tool_use with no matching tool_result — left by a mid-tool-call restart); synthesize cancellation tool_results for each orphan and insert them after their respective assistant turns.
   c. Set `agent.state.messages` for the `default` conversation; other conversations are loaded lazily on first message.
9. **Subscribe to pi-agent-core events** (`agent.subscribe(listener)`) — see § Event handling.
10. **Emit `ready` frame** on stdout. CP marks the agent ready.
11. **Enter main loop:** read frames from stdin, dispatch.

If any step 1-10 fails fatally, emit `fatal` frame and exit non-zero. CP applies crash backoff per the supervisor pattern.

---

## System prompt assembly

The system prompt is built once at boot and rebuilt only on the events listed below. It is **not** rebuilt every turn (that would invalidate Anthropic's prompt cache — see § Prompt caching).

### Assembly algorithm

```
1. Read base instructions          (built-in, version-locked, never changes mid-session)
2. Append <persistence>            (tells the agent how its memory persists)
3. Read memory/system/*.md         (in alphabetical order by filename)
   For each file:
     <memory path="system/<file>">
     <content>
   </memory>
4. Append memory tree              (Glob-style listing of memory/, excluding system/, max depth 3)
5. Append available subagents      (resolved registry: name + description for each)
6. Append available skills         (name + one-line description from frontmatter)
7. Append <environment>            (cwd, agent name, model, provider, current UTC date — date placed
                                    AFTER the cache breakpoint, see § Prompt caching)
8. Done. systemPrompt = concatenation.
```

### Triggers for rebuild

System prompt is rebuilt and re-applied to `agent.state.systemPrompt` when:

- A memory tool wrote to `memory/system/*.md`. Tools detect their own writes and signal the harness to rebuild before next turn.
- A new subagent type was added (file written under any subagents dir).
- A new skill was added (file written under any skills dir).
- `channel.config_updated` frame received (rare; only if config changes prompt content).

System prompt is **NOT** rebuilt on:

- Every turn (cache-hostile; a no-op rebuild already invalidates cache if any byte changes).
- Memory writes outside `system/*.md` (tree listing is in the prompt but only at depth 3 — file content under e.g. `memory/notes/` doesn't appear).
- Conversation activity (messages append to `state.messages`, not to system prompt).

### `transformContext` for volatile state

For state that *should* be visible to the LLM but *must not* live in the cached system prompt:

```typescript
agent.transformContext = (messages, signal) => {
  // Inject any per-turn-volatile context as a synthetic "system reminder" message
  // appended to the messages array (cache breakpoint floats to last message).
  const reminder = {
    role: "user",
    content: `<system-reminder>
Current UTC time: ${new Date().toISOString()}
Active conversation: ${currentConversationId}
Memory state digest: ${memoryDigestHash}
</system-reminder>`
  };
  return [...messages, reminder];
};
```

This is the pattern for keeping cache hits high while still making volatile context available. **Do not put any time-varying content in `agent.state.systemPrompt`.** See § Prompt caching for why.

---

## Tool registration

All tools registered via Pi's `Agent.tools` interface. Tools are pure functions (with side effects) — they take args, return results, can stream partial results via `onUpdate`.

### v1 tool roster

| Tool | Args | Behavior |
|---|---|---|
| `Read` | `{ path: string, line?: number, limit?: number }` | Read file content. Path can be absolute or relative to agent's `data_dir`. |
| `Write` | `{ path: string, content: string }` | Atomic write. Creates parent dirs. |
| `Edit` | `{ path: string, old_string: string, new_string: string, replace_all?: boolean }` | Letta-code-style replacement. |
| `Grep` | `{ pattern: string, path?: string, output_mode?: "content"\|"files_with_matches"\|"count", -i?, -n?, glob?, type? }` | Wraps `ripgrep` if installed; falls back to JS regex. |
| `Glob` | `{ pattern: string, path?: string }` | Glob match, sorted by mtime. |
| `Bash` | `{ command: string, timeout?: number, run_in_background?: boolean, description: string }` | Shell exec via Bun.spawn. |
| `BashOutput` | `{ bash_id: string, filter?: string }` | Read output of backgrounded shell. |
| `KillShell` | `{ shell_id: string }` | SIGTERM a backgrounded shell. |
| `Skill` | `{ skill: string, args?: string }` | Load a skill's body into the agent's context as a synthetic user message. |
| `Agent` | `{ agent_type: string, task: string, model_override?, tools_override?, inherit_memory? }` | Spawn a subagent. See `subagents.md`. |
| `SendAgentMessage` | `{ to_agent: string, content: string, conversation_id? }` | Send a peer agent a message. See `inter-agent.md`. |
| `ListAgents` | `{}` | List active agents. |
| `cron_add` | `{ name, cron, prompt, timezone? }` | Schedule a cron task. CP-mediated. See `cron.md`. |
| `cron_list` | `{}` | List the agent's cron tasks. |
| `cron_remove` | `{ id: string }` | Remove a task. |
| `cron_pause` / `cron_resume` | `{ id: string }` | Toggle. |
| `EnterPlanMode` / `ExitPlanMode` | `{ plan?: string }` | Toggle plan mode (no edits during plan). |
| `TodoWrite` | `{ todos: Todo[] }` | Update agent's task list. |
| `AskUserQuestion` | `{ message, options }` | Pause for user input. v1: only works in PWA / interactive contexts; in channel mode, posts the question and waits for next inbound. |

### Lifted vs new

- `Read`, `Write`, `Edit`, `Grep`, `Glob`, `Bash`, `BashOutput`, `KillShell`, `Skill`, `EnterPlanMode`, `ExitPlanMode`, `TodoWrite`, `AskUserQuestion` — **lifted** from `letta-code/src/tools/impl/`. Adapter changes: tool registration interface, no Letta-server calls.
- `Agent`, `SendAgentMessage`, `ListAgents`, `cron_*` — **new**, designed to use our CP-mediated tool pattern (emit `tool.request` or domain-specific frame, await response).

### CP-mediated tool pattern

Tools that need CP state (cron, inter-agent, agent-spawn, memory.write_proxy):

1. Tool body emits a frame on stdout — either domain-specific (`agent.spawn`, `inter_agent.send`) or generic (`tool.request` with `tool: "cron.add"`).
2. Tool body awaits a response frame on stdin matching the `request_id`.
3. CP processes the request (e.g., adds cron task to DB, persists, returns ack).
4. CP writes `tool.response` to harness stdin.
5. Harness's stdin reader resolves the awaiting promise.
6. Tool returns the result to pi-agent-core's loop.

This is synchronous from pi-agent-core's perspective (the agent waits for the tool to return), asynchronous from the harness's stdio perspective.

### Visual relabeling

When a tool's path argument resolves under `$MEMORY_DIR` (i.e., `~/.argos/agents/<id>/memory/`), the renderer (CLI or future PWA) substitutes a friendlier label:

| Tool + memory path | Display |
|---|---|
| `Read memory/system/persona.md` | "Reading memory: persona" |
| `Edit memory/system/human.md` | "Updating memory: human" |
| `Write memory/notes/idea.md` | "Writing note: idea" |
| `Grep memory/...` | "Searching memory" |
| `Glob memory/...` | "Listing memory" |

The tool itself emits a normal `tool_execution_*` event with the raw path. **Renderer-side logic** detects the path prefix and re-labels for display. This keeps tool implementation unchanged and the visual UX clean.

---

## Conversation dispatch

The harness can run the agent loop in any of N conversations. Each inbound message frame includes a `conversation_id`. The `handleUserMessage` method follows a strict three-phase flow:

**Phase 1 — compaction gate.** Before doing anything, await any in-flight auto-compaction for this conversation. The MemGPT atomic-eviction property: the next turn must see the post-summary message array. If a reshape is also pending (set by the previous compaction completing while a turn was running), apply it now.

**Phase 2 — conversation swap.** If the incoming `conversation_id` differs from the one currently loaded in the processor:
1. Snapshot the outgoing conversation's running messages back to the cache.
2. Load the incoming conversation from cache → disk replay → empty (in that order).
3. Call `processor.setInitialMessages(loaded)`.

**Phase 3 — run.** Persist the user transcript entry first (wrapped with `<inbound …>` channel metadata if source is not cli). Then call `processor.handleUserMessage(...)` and let it run to `agent_end`.

Transcript entries (assistant + tool_call) are persisted in `handleProcessorEvent`, which intercepts every processor event at canonical boundaries before forwarding to CP and channel adapters.

### Turn queue and mid-stream steering

All inbound sources (CLI, Matrix, cron, inter-agent) enqueue via `enqueueTurn()`. A single `drainQueue()` loop serialises turns across all conversations so the processor state is never touched concurrently.

Exception: if a message arrives for the **same** conversation currently in flight and the processor supports steering (`processor.steer()`), the harness skips the queue and injects the message mid-turn. The user entry is persisted first so a crash during steering doesn't lose it. If steering races with turn completion, the message falls back to normal queue push.

### Channel-originated messages

Channel adapters (Matrix, etc.) live in the harness process. They directly call `enqueueTurn()` — no stdio roundtrip. Inbound flow:

```
Matrix sync → matrix-bot-sdk handler → adapter normalize →
  enqueueTurn → drainQueue → handleUserMessage → processor
```

Outbound flow (streaming):

```
handleProcessorEvent → emitAgentEvent → adapter.handleAgentEvent → matrix-bot-sdk send/edit
```

For per-token edit-in-place (Matrix `m.replace`, future Telegram `editMessageText`), the adapter consumes the `message_update` events directly and updates the platform message progressively.

### Channel-originated messages

Channel adapters (Matrix, etc.) live in the harness process. They directly call into the conversation dispatcher — no stdio roundtrip. Inbound flow:

```
Matrix sync → matrix-bot-sdk handler → adapter normalize →
  conversation dispatcher → enqueue message → run loop
```

Outbound flow (streaming):

```
agent_event delta → adapter renders → matrix-bot-sdk send/edit
```

For per-token edit-in-place (Matrix `m.replace`, future Telegram `editMessageText`), the adapter consumes the `message_update` events directly and updates the platform message progressively.

---

## Compaction (harness-internal)

Triggered by:

- **Self-trigger:** harness tracks token usage from pi-agent-core's response metadata. When `usage > init.compaction_threshold * init.context_window`, harness fires compaction.
- **Force:** CP sends `{type:"compact", conversation_id}` frame.

Algorithm:

1. Emit `compaction.start` frame on stdout.
2. Build summarizer input: system prompt of the summarizer (a constant), plus the conversation messages to summarize (everything but the most recent ~20-30% retained as the tail).
3. Make pi-ai LLM call (using the agent's own model + provider, separate from the agent's loop).
4. Compose `compaction` transcript entry: `{kind:"compaction", evict_range:[first_id, last_id], summary, tokens_before, tokens_after}`.
5. Append to today's transcript.
6. Update active context cache: replace evicted messages with the summary as a synthetic assistant message.
7. Update `agent.state.messages` so the next turn sees the compacted context.
8. Emit `compaction.complete` frame.
9. If "on-compaction" sleeptime trigger configured (per agent config), emit `agent.spawn` frame for sleeptime subagent with input being the evicted content.
10. Resume — accept next turn.

See `compaction.md` for the summarizer prompt and edge cases.

---

## Sleeptime triggers

Two trigger conditions; both checked by the harness:

- **Every N turns:** harness counts `agent_end` events for the conversation. When `count % N == 0`, fire.
- **On compaction:** fire after compaction completes (step 9 above).

Both fire by emitting a `agent.spawn` frame with `agent_type: "sleeptime"` and `task` containing the relevant context (recent assistant messages for "every N", evicted content for "on-compaction"). CP spawns the sleeptime child harness; child runs, mutates memory blocks via memory-write-proxy tool, terminates. See `subagents.md`.

---

## Event handling

The processor calls back into `handleProcessorEvent(event, conversationId)` for each event it emits. This is the single path for both forwarding to CP and persisting transcript entries — they must not be separated, because persistence must happen at the same canonical boundaries the forwarded events describe.

```typescript
handleProcessorEvent(event, conversationId) {
  // 1. Forward to CP + channel adapters
  emitAgentEvent(event);  // tags with conversation_id, suppresses channel fanout for cron-source turns

  // 2. Persist transcript entries at canonical boundaries
  switch (event.type) {
    case "message_end":
      // Filter to role === "assistant" only (pi-agent-core emits message_end for user echoes too)
      appendTranscriptEntry(conversationId, assistantEntry);
      break;
    case "tool_execution_end":
      appendTranscriptEntry(conversationId, toolCallEntry);
      break;
    case "agent_end":
      // a. Auto-compaction threshold check
      evaluateAutoCompaction(conversationId, event);
      // b. Subagent mode: emit agent.result frame + schedule exit
      if (init.is_subagent) emitAgentResult(...);
      // c. MemFS: gitCommitIfDirty + gitPush (skipped for subagents and echo mode)
      if (!is_subagent && mode !== "echo") gitCommitIfDirty(...); gitPush(...);
      // d. Sleeptime every-N trigger
      sleeptime.onAgentEnd(emit, recentAssistantText, conversationId);
      break;
  }
}
```

Note: `emitAgentEvent` suppresses channel-adapter fanout when `currentTurnSource === "cron"`. Cron-triggered turns run silently; the agent must call `NotifyUser` to surface anything to the operator.

---

## Stdin frame router

Single function reading line-buffered stdin:

```typescript
for await (const line of readlines(process.stdin)) {
  const frame = JSON.parse(line);
  switch (frame.type) {
    case "init":                    /* should never see this post-boot; log and ignore */
    case "shutdown":                gracefulShutdown(frame.grace_ms); break;
    case "user_message":            enqueueUserMessage(frame); break;
    case "inter_agent.receive":     enqueueInterAgentMessage(frame); break;
    case "cron_trigger":            enqueueCronTrigger(frame); break;
    case "abort":                   abortRun(frame.conversation_id); break;
    case "compact":                 triggerCompaction(frame.conversation_id); break;
    case "memory.write":            handleMemoryWrite(frame); break;
    case "agent.event":             routeChildEventToParent(frame); break;
    case "agent.result":            resolveAgentSpawnPromise(frame); break;
    case "tool.response":           resolveToolRequestPromise(frame); break;
    case "channel.config_updated":  reloadChannel(frame.channel, frame.config); break;
    default:                        log warning, drop;
  }
}
```

---

## Prompt caching

Anthropic prompt caching is wired by `pi-ai` at three breakpoints (`packages/ai/src/providers/anthropic.ts:874-916, 1115-1116`):

1. End of system prompt block.
2. End of last tool definition.
3. End of last user/assistant message.

For high cache hit rates:

- **Keep `agent.state.systemPrompt` byte-stable** for the agent's lifetime (rebuild only on the events listed in § System prompt assembly triggers).
- **Place volatile content via `transformContext`** — appended to messages, where the per-message cache breakpoint floats to absorb each new turn cleanly.
- **Avoid `Current date: YYYY-MM-DD` in the system prompt** (Pi's default puts it there, breaks cache at midnight). Move it into `<environment>` as part of `transformContext` injection instead, so it lands in the floating message cache breakpoint.

Verify with token usage logs after PR landing: `cached_input_tokens / prompt_tokens` should approach 1.0 for steady-state turns within the same conversation.

---

## Subagent mode

When `init.is_subagent: true`:

- Skip channel adapter init (subagents don't directly serve channels).
- Skip MemFS auto-pull and auto-push (subagent's memory operations don't pollute parent's git history; if subagent modifies memory, commit happens via parent's `agent_end`).
- Skip cron scheduler integration.
- Permission mode forced to `bypassPermissions` (subagent inherits parent's authority for the duration of its task).
- Send a single `agent_event` `{type: "agent_end"}` triggers final result emit and process exit.
- On exit, last action is to emit `agent.result` frame on stdout (with full result payload), then exit 0.

---

## Failure modes

- **`init` malformed:** emit `fatal`, exit 1.
- **MemFS pull fails:** log warning to stderr, continue with local checkout. Don't exit.
- **Channel adapter init fails:** log error, mark adapter unavailable, continue. Don't exit (other channels and CLI/inter-agent still work).
- **pi-agent-core throws unrecoverably:** emit `fatal`, exit 1. Supervisor respawns.
- **Stdin EOF:** CP closed the pipe. Treat as graceful shutdown; flush, exit 0.
- **Stdout backpressure:** OS handles; agent loop pauses naturally. Recover when CP catches up.

---

## Open considerations

- **Concurrent conversations and tool state.** If two conversations run sequentially in the same process, can a Bash backgrounded shell from conversation A interfere with conversation B? The bash_id namespace is per-tool-call; should be fine. Verify in implementation.
- **Memory tool + MemFS auto-commit race.** Agent calls `Edit memory/system/persona.md`, then runs Bash `git status`. The commit hasn't happened yet (only on `agent_end`). The agent sees uncommitted changes. Decision: this is fine; the agent shouldn't be running git commands itself anyway. If it does, it gets accurate live state. We just skip the system-prompt git boilerplate from letta-code.
- **AskUserQuestion in channel mode.** When the agent invokes this in a channel-driven conversation, what happens? Resolution: post the question via the channel adapter (formatted as a question with options if structured), wait for next inbound message in the same conversation, treat the response as the answer. May need timeout handling.

---

## PR sequence (preview)

1. `harness` package skeleton (Bun entrypoint, stdio JSONL reader/writer, init handshake).
2. Tool registration and Pi `Agent` instantiation.
3. File tools (lift from letta-code, adapter changes).
4. System prompt assembly (cache-friendly).
5. `transformContext` wiring for volatile state.
6. Conversation dispatcher (queue, replay, run, persist).
7. Memory tools (mostly file-tool reuse + write-back signal for prompt rebuild).
8. Compaction (self-trigger + force-trigger + summarizer prompt).
9. Sleeptime triggers (every-N + on-compaction).
10. Subagent mode (init.is_subagent handling, result emit).
11. Channel adapter loader (per-channel modules; v1: Matrix only).
