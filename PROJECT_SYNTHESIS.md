# AICryptoTrader / "Coin Rich" — Full Synthesis & Deep-Scan Verification

> Generated 2026-09-04. Covers a security deep-scan, a project specification,
> and an explanation of how the system works. Reflects the repository state
> **after** the malware remediation, `npm install`, and Clerk relinking performed
> in the same session.

---

## PART 1 — Deep-Scan Verification

Full audit re-run against the current tree (post-remediation, post-`npm install`,
post-`skills add`).

### Security scan results

| # | Check | Result |
|---|---|---|
| A | Full file enumeration (107 source/config files) | Baselined |
| B | Obfuscation & loader markers (`_$_`, `jsoToArr`, `createRequire`, `fromCharCode(1xx)`, `global[...]`) | **CLEAN** — 0 hits |
| C | Config file size/line-length sanity | All normal — longest line in any config is **77 chars** (was 1,767+ when infected) |
| D | Dynamic execution (`eval`, `new Function`, string timers) | **CLEAN** |
| E | Node process/FS/OS access in app code | **CLEAN** |
| F | Whitespace-concealment runs (>=80 spaces) | **CLEAN** |
| G | Non-ASCII / bidi / homoglyph | 1 hit — fire emoji at `src/pages/Market.tsx:148` |
| H | SVG `<script>`/`onload` injection | **CLEAN** |
| I | Base64 blobs & decoders | Only the Supabase anon JWT + legit `atob` JWT-expiry parse at `src/contexts/AuthContext.tsx:61` |
| J | `package.json` lifecycle scripts | **None** (`dev, build, build:dev, lint, preview`) |
| K | Non-registry dep specs (git/URL/file) | **None** |
| L | Registry hijack (`.npmrc`/`.yarnrc`/`bunfig`) | **No such files** |
| M | Installed tree — malicious postinstalls | Only `@clerk/shared` (telemetry notice), `@swc/core` + `esbuild` (native binary download) — all inspected, benign |

### Build & quality verification

```
npx tsc --noEmit    -> PASSES, zero type errors
npm run build       -> PASSES, 2658 modules, 1.28 MB bundle (357 kB gzip)
npm run dev         -> HTTP 200, stable
npm run lint        -> 28 problems (17 errors, 11 warnings)
```

Lint errors are all style-level, not correctness: mostly
`@typescript-eslint/no-explicit-any` (e.g. `src/services/portfolioService.ts:110`)
plus one `no-require-imports` at `tailwind.config.ts:95`. Zero runtime risk.

### Verdict

**The repository is clean and safe to run.** The two injected payloads in
`vite.config.ts` and `postcss.config.js` are fully eradicated with no remnants.
Every outbound endpoint is a legitimate, expected third party. The only residual
caveat is that the internals of all 359 installed transitive packages were not
individually inspected — but all resolve to `registry.npmjs.org` and none carry
suspicious install hooks.

---

## PART 2 — What This Project Is

**"Coin Rich"** (repo `AICryptoTrader`, package `vite_react_shadcn_ts`, header
brand `CryptoTracker`) is a **client-side cryptocurrency portfolio tracker and
market dashboard**. Despite the naming, it does **no trading** — there is no
exchange integration, no order placement, no wallet signing.

It was scaffolded on **Lovable** (evidence: `lovable-tagger` build plugin,
`/lovable-uploads/` asset paths, template README) and then hand-extended with
Clerk auth and a Supabase persistence layer.

### Stack specification

| Layer | Technology |
|---|---|
| Build | Vite 5.4.21, `@vitejs/plugin-react-swc`, port **8080** |
| Language | TypeScript 5.5 — **strictness largely disabled** (`strictNullChecks: false`, `noImplicitAny: false`, `noUnusedLocals: false`) |
| UI | React 18.3, Tailwind 3.4, complete shadcn/ui set (49 Radix-backed primitives) |
| Routing | react-router-dom 6.26 |
| Server state | TanStack Query 5.56 (polling-based, no websockets) |
| Charts | Recharts 2.15 + embedded TradingView widget |
| Identity | Clerk (`@clerk/clerk-react` 5.31) |
| Database | Supabase Postgres with RLS |
| Backend | **None** — no server, no API routes, no edge functions |

