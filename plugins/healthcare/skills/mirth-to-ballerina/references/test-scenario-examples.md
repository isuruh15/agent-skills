# Test Scenario Examples (workflow-based)

Consulted during Phase 13 (Generating a Test Scenario) of `SKILL.md`. Every migrated project from
this skill includes a runnable test that exercises the workflow end-to-end without needing a real
Temporal server, plus one test that specifically proves the skip-on-failure behavior for a
non-critical destination.

## `tests/Config.toml` — force in-memory mode for tests

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

## `tests/resources/sample-input.json` (or `.hl7`) — one representative message

Keep this small and representative rather than exhaustive — a single ADT^A01 (or the channel's
actual primary message type) with the fields the workflow's filters/transformers actually branch
on.

## `tests/workflow_test.bal` — happy path

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

## `tests/workflow_test.bal` — skip-on-failure scenario, with the human-review step in the loop

Because every destination in this design uses a human-review `retryPolicy` (Phase 7/12), a failing
activity does not resolve itself the moment the call fails — it raises a review task and the
workflow durably pauses at that step until the task is resolved. So a test proving the
skip-on-failure path has one more beat than a plain retry-based test would: drive the underlying
call to fail, then resolve the resulting review task as "fail it" (via the module's human-task
completion surface — see the caveat in `references/error-handling-example.md` about confirming the
exact decision payload shape against the live module reference before hardcoding it here), and only
then assert on the completed, skipped outcome:

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

## `tests/workflow_test.bal` — the source connector never surfaces an error

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

## Notes

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
