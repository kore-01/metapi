# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Summary

Metapi is a meta-aggregation gateway that sits in front of other AI API aggregators (New API, One API, OneHub, DoneHub, Veloera, AnyRouter, Sub2API, plus Claude/Codex/Gemini OAuth). Downstream clients see a single OpenAI/Claude/Gemini-compatible endpoint; Metapi handles upstream discovery, smart routing, failover cooldown, billing, and centralized account/token management.

The same backend ships in three shapes: a standalone Fastify server (Docker / Render / Zeabur), a desktop Electron app, and a Vite-built React admin UI. Read `AGENTS.md` first — it contains hard architectural rules that are enforced by `npm run repo:drift-check`.

## Build, Run, Test

Node `>=25.0.0` per `package.json` engines (README says 22.15+; CI / contributors typically use 22+). Package manager: `npm` (lockfile committed) — `pnpm-lock.yaml` is also tracked but `npm` is the primary.

```bash
# Web dev (concurrent backend + Vite frontend, hot reload)
npm run dev              # backend on :4000, frontend on :5173 with /api /v1 /monitor-proxy proxied
npm run dev:server       # backend only (tsx watch)
npm run dev:desktop      # backend + Vite + tsc desktop watch + Electron, waits on tcp:4000 and http:5173

# Build (each tsconfig builds a different target)
npm run build            # build:web + build:server + build:desktop
npm run build:web        # vite build (icons regenerated first)
npm run build:server     # tsc -p tsconfig.server.json + copy runtime DB-generated artifacts
npm run build:desktop    # tsc -p tsconfig.desktop.json
npm run dist:desktop     # electron-builder package (run `npm run build` first — full, not desktop-only)

# Typecheck (four separate projects; run them all before declaring victory)
npm run typecheck        # web + web:test + server + desktop

# Tests (vitest, root pinned to repo root, .worktrees excluded)
npm test                 # all tests
npm run test:watch
npx vitest run path/to/file.test.ts          # single file
npx vitest run -t "test name"                 # single test by name

# Schema-specific test buckets (live DB tests need real MySQL/Postgres, not run by `npm test`)
npm run test:schema:unit      # offline schema contract / parity / introspection
npm run test:schema:parity    # live drift between dialects
npm run test:schema:upgrade   # live upgrade path
npm run test:schema:runtime   # runtime bootstrap

# Smoke tests (spin up real DB, exercise full schema bootstrap)
npm run smoke:db              # SQLite (default)
npm run smoke:db:mysql
npm run smoke:db:postgres

# Database (Drizzle)
npm run db:generate           # generate Drizzle SQL from schema.ts
npm run db:migrate            # apply migrations
npm run schema:contract       # regenerate checked-in schema contract artifact
npm run schema:generate       # db:generate + schema:contract (use this for any schema change)

# Architecture / drift (run before submitting any boundary-touching work)
npm run repo:drift-check      # boundary + semantic-boundary tests — enforced by CI; failure blocks merge

# Docs (VitePress, separate site under docs/)
npm run docs:dev              # :4173
npm run docs:build
```

Required env vars (see `.env.example`): `AUTH_TOKEN` (admin login token), `PROXY_TOKEN` (downstream `/v1/*` token), `ACCOUNT_CREDENTIAL_SECRET` (32+ byte secret for at-rest credential encryption). `DATA_DIR` defaults to `./data`; SQLite lives at `${DATA_DIR}/hub.db`. Test runs override `NODE_ENV` to `test` automatically (see `vitest.config.ts`) — don't ship code that flips on `NODE_ENV !== 'production'` without thinking about test mode.

Windows note: use `restart.bat` to clear port locks; the dev scripts run under bash but the project is regularly developed on Windows.

## Architecture: the three-layer proxy stack

The proxy pipeline has strict layering enforced by tests in `src/server/routes/proxy/architecture-*.test.ts` and `routeRefreshWorkflow.architecture.test.ts`. **Violating these will fail `repo:drift-check`.**

### Layer 1 — `src/server/routes/proxy/**` (adapters only)

Files like `chat.ts`, `completions.ts`, `embeddings.ts`, `responses.ts` are Fastify handlers. They parse the request, build downstream client context, and delegate. They **must not** own:

- protocol conversion (OpenAI ⇄ Claude ⇄ Gemini) — that lives in `transformers/`
- retry policy / channel selection — that lives in `proxy-core/`
- billing / persistence — that lives in `services/`
- stream lifecycle — that lives in `proxy-core/conductor/`

