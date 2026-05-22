# `storage` — On-Disk Layouts

> **Depends on:** [`architecture.md`](architecture.md)
> **Owns:** every file path Argos v2 reads or writes. Source of truth for filesystem layout, file formats, and migration targets.

---

## Top-level directory tree

All Argos v2 state lives under `~/.argos/` (override via `ARGOS_HOME` env var).

```
~/.argos/
├── control-plane.db          # SQLite — registry, channel config, cron tasks, settings
├── control-plane.pid         # PID of the running CP process
├── control-plane.log         # CP stdout/stderr (rotated)
├── subagents/                # globally-available subagent type definitions
│   ├── domain-expert.md
│   └── ...
├── logs/
│   ├── <agent_id>.stderr.log # one file per agent harness, rotated
│   └── ...
└── agents/
    └── <agent_id>/           # one directory per agent
        ├── home/             # agent's HOME directory — SSH keypair, .gitconfig, .profile
        │   ├── .ssh/
        │   │   └── id_ed25519      # ed25519 private key (mode 600)
        │   │   └── id_ed25519.pub  # ed25519 public key
        │   ├── .gitconfig          # user.name, user.email, init.defaultBranch
        │   ├── .tea/               # tea CLI config (Forgejo/GitHub login)
        │   │   └── config.yml
        │   └── .profile            # minimal env (PATH, HOME)
        ├── memory/           # MemFS git working tree (see § MemFS)
        ├── conversations/    # per-conversation transcript dirs (see § Transcripts)
        ├── subagents/        # agent-scoped subagent type definitions
        └── channels/         # per-channel runtime state (e.g., Matrix E2EE store)
            ├── matrix/
            │   └── store/    # matrix-bot-sdk persisted store (per-device keys, etc.)
            └── telegram/     # (post-v1)
```

---

## Agent home directory — `~/.argos/agents/<id>/home/`

Each agent gets its own HOME directory, provisioned at agent-creation time:

- **`.ssh/id_ed25519`** — Ed25519 SSH keypair (private mode 600, public `.pub`)
- **`.gitconfig`** — `user.name = <agent-name>`, `user.email = <agent-name>@argos.local`
- **`.profile`** — Minimal env (`PATH`, `HOME`), sourced by the agent's Bash tool
- **`.tea/config.yml`** — Tea CLI config (optional, populated via `argos agent setup-tea`)

The supervisor passes `HOME`, `USER`, `GIT_AUTHOR_*`, `GIT_COMMITTER_*` env vars
pointing at this directory for every spawned harness, so all git/SSH operations
resolve against the agent's own credentials.

---

## MemFS — `~/.argos/agents/<id>/memory/`

A real git working tree. Backed by a remote (bundled or external — see `control-plane.md` for git server hosting).

### Reserved structure

```
memory/
├── system/                  # files projected into the system prompt
│   ├── persona.md           # default block: agent personality
│   ├── human.md             # default block: what the agent knows about the user
│   ├── relationships.md     # other defaults can land here over time
│   └── ...
├── notes/                   # agent's free-form workspace (NOT projected, just present in tree index)
├── skills/                  # agent's user-defined skills
│   └── <name>/SKILL.md
└── ...                      # any file the agent wants to keep
```

**Projection rule:** files in `system/` are read by the harness and inserted into the system prompt as `<memory>` blocks (see `harness.md` for the exact assembly). All other files are present in the prompt only as a directory tree listing — agent uses `Read` / `Grep` / `Glob` to access content on demand. This mirrors letta-code's "in-context vs progressive" distinction (`letta-code/src/agent/prompts/system_prompt_memfs.md:8-12`).

**Limits:** v1 enforces no per-file character limits; the agent is trusted to keep `system/*` files reasonably sized. (Letta enforces 20K-50K char limits on blocks; we may add this later if drift becomes an issue.)

### Git management

- **Remote:** configured per agent in `control-plane.db` (default: bundled git server, optional external like Gitea/GitHub).
- **Auto-pull on harness boot:** harness runs `git pull` before initializing the agent loop. If pull fails (no remote, network issue), continue with local checkout, log warning.
- **Auto-commit + push on `agent_end`:** harness commits any tracked changes with message `"agent_end: <conversation_id> @ <ts>"` and pushes asynchronously. Agent does not need to run git commands itself (departure from letta-code, where the system prompt taught the agent to commit).
- **Conflict resolution:** `git pull --rebase` with strategy `-X theirs` for first cut. If remote diverges meaningfully (PWA edits while agent is mutating), favor remote. Loss of the agent's just-edited content is the price; revisit if it bites.
- **Suppress for subagents:** subagent harnesses are spawned with `is_subagent: true` in init; they skip auto-push and (configurable) skip auto-pull. Avoids subagent runs polluting commit history.

### File operations

