# Error Handling — Full Worked Example (workflow-based)

Consulted during Phase 12 (Error Handling) of `SKILL.md`. This is the full sequential
`ctx->callActivity()` pattern — critical destinations `check`'d, non-critical destinations
captured as `T|error` and skipped — referenced from Step 3 of Phase 12.

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

## Notes that make this correct, not just plausible-looking

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

## The two error types — declarations and usage

Consulted from "Error Types — Required in Every Project" in `SKILL.md`. Every project declares
exactly these two in `types.bal` and no others:

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
these two, never a bare `error(...)` — see `references/activity-examples.md` for the pattern.

Why this matters beyond naming: a reviewer looking at a raised human-review task (Phase 7/12) sees
the error type and its `detail()` fields as part of the task context, so `ConnectionError` vs.
`ExecutionError` is the first thing that tells them whether "retry once the network's back" or
"the payload itself needs a human decision" is the right call. Keep the distinction meaningful —
don't default everything to `ExecutionError` for convenience.

`ctx->callActivity()`'s inferred result type follows whatever the activity declares, so workflow
code binds to the union directly, e.g.
`string|ConnectionError|ExecutionError result = ctx->callActivity(...)`, or narrows with
`result is ConnectionError` / `result is ExecutionError` when the two need different handling
before falling through to the shared skip/fail logic in Phase 12.
