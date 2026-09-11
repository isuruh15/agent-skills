---
name: mirth-to-ballerina
description: Migrates a Mirth Connect channel XML to a compilable Ballerina project built on ballerina/workflow (Temporal-backed durable workflow orchestration) instead of xlibb/pipeline. Each Mirth destination connector becomes a separate, individually durable @workflow:Activity invoked sequentially via ctx->callActivity, always with a human-review retry policy, with non-critical destinations skipped on failure (graceful completion) rather than failing the whole run. Every project uses two standard error types, ConnectionError and ExecutionError, and source connectors never surface a workflow error to the caller — they always acknowledge receipt and log failures. Also generates a matching test scenario using ballerina/test and the module's IN_MEMORY mode. Use this skill whenever the user - Shares a Mirth Connect channel.xml file or snippet and wants a Ballerina project built on the durable workflow library rather than a pipeline library - Asks for a workflow-based (Temporal-backed) Ballerina migration of a Mirth channel - Wants destination sends modeled as durable, individually-retryable, skippable, human-reviewable activities instead of a pipeline destination list - Needs a generated Ballerina project plus a runnable test for a Mirth migration
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
these two, never a bare `error(...)`:

```ballerina
@workflow:Activity
isolated function sendToFhirServer(PatientRecord patient) returns string|ConnectionError|ExecutionError {
    http:Client|error fhirClient = new (fhirServerUrl);
    if fhirClient is error {
        return error ConnectionError("Could not connect to FHIR server", fhirClient, url = fhirServerUrl);
    }
    json|error response = fhirClient->/Patient.post(patient);
    if response is error {
        return error ExecutionError("FHIR server rejected the request", response, patientId = patient.patientId);
    }
    string|error id = response.id;
    if id is error {
        return error ExecutionError("FHIR response missing id field", id);
    }
    return id;
}
```

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

### MLLP / HL7v2 listener

Same `Hl7Listener`/`Hl7Service` API as any HL7v2 Ballerina integration — the framing and parsing
are handled for you. Start the workflow, then immediately return (accepting the message);
separately await the result **without blocking the ack** so a failure gets logged rather than
silently dropped:

```ballerina
import ballerina/log;
import ballerina/workflow;
import ballerinax/health.hl7v2;
import ballerinax/health.hl7v23;

configurable int mllpPort = 2575;

listener hl7v2:Hl7Listener mllpListener = new (mllpPort);

service hl7v2:Hl7Service on mllpListener {
    // No `|error` in the return type — see the convention above. A failure to even start the
    // workflow is caught and logged here, never returned to the caller.
    isolated remote function onMessage(hl7v2:Hl7Client caller, hl7v2:Message message) returns error? {
        // The workflow input must be `anydata` — wrap the parsed message plus any sourceMap-equivalent
        // metadata into a single input record (see ChannelInput in types.bal).
        ChannelInput input = {rawMessage: message.toBaOm().toJson()};

        string|error workflowId = workflow:run(processChannelMessage, input);
        if workflowId is error {
            log:printError("Failed to start workflow for inbound HL7 message", 'error = workflowId);
            return; // still ack — see convention above; ballerina's default MLLP ack is sent here
        }
        log:printInfo("Started workflow", workflowId = workflowId);

        // This is ordinary service-level code, NOT inside a @workflow:Workflow function, so
        // `start` is perfectly fine here (unlike inside the workflow function itself — see
        // Phase 4). Await the result off to the side so the ack above isn't held up by it.
        _ = start logWorkflowOutcome(workflowId);
        // Function returns normally here — the MLLP ack is sent regardless of eventual
        // workflow outcome. A failed workflow has already raised a review task (Phase 7/12);
        // it does not need a second, synchronous signal back to the sender.
    }
}

// Never called from inside workflow code — this observes a workflow from the outside.
isolated function logWorkflowOutcome(string workflowId) {
    anydata|error result = workflow:getWorkflowResult(workflowId);
    if result is error {
        log:printError("Workflow ended in error", 'error = result, workflowId = workflowId);
    }
}
```

