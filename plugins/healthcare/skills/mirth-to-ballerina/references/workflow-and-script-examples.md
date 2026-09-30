# Workflow Function & Channel-Script Examples (workflow-based)

Consulted during Phase 4 (Model the Message Flow) and Phase 6 (Channel-Level Scripts) of
`SKILL.md`.

## Phase 4 — the `@workflow:Workflow` function shape

```ballerina
import ballerina/workflow;
import ballerinax/health.hl7v23;

@workflow:Workflow
function processChannelMessage(workflow:Context ctx, ChannelInput input) returns ChannelResult|error {
    // 1. Preprocessor equivalent — pure mutation, plain function call, no callActivity needed.
    string normalized = normalizeRawMessage(input.rawMessage);

    // 2. Source filter equivalent — plain boolean check, early return drops the message
    //    (silent drop, matching Mirth's FilterConfig returning false).
    if !passesSourceFilter(normalized) {
        return {status: "FILTERED"};
    }

    // 3. Source transformer equivalent — pure data shaping, plain function call. Parses into the
    //    library's own typed message (hl7v23:ADT_A01) rather than a hand-rolled record — see
    //    references/activity-examples.md for why.
    hl7v23:ADT_A01 adt = check parseAdt(normalized);
    string patientId = adt.pid?.pid3?[0]?.cx1 ?: "";

    // 4. Side-effect / lookup step with external I/O — this DOES need an activity. Every
    //    callActivity call always carries a human-review retryPolicy (Phase 7) — never
    //    AutoRetry, never the default NoAutomaticRetry.
    boolean patientExists = check ctx->callActivity(lookupPatientInDb, {"patientId": patientId},
            stepId = "lookup_patient_in_db",
            retryPolicy = {userRoles: "OPS", title: "Patient lookup failed"});

    // 5. Destinations — separate durable activities, called sequentially, each critical or
    //    non-critical, each with a human-review retryPolicy. See Phase 12 for the full pattern
    //    with skip-on-failure. Activities return ConnectionError|ExecutionError on failure
    //    (Phase "Error Types"), never a bare error. The FHIR bundle is derived from the raw
    //    message via the library's own v2ToFhir() mapper, not a custom "patient" shape.
    json fhirBundle = check translateToFhirBundle(normalized);
    string reservationId = check ctx->callActivity(sendToFhirServer, {"fhirBundle": fhirBundle},
            stepId = "send_to_fhir",
            retryPolicy = {userRoles: "MANAGER", title: "Failure sending patient to FHIR server"});

    string|ConnectionError|ExecutionError auditResult = ctx->callActivity(writeAuditLog,
            {"patientId": patientId}, stepId = "write_audit_log",
            retryPolicy = {userRoles: "OPS", title: "Failure writing audit log"});

    // 6. Postprocessor equivalent — results are already local variables, just assemble the outcome.
    string[] skipped = [];
    if auditResult is error {
        skipped.push("write_audit_log");
    }
    return {status: "COMPLETED", reservationId, skippedSteps: skipped};
}
```

## Phase 6 — channel-level scripts

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
    // Original used: message.replace('﻿', '').replace('\r\n', '\r')
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
`workflow:getWorkflowResult()`'s return value — see the MLLP example in
`references/source-connector-examples.md`.

---

## Mirth concept → `ballerina/workflow` equivalent

Consulted from Phase 4 of `SKILL.md`. This is the lookup that drives the translation decisions; the
`processChannelMessage` example above shows the result assembled into one workflow function.

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
| Multiple destinations | Sequential `ctx->callActivity()` calls, **not** parallel |
| Destination filter | Plain boolean check before the corresponding `callActivity` call |
| Response Transformer | Inline code immediately after that destination's `callActivity` call, using its return value |
| Postprocessor Script | Inline code near the end of the workflow function, working off local result variables |
| Any destination's retry/queue behavior | Not `AutoRetry` — a human-review `retryPolicy` (`ReviewTaskDefinition`), always. See Phase 7/12. |

---

## Mirth variable maps → Ballerina

Consulted from Phase 5 of `SKILL.md`. Because the whole channel is one workflow function
invocation rather than a pipeline object threading a `MessageContext` through separate processor
calls, most of Mirth's variable maps collapse into ordinary local variables.

| Mirth map | JS variable | Scope | `ballerina/workflow` equivalent |
|---|---|---|---|
| **Connector Map** | `connectorMap` / `$co` | Current message, current connector only | Local variable inside the relevant activity function |
| **Channel Map** | `channelMap` / `$c` | Current message, shared across destinations | Local variable in the workflow function, passed as an argument to whichever activity needs it |
| **Source Map** | `sourceMap` / `$s` | Current message, read-only, injected by source | Fields on the `ChannelInput` record passed to `workflow:run()` |
| **Response Map** | `responseMap` / `$r` | Current message, destination responses | The return value of each `ctx->callActivity()` call, held in a local variable |
| **Global Channel Map** | `globalChannelMap` / `$gc` | All messages in this channel, in-memory only | Must be read/written via an activity — see the caveat in Phase 5 |
| **Global Map** | `globalMap` / `$g` | All messages, all channels, in-memory only | Same as above |
| **Configuration Map** | `configurationMap` / `$cfg` | Read-only server config | `configurable` Ballerina variables in `Config.toml` |

For the `globalChannelMap`/`globalMap` pattern — a `lock{}`-guarded module-level map behind
`readGlobalChannelState`/`writeGlobalChannelState` activities, with its restart/scaling caveats and
why these two are an exception to the `ConnectionError`/`ExecutionError` typing rule — see
`references/activity-examples.md`.

### `sourceMap` variables injected by specific connectors

| Mirth source connector | Automatic sourceMap keys | Ballerina approach |
|---|---|---|
| File Reader | `originalFilename`, `fileDirectory`, `fileSize`, `fileLastModified` | Fields on `ChannelInput` (Phase 3) |
| HTTP Listener | `remoteAddress`, `localAddress`, HTTP headers as `http.*` | Extract from `http:Request` before calling `workflow:run()`, pass as `ChannelInput` fields |
| Database Reader | Column names from the query result | Fields on `ChannelInput` |
| Channel Writer (upstream) | Any variables injected by upstream channel | Document as a TODO — requires tracing the upstream channel |
