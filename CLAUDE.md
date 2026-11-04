# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This is the **umbrella repo** for SynapCart, an AI-assisted shopping assistant (BTK competition project). It holds no application code, only git submodules, `design/` assets (logo, `design/screenshots/`), `docs/` and the README. Each submodule is its own GitHub repo with its own CLAUDE.md:

| Path | Repo | Stack |
|---|---|---|
| `backend/` | `synapcart_backend` | FastAPI, SQLAlchemy + Alembic, LangGraph, Gemini, SerpAPI (Google Shopping) |
| `mobile/` | `synapcart_mobile` | Kotlin, Jetpack Compose, Hilt, Retrofit/OkHttp, Play Billing, Firebase Messaging |
| `web/` | `synapcart_web` | Next.js (App Router): landing, legal, support, account deletion, password reset |

```bash
git submodule update --init --recursive   # fetch all modules
```

Make code changes in the submodule's own repo. Commits here only bump submodule pointers (for example "chore: sync submodules") or change the README, `docs/` and `design/`. To bump pointers without checking the submodules out: `git update-index --cacheinfo 160000,<sha>,backend` (likewise `mobile`, `web`).

## Roadmap

`docs/ROADMAP.md` is the publishing roadmap: the phase status table, the plan for each phase, the owner's decisions and the setup notes for cloud sessions (Android SDK, Maven mirror). Read it before starting any roadmap work, and update its Status table when a phase lands.

## How the pieces connect

- The **backend** serves everything under `/api/v1` (`/auth`, `/chat`, `/favorites`, `/user`, `/billing`, `/health`) and runs locally on port **8001**. In production it runs on Render (`https://synapcart-backend.onrender.com`). The free tier causes slow cold starts.
- The **mobile** app reads its backend from `BuildConfig.BASE_URL`, set per build type in `app/build.gradle.kts` (debug defaults to the emulator loopback on port 8001; override with `-PdebugBaseUrl=`). Agent chat uses the SSE endpoint `POST /api/v1/chat/agent/{id}/stream`, whose `{status, analysis, products}` events the app parses directly. That stream and the non-streaming `POST /chat/agent/{id}` run the same LangGraph graph (`agent_service.app_agent`; the stream goes through `stream_agent()`), and only the final answer is saved.
- **Auth:** JWTs last `ACCESS_TOKEN_EXPIRE_MINUTES` (7 days by default) and embed `token_version`. A password reset or change bumps it and revokes all tokens. The app clears the session on any authenticated 401 and returns to Login.
- **Plans and quota:** the backend's `app/services/plan_service.py` is the source of truth. Free has 5 agent messages per day (UTC), 3 products per search, `gemini-2.5-flash-lite` and the last 10 chats (older ones are `locked`). Premium has a 100/day soft cap, 8 products, `PREMIUM_ANALYST_MODEL`, unlimited history, price alerts and compare. Limit errors are HTTP 402 `{detail, code}` (`quota_exceeded`, `premium_required`, `history_locked`), and the app opens its paywall on them. The same numbers appear in `web/lib/site.ts` (`plans`) and the app's paywall strings, so change all three together.
- **Billing (`/billing`):** the app buys the Play subscription (`PREMIUM_PRODUCT_ID`/`PREMIUM_PLAN_*` in `mobile/app/build.gradle.kts`, matching `PLAY_SUBSCRIPTION_IDS` on the backend) with the user id as `obfuscatedAccountId`, and sends the token to `POST /billing/google/verify` (Play `purchases.subscriptionsv2`). It acknowledges the purchase only after the backend has verified it. Play RTDN pushes renewals and cancellations to `POST /billing/google/rtdn?token=`.
- **Price alerts:** `PATCH /favorites/{id}` sets `target_price` (Premium), and the app registers its FCM token with `POST /user/me/device-token`. The backend's daily Render cron job `app/jobs/price_check.py` sends FCM HTTP v1 pushes (channel `price_alerts`, data `{type, favorite_id, link, price}`), and the app opens that product when the notification is tapped. Without `google-services.json` (mobile) or `FIREBASE_SERVICE_ACCOUNT_JSON` (backend), push is a no-op.
- **Compare:** `POST /chat/compare {products: [2–4]}` (Premium) returns `{items: [{title, link, pros, cons}], verdict, best_index}`, which `CompareScreen` renders.
- **Password reset** crosses all three: the backend emails `{APP_FRONTEND_URL}/reset-password?token=…`, the **web** portal (Vercel, `https://synapcart-web.vercel.app`) renders the form and POSTs `{token, new_password}` to `/api/v1/auth/reset-password` through its Next.js rewrite, and the mobile app also registers deep links for `https://synapcart.app/reset-password` and `synapcart://reset-password`.
- **Account deletion** (Play's web deletion URL): the web `/delete-account` page POSTs `/auth/delete-request`. The backend emails `{APP_FRONTEND_URL}/delete-account?token=…`, and the page then POSTs `/auth/delete-confirm`. In the app, Profile deletes the account with `DELETE /user/me`.
- When you change an API response shape, update the backend schema, the mobile DTOs in `data/model` and, for auth, reset or deletion, the web pages together.

User-facing text is Turkish by default. The mobile app also ships English (`values-en`) with a per-app language picker; agent replies follow the user's `language`.
