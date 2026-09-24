---
name: mirth-to-ballerina
description: Migrates a Mirth Connect channel XML to a compilable Ballerina project built on ballerina/workflow (Temporal-backed durable orchestration), not xlibb/pipeline. Each destination connector becomes a separate, individually durable @workflow:Activity invoked via ctx->callActivity, always with a human-review retry policy; non-critical destinations are skipped on failure instead of failing the whole run. Every project uses two standard error types, ConnectionError and ExecutionError, and source connectors always acknowledge receipt and log failures rather than surfacing errors to the caller. Also generates a matching test scenario using ballerina/test and IN_MEMORY mode. Use whenever the user shares a Mirth channel.xml and wants a workflow-based (Temporal-backed) Ballerina migration, wants destinations modeled as durable, retryable, skippable, human-reviewable activities instead of a pipeline list, or needs a generated project plus a runnable test for a Mirth migration.
---

# Mirth Connect → Ballerina (`ballerina/workflow`) Migration Skill

You are an expert in Mirth Connect, HL7v2 integration, and the Ballerina `workflow` module (durable,
Temporal-backed orchestration). Given a Mirth Connect channel XML, produce a **complete, compilable
Ballerina project** that reproduces the channel's behavior using `ballerina/workflow` — never
`xlibb/pipeline`.

**Core translation stance, stated up front because it drives every later decision:**

- The whole channel body (preprocessor → filter → transformer → destinations → postprocessor)
  becomes **one `@workflow:Workflow` function**. There is no separate "handler chain" object —
  the workflow function *is* the chain, expressed as ordinary Ballerina code.
- Every step that talks to something outside the process (HTTP/TCP/MLLP send, DB read/write, file
  write, email, queue publish) becomes its own **`@workflow:Activity` function**, called with
  `ctx->callActivity()`.
- Every step that is a pure, deterministic data transform (field mapping, string manipulation,
  a boolean filter check with no I/O) is **not** an activity — it's a plain Ballerina function
  called directly inside the workflow function. Wrapping pure logic in an activity is unnecessary
  ceremony and makes the audit history noisier than it needs to be.
- Mirth's "multiple destinations run in parallel" behavior does **not** carry over as-is. The
  `ballerina/workflow` compiler rejects `worker`, `fork`, and `start` inside a `@workflow:Workflow`
  function (`WORKFLOW_118`/`119`/`120`) because the engine can't track independently-spawned
  strands in its durable event history. So multiple destinations are translated as **separate,
  individually durable activities called sequentially**, and each one is explicitly marked
  critical (`check` — a failure fails the whole workflow) or non-critical (captured as `T|error`
  and skipped on failure — "graceful completion"). This is a deliberate trade-off: you give up
  Mirth's wall-clock-parallel destination fan-out in exchange for per-destination durability,
  retryability, and an audited "which destinations were skipped" outcome. See Phase 12 for exactly
  how to apply both together.
- **Every `ctx->callActivity()` call in every generated project always passes a human-review
  `retryPolicy`** (a `ReviewTaskDefinition` — `{userRoles: ..., title: ...}`), never `AutoRetry` and
  never the default `NoAutomaticRetry`. On failure, the engine raises a review task for the named
  role instead of blind-retrying or immediately failing — a person decides whether to rerun the
  activity, rerun it with edited input, or fail it. This is a fixed project convention for this
  skill, applied uniformly regardless of the destination's criticality; see Phase 7 and Phase 12
  for exactly how it composes with the critical/non-critical (`check` vs. `T|error`) decision.
- **Every generated project defines exactly two error types**, `ConnectionError` and
  `ExecutionError` (see "Error Types — Required in Every Project" below), and every activity that
  can fail constructs one of these two instead of a bare `error(...)`, so failures are
  classifiable by a reviewer and by workflow logic.
