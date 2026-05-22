# `control-plane` — Supervisor + State Owner

> **Depends on:** [`architecture.md`](architecture.md), [`protocol.md`](protocol.md), [`storage.md`](storage.md)
> **Owns:** harness lifecycle, agent registry, channel config storage, cron scheduling, inter-agent routing, subagent spawn coordination, REST surface, auth.

---

## What the control plane is

A long-running Bun service. The single source of truth for "which agents exist" and "what config they have." Spawns and supervises per-agent harnesses. Routes IPC frames between harnesses (inter-agent, subagent parent↔child). Exposes a REST surface for CLI (and future PWA) consumers.

**One CP per host.** If we ever want multi-host, CP becomes the unit of replication; v1 stays single-host.

The CP **does not** run agent loops itself. It does not parse LLM output. It does not implement tools. It coordinates and stores.

---

## Process model

Single Bun process. Started by `argos start`, stopped by `argos stop`. PID written to `~/.argos/control-plane.pid`. Stdout/stderr → `~/.argos/control-plane.log` (rotated).

**On start:**

1. Acquire PID file lock (refuse if another CP is already running).
2. Initialize SQLite at `~/.argos/control-plane.db` (run migrations if needed).
3. Initialize bundled git server (if configured) — see § MemFS git server.
4. Start REST server (Bun.serve) on configured port (default 8080, loopback only).
5. Walk `agents` table; spawn a harness for each non-ephemeral agent.
6. Start cron scheduler.
7. Mark CP "ready"; CLI `argos status` returns OK.

**On stop:**

1. Stop accepting new REST requests.
2. Send `shutdown` frame to every running harness.
3. Wait up to `grace_ms` (default 5s) for each to exit.
4. SIGTERM stragglers, then SIGKILL after another 5s.
5. Stop cron scheduler.
6. Close SQLite.
7. Release PID file lock.
8. Exit 0.

---

## Supervisor

Reuses the **pattern** from `Argos/middleware/src/agents/supervisor.ts:101-553` (one of the more battle-tested pieces of v1). Reimplemented for v2's shape:

- **Spawn:** `Bun.spawn(harnessBin, ['--agent', agent_id], { stdio: ['pipe', 'pipe', 'pipe'], env: {...minimal env...}, cwd: '/tmp' })`.
- **Init handshake:** immediately after spawn, write `init` frame to harness stdin (built from `agents` row + `channels` rows + `cron_tasks` rows + applicable settings).
- **Wait for `ready`:** harness emits `ready` frame; CP marks agent state `ready` in registry. Timeout: 30s. If harness fails to emit `ready`, treat as fast-crash.
- **Per-agent stdin writer:** serializes outbound frames so they don't interleave.
- **Per-agent stdout reader:** line-buffered, parses JSON, dispatches per § Frame routing.
- **Per-agent stderr capturer:** writes to `~/.argos/logs/<agent_id>.stderr.log`, rotated.
- **Crash policy:** if harness exits with non-zero code OR exits within 2s of spawn, apply 5s backoff then respawn. Crash within 2s twice → exponential backoff capped at 5min. Persist crash count for inspection via CLI.
- **Process group:** spawn detached so SIGTERM hits the whole tree (covers any tools the harness spawned). On kill: `process.kill(-pid)`.
- **Orphan cleanup on CP boot:** since stdio is the channel, orphan harnesses are automatically detected (they have no parent to read from; they exit). Belt-and-suspenders: scan running processes for our harness binary path and kill any not in registry. Simpler than v1's env-name-sentinel matching.

### Harness handle (in-memory)

```typescript
type HarnessHandle = {
  agent_id: string;
  state: "spawning" | "ready" | "shutting_down" | "crashed" | "backoff";
  child: Subprocess;
  spawned_at: Date;
  ready_at: Date | null;
  crash_count: number;
  last_crash_at: Date | null;
  parent_id: string | null;       // set for subagents
  request_correlator: Map<string, Deferred<any>>;  // for tool.request awaits
};
```

CP holds `Map<agent_id, HarnessHandle>`. Subagent harnesses are first-class entries with `parent_id` set.

---

## Agent registry

Stored in SQLite `agents` table (see `storage.md`). CRUD via REST and CLI.

**Defaults written at agent create:**

