# SynapCart: road to publishing (Play Store + web)

> The working roadmap for making SynapCart publishable. All work happens on branch
> `claude/init-dno428` in each repo (`synapcart`, `synapcart_backend`, `synapcart_mobile`,
> `synapcart_web`). Update the **Status** table whenever a phase lands.

## Status

| Phase | Repo | State | Commit |
|---|---|---|---|
| 1. Backend hardening | synapcart_backend | **Done**: 23 pytest tests pass | `f32fccd` |
| 3. Mobile release readiness and bugs | synapcart_mobile | **Done**: builds, 9 JVM tests pass, lint has 0 errors, contract-tested against a local backend | `bb11380` |
| 4. Mobile design system and i18n | synapcart_mobile | **Done**: new tokens, components and screens; TR default plus EN strings with a per-app language picker; splash, monochrome icon; drawer rename/delete. 14 JVM tests pass, lint has 0 errors, release builds. Compose BOM held at 2026.06.01 and lifecycle at 2.10.0 because newer versions need compileSdk 37. Paywall, Compare and price alerts are UI stubs until Phases 2 and 5. `POST_NOTIFICATIONS` is still deferred to Phase 5. | `2223e75` |
| 6. Web: landing, legal pages, delete-account, assetlinks | synapcart_web (+ backend, mobile) | **Done**: landing, `/privacy` (KVKK + GDPR), `/terms`, `/support`, `/delete-account`, rebuilt `/reset-password`, 404, robots, sitemap, metadata, icons and OG image. Backend adds `POST /auth/delete-request` and `/auth/delete-confirm` (migration 0003; 25 pytest tests pass). The mobile app opens from `synapcart://login`. Lint and build pass, and every flow was checked end to end with Playwright against a local backend. Placeholders to replace: support email and data controller in `lib/site.ts`, the SHA-256 in `assetlinks.json`, and the Play badge artwork. | web `2ae7e59`, backend `1edc140`, mobile `81a4a36` |
| 2. Premium backend | synapcart_backend | **Done**: `plan_service` (free 5 msgs/3 products/flash-lite/10 sessions; premium 100/8/`PREMIUM_ANALYST_MODEL`/unlimited), `usage_daily` quota with 402 `{detail, code}` (`quota_exceeded`, `premium_required`, `history_locked`), tiered agent, locked sessions, `POST /billing/google/verify` (subscriptionsv2) and the RTDN push endpoint, price alerts (`PATCH /favorites/{id}`, device tokens, daily `price_check` cron with FCM HTTP v1), `POST /chat/compare`. Migration 0004; 46 pytest tests pass, `alembic check` clean. Service-account tokens are minted with python-jose + httpx, so no new dependencies. | `b99be0d` |
| 5. Mobile premium (Billing, paywall, FCM, compare) | synapcart_mobile (+ web) | **Done**: Play Billing 9 (`BillingRepository`: offers, purchase bound to the user id, server verify, acknowledge, restore on startup), real paywall with Play prices and renewal terms, 402 → paywall by code, locked chats in the drawer, price-alert dialog, FCM (`google-services` plugin applied only when `google-services.json` exists), `POST_NOTIFICATIONS` after the first favorite and on setting an alert, notification tap opens the product, `CompareScreen` calls `/chat/compare`. 18 JVM tests pass, lint has 0 errors, release builds; contract-tested against a local backend. Web pricing copy updated. Placeholders: the subscription and base-plan ids, `google-services.json`, and the Render billing/FCM secrets. | mobile `bc1acf2`, web `e2a7d26` |
| 7. Umbrella: submodule bump, README, docs | synapcart (+ backend) | **Done**: submodules point at the latest `claude/init-dno428` commits; the README is rewritten (product, plans from `plan_service`, architecture, local dev, legal links, screenshots), and the debug-APK and old video links are gone; web screenshots (landing, privacy, reset-password, delete-account at 390px and 1440px) are in `design/screenshots/`; the umbrella CLAUDE.md covers `/billing`, plans and quota, price alerts and compare. Each repo's CLAUDE.md was checked against the code. Placeholder: the app screenshots (no emulator in the cloud; the README lists which screens to capture). | backend `4ac8a26` |

