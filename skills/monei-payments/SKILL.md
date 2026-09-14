---
name: monei-payments
description: Add the payment leg to any storefront or commerce agent with MONEI — Bizum (native acquiring), cards with 3DS/SCA, Apple Pay, Google Pay and SEPA Direct Debit — through MONEI's platform connectors (commercetools today; Shopify, Medusa, Saleor, Salesforce and WooCommerce adapters to follow) or its REST API. Covers requirements, connector config, Bizum's redirect/RTP flow, webhook reconciliation, test mode, and how a shopping agent (Claude commerce-agents or any MCP/UCP client) hands a cart off to a licensed payment institution and gets a settled, auditable result back. Use whenever a project sells in Spain, mentions Bizum, or needs a Banco de España-licensed PSP behind an agent.
when_to_use:
  - "Integrating or configuring a MONEI payment connector on any commerce platform (commercetools, Shopify, Medusa, Saleor, Salesforce Commerce Cloud, WooCommerce, PrestaShop) or calling the MONEI API directly"
  - "Accepting Bizum in a storefront, hosted checkout, or custom frontend"
  - "Adding a payment step to a shopping agent built on the Claude commerce-agents blueprint, where the merchant sells in Spain or Southern Europe"
  - "Looking up a MONEI connector's config keys and routes, the webhook signature scheme, or test cards / Bizum test amounts"
  - "Deciding between MONEI's card, Bizum, wallet and SEPA methods for a given cart (amount limits, capture modes, refund behaviour)"
  - "Debugging a MONEI payment stuck in PENDING, a commercetools Payment transaction that never reaches Success, or a rejected webhook signature"
  - "Implementing a spec/plan/tasks.md task annotated [SKILL: monei-payments]"
metadata:
  contentType: SKILL
  area:
    - Integrations
    - payments
    - platform-agnostic
    - psp
    - bizum
    - agentic-commerce
---

# MONEI payments — the payment leg for storefronts and agents

MONEI is a Payment Institution licensed by Banco de España (EP #6911) and the only native Bizum acquirer in Europe. Bizum is the payment method that decides conversion for any merchant selling in Spain — roughly 28M users pay with it from their bank app — and it is not available through the card-first PSPs most platform references cover.

This skill is **platform-agnostic by design**. The workflow, the Bizum reference and the agent checkout reference apply to any commerce backend; the platform-specific facts (config keys, routes, bundle names, scopes) live in one adapter file per platform under `references/platforms/`. It fills in the **payment leg** that commerce platforms and the Claude commerce-agents blueprint deliberately leave open: the blueprint's `StorefrontBackend` stops before money moves, and every platform's cart/order model stops at "a Payment exists".

| Platform | Adapter | Status |
|---|---|---|
| commercetools (Connect connector) | [references/platforms/commercetools.md](./references/platforms/commercetools.md) | available |
| Shopify (MONEI Payments / MONEI Pay · Bizum apps) | `references/platforms/shopify.md` | planned |
| Medusa | `references/platforms/medusa.md` | planned |
| Saleor | `references/platforms/saleor.md` | planned |
| Salesforce Commerce Cloud | `references/platforms/salesforce.md` | planned |
| WooCommerce / PrestaShop / Adobe Commerce | — | planned |
| No platform — MONEI REST API + Hosted Payment Page | covered inline in this file and in `agentic-checkout.md` | available |

Every adapter answers the same six questions in the same order so that switching platform never changes the workflow: where the connector lives, its config keys (secured vs standard), its routes or hooks, the frontend touchpoint, the status map onto the platform's payment model, and its platform-specific pitfalls.

Three things are different about MONEI compared with the card-first PSPs, and most of the guidance below follows from them:

1. **Bizum is a redirect / request-to-pay method, not a card form.** There is no PAN, no CVC and no 3DS challenge; the customer authorises in their own banking app. Authentication is therefore always strong (it happens inside the bank), and there is no separate "authorize now, capture later" — Bizum is always an immediate sale.
2. **Cards route across several Spanish acquirers.** The connector and MONEI's routing pick the acquirer; you configure methods, not acquirers. 3DS/SCA applies to cards as usual and MONEI handles the challenge in its secure iframe or redirect.
3. **The settlement record has legal standing.** Because the acquirer is a licensed institution, the MONEI payment (`id`, `status`, `orderId`, timestamps, method) is the audit trail. Treat it — reflected in the commercetools `Payment` via webhook — as the source of truth, never the browser's `onComplete`.

## Workflow

Follow these steps in order. Steps 1–2 decide *what* to build; steps 3–5 build and prove it.

### Step 0 — Ground yourself in the live docs (required)

MONEI publishes its documentation for agents. Before writing any code, search the docs rather than answering from memory:

- MCP: `https://docs-mcp.monei.com/mcp` (no key, read-only; tools `algolia_search_index_monei` and `algolia_search_for_facet_values`)
- HTTP: `https://docs-search.monei.com/1/indexes/monei?query=<terms>&filters=lang:en`
- Any page as Markdown: drop the trailing slash and append `.md` — e.g. `https://docs.monei.com/payments/capture.md`
- Index of every page: `https://docs.monei.com/llms.txt`
- OpenAPI: `https://js.monei.com/api/v1/openapi.json`

Use the platform's own docs tooling for its side (e.g. the commercetools Knowledge MCP from `commercetools-ai-plugins`, Shopify's `agent-skills`). Neither replaces the other.

