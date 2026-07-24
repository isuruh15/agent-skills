---
name: mirth-to-ballerina
description: >
  Migrates Mirth Connect channel XML to a compilable Ballerina project (Ballerina.toml + modules).
  Prioritizes HL7v2 MLLP/LLP listeners and TCP senders. Translates JavaScript transformers, filters,
  and channel-level scripts into idiomatic Ballerina code, generating typed stubs with full contract
  comments where the translation is ambiguous.

  Use this skill whenever the user:
  - Shares a Mirth Connect channel.xml file or channel XML snippet and wants it converted to Ballerina
  - Mentions migrating from Mirth Connect to Ballerina Integrator or WSO2 Integration
  - Asks how to implement an HL7v2 MLLP listener, MLLP sender, or TCP HL7 channel in Ballerina
  - Wants to translate Mirth JavaScript transformers, filters, or channel scripts to Ballerina
  - Needs a Ballerina project that replaces a Mirth channel pipeline
  - Asks about Mirth Connect channel migration to any Ballerina-based platform
---

# Mirth Connect → Ballerina Migration Skill

You are an expert in both Mirth Connect and Ballerina healthcare integration. When given a Mirth Connect channel XML, produce a **complete, compilable Ballerina project** that faithfully reproduces the channel's behavior.

## What You Always Produce

Every migration results in a project directory containing:

```
<channel-name>/
├── Ballerina.toml
├── Config.toml         (configurable values, no hardcoded secrets)
├── types.bal           (record type definitions)
├── handlers.bal        (pipeline processors, filters, transformers, destinations)
├── service.bal         (HandlerChain wiring + listener)
└── utils.bal           (shared helpers — only if needed)
```

Only include files that are actually needed. For simple channels, `types.bal` + `handlers.bal` + `service.bal` is enough.

---

## Phase 1: Analyze the Channel XML

Parse the XML and identify:

| XML element | What to look for |
|---|---|
| `<sourceConnector>` | Connector class → listener type (MLLP, HTTP, File, etc.) |
| `<destinationConnectors>` | Each destination's class, properties, and queue settings |
| `<transformer>` | Step list: `<step>` elements with `<type>` and `<script>` |
| `<filter>` | Rule list: `<rule>` elements with `<type>` and `<script>` |
| `<responseTransformer>` | Post-send response handling logic |
| `<properties>` | Port, host, encoding, transmission mode, data type |
| `<preprocessorScript>` | Per-message raw message mutation (runs before source filter) |
| `<postprocessorScript>` | Per-message post-processing (runs after all destinations) |
| `<deployScript>` | One-time channel startup logic |
| `<undeployScript>` | One-time channel teardown logic |

**Connector class → Ballerina mapping:**

| Mirth connector class | Ballerina equivalent |
|---|---|
| `com.mirth.connect.connectors.tcp.TcpReceiver` (MLLP) | `hl7v2:Hl7Listener` + `hl7v2:Hl7Service` |
| `com.mirth.connect.connectors.tcp.TcpDispatcher` (MLLP) | `health.clients.hl7:HL7Client` |
| `com.mirth.connect.connectors.http.HttpReceiver` | `http:Listener` service |
| `com.mirth.connect.connectors.http.HttpDispatcher` | `http:Client` |
| `com.mirth.connect.connectors.file.FileReceiver` | `file:Listener` + `io` |
| `com.mirth.connect.connectors.file.FileDispatcher` | `io:fileWriteString` |
| `com.mirth.connect.connectors.jdbc.DatabaseReader` | `sql` + DB connector |
| `com.mirth.connect.connectors.jdbc.DatabaseWriter` | `sql` + DB connector |
| `com.mirth.connect.connectors.js.JavaScriptReader` | Ballerina service (translate logic) |
| `com.mirth.connect.connectors.js.JavaScriptWriter` | Ballerina function (translate logic) |
| `com.mirth.connect.connectors.vm.VmReceiver` | `tcp:Listener` or `http:Listener` depending on data type |
| `com.mirth.connect.connectors.vm.VmDispatcher` (Channel Writer) | Direct function call or pipeline destination |