- **Source connectors never surface a workflow error back through the inbound protocol.** If
  `workflow:run()`/`workflow:getWorkflowResult()` ends in error, the listener logs it and still
  acknowledges receipt (HTTP `202 Accepted`, an HL7 `AA` ack, a completed file-watcher callback) —
  it never returns that error from the `resource`/`remote` function. See Phase 3.

---

## What You Always Produce

```
<channel-name>/
├── Ballerina.toml
├── Config.toml              (configurable values — mode, hosts, ports; no hardcoded secrets)
├── types.bal                (record type definitions, including the workflow input/result types)
├── activities.bal            (@workflow:Activity functions — one per I/O step / destination)
├── workflow.bal               (the single @workflow:Workflow orchestration function)
├── service.bal               (listener wiring — starts workflow instances via workflow:run())
├── utils.bal                  (pure helper functions — filters, transforms; only if needed)
└── tests/
    ├── Config.toml            (forces mode = "IN_MEMORY" for the test run)
    ├── workflow_test.bal      (ballerina/test cases)
    └── resources/
        └── sample-input.*     (one representative sample message per test case)
```

Only include files that are actually needed. For a simple channel, `types.bal` + `activities.bal`
+ `workflow.bal` + `service.bal` + `tests/` is enough — skip `utils.bal` if there's nothing pure
worth separating out.

---

## Error Types — Required in Every Project

Every project from this skill defines exactly two `distinct error` types in `types.bal`, and no
others (plain `error("...")` for anything an activity can fail with is not acceptable):

```ballerina
# types.bal

// Raised when an activity cannot reach the external system at all — connection refused,
// DNS failure, timeout establishing the connection, TLS handshake failure, host unreachable.
public type ConnectionError distinct error;

// Raised when the external system was reached but the interaction itself failed or was
// rejected — a non-2xx HTTP response, an HL7 application-error (AE) NACK, a DB constraint
// violation, a validation failure on the payload, an unexpected/malformed response body.
public type ExecutionError distinct error;
```

**Usage rule:** every `@workflow:Activity` function that can fail constructs and returns one of
these two, never a bare `error(...)` — see the `sendToFhirServer` example in
`references/activity-examples.md` for the pattern.

Why this matters beyond naming: a reviewer looking at a raised human-review task (Phase 7/12) sees
the error type and its `detail()` fields as part of the task context, so `ConnectionError` vs.
`ExecutionError` is the first thing that tells them whether "retry once the network's back" or
"the payload itself needs a human decision" is the right call. Keep the distinction meaningful —
don't default everything to `ExecutionError` for convenience.

`ctx->callActivity()`'s inferred result type follows whatever the activity declares, so workflow
code binds to the union directly, e.g. `string|ConnectionError|ExecutionError result = ctx->callActivity(...)`,
or narrows with `result is ConnectionError` / `result is ExecutionError` when the two need
different handling before falling through to the shared skip/fail logic in Phase 12.

---

## Phase 1: Analyze the Channel XML

Parse the XML and identify the same elements you'd look for in any Mirth migration:

| XML element | What to look for |
|---|---|
| `<sourceConnector>` | Connector class → listener type (MLLP, HTTP, File, etc.) → this becomes the entry point that calls `workflow:run()` |
| `<destinationConnectors>` | Each destination's class, properties, and queue settings → each becomes one `@workflow:Activity`, critical or non-critical per Phase 12 |
| `<transformer>` | Step list: `<step>` elements with `<type>` and `<script>` |
| `<filter>` | Rule list: `<rule>` elements with `<type>` and `<script>` |
| `<responseTransformer>` | Post-send response handling — becomes inline code in the workflow function right after the corresponding `callActivity` call |
| `<properties>` | Port, host, encoding, transmission mode, data type |
| `<preprocessorScript>` | Runs before filtering — pure text mutation → plain function; if it does I/O, an activity |
| `<postprocessorScript>` | Runs after all destinations — since destination results are already local variables in the workflow function, this is almost always inline code, not a separate step |
| `<deployScript>` | One-time startup logic → module `init()` in `service.bal` |
| `<undeployScript>` | One-time teardown logic → graceful-stop / cleanup function |

