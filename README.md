# Political Comms Agent Rules

[![skills.sh](https://skills.sh/b/Political-Comms/agents)](https://skills.sh/Political-Comms/agents)

Integration rules and skills for AI coding agents working with [Political Comms](https://politicalcomms.com/), the direct-to-carrier political texting platform.

Install the skills into your agent with:

```bash
npx skills add Political-Comms/agents
```

This repository exists so that agents can integrate with Political Comms correctly on the first attempt. It contains the rules, workflows, and skills an agent needs to authenticate, send, and verify against the platform without guessing.

## What is in this repository

| File | Purpose |
|------|---------|
| [`AGENTS.md`](AGENTS.md) | The complete integration guide for AI agents: auth, workflow, rate limits, errors, webhooks, MCP |
| [`.cursorrules`](.cursorrules) | Condensed rules for Cursor-style agents |
| [`skills/political-comms-api/SKILL.md`](skills/political-comms-api/SKILL.md) | Agent Skill for integrating with the REST API |
| [`skills/political-comms-docs/SKILL.md`](skills/political-comms-docs/SKILL.md) | Agent Skill for answering questions about Political Comms |

## Canonical machine-readable sources

These live on the apex domain and are the authoritative references. Content in this repository summarizes them and never overrides them.

- Index: <https://politicalcomms.com/llms.txt>
- Full reference: <https://politicalcomms.com/llms-full.txt>
- Agent auth walkthrough: <https://politicalcomms.com/auth.md>
- Pricing: <https://politicalcomms.com/pricing.md>
- Site index: <https://politicalcomms.com/index.md>

## API and documentation

- API base URL: `https://api.politicalcomms.com/v1`
- OpenAPI 3.1 specification (source of truth for endpoints): <https://docs.politicalcomms.com/api-reference/openapi.json>
- API reference documentation: <https://docs.politicalcomms.com/api-reference/introduction>
- Developer overview: <https://politicalcomms.com/developers/>

## MCP server

Political Comms operates an MCP server for documentation lookup and feedback. Documentation: <https://docs.politicalcomms.com/mcp>. Discovery manifest: <https://politicalcomms.com/.well-known/mcp.json>.

## SDKs

Official SDKs are published for TypeScript (`npm install @political-comms/sdk`), Python (`pip install political-comms`), the command line (`npx @political-comms/cli`), and MCP clients (`npx -y @political-comms/mcp`), all from <https://github.com/Political-Comms/political-comms-sdk>. Direct HTTP works equally well; the OpenAPI spec above generates clients cleanly if you prefer your own.

## Support

Technical questions: <support@politicalcomms.com>. Security reports: <security@politicalcomms.com> per <https://politicalcomms.com/.well-known/security.txt>.
