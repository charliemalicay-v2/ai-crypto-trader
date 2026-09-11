# AICryptoTrader / "Coin Rich" — Project Summary

> Condensed overview (Parts 2-6). Full detail and the security deep-scan are in
> `PROJECT_SYNTHESIS.md`.

---

## What it is

A **client-side cryptocurrency portfolio tracker and market dashboard**. Despite
the "AICryptoTrader" name, it does **no trading** — no exchange integration, no
order placement, no wallet signing. Scaffolded on **Lovable**, then hand-extended
with Clerk auth and a Supabase persistence layer.

Names in use: repo `AICryptoTrader`, package `vite_react_shadcn_ts`, browser title
`Coin Rich`, header brand `CryptoTracker`.

## Stack

| Layer | Technology |
|---|---|
| Build | Vite 5.4, `plugin-react-swc`, dev port 8080 |
| Language | TypeScript 5.5 — strictness largely **disabled** |
| UI | React 18.3, Tailwind 3.4, full shadcn/ui (49 primitives) |
| Routing | react-router-dom 6.26 |
| Server state | TanStack Query 5.56 (polling, no websockets) |
| Charts | Recharts 2.15 + embedded TradingView widget |
| Identity | Clerk (`@clerk/clerk-react` 5.31) |
| Database | Supabase Postgres + RLS |
| Backend | **None** (no server / API routes / edge functions) |

~9,400 LOC total; ~50% is generated shadcn/ui boilerplate.

---

## How it works

### Auth — dual-identity bridge (the most sophisticated part)

Clerk owns identity, Supabase owns data, a Clerk-minted JWT links them:

1. `ClerkProvider` boots with `VITE_CLERK_PUBLISHABLE_KEY`.
2. User signs in via Clerk `<SignIn/>` at `/auth`.
3. `AuthContext` calls `getToken({ template: 'supabase' })` — Clerk mints a JWT
   **signed with the Supabase JWT secret**.
4. `AuthContext` upserts a `profiles` row keyed by Clerk user ID.
5. Every DB call builds a per-request Supabase client with
   `Authorization: Bearer <clerk-jwt>`.
6. Postgres RLS gates every row on `user_id = jwt.claims->>'sub'`.

Token handling (`src/contexts/AuthContext.tsx`): manual `exp` check via
base64-decode, lazy refresh on expiry, background force-refresh every 45 min.
`ProtectedRoute` is client-side only — real enforcement is RLS at the DB.

**Hard dependency:** a Clerk JWT template named exactly `supabase` must exist or
the entire Portfolio feature throws.

### Data sources

```
CoinGecko (no API key)  -> MarketStats, CryptoList, Market page, CryptoChart,
                            portfolio symbol resolution + live prices
alternative.me          -> Fear & Greed Index (real, mock fallback)
s3.tradingview.com      -> TradingView widget (runtime <script> inject)
api.covalenthq.com      -> Screener wallet lookup (public "ckey_demo" key)
Supabase                -> portfolio_holdings + profiles (RLS-scoped)
```

### Portfolio engine (`src/services/portfolioApiService.ts`, ~344 LOC — the real logic)

- **Symbol resolution:** 3-tier CoinGecko fallback chain + 5-min in-memory price cache.
- **Aggregation:** multiple buys of one symbol collapse into a position with a
  correct weighted-average cost basis; live prices batch-fetched in one
  `/simple/price` call.
- **Failure mode:** price fetch error -> price `0` -> silently shows -100% loss.
- **Writes:** delegated to `portfolioService.ts` via lazy `await import()`
  (code-split chunk).

### Data model

```
portfolio_holdings(id, user_id TEXT, symbol, name, coin_id, amount NUMERIC,
                   avg_price NUMERIC, purchase_date DATE, notes, timestamps)
profiles(id, email, full_name, avatar_url, timestamps)
```

`user_id` is `TEXT` to match Clerk string IDs. Schema stores individual
**transactions**, not positions — aggregation is done client-side (hence the
"N transactions" + per-entry delete buttons in the UI).

---

## Feature reality check

**Roughly half the advertised features are non-functional facades.**

