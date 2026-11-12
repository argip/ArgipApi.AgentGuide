# Argip API Agent Guide

Documentation written for AI coding agents (and the people who work with them) that build
integrations with the **Argip Client API**: products, stock, your prices, order calculation and
order placing for Argip business customers.

- API: `https://argipapi.argip.com.pl/`
- Interactive reference (Swagger UI): https://argipapi.argip.com.pl/swagger/ui/index.html
- Credentials: contact the Argip sales department (export@argip.com.pl)

## What is here

| Path | Content |
|---|---|
| [`AGENTS.md`](AGENTS.md) | Mandatory rules for any integration; the file agents read first |
| [`docs/`](docs) | Authentication, endpoints and models, products, quantities and prices, orders, customer data, errors, recipes |
| [`examples.md`](examples.md) | Links to working clients in C#, Python, PHP and TypeScript |
| [`openapi.json`](openapi.json) | OpenAPI 3.1 schema of the API (the docs win where it differs) |
| [`llms.txt`](llms.txt) | Index for LLM tools ([llmstxt.org](https://llmstxt.org)) |
| [`skills/argip-api/SKILL.md`](skills/argip-api/SKILL.md) | Skill with a delivery checklist |

## Use it with an agent

Point your agent at this repository (clone it, or give it the `llms.txt` URL, `https://raw.githubusercontent.com/argip/ArgipApi.AgentGuide/master/llms.txt`). `CLAUDE.md` and
`AGENTS.md` are picked up by Claude Code and other agents that read `AGENTS.md`.

To install the skill for Claude Code:

```bash
# for all your projects
cp -r skills/argip-api ~/.claude/skills/

# or for one project, from that project's root
mkdir -p .claude/skills && cp -r /path/to/ArgipApi.AgentGuide/skills/argip-api .claude/skills/
```

## The most important rules

- Cache the access token (valid 8 hours); never request one per call.
- Quantities are in pieces; prices are per 100 pieces.
- Always `Order/Calculate` before `Order/Save`, show the warnings, and inspect the `Save` result: `200` does not mean saved.
- Test order placing with an account in the testing phase; `Save` creates a real order otherwise. In the testing phase the other writes are validated but not saved (the response has a `Message` field).

The full list is in [`AGENTS.md`](AGENTS.md).

## About `openapi.json`

Snapshot of the OpenAPI document of Argip API release 12.0: OpenAPI 3.1, PascalCase property names, `servers` set to the production URL. The real JSON responses use PascalCase, as this guide and the examples do. The file contains no data.

The schema cannot express everything. Where its required/nullable flags differ from the docs, the docs
win: `NewShippingAddress.Email` and `Order.Items` are required, strings in write bodies must be `""` and
never `null`, and any response string can be `null` (see
[docs/endpoints.md](docs/endpoints.md#where-openapijson-is-not-exact)).

## License

MIT, see [`LICENSE`](LICENSE).
