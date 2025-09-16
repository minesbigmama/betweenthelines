# BetweenTheLines — Full Build with Stripe (env-ready)

1) Unzip, open a terminal in the folder, then run:
```bash
npm install
npm run dev
```
2) Create Stripe Products & Prices. Copy the **Price IDs**.

3) On **Vercel → Project → Settings → Environment Variables**, add:
```
STRIPE_SECRET_KEY=sk_test_...
PRICE_PRO_MONTHLY=price_...
PRICE_PRO_YEARLY=price_...
PRICE_LIFETIME=price_...
PRICE_ADDON_COACHING=price_...
APP_ORIGIN=https://YOUR-VERCEL-URL
```
4) Deploy on Vercel (connect repo and Deploy). Upgrade buttons will open Stripe Checkout and return to `/?plan=pro` or `/?plan=lifetime`.

> To hardcode price IDs instead, edit `api/create-checkout-session.js` and fill the commented `priceMap`.