### Step 1 — Extract the payment requirements

Ask, don't assume. Each answer maps to a config key or an architectural choice in Step 2:

1. **Which methods?** Bizum, card, Apple Pay, Google Pay, SEPA Direct Debit. Default for a Spanish merchant: `bizum,card,applePay,googlePay`. Bizum first — it is the reason to pick MONEI.
2. **Capture mode for cards?** Immediate sale (`SALE`) or pre-authorization then capture (`AUTH`). Bizum, wallets and SEPA are always immediate — say so explicitly so nobody designs an authorise-then-capture flow for Bizum.
3. **Ticket size.** Online Bizum has no maximum on MONEI's side; the customer's issuing bank may apply its own limit, so a high-ticket Bizum decline is usually the bank, not the integration (see `references/bizum.md`). Keep cards available as the fallback. In **test mode Bizum only approves amounts below €5** — design the test plan around that before anyone declares the integration broken.
4. **Refunds / partial refunds?** All methods support full refunds; card and Bizum support partial. SEPA cannot be cancelled once submitted.
5. **Return URL and storefront origin(s)** — for the redirect methods (Bizum, 3DS challenge) the customer leaves the page and comes back to `completeUrl` / `cancelUrl`.
6. **Which platform, and hosted checkout or custom storefront?** Pick the adapter in `references/platforms/`; if there is none yet, the MONEI REST API + Hosted Payment Page path applies (see `agentic-checkout.md` — it needs no connector). The frontend contract differs per choice (Step 3).
7. **Is an agent placing the order?** If yes, read `references/agentic-checkout.md` — the approval gate and the handoff are the design, not an afterthought.
8. **Anything special?** Subscriptions, MOTO, split settlement, marketplace payouts (Bizum Payout exists), multi-currency (MONEI settles in EUR; cards can be presented in other currencies — check the docs).

Write the answers as a short requirements block and confirm them before deriving config.

### Step 2 — Connector: install, configure, or fork

Walk the same ladder on every platform:

1. **The platform's MONEI connector covers everything** → install it from the platform's marketplace and derive its config from Step 1 using the adapter file. (commercetools: `references/platforms/commercetools.md`.)
2. **Gap looks like a capability** → check whether it is a connector toggle or a MONEI dashboard setting. Payment methods are enabled **per account** in MONEI Dashboard → Settings → Payment methods; connector variables only filter what is shown.
3. **Genuine gap** → fork the connector (MONEI's connectors are MIT on `github.com/MONEI`) or, for platforms without one, integrate the REST API directly: `POST https://api.monei.com/v1/payments` with `amount` (cents), `currency`, `orderId`, `callbackUrl`, then either the Hosted Payment Page redirect or MONEI.js components on your page.

Produce for the user: the filled config block, the secret key names (never values), the platform scopes/permissions, and a one-line rationale per non-obvious key.

### Step 3 — Frontend touchpoint

With a **platform-hosted checkout** (commercetools Checkout, Shopify checkout): install the connector, enable MONEI in the checkout configuration and order the methods. Nothing else to build in the browser.

With a **custom storefront**: load the platform adapter's browser bundle (commercetools: `monei-enabler.js`), or MONEI.js directly, and mount the method components (`bizum`, `card`, `applePay`, `googlePay`, `sepaDirectDebit`). Card data never touches your DOM — MONEI.js renders secure iframes, which is what keeps the storefront out of PCI scope.

For redirect methods, `submit()` resolves with a `redirectUrl`. Navigate to it; the customer returns to your `completeUrl`. **Do not create the Order on return.** Wait for the webhook (Step 4).

### Step 4 — Backend: Order, post-purchase, reconciliation (test-first)

Same invariants as every payment connector, with the MONEI specifics:

1. **Server-side payment creation** — the amount is computed on the server from the platform's cart; the browser only receives what it needs to render (a session id, a payment id, or a hosted-page URL). Never a secret.
2. **Order creation** — after the platform's payment record shows a successful charge (or authorization). For Bizum this is always the `SUCCEEDED` status arriving on the webhook; the redirect back to `completeUrl` is not confirmation.
3. **Capture / cancel / refund** — through the connector's server-side routes or the MONEI API from your backend, never from the storefront. Capture applies to card `AUTH` only.
4. **Webhook reconciliation** — MONEI posts the Payment object to `callbackUrl` when the payment reaches a final state, signed with `MONEI-Signature: t=<ts>,v1=<hmac>`. Map `SUCCEEDED → charge success`, `AUTHORIZED → authorization success`, `FAILED/CANCELED/EXPIRED → failure`, `REFUNDED/PARTIALLY_REFUNDED → refund`. Verify the signature against the **raw body**; a re-serialised JSON body will not match.

Write the tests before the code, mock the outbound boundary (connector, MONEI API, platform API), and assert on what your code decided to do. The platform adapter lists the exact status map and the pitfalls each test should pin.

### Step 5 — Prove the round trip

In **test mode** (test API key + test Account ID from Dashboard → Settings → API Access, `MONEI_ENVIRONMENT=test`):

- **Bizum**: phone `+34 500 000 000`, amount **< €5** → `SUCCEEDED`. €5–€10 → declined (`E506`). €10–€15 → approved via redirect flow. > €15 → "phone not registered". These bands are a feature: use them to test every branch.
- **Cards**: `4444 4444 4444 4406` (3DS challenge), `4444 4444 4444 4422` (frictionless), `5555 5555 5555 5565` (Mastercard challenge); expiry `12/34`, CVC `123`.
- Confirm in MONEI Dashboard → Payments (test mode) **and** in the platform that the payment record shows success, then that a refund lands as a refund transaction on both sides.

Then turn it into one full-flow integration test against the real connector and test data, asserting the platform-side trace at each commit point.

## References

| Need | Read |
|---|---|
| **commercetools**: exact `connect.yaml` keys, scopes, processor routes, enabler bundle, status map, webhook signature, pitfalls | [references/platforms/commercetools.md](./references/platforms/commercetools.md) — written as a sibling of `stripe.md` in `commercetools-integrations` |
| Other platforms | `references/platforms/<platform>.md` as they land; until then the REST API + Hosted Payment Page path in `agentic-checkout.md` |
| Bizum as a method: flows (redirect vs request-to-pay), limits, refunds, test bands, what "SCA" means for Bizum | [references/bizum.md](./references/bizum.md) |
| An agent placing the order: the approval gate, MONEI's MCP server, UCP Bizum handler, what to log for the audit trail | [references/agentic-checkout.md](./references/agentic-checkout.md) |
| Platform-side contract (session/BFF, Order timing, webhook is truth) | the platform's own skills (e.g. `commercetools-integrations/references/payment/backend-integration.md`, Shopify `agent-skills`) |
| Building or forking a connector | the platform's connector template/skill (e.g. `commercetools-connect`); MONEI connectors on `github.com/MONEI` are MIT |

## Checklist

- [ ] Requirements block confirmed: methods, capture mode, ticket size vs Bizum limits, refunds, return URLs, agent or human checkout
- [ ] Platform adapter chosen; connector installed in **test** mode (or REST API path with the test key)
- [ ] Config filled from the adapter; secrets supplied by the user, never by you
- [ ] Platform credentials carry the scopes/permissions the connector declares
- [ ] Webhook endpoint registered in MONEI Dashboard; `MONEI_WEBHOOK_SECRET` set; signature verified on raw body
- [ ] Order created only on webhook `Success`, not on redirect return
- [ ] Bizum test paid with `+34500000000` below €5; card test with a 3DS challenge card
- [ ] Refund through the processor reflected as a `Refund` transaction
- [ ] If an agent checks out: approval gate before `POST /payments`, MONEI payment id + approver recorded on the Order

## Quick reference

- Credentials: test and live **API key + Account ID pairs** are different (Dashboard → Settings → API Access). `Authorization: <API key>` on `https://api.monei.com/v1`.
- Methods: `bizum`, `card`, `applePay`, `googlePay`, `sepaDirectDebit` — enabled per account in the Dashboard, then filtered by the connector.
- Capture mode is per payment (`transactionType: SALE | AUTH`); AUTH is cards only.
- Bizum in test mode: **only amounts below €5 succeed.**
- Platform specifics (keys, routes, bundles): `references/platforms/<platform>.md`.
- Webhook header: `MONEI-Signature: t=…,v1=…` — HMAC-SHA256 over the raw body; ignore any scheme other than `v1`.
- Docs for agents: `https://docs-mcp.monei.com/mcp` · `https://docs.monei.com/llms.txt`
