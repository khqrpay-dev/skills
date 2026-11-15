# Quickstart

1. Sign in with Google and create a store. You can leave the ABA payment link empty while you build in test mode.
2. Under *API keys*, create a test key (`kp_test_`). Under *Webhooks*, add a *Test* endpoint for your server.
3. Create a payment from your server and send the customer to its `checkout_url`.
4. Simulate `paid` from the dashboard or the API, and fulfil the order when `payment.completed` arrives.
5. To take real money, connect your ABA payment link, then create a live key and a *Live* endpoint. See [Going live](https://khqrpay.dev/docs/going-live).

In the dashboard’s *Payments* page, the Live/Test switch controls the list and the create action follows that choice. The dialog shows a read-only mode badge for confirmation; change the page’s Live/Test selection before opening the dialog when you need to create the other kind of payment.

## Authentication

Send your secret key as a bearer token on every request. Each key belongs to one store and one mode: `kp_live_` keys create real KHQRs on your ABA link, and `kp_test_` keys create simulated ones. After the prefix come 32 letters, digits, `-` or `_`. Keys are shown once and stored only as a SHA-256 hash. Keep them on your server, never in browser or mobile app code.

**Header**

```
Authorization: Bearer kp_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Without the header you get `401 unauthorized` with “Missing bearer token”. A wrong or revoked key gets “Invalid API key”. You can create a live key only after the store’s ABA payment link is connected. A store can have up to 10 active keys in each mode; revoke one you no longer use to make room.

For AI tools: every docs page is listed in [llms.txt](https://khqrpay.dev/llms.txt).