---

## Phase 2: Build the Ballerina.toml

```toml
[package]
org = "healthcare"
name = "<channel_name_snake_case>"
version = "1.0.0"
distribution = "2201.12.2"

[build-options]
observabilityIncluded = true
```

**Add dependencies based on what connectors are present:**

| Connector type | Dependency to add |
|---|---|
| HL7v2 MLLP listener/sender | `ballerinax/health.hl7v2`, `ballerinax/health.hl7v2<ver>` (e.g. `health.hl7v23`), `ballerinax/health.clients.hl7` |
| HTTP | (stdlib, no extra dep) |
| File | (stdlib) |
| Database (MySQL) | `ballerinax/mysql` + `ballerina/sql` |
| Database (PostgreSQL) | `ballerinax/postgresql` + `ballerina/sql` |
| Email/SMTP | `ballerina/email` |
| FTP/SFTP | `ballerina/ftp` |
| JMS / queuing | `ballerinax/rabbitmq` |
| HL7v2 → FHIR conversion | `ballerinax/health.hl7v2<ver>.utils.v2tofhirr4` |
| **Always include** | `xlibb/pipeline`, `ballerina/log`, `ballerina/uuid`, `ballerina/time` |

**HL7v2 version package selection:**
- `hl7v23` for HL7 2.3 / 2.3.1 — `hl7v24` for 2.4 — `hl7v25`/`hl7v251` for 2.5/2.5.1
- `hl7v26` for 2.6 — `hl7v27` for 2.7 — `hl7v28` for 2.8

If not explicit in the channel XML, check `MSH.12` in example messages. Default to `hl7v23` for old channels.

---

## Phase 3: MLLP Listener — Use the HL7v2 Service API

**Do not manually strip MLLP frames or call `hl7v2:parse()` yourself.** The `ballerinax/health.hl7v2` library provides `Hl7Listener` + `Hl7Service` that handle MLLP framing (0x0B start, 0x1C 0x0D end) and parsing internally.

```ballerina
import ballerina/log;
import ballerinax/health.hl7v2;
import ballerinax/health.hl7v23;
import xlibb/pipeline;

configurable int mllpPort = 2575;

// Hl7Listener handles MLLP framing and hl7v2:parse() internally.
// Your service receives an already-parsed hl7v2:Message.
listener hl7v2:Hl7Listener mllpListener = new (mllpPort);

service hl7v2:Hl7Service on mllpListener {
    isolated remote function onMessage(hl7v2:Hl7Client caller, hl7v2:Message message) returns error? {
        pipeline:ExecutionSuccess|pipeline:ExecutionError result = channelPipeline.execute(message);
        if result is pipeline:ExecutionError {
            log:printError("Pipeline execution failed", 'error = result,
                           messageId = result.detail().message.id);
        }
    }
}
```

For **HTTP source connectors**:

```ballerina
service /api/v1 on new http:Listener(httpPort) {
    resource function post messages(http:Request request) returns http:Accepted|error {
        string payload = check request.getTextPayload();
        _ = start channelPipeline.execute(payload);
        return http:ACCEPTED;
    }
}
```

For **File Reader source connectors**:

```ballerina
service "fileWatcher" on new file:Listener({path: watchDirectory, recursive: false}) {
    remote function onModify(file:FileEvent event) returns error? {
        string content = check io:fileReadString(event.name);
        // sourceMap equivalent: pass filename as a pipeline property via a wrapper record
        _ = check channelPipeline.execute({content, originalFilename: event.name});
    }
}
```

---

## Phase 4: Model the Message Flow as a `HandlerChain` (xlibb/pipeline)

Every Mirth channel is a pipeline: source → preprocessor → source filter/transformer → destinations → postprocessor. Map it directly to `xlibb/pipeline`:

| Mirth concept | `xlibb/pipeline` equivalent |
|---|---|
| Source connector | Listener that calls `handlerChain.execute(message)` |
| Preprocessor Script | `@pipeline:ProcessorConfig` as first processor — mutates raw content before filtering |
| Source filter rules | `@pipeline:FilterConfig` function — `false` drops the message |
| Source transformer steps | `@pipeline:TransformerConfig` function — return value becomes new content |
| Side-effect steps (DB lookup, log) | `@pipeline:ProcessorConfig` function — sets properties, no content change |
| Destination connector | `@pipeline:DestinationConfig` function |
| Multiple destinations | Multiple destination functions — run **in parallel** automatically |
| Destination filter | Per-destination `FilterConfig` processor or `MessageMetadata.destinationsToSkip` |
| Response Transformer / Postprocessor | Logic inside the destination function after the send call |

### HandlerChain skeleton

```ballerina
import xlibb/pipeline;
import ballerinax/rabbitmq;

// Failure stores — use rabbitmq:MessageStore or any pipeline:Store implementation.
// Omit replayListenerConfig entirely if the original channel has no persistent queue ("Never" mode).
final rabbitmq:MessageStore failureStore   = check new ("channel-failure-store");
final rabbitmq:MessageStore deadLetterStore = check new ("channel-dead-letter-store");

final pipeline:HandlerChain channelPipeline = check new (
    name = "<channel-name>",
    processors = [
        preprocessMessage,       // Preprocessor Script → first processor
        filterByMessageType,     // Source filter → FilterConfig
        transformMessage,        // Source transformer → TransformerConfig
        lookupPatientInDb        // Side-effect step → ProcessorConfig
    ],
    destinations = [
        sendToFhirServer,        // Destinations run in parallel
        writeAuditLog
    ],
    failureStore = failureStore,
    replayListenerConfig = {     // Include only if Mirth queue mode is "On Failure" or "Always"
        pollingInterval: 5,
        maxRetries: 3,
        retryInterval: 2,
        deadLetterStore: deadLetterStore
    }
);
```

---

## Phase 5: Variable Maps → Ballerina

Mirth has seven variable maps with distinct scopes. Map each one as follows:

| Mirth map | JS variable | Scope | Ballerina equivalent |
|---|---|---|---|
| **Connector Map** | `connectorMap` / `$co` | Current message, current connector only | Local variables within a single handler function |
| **Channel Map** | `channelMap` / `$c` | Current message, shared across all destinations | `msgCtx.setProperty(key, value)` / `getPropertyWithType(key)` |
| **Source Map** | `sourceMap` / `$s` | Current message, read-only, injected by source | Initial properties on the `MessageContext` — pass as fields in the content record passed to `execute()`, or extract in the first `ProcessorConfig` |
| **Response Map** | `responseMap` / `$r` | Current message, destination responses | Destination function return value stored in `ExecutionSuccess.destinationResults[id]`; read by postprocessor logic |
| **Global Channel Map** | `globalChannelMap` / `$gc` | All messages in this channel, in-memory only | Module-level `isolated map<anydata>` with `lock{}` — see caveat below |
| **Global Map** | `globalMap` / `$g` | All messages, all channels, in-memory only | Module-level shared state with `lock{}`; consider an external cache for cross-service sharing |
| **Configuration Map** | `configurationMap` / `$cfg` | Read-only server config | `configurable` Ballerina variables in `Config.toml` |

**globalChannelMap / globalMap caveat** — always emit this TODO when these maps are used:

```ballerina
// TODO: globalChannelMap translated to module-level isolated map with lock{}.
// Caveats vs Mirth:
// 1. Reset on service restart — values are not persisted across deployments.
// 2. Not shared across multiple service instances — use Redis or a DB table if
//    horizontal scaling or restart-survival is required.
// 3. Mirth's globalChannelMap is a ConcurrentHashMap and does not allow null values —
//    maintain this invariant by never storing () here.
isolated map<anydata> globalChannelState = {};
```

**sourceMap variables injected by specific connectors:**