> `hl7v2:Message` is not itself `anydata` in a form `workflow:run()` can accept directly — the
> workflow input parameter must be a subtype of `anydata` (`WORKFLOW_101`). Convert or wrap it into
> a plain record/JSON-friendly type in `types.bal` before calling `workflow:run()`. Re-parsing
> inside the workflow (via an early pure helper, or the first activity if parsing needs external
> terminology lookups) is the usual pattern — do **not** try to smuggle a non-`anydata` object
> through as workflow input.

### HTTP source connector

Respond `202 Accepted` immediately after successfully starting the workflow; never return
`http:Accepted|error` — resolve any startup failure to a logged error and a `500`/`202` you choose
deliberately, not an unhandled `check` that lets the error escape as-is:

```ballerina
import ballerina/http;
import ballerina/log;
import ballerina/workflow;

service /api/v1 on new http:Listener(httpPort) {
    resource function post messages(http:Request request) returns http:Accepted {
        json|error payload = request.getJsonPayload();
        if payload is error {
            log:printError("Could not read inbound payload", 'error = payload);
            return http:ACCEPTED; // malformed input is still acknowledged, never surfaced as an error
        }

        string|error workflowId = workflow:run(processChannelMessage, {rawMessage: payload});
        if workflowId is error {
            log:printError("Failed to start workflow", 'error = workflowId);
            return http:ACCEPTED;
        }

        _ = start logWorkflowOutcome(workflowId);
        return http:ACCEPTED;
    }
}
```

### File Reader source connector

```ballerina
import ballerina/file;
import ballerina/io;
import ballerina/log;
import ballerina/workflow;

service "fileWatcher" on new file:Listener({path: watchDirectory, recursive: false}) {
    // No `|error` in the return type, for the same reason as the MLLP and HTTP listeners above.
    remote function onModify(file:FileEvent event) returns error? {
        string|error content = io:fileReadString(event.name);
        if content is error {
            log:printError("Could not read file", 'error = content, fileName = event.name);
            return;
        }
        // sourceMap equivalent (originalFilename etc.) travels in as input fields, not as
        // separately-set properties — see Phase 5.
        ChannelInput input = {rawMessage: content, originalFilename: event.name};
        string|error workflowId = workflow:run(processChannelMessage, input);
        if workflowId is error {
            log:printError("Failed to start workflow for file", 'error = workflowId, fileName = event.name);
            return;
        }
        _ = start logWorkflowOutcome(workflowId);
    }
}
```

In every case: **the listener starts exactly one workflow instance per inbound message, does no
orchestration itself, and never lets a workflow-side failure become a protocol-level error.** All
branching, transformation, and destination sends live in `processChannelMessage` (Phase 4
onward); all failure escalation lives in the human-review policy (Phase 7/12), not in the source
connector's response.

---

## Phase 4: Model the Message Flow as the `@workflow:Workflow` Function

This is the structural heart of the translation. One Mirth channel → one workflow function, with
this shape:

```ballerina
import ballerina/workflow;

@workflow:Workflow
function processChannelMessage(workflow:Context ctx, ChannelInput input) returns ChannelResult|error {
    // 1. Preprocessor equivalent — pure mutation, plain function call, no callActivity needed.
    string normalized = normalizeRawMessage(input.rawMessage);

    // 2. Source filter equivalent — plain boolean check, early return drops the message
    //    (silent drop, matching Mirth's FilterConfig returning false).
    if !passesSourceFilter(normalized) {
        return {status: "FILTERED"};
    }

    // 3. Source transformer equivalent — pure data shaping, plain function call.
    PatientRecord patient = check extractPatient(normalized);

    // 4. Side-effect / lookup step with external I/O — this DOES need an activity. Every
    //    callActivity call always carries a human-review retryPolicy (Phase 7) — never
    //    AutoRetry, never the default NoAutomaticRetry.
    boolean patientExists = check ctx->callActivity(lookupPatientInDb, {"patientId": patient.patientId},
            stepId = "lookup_patient_in_db",
            retryPolicy = {userRoles: "OPS", title: "Patient lookup failed"});

    // 5. Destinations — separate durable activities, called sequentially, each critical or
    //    non-critical, each with a human-review retryPolicy. See Phase 12 for the full pattern
    //    with skip-on-failure. Activities return ConnectionError|ExecutionError on failure
    //    (Phase "Error Types"), never a bare error.
    string reservationId = check ctx->callActivity(sendToFhirServer, {"patient": patient},
            stepId = "send_to_fhir",
            retryPolicy = {userRoles: "MANAGER", title: "Failure sending patient to FHIR server"});

    string|ConnectionError|ExecutionError auditResult = ctx->callActivity(writeAuditLog,
            {"patientId": patient.patientId}, stepId = "write_audit_log",
            retryPolicy = {userRoles: "OPS", title: "Failure writing audit log"});

    // 6. Postprocessor equivalent — results are already local variables, just assemble the outcome.
    string[] skipped = [];
    if auditResult is error {
        skipped.push("write_audit_log");
    }
    return {status: "COMPLETED", reservationId, skippedSteps: skipped};
}
```