The order is 4, then 6, then 2 and 5 together, then 7.

**Phase 3 follow-ups to fold into Phase 4:**
- The chat drawer needs rename and delete UI; `DashboardViewModel.renameSession` and `deleteSession` already exist.
- The Profile, Register and legal links point to `{WEB_URL}privacy`, `{WEB_URL}terms` and `{WEB_URL}support`. Phase 6 must create those pages.
- The Compose BOM (2024.09) and lifecycle (2.6.1) are old, and lint suggests upgrades. Upgrade them during Phase 4.
- `POST_NOTIFICATIONS` is deferred to Phase 5 (FCM).

## Session setup notes (cloud containers)

- **Android SDK** is not preinstalled.
  ```bash
  mkdir -p /opt/android-sdk/cmdline-tools && cd /tmp
  curl -sSL -o clt.zip https://dl.google.com/android/repository/commandlinetools-linux-13114758_latest.zip
  unzip -q clt.zip -d /opt/android-sdk/cmdline-tools && mv /opt/android-sdk/cmdline-tools/cmdline-tools /opt/android-sdk/cmdline-tools/latest
  export ANDROID_HOME=/opt/android-sdk
  yes | $ANDROID_HOME/cmdline-tools/latest/bin/sdkmanager --licenses
  $ANDROID_HOME/cmdline-tools/latest/bin/sdkmanager "platforms;android-36.1" "build-tools;36.0.0" "platform-tools"
  echo "sdk.dir=/opt/android-sdk" > local.properties   # in the synapcart_mobile root; gitignored
  ```
- **Maven Central rate-limits** the proxy with HTTP 429. Use Google's mirror through a local init script, and never commit it. Gradle treats a 429 as fatal instead of trying the next repository, so the script also points every `repo.maven.apache.org` repository (including the plugin portal's redirect target) at the mirror:
  ```bash
  mkdir -p ~/.gradle/init.d && cat > ~/.gradle/init.d/mirror.gradle.kts <<'EOT'
  val mirror = "https://maven-central.storage-download.googleapis.com/maven2/"
  fun RepositoryHandler.redirectCentral() {
      all {
          if (this is MavenArtifactRepository && url.host == "repo.maven.apache.org") url = uri(mirror)
      }
  }
  beforeSettings {
      pluginManagement.repositories { maven(mirror); redirectCentral() }
      dependencyResolutionManagement.repositories { maven(mirror); redirectCentral() }
  }
  EOT
  ```
  Then run `./gradlew --no-daemon :app:compileDebugKotlin :app:testDebugUnitTest :app:lintDebug :app:assembleRelease`.
- **Backend** needs a venv and `pip install -r requirements-dev.txt`, then `pytest`. To run it locally with dummy keys:
  `ENV=dev GEMINI_API_KEY=x SERP_API_KEY=x DATABASE_URL=sqlite:////tmp/sc.db SECRET_KEY=<32+ chars> SMTP_USER=x SMTP_PASSWORD=x uvicorn app.main:app --port 8001`.
  The agent endpoints need real Gemini and SerpAPI keys; everything else works with dummy keys.
- **Mobile ↔ backend contract check:** write a temporary JVM test that builds Retrofit against `http://localhost:8001/` with the real `*ApiService` interfaces and `safeApiCall`, run it while the backend is up, then delete it. This verified every Phase 3 endpoint.
- **Web** needs `npm install`, then `npm run lint && npm run build`. Use the preinstalled Playwright and Chromium for screenshots.
- **Conventions:** user-facing text is Turkish (EN added in Phase 4). Commit messages are in English. Push with `git push -u origin claude/init-dno428`. Don't open PRs unless asked.

