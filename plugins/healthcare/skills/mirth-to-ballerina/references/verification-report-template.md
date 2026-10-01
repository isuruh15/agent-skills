# Verification Report Template

Consulted during Phase 13.3 of `SKILL.md`. The generated report is written to
`verification/report/migration-verification-report.md`; a condensed version of the headline figures
goes into Migration Notes §7.

**The one rule that outranks the template: report only what the run actually produced.** A metric
the environment did not expose is written as "not available in this environment". An assertion the
scenarios could not check is listed under Not verified, not silently dropped from the denominator
without mention. A report whose numbers cannot be traced back to `verification/report/receipts/` is
not a verification report.

**The second rule: state the scope every time.** This run exercises success paths only — every mock
accepts. Failure handling is covered by the Phase 13.1 test suite. A reader who sees `100%` without
that context will over-read it, and the report is responsible for preventing that, not the reader.

---

## Template

````markdown
# Migration Verification Report — <channel name>

| | |
|---|---|
| Source channel | `<channel.xml>` (Mirth channel id `<id>`) |
| Generated package | `healthcare/<channel_name>` |
| Run date | `<ISO 8601>` |
| Iteration | `<n>` (see Iteration history below) |
| Workflow mode | `IN_MEMORY` (no Temporal server) |
| Mocks | `<n>` destination mocks, `<n>` source driver(s) — all accepting |
| Scope | Functional success paths only; failure handling is covered by `bal test` (Phase 13.1) |
| Raw receipts | `verification/report/receipts/` |

## Headline

| Metric | Result |
|---|---|
| Scenarios passed | `<p>/<t>` |
| Accuracy (verifiable assertions) | `<x>%` (`<matched>/<verifiable>`) |
| Assertions not verifiable | `<n>` — see Not verified |
| End-to-end latency, p50 / max | `<a> ms` / `<b> ms` |
| Agent token usage, total | `<n>` or "not available in this environment" |
| Blocking findings | `<n>` |
| `bal test` suite (Phase 13.1) | `<p>/<t>` passing — failure-path coverage |

## Tested scenarios

Every scenario below is a success path: all mock destinations accept, and each is expected to
complete with nothing skipped. What varies between them is the **input** — message type, routing
branch, optional segments, boundary data.

| ID | Description | Source | Input | Destinations exercised | Expected | Actual | Result | Accuracy | E2E latency |
|---|---|---|---|---|---|---|---|---|---|
| SC-001 | ADT^A01 admit, all destinations | MLLP | `adt-a01.hl7` | `send_hl7_downstream`, `write_audit_log` | COMPLETED, 2 receipts | COMPLETED, 2 receipts | PASS | 12/12 | 41 ms |
| SC-002 | Non-A01 message, source filter drops | MLLP | `orm-o01.hl7` | — | FILTERED, 0 receipts | FILTERED, 0 receipts | PASS | 4/4 | 9 ms |
| SC-003 | ADT^A08 update, audit filter excludes audit | MLLP | `adt-a08.hl7` | `send_to_fhir` | COMPLETED, 1 receipt | COMPLETED, 1 receipt | PASS | 11/11 | 38 ms |
| SC-004 | PID-5 absent (optional segment) | MLLP | `adt-a01-no-pid5.hl7` | `send_to_fhir` | COMPLETED, name omitted | COMPLETED, name omitted | PASS | 7/7 | 35 ms |
| SC-005 | Repeating OBX, non-ASCII patient name | HTTP | `adt-a01-utf8.json` | `send_to_fhir` | COMPLETED, 3 OBX mapped | COMPLETED, 2 OBX mapped | FAIL | 9/11 | 44 ms |

Every scenario mandated by Phase 1B.3 must appear here. If one is missing, say why in Not verified
rather than leaving the reader to notice the gap.

## Accuracy detail

### Per destination

