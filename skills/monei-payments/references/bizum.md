---
name: bizum-payment-method
description: Bizum as a payment method through MONEI — who can pay with it, the two flows (request-to-pay in the bank app vs redirect), why there is no authorize/capture, limits and refund window, how authentication works (it is SCA by construction), and the test-mode amount bands. Read when designing or debugging any checkout that offers Bizum.
when_to_use:
  - "Offering Bizum in a commercetools storefront, Checkout application, or agent checkout"
  - "Deciding whether Bizum fits a cart (customer country, amount, capture needs)"
  - "Debugging a Bizum payment stuck in PENDING, declined, or 'phone not registered'"
metadata:
  contentType: REFERENCE
  area:
    - payments
    - bizum
    - monei
---

# Bizum through MONEI

Bizum is the payment scheme run by the Spanish banks. A customer links their Spanish IBAN to their mobile number once, in their own banking app; after that they pay online by confirming a push in that app. MONEI acquires Bizum natively — the transaction is authorised and settled through MONEI, not through a third party — which is why it is the only way to offer Bizum inside a commercetools connector.

## Who can pay

Customers with a **bank account held in Spain** that is enrolled in Bizum. Do not show the Bizum button to carts whose billing country is not `ES`; it will only fail at the last step. Cards (and wallets) stay as the fallback for everyone else.

## The two flows

| Flow | What the customer sees | When MONEI uses it |
|---|---|---|
| **Request-to-pay (RTP)** | enters their phone number in the Bizum overlay on your page, confirms the push notification in their banking app, your page updates | default on modern integrations |
| **Redirect** | is sent to a Bizum/bank page, authenticates there, returns to your `completeUrl` | some banks / some amounts; the connector returns a `redirectUrl` and you navigate to it |

Your code must handle both: the payment is not done when the overlay closes or when the customer lands on `completeUrl`. It is done when the MONEI payment reaches `SUCCEEDED`, which arrives on the `callbackUrl` webhook. Give the return page a polling fallback on `GET /payments/:id`, and set a timeout after which the customer can retry or pick another method.

## No authorize / capture

Bizum is **always an immediate sale**. There is no `AUTHORIZED` state and no capture step. If the storefront's checkout is designed around "authorise at checkout, capture on shipment", branch on method: `transactionType: 'AUTH'` for cards, `'SALE'` for Bizum. Trying to capture a Bizum payment returns an error; trying to design a partial-capture flow for it is a design error.

Once `SUCCEEDED`, the only reversal is a **refund**.

## Limits

- **MONEI imposes no maximum** on online Bizum transactions and no monthly cap on the number of purchases.
- The **issuing bank** may apply its own per-operation or daily limit, and the customer must have funds. A high-ticket decline is almost always the bank, not the integration — offer the card fallback rather than opening a ticket.
- Test mode is different: see *Test bands* below.

## Refunds

- Full or partial, within **180 days** of the original payment.
- A refund can fail if the customer has since disconnected Bizum or re-linked their phone number to another IBAN, or during an issuer incident. Surface the failure and fall back to a manual transfer rather than retrying in a loop.
- Refunds go through the processor (`POST /payments/:id/refund`) and land as a `Refund` transaction on the commercetools Payment via webhook.

## Authentication — what "SCA" means here

Bizum has no card number to steal and no 3DS challenge to bolt on: the customer authenticates **inside their bank's app** with the bank's own factors (device + PIN/biometrics). That is Strong Customer Authentication by construction, and it produces a bank-side record of who authorised what and when. For an agent-driven checkout this is the important property: the human approval step *is* the payment authorisation, executed by the payer's bank, and MONEI's payment record (id, status, timestamps, `orderId`) is the receipt. See [agentic-checkout.md](./agentic-checkout.md).

## Test bands

Test mode uses one phone number and selects the outcome by amount:

| Amount | Result | Flow |
|---|---|---|
| < €5 | approved (`E000`) | RTP |
| €5 – €10 | declined (`E506`, "error during payment authorization") | RTP |
| €10 – €15 | approved (`E000`) | redirect |
| > €15 | "phone number is not registered in Bizum" | — |

Phone: **`+34 500 000 000`**. Use the bands deliberately — one cart per branch gives you the approved-RTP, declined, approved-redirect and hard-failure paths without any mocking. And remember the corollary: **on the happy path only amounts under €5 succeed in test mode**, so a €49.90 test cart failing is expected, not a bug.

## Fees (for the requirements conversation)

Flat percentage plus a fixed amount per successful transaction plus a small acquiring fee; MONEI does not charge for refunds. Current numbers are on [docs.monei.com/manage-account/monei-fees](https://docs.monei.com/manage-account/monei-fees) — quote from there, not from memory.

## Related

- Bizum **Payouts** (sending money *to* a customer's phone) exist as a gated feature paid from a prefunded balance — relevant for marketplaces and refunds-outside-the-window, not for checkout.
- Method reference: [docs.monei.com/payment-methods/bizum](https://docs.monei.com/payment-methods/bizum)
- Test data: [docs.monei.com/testing](https://docs.monei.com/testing)