## Context (original audit, before Phase 1)
SynapCart is a working competition prototype. A full audit of all three repos found:
- **Security blockers:**
  - Anyone can read any chat (`GET /chat/agent/{id}/messages` has no auth).
  - Open CORS.
  - No rate limiting.
  - The release APK logs passwords and tokens (OkHttp `Level.BODY`).
  - Cleartext traffic is allowed.
- **Real bugs:**
  - Login tokens expire after 15 minutes because `expires_delta` isn't passed (`auth.py:35`).
  - The stream endpoint saves status text like "Router karar verdi…" into the answer (`chat.py:129`).
  - The 401 → logout path can never fire, because the repositories wrap `HttpException`.
  - Favorite toggles only ever add, which creates duplicates.
  - The onboarding flag is wiped on logout.
  - `java.time` crashes on API 24–25.
  - Chat history is ignored by the agent.
- **Publishing gaps:**
  - No release signing or R8, no privacy policy or terms, no web account-deletion URL, no `assetlinks.json`.
  - The web landing page is still the create-next-app placeholder.
  - There are no migrations, and `requirements.txt` is unpinned and missing `httpx`.
- **No monetization:** there is no billing or plan code anywhere.
- **Design:**
  - Two competing palettes (teal `#27D1B4` vs cyan `#00C3FF`), with colors redefined inside `DashboardScreen`.
  - 62 hardcoded colors and mixed Serif/Sans fonts.
  - Light mode is half-broken.
  - Only 2 shared components exist.
  - Edge-to-edge insets are applied twice, and the keyboard covers the input.

**Decisions made by the owner:**
- **Monetization:** a freemium Play subscription.
- **Premium features:** more results with deep analysis, price-drop alerts, product comparison, and unlimited history.
- **Design:** a refined dark/neon look, dark-only.
- **Languages:** Turkish and English.

Work is phased so each phase can ship on its own. Commit and push per repo on `claude/init-dno428`.

---

## Phase 1: Backend hardening (`synapcart_backend`), DONE
> This is the original plan. The implementation differs in places: no gunicorn, a migration script plus `AUTO_MIGRATE`, and the `google-genai` SDK. The backend `CLAUDE.md` describes the actual state.

1. **Dependencies and deploy**
   - Pin `requirements.txt`: add `httpx`, `uvicorn[standard]`, `gunicorn` and `slowapi`; remove the duplicate `sqlalchemy`, `langchain-google-genai` and `python-dotenv`.
   - Add `render.yaml` with a start command that runs `alembic upgrade head`, then gunicorn with uvicorn workers.
2. **Alembic**
   - Init Alembic and write a baseline migration from the current models.
   - Remove `create_all` from `main.py` (keep it behind `settings.ENV == "dev"`).
3. **Config** (`core/config.py`)
   - Add `ENV`, `CORS_ORIGINS` (list), `ACCESS_TOKEN_EXPIRE_MINUTES` (default 7 days) and the billing/FCM settings listed below.
   - Validate that `SECRET_KEY` is at least 32 characters.
   - Disable `/docs` in prod.
4. **Security fixes**
   - `chat.py:36`: require auth and an ownership check on `/messages`.
   - `auth.py:35`: pass `expires_delta` from settings, and return `user_name` in `Token`.
   - CORS: use the explicit origins from settings.
   - Rate limiting with `slowapi`:
     - auth: 5/min per IP
     - forgot-password: 3/hour per email
     - agent: per user
   - Lowercase emails on register and login.
   - Password rules: 8–72 bytes.
   - Field length limits; `ChatRequest.message` capped at 1000 characters.
   - `token_version` on `User`, embedded in the JWT and bumped on password reset and deletion (`deps.py` checks it).
   - Hash reset tokens (SHA-256), use `RESET_TOKEN_EXPIRE_HOURS`, and return the same response even when SMTP fails.
   - `DELETE /user/me` requires the password in the body.
