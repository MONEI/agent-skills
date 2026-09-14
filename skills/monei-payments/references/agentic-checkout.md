---
name: monei-agentic-checkout
description: How a shopping agent — the Claude commerce-agents blueprint on commercetools, or any MCP/UCP client — completes the payment leg with MONEI. Covers the checkout_handoff hook, why the payment URL never passes through the model, the approval gate (SCA / bank-app confirmation is the human-in-the-loop), MONEI's MCP and UCP surfaces, and what to record so the order carries a legally standing audit trail.
when_to_use:
  - "Adding payment to a shopping agent built on anthropics/commerce-agents with a commercetools StorefrontBackend"
  - "Wiring an MCP or UCP-speaking agent to a payment institution instead of a sandbox"
  - "Designing the approval / audit trail for agent-initiated purchases in Spain or the EU"
metadata:
  contentType: REFERENCE
  area:
    - agentic-commerce
    - payments
    - monei
    - bizum
    - mcp
    - ucp
---

# Agent checkout with MONEI

The Claude commerce-agents blueprint is explicit: nothing in it places an order or takes payment. The shopping agent's `checkout` tool renders the cart and hands off; `StorefrontBackend.checkout_handoff()` may return a URL that the executor attaches to the card **after** the model's call, so a payment URL is never a tool argument and never reaches the model. commercetools owns the cart, catalog and order. This reference is the third piece — the payment leg — and it keeps the same discipline.

## The shape

```
shopping agent (Claude)          commercetools               MONEI (licensed PI, EP #6911)
        │                              │                              │
   checkout tool ──► cart rendered     │                              │
        │                              │                              │
   checkout_handoff() ─────────────────┼──► POST /payments (SALE) ───►│  MONEI payment created
        │        (host code, not the model)                           │  returns hosted page / redirectUrl
        │◄── card gets the URL (never in model context) ◄─────────────┤
        │                              │                              │
   customer taps → pays in bank app (Bizum) / 3DS (card)              │  ← the approval, done by the payer's bank
        │                              │                              │
        │                              │◄── callbackUrl webhook ──────┤  SUCCEEDED, signed
        │                              │  Payment.transaction Success │
        │                              │  Order created (host)        │
   next turn reads app event "paid" ◄──┤                              │
```

Three properties fall out of this and are worth stating to whoever designs the agent:

1. **The model never touches money.** It never sees the payment URL, the API key, or a card number. It sees "the customer paid" as an app event on the next turn — exactly how the blueprint tells hosts to relay out-of-conversation steps.
2. **The human approval is the payment authorisation.** Bizum is confirmed in the customer's bank app; cards go through 3DS. That is Strong Customer Authentication performed by the payer's own bank, with the bank's own record. You do not have to invent a "who approved this" mechanism for the purchase — the regulated one already exists; you only have to record its result.
3. **The audit trail has legal standing.** The MONEI payment (`id`, `orderId`, `amount`, `status`, `paymentMethod`, timestamps) is a record held by a Banco de España-licensed institution. Write the MONEI payment `id` and the approving factor (Bizum / 3DS) onto the commercetools Order; that is the line the finance team, and a dispute, will look for.

## Implementing `checkout_handoff` on a commercetools backend

The hook receives the session and the cart. In host code (never in a tool the model can call):

1. Compute the amount server-side from the commercetools cart (`taxedPrice.totalGross.centAmount` — never from anything the model produced).
2. Create the MONEI payment through the **connector's processor** (`POST /payments` with `transactionType: 'SALE'`, `orderId` = your pre-generated order number, `completeUrl`/`cancelUrl` pointing at host routes, optional `paymentMethodType: 'bizum'` when the billing country is `ES` and the customer prefers it) — or, without the connector, directly via MONEI's REST API (`POST https://api.monei.com/v1/payments`, header `Authorization: <API key>`, body `amount`, `currency`, `orderId`, `callbackUrl`, `customer`, `allowedPaymentMethods`). The response carries the URL of MONEI's hosted payment page for this payment.
3. Return `[CheckoutHandoff(url=<that https URL>, label=…)]`. The executor puts it on the card.
4. On the `callbackUrl` webhook with `status: SUCCEEDED` (signature verified, see the platform adapter (`platforms/commercetools.md`)), create the Order in commercetools with the MONEI payment id on it, then **queue an app event** on the agent session so the next turn can say "paid, order #… confirmed".
5. On `FAILED` / `EXPIRED` / `CANCELED`, queue the corresponding event; the agent offers to retry or switch method. Never auto-retry a payment.