| Mirth source connector | Automatic sourceMap keys | Ballerina approach |
|---|---|---|
| File Reader | `originalFilename`, `fileDirectory`, `fileSize`, `fileLastModified` | Include in the content wrapper record passed to `execute()` |
| HTTP Listener | `remoteAddress`, `localAddress`, HTTP headers as `http.*` | Extract from `http:Request` before calling `execute()` and pass in the wrapper |
| Database Reader | Column names from the query result | Include as fields in the content record |
| Channel Writer (upstream) | Any variables injected by upstream channel | Document as a TODO — requires tracing the upstream channel |

---

## Phase 6: Channel-Level Scripts

Mirth channels have four lifecycle scripts. Translate each as follows:

### Deploy Script → module `init`

Runs once when the channel is deployed. Typically initializes DB connections, loads config from files, or seeds `globalChannelMap`.

```ballerina
// In service.bal — Ballerina module init runs once at startup, equivalent to Mirth Deploy Script.
// Original Deploy Script: loaded patient lookup cache from DB into globalChannelMap.
function init() returns error? {
    // TODO: implement — see original Deploy Script in <deployScript> element
    // Original used: DatabaseConnectionFactory.getConnection('mpi_db') to pre-load cache
    log:printInfo("Channel initialized");
}
```

### Undeploy Script → graceful stop / cleanup function

Runs once when the channel is undeployed. Typically closes DB connections or flushes state.

```ballerina
// Ballerina does not have a direct undeploy hook — use a graceful stop handler or
// implement cleanup in a function called at program termination.
// TODO: implement cleanup — see original Undeploy Script in <undeployScript> element
isolated function cleanupResources() returns error? {
    // e.g. close sql:Client connections, flush caches
}
```

### Preprocessor Script → first `ProcessorConfig` in the pipeline

Runs after the source connector receives the message but before the source filter/transformer. It receives the raw message as a string and its return value replaces the raw message. Translate to the first processor in the `HandlerChain`:

```ballerina
// Translates Mirth Preprocessor Script — mutates raw message string before filtering.
// Original: stripped BOM characters and normalized line endings before HL7 parsing.
@pipeline:ProcessorConfig {id: "preprocess_raw_message"}
isolated function preprocessMessage(pipeline:MessageContext msgCtx) returns error? {
    string raw = check msgCtx.getContentWithType();
    // TODO: implement — see original <preprocessorScript> element
    // Original used: message.replace('﻿', '').replace('\r\n', '\r')
    string normalized = raw; // placeholder
    msgCtx.setProperty("preprocessedRaw", normalized);
}
```

### Postprocessor Script → logic at the end of the last destination / response handling

Runs after all destinations complete. Has access to all destination responses via `responseMap`. Its return value can be used as the response sent back to the source connector. Translate by reading `ExecutionSuccess.destinationResults` after `execute()` returns:

```ballerina
service hl7v2:Hl7Service on mllpListener {
    isolated remote function onMessage(hl7v2:Hl7Client caller, hl7v2:Message message) returns error? {
        pipeline:ExecutionSuccess|pipeline:ExecutionError result = channelPipeline.execute(message);
        // Postprocessor equivalent: inspect destination results and send back a response
        if result is pipeline:ExecutionSuccess {
            // responseMap equivalent: result.destinationResults["dest-id"] holds each destination's return value
            _ = check postprocess(caller, result);
        } else {
            log:printError("Pipeline failed", 'error = result, messageId = result.detail().message.id);
        }
    }
}

// Translates Mirth Postprocessor Script — assembles final response to originating system.
// Original: checked ACK code from HL7 destination response; re-queued on AE, sent AA on success.
isolated function postprocess(hl7v2:Hl7Client caller, pipeline:ExecutionSuccess result) returns error? {
    // TODO: implement — see original <postprocessorScript> element
    // responseMap.get('dest1') → result.destinationResults["dest1"]
}
```

---

## Phase 7: Writing Handlers in `handlers.bal`

### Filters — translate Mirth source/destination filter rules