- `internal_token`: random 32-byte hex.
- `data_dir`: `~/.argos/agents/<id>`. CP creates the dir + standard subdirs (`memory/`, `conversations/`, `subagents/`, `channels/`).
- `memory/`: `git init`, set remote (per `settings.memfs_remote_default` or per-agent), set agent's git config (committer name = "argos-agent-<short_id>"), seed with default `system/persona.md` and `system/human.md`.
- `home/`: per-agent identity directory. CP creates `.ssh/id_ed25519` keypair (ed25519, no passphrase), `.gitconfig` (user.name = agent name, user.email = `<name>@argos.local`), `.profile` (minimal env), and `.tea/` (empty, for tea CLI login). See `storage.md` § Agent home directory for the full layout.

**Defaults at agent delete:**

- Stop harness.
- Delete `agents` row (cascades to `channels`, `channel_routes`, `cron_tasks`, `pairing_codes`).
- Optionally delete `~/.argos/agents/<id>/` (gated on `?force=true` query param). Default keeps data on disk for recovery.

---

## REST surface

Bun.serve, loopback-only (`127.0.0.1`). Auth: `Authorization: Bearer <pwa_bearer_token>` (stored in `settings`). All endpoints below require it unless noted.

### Agents

| Method | Path | Body / Query | Returns |
|---|---|---|---|
| GET | `/api/agents` | — | `{ agents: AgentRecord[] }` |
| POST | `/api/agents` | `{ name, model, provider?, ... }` | `{ agent: AgentRecord }` (also spawns harness) |
| GET | `/api/agents/:id` | — | `{ agent: AgentRecord }` |
| GET | `/api/agents/:id/status` | — | `{ state, spawned_at, ready_at, crash_count, ... }` |
| PATCH | `/api/agents/:id` | partial `AgentRecord` | updated record (some fields trigger respawn) |
| DELETE | `/api/agents/:id` | `?force=true` | 204; deletes data if force |
| POST | `/api/agents/:id/abort` | `{ conversation_id? }` | 204; sends abort frame |
| POST | `/api/agents/:id/compact` | `{ conversation_id }` | 204; sends compact frame |

### Conversations / messages

| Method | Path | Returns |
|---|---|---|
| GET | `/api/agents/:id/conversations` | `{ conversations: { id, last_message_at, message_count }[] }` |
| GET | `/api/agents/:id/conversations/:conv_id/messages` | `{ messages: TranscriptEntry[] }` (paged, default last 50) |
| POST | `/api/agents/:id/conversations/:conv_id/messages` | `{ content }` → forwards to harness as `user_message` frame, returns ack |

(Reads of transcripts come from the filesystem directly; CP just serves the file content as JSON.)

### Memory

| Method | Path | Returns |
|---|---|---|
| GET | `/api/agents/:id/memory` | `{ files: { path, size, updated_at }[] }` (recursive list of `system/`) |
| GET | `/api/agents/:id/memory/:path` | raw file content |
| PATCH | `/api/agents/:id/memory/:path` | request/response: forwards to harness via `memory.write` frame |