5. **Agent fixes** (`services/agent_service.py`, `gemini_service.py`)
   - Make the graph the single source: implement streaming via `app_agent.astream(stream_mode="updates")` and map node completions to the existing SSE `status` values (`routing` / `searching` / `thinking` / `completed` / `out_of_scope` / `error`). **Keep the SSE field names.** Delete the hand-written `stream_agent_invoke` and the duplicated block in `chat.py`.
   - Save only the final analysis. Stop leaking "Router karar verdi" to clients.
   - Pass the history and the user's question into the keyword and analyst prompts, and add a "reply in {language}" instruction. Fix the double-appended current message.
   - Check the SerpAPI status and skip the analyst when there are 0 products. Catch Gemini blocked or empty responses and add timeouts.
   - Fix the refusal-string typos (`agent_service.py:75,78,79`).
   - Persist `products` JSON on `ChatMessage` (new nullable column), and return it from `/messages` so history restores the product cards.
6. **Missing endpoints:** `DELETE` and `PATCH /chat/sessions/{id}`, `POST /user/me/password`, `/chat/sessions` ordered newest-first, and a `/health` that pings the DB.
7. **Logging:** replace `print` with `logging` (no message content), and add a `RequestValidationError` handler that returns a Turkish or English `detail`.
8. **Favorites:** add a unique constraint on (`user_id`, `link`), `currency`, `created_at`, and a nullable `price` (SerpAPI can return none). Add `DELETE /favorites/by-link` or return the existing ID on a duplicate add.

## Phase 2: Premium on the backend
- **User:** add `plan` (`free`/`premium`), `plan_expires_at`, `play_purchase_token`, `created_at`, and the new table `usage_daily(user_id, date, messages)`. All via an Alembic migration.
- **Quota:**
  - Add `app/services/plan_service.py` with `get_limits(user)`. Free: 5 agent messages/day, 3 products, flash-lite, last 10 sessions. Premium: 100/day soft cap, 8 products, a stronger Gemini model, unlimited sessions.
  - Add a `require_quota` dependency next to `get_current_user_required` (`api/deps.py`), used on both agent routes. It returns 402 with a `detail` plus `code: "quota_exceeded"`.
- **Agent tiering:** thread `limits` through `AgentState`. `searcher_node` slices `[:limits.max_products]` (currently `[:3]`, `agent_service.py:98`), and `gemini_service` takes a per-call model name.
- **Billing**
  - New `api/v1/endpoints/billing.py`:
    - `POST /billing/google/verify {purchase_token, product_id}` validates through the Google Play Developer API (`purchases.subscriptionsv2.get`) using a service account (`GOOGLE_PLAY_SERVICE_ACCOUNT_JSON`, `PLAY_PACKAGE_NAME` in settings), then sets the plan and expiry.
    - `POST /billing/google/rtdn` is the Pub/Sub push endpoint for renewals and cancellations, protected by a shared token.
    - `GET /user/me` now includes `plan`, `plan_expires_at` and `usage_today`/`daily_limit`.
- **Price alerts:** `Favorite.target_price` and `last_checked_price`, plus `PATCH /favorites/{id}` to set the target. `app/jobs/price_check.py` re-queries SerpAPI for premium users' favorites (exact-title search, match by link or source) and sends FCM through `firebase-admin` when price ≤ target. It runs as a Render Cron Job (daily) declared in `render.yaml`. A new `device_tokens(user_id, token)` table backs `POST /user/me/device-token`.
- **Comparison:** `POST /chat/compare {products: [...2–4]}` (premium only) makes one Gemini call that returns a structured JSON comparison (pros/cons per item plus a verdict) in the user's language.
- **History limit:** free users' `/chat/sessions` returns the last 10; older ones are kept but marked `locked`.

## Phase 3: Mobile, release readiness and bugs (`synapcart_mobile`), DONE
> This is the original plan. The mobile `CLAUDE.md` describes the actual state. `POST_NOTIFICATIONS` moved to Phase 5.