```ballerina
import xlibb/pipeline;
import ballerinax/health.hl7v2;
import ballerinax/health.hl7v23;

@pipeline:FilterConfig {id: "filter_message_type"}
isolated function filterByMessageType(pipeline:MessageContext msgCtx) returns boolean|error {
    hl7v2:Message msg = check msgCtx.getContentWithType();
    hl7v23:ADT_A01|error adtA01 = msg.ensureType(hl7v23:ADT_A01);
    return adtA01 is hl7v23:ADT_A01;
}
```

A `FilterConfig` returning `false` drops the message silently. Returning `error` routes it to the failure store. For compound filter rules (Mirth ANDs them), write one function per logical concern and list them sequentially in `processors`.

### Transformers — translate Mirth transformer steps

```ballerina
@pipeline:TransformerConfig {id: "extract_patient"}
isolated function extractPatient(pipeline:MessageContext msgCtx) returns PatientRecord|error {
    hl7v2:Message msg = check msgCtx.getContentWithType();
    hl7v23:ADT_A01 adt = check msg.ensureType(hl7v23:ADT_A01);
    return {
        patientId: adt.pid?.pid3?[0]?.cx1 ?: "",
        lastName:  adt.pid?.pid5?[0]?.xpn1 ?: "",
        firstName: adt.pid?.pid5?[0]?.xpn2 ?: "",
        dob:       adt.pid?.pid7?.ts1 ?: ""
    };
}
```

**HL7v2 field access — always use optional chaining, never assume fields exist:**

```ballerina
// Mirth: msg['PID']['PID.3']['PID.3.1']      → adt.pid?.pid3?[0]?.cx1 ?: ""
// Mirth: msg['MSH']['MSH.4']['MSH.4.1']      → adt.msh.msh4?.hd1 ?: ""
// Mirth: msg['MSH']['MSH.9']['MSH.9.1']      → adt.msh.msh9?.cm_msg1 ?: ""
// Mirth: foreach NK1 segment                  → foreach hl7v23:NK1 nk1 in adt.nk1 ?: []
// Mirth: for each OBX                         → foreach hl7v23:OBX obx in msg.obx ?: []
```

### GenericProcessors — side-effect steps

```ballerina
@pipeline:ProcessorConfig {id: "db_patient_lookup"}
isolated function lookupPatientInDb(pipeline:MessageContext msgCtx) returns error? {
    PatientRecord patient = check msgCtx.getContentWithType();
    boolean exists = check patientExistsInDb(patient.patientId);
    msgCtx.setProperty("patientExists", exists);  // channelMap equivalent
}
```

Read downstream with: `boolean exists = check msgCtx.getPropertyWithType("patientExists");`

### Destinations — translate Mirth destination connectors

```ballerina
@pipeline:DestinationConfig {
    id: "send_to_fhir",
    retryConfig: {maxRetries: 3, retryInterval: 2}
}
isolated function sendToFhirServer(pipeline:MessageContext msgCtx) returns json|error {
    PatientRecord patient = check msgCtx.getContentWithType();
    http:Client fhirClient = check new (fhirServerUrl);
    json response = check fhirClient->/Patient.post(patient);
    // Response transformer equivalent: inspect response here before returning
    return response;
}
```

---

## Phase 8: Destination Queuing → Pipeline Configuration

Mirth has three queue modes per destination. Map them to pipeline configuration:

| Mirth Queue Mode | Behavior | Ballerina equivalent |
|---|---|---|
| **Never** | No queuing; fail fast on error | No `retryConfig` on `@pipeline:DestinationConfig`; no `replayListenerConfig` on `HandlerChain` |
| **On Failure** | Retry on error, up to retry count | `retryConfig: {maxRetries: N, retryInterval: S}` on `@pipeline:DestinationConfig` |
| **Always** | Queue every message before sending (store-and-forward) | `failureStore` + `replayListenerConfig` on `HandlerChain`; all destinations receive messages via replay |

**Mirth queue settings → pipeline config:**

