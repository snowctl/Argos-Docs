# `testing` — Independent Verification Strategy

> **Depends on:** [`architecture.md`](architecture.md), [`protocol.md`](protocol.md)
> **Owns:** the answer to "how does an agent (or human) independently verify a v2 PR works." Test pyramid, mocking strategy, the probe CLI, CI configuration.

---

## Philosophy

Every PR must be verifiable end-to-end without human review. "Looks right" is not acceptance; "the test passes and the probe agrees" is. This is the contract that lets agents work AFK on the issue queue.

Three layers of verification, each strictly stronger than the previous:

1. **Type checks + lint pass.** `bun run check`. No "looks compiled" — actually compiles.
2. **Tests pass.** `bun test`. Unit + integration + e2e, all green.
3. **Probe agrees.** `argos probe <scenario>` runs against a live local stack and produces machine-checkable output that asserts the system actually does the thing.

A PR that ships green CI but a failing probe is not done. A PR that ships a passing probe but the demo described in the issue can't be reproduced manually is not done. Both must hold.

---

## Test pyramid

```
                    ┌─────────────────────┐
                    │  Real-LLM smoke     │   <10 tests, env-gated, slow
                    │  (gated on API key) │   `bun test:llm`
                    └─────────────────────┘
                  ┌────────────────────────┐
                  │  E2E (full stack)      │   ~20 tests, real subprocesses,
                  │  bun test:e2e          │   mocked LLM provider
                  └────────────────────────┘
              ┌──────────────────────────────┐
              │  Integration                  │  ~50 tests, real cross-package
              │  bun test:integration         │  calls, mocked externals
              └──────────────────────────────┘
        ┌──────────────────────────────────────┐
        │  Unit                                 │  hundreds, fast, isolated
        │  bun test                             │  `bun test` (default)
        └──────────────────────────────────────┘
```

Default `bun test` runs unit only (fast feedback). CI runs all four levels.

---

## Unit tests

Per-package, alongside source. File convention: `<package>/src/foo.ts` → `<package>/test/foo.test.ts` (or co-located in `__tests__/`). Pure functions, no I/O, no subprocesses.

What to unit-test:

- Frame parsers / serializers (`packages/protocol`).
- Path resolution helpers (`packages/storage`).
- Cron expression evaluation (`packages/control-plane`).
- Tool argument validators (`packages/harness`).
- Subagent type frontmatter parsers.
- Eviction range computation.
- Markdown→HTML transformations.

Example pattern:

```typescript
// packages/protocol/test/frames.test.ts
import { test, expect } from "bun:test";
import { parseFrame, serializeFrame } from "../src/frames";

test("init frame round-trips", () => {
  const original = { type: "init" as const, agent_id: "a", /* ... */ };
  expect(parseFrame(serializeFrame(original))).toEqual(original);
});

test("malformed JSON throws ParseError", () => {
  expect(() => parseFrame("not json")).toThrow("ParseError");
});
```

**Rule:** every PR adds unit tests for new code. If a function has no test, the PR isn't done.

---

## Integration tests

Cross-package, but no real subprocesses. Mock CP↔harness stdio with in-memory pipes. Mock LLM with a stub provider that returns canned responses based on input shape.

What to integration-test:

- Frame router dispatches correctly across all frame types.
- Memory tool writes propagate to system-prompt rebuild on next turn (in-process, no spawn).
- Compaction algorithm against synthetic conversations (replay-shape correctness).
- Subagent coordinator's lifecycle (mocked child harness).
- Channel adapter contract conformance (mocked matrix-bot-sdk).

Example pattern:

```typescript
// packages/control-plane/test/integration/inter-agent.test.ts
import { test, expect } from "bun:test";
import { makeMockHarness, makeFrameBus } from "../../test-harness";
import { CP } from "../../src/cp";

test("inter_agent.send routes to target's stdin", async () => {
  const bus = makeFrameBus();
  const alice = makeMockHarness("alice", bus);
  const bob = makeMockHarness("bob", bus);
  const cp = new CP({ harnesses: { alice, bob } });

  await cp.handleOutbound(alice, {
    type: "inter_agent.send",
    to_agent: "bob",
    content: "hello bob",
    request_id: "req-1",
  });

  expect(bob.received[0]).toMatchObject({
    type: "inter_agent.receive",
    conversation_id: "inter-alice",
    from_agent: "alice",
    content: "hello bob",
  });
  expect(alice.received[0]).toMatchObject({
    type: "tool.response",
    request_id: "req-1",
    ok: true,
  });
});
```

Run via `bun test:integration`. Lives in `<package>/test/integration/` to keep the unit-test default fast.

---

## E2E (full stack) tests

Real CP process, real harness subprocess, real SQLite, real filesystem, real git server. **Mocked LLM provider** (because real LLM is non-deterministic and slow). Mocked Matrix homeserver (use `matrix-bot-sdk`'s test utilities or a stub).

These tests live at the repo root in `packages/e2e/` (a dedicated package).

What to e2e-test:

- "Spawn CP, create agent, send message via CLI, assert response shape" — the bedrock smoke.
- "Edit memory file, restart harness, verify memory persisted via next turn's prompt".
- "Force compaction on a long conversation, verify transcript has compaction entry and next prompt is shorter".
- "Spawn two agents, A sends to B via SendAgentMessage, B's transcript contains the message".
- "Add cron task with --run-now, verify trigger fires and result lands in cron-<id> conversation".

Each test is a black-box assertion against the running stack:

```typescript
// packages/e2e/test/smoke.test.ts
import { test, expect } from "bun:test";
import { startCP, stopCP, argos } from "../src/harness";

test("echo agent end-to-end", async () => {
  await startCP({ llm: "mock", mock_responses: { "hello": "world" } });
  await argos("agent create --name test-agent --model claude-sonnet-4-6");

  const out = await argos("chat test-agent 'hello'");

  expect(out.stdout).toContain("world");
  expect(out.exit).toBe(0);

  // Verify transcript was written
  const transcript = readTranscript("test-agent", "default");
  expect(transcript).toMatchObject([
    { kind: "user", content: "hello" },
    { kind: "assistant", content: "world" },
  ]);

  await stopCP();
});
```

`startCP()` spins up CP in a tempdir with overridden `ARGOS_HOME`. Mock LLM is a stub provider configured at boot. Tests are isolated by tempdir.

Run via `bun test:e2e`. Slow (each test ~1-3 seconds for CP boot). Run in CI on every PR.

---

## Real-LLM smoke tests

Same shape as E2E but with a real Anthropic API key. Gated by env var; skipped if absent. Used for verifying things only a real LLM exhibits — tool-use trajectories, prompt-cache behavior, instruction-following on edge cases.

```typescript
// packages/e2e/test/real-llm/cache-hits.test.ts
import { test, expect } from "bun:test";
import { startCP, argos, getUsageLog } from "../../src/harness";

const skipIfNoKey = process.env.ANTHROPIC_API_KEY ? test : test.skip;

skipIfNoKey("steady-state cache hit rate > 80%", async () => {
  await startCP({ llm: "anthropic" });
  await argos("agent create --name cache-test --model claude-sonnet-4-6");

  // First turn — primes the cache
  await argos("chat cache-test 'hello, my name is alice'");
  // Subsequent turns — should hit cache for system + tools + most messages
  for (let i = 0; i < 5; i++) {
    await argos(`chat cache-test 'turn ${i}'`);
  }

  const usage = getUsageLog("cache-test", "default");
  const cacheRatio = usage.slice(1).reduce(
    (acc, u) => acc + u.cached_input_tokens / u.prompt_tokens, 0
  ) / 5;

  expect(cacheRatio).toBeGreaterThan(0.8);
});
```

Run via `bun test:llm`. Skipped without `ANTHROPIC_API_KEY`. Run in CI nightly with a budget-capped API key.

---

## Matrix testing

For unit tests, mock matrix-bot-sdk's `MatrixClient` interface.

For integration tests, use [`testcontainers`](https://node.testcontainers.org/) to spin up a Synapse Docker container per test (or per test suite). Slow but real:

```typescript
// packages/e2e/test/matrix/encrypted-room.test.ts
import { test, expect, beforeAll, afterAll } from "bun:test";
import { GenericContainer, StartedTestContainer } from "testcontainers";
import { startCP, argos } from "../../src/harness";

let synapse: StartedTestContainer;
let homeserverUrl: string;

beforeAll(async () => {
  synapse = await new GenericContainer("matrixdotorg/synapse:latest")
    .withExposedPorts(8008)
    // ... config
    .start();
  homeserverUrl = `http://localhost:${synapse.getMappedPort(8008)}`;
});

