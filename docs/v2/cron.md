# `cron` — CP Scheduler, Trigger Frames, Agent Tools

> **Depends on:** [`architecture.md`](architecture.md), [`protocol.md`](protocol.md), [`storage.md`](storage.md), [`control-plane.md`](control-plane.md), [`harness.md`](harness.md)
> **Owns:** the CP-hosted cron scheduler, frame-driven triggers, CLI-based agent scheduling via built-in skill.

---

## Architecture shift from v1

Letta-code's cron lived **in each agent's process** (`letta-code/src/cron/scheduler.ts`) — one `setInterval` loop per agent, persisting to per-agent `crons.json`. v2 inverts this: **one scheduler in the CP, all agents' tasks in one DB table**, fires happen via `cron_trigger` frames.

Reasons:

- One scheduler is easier to reason about than N.
- Tasks survive harness restarts cleanly (state lives in CP DB, not per-agent file).
- Adding a new task doesn't require harness restart — CP picks it up on next tick.
- Manual runs (`run-now`) can be triggered without the harness intermediating.

**v2 also moves cron management out of the agent's tool surface.** Instead of 5 `Cron*` tools, the agent uses the `argos cron` CLI via Bash, guided by the `scheduling` built-in skill. This saves ~535 tokens per turn from the cached prefix (the 5 tool descriptions + base prompt section are gone), at the cost of 2-3 extra turns the first time per session the agent loads the skill.

---

## Storage

`cron_tasks` table in CP's SQLite (schema in `storage.md`). Columns:

```sql
id              TEXT PRIMARY KEY     -- "cron-<uuid>"
agent_id        TEXT NOT NULL
name            TEXT NOT NULL
description     TEXT
cron_expression TEXT                -- standard 5-field (required for "cron" type, null for "fire_window")
timezone        TEXT NOT NULL        -- e.g., "America/New_York"
prompt          TEXT NOT NULL        -- user-message-shaped task input
conversation_id TEXT NOT NULL        -- always "cron-<task_id>"
enabled         INTEGER NOT NULL DEFAULT 1
expires_at      TEXT
last_fired_at   TEXT
fire_count      INTEGER NOT NULL DEFAULT 0
created_at      TEXT NOT NULL
updated_at      TEXT NOT NULL
schedule_type   TEXT NOT NULL DEFAULT 'cron'   -- "cron" | "fire_window"
jitter_minutes  INTEGER               -- random delay up to N minutes before firing
jitter_delay_until TEXT               -- ISO timestamp; set when jitter delay is active
fire_window     TEXT                  -- JSON: { start: "HH:MM", end: "HH:MM", max_fires: N }
pending_fires   TEXT                  -- JSON array of ISO timestamps (fire_window tasks)
max_fires       INTEGER               -- max fires before auto-disable; null=unlimited
```

### Schedule types

Two schedule types are supported:

1. **`cron`** (default) — standard 5-field cron expression. Evaluated with `cron-parser`. Optional jitter adds a random delay of up to `jitter_minutes` before firing.

2. **`fire_window`** — random fires within a daily time window. When the scheduler first detects `now` is inside the window, it generates `max_fires` random timestamps within the window with minimum spacing (`window_duration / (max_fires * 2)`). Each tick, it fires any due pending fire. Optional jitter adds an extra random delay per fire.

---

## Scheduler (CP-side)

A single in-process loop in CP. Each tick (30s) queries all enabled, non-expired tasks and dispatches to the appropriate evaluator:

```typescript
const TICK_MS = 30_000;

function tick() {
  for (const task of tasks) {
    if (!task.enabled) continue;
    if (task.expires_at && new Date(task.expires_at) <= now) { auto-disable; continue; }

    if (task.schedule_type === "fire_window") {
      evaluateFireWindowTask(task, now, db, fire);
    } else {
      evaluateCronTask(task, now, db, fire);
    }
  }
}
```

### Cron-expression evaluation

Use `cron-parser` (pure JS, Bun-compatible) to compute next-fire times. For each task: `next_fire_at` computed from `cron_expression + timezone + last_fired_at` (or `created_at` if never fired).

### Jitter (both types)

When a task with `jitter_minutes > 0` is due, the scheduler computes a random delay of `[0, jitter_minutes]` minutes and stores the absolute deadline in `jitter_delay_until`. On subsequent ticks, it skips the task until `jitter_delay_until` passes, then fires and clears the column.

### Fire window algorithm

1. Parse `fire_window` JSON: `{ start: "HH:MM", end: "HH:MM", max_fires: N }`.
2. Compute today's window boundaries in the task's timezone using `Intl.DateTimeFormat`.
3. Handle midnight-crossing windows (e.g., 22:00-02:00) by extending `windowEnd` to the next day.
4. On first entry to the window, generate `max_fires` random offsets with minimum spacing.
5. Sort offsets, convert to absolute timestamps, store in `pending_fires`.
6. Each subsequent tick: find the first pending fire <= now, apply jitter if configured, fire it, remove from pending.
7. When all pending fires are consumed, no more fires happen. Next day's window generates fresh fires.

### Missed-fire policy

If CP wakes up after sleeping (e.g., laptop was suspended), tasks may be overdue by hours. Policy:

- **Fire once** for each overdue task — not "catch up by firing N times."
- Log count of missed fires for observability.
- For `fire_window` tasks: missed windows are skipped entirely — no makeup fires. A missed glitch is itself a simulation event.

---

## Inbound frame: `cron_trigger`

Per `protocol.md`:

```jsonc
{
  "type": "cron_trigger",
  "task_id": "cron-uuid",
  "task_name": "morning_summary",
  "conversation_id": "cron-cron-uuid",
  "prompt": "Summarize what's on my calendar today."
}
```

