# Workflow Function & Channel-Script Examples (workflow-based)

Consulted during Phase 4 (Model the Message Flow) and Phase 6 (Channel-Level Scripts) of
`SKILL.md`.

## Phase 4 — the `@workflow:Workflow` function shape

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
