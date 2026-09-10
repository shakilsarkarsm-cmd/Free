# bKash + Nagad Payment Website

A minimal Node.js/Express site that lets a customer pay a one-time amount via
**bKash** or **Nagad**, using each provider's official checkout API.

## What's included
```
payment-site/
├── server.js              # Express app entry point
├── routes/payment.js       # /api/pay routes: create payment, callbacks, status
├── services/bkash.js       # bKash Tokenized Checkout (token grant, create, execute)
├── services/nagad.js       # Nagad Checkout API (RSA encrypt/sign, initialize, complete)
├── public/index.html       # Checkout page (amount + Pay with bKash / Nagad)
├── public/success.html     # Shown after a successful payment
├── public/cancel.html      # Shown after a cancelled/failed payment
└── .env.example            # All required config, copy to .env
```

## 1. You need real merchant accounts first
Neither bKash nor Nagad let just anyone accept payments — this code talks to
their APIs, but it can't work until you're an approved merchant:

- **bKash**: register at https://developer.bka.sh — sign up for the
  "Tokenized Checkout" sandbox to get `app_key`, `app_secret`, and sandbox
  username/password immediately. Going live requires a business merchant
  account application reviewed by bKash.
- **Nagad**: apply at https://home.mynagad.com/merchant-integration/ — Nagad
  issues you an RSA key pair and their public key during onboarding
  (sandbox access is also gated behind this application, unlike bKash).

Until you have those credentials, this code will run but every payment
attempt will fail at the API call — that's expected, not a bug.

## 2. Install & configure
```bash
npm install
cp .env.example .env
# then fill in .env with your real (or sandbox) credentials
```

For Nagad, save the two PEM key files it gives you somewhere like
`./keys/nagad_merchant_private_key.pem` and `./keys/nagad_public_key.pem`,
and point `.env` at them.

## 3. Run it
```bash
npm start
# open http://localhost:3000
```

## 4. How the flow works
**bKash:** grant token → create payment (returns a bKash-hosted URL) →
redirect the customer there → bKash redirects back to
`/api/pay/bkash/callback` → you call execute → done.

**Nagad:** initialize (encrypted payload, signed) → complete (returns a
Nagad-hosted URL) → redirect the customer there → Nagad redirects back to
`/api/pay/nagad/callback` with the result.

Orders are tracked in an in-memory `Map` in `routes/payment.js` — good
enough to test the flow, but replace it with a real database (Postgres,
MongoDB, etc.) before handling real money, and treat the callback as
untrusted input (verify the order + amount server-side, don't trust query
params alone).

## 5. Before going live — important
- **Never commit `.env` or your Nagad private key to version control.**
- Switch `BKASH_BASE_URL` / `NAGAD_BASE_URL` from sandbox to production
  once bKash/Nagad approve your live merchant account.
- Add HTTPS (both providers require a live `https://` callback URL).
- Add a real database and idempotency checks (a callback can arrive more
  than once for the same order).
- Add server-side logging/reconciliation — periodically call
  `bkash.queryPayment()` for any order stuck in "pending" in case a
  callback was missed.
