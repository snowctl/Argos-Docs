# Argos v2 — Master Architectural Spec

> **Status:** design-complete, pre-implementation. Resolved through interview rounds 2026-05-05; see `~/Pi Migration Feasibility Report.md` for the prior feasibility analysis that gated this work.

> **Branch:** `v2`. Argos v1 (the current `main`) keeps running unchanged for daily use until cutover.

---

## Goal

Replace the Letta runtime (Letta Server + Letta Code) with a leaner, owned stack built on `@mariozechner/pi-agent-core`. Argos becomes the harness instead of being a wrapper around someone else's harness. Three codebases collapse to one.

**The product the user gets is the same** — a personal companion agent reachable via channels — but every architectural decision below is now ours, not Letta's.

---

## Migration shape

**Greenfield, not strangler.** A new monorepo containing all v2 code. Daily-use Argos v1 continues running on `main`; v2 work happens on this `v2` branch (specs first) and ultimately becomes a new `argos/` repo. Cutover is a one-shot: spin up v2 process, run migration script (see `migration.md`), point external integrations at v2, retire v1.

Specs live here in `docs/v2/` of the existing Argos repo on the `v2` branch. When the new repo is created (PR #1 of the implementation phase), specs are copied over to seed its `docs/`.

---

## Stack

| Layer | Choice | Why |
|---|---|---|
| Agent loop | `@mariozechner/pi-agent-core` (~2K LoC, MIT) | Smallest correct primitive. Pure loop + tool calling + event stream. Pi's `pi-coding-agent` SDK is too coding-CLI-opinionated to wrap; we own the harness runtime around it. `pi-agent-core` handles the LLM loop; the harness (`packages/harness/`) handles everything else: conversation dispatch, transcript persistence, compaction, channel adapters, subagent correlation, sleeptime. See `harness.md`. |
| Runtime | Bun (CP, harness, CLI) | Faster cold start (matters for harness respawn), built-in TS, ergonomic spawn. PWA stays Node-flavoured (Next.js). Bun must be validated by the first PR before we commit. |
| Repo | Monorepo with workspaces | Cross-cutting PRs (protocol + CP + harness) land atomically. Pi convention of `packages/` everything. |
| Process model | One process per agent | OS-level isolation (Bash, channel adapters, native modules). Crash blast radius = one agent. Memory cost ~150-200 MB × N is acceptable for personal-companion scale (1-15 agents). |
| Wire | JSONL on stdio (CP ↔ harness) | Pi's RPC mode validated this exact pattern. No port allocation, lifecycle is unified, stderr stays for logs. See `protocol.md`. |
| Memory | Filesystem (MemFS) — git-backed working tree per agent | Memory IS files. Agent uses normal file tools (Read/Edit/Write/Grep). Workspace + projected `system/*.md` files. Git is sync/backup. See `storage.md`. |
| Conversation history | Per-conversation directory + per-day JSONL transcript files | Agent can grep its own history. Append-only. Compaction is a first-class transcript entry, not file rotation. See `storage.md`, `compaction.md`. |
| Channel runtime | In harness | Process-per-agent extends naturally. Per-agent E2EE state, failure isolation, lower streaming latency. CP only stores config; harness instantiates clients. See `channels.md`. |
| State store | Filesystem (per-agent dirs) + small SQLite for CP-owned state (registry, channel config, cron tasks) | No Postgres dependency. Backup story = `rsync ~/.argos/`. |

---

## Topology

```
┌───────────────────────────────────────────────────────────────┐
│ External: CLI / Matrix / (future PWA, Telegram)                │
└──────────────┬─────────────────────────┬──────────────────────┘
               │ HTTP (CLI)              │ Matrix protocol
               ▼                         │
┌───────────────────────────────┐       │
│ Control Plane (Bun service)   │       │
│  ├ Supervisor                  │       │
│  ├ Agent registry              │       │
│  ├ Channel config store        │       │
│  ├ Cron scheduler              │       │
│  ├ Inter-agent router          │       │
│  ├ Auth (bearer + internal)    │       │
│  ├ Subagent spawn coordinator  │       │
│  └ HTTP/REST surface           │       │
└──┬────────────────────────┬───┘       │
   │ JSONL stdin/stdout     │           │
   ▼                        ▼           ▼
┌─────────────────┐  ┌─────────────────┐
│ Harness (per    │  │ Harness (per    │  ← Matrix client lives here,
│ agent process)  │  │ agent process)  │    opens its own connection
│  ├ pi-agent-core│  │                 │
│  ├ Tool registry│  │                 │
│  ├ Channel adptrs│ │                 │
│  ├ Subagent ctrl│  │                 │
│  └ Memory + tx  │  │                 │
└─────────────────┘  └─────────────────┘
       │                     │
       └─────────┬───────────┘
                 ▼
       Filesystem: ~/.argos/agents/<id>/{memory,conversations,...}
       Plus a git server for MemFS (bundled or external)
```

All on one host. Multi-host is a v3 consideration.

---

## Resolved decisions (the design tree, locked)

| # | Decision | Resolution |
|---|---|---|
| Q1 | Migration shape | Greenfield (new repo, new branch, no `LettaClient`-shaped wrapping). |
| Q2 | Pi layer | `pi-agent-core` directly + thin harness we own. Not `pi-coding-agent`. |
| Q3 | Process model | One process per agent. |
| Q4 | Memory representation | MemFS-as-files (git-backed working tree, agent uses Read/Edit/Write). Not DB-backed blocks. |
| Q5 | Conversation history | Per-conversation directory; per-UTC-day JSONL files; append-only with first-class `compaction` entries (no separate state.json); usage stats in sibling `usage.jsonl`. |
| Q6 | Channel runtime | In harness. Multiple adapters per agent supported. CP holds channel config. |
| Q7 | CP↔harness wire | JSONL frames over stdio. Stderr for logs. |
| Q8 | Repo / runtime | Monorepo workspaces; Bun (CP+harness+CLI) pending PR-1 validation; PWA stays on Node/Next.js. |
| Q9 | v1 scope | Personal companion via Matrix, with subagents, cron, skills, inter-agent, compaction, sleeptime. PWA + Telegram deferred to v2 of v2 (i.e., post-cutover). |
| Q10 | Compaction in v1 | Yes, required for sleeptime "on-compaction" trigger. |
| Q11 | Subagent spawn | CP spawns child harnesses on parent's behalf. Parent emits `agent.spawn` frame; CP routes child events back as `agent.event` frames; result returned as `agent.result`. |
| Q12 | Compaction trigger | Harness self-triggers at 85% of context window; CP can also force via `{type:"compact"}` frame. |
| Q13 | Frame protocol | JSONL with `type` discriminator, `request_id` for request/response, routing fields where relevant. See `protocol.md`. |
| Q14 | Specs location | `docs/v2/` on `v2` branch in this repo. Master + component specs only for now (no PR specs yet). |

---

## v1 scope (what gets built before cutover)

**In:**

1. Monorepo skeleton + Bun validation + CI
2. `protocol` package (frame schemas, shared TS types)
3. `harness` package — pi-agent-core wrapping, JSONL stdio I/O, system prompt assembly, agent loop
4. `control-plane` package — supervisor, registry, REST, channel config store, cron scheduler, inter-agent router, subagent coordinator, auth
5. `cli` package — `argos start`, `stop`, `agents`, `chat`, `compact`, `cron`, `channels`
6. File tools (Read/Edit/Write/Grep/Glob/Bash) registered in harness
7. MemFS git integration (auto-pull on boot, auto-commit/push on agent_end)
8. Per-conversation transcript files with compaction entries
9. Matrix channel adapter in harness (E2EE, edit-in-place streaming, DM policies)
10. Subagent infrastructure (Agent tool, agent.spawn frame, types registry, lifecycle)
11. Built-in subagent types: general-purpose, explore, recall, reflection, fork, sleeptime
12. Skills system (built-in + MemFS-resident user skills, `Skill` tool)
13. Cron scheduler in CP + agent-callable `cron_*` tools
14. Inter-agent messaging (`Agent` for sub, `SendAgentMessage` + `ListAgents` for peer)
15. Compaction (summarizer, eviction, trigger)
16. Sleeptime (subagent type + every-N-turns trigger + on-compaction trigger)
17. Migration script (Letta blocks + messages → greenfield format)
18. End-to-end smoke + integration tests

**Out (deferred to post-cutover v2 of v2):**

- PWA + WS streaming UI fix
- Telegram, Slack, Discord channels
- Multi-host distribution
- Tool approval queue (use `bypassPermissions` mode for v1)
- Named conversations beyond auto-created (`default`, channel-derived, inter-agent pairs, cron-triggered)
- Anything else not in the In list above

**Estimated:** ~30-40 PRs, ~10-14 weeks of single-agent execution. Some PRs land in parallel (channel adapter independent of subagents; compaction independent of cron).

---

## Lift-and-shift inventory

Substantial pieces have prior art in letta-code or letta-server (MIT-equivalent for our use). The pattern is "lift the implementation, swap the integration points":

| v2 piece | Lifted from | Adaptation |
|---|---|---|
| File tools (Read/Edit/Write/Grep/Glob/Bash) | `letta-code/src/tools/impl/` | Tool registration interface (Pi's vs Letta's) |
| Matrix adapter | `letta-code/src/channels/matrix/` | Dispatch target = our harness conversation router; config from CP RPC instead of `accounts.json` |
| Pairing / DM policy logic | `letta-code/src/channels/{pairing,registry,routing}.ts` | Storage = CP DB instead of per-cwd files |
| Cron scheduler | `letta-code/src/cron/` | Move from per-agent in-process to CP-hosted single scheduler; trigger via stdin frame |
| Subagent infrastructure | `letta-code/src/agent/subagents/manager.ts` | Spawn our harness binary; CP-mediated event routing |
| Subagent types (markdown bodies) | `letta-code/src/agent/subagents/builtin/` | Copied verbatim with frontmatter cleanup |
| Skills (built-in markdown bundles) | `letta-code/src/skills/builtin/` | Verbatim |
| Permissions / hooks | `letta-code/src/permissions/`, `src/hooks/` | Storage path; Pi's `beforeToolCall` integration |
| MemoryGit | `letta-code/src/agent/memoryGit.ts` | Auto-commit/push automated by harness; agent doesn't need git knowledge |
| Compaction summarizer | `letta/services/summarizer/summarizer.py` | Ported to TS, wired to our transcript model |
| Supervisor pattern | `Argos/middleware/src/agents/supervisor.ts` | Lifted as pattern; rewritten for stdio (no per-agent ports) |

**Net:** roughly half the v1 PR count is "port + adapt" of prior art. This is what brings the timeline from 16-20 weeks (full rewrite) to 10-14 weeks.

---

## Component specs

Each component has its own spec. Read them in dependency order if reading top-to-bottom; they all depend on this master.

| Spec | Purpose |
|---|---|
| [`protocol.md`](protocol.md) | JSONL frame schema for CP↔harness IPC. Source of truth for the wire. |
| [`storage.md`](storage.md) | Filesystem layouts: MemFS, transcripts, agent registry, on-disk conventions. |
| [`harness.md`](harness.md) | Per-agent runtime: boot, agent loop, system prompt assembly, memory projection. |
| [`control-plane.md`](control-plane.md) | Supervisor, registry, REST surface, channel config storage, auth. |
| [`memory.md`](memory.md) | MemFS as files: how blocks become prompt, projection rules, agent's mental model. |
| [`subagents.md`](subagents.md) | `Agent` tool, types registry, spawn lifecycle, sleeptime triggers. |
| [`channels.md`](channels.md) | Adapter pattern, Matrix specifics, multiple-adapters-per-agent, conversation routing. |
| [`compaction.md`](compaction.md) | Summarizer prompt, eviction policy, trigger, sleeptime hookup. |
| [`cron.md`](cron.md) | CP scheduler, trigger frames, agent-callable cron tools. |
| [`inter-agent.md`](inter-agent.md) | Tool, frame routing, per-pair conversation namespacing. |
| [`migration.md`](migration.md) | One-shot import from Argos v1 (Letta blocks + messages → greenfield format). |
| [`testing.md`](testing.md) | Independent verification strategy: test pyramid, mock LLM, probe CLI, CI. The contract for "how does an agent know its PR works." |
| [`developing.md`](developing.md) | **Runbook** for an agent (or human) about to pick up a v2 issue. Concrete commands for the AFK loop, branch conventions, how to add tests / probe scenarios / frame types / REST endpoints, known quirks. |

---

## Out-of-scope for these specs (live elsewhere)

- **PR specs.** Not written yet. Each component spec ends with a "PR sequence" section listing the PRs that implement it; PR-level specs come when we're ready to begin implementation. `testing.md` carries the cross-cutting verification contract.
- **Bun validation.** First PR's deliverable; criteria listed in `harness.md` and issue #79.
- **PWA design.** Deferred to post-cutover v2 of v2.
- **Multi-host scaling.** v3 concern.

---

## Glossary

- **Harness** — per-agent Bun process running pi-agent-core + our shell. One harness == one agent.
- **Control plane (CP)** — Bun service that supervises harnesses, owns shared state, exposes REST.
- **MemFS** — per-agent git working tree at `~/.argos/agents/<id>/memory/`, where memory blocks live as files.
- **Conversation** — first-class harness concept; each conversation has its own transcript and (eventually) its own context budget. ID namespacing: `default`, `tg-<chat>`, `matrix-!<room>:<server>`, `inter-<peer>`, `cron-<task>`.
- **Frame** — JSONL message between CP and harness over stdio.
- **Subagent** — short-lived child agent dispatched by a parent via the `Agent` tool. Spawned by CP, runs in its own harness process, returns a result, terminates.
- **Sleeptime** — subagent type that fires on triggers (every N turns, or after compaction) to consolidate memory in the background.
- **Compaction** — eviction of old in-context messages, replaced by an LLM-generated summary, recorded as a first-class transcript entry.

---

## Open questions (NOT to be resolved during spec writing)

- **Bun viability.** PR-1 is the validation. If a transitive dep fails under Bun (rare; verify pi-agent-core, matrix-bot-sdk, key utility libs), we fall back to Node. Architecture is unchanged.
- **Matrix E2EE storage durability.** matrix-bot-sdk persists per-device session keys. We need to verify our chosen store (probably SQLite via the SDK's storage adapter) survives harness restarts cleanly. To be confirmed during channel adapter PR.
- **Subagent context budget when parent spawns 5+ in parallel.** Each subagent gets its own context window. Cost adds up. Defer to spec time when implementing parallel `Agent` invocations.
- **How "ad-hoc subagents not in the type registry" work.** Resolved-ish: parent uses `agent_type: "general-purpose"` with a detailed task. If the agent wants a reusable type, it `Edit`s a new file under `~/.argos/agents/<id>/subagents/`. No inline custom system prompt in the `Agent` tool call.

---

End of master spec. See component specs for detail.