afterAll(async () => {
  await synapse.stop();
});

test("agent responds in encrypted room", async () => {
  // Provision a bot account on the test homeserver
  // Configure agent with channels: matrix
  // Create encrypted room, invite bot
  // Send message as user account
  // Assert agent's message appears in the room (decrypted)
});
```

Synapse boot is ~10-30s. Tests are gated by `RUN_MATRIX_TESTS=1` to keep the default `bun test:e2e` fast.

In CI: a dedicated `matrix` job runs these on every PR that touches `packages/harness/src/channels/matrix/` or related code (path-based GitHub Actions filter).

---

## Git testing

The bundled git server in CP is tested by:

- Unit: HTTP request/response shape mocking.
- Integration: spin up the bundled server in a tempdir, `git clone` from it, write commits, push, verify the bare repo received them.

```typescript
// packages/control-plane/test/integration/git-server.test.ts
test("MemFS round-trip via bundled git server", async () => {
  const { url, port } = await startBundledGitServer({ baseDir: tempdir() });
  await initBareRepo(tempdir() + "/agent-a.git");

  const workdir = await mkdtemp("workdir-");
  await execGit(workdir, "clone", `${url}/agent-a.git`, ".");
  await writeFile(workdir + "/system/persona.md", "I am alice");
  await execGit(workdir, "add", "-A");
  await execGit(workdir, "commit", "-m", "test");
  await execGit(workdir, "push");

  // Verify by re-cloning to a different dir
  const verify = await mkdtemp("verify-");
  await execGit(verify, "clone", `${url}/agent-a.git`, ".");
  expect(await readFile(verify + "/system/persona.md", "utf8")).toBe("I am alice");
});
```

---

## The probe CLI

`argos probe <scenario>` runs a canned scenario against the running CP and produces machine-checkable output. The agent uses this to verify "the system actually does the thing the issue describes."

Patterned after Argos v1's `argos-cli probe` (which has a dedicated skill in `.claude/skills/argos-cli-probe/`). For v2, the probe CLI ships incrementally — each PR that adds a capability also adds a probe scenario for that capability.

### Scenarios (added by their respective PRs)

| Scenario | Added by PR | What it verifies |
|---|---|---|
| `argos probe spawn` | #79/#80 | CP starts, an agent can be created, harness reaches `ready` state |
| `argos probe echo` | #80 | Send a message, receive an agent_event, no real LLM |
| `argos probe llm` | #81 | Real LLM round-trip (gated on API key) |
| `argos probe tools` | #82 | Each file tool executes successfully (Read /tmp, Write/Edit/Bash on a tempdir) |
| `argos probe transcripts` | #83 | Send N messages, kill harness, restart, verify transcript replay restored prior context (next turn references prior content) |
| `argos probe memory` | #84 | Edit memory file via Edit tool, verify next prompt's `<memory_blocks>` reflects change; verify CLI relabel for memory paths |
| `argos probe memfs-git` | #85 | Trigger agent_end, verify commit landed on bundled git server bare repo |
| `argos probe compact` | #86 | Force compaction, verify transcript has compaction entry, verify next prompt is shorter |
| `argos probe subagent` | #87 | Invoke Agent tool, verify subagent runs and result returns; verify nested transcript |
| `argos probe sleeptime` | #89 | Configure every-N=2, verify sleeptime fires after 2nd turn |
| `argos probe skill` | #90 | Invoke Skill tool, verify body appears in agent's next-turn context |
| `argos probe cron` | #91 | Add cron with --run-now, verify trigger fires |
| `argos probe inter-agent` | #92 | Two agents, A sends to B, verify B's transcript |
| `argos probe matrix` | #94 | Gated on env: `MATRIX_HOMESERVER`, `MATRIX_BOT_USER`, `MATRIX_BOT_TOKEN`. Asserts channel persists, lists, and agent reaches ready. Sending an actual Matrix message + asserting reply requires a sender bot + test room — out of #94 scope, set up by operator. Set `MATRIX_DISABLE_E2EE=1` for unencrypted dev probe. |
| `argos probe migrate` | #97 | Dry-run on a fixture v1 install, verify report matches expected |

### Output format

JSON (machine-parseable) by default; pretty-print with `--human`. Each scenario emits a structured assertion log:

```json
{
  "scenario": "echo",
  "ok": true,
  "duration_ms": 542,
  "assertions": [
    { "name": "agent_created", "ok": true },
    { "name": "ready_received", "ok": true, "elapsed_ms": 287 },
    { "name": "echo_response_matches", "ok": true, "expected": "echo: hello", "actual": "echo: hello" }
  ]
}
```

Exit code 0 on `ok: true`, non-zero otherwise. CI runs probe scenarios as a GitHub Actions step; agents working PRs run them locally and paste the JSON output into the PR description.

---

## Snapshot tests

For things where the *content* matters, not just the *behavior*. Use Bun's snapshot testing.

What to snapshot:

- Assembled system prompt for a fresh agent (verifies cache-friendly assembly hasn't drifted).
- Frame schemas (verifies wire compatibility hasn't broken).
- Default seed memory file content (`system/persona.md`, `system/human.md`).
- Compaction summarizer prompt (so prompt changes are explicit, reviewed events).

```typescript
test("default system prompt assembly", () => {
  const prompt = assembleSystemPrompt({
    memory_root: "/tmp/test/memory",
    subagents: [],
    skills: [],
  });
  expect(prompt).toMatchSnapshot();
});
```

Snapshots live in `__snapshots__/`. Updates require `bun test --update-snapshots` and explicit reviewer approval (snapshots show up as a diff in the PR).

---

## CI configuration

`.github/workflows/ci.yml` (or Forgejo equivalent if available):

```yaml
name: CI
on: [push, pull_request]
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bun run check       # biome lint + format + tsc
      - run: bun test            # unit only (fast)

  integration-and-e2e:
    runs-on: ubuntu-latest
    needs: unit
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bun test:integration
      - run: bun test:e2e        # real subprocesses, mocked LLM

  matrix-e2e:
    runs-on: ubuntu-latest
    needs: unit
    if: contains(github.event.pull_request.changed_files, 'packages/harness/src/channels/matrix')
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: RUN_MATRIX_TESTS=1 bun test:e2e -- matrix

  real-llm-nightly:
    runs-on: ubuntu-latest
    if: github.event.schedule
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bun test:llm
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