**Code volume:** ~9,400 LOC, of which **4,750 (~50%) is generated shadcn/ui
boilerplate**. Actual authored application code is ~4,600 LOC.

---

## PART 3 — How It Works

### 3.1 Authentication flow — the most sophisticated part

This is a **dual-identity bridge**: Clerk owns identity, Supabase owns data, and
a Clerk-minted JWT links them.

```
 Browser
   |
   |- 1. ClerkProvider (main.tsx) boots with VITE_CLERK_PUBLISHABLE_KEY
   |
   |- 2. User signs in via Clerk <SignIn/> at /auth
   |
   |- 3. AuthContext detects clerkUser, calls:
   |       getToken({ template: 'supabase', skipCache: true })
   |     -> Clerk mints a JWT signed with YOUR SUPABASE JWT SECRET
   |
   |- 4. AuthContext upserts a `profiles` row (id = Clerk user ID)
   |
   |- 5. Every DB call builds a per-request client:
   |       createAuthedSupabaseClient(token)
   |       -> attaches  Authorization: Bearer <clerk-jwt>
   |
   `- 6. Postgres RLS evaluates:
           user_id = current_setting('request.jwt.claims')::json->>'sub'
```

**Token lifecycle** (`src/contexts/AuthContext.tsx`):
- Manual expiry check by base64-decoding the JWT payload and comparing `exp` (lines 59-68)
- `getValidToken()` lazily refreshes on expiry (lines 71-77)
- A background `setInterval` force-refreshes every **45 minutes** (lines 171-180)
- Exported to consumers as `refreshToken` — a naming quirk: the context exposes `getValidToken` *under the name* `refreshToken` (line 195)

**Route protection** (`src/components/ProtectedRoute.tsx`) is purely client-side —
it waits for `isLoaded`, then redirects to `/auth` if no user. Security is *not*
enforced here; it's enforced by RLS at the database.

**Critical dependency:** if the Clerk JWT template named exactly `supabase`
doesn't exist, `getToken()` returns null, `getAuthedClient()` throws
`"Failed to get valid authentication token"`, and the entire Portfolio feature
fails.

### 3.2 Data flow

```
CoinGecko API --+-> MarketStats      (/global,         60s poll)
   (no API key) +-> CryptoList       (/markets top-20, 30s poll)
                +-> Market page      (/markets top-100 + /search/trending)
                +-> CryptoChart      (/coins/bitcoin/market_chart, 30s)
                `-> portfolioApiService (symbol->id resolution + live prices)

alternative.me ---> FearGreedIndex   (real, 1h poll, mock fallback "64/Greed")

s3.tradingview.com -> TradingViewChart (runtime <script> inject, BINANCE:BTCUSDT)

api.covalenthq.com -> Screener wallet lookup (public "ckey_demo" key)

Supabase ---------> portfolio_holdings + profiles (RLS-scoped)
```

### 3.3 Portfolio engine — the real business logic

`src/services/portfolioApiService.ts` is the most substantive authored code (344 LOC):

**Symbol resolution** — a 3-tier fallback chain with a 5-minute in-memory price cache:
1. `/coins/markets?ids=<symbol>` (fast path)
2. `/coins/markets?per_page=250` then match by symbol
3. `/coins/list` full catalogue -> `/simple/price`

**Aggregation** — multiple buy entries of the same symbol collapse into one position:
```
average_buy_price  = sum(amount * avgPrice) / sum(amount)     <- weighted, correct
current_value      = total_quantity * live_price
profit_or_loss     = current_value - total_invested
P&L %              = (P&L / total_invested) * 100
```
Live prices are batch-fetched in a single `/simple/price?ids=a,b,c` call — a
genuine optimization. On failure it degrades to `0`, which silently displays a
**-100% loss** rather than an error state.

