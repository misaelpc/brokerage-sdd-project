# brokerage-web — Build Specification

> **Purpose:** Phased implementation guide for the simulated trading frontend **brokerage-web**, built with **Vite + React + TypeScript**, consuming **brokerage-core-api** GraphQL, with **Vitest + React Testing Library** and **GitHub Actions CI**.
>
> **Sources:** `BUILD_SPEC_BE.md`, `brokerage-core-api/src/main/resources/graphql/schema.graphqls`, Stockify Web design/plan (reference patterns)
>
> **Target stack:** asdf (`.tool-versions`) · Node 20 · Vite 6 · React 19 · TypeScript 5 · Apollo Client 3 · Tailwind CSS 3 · Vitest · ESLint · Prettier

---

## Table of Contents

1. [Overview](#1-overview)
   - [1.0 Agent instructions (mandatory)](#10-agent-instructions-mandatory)
   - [1.5 Repository layout](#15-repository-layout)
   - [1.6 Dev constraints](#16-dev-constraints)
2. [Phase 1 — Vite Project Creation](#2-phase-1--vite-project-creation)
3. [Phase 2 — Read-Only Dashboard (v1 Queries)](#3-phase-2--read-only-dashboard-v1-queries)
4. [Phase 3 — Trading UI (Mutations)](#4-phase-3--trading-ui-mutations)
5. [Phase 4 — Orders, Activity & Cash Management](#5-phase-4--orders-activity--cash-management)
6. [Phase 5 — Testing & CI](#6-phase-5--testing--ci)
7. [Delivery Milestones](#7-delivery-milestones)
8. [Appendix A — GraphQL Operations Used by the UI](#appendix-a--graphql-operations-used-by-the-ui)
9. [Appendix B — Demo Data Alignment](#appendix-b--demo-data-alignment)
10. [Appendix C — UI ↔ Backend Traceability](#appendix-c--ui--backend-traceability)

---

## 1. Overview

### 1.0 Agent instructions (mandatory)

When implementing **any phase** of this build specification, the agent **must**:

1. Read this document and [`BUILD_SPEC_BE.md`](./BUILD_SPEC_BE.md) at the start of each session.
2. Scaffold and implement all application code under **`brokerage-web/`** (see [§1.5](#15-repository-layout)). Do not place Vite/npm artifacts at the repository root.
3. Target the Apollo Client URI as the **relative path** `/graphql` — never hardcode `http://localhost:8080`. Vite's dev-server proxy bridges to the backend.
4. Use **asdf** with the committed **`brokerage-web/.tool-versions`** for Node.js — run `asdf install` before `npm install`; verify `node -v` matches the pinned version.
5. Format all money values with the shared `mxn()` helper in `src/format.ts`; never format currency inline.
6. Every data-fetching component renders **loading** and **error** states.
7. Follow the 3-gate workflow when using AI assistance: understand spec → propose plan (Gate 1) → implement → present diff for developer commit (Gate 2) → verify with tests + manual check (Gate 3).

**Conflict resolution:** If this document and `BUILD_SPEC_BE.md` disagree on *GraphQL contract or business rules*, follow the backend schema and BE spec. If they disagree on *frontend scope, phases, or deliverables*, follow `BUILD_SPEC_FE.md`.

### 1.1 Product goal

**brokerage-web** is a simulated stock-trading dashboard for the **brokerage-core-api** Quarkus backend. Clients interact via **GraphQL over HTTP** (proxied in dev). The UI supports:

| Slice | Phase | Capability |
|-------|-------|------------|
| **Account** | 2 | Demo investor balance (available + reserved cash) |
| **Catalog** | 2 | Active instruments with last simulated price |
| **Portfolio** | 2 | Positions with market value; empty-state when no holdings |
| **Trading** | 3 | Place market/limit buy & sell orders; cancel pending orders; execute pending orders |
| **Orders** | 4 | List open and historical orders for the demo investor |
| **Activity** | 4 | Activity feed (deposits, trades, cancellations) |
| **Cash** | 4 | Deposit cash mutation (demo flow) |

Trading is **simulated** — the UI reflects backend state only; no real market execution.

**Demo persona:** Andrea López (`andrea.lopez@example.com`), seeded with **100,000.00 MXN** available cash and an empty portfolio.

### 1.2 Architectural principles

| Principle | Requirement |
|-----------|-------------|
| Feature ownership | One UI slice = one component (or small folder) that owns its GraphQL query/mutation |
| Backend isolation | No business rules duplicated in the UI — display backend data and surface GraphQL errors |
| Proxy, not CORS | Dev server proxies `/graphql` → `http://localhost:8080`; no backend CORS changes |
| Typed GraphQL | Query/mutation documents and TypeScript types live in `src/graphql/` |
| Null safety | Handle missing data and GraphQL errors explicitly; no silent failures |
| Currency | MXN formatting via shared helper; respect `MoneyAmount.currency` when present |
| Progressive delivery | Phase 2 is read-only (vertical slice); mutations added in Phases 3–4 |

### 1.3 UI slices ↔ backend bounded contexts

| UI slice | Backend context | Primary GraphQL operations |
|----------|-----------------|---------------------------|
| Account | Cash Account | `balance`, `cashAccount` |
| Catalog | Market Catalog + Market Data | `catalog`, `instrument`, `instrumentBySymbol` |
| Portfolio | Portfolio | `portfolio` |
| Trading | Trading (Core) | `placeBuyOrder`, `placeSellOrder`, `cancelOrder`, `executePendingOrders` |
| Orders | Trading | `orders` |
| Activity | Activity Feed | `activityFeed` |
| Registration | Investor Management | `registerInvestor` (optional admin/demo flow) |

### 1.4 Data flow

```
Browser (:5173)
  → React components (useQuery / useMutation per slice)
    → Apollo Client (uri: "/graphql")
      → Vite dev-server proxy  /graphql → http://localhost:8080/graphql
        → brokerage-core-api (Quarkus + SmallRye GraphQL)
```

- Single-page dashboard in Phase 2; optional tabbed sections in Phase 4 for Orders / Activity.
- `investorId` for mutations comes from the `balance` query response (`investorId` field) — store in React context after first load.

### 1.5 Repository layout

The repository root holds **specification and agent guidance** (`BUILD_SPEC_BE.md`, `BUILD_SPEC_FE.md`, `ddd_skill.md`, etc.). The runnable Vite application **must** live in a dedicated subdirectory:

```text
<repository-root>/
├── BUILD_SPEC_BE.md
├── BUILD_SPEC_FE.md          ← this document
├── ddd_skill.md
├── .github/workflows/        ← CI may include brokerage-web job
├── brokerage-core-api/       ← Quarkus backend (application root)
└── brokerage-web/            ← Vite/React frontend (application root)
    ├── .tool-versions        ← asdf: pinned Node.js (and optional npm)
    ├── package.json
    ├── vite.config.ts
    ├── index.html
    ├── src/
    │   ├── main.tsx
    │   ├── apollo.ts
    │   ├── App.tsx
    │   ├── format.ts
    │   ├── context/
    │   │   └── DemoInvestorContext.tsx
    │   ├── graphql/
    │   │   ├── queries.ts
    │   │   └── mutations.ts
    │   └── components/
    │       ├── BalanceCard.tsx
    │       ├── Catalog.tsx
    │       ├── Portfolio.tsx
    │       ├── TradePanel.tsx
    │       ├── OrdersList.tsx
    │       ├── ActivityFeed.tsx
    │       └── DepositForm.tsx
    └── …
```

| Term | Path | Contents |
|------|------|----------|
| **Repository root** | `.` | Build specs, design docs, monorepo CI |
| **Backend application root** | `brokerage-core-api/` | Quarkus/Maven source, Docker Compose, tests |
| **Frontend application root** | `brokerage-web/` | Vite/npm source, frontend tests, governance |

**Do not** generate `package.json`, `vite.config.ts`, or `src/` at the repository root.

### 1.6 Dev constraints

Local setup rules for agents and developers. Resolve mismatches before `npm run dev`.

| Area | Constraint |
|------|------------|
| **Runtime (asdf)** | Node.js is pinned in **`brokerage-web/.tool-versions`**. The environment uses **asdf** (already installed on dev machines). From `brokerage-web/`: `asdf install` then verify `node -v` and `which node` point at asdf shims — not a stale system Node. |
| **Node** | Node.js **≥ 20** (LTS). The exact patch is declared in `.tool-versions`; keep it aligned with the CI `node-version` (major **20**). |
| **Backend first** | From `brokerage-core-api/`: `docker compose up -d` → `./mvnw quarkus:dev -Djacoco.skip=true`. Backend must respond at `http://localhost:8080/q/health` and GraphiQL at `/graphiql`. |
| **Frontend** | From `brokerage-web/`: `asdf install` → `npm install` → `npm run dev` on port **5173**. |
| **Proxy** | `vite.config.ts` must proxy `/graphql` → `http://localhost:8080` (Quarkus root path is `/graphql`). |
| **Order type** | Mutation argument `type` is a **String** (`"MARKET"`, `"LIMIT"`) — not a GraphQL enum. |
| **BigDecimal** | GraphQL returns decimals as strings or numbers depending on serializer; normalize to `number` in mappers if needed. |
| **Smoke test** | Backend up → frontend dev server → dashboard shows Andrea López, catalog (5 instruments), empty portfolio. |

**Two-terminal workflow:**

```bash
# Terminal 1 — backend
cd brokerage-core-api
docker compose up -d
./mvnw quarkus:dev -Djacoco.skip=true

# Terminal 2 — frontend
cd brokerage-web
asdf install    # reads .tool-versions; no-op if already installed
npm install
npm run dev
```

Open `http://localhost:5173`.

---

## 2. Phase 1 — Vite Project Creation

### 2.1 Generate the project

From the **repository root**, scaffold the Vite + React + TypeScript app inside **`brokerage-web/`**:

```bash
npm create vite@latest brokerage-web -- --template react-ts
cd brokerage-web
asdf install
npm install
```

### 2.2 Runtime versions — `.tool-versions` (asdf)

Pin the Node.js runtime in **`brokerage-web/.tool-versions`** so every developer and agent uses the same toolchain via asdf (mirrors how the backend pins JDK in `brokerage-core-api/.sdkmanrc`).

**Create `brokerage-web/.tool-versions`:**

```text
nodejs 20.18.0
```

| Rule | Detail |
|------|--------|
| **Plugin** | asdf **nodejs** plugin (`asdf plugin add nodejs` if missing). |
| **Install** | Run `asdf install` from `brokerage-web/` after cloning or when `.tool-versions` changes. |
| **Verify** | `node -v` → `v20.18.0` (or whatever is pinned); `which node` → `~/.asdf/shims/node`. |
| **CI parity** | GitHub Actions uses `node-version: '20'` — bump `.tool-versions` and CI together when upgrading Node. |
| **Commit** | `.tool-versions` is **source-controlled**; do not gitignore it. |

Optional: add an `engines` field to `package.json` as a secondary guard:

```json
"engines": {
  "node": ">=20"
}
```

**Core dependencies to add:**

| Package | Purpose |
|---------|---------|
| `@apollo/client` | GraphQL client |
| `graphql` | GraphQL core |

**Dev dependencies to add:**

| Package | Purpose |
|---------|---------|
| `tailwindcss`, `postcss`, `autoprefixer` | Styling |
| `vitest`, `jsdom`, `@testing-library/react`, `@testing-library/jest-dom` | Testing |
| `eslint`, `@eslint/js`, `@typescript-eslint/*`, `eslint-plugin-react-hooks`, `eslint-config-prettier` | Linting |
| `prettier` | Formatting |

### 2.3 `package.json` scripts

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint .",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

### 2.4 Vite configuration

**`brokerage-web/vite.config.ts`:**

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/graphql': 'http://localhost:8080',
    },
  },
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './vitest.setup.ts',
  },
});
```

### 2.5 TypeScript configuration

- `strict: true`, `noUnusedLocals`, `noUnusedParameters`
- Include Vitest globals and `@testing-library/jest-dom` types
- Separate `tsconfig.node.json` for Vite config files

### 2.6 Tailwind CSS

- `tailwind.config.js` — content: `['./index.html', './src/**/*.{ts,tsx}']`
- `postcss.config.js` — tailwindcss + autoprefixer
- `src/index.css` — `@tailwind base/components/utilities`; import first in `main.tsx`

### 2.7 ESLint + Prettier

- Flat ESLint config (`eslint.config.js`) with TypeScript + React Hooks rules
- `.prettierrc.json`: single quotes, semicolons, trailing commas, printWidth 100
- `.prettierignore`: `dist`, `node_modules`, `package-lock.json`

### 2.8 Apollo client bootstrap

**`src/apollo.ts`:**

```ts
import { ApolloClient, InMemoryCache } from '@apollo/client';

export const client = new ApolloClient({
  uri: '/graphql',
  cache: new InMemoryCache(),
});
```

Wrap `<App />` in `<ApolloProvider client={client}>` in `main.tsx`.

### 2.9 Project skeleton directories

```text
brokerage-web/
├── .tool-versions
├── index.html
├── package.json
├── vite.config.ts
├── vitest.setup.ts
├── tailwind.config.js
├── postcss.config.js
├── eslint.config.js
├── .prettierrc.json
├── .gitignore
└── src/
    ├── main.tsx
    ├── apollo.ts
    ├── App.tsx
    ├── index.css
    ├── format.ts
    ├── vite-env.d.ts
    ├── graphql/
    │   ├── queries.ts
    │   └── mutations.ts
    ├── context/
    │   └── DemoInvestorContext.tsx
    └── components/
        └── (filled in Phases 2–4)
```

### 2.10 `.gitignore` essentials

```text
node_modules/
dist/
*.local
.DS_Store
.env
```

### 2.11 Phase 1 exit criteria

- [ ] `brokerage-web/.tool-versions` committed; `asdf install` resolves Node **20.x**
- [ ] `node -v` matches pinned version; `which node` uses asdf shims
- [ ] From `brokerage-web/`: `npm install` succeeds
- [ ] `npm run dev` starts on port 5173
- [ ] `npm test` runs (smoke test passes)
- [ ] `npm run build` produces `dist/`
- [ ] `npm run lint` and `npm run format:check` pass
- [ ] Apollo provider wired; placeholder `App` renders

---

## 3. Phase 2 — Read-Only Dashboard (v1 Queries)

> **Backend alignment:** Implements the v1 vertical slice from `BUILD_SPEC_BE.md` §3.7 — `balance`, `catalog`, `portfolio`.

### 3.1 GraphQL query documents

**`src/graphql/queries.ts`** — types and documents aligned with the **actual** backend schema (not Stockify's simplified shape):

```graphql
# Balance (v1 demo query)
query Balance {
  balance {
    investorId
    investorName
    available { amount currency }
    reserved { amount currency }
  }
}

# Catalog (v1 — active instruments with last price)
query Catalog {
  catalog {
    id
    symbol
    name
    type
    market
    status
    lastPrice
    currency
  }
}

# Portfolio (v1 demo query)
query Portfolio {
  portfolio {
    investorId
    totalMarketValue
    currency
    positions {
      instrumentId
      symbol
      quantity
      marketPrice
      marketValue
      currency
    }
  }
}
```

**TypeScript types (example):**

```ts
export type MoneyAmount = { amount: number; currency: string };

export type Balance = {
  investorId: string;
  investorName: string;
  available: MoneyAmount;
  reserved: MoneyAmount;
};

export type Instrument = {
  id: string;
  symbol: string;
  name: string;
  type: string;
  market: string;
  status: string;
  lastPrice: number | null;
  currency: string | null;
};

export type Position = {
  instrumentId: string;
  symbol: string;
  quantity: number;
  marketPrice: number;
  marketValue: number;
  currency: string;
};

export type Portfolio = {
  investorId: string;
  positions: Position[];
  totalMarketValue: number;
  currency: string;
};
```

### 3.2 MXN formatter

**`src/format.ts`:**

```ts
const formatter = new Intl.NumberFormat('en-US', {
  style: 'currency',
  currency: 'MXN',
  currencyDisplay: 'narrowSymbol',
});

export function mxn(value: number): string {
  return formatter.format(value);
}
```

Unit tests in `src/format.test.ts` — at minimum: `100000` → `$100,000.00`, zero case.

### 3.3 Demo investor context

**`src/context/DemoInvestorContext.tsx`**

- On mount, run `BALANCE` query (or receive `investorId` from `BalanceCard`)
- Expose `{ investorId, investorName, loading, error }` via React context
- Phases 3–4 mutations consume `investorId` from this context

### 3.4 Components

| Component | Query | Behavior |
|-----------|-------|----------|
| **BalanceCard** | `BALANCE` | Shows investor name, available cash (primary), reserved cash (secondary label) |
| **Catalog** | `CATALOG` | Table: symbol, name, type, market, last price (MXN). **No Buy button in Phase 2.** |
| **Portfolio** | `PORTFOLIO` | Table: symbol, quantity, market value. Empty state: **"No holdings yet"**. Shows `totalMarketValue` when positions exist. **No Sell button in Phase 2.** |

Each component:

- Owns its `useQuery` call
- Renders loading skeleton/text and error message
- Uses `mxn()` for all monetary display

### 3.5 Dashboard layout

**`src/App.tsx`:**

```tsx
// Layout: header "Brokerage" → BalanceCard → 2-column grid (Catalog | Portfolio)
// Wrap in DemoInvestorProvider
// min-h-screen bg-gray-100, max-w-4xl centered, Tailwind card styling
```

### 3.6 Tests (Phase 2 minimum)

| Test | Asserts |
|------|---------|
| `format.test.ts` | MXN formatting |
| `BalanceCard.test.tsx` | Mocked `balance` → "Andrea López" + formatted available cash |
| `Catalog.test.tsx` | Mocked `catalog` → symbols and prices render |
| `Portfolio.test.tsx` | Empty `positions: []` → "No holdings yet" |

Use `@apollo/client/testing` `MockedProvider` for component tests.

### 3.7 Phase 2 exit criteria

- [ ] Dashboard loads against running backend without CORS errors
- [ ] Balance shows Andrea López + **100,000.00 MXN** available (seed data)
- [ ] Catalog lists 5 instruments (AAPL, TSLA, NVDA, MSFT, IVVPESO) with prices
- [ ] Portfolio shows empty state initially
- [ ] All Phase 2 unit tests pass
- [ ] `npm run build` succeeds

---

## 4. Phase 3 — Trading UI (Mutations)

> **Backend alignment:** `BUILD_SPEC_BE.md` §3.4 commands — place buy/sell, cancel, execute pending.

### 4.1 GraphQL mutation documents

**`src/graphql/mutations.ts`:**

```graphql
mutation PlaceBuyOrder(
  $investorId: ID!
  $instrumentId: ID!
  $type: String!
  $quantity: BigDecimal!
  $limitPrice: BigDecimal
  $currency: String!
) {
  placeBuyOrder(
    investorId: $investorId
    instrumentId: $instrumentId
    type: $type
    quantity: $quantity
    limitPrice: $limitPrice
    currency: $currency
  ) {
    id
    status
    side
    type
    quantity
    symbol
  }
}

mutation PlaceSellOrder(/* same args */) { ... }

mutation CancelOrder($orderId: ID!) {
  cancelOrder(orderId: $orderId) {
    id
    status
  }
}

mutation ExecutePendingOrders($investorId: ID!) {
  executePendingOrders(investorId: $investorId) {
    id
    status
    side
    symbol
    executedPrice
  }
}
```

### 4.2 TradePanel component

**`src/components/TradePanel.tsx`**

| Control | Mutation | Notes |
|---------|----------|-------|
| Buy form | `placeBuyOrder` | Fields: instrument (select from catalog), type (MARKET/LIMIT), quantity, limit price (conditional) |
| Sell form | `placeSellOrder` | Same fields; validate quantity ≤ portfolio position |
| Execute button | `executePendingOrders` | Triggers backend execution policy for pending orders |
| Cancel (per order) | `cancelOrder` | Wired in Phase 4 OrdersList; optional inline in Phase 3 |

**UX rules:**

- Default currency: `MXN`
- LIMIT orders: show `limitPrice` field; hide for MARKET
- On success: refetch `balance`, `portfolio`, and `orders` (when available) via Apollo `refetchQueries` or cache eviction
- On `BAD_REQUEST` GraphQL error: show user-friendly message (insufficient cash, insufficient position, invalid state)
- Disable trade controls while mutation in flight

### 4.3 Catalog integration

- Add **Buy** action per catalog row (or select instrument in TradePanel)
- Pass `instrumentId` from catalog row to buy form

### 4.4 Portfolio integration

- Add **Sell** action per position row
- Pre-fill instrument and max quantity from position

### 4.5 Tests (Phase 3)

| Test | Asserts |
|------|---------|
| `TradePanel.test.tsx` | Buy form submits with MARKET type; mock mutation success |
| `TradePanel.test.tsx` | LIMIT form requires limit price field |
| `TradePanel.test.tsx` | GraphQL error surfaces as user-visible message |

### 4.6 Phase 3 exit criteria

- [ ] Place market buy order for AAPL → order accepted; reserved cash increases
- [ ] Execute pending orders → portfolio shows AAPL position; available cash decreases
- [ ] Place sell order + execute → position reduced; cash credited
- [ ] Cancel pending order → reserved cash released
- [ ] UI refetches balance/portfolio after each mutation
- [ ] Phase 3 tests pass

**Manual demo script (mirrors BE §6):**

1. Query dashboard — empty portfolio, 100k balance
2. Buy AAPL market order (qty 1)
3. Click "Execute pending orders"
4. Portfolio shows AAPL; balance updated
5. Sell AAPL + execute — back to empty portfolio with updated cash

---

## 5. Phase 4 — Orders, Activity & Cash Management

### 5.1 Additional queries

```graphql
query Orders($investorId: ID!) {
  orders(investorId: $investorId) {
    id
    symbol
    side
    type
    status
    quantity
    limitPrice
    executedPrice
    placedAt
  }
}

query ActivityFeed($investorId: ID!) {
  activityFeed(investorId: $investorId) {
    id
    activityType
    description
    amount
    currency
    occurredAt
  }
}
```

### 5.2 Deposit mutation

```graphql
mutation DepositCash($investorId: ID!, $amount: BigDecimal!, $currency: String!) {
  depositCash(investorId: $investorId, amount: $amount, currency: $currency) {
    investorId
    investorName
    available { amount currency }
    reserved { amount currency }
  }
}
```

### 5.3 Components

| Component | Operations | Behavior |
|-----------|------------|----------|
| **OrdersList** | `orders`, `cancelOrder` | Table: symbol, side, type, status, qty, placedAt. Cancel button for PENDING/ACCEPTED orders |
| **ActivityFeed** | `activityFeed` | Chronological list: type, description, amount, timestamp |
| **DepositForm** | `depositCash` | Amount input + submit; refetch balance on success |

### 5.4 Layout update

- Tabbed or stacked sections below the Phase 2 grid: **Orders** | **Activity** | **Deposit**
- Optional: collapsible panels on mobile

### 5.5 Tests (Phase 4)

| Test | Asserts |
|------|---------|
| `OrdersList.test.tsx` | Renders mocked orders; cancel button visible for pending |
| `ActivityFeed.test.tsx` | Renders activity items with descriptions |
| `DepositForm.test.tsx` | Successful deposit refetches balance |

### 5.6 Phase 4 exit criteria

- [ ] Orders list reflects placed/executed/cancelled orders
- [ ] Activity feed shows deposits and trade events
- [ ] Deposit increases available balance
- [ ] Cancel from OrdersList works and updates balance
- [ ] All component tests pass

---

## 6. Phase 5 — Testing & CI

### 6.1 Test pyramid

```
        ┌─────────────┐
        │  Manual E2E │  ~5%  — full demo script against live backend
        ├─────────────┤
        │  Component  │  ~60% — MockedProvider tests per slice
        ├─────────────┤
        │  Unit       │  ~35% — formatters, mappers, pure helpers
        └─────────────┘
```

### 6.2 Coverage target

| Metric | Target | Enforced by |
|--------|--------|-------------|
| **Vitest** | All tests pass | `npm test` in CI |
| **Typecheck** | Zero TS errors | `npm run build` |
| **Lint** | Zero ESLint errors | `npm run lint` |

Frontend does not gate on line-coverage percentage initially; aim for meaningful behavior tests on each slice.

### 6.3 CI workflow

Add job to **`.github/workflows/ci.yml`** (or `brokerage-web-ci.yml`):

```yaml
  frontend:
    name: Frontend Build & Test
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: brokerage-web
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm
          cache-dependency-path: brokerage-web/package-lock.json
      - run: npm ci
      - run: npm run format:check
      - run: npm run lint
      - run: npm test
      - run: npm run build
```

Backend integration tests remain in the `brokerage-core-api` job; frontend CI does **not** require a running backend (all tests use mocks).

**Node version:** CI uses `node-version: '20'` to match the major in `brokerage-web/.tool-versions`. When bumping the pinned patch (e.g. `20.18.0` → `20.19.0`), update `.tool-versions` only; CI stays on major `20` unless you intentionally upgrade the whole stack.

### 6.4 Governance (optional)

Mirror Stockify's `.claude/` pattern inside `brokerage-web/`:

- `CLAUDE.md` — workflow + commands
- `.claude/rules/workflow.md` — 3-gate flow
- `.claude/rules/conventions.md` — Apollo, Tailwind, component conventions
- Stop hooks: `prettier` + `eslint` + `vitest`

### 6.5 Phase 5 exit criteria

- [ ] CI runs frontend job on every PR
- [ ] `npm test`, `npm run lint`, `npm run build` pass in CI
- [ ] README in `brokerage-web/` documents local dev steps (asdf + `.tool-versions`, backend proxy, two-terminal workflow)
- [ ] Manual demo script (§4.6 + orders/activity) verified end-to-end

---

## 7. Delivery Milestones

| Milestone | Phases | Deliverable | BE milestone alignment |
|-----------|--------|-------------|------------------------|
| **M1 — Bootstrap** | Phase 1 | Runnable Vite app, Apollo, Tailwind, toolchain | BE M1 |
| **M2 — Read dashboard** | Phase 2 | Balance + catalog + portfolio (read-only) | BE M2 |
| **M3 — Trading** | Phase 3 | Buy, sell, execute, cancel with live refetch | BE M4 |
| **M4 — Full demo** | Phase 4 | Orders, activity, deposit | BE M5 |
| **M5 — CI-ready** | Phase 5 | GitHub Actions green, README, governance | BE M5 |

### Demo script (acceptance)

**Prerequisites:** Backend running (`docker compose up -d`, `./mvnw quarkus:dev`), frontend `npm run dev`.

1. Open `http://localhost:5173` — Andrea López, $100,000.00 available, 5 instruments, empty portfolio
2. Buy 1 AAPL market order → Execute pending orders → portfolio shows AAPL
3. Sell 1 AAPL → Execute → portfolio empty, cash updated
4. Deposit $10,000 → balance increases; activity shows deposit
5. Orders tab shows order history; Activity tab shows events

---

## Appendix A — GraphQL Operations Used by the UI

Full schema: `brokerage-core-api/src/main/resources/graphql/schema.graphqls`.

### Queries (by phase)

| Query | Phase | Used by |
|-------|-------|---------|
| `balance` | 2 | BalanceCard, DemoInvestorContext |
| `catalog` | 2 | Catalog, TradePanel (instrument picker) |
| `portfolio` | 2 | Portfolio, TradePanel (sell validation) |
| `orders` | 4 | OrdersList |
| `activityFeed` | 4 | ActivityFeed |

### Mutations (by phase)

| Mutation | Phase | Used by |
|----------|-------|---------|
| `placeBuyOrder` | 3 | TradePanel |
| `placeSellOrder` | 3 | TradePanel |
| `executePendingOrders` | 3 | TradePanel |
| `cancelOrder` | 3–4 | TradePanel, OrdersList |
| `depositCash` | 4 | DepositForm |

### Error contract (display mapping)

| GraphQL classification | UI behavior |
|------------------------|-------------|
| `NOT_FOUND` | "Resource not found" + optional retry |
| `BAD_REQUEST` | Show backend message (business rule violation) |
| Network / 500 | "Unable to reach server" + check backend health |

### Stockify schema differences (do not copy blindly)

| Stockify field | brokerage-core-api field |
|----------------|--------------------------|
| `balance.ownerName` | `balance.investorName` |
| `balance.cash` | `balance.available.amount` (+ `reserved`) |
| `catalog.price` | `catalog.lastPrice` |
| `portfolio` (flat array) | `portfolio.positions[]` + `totalMarketValue` |

---

## Appendix B — Demo Data Alignment

From `BUILD_SPEC_BE.md` Appendix B — UI expectations when backend seed is loaded:

### Investor

| Field | Expected value |
|-------|----------------|
| Name | Andrea López |
| Email | andrea.lopez@example.com |
| Available cash | 100,000.00 MXN |
| Reserved cash | 0.00 MXN (initially) |
| Portfolio | Empty (initially) |

### Instruments (catalog)

| Symbol | Name | Price (MXN) |
|--------|------|-------------|
| AAPL | Apple Inc. | 3,200.00 |
| TSLA | Tesla Inc. | 4,500.00 |
| NVDA | NVIDIA Corp. | 6,100.00 |
| MSFT | Microsoft Corp. | 7,200.00 |
| IVVPESO | ETF S&P 500 MXN | 95.00 |

---

## Appendix C — UI ↔ Backend Traceability

| # | Business rule (BE) | UI validation |
|---|-------------------|---------------|
| 1 | Only active investors can trade | Backend rejects; UI shows error on mutation |
| 2 | Only active instruments tradable | Catalog shows status; backend rejects inactive |
| 3 | Cannot buy without sufficient cash | Error message on placeBuyOrder |
| 4 | Cannot sell without sufficient position | Sell form max qty; backend rejects overflow |
| 5 | Executed orders cannot cancel | Cancel hidden/disabled for EXECUTED status |
| 6 | LIMIT orders require limit price | TradePanel validates before submit |
| 7 | MARKET orders have no limit price | Limit field hidden when type = MARKET |
| 8 | Execute pending orders batch | Execute button triggers `executePendingOrders` |
| 9 | Activity feed reflects trades/deposits | ActivityFeed refetches after mutations |
| 10 | Reserved cash visible | BalanceCard shows `reserved` amount |

---

*Document version: 1.0 — brokerage-web build specification*
