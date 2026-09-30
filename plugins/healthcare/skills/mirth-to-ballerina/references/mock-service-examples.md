# Mock Connector & Verification Scenario Examples

Consulted during Phase 1B (Generate Mock Connectors and the Verification Scenario) and Phase 13
(Verify Against the Mocks, and Report) of `SKILL.md`.

Everything here lives in `verification/`, which is **a separate Ballerina package from the migrated
project**. Its `Ballerina.toml` is its own; its dependencies (`ballerina/tcp`, `ballerina/http`,
`ballerinax/health.hl7v2`, …) never appear in the migrated project's `Ballerina.toml`. The
integration under test must remain shippable without any of this.

```toml
# verification/mocks/Ballerina.toml
[package]
org = "verification"
name = "<channel_name_snake_case>_mocks"
version = "1.0.0"
distribution = "2201.13.0"
```

> **Scope: these mocks always succeed.** There is no failure-injection mode, no unbinding a port to
> simulate an outage, no NACK-on-demand. The manifest they serve exercises functional behavior on
> the success path \u2014 does the right payload reach the right destination in the right order.
> Failure handling (skip-on-failure, critical propagation, the source's always-acknowledge rule) is
> verified by the in-process test suite in Phase 13.1, and belongs there. Do not add a mode switch
> to these mocks "while you're in there" \u2014 the two kinds of verification stay separate on purpose,
> and mixing them makes a red run ambiguous.

---

## The control surface

Phase 13.2 runs scenarios back to back and needs a clean recorder between each, so the mocks expose
a small admin service. The harness drives this; nothing in the migrated project ever calls it.

```ballerina
// verification/mocks/control.bal
import ballerina/http;

configurable int controlPort = 19000;

service /control on new http:Listener(controlPort) {

    // Persist the previous scenario's receipts and clear the recorder for the next one.
    resource function post reset(string scenarioId) returns http:Ok {
        resetRecorder(scenarioId);
        return http:OK;
    }

    // Everything the mocks received during the current scenario, in arrival order.
    resource function get receipts() returns Receipt[] {
        return currentReceipts();
    }

    // Readiness probe for Phase 13.2 step 2 — every mock listener is up and bound.
    resource function get health() returns http:Ok {
        return http:OK;
    }
}
```

---

## The recorder

Every mock funnels receipts through one recorder so arrival *order across destinations* is
observable — that is what `expect.destinationOrder` in a scenario is checked against, and with
sequential `ctx->callActivity()` calls (Phase 4) the order is deterministic and worth asserting.

```ballerina
// verification/mocks/recorder.bal
import ballerina/time;

public type Receipt record {|
    string destination;     // logical name; matches the stepId used at the callActivity site
    int sequence;           // monotonic arrival order within the scenario
    string receivedAt;      // RFC 3339 — the end of the end-to-end latency measurement
    string payload;         // exactly what arrived, unparsed
|};

isolated int sequenceCounter = 0;
isolated Receipt[] receipts = [];

public isolated function recordReceipt(string destination, string payload) {
    lock {
        sequenceCounter += 1;
        receipts.push({
            destination: destination,
            sequence: sequenceCounter,
            receivedAt: time:utcToString(time:utcNow()),
            payload: payload
        });
    }
}

public isolated function currentReceipts() returns Receipt[] {
    lock {
        return receipts.clone();
    }
}

public isolated function resetRecorder(string scenarioId) {
    lock {
        // Persist before clearing — Phase 13.2 requires the raw receipts to stay next to the
        // report so any accuracy figure is traceable back to actual received bytes.
        persist(scenarioId, receipts.clone());
        receipts = [];
        sequenceCounter = 0;
    }
}
```

---

## Destination mocks

Each destination in `<destinationConnectors>` gets a stub that speaks the same protocol as the real
system, records everything it receives, and returns the success response that system would return:

| Mirth destination connector | Mock shape | Success response it must return |
|---|---|---|
| `TcpDispatcher` (MLLP) | `tcp:Listener` that parses with `hl7v2:parse()` | an `AA` ACK carrying the inbound control id |
| `HttpDispatcher` | `http:Listener` service on the destination's path | the `2xx` and body shape the real endpoint returns |
| FHIR destination | `http:Listener` accepting the FHIR paths | the posted resource echoed back with an assigned `id` and a `Location` header |
| `FileDispatcher` | a temp output directory | the written file itself is the receipt |
| `DatabaseWriter` | a real local database, not a fake SQL surface | the inserted row |
| Email / SMTP | a capture service recording the message | a normal accept |

A mock must be as strict as the real system about what it accepts — correct MLLP framing, required
headers, the schema the endpoint enforces. A permissive mock turns a genuine defect into a green
run, which is worse than no verification at all.

### MLLP / HL7v2 destination (`TcpDispatcher`)

MLLP messages are framed `0x0B <payload> 0x1C 0x0D`, and TCP gives no guarantee that one read
equals one message — a block can arrive split across reads or coalesced with the next one. Both the
mock and the driver therefore accumulate until they see the end-of-block delimiter rather than
parsing whatever a single read returned:

```ballerina
// verification/mocks/framing.bal
public const byte MLLP_START = 0x0B;
public const byte MLLP_END = 0x1C;
public const byte MLLP_CR = 0x0D;

# Whether the buffer ends with the MLLP end-of-block delimiter.
public isolated function isCompleteMllpBlock(byte[] buffer) returns boolean {
    int len = buffer.length();
    return len >= 2 && buffer[len - 2] == MLLP_END && buffer[len - 1] == MLLP_CR;
}
```

> If the destination under test turns out not to use MLLP framing (see the caveat below), this
> completion rule never matches and every read falls through to the deadline. In that case replace
> `isCompleteMllpBlock` with the real completion rule for that transport — do not simply raise the
> timeout, which converts a wiring bug into a slow test.

```ballerina
// verification/mocks/mock_destinations.bal
import ballerina/log;
import ballerina/tcp;
import ballerinax/health.hl7v2;
import ballerinax/health.hl7v23;

configurable int hl7DestPort = 12576;

service on new tcp:Listener(hl7DestPort) {
    remote function onConnect(tcp:Caller caller) returns tcp:ConnectionService {
        return new MockHl7Connection();
    }
}

service class MockHl7Connection {
    *tcp:ConnectionService;

    // One instance per connection, so this buffer is per-connection state.
    private byte[] buffer = [];

    remote function onBytes(tcp:Caller caller, readonly & byte[] data) returns tcp:Error? {
        self.buffer.push(...data);
        if !isCompleteMllpBlock(self.buffer) {
            return;   // partial block — wait for the rest rather than failing to parse it
        }
        byte[] message = self.buffer;
        self.buffer = [];

        string|error payload = string:fromBytes(message);
        recordReceipt("send_hl7_downstream", payload is string ? payload : "<undecodable>");

        // The mock parses rather than blindly acking: if the migrated activity sends something
        // the real downstream could not read, the parse failure must surface as a finding here
        // rather than being papered over with an unconditional AA.
        hl7v2:Message|hl7v2:HL7Error parsed = hl7v2:parse(message);
        if parsed is hl7v2:HL7Error {
            log:printError("Mock destination could not parse inbound message", 'error = parsed);
            // Close rather than staying silent: an unanswered connection would leave the driver
            // waiting out its deadline, turning a real defect into a timeout. Closing fails the
            // scenario immediately, which is the finding we want.
            return caller->close();
        }
        string controlId = parsed is hl7v23:ADT_A01 ? parsed.msh.msh10 : "UNKNOWN";

        hl7v23:ACK ack = {
            msh: {msh9: {cm_msg1: "ACK"}, msh10: controlId, msh12: "2.3"},
            msa: {msa1: "AA", msa2: controlId}
        };
        byte[]|hl7v2:HL7Error encoded = hl7v2:encode("2.3", ack);
        if encoded is byte[] {
            check caller->writeBytes(encoded);
        } else {
            log:printError("Mock could not encode ACK", 'error = encoded);
            return caller->close();   // same reasoning — never leave the driver hanging
        }
    }

    remote function onError(tcp:Error err) {
        log:printError("Mock HL7 destination connection error", 'error = err);
    }
}
```

> Same caveat as the real listener in `references/source-connector-examples.md`: whether the MLLP
> start/end block bytes (`0x0B` … `0x1C 0x0D`) are present depends on transport configuration, and
> `hl7v23:ACK`'s `msa1`/`msa2` field names should be confirmed against the live module reference.
> **The mock must frame exactly the way the real downstream system does** — a mock that silently
> tolerates missing framing turns a genuine framing defect into a green verification run, which is
> worse than no verification at all. If framing cannot be confirmed, say so in the report's
> Findings rather than assuming it works.

### HTTP destination (`HttpDispatcher`)

```ballerina
configurable int httpDestPort = 12080;

http:Service mockHttpDest = service object {
    resource function post [string... path](@http:Payload json payload) returns http:Ok {
        recordReceipt("send_to_downstream_api", payload.toJsonString());
        return {body: {status: "received"}};
    }
};
```

Match the real endpoint's contract: if it requires a content type, an auth header or a particular
body schema, the mock enforces the same. A mock that accepts anything verifies nothing about how
the activity builds its request.

### FHIR destination

Same shape, but return what a FHIR server actually returns on create — the posted resource echoed
back with an assigned `id` and a `Location` header — so the activity's response handling is
exercised. A bare `200 {}` hides defects in how the `FHIRConnector` result is read.

### File destination (`FileDispatcher`)

No service. Point the project's output directory at `verification/mocks/out/` and have the harness
read that directory as the receipt source, recording file name, write time and contents.

### Database destination (`DatabaseWriter`)

Prefer a real local database over a fake SQL surface: the activity's SQL, parameter binding and
type mapping are exactly what needs verifying, and a hand-rolled stub verifies none of it. The
harness reads back the inserted rows as receipts.

---

## Source drivers

Drivers inject into the **real** source connector — the generated listener is never mocked, because
its framing, parsing and config wiring are part of what is being verified.

```ballerina
// verification/mocks/source_drivers.bal
import ballerina/http;
import ballerina/log;
import ballerina/tcp;
import ballerina/time;

public type DriverResult record {|
    string sentAt;          // RFC 3339 — the start of the end-to-end latency measurement
    string ack;             // "AA" for MLLP, the status code for HTTP, "OK" for file
|};

// Per-read timeout and total per-message deadline. Both are needed: the first stops one read
// blocking forever, the second stops a chatty peer from resetting the clock indefinitely.
configurable decimal ackReadTimeout = 5;
configurable decimal ackDeadline = 15;

public function driveMllpSource(string host, int port, byte[] framedMessage)
        returns DriverResult|error {
    string sentAt = time:utcToString(time:utcNow());
    tcp:Client sender = check new (host, port, timeout = ackReadTimeout);

    // Ballerina has no defer/finally, so the result is BOUND rather than `check`ed: a `check`
    // here would return early and leak the socket. Close unconditionally, then act on the result.
    byte[]|error ack = sendAndReadAck(sender, framedMessage);
    tcp:Error? closed = sender->close();
    if closed is tcp:Error {
        log:printWarn("Driver could not close MLLP connection", 'error = closed);
    }
    if ack is error {
        return ack;
    }
    return {sentAt: sentAt, ack: extractAckCode(ack)};
}

isolated function sendAndReadAck(tcp:Client sender, byte[] framedMessage) returns byte[]|error {
    check sender->writeBytes(framedMessage);

    byte[] buffer = [];
    time:Utc deadline = time:utcAddSeconds(time:utcNow(), ackDeadline);

    while true {
        if time:utcDiffSeconds(deadline, time:utcNow()) <= 0d {
            return error(string `Timed out after ${ackDeadline}s waiting for a complete MLLP ACK`);
        }
        // Errors on the per-read timeout, so a server that holds the connection open — normal for
        // MLLP — cannot block the harness indefinitely.
        readonly & byte[] chunk = check sender->readBytes();
        if chunk.length() == 0 {
            return error("Connection closed before a complete MLLP ACK was received");
        }
        buffer.push(...chunk);
        if isCompleteMllpBlock(buffer) {
            return buffer;
        }
    }
}

public function driveHttpSource(string url, json payload) returns DriverResult|error {
    string sentAt = time:utcToString(time:utcNow());
    http:Client c = check new (url, timeout = ackDeadline);
    http:Response res = check c->post("/api/v1/messages", payload);
    return {sentAt: sentAt, ack: res.statusCode.toString()};
}
```

Three failure modes this shape turns into fast, reportable errors instead of a hung run: the
destination never answers (per-read timeout), it answers slowly forever without ever completing a
block (deadline), and it closes mid-ACK (zero-length read). All three surface as a failed scenario
with a message naming the cause — which is what Phase 13.2 wants, since on an all-success manifest
any of them is a defect in the generated project.

> Confirm `tcp:ClientConfiguration`'s timeout field name and whether it applies to reads against
> the live `ballerina/tcp` reference before relying on it. If it turns out not to bound reads, the
> deadline check alone is not enough — a single `readBytes()` would still block past it — and the
> read needs to move onto a worker the driver can abandon. Do not ship the harness assuming the
> timeout works without having seen it fire.

For a File Reader source, the driver writes the input file into the watched directory and records
the write timestamp; there is no ack to capture, so the scenario's `expect.sourceAck` is `"OK"`.

---

## `scenarios.json` schema

```jsonc
{
  "scenarios": [
    {
      "id": "SC-003",
      "description": "ADT^A08 update — routes to the FHIR destination only, audit filter excludes it",
      "source": "mllp_listener",
      "input": "inputs/adt-a08.hl7",

      "expect": {
        "sourceAck": "AA",
        "workflowStatus": "COMPLETED",
        "skippedSteps": [],                       // always empty — every mock accepts
        "destinationOrder": ["send_to_fhir"],
        "destinations": {
          "send_to_fhir": {
            "calls": 1,
            "fields": {
              "resourceType": "Patient",
              "identifier[0].value": "12345",
              "name[0].family": "DOE",
              "name[0].given[0]": "JANE"
            }
          },
          "write_audit_log": { "calls": 0 }        // destination filter excludes A08
        }
      },

      "unverifiable": [
        { "field": "extension[0].valueString",
          "reason": "Case C stub `enrichFromLegacyCache` — not derivable from the channel XML" }
      ]
    }
  ]
}
```

There is deliberately **no** `destinationModes`, `reviewDecision`, or `errorTypes` field. If one
appears in a generated manifest, it has drifted from this design — the failure paths those fields
would drive belong in `tests/workflow_test.bal` (Phase 13.1).

**Field addressing in `fields`** uses HL7 `SEG-field[.component]` notation for HL7 payloads and
JSONPath-style dotted keys for JSON and FHIR payloads. Assert only fields the Mirth transformer
actually sets or copies — asserting a field the channel never touches adds a denominator without
adding coverage, which quietly deflates the accuracy figure for no benefit.

---

## Minimum scenario coverage

All of these must exist, because each pins a behavior the channel actually has. Coverage here comes
from *inputs*, not from injected faults, so this table is the only lever for widening what the run
proves:

| Scenario | What it proves |
|---|---|
| Happy path, one per distinct message type the channel handles | The transform is faithful to the Mirth transformer |
| Each source filter branch — one message that passes, one that is dropped | A dropped message returns `FILTERED` with zero destination calls (Phase 7) |
| Each destination filter branch | The destination records a receipt when its rule passes, and none when it does not |
| Each conditional path in the transformer — optional segment present and absent, repeating segments, each branch of a switch on message type or event | Field mapping is correct under every shape the channel branches on, not just the sample message |
| A run where every destination is exercised at least once | No destination is silently never called — a mapping that was never invoked was never verified |
| Boundary data — maximum-length fields, repeating segments, non-ASCII characters | Encoding, escaping and repetition survive the translation |

When the channel's routing depends on message content, one scenario per routing outcome is the
floor, not a nice-to-have: an accuracy figure computed from one happy-path message says almost
nothing about a channel with three branches.

**`unverifiable` entries are mandatory where they apply.** Anything whose expected value depends on
a Case B/C JavaScript stub (Phase 10) or on a gap documented in the Migration Notes goes there with
the stub's name. These are excluded from the accuracy denominator and reported separately — never
guessed at, and never quietly counted as a pass.

---

## The run procedure (Phase 13.2)

1. **Build both packages** — `bal build` the migrated project, and `bal build verification/mocks`.
   A compile failure is itself the first reportable finding: record it, fix it, and start again.
   Never report metrics for a project that did not build.
2. **Start the mocks**, then wait for readiness by polling each mock's port until it accepts a
   connection, with a bounded timeout — not a fixed sleep. Every mock must be up before the first
   scenario runs; a destination that is not listening would fail a scenario that expects it to
   receive, which is a harness fault reported as such, not a finding about the migration.
3. **Start the migrated project pointed at the mocks** —
   `BAL_CONFIG_FILES=verification/Config.verification.toml bal run`. Configuration only. If a
   `.bal` file has to be edited to reach a mock, stop and fix the hardcoded endpoint in the project
   (Phase 11); record it as a finding rather than hand-patching the project for the run.
4. **Run the scenarios one at a time, in a declared order**, so the recorder's arrival order stays
   meaningful. For each: send the input through the source driver, capture the ack the source
   returned, then read the workflow result and the recorder's receipts. Only the input varies
   between scenarios — the mocks behave identically in all of them and raise no review tasks.
5. **Reset the recorder between scenarios**, then tear everything down at the end. A scenario that
   passes only because of a previous scenario's leftovers is worse than a failing one. Keep the raw
   recorder output next to the report so any number is traceable to received bytes.

If a scenario hangs or a workflow ends in error, that is a finding about the generated project —
diagnose it, do not reach for the review-task machinery to unstick the run.

**Give every scenario a wall-clock deadline of its own**, above the per-read timeouts in the
driver. The driver's timeouts bound one socket read; they do not bound a workflow that accepted the
message and then never called a destination, which would otherwise leave step 4 waiting on receipts
that never arrive. On expiry, record the scenario as failed with what the recorder had received so
far, then move to the next one — a harness that cannot finish produces no report, and no report is
strictly worse than a report with one red row in it.

---

## Notes

- **Derive expectations from the channel XML, not from the generated code.** If an expected value
  can only be produced by reading `activities.bal`, it is not verifying the migration — it is
  restating it. Where the XML genuinely does not determine the value (a Case B/C stub, an upstream
  Channel Writer variable), that field belongs in `unverifiable`, not in `fields`.
- **Coverage comes from inputs, not from failure modes.** Since nothing is injected, the only way
  to broaden what this run proves is more and better input messages: each routing branch, each
  optional segment present and absent, repeating segments, boundary lengths, non-ASCII names. A
  manifest with one happy-path message and a 100% score has verified very little, and the report
  should not let that read as a strong result.
- **Ports:** allocate mocks from one declared block well clear of the project's own listeners
  (the examples here use `12xxx`/`19000` against a project on `2575`/`8090`). Record the block in
  `verification/mocks/Config.toml` so a port clash is diagnosable rather than mysterious. That file
  holds the ports and the driver timeouts and nothing else: there is no per-destination behaviour to
  configure, because the mocks have none to vary.
- **Determinism:** no wall-clock-dependent expectations, no assertions on generated control IDs or
  timestamps, and no reliance on a real network. A flaky verification run is worse than none — it
  trains the reader to ignore the report.
- **A hang is a finding.** Every mock accepts, so no activity should fail and no review task should
  be raised. If a scenario stalls, something failed that was expected to succeed: diagnose it as a
  defect rather than reaching for `workflow:completeHumanTask()` to unstick the run. (That call, and
  the caveat about its unconfirmed decision payload, stays confined to the Phase 13.1 tests — see
  `references/test-scenario-examples.md`.) Nothing in the harness may wait unbounded: every socket
  read carries a timeout, every message read carries a deadline, and every scenario carries a
  wall-clock limit, so "it hung" always arrives as a named failure rather than a run that never
  returns.
- The mocks package is **never** added to the migrated project's `Ballerina.toml`, and nothing in
  `verification/` is imported by project code. If the project needs a change to be verifiable, the
  change belongs in its configuration (Phase 11), not in a test hook compiled into the shipped
  integration.
