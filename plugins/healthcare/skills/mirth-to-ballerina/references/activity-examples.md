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

**Parse into the library's own typed message, don't repackage into a custom record.** The version
package's message type (`hl7v23:ADT_A01`, etc.) already models every field Mirth's JS would have
read off `msg['PID']...` — there's no need for a hand-rolled `PatientRecord`-style type that just
duplicates a subset of `PID`'s fields under different names:

```ballerina
isolated function parseAdt(string normalized) returns hl7v23:ADT_A01|error
    => hl7v2:parse(normalized).ensureType(hl7v23:ADT_A01);
```

Downstream code reads whatever fields it needs directly off the parsed message — e.g.
`string patientId = adt.pid?.pid3?[0]?.cx1 ?: "";` — rather than through a custom accessor, exactly
as shown in Ballerina's own HL7 guides (`adtMsg.pid.pid5`, etc.).

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
        {"patientId": patientId}, stepId = "lookup_patient_in_db",
        retryPolicy = {userRoles: "OPS", title: "Patient lookup failed"});
```

## Destinations — activities, called sequentially (see Phase 12 for critical/non-critical)

**When the destination is a FHIR server, don't hand-build a request against a raw `http:Client`.**
`ballerinax/health.clients.fhir` provides a dedicated `FHIRConnector` with typed `create`/`update`/
`'transaction`/`search` operations, and `ballerinax/health.hl7v2<ver>.utils.v2tofhirr4` converts an
HL7v2 message straight into a standards-compliant FHIR Bundle (`v2ToFhir()`) instead of the skill
hand-mapping PID fields into a custom "patient" shape:

```ballerina
import ballerinax/health.clients.fhir;
import ballerinax/health.hl7v23.utils.v2tofhirr4;

configurable string fhirServerUrl = "https://fhir.example.com/fhir/r4";

final fhir:FHIRConnector fhirConnector = check new ({baseURL: fhirServerUrl, mimeType: fhir:FHIR_JSON});

// Pure — v2ToFhir() maps segments to resources per the official HL7 v2-to-FHIR IG; no custom
// per-field extraction to hand-maintain.
isolated function translateToFhirBundle(string normalized) returns json|error
    => v2tofhirr4:v2ToFhir(normalized);

@workflow:Activity
isolated function sendToFhirServer(json fhirBundle) returns string|ConnectionError|ExecutionError {
    fhir:FHIRResponse|error response = fhirConnector->'transaction(fhirBundle);
    if response is error {
        return error ExecutionError("FHIR server rejected the request", response);
    }
    // Response transformer equivalent: inspect the response here before returning.
    string|error id = response.'resource.id;
    if id is error {
        return error ExecutionError("FHIR response missing id field", id);
    }
    return id;
}
```

`FHIRConnector`'s own client-level connection failures (DNS, refused connection, TLS) surface as
`error` from the `->` call same as `response is error` above — classify a failure at that layer as
`ConnectionError` instead of `ExecutionError` if the two need to be told apart for a given channel;
the example above keeps it simple because `'transaction()` doesn't distinguish the two cases in its
return type.

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

@workflow:Activity
isolated function sendToDownstream(string encodedMessage) returns json|ConnectionError|ExecutionError {
    hl7v2:Message|hl7v2:HL7Error msg = hl7v2:parse(encodedMessage);
    if msg is hl7v2:HL7Error {
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

> `HL7Client.sendMessage()`'s own implementation (`writeToHL7Stream`) writes the encoded bytes
> straight to the TCP socket with no visible MLLP start/end block wrapping — confirm whether the
> target server expects a raw encoded message or a full MLLP envelope (`0x0B` … `0x1C 0x0D`) around
> it before shipping; do not assume framing is handled for you on the sending side either.

Called from the workflow function exactly like any other destination activity — always with the
human-review `retryPolicy`:

```ballerina
json|ConnectionError|ExecutionError ackResult = ctx->callActivity(sendToDownstream,
        {"encodedMessage": normalized}, stepId = "send_hl7_downstream",
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

---

## Annotation and signature rules that must hold, always

Consulted from Phase 7 of `SKILL.md`. Each violation's compiler error code is given so a failed
build maps straight back to the rule it broke.

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

## Writing the human-review `retryPolicy`

`retryPolicy` is a `ReviewTaskDefinition`, never an `AutoRetry` record and never the default
`NoAutomaticRetry`. On failure the engine raises a review task for the named role instead of
retrying automatically or failing outright; a person decides whether to rerun the activity, rerun
it with edited input, or fail it. Two fields matter most:

| Field | What to put there |
|---|---|
| `userRoles` | The role that should see and act on this failure — carry over from context if the original Mirth channel had an operational owner/team, otherwise a sensible default such as `"OPS"` for infrastructure-facing steps and `"MANAGER"` for business-facing destination sends |
| `title` | Short, specific, and namable in an incident — "Failure in HL7 Sender", "Failure sending patient to FHIR server" — not a generic "Activity failed" |

This applies uniformly: critical and non-critical destinations, side-effect steps, and downstream
sends all use this same `retryPolicy` shape. What differs between critical and non-critical is only
whether the workflow function `check`s the result or captures it as `T|error` — see Phase 12 and
`references/retry-and-skip-mapping.md`.