- **Build** (`app/build.gradle.kts`)
  - Enable `buildConfig` and set `BASE_URL` per build type (debug `http://10.0.2.2:8001/`, release Render). Delete `Constants.kt` and use `BuildConfig.BASE_URL` in `NetworkModule` and `ChatStreamRepository`.
  - Release: `isMinifyEnabled`/`isShrinkResources = true` and a signing config read from `keystore.properties` (gitignored).
  - Add keep rules for `data/model/**`, or annotate the DTOs with `@SerializedName`/`@Keep`.
  - Bump to versionCode 2, versionName 1.0.0.
  - Enable core library desugaring for `java.time` (removes the API 24–25 crash).
- **Manifest**
  - Remove `usesCleartextTraffic`; add a `network_security_config` that allows cleartext only for `10.0.2.2` in debug.
  - `allowBackup=false`, or exclude the datastore in the backup rules.
  - Add `POST_NOTIFICATIONS`.
- **Network:** logging at `NONE` in release and `BODY` in debug, redacting `Authorization`.
- **Session expiry:** add an OkHttp `Authenticator` or interceptor that, on 401, clears the session and emits on a new `SessionEvents` flow. `SynapCartNavHost` collects it and navigates to Login. Remove the dead `HttpException` checks in `DashboardViewModel`.
- **Error helper:** one `safeApiCall {}` in `data/remote/` that replaces the roughly 16 copy-pasted try/catch blocks. It maps IOException to a localized "connection" message and 402 to a typed `QuotaExceeded` error.
- **SSE** (`ChatStreamRepository`): check `response.isSuccessful` and emit an error event; use `awaitClose { call.cancel() }`; remove the `!!`.
- **Fixes**
  - `SessionManager.clearSession` keeps the onboarding flag.
  - Real favorite toggle state, from a `favoriteLinks` set in a shared `FavoritesRepository` cache, that adds and removes.
  - Register doesn't log in twice.
  - Show prices in the user's currency.
  - WebView `update` no longer reloads.
  - Navigate via `Screen.*.route` instead of string literals.
- **Product navigation:** stop passing JSON in routes. Use a `SelectedProductHolder` singleton (or a `SavedStateHandle` key) and pass only an id.
- **Dead code:** delete guest-mode leftovers, `PlaceholderScreen`, `Screen.Chat`, `SendMessageUseCase`, and the unused domain models.
- **Compliance UI:** in Profile, add Privacy Policy and Terms links (to web), show the app version, and ask for the password in the delete-account dialog. On Register, add a consent line with links.

## Phase 4: Mobile design system and overhaul
- **Tokens** (`ui/theme/`)
  - **Color.kt:** one palette derived from the logo:

    | Token | Value |
    |---|---|
    | bg | `#060B14` |
    | surface | `#0E1624` |
    | surfaceHigh | `#152034` |
    | outline | `#22304A` |
    | primary (brand teal) | `#2DD4B4` |
    | secondary (cyan) | `#33D6FF` |
    | brand gradient | primary → secondary |
    | error | `#F87171` |
    | warning | `#F59E0B` |
    | premium gold | `#F5C451` |
    | text | `#E8EEF8` |
    | textMuted | `#9AA8C1` (meets WCAG AA on bg) |

    Fill every M3 role in `darkColorScheme`, and delete `DarkBg`/`NeonAccent`/`SurfaceDark` from the Dashboard.
  - **Type.kt:** a Plus Jakarta Sans downloadable or bundled font with the full M3 type scale. No more inline `fontSize`.
  - **Shape.kt:** 8/12/16/24 dp scale.
  - **Spacing:** a `Dimens` object.
  - **Theme.kt:** dark-only; drop `statusBarColor`; switch the window theme to `Theme.SplashScreen` plus `core-splashscreen` (no white flash).
- **Components** (`ui/components/`)
  - `SynapTopBar`
  - `PrimaryButton` (gradient), `SecondaryButton`
  - `SynapTextField`
  - `ProductCard` (image with Coil placeholder, price chip, favorite button with 48dp target and contentDescription)
  - `ChatBubble` (user/AI; replace the regex markdown with a small `AnnotatedString` builder that handles bold and bullets)
  - `StatusIndicator` (animated agent steps)
  - `SelectableChip` (with `Role.RadioButton`)
  - `EmptyState`, `ErrorState` (with retry), `LoadingSkeleton`
  - `PremiumBadge`, `QuotaBanner`
