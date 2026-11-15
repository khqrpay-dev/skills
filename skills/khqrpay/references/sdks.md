# Official SDKs

Optional: the API is plain HTTPS and JSON, so any language works without one. The SDKs have no dependencies, verify webhook signatures for you, and turn API errors into one error type with the code.

- **Node.js**: `npm install khqrpay`, [View on npm](https://www.npmjs.com/package/khqrpay), [Read the Node.js guide](https://khqrpay.dev/guides/khqr-api-nodejs)
- **Python**: `pip install khqrpay`, [View on PyPI](https://pypi.org/project/khqrpay/), [Read the Python guide](https://khqrpay.dev/guides/khqr-api-python)
- **PHP**: `composer require khqrpay/khqrpay-php`, [View on Packagist](https://packagist.org/packages/khqrpay/khqrpay-php), [Read the PHP guide](https://khqrpay.dev/guides/khqr-api-php-laravel)

**Node.js**

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

**Python**

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

**PHP**

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

For AI tools: every docs page is listed in [llms.txt](https://khqrpay.dev/llms.txt).
