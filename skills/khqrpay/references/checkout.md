# Hosted checkout and QR image

`checkout_url` opens a mobile-first page in your store’s language, Khmer or English, and customers can switch with one tap. It shows:

- The KHQR card with the amount and the bank account holder’s name from the QR. On a converted payment, also your price and the rate.
- A countdown, live status and your support email.
- On phones, an “Open ABA Mobile” button in your brand colour, on live payments.
- A “Save QR image” button that saves the whole KHQR card as a PNG, for a customer who pays from another bank’s app. On an iPhone it opens the Share menu. Inside some apps’ own browsers it saves the QR alone.

Your logo and store name appear on payment links (`/l/…`), not here. At 0:00 the QR disappears while we check with the bank for a last-second payment. Every page is served with HSTS and a Content-Security-Policy, and can’t be framed (`X-Frame-Options: DENY`), so link or redirect to the checkout rather than embedding it.

### Redirects

Set success and failure URLs (https only) in *Store settings*. When the payment ends while the checkout is open, or the checkout is opened within 10 minutes after the payment ended, the page shows a *Continue* link and, 2.5 seconds later, sends the customer to your URL with `status` (success or failed), `payment_id` and `reference_id` appended:

**Success redirect**

```
https://shop.example.com/thanks?status=success&payment_id=pFQ3xV8aN2rL7kTm9sB4cWd6hJ1yZ0eU&reference_id=INV-2048
```

- **A hint, not proof.** Confirm with the webhook or `GET /v1/payments/:id` before you fulfil: anyone can type your success URL with these parameters.
- **Old links.** A checkout opened more than 10 minutes after the payment ended shows how it ended and stays there, so an old link doesn’t send anyone to your site.
- **Testing.** Keep the test payment’s checkout open while you simulate the result, or open it within 10 minutes.
- **Late payments.** A customer sent to your failure URL can still pay late, so expect a later `payment.completed`.
- **Listening after expiry.** An expired checkout that isn’t sending the customer to your failure URL keeps listening for a late approval until 30 minutes after `expires_at`. That happens when you set no failure URL, or when the checkout was opened more than 10 minutes after the QR expired. If the bank confirms, the checkout switches to paid and sends the customer to your success URL, if you set one.

### Your own payment screen

Render `qr_string` at error-correction level H on white with a quiet zone, show the amount, and count down to `expires_at`, reading it again after a scan, which can move it. Hide the QR at 0:00. Poll your own backend, never our API from the browser. Or use the ready-made card below.

## QR image

`GET /api/render/khqr/<percent-encoded qr_string>.svg` returns a ready-made KHQR card for any valid KHQR, not only KHQRPay’s. It needs no API key, so it works in an `<img>`, an email or an invoice.

![Rendered live from a test QR, which banks won’t pay. Open the SVG](https://khqrpay.dev/api/render/khqr/00020101021229210017khqrpay_test%40test520459995303840540516.885802KH5913Nita%20Pharmacy6010Phnom%20Penh62360120pFQ3xV8aN2rL7kTm9sB40708TESTMODE630494E5.svg)

Rendered live from a test QR, which banks won’t pay. [Open the SVG](https://khqrpay.dev/api/render/khqr/00020101021229210017khqrpay_test%40test520459995303840540516.885802KH5913Nita%20Pharmacy6010Phnom%20Penh62360120pFQ3xV8aN2rL7kTm9sB40708TESTMODE630494E5.svg)

| Property | Value |
| --- | --- |
| Size | 300 × 430 SVG: red KHQR header, payee, amount and the QR at error-correction level H |
| Payee name | The bank account holder’s name from the QR (tag 59). The hosted checkout’s card prints the same name. In test mode, tag 59 is your store’s name, so a live card can read differently from your tests |
| `Content-Type` | image/svg+xml; charset=utf-8 |
| `Cache-Control` | public, max-age=31536000, immutable: the same string always gives the same image |
| `Access-Control-Allow-Origin` | \* |
| `Content-Security-Policy` | default-src 'none'; style-src 'unsafe-inline'; img-src data: |
| Rate limit | 120 renders per minute per IP. Repeats served from a CDN or browser cache don’t count. Over it: 429 rate_limited with Retry-After |
| Errors | 400 invalid_khqr: bad CRC, malformed TLV, missing required tags or over 512 characters. Broken percent-encoding gets a plain 400 HTML page before it reaches the API |

End the path in `.png` instead for the QR alone, about 800 pixels square, black on white with a quiet zone. Bank apps’ “scan from photo” pickers can open it and can’t open an SVG, so it’s the one to offer a customer paying on their own phone. The same cache, CORS header, limit and errors apply.

**JavaScript**

```js
const src = `https://khqrpay.dev/api/render/khqr/${encodeURIComponent(payment.qr_string)}.svg`;
```

**HTML**

```html
<img src="https://khqrpay.dev/api/render/khqr/00020101021229210017khqrpay_test%40test520459995303840540516.885802KH5913Nita%20Pharmacy6010Phnom%20Penh62360120pFQ3xV8aN2rL7kTm9sB40708TESTMODE630494E5.svg"
     alt="KHQR: pay $16.88 to Nita Pharmacy" width="300" height="430">
```

For AI tools: every docs page is listed in [llms.txt](https://khqrpay.dev/llms.txt).
