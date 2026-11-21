---
name: khqrpay
description: "Use this skill when integrating KHQR payments in Cambodia through the KHQRPay API: creating a payment from a server, sending the customer to the hosted checkout or rendering the KHQR from qr_string, verifying signed webhooks, building in test mode with kp_test_ keys and the simulate endpoint, and going live with kp_live_ keys. Covers the REST API and the official Node.js, Python and PHP SDKs."
license: Proprietary. LICENSE.txt has complete terms
---

# KHQRPay

KHQRPay is a REST API for KHQR payments in Cambodia. The merchant’s server creates a payment, the customer scans the KHQR with any Cambodian banking app, and the money goes straight to the merchant’s own ABA account. KHQRPay never holds funds. Base URL: `https://khqrpay.dev`. JSON in and out.

## The flow

1. Keep the secret key on the server. Read it from an environment variable and send it as `Authorization: Bearer <secret key>`. Never send the secret key or a webhook signing secret to a browser or a mobile app.
2. Create the payment from the server: `POST /v1/payments` with an `Idempotency-Key` header and a body such as `{"amount": "16.88", "currency": "USD", "reference_id": "INV-2048"}`. Use one key per checkout attempt (`INV-2048-attempt-1`), not per order. Send `reference_id` as a string.
3. Send the customer to `checkout_url`, or render `qr_string` as a QR code (error-correction level H) with the amount and a countdown to `expires_at`. Re-read `expires_at`; never hard-code a duration.
4. Fulfil the order only on a verified `payment.completed` webhook. Never on the checkout redirect, on `payment.scanned` or on a status the browser reports. If the order was priced in a different currency from the ABA link’s, check `requested_amount` and `requested_currency`.
5. Handle late payments. A live payment that expired can still be paid: KHQRPay then sends `payment.completed` with `data.payment.late` set to `true`, after `payment.expired`, though deliveries can arrive in either order. Honour it. For anything you can’t undo, such as cancelling an order or releasing its stock, wait at least 70 minutes after the payment’s `expires_at` (read it with `GET /v1/payments/:id`: events don’t carry it, and a scan moves it), then re-read the payment and act only if it’s still `expired` and `monitoring_gap` isn’t true. Never let a `payment.expired` undo an order already marked paid: events can arrive in either order.
6. Build and test with a `kp_test_` key and a test webhook endpoint. Drive payments with `POST /v1/test/payments/:id/simulate` and `{"status": "paid"}`; simulate `expired`, then `paid`, for a late payment. For an event with `livemode: false` or `test: true`, answer 2xx and fulfil nothing. Webhook endpoints must be https addresses on the public internet; on a development machine, use an HTTPS tunnel.
7. Go live with a `kp_live_` key and a live webhook endpoint. Read [references/going-live.md](references/going-live.md) first.

## Rules that bite

- **The raw body.** The signature covers the exact bytes. Read the raw request body and verify `X-KHQRPay-Signature` (`t=<unix seconds>,v1=<hex>`, the HMAC-SHA256 of `<t>.<raw body>` keyed with the endpoint’s `whsec_` secret) before any JSON parser or body-parsing middleware touches it. Compare in constant time. A missing or malformed header gets a 400, never a crash.
- **A missing secret.** If the endpoint’s signing secret is unset or empty, refuse every event: HMAC accepts an empty key, so anyone could sign one.
- **5-minute tolerance.** Reject a timestamp more than 5 minutes from the current time. The SDKs’ `constructEvent` (Python: `construct_event`) checks both and throws on a bad signature: catch that and answer 400.
- **Retries and duplicates.** Answer 2xx within 10 seconds and do slow work afterwards. Anything else is retried on a live endpoint, 9 attempts at most over about 16 hours. Events can arrive twice or out of order: deduplicate on the event `id`, never on the payment id.
- **429 and Retry-After.** On 429 `rate_limited`, wait `Retry-After` seconds, then retry. A `Retry-After` longer than a minute is an hour-long or day-long limit: fail the call instead of waiting. On 503 `temporarily_unavailable`, wait `Retry-After` seconds too. On 500, 502 and 503, retry with the same `Idempotency-Key`.
- **New Free accounts.** Until a Free account’s first real payment, it can leave fewer unpaid live QRs a day (see references/errors-and-limits.md). The real $0.01 payment in going live lifts it.
- **Never poll in a loop.** Reads are counted per account per day, so follow payments by webhooks. Read `GET /v1/payments/:id` only to check one payment, such as before cancelling an order.
- **Error codes.** Errors are `{"error": "<code>", "message": "<text>"}`. Branch on the code, never on the message.
- **Payment links.** A payment made through a KHQRPay payment link has `reference_id` `link:<id>`. If it isn’t one of your orders, answer 2xx.

## Reference files

Read the one the task needs, when it needs it:

- [references/quickstart.md](references/quickstart.md): A first test payment: a test key, one create call, a simulated payment and a signed webhook.
- [references/sdks.md](references/sdks.md): Node.js, Python and PHP packages that create payments and check webhook signatures for you.
- [references/payments.md](references/payments.md): Create a payment, read the payment object, and retrieve or list payments.
- [references/checkout.md](references/checkout.md): Send customers to the hosted checkout, or show the QR on your own page.
- [references/webhooks.md](references/webhooks.md): Signed payment events, checking signatures, late payments and replaying deliveries.
- [references/test-mode.md](references/test-mode.md): Build with kp_test_ keys and simulate every outcome, a late payment included.
- [references/going-live.md](references/going-live.md): Connect your ABA payment link, switch to live keys and make a real $0.01 payment.
- [references/errors-and-limits.md](references/errors-and-limits.md): Error codes, rate limits, plan quotas and how to check your usage.
- [references/payment-links.md](references/payment-links.md): No-code KHQR pages to share: prices in riel or dollars, receipts, webhooks and limits.
- [references/code-samples.md](references/code-samples.md): Working code to start from: create and simulate with cURL, the Node.js, Python and PHP SDKs, a signature check without an SDK, and webhook endpoints for Express, Next.js, Laravel and Flask.

For every field and endpoint, the [OpenAPI 3.1 description](https://khqrpay.dev/openapi.json). For everything in one page, the [API brief](https://khqrpay.dev/llms-full.txt).