- **Insets:** remove `innerPadding` from the root Scaffold in `MainActivity` and let screens own insets. Add `imePadding()` on the Dashboard input. Host snackbars per screen, or offset them above the input.
- **Screens**
  - Login, Register and Splash share one `BrandHeader`, are scrollable and theme-driven.
  - Onboarding is a 2–3 step pager (value proposition, language/currency, premium teaser).
  - Dashboard:
    - Drawer: session list with relative dates, plus rename and delete.
    - Top bar: logo mark and plan badge.
    - Products: `LazyVerticalGrid` of `ProductCard`s.
    - Suggestion chips that don't depend on emoji substring tricks.
    - Quota banner, e.g. "3/5 messages left today".
  - Product details: hero image, price, store, an "Add price alert" button (premium) and a "Compare" button (premium).
  - Favorites: grid, alert state shown on each card, multi-select into Compare.
  - Profile: plan card (upgrade / manage subscription), settings, and links.
  - New screens: `PaywallScreen` and `CompareScreen`.
- **Strings:** move every string into `res/values/strings.xml` (Turkish) and `values-en/strings.xml`. The language picker applies `AppCompatDelegate.setApplicationLocales` and PATCHes the backend. Add `locales_config.xml`.
- **Icon:** a proper monochrome layer (a single-color vector of the cart mark) for themed icons.

## Phase 5: Mobile premium features
- **Billing**
  - Add `com.android.billingclient:billing-ktx` (v7+).
  - New `data/billing/BillingRepository` that connects, queries `ProductDetails` for `synapcart_premium` (monthly and yearly base plans), launches the flow, sends the purchase token to `/billing/google/verify`, then acknowledges it and refreshes `/user/me`.
  - Restore purchases on startup.
  - Add a `PlanState` flow in a `UserPlanRepository` that the UI observes.
- **Paywall:** feature comparison table, monthly and yearly toggle (yearly shows the saving), "Manage subscription" deep link to the Play subscriptions page, and the legally required renewal/cancel text.
- **Gating:** when the server returns 402 (`QuotaExceeded`), the Dashboard shows the paywall sheet. Premium-only buttons open the paywall for free users.
- **Push:** Firebase Messaging (adds the `google-services` plugin and `google-services.json`, which you must provide). Register the token with `/user/me/device-token`, ask for notification permission after the first favorite, and deep-link price-alert notifications to the product.
- **Comparison:** `CompareScreen` calls `/chat/compare` and renders a column per product with pros/cons and the verdict.
- Optionally add Crashlytics in the same Firebase setup.

## Phase 6: Web (`synapcart_web`)
- **Design tokens:** move the reset page's palette into `globals.css` CSS variables matching the mobile tokens. Use `next/font` with Plus Jakarta Sans and remove the DM Sans `@import`. Set `lang="tr"`.
- **Layout and metadata:** real metadata in `layout.tsx` (title template, description, `metadataBase`, OpenGraph and Twitter, theme color). Add the icon and OG image from `synapcart/design/logo_foreground.png`. Delete the template SVGs.
- **Pages**
  - `/`: landing page with hero, features, how it works, free vs premium pricing, a Play Store badge, FAQ and footer.
  - `/privacy`: KVKK plus GDPR-style, covering data collected (email, name, chats, favorites), processors (Google Gemini, SerpAPI, Render, Vercel, Firebase) and retention.
  - `/terms`, including subscription and refund terms that point to Google Play.
  - `/delete-account`, with in-app instructions and a form that POSTs to a new backend `POST /auth/delete-request`, which emails a confirm link. This satisfies Play's requirement for a web deletion URL.
  - `/support`, a 404 page, `robots.ts` and `sitemap.ts`.