### Channels

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/api/agents/:id/channels` | — | `{ channels: ChannelConfig[] }` (tokens redacted) |
| PUT | `/api/agents/:id/channels/matrix` | `{ homeserver, user_id, access_token, dm_policy, allowed_users? }` | persists; sends `channel.config_updated` to harness |
| PATCH | `/api/agents/:id/channels/matrix` | partial | persists + frame |
| DELETE | `/api/agents/:id/channels/:channel` | — | persists + frame |
| POST | `/api/agents/:id/channels/matrix/bind` | `{ room_id, conversation_id? }` | adds a `channel_routes` row |

### Cron

| Method | Path | Returns |
|---|---|---|
| GET | `/api/agents/:id/cron` | `{ tasks: CronTask[] }` |
| POST | `/api/agents/:id/cron` | `{ name, cron, prompt, ... }` → row + scheduler picks up |
| PATCH | `/api/agents/:id/cron/:task_id` | partial |
| DELETE | `/api/agents/:id/cron/:task_id` | 204 |
| POST | `/api/agents/:id/cron/:task_id/run-now` | sends `cron_trigger` frame immediately |

### Settings

| Method | Path | Returns |
|---|---|---|
| GET | `/api/settings` | `{ pwa_bearer_token: "(redacted)", memfs_remote_default, ... }` |
| PUT | `/api/settings` | partial |

### Health

| Method | Path | Returns |
|---|---|---|
| GET | `/healthz` | `{ ok, version, agents_total, agents_ready, uptime_s }` (no auth) |

---

## Frame routing

CP's per-agent stdout reader dispatches outbound frames:

| Frame | Action |
|---|---|
| `ready` | Mark harness state `ready` in registry; CLI `argos agents` shows it. |
| `fatal` | Log error; expect harness to exit imminently; supervisor will respawn. |
| `agent_event` | (Future) forward to PWA WS subscribers. v1: log at debug; no further action unless CLI is `tail`-ing. |
| `inter_agent.send` | Look up `to_agent` in registry; if running, write `inter_agent.receive` to its stdin (with full envelope). If not running or unknown, return error via `tool.response`. See `inter-agent.md`. |
| `agent.spawn` | Spawn child harness per `agent_type` resolution + overrides. Track `parent_id`. Forward child's `agent_event`s back to parent's stdin as `agent.event` frames. On child exit, send `agent.result` to parent. See `subagents.md`. |
| `tool.request` | Dispatch by `tool` field (cron.add, cron.remove, agents.list, ...). Process. Return `tool.response`. |
| `compaction.start` / `compaction.complete` | Log; emit metrics; (future) push to PWA WS. |
| `transcript.appended` | Log; (future) push to PWA WS for live updates. v1: no-op beyond logging. |

---

## Cron scheduler

In-process, single-threaded, runs in CP. Replaces letta-code's per-agent in-process scheduler.

- **Wake interval:** 30 seconds (cron precision is minute-level, 30s sleep is ample).
- **On wake:** SELECT enabled `cron_tasks` whose next-fire time is in the past. For each, send `cron_trigger` frame to that agent's harness, update `last_fired_at` and `fire_count`.
- **Skip if agent harness not running:** mark task as missed for that fire (log; don't queue indefinitely).
- **Lease coordination:** not needed (single CP per host).

See `cron.md` for full design.

---

## Subagent coordinator

When an `agent.spawn` frame is received from a parent harness:

1. Resolve `agent_type` against (parent's `subagents/` → global `~/.argos/subagents/` → built-in). Read frontmatter + body.
2. Compose subagent init: `is_subagent: true`, `parent_agent_id: <parent>`, model = override || default, tools = override || default. Use the body as `system_prompt_override` (passed via init).
3. Generate a transient agent_id (UUID prefixed `subagent-`).
4. Spawn harness as usual. Track in `Map<agent_id, HarnessHandle>` with `parent_id`.
5. Forward child's outbound `agent_event` frames as `agent.event` (with `agent_id: child_id`) to parent's stdin.
6. When child harness exits, collect final result (last assistant message before exit) and send `agent.result` to parent's stdin (with the parent's original `request_id`).
7. Clean up: remove from map, no respawn.

Subagent harnesses live entirely under CP supervision (Option A from Q11). Parent harness has no direct process handle to its child.

See `subagents.md` for full design.

---

## Inter-agent router

When `inter_agent.send` frame arrives from harness A:

1. Resolve `to_agent` (UUID or name).
2. If unknown or not in `ready` state → if `request_id` present, send `tool.response` with `ok: false, error`; else log and drop.
3. Otherwise, write `inter_agent.receive` to B's stdin: `{type, conversation_id: "inter-<A's id>" (default) or per spec, from_agent: A.id, from_agent_name: A.name, content}`.
4. B's harness handles via § Conversation dispatch in `harness.md` — appends to `conversations/inter-<A's id>/<today>.jsonl`, runs the agent, etc.
5. If `request_id` was present on A's send, after B accepts, send `tool.response` `{ok: true}` to A.

See `inter-agent.md` for routing edge cases.

---

## Channel config flow

1. PWA / CLI calls REST `PUT /api/agents/:id/channels/matrix` with new config.
2. CP writes/updates `channels` table row.
3. CP writes `channel.config_updated` frame to harness's stdin.
4. Harness reloads adapter (tear down old client, instantiate new with new config).
5. CP returns 200 to caller.

Bot tokens / access tokens **never persist outside the SQLite DB** (encrypted at rest later if needed). Passed to harness in memory via init / config_updated frames.

---

## MemFS git server

CP includes a bundled git server (HTTP/Smart protocol) listening on a configurable port (default 9418, loopback by default). Each agent's MemFS remote points at `http://127.0.0.1:9418/<agent_id>.git`.

**Bundling rationale:** removes the v1 sidecar (`argos-git-memfs` Docker container), simplifies deployment to a single Bun process. Implementation: a small HTTP wrapper around `git http-backend` (~150 LoC).

**Bare repo per agent** at `~/.argos/git-server/<agent_id>.git`. Created by CP at agent-create time. Backed up via `rsync ~/.argos/`.