CI is set up in #79 (Bun + monorepo). Each subsequent PR adds tests as required by its acceptance criteria.

---

## What every PR includes

Acceptance-criteria checklist every PR must satisfy:

1. **`bun run check` passes** — biome lint + format + tsc. No warnings.
2. **`bun test` passes** — unit tests for new code added; existing tests not broken.
3. **`bun test:integration` passes** — integration tests for new cross-package code added.
4. **`bun test:e2e` passes** — at least one e2e scenario covering the slice (probe scenario counts).
5. **Probe scenario added (where applicable)** — the table above tells you which.
6. **Probe output captured in PR description** — JSON output of the relevant `argos probe <scenario>` runs, demonstrating the demo from the issue actually works.
7. **Snapshots updated and reviewed** if the change affects assembled prompts, frame schemas, or default content.
8. **Spec updated** if the implementation diverges from the spec — the spec is the contract; if the contract changes, the doc changes.

A PR that ticks 1-2 but not 3-7 is not done. The agent re-checks before opening for review.

---

## Independent verification — the AFK loop

How an agent works through an issue:

1. **Read issue + referenced specs.** Understand the slice's contract.
2. **Plan the implementation.** Identify packages touched, files to create/modify.
3. **Write tests first** (where TDD applies):
   - Unit tests for new functions.
   - Integration test for the slice's main path.
   - E2E probe scenario (script the demo from the issue).