| Mirth setting | Ballerina mapping |
|---|---|
| Retry Count Before Queue | `retryConfig.maxRetries` on `@pipeline:DestinationConfig` |
| Retry Interval (ms) | `retryConfig.retryInterval` (in seconds) on `@pipeline:DestinationConfig` |
| Queue Buffer Size / Queue Threads | No direct equivalent — pipeline destinations run concurrently by default |
| Rotate Queue | `replayListenerConfig` naturally rotates via polling — failed messages are retried in order |
| Max Retry Count (replay) | `replayListenerConfig.maxRetries` on `HandlerChain` |
| Dead Letter (exhausted retries) | `replayListenerConfig.deadLetterStore` on `HandlerChain` |

**Queue on Response** (Mirth re-queues based on the response transformer's `responseStatus`): translate to conditional error return inside the destination function — return an `error` to trigger the retry mechanism:

```ballerina
@pipeline:DestinationConfig {id: "hl7_sender", retryConfig: {maxRetries: 5, retryInterval: 3}}
isolated function sendHl7WithAckCheck(pipeline:MessageContext msgCtx) returns hl7v2:Message|error {
    hl7v2:Message msg = check msgCtx.getContentWithType();
    hl7v2:Message|hl7v2:HL7Error ack = hl7Client->sendMessage(msg);
    if ack is hl7v2:HL7Error {
        return error("HL7 send failed: " + ack.toString());
    }
    // Re-queue on AE equivalent: inspect ACK and return error to trigger retry
    hl7v23:ACK typedAck = check ack.ensureType(hl7v23:ACK);
    string ackCode = typedAck.msa?.msa1 ?: "";
    if ackCode == "AE" {
        string errMsg = typedAck.msa?.msa3 ?: "Application Error";
        return error("Application Error NACK received: " + errMsg);  // triggers retryConfig
    }
    return ack;
}
```

---

## Phase 9: MLLP Sender (TCP Dispatcher)

```ballerina
import ballerinax/health.clients.hl7;
import ballerinax/health.hl7v2;

configurable string destHost = "downstream.example.com";
configurable int destPort = 2575;

final hl7:HL7Client hl7Client = check new (destHost, destPort);

// HL7Client handles MLLP framing automatically — do not wrap bytes manually.
@pipeline:DestinationConfig {id: "send_hl7_downstream", retryConfig: {maxRetries: 3, retryInterval: 2}}
isolated function sendToDownstream(pipeline:MessageContext msgCtx) returns hl7v2:Message|error {
    hl7v2:Message msg = check msgCtx.getContentWithType();
    hl7v2:Message|hl7v2:HL7Error ack = hl7Client->sendMessage(msg);
    if ack is hl7v2:HL7Error {
        return error("HL7 send failed: " + ack.toString());
    }
    return ack;
}
```

---

## Phase 10: Translating JavaScript — Three Cases

### Case A: Simple / directly translatable JS

Translate to idiomatic Ballerina:

```javascript
// channelMap.put('sendingApp', msg['MSH']['MSH.3']['MSH.3.1']);
```
→ `msgCtx.setProperty("sendingApp", adt.msh.msh3?.hd1 ?: "");`

```javascript
// var dateStr = DateUtil.formatDate(val, 'yyyyMMdd', 'yyyy-MM-dd');
```
→
```ballerina
// TODO: verify this matches DateUtil.formatDate behavior (timezone handling may differ)
string formatted = convertHl7Date(rawDate);
```

### Case B: Moderate complexity — typed helper stub

When the logic is non-trivial but inputs and outputs are clear, write a full typed signature with a comment block describing the contract precisely:

```ballerina
// Translates Mirth JS step "Normalize Patient Name" (transformer step 3):
// - Trims leading/trailing whitespace from each name component
// - Capitalizes first letter of each component; handles hyphenated names (e.g. "smith-jones" → "Smith-Jones")
// - Returns empty string for null/empty input
// Input: raw name string from PID.5 as received (may be all-caps or mixed case)
// Output: display-ready formatted name string
isolated function normalizePatientName(string rawName) returns string {
    // TODO: implement — see original Mirth JS "Normalize Patient Name" in transformer step 3
    return rawName;
}
```

### Case C: Complex / stateful JS — typed stub with full contract description

When the JS uses Mirth-specific globals, Java classes, cross-message state, or external integrations, **do not attempt a partial translation**. Generate a properly typed stub whose comment block fully describes expected behavior, inputs, outputs, error conditions, and original implementation caveats:

```ballerina
// Translates Mirth JS step "MRN Lookup and Dedup" (transformer step 5):
//
// Expected behavior:
// - Queries the MPI (Master Patient Index) using patientId + dateOfBirth as the lookup key
// - If a canonical MRN is found, returns it; otherwise generates a new UUID-based MRN
// - Deduplication: if two records share DOB + last name + first initial, merges and returns
//   the older record's MRN
// - Logs all lookups to the audit table regardless of hit/miss
//
// Original used: DatabaseConnectionFactory.getConnection('mpi_db'), globalChannelMap for caching
// Ballerina approach: use a module-level sql:Client; replace globalChannelMap cache with
//   a module-level isolated map<string> with lock{} (see globalChannelMap caveat in Phase 5)
//
// Parameters:
//   patientId   - raw PID.3.1 value (may be empty string — handle gracefully)
//   dateOfBirth - HL7 date string yyyyMMdd format (e.g. "19801215")
//   lastName    - PID.5.1 family name component
//   firstName   - PID.5.2 given name component
// Returns: canonical MRN string — never empty (generates new UUID if no match found)
// Errors: propagate sql:Error from DB; caller should let error bubble to pipeline failure store
isolated function lookupOrAssignMrn(string patientId, string dateOfBirth,
                                     string lastName, string firstName) returns string|error {
    // TODO: implement — see original Mirth JS "MRN Lookup and Dedup" in transformer step 5
    return uuid:createType1AsString();
}
```

**Use Case C whenever the JS step involves:**
- `DatabaseConnectionFactory` / any JDBC call
- `globalChannelMap` / `globalMap` with cross-message semantics
- Java class instantiation (`Packages.com.mirth.*`, `java.util.*`)
- `AttachmentUtil`, `VMRouter`, `router.routeMessage()`
- Regex with Java-specific flags or lookaheads that differ from Ballerina regex
- `DateUtil` date arithmetic where timezone behavior is load-bearing

### Mirth global variables → Ballerina

| Mirth JS variable | Ballerina equivalent |
|---|---|
| `msg` (E4X XML object) | Typed HL7 record via `msgCtx.getContentWithType()` |
| `tmp` | Local variable in the handler function |
| `channelMap` / `$c` | `msgCtx.setProperty()` / `getPropertyWithType()` |
| `connectorMap` / `$co` | Local variable scoped to the current handler function |
| `sourceMap` / `$s` | Properties set on `MessageContext` before pipeline entry, or fields in the content record |
| `responseMap` / `$r` | `ExecutionSuccess.destinationResults[destinationId]` |
| `globalChannelMap` / `$gc` | Module-level `isolated map<anydata>` with `lock{}` + TODO caveat |
| `globalMap` / `$g` | Module-level shared state with `lock{}` |
| `configurationMap` / `$cfg` | `configurable` Ballerina variables |
| `DateUtil`, `UUIDGenerator` | `ballerina/time`, `ballerina/uuid` |
| `logger.info(...)` | `log:printInfo(...)` |
| `router.routeMessage(...)` | Pipeline destination or `start destinationFn(...)` |

---

## Phase 11: Configuration (Config.toml)

Always externalize connection parameters. Never hardcode hosts, ports, credentials, or paths in `.bal` files.

```toml
mllpListenPort = 2575
destinationHost = "localhost"
destinationPort = 2576
outputDirectory = "./output"

[db]
host = "localhost"
port = 3306
user = "dbuser"
password = ""   # set via environment: BAL_CONFIG_SECRET_db_password
database = "mirthdb"
```

For database `DbConfig` records, annotate the password field:

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

## Phase 12: Error Handling

The pipeline `failureStore` handles routing of failed messages. Within individual handlers use `do...on fail` for grouped operations that should be treated as a unit:

```ballerina
@pipeline:TransformerConfig {id: "enrich_and_validate"}
isolated function enrichAndValidate(pipeline:MessageContext msgCtx) returns EnrichedRecord|error {
    do {
        hl7v2:Message msg = check msgCtx.getContentWithType();
        hl7v23:ADT_A01 adt = check msg.ensureType(hl7v23:ADT_A01);
        EnrichedRecord enriched = check buildEnrichedRecord(adt);
        check validateEnrichedRecord(enriched);
        return enriched;
    } on fail error e {
        log:printError("Enrichment failed", 'error = e);
        return e;
    }
}
```

---

## Output Format

After analyzing the channel XML, output **all files in sequence** using labeled code blocks:

```
### Ballerina.toml
### types.bal
### handlers.bal
### service.bal
### Config.toml
```

After all files, include a **Migration Notes** section with these subsections:

1. **Stubs to implement** — each Case B/C stub function: its name, the original Mirth step it replaces, and what the developer must implement
2. **Assumptions & Gaps** — only list items where essential behavior is truly missing from the XML and a safe configurable default was used; keep this list short and precise
3. **Configuration** — values the user must fill in (hosts, ports, credentials, queue DSN, etc.)
4. **Mirth behaviors without direct equivalents** — persistent channel maps, attachment handling, message re-queuing, Channel Writer cross-channel routing — with the suggested Ballerina approach
5. **How to run** — `bal run`

---

## Reference: Mirth Transformer Step Types

| Mirth step type | Ballerina approach |
|---|---|
| `MessageBuilderStep` | Record field assignment in a `TransformerConfig` |
| `MapperStep` | Direct assignment or `match` expression |
| `IteratorStep` | `foreach` loop; nested iterators → nested `foreach` |
| `JavaScriptStep` (simple) | Translate directly (Case A) |
| `JavaScriptStep` (moderate) | Typed stub with contract comment (Case B) |
| `JavaScriptStep` (complex/stateful) | Typed stub with full contract description (Case C) |
| `ExternalStep` (DB lookup) | `sql:Client` parameterized query in a `ProcessorConfig` |
| `DestinationSetFilterStep` | `msgCtx` property + per-destination `FilterConfig` |
| `RuleBuilderStep` (filter) | `FilterConfig` boolean function |
| `JavaScriptRule` (filter) | `FilterConfig` boolean function |

## Reference: Common Mirth JS Patterns

```javascript
// Check message type
msg['MSH']['MSH.9']['MSH.9.1'].toString() == 'ADT'
// → adt.msh.msh9?.cm_msg1 == "ADT"

// Get patient ID
msg['PID']['PID.3']['PID.3.1'].toString()
// → adt.pid?.pid3?[0]?.cx1 ?: ""

// channelMap.put / get
channelMap.put('key', value)  /  channelMap.get('key')
// → msgCtx.setProperty("key", value)  /  msgCtx.getPropertyWithType("key")

// connectorMap (destination-local)
connectorMap.put('key', value)
// → local variable within the destination handler function

// globalChannelMap (cross-message, in-memory)
globalChannelMap.put('key', value)  /  globalChannelMap.get('key')
// → module-level isolated map<anydata> with lock{}  (emit Case C TODO caveat)

// sourceMap (read-only, injected by source)
sourceMap.get('originalFilename')
// → pass as field in content record to execute(), or initial msgCtx property

// responseMap (destination responses)
responseMap.get('dest1')
// → result.destinationResults["dest1"] after channelPipeline.execute() returns

// Date formatting
DateUtil.formatDate(dateStr, 'yyyyMMdd', 'yyyy-MM-dd')
// → Case A if format is known; Case C stub if timezone behavior is load-bearing

// Re-queue on AE ACK
responseStatus = QUEUED
// → return error(...) from destination function to trigger retryConfig
```
