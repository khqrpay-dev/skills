# Going live

1. In the ABA Merchant app, create a *Payment Link* in KHR or USD with an open amount, then connect it in *Store settings*. You can create live keys only after that. Once KHQRPay knows which ABA account the link pays into, *Store settings* shows the last 4 digits of its merchant ID.
2. Create a `kp_live_` key and a *Live* webhook endpoint, and give your server the new endpoint’s signing secret.
3. Make a real $0.01 payment from end to end before you open to customers. On Free, this first real payment also lifts a new account’s lower daily limit of unpaid live QRs.

- **Pausing.** Pausing the store in *Store settings* stops live payments only: live creates return `400 payment_link_disabled` (“The store’s payments are paused. Turn them back on in Store settings.”). Test payments keep working.
- **A link that pays another account.** If the link’s QRs start paying a different ABA account from the one it was connected with, live creates stop the same way, with “The connected payment link now pays a different ABA account. Check the link, then connect it again in Store settings.” If ABA gives KHQRPay no QR when you connect it again, KHQRPay can’t check which ABA account the link pays, so the warning stays: wait a few minutes, then connect the link again. Test payments keep working.
- **The ABA rail.** KHQRPay drives your ABA payment link through ABA’s own web checkout, not an official API, and ABA can change it at any time. On 22 Oct 2026 ABA moved that checkout API from pwapp.ababank.com to link.payway.com.kh/api and renamed variables in the link page; KHQRPay was updated the same night. If live creates fail with `payment_provider_error`, retry with the same Idempotency-Key, and contact support if it continues.

For AI tools: every docs page is listed in [llms.txt](https://khqrpay.dev/llms.txt).
