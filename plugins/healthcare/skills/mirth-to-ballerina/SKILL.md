---
name: mirth-to-ballerina
description: Migrates a Mirth Connect channel XML to a compilable Ballerina project built on ballerina/workflow (Temporal-backed durable orchestration), not xlibb/pipeline. Each destination connector becomes a separate, individually durable @workflow:Activity invoked via ctx->callActivity, always with a human-review retry policy; non-critical destinations are skipped on failure instead of failing the whole run. Every project uses two standard error types, ConnectionError and ExecutionError, and source connectors always acknowledge receipt and log failures rather than surfacing errors to the caller. Also generates mock services for every source and destination connector plus a scenario suite derived from the original channel, a matching test scenario using ballerina/test and IN_MEMORY mode, and an end-to-end verification run against those mocks that produces a report on accuracy, latency and token usage with the tested scenarios listed. Use whenever the user shares a Mirth channel.xml and wants a workflow-based (Temporal-backed) Ballerina migration, wants destinations modeled as durable, retryable, skippable, human-reviewable activities instead of a pipeline list, needs a generated project plus a runnable test for a Mirth migration, or wants the migration verified against mock connectors with a metrics report.
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
  strands in its durable event history. So destinations become **separate, individually durable
  activities called sequentially**, each marked critical (`check` — failure fails the whole
  workflow) or non-critical (captured as `T|error` and skipped — "graceful completion"). A
  deliberate trade-off: wall-clock fan-out is given up for per-destination durability, retryability
  and an audited record of what was skipped. See Phase 12.
- **Every `ctx->callActivity()` call in every generated project always passes a human-review
  `retryPolicy`** (a `ReviewTaskDefinition` — `{userRoles: ..., title: ...}`), never `AutoRetry` and
  never the default `NoAutomaticRetry`. On failure the engine raises a review task for the named
  role instead of blind-retrying or failing outright, and a person decides whether to rerun, rerun
  with edited input, or fail it. A fixed convention, applied uniformly regardless of the
  destination's criticality; see Phase 7 and Phase 12 for how it composes with the critical /
  non-critical (`check` vs. `T|error`) decision.
- **Every generated project defines exactly two error types**, `ConnectionError` and
  `ExecutionError` (see "Error Types — Required in Every Project" below), and every activity that
  can fail constructs one of these two instead of a bare `error(...)`, so failures are
  classifiable by a reviewer and by workflow logic.
- **Source connectors never surface a workflow error back through the inbound protocol.** If
  `workflow:run()`/`workflow:getWorkflowResult()` ends in error, the listener logs it and still
  acknowledges receipt (HTTP `202 Accepted`, an HL7 `AA` ack, a completed file-watcher callback) —
  it never returns that error from the `resource`/`remote` function. See Phase 3.
