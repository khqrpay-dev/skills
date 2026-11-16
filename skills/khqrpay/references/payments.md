# Payments

To take a KHQR payment, send `POST /v1/payments` from your server with an `amount` and a `currency`, USD or KHR. You get back a `qr_string` to show as a QR and a `checkout_url` to send the customer to, and your [webhook](https://khqrpay.dev/docs/webhooks) endpoint gets `payment.completed` when it’s paid.

A payment is one KHQR for one amount, priced in dollars or riel. Read it back with `GET /v1/payments/:id`, or list a store’s payments with `GET /v1/payments`.

## Create a payment

`POST /v1/payments` returns `201 Created` with the [payment object](https://khqrpay.dev/docs/payments#object), or `200` when an idempotency key replays an earlier request.

| Parameter | Type | Description |
| --- | --- | --- |
| `amount` | number \| string | Required: USD amounts use up to 2 decimals; KHR amounts must be whole riel. The minimum is 0.01 USD or 1 KHR; a KHR amount on a USD link has a higher one (see Currency and conversion). |
| `currency` | string | The currency you price in: USD or KHR, case-insensitive; defaults to USD. When it isn’t your ABA link’s currency, KHQRPay converts the amount. |
| `reference_id` | string | Your order id as a string, up to 255 characters. Send a numeric id as a string ("2048"): a number returns invalid_request. Echoed in webhooks and redirects. |
| `metadata` | object | Any JSON object: up to 50 keys, 4 KB (counted in UTF-8 bytes, so Khmer text uses about 3 a character) and 8 levels deep. Numbers longer than 15 digits can lose precision, so send long ids as strings. |
| `idempotency_key` | string | Up to 255 characters, or send the Idempotency-Key header, which wins if you send both. Spaces around a key are ignored, and a blank key counts as none. See Idempotency below. |

**cURL**

```bash
curl https://khqrpay.dev/v1/payments \
  -H "Authorization: Bearer $KHQRPAY_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: INV-2048-attempt-1" \
  -d '{"amount": 16.88, "currency": "USD", "reference_id": "INV-2048"}'
```

**Node.js**

```js
const res = await fetch("https://khqrpay.dev/v1/payments", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.KHQRPAY_SECRET_KEY}`,
    "Content-Type": "application/json",
    // One key per checkout attempt, not per order: a replay returns the original payment
    // for 24 h, even after its QR expired.
    "Idempotency-Key": `${order.id}-attempt-${attempt}`,
  },
  // reference_id must be a string, so convert numeric order ids.
  body: JSON.stringify({ amount: order.total, currency: "USD", reference_id: String(order.id) }),
});
const payment = await res.json();
if (!res.ok) throw new Error(`${payment.error}: ${payment.message}`); // branch on payment.error

// Send the customer to the hosted checkout…
redirect(payment.checkout_url);
// …or render payment.qr_string yourself (error correction level H), counting down to payment.expires_at.
```

**PHP (Laravel)**

```php
// Laravel. config/services.php: 'khqrpay' => ['secret' => env('KHQRPAY_SECRET_KEY'), ...]
$payment = Http::withToken(config('services.khqrpay.secret'))
    ->withHeaders(['Idempotency-Key' => "{$order->id}-attempt-{$attempt}"])
    ->post('https://khqrpay.dev/v1/payments', [
        'amount' => $order->total,
        'currency' => 'USD',
        'reference_id' => (string) $order->id,
    ])
    ->throw()
    ->json();

return redirect()->away($payment['checkout_url']);
```

**Python**

```python
import os, requests