Harness handler:

1. Look up the conversation `cron-<task_id>` (create if first time).
2. Append a transcript entry: `{kind: "user", content: prompt, source: "cron", source_meta: {task_id, task_name}}`.
3. Enqueue an agent run on this conversation.
4. Run proceeds as a normal turn. Result lands in the conversation's transcript.
5. **No reply destination** — cron-triggered runs don't have a user-facing channel. Output suppression is automatic (runtime checks `source === "cron"` and skips channel-adapter fanout). The agent must call `NotifyUser` to surface anything.

---

## Agent scheduling surface (skill + CLI)

The 5 cron tools (`CronAdd`, `CronList`, `CronRemove`, `CronPause`, `CronResume`) were removed from the tool surface. The agent schedules tasks by:

1. Loading the `scheduling` built-in skill (via the `Skill` tool).
2. Using `Bash` to invoke the `argos cron` CLI.

The scheduling skill lives at `packages/harness/builtin-skills/scheduling/SKILL.md` and covers: when to schedule, CLI usage, cron expression cheat-sheet, and examples for both standard cron and fire_window tasks.

---

## CLI integration

```bash
argos cron list <agent>                                            # list agent's cron tasks
argos cron add <agent> --name X --cron "..." --prompt "..."        # create standard cron task
argos cron add <agent> --name X --schedule-type fire_window --fire-window '{"start":"10:00","end":"18:00","max_fires":2}' --prompt "..." # create fire_window task
argos cron add <agent> --name X --cron "..." --prompt "..." --max-fires 1                  # one-shot task
argos cron add <agent> --name X --cron "..." --prompt "..." --jitter-minutes 15              # with jitter
argos cron add <agent> --name X --cron "..." --prompt "..." --jitter-minutes 15 --max-fires 3  # with jitter + max fires
argos cron remove <agent> --id <task-id>                           # delete
argos cron pause <agent> --id <task-id>
argos cron resume <agent> --id <task-id>
argos cron run-now <agent> --id <task-id>                          # immediate trigger
```

`run-now` calls `POST /api/agents/:id/cron/:task_id/run-now` which sends the `cron_trigger` frame immediately, bypassing the schedule.

---

## REST surface

```
GET    /api/agents/:id/cron
POST   /api/agents/:id/cron             # accepts schedule_type, jitter_minutes, fire_window, max_fires
PATCH  /api/agents/:id/cron/:task_id
DELETE /api/agents/:id/cron/:task_id
POST   /api/agents/:id/cron/:task_id/run-now
```

The `POST` endpoint is the consolidated single validation path for cron creation. It validates:
- `schedule_type === "cron"`: requires `name`, `cron`, `prompt`; validates cron expression.
- `schedule_type === "fire_window"`: requires `name`, `fire_window`, `prompt`; validates window shape (HH:MM format, max_fires 1-24).
- `jitter_minutes`: validated to be between 1 and 1440.
- `max_fires`: validated to be >= 1 if provided (null = unlimited).

---

## Conversation namespacing

Each cron task gets its own conversation: `cron-<task_id>`. This means:

- Each task has an isolated context — long-running task A doesn't pollute task B.
- Each task accumulates history over fires; the agent recalls the previous run's state on the next fire.
- Compaction operates per-task-conversation (independent budgets).

---

## Failure modes

- **Cron expression parse error at insert:** REST returns 400.
- **Timezone unknown:** similar.
- **Task fires while harness is down:** scheduler logs skip; task is not re-tried (one-fire-per-tick policy).
- **CP crash mid-tick:** on restart, scheduler recomputes `next_fire_at` for all tasks. Tasks may miss a tick (tens of seconds late). Acceptable.
- **DB write failure on `last_fired_at` update:** task is fired but record not updated; next tick may re-fire (idempotency depends on agent's prompt). Mitigation: fire and update in a single transaction; if update fails, don't fire (skip this tick). Conservative.

---

## Open considerations

- **Cron output routing.** v1: result in transcript only. v2 ideas: per-task `notify_channel` config (post to Matrix room when done), or user-facing summary aggregator.
- **Per-task timeouts.** A cron-triggered run could in principle run for hours. v1: no enforced timeout (matches user/external messages). If needed, add a `max_runtime_s` config per task, abort run when exceeded.
- **Cron-driven inter-agent.** A cron task is just a frame the harness handles. Could a cron be configured to invoke another agent (via `SendAgentMessage`) instead of itself? v1: no — cron talks to the owning agent. To dispatch to another agent, the owning agent uses the inter-agent tool itself in response to its cron prompt. Composable.
- **Scheduling skew across timezones.** If the user travels and the host's system clock changes, IANA timezone names handle DST and offsets correctly. We use the task's `timezone` for evaluation, not the system's local timezone. Solid.

---

## PR sequence (final)

1. `cron_tasks` table + CRUD helpers in CP.
2. Cron-parser integration + next-fire computation.
3. Scheduler tick loop (30s interval).
4. `cron_trigger` frame writer in CP, handler in harness.
5. ~~Agent-callable `cron_*` tools (CP-mediated request/response).~~ **Removed in #149** — replaced by scheduling skill + CLI.
6. CLI `cron list/add/remove/pause/resume/run-now` commands (+ `--schedule-type`, `--jitter-minutes`, `--fire-window` flags).
7. REST endpoints.
8. Jitter + fire-window schedule types. Storage schema extension, scheduler refactor, tests.
9. Tests: cron parsing, jitter delay, fire window generation, midnight crossing, backward compat, expired tasks.