Idempotency: derive the `orderId` from the session id + a hash of the cart lines, as the blueprint recommends for deduplicated writes. MONEI's `orderId` is your reconciliation key.

## Talking to MONEI over MCP

MONEI runs a remote MCP server at **`https://mcp.monei.com`** (OAuth 2.0 with PKCE, Streamable HTTP + SSE). It exposes the payment operations an agent legitimately needs — creating payment requests and links, reading a payment's status, refunds, and account-level reads — scoped to the merchant who authorised it. Use it when the *merchant's* agent needs to act ("refund order 14379 in full", "what's the status of yesterday's Bizum payments?"), not to let a *shopping* agent pay: in the flow above the shopper never needs an MCP tool, because the hand-off URL does the work and the bank does the approval.

> Tool names and scopes: read them from the server's tool list after connecting rather than from memory; they evolve. The docs server (`https://docs-mcp.monei.com/mcp`, no auth) is separate and read-only — it searches documentation, nothing else.

Rules of thumb for a merchant agent with MONEI tools:

- **Reads are free; writes are staged.** Same as the blueprint's merchant agent: a refund is a staged change the host's approval surface applies. Do not let the model call a refund tool directly on the strength of a chat message.
- **Amounts in cents, `orderId` always set.** Every MONEI write should carry the merchant's `orderId` so the dashboard, the webhook and the commercetools Order agree.
- **Never put an API key in the model's context.** OAuth handles it; the token lives in the MCP client.

## UCP and other agent protocols

MONEI contributed the **Bizum handler** to Google's Universal Commerce Protocol (namespace `org.bizum.bizum`), with MONEI as reference implementer. For a UCP-speaking agent this means Bizum appears as a declared payment handler and the checkout completes through MONEI without a bespoke integration. The commercetools side is unchanged — the cart and order still live there; the handler only replaces the hand-off step. Read the UCP handler definition before wiring it; the mapping from a UCP payment completion to the connector's webhook/status model is the same table as the platform adapter.

## What to write on the Order

Minimum, as custom fields or `paymentInfo`:

| Field | Value | Why |
|---|---|---|
| MONEI payment `id` | from the webhook | the primary key at the licensed institution |
| `paymentMethod.type` | `bizum`, `card`, `applePay`, `googlePay`, `sepaDirectDebit` | tells you which SCA path authorised it |
| `authorisation` | `bizum-bank-app` / `3ds-challenge` / `3ds-frictionless` / `wallet` | the human approval factor, for the audit question "who signed off" |
| `agentSessionId` | the shopping session | ties the conversation to the money |
| `orderId` (yours) | the idempotency key | reconciliation across all three systems |

Keep card details out entirely — MONEI already tokenises them and the connector never receives a PAN.

## Test plan for an agent checkout

1. Cart in **test mode**, billing country `ES`, total **< €5** (Bizum test band), phone `+34 500 000 000` → card shows the hand-off, customer "pays", webhook `SUCCEEDED`, Order created, next agent turn reads the paid event.
2. Same cart at €7 → `E506` decline → agent offers card; card `4444 4444 4444 4406` → 3DS challenge → `SUCCEEDED`.
3. Abandon at the bank-app step → `EXPIRED` → agent offers to retry; **no Order** exists.
4. Refund from the merchant agent → staged → applied → `Refund` transaction on the Payment, `REFUNDED` in MONEI Dashboard.

If all four pass, the payment leg is done and the audit trail is real.
