# Connector & Dependency Mappings (workflow-based)

Consulted during Phase 1 (Analyze) and Phase 2 (Build the Ballerina.toml) of `SKILL.md`.

Same connector classes as any Mirth migration; the difference from a pipeline-based translation is
in the last column, where the Mirth destination becomes an `@workflow:Activity` rather than a
`@pipeline:DestinationConfig` function, and no `xlibb/pipeline` types are involved anywhere.

## Connector class → Ballerina equivalent

| Mirth connector class | Ballerina equivalent | Role in the workflow design |
|---|---|---|
| `com.mirth.connect.connectors.tcp.TcpReceiver` (MLLP) | `hl7v2:Hl7Listener` + `hl7v2:Hl7Service` | Source — starts a workflow via `workflow:run()` per message |
| `com.mirth.connect.connectors.tcp.TcpDispatcher` (MLLP) | `health.clients.hl7:HL7Client` | Destination — wrapped in an `@workflow:Activity` (Phase 9) |
| `com.mirth.connect.connectors.http.HttpReceiver` | `http:Listener` service | Source — starts a workflow via `workflow:run()` |
| `com.mirth.connect.connectors.http.HttpDispatcher` | `http:Client` | Destination — wrapped in an `@workflow:Activity` |
| `com.mirth.connect.connectors.file.FileReceiver` | `file:Listener` + `io` | Source — starts a workflow via `workflow:run()` |
| `com.mirth.connect.connectors.file.FileDispatcher` | `io:fileWriteString` | Destination — wrapped in an `@workflow:Activity` |
| `com.mirth.connect.connectors.jdbc.DatabaseReader` | `sql` + DB connector | Source, or a lookup step — if a lookup mid-channel, an `@workflow:Activity` |
| `com.mirth.connect.connectors.jdbc.DatabaseWriter` | `sql` + DB connector | Destination — wrapped in an `@workflow:Activity` |
| `com.mirth.connect.connectors.js.JavaScriptReader` | Ballerina service (translate logic) | Source — translated logic runs before/inside `workflow:run()`'s caller |
| `com.mirth.connect.connectors.js.JavaScriptWriter` | Ballerina function | Pure → plain function; I/O → `@workflow:Activity` |
| `com.mirth.connect.connectors.vm.VmReceiver` | `tcp:Listener` or `http:Listener` depending on data type | Source |
| `com.mirth.connect.connectors.vm.VmDispatcher` (Channel Writer) | A second `workflow:run()` call (fire-and-forget child workflow), or a direct `@workflow:Activity` call if the target logic is simple enough to inline | See note below |

**Channel Writer (VmDispatcher) note:** `ballerina/workflow` does not document a synchronous
"call one workflow from another and block for its result" primitive — composing two workflows
means an activity calling the ordinary `workflow:run()` function, which starts a second,
independently durable, fire-and-forget instance rather than a synchronous sub-call. If the
original Mirth channel relies on the downstream channel's result to decide what happens next,
call this out explicitly as a gap in the Migration Notes rather than faking synchronicity.

## Dependencies by connector type

Add to `Ballerina.toml` based on what connectors are present:

| Connector type | Dependency to add |
|---|---|
| HL7v2 MLLP listener/sender | `ballerinax/health.hl7v2`, the matching version package (e.g. `health.hl7v23`), `ballerinax/health.clients.hl7` |
| HTTP | (stdlib `ballerina/http`, no extra dep) |
| File | (stdlib `ballerina/file`, `ballerina/io`) |
| Database (MySQL) | `ballerinax/mysql` + `ballerina/sql` |
| Database (PostgreSQL) | `ballerinax/postgresql` + `ballerina/sql` |
| Email/SMTP | `ballerina/email` |
| FTP/SFTP | `ballerina/ftp` |
| HL7v2 → FHIR conversion | `ballerinax/health.hl7v2<ver>.utils.v2tofhirr4` |
| **Always include** | `ballerina/workflow`, `ballerina/log`, `ballerina/uuid`, `ballerina/time` |
| **Never include** | `xlibb/pipeline` — no pipeline object exists in this design; `ballerinax/rabbitmq` for failure/dead-letter storage — the workflow engine's own event history is the durability mechanism, so a separate message-store dependency is not needed unless the channel also uses RabbitMQ as an actual destination/source in its own right |

## HL7v2 version package selection

- `hl7v23` for HL7 2.3 / 2.3.1 — `hl7v24` for 2.4 — `hl7v25`/`hl7v251` for 2.5/2.5.1
- `hl7v26` for 2.6 — `hl7v27` for 2.7 — `hl7v28` for 2.8

If not explicit in the channel XML, check `MSH.12` in example messages. Default to `hl7v23` for
old channels.
