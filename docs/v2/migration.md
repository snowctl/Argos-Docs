# `migration` — Argos v1 → v2 One-Shot Import

> **Depends on:** [`architecture.md`](architecture.md), [`storage.md`](storage.md), [`control-plane.md`](control-plane.md)
> **Owns:** the script that, run once before cutover, imports Argos v1's state (Letta-server agents + memory blocks + messages, middleware's `agents.json` + channel configs + cron tasks) into v2's filesystem and SQLite layouts.

---

## When this runs

**Once, at cutover.** Daily-use Argos v1 is gracefully stopped. Migration script runs against v1's data sources (Letta-server REST + Argos middleware files + `~/.letta/`). v2 is started fresh on v1's data. v1 is archived.

Not designed for incremental sync. Not designed for ongoing dual-running. One direction, one shot.

**Pre-flight:** v1 must be running so the migration can read from Letta-server's REST API. Take a backup of v1's `data/` and `~/.letta/` directories before running.

---

## What gets migrated

### Agents

Source: Argos v1's `data/middleware/agents.json` (v3 schema per `Argos/middleware/src/agents/types.ts:52-60`).

Target: rows in v2's `agents` table (`storage.md` schema).

Mapping:

| v1 field | v2 column | Notes |
|---|---|---|
| `id` | `id` | Preserved exactly — same agent identity in v2 |
| `name` | `name` | |
| `description` | `description` | |
| `model` | `model` | Letta uses `provider/model-name` format (e.g., `openrouter/anthropic/claude-haiku-4.5`); v2 splits into separate columns — `provider` + `model` |
| `isDefault` | `is_default` | |
| `ephemeral` | `ephemeral` | Skipped if true (don't migrate ephemeral test agents) |
| `internal_token` | `internal_token` | Preserved |
| `gateway_port` | (dropped — stdio replaces this) | |
| `inter_agent.allowed_senders` | (dropped for v1 of v2; schema slot reserved) | |
| `createdAt` | `created_at` | |
| (derived) | `data_dir` | `~/.argos/agents/<id>` |
| (derived) | `context_window` | Looked up from model registry; fallback 200000 |

Per-agent data directory layout (`~/.argos/agents/<id>/`) is created fresh.

### Memory blocks

Source: Letta-server REST `GET /v1/agents/:id/core-memory/blocks` (per `Argos/middleware/src/letta-client/letta-client.ts:155-160`).

For each agent, fetch all blocks. For each block, write a markdown file to `~/.argos/agents/<id>/memory/system/<label>.md`:

```markdown
<!-- Migrated from Letta block: <label>, limit=<n>, read_only=<bool>, description="<desc>" -->
<value>
```

The HTML comment header preserves Letta-side metadata for inspection (won't appear in the assembled prompt because the projector reads file content as-is — wait, it would. So skip the comment, or use a markdown frontmatter block that the projector strips).

Decision: **frontmatter block, projector ignores frontmatter:**

```markdown
---
migrated_from_letta: true
original_label: human/core
original_limit: 20000
read_only: false
description: User's core identity
---
<value>
```

Projector in v2 (`memory.md` § How `system/*` files reach the system prompt) is updated to strip frontmatter before injecting content. This way the file is self-describing without leaking metadata into prompts.

**Label normalization:** Letta labels can contain `/` (e.g., `human/core`, `system/persona`). v2's filesystem maps `/` → directory. So `human/core` becomes `system/human/core.md`. The projector recurses one level into `system/`. (Alternative: flatten to `system/human-core.md`. Less faithful to Letta's hierarchy but simpler. Decision: preserve hierarchy.)

If the agent had any `system/*` block in v1, that's already at the right path; no transformation needed.

### Messages (conversation transcripts)

Source: Letta-server REST `GET /v1/agents/:id/messages?limit=<all>&before=<paginate>` (`Argos/middleware/src/letta-client/letta-client.ts:98-111`).

Target: per-conversation, per-day JSONL files in `~/.argos/agents/<id>/conversations/`.

Process:

1. For each agent, fetch all messages from the default conversation (paginated).
2. Group by UTC date of `created_at`.
3. For each (date, message) pair, append to `~/.argos/agents/<id>/conversations/default/<date>.jsonl`.
4. Translate Letta message types (`user_message`, `assistant_message`, `tool_call_message`, `tool_return_message`, `reasoning_message`, etc.) to v2's transcript entry kinds:

   | Letta `message_type` | v2 `kind` | Notes |
   |---|---|---|
   | `user_message` | `user` | Content extracted from Letta's structured payload |
   | `assistant_message` | `assistant` | |
   | `tool_call_message` | `tool_call` | Pair with following `tool_return_message` to combine into one v2 entry. Args from Letta are full (good — we preserve). |
   | `tool_return_message` | (merged into preceding tool_call) | |
   | `reasoning_message` | `reasoning` | Optional: only persist if user opts in via flag |
   | `summary_message` | `compaction` | Treated as a Letta-generated summary; insert as v2 compaction entry with `evict_range: []` (nothing to evict — this just records that Letta compacted at this point) |
   | `event_message` | (drop) | v2 doesn't have these |
   | `approval_request_message` / `approval_response_message` | (drop) | v1's approval flow not preserved |

5. For named conversations (PKL: `GET /v1/conversations?agent_id=<id>` → for each, `GET /v1/conversations/:id/messages`), create a separate conversation directory `~/.argos/agents/<id>/conversations/conv-<letta_conv_id>/` and apply same date-grouped transcript writing. (Most users have only `default`; this handles the rare named-conversation case.)

6. Update `current.jsonl` symlink to today's file (or latest existing).

### Channel config

Source: Argos v1 has channel config in two places:

- `data/middleware/agents.json` → `channels` field (TelegramChannelConfig + MatrixChannelConfig per agent).
- Filesystem at `~/.argos/<agent_id>/.letta/channels/<channel>/{accounts.json,routing.yaml}`.

The middleware writes both; they should be in sync. We migrate from `agents.json` (more authoritative) and verify against filesystem.

For each `agent.channels.matrix`:

- Insert row in `channels` table: `(agent_id, "matrix", account_id, config_json, dm_policy, ...)`.
- For each route in the corresponding `routing.yaml`: insert a `channel_routes` row.
- Copy Matrix E2EE store from `~/.argos/<agent_id>/.letta/channels/matrix/store/` → `~/.argos/agents/<id>/channels/matrix/store/`.

For each `agent.channels.telegram`: same pattern (though Telegram is post-v1 for v2; we still migrate the config so it's present, just won't be active until Telegram adapter ships in v2 of v2).

### Cron tasks

Source: `~/.letta/crons.json` per agent (post Snow's fork commit `55f32582 fix(cron): scope crons.json by cwd instead of $HOME`, this lives at `~/.argos/<agent_id>/.letta/crons.json`).

Target: rows in `cron_tasks` table.

Mapping straightforward — Letta's cron schema is essentially the same shape as v2's (per `cron.md` § Storage). Field-level transforms:

| Letta field | v2 column |
|---|---|
| `id` | `id` |
| `agent_id` | `agent_id` (validated) |
| `conversation_id` | `conversation_id` — overridden to `cron-<id>` per v2 namespacing |
| `name`, `description`, `cron`, `timezone`, `prompt`, `enabled`, `expires_at`, `last_fired_at`, `fire_count` | direct copy |

### Settings

Source: `data/middleware/settings.json` (Argos v1 settings — `gitSync` block per `Argos/middleware/src/settings/`).

Target: rows in `settings` table.

Specific mappings:

- `gitSync.remoteUrl` → `settings.memfs_remote_default` (this becomes the default git remote for new agents; existing agents may have per-agent overrides).
- `gitSync.token` → embedded in remote URL or stored as `settings.memfs_remote_default_token` (sensitive — encrypted in v2's DB, eventually).

Argos v1's `pwa_bearer_token` (used for PWA auth) → `settings.pwa_bearer_token`.

### MemFS git working trees

Source: `~/.letta/agents/<agent_id>/memory/` (each is a git repo with a remote pointing at `argos-git-memfs` sidecar or external).

Target: same path, but under `~/.argos/agents/<id>/memory/`.

Process:

1. For each agent, create `~/.argos/agents/<id>/memory/`.
2. `git init` + add v2's bundled git remote (or preserve the existing external remote URL from v1 settings).
3. Walk v1's `~/.letta/agents/<id>/memory/` working tree. Copy non-`.git` files into v2's `memory/`. (`system/*` already migrated above; copy notes, skills, etc.)
4. Initial commit: `git add -A && git commit -m "Migrated from Argos v1"`.
5. `git push origin main` to v2's bundled git server.
6. Verify clone-back works.

**Alternative considered:** clone v1's bare repo directly into v2's git server. Faster, preserves history. Use this if v1's git server exposes the bare repos accessibly (it does — `argos-git-memfs` does). This is the better path:

```bash
# For each agent with a v1 MemFS:
git clone --mirror http://<v1-memfs>:8285/v1/git/<agent_id>/state.git \
  ~/.argos/git-server/<agent_id>.git
# Then in agent's working tree:
git clone http://127.0.0.1:9418/<agent_id>.git ~/.argos/agents/<id>/memory
```

Preserves full git history (all the agent's mutations over time). **This is the chosen path.** Step (4)-(5) above is the fallback if v1 git is unreachable.

---

## What is NOT migrated

| v1 thing | Why not |
|---|---|
| Argos v1 PIDs, runtime state | Ephemeral; not data |
| Token registry / inter-agent rate limit state | Ephemeral; rebuilt on first use |
| ChatGPT OAuth sessions | Out of scope for v2 (provider connection model TBD) |
| Letta-server's `runs` / `jobs` / `steps` tables | Operational, not user data |
| Letta-server's `archival_passages` (vector store) | v2 doesn't have archival memory yet (deferred); if a user has data here, it's lost. v1 was barely using it; acceptable. |
| Letta-server's named-conversation `isolated_blocks` | Rare; can be migrated as plain copies under each named conversation's `system/` if needed. v1: skip; flag in migration log if any agent has them. |
| Sleeptime / multi-agent group state | Letta-server-side; not used by Argos v1 |
| User skills in MemFS | Wait — these ARE migrated (they're in `memory/skills/`, which is part of the working tree we copy). Just noting for clarity. |
| External gateway tokens | If `EXTERNAL_GATEWAY_TOKEN` env was set, it's migrated as `settings.external_gateway_token` |

---

## Verification

The migration script produces a verification report:

```
Migration completed at 2026-XX-XX HH:MM:SSZ.

Agents migrated: 7
  - echo (id=...): memory blocks=15, messages=4521, channels=[matrix], cron tasks=2
  - field-scout (id=...): ...
  ...

Channels migrated:
  - 5 matrix configs, 5 routes, 4 E2EE stores copied
  - 2 telegram configs (will be inactive until v2 telegram adapter ships)

Cron tasks migrated: 11

MemFS:
  - 7 git repos cloned via mirror; 7 working trees populated
  - History preserved (longest: 2,847 commits over 14 months)

Skipped:
  - ephemeral agents: 3 (skipped per policy)
  - archival_passages: 0 found (none to migrate)
  - approval messages: 23 dropped (v1 only)

Warnings:
  - 3 agents previously had their private git remotes go stale; operator should verify.
  - new pubkeys required: every v2 agent gets a fresh ed25519 SSH keypair at
    `~/.argos/agents/<id>/home/.ssh/id_ed25519.pub`. Operator must add each
    agent's new pubkey to the remote git server (Forgejo, GitHub, etc.) before
    the agent can push. See `argos agent show-pubkey <name>`.
  - agent "research-bot" has 1 named conversation "lit-review" with 18 messages. Migrated to conversations/conv-<letta_id>/ (verify post-migration).

Pre-cutover checklist:
  [ ] Verify all agents listed above match expected
  [ ] Run `argos start` against migrated data
  [ ] Check `argos agents` shows all agents in 'ready' state
  [ ] Send a test message via CLI to each agent; verify response
  [ ] Verify Matrix channels reconnect (check matrix-bot-sdk logs)
  [ ] Verify cron schedules with `argos cron list <agent>`
  [ ] Stop v1 supervisor; archive v1 data dir; switch DNS / external integrations to v2
```

---

## Failure handling

- **Letta-server unreachable mid-migration:** abort, leave v2 data dirs partially populated. Script is **resumable** — re-run with `--resume`; skips agents already fully migrated, picks up incomplete ones. Per-agent state file (`<v2_data>/migration-state.json`) tracks progress.
- **Disk full:** abort. Free disk, resume.
- **MemFS git clone fails:** fall back to working-tree-copy + fresh init (lose history; log warning).
- **Letta block fetch fails for one block:** log error, skip that block (write empty file with frontmatter noting failure). Don't abort whole migration.
- **Invalid character in label:** sanitize for filesystem (replace with `_`), log mapping in report.

---

## Cutover sequence

```
1. Backup: cp -a ~/.letta ~/.letta.backup-pre-v2
            cp -a /path/to/Argos/data /path/to/Argos/data.backup-pre-v2

2. Verify v1 stable: `argos-cli agents` lists expected agents, all in expected state.
                     Test one channel roundtrip.

3. Run migration (dry-run first):
   argos-v2 migrate --from-v1 ~/.letta --argos-data /path/to/Argos/data \
                     --letta-server http://127.0.0.1:8283 --dry-run

   Review the verification report; fix any flagged issues.

4. Run migration for real:
   argos-v2 migrate --from-v1 ... --commit

5. Stop v1: cd /path/to/Argos && argos-cli stop

6. Start v2: argos-v2 start

7. Verify: argos-v2 agents (all ready)
           argos-v2 chat <agent-name> "hello"  (sanity check each agent)
           Check channel reconnects in matrix-bot-sdk logs

8. Switch external integrations:
   - If a Matrix bot's access token / homeserver is the same, nothing to do
   - If you have webhooks pointing at v1's external gateway, point them at v2

9. Archive v1: mv ~/.letta ~/.letta.archived-2026-XX-XX
              # leave Argos repo on `main` branch as the v1 archive

10. Live in v2.
```

---

## CLI

The migration script is a CLI subcommand on the v2 binary:

```
argos migrate --from-v1 <path-to-letta-home>
              --argos-data <path-to-v1-argos-data>
              --letta-server <url>
              [--dry-run]
              [--commit]
              [--resume]
              [--skip-messages]      # for testing migration speed without bulk imports
              [--skip-memfs]         # ditto
              [--report-file <path>] # default stdout
```

---

## Open considerations

- **Reading from a stopped v1.** Step 1-3 above run with v1 still up. If user can't keep v1 up during migration (e.g., crash), we need an offline-mode that reads from v1's filesystem dumps and skips Letta-server-only data sources. v1 of v2: require v1 to be reachable.
- **Block-history preservation.** Letta tracks per-block edit history (`block_history` table). v2's MemFS git history serves the same role going forward, but Letta's pre-v2 history isn't imported. Acceptable loss for v1.
- **Agent-name uniqueness across migration.** v1 may have agents with non-unique names (no constraint enforced). v2's `agents.name` is UNIQUE. Migration aborts if collisions found; user must rename in v1 first (or add a `--rename-suffix` flag that auto-deduplicates).
- **Migration idempotency.** If run twice without `--resume`, the second run sees existing v2 dirs and... overwrites? skips? errors? v1 of v2: error if v2 data dir non-empty unless `--resume` or `--force`. Conservative.
- **Encrypted secrets.** Channel access tokens, Letta API keys, etc., are migrated as plaintext into SQLite. Long-term: encrypt at rest. v1 of v2: file permissions on `~/.argos/control-plane.db` (0600) is the only protection.
- **Down-migration / rollback.** Not provided. v2 → v1 is not a designed path. Backup is your rollback.

---

## PR sequence (preview)

1. CLI `argos migrate` skeleton (arg parsing, dry-run vs commit, state file).
2. Read v1 `agents.json`, populate v2 `agents` table.
3. Letta REST client (only the read endpoints we need).
4. Block fetch + MD file write.
5. Message fetch + JSONL write (paginated).
6. MemFS clone via mirror.
7. Channel config import (DB + E2EE store copy).
8. Cron task import.
9. Settings import.
10. Verification report generator.
11. Resumable state file.
12. End-to-end migration test on a snapshot of real v1 data.
