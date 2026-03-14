# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
pnpm dev          # Next.js dev server with Turbopack (http://localhost:3000)
pnpm build        # Production build (Turbopack)
pnpm start        # Run production server
pnpm lint         # ESLint
pnpm test:db      # Validate MongoDB connectivity (runs scripts/test-db.mjs)

# Inngest local dev (background jobs, cron, AI)
npx inngest-cli@latest dev
```

**Note:** Both `pnpm` and `npm` are supported. `next.config.ts` has `ignoreDuringBuilds: true` for both ESLint and TypeScript, so builds won't fail on type errors.

## Architecture Overview

OpenStock is a **Next.js 15 App Router** stock market tracking app. Key architectural decisions:

### Route Groups
- `app/(auth)/` — sign-in, sign-up pages (public)
- `app/(root)/` — all protected pages (dashboard, watchlist, stocks/[symbol], etc.)
- `app/api/inngest/route.ts` — Inngest webhook endpoint

### Authentication Flow
Auth uses **Better Auth** with a MongoDB adapter. The auth instance is lazily initialized as a singleton in `lib/better-auth/auth.ts` (connecting to MongoDB on first call). Middleware at `middleware/index.ts` checks only for the session cookie presence (not full validation) and redirects unauthenticated users to `/sign-in`. Public paths excluded from middleware: `api`, `_next/*`, `favicon.ico`, `sign-in`, `sign-up`, `assets`.

### Data Layer
- **MongoDB + Mongoose** via `database/mongoose.ts` (cached connection)
- Models: `database/models/watchlist.model.ts`, `database/models/alert.model.ts`
- Better Auth manages the `user` and `session` collections directly

### Server Actions (`lib/actions/`)
All data fetching goes through server actions:
- `finnhub.actions.ts` — Finnhub API calls (quotes, profiles, news, search). Uses `NEXT_PUBLIC_FINNHUB_API_KEY` server-side too (not just client). Caching strategy: profiles cached 24h, search 30m, news 5m, quotes no-store.
- `watchlist.actions.ts` — CRUD for user watchlists
- `alert.actions.ts` — CRUD for price alerts
- `auth.actions.ts` — sign-in/sign-up/sign-out wrappers
- `user.actions.ts` — user profile queries for Inngest

### Background Jobs (Inngest — `lib/inngest/functions.ts`)
- `app/user.created` event → AI-personalized welcome email via Gemini (`gemini-2.5-flash-lite`), with Siray.ai (`SIRAY_API_KEY`) as fallback
- Cron `0 9 * * 1` (Monday 9AM) → Weekly market news summary broadcast via Kit (ConvertKit API at `lib/kit.ts`)
- Cron `*/5 * * * *` → Check stock price alerts against Finnhub quotes, mark triggered alerts in MongoDB
- Cron `0 10 * * *` → Re-engagement emails for users inactive >30 days

### Email
- **Nodemailer** (`lib/nodemailer/`) — Gmail transport for direct transactional emails (welcome email)
- **Kit/ConvertKit** (`lib/kit.ts`) — broadcast newsletters (weekly news summaries, re-engagement). Requires `KIT_API_KEY`, `KIT_API_SECRET`, optionally `KIT_WELCOME_FORM_ID`.

### TradingView Widgets
TradingView widgets are embedded via `components/TradingViewWidget.tsx` using the `useTradingViewWidget` hook (`hooks/useTradingViewWidget.tsx`). All widget configs live in `lib/constants.ts`. Symbol format conversion for A-Share/HK stocks (`.SS` → `SSE:`, `.SZ` → `SZSE:`, `.HK` → `HKEX:`) is handled by `formatSymbolForTradingView` in `lib/utils.ts`.

### Styling
Tailwind CSS v4 via `@tailwindcss/postcss` (no `tailwind.config` file needed). shadcn/ui components in `components/ui/`. The `cn()` utility in `lib/utils.ts` merges Tailwind classes.

## Environment Variables

Required:
```
MONGODB_URI
BETTER_AUTH_SECRET
BETTER_AUTH_URL
NEXT_PUBLIC_FINNHUB_API_KEY
FINNHUB_BASE_URL=https://finnhub.io/api/v1
GEMINI_API_KEY
INNGEST_SIGNING_KEY
NODEMAILER_EMAIL
NODEMAILER_PASSWORD
```

Optional (for Kit/newsletter and Siray.ai fallback):
```
KIT_API_KEY
KIT_API_SECRET
KIT_WELCOME_FORM_ID
SIRAY_API_KEY
```

For Docker local dev, use `MONGODB_URI=mongodb://root:example@mongodb:27017/openstock?authSource=admin`.

## Key Patterns

- **Server actions** use `'use server'` directive and are called directly from components — no separate API routes for data.
- The **watchlist** stores one document per user with an array of symbols (unique per user enforced at DB level).
- **Alert model** uses `condition: 'ABOVE' | 'BELOW'` (not `alertType`). The `lib/utils.ts` `getAlertText` function references a legacy `Alert` type with `alertType`/`threshold` — these types may diverge from the DB model.
- Image domains allowlisted in `next.config.ts`: `i.ibb.co` and `static2.finnhub.io`.
- `lib/constants.ts` holds all TradingView widget configurations and the `POPULAR_STOCK_SYMBOLS` list used for default search results.