| Destination | Critical? | Assertions | Matched | Accuracy | Notes |
|---|---|---|---|---|---|
| `send_hl7_downstream` | Critical | 18 | 18 | 100% | |
| `send_to_fhir` | Critical | 22 | 20 | 91% | Third OBX repetition dropped — see F-001 |
| `write_audit_log` | Non-critical | 6 | 6 | 100% | |

A high overall figure hiding one destination sitting well below it is the finding — this table
exists so it cannot be averaged away.

### Functional checks

Asserted per scenario, and each one is about whether the translation is faithful to the channel:

| Check | Result |
|---|---|
| Every destination the channel would have called received exactly the expected number of messages | `<n>/<n>` |
| Destination call order matched the channel's destination order | `<n>/<n>` |
| Filtered messages produced zero destination receipts | `<n>/<n>` |
| Payload fields matched the Mirth transformer's output | `<n>/<n>` |
| No scenario completed with a skipped or failed step | `<n>/<n>` |
| Mocks reached by configuration override alone, no `.bal` edits (Phase 11) | Yes / No |

## Not verified

| Assertion | Scenario | Reason |
|---|---|---|
| `extension[0].valueString` | SC-003 | Case C stub `enrichFromLegacyCache` — not derivable from the channel XML |
| Upstream-injected `sourceMap` keys | SC-006 | Channel Writer source; requires tracing the upstream channel |
| All failure-path behavior | — | Out of scope here by design; covered by the Phase 13.1 test suite |

Excluded from the accuracy denominator. This table is the honest boundary of the headline figure —
anyone reading `96%` should be able to see here exactly what that 96% does not cover.

## Latency

| Scenario | E2E (ms) | Slowest activity |
|---|---|---|
| SC-001 | 41 | `send_hl7_downstream` (28 ms) |
| SC-003 | 38 | `send_to_fhir` (26 ms) |

p50 `<a> ms`, max `<b> ms`.

> These are `IN_MEMORY`-mode figures against local mocks on one machine. They are useful for
> comparing scenarios against each other and for spotting a pathologically slow step; they are not
> a prediction of production latency, which will be dominated by the real Temporal backend, real
> network round-trips and real downstream systems.

## Token usage (migration agent)

Consumption by the agent while producing this migration — the generated Ballerina project itself
consumes no tokens.

| Stage | Tokens |
|---|---|
| Phase 1 — channel analysis | `<n>` |
| Phase 1B — mocks and scenarios | `<n>` |
| Phases 2–12 — project generation | `<n>` |
| Phase 13 — verification and repair (iteration 1) | `<n>` |
| Phase 13 — verification and repair (iteration 2) | `<n>` |
| **Total** | `<n>` |

If the session does not expose token counts, replace this whole table with
"Not available in this environment." Do not estimate: an invented number here is indistinguishable
from a measured one to everyone who reads it later.

## Findings

| ID | Severity | Scenario | Symptom | Root cause | Fix belongs in |
|---|---|---|---|---|---|
| F-001 | Major | SC-005 | Third OBX repetition missing at the FHIR destination | Transformer loops over the first two repetitions only | Generated project — `utils.bal` |
| F-002 | Minor | SC-003 | Expected `skippedSteps` naming mismatch | Scenario used the activity name, not the `stepId` | Scenario expectation |
| F-003 | Won't fix | SC-001 | Destinations arrive sequentially, not in parallel | Mirth's parallel fan-out has no equivalent by design | Migration Notes §6 |

Classify every mismatch into one of those homes. A mismatch with no root cause identified is still
listed, marked `Unresolved` — an unexplained failure is information, and dropping it because it
resisted diagnosis is the one thing that makes a report untrustworthy.

Because every mock accepts, **any** workflow error, skipped step or stalled scenario is unexpected
by construction and belongs in this table, not in the narrative as an exercised path.

## Iteration history

