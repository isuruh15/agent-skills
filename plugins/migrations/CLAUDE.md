# migrations Plugin — Agent Conventions

## Skills in this Plugin

- [`skills/mirth-to-ballerina/`](./skills/mirth-to-ballerina/SKILL.md) — migrate a Mirth Connect channel to a compilable Ballerina project

## mirth-to-ballerina Skill

Twelve-phase migration, always producing `Ballerina.toml` + `Config.toml` + `types.bal` + `handlers.bal` + `service.bal` (only the files a given channel actually needs):
1. **Analyze** — parse the channel XML, map connector classes to Ballerina listener/client types
2. **Model the flow** — every channel becomes an `xlibb/pipeline` `HandlerChain` (processors, filters, transformers, destinations)
3. **Translate variable maps** — the seven Mirth maps (`channelMap`, `sourceMap`, `globalChannelMap`, etc.) map to `MessageContext` properties, content-record fields, or module-level `isolated` state
4. **Translate JavaScript** — three tiers: direct translation, typed stub with contract comment, or typed stub with full contract description for complex/stateful logic — never a partial translation
5. **Externalize config** — hosts, ports, credentials always go in `Config.toml`, never hardcoded

## Key Reference Files

This skill is self-contained — no `references/`, `scripts/`, or `assets/` siblings. All connector mappings, queue-mode tables, and JS-translation examples live inline in `SKILL.md`.
