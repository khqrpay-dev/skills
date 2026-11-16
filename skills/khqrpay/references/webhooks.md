# Webhooks

Add endpoints under *Webhooks*. Each endpoint has a mode: **live** endpoints receive events for live payments only, and **test** endpoints for test payments only. A new endpoint subscribes to all four events; *Edit events* changes that later and keeps the signing secret. When a payment changes status, we POST the event to every enabled endpoint of the payment’s mode that subscribes to it. A store can have up to 10 live endpoints and 3 test ones.

Endpoint URLs must use https and resolve to public internet addresses. We refuse private and internal addresses, such as `localhost`, `10.x.x.x` or `192.168.x.x` (a KHQRPay server in local development accepts them). To receive events on your own machine, use an HTTPS tunnel.

| Event | When |
| --- | --- |
| `payment.completed` | The payment is paid, including late approvals (late: true). Fulfil the order here |
| `payment.scanned` | The customer scanned the QR. Not paid yet. expires_at is now at least 2 minutes after the scan, so read it again |
| `payment.expired` | Not paid in time, which includes a declined or cancelled payment. Usually final, but see Late payments |
| `payment.failed` | Rare: we had to stop checking the payment because of an error on our side, or you simulated failed in test mode. Never sent for a decline |

**Request (body indented here; we send compact JSON and sign its exact bytes)**

```http
POST /webhooks/khqrpay HTTP/1.1
Content-Type: application/json
User-Agent: KHQRPay-Webhooks/1.0
X-KHQRPay-Event: payment.completed
X-KHQRPay-Signature: t=1792721679,v1=74b9251d77a1d64b64d5c6acb31792c673ee4a2dab6de3a0bc78afc2823935c3
X-KHQRPay-Delivery: c3a8f0d2-61b7-4e95-8d2a-7f4b9e0c1a36

{
  "id": "5b1e9c2a-7d4f-4e8b-9a3c-2f6d8e1b0c47",
  "type": "payment.completed",
  "livemode": true,
  "created": "2026-10-23T02:14:38.000Z",
  "data": {
    "payment": {
      "id": "pFQ3xV8aN2rL7kTm9sB4cWd6hJ1yZ0eU",
      "status": "paid",
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
      "approved_at": "2026-10-23T02:14:38.000Z"
    }
  }
}
```

| Field | Description |
| --- | --- |
| `id` | The event id. Every endpoint and every retry of the event gets the same id: deduplicate on it |
| `type` | One of the four events, also sent as X-KHQRPay-Event |
| `livemode` | false for test payments and Send test events |
| `created` | When the event happened (ISO 8601) |
| `data.payment` | id, status, amount, currency, requested_amount, requested_currency, settlement_amount, settlement_currency, conversion_rate, reference_id, metadata and approved_at, plus late: true on late payments and monitoring_gap: true on events sent after a gap in our checks |
| `test` | true on events sent with Send test, absent otherwise |

