# KHQRPay code samples

The samples from the [KHQRPay docs](https://khqrpay.dev/docs), exactly as published there. The keys come from environment variables: `KHQRPAY_SECRET_KEY` (or `KHQRPAY_TEST_KEY` for a `kp_test_` key) and `KHQRPAY_WEBHOOK_SECRET`.

## Create a payment (cURL)

```bash
curl https://khqrpay.dev/v1/payments \
  -H "Authorization: Bearer $KHQRPAY_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: INV-2048-attempt-1" \
  -d '{"amount": 16.88, "currency": "USD", "reference_id": "INV-2048"}'
```

## Simulate a test payment (cURL)

```bash
curl https://khqrpay.dev/v1/test/payments/pFQ3xV8aN2rL7kTm9sB4cWd6hJ1yZ0eU/simulate \
  -H "Authorization: Bearer $KHQRPAY_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "paid"}'
```

## Create a payment and verify a webhook with the Node.js SDK

```js
import KHQRPay, { webhooks } from "khqrpay";

const khqrpay = new KHQRPay(process.env.KHQRPAY_SECRET_KEY);

// Create a payment, then send the customer to payment.checkout_url
const payment = await khqrpay.payments.create(
  { amount: "16.88", currency: "USD", reference_id: "INV-2048" },
  { idempotencyKey: "INV-2048-attempt-1" },
);

// In your webhook route, on the raw body. It throws on a bad signature: catch that and answer 400.
// Next.js: rawBody = await request.text(), signatureHeader = request.headers.get("x-khqrpay-signature")
// Express, with express.raw() on the route: rawBody = req.body, signatureHeader = req.get("X-KHQRPay-Signature")
const event = webhooks.constructEvent(rawBody, signatureHeader, process.env.KHQRPAY_WEBHOOK_SECRET);
```

## Create a payment and verify a webhook with the Python SDK

```python
import os
import khqrpay

client = khqrpay.KHQRPay(os.environ["KHQRPAY_SECRET_KEY"])

# Create a payment, then send the customer to payment["checkout_url"]
payment = client.payments.create(
    {"amount": "16.88", "currency": "USD", "reference_id": "INV-2048"},
    idempotency_key="INV-2048-attempt-1",
)

# In your webhook view, on the raw body. It raises KHQRPayError on a bad signature: catch that and answer 400.
event = khqrpay.webhooks.construct_event(raw_body, signature_header, os.environ["KHQRPAY_WEBHOOK_SECRET"])
```

## Create a payment and verify a webhook with the PHP SDK

```php
$khqrpay = new \KHQRPay\Client(getenv('KHQRPAY_SECRET_KEY'));

// Create a payment, then send the customer to $payment['checkout_url']
$payment = $khqrpay->payments->create(
    ['amount' => '16.88', 'currency' => 'USD', 'reference_id' => 'INV-2048'],
    ['idempotency_key' => 'INV-2048-attempt-1']
);

// In your webhook route, on the raw body. It throws \KHQRPay\KHQRPayError on a bad signature: catch that and answer 400.
$event = \KHQRPay\Webhook::constructEvent($rawBody, $signatureHeader, getenv('KHQRPAY_WEBHOOK_SECRET'));
```

## Verify a webhook signature without an SDK (JavaScript)

```js
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
```

## Webhook endpoint: Node.js with Express

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

## Webhook endpoint: Next.js route handler

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

## Webhook endpoint: PHP with Laravel

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

## Webhook endpoint: Python with Flask

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
