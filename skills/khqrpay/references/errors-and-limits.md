# Errors and limits

An API error is an HTTP status and a JSON body with a machine-readable `error` code and a `message`: branch on the code, never on the message. Over a rate limit you get `429 rate_limited` with a `Retry-After` header saying how many seconds to wait, and at your plan’s limit live creates get `402 quota_exceeded`; `GET /v1/usage` shows how much of the plan is left.

The sections below list every error code, the rate limits, the plans and their quotas, and how to check your usage.

## Errors

Errors return an HTTP status and a JSON body with a machine-readable `error` code and a human-readable `message`. Branch on `error`: codes are stable, but message text can change at any time. Every `/v1` response from KHQRPay is JSON, including `404 not_found` for an unknown path and `405 method_not_allowed` (with an `Allow` header) for a wrong method. Three things aren’t: Cloudflare answers a URL with broken percent-encoding, such as `%ZZ`, with its own `400` HTML page before it reaches us; a trailing slash, as in `/v1/payments/`, gets a `308` redirect to the same path without it; and `OPTIONS` gets an empty `204`.

**Response · 400**

```json
{ "error": "amount_too_low", "message": "Minimum amount is 0.01 USD" }
```

| Code | HTTP | When |
| --- | --- | --- |
| `invalid_request` | 400 | The body isn’t valid JSON, a field failed validation (the message names it), or the Idempotency-Key header is over 255 characters |
| `invalid_amount` | 400 | amount isn’t a number (for example "abc"), a USD amount has more than 2 decimals, or a KHR amount isn’t whole riel |
| `amount_too_low` | 400 | amount is below 0.01 USD or 1 KHR, or a KHR amount on a USD link converts to less than 0.01 USD (see Currency and conversion) |
| `amount_too_high` | 400 | Above your ABA link’s maximum, checked after conversion, in the link’s currency |
| `idempotency_conflict` | 400 | Idempotency-Key reused within 24 h with different requested or settlement details, or a different reference_id |
| `invalid_status` | 400 | Unknown status in the list filter |
| `invalid_cursor` | 400 | The list cursor is malformed. Send next_cursor back exactly as you got it |
| `payment_link_disabled` | 400 | The store’s live payments are paused, ABA disabled the payment link or its KHQR, or the link’s currency changed in ABA since you connected it |
| `invalid_khqr` | 400 | The QR image endpoint got a malformed payload or a bad CRC |
| `account_blocked` | 403 | Account access is suspended; contact support |
| `api_key_blocked` | 403 | API key blocked after sustained excessive traffic; contact support |
| `https_required` | 403 | The call used plain http. API calls aren’t redirected, so a POST can’t turn into a GET: send it to https://khqrpay.dev |
| `unauthorized` | 401 | Missing bearer token, or an invalid or revoked API key |
| `quota_exceeded` | 402 | Your plan’s paid-transaction limit for this period is reached, or a Free account’s ABA link has used this month’s Free transactions. Open live QRs that hold a place for this create count toward both (see /docs/errors-and-limits#usage). Live payments only |
| `not_found` | 404 | No such /v1 endpoint |
| `payment_link_not_found` | 404 | No ABA payment link is connected to the store, or ABA no longer has the connected link (deleted or invalid). Reconnect it in Store settings (live payments) |
| `payment_not_found` | 404 | No such payment for this store and mode |
| `method_not_allowed` | 405 | Wrong HTTP method for the endpoint. The Allow header lists the right ones |
| `payload_too_large` | 413 | Body over 32 KB |
| `rate_limited` | 429 | Too many requests, too many QRs open or left unpaid for the day, the account’s reads for the day are used up, API keys that don’t exist were looked up over 30 times in a minute from your network (then its calls are refused until the minute ends, except with a key that has worked recently), or very many calls at once from your network used keys not yet seen. Wait Retry-After seconds, then retry; when Retry-After is longer than a minute it’s a day-long limit, so fail the call instead of waiting |
| `internal_error` | 500 | Something broke on our side. Safe to retry with the same Idempotency-Key |
| `payment_provider_error` | 502 | ABA’s checkout didn’t answer as expected. Safe to retry with the same Idempotency-Key |
| `temporarily_unavailable` | 503 | We can’t reach our database, or a limit at the edge couldn’t be checked. Wait Retry-After seconds, then retry with the same Idempotency-Key |

## Rate limits and plans

- Per API key: 60 creates (simulate included) and 600 reads per minute, in fixed one-minute windows. Successful responses and 429s carry `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset` (Unix seconds). Ahead of the API, our network protection lets up to 6,000 requests a minute through from one IP address, 700 a minute with one API key (more than the key’s own limits add up to, in the same one-minute windows), and 120 a minute for [QR images](https://khqrpay.dev/docs/checkout#qr); its 429 carries only `Retry-After`.
- Per network, API keys that don’t exist: once keys we can’t find (made up, mistyped or revoked) are looked up more than 30 times in a minute from one network (an IPv4 address or an IPv6 /64), calls from that network get `429 rate_limited` until the minute ends, with a `Retry-After` to then. A call with a key that works is never counted. A key that has worked recently isn’t held up by its network’s count, and very many calls at once from one network with keys not yet seen get `429 rate_limited`, to retry after `Retry-After`.
- Per store: 120 live creates per minute across all its keys. A 429 from this limit reports the store’s numbers in the `X-RateLimit-*` headers, so it can arrive while your key still has room; wait `Retry-After` seconds either way. A store can have up to 10 active keys in each mode; revoke one to create another. It can also have up to 10 live webhook endpoints and 3 test ones, and up to 500 payment links, counting paused ones.
- Per account: 60 [list](https://khqrpay.dev/docs/payments#list) requests per minute across all its stores and keys. List responses report whichever of this limit, your key’s and the account’s reads you’re closest to, and a 429 from this limit reports the account’s numbers. Simulate has the same cap of its own: 60 requests per minute across all the account’s keys.
- Reads per account: 600 a minute on Free, 1,200 a minute on Starter, 3,000 a minute on Pro, 6,000 a minute on Scale, 20,000 a minute on Enterprise, across all its stores and keys. That’s retrieve, list and usage together. Responses report whichever of this limit and your key’s you’re closer to. To follow a payment, prefer webhooks to polling.
- Reads per account per day, by size: 20 MB on Free, 40 MB on Starter, 100 MB on Pro, 200 MB on Scale, 500 MB on Enterprise, across all its stores and keys, test and live together. A read counts the size of its response, and 2 KB at least, so 20 MB is about 10,000 ordinary reads (10,240 on Free, 20,480 on Starter, 51,200 on Pro, 102,400 on Scale, 256,000 on Enterprise). The day doesn’t grow with a plan’s transactions, so at volume there isn’t room to poll every payment: follow payments by webhooks. Test-mode reads stop once the account’s reads for the day, test and live together, reach half its day, so they can’t use up the half that live reads need. Retrieve, list, usage and simulate count, and so does a create that repeats an `Idempotency-Key`, since it returns the stored payment. Over it, these return `429 rate_limited` until 07:00 Cambodia time (midnight UTC), while creating new payments and webhooks keep working. This 429 carries `Retry-After`, `X-RateLimit-Remaining: 0` and an `X-RateLimit-Reset` at the day’s end, and no `X-RateLimit-Limit`, which counts requests. Webhooks tell you about a payment without a single read, so they’re better than polling.
- How much of the day is left: every successful response the day is charged for, and the day’s 429, carries `X-ReadBudget-Limit` and `X-ReadBudget-Remaining`, in kilobytes, and `X-ReadBudget-Reset`, the Unix time of the next 00:00 UTC. For a test-mode key the limit is its half of the day. The `X-RateLimit-*` headers keep counting requests in the minute.
- Open QRs per account: live QRs from the API and dashboard that are still open are capped at 20 on Free, 50 on Starter, 150 on Pro, 300 on Scale, 1,000 on Enterprise, and test QRs at 50 on any plan. Payment-link QRs don’t count toward this cap. They hold a place in your quota only against other payment-link QRs, so visitors opening them can’t stop your own API and dashboard creates. Over a cap, creates return `429 rate_limited` until a QR closes. This 429 carries your key’s `X-RateLimit-*` numbers, which can show room: go by `Retry-After`, and expect to wait more than once.
- Creates per account: as many live QRs a minute as the plan can hold open, and 60 test QRs a minute and 1,000 a day (the day starts at 07:00 Cambodia time, midnight UTC), across all its stores and keys. A 429 from this limit reports the account’s numbers. Every live create that reaches the bank counts toward the minute, including one the bank refuses. Every test create with a valid body counts, a repeated Idempotency-Key included, and a test create’s `X-RateLimit-*`headers report whichever of your key’s minute and the account’s has less room left. Creates sent at the same instant can go over an open-QR cap briefly, by up to this many.
- Unpaid live QRs per account per day: 100 on Free, 200 on Starter, 1,000 on Pro, 2,500 on Scale, 10,000 on Enterprise, from the API and the dashboard across all its stores and keys. A live QR counts from its create until it’s paid, a late payment included, so paid QRs never use the day up; a create that makes no QR, such as one the bank refuses, doesn’t count. Every live create that reaches the bank, paid or refused, also counts toward three times that a day (300 on Free, 600 on Starter, 3,000 on Pro, 7,500 on Scale, 30,000 on Enterprise), so a retry loop against a failing bank still stops. The day starts at 07:00 Cambodia time (midnight UTC). Over it, live creates return `429 rate_limited` with `X-RateLimit-Reset` and `Retry-After` at the day’s end; one of the day’s QRs being paid frees a place at once, and so does an upgrade. Create a QR when the customer chooses to pay, not each time a page loads. Payment-link QRs don’t count: each link has caps of its own, per hour and per payer’s network, so visitors on your links can’t use up your day.
- Over the limit you get `429 rate_limited` with a `Retry-After` header in seconds. Wait that long; don’t retry in a loop. A `Retry-After` longer than a minute means a day-long limit: fail the call at once, never sleep inside a request, and follow payments by webhooks. An upgrade lifts the day’s reads within 10 minutes. One exception: a create that repeats an `Idempotency-Key` and gets this 429 with `X-ReadBudget-Remaining: 0` goes through when sent again with a new key.
- Request bodies can be up to 32 KB, and metadata up to 4 KB.
- Only **paid live** payments count toward your plan, over 30-day periods. Open live QRs hold a place until they close, as [Check usage](https://khqrpay.dev/docs/errors-and-limits#usage) explains, and at the limit live creates return `402 quota_exceeded`.

| Plan | Price / 30 days | Paid transactions | Stores |
| --- | --- | --- | --- |
| Free | Free | 100 | 3 |
| Starter | $5 | 1,000 | 10 |
| Pro | $25 | 7,500 | 25 |
| Scale | $49 | 20,000 | 100 |
| Enterprise | Contact us | By agreement | By agreement |

You pay for a plan by KHQR from *Billing*: one payment for 30 days, not a subscription, so nothing renews by itself. One payment can also buy 6 months for the price of 5.5, or 12 for the price of 10 (a month is 30 days). It gives all those months’ paid transactions at once, to use any time before the plan ends, and `limit` in [Check usage](https://khqrpay.dev/docs/errors-and-limits#usage) shows them. Prices are in US dollars. When KHQRPay’s billing link is in riel, the QR asks for riel at KHQRPay’s rate (see [Currency and conversion](https://khqrpay.dev/docs/payments#currency)). The plan checkout says it’s for your KHQRPay plan, with a *Back to billing* link and, if the QR expires, *Get a new QR*. The plan’s name and invoice number are in *Billing*. A plan payment the bank approves late still counts: it activates the plan, or is added as plan time if it no longer matches its quote (see [Check usage](https://khqrpay.dev/docs/errors-and-limits#usage)).

## Check usage

`GET /v1/usage` returns your plan quota, which every live create is checked against. The quota belongs to your account, so every store and every key sees the same figures.

**cURL**

```bash
curl https://khqrpay.dev/v1/usage \
  -H "Authorization: Bearer $KHQRPAY_SECRET_KEY"
```

**Response**

```json
{
  "plan": "starter",
  "limit": 1500,
  "used": 284,
  "remaining": 1216,
  "period_start": "2026-10-10T02:14:09.000Z",
  "period_end": "2026-11-09T02:14:09.000Z"
}
```

| Field | Type | Description |
| --- | --- | --- |
| `plan` | string | The plan in effect now: free, starter, pro, scale or enterprise |
| `limit` | number | Paid live transactions allowed this period, including any carried over or added with plan time |
| `used` | number | Paid live transactions so far this period. Test payments never count |
| `remaining` | number | limit minus used, never below 0. Open live QRs aren’t subtracted here, but a live create counts the ones that hold a place for it (see Limit reached) |
| `period_start` | string | Start of the current period (ISO 8601), never later than now. A period is 30 days, or 180 or 360 days for a 6 or 12 month plan; renewing adds the time paid for to it, and an upgrade paid for by what’s left of your plan can be longer or shorter |
| `period_end` | string | End of the period, when used goes back to 0 |
| `free_link` | object | Only on Free, for a key whose store has an ABA link: that link’s own count this calendar month (Cambodia time), shared by every Free account on the link. It holds limit, used, remaining and period_end, when the count starts again. Live creates stop when either this or the account’s own remaining runs out |

- **Periods** last 30 days from when the bank approved your plan payment, not calendar months, and a 6 or 12 month plan is one period of 180 or 360 days. Renewing adds the time you pay for to the end, and an upgrade paid for by what’s left of your plan runs longer or shorter. On Free, a period starts at sign-up, or at your first dashboard visit, usage check or live create after the last period ended.
- **Renewing:** pay again for the plan you’re on before it ends, and the time you pay for (30 days, or 6 or 12 periods of 30 days) is added to its end date and a month’s transactions for each 30 days to your `limit`, so the days you have left aren’t lost. You can pay up to about two years ahead. A renewal QR you open before the end date keeps your unused transactions even if the bank approves it a little after (up to 2 hours); the time you paid for then starts from approval.
- **Switching:** a different plan you pay for while a paid plan is active starts a fresh period (30 days, or 6 or 12 periods of 30 days) as soon as the bank approves the payment, replacing what’s left of the current period, and a cheaper plan gets no credit for it. An upgrade to a pricier plan is priced with a credit for the unused part of what that period is worth: the share of its time left, or of its transactions left if that’s smaller, so transactions you’ve used are never credited. The credit is fixed when the QR is created. If it’s worth the new plan’s price, or all but 1% of it, there’s nothing to pay: the upgrade applies at once and runs for as long as that value buys, longer than 30 days with extra transactions if it’s worth more, a little shorter if it falls just short.
- **Carry-over:** switch while a paid plan is active, and its unused transactions are added to the new period’s `limit`, up to one month of the new plan’s transactions. An upgrade priced with a credit for what you haven’t used starts fresh. Paying from Free carries nothing over.
- **Paid twice, or an old QR?** If a plan payment no longer matches its quote, its value is added as plan time automatically: the plan you paid for runs for a longer or shorter period, or your current plan gets extra time. The invoice in *Billing* shows how. Only a payment for a plan that’s no longer offered, an amount that isn’t a valid price or a result outside safe limits waits for KHQRPay to check it.
- **Lapse:** if a paid period ends without a renewal, the account drops to Free (100 paid transactions per 30 days) for a fresh period, and any carry-over is lost.
- **Limit reached:** a live create also counts your open live QRs from the API and dashboard (pending or scanned, before `expires_at`). When `remaining` minus those reaches 0, live creates return `402 quota_exceeded` until an open QR expires, you renew or upgrade, or the period ends. A payment link’s QR counts open payment-link QRs as well, so visitors opening your links can’t stop your own API and dashboard creates. Test payments keep working.
- **Free, per ABA link:** Free transactions are also counted per ABA link each calendar month (Cambodia time), apart from your own 30-day period, so Free accounts that share one ABA link share those 100. Your own open live QRs on that link count against them too, the same ones as above. Once they’re used, live creates return `402 quota_exceeded` until the 1st of the next month, even while `remaining` is above 0 and even if yours is the only account on the link.

For AI tools: every docs page is listed in [llms.txt](https://khqrpay.dev/llms.txt).
