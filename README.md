# Summarist — Frontend Simplified Virtual Internship v2

A Next.js (App Router) + TypeScript clone of [summarist.vercel.app](https://summarist.vercel.app/), using Firebase Authentication, Firestore, Stripe (via the Firebase "Run Payments with Stripe" extension), Redux Toolkit and react-icons.

## Project structure

```
public/                    logo.png, landing.png, google.png, login.png, pricing-top.png
firestore.rules            security rules for user data + the Stripe extension
src/
  app/
    page.tsx               home page (ported from summarist-home-page)
    choose-plan/           sales page + Stripe checkout (no sidebar/search)
    (app)/layout.tsx       sidebar + search bar shell
    (app)/for-you/         Selected / Recommended / Suggested
    (app)/book/[id]/       book details + Read/Listen gating + My Library
    (app)/player/[id]/     summary + custom audio player
    (app)/settings/        plan + email
    (app)/library/         saved + finished books
  components/              AuthModal, Sidebar, SearchBar, BookCard, AudioPlayer, Skeleton, ...
  hooks/                   useAudioDuration, useDebounce, useBookAccess
  lib/                     firebase.ts, api.ts, stripe.ts, userData.ts, format.ts
  redux/                   store + modal / user / ui slices
  types/book.ts            Book type
```

## 1. Run locally

Requires Node 20.9+.

```bash
npm install
cp .env.example .env.local   # then fill in the values (steps below)
npm run dev                  # http://localhost:3000
```

## 2. Firebase project

1. Go to <https://console.firebase.google.com> → **Add project**.
2. **Project settings → General → Your apps → Web (`</>`)** → register an app. Copy the config values into `.env.local` (`NEXT_PUBLIC_FIREBASE_*`).
3. **Build → Firestore Database → Create database** (production mode, any region).
4. **Firestore → Rules**: paste the contents of `firestore.rules` and **Publish**.
5. Upgrade the project to the **Blaze** (pay-as-you-go) plan — the Stripe extension needs it (Cloud Functions).

### Authentication

1. **Build → Authentication → Get started**.
2. **Sign-in method** → enable **Email/Password**.
3. **Sign-in method** → enable **Google** (choose a support email) → Save.
4. **Settings → User actions** → untick **Email enumeration protection** and save. With it on, Firebase returns a generic `invalid-credential` error and the modal can only say "User not found or incorrect password"; with it off the modal shows **"User not found"** exactly as required.
5. **Settings → Authorized domains**: `localhost` is there by default. After deploying, add your Vercel domain (e.g. `your-app.vercel.app`) so Google login works in production.

### Guest user

**Authentication → Users → Add user**: email `guest@gmail.com`, password `guest123`. (If you use different credentials, set `NEXT_PUBLIC_GUEST_EMAIL` / `NEXT_PUBLIC_GUEST_PASSWORD`.)

## 3. Stripe

### Products and prices

In the [Stripe Dashboard](https://dashboard.stripe.com) (test mode):

1. **Product catalog → Add product** → name **Premium Plus** → recurring price **$99.99 / year** → Save. Copy the price ID (`price_...`) into `NEXT_PUBLIC_STRIPE_PREMIUM_PLUS_YEARLY_PRICE_ID`.
2. **Add product** → name **Premium** → recurring price **$9.99 / month** → Save. Copy the price ID into `NEXT_PUBLIC_STRIPE_PREMIUM_MONTHLY_PRICE_ID`.

**7-day free trial:** the app passes `trial_period_days: 7` when it creates the yearly checkout session, so you don't need to configure a trial on the price. The Checkout page will show "7 days free".

### Install the "Run Payments with Stripe" extension

1. Firebase console → **Extensions** → search **Run Payments with Stripe** (by Stripe) → **Install**.
2. Configuration:
   - **Products and pricing plans collection**: `products`
   - **Customer details and subscriptions collection**: `customers` (if you choose another name, set `NEXT_PUBLIC_STRIPE_CUSTOMERS_COLLECTION` to match)
   - **Sync new users to Stripe customers**: *Sync* (optional)
   - **Automatically delete Stripe customer objects**: as you like
   - **Stripe API key with restricted access**: create a **restricted key** in Stripe (**Developers → API keys → Create restricted key**) with *write* access to Customers, Checkout Sessions and Customer portal, and *read* access to Subscriptions and Prices. Paste it here.
   - **Stripe webhook secret**: leave the placeholder for now.
3. After it installs, open the extension's **How this extension works** tab and copy the **webhook URL**.
4. In Stripe: **Developers → Webhooks → Add endpoint** → paste the URL and select these events:
   `product.created`, `product.updated`, `product.deleted`, `price.created`, `price.updated`, `price.deleted`, `checkout.session.completed`, `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`, `invoice.payment_succeeded`, `invoice.payment_failed`, `invoice.upcoming`, `invoice.marked_uncollectible`, `invoice.payment_action_required`, `payment_intent.processing`, `payment_intent.succeeded`, `payment_intent.canceled`, `payment_intent.payment_failed`, `tax_rate.created`, `tax_rate.updated`.
5. Copy the endpoint's **signing secret** (`whsec_...`) → Firebase **Extensions → Run Payments with Stripe → Reconfigure** → paste it as the webhook secret → Save.
6. Edit each product in Stripe once (or create them after the webhook exists) so they sync into the Firestore `products` collection.

Test with card `4242 4242 4242 4242`, any future date, any CVC.

### How the app uses it

- **Checkout** (`src/lib/stripe.ts`): writes `customers/{uid}/checkout_sessions/{id}` with `price`, `success_url` (`/settings`), `cancel_url` (`/choose-plan`) and, for the yearly plan, `trial_period_days: 7`. It listens to that document and redirects to the `url` the extension adds.
- **Subscription status** (`src/components/providers/AuthListener.tsx`): listens to `customers/{uid}/subscriptions` where `status` is `trialing` or `active`. The plan is **Premium Plus** when the price matches the yearly price ID (or the interval is `year`), **Premium** for the monthly price, otherwise **Basic**. It's stored in Redux and used by the book page gating, the player and Settings.

## 4. Environment variables

| Variable | Where it comes from |
| --- | --- |
| `NEXT_PUBLIC_FIREBASE_API_KEY` … `NEXT_PUBLIC_FIREBASE_APP_ID` | Firebase web app config |
| `NEXT_PUBLIC_STRIPE_PREMIUM_PLUS_YEARLY_PRICE_ID` | Stripe yearly price ID |
| `NEXT_PUBLIC_STRIPE_PREMIUM_MONTHLY_PRICE_ID` | Stripe monthly price ID |
| `NEXT_PUBLIC_STRIPE_CUSTOMERS_COLLECTION` | Extension's customers collection (default `customers`) |
| `NEXT_PUBLIC_GUEST_EMAIL` / `NEXT_PUBLIC_GUEST_PASSWORD` | Guest account (default `guest@gmail.com` / `guest123`) |

Stripe secret/restricted keys never go in this app — only in the extension's configuration.

## 5. Deploy to Vercel

1. Push the project to GitHub (`.env.local` is git-ignored).
2. <https://vercel.com/new> → import the repo. Framework preset: **Next.js** (defaults are fine).
3. **Environment Variables**: add every variable from `.env.example` with your values.
4. **Deploy**.
5. Add the Vercel domain to **Firebase → Authentication → Settings → Authorized domains**.

## Data model (Firestore)

```
users/{uid}/library/{bookId}    book saved with "Add title to My Library" (copy of the book + addedAt)
users/{uid}/finished/{bookId}   book whose audio played to the end (copy of the book + addedAt)
customers/{uid}/...             managed by the Stripe extension
```

## Notes

- Book data, covers and audio come from the `summaristt` Cloud Functions API and Firebase Storage. `next.config.ts` allows `firebasestorage.googleapis.com` for `next/image`.
- Audio durations are read from each `audioLink`'s metadata in the browser (`useAudioDuration`) and cached for the session.
- Login/register from the home page redirects to `/for-you`. Logging in from another page (e.g. a book page) keeps you on that page, as on the reference site.
- The player page shows the "Aa" font-size controls in the sidebar, like the reference site.
#   s u m m a r i s t  
 