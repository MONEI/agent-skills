---
name: monei-shopify-adapter
description: Shopify adapter for the monei-payments skill — which MONEI Shopify apps exist (Payments hosted page, Onsite cards, Bizum, MB WAY, Multibanco), how they install and bind to a MONEI account, test vs live mode, how payment state reaches the Shopify order, reconciliation via the MONEI Order ID, country-based method visibility, headless/Hydrogen, and how agent checkout (UCP / Shop Pay / checkout hand-off) interacts with MONEI methods. Read for any Shopify store that sells in Spain or Portugal, or that mentions Bizum, MB WAY or Multibanco.
when_to_use:
  - "Setting up or debugging MONEI on a Shopify store (any plan, including Plus and Hydrogen/headless)"
  - "Offering Bizum, MB WAY or Multibanco in Shopify checkout"
  - "Deciding which MONEI Shopify app(s) to install for a given set of payment methods"
  - "Matching a MONEI payment to a Shopify order, or a Shopify order to a MONEI payment"
  - "Designing agent checkout for a Shopify merchant where MONEI methods must stay available"
metadata:
  contentType: REFERENCE
  area:
    - payments
    - shopify
    - monei
    - bizum
---

# MONEI on Shopify

Shopify adapter for the `monei-payments` skill. Same six questions as every adapter, same order. The generic workflow, `../bizum.md` and `../agentic-checkout.md` still apply; this file only fills in what is Shopify-specific.

**The shape is different from a connector platform.** On Shopify there is no processor to deploy, no config file and no storefront code: MONEI is a set of **payment provider apps** built on Shopify's Payments Apps API. MONEI runs the provider side; the merchant installs, connects and activates. Most of the work is choosing the right apps and placing the methods well in a checkout that also contains Shop Pay.

## 1. Where the integration lives

Five apps, each adding its own option to Shopify checkout. Install as many as the method mix needs; every active app is a separate button.

| App | Accepts | Checkout experience | Shopify alternative-provider id |
|---|---|---|---|
| **MONEI Onsite** | credit & debit cards | native card fields inside Shopify checkout | `44761089` |
| **MONEI Payments** | every method enabled on the MONEI account, incl. Click to Pay and digital wallets | redirect to MONEI's Hosted Payment Page, return to store | `2129921` |
| **MONEI Pay · Bizum** | Bizum | one button → redirect to Bizum → return | `1540097` |
| **MONEI MB WAY** | MB WAY (Portugal) | one button → redirect → return | see docs |
| **MONEI Multibanco** | Multibanco (Portugal) | one button → redirect → return | see docs |