4. **Implement** until tests pass.
5. **Run the full local CI matrix:** `bun run check && bun test && bun test:integration && bun test:e2e`.
6. **Run the probe scenario:** `argos probe <scenario>`. Capture JSON.
7. **Verify the demo manually** (simple commands from the issue's "Demo" line). Capture transcript.
8. **Open PR** with: tests added, probe output, demo transcript, spec updates.

If any step fails, the agent debugs. If they can't resolve, they either (a) revise the implementation, (b) push back on the spec/issue with a question (PR description, not bypass), (c) escalate.

**Critical rule:** "I think it works" is not done. "The probe says it works AND I reproduced the demo locally AND the test suite is green" is done. Evidence > belief.

---

## Mock LLM provider

A package-level utility that swaps Anthropic with a deterministic stub. Used by integration + e2e tests (real LLM is for `bun test:llm` only).

Configuration via env / init:

```typescript
// packages/harness/src/llm-mock.ts
export function makeMockProvider(responses: Record<string, string>) {
  return {
    async stream(messages, opts) {
      const last = messages[messages.length - 1].content;
      const reply = responses[last] ?? `mock response for: ${last}`;
      yield { type: "message_start", message_id: "mock-1" };
      for (const ch of reply) {
        yield { type: "message_update", delta: ch };
      }
      yield { type: "message_end" };
      return { usage: { input_tokens: 100, output_tokens: reply.length } };
    },
  };
}
```

CP startup accepts `--llm mock --mock-responses <json-file>` to wire this in. E2E tests use it for predictability.

For more sophisticated stubbing (e.g., tool-call trajectories), the mock can be configured with branching response patterns matched against the message thread.

---

## Open considerations

- **Real Matrix in CI is heavy.** Synapse Docker boot is ~10-30s. Mitigation: only run on PRs touching matrix code (path filter); cache image; consider a leaner stub homeserver if Synapse becomes a bottleneck.
- **Probe scenarios as living docs.** The probe scenarios ARE the demo. We could publish them as runnable examples in `docs/v2/probes/` for users to copy.
- **Determinism of LLM mocks vs real-world drift.** Mocked tests catch regressions in our code; they don't catch model behavior changes. Real-LLM nightly is the safety net. Manual probe runs (with real LLM, ad-hoc) catch the rest.
- **Test data for migration.** A fixture Argos v1 install (small, anonymized) committed to the repo for migration script tests. ~1 MB of fake-agent data. Lives at `packages/cli/test/fixtures/argos-v1-snapshot/`.
- **Coverage gates.** Not enforced for v1; encouraged. If coverage discipline slips, add a Codecov gate at e.g. 70% for new code.

---

## PR sequence (preview)

This spec doesn't have its own PRs — it's a strategy doc. The pieces it describes land across many PRs:

- `bun run check` setup, basic `bun test` config: in #79.
- Test harness helpers (`packages/e2e/`): bootstrapped in #80, expanded as new slices land.
- Mock LLM provider: in #81 (when real LLM lands, mock for testing it lands too).
- Probe CLI scaffolding: in #80 (`argos probe spawn`, `argos probe echo`).
- Each subsequent PR adds its probe scenario per the table above.
- `bun test:integration` + `bun test:e2e` scripts in root package.json: in #79.
- CI workflow: in #79.
- Synapse Docker testcontainers integration: in #94 (Matrix adapter).