### Mapping table

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

### Why "pure vs. activity" matters for correctness, not just style

`@workflow:Workflow` functions must be **deterministic** — replay must reach the same decisions
given the same recorded history. Reading `time:utcNow()`, calling an external endpoint directly,
or reading mutable module-level state from inside the workflow function are all determinism
violations; only some of them (`worker`/`fork`/`start`, direct calls to `@workflow:Activity`
functions) are caught at compile time. Direct HTTP/DB/file calls inside workflow code are **not**
caught by the compiler and will silently misbehave on recovery — so when in doubt about whether a
translated Mirth step needs to be an activity, the test is: *does this step touch anything outside
the current process?* If yes, activity. If it's pure computation over values already in scope,
plain function.

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
must go through an `@workflow:Activity` so the engine records the interaction:

```ballerina
// TODO: globalChannelMap translated to a module-level isolated map, accessed ONLY via activities
// (never read/written directly inside @workflow:Workflow code — that's an undetected determinism
// violation in this library). Caveats vs Mirth:
// 1. Reset on process restart — values are not persisted across deployments.
// 2. Not shared across multiple worker instances — use Redis or a DB table if horizontal
//    scaling or restart-survival of this state is required.
// 3. Mirth's globalChannelMap is a ConcurrentHashMap and does not allow null values —
//    maintain this invariant by never storing () here.
isolated map<anydata> globalChannelState = {};

@workflow:Activity
isolated function readGlobalChannelState(string key) returns anydata|error {
    lock {
        return globalChannelState[key];
    }
}

@workflow:Activity
isolated function writeGlobalChannelState(string key, anydata value) returns error? {
    lock {
        globalChannelState[key] = value;
    }
}
```

These two are an exception to the `ConnectionError`/`ExecutionError` typing convention (Error
Types section, above) — a `lock{}`-guarded in-memory map read/write doesn't fail in a way that's
meaningfully "connection" or "execution," so plain `anydata|error`/`error?` is fine here. They
still carry a human-review `retryPolicy` when called via `ctx->callActivity()`, per the project
convention in Phase 7 — even an in-memory operation gets the same treatment, since the convention
is "every activity," not "every activity that talks to a network."

**`sourceMap` variables injected by specific connectors:**

| Mirth source connector | Automatic sourceMap keys | Ballerina approach |
|---|---|---|
| File Reader | `originalFilename`, `fileDirectory`, `fileSize`, `fileLastModified` | Fields on `ChannelInput` (Phase 3) |
| HTTP Listener | `remoteAddress`, `localAddress`, HTTP headers as `http.*` | Extract from `http:Request` before calling `workflow:run()`, pass as `ChannelInput` fields |
| Database Reader | Column names from the query result | Fields on `ChannelInput` |
| Channel Writer (upstream) | Any variables injected by upstream channel | Document as a TODO — requires tracing the upstream channel |

---

## Phase 6: Channel-Level Scripts

### Deploy Script → module `init()`

Same as any Ballerina service — runs once at process startup, independent of any single workflow
instance:

```ballerina
// In service.bal — module init() runs once at process startup, equivalent to Mirth's Deploy Script.
// Original Deploy Script: loaded patient lookup cache from DB into globalChannelMap.
function init() returns error? {
    // TODO: implement — see original Deploy Script in <deployScript> element
    // Original used: DatabaseConnectionFactory.getConnection('mpi_db') to pre-load cache
    log:printInfo("Channel initialized");
}
```