Install links, screenshots and the current ids: [docs.monei.com/e-commerce/shopify/overview](https://docs.monei.com/e-commerce/shopify/overview). Demo store: `monei-demo.myshopify.com` (password `demo`).

**Recommended combinations**

- Spain, best conversion: **Onsite + Bizum** (native cards, dedicated Bizum button), plus **Payments** if wallets / Click to Pay matter.
- Iberia: add **MB WAY + Multibanco**, hidden outside Portugal (see §4).
- Simplest: **Payments** alone — one button, every enabled method on the hosted page. Lower friction to set up, one extra redirect for cards.

MONEI Payments is the only MONEI app that offers Click to Pay and digital wallets on Shopify.

## 2. Configuration

Nothing lives in code or env vars. Three places hold state:

| Where | What |
|---|---|
| **MONEI Dashboard → Settings → Payments → Payment methods** | which methods the account can actually accept. This is the source of truth; nothing in Shopify enables a method. |
| **MONEI Dashboard → Settings → Payments → Integrations → Shopify** | connected Shopify stores per mode; per store: **test mode on/off** and **checkout country → MONEI store** routing (multi-store setups) |
| **Shopify Admin → Settings → Payments → \<MONEI app\>** | **Activate** the provider; method icon toggles |

Install sequence: install link → MONEI Dashboard **Connect** screen (binds the shop to the account you are logged into) → back to Shopify → **Activate**. Programmatic equivalent, if an agent is automating onboarding through the MONEI GraphQL API: `connectShopifyShop`, `getShopifyShopSettings`, `updateShopifyShopSettings`, `shopifyStores` ([docs.monei.com/apis/graphql](https://docs.monei.com/apis/graphql/operations/mutations/connect-shopify-shop)).

**Test vs live:** the app binds to whichever mode the dashboard is in *at install time*. Switching a connection between modes afterwards is done per store from the Integrations page (it re-binds to the paired test/live account). Install in test mode first, prove the round trip, then flip.

**Capture mode:** Shopify's own *Payment capture* setting (automatic vs manual) governs cards where the provider supports authorization; Bizum, MB WAY and Multibanco are always immediate. Verify current support for manual capture with MONEI before promising it to a merchant.

## 3. Routes and hooks

None for the merchant or the storefront. The Payments Apps API session flow (payment session → provider processes → session resolved/rejected → Shopify records the transaction) runs between Shopify and MONEI. Refunds and captures started in Shopify Admin flow to MONEI through the same channel; a refund issued in MONEI Dashboard also reaches the Shopify order.

There is therefore **no webhook to register and no signature to verify** on the merchant side. If you need an event stream anyway (ERP, agent app events), subscribe to Shopify's `orders/paid`, `orders/updated` and `refunds/create` webhooks, or to MONEI account webhooks (`charge.succeeded`, `charge.refunded`) — and reconcile with the key in §5.

## 4. Frontend touchpoint

Shopify renders every active MONEI app as a payment option in its checkout. What you can control:

- **Order and visibility by country.** Bizum should only show to buyers in Spain (and Andorra), MB WAY / Multibanco only in Portugal. Shopify Plus: **Checkout Blocks** → Functions → Payment/Hide; MONEI publishes ready-made rule files for Bizum and MB WAY. Other plans: a third-party hide/sort app (Puco, ETP Sort & Hide Payment Methods). Guide: [hide payment methods by country](https://docs.monei.com/e-commerce/shopify/hide-payment-methods).
- **Placement next to Shop Pay.** Shop Pay is Shopify's accelerated checkout and will sit above the payment list. For a Spanish audience the Bizum button is the conversion lever; make sure it is not buried below four card options and two wallets. A single-button app per regional method (rather than everything inside the hosted page) exists precisely so Bizum can be a first-class option.
- **Headless / Hydrogen.** Payment still happens in Shopify checkout: the storefront sends the buyer to the cart's `checkoutUrl` and MONEI appears there. There is nothing to mount in the headless app.
- **Checkout UI extensions** can add copy around the payment step ("Pay with Bizum from your bank app — no card needed") but cannot render a custom payment form; that is by design.

## 5. State onto the Shopify order

| MONEI payment status | Shopify order | Notes |
|---|---|---|
| `SUCCEEDED` | financial status **Paid** (transaction `sale` / `capture` success) | order exists and is paid; fulfilment can start |
| `AUTHORIZED` (cards, manual capture) | **Authorized** | capture from Shopify Admin before expiry |
| `PENDING` (buyer in bank app / 3DS) | no paid order yet; Shopify holds the checkout | do not treat the return to the store as payment |
| `FAILED` / `CANCELED` / `EXPIRED` | no order, or order with a failed transaction | buyer can retry with another method |
| `REFUNDED` / `PARTIALLY_REFUNDED` | **Refunded** / **Partially refunded** | whichever side initiated it |

**Reconciliation key:** Shopify stores the MONEI payment id in the order's gateway data. In Shopify Admin → Orders search `receipt.payment_id:<MONEI Order ID>` (the Order ID shown in MONEI Dashboard → Payments → Payment details); the same value appears under *Information from the gateway* on the Shopify order. Use it in any exporter, agent tool or dispute workflow that has to map one system to the other.

## 6. Pitfalls specific to Shopify

1. **The method icon toggles in Shopify do nothing to acceptance.** They only show or hide icons at checkout. Methods are enabled in MONEI Dashboard → Payment methods. A method "missing" on Shopify is almost always disabled on the MONEI account.
2. **Mode is fixed at install.** Installing while the dashboard is in live mode binds a live connection; test payments will not work until the store is switched to test from the Integrations page (or the app reinstalled in test mode).
3. **One app, one button.** Installing MONEI Payments *and* MONEI Onsite gives buyers two card paths (hosted page and native fields). Decide which is the card path and hide the other's card icons, or only install one for cards.
4. **Regional methods shown everywhere.** Bizum outside Spain fails at the last step and looks like a broken checkout. Hide by country from day one.
5. **Bizum test amounts.** In test mode only Bizum amounts **below €5** succeed (`+34 500 000 000`); €5–€10 declines, €10–€15 approves via redirect, above €15 "phone not registered". A €49.90 test cart failing is expected.
6. **Multi-store routing is by checkout country.** If the merchant has several MONEI stores (per country/entity), map each checkout country to the right MONEI store on the Integrations page, or payments settle into the default store.
7. **Two refund entry points.** Support teams refund from Shopify, finance from MONEI. Both work and both sync; agree on one to keep the audit trail in one place.
8. **Shop Pay in agent flows.** See §7 — an agent that completes checkout through Shop Pay never shows the MONEI methods.

## 7. Agent checkout on Shopify

Shopify advertises UCP payment handlers for its merchants automatically, and **Shop Pay is built in** as the delegated instrument: an agent that holds a Shop Pay token can complete a Shopify checkout in-agent without ever rendering the merchant's payment list. That is the path where MONEI methods — Bizum above all — are not seen.

What keeps MONEI in the loop:

- **Checkout hand-off (the blueprint's `checkout_handoff`) and Shopify's embedded checkout (ECP).** Both render the merchant's real checkout, MONEI buttons included. For a Spanish buyer without Shop Pay, or one who prefers to confirm in their bank app, this is the higher-converting path. An agent should offer it, not default silently to Shop Pay.
- **UCP payment handlers.** MONEI contributed the Bizum handler (`org.bizum.bizum`) to UCP. Whether a given Shopify merchant's UCP profile advertises it alongside `dev.shopify.shop_pay` depends on Shopify's handler negotiation, not on the merchant; treat it as "check the merchant profile", not as a given.
- **Audit trail.** Whichever path, write the MONEI payment id (from the order's gateway data) and the authorisation factor (`bizum-bank-app`, `3ds`, `wallet`) on the order or in the agent's ledger, as `../agentic-checkout.md` describes. On Shopify the order already carries the payment id; the agent only has to read it back after `orders/paid`.

For a merchant agent (refunds, status lookups) prefer the MONEI MCP server (`mcp.monei.com`) or MONEI GraphQL over scraping Shopify: MONEI is the acquirer of record for those transactions and its record is the one with legal standing.

## Quick reference

- Apps: Onsite (cards, native), Payments (all methods, hosted page), Bizum, MB WAY, Multibanco. One button each.
- Enable methods in **MONEI Dashboard**, activate providers in **Shopify Settings → Payments**, route countries in **Dashboard → Integrations → Shopify**.
- Test/live fixed at install; flip per store from the Integrations page.
- Find the order: `receipt.payment_id:<MONEI Order ID>`.
- Hide Bizum outside ES/AD (Checkout Blocks on Plus; Puco/ETP otherwise).
- Agent checkout: offer the checkout hand-off / ECP path so Bizum is reachable; Shop Pay-only flows bypass MONEI.
- Docs: [docs.monei.com/e-commerce/shopify/overview](https://docs.monei.com/e-commerce/shopify/overview) · every page also at `…/overview.md` · `https://docs-mcp.monei.com/mcp`
