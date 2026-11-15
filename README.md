# KHQRPay Agent Skill

The official Agent Skill for [KHQRPay](https://khqrpay.dev), a KHQR payment gateway and API for Cambodia.

It teaches a coding agent to take KHQR payments with the KHQRPay API:
- create a payment from your server;
- send the customer to the hosted checkout, or render the KHQR yourself;
- verify signed webhooks;
- build in test mode;
- go live.

It covers the REST API and the official Node.js, Python and PHP SDKs.

## Install

With the [skills](https://skills.sh) command, for Claude Code, Cursor, Codex and other agents:

```
npx skills add https://khqrpay.dev
```

As a Claude Code plugin:

```
claude plugin marketplace add khqrpay-dev/skills
claude plugin install khqrpay@khqrpay
```

## Where the files come from

`skills/khqrpay/` is copied from https://khqrpay.dev/.well-known/agent-skills/ once a day by a GitHub Action, after its digest is checked. The skill is built from the KHQRPay docs, so it changes when they do. Please don’t edit these files here: report a problem to support@khqrpay.dev instead.

## Terms

The skill’s code samples are yours to use; its terms are in [LICENSE.txt](skills/khqrpay/LICENSE.txt).

## Official channels

- Website: https://khqrpay.dev
- Docs: https://khqrpay.dev/docs
- Telegram bot: [@khqragent](https://t.me/khqragent)
- Email: support@khqrpay.dev