- **Reset page:** extract it into `components/`; add accessible labels, `aria-label`s, visible focus and an `aria-live` error; let users retry or "request a new link"; parse non-JSON errors; make the success button open `synapcart://` and fall back to `/`.
- **`public/.well-known/assetlinks.json`:** add it with `com.neilb.synapcart` and the SHA-256 of the Play app signing certificate (a placeholder until you provide it).
- **Env:** `.env.example`; `next.config.ts` throws if `NEXT_PUBLIC_API_URL` is missing at build time. Fix the README's "Next.js 14".
- **Domain:** the app's deep links use `synapcart.app`. Either point that domain at Vercel or switch the manifest and `APP_FRONTEND_URL` to the domain you actually own. The plan defaults to keeping `synapcart.app`.

## Phase 7: Umbrella and docs
- Bump the submodule pointers.
- Update the README with screenshots, premium, and remove the debug-APK link.
- Update each CLAUDE.md: token expiry, Alembic, the plan/quota system, billing, and the single agent stream.

---

## Owner setup for Phases 2 and 5 (placeholders in the code until done)
- **Play Console:** create the subscription `synapcart_premium` with base plans `monthly` and `yearly` (or change `PREMIUM_PRODUCT_ID` / `PREMIUM_PLAN_*` in `mobile/app/build.gradle.kts` and `PLAY_SUBSCRIPTION_IDS` on the backend). Add license testers and upload a build to a test track; billing only works for apps installed from Play.
- **Service account** for verification: link a Google Cloud project in Play Console, give the service account "View financial data" (and "Manage orders" if you want refunds from the API), then put the key JSON in Render as `GOOGLE_PLAY_SERVICE_ACCOUNT_JSON`.
- **RTDN:** create a Pub/Sub topic, set it under Play Console › Monetization setup, and add a push subscription to `https://synapcart-backend.onrender.com/api/v1/billing/google/rtdn?token=<PLAY_RTDN_TOKEN>` (any long random string, also set on Render).
- **Firebase:** create a project with the Android app `com.neilb.synapcart` (add the debug and Play app-signing SHA-256s), put `google-services.json` in `mobile/app/`, and set a service-account key with the "Firebase Cloud Messaging API Admin" role as `FIREBASE_SERVICE_ACCOUNT_JSON` on Render (the web service and the `synapcart-price-check` cron job, which is a paid Render service).

## Things only the owner can provide (use placeholders until then)
- Play Console app, the `synapcart_premium` subscription with monthly and yearly base plans, an upload keystore, the app-signing SHA-256, and a service account with "View financial data" for verification.
- A Firebase project and `google-services.json`, plus an FCM service account for the backend.
- The domain (`synapcart.app`?), a support email, and legal entity details for the privacy policy.
- Render env vars: `CORS_ORIGINS`, `ENV=prod`, the billing and FCM secrets, and a strong `SECRET_KEY`.

## Verification
- **Backend:**
  - Add `pytest` with `tests/` using SQLite and mocking Gemini and SerpAPI with monkeypatch. Cover auth (expiry, token_version invalidation), the ownership check on `/messages`, quota 402 after 5 messages, the SSE event sequence and the persisted final text, the verify endpoint (mocked Google API), and the compare schema.
  - Run `uvicorn` locally and curl the stream endpoint.
- **Mobile:**
  - `./gradlew lint test assembleRelease`, which checks R8 keep rules, using a debug-signed release locally.
  - Add ViewModel unit tests for the Dashboard stream handling and the favorite toggle.
  - Manual pass on an emulator against the local backend: register, onboarding, chat (stream), quota paywall, Play Billing test track with license testers, favorite and alert, compare, 401 expiry back to Login, EN locale, keyboard and insets, TalkBack on the main flows.
- **Web:**
  - `npm run lint && npm run build`.
  - Use Playwright to screenshot `/`, `/privacy`, `/reset-password?token=x` and `/delete-account` at phone and desktop widths.
  - Check `/.well-known/assetlinks.json` with Google's Statement List tester after deploy.

## Suggested execution order
Phase 1 → 3 (ships a safe free version) → 4 (design) → 6 (web legal pages are needed before the store submission) → 2 + 5 (premium) → 7.