### Undeploy Script → graceful stop / cleanup function

```ballerina
// Ballerina has no direct undeploy hook. Implement cleanup in a function invoked from a graceful
// stop handler, or at explicit program termination.
// TODO: implement cleanup — see original Undeploy Script in <undeployScript> element
isolated function cleanupResources() returns error? {
    // e.g. close sql:Client connections, flush caches
}
```

### Preprocessor Script → plain function, or first activity if it does I/O

```ballerina
// Translates Mirth Preprocessor Script — mutates raw message string before filtering.
// Original: stripped BOM characters and normalized line endings before HL7 parsing.
// Pure string manipulation — no callActivity needed, called directly from the workflow function.
isolated function normalizeRawMessage(json rawMessage) returns string {
    // TODO: implement — see original <preprocessorScript> element
    // Original used: message.replace('\uFEFF', '').replace('\r\n', '\r')
    return rawMessage.toString(); // placeholder
}
```

### Postprocessor Script → inline code near the end of the workflow function

Because destination results are already local variables (Phase 4/5), there is no separate
`responseMap` lookup step — just read the variables you already have:

```ballerina
@workflow:Workflow
function processChannelMessage(workflow:Context ctx, ChannelInput input) returns ChannelResult|error {
    // ... filters, transforms, destinations as above ...

    // Translates Mirth Postprocessor Script — assembles final response to originating system.
    // Original: checked ACK code from HL7 destination response; re-queued on AE, sent AA on success.
    // TODO: implement — see original <postprocessorScript> element
    // responseMap.get('dest1')  →  the local variable holding dest1's callActivity result, above
    return {status: "COMPLETED", reservationId, skippedSteps: skipped};
}
```

If the postprocessor needs to send something back to the *source* connector (e.g. a custom HL7
ACK), that logic lives back at the call site in `service.bal`, using
`workflow:getWorkflowResult()`'s return value — see the MLLP example in Phase 3.

---

## Phase 7: Writing Activities in `activities.bal`

### Filters — plain functions, not activities

```ballerina
isolated function passesSourceFilter(string normalized) returns boolean {
    // TODO: implement — see original <filter><rule> elements
    return true;
}
```

A filter returning `false` maps to an early `return {status: "FILTERED"};` in the workflow
function (Phase 4) — there is no separate "drop silently" mechanism to invoke.

### Transformers — plain functions unless they need external data

```ballerina
isolated function extractPatient(string normalized) returns PatientRecord|error {
    hl7v23:ADT_A01 adt = check parseAdt(normalized);
    return {
        patientId: adt.pid?.pid3?[0]?.cx1 ?: "",
        lastName:  adt.pid?.pid5?[0]?.xpn1 ?: "",
        firstName: adt.pid?.pid5?[0]?.xpn2 ?: "",
        dob:       adt.pid?.pid7?.ts1 ?: ""
    };
}
```

**HL7v2 field access — always use optional chaining, never assume fields exist:**

```
// Mirth: msg['PID']['PID.3']['PID.3.1']      → adt.pid?.pid3?[0]?.cx1 ?: ""
// Mirth: msg['MSH']['MSH.4']['MSH.4.1']      → adt.msh.msh4?.hd1 ?: ""
// Mirth: msg['MSH']['MSH.9']['MSH.9.1']      → adt.msh.msh9?.cm_msg1 ?: ""
// Mirth: foreach NK1 segment                  → foreach hl7v23:NK1 nk1 in adt.nk1 ?: []
// Mirth: for each OBX                         → foreach hl7v23:OBX obx in msg.obx ?: []
```

### Side-effect / generic-processor steps — activities

Every activity that can fail returns `ConnectionError|ExecutionError` (or `T|ConnectionError|ExecutionError`
for one that also returns a value on success) rather than a bare `error`, per "Error Types —
Required in Every Project" above:

```ballerina
@workflow:Activity
isolated function lookupPatientInDb(string patientId) returns boolean|ConnectionError|ExecutionError {
    sql:Client|error dbClient = getDbClient();
    if dbClient is error {
        return error ConnectionError("Could not connect to patient DB", dbClient);
    }
    boolean|error exists = checkPatientExists(dbClient, patientId);
    if exists is error {
        return error ExecutionError("Patient lookup query failed", exists, patientId = patientId);
    }
    return exists;
}
```

Called with a human-review `retryPolicy`, exactly like every other activity in the project:

```ballerina
boolean|ConnectionError|ExecutionError patientExists = ctx->callActivity(lookupPatientInDb,
        {"patientId": patient.patientId}, stepId = "lookup_patient_in_db",
        retryPolicy = {userRoles: "OPS", title: "Patient lookup failed"});
```

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

**Project convention — human-review `retryPolicy` on every call, no exceptions:**

```ballerina
string|ConnectionError|ExecutionError actResult = ctx->callActivity(sendToRadiMetrics,
        {"hl7Message": message}, stepId = "send_to_radimetrics",
        retryPolicy = {userRoles: "MANAGER", title: "Failure in HL7 Sender"});
```

`retryPolicy` here is a `ReviewTaskDefinition`, not an `AutoRetry` record. On failure the engine
raises a review task for the named role(s) instead of retrying automatically or failing outright;
a person decides whether to rerun the activity, rerun it with edited input, or fail it. Two fields
matter most when writing one of these for a generated activity:

| Field | What to put there |
|---|---|
| `userRoles` | The role that should see and act on this failure — carry over from context if the original Mirth channel had an operational owner/team, otherwise a sensible default such as `"OPS"` for infrastructure-facing steps and `"MANAGER"` for business-facing destination sends |
| `title` | Short, specific, and namable in an incident — "Failure in HL7 Sender", "Failure sending patient to FHIR server" — not a generic "Activity failed" |

This applies uniformly: critical and non-critical destinations, side-effect steps, and downstream
sends all use this same `retryPolicy` shape. What differs between critical and non-critical is
only whether the workflow function `check`s the result or captures it as `T|error` — see Phase 12.

### Destinations — activities, called sequentially (see Phase 12 for critical/non-critical)

```ballerina
@workflow:Activity
isolated function sendToFhirServer(PatientRecord patient) returns string|ConnectionError|ExecutionError {
    http:Client|error fhirClient = new (fhirServerUrl);
    if fhirClient is error {
        return error ConnectionError("Could not connect to FHIR server", fhirClient, url = fhirServerUrl);
    }
    json|error response = fhirClient->/Patient.post(patient);
    if response is error {
        return error ExecutionError("FHIR server rejected the request", response, patientId = patient.patientId);
    }
    // Response transformer equivalent: inspect the response here before returning.
    string|error id = response.id;
    if id is error {
        return error ExecutionError("FHIR response missing id field", id);
    }
    return id;
}
```

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

```ballerina
import ballerinax/health.clients.hl7;
import ballerinax/health.hl7v2;
import ballerina/workflow;

configurable string destHost = "downstream.example.com";
configurable int destPort = 2575;

final hl7:HL7Client hl7SenderClient = check new (destHost, destPort);

// HL7Client handles MLLP framing automatically — do not wrap bytes manually.
@workflow:Activity
isolated function sendToDownstream(json messageJson) returns json|ConnectionError|ExecutionError {
    hl7v2:Message|error msg = hl7v2:parse(messageJson.toString());
    if msg is error {
        return error ExecutionError("Could not re-parse HL7 message before send", msg);
    }
    hl7v2:Message|hl7v2:HL7Error ack = hl7SenderClient->sendMessage(msg);
    if ack is hl7v2:HL7Error {
        // A send/connect failure at the transport level — classify as ConnectionError so a
        // reviewer knows this is "the endpoint was unreachable," not "the payload was rejected."
        return error ConnectionError("HL7 send failed", ack, host = destHost, port = destPort);
    }
    return ack.toJson();
}
```

Called from the workflow function exactly like any other destination activity — always with the
human-review `retryPolicy`:

```ballerina
json|ConnectionError|ExecutionError ackResult = ctx->callActivity(sendToDownstream,
        {"messageJson": normalized}, stepId = "send_hl7_downstream",
        retryPolicy = {userRoles: "MANAGER", title: "Failure in HL7 Sender"});
```

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
- For database `DbConfig` records, annotate the password field:

```ballerina
public type DbConfig record {|
    string host;
    int port;
    string user;
    @sql:SensitiveConfig
    string password;
    string database;
|};
```

---

## Phase 12: Error Handling — Required Pattern for This Skill

This is the pattern that must be applied to **every** channel this skill migrates: for each
destination connector, decide critical or non-critical, then translate all destinations as
**separate durable activities, called sequentially, every one carrying a human-review
`retryPolicy`**, applying **skip-on-failure to the non-critical ones** once a reviewer has had the
chance to act (or the review task itself times out / is resolved as "fail it" — see the note on
review-task resolution below).

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

```ballerina
public type ConnectionError distinct error;
public type ExecutionError distinct error;
```

(See "Error Types — Required in Every Project" near the top of this file for the full usage rule.)

### Step 3 — translate as sequential `ctx->callActivity()` calls, every one with a human-review `retryPolicy`

```ballerina
@workflow:Workflow
function processOrder(workflow:Context ctx, OrderInput input) returns OrderResult|error {
    // CRITICAL destination — check propagates failure, fails the whole workflow. Still uses the
    // human-review retryPolicy: on failure a reviewer is paged before the workflow is allowed to
    // fail, rather than failing silently/automatically.
    string reservationId = check ctx->callActivity(reserveInventory, {
        "orderId": input.orderId, "item": input.item, "quantity": input.quantity
    }, stepId = "reserve_inventory",
       retryPolicy = {userRoles: "OPS", title: "Failure reserving inventory"});

    // CRITICAL destination — depends on the first; still `check`'d, still sequential, still
    // human-reviewed on failure.
    string paymentTxnId = check ctx->callActivity(chargePayment, {
        "orderId": input.orderId, "amount": input.amount
    }, stepId = "charge_payment",
       retryPolicy = {userRoles: "MANAGER", title: "Failure charging payment"});

    // NON-CRITICAL destination — captured as T|ConnectionError|ExecutionError. On failure a
    // reviewer is paged (same retryPolicy shape as the critical steps above); once that review
    // resolves without a successful rerun, the workflow continues and this step is skipped.
    string|ConnectionError|ExecutionError emailResult = ctx->callActivity(sendConfirmationEmail, {
        "email": input.customerEmail, "orderId": input.orderId
    }, stepId = "send_confirmation_email",
       retryPolicy = {userRoles: "OPS", title: "Failure sending confirmation email"});

    // NON-CRITICAL destination — same pattern, independent step, independently durable.
    string|ConnectionError|ExecutionError auditResult = ctx->callActivity(writeAuditLog, {
        "orderId": input.orderId, "reservationId": reservationId
    }, stepId = "write_audit_log",
       retryPolicy = {userRoles: "OPS", title: "Failure writing audit log"});

    string[] skipped = [];
    if emailResult is error {
        skipped.push("send_confirmation_email");
    }
    if auditResult is error {
        skipped.push("write_audit_log");
    }

    return {
        orderId: input.orderId,
        status: "COMPLETED",
        reservationId,
        paymentTxnId,
        skippedSteps: skipped
    };
}
```

Notes that make this correct, not just plausible-looking:

- Every `ctx->callActivity()` call gets an explicit, constant `stepId` string that mirrors the
  Mirth destination's name — this is what shows up in the Temporal Web UI's event history and in
  the review task, so name it the way you'd want to find it during an incident.
- Every `ctx->callActivity()` call gets a `retryPolicy = {userRoles: ..., title: ...}` — no
  exceptions, no `AutoRetry`, no unset (default `NoAutomaticRetry`) `retryPolicy`. This is a fixed
  project convention (Phase 7), applied to critical and non-critical destinations alike.
- Critical destinations use `check` — once the raised review task resolves without a successful
  rerun, the error propagates and fails the workflow run, exactly like Mirth's "Never queue, fail
  the message," except a human had the chance to intervene first.
