# Workspace

## Overview

pnpm workspace monorepo using TypeScript. Each package manages its own dependencies.

## Stack

- **Monorepo tool**: pnpm workspaces
- **Node.js version**: 24
- **Package manager**: pnpm
- **TypeScript version**: 5.9
- **API framework**: Express 5
- **Database**: PostgreSQL + Drizzle ORM
- **Validation**: Zod (`zod/v4`), `drizzle-zod`
- **API codegen**: Orval (from OpenAPI spec)
- **Build**: esbuild (CJS bundle)

## Structure

```text
artifacts-monorepo/
├── artifacts/              # Deployable applications
│   ├── api-server/         # Express API server
│   └── hn/                 # Arcan HN - Manhwa/Manga Tracker (React + Vite)
├── lib/                    # Shared libraries
│   ├── api-spec/           # OpenAPI spec + Orval codegen config
│   ├── api-client-react/   # Generated React Query hooks
│   ├── api-zod/            # Generated Zod schemas from OpenAPI
│   └── db/                 # Drizzle ORM schema + DB connection
├── scripts/                # Utility scripts (single workspace package)
│   └── src/                # Individual .ts scripts, run via `pnpm --filter @workspace/scripts run <script>`
├── pnpm-workspace.yaml     # pnpm workspace (artifacts/*, lib/*, lib/integrations/*, scripts)
├── tsconfig.base.json      # Shared TS options (composite, bundler resolution, es2022)
├── tsconfig.json           # Root TS project references
└── package.json            # Root package with hoisted devDeps
```

## Arcan HN — Manhwa Tracker

URL: `/hn/`

A full-stack manhwa/manga tracker with dark cosmic theme.

### Features
- **Passwordless login** — username only (min 3 chars), auto-registers
- **Library management** — track manhwa with tabs: All / New / Reading / History
- **Manga search** — search across AsuraScans, MangaDex, AsuraComic, MangaRead, MangaBuddy, Toonily
- **Chapter tracking** — track current chapter vs latest available
- **Toast notifications** — all actions have feedback
- **Smooth animations** — Framer Motion transitions

### Theme
- Primary background: `#070914`
- Card: `#101626`
- Accent: `#FF6F61` (coral)
- Dark cosmic mode only

### Pages
- `/` — Login page
- `/dashboard` — Stats, recently added, quick nav
- `/library` — Full library with tabs/search/sort
- `/manga/:id` — Manga detail with progress bar, chapter update
- `/add` — Search and add new manga

### API Endpoints (at `/api/`)
- `POST /api/auth/login` — login/register
- `POST /api/auth/logout`
- `GET /api/auth/me`
- `GET /api/manga/search?q=&source=`
- `GET /api/library` — with filters: category, sort, search
- `POST /api/library/add`
- `GET /api/library/:id`
- `PATCH /api/library/:id`
- `DELETE /api/library/:id`
- `GET /api/library/stats`

### Database Schema (PostgreSQL)
- `users` — id, username, created_at, last_login
- `manga` — id, title, source, url, cover_image, latest_chapter, created_at
- `user_library` — id, user_id, manga_id, current_chapter, latest_chapter, status, added_at, last_read_at

## TypeScript & Composite Projects

Every package extends `tsconfig.base.json` which sets `composite: true`. The root `tsconfig.json` lists all packages as project references. This means:

- **Always typecheck from the root** — run `pnpm run typecheck` (which runs `tsc --build --emitDeclarationOnly`). This builds the full dependency graph so that cross-package imports resolve correctly. Running `tsc` inside a single package will fail if its dependencies haven't been built yet.
- **`emitDeclarationOnly`** — we only emit `.d.ts` files during typecheck; actual JS bundling is handled by esbuild/tsx/vite...etc, not `tsc`.
- **Project references** — when package A depends on package B, A's `tsconfig.json` must list B in its `references` array. `tsc --build` uses this to determine build order and skip up-to-date packages.

## Root Scripts

- `pnpm run build` — runs `typecheck` first, then recursively runs `build` in all packages that define it
- `pnpm run typecheck` — runs `tsc --build --emitDeclarationOnly` using project references

## Packages

### `artifacts/api-server` (`@workspace/api-server`)

Express 5 API server. Routes live in `src/routes/` and use `@workspace/api-zod` for request and response validation and `@workspace/db` for persistence.

- Entry: `src/index.ts` — reads `PORT`, starts Express
- App setup: `src/app.ts` — mounts CORS, JSON/urlencoded parsing, session, routes at `/api`
- Routes: `src/routes/index.ts` — health, auth, manga, library
- Depends on: `@workspace/db`, `@workspace/api-zod`
- `pnpm --filter @workspace/api-server run dev` — run the dev server
- `pnpm --filter @workspace/api-server run build` — production esbuild bundle (`dist/index.cjs`)

### `artifacts/hn` (`@workspace/hn`)

Arcan HN — React + Vite manhwa tracker frontend. Served at `/hn/`.

- Entry: `src/main.tsx`
- Pages: login, dashboard, library, manga-detail, add-manga
- Uses: `@workspace/api-client-react` for generated React Query hooks
- Depends on: framer-motion, sonner (toasts), lucide-react

### `lib/db` (`@workspace/db`)

Database layer using Drizzle ORM with PostgreSQL. Exports a Drizzle client instance and schema models.

- `src/schema/users.ts` — users table
- `src/schema/manga.ts` — manga table
- `src/schema/library.ts` — user_library table

### `lib/api-spec` (`@workspace/api-spec`)

Owns the OpenAPI 3.1 spec (`openapi.yaml`) and the Orval config (`orval.config.ts`). Running codegen produces output into two sibling packages.

Run codegen: `pnpm --filter @workspace/api-spec run codegen`

### `lib/api-zod` (`@workspace/api-zod`)

Generated Zod schemas from the OpenAPI spec.

### `lib/api-client-react` (`@workspace/api-client-react`)

Generated React Query hooks and fetch client from the OpenAPI spec.

### `scripts` (`@workspace/scripts`)

Utility scripts package. Each script is a `.ts` file in `src/` with a corresponding npm script in `package.json`. Run scripts via `pnpm --filter @workspace/scripts run <script>`. Scripts can import any workspace package (e.g., `@workspace/db`) by adding it as a dependency in `scripts/package.json`.
