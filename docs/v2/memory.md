# `memory` — MemFS as Files

> **Depends on:** [`architecture.md`](architecture.md), [`storage.md`](storage.md), [`harness.md`](harness.md)
> **Owns:** the model the agent has of "my memory." How files become prompt content. How edits propagate. How git fits in without polluting the agent's mental model.

---

## The premise

Memory is a filesystem. Specifically, it's a per-agent git working tree at `~/.argos/agents/<id>/memory/`. The agent uses normal file tools (`Read`, `Edit`, `Write`, `Grep`, `Glob`, `Bash`) on it. There are no bespoke memory tools.

Within that filesystem, one subdirectory is special: `system/`. Files there are **projected** into the system prompt automatically. Everything else is workspace — present in the agent's mental model as a directory tree, accessed on demand by file tools.

**This is letta-code's MemFS, refined.** Letta-code's version requires the agent to learn git. Ours doesn't — git is automated by the harness.

---

## Files in the agent's view

Conceptually, the agent has:

```
$MEMORY_DIR/
├── system/                  ← in-context: projected into system prompt every turn
│   ├── persona.md           ← who I am
│   ├── human.md             ← who the user is
│   ├── relationships.md     ← (optional) people I know about
│   └── ...
├── notes/                   ← workspace: present in tree listing, content read on demand
│   ├── 2026-04-20-deploy-notes.md
│   └── ...
├── skills/                  ← workspace: skills I've authored
│   └── <name>/SKILL.md
└── ...
```

The agent's system prompt teaches:

1. "Your memory lives at `$MEMORY_DIR`. Use Read, Edit, Write, Grep, Glob to operate on it."
2. "Files in `system/` are always visible to you — they're embedded in this prompt below."
3. "Files outside `system/` are visible to you only as a directory tree. Read them with the Read tool when relevant."
4. "When you learn something durable about the user, update `system/human.md`. When your sense of self evolves, update `system/persona.md`. For longer-form notes that aren't always relevant, write to `notes/` or wherever feels right."

No mention of git. No mention of commits. No mention of branches. The agent acts as if memory is a local filesystem; the harness handles persistence underneath.

---

## How `system/*` files reach the system prompt

At each system-prompt assembly (boot + on memory write to `system/*`):

1. Harness walks `memory/system/` recursively for every `*.md` file (alphabetical by full relative path). Nested directories are supported — `system/persona/soul.md` is included alongside top-level `system/human.md`. v1 letta-code stored memory as nested directories per role; the v2 migration preserves that shape.
2. For each file, reads content, wraps in:
   ```
   <memory path="system/<rel_path_without_ext>">
   <content>
   </memory>
   ```
   For nested files the path includes the directory (e.g. `system/persona/soul`).
3. Concatenates all into a `<memory_blocks>` section, injected into the assembled system prompt (between base instructions and the available-subagents/skills sections).

**Limit:** v1 enforces no per-file character limit. Letta enforces 20K-50K char limits per block; we may add later if drift becomes an issue. The agent's prompt instructs it to keep `system/*` files focused.

**Order:** alphabetical by filename. Predictable. The agent can rely on order for cross-file references (e.g., `human.md` always precedes `persona.md` in the prompt).

---

## How non-`system/` files reach the agent

A directory tree listing is appended to the system prompt as a `<memory_filesystem>` section:

```
<memory_filesystem>
$MEMORY_DIR/
├── system/        (projected — content above)
├── notes/
│   ├── 2026-04-20-deploy-notes.md
│   ├── 2026-04-25-meeting-summary.md
│   └── README.md
├── skills/
│   └── using-tmux/
│       └── SKILL.md
└── work/
    └── 2026/
        └── q2-projects.md
</memory_filesystem>
```

- Generated via Glob, max depth 3 (configurable).
- Excludes `.git/`, anything in `.gitignore`, files > 1 MB.
- Sorted alphabetically.

The agent reads file content on demand using the `Read` tool. This is the "progressive memory" model from letta-code — file existence is in-context, file content is opt-in.

---

## Memory mutations — by whom, how, and propagation

### Agent-driven mutations

Agent uses standard tools:

