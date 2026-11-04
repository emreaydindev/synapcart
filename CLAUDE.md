# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This is the **umbrella repo** for SynapCart, an AI-assisted shopping assistant (BTK competition project). It holds no application code, only git submodules, `design/` assets (logo, `design/screenshots/`), `docs/` and the README. Each submodule is its own GitHub repo with its own CLAUDE.md:

| Path | Repo | Stack |
|---|---|---|
| `backend/` | `synapcart_backend` | Cloudflare Worker (TypeScript, Hono), D1, Gemini REST, SerpAPI, Email Service, Cron Triggers |
| `mobile/` | `synapcart_mobile` | Kotlin Multiplatform `shared/` (Ktor, kotlinx.serialization, DataStore), SwiftUI iOS app (StoreKit 2, Firebase Messaging), Compose Android app (Hilt, Play Billing, FCM) |
| `web/` | `synapcart_web` | Next.js (App Router) on Cloudflare Workers via OpenNext: landing, legal, support, account deletion, password reset |

```bash
git submodule update --init --recursive   # fetch all modules
```

Make code changes in the submodule's own repo. Commits here only bump submodule pointers (for example "chore: sync submodules") or change the README, `docs/` and `design/`. To bump pointers without checking the submodules out, run `git update-index --cacheinfo 160000,<sha>,backend` (likewise for `mobile` and `web`).

## Roadmap

`docs/ROADMAP.md` is the publishing roadmap. It holds the phase status table, the plan for each phase (Phase 8: Cloudflare, Phase 9: iOS and the App Store), the owner's checklist and the setup notes for cloud sessions. Read it before starting roadmap work, and update its Status table when a phase lands.

## How the pieces connect

- **Domains:**

  | | Production | Staging |
  |---|---|---|
  | Website | `https://synapcart.emreaydin.dev` | `https://staging.synapcart.emreaydin.dev` |
  | API | `https://api.synapcart.emreaydin.dev` | `https://staging-api.synapcart.emreaydin.dev` |

  These are Workers Custom Domains in each repo's `wrangler.jsonc`. Locally, the backend runs on port **8001**: `npm run dev -- --port 8001`.
- **API:** everything is under `/api/v1` (`/auth`, `/chat`, `/favorites`, `/user`, `/billing`, `/health`).
  - Agent chat uses the SSE endpoint `POST /api/v1/chat/agent/{id}/stream`, whose `{status, analysis, products}` events the shared KMP client parses (`SynapCartApi.streamAgent`). The stream and the non-streaming route run the same agent (`backend/src/services/agent.ts`), and only the final answer is saved.
  - Keep the event field names stable.
- **Auth:**
  - JWTs (HS256, 7 days, `sub` = email) embed `token_version`. A reset or password change bumps it, which revokes every token.
  - The shared client clears the session on any authenticated 401 and fires `SessionEvents.expired`, and both apps return to Login.
- **Plans and quota:**
  - `backend/src/services/plan.ts` is the source of truth. Free has 5 messages per day (UTC), 3 products, flash-lite and the 10 newest chats. Premium has 100 per day, 8 products, `PREMIUM_ANALYST_MODEL`, unlimited history, price alerts and compare.
  - Limit errors are HTTP 402 `{detail, code}` with `quota_exceeded`, `premium_required` or `history_locked`, and both apps open their paywall on them.
  - The numbers also appear in `web/lib/site.ts`, the Android strings and the iOS paywall table; change them together.
- **Billing:** `plan_source` (`apple` or `google`) records the store, and one store's inactive subscription never downgrades the other's active one.
  - **iOS:** StoreKit 2 products `synapcart_premium_monthly`/`_yearly`, with `appAccountToken` set to the user id. The app sends the signed transaction to `POST /billing/apple/verify`, and the backend verifies the x5c chain against the pinned Apple Root CA G3. The app finishes the transaction only after the server accepts it. App Store Server Notifications V2 go to `/billing/apple/notifications`.
  - **Android:** Play `synapcart_premium` with `obfuscatedAccountId` set to the user id, verified at `POST /billing/google/verify`. RTDN goes to `/billing/google/rtdn?token=`.
- **Price alerts:**
  - `PATCH /favorites/{id}` sets `target_price` (Premium).
  - Both apps register FCM tokens with `POST /user/me/device-token {token, platform}`.
  - The backend's daily Cron Trigger (`src/jobs/priceCheck.ts`) sends FCM HTTP v1 pushes with an APNs payload. The data is `{type, favorite_id, link, price}`, and tapping the notification opens that product.
  - Without the Firebase config files (`google-services.json` / `GoogleService-Info.plist`) or `FIREBASE_SERVICE_ACCOUNT_JSON`, push is a no-op.
- **Password reset** crosses all three:
  1. The backend emails `{APP_FRONTEND_URL}/reset-password?token=…` through Cloudflare Email Service.
  2. The web page POSTs to `/api/v1/auth/reset-password` through its `/api` rewrite.
  3. On iOS, the AASA file makes that link open the app's reset screen. Android App Links and the `synapcart://reset-password` scheme do the same on Android.
- **Account deletion:** the web `/delete-account` page handles `/auth/delete-request` and `/auth/delete-confirm` (the stores' web deletion URL). Both apps delete in-app with `DELETE /user/me`, which App Store Guideline 5.1.1(v) requires.
- **API changes:** when you change an API response shape, update the backend route, the shared DTOs in `mobile/shared/.../data/model/Models.kt` and, for auth, reset or deletion, the web pages together.

User-facing text is Turkish by default, with English for both apps (the iOS strings are generated from the Android ones) and agent replies in the user's `language`.
