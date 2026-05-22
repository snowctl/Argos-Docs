# `inter-agent` — Peer Messaging Between Agents

> **Depends on:** [`architecture.md`](architecture.md), [`protocol.md`](protocol.md), [`storage.md`](storage.md), [`harness.md`](harness.md), [`control-plane.md`](control-plane.md)
> **Owns:** the simpler-than-v1 routing for agent ↔ agent messages, per-pair conversation namespacing, the `SendAgentMessage` and `ListAgents` tools.

---

## What changed from v1

Argos v1's inter-agent flow was 4-hop:

1. Agent A's `SendAgentMessage` tool → HTTP POST to Argos `/api/agents/internal/inter_agent` (with per-agent `ARGOS_INTERNAL_TOKEN`).
2. Argos validates, looks up agent B's per-agent loopback gateway port.
3. Argos forwards via `POST http://127.0.0.1:<gateway_port>/v1/messages`.
4. Agent B's letta-code gateway server receives, looks up route, calls `registry.injectDelivery()`.

Three primitives (HTTP server in each harness, per-agent ports, per-agent tokens) existed because both A and B were external processes the supervisor couldn't talk to directly.

v2: stdio is a direct CP↔harness channel. **One frame from A through CP to B's stdin.** Three primitives collapse to one frame router in CP.

---

## v2 flow

```
Agent A (in harness 1):
  invokes SendAgentMessage({to: "research-bot", content: "Found three papers..."})
    │
    │ (tool body in harness 1)
    ▼
  emits stdout frame:
    { type: "inter_agent.send", to_agent: "research-bot",
      conversation_id: null, content: "Found three papers..." }
    │
    │ (CP reads from harness 1's stdout)
    ▼
Control Plane:
  resolves "research-bot" → agent_id
  looks up harness handle (must be ready)
  writes stdin frame to harness 2:
    { type: "inter_agent.receive",
      conversation_id: "inter-<harness_1_agent_id>",
      from_agent: "<harness_1_agent_id>", from_agent_name: "field-scout",
      content: "Found three papers..." }
    │
    │ (harness 2 reads from stdin)
    ▼
Agent B (in harness 2):
  conversation dispatcher receives
  appends transcript entry to conversations/inter-<harness_1>/<today>.jsonl
  triggers agent run on conversation_id "inter-<harness_1>"
    │
    │ (B's agent_event stream, tagged conversation_id="inter-<harness_1>")
    ▼
  B's Tool: SendAgentMessage({to: "field-scout", content: "Got them, summarizing..."})
    → loops back: B → CP → A's stdin
```

**Net:** 1 round-trip per direction, no HTTP, no per-agent ports, no separate auth tokens.

---

## Frame definitions

(Already in `protocol.md`; recapped here.)

### Outbound from sender (agent A's harness):

```jsonc
{
  "type": "inter_agent.send",
  "request_id": "uuid",                  // optional; if present, A awaits ack
  "to_agent": "research-bot",            // UUID or human-friendly name
  "conversation_id": null,               // null → use "inter-<sender_id>" on receiver
  "content": "Found three papers..."
}
```

`to_agent` accepts either a UUID (`agent-abc123...`) or a name (`research-bot`). CP resolves both via the `agents` table.

### Inbound to receiver (agent B's harness):

```jsonc
{
  "type": "inter_agent.receive",
  "conversation_id": "inter-<sender_agent_id>",
  "from_agent": "<sender_agent_id>",
  "from_agent_name": "field-scout",
  "content": "Found three papers..."
}
```

---

## Conversation namespacing — per-pair

Each unordered pair of agents gets its own conversation thread on each side:

| Side A's perspective | Side B's perspective |
|---|---|
| `inter-<B's_id>` | `inter-<A's_id>` |

The conversation thread is **symmetric in name structure but separate in content** — A's transcript of A↔B and B's transcript of B↔A live in their own data dirs. Each side writes its own perspective; they're not synchronized.

