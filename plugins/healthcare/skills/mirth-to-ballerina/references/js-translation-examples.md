# JavaScript Translation Examples (workflow-based)

Consulted during Phase 10 (Translating JavaScript) of `SKILL.md`. Pick the tier per the criteria
below, then use these examples as a template for the comment block and signature. The tiering
itself is unchanged from a pipeline-based migration — only the `channelMap` row differs, because
this design carries state as local variables through the workflow function rather than through a
`MessageContext` object.

## Case A: Simple / directly translatable JS

Translate to idiomatic Ballerina:

```javascript
// channelMap.put('sendingApp', msg['MSH']['MSH.3']['MSH.3.1']);
```
→ `string sendingApp = adt.msh.msh3?.hd1 ?: "";` (a local variable in the workflow function, not a
map write — see the `channelMap` row below)

```javascript
// var dateStr = DateUtil.formatDate(val, 'yyyyMMdd', 'yyyy-MM-dd');
```
→
```ballerina
// TODO: verify this matches DateUtil.formatDate behavior (timezone handling may differ)
string formatted = convertHl7Date(rawDate);
```

## Case B: Moderate complexity — typed helper stub

When the logic is non-trivial but inputs and outputs are clear, write a full typed signature with
a comment block describing the contract precisely. If it's pure (no I/O), it's a plain function
called directly from the workflow function; if it needs external data, make it an
`@workflow:Activity` instead and give it the activity annotation:

```ballerina
// Translates Mirth JS step "Normalize Patient Name" (transformer step 3):
// - Trims leading/trailing whitespace from each name component
// - Capitalizes first letter of each component; handles hyphenated names (e.g. "smith-jones" → "Smith-Jones")
// - Returns empty string for null/empty input
// Input: raw name string from PID.5 as received (may be all-caps or mixed case)
// Output: display-ready formatted name string
// Pure transform — plain function, called directly from the workflow function, no callActivity.
isolated function normalizePatientName(string rawName) returns string {
    // TODO: implement — see original Mirth JS "Normalize Patient Name" in transformer step 3
    return rawName;
}
```

## Case C: Complex / stateful JS — typed stub with full contract description

When the JS uses Mirth-specific globals, Java classes, cross-message state, or external
integrations, **do not attempt a partial translation**. Generate a properly typed stub whose
comment block fully describes expected behavior, inputs, outputs, error conditions, and original
implementation caveats. Anything in this tier almost always does I/O, so it's an
`@workflow:Activity`, not a plain function:

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
// Ballerina approach: use a module-level sql:Client inside the activity function; replace
//   globalChannelMap cache with a module-level isolated map<string> with lock{}, itself only
//   ever read/written via a dedicated activity (see globalChannelMap caveat, SKILL.md Phase 5) —
//   never read/write module state directly inside @workflow:Workflow code.
//
// Parameters:
//   patientId   - raw PID.3.1 value (may be empty string — handle gracefully)
//   dateOfBirth - HL7 date string yyyyMMdd format (e.g. "19801215")
//   lastName    - PID.5.1 family name component
//   firstName   - PID.5.2 given name component
// Returns: canonical MRN string — never empty (generates new UUID if no match found)
// Errors: sql:Error from the DB is wrapped as ConnectionError (can't reach the MPI DB) or
//   ExecutionError (query ran but failed/returned something unexpected) — see "Error Types —
//   Required in Every Project" in SKILL.md. Caller decides critical vs. non-critical per Phase 12,
//   and every ctx->callActivity() call for this activity always carries a human-review
//   retryPolicy (Phase 7/12) — never AutoRetry, never left unset.
@workflow:Activity
isolated function lookupOrAssignMrn(string patientId, string dateOfBirth,
                                     string lastName, string firstName) returns string|ConnectionError|ExecutionError {
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

## Mirth global variables → Ballerina (workflow-based)

| Mirth JS variable | Ballerina equivalent |
|---|---|
| `msg` (E4X XML object) | Typed HL7 record, produced by a pure parse/extract function called early in the workflow function |
| `tmp` | Local variable in the workflow function or the relevant activity |
| `channelMap` / `$c` | **Local variable in the workflow function**, passed explicitly as an argument to whichever activity needs it — not a shared mutable context object |
| `connectorMap` / `$co` | Local variable scoped to the relevant activity function |
| `sourceMap` / `$s` | Fields on the `ChannelInput` record passed to `workflow:run()` |
| `responseMap` / `$r` | The return value of the corresponding `ctx->callActivity()` call, held in a local variable |
| `globalChannelMap` / `$gc` | Module-level `isolated map<anydata>` with `lock{}`, read/written **only via an `@workflow:Activity`** + TODO caveat (SKILL.md Phase 5) |
| `globalMap` / `$g` | Same as above, shared across channels |
| `configurationMap` / `$cfg` | `configurable` Ballerina variables |
| `DateUtil`, `UUIDGenerator` | `ballerina/time`, `ballerina/uuid` |
| `logger.info(...)` | `log:printInfo(...)` |
| `router.routeMessage(...)` | An `@workflow:Activity` that calls `workflow:run()` on a target workflow (fire-and-forget child instance — see Channel Writer note in `connector-activity-mappings.md`) |

## Reference: Mirth Transformer Step Types

| Mirth step type | Ballerina approach |
|---|---|
| `MessageBuilderStep` | Record field assignment, inline in a plain transform function |
| `MapperStep` | Direct assignment or `match` expression, inline |
| `IteratorStep` | `foreach` loop; nested iterators → nested `foreach` |
| `JavaScriptStep` (simple) | Translate directly (Case A) |
| `JavaScriptStep` (moderate, pure) | Typed stub, plain function (Case B) |
| `JavaScriptStep` (moderate, does I/O) | Typed stub, `@workflow:Activity` (Case B) |
| `JavaScriptStep` (complex/stateful) | Typed stub with full contract description, `@workflow:Activity` (Case C) |
| `ExternalStep` (DB lookup) | `@workflow:Activity` with a `sql:Client` parameterized query |
| `DestinationSetFilterStep` | A local `boolean` before the relevant `callActivity` call, per-destination |
| `RuleBuilderStep` (filter) | Plain boolean-returning function |
| `JavaScriptRule` (filter) | Plain boolean-returning function |

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
// → a local variable named `key`, declared once in the workflow function and passed to
//   whichever activities need it as a `callActivity` argument

// connectorMap (destination-local)
connectorMap.put('key', value)
// → local variable within the destination activity function

// globalChannelMap (cross-message, in-memory)
globalChannelMap.put('key', value)  /  globalChannelMap.get('key')
// → dedicated read/write activities over a module-level isolated map<anydata> with lock{}
//   (emit Case C TODO caveat; never touch this map directly from workflow code)

// sourceMap (read-only, injected by source)
sourceMap.get('originalFilename')
// → a field on the ChannelInput record passed to workflow:run()

// responseMap (destination responses)
responseMap.get('dest1')
// → the local variable holding dest1's ctx->callActivity() result

// Date formatting
DateUtil.formatDate(dateStr, 'yyyyMMdd', 'yyyy-MM-dd')
// → Case A if format is known; Case C stub if timezone behavior is load-bearing

// Re-queue on AE ACK
responseStatus = QUEUED
// → return error(...) from the activity function to trigger retryPolicy on the caller
```