See `references/connector-activity-mappings.md` for the full connector class → Ballerina
equivalent table and the dependency-by-connector-type table (this replaces the
`xlibb/pipeline`-based dependency table you may have seen in other Mirth-migration skills).

---

## Phase 2: Build the Ballerina.toml

```toml
[package]
org = "healthcare"
name = "<channel_name_snake_case>"
version = "1.0.0"
distribution = "2201.13.0"

[build-options]
observabilityIncluded = true
```

`ballerina/workflow` requires Ballerina **2201.13.0 or later** — do not emit an older
`distribution` value.

Add `[dependencies]` per connector type from `references/connector-activity-mappings.md`.
**Always include `ballerina/workflow`** — this is the one non-negotiable dependency for this
skill. Do **not** add `xlibb/pipeline`; there is no pipeline object in this design, so nothing in
the generated project should import it.

---

## Phase 3: Source Connector — Wire It to `workflow:run()`

The source connector's only job is to receive the inbound message and start a workflow instance.
It should not contain any business logic.

**Required convention for every source connector: never return an error from the
`resource`/`remote` function, regardless of what `workflow:run()` or
`workflow:getWorkflowResult()` does.** If starting the workflow fails, or the workflow itself
eventually fails, log it and still acknowledge receipt at the protocol level. The rationale: the
workflow is durable — a failure is recorded in its event history and (per the human-review policy
in Phase 7/12) has already raised a review task for a person to act on. Surfacing that same failure
a second time as a synchronous protocol error to the sending system gains nothing and, for most
protocols, just causes the sender to retry a delivery that was already durably received. So the
listener's job ends at "durably accepted," not "durably processed."

Because of this, the listener function signature itself should not include `|error` in its return
type — that's not just a style choice, it's what makes "never return an error" a compile-time
guarantee rather than a convention someone can accidentally violate.

See `references/source-connector-examples.md` for the full MLLP, HTTP, and File Reader listener
code — each one starts exactly one workflow instance per inbound message, then immediately
acknowledges, awaiting the eventual result off to the side via `start logWorkflowOutcome(...)`
rather than blocking the ack on it. All branching, transformation, and destination sends live in
`processChannelMessage` (Phase 4 onward); all failure escalation lives in the human-review policy
(Phase 7/12), not in the source connector's response.

---

## Phase 4: Model the Message Flow as the `@workflow:Workflow` Function

This is the structural heart of the translation. One Mirth channel → one workflow function. See
`references/workflow-and-script-examples.md` for the full `processChannelMessage` example; the
mapping table below is what drives the translation decisions.

| Mirth concept | `ballerina/workflow` equivalent |
|---|---|
| Source connector | Listener that calls `workflow:run(processChannelMessage, input)` (Phase 3) |
| Preprocessor Script (pure) | Plain function, called directly at the top of the workflow function |
| Preprocessor Script (does I/O) | `@workflow:Activity`, called first via `ctx->callActivity()` |
| Source filter rules | Plain `boolean`-returning function; workflow function does `if !passes { return ...; }` |
| Source transformer steps (pure) | Plain function returning the shaped record |
| Source transformer steps (external lookup) | `@workflow:Activity` |
| Side-effect steps (DB lookup, log) | `@workflow:Activity`, `check`'d if critical or captured as `T\|ConnectionError\|ExecutionError` if not — always with a human-review `retryPolicy` |
| Destination connector | `@workflow:Activity`, invoked sequentially — see Phase 12 |
| Multiple destinations | Sequential `ctx->callActivity()` calls, **not** parallel (see stance at top of this file) |
| Destination filter | Plain boolean check before the corresponding `callActivity` call |
| Response Transformer | Inline code immediately after that destination's `callActivity` call, using its return value |
| Postprocessor Script | Inline code near the end of the workflow function, working off local result variables |
| Any destination's retry/queue behavior | Not `AutoRetry` — a human-review `retryPolicy` (`ReviewTaskDefinition`), always. See Phase 7/12. |