- **Agent reads/writes:** via standard tools (`Read`, `Write`, `Edit`, `Grep`, `Glob`, `Bash`). No bespoke memory tools (departure from letta-code's `core_memory_replace` etc.).
- **PWA / CLI writes:** routed via CP → harness via `memory.write` frame (request/response). CP rejects writes if the harness is not running.
- **Concurrent writes:** harness is single writer for its own files. PWA edits are serialized through CP → harness → file. No file locking needed because there's only ever one writer process.

---

## Conversation transcripts — `~/.argos/agents/<id>/conversations/<conv_id>/`

Per-conversation directories, each containing per-UTC-day JSONL transcripts.

```
conversations/
├── default/
│   ├── 2026-05-04.jsonl
│   ├── 2026-05-05.jsonl
│   ├── current.jsonl -> 2026-05-05.jsonl  # symlink, updated at UTC midnight on next append
│   └── usage.jsonl
├── matrix-!abc:matrix.org/
│   ├── 2026-05-04.jsonl
│   ├── 2026-05-05.jsonl
│   ├── current.jsonl -> 2026-05-05.jsonl
│   └── usage.jsonl
├── inter-<peer_agent_id>/
│   └── ...
└── cron-<task_id>/
    └── ...
```

### Transcript file format

JSONL — one JSON object per line. Append-only. Files are never rewritten under normal operation. Each entry has a stable `id` referenceable across the system.

**Entry kinds:**

```jsonc
// User message
{ "id": "msg_01H...", "ts": "2026-05-05T20:30:00Z", "kind": "user",
  "content": "What's on my calendar today?",
  "source": "matrix" | "cli" | "external_gateway" | "inter_agent" | "cron",
  "source_meta": { /* source-specific, e.g., matrix sender, inter_agent from_agent */ } }

// Assistant text turn
{ "id": "msg_01H...", "ts": "2026-05-05T20:30:01Z", "kind": "assistant",
  "content": "Let me check.",
  "parent_id": null }

// Tool call (with full args + full result)
{ "id": "msg_01H...", "ts": "2026-05-05T20:30:02Z", "kind": "tool_call",
  "name": "Bash",
  "args": { "command": "docker ps" },
  "result": "CONTAINER ID  IMAGE  ... (full stdout)",
  "ok": true,
  "duration_ms": 42,
  "parent_id": "msg_<assistant_turn_id>" }

// Compaction (first-class entry — replaces a state.json approach)
{ "id": "msg_01H...", "ts": "2026-05-05T20:30:05Z", "kind": "compaction",
  "evict_range": ["msg_first_evicted", "msg_last_evicted"],
  "summary": "User and agent discussed the Argos migration. Key decisions: ...",
  "tokens_before": 174000, "tokens_after": 32000 }

// Reasoning block (optional, off by default; enable per-agent for debugging)
{ "id": "msg_01H...", "ts": "2026-05-05T20:30:01.5Z", "kind": "reasoning",
  "content": "The user is asking about ..." }

// System reminder / synthetic context injection (used by harness's transformContext)
// (NOT persisted to transcript by default; only shown in active context for one turn)
```

**What is NOT persisted:**

- Streaming chunks (per-token deltas). Only the final assembled message lands in transcript.
- System prompts (regenerated each turn from current memory state).
- Provider raw envelopes.
- `usage_statistics` and `ping` events from pi-agent-core.

### Active context derivation (boot replay)

On harness boot, the harness needs to seed pi-agent-core's `agent.state.messages` from disk. Algorithm:

1. List day-files in conversation dir, sorted oldest → newest.
2. Stream-read entries in order, building a list.
3. When a `compaction` entry is encountered, drop entries in `evict_range` from the list and insert the `summary` content as a synthetic assistant message at the eviction point.
4. Result is the active context.

**Optimization:** scan backward from newest; once you hit the most recent `compaction` entry, you have everything you need (everything before it is already summarized). Avoids walking the full year of files for a long-lived companion.

**Cache:** harness caches the result in memory. Subsequent reads (e.g., after appending a turn) just append to the cache; full re-replay only on boot or on `compact` frame.

### `usage.jsonl`

Sibling file, one entry per agent-step, append-only. Tracks token / cost / latency. Separate from transcript because access pattern is analytics-only (PWA dashboards, billing summaries).

```jsonc
{ "id": "step_01H...", "ts": "...", "conversation_id": "default",
  "model": "claude-sonnet-4-6",
  "prompt_tokens": 1200, "completion_tokens": 350,
  "cached_input_tokens": 800, "cache_write_tokens": 0,
  "reasoning_tokens": 200,
  "wall_ms": 2400, "api_ms": 2100,
  "tools_called": 1 }
```

Format borrowed from `letta-code/src/agent/sessionHistory.ts`.

---

## Subagent type definitions

Markdown files with frontmatter. Three resolution layers (most-specific wins):

1. **Per-agent:** `~/.argos/agents/<agent_id>/subagents/<name>.md`
2. **Global:** `~/.argos/subagents/<name>.md`
3. **Built-in:** `packages/harness/builtin-subagents/<name>.md` (bundled with harness binary)

### Frontmatter schema

```yaml
---
name: recall                      # required; used as agent_type in Agent tool
description: |
  Searches the agent's conversation transcripts and memory for relevant past context.
  Best for "did I talk about X?" or "what was that thing we decided?"
default_model: claude-sonnet-4-6  # optional; fallback to parent's model
default_tools:                    # optional; fallback to parent's full toolset
  - Read
  - Grep
  - Glob
default_inherit_memory: false     # optional; default false
---
```

Body of the markdown file is the system prompt sent to the spawned subagent.

### Loader behavior

On each `Agent` tool invocation:

1. Resolve `agent_type` against the three dirs in order.
2. If not found in any → tool error returned to parent: `unknown agent_type: '<name>'`.
3. Frontmatter parsed, body extracted, overrides from tool args applied.
4. CP spawns child harness with the resolved configuration.

Loaders re-walk dirs each invocation (filesystem reads are cheap). No restart needed when adding a new subagent type — just write the file.

---

## Channel runtime state

```
~/.argos/agents/<agent_id>/channels/
├── matrix/
│   └── store/                # matrix-bot-sdk's storage adapter
│       ├── account.json
│       ├── crypto/           # E2EE: account, devices, sessions, megolm sessions
│       └── ...
└── telegram/                 # post-v1
    └── ...
```

The harness instantiates the appropriate channel runtime on boot per `init.channels[*]`. State lives per-agent so process boundary == E2EE-trust boundary. Bot tokens live in `control-plane.db`, passed to harness via `init` frame in memory only — never written to the agent's data dir.

---

## Control-plane SQLite schema (`~/.argos/control-plane.db`)

Lightweight, single file, WAL mode. CP-only writer.

```sql
-- Agents (replaces v1's agents.json)
CREATE TABLE agents (
  id              TEXT PRIMARY KEY,    -- UUID
  name            TEXT NOT NULL UNIQUE,
  description     TEXT,
  model           TEXT NOT NULL,
  provider        TEXT NOT NULL,
  context_window  INTEGER NOT NULL,
  is_default      INTEGER NOT NULL DEFAULT 0,
  ephemeral       INTEGER NOT NULL DEFAULT 0,
  internal_token  TEXT NOT NULL,       -- 32-byte hex
  created_at      TEXT NOT NULL,
  data_dir        TEXT NOT NULL        -- always ~/.argos/agents/<id> in v1; reserved for relocation
);

-- Channel config (replaces v1's accounts.json + routing.yaml)
CREATE TABLE channels (
  agent_id        TEXT NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
  channel         TEXT NOT NULL,        -- "matrix" | "telegram"
  account_id      TEXT NOT NULL,        -- stable UUID; preserved across edits (v1 lesson)
  config_json     TEXT NOT NULL,        -- channel-specific (homeserver, token, e2ee paths, etc.)
  dm_policy       TEXT NOT NULL,        -- "pairing" | "allowlist" | "open"
  created_at      TEXT NOT NULL,
  updated_at      TEXT NOT NULL,
  PRIMARY KEY (agent_id, channel)       -- one of each channel type per agent (multi-bot-per-channel: post-v1)
);

-- Channel routing (chat_id → agent binding)
CREATE TABLE channel_routes (
  account_id      TEXT NOT NULL,
  chat_id         TEXT NOT NULL,
  chat_type       TEXT NOT NULL,        -- "direct" | "group" | "room"
  thread_id       TEXT,
  agent_id        TEXT NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
  conversation_id TEXT NOT NULL,        -- always tg-<chat_id> or matrix-!<room>:<server>
  enabled         INTEGER NOT NULL DEFAULT 1,
  created_at      TEXT NOT NULL,
  updated_at      TEXT NOT NULL,
  PRIMARY KEY (account_id, chat_id, thread_id)
);

-- Cron tasks
CREATE TABLE cron_tasks (
  id              TEXT PRIMARY KEY,     -- UUID
  agent_id        TEXT NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
  name            TEXT NOT NULL,
  description     TEXT,
  cron_expression TEXT NOT NULL,        -- standard 5-field cron
  timezone        TEXT NOT NULL,
  prompt          TEXT NOT NULL,
  conversation_id TEXT NOT NULL,        -- always cron-<id>
  enabled         INTEGER NOT NULL DEFAULT 1,
  expires_at      TEXT,
  last_fired_at   TEXT,
  fire_count      INTEGER NOT NULL DEFAULT 0,
  created_at      TEXT NOT NULL,
  updated_at      TEXT NOT NULL
);

-- Pairing codes (Matrix DM policy "pairing")
CREATE TABLE pairing_codes (
  code            TEXT PRIMARY KEY,
  account_id      TEXT NOT NULL,
  agent_id        TEXT NOT NULL,
  created_at      TEXT NOT NULL,
  expires_at      TEXT NOT NULL         -- TTL: 15 minutes
);

-- Settings
CREATE TABLE settings (
  key             TEXT PRIMARY KEY,
  value_json      TEXT NOT NULL
);
-- Known keys: pwa_bearer_token, external_gateway_token, default_model,
--   memfs_remote_default, memfs_remote_per_agent (json), git_server_port
```

**Migrations:** schema versioned via a `schema_version` setting; CP runs migrations on boot. v1 is version 1; future versions add ALTER TABLE statements in CP startup.

---

## Backup & recovery

- **MemFS:** git remote is the backup. Bundled remote stored in CP-managed git server's data dir (`~/.argos/git-server/<agent_id>.git`); backed up via `rsync ~/.argos/`. External remote: user's responsibility.
- **Transcripts + control-plane.db:** included in `rsync ~/.argos/`. SQLite WAL needs `sqlite3 ... ".backup"` for hot backup; cold backup (CP stopped) is just a file copy.
- **Channel state (Matrix E2EE):** included in `rsync ~/.argos/`. Lose this and the agent has to re-verify with conversation partners; Matrix protocol handles this gracefully.

---

## Migration targets (from Argos v1)

The migration script (see `migration.md`) reads from v1's data sources and produces v2 layouts:

| v1 source | v2 target |
|---|---|
| Letta-server `block` table → `system/<label>.md` files | `~/.argos/agents/<id>/memory/system/<label>.md` |
| Letta-server `messages` table → JSONL | `~/.argos/agents/<id>/conversations/default/<date>.jsonl` (split by date) |
| `data/middleware/agents.json` | `agents` table in control-plane.db |
| `data/middleware/settings.json` | `settings` rows in control-plane.db |
| `~/.argos/<id>/.letta/channels/*` | `channels` + `channel_routes` rows in control-plane.db; matrix store copied to `<id>/channels/matrix/store/` |
| `~/.letta/crons.json` (per-agent) | `cron_tasks` rows in control-plane.db |
| MemFS git working trees | initialized fresh from new git server; existing tracked files copied as the seed commit |

Migration is **destructive-to-v1's-position**: post-migration, v1 is read-only / archived. v2 is canonical.

---

## File operation patterns the harness implements

| Pattern | Where in v1 | Where in v2 |
|---|---|---|
| Atomic write (replace) | `fs.writeFile` (non-atomic, can corrupt on crash) | `fs.writeFile` to temp + `fs.rename` (atomic) |
| Append (transcript) | `fs.appendFile` (atomic for chunks < PIPE_BUF) | Same; one entry per call, < PIPE_BUF guaranteed |
| Day rollover | n/a | On first append after UTC midnight: open new day file, update `current.jsonl` symlink |
| Symlink update | n/a | `fs.unlink` + `fs.symlink` (race-tolerable for our use) |

---

## Open considerations

- **Compaction-rewrites-transcript edge case:** what if the user wants to delete a message? Current design says transcripts are append-only. Deletion would require either (a) a `delete` entry that the replay logic respects, or (b) explicit transcript rewrite. Defer to v2 of v2.
- **MemFS conflict behavior under concurrent edits.** The current "favor remote on conflict" might surprise the agent if PWA edits during an agent run. Real risk: PWA "save" overwrites the agent's in-progress thinking. Mitigation options: file-level locking, or render PWA edits as suggestions the agent acknowledges. Defer; mitigate if observed.
- **Transcript file size cap.** A chatty agent producing huge tool results (e.g., `cat large.log`) could blow a single day-file past reasonable size. v1 has no cap. Mitigation if needed: hard cap at e.g. 50 MB per day file with rotation to `<date>-1.jsonl`, `<date>-2.jsonl`. Leave for v2.
- **Subagent transcripts.** Should subagent runs persist their own transcripts? Yes — under the parent agent's tree at `conversations/<conv_id>/subagents/<subagent_id>/<date>.jsonl`. This way the parent (and the user via PWA later) can audit subagent trajectories. Add to subagents.md spec.

---

## PR sequence (preview)

1. Filesystem layout helpers in a `storage` package (path resolvers, atomic writers, symlink updaters).
2. SQLite migration runner + schema v1.
3. Transcript writer (append + day rollover + symlink update).
4. Transcript reader with compaction replay.
5. MemFS git wrapper (clone, pull, commit, push).
6. Subagent type loader (frontmatter parser, three-dir resolver).
7. Tests for each.
