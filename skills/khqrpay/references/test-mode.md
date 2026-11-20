# Test mode

- Requests with a `kp_test_` key never contact the bank, never count toward your quota and need no ABA link. They keep working while the store’s live payments are paused.
- The QR is a well-formed KHQR that banks won’t pay, and the hosted checkout shows a test banner. A test payment’s hosted page, its title and its link preview are marked as a test, with a grey TEST MODE card in place of the red KHQR one, and “Test payment: no money was sent” when paid.
- Test payments last 5 minutes and turn expired within a few seconds of `expires_at`. A finished one is deleted 30 days after its last change, or 2 days after it once the account has 1,000 newer ones, and its webhook delivery log goes with it. Live payments aren’t deleted this way: they’re kept while your account is open, and after it closes for as long as the law requires.
- Their events go to test endpoints only, with `livemode: false`. A test endpoint’s log keeps only its newest 100 deliveries: an older one is deleted as newer ones are sent. Each delivery to a test endpoint is tried once and never retried; *Resend* and *Retry failed* send it once more, with metadata null once its test payment has been deleted. A store can have up to 3 test endpoints.
- Idempotency keys are per mode, so a test key never replays a live payment.
- Test-mode reads stop once the account’s [reads for the day](https://khqrpay.dev/docs/errors-and-limits#limits), test and live together, reach half its day (10 MB of Free’s 20), so a test script left running can’t use up the half that live reads need. Simulate in the dashboard keeps working.

Move a test payment through its lifecycle with *Simulate* on its page in the dashboard, or with `POST /v1/test/payments/:id/simulate` and a status of `scanned`, `paid`, `expired` or `failed`. The response is the updated payment. A live key gets `400 invalid_request`, and a live or unknown payment id `404 payment_not_found`.

**cURL**

```bash
curl https://khqrpay.dev/v1/test/payments/pFQ3xV8aN2rL7kTm9sB4cWd6hJ1yZ0eU/simulate \
  -H "Authorization: Bearer $KHQRPAY_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "paid"}'
```

To test a late payment, simulate `expired` and then `paid` (in the dashboard, *Simulate late approval*): you get `payment.completed` with `late: true`. Simulating a status the payment can’t move to, such as `paid` after `failed`, changes nothing.

## Try a late WooCommerce payment

1. Place a KHQR order on your [WooCommerce test shop](https://khqrpay.dev/guides/khqr-website-without-plugin#woocommerce). Close the QR tab after it opens so checkout cannot return to the shop while you test the webhook.
2. In *Payments → Test*, open the matching payment. Choose *Simulate → Expired*, then confirm the order is still *Pending payment* in WooCommerce.
3. Choose *Simulate late approval*. In *Webhooks*, confirm the test endpoint delivered `payment.completed`. The event body carries `late: true`. Refresh the order in WooCommerce and look for *Processing* (or *Completed* for a downloadable order) and a KHQRPay payment note.

For AI tools: every docs page is listed in [llms.txt](https://khqrpay.dev/llms.txt).