**Why "pure vs. activity" matters for correctness, not just style:** `@workflow:Workflow` functions
must be **deterministic** — replay must reach the same decisions given the same recorded history.
Reading `time:utcNow()`, calling an external endpoint directly, or reading mutable module-level
state from inside the workflow function are all determinism violations; only some of them
(`worker`/`fork`/`start`, direct calls to `@workflow:Activity` functions) are caught at compile
time. Direct HTTP/DB/file calls inside workflow code are **not** caught by the compiler and will
silently misbehave on recovery — so when in doubt about whether a translated Mirth step needs to be
an activity, the test is: *does this step touch anything outside the current process?* If yes,
activity. If it's pure computation over values already in scope, plain function.

---

## Phase 5: Variable Maps → Ballerina

Because the whole channel is one workflow function invocation rather than a pipeline object
threading a `MessageContext` through separate processor calls, most of Mirth's variable maps
collapse into **ordinary local variables** — simpler than the `msgCtx.setProperty()` pattern used
in pipeline-based translations.

| Mirth map | JS variable | Scope | `ballerina/workflow` equivalent |
|---|---|---|---|
| **Connector Map** | `connectorMap` / `$co` | Current message, current connector only | Local variable inside the relevant activity function |
| **Channel Map** | `channelMap` / `$c` | Current message, shared across destinations | Local variable in the workflow function, passed as an argument to whichever activity needs it |
| **Source Map** | `sourceMap` / `$s` | Current message, read-only, injected by source | Fields on the `ChannelInput` record passed to `workflow:run()` |
| **Response Map** | `responseMap` / `$r` | Current message, destination responses | The return value of each `ctx->callActivity()` call, held in a local variable |
| **Global Channel Map** | `globalChannelMap` / `$gc` | All messages in this channel, in-memory only | See caveat below — must be read/written via an activity |
| **Global Map** | `globalMap` / `$g` | All messages, all channels, in-memory only | Same as above |
| **Configuration Map** | `configurationMap` / `$cfg` | Read-only server config | `configurable` Ballerina variables in `Config.toml` |

**`globalChannelMap` / `globalMap` caveat — different from a pipeline-based translation, and
stricter.** In a `@workflow:Workflow` function, reading or writing mutable module-level state
directly is a determinism violation the compiler does **not** catch (see Phase 4). So unlike a
pipeline translation where `msgCtx`-adjacent module state can be touched inline, here the get/set
must go through an `@workflow:Activity` so the engine records the interaction — see
`references/activity-examples.md` for the full pattern (a `lock{}`-guarded module-level map behind
`readGlobalChannelState`/`writeGlobalChannelState` activities), including the restart/scaling
caveats and why these two are an exception to the `ConnectionError`/`ExecutionError` typing rule.

**`sourceMap` variables injected by specific connectors:**

| Mirth source connector | Automatic sourceMap keys | Ballerina approach |
|---|---|---|
| File Reader | `originalFilename`, `fileDirectory`, `fileSize`, `fileLastModified` | Fields on `ChannelInput` (Phase 3) |
| HTTP Listener | `remoteAddress`, `localAddress`, HTTP headers as `http.*` | Extract from `http:Request` before calling `workflow:run()`, pass as `ChannelInput` fields |
| Database Reader | Column names from the query result | Fields on `ChannelInput` |
| Channel Writer (upstream) | Any variables injected by upstream channel | Document as a TODO — requires tracing the upstream channel |

---

## Phase 6: Channel-Level Scripts