`amount` and `currency` are what the payer paid, in your ABA link’s currency. If you priced the order in the other currency, check it against `requested_amount` and `requested_currency` (see [Currency and conversion](https://khqrpay.dev/docs/payments#currency)).

### Delivery and retries

- Reply with any 2xx within 10 seconds. A timeout, a network error, a redirect (3xx; we don’t follow it) or any other status counts as a failure.
- A delivery gets up to 9 attempts: the first, then retries 1 m, 5 m, 15 m, 30 m, 1 h, 2 h, 4 h and 8 h apart, about 16 hours in total. Every attempt is signed afresh, so the 5-minute timestamp check still passes. A delivery to a test endpoint gets one attempt and no retries (see [Test mode](https://khqrpay.dev/docs/test-mode)).
- Events may arrive out of order or more than once. Deduplicate on the event `id`, make fulfilment idempotent, and check the payment’s current status before acting on an older event.
- Pausing an endpoint stops it: a retry that comes due while it’s paused is marked failed (“Not sent: the endpoint is paused”), and events that happen while it’s paused are never queued. After you turn it back on, use *Retry failed* for the stopped retries and `GET /v1/payments` to catch up on the rest.
- Use delivery logs to investigate recent attempts; keep your own records if you need a durable audit history. For a spreadsheet of a month’s payments, use *Export* on the dashboard’s *Payments* page.

### Behind Cloudflare or a firewall

If your site is behind Cloudflare or another firewall, its country block, bot protection or security level can stop our calls before they reach your server. The delivery log then shows HTTP 403 and “Blocked by your firewall”. Our calls come from Cloudflare’s network, from a location that varies, so some deliveries may get through while others are blocked. We can’t give you a fixed list of IP addresses to allow. Let the webhook through by its path instead:

- In Cloudflare, add a WAF custom rule that matches `POST` requests to your webhook’s path (for example `http.request.uri.path eq "/api/webhooks/khqrpay" and http.request.method eq "POST"`), with the action *Skip* for your other custom rules, the security level and the browser integrity check. Put it first, above any country rule.
- Open only that path, only for POST. Keep verifying [the signature](https://khqrpay.dev/docs/webhooks#verify): it’s what proves a call came from KHQRPay.
- Then use *Resend* or *Retry failed* on the blocked deliveries.

## Verify signatures

`X-KHQRPay-Signature` is `t=<unix seconds>,v1=<hex>`, where v1 is the HMAC-SHA256 of `` `${t}.${rawBody}` `` keyed with the endpoint’s signing secret. Secrets are `whsec_` followed by base64url characters, so they can contain `-` and `_`. Verify the raw body before you parse it, compare in constant time, and reject timestamps more than 5 minutes away from your clock.

Each handler below returns 400 for a missing or bad signature, acknowledges test events without fulfilling them, deduplicates on the event id and fulfils only on `payment.completed`. `saveEventOnce` is your code: insert the event id into a table with a unique key (`INSERT … ON CONFLICT DO NOTHING`) and fulfil new rows in a background job, as the Laravel tab does with `insertOrIgnore`. Make fulfilment idempotent per order too.

**Node.js (Express)**

```js
import crypto from "node:crypto";
import express from "express";
import { saveEventOnce } from "./orders.js"; // your code: INSERT … ON CONFLICT (event_id) DO NOTHING

const app = express();

function verifyKhqrPay(rawBody, header, secret, toleranceSec = 300) {
  if (!secret) return false; // unset: refuse everything, or anyone could sign with an empty key
  const h = String(header ?? "");
  const t = /(?:^|,)\s*t=(\d{1,12})\s*(?=,|$)/.exec(h)?.[1];
  const v1 = /(?:^|,)\s*v1=([0-9a-fA-F]{64})\s*(?=,|$)/.exec(h)?.[1];
  if (!t || !v1 || Math.abs(Date.now() / 1000 - Number(t)) > toleranceSec) return false;
  const expected = crypto.createHmac("sha256", secret).update(`${t}.${rawBody}`).digest();
  const given = Buffer.from(v1, "hex");
  return given.length === expected.length && crypto.timingSafeEqual(given, expected);
}

// express.raw keeps the exact bytes the signature covers. Register this route before
// app.use(express.json()), or the JSON parser reads the body first and every check fails.
// The limit is only a margin: events stay small, because metadata is capped at 4 KB.
app.post("/webhooks/khqrpay", express.raw({ type: "application/json", limit: "5mb" }), async (req, res) => {
  const raw = Buffer.isBuffer(req.body) ? req.body.toString("utf8") : ""; // not a Buffer unless the body is JSON
  if (!verifyKhqrPay(raw, req.get("X-KHQRPay-Signature") ?? "", process.env.KHQRPAY_WEBHOOK_SECRET)) {
    return res.status(400).send("Invalid signature");
  }
  const event = JSON.parse(raw);

  // Send-test events and test-mode payments: acknowledge, never fulfil.
  if (event.test === true || event.livemode !== true) return res.sendStatus(200);

  if (event.type === "payment.completed") {
    // Paid. Also when data.payment.late is true: the bank approved after payment.expired.
    // Every retry and every endpoint repeats event.id, so store it under a unique key
    // and fulfil the order in a background job, not in this request.
    await saveEventOnce(event.id, event.data.payment);
  }
  // payment.expired isn't always final: don't cancel anything you can't undo.
  res.sendStatus(200); // within 10 seconds
});

app.listen(3000);
```

**Next.js**

```ts
// app/webhooks/khqrpay/route.ts
import crypto from "node:crypto";
import { saveEventOnce } from "@/lib/orders"; // your code: INSERT … ON CONFLICT (event_id) DO NOTHING

function verifyKhqrPay(rawBody: string, header: string | null | undefined, secret: string, toleranceSec = 300): boolean {
  if (!secret) return false; // unset: refuse everything, or anyone could sign with an empty key
  const h = String(header ?? "");
  const t = /(?:^|,)\s*t=(\d{1,12})\s*(?=,|$)/.exec(h)?.[1];
  const v1 = /(?:^|,)\s*v1=([0-9a-fA-F]{64})\s*(?=,|$)/.exec(h)?.[1];
  if (!t || !v1 || Math.abs(Date.now() / 1000 - Number(t)) > toleranceSec) return false;
  const expected = crypto.createHmac("sha256", secret).update(`${t}.${rawBody}`).digest();
  const given = Buffer.from(v1, "hex");
  return given.length === expected.length && crypto.timingSafeEqual(given, expected);
}

export async function POST(req: Request) {
  const raw = await req.text(); // the exact body: verify before parsing
  if (!verifyKhqrPay(raw, req.headers.get("x-khqrpay-signature"), process.env.KHQRPAY_WEBHOOK_SECRET!)) {
    return new Response("Invalid signature", { status: 400 });
  }
  const event = JSON.parse(raw);

  // Send-test events and test-mode payments: acknowledge, never fulfil.
  if (event.test === true || event.livemode !== true) return new Response("ok");

  if (event.type === "payment.completed") {
    // Paid. Also when data.payment.late is true: the bank approved after payment.expired.
    // Every retry and every endpoint repeats event.id, so store it under a unique key
    // and fulfil the order in a background job, not in this request.
    await saveEventOnce(event.id, event.data.payment);
  }
  // payment.expired isn't always final: don't cancel anything you can't undo.
  return new Response("ok"); // within 10 seconds
}
```

**PHP (Laravel)**

```php
<?php
// app/Http/Controllers/KhqrPayWebhookController.php
// routes/api.php (Laravel 11+: run `php artisan install:api` first):
//   Route::post('/webhooks/khqrpay', \App\Http\Controllers\KhqrPayWebhookController::class);
// API routes skip the CSRF check (which would answer 419) and are served under /api, so the
// endpoint URL to add in the dashboard is https://shop.example.com/api/webhooks/khqrpay.
// config/services.php: 'khqrpay' => [..., 'webhook_secret' => env('KHQRPAY_WEBHOOK_SECRET')]

namespace App\Http\Controllers;

use App\Jobs\FulfilOrder;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class KhqrPayWebhookController extends Controller
{
    public function __invoke(Request $request)
    {
        $raw = $request->getContent(); // the exact body: verify before decoding
        $header = (string) $request->header('X-KHQRPay-Signature', '');
        if (! self::verify($raw, $header, (string) config('services.khqrpay.webhook_secret'))) {
            return response('Invalid signature', 400);
        }
        $event = json_decode($raw, true);

        // Send-test events and test-mode payments: acknowledge, never fulfil.
        if (($event['test'] ?? false) === true || ($event['livemode'] ?? false) !== true) {
            return response('ok', 200);
        }

        if ($event['type'] === 'payment.completed') {
            // Paid. Also when data.payment.late is true: the bank approved after payment.expired.
            // Every retry and every endpoint repeats the event id; the primary key drops copies.
            DB::transaction(function () use ($event) {
                if (DB::table('khqrpay_events')->insertOrIgnore(['id' => $event['id']]) === 1) {
                    FulfilOrder::dispatch($event['data']['payment']); // queued, not in this request
                }
            });
        }
        // payment.expired isn't always final: don't cancel anything you can't undo.

        return response('ok', 200); // within 10 seconds
    }

    public static function verify(string $raw, string $header, string $secret, int $tolerance = 300): bool
    {
        if ($secret === '') { // unset: refuse everything, or anyone could sign with an empty key
            return false;
        }
        $parts = [];
        foreach (explode(',', $header) as $item) {
            $pair = explode('=', trim($item), 2);
            if (count($pair) === 2) {
                $parts[$pair[0]] = $pair[1];
            }
        }
        $t = $parts['t'] ?? '';
        $v1 = strtolower($parts['v1'] ?? '');
        if (! ctype_digit($t) || ! preg_match('/^[0-9a-f]{64}$/', $v1) || abs(time() - (int) $t) > $tolerance) {
            return false;
        }

        return hash_equals(hash_hmac('sha256', $t.'.'.$raw, $secret), $v1);
    }
}
```

**Python (Flask)**

```python
import hashlib, hmac, json, os, re, time
from flask import Flask, request

from orders import save_event_once  # your code: INSERT … ON CONFLICT (event_id) DO NOTHING

app = Flask(__name__)


def verify_khqrpay(raw: bytes, header: str, secret: str, tolerance: int = 300) -> bool:
    if not secret:  # unset: refuse everything, or anyone could sign with an empty key
        return False
    parts = dict(item.strip().split("=", 1) for item in (header or "").split(",") if "=" in item)
    t, v1 = parts.get("t", ""), parts.get("v1", "")
    if not re.fullmatch(r"[0-9]{1,12}", t) or not re.fullmatch(r"[0-9a-fA-F]{64}", v1):
        return False
    if abs(time.time() - int(t)) > tolerance:
        return False
    expected = hmac.new(secret.encode(), t.encode() + b"." + raw, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, v1.lower())


@app.post("/webhooks/khqrpay")
def khqrpay_webhook():
    raw = request.get_data()  # the exact body: verify before parsing
    if not verify_khqrpay(raw, request.headers.get("X-KHQRPay-Signature", ""), os.environ["KHQRPAY_WEBHOOK_SECRET"]):
        return "Invalid signature", 400
    event = json.loads(raw)

    # Send-test events and test-mode payments: acknowledge, never fulfil.
    if event.get("test") is True or event.get("livemode") is not True:
        return "ok", 200

    if event["type"] == "payment.completed":
        # Paid. Also when data.payment.late is true: the bank approved after payment.expired.
        # Every retry and every endpoint repeats event["id"], so store it under a unique key
        # and fulfil the order in a background job, not in this request.
        save_event_once(event["id"], event["data"]["payment"])

    # payment.expired isn't always final: don't cancel anything you can't undo.
    return "ok", 200  # within 10 seconds
```

On a staging server wired to a test endpoint, you can drop the livemode check to run your fulfilment code on simulated payments. Keep it in production.

## Late payments

Since 22 Oct 2026 an ABA QR stays payable at the bank for 30 minutes after it was created (the QR’s own expiry, KHQR tag 99), longer than the checkout session behind `expires_at`. So after a live payment expires, we keep checking it until the QR’s own expiry: at least 10 and at most 60 minutes after `expires_at`.

If the bank approves in that window, the payment moves from `expired` to `paid`, you get `payment.completed` after `payment.expired`, and `data.payment.late` is `true`:

**Late payment.completed**

```json
{
  "id": "9e4d71c3-7d4f-4e8b-9a3c-2f6d8e1b0c47",
  "type": "payment.completed",
  "livemode": true,
  "created": "2026-10-23T02:23:21.000Z",
  "data": {
    "payment": {
      "id": "pFQ3xV8aN2rL7kTm9sB4cWd6hJ1yZ0eU",
      "status": "paid",
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
      "approved_at": "2026-10-23T02:23:21.000Z",
      "late": true
    }
  }
}
```

- The money arrived: fulfil the order, even if your checkout already said the payment expired.
- Don’t treat `expired` as final for anything you can’t undo (cancelling the order, releasing a one-off item, refunding) until an hour after `expires_at`.
- Deduplicate on the event `id`, not the payment id: a handler that skips events for a payment it has already seen drops this one.
- Try it in test mode: simulate `expired`, then `paid`.

A live payment gets `monitoring_gap: true` when:

- Our checks on it fall more than 5 minutes behind, for example during an outage on our side.
- Its checks with the bank all fail for about a minute (55 seconds from the first failed one) while it can still be paid, before or after `expires_at`. Shorter failures aren’t flagged.
- The bank answers for it with a state we don’t know.

ABA only shows an approval while its session is fresh, so a payment made in that gap may never be reported.

- **Webhooks** sent after the gap carry the flag too, but setting it sends no event, so a `payment.expired` that went out earlier won’t show it: re-read `GET /v1/payments/:id` before you cancel an order.
- **Not paid?** If the flag is there and the status isn’t `paid`, treat the payment as unknown and check your ABA app.
- **Paid?** On a paid payment it changes nothing.

ABA can report an approval that comes after its 3-minute session: in a test on 3 November 2026, one showed about 4 minutes after the session ended. It read as unpaid again within 5 minutes, so the late watch is designed to catch these payments, not guaranteed to.

## Testing and replaying deliveries

Each endpoint under *Webhooks* has these tools. *Recent deliveries* logs every delivery with its time, attempt number, endpoint and HTTP status, plus the start of your response when an attempt failed. It lists the newest first and pages back through older deliveries.

| Action | What it does |
| --- | --- |
| **Send test** | Sends the event you pick right away and shows your endpoint’s HTTP answer. The payment is made up (id pay_test_…, reference_id test_order, $1.00, status matching the event), livemode is false and the body has "test": true. Logged, never retried. The log keeps each endpoint’s newest 20; a test endpoint keeps only its newest 100 deliveries, Send tests included |
| **Retry now** | On a delivery still backing off (“retrying · next attempt …”): sends the next attempt immediately |
| **Resend** | On a finished delivery, succeeded or failed: sends the same event again, with the same event id and a fresh signature. A Send-test delivery is sent once more and never retried |
| **Retry failed (last 24 h)** | Queues again the failed deliveries of events from the last 24 hours, for example after an outage or a pause: 100 a click, the least recently tried first, and up to 500 an hour across your stores, counted apart for live and test endpoints. Click again for the next ones; for older ones, use Resend. Send-test events aren’t included |

**Send test body**

```json
{
  "id": "1f7c2e94-7d4f-4e8b-9a3c-2f6d8e1b0c47",
  "type": "payment.completed",
  "livemode": false,
  "created": "2026-10-23T02:14:09.000Z",
  "data": {
    "payment": {
      "id": "pay_test_9sB4cWd6hJ1yZ0eU",
      "status": "paid",
      "amount": "1.00",
      "currency": "USD",
      "requested_amount": "1.00",
      "requested_currency": "USD",
      "settlement_amount": "1.00",
      "settlement_currency": "USD",
      "conversion_rate": null,
      "reference_id": "test_order",
      "metadata": null,
      "approved_at": "2026-10-23T02:14:09.000Z"
    }
  },
  "test": true
}
```

| Header | Value |
| --- | --- |
| `Content-Type` | application/json |
| `User-Agent` | KHQRPay-Webhooks/1.0 |
| `X-KHQRPay-Event` | The event type, same as type in the body |
| `X-KHQRPay-Signature` | t=\<unix seconds>,v1=\<hex HMAC-SHA256>, signed afresh on every attempt, resends included |
| `X-KHQRPay-Delivery` | The delivery’s id (one event to one endpoint). Its retries and resends repeat it; each Send test gets a new one. Quote it to support, but deduplicate on the body’s id |

To run the whole flow, fulfilment code included, create payments with a `kp_test_` key and [simulate](https://khqrpay.dev/docs/test-mode) them: their events reach your test endpoints, signed like live ones.

For AI tools: every docs page is listed in [llms.txt](https://khqrpay.dev/llms.txt).