**Writes** delegate to `src/services/portfolioService.ts` via a lazy
`await import()` — dynamic-import code-splitting, visible in the build output as a
separate `portfolioService-*.js` chunk.

### 3.4 Data model

```sql
portfolio_holdings
  id            UUID PK
  user_id       TEXT      -- TEXT, not UUID: matches Clerk's "user_xxx" string IDs
  symbol, name, coin_id   TEXT
  amount, avg_price       NUMERIC
  purchase_date DATE
  notes         TEXT
  created_at / updated_at TIMESTAMPTZ  (trigger-maintained)

  RLS: 4 policies (SELECT/INSERT/UPDATE/DELETE), all gated on
       user_id = jwt.claims->>'sub'
  Indexes: user_id, symbol
```

Design note: the schema stores **individual purchase transactions**, not
positions. Aggregation happens in the client. This is why the UI shows
"N transactions" per asset and renders one delete button per entry.

---

## PART 4 — Feature Reality Check

The most important section. **Roughly half the advertised features are
non-functional facades.**

| Route | Status | Reality |
|---|---|---|
| `/` Landing | Real | 214-line marketing page, pure presentation |
| `/auth` | **Real** | Clerk `<SignIn/>`/`<SignUp/>` with custom dark `appearance` overrides |
| `/portfolio` | **Real** | Full CRUD, Supabase-persisted, live CoinGecko pricing, weighted-average cost basis. **The genuine core of the app.** |
| `/market` | Real | Top-100 table, global stats, trending — all live CoinGecko |
| `/dashboard` | **Mixed** | MarketStats/CryptoList/CryptoChart/TradingView/PortfolioCard are real; Fear&Greed is real; **SentimentAnalysis and MarketPulse are fabricated** |
| `/screener` | **Mostly fake** | 6 of 7 "VC" entries are hardcoded `MOCK_HOLDINGS`. Only the Alameda wallet row does a real Covalent call |
| `/chat` | **Fake** | No LLM. `generateAIResponse()` is a keyword `if/else` returning 6 canned paragraphs, wrapped in a `setTimeout(1500)` to simulate "thinking" |
| `/nft` | **Placeholder** | "Coming Soon" page; its 5 referenced images don't exist in `public/` |

### Specific fabrications

**Sentiment & Market Pulse** (`src/services/marketDataService.ts:55-90`) — pure `Math.random()`:
```ts
trading_volume: 75 + (Math.random() - 0.5) * 20
whale_activity: confidence + (Math.random() - 0.5) * 30
```
These re-roll every 30s / 5min, producing convincing-looking but meaningless
"on-chain metrics" and "whale activity."

**Portfolio Analytics** (`src/components/PortfolioAnalytics.tsx:35-43`) — the
"Portfolio Performance" chart is **not historical data**. It back-projects fixed
multipliers off the current value:
```ts
{ date: 'YTD', value: totalValue * 0.45 }   // always shows -55% YTD
```
So every user sees an identical-shaped curve. Likewise hardcoded:
"Best Performer +34.2%", "Worst -12.1%", "Avg. Hold Time 127 days",
"Risk Score: Medium". The 4 "Quick Actions" buttons (Rebalance / Add Funds /
Set Alerts / Analyze) have **no `onClick` handlers** — decorative only.

**PortfolioCard** (`src/components/PortfolioCard.tsx:29-40`) falls back to
invented figures ($41,550 value, BTC/ETH holdings) when real data is absent — so
a brand-new user's dashboard shows a fictional portfolio.

---

## PART 5 — Engineering Assessment

### Genuine strengths
- **Auth architecture is thoughtfully built** — the Clerk<->Supabase JWT bridge with manual expiry checking and proactive refresh is non-trivial and correctly implemented
- **RLS is properly configured** — security enforced at the database, not just the client
- **Portfolio math is correct** — weighted average cost basis, proper P&L
- **Sensible query hygiene** — differentiated `staleTime`/`refetchInterval` per data volatility, batch price fetching, dynamic imports, `useMemo`/`useCallback` in `Portfolio.tsx`
- **Clean, consistent visual design** — cohesive black/red/orange theme with custom CSS animations (`blob`, `scroll-infinite`, `glass-card`)