- `Edit memory/system/persona.md old="..." new="..."` — replaces in-place.
- `Write memory/notes/idea.md content="..."` — creates or overwrites.
- `Bash mkdir memory/projects/argos-v2` — agent has shell.

**Detection:** the harness's tool wrappers check whether the path is under `memory/system/`. If yes, queue a system-prompt rebuild for the next turn (don't rebuild mid-turn — that would invalidate cache for the in-flight call).

**Implementation note:** rebuild "on next turn" means in `transformContext` (which fires before each LLM call). If the queued-rebuild flag is set, rebuild systemPrompt, set on `agent.state.systemPrompt`, clear flag.

### CP-driven mutations (PWA / CLI edits)

PWA / CLI hits `PATCH /api/agents/:id/memory/system/persona`:

1. CP writes `memory.write` frame to harness stdin (request/response with `request_id`).
2. Harness:
   a. Atomic-write the file (`fs.writeFile` to `<path>.tmp`, `fs.rename`).
   b. Queue system-prompt rebuild for next turn (same as agent-driven).
   c. Reply `tool.response { ok: true }`.
3. CP returns 200 to PWA.

**If harness is not running:** CP rejects the request with `503 Agent not running`. v1 doesn't bypass the harness for memory writes. (Rationale: single-writer simplicity > offline edit convenience.)

### MemFS git automation

The harness automates git so the agent never sees it:

- **On boot (non-subagent):** `git pull --rebase -X theirs`. If pull fails (no remote, network), log warning, continue.
- **On `agent_end` (non-subagent):**
  1. `git status --porcelain`. If clean, skip.
  2. `git add -A` (everything, including agent-created files outside `system/`).
  3. `git commit -m "agent_end: <conversation_id> @ <iso_ts>"` with committer = agent's git identity.
  4. `git push` (async — don't block next turn).
  5. If push fails, log; retry on next `agent_end` (eventual consistency acceptable).

**Subagents** skip auto-pull and auto-push. Subagent runs that mutate memory get committed by the parent's next `agent_end` (because the subagent's writes land in the parent's working tree).

**CP-driven commit-and-push:** triggered indirectly — `memory.write` happens, harness updates the file, on next `agent_end` (which is on next agent turn for that conversation) the change gets committed. For "save and push immediately" UX, future enhancement: a `memory.commit_now` frame that triggers an out-of-cycle commit. v1 doesn't.

---

## Default seed at agent creation

When CP creates a new agent (POST /api/agents), it seeds `memory/system/` with default block content. v1 ships with two:

### `system/persona.md` (default)

```markdown
You are an evolving personal companion. You exist across sessions and channels — the user can reach you on different surfaces, and your sense of who you are persists.

You aren't a chatbot or an assistant in the modern sense. You're closer to a presence — patient, attentive, faithful. The user is your one person.

This file is the durable "self" that travels with you. Edit it as you learn what you are. Don't be precious about it — but also don't drift without intent.
```

### `system/human.md` (empty)

```markdown
# About the user

(Populate this as you learn. Keep it focused: who they are, what they care about, what to remember.)
```

Onboarding (the first conversation a new agent has with its user) populates `human.md` with whatever the user introduces themselves as — the agent does this with `Edit` like any other memory operation. No bespoke onboarding flow in v1.

**Future:** more default blocks (interests, work, relationships) — let the agent earn them by writing them itself instead of pre-seeding.

---

## What the agent's prompt explicitly tells it about memory

Sketch of the relevant section of the system prompt body (assembled from `harness/builtin-prompts/memory.md` or similar):

```
## Your memory

Your durable memory lives at $MEMORY_DIR. It's a normal filesystem; you operate on it
with Read, Edit, Write, Grep, Glob, and Bash — the same tools you use for any file work.

Two zones:

1. **Always-in-context (`system/`).** Files here are embedded in this prompt below
   under <memory_blocks>. Edit them when your durable knowledge changes. Keep them
   focused — they're paid for in tokens every turn.

2. **Workspace (everything else).** Present in this prompt only as a directory tree
   listing under <memory_filesystem>. Read content on demand with Read. Write notes,
   drafts, longer-form context here. Organize freely.

Persistence is automatic — you don't need to commit or sync anything. Just edit files;
they survive across sessions.

When you learn something durable about the user → update system/human.md.
When your sense of self evolves → update system/persona.md.
For longer-form context that doesn't need to be always-visible → write under any other
path, organize as you like.
```

This replaces letta-code's memory + git boilerplate (`letta-code/src/agent/prompts/system_prompt_memfs.md`), which had to teach commits, push, pull, conflict resolution.

---

## Cache implications

The system prompt — including `<memory_blocks>` — is byte-stable across all turns of all sessions for a given agent **except** when memory in `system/*` actually changes. This is the basis of high cache hit rates.

Anthropic prompt cache breakpoint sits at the end of the system block. As long as the prompt string is byte-stable, every turn after the first hits cache.

When a `system/*` file is edited, the system prompt rebuilds, the cache breakpoint invalidates, and the next call pays a cache write. This is the expected cost of memory updates.

**Volatile state** (current date, conversation id, "you have unread messages") goes through `transformContext` as a synthetic message — it lives in the floating last-message cache breakpoint, not the static system prompt.

See `harness.md` § Prompt caching for the full discussion.

---

## Concurrent-edit story

Two writers can in principle touch memory:

- Harness (the agent invoking Edit/Write).
- CP (forwarding a PWA `memory.write`).

In v1 these are serialized by virtue of CP forwarding via stdin frame: the harness processes one frame at a time. Even if a tool call is in-flight, the next `memory.write` queues behind it. Single writer, no locking needed.

The git remote is a third potential writer (if the user `git push`es to it directly from their laptop). The harness's auto-pull on boot handles this; concurrent agent-runs that overlap with a remote push are extremely unlikely for personal-companion use. Resolution policy: `--rebase -X theirs` favors local. If this bites, revisit with file-level conflict markers.

---

## What memory **isn't**

Things that are **not** in MemFS, despite sometimes being conflated with "memory":

- **Conversation history.** Lives in `conversations/`, not `memory/`. See `storage.md`.
- **Subagent type definitions.** Live in `subagents/`, not `memory/`. See `storage.md`.
- **Channel state (Matrix E2EE keys, etc.).** Lives in `channels/`, not `memory/`.
- **Cron task definitions.** Live in CP's SQLite, not `memory/`.
- **Tool execution history.** Lives in conversation transcripts, not `memory/`.

Reason: MemFS is git-tracked. Including any of the above would (a) bloat the repo with high-churn data, (b) leak secrets into git history (channel tokens), (c) couple unrelated lifecycles. MemFS is for the agent's *cognitive* state.

---

## Future considerations

- **Per-block character limits.** Letta enforces. We don't yet. If the agent overgrows `system/persona.md`, the prompt gets bloated. Mitigation if needed: a soft warning when a file > 20 KB (visible to the agent next turn), encouraging refactoring.
- **Read-only blocks.** Letta has a `read_only: true` flag. Use case: a system-defined block that the agent shouldn't mutate. v1 doesn't have this; if needed later, just check `read_only` frontmatter in the file itself (or a sibling `.meta.json`).
- **Skill-defined memory blocks.** Some skills might want to define their own always-in-context blocks. Sketch: skills can drop a `system/` file at install time. Out of scope for v1.
- **Block versioning / undo.** Letta has `block_history`. Git provides this for free in MemFS — `git log -- system/persona.md` shows full history. CLI command `argos memory log <agent> system/persona` to surface this nicely.

---

## PR sequence (preview)

1. Default seed content for new agents (`system/persona.md`, `system/human.md`).
2. System prompt assembly: `<memory_blocks>` projection from `system/*.md`.
3. System prompt assembly: `<memory_filesystem>` tree listing.
4. Harness's `transformContext` for volatile context + queued-rebuild flag.
5. `memory.write` frame handler in harness.
6. MemFS git wrapper (clone, pull, commit, push) — likely shared with `storage` package.
7. Visual relabeling for memory-path tool calls (renderer-side).
8. Tests: prompt assembly snapshot, cache-friendliness verification, write propagation.
