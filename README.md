# SynapCart

**An AI shopping assistant for iPhone and Android.** Tell SynapCart what you're looking for in plain language. It searches Google Shopping across stores, compares the offers and gives you a short, clear recommendation with product cards you can save, track and compare.

This is the umbrella repository. It holds the three projects as git submodules, plus the shared design assets and the docs:

| Path | Repo | What it is |
|---|---|---|
| [`backend/`](https://github.com/emreaydindev/synapcart_backend) | `synapcart_backend` | Cloudflare Worker (TypeScript, Hono) on D1: shopping agent (Gemini, SerpAPI Google Shopping), plans and quota, App Store and Google Play billing, price alerts |
| [`mobile/`](https://github.com/emreaydindev/synapcart_mobile) | `synapcart_mobile` | Kotlin Multiplatform: `shared/` (Ktor client, repositories, rules), the SwiftUI **iOS** app (StoreKit 2) and the Compose **Android** app (Play Billing) |
| [`web/`](https://github.com/emreaydindev/synapcart_web) | `synapcart_web` | Next.js site on Cloudflare Workers (OpenNext): landing, privacy/terms/support, account deletion, password reset, Universal/App Links files |
| `design/` | | Logo and screenshots |
| `docs/` | | [Publishing roadmap](docs/ROADMAP.md) |

```bash
git clone --recurse-submodules https://github.com/emreaydindev/synapcart
# or, in an existing clone:
git submodule update --init --recursive
```

**Domains**

| | Production | Staging |
|---|---|---|
| Website | [synapcart.emreaydin.dev](https://synapcart.emreaydin.dev) | [staging.synapcart.emreaydin.dev](https://staging.synapcart.emreaydin.dev) |
| API | [api.synapcart.emreaydin.dev](https://api.synapcart.emreaydin.dev/api/v1/health) | [staging-api.synapcart.emreaydin.dev](https://staging-api.synapcart.emreaydin.dev/api/v1/health) |

All four are Workers Custom Domains declared in each repo's `wrangler.jsonc`. iOS ships first; Android follows.

The apps are in Turkish by default and also ship in English. The assistant replies in the user's profile language.

## Plans

SynapCart is free to use, with an optional **Premium** subscription. On iPhone it is sold through the **App Store**; on Android, through **Google Play**. Both offer monthly and yearly plans. The limits come from [`backend/src/services/plan.ts`](https://github.com/emreaydindev/synapcart_backend/blob/claude/awesome-maxwell-rc20me/src/services/plan.ts), and the web landing page and both paywalls show the same numbers.

| | Free | Premium |
|---|---|---|
| Assistant messages per day | 5 | 100 (fair-use cap) |
| Products per search | 3 | 8 |
| Analysis model | `gemini-2.5-flash-lite` | `gemini-2.5-flash` (`PREMIUM_ANALYST_MODEL`) |
| Chat history | Last 10 chats (older ones are locked, not deleted) | Unlimited |
| Price-drop alerts | – | ✓ |
| AI product comparison (2–4 products) | – | ✓ |

Daily quotas reset at 00:00 UTC. Premium falls back to Free automatically once the subscription expires. The store that sold the subscription handles payments, renewals, cancellations and refunds.

## Architecture

```
 ┌───────────────────────────┐   HTTPS /api/v1   ┌──────────────────────────────────┐
 │  iOS app (SwiftUI)        │ ────────────────▶ │  Cloudflare Worker (Hono)        │
 │  Android app (Compose)    │ ◀── SSE stream ── │  api. / staging-api.             │
 │   └ shared KMP module     │                   │  agent: Gemini · SerpAPI         │
 │  StoreKit 2 ─ signed JWS ─┼─────────────────▶ │  D1 (SQLite) · Cache API         │
 │  Play Billing ─ token ────┼─────────────────▶ │  Email Service · Rate Limiting   │
 │  FCM/APNs ◀───────────────┼── price alerts ── │  Cron Trigger: daily price check │
 └───────────────────────────┘                   └──────────────▲───────────────────┘
          ▲ Universal Links / App Links                         │ /api rewrite
 ┌────────┴──────────────────┐                                  │
 │  Next.js web (OpenNext)   │ ─────────────────────────────────┘
 │  synapcart.emreaydin.dev  │  reset-password, delete-request/confirm
 └───────────────────────────┘
   App Store Server Notifications V2 ──▶ /billing/apple/notifications
   Google Play RTDN (Pub/Sub push)   ──▶ /billing/google/rtdn
```

- **Backend** (`backend/`): a Worker serving everything under `/api/v1`: `/auth`, `/chat`, `/favorites`, `/user`, `/billing` and `/health`. The HTTP contract is unchanged from the earlier FastAPI version.
  - **Auth:** HS256 JWTs valid for 7 days. A password change or reset bumps `token_version`, which revokes every token already issued. Passwords use PBKDF2-SHA256.
  - **Schema:** D1 migrations (`migrations/`) are the source of truth. Deleting a user cascades to all of their data.
  - **Agent:** a state machine (router → searcher → analyst, or a refusal). `POST /chat/agent/{id}/stream` streams Server-Sent Events: `routing` → `searching` → `thinking` (with products) → `completed` (with the analysis and products). It can end with `out_of_scope` or `error` instead. Only the final answer and its products are saved.
- **Plans and quota:** daily usage is counted in `usage_daily`. When a limit is hit, the API returns HTTP 402 `{detail, code}` with `code` set to `quota_exceeded`, `premium_required` or `history_locked`, and both apps open their paywall.
- **Billing:**
  - **iOS:** the app buys with StoreKit 2, using `appAccountToken` set to the user id. It sends the signed transaction to `POST /billing/apple/verify`, which checks the x5c chain up to Apple Root CA G3 plus the signature, bundle and product. The app finishes the transaction only after the server accepts it. App Store Server Notifications V2 keep renewals, expiries and refunds in sync.
  - **Android:** Play Billing sends the purchase token to `POST /billing/google/verify` (Play Developer API), and RTDN keeps it in sync.
  - `plan_source` records which store sold the plan. An inactive subscription from one store never cancels an active one from the other.
- **Price alerts:** a Premium user sets a target price on a favorite. Both apps register FCM tokens; iOS goes through Firebase with an APNs key. A daily Cron Trigger re-searches those products and pushes through FCM HTTP v1 when a price reaches the target. Tapping the notification opens the product.
- **Compare:** `POST /chat/compare` asks Gemini for pros and cons per product, the recommended item and a verdict, in the user's language.
- **Web** (`web/`): the Next.js site runs on Workers via OpenNext and rewrites `/api/*` to the API.
  - It hosts the pages both stores need: privacy, terms, support and the web account-deletion page.
  - The backend's emails link to `/reset-password?token=…` and `/delete-account?token=…`.
  - `/.well-known/apple-app-site-association` makes reset links open the iOS app, and `assetlinks.json` does the same for Android.

## Local development

Run the backend first, because the apps and the website talk to it on port **8001**.

### Backend

```bash
cd backend
npm install
cp .dev.vars.example .dev.vars      # SECRET_KEY, GEMINI_API_KEY, SERP_API_KEY
npm run db:migrate:local
npm run dev -- --port 8001          # http://localhost:8001/api/v1/health
npm test                            # Vitest in workerd with a local D1; external APIs mocked
```

### Mobile

```bash
cd mobile
./gradlew :shared:testAndroidHostTest :app:testDebugUnitTest
./gradlew :app:installDebug                       # Android emulator -> http://10.0.2.2:8001/
cd iosApp && xcodegen generate && open SynapCart.xcodeproj   # macOS: Debug scheme -> http://localhost:8001/
```

See `mobile/README.md` and [`mobile/docs/IOS_RELEASE.md`](https://github.com/emreaydindev/synapcart_mobile/blob/claude/awesome-maxwell-rc20me/docs/IOS_RELEASE.md), which is the App Store submission checklist.

### Web

```bash
cd web
npm install
cp .env.example .env.local          # NEXT_PUBLIC_API_URL=http://localhost:8001
npm run dev                         # http://localhost:3000
npm run lint && npm run build
npm run preview                     # the production build in workerd
```

### Deploy

| Repo | Staging | Production |
|---|---|---|
| backend | `npm run db:migrate:staging && npm run deploy:staging` | `npm run db:migrate:prod && npm run deploy:prod` |
| web | `npm run deploy:staging` | `npm run deploy` |
| iOS | `bundle exec fastlane beta_staging` | `bundle exec fastlane beta`, then `release` |

GitHub Actions deploys `main` to staging and `v*` tags to production in the backend and web repos. This needs the repo secrets `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID`.

## Legal and support pages

- [Privacy Policy](https://synapcart.emreaydin.dev/privacy) (KVKK and GDPR)
- [Terms of Use](https://synapcart.emreaydin.dev/terms) (including App Store and Google Play subscription terms)
- [Support](https://synapcart.emreaydin.dev/support)
- [Delete your account](https://synapcart.emreaydin.dev/delete-account)

## Screenshots

### Web

| Landing | Privacy | Reset password | Delete account |
|---|---|---|---|
| ![Landing, mobile](design/screenshots/web-landing-mobile.png) | ![Privacy, mobile](design/screenshots/web-privacy-mobile.png) | ![Reset password, mobile](design/screenshots/web-reset-password-mobile.png) | ![Delete account, mobile](design/screenshots/web-delete-account-mobile.png) |

![Landing page, desktop](design/screenshots/web-landing-desktop.png)

Desktop captures of every page are in [`design/screenshots/`](design/screenshots/).

### Apps

> **TODO (owner):** app screenshots aren't captured yet, because the cloud build environment has no simulator or emulator. Capture the iOS ones at 6.9" and 6.5" (they're also needed for App Store Connect) and add them to `design/screenshots/` as `ios-<screen>.png` / `android-<screen>.png`.
>
> Screens to capture: onboarding, chat with a streamed answer and product cards, product details (with the price-alert sheet), favorites, compare, paywall, and profile.
