# Activity Examples (workflow-based)

Consulted during Phase 7 (Writing Activities) and Phase 9 (MLLP Sender) of `SKILL.md`. Every
activity that can fail returns `ConnectionError|ExecutionError` (or `T|ConnectionError|ExecutionError`
for one that also returns a value on success) rather than a bare `error`, per "Error Types —
Required in Every Project" in `SKILL.md`.

## Filters — plain functions, not activities

```ballerina
isolated function passesSourceFilter(string normalized) returns boolean {
    // TODO: implement — see original <filter><rule> elements
    return true;
}
```

A filter returning `false` maps to an early `return {status: "FILTERED"};` in the workflow
function (Phase 4) — there is no separate "drop silently" mechanism to invoke.

## Transformers — plain functions unless they need external data

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

## Side-effect / generic-processor steps — activities

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

## Destinations — activities, called sequentially (see Phase 12 for critical/non-critical)

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

Project convention — human-review `retryPolicy` on every call, no exceptions:

```ballerina
string|ConnectionError|ExecutionError actResult = ctx->callActivity(sendToRadiMetrics,
        {"hl7Message": message}, stepId = "send_to_radimetrics",
        retryPolicy = {userRoles: "MANAGER", title: "Failure in HL7 Sender"});
```

## MLLP Sender (TCP Dispatcher) as an activity — Phase 9

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

## `globalChannelMap` / `globalMap` state — Phase 5

In a `@workflow:Workflow` function, reading or writing mutable module-level state directly is a
determinism violation the compiler does **not** catch. So unlike a pipeline translation where
`msgCtx`-adjacent module state can be touched inline, here the get/set must go through an
`@workflow:Activity` so the engine records the interaction:

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

These two are an exception to the `ConnectionError`/`ExecutionError` typing convention (see "Error
Types — Required in Every Project" in `SKILL.md`) — a `lock{}`-guarded in-memory map read/write
doesn't fail in a way that's meaningfully "connection" or "execution," so plain
`anydata|error`/`error?` is fine here. They still carry a human-review `retryPolicy` when called
via `ctx->callActivity()`, per the project convention in Phase 7 — even an in-memory operation gets
the same treatment, since the convention is "every activity," not "every activity that talks to a
network."
```
