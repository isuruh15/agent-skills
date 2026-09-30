# healthcare Plugin

Agent skills for writing services and integrations in Ballerina for healthcare applications,
including HL7v2 and FHIR support.

## Skills

| Skill | Triggers |
|-------|----------|
| **mirth-to-ballerina** | Migrate a Mirth Connect channel XML to a compilable Ballerina project; implement HL7v2 MLLP/LLP listeners or TCP senders; translate Mirth JavaScript transformers, filters, or channel scripts to Ballerina; verify a migrated channel against mock connectors and report on accuracy, latency and token usage |

## Getting Started

See the [installation instructions](../../README.md#installation) in the main README to install this plugin. Once installed, just share a channel XML or describe what you want in plain language — the skill will pick up the request. Below are some prompts to try.

### Migrating a Mirth Channel

```
> Migrate this Mirth Connect channel.xml to a Ballerina project.
```
```
> I have an HL7v2 MLLP channel in Mirth — convert it to Ballerina Integrator.
```
```
> Here's a Mirth channel with a JavaScript transformer that looks up a patient in a DB — translate it to Ballerina.
```

The skill parses the channel's source connector, destinations, filters, transformers, and lifecycle scripts, then produces a complete Ballerina project (`Ballerina.toml`, `Config.toml`, `types.bal`, `activities.bal`, `workflow.bal`, `service.bal`) using the `ballerina/workflow` module (Temporal-backed durable orchestration) to model the message flow. JavaScript logic that can't be translated directly is emitted as a typed stub with a full contract comment describing exactly what to implement, so nothing is silently dropped.

Before writing any of that code, the skill also generates a mock service for every source and
destination connector in the channel, plus a scenario suite whose expected outputs are derived from
the original Mirth XML rather than from the generated Ballerina. The final phase stands the project
up against those mocks, runs every scenario, and writes a verification report covering accuracy,
latency and the agent's own token usage, with each tested scenario listed and anything it could not
verify named explicitly.