| Route | Status | Reality |
|---|---|---|
| `/` Landing | Real | Marketing page, presentation only |
| `/auth` | **Real** | Clerk `<SignIn/>`/`<SignUp/>`, custom dark theme |
| `/portfolio` | **Real** | Full CRUD, Supabase-persisted, live pricing, weighted-average P&L. **The genuine core.** |
| `/market` | Real | Live CoinGecko top-100, global stats, trending |
| `/dashboard` | **Mixed** | Market data + Fear&Greed real; **SentimentAnalysis & MarketPulse fabricated** |
| `/screener` | **Mostly fake** | 6 of 7 "VC" entries are hardcoded mocks; only the Alameda wallet row is real |
| `/chat` | **Fake** | No LLM — keyword `if/else` returning canned text behind a fake 1.5s delay |
| `/nft` | **Placeholder** | "Coming Soon"; its 5 referenced images don't exist |

### Specific fabrications

- **Sentiment / Market Pulse** (`marketDataService.ts`): pure `Math.random()`,
  re-rolled every 30s-5min.
- **Portfolio Analytics** (`PortfolioAnalytics.tsx`): the "performance chart"
  back-projects fixed multipliers off current value (`YTD = value * 0.45`), so
  every user sees an identical curve. "Best Performer +34.2%", "127 days",
  "Risk: Medium" are hardcoded. The 4 Quick Action buttons have no handlers.
- **PortfolioCard** (`PortfolioCard.tsx`): falls back to invented figures
  ($41,550, fake BTC/ETH holdings) when no real data — new users see a fictional
  portfolio.

---

## Engineering assessment

### Strengths
- Clerk<->Supabase JWT bridge is non-trivial and correctly implemented
- RLS properly enforces security at the database
- Portfolio math is correct (weighted-average cost basis, proper P&L)
- Good query hygiene: per-source `staleTime`/`refetchInterval`, batch fetching,
  dynamic imports, memoization
- Cohesive black/red/orange visual design with custom CSS animations

### Weaknesses
| Issue | Detail |
|---|---|
| Rate limiting | 12 keyless CoinGecko calls + aggressive polling -> will 429 on free tier |
| Type safety off | `strictNullChecks`, `noImplicitAny`, `no-unused-vars` all disabled |
| No tests / no CI | Zero test files |
| Prod debug logging | ~25 `console.log`, including token-state logs |
| Silent failures | Price errors render as -100% loss, not an error state |
| Dead code | `src/pages/index.tsx` unrouted; `zod`/`date-fns`/`@hookform/resolvers` unused |
| Bundle | Single 1.28 MB chunk, no manual chunking |
| Missing assets | 5 NFT images referenced, none present |

---

## Current state (post-session)

| Area | State |
|---|---|
| Malware | Removed from `vite.config.ts` + `postcss.config.js`; verified zero remnants |
| `.gitignore` | Now covers `.env`, `*.env`, `.claude/`, `.agents/`, `skills-lock.json` |
| Secrets | `back.env` deleted; `.env` reduced to `VITE_CLERK_PUBLISHABLE_KEY` only |
| Clerk | Relinked to app `app_3IraXfhohQI3jDAODlHFjDpgl2X` ("Test App", instance `deep-buffalo-9291`) |
| Deps | 359 installed via `--ignore-scripts`; dropped dead `axios` + `socket.io-client` |
| Vite | Added `server.watch.ignored` for `.agents`/`.claude` (fixes chokidar `EBUSY` crash) |
| Build/typecheck/dev | All pass; dev server serves HTTP 200 |

### Known-open items
1. **Create the `supabase` JWT template** in the new Clerk app (name exactly
   `supabase`, signed with the Supabase JWT secret) — otherwise `/portfolio`
   401s on every request. This is the one blocker to the app's only genuinely
   functional feature.
2. **Old data won't carry over** — new Clerk instance = new user IDs = existing
   `portfolio_holdings` / `profiles` rows won't match RLS.
3. **4 npm advisories** (`esbuild <=0.24.2` via Vite; `react-router` 6.x
   open-redirect) — known framework CVEs, breaking-major fixes only.

---

## Bottom line

Visually accomplished, architecturally sound in parts, but substantially hollow.
The Clerk->Supabase auth bridge and the portfolio tracking engine are real,
correct, non-trivial work. Everything branded "AI" — chat, sentiment, market
pulse, predictive analytics — is `Math.random()` and hardcoded strings. Honest
description: **"a crypto portfolio tracker with a market dashboard,"** not an AI
trading platform.