This means: B has a complete record of "everything A said to me and everything I said back to A," accessible at `~/.argos/agents/<B_id>/conversations/inter-<A_id>/`. The agent can `Grep` it for recall, `Read` specific days, etc.

If A wants to deliver a message to a specific *named* conversation on B (e.g., a side thread), A specifies `conversation_id: "topic-foo"` in the `inter_agent.send` frame. B receives it on `topic-foo` instead. v1: rare; default is per-pair.

---

## Conversation-routing semantics

CP's resolution of `inter_agent.send.conversation_id`:

| Value | What CP forwards as `inter_agent.receive.conversation_id` |
|---|---|
| `null` (omitted) | `inter-<sender_id>` (default per-pair) |
| explicit string | use as-is on receiver |

The receiver creates the conversation directory on first reference (no pre-registration).

---

## CP routing logic

```typescript
async function handleInterAgentSend(senderHarness, frame) {
  // Resolve to_agent
  const targetId = await resolveAgentId(frame.to_agent);  // UUID or name lookup
  if (!targetId) {
    return respondToSender(frame.request_id, { ok: false, error: "unknown agent" });
  }

  // Get target harness
  const target = supervisor.getHandle(targetId);
  if (!target || target.state !== "ready") {
    return respondToSender(frame.request_id, { ok: false, error: "agent not running" });
  }

  // Build receive frame
  const conversationId = frame.conversation_id ?? `inter-${senderHarness.agentId}`;
  await target.writeFrame({
    type: "inter_agent.receive",
    conversation_id: conversationId,
    from_agent: senderHarness.agentId,
    from_agent_name: senderHarness.agentName,
    content: frame.content,
  });

  // Optionally ack sender
  if (frame.request_id) {
    await respondToSender(frame.request_id, { ok: true });
  }
}
```

Ack is delivered as a `tool.response` frame back to the sender (since `SendAgentMessage` tool waits on `request_id` if it sets one).

---

## `SendAgentMessage` tool (sender side)

Args: `{to: string, content: string, conversation_id?: string, await_ack?: boolean}`.

Tool body:

```typescript
async function sendAgentMessageTool(args, ctx) {
  const request_id = args.await_ack !== false ? generateUUID() : undefined;
  emitFrame({
    type: "inter_agent.send",
    request_id,
    to_agent: args.to,
    conversation_id: args.conversation_id ?? null,
    content: args.content,
  });

  if (request_id) {
    const ack = await awaitToolResponse(request_id, { timeout_ms: 5000 });
    if (!ack.ok) throw new Error(ack.error);
    return { ok: true, delivered: true };
  } else {
    return { ok: true, delivered: "fire-and-forget" };
  }
}
```

Default `await_ack: true` (synchronous delivery confirmation; latency ~5ms loopback).

---

## `ListAgents` tool

Args: `{}` (or future filters).

Tool body:

```typescript
async function listAgentsTool(args, ctx) {
  const request_id = generateUUID();
  emitFrame({
    type: "tool.request",
    request_id,
    tool: "agents.list",
    args: {},
  });
  const response = await awaitToolResponse(request_id);
  return response.result;  // [{ id, name, description, state, capabilities? }, ...]
}
```

CP responds with the registry contents (excluding the calling agent itself by default; future: filter by `inter_agent.allowed_senders` if we add that ACL).

Returned shape:

```jsonc
[
  { "id": "<uuid>", "name": "research-bot", "description": "Helps with literature reviews", "state": "ready",
    "model": "claude-sonnet-4-6" },
  { "id": "<uuid>", "name": "homelab-ops", "description": "Manages servers", "state": "ready",
    "model": "claude-sonnet-4-6" }
]
```

The agent uses `description` to decide whom to message for what.

---

## Permissions / allowlists

v1 default: any agent can message any other agent. No ACL.

