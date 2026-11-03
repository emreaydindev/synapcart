# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This is the **umbrella repo** for SynapCart, an AI-assisted shopping assistant (BTK competition project). It holds no application code, only git submodules, `design/` assets and the README. Each submodule is its own GitHub repo with its own CLAUDE.md:

| Path | Repo | Stack |
|---|---|---|
| `backend/` | `synapcart_backend` | FastAPI, SQLAlchemy, LangGraph, Gemini, SerpAPI (Google Shopping) |
| `mobile/` | `synapcart_mobile` | Kotlin, Jetpack Compose, Hilt, Retrofit/OkHttp |
| `web/` | `synapcart_web` | Next.js (App Router), password-reset page only |

```bash
git submodule update --init --recursive   # fetch all modules
```

Make code changes in the submodule's own repo. Commits here only bump submodule pointers (for example "chore: sync submodules").

## Roadmap

`docs/ROADMAP.md` is the publishing roadmap: the phase status table, the remaining work (design overhaul, web legal pages, premium/billing), the owner's decisions and the setup notes for cloud sessions (Android SDK, Maven mirror). Read it before starting any roadmap work, and update its Status table when a phase lands.

## How the pieces connect

- The **backend** serves everything under `/api/v1` (auth, chat, favorites, user) and runs locally on port **8001**. In production it runs on Render (`https://synapcart-backend.onrender.com`). The free tier causes slow cold starts.
- The **mobile** app hard-codes its backend in `util/Constants.kt` (`BASE_URL`). To develop against a local backend, point it at port 8001. Agent chat uses the SSE endpoint `POST /api/v1/chat/agent/{id}/stream`, whose `{status, analysis, products}` events the app parses directly.
- **Password reset** crosses all three: the backend emails `{APP_FRONTEND_URL}/reset-password?token=…`, the **web** portal (Vercel, `https://synapcart-web.vercel.app`) renders the form and POSTs `{token, new_password}` to `/api/v1/auth/reset-password` through its Next.js rewrite, and the mobile app also registers deep links for `https://synapcart.app/reset-password` and `synapcart://reset-password`.
- When you change an API response shape, update the backend schema, the mobile DTOs in `data/model` and, for auth/reset, the web page together.

All user-facing text across the project is in Turkish.
