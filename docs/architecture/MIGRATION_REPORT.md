# GLU — Production Migration Report
## Express + Drizzle → NestJS + Prisma

**Document Status:** Awaiting Approval Before Implementation  
**Prepared:** 2025-07-24  
**Architecture Reference:** `docs/architecture/ARCHITECTURE.md` + ADR-001 through ADR-012  
**Scope:** Full backend replacement; frontend preserved with targeted additions

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [File-by-File Classification](#2-file-by-file-classification)
3. [Risk Assessment](#3-risk-assessment)
4. [Estimated Migration Time](#4-estimated-migration-time)
5. [Breaking Changes](#5-breaking-changes)
6. [Dependency Changes](#6-dependency-changes)
7. [Database Migration Plan](#7-database-migration-plan)
8. [Rollback Plan](#8-rollback-plan)
9. [Testing Strategy](#9-testing-strategy)

---

## 1. Executive Summary

The current codebase is a **scaffold with no business logic**. There are zero implemented routes beyond a health check. The database schema is empty. This means the migration is not a refactor of a running system — it is a **greenfield build on top of preserved shared infrastructure**.

The critical insight is that the most valuable assets in the project are entirely preserved:

| Asset | Value | Action |
|---|---|---|
| `lib/api-spec/openapi.yaml` | Complete API contract for 30+ endpoints | **Keep — source of truth** |
| `lib/api-zod/` (55 files, 907 lines) | All Zod validators + TypeScript types | **Keep — regenerated on spec change** |
| `lib/api-client-react/` | Full React Query client + custom fetch | **Keep — zero changes** |
| `artifacts/glu-web/src/components/ui/` | 55 Shadcn/UI components | **Keep — zero changes** |
| `artifacts/glu-web/vite.config.ts` | Correct Replit environment config | **Keep — zero changes** |

The backend (`artifacts/api-server/`) is a 5-file skeleton. Nothing there represents implemented business logic. The migration risk from the **backend** is essentially zero — there is nothing to lose.

The risk profile for this project is therefore:
- **Backend**: Low risk (empty skeleton, clean replacement)
- **Frontend shell**: Low risk (kept entirely)
- **Shared libs**: Low risk (kept entirely, zero changes)
- **Database**: Zero risk (schema is empty, no data to lose)

---

## 2. File-by-File Classification

### Legend
| Category | Meaning |
|---|---|
| **Keep** | Used as-is. Zero changes. |
| **Refactor** | Core logic preserved; adapted to new framework patterns. |
| **Rewrite** | Same concept needed; implementation replaced completely. |
| **Delete** | Superseded, incompatible, or made redundant. |

---

### 2.1 API Server — `artifacts/api-server/`

#### `src/app.ts` — **Rewrite**
**Current:** Express app factory. Mounts pino-http, CORS, body parsing, and the `/api` router.  
**Why rewrite:** NestJS replaces every line of this file. App configuration lives in `main.ts` (Helmet, CORS, global pipes, global interceptors, Swagger setup) and `AppModule`. There is no Express app object — NestJS wraps platform-express internally.  
**What carries over (conceptually):**
- The pino-http redact config (authorization, cookie, set-cookie) is reused in the `nestjs-pino` configuration.
- The `/api` prefix becomes `app.setGlobalPrefix('api')` in `main.ts`.
- CORS origins move to a typed `AppConfig`.

---

#### `src/index.ts` — **Refactor**
**Current:** 25-line entry point. Reads `PORT` from env, validates it is numeric and positive, starts the server.  
**Why refactor:** NestJS bootstrap needs a `main.ts` in its own shape, but the PORT validation logic — reading from env, throwing with a descriptive error, guarding against NaN — is correct and worth preserving verbatim. It moves into `main.ts` before `NestFactory.create()`.  
**Reused exactly:**
```typescript
const rawPort = process.env['PORT'];
if (!rawPort) throw new Error('PORT environment variable is required but was not provided.');
const port = Number(rawPort);
if (Number.isNaN(port) || port <= 0) throw new Error(`Invalid PORT value: "${rawPort}"`);
```

---

#### `src/lib/logger.ts` — **Refactor**
**Current:** 20-line Pino logger. Environment-aware, with redact rules and pino-pretty in dev.  
**Why refactor:** This is the best-written file in the current backend. Every decision is correct:
- `process.env.LOG_LEVEL ?? "info"` as configurable default
- Sensitive field redaction list (auth header, cookie, set-cookie)
- pino-pretty for dev only (zero performance cost in production)

In NestJS, Pino integrates via `nestjs-pino`. The `params` object from this logger (`level`, `redact`, `transport`) map 1:1 to `LoggerModule.forRoot({ pinoHttp: { ... } })`. The redact list and transport config are copied verbatim into `src/config/logger.config.ts`.

---

#### `src/routes/index.ts` — **Delete**
**Current:** 8-line Express Router aggregator.  
**Why delete:** NestJS has no equivalent file. Routing is declared within each module's controller. `AppModule.imports[]` is the aggregator. The concept is preserved in the module system; the file itself is not.

---

#### `src/routes/health.ts` — **Rewrite**
**Current:** One Express GET `/healthz` handler that calls `HealthCheckResponse.parse({ status: "ok" })`.  
**Why rewrite:** NestJS uses `@nestjs/terminus` for health checks. The new implementation:
- Uses `HealthCheckService` and `PrismaHealthIndicator`, `RedisHealthIndicator`, `MeilisearchHealthIndicator`
- Responds to `GET /api/healthz` with the same `{ status }` shape
- The Zod schema (`HealthCheckResponse`) is still used to validate the response shape in tests

The pattern of validating health response output with the generated Zod schema carries over into tests.

---

#### `src/middlewares/.gitkeep` — **Delete**
**Current:** Empty placeholder file.  
**Why delete:** NestJS global middleware is registered in `main.ts`. Module-level middleware goes in each module's `configure()` method. The directory concept is not needed.

---

#### `package.json` — **Rewrite**
**Current:** Express, Drizzle-orm, Pino, cors, cookie-parser. ESM module type.  
**Why rewrite:** The entire dependency set changes. NestJS requires CommonJS (`"type"` field removed). See [Section 6: Dependency Changes](#6-dependency-changes) for the complete list.

---

#### `tsconfig.json` — **Rewrite**
**Current:** Extends `../../tsconfig.base.json` (ESM, `moduleResolution: "bundler"`, no decorators).  
**Why rewrite:** NestJS is fundamentally incompatible with the base tsconfig. Requirements:
- `"module": "CommonJS"` (NestJS runtime)
- `"moduleResolution": "node"` (compatible with CommonJS)
- `"experimentalDecorators": true` (all NestJS decorators)
- `"emitDecoratorMetadata": true` (required for DI reflection)
- `"target": "ES2021"` (modern Node.js features)
- `"strict": true`

The new `tsconfig.json` does **not** extend the workspace base — it is a standalone NestJS tsconfig. The workspace base continues serving all ESM packages (lib/*, glu-web) unchanged.

---

#### `build.mjs` — **Delete**
**Current:** esbuild config. Outputs ESM bundle with CJS interop banner. Externalizes native modules.  
**Why delete:** NestJS builds are managed by `@nestjs/cli` (`nest build`), which uses `tsc` under the hood. The external list is informative — it captures which native packages cannot be bundled — but the esbuild approach is replaced. The external list's signal (argon2, @prisma/client, @aws-sdk/*, sharp) guides what to mark as external in the NestJS CLI config.  
**What carries over:** The `nest-cli.json` configuration will reference the same set of externals in the webpack config override.

---

### 2.2 Database Library — `lib/db/`

#### `src/index.ts` — **Delete**
**Current:** Creates `pg.Pool` and wraps it in `drizzle()`. Exports `db`, `pool`, and all schema.  
**Why delete:** Prisma replaces Drizzle entirely. The `PrismaService` in `artifacts/api-server/src/database/` owns the database connection. The `lib/db` package's role as "shared DB access" ends — NestJS DI handles connection sharing within the server.

---

#### `src/schema/index.ts` — **Delete**
**Current:** Empty file. Contains only a template comment with an example schema.  
**Why delete:** The schema is empty. Nothing to lose. Replaced by `artifacts/api-server/prisma/schema.prisma`, which contains the full domain model derived from `lib/api-spec/openapi.yaml`.

---

#### `package.json` — **Rewrite**
**Current:** Drizzle-orm, drizzle-zod, pg, zod.  
**Why rewrite:** The `lib/db` package is repurposed or removed. Two options:
- **Option A (recommended):** Keep `lib/db` as a thin re-export of `@prisma/client` types, so both the api-server and any future services share the same generated Prisma types.
- **Option B:** Remove `lib/db` entirely; api-server uses `@prisma/client` directly.

Option A is chosen. `lib/db` becomes `{ dependencies: { "@prisma/client": "..." }, exports: { ".": "./src/index.ts" } }` where `src/index.ts` re-exports from `@prisma/client`.

---

#### `drizzle.config.ts` — **Delete**
**Current:** Drizzle Kit config pointing to the schema file.  
**Why delete:** Prisma replaces Drizzle Kit. Schema management is via `prisma migrate dev` and `prisma db push`.

---

### 2.3 API Specification — `lib/api-spec/`

#### `openapi.yaml` — **Keep**
**Justification:** This is the most important file in the project. It is the contract between backend and frontend. Every NestJS controller endpoint must match this spec exactly. The file is not touched during migration — it is the migration's target.

---

#### `orval.config.ts` — **Keep**
**Justification:** The Orval codegen pipeline (OpenAPI → Zod validators + React Query hooks) is architecture-independent. It runs against `openapi.yaml` and outputs to `lib/api-zod` and `lib/api-client-react`. No changes.

---

#### `package.json` — **Keep**
**Justification:** Orval dependency. No changes.

---

### 2.4 Generated Zod Library — `lib/api-zod/`

#### `src/generated/api.ts` (907 lines) — **Keep**
**Justification:** Generated from `openapi.yaml`. Contains Zod validators for every request body, query param set, and response shape across all 30+ endpoints. These validators become the NestJS controller's validation layer — imported directly into controllers and used in the global `ZodValidationPipe`.

Critically: the generated request body validators (`RegisterBody`, `LoginBody`, `CreateReviewBody`, etc.) replace the need to write manual NestJS DTOs. The OpenAPI spec → generated validator → NestJS pipe chain is a major DX win.

---

#### `src/generated/types/*.ts` (55 files) — **Keep**
**Justification:** Generated TypeScript interfaces for all domain entities. These are the types returned by NestJS controllers. No changes.

---

#### `package.json` — **Keep**
**Justification:** Zod dep only. No changes.

---

### 2.5 React Client Library — `lib/api-client-react/`

#### `src/custom-fetch.ts` (371 lines) — **Keep**
**Justification:** This is production-grade, battle-hardened code that handles every edge case of HTTP communication:
- BOM stripping on JSON responses
- `HEAD`/`204`/`205`/`304` no-body detection
- Automatic JSON/text/blob content-type inference
- `ApiError` and `ResponseParseError` typed error classes
- Configurable bearer token injection (for Expo/React Native)
- Configurable base URL (for Expo/React Native)
- React Native compatible (`response.body === undefined` handled correctly)

This file is completely framework-agnostic. The NestJS migration has zero impact on it. **Do not touch.**

---

#### `src/index.ts` — **Keep** (with minor cleanup)
**Current:** Re-exports generated files + `setBaseUrl` + `setAuthTokenGetter`. Has one duplicate `export * from './generated/api'` (lines 1 and 5 are identical).  
**Action:** Remove the duplicate export. Otherwise keep unchanged.

---

#### `src/generated/api.ts` — **Keep**
**Justification:** Generated React Query hooks. Regenerated from `openapi.yaml` via Orval.

---

#### `src/generated/api.schemas.ts` — **Keep**
**Justification:** Generated schema types. Regenerated from `openapi.yaml` via Orval.

---

#### `package.json` — **Keep**
**Justification:** React Query dependency. No changes.

---

### 2.6 Web Application — `artifacts/glu-web/`

#### `src/App.tsx` — **Refactor**
**Current:** Placeholder "building" home page. The routing skeleton (WouterRouter, QueryClient, TooltipProvider, Toaster) is correct.  
**Why refactor:** Add real routes as each feature module is built. The architectural shell (router, providers, error boundaries) remains unchanged. Pages are added one by one as backend routes become available.

---

#### `src/main.tsx` — **Keep**
**Justification:** 7-line React root render. Standard, no changes.

---

#### `src/index.css` — **Keep**
**Justification:** Tailwind CSS v4 configuration with design tokens. No changes.

---

#### `src/lib/utils.ts` — **Keep**
**Justification:** `cn()` utility (clsx + tailwind-merge). Standard Shadcn pattern. No changes.

---

#### `src/pages/not-found.tsx` — **Keep**
**Justification:** 404 fallback page. Needed. No changes.

---

#### `src/hooks/use-mobile.tsx` — **Keep**
**Justification:** `useIsMobile()` hook using `matchMedia`. Standard responsive utility. No changes.

---

#### `src/hooks/use-toast.ts` — **Keep**
**Justification:** Toast state management powering the `Toaster` component. No changes.

---

#### `src/components/ui/*.tsx` (55 files) — **Keep all**
**Justification:** Complete Shadcn/UI component library. These are:
- Framework-agnostic React primitives built on Radix UI
- Production-ready (accessibility, keyboard navigation, theming)
- The foundation for every future page in GLU
- Zero dependency on the backend framework

No file in this directory changes during the migration. They are consumed as-is when pages are built.

**Full list:** accordion, alert-dialog, alert, aspect-ratio, avatar, badge, breadcrumb, button-group, button, calendar, card, carousel, chart, checkbox, collapsible, command, context-menu, dialog, drawer, dropdown-menu, empty, field, form, hover-card, input-group, input-otp, input, item, kbd, label, menubar, navigation-menu, pagination, popover, progress, radio-group, resizable, scroll-area, select, separator, sheet, sidebar, skeleton, slider, sonner, spinner, switch, table, tabs, textarea, toaster, toast, toggle-group, toggle, tooltip.

---

#### `vite.config.ts` — **Keep**
**Justification:** Correctly configured for the Replit environment — PORT from env, BASE_PATH from env, 0.0.0.0 host, `allowedHosts: true`, Replit plugins (cartographer, dev-banner, runtime-error-modal), path aliases. This is one of the most important config files in the project and must not be changed.

---

#### `package.json` — **Keep**
**Justification:** All correct frontend dependencies already present (React 19, Radix UI, TanStack Query, Wouter, Tailwind v4, Framer Motion, Lucide, react-hook-form, Zod, recharts, sonner, vaul, etc.). No additions or removals needed.

---

### 2.7 Workspace Root

#### `tsconfig.base.json` — **Keep**
**Justification:** ESM base config serves all workspace packages correctly. The api-server deliberately diverges (NestJS needs CJS). No changes to the base — it continues serving `lib/*` and `artifacts/glu-web`.

---

#### `pnpm-workspace.yaml` — **Keep**
**Justification:** Workspace package discovery config. No structural changes to the workspace layout.

---

#### `package.json` (root) — **Keep**
**Justification:** Root scripts, shared dev dependencies. No changes required.

---

### 2.8 Classification Summary

| File | Classification | Reason (summary) |
|---|---|---|
| `api-server/src/app.ts` | Rewrite | Express → NestJS bootstrap |
| `api-server/src/index.ts` | Refactor | PORT validation logic reused in main.ts |
| `api-server/src/lib/logger.ts` | Refactor | Pino config logic reused in logger.config.ts |
| `api-server/src/routes/index.ts` | Delete | Replaced by NestJS module system |
| `api-server/src/routes/health.ts` | Rewrite | Replaced by @nestjs/terminus controller |
| `api-server/src/middlewares/.gitkeep` | Delete | Empty placeholder |
| `api-server/package.json` | Rewrite | Full dependency replacement |
| `api-server/tsconfig.json` | Rewrite | NestJS requires CJS + decorators |
| `api-server/build.mjs` | Delete | Replaced by NestJS CLI |
| `lib/db/src/index.ts` | Delete | Drizzle → Prisma; replaced by PrismaService |
| `lib/db/src/schema/index.ts` | Delete | Empty; replaced by schema.prisma |
| `lib/db/package.json` | Rewrite | Repurposed as @prisma/client wrapper |
| `lib/db/drizzle.config.ts` | Delete | Replaced by Prisma migration system |
| `lib/api-spec/openapi.yaml` | **Keep** | Source of truth — do not touch |
| `lib/api-spec/orval.config.ts` | **Keep** | Codegen pipeline unchanged |
| `lib/api-spec/package.json` | **Keep** | No changes |
| `lib/api-zod/generated/api.ts` | **Keep** | Generated — Zod validators for all endpoints |
| `lib/api-zod/generated/types/*.ts` (55 files) | **Keep** | Generated — TypeScript interfaces |
| `lib/api-zod/package.json` | **Keep** | No changes |
| `lib/api-client-react/src/custom-fetch.ts` | **Keep** | Production-quality, framework-agnostic |
| `lib/api-client-react/src/index.ts` | **Keep** (minor fix) | Remove duplicate export |
| `lib/api-client-react/src/generated/*` | **Keep** | Generated — React Query hooks |
| `lib/api-client-react/package.json` | **Keep** | No changes |
| `glu-web/src/App.tsx` | Refactor | Add real routes progressively |
| `glu-web/src/main.tsx` | **Keep** | Standard entry point |
| `glu-web/src/index.css` | **Keep** | Tailwind v4 config |
| `glu-web/src/lib/utils.ts` | **Keep** | cn() utility |
| `glu-web/src/pages/not-found.tsx` | **Keep** | 404 fallback |
| `glu-web/src/hooks/*.ts` | **Keep** | use-mobile, use-toast |
| `glu-web/src/components/ui/*.tsx` (55 files) | **Keep** | Full Shadcn library |
| `glu-web/vite.config.ts` | **Keep** | Correct Replit environment config |
| `glu-web/package.json` | **Keep** | All correct deps already present |
| `tsconfig.base.json` | **Keep** | ESM base for all non-NestJS packages |
| `pnpm-workspace.yaml` | **Keep** | Workspace layout unchanged |
| `package.json` (root) | **Keep** | No changes |

**Totals:**
- Keep: 35 files / file groups
- Refactor: 3 files
- Rewrite: 4 files
- Delete: 7 files

---

## 3. Risk Assessment

### 3.1 Risk Matrix

| Area | Risk Level | Likelihood | Impact | Reasoning |
|---|---|---|---|---|
| Backend rewrite | 🟢 Low | Low | Low | Empty skeleton; no business logic to lose |
| Database schema | 🟢 Low | Low | Low | Schema is empty; no data exists |
| Shared lib compatibility | 🟢 Low | Low | Low | lib/api-zod, lib/api-client-react untouched |
| Frontend app | 🟢 Low | Low | Low | Components and config preserved entirely |
| OpenAPI contract drift | 🟡 Medium | Medium | High | NestJS impl must match spec exactly; drift would break generated client |
| NestJS + pnpm workspace CJS/ESM boundary | 🟡 Medium | Medium | Medium | The workspace mixes ESM (libs, frontend) and CJS (NestJS); import paths need care |
| Prisma + TypeScript project references | 🟡 Medium | Medium | Medium | Prisma generates its own types; workspace references may need updating |
| Decorator metadata in NestJS | 🟡 Medium | Medium | High | Missing `emitDecoratorMetadata` or `reflect-metadata` import breaks DI silently |
| NestJS circular dependencies | 🟡 Medium | Low | High | NestJS DI with many modules risks circular deps if not carefully structured |
| AI provider API key management | 🟡 Medium | Medium | Medium | Keys must be in env secrets, not code; missing key = startup failure |
| Qdrant not available locally | 🟡 Medium | Medium | Medium | If Qdrant docker isn't running, AI/vector features fail; must degrade gracefully |
| Production data loss | 🔴 N/A | N/A | N/A | No production deployment exists yet; no data to lose |

---

### 3.2 Low Risk Areas

**All shared libraries (`lib/api-zod`, `lib/api-client-react`, `lib/api-spec`)**
These are read-only from the backend's perspective and completely unchanged. The risk of breaking the frontend-backend type contract during migration is near-zero because the OpenAPI spec is the source of truth and it does not change.

**Frontend component library (`glu-web/src/components/ui/`)**
55 Shadcn components with zero backend dependencies. Identical before and after migration.

**Vite configuration**
Already correctly handles all Replit environment requirements. Not touched.

**Database (empty schema)**
There is no data to migrate, no existing queries to break, no foreign key constraints to worry about. Starting fresh with Prisma on an empty database is the lowest-risk database scenario possible.

---

### 3.3 Medium Risk Areas

**CJS/ESM boundary in pnpm workspace**  
The workspace runs ESM packages (`lib/*`, `glu-web`) alongside a CJS NestJS server. The boundary is enforced by the `api-server` having its own standalone `tsconfig.json` (not extending the ESM base). Risk: If any ESM-only package is inadvertently imported into NestJS source, it fails at runtime with `ERR_REQUIRE_ESM`. Mitigation: strict `tsconfig.json` separation and careful import auditing during development.

**OpenAPI contract drift**  
The generated client (`lib/api-client-react`) is generated from `openapi.yaml`. If a NestJS controller returns a field with a different name or type than what the spec declares, the frontend Zod parsing silently passes (extra fields) or fails (missing required fields). Mitigation: integration tests validate every endpoint response against its generated Zod schema before deployment.

**NestJS Prisma module setup**  
Prisma needs to be imported carefully — `PrismaService` must extend `OnApplicationShutdown` to properly disconnect. Missing this causes connection leaks in tests. Mitigation: follow the exact setup from `nestjs-prisma` documentation.

---

### 3.4 High Risk Areas

**None identified** for this migration, given the clean-slate backend state.

The only scenario that would create high risk is scope expansion during migration — for example, if other engineers merge changes to shared libraries while the migration is in progress. Mitigation: freeze shared library changes during the migration sprint.

---

## 4. Estimated Migration Time

All estimates assume one senior engineer working full-time. Estimates are wall-clock implementation time, not calendar time.

### Phase 1 — Infrastructure Setup

| Task | Effort | Notes |
|---|---|---|
| NestJS bootstrap (main.ts, app.module.ts, tsconfig, nest-cli.json) | 2h | Standard NestJS setup |
| Prisma schema (full domain model from openapi.yaml) | 3h | 20+ models, relations, indexes, enums |
| PrismaModule + PrismaService | 1h | Singleton, shutdown hook, health indicator |
| Config module (typed, validated env vars) | 2h | 8 config namespaces |
| **Phase 1 Total** | **8h** | |

### Phase 2 — Core Infrastructure Modules

| Module | Effort | Notes |
|---|---|---|
| CacheModule (Redis, ioredis) | 2h | Cache service, TTL constants, key factory |
| EventBusModule (IEventBus + BullMQ impl + InMemory impl) | 3h | Interface + 2 providers + domain event types |
| StorageModule (IStorageProvider + MinIO + R2 providers) | 4h | Interface + 2 providers, signed URLs, CDN URL |
| SearchModule (Meilisearch abstraction + index definitions) | 3h | Interface + provider + 3 index definitions |
| VectorModule (IVectorStore + Qdrant stub) | 2h | Interface + Qdrant provider (basic upsert/search) |
| ObservabilityModule (health checks, OTEL stubs) | 3h | Terminus health for DB, Redis, Meili, Qdrant |
| QueueModule (BullMQ queue definitions, 8 queues) | 2h | Queue defs + processor stubs |
| **Phase 2 Total** | **19h** | |

### Phase 3 — Security & Cross-Cutting

| Module | Effort | Notes |
|---|---|---|
| Common (filters, guards, interceptors, pipes, decorators) | 4h | GlobalExceptionFilter, ZodValidationPipe, TransformInterceptor, AuditInterceptor |
| RBACModule (roles, permissions, policies, guard) | 3h | PermissionsRegistry, PermissionsGuard, policy base |
| FeatureFlagsModule (Redis-backed, Guard, Registry) | 2h | Flag evaluation, rollout %, user-based |
| **Phase 3 Total** | **9h** | |

### Phase 4 — Feature Modules (Core Domain)

| Module | Effort | Notes |
|---|---|---|
| AuthModule (JWT, register, login, logout, /me, argon2) | 6h | JWT strategy, passport, refresh token, guards |
| UsersModule (profile, update, stats) | 3h | CRUD + stats aggregation |
| CategoriesModule (list) | 1h | Read-only, seeded |
| AuthorsModule (list, get, follow, unfollow, books) | 3h | Includes follow relationship |
| BooksModule (list, featured, trending, new releases, get, like, search) | 6h | Complex queries, Meilisearch sync, view count |
| ChaptersModule (get book chapters, get chapter content) | 2h | Content serving, word count |
| ReviewsModule (list, create, delete) | 3h | Rating aggregation, event emission |
| LibraryModule (get, add, update, remove) | 3h | User reading status + progress |
| BookmarksModule (get, create, delete) | 2h | Simple CRUD with chapter join |
| **Phase 4 Total** | **29h** | |

### Phase 5 — AI Module

| Module | Effort | Notes |
|---|---|---|
| AI Gateway + provider interface | 3h | Routing, failover chain, cost-aware dispatch |
| OpenAI provider implementation | 3h | Chat, stream, embeddings, moderation |
| Prompt loader (file-based, Handlebars) | 2h | Load .md files, render templates |
| Prompt files (all 7 prompt categories) | 2h | summary, teacher, quiz, translator, writing, moderator, librarian |
| EmbeddingsService (with Redis cache) | 2h | Embedding + 7-day TTL cache |
| RAG service (retrieve + context build) | 4h | Vector search → rerank → context window → prompt |
| Memory service (short-term Redis, long-term Qdrant) | 3h | Session memory + user preference vectors |
| AI Cost module (budget enforcer, tracker) | 3h | Per-user daily/monthly limits, Redis counters |
| AI endpoints (summarize + chat with book + history) | 4h | Wire gateway → agents → RAG → response |
| **Phase 5 Total** | **26h** | |

### Phase 6 — Knowledge Engine

| Module | Effort | Notes |
|---|---|---|
| Pipeline orchestrator + stage interfaces | 2h | Normalize → Parse → Chunk → Embed → Vector |
| Content processors (PDF, EPUB, HTML, Markdown) | 5h | One processor per content type |
| Knowledge engine service (book ingest) | 3h | Wire pipeline for book content |
| Knowledge ingest queue processor | 1h | Wire BullMQ → knowledge engine |
| **Phase 6 Total** | **11h** | |

### Phase 7 — Database Seeding

| Task | Effort | Notes |
|---|---|---|
| Prisma seed (categories, sample books, admin user) | 3h | Realistic seed data for development |
| **Phase 7 Total** | **3h** | |

### Phase 8 — Frontend Pages

| Task | Effort | Notes |
|---|---|---|
| Home page (featured books, trending, stats) | 4h | |
| Books list + search + filters | 4h | |
| Book detail page + chapters | 4h | |
| Auth pages (login, register) | 3h | |
| User library + reading progress | 3h | |
| Author profile page | 2h | |
| AI chat interface (within book reader) | 4h | |
| User profile + reading stats | 2h | |
| **Phase 8 Total** | **26h** | |

### Phase 9 — Testing

| Task | Effort | Notes |
|---|---|---|
| Unit tests (services, repositories, utilities) | 8h | |
| Integration tests (controllers vs openapi spec) | 6h | |
| E2E tests (critical flows: auth, read, AI chat) | 6h | |
| **Phase 9 Total** | **20h** | |

### Total Estimates

| Phase | Hours |
|---|---|
| Phase 1: Infrastructure Setup | 8h |
| Phase 2: Core Infrastructure Modules | 19h |
| Phase 3: Security & Cross-Cutting | 9h |
| Phase 4: Feature Modules (Core Domain) | 29h |
| Phase 5: AI Module | 26h |
| Phase 6: Knowledge Engine | 11h |
| Phase 7: Database Seeding | 3h |
| Phase 8: Frontend Pages | 26h |
| Phase 9: Testing | 20h |
| **Grand Total** | **~151h** |

**Recommended implementation order:** Phases 1 → 2 → 3 → 4 (MVP backend) → 7 → 8 (MVP frontend) → 5 → 6 → 9

The MVP (auth + books + library + basic UI) is deliverable after ~95h of work.

---

## 5. Breaking Changes

### 5.1 No Breaking Changes to External Consumers

The `lib/api-spec/openapi.yaml` is preserved unchanged. All generated types in `lib/api-zod` and `lib/api-client-react` remain identical. The frontend React Query hooks call the same endpoints with the same request/response shapes. **There are zero breaking changes to the API contract.**

---

### 5.2 Internal Breaking Changes (Between Modules)

The following changes are internal to the server and break nothing externally, but engineers working on the project must be aware:

#### Database ORM
| Before | After |
|---|---|
| `import { db } from '@workspace/db'` | Inject `PrismaService` via NestJS DI |
| `db.select().from(table).where(eq(col, val))` | `this.prisma.book.findMany({ where: { ... } })` |
| Drizzle `integer`, `text`, `pgTable` types | Prisma `Int`, `String`, `@db.Text` in schema.prisma |
| `drizzle-kit push` for schema changes | `prisma migrate dev` for schema changes |
| `drizzle-zod` for schema-derived validation | Prisma + manually written Zod schemas (or `zod-prisma-types`) |

#### HTTP Framework
| Before | After |
|---|---|
| `import { Request, Response } from 'express'` | `@Req()`, `@Res()`, `@Body()`, `@Param()` decorators |
| `res.json(data)` | `return data` (NestJS serializes automatically) |
| `res.status(201).json(data)` | `@HttpCode(201)` decorator on controller method |
| `req.user` (from passport middleware) | `@CurrentUser()` custom decorator |
| Error: `res.status(404).json({ error: '...' })` | `throw new NotFoundException('...')` |
| Manual try/catch in routes | GlobalExceptionFilter handles all errors |

#### Module Imports
| Before | After |
|---|---|
| `import { HealthCheckResponse } from '@workspace/api-zod'` (in server) | Generated schemas still importable, but used differently in NestJS pipes |
| `import router from './routes'` | `imports: [BooksModule, AuthModule, ...]` in AppModule |

#### Build System
| Before | After |
|---|---|
| `pnpm run build` → `node build.mjs` → esbuild → `dist/index.mjs` | `pnpm run build` → `nest build` → tsc → `dist/main.js` |
| `pnpm run start` → `node dist/index.mjs` | `pnpm run start` → `node dist/main.js` |
| `pnpm run dev` → build + start | `pnpm run dev` → `nest start --watch` (hot reload) |
| ESM output (`"type": "module"`) | CJS output (no `"type"` field = CJS default) |

#### Configuration
| Before | After |
|---|---|
| `process.env.PORT` read directly | Read in `main.ts` before bootstrap; all other env vars read via `@nestjs/config` typed providers |
| No env validation | `ConfigModule` validates all required env vars at startup; missing vars = crash with clear message |

---

### 5.3 Behavior Changes (Intentional Improvements)

These are not breaking changes but are deliberate behavioral differences:

| Behavior | Before | After |
|---|---|---|
| Auth | No auth (not implemented) | JWT RS256, argon2id, refresh token rotation |
| Error format | Not standardized | RFC 7807 Problem Details (`{ status, title, detail }`) |
| Validation errors | Not implemented | Structured Zod errors with field paths |
| Rate limiting | None | 100 req/min general, 10 req/min for auth endpoints |
| CORS | `cors()` with defaults (all origins) | Restricted to configured origins in AppConfig |
| Health check | Returns `{ status: "ok" }` always | Returns component health (DB, Redis, Meili, Qdrant) with individual status |
| Logging | HTTP req/res logged via pino-http | Same + traceId on every log line |

---

## 6. Dependency Changes

### 6.1 Packages Added (api-server)

**NestJS Core**

| Package | Version | Purpose |
|---|---|---|
| `@nestjs/common` | `^11.0` | Core decorators, pipes, guards, interceptors |
| `@nestjs/core` | `^11.0` | NestJS application kernel, DI container |
| `@nestjs/platform-express` | `^11.0` | Express HTTP adapter |
| `@nestjs/config` | `^3.3` | Typed env config module |
| `@nestjs/jwt` | `^10.2` | JWT sign/verify |
| `@nestjs/passport` | `^10.0` | Passport.js integration |
| `@nestjs/cache-manager` | `^2.3` | Cache abstraction (Redis) |
| `@nestjs/throttler` | `^6.4` | Rate limiting (Redis store) |
| `@nestjs/terminus` | `^10.2` | Health checks |
| `@nestjs/swagger` | `^8.1` | OpenAPI/Swagger UI generation |
| `@nestjs/bullmq` | `^10.2` | BullMQ queue integration |
| `@nestjs/event-emitter` | `^2.0` | Synchronous in-process event emitter |
| `@nestjs/schedule` | `^4.0` | Cron/interval job scheduling |
| `reflect-metadata` | `^0.2` | Required for decorator metadata |
| `rxjs` | `^7.8` | Required by NestJS core |

**Database**

| Package | Version | Purpose |
|---|---|---|
| `@prisma/client` | `^5.22` | Generated Prisma client |
| `prisma` (dev) | `^5.22` | Prisma CLI (migrate, generate) |

**Auth & Security**

| Package | Version | Purpose |
|---|---|---|
| `argon2` | `^0.41` | Password hashing (Argon2id) |
| `passport` | `^0.7` | Auth middleware base |
| `passport-jwt` | `^4.0` | JWT strategy |
| `passport-local` | `^1.0` | Local email+password strategy |
| `helmet` | `^8.0` | Security headers middleware |
| `@types/passport` (dev) | | |
| `@types/passport-jwt` (dev) | | |
| `@types/passport-local` (dev) | | |

**Cache & Queue**

| Package | Version | Purpose |
|---|---|---|
| `ioredis` | `^5.3` | Redis client (used by cache, throttler, bullmq) |
| `cache-manager` | `^5.7` | Cache manager abstraction |
| `cache-manager-ioredis-yet` | `^2.1` | Redis store for cache-manager |
| `bullmq` | `^5.1` | Background job queue |
| `@bull-board/express` | `^5.25` | BullMQ dashboard UI |
| `@bull-board/nestjs` | `^5.25` | NestJS adapter for bull-board |

**Search & Vector**

| Package | Version | Purpose |
|---|---|---|
| `meilisearch` | `^0.46` | Meilisearch Node.js client |
| `@qdrant/js-client-rest` | `^1.11` | Qdrant vector DB client |

**Storage**

| Package | Version | Purpose |
|---|---|---|
| `@aws-sdk/client-s3` | `^3.740` | S3-compatible storage (R2, MinIO) |
| `@aws-sdk/s3-request-presigner` | `^3.740` | Presigned URL generation |
| `@aws-sdk/lib-storage` | `^3.740` | Multipart upload support |

**AI Platform**

| Package | Version | Purpose |
|---|---|---|
| `openai` | `^4.77` | OpenAI API client |
| `@anthropic-ai/sdk` | `^0.33` | Anthropic Claude client |
| `@google/generative-ai` | `^0.21` | Google Gemini client |
| `handlebars` | `^4.7` | Prompt template rendering |
| `tiktoken` | `^1.0` | Token counting for cost estimation |

**Utilities**

| Package | Version | Purpose |
|---|---|---|
| `zod` | `catalog:` | Schema validation (already in workspace catalog) |
| `nestjs-zod` | `^3.0` | Zod + NestJS Swagger integration |
| `pino` | `^9.14` | Structured logging (already in server) |
| `nestjs-pino` | `^4.0` | NestJS Pino integration |
| `compression` | `^1.8` | Gzip response compression |
| `uuid` | `^11.0` | UUID generation |
| `ms` | `^2.1` | Human-readable duration strings |
| `sharp` | `^0.33` | Image processing (cover resizing) |
| `@types/compression` (dev) | | |
| `@types/uuid` (dev) | | |
| `@types/ms` (dev) | | |

**Build & Dev**

| Package | Version | Purpose |
|---|---|---|
| `@nestjs/cli` (dev) | `^10.4` | NestJS build system, `nest start --watch` |
| `@nestjs/schematics` (dev) | `^10.2` | Required by CLI |
| `@nestjs/testing` (dev) | `^11.0` | `Test.createTestingModule()` |
| `@types/node` | `catalog:` | (already in workspace catalog) |
| `ts-node` (dev) | `^10.9` | TypeScript execution for scripts |
| `tsconfig-paths` (dev) | `^4.2` | Path alias resolution at runtime |
| `source-map-support` | `^0.5` | Source maps in production errors |
| `pino-pretty` (dev) | `^13.1` | (already in server dev deps) |

---

### 6.2 Packages Removed (api-server)

| Package | Reason |
|---|---|
| `express` | Replaced by `@nestjs/platform-express` (still uses Express internally) |
| `cors` | Replaced by NestJS CORS built-in (`app.enableCors()`) |
| `cookie-parser` | Replaced by NestJS cookie handling |
| `drizzle-orm` | Replaced by Prisma |
| `esbuild` | Replaced by `@nestjs/cli` (tsc) |
| `esbuild-plugin-pino` | No longer needed |
| `pino-http` | Replaced by `nestjs-pino` |
| `@types/express` | Still needed (platform-express uses it) — **keep** |
| `@types/cookie-parser` | Removed |
| `@types/cors` | Removed |
| `thread-stream` | Removed (pino internal, managed by nestjs-pino) |

---

### 6.3 Packages Removed (lib/db)

| Package | Reason |
|---|---|
| `drizzle-orm` | Replaced by `@prisma/client` |
| `drizzle-zod` | Replaced by manually written Zod schemas or `zod-prisma-types` |
| `pg` | Prisma manages its own connection pool |
| `drizzle-kit` (dev) | Replaced by Prisma CLI |

**Added to lib/db:**
| Package | Reason |
|---|---|
| `@prisma/client` | Thin re-export of generated Prisma types |

---

### 6.4 Packages Added (workspace catalog — pnpm-workspace.yaml)

These are version-pinned entries for new packages shared across multiple workspace packages:

| Package | Reason |
|---|---|
| `ioredis` | Shared version for consistency |
| `bullmq` | Shared version |

---

## 7. Database Migration Plan

### 7.1 Current State

The `lib/db/src/schema/index.ts` contains **zero table definitions**. The file is a template with examples commented out. There is no existing database schema, no existing tables, no existing data.

**Consequence: There is no data migration.** There is no data to move, transform, or preserve. This is a greenfield schema creation.

---

### 7.2 New Schema Source

The Prisma schema is derived entirely from:
1. **`lib/api-spec/openapi.yaml`** — defines all entity shapes, fields, types, and relationships
2. **`docs/architecture/ARCHITECTURE.md` Section 8** — adds infrastructure tables (RBAC, FeatureFlags, KnowledgeItems, AIBudget, ModerationReports, ImportJobs, ApiKeys, AuditLog)

No field in the OpenAPI spec is lost. Every response type in the spec maps to one or more Prisma models.

---

### 7.3 Schema Creation Process

```
Step 1: Write prisma/schema.prisma
        Based on openapi.yaml entity types + architecture extension tables.
        All models, relations, indexes, enums defined here.

Step 2: prisma migrate dev --name init
        Generates SQL migration file: prisma/migrations/0001_init.sql
        Applies migration to local dev database.
        This is the ONLY migration needed for a greenfield DB.

Step 3: prisma generate
        Generates @prisma/client TypeScript types.
        Types are then consumed by PrismaService + all Repositories.

Step 4: prisma db seed
        Runs seed/seed.ts which populates:
        - Categories (Fiction, Non-Fiction, Science, History, etc.)
        - Sample books (5-10 demo books for development)
        - Admin user (for local testing)
        - Default roles and permissions (reader, author, admin)
        - Default feature flags (all disabled)
```

---

### 7.4 OpenAPI → Prisma Type Mapping

| OpenAPI Type | Prisma Type |
|---|---|
| `type: integer` | `Int` |
| `type: string` | `String` |
| `type: string, format: date-time` | `DateTime` |
| `type: boolean` | `Boolean` |
| `type: number` | `Float` |
| `type: array, items: string` | `String[]` |
| `nullable: true` | `?` (optional) |
| `enum: [a, b, c]` | `enum EnumName { A B C }` |

---

### 7.5 Entity → Prisma Model Mapping (Key Entities)

| OpenAPI Entity | Prisma Model | Notable fields added |
|---|---|---|
| `User` | `User` | passwordHash, refreshTokenHash, emailVerified, twoFactorSecret |
| `Book` | `Book` | isPublished, publishedAt, slug, content |
| `Author` | `Author` (extends User via relation) | earnings, isVerified |
| `Chapter` | `Chapter` | content (Text), orderIndex |
| `Category` | `Category` | slug, coverUrl, bookCount (computed) |
| `Review` | `Review` | rating (1-5), content |
| `LibraryItem` | `LibraryItem` | status enum, progress (0-100), addedAt |
| `Bookmark` | `Bookmark` | chapterId (nullable), note |
| N/A (architecture) | `Role`, `Permission`, `RolePermission`, `UserRole` | RBAC |
| N/A (architecture) | `FeatureFlag` | Feature flags |
| N/A (architecture) | `KnowledgeItem` | Knowledge engine tracking |
| N/A (architecture) | `AIBudget` | Per-user AI token budget |
| N/A (architecture) | `ModerationReport` | Content moderation |
| N/A (architecture) | `ImportJob` | Import pipeline tracking |
| N/A (architecture) | `ApiKey` | Public API key auth |
| N/A (architecture) | `AuditLog` | Security audit trail |

---

### 7.6 Future Schema Changes (After Go-Live)

Once data exists in production, all schema changes must go through `prisma migrate dev` → review → `prisma migrate deploy`. The migration workflow is:

```
1. Developer changes schema.prisma locally
2. prisma migrate dev --name <description> → generates SQL
3. SQL migration reviewed in PR
4. Deployed via: prisma migrate deploy (runs in CI/CD, not manually)
5. Rollback: prisma migrate resolve --rolled-back <migration-name>
```

---

## 8. Rollback Plan

### 8.1 Context

**At the time of this migration, there is no production deployment.** The rollback plan describes what to do if a deployment fails at any point during or after the initial production launch.

---

### 8.2 Pre-Deployment Rollback (During Development)

**Trigger:** A migration step fails, or the new implementation is found incorrect before any production deployment.

**Action:** Use Replit Checkpoints. Every significant commit creates a checkpoint. Rolling back to a checkpoint restores both code and chat context.

No database rollback is needed during development — `prisma migrate dev` can be run on a fresh database at any time since there is no live data.

---

### 8.3 Deployment Rollback Strategy

Once GLU has its first production deployment, the rollback strategy operates at multiple levels:

#### Level 1 — Code Rollback (Fastest, < 5 minutes)
**What:** Deploy the previous Docker image tag.  
**When:** Application crashes immediately on startup or health check fails.  
**How:**
```bash
# On Replit: redeploy previous checkpoint
# On custom infrastructure:
docker service update --image glu-api:previous-tag api
```
**Limitation:** Does not roll back database schema changes.

#### Level 2 — Schema Rollback (< 30 minutes)
**What:** Revert the most recent Prisma migration.  
**When:** A migration introduced a breaking schema change and data is corrupt.  
**How:**
```bash
# Mark the migration as rolled back
prisma migrate resolve --rolled-back 20250724_some_migration

# Restore database from pre-migration snapshot
# Using pg_restore from the WAL-archive snapshot taken before the migration
pg_restore -d $DATABASE_URL backup_pre_migration.dump
```
**Prerequisite:** A database snapshot must be taken before every production migration (automated in CI/CD).

#### Level 3 — Point-in-Time Recovery (< 1 hour)
**What:** Restore the database to any second before the problem.  
**When:** Data corruption discovered after multiple migrations.  
**How:** Restore from WAL archive to the timestamp before the failed deployment:
```bash
# Restore PostgreSQL to T-minus-10-minutes using WAL archive in R2
restore-db --target-time "2025-07-24T14:55:00Z" --from-wal-archive r2://glu-backups/wal/
```

#### Level 4 — Full Environment Rollback (< 2 hours)
**What:** Restore entire environment from daily snapshot.  
**When:** Catastrophic failure affecting all components.  
**How:** Restore PostgreSQL from daily snapshot + revert code + restart all services.

---

### 8.4 Rollback Decision Matrix

| Scenario | Rollback Level | Time to Recovery |
|---|---|---|
| App won't start (missing env var, import error) | Level 1 (code) | 5 min |
| Health check fails (DB can't connect) | Level 1 (code) | 5 min |
| 5xx rate > 5% post-deploy | Level 1 (code) | 5 min |
| Data corruption in one table | Level 2 (schema + table restore) | 30 min |
| Migration broke multiple tables | Level 3 (PITR) | 60 min |
| Systemic failure across all components | Level 4 (full restore) | 2h |

---

### 8.5 Rollback Prevention Measures

These practices reduce the probability of needing a rollback:

1. **Blue-Green Deployment**: New version receives 0% traffic until health checks pass. No traffic switches until all indicators are green.
2. **Pre-migration snapshot**: Automated DB snapshot before every `prisma migrate deploy` in CI/CD.
3. **Staged rollout**: 5% → 25% → 50% → 100% traffic over 30 minutes. Rollback triggered automatically if error rate exceeds threshold.
4. **Schema migration dry-run**: Every migration runs with `--preview-feature` flag in staging first.
5. **Canary migration**: For destructive changes (column removal), maintain backward-compatible schema for one deploy cycle (old column kept, new added), then clean up in next deploy.

---

## 9. Testing Strategy

### 9.1 Testing Philosophy

Tests are not written after the code — they run alongside it. Every module written has a corresponding test file. The test suite gates every deployment.

**Test hierarchy:**
```
Unit Tests        — Fast, isolated, no I/O, mock everything external
Integration Tests — Test one module against real DB / real Redis in Docker
API Tests         — Test every endpoint against the OpenAPI spec using generated Zod validators
E2E Tests         — Test complete user journeys from HTTP request to DB state
Migration Tests   — Validate schema integrity, data constraints, seed correctness
```

---

### 9.2 Unit Tests

**Scope:** Business logic in Services and Repositories.  
**Tool:** Jest + `@nestjs/testing`  
**Speed:** < 10 seconds for full suite  
**What is mocked:** PrismaService, IEventBus (InMemoryEventBus), IStorageProvider, CacheService, AI providers  
**What is NOT mocked:** Domain logic, validation, mappers, business rules  

**Coverage targets:**
- Services: 90%+
- Repositories: 80%+ (happy path + error paths)
- Guards, pipes, filters: 100%
- AI agents: 70% (mocked provider responses)

**Example pattern:**
```typescript
// books.service.spec.ts
describe('BooksService', () => {
  let service: BooksService;
  let mockRepo: jest.Mocked<BooksRepository>;
  let mockEventBus: InMemoryEventBus;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        BooksService,
        { provide: BooksRepository, useValue: createMockRepository() },
        { provide: IEventBus, useClass: InMemoryEventBus },
      ],
    }).compile();

    service = module.get(BooksService);
    mockEventBus = module.get(IEventBus);
  });

  it('publishes BookPublishedEvent on book publish', async () => {
    await service.publishBook('book-id-1');
    expect(mockEventBus.published).toContainEqual(
      expect.objectContaining({ eventName: 'book.published' })
    );
  });
});
```

---

### 9.3 Integration Tests

**Scope:** Each module's Repository + Controller against a real database and Redis.  
**Tool:** Jest + Testcontainers (PostgreSQL + Redis Docker containers spun up per test suite)  
**Speed:** 30-120 seconds for full suite  
**What is real:** PostgreSQL (via Testcontainers), Redis (via Testcontainers), Prisma client  
**What is mocked:** External AI providers, Meilisearch, Qdrant, Storage  

**Pattern:**
```typescript
// Before all tests in a suite:
beforeAll(async () => {
  // Start PostgreSQL container
  // Run prisma migrate deploy
  // Run prisma db seed
  // Create NestJS test application
});

afterAll(async () => {
  // Stop containers
  // Close NestJS app
});

afterEach(async () => {
  // Truncate all tables (fast reset between tests)
});
```

**Coverage targets:**
- All repository CRUD operations
- All controller → service → repository flows
- Auth guard behavior (protected routes reject unauthenticated requests)
- Rate limiting behavior
- Cache hit/miss behavior

---

### 9.4 API Contract Tests

**Purpose:** Guarantee that every NestJS endpoint response matches the OpenAPI spec's Zod schema exactly. If the spec and the implementation drift, these tests fail.

**Tool:** Jest + Supertest + generated Zod validators from `lib/api-zod`  
**Speed:** 30-60 seconds  

**Pattern:**
```typescript
// contract/books.contract.spec.ts
import { GetBookResponse, ListBooksResponse } from '@workspace/api-zod';

it('GET /api/books/:id matches OpenAPI spec', async () => {
  const response = await request(app.getHttpServer())
    .get('/api/books/1')
    .set('Authorization', `Bearer ${testToken}`);

  expect(response.status).toBe(200);
  // This is the critical assertion — validates shape matches spec exactly
  expect(() => GetBookResponse.parse(response.body)).not.toThrow();
});
```

This test pattern covers **every endpoint in the OpenAPI spec**. It is the automated guardian of the API contract.

---

### 9.5 End-to-End Tests

**Scope:** Complete user journeys spanning multiple endpoints and DB state changes.  
**Tool:** Jest + Supertest against a fully-started NestJS application  
**Speed:** 60-180 seconds  
**Environment:** Full Docker stack (PostgreSQL, Redis, Meilisearch seeded with test data)  

**Critical journeys tested:**

| Journey | Steps |
|---|---|
| Full auth flow | Register → Login → Get /me → Update profile → Logout |
| Reading journey | Browse books → Add to library → Update progress → Mark finished |
| Review flow | Get book → Create review → Verify rating recalculated → Delete review |
| Bookmark flow | Get chapter → Create bookmark → List bookmarks → Delete |
| AI summary | Login → Summarize book → Verify response shape → Verify cost recorded |
| AI chat | Login → Start chat session → Multiple messages → Get history |
| Author follow | Login → List authors → Follow author → Verify in follower count |
| Admin moderation | Login as admin → Report content → Review moderation report |

---

### 9.6 Database Migration Tests

**Purpose:** Validate schema integrity, constraint correctness, and seed data quality.  
**When:** Run once after `prisma migrate dev` creates the initial migration, and again after every schema change.  

**Tests:**
```typescript
describe('Schema constraints', () => {
  it('cannot create a book without a category', async () => {
    await expect(prisma.book.create({
      data: { title: 'Test', authorId: 1 }  // missing categoryId
    })).rejects.toThrow();
  });

  it('cascade deletes library items when user is deleted', async () => {
    const user = await createTestUser();
    const book = await createTestBook();
    await prisma.libraryItem.create({ data: { userId: user.id, bookId: book.id, status: 'reading' } });
    await prisma.user.delete({ where: { id: user.id } });
    const items = await prisma.libraryItem.findMany({ where: { userId: user.id } });
    expect(items).toHaveLength(0);
  });

  it('seed data has correct category slugs', async () => {
    const categories = await prisma.category.findMany();
    for (const cat of categories) {
      expect(cat.slug).toMatch(/^[a-z0-9-]+$/);
    }
  });
});
```

---

### 9.7 Performance & Load Tests

**Scope:** Not blocking the migration, but planned for post-MVP.  
**Tool:** k6 or autocannon  
**Targets (MVP):**
- `GET /api/books` (cached): < 50ms p95
- `GET /api/books/:id` (cached): < 50ms p95
- `POST /api/ai/summarize`: < 5000ms p95 (AI call)
- `POST /api/ai/chat`: < 3000ms p95 first token (streaming)
- Auth endpoints: < 300ms p95 (argon2 is intentionally slow)

---

### 9.8 Security Tests

**What is verified before deployment:**

| Check | Method |
|---|---|
| No secrets in source code | Gitleaks scan in CI |
| No OWASP Top 10 vulnerabilities | SAST scan (Semgrep) |
| No known vulnerable dependencies | `pnpm audit` (fail on critical) |
| Rate limiting works correctly | Integration test: 101 requests → 429 |
| JWT cannot be forged | Unit test: tampered token rejected |
| Argon2 passwords not stored in plaintext | Integration test: DB row has no plaintext |
| SQL injection not possible | Prisma parameterized queries + integration test |
| Auth-protected routes return 401 without token | Contract test on every protected route |

---

### 9.9 CI/CD Gate Order

Every deployment must pass these gates in order:

```
1. pnpm typecheck         (TypeScript compile errors → fail immediately)
2. pnpm run test:unit     (Unit tests, < 10s)
3. pnpm run test:contract (API contract tests vs OpenAPI spec)
4. pnpm run test:e2e      (End-to-end journeys)
5. pnpm audit             (Dependency vulnerability scan)
6. semgrep --config auto  (SAST security scan)
7. prisma migrate deploy  (Apply pending migrations to production DB)
8. docker build + push    (Build and push image)
9. Deploy to 5% of traffic
10. Monitor error rate for 5 minutes → if < 1%: ramp to 100%
11. If error rate > 1% at any point: automatic rollback (Level 1)
```

---

*This document is complete. No implementation begins until this report is approved.*