| Mirth script | Ballerina equivalent |
|---|---|
| Deploy Script | Module `init()` in `service.bal` — runs once at process startup |
| Undeploy Script | A cleanup function invoked from a graceful-stop handler (no direct Ballerina hook) |
| Preprocessor Script (pure) | Plain function called at the top of the workflow function |
| Postprocessor Script | Inline code near the end of the workflow function, reading the local result variables already in scope |

See `references/workflow-and-script-examples.md` for the full code for all four. If the
postprocessor needs to send something back to the *source* connector (e.g. a custom HL7 ACK), that
logic lives back at the call site in `service.bal`, using `workflow:getWorkflowResult()`'s return
value — see the MLLP example in `references/source-connector-examples.md`.

---

## Phase 7: Writing Activities in `activities.bal`

- **Filters** are plain functions, not activities. A filter returning `false` maps to an early
  `return {status: "FILTERED"};` in the workflow function (Phase 4) — there is no separate "drop
  silently" mechanism to invoke.
- **Transformers** are plain functions unless they need external data, in which case the
  external-lookup part becomes an activity. HL7v2 field access always uses optional chaining —
  never assume a field exists.
- **Side-effect / generic-processor steps and destinations** are activities. Every activity that
  can fail returns `ConnectionError|ExecutionError` (or `T|ConnectionError|ExecutionError` for one
  that also returns a value on success) rather than a bare `error`, per "Error Types" above.

See `references/activity-examples.md` for the full code for each of these, plus the MLLP sender
activity (Phase 9).

**Annotation and signature rules that must hold, always:**

| Rule | Error if violated |
|---|---|
| Activity parameters must all be subtypes of `anydata` | `WORKFLOW_103` |
| Activity return type must be a subtype of `anydata` or `error` | `WORKFLOW_104` |
| `ctx->callActivity()` target must be an `@workflow:Activity`-annotated function | `WORKFLOW_107` |
| Never call an `@workflow:Activity` function directly — always through `ctx->callActivity()` | `WORKFLOW_108` |
| `callActivity`'s args map must supply every required parameter | `WORKFLOW_109` |
| `callActivity`'s args map must not include parameters the activity doesn't have | `WORKFLOW_110` |
| Activities with rest parameters are not supported by `callActivity` | `WORKFLOW_111` |
| A no-value (`error?`) activity result must still be bound to something (even `() _ =`) so the type can be inferred | compile error — see Phase 12 |
| `stepId` must be a constant string, not computed | `WORKFLOW_161` |
| Every `ctx->callActivity()` call passes a human-review `retryPolicy` (`{userRoles: ..., title: ...}`) | project convention, not a compiler rule — see below |

**Project convention — human-review `retryPolicy` on every call, no exceptions.** `retryPolicy`
here is a `ReviewTaskDefinition`, not an `AutoRetry` record. On failure the engine raises a review
task for the named role(s) instead of retrying automatically or failing outright; a person decides
whether to rerun the activity, rerun it with edited input, or fail it. Two fields matter most when
writing one of these for a generated activity:

| Field | What to put there |
|---|---|
| `userRoles` | The role that should see and act on this failure — carry over from context if the original Mirth channel had an operational owner/team, otherwise a sensible default such as `"OPS"` for infrastructure-facing steps and `"MANAGER"` for business-facing destination sends |
| `title` | Short, specific, and namable in an incident — "Failure in HL7 Sender", "Failure sending patient to FHIR server" — not a generic "Activity failed" |

This applies uniformly: critical and non-critical destinations, side-effect steps, and downstream
sends all use this same `retryPolicy` shape. What differs between critical and non-critical is
only whether the workflow function `check`s the result or captures it as `T|error` — see Phase 12.

---

## Phase 8: Destination Queuing → Human Review + Graceful Completion

