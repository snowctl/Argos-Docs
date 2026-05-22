# `subagents` — `Agent` Tool, Spawn Lifecycle, Sleeptime

> **Depends on:** [`architecture.md`](architecture.md), [`protocol.md`](protocol.md), [`storage.md`](storage.md), [`harness.md`](harness.md), [`control-plane.md`](control-plane.md)
> **Owns:** the `Agent` tool, agent type registry, spawn flow, parent↔child event routing, sleeptime triggers and behavior.

---

## Concept

A subagent is a short-lived child agent dispatched by a parent to complete a focused task. It runs in its own harness process with its own context, returns one result, and terminates. Spawned by CP on parent's behalf (Q11 Option A); never spawned by parent directly.

A subagent is, structurally, **just an agent**. Same harness binary, same agent loop, same file tools. The only differences are configuration:

- `is_subagent: true` in `init` (skips channel adapter init, MemFS auto-pull/push, cron integration).
- A focused system prompt (loaded from a `.md` type definition file).
- A short context (one user turn = the parent's task).
- An ephemeral data dir (or shared-with-parent for memory access).
- Parent gets the final result via an `agent.result` frame.

---

## Subagent type registry

Type definitions are markdown files with frontmatter. Three resolution layers (most-specific first):

1. **Per-agent:** `~/.argos/agents/<parent_agent_id>/subagents/<name>.md`
2. **Global:** `~/.argos/subagents/<name>.md`
3. **Built-in:** bundled with harness package at `packages/harness/builtin-subagents/<name>.md`

Resolution: walk in order, return first match. Per-agent wins. Built-in is the fallback.

### Type definition format

```yaml
---
name: recall                       # required; unique within scope
description: |                     # required; shown to parent agent in Agent tool docs
  Searches the agent's conversation transcripts and memory for relevant past
  context. Best for "did I talk about X?" or "what was that thing we decided?"
  Returns a focused summary; does not modify state.
default_model: claude-sonnet-4-6   # optional; defaults to parent's model
default_tools:                     # optional; defaults to parent's full toolset
  - Read
  - Grep
  - Glob
default_inherit_memory: read-only  # "none" | "read-only" | "read-write"; default "none"
---

You are a recall subagent dispatched by a parent agent to find relevant historical context.

Your task is in your first user message.

Procedure:
1. Identify keywords and time bounds in the task.
2. Use Glob/Grep on $CONVERSATIONS_DIR (and optionally $MEMORY_DIR if relevant) to locate hits.
3. Read the matching transcript day-files in full to assemble context.
4. Return a focused, scannable summary: what was discussed, when, key conclusions.
   Quote excerpts where useful. Cite file paths and dates.

Do not modify any state. Your work is read-only.
```

The body of the file (everything after the frontmatter) is the subagent's system prompt, sent verbatim.

### `default_inherit_memory` semantics

| Value | Subagent's MemFS access |
|---|---|
| `none` | No MemFS at all. Subagent's data dir contains an empty `memory/` (no git remote). Agent can write there but it's discarded on subagent exit. |
| `read-only` | Subagent's `memory/` is a read-only bind mount of parent's `memory/`. Agent can Read/Grep/Glob but not Write/Edit. Use case: recall, explore. |
| `read-write` | Subagent operates directly on parent's `memory/`. Writes persist. Use case: reflection, sleeptime, fork (with caveats). |

Implementation: for `read-only`, mount via `bindfs`-equivalent or symlink + harness-level write rejection (simpler). For `read-write`, just point the subagent's `init.memory_root` at the parent's path.

---

## Built-in subagent types (v1)

### `general-purpose`

```yaml
---
name: general-purpose
description: |
  General-purpose subagent for tasks that don't fit a specialized type. Provide a
  detailed prompt describing exactly what you want done. The subagent will work
  contained — it doesn't modify your state unless you explicitly tell it to.
default_tools: [Read, Write, Edit, Grep, Glob, Bash]
default_inherit_memory: none
---

You are a focused subagent dispatched by a parent agent to complete a specific task.
The task description is in your first user message.

Work the task to completion. When done, respond with a clear summary of what you
did and what you found. If you cannot complete the task, explain why.

Your work is contained: do not modify state outside what your task requires.
```

### `explore`

```yaml
---
name: explore
description: |
  Read-only exploration of files or topics. Use when you need to understand a
  codebase, document, or set of files without modifying anything. Returns a
  synthesized overview, not raw output.
default_tools: [Read, Grep, Glob]
default_inherit_memory: read-only
---

You are an exploration subagent. Your task is to investigate, not to act.

Procedure:
1. Identify the scope from the task.
2. Use Read/Grep/Glob to gather information.
3. Synthesize into a focused report: what's there, how it's organized, key findings.

Don't list raw greps. Synthesize. Quote sparingly.
```

### `recall`

(See example above.)

### `reflection`

```yaml
---
name: reflection
description: |
  Reflects on recent activity and updates memory blocks accordingly. Reads recent
  conversation transcripts, identifies durable insights, and writes them to
  system/* files. Used by the parent agent for periodic self-improvement.
default_tools: [Read, Edit, Write, Grep, Glob]
default_inherit_memory: read-write
---

You are a reflection subagent. Your role is to consolidate the parent agent's
recent experience into durable memory.

Procedure:
1. Read the recent conversation transcripts described in your task.
2. Identify content that should become durable memory: facts about the user,
   shifts in your sense of self, new relationships, recurring themes.
3. Update the relevant system/*.md files in $MEMORY_DIR with focused, additive
   edits. Don't rewrite whole files unless the existing content is wrong.
4. Return a summary of the changes you made and why.

Be conservative: prefer additive edits to rewrites; prefer focused changes to
broad ones; don't delete content you don't fully understand the purpose of.
```

### `fork`

```yaml
---
name: fork
description: |
  Spawn a clone of yourself to explore an alternative path or hypothesis without
  committing your main thread to it. Receives the same context but its work is
  isolated; you receive a summary of what it found.
default_tools: [Read, Write, Edit, Grep, Glob, Bash]
default_inherit_memory: none
---

You are a forked instance of the parent agent, dispatched to explore an alternative.

Your task is in your first user message. Treat it as a hypothesis or exploration
that the main agent doesn't want to commit to yet. Do the exploration thoroughly
and return findings.

Your work is isolated — your main agent will receive your summary but no other
state changes from your run.
```

### `sleeptime`

```yaml
---
name: sleeptime
description: |
  Background memory consolidation. Triggered automatically every N turns or after
  compaction; not typically invoked manually. Reads recent activity and updates
  memory blocks to capture durable insights before context is compacted away.
default_tools: [Read, Edit, Write, Grep, Glob]
default_inherit_memory: read-write
---

You are a sleeptime subagent. Your role is to consolidate the parent agent's
recent activity into durable memory before that activity is lost from active context.

Your task input describes what triggered you (turn count, compaction event) and
includes the relevant content (recent assistant turns, or the compaction's evicted
range).

Procedure:
1. Read the input. Identify durable insights: stable facts about the user, shifts
   in self-understanding, new relationships, learned preferences.
2. Update the relevant system/*.md files in $MEMORY_DIR with focused, additive edits.
3. If the trigger was compaction and the evicted content contains tasks, decisions,
   or commitments not yet captured anywhere, write a focused note to memory/notes/
   so it's preserved.
4. Return a one-paragraph summary of what you consolidated.

Be conservative — your edits will be visible to your parent on its next turn.
```

---

## The `Agent` tool

Registered in the harness's tool registry. Args:

```typescript
{
  agent_type: string;          // required; resolved against subagent dirs
  task: string;                 // required; sent as user message to the subagent
  model_override?: string;      // optional; overrides default_model
  tools_override?: string[];    // optional; overrides default_tools
  inherit_memory?: "none" | "read-only" | "read-write";  // overrides default
}
```

### Tool body

```typescript
async function agentTool(args, ctx) {
  const subagent_id = generateUUID();
  const request_id = generateUUID();

  // Emit spawn frame on stdout
  emitFrame({
    type: "agent.spawn",
    request_id, agent_id: subagent_id,
    agent_type: args.agent_type,
    task: args.task,
    model_override: args.model_override ?? null,
    tools_override: args.tools_override ?? null,
    inherit_memory: args.inherit_memory ?? null,  // null = use type's default
  });

  // Stream child events to user-facing tool output via onUpdate
  const subscription = onChildEvent(subagent_id, (event) => {
    ctx.onUpdate(formatChildEvent(event));
  });

  // Wait for agent.result with matching request_id
  const result = await awaitResult(request_id);
  subscription.close();

  if (!result.ok) throw new Error(result.error);
  return {
    summary: result.result,
    total_tokens: result.total_tokens,
    duration_ms: result.duration_ms,
  };
}
```

### Tool description (shown to parent agent)

Generated dynamically from the registry:

```
Agent — Spawn a subagent to handle a focused task.

Available agent_types:
  general-purpose: General-purpose subagent for tasks that don't fit a specialized type...
  explore: Read-only exploration of files or topics...
  recall: Searches conversation transcripts and memory for past context...
  reflection: Reflects on recent activity and updates memory blocks...
  fork: Spawn a clone of yourself to explore an alternative...

Args:
  agent_type — string, required
  task — string, required
  model_override — string, optional
  tools_override — string[], optional
  inherit_memory — "none" | "read-only" | "read-write", optional
```

The dynamic listing means agents (and users authoring custom subagent types) can add their own without touching the tool description.

---

## Spawn flow (CP-side)

When `agent.spawn` arrives from a parent harness:

1. **Resolve type:** walk subagents dirs, find `<agent_type>.md`. If not found → respond to parent with `agent.result { ok: false, error: "unknown agent_type: ..." }`.
2. **Parse frontmatter + body.** Apply overrides from spawn frame.
3. **Build subagent data dir:**
   - `~/.argos/agents/subagent-<id>/` (transient).
   - Create `memory/`, `conversations/`, `subagents/`, `channels/` dirs.
   - For `read-only` / `read-write` inherit_memory: mount or symlink parent's `memory/`.
   - For `none`: empty `memory/`, no git remote.
4. **Compose `init` frame** with:
   - `is_subagent: true`
   - `parent_agent_id`
   - `system_prompt_override = <body of type .md>`
   - Resolved model + tools + memory config
   - `internal_token` (transient)
5. **Spawn harness** via `Bun.spawn(harnessBin, [...], { stdio: ['pipe', 'pipe', 'pipe'] })`. Track in CP's HarnessHandle map with `parent_id`.
6. **Write init frame** to subagent's stdin.
7. **Wait for `ready`** frame on subagent's stdout. Timeout 30s.
8. **Write `user_message` frame** with `content: task` to subagent's stdin.
9. **Stream events:** subagent's `agent_event` frames are forwarded to parent's stdin as `agent.event` (with `agent_id: subagent_id`).
10. **Subagent runs to completion.** It naturally exits after `agent_end` because there are no more pending messages and the harness in subagent mode exits on `agent_end`.
11. **Collect result:** the harness's last action before exit is to emit `agent.result { request_id, agent_id, ok, result, total_tokens, duration_ms }` directly. CP forwards this verbatim to parent's stdin.
12. **Cleanup:** remove from HarnessHandle map. If `inherit_memory == "none"`, delete data dir. Otherwise leave (tracked under parent's tree).

If the subagent harness crashes:

- Apply same fast-crash detection as top-level agents.
- For subagents, **do not respawn** — return `agent.result { ok: false, error: "subagent crashed: <reason>" }` to parent.
- Parent's tool handles the error (most subagents are non-essential; parent decides whether to retry, escalate, or proceed).

---

## Subagent transcripts

Subagent activity is recorded in its own transcript directory under the parent's tree:

```
~/.argos/agents/<parent_id>/conversations/<conv_id>/subagents/<subagent_id>/
├── 2026-05-05.jsonl
└── usage.jsonl
```

Where `<conv_id>` is the parent's conversation that triggered the spawn. Subagent's transcript captures its own user message (the task), its assistant turns, its tool calls, its result.

This means: PWA (post-v1) can show subagent runs nested under parent runs. The agent itself can recall what its subagents did via Grep over `conversations/*/subagents/*/`.

For `inherit_memory != "none"` subagents, this also serves as an audit trail of memory mutations the subagent made.

---

## Sleeptime — triggers + behavior

### Trigger evaluation

Both checked by the parent harness in its `agent_end` event handler:

**Every-N-turns trigger:**

- Configured per agent: `sleeptime_every_n_turns` (default: disabled in v1; per-agent setting via CP). When set to e.g. 10, fires every 10th `agent_end` for any conversation.
- Counter is per-conversation: `turnCounter[conversation_id]++` on each `agent_end`.
- When `turnCounter[conv] % N == 0`, fire.

**On-compaction trigger:**

- Configured per agent: `sleeptime_on_compaction` (default: enabled in v1).
- Fires immediately after a `compaction.complete` frame is emitted, before next user message is processed.
- Trigger payload includes the evicted content (the actual messages, not just IDs) so sleeptime sees what's about to be lost from active context.

### Sleeptime task input

When triggered, the harness emits `agent.spawn` with:

```jsonc
{
  "type": "agent.spawn",
  "request_id": "...",
  "agent_id": "subagent-...",
  "agent_type": "sleeptime",
  "task": "<formatted task>"
}
```

Where `<formatted task>` is structured (in the prompt) like:

```
Trigger: <"every-10-turns" | "compaction">
Conversation: <conversation_id>

<For every-N-turns:>
Recent activity (last 10 assistant turns):
[turn 1 summary or excerpt]
[turn 2 ...]
...

<For compaction:>
Evicted content (about to be lost from active context):
[message 1]
[message 2]
...

Compaction summary that replaced these messages:
[summary text]

Your task: review this content. Identify durable insights worth consolidating
into the parent agent's system/* memory. Write focused, additive edits.
```

### Sleeptime behavior

Sleeptime subagent:

- Has `read-write` access to parent's `memory/`. Edits are visible to parent on parent's next turn.
- Has standard tools (Read, Edit, Write, Grep, Glob).
- Writes a one-paragraph result summarizing what it consolidated.
- Result flows back to parent — parent sees `agent.result` and can decide whether to surface it (typically silently logged, not user-visible).

---

## Subagent vs skills — distinction

| | Subagent | Skill |
|---|---|---|
| **What it is** | A separately-spawned agent process | A markdown playbook |
| **How invoked** | `Agent({agent_type: "recall", task: "..."})` | `Skill({skill: "using-tmux", args: "..."})` |
| **Where it runs** | Its own Bun harness process | In the calling agent's context |
| **What it returns** | A summary string + token usage | Loads its body into the agent's next turn as a synthetic message |
| **Use case** | Focused task with own context window | Procedural reference the agent reads inline |

Both are file-based (markdown bodies). Different mechanics.

---

## Open considerations

- **Concurrent subagent fanout.** If parent dispatches 5 subagents in parallel via 5 `Agent` tool calls, CP spawns 5 child harnesses. Memory cost ~1 GB peak. Soft limit: 3 concurrent per parent (configurable). When exceeded, queue.
- **Subagent invoking subagent.** Should a subagent be able to call `Agent` itself? Defaulting to yes (no special restriction); recursive depth is the parent's concern. CP doesn't enforce limits in v1.
- **Inheriting context vs starting fresh.** Subagents default to a fresh context seeded only by the type's system prompt + the task (`inherit_history: "none"`). Use `inherit_history: "full"` to clone all parent messages (filtered: tool I/O and reasoning blocks dropped), or `{ mode: "last_n_turns", n: N }` to clone the last N messages. N is capped at 20. All inherited content is also truncated at 8 KB per user message.
- **Sleeptime racing with next user message.** If user sends a message to parent the millisecond after compaction completes, does sleeptime get to run first? Yes — the harness queues sleeptime spawn before processing the new user message. The user might wait 5-10s extra for sleeptime to complete; acceptable for the kind of reflection it does.
- **Sleeptime memory writes invalidating parent's prompt cache.** Sleeptime writes to `system/*` → parent's prompt rebuilds → next turn's cache miss. This is the cost of sleeptime; expected.

---

## PR sequence (preview)

1. Subagent type loader (frontmatter parse, three-dir resolution).
2. Built-in subagent type definitions (lift bodies from letta-code patterns + the six above).
3. `Agent` tool implementation (emit spawn frame, await result, stream events to onUpdate).
4. CP subagent coordinator (spawn handler, init build, event forwarding, result return).
5. Subagent harness mode (`is_subagent: true` handling in init, exit-on-agent-end, result emit).
6. `inherit_memory` modes (none / read-only / read-write).
7. Subagent transcripts (sub-directory under parent's conversation).
8. Sleeptime trigger system in harness (counter, evaluator, dispatch).
9. Sleeptime task formatting (every-N + on-compaction payloads).
10. Tests: spawn/result roundtrip, parallel fanout, sleeptime triggers.
