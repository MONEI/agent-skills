# MONEI agent skills

Skills that teach coding agents and commerce agents how to take payment through **MONEI** — a Payment Institution licensed by Banco de España (EP #6911) and the only native Bizum acquirer in Europe.

They follow the open [Agent Skills](https://agentskills.io) convention (`SKILL.md` + `references/`), so they work in Claude Code, Cursor, Codex, OpenCode and any agent that loads skills. They are written to sit next to the skills each commerce platform publishes for its own side (commercetools for Builders, Shopify agent-skills, Salesforce B2C tooling…): the platform teaches the agent the cart and the order; these teach it the money.

## Skills

| Skill | What it covers |
|---|---|
| [`skills/monei-payments`](./skills/monei-payments/SKILL.md) | The payment leg for any storefront or shopping agent: requirements → connector config → frontend → order/webhook backend → proof. Bizum, cards with 3DS/SCA, Apple Pay, Google Pay, SEPA. Platform specifics live in one adapter file per platform. |

### Platform adapters (`skills/monei-payments/references/platforms/`)

| Platform | Status |
|---|---|
| commercetools (Connect connector) | ✅ available |
| Shopify (MONEI Payments · MONEI Pay Bizum apps) | planned |
| Medusa | planned |
| Saleor | planned |
| Salesforce Commerce Cloud | planned |
| WooCommerce · PrestaShop · Adobe Commerce | planned |
| No platform — MONEI REST API + Hosted Payment Page | ✅ covered in the skill itself |

Each adapter answers the same six questions in the same order — where the connector lives, its config keys, its routes or hooks, the frontend touchpoint, the status map onto the platform's payment model, and its pitfalls — so the workflow never changes when the platform does. Contributions of new adapters are welcome; open a PR with the same structure as `commercetools.md`.

## Agentic commerce

`references/agentic-checkout.md` is platform-independent: how a shopping agent built on [anthropics/commerce-agents](https://github.com/anthropics/commerce-agents) (or any MCP / UCP client) hands a cart off to a licensed payment institution without the payment URL or any credential ever reaching the model, why the customer's bank-app confirmation (Bizum) or 3DS (cards) *is* the human approval, and which fields to write on the order so the audit trail has legal standing. MONEI also runs a merchant-facing MCP server (`mcp.monei.com`) and contributed the Bizum handler to Google's Universal Commerce Protocol.

## Install

```bash
# skills only, any agent
npx skills add MONEI/agent-skills
# or copy skills/monei-payments into ~/.claude/skills/
```

## Grounding

Written from MONEI connector source code (`github.com/MONEI`), MONEI's documentation — every page is served as Markdown, indexed in `https://docs.monei.com/llms.txt`, and searchable through a public MCP server at `https://docs-mcp.monei.com/mcp` — and the platforms' own published skills and blueprints. Re-verify config keys against the deployed connector version before relying on them.

## License

Markdown: CC BY 4.0. Code snippets: MIT. See [LICENSE](./LICENSE).