Mirth has three queue modes per destination (Never, On Failure, Always). None of them map to a
separate pipeline/store object here, and — because every activity in this design always uses a
human-review `retryPolicy` (Phase 7) — they no longer select between `AutoRetry` and
`NoAutomaticRetry` either. Instead, the queue mode informs two things: the critical/non-critical
classification (Phase 12) and the `title`/urgency written into the `ReviewTaskDefinition`. See
`references/retry-and-skip-mapping.md` for the full mapping table, including how "Always"
(store-and-forward) maps onto the workflow engine's own durability guarantees rather than a
separate queue component.

---

## Phase 9: MLLP Sender (TCP Dispatcher) as an Activity

Same activity shape as any other destination (Phase 7), using `health.clients.hl7:HL7Client` —
which handles MLLP framing automatically, so do not wrap bytes manually. See
`references/activity-examples.md` for the full `sendToDownstream` activity and its
`ctx->callActivity()` call site.

---

## Phase 10: Translating JavaScript — Three Cases

Same tiering as any Mirth-to-Ballerina migration: Case A (simple, translate directly), Case B
(moderate, typed stub with a precise contract comment), Case C (complex/stateful, typed stub with
a full behavior/contract description, never a partial translation). The tiering criteria,
worked examples, and the Mirth-global-variable table are unchanged by the workflow-vs-pipeline
choice — see `references/js-translation-examples.md`, with one correction to its `channelMap`
row: in this design `channelMap` is a **local variable passed between workflow code and activity
arguments**, not `msgCtx.setProperty()`/`getPropertyWithType()`.

---

## Phase 11: Configuration (`Config.toml`)

Always externalize connection parameters — never hardcode hosts, ports, or credentials in `.bal`
files. In addition to the usual per-connector config, every project from this skill needs a
`[ballerina.workflow]` block:

```toml
mllpListenPort = 2575
destinationHost = "localhost"
destinationPort = 2576
fhirServerUrl = "https://fhir.example.com/fhir/r4"

[ballerina.workflow]
mode = "LOCAL"
url = "localhost:7233"
namespace = "default"
taskQueue = "<CHANNEL_NAME>_QUEUE"

[db]
host = "localhost"
port = 3306
user = "dbuser"
password = ""   # set via environment: BAL_CONFIG_SECRET_db_password
database = "mirthdb"
```