Future (v2 of v2): per-agent `inter_agent_allowed_senders` setting (list of agent IDs/names, or `"*"` for all). CP enforces; rejects sends from disallowed sources.

This was a v1 Argos concept (`AgentRecord.inter_agent.allowed_senders`); we keep the schema slot in the SQLite `agents` table but don't enforce in v1.

---

## Inbound message handling in receiver

When B receives `inter_agent.receive`:

1. Conversation dispatcher takes input.
2. Append transcript entry: `{kind:"user", content, source:"inter_agent", source_meta:{from_agent, from_agent_name}}`.
3. Enqueue agent run on `conversation_id`.
4. Run proceeds normally. Agent's response goes to its conversation transcript.
5. If agent's response includes a `SendAgentMessage` tool call back to A, the loop closes naturally.

**No special "inter-agent context" injection.** The conversation transcript itself is the context — over time, the agent and its peer build up a shared history visible to both (each side has its own copy).

---

## Permission mode for inter-agent runs

Inter-agent-triggered turns run in the agent's normal permission mode (default, allowlist, etc.). This is consistent with channel-triggered turns. A peer can request things of an agent that the user might also request; the same gates apply.

If a per-peer permission override is wanted (e.g., "trust agent X more than humans"), it's a v2 feature.

---

## CLI integration

```bash
argos send <from-agent> <to-agent> "message text"     # manual inter-agent send (mostly for debug/testing)
argos list-agents                                      # list all agents on this host (no auth context)
```

These wrap the existing REST endpoints; the agent normally invokes via tools, not CLI.

---

## Failure modes

- **Target agent not ready:** `tool.response { ok: false, error: "agent not running" }`. Sender's tool throws; the agent decides whether to retry, escalate, or proceed.
- **Target name resolves to multiple agents:** CP rejects with `error: "ambiguous name"`. Forces unique names (already enforced via UNIQUE constraint on `agents.name`).
- **CP crashes mid-routing:** sender gets no response, awaits timeout. After CP restarts, sender retries via tool. Idempotency: receiver might process the same message twice — for v1, accept this. To improve: include `idempotency_key` in `inter_agent.send` and have receiver dedupe within a window.
- **Receiver harness crashes during the run:** receiver's run is lost; sender's `await_ack` already returned ok (we ack on delivery to receiver's stdin, not on completion). The sender doesn't know about the receiver's crash. If the sender expected a reply, it will time out waiting. v1: acceptable.

---

## Open considerations

- **Idempotency.** Should `inter_agent.send` have an idempotency key (like the v1 external gateway does)? For v1 → no, simple. If repeated retries from sender become a problem, add it.
- **Inter-agent broadcast.** Send-to-many. Not v1. If wanted: tool variant `BroadcastAgentMessage({to: ["agent1", "agent2"], content: "..."})`.
- **Cross-host inter-agent.** Multi-CP / multi-host. v3.
- **Cyclic message loops.** Agent A sends to B; B replies to A; A replies to B; ... infinitely. Mitigation: rate-limit per-pair (CP-side, e.g., max 10 msgs/min). v1: log warnings if > 5 msgs/sec; rely on agent's good judgment otherwise.
- **Should inter-agent messages count as user activity for sleeptime triggers?** v1 default: yes (a message arrived, agent ran, `agent_end` fired, counter incremented). Future: configurable per-agent if it causes drift.

---

## PR sequence (preview)

1. `SendAgentMessage` tool body (emit frame, await ack).
2. `ListAgents` tool body (CP-mediated request).
3. CP `inter_agent.send` handler (resolve, route, ack).
4. CP `tool: "agents.list"` handler.
5. Receiver-side dispatch (just plumbing in conversation dispatcher to handle source: "inter_agent").
6. Agent-name uniqueness enforcement (DB constraint + REST validation).
7. Tests: roundtrip, ack timing, name-resolution, error cases.