- **HL7v2 and FHIR data stay in the library's own types — never repackaged into a hand-rolled
  record that duplicates fields under different names.** A parsed message is an `hl7v23:ADT_A01`
  (or the matching version's message type), read directly (`adt.pid?.pid5?[0]?.xpn1`), not
  flattened into a custom "patient" record. FHIR-bound data goes through the matching
  `health.hl7v2<ver>.utils.v2tofhirr4` mapping functions into the FHIR library's own resource/Bundle
  shape, sent via `health.clients.fhir:FHIRConnector`, never a custom record posted through a raw
  `http:Client`. A custom type is still fine with no library equivalent (`ChannelInput`/`ChannelResult`)
  — the rule is not reinventing what the library already models. See Phase 3, Phase 7, and Phase 9.

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
├── tests/
│   ├── Config.toml            (forces mode = "IN_MEMORY" for the test run)
│   ├── workflow_test.bal      (ballerina/test cases)
│   └── resources/
│       └── sample-input.*     (one representative sample message per test case)
└── verification/              (Phase 1B; run in Phase 13 — never shipped with the integration)
    ├── Config.verification.toml  (overrides only — repoints the project at the mocks)
    ├── mocks/                 (a SEPARATE Ballerina package — its deps never touch the project's)
    ├── scenarios/             (scenarios.json + one input message per scenario)
    └── report/                (migration-verification-report.md)
```

Only include files that are actually needed — for a simple channel, `types.bal` + `activities.bal`
+ `workflow.bal` + `service.bal` + `tests/` is enough, and `utils.bal` can be skipped if there is
nothing pure worth separating out. `verification/` is not optional in the same way: every migration
produces mocks, scenarios and a report, because that is what turns "it compiles" into "it behaves
like the channel it replaced."

---

## Error Types — Required in Every Project

Every project defines exactly two `distinct error` types in `types.bal` and no others; a bare
`error("...")` from anything an activity can fail with is not acceptable:

- **`ConnectionError`** — the external system could not be reached at all: connection refused, DNS
  failure, connect timeout, TLS handshake failure, host unreachable.
- **`ExecutionError`** — the system was reached and the interaction failed or was rejected: a
  non-2xx response, an HL7 `AE` NACK, a DB constraint violation, a payload validation failure, a
  malformed response body.

Every `@workflow:Activity` that can fail returns one of these two. The distinction is what tells a
reviewer, from the raised review task alone, whether "retry once the network is back" or "the
payload itself needs a human decision" is the right call — so keep it meaningful rather than
defaulting everything to `ExecutionError` for convenience.

See `references/error-handling-example.md` for the declarations, the usage pattern, and how
workflow code binds or narrows the returned union.

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

## Phase 1B: Generate Mock Connectors and the Verification Scenario

Run this immediately after Phase 1 and **before generating any of the migrated project's own
code**. The ordering is the point: expected outputs must be derived from the *original Mirth
channel*, not from the Ballerina you are about to write. A scenario authored after the code only
proves the generated project agrees with itself, which is not a migration check at all.

**Scope — this harness verifies functionality, not failure handling.** Every mock always succeeds.
No scenario injects a connection failure, a rejected send, or any other error, and none asserts on
`ConnectionError`/`ExecutionError`, `skippedSteps`, or a raised review task. The question it
answers is narrow and worth answering on its own: *given a message the original channel would have
handled, does the migrated integration route it to the right destinations, in the right order,
carrying the right fields?* Failure behavior — skip-on-failure for non-critical destinations,
propagation for critical ones, the source's always-acknowledge convention — is proven by the
in-process test suite in Phase 13.1 and stays a hard requirement there. Keeping the two separate is
what keeps each readable: a failed functional scenario means the translation is wrong, full stop.

**It relaxes nothing else either.** Mocks are reached purely through configuration overrides
(Phase 11); if one can only be reached by editing a `.bal` file, that is a hardcoded endpoint to
fix in the generated project, never to work around here.

What this phase produces: a **mock destination service** per `<destinationConnectors>` entry
(speaking the real protocol, always accepting, recording every receipt in arrival order); a
**source driver** per source connector, which injects into the **real** listener, since the
generated source connector is never mocked — its framing, parsing and config wiring are part of
what is verified; **`scenarios/scenarios.json`**, holding field-level expected outputs derived
from the XML plus an `unverifiable` list for anything gated behind a Case B/C stub (Phase 10); and
**`Config.verification.toml`**, overrides only — mock ports plus `[ballerina.workflow] mode =
"IN_MEMORY"`, so the run needs no Temporal server.

Because nothing is injected, coverage comes entirely from inputs: every message type, both sides of
every filter, each conditional path in the transformer, and boundary data. One useful consequence —
since no activity fails, no review task is ever raised and the run never pauses, so the harness does
not depend on the review-task decision payload whose shape is still unconfirmed (Phase 12).

See `references/mock-service-examples.md` for the mock, driver and recorder code per connector
type, the full `scenarios.json` schema, and the mandatory coverage table.

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
`distribution` value. Add `[dependencies]` per connector type from
`references/connector-activity-mappings.md`. **Always include `ballerina/workflow`** — the one
non-negotiable dependency for this skill. Do **not** add `xlibb/pipeline`; there is no pipeline
object in this design, so nothing in the generated project should import it.

---

## Phase 3: Source Connector — Wire It to `workflow:run()`

The source connector's only job is to receive the inbound message and start a workflow instance.
It should not contain any business logic.

**Required convention for every source connector: never return an error from the
`resource`/`remote` function, regardless of what `workflow:run()` or
`workflow:getWorkflowResult()` does.** If starting the workflow fails, or the workflow itself
eventually fails, log it and still acknowledge receipt at the protocol level. The workflow is
durable — the failure is already in its event history and has already raised a review task (Phase
7/12) for a person to act on. Surfacing it a second time as a synchronous protocol error gains
nothing and, for most protocols, just makes the sender retry a delivery that was already durably
received. The listener's job ends at "durably accepted," not "durably processed." So the listener
function's signature must not include `|error` in its return type — that is what turns the
convention into a compile-time guarantee rather than something someone can accidentally violate.

See `references/source-connector-examples.md` for the full MLLP, HTTP, and File Reader listener
code — each starts exactly one workflow instance per inbound message, then acknowledges
immediately, awaiting the eventual result off to the side via `start logWorkflowOutcome(...)`
rather than blocking the ack on it.

---

## Phase 4: Model the Message Flow as the `@workflow:Workflow` Function

This is the structural heart of the translation. One Mirth channel → one workflow function. See
`references/workflow-and-script-examples.md` for the full `processChannelMessage` example and the
mapping table of every Mirth concept to its `ballerina/workflow` equivalent — consult that table
while translating; the decision rule below is what it encodes.

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
in pipeline-based translations. In short: connector map → a local inside the relevant activity;
channel map → a local in the workflow function, passed as an argument to whichever activity needs
it; source map → fields on `ChannelInput`; response map → the return value of each
`ctx->callActivity()`; configuration map → `configurable` variables in `Config.toml`.

**`globalChannelMap` / `globalMap` caveat — different from a pipeline-based translation, and
stricter.** In a `@workflow:Workflow` function, reading or writing mutable module-level state
directly is a determinism violation the compiler does **not** catch (see Phase 4). So unlike a
pipeline translation where `msgCtx`-adjacent module state can be touched inline, here the get/set
must go through an `@workflow:Activity` so the engine records the interaction.

See `references/workflow-and-script-examples.md` for the full map-by-map table and the
per-connector `sourceMap` key table, and `references/activity-examples.md` for the `lock{}`-guarded
global-state activity pattern with its restart/scaling caveats.

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
activity (Phase 9), the table of `WORKFLOW_1xx` signature rules that must hold for every activity
and `callActivity` call, and field-by-field guidance for writing a `ReviewTaskDefinition`.

**Project convention — a human-review `retryPolicy` on every call, no exceptions.** `retryPolicy`
here is a `ReviewTaskDefinition` (`{userRoles: ..., title: ...}`), not an `AutoRetry` record and
never the default `NoAutomaticRetry`. On failure the engine raises a review task for the named
role(s) instead of retrying automatically or failing outright; a person decides whether to rerun
the activity, rerun it with edited input, or fail it. `userRoles` names the step's operational
owner — `"OPS"` for infrastructure-facing steps, `"MANAGER"` for business-facing sends, or an owner
carried over from the Mirth channel. `title` must be specific enough to name in an incident
("Failure in HL7 Sender"), never a generic "Activity failed".

This applies uniformly: critical and non-critical destinations, side-effect steps, and downstream
sends all use the same `retryPolicy` shape. What differs between critical and non-critical is only
whether the workflow function `check`s the result or captures it as `T|error` — see Phase 12.

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
its own implementation writes encoded bytes straight to the TCP socket with no visible MLLP
envelope wrapping, so confirm with the target server whether framing needs to be added explicitly
rather than assuming it's handled for you. See `references/activity-examples.md` for the full
`sendToDownstream` activity and its `ctx->callActivity()` call site.

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
files. In addition to the usual per-connector config (listener ports, destination hosts, service
URLs, a `[db]` block), every project from this skill needs a `[ballerina.workflow]` block:

```toml
[ballerina.workflow]
mode = "LOCAL"
url = "localhost:7233"
namespace = "default"
taskQueue = "<CHANNEL_NAME>_QUEUE"
```

`mode = "LOCAL"` targets `temporal server start-dev`; `CLOUD` or `SELF_HOSTED` for real deployments
(see `ballerina/workflow`'s own configuration guide for those fields). `taskQueue` must be unique
per deployment — name it after the channel, e.g. `ORDER_ADMIT_CHANNEL_QUEUE`, so it cannot collide
with other channels' workers sharing a Temporal cluster/namespace. Secrets never go in the file
(set them via `BAL_CONFIG_SECRET_*`), and a `DbConfig` password field is annotated
`@sql:SensitiveConfig`. Phase 1B's `Config.verification.toml` overrides these same keys — nothing
else — to point the project at the mocks.

---

## Phase 12: Error Handling — Required Pattern for This Skill

This is the pattern that must be applied to **every** channel this skill migrates: for each
destination connector, decide critical or non-critical, then translate all destinations as
**separate durable activities, called sequentially, every one carrying a human-review
`retryPolicy`**, applying **skip-on-failure to the non-critical ones** once a reviewer has had the
chance to act (or the review task itself times out / is resolved as "fail it").

### Step 1 — classify each destination

Critical when the channel's primary business outcome depends on it succeeding (the record store,
the primary downstream system), or its queue mode is "Never" with no fallback path. Non-critical
when it is a side-channel effect — notification, audit log, analytics export, a secondary copy —
or its queue mode is "On Failure" and the channel keeps going regardless. Unlike a project where
the retry mechanism varies by classification, here classification only decides `check` vs.
`T|error`; the `retryPolicy` shape (`ReviewTaskDefinition`) is the same either way (Phase 7). See
`references/retry-and-skip-mapping.md` for the full signal table.

### Step 2 — translate as sequential `ctx->callActivity()` calls, every one with a human-review `retryPolicy`

See `references/error-handling-example.md` for the full worked example (`processOrder`, with two
critical and two non-critical destinations) and the notes that make it correct rather than just
plausible-looking — in particular: every call needs an explicit constant `stepId`; non-critical
results must be bound (never discarded with `_`) so the `is error` branch can record the skip;
and resolving a raised review task is a separate concern handled through the module's human-task
completion surface, whose exact decision-payload shape should be confirmed against the live
`ballerina/workflow` reference rather than guessed at.

---

## Phase 13: Verify Against the Mocks, and Report

This phase has two halves and **both are required**. 13.1 is the in-process `bal test` suite that
ships with the project, and it is where **all failure behavior** is verified — skip-on-failure,
critical-failure propagation, and the source's always-acknowledge convention. 13.2–13.3 stand the
whole integration up against the Phase 1B mocks and verify **functional correctness on the success
path only**: real listeners, real clients, real config, every mock accepting. Neither replaces the
other — a red test suite means the logic mishandles failure; a red verification run means the
translation itself is wrong.

### 13.1 The in-process test suite

Every migrated project includes a runnable test that exercises the workflow end-to-end without a
real Temporal server, plus tests that specifically prove: skip-on-failure for a non-critical
destination, that a critical-destination failure propagates and fails the workflow, and that the
source connector always acknowledges even when the underlying workflow fails.
`references/test-scenario-examples.md` has `tests/Config.toml` (forcing `mode = "IN_MEMORY"`) and
all four tests, plus two things the generated suite must get right rather than merely resemble: a
test that drives a destination to fail and then calls `workflow:getWorkflowResult()` **will hang**
unless it first resolves the review task that failure raised (every `// TODO: resolve the review
task` there is a real gap to close), and failure must be driven through test configuration — an
address nothing listens on, an invalid input — since `ballerina/workflow` has no documented
activity-mocking facility to fabricate.

### 13.2 Run the generated project against the mocks

Build both packages, start the mocks and poll their ports for readiness, start the project with
`BAL_CONFIG_FILES=verification/Config.verification.toml bal run`, then drive each scenario through
its source driver in a declared order, resetting the recorder between scenarios. Configuration
only: if a `.bal` file has to be edited to reach a mock, that is a hardcoded endpoint to fix in the
project (Phase 11) and record as a finding. Every mock behaves identically in every scenario, so
only the input varies and no review task is ever raised to resolve — a scenario that hangs or ends
in error is a finding about the generated project, not a path being exercised. Never report metrics
for a project that did not build. `references/mock-service-examples.md` has the full run procedure.

### 13.3 The verification report

Write `verification/report/migration-verification-report.md` covering **accuracy** (assertions
matched over total *verifiable* assertions — `unverifiable` entries excluded and listed separately
— reported overall, per scenario and per destination), **latency** (per scenario and per activity,
with the `IN_MEMORY`/local-mock caveat stated so nobody reads it as an SLA), **token usage** (the
migration agent's own consumption by stage; never estimated — "not available in this environment"
when the session does not expose it), the **tested scenarios** table, and **findings** classified
by where the fix belongs. State the scope plainly: every scenario is a success path, and failure
handling is covered by 13.1, or a high figure will be over-read. Re-run after repairs and report
the delta across iterations.

`references/verification-report-template.md` has the full skeleton and the metric definitions —
read it before computing any number, since a figure with the wrong denominator is worse than none.

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
### verification/Config.verification.toml
### verification/mocks/   (Ballerina.toml, Config.toml, mock_destinations.bal, source_drivers.bal, recorder.bal)
### verification/scenarios/   (scenarios.json + inputs/<scenario-id>.* — one per scenario)
### verification/report/migration-verification-report.md
```

The `verification/*` files are authored in Phase 1B — before the project's own code — even though
they are emitted last here; the report is the one file filled in at the end, from the Phase 13 run.

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
7. **Verification summary** — headline accuracy, latency and token-usage figures from the Phase 13
   run, the scenario pass/fail count, and a pointer to
   `verification/report/migration-verification-report.md`. Name anything the run could not verify
   (Phase 1B `unverifiable` entries) here too, so nobody reads a high accuracy figure as broader
   coverage than it was.
8. **How to run** — local dev: `temporal server start-dev`, then `bal run` (`mode = "LOCAL"` from
   the root `Config.toml`). Tests: `bal test` (`mode = "IN_MEMORY"` from `tests/Config.toml`, no
   server needed). Verification: `bal run verification/mocks`, then
   `BAL_CONFIG_FILES=verification/Config.verification.toml bal run` in the project, then drive the
   scenarios (Phase 13.2) — no Temporal server needed, the override sets `IN_MEMORY` mode.
