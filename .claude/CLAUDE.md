# DataCat

@~/.claude/CLAUDE.md

---

## Overview

**DataCat** is a universal data ingestion platform with AI analysis. Custom forms capture any data type, AI processes it, and actions are delivered to humans or machines.

**Workflow**: Data Ingestion → AI Analysis → Action Delivery

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | Next.js 15.3, React 19, TypeScript, Tailwind, Zustand |
| Backend | Node.js, Express.js, tRPC |
| Database | PostgreSQL (Prisma ORM), Redis |
| AI | Multi-LLM (OpenAI, Claude, custom) |
| Testing | Playwright |

---

## Project Structure (Monorepo)

```
datacat/
├── frontend/            # Next.js 15 (port 3000)
│   ├── src/app/        # App Router pages
│   ├── src/components/ # React components
│   └── src/stores/     # Zustand state
├── backend/             # Express.js (port 5001) — NO src/ dir; flat layout
│   ├── index.js        # Entrypoint
│   ├── routes/         # API routes
│   ├── controllers/    # Request handlers
│   ├── services/       # Business logic (incl. image/video/audio ingestion)
│   ├── trpc/           # tRPC routers
│   ├── prisma/         # Database schema
│   ├── jobs/ middleware/ lib/ utils/ config/ db/
│   └── (backend has ZERO tests — "test" script exits 1; change with care)
└── docker-compose.yml   # Infrastructure
```

---

## Quick Start

```bash
# Start both servers
npm run dev
# Frontend: http://localhost:3000
# Backend: http://localhost:5001

# Or individually
npm run dev:frontend
npm run dev:backend

# Docker
npm run docker:dev

# Verify a change before declaring it done (mirrors CI: lint + typecheck + build)
npm run verify
```

**Before declaring any change done, run `npm run verify`.** It runs the same
hermetic gates as CI (`.github/workflows/ci.yml`: frontend lint + typecheck +
build), so green locally means green on `main`.

The build is in the gate deliberately. It used to be deferred, and that gap has
a receipt: contentlayer 0.3.1 reaches into a React internal that React 19
removed, so `/blog/[slug]` threw `d.getOwner is not a function` during prerender
and production sat on a stale build for weeks. Lint and typecheck were green the
whole time — only a build could have caught it.

Note: the build needs Node 18 (`.nvmrc`); contentlayer crashes on exit under
Node 20+. The full-stack Playwright e2e is still not gated — it needs both
servers plus a seeded DB, so run it manually until that is wired.

> **Prerequisite:** `verify`'s typecheck reads `frontend/.contentlayer/generated`
> (git-ignored, produced by `frontend`'s `postinstall` → `contentlayer build`).
> CI regenerates it via `npm ci`. Locally, if `verify` fails with
> `TS2307: Cannot find module '../../../.contentlayer/generated'` for the blog
> pages, you skipped that step — run `cd frontend && npx contentlayer build`
> once (it prints a harmless `ERR_INVALID_ARG_TYPE` on exit under Node 20+ but
> still generates the types), then re-run `verify`.

---

## Critical: Monorepo Rules

- `frontend/` - React components, pages, Zustand state
- `backend/` - API routes, database, business logic
- Root `package.json` - orchestration scripts only
- **Never mix** frontend/backend code

---

## Rebranding System

DataCat supports white-labeling via environment variables:

```bash
# Apply preset
./scripts/dev/rebrand.sh medical

# Custom brand
./scripts/dev/rebrand.sh custom "MyApp" "Data Capture"
```

**Presets**: `datacat`, `hr`, `medical`, `legal`, `government`, `generic`

---

## Environment Variables

**Frontend** (`frontend/.env.local`):
```bash
NEXT_PUBLIC_API_URL=http://localhost:5001
NEXT_PUBLIC_BRAND_NAME=DataCat
```

**Backend** (`backend/.env`):
```bash
DATABASE_URL=postgresql://...
REDIS_URL=redis://localhost:6379
OPENAI_API_KEY=...
```

---

## Design System

**File**: `frontend/src/app/globals.css` — SSOT for all design tokens.
**Tailwind config**: `frontend/tailwind.config.ts`
**UI library**: No component library (no shadcn, no Radix UI). Plain Tailwind + custom CSS utilities.

### CSS Custom Properties (from `globals.css`)

```css
:root {
  --background: #ffffff;   /* light mode */
  --foreground: #171717;   /* light mode */
}

@media (prefers-color-scheme: dark) {
  :root {
    --background: #0a0a0a;
    --foreground: #ededed;
  }
}
```

Only two semantic tokens are currently defined. `body` also uses `var(--font-geist-sans)` (set via Next.js font loader, not a CSS var defined in this file).

### Tailwind Config (from `tailwind.config.ts`)

The two CSS vars are correctly mapped:
```ts
colors: {
  background: 'var(--background)',
  foreground: 'var(--foreground)',
},
fontFamily: {
  sans: ['var(--font-geist-sans)', 'system-ui', 'sans-serif'],
  mono: ['var(--font-geist-mono)', 'monospace'],
},
```
Tailwind config is clean — no literal hex values.

### Known Violations in `globals.css` (fix when touching UI)

Several utility classes in `globals.css` contain hardcoded hex values that should become CSS vars:

| Class | Hardcoded value | Should become |
|-------|----------------|---------------|
| `.focus-enhanced:focus-visible` | `#6366f1` (outline) | `--color-focus-ring` |
| `.gradient-mesh` | `#667eea`, `#764ba2` | `--color-gradient-start`, `--color-gradient-end` |
| `.gradient-mesh-alt` | `#f093fb`, `#f5576c` | `--color-gradient-alt-start`, `--color-gradient-alt-end` |
| `.skeleton` (light) | `#f0f0f0`, `#e0e0e0` | `--color-skeleton-base`, `--color-skeleton-shine` |
| `.skeleton` (dark media) | `#374151`, `#4b5563` | `--color-skeleton-base-dark`, `--color-skeleton-shine-dark` |

### SSOT Rule

All design tokens live in the main CSS file only. Tailwind config MUST reference CSS vars (`'var(--name)'`), never literal values. Components MUST use semantic Tailwind classes, never arbitrary values like `bg-[#hex]`.

**Violations to fix when touching UI:**
- `bg-[#hex]` / `text-[#hex]` in className → CSS var + semantic class
- `style={{ color: '#hex' }}` → CSS var + className
- Literal hex in tailwind.config → `'var(--color-name)'`
- Same token defined in 2+ files → consolidate to main CSS file

**Audit:** `grep -r '\[#' src/` — every result is a violation.

---

## Don't

- Mix frontend/backend code
- Hardcode brand names (use env vars)
- Skip Prisma migrations
- Commit API keys

---

**Last Updated**: 2026-01-23