If a helper under `routes/proxy/` is imported from anywhere else, it's misplaced.

### Layer 2 — `src/server/proxy-core/**` (orchestration)

Owns the request lifecycle:

- `conductor/DefaultProxyConductor.ts` — main orchestrator
- `orchestration/executeEndpointFlow` — endpoint fallback / retry across channels (use this, not ad-hoc try/catch)
- `surfaces/sharedSurface.ts` — channel + session bookkeeping
- `runtime/readRuntimeResponseText()` — use this for whole-body upstream reads, not raw `.text()`
- `channelSelection.ts` — weighted selection (cost 40% / balance 30% / usage 30%, plus cooldowns)
- `firstByteTimeout.ts` — first-byte timeout policy

### Layer 3 — `src/server/transformers/**` (protocol-pure)

Format conversion only. **Forbidden imports**: anything under `src/server/routes/`, Fastify, OAuth services, `tokenRouter`, runtime dispatch. If a transformer needs shared state, lift the contract into a neutral module first.

### Services and platform adapters

`src/server/services/` is the business-logic catch-all: balance refresh, checkin scheduler, alert/notify pipeline, route decision snapshots, proxy log retention, OAuth flows, etc. Most files have a colocated `*.test.ts`.

`src/server/services/platforms/` holds one file per upstream platform (`newApi.ts`, `oneApi.ts`, `oneHub.ts`, `doneHub.ts`, `veloera.ts`, `anyrouter.ts`, `sub2api.ts`, plus OAuth providers `claude.ts`, `codex.ts`, `gemini.ts`, `geminiCli.ts`, `antigravity.ts`). Each implements parts of `base.ts` / `standardApiProvider.ts`.

**Platform behavior must be explicit.** When adding a capability that varies per platform, declare it as a capability story (see `proxy-core/capabilities/`), not as scattered `if (platform === 'new-api')` branches. Thin adapters must not advertise features the upstream doesn't actually support — don't let inherited defaults lie.

Retry classification (proxy core) and routing health classification (services) should share the same failure vocabulary — see `proxyFailureJudge.ts` and `proxyRetryPolicy.ts`.

### Token routing

`services/tokenRouter.ts` selects a downstream-key → upstream-channel mapping per request. It has heavy test coverage (`tokenRouter.*.test.ts`) covering caching, downstream policy, OAuth route units, patterns, selection, session decoupling, site status. Touch it carefully and run the full router test bucket.

## Database rules

The schema lives in `src/server/db/schema.ts` (Drizzle). Three dialects are supported: SQLite (default, `better-sqlite3`), MySQL (`mysql2`), Postgres (`pg`). Cross-dialect bootstrap and upgrade SQL is **generated from the schema contract**, not hand-written.

**Any schema change requires all three outputs updated together:**

1. Update `src/server/db/schema.ts`
2. Regenerate SQLite migration history: `npm run db:generate`
3. Regenerate the checked-in schema contract artifact: `npm run schema:contract`

Run `npm run schema:generate` (steps 2+3 combined) after any schema edit. Don't hand-edit MySQL/Postgres schema patches in feature code — extend the contract instead.

Legacy schema compatibility shims (the various `ensure*Columns` calls in `src/server/index.ts`) are temporary, narrow, and tied to specific feature compatibility specs. Don't grow them casually.

## Web rules (`src/web`)

- Pages (`src/web/pages/<page>/`) are orchestration surfaces. **Pages do not import from other pages.**
- Reuse mobile primitives before inventing your own: `ResponsiveFilterPanel`, `ResponsiveBatchActionBar`, `MobileCard`, `useIsMobile`, `mobileLayout.ts`.
- When a page sprouts a second complex modal/drawer/panel family, extract it into a domain subfolder before adding more inline state.
- Generic display components go in `components/`. Single-page-only logic goes in `pages/<page>/helpers/`.

## Conventions

- Tests are colocated next to source (`foo.ts` + `foo.test.ts`) and named `*.test.ts(x)`. Test files for architectural boundaries end in `.architecture.test.ts` or `architecture-*.test.ts`.
- Runtime data → `data/`. Throwaway debug artifacts → `tmp/`. Never the repo root.
- Local planning files go in `docs/plans/` (gitignored, not published documentation). VitePress-published docs are the markdown files directly under `docs/`.
- Commit messages: conventional commits (`feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`).
- One coherent change per patch; sweep adjacent paths when fixing a repeated pattern (the "fix the family" rule from `AGENTS.md`).
