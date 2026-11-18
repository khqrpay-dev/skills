# Payment links

A KHQRPay payment link is a KHQR payment page you make in the dashboard, with no code, and share anywhere: a chat, a Facebook post, a Telegram channel. A customer opens it, presses *Continue to pay* and pays the QR from any Cambodian banking app, and the money goes to your store’s ABA payment link. Each link has its own address, `khqrpay.dev/l/` and the link’s id.

Two things are called a payment link. Your *ABA payment link* is the one you make in the ABA Merchant app and connect in *Store settings*: it’s where the money goes. KHQRPay’s payment links, under *Payment links* in the dashboard, are pages you share with customers, and every payment on one goes to the ABA payment link the store has connected.

This page is the reference. To make and share your first link step by step, see [Sell with a KHQR payment link, no code](https://khqrpay.dev/guides/khqr-payment-link-no-code).

## When to use a payment link

Use a link when you sell without a website or a developer: one link per product with a fixed price, or one without a price for a deposit, a tip jar or a custom order. Use [the API](https://khqrpay.dev/docs/payments) when your own site or app knows the order: it makes one QR per order, with your order id in `reference_id` and your own `metadata`. A link can’t carry your order id: every payment on it has the link’s reference.

Links make live payments only, so there’s no test-mode link, and making one needs a connected ABA payment link. To try the payment flow without real money, use the API in [test mode](https://khqrpay.dev/docs/test-mode). There’s no API for links themselves either: you make them, pause them and see them in the dashboard.

## Making a link

Under *Payment links*, choose *Create link*. A link has:

- **Title.** One line, shown as the page’s heading. It can’t include a word that reads as the name of a bank, a payment network or a government body, so customers can’t mistake the link for one.
- **Description.** Optional. Shown under the title.
- **Customer price.** A currency, USD or KHR, and an amount. Leave the amount empty and the customer enters it. See [Amounts and currency](https://khqrpay.dev/docs/payment-links#amounts).

The store’s name, logo, brand colour and default checkout language come from *Store settings* → *Checkout branding*, and a change there shows on every one of the store’s links at once.

A link can’t be changed once it’s made. To change its title, description or price, create a new link and deactivate the old one (see [Deactivating a link](https://khqrpay.dev/docs/payment-links#deactivate)). The new link has a new address, so share it again.

Each link in the list has *Copy link*, which copies its address, and *Details*, which shows its title, price, description, address and when it was made. For a price in the other currency than your ABA payment link’s, *Details* also shows what the QR asks for and the rate: the one locked when you saved the link, or today’s rate for a link a currency switch left in the other currency.

## What the payer sees

The link’s page shows the store’s logo and name, the link’s title and description, the price or a field for the amount, and *Continue to pay* in the store’s brand colour, with *Report this page* at the foot. It opens in the store’s default checkout language, Khmer or English, unless the payer chose one before, and they can switch with one tap.

*Continue to pay* first runs a person check from Cloudflare, which stays invisible unless Cloudflare needs the payer to do something. Then it opens the [hosted checkout](https://khqrpay.dev/docs/checkout) with a new KHQR: the same page an API payment opens. A payer who presses it again for the same amount while that QR is still open is sent back to it instead of getting another.

Posted in a chat, a link shows a preview: the link’s title and the store’s name; the link’s description or, when it has none, a line asking to pay the store with any Cambodian banking app, in the store’s language; and a card with the price, or “Scan to pay” for a link without one. Search engines are asked not to index link pages.

If KHQRPay blocks an account, for example after a report, its links answer as unknown pages, previews included.

## Amounts and currency

- **Fixed price.** The payer sees the amount and can’t change it.
- **Amount left empty.** The payer enters the amount before they get the QR: dollars with up to two decimal places, riel in whole riel.
- **Smallest and largest.** The same minimums as an API payment, set out in [Currency and conversion](https://khqrpay.dev/docs/payments#currency). The most a payer can pay is your ABA payment link’s own maximum, where ABA sets one, in the link’s currency. A payer outside these is told so on the page.

Every QR asks for the currency of your ABA payment link. A price in the other currency is converted at KHQRPay’s rate, locked when you save the link, for a fixed price and an amount the payer enters alike. The page shows the price with what the QR charges under it, and for an amount the payer enters, the currency it’s paid in and the rate. How amounts convert and round is in [Currency and conversion](https://khqrpay.dev/docs/payments#currency).

If you connect an ABA payment link in the other currency, links you made before keep their price and their address, and their QRs ask for the new link’s currency:

- **A price already in that currency** is charged as it is.
- **A price in the other currency** is converted at KHQRPay’s rate when the payer presses *Continue to pay*, not at a locked one, and has to be within the new link’s maximum.
- **What shows where:** the link’s page shows what the QR will charge now, at today’s rate where a price is converted, and *Details* in the dashboard shows the same charge and today’s rate. The list of links still shows the price you saved.
- **To lock a rate instead,** create a new link and deactivate the old one.

## Knowing a link was paid

- **Dashboard.** Every payment on a link is a live payment of the store, under *Payments*, with the reference `link:` and the link’s id: the part of its address after `/l/`.
- **Telegram.** A group linked to the store under *Store settings* → *Telegram alerts* gets a receipt for each confirmed payment, with its amount, the store, the reference, the payment ID and the time, and a *View payment* button. See [KHQR payments in Telegram](https://khqrpay.dev/guides/khqr-telegram-bot).
- **Webhooks.** The store’s live endpoints get the same events for a link payment as for an API one: `payment.scanned`, `payment.completed`, `payment.expired` and `payment.failed`, each to the endpoints that take it, signed the same way. See [Webhooks](https://khqrpay.dev/docs/webhooks).

In the [payment object](https://khqrpay.dev/docs/payments#object), as webhooks and [retrieve](https://khqrpay.dev/docs/payments#retrieve) return it, a link payment has `reference_id` set to `link:` and the link’s id. Its `metadata` is null, or, when the link’s price is in the other currency from the QR, `pricing` with `display_amount`, `display_currency`, `settlement_amount`, `settlement_currency` and `conversion_rate`. If your server takes webhooks for its own orders, answer `2xx` to a link payment it doesn’t know, so the delivery isn’t retried.

## Limits

- **Links per store.** Up to 500, counting deactivated ones, since a link can’t be deleted. At 500, *Create link* says so; add another store for more.
- **Your plan.** A paid link payment counts toward your plan’s paid transactions like any live payment. Before a link makes a QR, the open live QRs on your links count against what’s left, as well as your own; your API and dashboard creates count only their own.
- **On Free,** a paid link payment also counts toward this month’s Free transactions of the ABA account your ABA payment link pays into, which all of that account’s payment links share. When none are left, a payer who presses *Continue to pay* is told the shop can’t take payments right now. See [Check usage](https://khqrpay.dev/docs/errors-and-limits#usage).
- **What links don’t use up.** QRs on links don’t count toward the live QRs your account can hold open, leave unpaid in a day or ask the bank for in a day, from the API and the dashboard. So visitors on your links can’t stop your own creates. Link QRs have a day of their own instead: the same number of unpaid live QRs across all your links, and on Free three times that in requests to the bank. See [Rate limits](https://khqrpay.dev/docs/errors-and-limits#limits).

**Each link’s own caps.** A payer over one is told why on the page, and when to try again:

| Cap | What it counts | What the payer is told |
| --- | --- | --- |
| Open QRs from one browser | QRs open on this link at once from the same browser | They already have open QRs for this link: pay one, or wait a few minutes for them to expire |
| Open QRs from one network | QRs open on this link at once from the same internet connection | Many people on their connection are paying with this link: wait a little, or switch between Wi-Fi and mobile data |
| Open QRs in all | QRs open on this link at once | Too many tries: wait a minute |
| Unpaid QRs in an hour, from one network | QRs left unpaid on this link this hour from the same connection | Too many went unpaid from their connection: try again after the time shown, or switch between Wi-Fi and mobile data |
| Unpaid QRs in an hour, in all | QRs left unpaid on this link this hour | The link can’t take new payments until the time shown (see A full link) |
| Presses of *Continue to pay* in a minute | Presses from one network, on this link and on all links, and from everyone across the store’s links | Too many tries: wait a minute |

- **Busy hours on Free.** On Free, links can also wait for the next hour when your links have sent very many QR requests to the bank in the hour, refused ones included, or very many Free accounts are making them at once. All your links count together, so a new link waits too. Your links’ hour is kept apart from your own API and dashboard creates, so visitors on your links can’t use those up. Paid plans never wait.
- **A full link.** Once a link has had as many QRs left unpaid in an hour as it allows, it turns new payers away until the next hour, and its page says until when. A QR that’s paid frees its place. Real buyers usually pay, so someone may be flooding the link: deactivate it, and share a new one if you need one.
- **Who hears about it.** On a paid plan, a Telegram group linked to the store hears it once a day for that link, headed “Payment link full until” and the hour it takes payments again, in Cambodia time, or “Payment links full for today” when all your links are full too.
- **All your links full.** Once your links together have left the day’s unpaid QRs (see [Rate limits](https://khqrpay.dev/docs/errors-and-limits#limits)), every link turns new payers away, and a new link would too, until one of those QRs is paid or the day starts again in the morning, Cambodia time. On Free they also stop until the morning once they have sent the bank three times that many QR requests in the day, paid or refused, as a link whose ABA payment link refuses every QR soon would. The payer is told the shop’s payment links can’t take new payments right now.
- **On Free** the store’s Telegram group isn’t told when your links wait for the next hour or are full for the day: the payment page tells the payer, and the dashboard shows a banner while your links turn payers away.

## Deactivating a link

Open the link’s *Details* and choose *Deactivate*. Its address keeps working, but the page says “This payment link is no longer active.” and has no *Continue to pay* button, and the list marks it *Inactive*. A QR a payer already has can still be paid. *Activate*, in the same place, turns it back on.

Links can’t be deleted. *Pause live payments* in *Store settings*, disconnecting the ABA payment link, or a link whose QRs now pay a different ABA account from the one it was connected with, stops new payments on all of the store’s links at once: a payer who presses *Continue to pay* is then told the shop can’t take payments right now. See [Going live](https://khqrpay.dev/docs/going-live).

For AI tools: every docs page is listed in [llms.txt](https://khqrpay.dev/llms.txt).