### Real weaknesses

| Issue | Detail |
|---|---|
| **Rate limiting** | 12 direct CoinGecko calls, no API key, aggressive polling. Free tier ~10-30 req/min — the dashboard alone will 429 |
| **Type safety disabled** | `strictNullChecks: false` + `noImplicitAny: false` + `no-unused-vars: off`. Typecheck "passing" means little |
| **No tests** | Zero test files, no CI |
| **Debug logging in prod** | ~25 `console.log` calls ship to production, including token-state logging |
| **Silent failure modes** | Price fetch failure -> `0` -> displays -100% loss rather than an error |
| **Dead code** | `src/pages/index.tsx` isn't routed; `zod`/`date-fns`/`@hookform/resolvers` declared but unimported |
| **Bundle size** | Single 1.28 MB chunk, no manual chunking |
| **Missing assets** | 5 NFT images referenced, none present |

---

## PART 6 — Current State (this session's changes)

| Area | State |
|---|---|
| Malware | Removed from both configs; verified zero remnants |
| `.gitignore` | Now covers `.env`, `*.env`, `.claude/`, `.agents/`, `skills-lock.json` |
| Secrets | `back.env` deleted; `.env` reduced to `VITE_CLERK_PUBLISHABLE_KEY` only |
| Clerk | Relinked to app `app_3IraXfhohQI3jDAODlHFjDpgl2X` ("Test App", instance `deep-buffalo-9291`) |
| Deps | 359 installed via `--ignore-scripts`; dropped dead `axios` + `socket.io-client`; fixed a malformed `"socket": {}` entry |
| Vite | Added `server.watch.ignored` for `.agents`/`.claude` to stop the chokidar `EBUSY` crash |
| Runtime | Dev server was live at http://localhost:8080 |

### Known-open items

1. **Create the `supabase` JWT template** in the new Clerk app, or `/portfolio`
   will 401 on every request — this is the single blocker to the app's one
   genuinely functional feature.
   - Clerk Dashboard -> "Test App" -> Configure -> JWT Templates -> New -> Supabase
   - Name must be exactly `supabase`
   - Signing key = Supabase project's JWT Secret (Settings -> API -> JWT Settings)
2. **Old data won't carry over** — the Supabase project keys rows by Clerk `sub`;
   a new Clerk instance means new user IDs, so existing `portfolio_holdings` /
   `profiles` rows won't match.
3. **4 npm advisories** (`esbuild <=0.24.2` via Vite; `react-router` 6.x
   open-redirect) — known framework CVEs, fixable only via breaking majors
   (`vite@8`, `react-router@7`).

---

## Bottom line

A **visually accomplished, architecturally sound-in-parts, but substantially
hollow** application. The Clerk->Supabase auth bridge and the portfolio tracking
engine are real, correct, and non-trivial engineering. Everything branded "AI" —
the chat assistant, sentiment analysis, market pulse, predictive analytics — is
simulated with `Math.random()` and hardcoded strings. The honest description is
**"a crypto portfolio tracker with a market dashboard,"** not an AI trading
platform.

---

## Appendix — Original malware details (now removed)

Two build-executed config files carried an obfuscated payload appended after a
large whitespace gap so it sat off-screen:

- `vite.config.ts` — was line 23, after `defineConfig(...)`
- `postcss.config.js` — was line 8

Pattern: a `createRequire` shim assigning `require`/`module`/`__dirname`/
`__filename` onto `global`, followed by a self-invoking obfuscated IIFE with
string-deshuffling helpers (`_$_a3f0`, `eEa`, `_$jsoToArr`). Because Node executes
these files on every `vite` / `npm run dev` / `npm run build`, the payload would
run at build time — a supply-chain / credential-stealer pattern targeting crypto
repos. Both files were restored to their minimal legitimate contents and
re-verified clean.

Also found and handled: a committed `CLERK_SECRET_KEY` in `.env` and a duplicate
`back.env`, neither covered by `.gitignore`.
