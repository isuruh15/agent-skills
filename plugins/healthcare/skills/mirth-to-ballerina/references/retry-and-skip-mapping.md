# Destination Queue Mode → Human Review + Skip-on-Failure Mapping

Consulted during Phase 8 (Destination Queuing) and Phase 12 (Error Handling) of `SKILL.md`.

**This project convention overrides the "choose a retry mechanism per queue mode" approach you
might expect from a Mirth migration.** Every `ctx->callActivity()` call in every generated project
always uses a human-review `retryPolicy` (a `ReviewTaskDefinition` — `{userRoles: ..., title:
...}`), never `AutoRetry` and never the default `NoAutomaticRetry`. Mirth's queue mode therefore no
longer selects *how the engine retries* — it informs two things instead: the critical/non-critical
classification (Phase 12, Step 1) and the `title`/`userRoles` written into the review task.

## Mirth Queue Mode → what it now informs

| Mirth Queue Mode | Original behavior | What it maps to here |
|---|---|---|
| **Never** | No queuing; fail fast on error | Usually **critical** — `check` the result. On failure a review task is still raised (`userRoles`/`title`) before the workflow is allowed to fail, giving a person one chance to intervene that Mirth's "fail fast" never offered |
| **On Failure** | Retry on error, up to a retry count, then give up | Usually **non-critical** — captured as `T\|ConnectionError\|ExecutionError`. On failure a review task is raised; once resolved without a successful rerun, the step is recorded in `skippedSteps` rather than failing the workflow |
| **Always** | Queue every message before sending (store-and-forward, retried indefinitely / until manually resolved) | Usually **critical**, with a `title` that reflects the urgency ("blocks downstream delivery until resolved"). The workflow engine's own durable event history already guarantees the activity call is attempted to completion or explicit failure across restarts — no separate store is needed, and a raised review task is the "manually resolved" step Mirth's queue would otherwise require ops to reach into the UI for |

## Writing the `ReviewTaskDefinition` for a given destination

| Field | Guidance |
|---|---|
| `userRoles` | Pick per the destination's operational owner. A reasonable default set: `"OPS"` for infrastructure-facing steps (DB writes, audit logs, internal queues), `"MANAGER"` for business-facing sends (payment, primary downstream system, anything a business stakeholder would want to know about immediately). Carry over an explicit owner from the Mirth channel's naming/comments if one is evident. |
| `title` | Short, specific, incident-namable. Prefer `"Failure in HL7 Sender"` or `"Failure sending patient to FHIR server"` over a generic `"Activity failed"` — the title is what a reviewer sees first in their task queue. |

```ballerina
// CRITICAL destination, Mirth queue mode "Always" — high urgency, ops-owned.
string ackResult = check ctx->callActivity(sendHl7WithAckCheck, {"messageJson": normalized},
        stepId = "send_hl7_downstream",
        retryPolicy = {userRoles: "OPS", title: "Failure in HL7 Sender — blocks downstream delivery"});

// NON-CRITICAL destination, Mirth queue mode "On Failure" — a secondary audit copy.
string|ConnectionError|ExecutionError auditResult = ctx->callActivity(writeAuditLog,
        {"orderId": input.orderId}, stepId = "write_audit_log",
        retryPolicy = {userRoles: "OPS", title: "Failure writing audit log"});
```

## Queue on Response

Mirth re-queues based on the response transformer's `responseStatus`. Translate to a conditional
error return inside the activity function — returning `ConnectionError` or `ExecutionError`
triggers the review task on the calling `ctx->callActivity()`, same as any other activity failure:

```ballerina
@workflow:Activity
isolated function sendHl7WithAckCheck(json messageJson) returns hl7v2:Message|ConnectionError|ExecutionError {
    hl7v2:Message|error msg = hl7v2:parse(messageJson.toString());
    if msg is error {
        return error ExecutionError("Could not re-parse HL7 message before send", msg);
    }
    hl7v2:Message|hl7v2:HL7Error ack = hl7SenderClient->sendMessage(msg);
    if ack is hl7v2:HL7Error {
        return error ConnectionError("HL7 send failed", ack);
    }
    hl7v23:ACK typedAck = check ack.ensureType(hl7v23:ACK);
    string ackCode = typedAck.msa?.msa1 ?: "";
    if ackCode == "AE" {
        string errMsg = typedAck.msa?.msa3 ?: "Application Error";
        // Rejected by the downstream application, not a connection problem — ExecutionError,
        // which raises this destination's review task with the AE reason in the error detail.
        return error ExecutionError("Application Error NACK received: " + errMsg);
    }
    return ack;
}
```

## Queue Buffer Size / Queue Threads / Rotate Queue

These Mirth settings control queue-worker concurrency and ordering within a single destination's
queue. Because destinations in this design are sequential `ctx->callActivity()` calls rather than
a queue with worker threads, there's no direct equivalent — note this as a gap in the Migration
Notes if the original channel relied on queue-thread parallelism for throughput within one
destination (as opposed to across destinations, which Phase 4/12 already address by design).

## Resolving a raised review task

A review task raised by a failing activity is resolved through `ballerina/workflow`'s general
human-task completion surface — `workflow:completeHumanTask()`, with pending tasks discoverable via
`management:listPendingHumanTasks(workflowId)`. The exact decision payload shape for a *review*
task specifically (rerun / rerun with edited input / fail) is not detailed in the reference
material this skill was authored from; confirm it against the live module's reference before
wiring it into generated `service.bal` management endpoints or `tests/workflow_test.bal` test
cases, rather than guessing at field names.
