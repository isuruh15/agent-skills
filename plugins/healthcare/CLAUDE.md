# healthcare Plugin — Agent Conventions

## Skills in this Plugin

- [`skills/mirth-to-ballerina/`](./skills/mirth-to-ballerina/SKILL.md) — migrate a Mirth Connect channel to a compilable Ballerina project built on `ballerina/workflow`

## mirth-to-ballerina Skill

Thirteen-phase migration onto `ballerina/workflow` (Temporal-backed durable orchestration), never `xlibb/pipeline`. `Ballerina.toml` and `Config.toml` are always mandatory; `types.bal`, `activities.bal`, `workflow.bal`, and `service.bal` are the source modules included based on channel needs (`utils.bal` is added only when shared pure helpers are required):
1. **Analyze** — parse the channel XML, map connector classes to Ballerina listener/client types
2. **Model the flow** — every channel becomes one `@workflow:Workflow` function; each I/O step becomes a separate `@workflow:Activity`, called sequentially (never in parallel) via `ctx->callActivity()`
3. **Translate variable maps** — the seven Mirth maps (`channelMap`, `sourceMap`, `globalChannelMap`, etc.) collapse into ordinary local variables passed through the workflow function, except `globalChannelMap`/`globalMap` which must go through an activity to stay deterministic
4. **Translate JavaScript** — three tiers: direct translation, typed stub with contract comment, or typed stub with full contract description for complex/stateful logic — never a partial translation
5. **Externalize config** — hosts, ports, credentials always go in `Config.toml`, never hardcoded
6. **Error handling** — every activity that can fail returns one of exactly two types, `ConnectionError`/`ExecutionError`, and every `ctx->callActivity()` call always carries a human-review `retryPolicy`; non-critical destinations are skipped on failure, critical ones fail the workflow

## Key Reference Files

- `skills/mirth-to-ballerina/references/connector-activity-mappings.md` — connector class → Ballerina equivalent table, dependency-by-connector-type table, HL7v2 version selection guide
- `skills/mirth-to-ballerina/references/retry-and-skip-mapping.md` — Mirth queue-mode → human-review/skip-on-failure mapping
- `skills/mirth-to-ballerina/references/js-translation-examples.md` — worked examples for JS translation Cases A/B/C, Mirth global variable table, transformer step type table, common JS pattern translations
- `skills/mirth-to-ballerina/references/source-connector-examples.md`, `workflow-and-script-examples.md`, `activity-examples.md`, `error-handling-example.md`, `test-scenario-examples.md` — full worked code for listeners, the workflow function, activities, error handling, and generated tests