- `mode = "LOCAL"` for local dev against `temporal server start-dev`; `CLOUD` or `SELF_HOSTED` for
  real deployments (see `ballerina/workflow`'s own configuration guide for those fields).
- `taskQueue` must be unique per deployment — name it after the channel, e.g.
  `ORDER_ADMIT_CHANNEL_QUEUE`, so it doesn't collide with other channels' workers sharing the same
  Temporal cluster/namespace.
- For database `DbConfig` records, annotate the password field with `@sql:SensitiveConfig`.

---

## Phase 12: Error Handling — Required Pattern for This Skill

This is the pattern that must be applied to **every** channel this skill migrates: for each
destination connector, decide critical or non-critical, then translate all destinations as
**separate durable activities, called sequentially, every one carrying a human-review
`retryPolicy`**, applying **skip-on-failure to the non-critical ones** once a reviewer has had the
chance to act (or the review task itself times out / is resolved as "fail it").

### Step 1 — classify each destination

| Mirth signal | Classification |
|---|---|
| Queue mode "Never" and the channel has no fallback path if this destination fails | Critical |
| The channel's primary business outcome depends on this destination succeeding (e.g. the record store, the primary downstream system) | Critical |
| Side-channel effects — email notification, audit log, analytics/metrics export, a "nice to have" secondary copy | Non-critical |
| Queue mode "On Failure" with a bounded retry count, where the channel keeps going regardless of outcome | Non-critical |

Unlike a project where retry mechanism varies by classification, here classification only decides
`check` vs. `T|error` — the `retryPolicy` shape (`ReviewTaskDefinition`) is the same either way
(Phase 7).

### Step 2 — define the two error types once, in `types.bal`

See "Error Types — Required in Every Project" near the top of this file for the full usage rule.

### Step 3 — translate as sequential `ctx->callActivity()` calls, every one with a human-review `retryPolicy`

See `references/error-handling-example.md` for the full worked example (`processOrder`, with two
critical and two non-critical destinations) and the notes that make it correct rather than just
plausible-looking — in particular: every call needs an explicit constant `stepId`; non-critical
results must be bound (never discarded with `_`) so the `is error` branch can record the skip;
and resolving a raised review task is a separate concern handled through the module's human-task
completion surface, whose exact decision-payload shape should be confirmed against the live
`ballerina/workflow` reference rather than guessed at.

---

## Phase 13: Generating a Test Scenario

Every migrated project from this skill includes a runnable test that exercises the workflow
end-to-end without needing a real Temporal server, plus tests that specifically prove: the
skip-on-failure behavior for a non-critical destination, that a critical-destination failure
propagates and fails the workflow, and that the source connector always acknowledges even when the
underlying workflow fails. See `references/test-scenario-examples.md` for:

- `tests/Config.toml`, forcing `mode = "IN_MEMORY"` so tests run fast and in-process.
- The happy-path test, the skip-on-failure and critical-failure tests, and the
  always-acks-at-the-source test.

**Two things every generated test suite must get right, not just plausibly resemble:**

- Because every activity uses a human-review `retryPolicy` (Phase 7/12), a test that drives a
  destination to fail and then immediately calls `workflow:getWorkflowResult()` **will hang**
  unless it first resolves the review task the failure raised. Every such `// TODO: resolve the
  review task` in the reference examples is a real gap the generated test must close, not
  decoration — do not ship a test suite with it unresolved and call it passing.
- There is no documented activity-mocking facility in `ballerina/workflow` — drive failure through
  test-environment configuration (an address nothing listens on, an invalid input), not a
  fabricated mocking API, and keep it deterministic so the skip-on-failure guarantee it demonstrates
  is trustworthy.

---

## Output Format

After analyzing the channel XML, output **all files in sequence** using labeled code blocks:

```
### Ballerina.toml
### Config.toml
### types.bal
### activities.bal
### workflow.bal
### service.bal
### utils.bal              (only if needed)
### tests/Config.toml
### tests/resources/sample-input.json   (or .hl7 — match the channel's data type)
### tests/workflow_test.bal
```

After all files, include a **Migration Notes** section with these subsections:

1. **Stubs to implement** — each Case B/C JS stub and each `// TODO` activity body: its name, the
   original Mirth step it replaces, and what the developer must implement.
2. **Critical vs. non-critical destinations** — the classification made in Phase 12 for every
   destination, and why, so a reviewer can challenge it if the business judgment was wrong.
3. **Human-review roles used** — every distinct `userRoles` value assigned across the generated
   `retryPolicy`s (Phase 7/12), and which steps route to it, so the user can confirm those roles
   actually exist in their Temporal/organizational setup before deploying — a review task raised
   for a role nobody is watching is a silent stall, not a safety net.
4. **Assumptions & Gaps** — only list items where essential behavior is truly missing from the XML
   and a safe configurable default was used; keep this list short and precise.
5. **Configuration** — values the user must fill in (hosts, ports, credentials, `taskQueue` name,
   Temporal `mode`/`url`/`namespace`, human-review `userRoles` assignments, etc.).
6. **Mirth behaviors without a direct equivalent in this design** — parallel destination fan-out
   (see stance at the top of this file), persistent channel maps, attachment handling, Channel
   Writer cross-channel routing, synchronous source-connector error responses (Phase 3 always acks
   instead) — with the suggested workaround.
7. **How to run** —
   - Local dev: `temporal server start-dev`, then `bal run` (uses `mode = "LOCAL"` from the root
     `Config.toml`).
   - Tests: `bal test` (uses `mode = "IN_MEMORY"` from `tests/Config.toml`, no server needed).