- Non-critical destinations bind the result to a `T|ConnectionError|ExecutionError` variable
  instead of `check`ing it. Do **not** discard the value with `_` if you intend to report skipped
  steps — you need the `is error` branch. A no-value non-critical activity (`error?` return) still
  needs an explicit binding for the type checker to work with:
  `ConnectionError|ExecutionError? emailResult = ctx->callActivity(...)`, then
  `if emailResult is error { skipped.push(...); }`.
- `ChannelResult`/`OrderResult` should carry a `skippedSteps: string[]` (or richer, a
  `map<string>` of stepId → skip reason, ideally including which error type it was) so the skip
  outcome is visible to whatever called `workflow:getWorkflowResult()`, not just buried in the
  Temporal event history and the review task log.
- Because each `callActivity()` call is individually recorded, a transient crash mid-run does
  **not** re-send to destinations that already succeeded — recovery resumes from the next
  un-recorded step. This is the durability payoff for giving up parallel fan-out.
- **Resolving the review task itself** is a separate concern from this workflow-function code: a
  person (or an operational tool acting on their behalf) resolves a raised review task through the
  module's general human-task completion surface (`workflow:completeHumanTask()`, discoverable via
  `management:listPendingHumanTasks()` — see `ballerina/workflow`'s own human-in-the-loop
  documentation). The exact decision payload shape for a *review* task specifically (rerun / rerun
  with edited input / fail) was not spelled out in the reference material this skill was built
  from — confirm the live module's reference before hardcoding that shape into generated code or
  tests, rather than guessing at field names.

---

## Phase 13: Generating a Test Scenario

Every migrated project from this skill includes a runnable test that exercises the workflow
end-to-end without needing a real Temporal server, plus one test that specifically proves the
skip-on-failure behavior for a non-critical destination.

### `tests/Config.toml` — force in-memory mode for tests

Ballerina automatically layers a `tests/Config.toml` on top of the root `Config.toml` when running
`bal test`. Use it to switch the workflow engine into `IN_MEMORY` mode so tests run fast, in-process,
and without any external Temporal server:

```toml
[ballerina.workflow]
mode = "IN_MEMORY"

# Point any external clients used by activities at local/mock endpoints for the test run.
fhirServerUrl = "http://localhost:9090/fhir/r4"
destHost = "localhost"
destPort = 8776
```

### `tests/resources/sample-input.json` (or `.hl7`) — one representative message

Keep this small and representative rather than exhaustive — a single ADT^A01 (or the channel's
actual primary message type) with the fields the workflow's filters/transformers actually branch
on.

### `tests/workflow_test.bal` — happy path

```ballerina
import ballerina/test;
import ballerina/workflow;
import ballerina/io;

@test:Config {}
function testProcessOrderHappyPath() returns error? {
    string sampleInput = check io:fileReadString("tests/resources/sample-input.json");
    OrderInput input = {orderId: "ORD-TEST-001", item: "widget", quantity: 1,
        amount: 19.99, customerEmail: "test@example.com"};

    string workflowId = check workflow:run(processOrder, input);

    anydata result = check workflow:getWorkflowResult(workflowId);
    OrderResult orderResult = check result.cloneWithType();

    test:assertEquals(orderResult.status, "COMPLETED");
    test:assertEquals(orderResult.orderId, "ORD-TEST-001");
    test:assertEquals(orderResult.skippedSteps.length(), 0);
}
```

### `tests/workflow_test.bal` — skip-on-failure scenario, with the human-review step in the loop

Because every destination in this design uses a human-review `retryPolicy` (Phase 7/12), a failing
activity does not resolve itself the moment the call fails — it raises a review task and the
workflow durably pauses at that step until the task is resolved. So a test proving the
skip-on-failure path has one more beat than a plain retry-based test would: drive the underlying
call to fail, then resolve the resulting review task as "fail it" (via the module's human-task
completion surface — see the caveat in Phase 12 about confirming the exact decision payload shape
against the live module reference before hardcoding it here), and only then assert on the
completed, skipped outcome:

```ballerina
@test:Config {}
function testNonCriticalDestinationSkippedOnFailure() returns error? {
    // tests/Config.toml points destHost/destPort (used by sendConfirmationEmail's underlying
    // client) at an address nothing is listening on, so this destination will fail every attempt
    // and raise a review task instead of resolving on its own.
    OrderInput input = {orderId: "ORD-TEST-002", item: "widget", quantity: 1,
        amount: 9.99, customerEmail: "unreachable@example.invalid"};

    string workflowId = check workflow:run(processOrder, input);

    // TODO: resolve the raised review task for the "send_confirmation_email" step as "fail it,"
    // using the module's human-task completion surface (workflow:completeHumanTask() /
    // management:listPendingHumanTasks()) — confirm the review-task decision payload shape
    // against the live ballerina/workflow reference before filling this in; it was not part of
    // the material this skill was authored from.

    anydata result = check workflow:getWorkflowResult(workflowId);
    OrderResult orderResult = check result.cloneWithType();

    // The workflow still completes — the failing destination is non-critical, and the raised
    // review task was resolved as "fail it" above rather than left pending.
    test:assertEquals(orderResult.status, "COMPLETED");
    test:assertTrue(orderResult.skippedSteps.indexOf("send_confirmation_email") is int);
}

@test:Config {}
function testCriticalDestinationFailurePropagates() returns error? {
    // Use an input that the critical activity (e.g. chargePayment) is written to reject,
    // to prove that a critical-path failure — once its review task is resolved as "fail it" —
    // fails the whole workflow rather than being skipped.
    OrderInput input = {orderId: "ORD-TEST-003", item: "widget", quantity: 1,
        amount: -1.00, customerEmail: "test@example.com"};

    string workflowId = check workflow:run(processOrder, input);

    // TODO: resolve the raised review task for the "charge_payment" step as "fail it" — see the
    // same caveat as the skip-on-failure test above.

    anydata|error result = workflow:getWorkflowResult(workflowId);

    test:assertTrue(result is error);
}
```

### `tests/workflow_test.bal` — the source connector never surfaces an error

Because Phase 3 requires every listener to always acknowledge receipt regardless of the eventual
workflow outcome, add a service-level test proving that, even when the workflow underneath fails
outright, the HTTP/MLLP/file entry point still responds success. For the HTTP source connector:

```ballerina
import ballerina/http;

@test:Config {}
function testServiceAlwaysRespondsAcceptedEvenOnWorkflowFailure() returns error? {
    http:Client testClient = check new ("http://localhost:8090");
    // A payload built to make the underlying workflow fail on a critical step.
    json badPayload = {"orderId": "ORD-TEST-004", "amount": -1.00};

    http:Response response = check testClient->post("/api/v1/messages", badPayload);

    // The resource function never returns |error (Phase 3) — this must be 202 regardless of
    // what happens to the workflow it started.
    test:assertEquals(response.statusCode, 202);
}
```

Notes:

- There is no documented activity-mocking facility in `ballerina/workflow` at the time of writing
  — these tests exercise the real activity functions end-to-end against `IN_MEMORY` mode, driving
  failure via test-environment configuration (bad endpoint, invalid input) rather than swapping in
  a mock. If the project already uses Ballerina's general-purpose `test:mock()` / module-level
  function mocking for its activities, that still works the normal Ballerina way — it's orthogonal
  to the workflow engine — but don't invent a workflow-specific mocking API that isn't documented.
- Keep the failing-destination test's failure *deterministic* (an address nothing listens on, not
  a flaky network call) — a flaky test here is worse than no test, since it undermines confidence
  in the skip-on-failure guarantee it's meant to demonstrate.
- Every review-task-resolution `TODO` above is a real gap to close, not decorative — a test that
  calls `workflow:getWorkflowResult()` right after `workflow:run()` without resolving the pending
  review task will hang (or time out per the test framework's own timeout) rather than reach the
  assertion, precisely because the human-review policy is now mandatory on every activity. Do not
  ship a generated test suite with this TODO unresolved and call it passing.
- For a channel with a durable-sleep or external-data-wait step (timers, human-in-the-loop
  approvals) *in addition to* the mandatory review-task-on-failure behavior, `IN_MEMORY` mode still
  checkpoints correctly but the test needs to send the relevant external data via
  `workflow:sendData()` (or resolve the relevant human task) before asserting — call this out as a
  `// TODO` in the generated test if the channel has such a step, rather than guessing at a safe
  wait duration.

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