**External remote override:** per-agent setting `memfs_remote_url` overrides the bundled remote (e.g., point to a Gitea/GitHub repo). CP just persists the URL; harness uses it as the git remote.

---

## Auth

**One bearer token** for all PWA/CLI calls — the `pwa_bearer_token` setting (initialized at first start, regeneratable). Sent as `Authorization: Bearer <token>`.

**Per-harness `internal_token`** is generated at agent create, stored in `agents` row, passed via `init` frame. Reserved for future use (today, stdio is the trust boundary; this token isn't validated). Future use case: when remote-host harnesses become a thing in v3, this token authenticates the network channel.

**No external gateway in v1.** v1's external gateway endpoint (the one Argos v1 exposes for "outside-the-box integrations") is deferred. If needed, add a dedicated bearer token + a `POST /v1/gateway/messages` endpoint that proxies into the right harness.

---

## CLI integration

CLI talks to CP via REST. See `cli.md` for command list (in summary):

- `argos start` — spawn CP daemon.
- `argos stop` — gracefully stop CP + all harnesses.
- `argos status` — `GET /healthz` + `GET /api/agents`.
- `argos agents` — list agents.
- `argos chat <agent> <text>` — `POST /api/agents/:id/conversations/default/messages`; tail `agent_event`s via a follow-up GET-or-WS (TBD).
- `argos compact <agent>` — `POST /api/agents/:id/compact`.
- `argos cron list/add/remove` — wraps cron REST.
- `argos channels add/remove/list` — wraps channel REST.

(`cli.md` to be drafted as part of the CLI component PR sequence.)

---

## Observability

- **Per-agent stderr:** captured to `~/.argos/logs/<agent_id>.stderr.log`, rotated daily.
- **CP stdout/stderr:** to `~/.argos/control-plane.log`, rotated daily.
- **Structured logging:** Pino-style JSON in CP; `pretty` mode for stderr in dev.
- **Metrics:** for v1, log-only. Future: Prometheus exporter on a side endpoint.
- **No PWA streaming in v1:** `agent_event` frames are received but not forwarded anywhere except logs (CLI `--watch` mode can subscribe).

---

## Failure modes

- **CP crash:** all harnesses lose stdin/stdout; they detect EOF and exit cleanly. On CP restart, registry is reloaded from SQLite, harnesses respawned. Conversations resume from disk state (transcripts intact).
- **Disk full:** SQLite writes fail → REST returns 500. Harness transcript writes fail → harness emits `fatal`, exits. Recovery is operational (free disk, restart).
- **SQLite corruption:** rare; recovery via WAL + last good backup. CLI `argos repair` (post-v1) can rebuild from filesystem state.
- **Cron scheduler skew:** if CP wakes late (system suspended), tasks accumulate. v1 fires only the most-recent occurrence per task per wake (not all missed). Configurable later.

---

## Open considerations

- **CP REST WebSocket for live events.** Eventually PWA wants `agent_event`s in real time. Plan: add a single CP→client WS endpoint that subscribes to all agents (auth-gated). Defer to PWA v2-of-v2.
- **Multi-tenant CP.** v1 = single-user. If we ever expose CP to multiple users, we need org/role/scoped auth. Not on v1 roadmap.
- **Backpressure on subagent fanout.** If the parent dispatches 5 subagents in parallel, CP holds 5 child harnesses + forwards their event streams + waits for all results. Memory cost for 5 short-lived child Bun processes is real (~1 GB peak). Soft limit per parent: 3 concurrent subagents to start. Configurable.
- **Settings UI for cron / channels in CLI.** Want this in v1, but the CLI can stay minimal (REST exists, CLI wraps thinly).

---

## PR sequence (preview)

1. CP package skeleton (Bun.serve, SQLite init, PID file, log setup).
2. Agent registry CRUD (REST + DB + helper functions).
3. Supervisor: spawn / kill / handle map / crash policy.
4. Init handshake (build init frame from DB rows, write to stdin, wait for ready).
5. Frame router skeleton (stdin reader, dispatch table; no-op handlers).
6. Channel config CRUD (REST + DB + frame on update).
7. Memory write proxy (REST + frame request/response).
8. Cron scheduler (poll loop, fire frames). See `cron.md`.
9. Inter-agent router (resolve target, write receive frame). See `inter-agent.md`.
10. Subagent coordinator (resolve type, spawn child, route events, return result). See `subagents.md`.
11. MemFS git server wrapper (HTTP backend, per-agent bare repos).
12. Auth (bearer middleware, settings init).
13. Health + status endpoints.
14. Tests for each.
