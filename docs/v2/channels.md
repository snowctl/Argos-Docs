# `channels` — Adapter Pattern, Matrix, Multiple-Per-Agent

> **Depends on:** [`architecture.md`](architecture.md), [`protocol.md`](protocol.md), [`storage.md`](storage.md), [`harness.md`](harness.md), [`control-plane.md`](control-plane.md)
> **Owns:** the in-harness channel adapter pattern, Matrix specifically (v1's only channel), conversation routing for channel messages.

---

## Implementation status

| Slice | Status |
|---|---|
| #94 — Matrix MVP (plain text + E2EE, open DM policy, markdown→HTML, multi-conv routing) | **Shipped on `v2`** |
| #95 — Matrix DM policies (pairing, allowlist) + pairing CLI | Deferred |
| #96 — Matrix streaming edit-in-place (m.replace + debounce) | Deferred |
| Telegram adapter | Deferred (post-cutover) |
| Slack / Discord adapters | Deferred (post-cutover) |
| Matrix media inbound (image / audio / file) | Deferred |
| Voice transcription | Deferred |

**What's actually wired in #94:**

- `harness/src/channels/types.ts` — `ChannelAdapter` interface, `ConversationDispatcher` contract.
- `harness/src/channels/factory.ts` — `makeAdapter(initConfig)` dispatch by channel name.
- `harness/src/channels/matrix/{adapter,client,format}.ts` — Matrix adapter, Bun-stable HTTP transport, markdown→HTML.
- `harness/src/runtime.ts` — multi-conversation queue, outbound `agent_event` routing to owning adapter.
- `storage/src/channels.ts` + migration v2 — `channels` and `channel_routes` tables, CRUD.
- `control-plane/src/rest.ts` — `GET/PUT/PATCH/DELETE /api/agents/:id/channels[/matrix]`.
- `control-plane/src/supervisor.ts` — `notifyChannelConfigUpdated` / `notifyChannelRemoved` send `channel.config_updated` frames.
- `cli/src/main.ts` — `argos channels list`, `argos channels matrix add/remove`.
- `cli/src/probe.ts` — `argos probe matrix` (gated on `MATRIX_HOMESERVER` / `MATRIX_BOT_USER` / `MATRIX_BOT_TOKEN`).

---

## Architecture

Channel runtimes live **in the harness** (Q6 Option B). Each agent's harness instantiates the channel clients it needs; bot tokens / access tokens come from the CP via `init.channels[*]` and `channel.config_updated` frames; channel state (E2EE keys, etc.) persists under the agent's data dir at `channels/<channel>/store/`.

CP holds **config**, not runtime: PWA/CLI edits go to CP REST → CP persists → CP forwards to harness via frame.

Multiple adapters can run in the same harness — Matrix + (future) Telegram + (future) Slack/Discord — each independent, each routing inbound messages to its own conversation namespace.

---

## Adapter contract

Every channel adapter implements:

```typescript
interface ChannelAdapter {
  channel: "matrix" | "telegram" | "slack" | "discord";

  /** Called once at harness init or on channel.config_updated. */
  start(config: ChannelConfig, dispatcher: ConversationDispatcher): Promise<void>;

  /** Called on channel removal or harness shutdown. */
  stop(grace_ms?: number): Promise<void>;

  /** Called by harness's agent_event handler for the conversations this adapter owns.
   *  Adapter decides how to render the event back to the platform (text send, edit-in-place, etc.). */
  handleAgentEvent(conversation_id: string, event: AgentEvent): Promise<void>;

  /** Returns true if this conversation_id belongs to this adapter (e.g., starts with "matrix-"). */
  ownsConversation(conversation_id: string): boolean;
}

interface ConversationDispatcher {
  /** Push a normalized message into a conversation. Triggers an agent run. */
  enqueue(input: { conversation_id: string; content: string;
                   source: "matrix" | "telegram" | ...;
                   source_meta?: any }): Promise<void>;
}
```

---

## Conversation routing

Each adapter owns conversations under a namespace:

| Adapter | Conversation prefix | Example |
|---|---|---|
| Matrix | `matrix-!<room_id>:<server>` | `matrix-!Wabc:matrix.org` |
| Telegram (v2) | `tg-<chat_id>` | `tg-12345` |
| Slack (v2+) | `slack-<channel_id>` | `slack-C012345` |
| Discord (v2+) | `discord-<channel_id>` | `discord-678901` |

The agent sees messages in these conversations the same way it sees `default` or `inter-<peer>` — just another conversation with its own transcript and context.

The harness's outbound `agent_event` dispatch checks each adapter's `ownsConversation(conversation_id)` and routes the event to the matching adapter for rendering.

---

## Matrix adapter (v1)

Built on `matrix-bot-sdk` (Bun-compatible — verify in PR-1 Bun validation).

### Config

Stored in CP's `channels` table (column `config_json`):

```jsonc
{
  "homeserver": "https://matrix.org",
  "user_id": "@argos-echo:matrix.org",
  "access_token": "syt_...",        // long-lived bot access token
  "e2ee_store_path": "/home/user/.argos/agents/<id>/channels/matrix/store",
  "device_id": "ARGOS_ECHO_DEVICE", // stable device identifier
  "auto_join_invites": true,
  "supported_msg_types": ["m.text", "m.image", "m.audio", "m.file"]
}
```

DM policy stored separately in `channels.dm_policy`:

- `pairing` — unknown room → adapter sends a one-time pairing message; user/CLI must `argos channels matrix pair <agent> <code>` to bind.
- `allowlist` — only `room_ids` in `channels.allowed_users` (config) admitted; auto-creates routing entry on first message.
- `open` — auto-bind any room.

### E2EE state

- Per-agent at `channels/matrix/store/`. Contains olm/megolm session keys, device list, identity keys, key backup pickle.
- matrix-bot-sdk's `RustSdkCryptoStorageProvider` (or `SqliteCryptoStorageProvider`) — pick the one with cleanest Bun compat in PR-1.
- **Critical:** if the store is lost, the agent loses ability to decrypt past messages. Backup story: `rsync ~/.argos/`. Don't put this in MemFS git (would leak secrets).

### Inbound message flow

```
matrix-bot-sdk sync loop (in harness)
  → on('room.message', handler)
  → adapter normalize
      • extract m.room.message body (text, or m.image with media URL, or m.audio with transcription via separate path)
      • for E2EE rooms: bot auto-decrypts via matrix-bot-sdk; we get plaintext
      • build content: text, or a structured representation if multimedia
  → check DM policy
      • pairing: if unknown room, send pairing prompt; do NOT enqueue
      • allowlist: if room not in allowlist, drop; do NOT enqueue
      • open: enqueue
  → dispatcher.enqueue({ conversation_id: "matrix-!<room>:<server>",
                         content, source: "matrix",
                         source_meta: { sender, event_id, room_id, timestamp } })
  → conversation dispatcher in harness:
      • appends transcript entry (kind: "user", source: "matrix")
      • enqueues agent run on this conversation_id
      • if no run in flight, immediately start
```

### Outbound message flow (streaming edit-in-place)

When the agent emits `agent_event` events for the matrix conversation:

```
agent_event: { type: "message_start", message_id }
  → adapter sends initial message: m.room.message with body "..."
  → record (matrix_event_id ↔ message_id) mapping

agent_event: { type: "message_update", message_id, delta }
  → adapter accumulates text
  → debounce 250ms (avoid spamming Matrix with edits per token)
  → adapter sends m.room.message with m.relates_to: { rel_type: "m.replace", event_id: <orig> }
     and body: { msgtype: "m.text", body: <full text so far>, formatted_body: <html> }

agent_event: { type: "message_end", message_id }
  → final flush of accumulated text via m.replace
  → mark message complete

agent_event: { type: "tool_execution_start", tool, args }
  → adapter sends a separate "I'm doing X..." message (visual breadcrumb)
     OR appends to a status thread (post-v1)

agent_event: { type: "tool_execution_end", tool, ok, result }
  → optional: edit the breadcrumb to "Did X (ok|err)"
```

### Markdown → Matrix HTML

Agent text often contains markdown. Matrix supports HTML in `formatted_body`. Use a markdown→HTML converter (e.g., `marked` or `markdown-it`) — lift `letta-code/src/channels/format.ts` style logic with light adaptation. Plain text fallback in `body`.

### Voice / image attachments

v1: receive only (transcription optional). The harness:

- Audio (`m.audio`): if `transcribe_voice: true` in config, download via Matrix media API, send to a transcription provider (e.g., OpenAI Whisper API), use the transcript as the message content.
- Image (`m.image`): download URL, attach to the agent's user message as a multimodal content block (Anthropic supports image inputs).
- File (`m.file`): for v1, just include the URL in the user message body; agent can `Bash curl` it if needed.

Outbound attachments (agent sending images/files): post-v1.

### Pairing flow

When `dm_policy: "pairing"` and an unknown room sends the agent a message:

1. Adapter generates a 6-character code (alphabet `ABCDEFGHJKLMNPQRSTUVWXYZ23456789`).
2. Adapter writes a `pairing_codes` row via CP-mediated `tool.request` (or harness has direct SQLite access — TBD; probably CP-mediated for clean separation).
3. Adapter sends to the room: "Hi — to pair this room with the agent, run: `argos channels matrix pair <agent> <code>`. Code expires in 15 minutes."
4. User runs the CLI command. CLI hits CP REST `POST /api/agents/:id/channels/matrix/pair` with `{code, room_id}`. CP validates code (matches account, not expired), inserts a `channel_routes` row for this room → conversation.
5. CP sends `channel.config_updated` to harness with refreshed routing.
6. Adapter rebuilds its routing map; subsequent messages from the room are processed.
7. CP sends a confirmation message via CP-mediated tool (or via CLI's REST response, with the agent posting "paired!" itself).

### Bang commands (DM-only)

Inside any DM room (exactly two members: bot + human), the user can manage
conversations directly without leaving Matrix:

| Command                          | Effect |
|---|---|
| `!conv help`                     | Show this list |
| `!conv list`                     | Show all conversations in this room with the active one marked |
| `!conv new [name]`               | Create a fresh conversation and switch to it. Without `name`, autonames `conv-N` |
| `!conv switch <name\|#>`         | Make another conversation active. `#` is the index from `!conv list` |
| `!conv delete <name>`            | Delete a conversation. The active one requires a second invocation with `confirm` |
| `!conv rename <old> <new>`       | Rename a conversation |

Mechanics:
- The first conversation in every room is labeled `general` automatically (backfill on migration v14).
- Subsequent conversations are stored as additional rows in `channel_routes` with
  the same `(account_id, chat_id, thread_id)` and different `conversation_id`/`label`.
  Exactly one row per chat has `active=1` (partial unique index in SQLite).
- The agent **does not** see the bang commands — the adapter intercepts them
  before enqueueing. A new conversation cold-starts (no system note).
- In rooms with more than two members the bot replies "Conversation commands are DM-only."

Forwards-compatibility: the `POST /api/agents/:id/conversations` CP endpoint takes a
`source` discriminator so CLI/PWA can hook in the same mechanism later without
new endpoints. Currently only `source.kind === "channel"` with `channel === "matrix"`
is implemented.

---

## Multiple adapters in one harness

Init sequence (from `init.channels[]`):

```typescript
for (const channelConfig of init.channels) {
  const adapter = createAdapter(channelConfig.channel);  // factory selects matrix vs telegram vs ...
  await adapter.start(channelConfig.config, dispatcher);
  adapters.push(adapter);
}
```

Outbound dispatch:

```typescript
function routeAgentEvent(conversation_id: string, event: AgentEvent) {
  const owner = adapters.find(a => a.ownsConversation(conversation_id));
  if (owner) owner.handleAgentEvent(conversation_id, event);
  // (else: not a channel-owned conversation — default, inter-, cron-, or PWA)
}
```

Adapters are independent. A bug in the Matrix adapter (e.g., transient network error) doesn't affect a Telegram adapter running in the same harness. Each adapter has its own retry/reconnect logic.

---

## Configuration changes mid-run

When PWA/CLI edits a channel's config (e.g., rotate access token):

1. CP `PATCH /api/agents/:id/channels/matrix` validates input.
2. CP updates `channels` row.
3. CP writes `channel.config_updated` frame to harness stdin.
4. Harness:
   a. Find adapter by `channel`.
   b. Call `adapter.stop(grace_ms: 5000)` (drain pending sends, disconnect).
   c. Construct new adapter instance with new config.
   d. Call `adapter.start(newConfig, dispatcher)`.
   e. Replace in `adapters` list.

Reload is per-channel — other channels keep running. No harness restart.

---

## Channel-config storage in CP — single source of truth

| What | Where | Why |
|---|---|---|
| Bot tokens / access tokens | `channels.config_json` in SQLite | One owner; never duplicated to disk in plaintext |
| DM policy + allowlist | `channels.dm_policy`, `channels.config_json.allowed_users` | Mutable from PWA |
| Routing (room → agent → conversation) | `channel_routes` table | Multi-row for many rooms per agent |
| Pairing codes | `pairing_codes` table | TTL-managed; cleaned up on expiry |
| E2EE crypto state | Filesystem at `<agent>/channels/matrix/store/` | Per-device, per-agent; not in DB (managed by SDK) |

PWA/CLI edits → CP DB → frame to harness → harness reloads. No file-on-disk channel config in v2 (departure from v1's per-cwd `accounts.json` / `routing.yaml`).

---

## Failure modes

- **Matrix homeserver unreachable on boot:** adapter.start() retries with exponential backoff (max 5min). Harness still emits `ready` (other adapters may be online); failed adapter is logged.
- **Access token expired or revoked:** SDK errors → adapter logs → all subsequent sends fail. CP can detect via missing `transcript.appended` events for outbound matrix messages, or via a future health endpoint per adapter. v1: log + manual rotation.
- **E2EE store corrupted:** harness logs error on init, falls back to non-E2EE rooms only (or stops the matrix adapter entirely — TBD). Recovery is manual: delete store, re-verify devices.
- **Sync gap (network outage):** matrix-bot-sdk handles via sync token persistence. On reconnect, fetches missed events. No special harness handling.
- **Inbound burst (100 messages while agent was down):** all enqueue to dispatcher, run sequentially. No drop. May appear sluggish.

---

## Open considerations

- **matrix-bot-sdk Bun compatibility.** Some versions use Node's `crypto` module deeply or have native bindings. Verify in PR-1 (Bun validation). If incompatible, options: (a) use `@vector-im/matrix-bot-sdk-crypto` rust bindings (works with Node's N-API which Bun mostly supports), (b) fall back to Node for the harness specifically (loses Bun cold-start win for harnesses with Matrix), (c) consider `simple-matrix-bot-sdk` or other lighter libs.
- **Edit-in-place spam.** Even with 250ms debounce, a long agent response produces many edits. Some clients render edits as separate messages or notifications. Mitigation: debounce more aggressively (500ms-1s), or send only on `message_end` for v1 (no per-token streaming UX). Trade-off TBD with real testing.
- **Multi-room threading.** Matrix supports threads (`m.thread` relation). Should we use a Matrix thread per agent conversation, or one room per conversation? v1 default: one room = one conversation. Threads add UX complexity; defer.
- **Bot identity verification.** Other Matrix users may want to verify the bot's keys before talking to it (cross-signing). v1: support it via matrix-bot-sdk's verification flows; document in CLI.
- **Channel send rate limits.** Matrix has per-room rate limits; sending many edits during a long agent response can hit them. SDK handles backoff; harness should not panic on 429.

---

## PR sequence (preview)

1. Channel adapter interface + factory (`harness/src/channels/adapter.ts`).
2. Matrix adapter skeleton — connect, sync, send plain text. (No streaming, no E2EE yet.)
3. Matrix E2EE store integration (storage path resolution, store init, key sharing).
4. Matrix DM policies (`pairing`, `allowlist`, `open`).
5. Matrix outbound — markdown → HTML formatting.
6. Matrix outbound — streaming edit-in-place (m.replace) with debounce.
7. Matrix inbound — multimedia handling (image, audio with optional transcription).
8. Pairing CLI command + CP REST endpoint.
9. Channel config CRUD CLI + REST.
10. `channel.config_updated` reload path in harness.
11. Tests: pairing flow, allowlist enforcement, message roundtrip in test homeserver.
