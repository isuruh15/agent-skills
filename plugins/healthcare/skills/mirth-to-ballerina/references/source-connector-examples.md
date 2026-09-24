# Source Connector Examples (workflow-based)

Consulted during Phase 3 (Source Connector) of `SKILL.md`. Full listener code for each connector
type — Phase 3 in `SKILL.md` has the conventions these examples follow (never return an error from
the `resource`/`remote` function; the listener's job ends at "durably accepted," not "durably
processed").

All three examples share this helper, defined once per project:

```ballerina
// Never called from inside workflow code — this observes a workflow from the outside.
isolated function logWorkflowOutcome(string workflowId) {
    anydata|error result = workflow:getWorkflowResult(workflowId);
    if result is error {
        log:printError("Workflow ended in error", 'error = result, workflowId = workflowId);
    }
}
```

## MLLP / HL7v2 listener

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
```

> `hl7v2:Message` is not itself `anydata` in a form `workflow:run()` can accept directly — the
> workflow input parameter must be a subtype of `anydata` (`WORKFLOW_101`). Convert or wrap it into
> a plain record/JSON-friendly type in `types.bal` before calling `workflow:run()`. Re-parsing
> inside the workflow (via an early pure helper, or the first activity if parsing needs external
> terminology lookups) is the usual pattern — do **not** try to smuggle a non-`anydata` object
> through as workflow input.

## HTTP source connector

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

## File Reader source connector

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