| Iteration | Accuracy | Scenarios passed | Blocking findings | What changed |
|---|---|---|---|---|
| 1 | 78% | 3/5 | 2 | Initial generated project |
| 2 | 96% | 5/5 | 0 | F-001 fixed |

## Reproducing this run

```bash
bal test                                                     # Phase 13.1 — failure-path coverage
bal build && bal build verification/mocks
bal run verification/mocks                                   # then wait for mock ports to accept
BAL_CONFIG_FILES=verification/Config.verification.toml bal run
# drive verification/scenarios/scenarios.json (Phase 13.2)
```
````

---

## Metric definitions

Read these before computing any number — a figure with the wrong denominator is worse than none.

**Accuracy** — assertions matched / total **verifiable** assertions. Each `expect` leaf is one
assertion: every expected field per destination, the call count, the destination order,
`skippedSteps`, the workflow status, the source ack. Entries listed under `unverifiable` are
excluded from both numerator and denominator, and listed separately by name with the stub blocking
them. Report overall, per scenario, **and per destination** — a 90% overall that conceals one
destination sitting at 40% is precisely the finding worth surfacing.

**Latency** — per scenario, driver send → last mock receipt (end-to-end); per activity from the
workflow event history where it is available; p50 and max across the run. Because every scenario is
a success path, there is no review-task pause to separate out and the figures are directly
comparable to each other. They are still `IN_MEMORY`-mode numbers against local mocks on one
machine — useful for comparing scenarios and spotting a pathologically slow step, not a prediction
of production latency, which will be dominated by the real Temporal backend, real network
round-trips and real downstream systems. State that in the report rather than leaving the reader to
assume otherwise.

**Token usage** — the *migration agent's own* consumption while producing the migration, not the
runtime's; the Ballerina project has no tokens. Break it down by stage: analysis (Phase 1), mock and
scenario authoring (Phase 1B), project generation (Phases 2–12), and verification and repair (Phase
13, per repair iteration — that is where cost actually accumulates on a hard channel). **Report only
what the session genuinely surfaces.** If the environment does not expose token counts, write "not
available in this environment" and move on — never estimate, and never present a plausible-looking
number as a measurement. This applies to every metric here: a figure the run did not produce is left
blank, not filled in.

**Tested scenarios** — one row per scenario, as in the table above. Not optional; it is what lets a
reviewer judge whether the accuracy figure covers the behavior they care about.

**Findings** — every mismatch with a root cause and where the fix belongs: a defect in the generated
project, a wrong expectation in the scenario, or a real Mirth behavior with no equivalent
(cross-reference Migration Notes §6). After repairing, re-run and report the delta across iterations
— "iteration 1: 78% → iteration 2: 96%" tells a reviewer what needed fixing, which a single final
percentage hides.

---

## Notes on filling it in

- **Accuracy is assertion-level, not scenario-level.** "4 of 6 scenarios passed" and "96% of
  assertions matched" answer different questions and both belong in the report; quoting only the
  friendlier one is a way of misleading without lying.
- **Count each `expect` leaf once.** Each expected field per destination, plus `calls`,
  `destinationOrder`, `skippedSteps`, `workflowStatus` and `sourceAck`. Do not inflate the
  denominator with assertions the channel's behavior does not actually determine — it makes a weak
  migration look better than it is.
- **Say what the score covers.** On an all-success manifest, a high accuracy figure means the
  translation is faithful for the inputs tried. It says nothing about resilience, and nothing about
  inputs nobody wrote a scenario for. Both limits belong in the report in plain words.
- **A run that did not complete is reported as such.** If the mocks would not start, the project
  would not build, or a scenario stalled, the report says that and the headline metrics read "not
  measured". A partial run presented as a full one is the single most damaging thing this document
  can contain.
- **Keep the raw receipts.** `verification/report/receipts/<scenario-id>.json` is what makes an
  accuracy number auditable months later, when the person reading the report is not the person who
  ran it.
