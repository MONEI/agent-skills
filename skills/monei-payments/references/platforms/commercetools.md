---
name: monei-payment-connector
description: MONEI-specific values for the commercetools MONEI payment connector — connector repo, exact connect.yaml keys (standard vs secured), API-client scopes, processor routes, enabler bundle and UMD global, MONEI→commercetools status map, webhook signature scheme, test cards and Bizum test amounts. Apply on top of the provider-agnostic connector-contract.
when_to_use:
  - "Integrating or configuring the commercetools MONEI payment connector"
  - "Looking up the MONEI connector's exact env var names, processor routes, enabler bundle, test cards or Bizum test amounts"
  - "Mapping a MONEI payment status to a commercetools transaction type/state"
metadata:
  contentType: REFERENCE
  area:
    - checkout
    - payments
    - psp
    - connect
    - monei
    - bizum
---

# MONEI payment connector

commercetools adapter for the `monei-payments` skill. Inside `commercetools-ai-plugins`, read the provider-agnostic `connector-contract.md` first for the flow and the shared pitfalls — this only fills in the MONEI-specific blanks. For Bizum as a payment method (limits, flows, what SCA means there) see [../bizum.md](../bizum.md).

## The connector

- **Connector:** MONEI Payment Connector for commercetools Connect. Listed in the Connect Marketplace; also deployable from source with the Connect CLI.
- **Source:** [`MONEI/connect-payment-integration-monei`](https://github.com/MONEI/connect-payment-integration-monei) (MIT) — a monorepo with `processor/` (`service`, Fastify + TypeScript, `@commercetools/connect-payments-sdk`) and `enabler/` (`assets`, webpack UMD bundle).
- **Merchant docs:** [docs.monei.com/e-commerce/commercetools](https://docs.monei.com/e-commerce/commercetools) — install, configure, go-live checklist.
- **Verify the version** before trusting any key below: config keys and routes are read from the repository's `main` at the time of writing. Re-check the deployment's `connect.yaml` if behaviour differs.

## Payment methods the connector exposes

| Method (`MONEI_PAYMENT_METHODS_ENABLED` value) | Type | Capture | Refund | Cancel |
|---|---|---|---|---|
| `card` (Visa, Mastercard; multi-acquirer routing) | web component (secure iframe) | `SALE` immediate or `AUTH` + capture | ✅ full/partial | ✅ (uncaptured AUTH) |
| `bizum` | redirect / request-to-pay | immediate only | ✅ full/partial | ✅ (pending) |
| `applePay` | web component | immediate | ✅ | ✅ |
| `googlePay` | web component | immediate | ✅ | ✅ |
| `sepaDirectDebit` | form input | immediate (async settlement) | ✅ | ✗ |

Methods must also be **enabled on the MONEI account** (Dashboard → Settings → Payment methods). The connector variable only filters what the enabler offers; it cannot enable a method the account does not have.

## Configuration keys (`connect.yaml`)

The processor application takes these. **Secured** values live in `securedConfiguration` and are never logged or returned.

Secured (secrets):

| Key | Purpose |
|---|---|
| `CTP_CLIENT_SECRET` | commercetools API client secret |
| `MONEI_API_KEY` | MONEI API key — **test and live keys are different**; match it to `MONEI_ENVIRONMENT` |
| `MONEI_WEBHOOK_SECRET` | HMAC key used to verify inbound MONEI webhook signatures |

Standard:

| Key | Notes |
|---|---|
| `CTP_PROJECT_KEY` | project key |
| `CTP_CLIENT_ID` | commercetools API client id (scopes below) |
| `CTP_AUTH_URL` / `CTP_API_URL` / `CTP_SESSION_URL` | region hosts; defaults point at `europe-west1.gcp` — set to your region |
| `CTP_JWKS_URL` / `CTP_JWT_ISSUER` | Merchant Center JWKS + issuer for session JWT validation; defaults for `europe-west1.gcp` |
| `MONEI_ACCOUNT_ID` | MONEI Account ID (Dashboard → Settings → API Access). Test and live IDs differ. |
| `MONEI_ENVIRONMENT` | `test` \| `live`. Default `test`. Switch to `live` only after the round trip is proven. |
| `MONEI_PAYMENT_METHODS_ENABLED` | comma-separated: `bizum,card,applePay,googlePay,sepaDirectDebit`. Default `bizum,card,applePay,googlePay`. |
| `STORED_PAYMENT_METHODS_ENABLED` | `"true"` enables stored payment methods (returning customers); default `"false"`. Needs a `customerId` on the cart. |
| `STORED_PAYMENT_METHODS_PAYMENT_INTERFACE` | `paymentInterface` written on stored payment methods; default `monei` |
| `STORED_PAYMENT_METHODS_INTERFACE_ACCOUNT` | `interfaceAccount` written on stored payment methods |

Requirements → keys: methods → `MONEI_PAYMENT_METHODS_ENABLED`; test/live → `MONEI_ENVIRONMENT` + matching `MONEI_API_KEY`/`MONEI_ACCOUNT_ID` pair; saved methods → `STORED_PAYMENT_METHODS_*`; region → the five `CTP_*_URL` keys. Capture mode is **per payment** (`transactionType: SALE | AUTH` in the `POST /payments` body), not a connector key.

`connect.yaml` must sit at the **repository root** (it does) and use only the documented envelope keys — same rule as every connector.

## API client scopes

The connector declares, for its own API client:

`manage_payments`, `manage_orders`, `view_sessions`, `view_api_clients`, `manage_checkout_payment_intents`, `introspect_oauth_tokens`, `manage_types`, `view_types`

A missing scope returns `400 invalid_scope` at token time, which surfaces as a generic "Permissions exceeded". The storefront BFF that creates sessions and Orders needs `manage_sessions` and `manage_orders` on **its own** client; keep the two clients separate in production.

## Processor routes

| Route | Purpose | Notes |
|---|---|---|
| `GET /payment-methods` | methods the enabler may render | filtered by `MONEI_PAYMENT_METHODS_ENABLED` |
| `POST /payments` | create the MONEI payment | body: `amount` (cents), `currency`, `orderId`, `paymentMethodType?`, `customer?`, `billingAddress?`, `shippingAddress?`, `completeUrl`, `cancelUrl`, `transactionType?` (`SALE` default \| `AUTH`). Returns `moneiPaymentId`, `status`, and `redirectUrl` for redirect methods. The processor sets `callbackUrl` to its own `/webhooks/monei`. |
| `POST /payments/:id/capture` | capture an `AUTHORIZED` card payment | body `{ amount? }` (cents) for partial capture |
| `POST /payments/:id/cancel` | release an uncaptured authorization / void a pending payment | |
| `POST /payments/:id/refund` | refund a captured payment | body `{ amount?, reason? }` |
| `GET /payments/:id` | current MONEI payment | use for reconciliation when a webhook is suspected lost |
| `POST /webhooks/monei` | MONEI callback | see *Webhooks* |
| `GET /health` | liveness | returns connector name + version |

Amounts are **integer cents** on both sides (commercetools `centAmount` ↔ MONEI `amount`), so no conversion is needed — €12.34 is `1234` in both.

## Enabler bundle (browser)

- File: **`monei-enabler.js`** (webpack UMD). UMD global: **`MoneiPaymentEnabler`** (default export).
- Loads **MONEI.js** at `init()` and renders card / wallet inputs in MONEI-hosted secure iframes — the storefront never sees card data.

```html
<script src="https://assets-….{region}.commercetools.app/monei-enabler.js"></script>
<script>
  const enabler = new MoneiPaymentEnabler();
  await enabler.init({ processorUrl, sessionId, locale: 'es-ES', onComplete, onError });
  const bizum = enabler.createComponent('bizum');   // 'card' | 'applePay' | 'googlePay' | 'sepaDirectDebit'
  bizum.mount('#payment-container');
  const result = await bizum.submit();             // redirect methods resolve with { redirectUrl }
  if (result.redirectUrl) window.location.assign(result.redirectUrl);
</script>
```

With the hosted **Checkout** product none of this is written by hand: install the connector, create a Checkout Application and select MONEI in its payment integrations. The connector supports both *web components* (per-method) and *drop-in* integration types there.

## Status map (MONEI → commercetools Payment transaction)

| MONEI `status` | Transaction type | State | Notes |
|---|---|---|---|
| `PENDING` / `PENDING_PROCESSING` | — | `Pending` | customer is in the bank app / 3DS; **do not create the Order** |
| `AUTHORIZED` | `Authorization` | `Success` | card `AUTH` only; capture before the authorization expires |
| `SUCCEEDED` (`SALE`) | `Charge` | `Success` | create the Order here |
| `SUCCEEDED` (`AUTH`, after capture) | `Charge` | `Success` | |
| `FAILED` | `Charge`/`Authorization` | `Failure` | show the failure, let the customer retry |
| `CANCELED` | `CancelAuthorization` | `Success` | |
| `EXPIRED` | — | `Failure` | pending payment never completed (customer closed the app) |
| `REFUNDED` / `PARTIALLY_REFUNDED` | `Refund` | `Success` | |
| `PAID_OUT` | — | — | settlement bookkeeping only; nothing to do in commercetools |

Full list with dashboard labels: [docs.monei.com/troubleshoot/payment-statuses](https://docs.monei.com/troubleshoot/payment-statuses).

## Webhooks

MONEI has two delivery shapes and the connector relies on the second:

- **Account webhooks** (Dashboard → Settings → Webhooks) send an **event envelope** (`{ type: "charge.succeeded", object: { …payment } }`) for every subscribed event.
- **Per-payment `callbackUrl`** sends the **Payment object directly** when the payment reaches a final state — even if the customer closed the browser mid-redirect. The connector sets `callbackUrl` on every `POST /payments`, so `/webhooks/monei` should expect a bare Payment.

Every delivery carries **`MONEI-Signature: t=<unix-ts>,v1=<hex-hmac-sha256>`**. Verify against the **raw request body** (Fastify parses JSON by default — register a raw-body parser for this route, exactly as the Stripe reference describes), ignore any scheme other than `v1`, compare with a constant-time function, then respond `200`. Retries are finite; a `4xx/5xx` after the last attempt loses the event, so reconcile with `GET /payments/:id` if a transaction is stuck `Pending`.

Setup: Dashboard → Settings → Webhooks → add `https://<processor-url>/webhooks/monei` → copy the signing key into `MONEI_WEBHOOK_SECRET`.

## Test mode

Switch the Dashboard to **Test** (header toggle); generate the **test** API key and Account ID. They only work with `MONEI_ENVIRONMENT=test`.

Cards (expiry `12/34`, CVC `123`):

| Card | Outcome |
|---|---|
| `4444 4444 4444 4406` | Visa, 3DS 2.1 challenge |
| `4444 4444 4444 4414` | Visa, 3DS direct (no challenge) |
| `4444 4444 4444 4422` | Visa, 3DS frictionless |
| `5555 5555 5555 5524` | Mastercard, direct |
| `5555 5555 5555 5565` | Mastercard, challenge |

Bizum — phone **`+34 500 000 000`** for every test; the **amount** selects the branch:

| Amount | Result | Flow |
|---|---|---|
| < €5 | approved (`E000`) | request-to-pay |
| €5 – €10 | declined (`E506`) | request-to-pay |
| €10 – €15 | approved (`E000`) | redirect |
| > €15 | "phone not registered in Bizum" | — |

**In test mode only Bizum amounts below €5 succeed on the happy path.** Put this in the test plan before anyone concludes the integration is broken.

## Pitfalls specific to MONEI

1. **Bizum has no AUTH.** A flow designed around authorize-then-capture silently breaks for Bizum. Branch on method: `AUTH` for cards when the business needs it, `SALE` for everything else.
2. **Redirect return ≠ payment success.** For Bizum and 3DS-challenge cards the customer comes back to `completeUrl` before (or without) the webhook. Gate Order creation on the webhook status, and give the return page a polling fallback on `GET /payments/:id`.
3. **Envelope vs bare Payment.** If someone also points an *account* webhook at `/webhooks/monei`, the handler receives envelopes. Either subscribe only the callback path or unwrap `object` when `type` is present.
4. **Re-serialised bodies fail signature checks.** `JSON.stringify(request.body)` is not the raw body. Verify on the raw bytes.
5. **Header format.** The signature header is `t=…,v1=…`, not a bare digest. Parse it, check the timestamp tolerance, compare `v1`.
6. **Test/live key pairs.** A live `MONEI_ACCOUNT_ID` with a test `MONEI_API_KEY` (or vice-versa) returns auth errors that look like scope problems. Rotate both together with `MONEI_ENVIRONMENT`.
7. **Methods enabled in two places.** `MONEI_PAYMENT_METHODS_ENABLED` filters the UI; the MONEI account must have the method active too. A method missing from `GET /payment-methods` is usually the dashboard, not the connector.
8. **SEPA is asynchronous.** `SUCCEEDED` can be days after submission and cannot be cancelled once submitted — treat like any async settlement: Order on webhook, no cancel button.

## Quick reference

- Bundle: `monei-enabler.js`, global `MoneiPaymentEnabler`.
- Secrets: `MONEI_API_KEY`, `MONEI_WEBHOOK_SECRET`, `CTP_CLIENT_SECRET`.
- Capture mode: per payment, `transactionType: 'SALE' | 'AUTH'` (cards only).
- Status → Order: create on `SUCCEEDED` (or `AUTHORIZED` if you capture later); never on redirect return.
- Webhook: `POST /webhooks/monei`, `MONEI-Signature: t=…,v1=…`, HMAC-SHA256 over raw body.
- Bizum test: `+34500000000`, amount < €5.
- Docs for agents: `https://docs-mcp.monei.com/mcp`, `https://docs.monei.com/llms.txt`, OpenAPI `https://js.monei.com/api/v1/openapi.json`.