r = requests.post(
    "https://khqrpay.dev/v1/payments",
    headers={
        "Authorization": f"Bearer {os.environ['KHQRPAY_SECRET_KEY']}",
        "Idempotency-Key": f"{order.id}-attempt-{attempt}",
    },
    json={"amount": str(order.total), "currency": "USD", "reference_id": str(order.id)},
    timeout=15,
)
r.raise_for_status()
payment = r.json()
print(payment["checkout_url"])
```

### Currency and conversion

Your ABA payment link has one currency, and every QR on it asks for that currency: riel on a KHR link, dollars on a USD link.

When the `currency` you send isn’t your link’s, KHQRPay converts the amount at its own fixed rate, currently 4,000 riel per US dollar. KHQRPay sets this rate; it isn’t ABA’s or the National Bank’s. Dollars become whole riel and riel become cents, rounded half up, so 16.88 USD on a KHR link is a 67,520 riel QR:

**Amount fields: 16.88 USD on a KHR link**

```json
{
  "amount": "67520",
  "currency": "KHR",
  "requested_amount": "16.88",
  "requested_currency": "USD",
  "settlement_amount": "67520",
  "settlement_currency": "KHR",
  "conversion_rate": 4000
}
```

- `amount` and `currency`, on the payment and in webhooks, are what the payer pays, in your link’s currency. `settlement_amount` and `settlement_currency` repeat them.
- `requested_amount` and `requested_currency` are what you sent, and `conversion_rate` is the rate used, in riel per US dollar. It’s null when nothing was converted.
- A payment keeps the rate it was created with. A new rate applies only to payments created after it.
- Test payments convert the same way once the store has an ABA link connected. Without a link, they keep the currency you sent.
- If you priced the order in the other currency, check it against `requested_amount` and `requested_currency`, not `amount`.

| You send | Your link | Minimum |
| --- | --- | --- |
| USD | USD | 0.01 USD |
| USD | KHR | 0.01 USD |
| KHR | KHR | 1 KHR |
| KHR | USD | 20 KHR: less converts to under 0.01 USD and returns amount_too_low |

Your link’s maximum applies to the converted amount, in the link’s currency.

### Idempotency

- Send an `Idempotency-Key` with every create. Keys belong to the store and the mode, and last 24 hours.
- A key can be up to 255 characters; a longer one returns `400 invalid_request`. Spaces around a key are ignored, and an empty or blank header counts as no key, so the request isn’t idempotent unless the body has an `idempotency_key`.
- The same key with the same amount, currency and reference_id returns the original payment with `200`, even if it has expired since. Metadata isn’t compared.
- The same key with a different amount, currency or reference_id returns `400 idempotency_conflict`. So does the same request after your link’s currency or, on a converted payment, KHQRPay’s rate has changed, because the converted amount no longer matches.
- So use one key per checkout attempt, like `INV-2048-attempt-1`, never just the order id: a customer who tries again after the QR expired needs a new payment.
- A create that repeats a key returns the stored payment, so it counts toward the account’s [reads for the day](https://khqrpay.dev/docs/errors-and-limits#limits). Once the day’s reads are used up, a create that repeats a key returns `429 rate_limited` until the day ends: send the create with a new key. A create that makes a new payment is never refused for this.

## The payment object

Create and retrieve return every field below, and [list](https://khqrpay.dev/docs/payments#list) items leave out `metadata`. Webhooks carry a subset in `data.payment`: id, status, amount, currency, requested_amount, requested_currency, settlement_amount, settlement_currency, conversion_rate, reference_id, metadata and approved_at, plus `late` on [late payments](https://khqrpay.dev/docs/webhooks#late) and `monitoring_gap` on webhooks sent after a [gap in our checks](https://khqrpay.dev/docs/webhooks#late).

**Payment object (live, just created)**

```json
{
  "id": "pFQ3xV8aN2rL7kTm9sB4cWd6hJ1yZ0eU",
  "object": "payment",
  "livemode": true,
  "status": "pending",
  "amount": "16.88",
  "currency": "USD",
  "requested_amount": "16.88",
  "requested_currency": "USD",
  "settlement_amount": "16.88",
  "settlement_currency": "USD",
  "conversion_rate": null,
  "reference_id": "INV-2048",
  "metadata": {
    "cart_id": "cart_7Hq2",
    "customer": "Sokha"
  },
  "qr_string": "000201010212…630494E5",
  "checkout_url": "https://khqrpay.dev/pay/pFQ3xV8aN2rL7kTm9sB4cWd6hJ1yZ0eU",
  "approved_at": null,
  "scanned_at": null,
  "created_at": "2026-10-23T02:14:09.000Z",
  "expires_at": "2026-10-23T02:17:09.000Z"
}
```

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | 32-character payment id, used in checkout_url and GET /v1/payments/:id |
| `object` | string | Always "payment" |
| `livemode` | boolean | true for payments made with a kp_live_ key, false for test payments |
| `status` | string | pending, scanned, paid, expired or failed (see below) |
| `amount` | string | What the payer pays, as a string: USD with 2 decimals ("16.88"), KHR in whole riel with no separators ("5000") |
| `currency` | string | "USD" or "KHR": your ABA link’s currency (see Currency and conversion) |
| `requested_amount` | string \| null | The amount you sent, in the same format. null only on older payments, created before this field existed |
| `requested_currency` | string \| null | The currency you sent. null only on older payments, like requested_amount |
| `settlement_amount` | string | Always the same as amount |
| `settlement_currency` | string | Always the same as currency |
| `conversion_rate` | number \| null | Riel per US dollar used to convert. null when nothing was converted |
| `reference_id` | string \| null | Your order id, as sent. A payment made through a payment link has link: and the link’s id |
| `metadata` | object \| null | Your metadata. A payment made through a payment link has null, or its price under pricing when the link converts currency |
| `qr_string` | string | The KHQR (EMVCo) payload. Render it at error-correction level H |
| `checkout_url` | string | The hosted checkout page for this payment |
| `approved_at` | string \| null | When the bank approved the payment (ISO 8601) |
| `scanned_at` | string \| null | When the customer scanned the QR |
| `created_at` | string | When the payment was created |
| `expires_at` | string | When the QR stops being offered. Live: about 3 minutes after creation (ABA’s checkout session), and a scan moves it to at least 2 minutes after the scan. Test: 5 minutes. Re-read it; don’t hard-code a duration |
| `monitoring_gap` | boolean | Only present, as true, when our checks on this payment fell over 5 minutes behind, every check on it with the bank failed for about a minute while it could still be paid, or the bank answered with a state we don’t know, so it may have been paid without us seeing it. See Late payments |

| Status | Final | Meaning |
| --- | --- | --- |
| `pending` | No | QR created, waiting for the customer to scan |
| `scanned` | No | Scanned; the customer is confirming in their bank app. Not paid yet. expires_at moves to at least 2 minutes after the scan |
| `paid` | Yes | The bank approved the payment and the money is in your ABA account. Counts toward your plan. Fires payment.completed |
| `expired` | Usually | Not paid in time, which includes a payment the customer declined or cancelled in their bank app. A live payment turns expired about 30 s after expires_at, a test payment within a few seconds of it. A late bank approval can still move it to paid |
| `failed` | Yes | Rare: we had to stop checking the payment because of an error on our side. Never a decline: a declined or cancelled payment ends as expired. In test mode, only Simulate sets it |

## Retrieve a payment

`GET /v1/payments/:id` returns the payment object with every field. Prefer webhooks. If you can’t use them, check only while a customer is paying, every few seconds at most, and stop at a final status. Don’t run a timer that checks payments around the clock.

**cURL**

```bash
curl https://khqrpay.dev/v1/payments/pFQ3xV8aN2rL7kTm9sB4cWd6hJ1yZ0eU \
  -H "Authorization: Bearer $KHQRPAY_SECRET_KEY"
```

| Parameter | In | Description |
| --- | --- | --- |
| `id` | path | The payment’s id. A payment of another store, or of the other mode (a test key asking for a live payment), returns 404 payment_not_found |

## List payments

`GET /v1/payments` returns the store’s payments in the key’s mode, newest first, one page at a time. List items leave out `metadata`: read it with `GET /v1/payments/:id`.

**cURL**

```bash
curl "https://khqrpay.dev/v1/payments?status=paid&limit=20" \
  -H "Authorization: Bearer $KHQRPAY_SECRET_KEY"

# The next page: send next_cursor back as cursor, with the same status
curl "https://khqrpay.dev/v1/payments?status=paid&limit=20&cursor=eyJjcmVhdGVkX2F0IjoiMjAyNi0xMC0yM1QwMjoxNDowOS4wMDBaIiwiaWQiOiJwRlEzeFY4YU4yckw3a1RtOXNCNGNXZDZoSjF5WjBlVSJ9" \
  -H "Authorization: Bearer $KHQRPAY_SECRET_KEY"
```

| Parameter | In | Description |
| --- | --- | --- |
| `status` | query | Optional: pending, scanned, paid, expired or failed. Anything else returns 400 invalid_status |
| `limit` | query | Optional: 1 to 100. Default 20 |
| `cursor` | query | Optional: next_cursor from the previous page. It only marks where that page ended, so send the same status with it. A malformed cursor returns 400 invalid_cursor |

**Response (data cut to one payment here)**

```json
{
  "object": "list",
  "data": [
    {
      "id": "pFQ3xV8aN2rL7kTm9sB4cWd6hJ1yZ0eU",
      "object": "payment",
      "livemode": true,
      "status": "paid",
      "amount": "16.88",
      "currency": "USD",
      "requested_amount": "16.88",
      "requested_currency": "USD",
      "settlement_amount": "16.88",
      "settlement_currency": "USD",
      "conversion_rate": null,
      "reference_id": "INV-2048",
      "qr_string": "000201010212…630494E5",
      "checkout_url": "https://khqrpay.dev/pay/pFQ3xV8aN2rL7kTm9sB4cWd6hJ1yZ0eU",
      "approved_at": "2026-10-23T02:14:38.000Z",
      "scanned_at": "2026-10-23T02:14:31.000Z",
      "created_at": "2026-10-23T02:14:09.000Z",
      "expires_at": "2026-10-23T02:17:09.000Z"
    }
  ],
  "has_more": true,
  "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi0xMC0yM1QwMjoxNDowOS4wMDBaIiwiaWQiOiJwRlEzeFY4YU4yckw3a1RtOXNCNGNXZDZoSjF5WjBlVSJ9"
}
```

`has_more` is true when older payments are left: send `next_cursor` back as `cursor` to get the next page. On the last page, `has_more` is false and `next_cursor` is null. Cursors don’t expire.

**Export to a spreadsheet.** Without code, *Export* on the dashboard’s *Payments* page downloads a CSV file of one Cambodia month: *Every payment* or *Daily totals*, live or test, for one store or all of the account’s stores. Each payment comes with its store, its payment link and its times in Cambodia time (UTC+7). A file holds up to 20,000 payments, newest first.

For AI tools: every docs page is listed in [llms.txt](https://khqrpay.dev/llms.txt).
