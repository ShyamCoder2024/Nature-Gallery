# 05 — Architecture & Engineering Design

Version 1.0 · Status: Draft for review

This is the complete engineering design. It describes how every requirement in [02-prd.md](./02-prd.md) is built: the system's parts, the tech stack, the code layout, multi-tenancy, the full database schema, the API, menu caching, the order pipeline, realtime, performance and offline behaviour, error handling, security and roles, the insights engine, the payment plug-in point, and the build phases.

## Contents

1. [Architecture principles](#1-architecture-principles)
2. [System overview](#2-system-overview)
3. [Tech stack](#3-tech-stack)
4. [Code structure](#4-code-structure)
5. [Multi-tenancy and tenant isolation](#5-multi-tenancy-and-tenant-isolation)
6. [Backend modules](#6-backend-modules)
7. [Database design](#7-database-design)
8. [API design](#8-api-design)
9. [Menu delivery and caching](#9-menu-delivery-and-caching)
10. [Order pipeline](#10-order-pipeline)
11. [Realtime](#11-realtime)
12. [Performance, slow networks, and offline](#12-performance-slow-networks-and-offline)
13. [Error handling](#13-error-handling)
14. [Security, access control, and privacy](#14-security-access-control-and-privacy)
15. [Insights engine](#15-insights-engine)
16. [Payment plug-in point](#16-payment-plug-in-point)
17. [Background jobs and notifications](#17-background-jobs-and-notifications)
18. [Monitoring](#18-monitoring)
19. [Scaling and capacity](#19-scaling-and-capacity)
20. [Build phases](#20-build-phases)

---

## 1. Architecture principles

| # | Principle | Consequence |
|---|-----------|-------------|
| 1 | **The server is the source of truth; the client is fast.** | Clients render optimistically and reconcile with the server. Prices, bills, and statuses are always decided by the server. |
| 2 | **Exactly-once orders.** | Every write that matters carries an idempotency key; orders are written in one transaction; events leave through a transactional outbox. |
| 3 | **Read path is cached, write path is small.** | Guest menu reads come from the CDN (immutable snapshots). Writes (orders, requests) are tiny JSON payloads. |
| 4 | **Realtime is an optimisation, not a dependency.** | Every screen can rebuild its full state from REST. Sockets only make it faster. |
| 5 | **Tenant isolation in two layers.** | Application checks plus PostgreSQL Row-Level Security. A bug in one layer does not leak data. |
| 6 | **Shared business logic, one implementation.** | Bill calculation, menu availability, and state machines live in one TypeScript package used by the server and the clients. |
| 7 | **Modular monolith.** | One deployable API with strict module boundaries. Simple to run; can be split later without rewriting. |
| 8 | **Configuration, not code, for business choices.** | Plans, features, limits, taxes, menus, roles, and texts are data. No developer needed for a customer change. |
| 9 | **Boring, proven technology.** | PostgreSQL, Redis, Node.js, React. Nothing experimental on the critical path. |
| 10 | **India-first defaults.** | INR, GST, +91, Asia/Kolkata, English + Hindi (D-02, D-11). |

---

## 2. System overview

### 2.1 Context diagram

```
                         ┌──────────────────────────────────────────────┐
  Guest phone ──QR──▶    │ Cloudflare (CDN, TLS, WAF, edge cache)       │
  (browser)              │  hotl.in/q/*   → Guest app (SSR)             │
                         │  hotl.in/api/* → API  (same origin, no CORS) │
                         │  img.hotl.in/* → Image storage (immutable)   │
  Staff / KDS / Owner ──▶│  app.<brand>/*  → Admin app                  │
  Founder ──────────────▶│  ops.<brand>/*  → Platform admin app         │
                         └───────────────┬──────────────────────────────┘
                                         │
              ┌──────────────────────────┼─────────────────────────────┐
              ▼                          ▼                             ▼
     ┌────────────────┐        ┌──────────────────┐         ┌──────────────────┐
     │  API (Fastify) │◀──────▶│ Realtime gateway │         │   Worker         │
     │  modular       │  Redis │ (Socket.IO)      │         │ (BullMQ jobs)    │
     │  monolith      │ pub/sub└──────────────────┘         │ images, insights │
     └──────┬─────────┘                                     │ SMS, exports,    │
            │                                               │ outbox publisher │
            │                                               └──────┬───────────┘
   ┌────────┴────────┬───────────────────┬─────────────────────────┘
   ▼                 ▼                   ▼
┌──────────┐   ┌───────────┐   ┌──────────────────┐   External: SMS provider (OTP),
│PostgreSQL│   │  Redis    │   │ Object storage   │   payment gateway plug-in (Phase 8),
│(primary +│   │ cache,    │   │ (Cloudflare R2)  │   web push (P1)
│ replica) │   │ queues,   │   │ photos, exports  │
└──────────┘   │ OTP, rate │   └──────────────────┘
               │ limits    │
               └───────────┘
```

### 2.2 Components

| Component | Responsibility | Runs as |
|-----------|----------------|---------|
| **Guest app** (`apps/guest`) | QR entry, menu, cart, ordering, status, bill, identity, Hindi switch. Server-rendered for instant first paint. | Next.js server + CDN |
| **Admin app** (`apps/admin`) | Owner/manager/cashier/waiter screens, KDS screens (`/kds`), staff quick switch. | Next.js |
| **Platform admin app** (`apps/platform`) | Founder's panel: tenants, plan builder, support access, health. Separate app and domain so a bug in the business admin can never expose platform powers. | Next.js |
| **API** (`apps/api`) | All business logic and data access. REST over HTTPS. | Node.js (Fastify), stateless, horizontally scaled |
| **Realtime gateway** (`apps/realtime`) | Holds WebSocket connections; authenticates; joins rooms; relays events from Redis. No business logic. | Node.js (Socket.IO), horizontally scaled |
| **Worker** (`apps/worker`) | Background jobs: image processing, menu snapshot builds, outbox publishing, insights, SMS, exports, timers. | Node.js (BullMQ) |
| **PostgreSQL** | System of record. | Managed, primary + read replica |
| **Redis** | Cache, sessions, OTP codes, rate limits, job queues, realtime pub/sub. | Managed |
| **Object storage** | Original and resized photos, QR card PDFs, report exports. | Cloudflare R2 (S3 API) |
| **CDN / edge** | TLS, caching of HTML shells, snapshots, images; WAF; bot protection; HTTP/3. | Cloudflare |

### 2.3 Domains

| Domain | Purpose |
|--------|---------|
| `hotl.in` (placeholder, Q-01) | Short QR links and the guest app. Short domain keeps QR codes small and easy to scan. `/api/*` on the same domain routes to the API so the guest app makes **same-origin** requests (no CORS preflight — saves one round trip per request on slow networks). |
| `img.hotl.in` | Images (immutable URLs, long cache). |
| `app.<brand>.com` | Admin app; `/api/*` routes to the API (same-origin). |
| `ops.<brand>.com` | Platform admin. |
| `rt.<brand>.com` | Realtime gateway (WebSocket). |

### 2.4 Request flow at a glance

1. **Guest scans QR** → edge serves cached HTML for `/q/{token}` (≤ 30 s old) → browser paints menu → JavaScript hydrates → loads immutable menu snapshot JSON (usually from browser/edge cache).
2. **Guest places order** → `POST /api/v1/guest/orders` with idempotency key → API validates, writes order + outbox event in one transaction → returns 201 → worker publishes event to Redis → realtime gateway pushes to staff board and KDS → guest gets status updates.
3. **Admin edits a price** → `PATCH /api/v1/admin/items/{id}` → change saved to draft → publish → worker builds a new snapshot version → pointer updates → guests see it within 60 s.

---

## 3. Tech stack

| Layer | Choice | Version policy | Why |
|-------|--------|----------------|-----|
| Language | **TypeScript** (strict) | Latest stable | One language for all code; shared types. |
| Runtime | **Node.js** | Active LTS (22.x) | Mature, fast I/O, huge ecosystem. |
| Monorepo | **pnpm** workspaces + **Turborepo** | Latest | Fast installs, task caching, shared packages. |
| Web framework | **Next.js** (App Router, React Server Components) | Latest stable | Server rendering + streaming for fast first paint; same stack for all three web apps. |
| UI | **React** 19, **Tailwind CSS** v4 with design tokens as CSS variables, **Radix UI** primitives (admin), custom lightweight primitives (guest) | Latest | Accessible primitives; tokens from doc 03. Guest app avoids heavy UI libraries. |
| Animation | **Motion** (`motion/react`, `LazyMotion` + `domAnimation`), CSS transitions, View Transitions API | Latest | Springs and gestures with a small footprint (doc 03 §6). |
| Client data (admin) | **TanStack Query** | v5 | Caching, background refresh, optimistic updates, retries. |
| Client storage (guest) | **IndexedDB** via `idb-keyval` | Latest | Cart and order outbox survive reloads and offline. |
| Service worker | **Serwist** (Workbox-based) | Latest | Precache app shell, runtime caching of snapshots and images. |
| Forms & validation | **Zod** + React Hook Form (admin) | Latest | One schema for client and server. |
| API server | **Fastify** | v5 | Fast, schema-driven, plugin architecture. |
| ORM / SQL | **Drizzle ORM** + SQL migrations (drizzle-kit) | Latest | Type-safe, close to SQL, no heavy runtime. |
| Database | **PostgreSQL** | 16+ | Relational integrity, RLS, JSONB, partitioning. |
| Connection pooling | **PgBouncer** (transaction mode) or provider pooler | — | Many API instances, few DB connections. |
| Cache / queues | **Redis** 7 + **BullMQ** | Latest | Queues with retries, delays, rate limits. |
| Realtime | **Socket.IO** v4 + `@socket.io/redis-adapter` | Latest | WebSocket with long-polling fallback, rooms, reconnection. |
| Images | **sharp** (libvips) | Latest | Fast resizing to AVIF/WebP; HEIC input. |
| Placeholders | **ThumbHash** | Latest | ~25-byte blurred previews inlined in menu data. |
| Search (guest) | **Fuse.js**-style fuzzy search on the client over the snapshot (or a tiny custom trigram matcher) | — | Works offline, no server search needed. |
| Charts (admin) | **visx** or **Recharts** (lazy-loaded) | Latest | Flexible, themable. |
| PDF (QR cards, bills, reports) | **pdf-lib** in worker; QR via **qrcode** lib | Latest | Server-side, print-ready output. |
| Spreadsheets | **ExcelJS** (worker) | Latest | Import/export menus and reports. |
| i18n | **next-intl** with ICU messages; `en` and `hi` catalogues in `packages/i18n` | Latest | Plurals, numbers, dates in Indian formats. |
| SMS / OTP | **MSG91** (DLT-registered templates) behind an `SmsProvider` interface | — | Indian delivery, Web OTP friendly format. |
| Errors & tracing | **Sentry** (frontend + backend), **OpenTelemetry** traces/metrics | — | See §18. |
| Logging | **pino** (JSON) | Latest | Fast structured logs. |
| Feature flags | Own implementation (plan features in DB, §7.3) | — | Plans are business data (D-13). |

**Libraries deliberately avoided on the guest app**: moment.js, lodash (full), large UI kits, client-side state libraries beyond React context + `useSyncExternalStore`, analytics SDKs that block rendering.

---

## 4. Code structure

### 4.1 Repository layout

```
/
├── apps/
│   ├── guest/                 Next.js guest web app
│   ├── admin/                 Next.js admin + staff + KDS app
│   ├── platform/              Next.js founder/platform admin app
│   ├── api/                   Fastify API (modular monolith)
│   ├── realtime/              Socket.IO gateway
│   └── worker/                BullMQ job processors
├── packages/
│   ├── contracts/             Zod schemas for every request/response, error codes, realtime event types
│   ├── domain/                Pure business logic shared by server and clients:
│   │                            billing calculator, menu availability resolver,
│   │                            state machines, business-day and time helpers, money
│   ├── db/                    Drizzle schema, migrations, seed data, RLS policies
│   ├── sdk/                   Typed API client generated from contracts (fetch-based, retry + idempotency built in)
│   ├── ui/                    Shared React components (admin + platform)
│   ├── guest-ui/              Lightweight guest components (no heavy deps)
│   ├── design-tokens/         Tokens (doc 03 §2) → CSS variables + TS constants; theme presets; contrast algorithm
│   ├── i18n/                  en + hi message catalogues, formatters (INR, dates)
│   └── config/                tsconfig, eslint, prettier presets
├── docs/                      These documents
└── turbo.json, pnpm-workspace.yaml, package.json
```

### 4.2 API module layout

```
apps/api/src/
├── main.ts                    Bootstrap, plugins, graceful shutdown
├── plugins/                   auth, tenant-context, rate-limit, idempotency, error-handler, request-id, rls
├── modules/
│   └── <module>/
│       ├── routes.ts          HTTP routes (thin: parse → authorize → call service → map response)
│       ├── service.ts         Use cases; transactions; emits domain events
│       ├── repository.ts      SQL via Drizzle; only this module's tables
│       ├── policies.ts        Permission rules for this module
│       ├── events.ts          Domain events this module emits
│       └── __tests__/
└── shared/                    db client, redis, clock, ids, logger, errors
```

**Module rules**
1. A module only reads/writes its own tables. To use another module's data it calls that module's service interface.
2. Cross-module side effects go through domain events (outbox) — e.g. `billing` listens to `order.placed` to update the running bill.
3. Routes never contain business logic; services never know about HTTP.
4. All time comes from an injectable `Clock` (testable business-day logic).

### 4.3 Coding conventions (essentials)

| Topic | Rule |
|-------|------|
| IDs | **UUIDv7** primary keys (time-sortable, index-friendly). Human-facing numbers (order #27, bill `BND/25-26/000123`) are separate columns. QR tokens: 12-character base-62 random strings (≈ 71 bits). |
| Money | Integers in paise; field names end in `Paise` (JSON) / `_paise` (SQL). Rates in **basis points** (`rate_bp`, 500 = 5 %). All arithmetic via `packages/domain/money`. |
| Time | Stored as `timestamptz` (UTC). Outlet-local times (hours, schedules) stored as `time` + outlet `time_zone`. Business dates as `date`. |
| JSON | camelCase in API; snake_case in SQL; mapping in repositories. |
| Naming | Tables plural snake_case; enums as `text` with `CHECK` constraints (easier migrations than Postgres enums). |
| Errors | Throw typed `AppError(code, details)` from services; the error plugin maps to HTTP (§13). |
| Strings | No user-facing string in code; always an i18n key. |
| Commits | Conventional commits (`feat(ordering): …`), referencing requirement IDs (`GST-ORD-002`). |

---

## 5. Multi-tenancy and tenant isolation

### 5.1 Model

**Shared database, shared schema, `tenant_id` on every tenant-owned row.** This is the simplest model to operate for thousands of small tenants (one migration run, one connection pool, cheap per tenant) and matches D-01.

Hierarchy: `tenant → outlets → areas → spots`. Menus, staff, customers, and settings hang off the tenant; operational data (orders, sessions, bills) also carries `outlet_id`.

### 5.2 Two layers of isolation

**Layer 1 — application**
- Every request resolves a **tenant context** before reaching a route handler:
  - *Admin/staff*: from the authenticated membership (user × tenant) plus the selected outlet (header `X-Outlet-Id`, validated against the membership's outlets).
  - *Guest*: from the QR token or from the session/order the request targets (resolved server-side, never trusted from the client).
  - *KDS device*: from the device token (bound to tenant + outlet + station).
  - *Platform*: no tenant context unless entering a tenant through an audited support session.
- Repositories require `tenantId` as a parameter for every query (enforced by a typed query helper; code review rejects raw queries without it).

**Layer 2 — database Row-Level Security**
- Every tenant-owned table has RLS enabled with:
  ```sql
  ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
  ALTER TABLE orders FORCE ROW LEVEL SECURITY;
  CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id', true)::uuid)
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);
  ```
- Each request runs inside a transaction that first executes `SET LOCAL app.tenant_id = '<uuid>'` (transaction-scoped, safe with PgBouncer transaction pooling).
- The API connects as role `app_rw`, which **does not** have `BYPASSRLS`. If `app.tenant_id` is missing, `current_setting(..., true)` is NULL and no rows match — fail closed.
- Platform-wide jobs (e.g. nightly insights over all tenants) iterate tenant by tenant, setting the context each time. Only migrations and the platform module's explicit cross-tenant queries use role `app_platform` (BYPASSRLS), and those queries are listed in one file and audit-logged.

**Other stores**
- Redis keys are prefixed `t:{tenantId}:` (e.g. `t:…:availability:{outletId}`).
- Object storage paths: `tenants/{tenantId}/media/{assetId}/…`, `tenants/{tenantId}/exports/…`. Export download links are short-lived signed URLs.
- Realtime rooms include the outlet ID and are joined only after the gateway verifies the token's tenant/outlet.

**Mandatory automated check**: the API test suite includes a cross-tenant check that creates two tenants and asserts every endpoint returns `404` (not `403`, to avoid confirming existence) for the other tenant's IDs.

### 5.3 Noisy-neighbour protection

- Per-tenant and per-outlet rate limits (§14.6).
- Heavy work (exports, imports, insights) runs in the worker with per-tenant concurrency limits.
- Reports read from the replica and from pre-aggregated tables (§15), never by scanning raw orders on the primary.

### 5.4 Plan features and limits at runtime

- `FeatureService.isEnabled(tenantId, 'kds')` resolves: tenant override → plan feature → `false`. Cached in Redis for 60 s, invalidated on change.
- `LimitService.check(tenantId, 'max_spots_per_outlet', currentCount + n)` throws `PLAN_LIMIT_REACHED` (§13.3).
- Route plugins declare `requireFeature('coupons')`; the admin app reads `/admin/me` → `features` and `limits` to hide or lock UI.

---

## 6. Backend modules

| Module | Owns (tables) | Main responsibilities | Emits events |
|--------|---------------|-----------------------|--------------|
| `identity` | users, user_sessions, memberships, membership_outlets, roles, role_permissions, invites, devices | Staff login, sessions, PIN quick switch, roles & permissions, device pairing | `staff.deactivated`, `device.revoked` |
| `platform` | plans, plan_features, plan_limits, tenants, tenant_feature_overrides, tenant_limit_overrides, platform_users, support_sessions, platform_audit_log | Tenant lifecycle, plan builder, support access | `tenant.created`, `tenant.suspended`, `tenant.plan_changed` |
| `tenancy` | tenant_settings, tenant_branding, outlets, outlet_hours, outlet_closures | Business profile, branding, outlets, hours, ordering rules | `outlet.updated`, `branding.updated` |
| `spots` | areas, spots, qr_codes | Areas, spots, QR tokens, QR card PDFs, floor state | `spot.updated`, `qr.regenerated` |
| `catalog` | menus, menu_schedules, menu_assignments, categories, menu_categories, items, item_media, item_variants, modifier_groups, modifier_options, item_modifier_groups, tags, item_tags, outlet_item_overrides, outlet_option_overrides, stations, tax_groups, taxes, tax_group_taxes, menu_snapshots, catalog_changes | Menu editing, drafts, publish, snapshots, availability overlay | `menu.published`, `availability.changed` |
| `media` | media_assets | Upload URLs, processing status, variants | `media.ready` |
| `ordering` | table_sessions, session_spots, orders, order_items, order_item_modifiers, order_events, outlet_counters, guest_carts | Cart validation, order placement, status changes, cancellations, sessions, move/merge | `order.placed`, `order.status_changed`, `order.item_cancelled`, `session.opened`, `session.moved`, `session.merged`, `session.closed` |
| `billing` | bills, bill_lines (view), bill_taxes, discounts, payments, bill_number_sequences, day_closes | Running bill, discounts, service charge, payments, settlement, voids, day close | `bill.updated`, `bill.requested`, `bill.settled`, `bill.voided`, `payment.recorded` |
| `customers` | customers, guest_devices, device_customer_links, customer_visits, customer_item_stats, consent_records, customer_notes, data_requests | OTP identity, recognition, history, reorder, consent, deletion | `customer.verified`, `customer.deleted` |
| `service` | service_request_types, service_requests | Call waiter, request bill, custom requests | `service_request.created`, `service_request.updated` |
| `feedback` (P1) | feedback, feedback_items | Ratings and comments | `feedback.low_rating` |
| `promotions` (P1) | coupons, coupon_redemptions | Coupons and automatic offers | — |
| `insights` | sales_* aggregate tables, insights | Aggregation, metrics, insight rules, reports, exports | `insight.created` |
| `notifications` | push_subscriptions (P1), notification_log | Staff notifications, SMS sending, web push | — |
| `audit` | audit_log | Append-only audit of sensitive actions | — |
| `infra` | outbox_events, idempotency_keys | Outbox, idempotency storage | — |

---

## 7. Database design

### 7.1 Conventions

- Every table has `id uuid primary key` (UUIDv7, generated in the app), `created_at timestamptz not null default now()`, `updated_at timestamptz not null default now()` (trigger-maintained) unless noted.
- Every tenant-owned table has `tenant_id uuid not null references tenants(id)` and RLS (§5.2). Composite indexes start with `tenant_id`.
- **Configuration entities** (menus, items, spots, staff…) use soft delete: `deleted_at timestamptz null`. Unique constraints are partial `WHERE deleted_at IS NULL`.
- **Financial and operational records** (orders, bills, payments, audit) are **never deleted or soft-deleted**; corrections are new rows (cancellations, voids, reversals).
- Mutable aggregates that can be edited concurrently (orders, bills, sessions, items) have `version int not null default 1`, incremented on each update; updates use `WHERE id = $1 AND version = $2` (optimistic locking, §13.5).
- Translatable text: English column (required) + `_hi` column (optional), e.g. `name`, `name_hi` (D-11).
- Money: `bigint` paise, `CHECK (x >= 0)` where applicable. Rates: `int` basis points.
- Status columns: `text` + `CHECK (status IN (...))`.

### 7.2 Entity relationship overview

```
plans ─┬─< plan_features            tenants ─┬─< outlets ─┬─< areas ─< spots ─< qr_codes
       └─< plan_limits                       │            ├─< outlet_hours / outlet_closures
tenants >── plans                            │            ├─< outlet_item_overrides >── items
                                             │            ├─< table_sessions ─┬─< session_spots >── spots
users ─< memberships >── tenants             │            │                   ├─< orders ─< order_items ─< order_item_modifiers
memberships >── roles ─< role_permissions    │            │                   │            └─< order_events
memberships ─< membership_outlets >── outlets│            │                   └── bills ─┬─< bill_taxes
                                             │            │                              ├─< discounts
tenants ─< menus ─< menu_categories >── categories        │                              └─< payments
menus ─< menu_schedules, menu_assignments                 ├─< service_requests
categories ─< items ─┬─< item_variants                    └─< menu_snapshots
                     ├─< item_media >── media_assets
                     ├─< item_modifier_groups >── modifier_groups ─< modifier_options
                     └─< item_tags >── tags
tenants ─< customers ─┬─< customer_visits, customer_item_stats, consent_records, customer_notes
guest_devices ─< device_customer_links >── customers
```

### 7.3 Platform tables

**plans**
| Column | Type | Notes |
|--------|------|-------|
| id | uuid | |
| name | text | e.g. "Café Basic" — defined by founder (D-13) |
| description | text null | |
| price_paise | bigint | Price per billing period |
| billing_period | text | `monthly` \| `yearly` |
| per_outlet | boolean | Price multiplies by outlet count |
| status | text | `active` \| `retired` (retired plans stay on existing tenants, can't be newly assigned) |
| sort_order | int | |

**plan_features** — `plan_id`, `feature_key text` (from doc 02 §9 list), `enabled boolean`; PK (`plan_id`, `feature_key`).
**plan_limits** — `plan_id`, `limit_key text`, `value int` (−1 = unlimited); PK (`plan_id`, `limit_key`).

**tenants**
| Column | Type | Notes |
|--------|------|-------|
| id | uuid | |
| display_name | text | |
| legal_name | text null | |
| slug | text unique | Internal reference, not used in guest URLs |
| plan_id | uuid → plans | |
| status | text | `trial` \| `active` \| `past_due` \| `suspended` \| `cancelled` \| `pending_deletion` |
| trial_ends_at | timestamptz null | |
| suspended_reason | text null | |
| deletion_scheduled_at | timestamptz null | 30-day grace (PLT-009) |
| gstin | text null | Validated format `^\d{2}[A-Z]{5}\d{4}[A-Z][1-9A-Z]Z[0-9A-Z]$` |
| fssai_number | text null | 14 digits |
| billing_contact | jsonb | name, phone, email for subscription invoices |

**tenant_feature_overrides** — `tenant_id`, `feature_key`, `enabled`, `reason`, `set_by`, `expires_at null`; PK (`tenant_id`, `feature_key`).
**tenant_limit_overrides** — same shape with `value int`.
**platform_users** — founder/support accounts: `email unique`, `name`, `password_hash`, `totp_secret_enc`, `role` (`super_admin` \| `support`), `status`.
**support_sessions** — `tenant_id`, `platform_user_id`, `mode` (`read_only` \| `edit`), `granted_by_membership_id null` (owner grant for edit), `starts_at`, `ends_at`, `reason`.
**platform_audit_log** — same shape as `audit_log` (§7.12), no tenant RLS.
**subscription_invoices** (P1) — `tenant_id`, `period_start`, `period_end`, `amount_paise`, `tax_paise`, `status` (`draft` \| `sent` \| `paid` \| `void`), `pdf_key`, `paid_at`, `reference`.

### 7.4 Identity tables

**users** (staff identity, global across tenants — one person can work for two businesses)
| Column | Type | Notes |
|--------|------|-------|
| id | uuid | |
| name | text | |
| email | citext null unique | |
| phone_e164 | text null unique | `+91XXXXXXXXXX` |
| password_hash | text null | argon2id; null if OTP-only login |
| status | text | `active` \| `disabled` |
| last_login_at | timestamptz null | |
| locale | text | `en` \| `hi` (ADM-SET-011) |
| CHECK | | email or phone must be present |

**memberships** — `tenant_id`, `user_id`, `role_id`, `status` (`invited` \| `active` \| `deactivated`), `pin_hash null` (argon2id of 4–6 digit quick-switch PIN), `all_outlets boolean`, `invited_by`, `deactivated_at`; unique (`tenant_id`, `user_id`).
**membership_outlets** — `membership_id`, `outlet_id`; PK both.
**roles** — `tenant_id null` (null = built-in template), `key` (`owner`, `manager`, `cashier`, `waiter`, `kitchen`, or custom slug), `name`, `is_system boolean`, `limits jsonb` (e.g. `{ "max_discount_bp": 1000 }`).
**role_permissions** — `role_id`, `permission text` (§14.3 keys); PK both.
**invites** — `tenant_id`, `email/phone`, `role_id`, `outlet_ids uuid[]`, `token_hash`, `expires_at` (7 days), `accepted_at`.
**user_sessions** — stored in Redis (`sess:{id}` → user, membership, device, expiry); a table `user_session_log` keeps login history (ip, user agent, at) for the audit screen.
**devices** — `tenant_id`, `outlet_id`, `type` (`kds` \| `counter`), `name`, `station_id null`, `token_hash`, `paired_by`, `paired_at`, `last_seen_at`, `revoked_at`.

### 7.5 Tenant setup tables

**tenant_settings** (1 row per tenant)
| Column | Type | Default |
|--------|------|---------|
| price_display | text | `inclusive` \| `exclusive` (default `exclusive`) |
| rounding | text | `none` \| `nearest_1` \| `nearest_0_50` (default `nearest_1`) |
| service_charge_enabled | boolean | false |
| service_charge_bp | int | 500 |
| service_charge_label | text | "Service charge" |
| service_charge_scope | text | `dine_in` \| `all` |
| acceptance_mode | text | `auto` \| `manual_all` \| `manual_first` (default `manual_first`, D-14) |
| require_identity | boolean | true |
| require_otp | boolean | true (D-05) |
| allow_anonymous_orders | boolean | false |
| browse_when_closed | boolean | true |
| max_qty_per_line | int | 20 |
| max_items_per_order | int | 50 |
| session_idle_timeout_min | int | 240 |
| guest_cancel_window | text | `until_preparing` |
| hindi_enabled | boolean | true |
| sold_out_publish_instant | boolean | true |
| order_sound | text | `chime` |
| privacy_notice_md | text | Default template |
| wifi_name / wifi_password | text null | |

**tenant_branding** — `tenant_id` PK, `theme_preset` (`ivory`\|`noir`\|`garden`\|`coastal`), `accent_hex`, `accent_adjusted_hex` (after contrast check), `on_accent_hex`, `font_pairing` (`classic`\|`modern`\|`elegant`\|`friendly`\|`geometric`), `logo_square_media_id`, `logo_wide_media_id`, `cover_media_id`, `welcome_text`, `welcome_text_hi`, `about_text`, `about_text_hi`, `social_links jsonb`.

**outlets**
| Column | Type | Notes |
|--------|------|-------|
| tenant_id | uuid | |
| name | text | |
| code | text | 3-letter code used in bill number prefix (`BND`); unique per tenant |
| address | jsonb | line1, line2, city, state, pincode; `state_code` for GST |
| geo | point null | For map link |
| phone | text null | |
| time_zone | text | `Asia/Kolkata` |
| currency | char(3) | `INR` |
| business_day_cutoff | time | `04:00` |
| status | text | `setup` \| `test` \| `live` \| `paused` \| `closed` |
| ordering_paused_until | timestamptz null | ADM-ORD-009 |
| ordering_paused_message | text null | |
| default_tax_group_id | uuid null | |
| gstin_override | text null | If outlets have different GST registrations |

**outlet_hours** — `outlet_id`, `weekday smallint` (0–6), `opens time`, `closes time` (may be < opens = past midnight), multiple rows per day allowed.
**outlet_closures** — `outlet_id`, `starts_at`, `ends_at`, `reason` (holidays, special closures).

**areas** — `tenant_id`, `outlet_id`, `name`, `name_hi`, `sort_order`, `deleted_at`.

**spots**
| Column | Type | Notes |
|--------|------|-------|
| tenant_id, outlet_id | uuid | |
| area_id | uuid null | |
| type | text | `table` \| `room` \| `counter` \| `other` |
| label | text | "Table 12", "Room 304"; unique per outlet (partial, not deleted) |
| label_hi | text null | |
| capacity | smallint null | |
| status | text | `active` \| `disabled` |
| layout | jsonb null | x, y, w, h, shape for floor plan (P1) |
| sort_order | int | |
| deleted_at | timestamptz null | |

**qr_codes**
| Column | Type | Notes |
|--------|------|-------|
| tenant_id, outlet_id | uuid | |
| spot_id | uuid null | null = outlet-level QR (GST-QR-006) |
| token | text unique | 12-char base-62; **globally unique index** (looked up without tenant context) |
| status | text | `active` \| `revoked` |
| revoked_at | timestamptz null | |
| scan_count | bigint | Incremented asynchronously from edge logs/beacon (ADM-TBL-009) |

Token lookup happens before the tenant is known, so `qr_codes` lookup uses a dedicated `SECURITY DEFINER` function `resolve_qr(token)` returning only `tenant_id, outlet_id, spot_id, status` — the one deliberate RLS exception on the guest path.

### 7.6 Catalog tables

**menus** — `tenant_id`, `name`, `name_hi`, `description`, `status` (`active`\|`archived`), `sort_order`, `deleted_at`.
**menu_schedules** — `menu_id`, `weekdays smallint[]`, `starts time`, `ends time`; no rows = always available.
**menu_assignments** — `menu_id`, `outlet_id`, `area_id null`, `spot_type null`; a menu is visible at a spot if any assignment matches (outlet, and area/spot type when set).
**categories** — `tenant_id`, `name`, `name_hi`, `description`, `description_hi`, `image_media_id null`, `layout` (`photo_list`\|`photo_grid`\|`text_list`), `is_hidden`, `deleted_at`.
**menu_categories** — `menu_id`, `category_id`, `sort_order`; PK (`menu_id`, `category_id`). A category can be in several menus.

**items**
| Column | Type | Notes |
|--------|------|-------|
| tenant_id | uuid | |
| category_id | uuid | Primary category |
| name / name_hi | text | Required English; optional Hindi |
| description / description_hi | text null | Limited markdown (bold, italic, line breaks), max 500 chars |
| dietary | text | `veg` \| `non_veg` \| `egg` \| `vegan` |
| spice_level | smallint | 0–3 |
| allergens | text[] | from fixed list: gluten, dairy, nuts, peanuts, soy, egg, fish, shellfish, sesame, mustard |
| badges | text[] | `bestseller`, `chefs_pick`, `new` (max 2 shown) |
| prep_time_min | smallint null | |
| station_id | uuid null | Kitchen station |
| tax_group_id | uuid null | Falls back to category → outlet default |
| allow_instructions | boolean | default true |
| max_qty | smallint null | Overrides tenant default |
| sku | text null | |
| is_hidden | boolean | |
| sort_order | int | Within category |
| available_from / available_to | time null | Item-level time window |
| deleted_at | timestamptz null | |

**item_variants** — `item_id`, `name`, `name_hi`, `price_paise`, `is_default`, `sku`, `sort_order`, `deleted_at`. Every item has ≥ 1 variant; single-price items have one default variant with empty name.
**item_media** — `item_id`, `media_id`, `sort_order` (0 = cover), max 5.
**modifier_groups** — `tenant_id`, `name`, `name_hi`, `selection` (`single`\|`multiple`), `min_select`, `max_select`, `allow_option_qty boolean`, `deleted_at`; `CHECK (min_select <= max_select)`.
**modifier_options** — `group_id`, `name`, `name_hi`, `price_paise` (≥ 0), `is_default`, `dietary null`, `sort_order`, `linked_item_id null` (combos, GST-ITEM-008), `deleted_at`.
**item_modifier_groups** — `item_id`, `group_id`, `sort_order`, `min_override null`, `max_override null`.
**tags** — `tenant_id`, `name`, `name_hi`, `kind` (`filter` e.g. Jain/Gluten free, `info`), `icon`.
**item_tags** — `item_id`, `tag_id`.
**stations** — `tenant_id`, `outlet_id`, `name` ("Kitchen", "Bar"), `sort_order`.

**outlet_item_overrides**
| Column | Type | Notes |
|--------|------|-------|
| tenant_id, outlet_id | uuid | |
| item_id | uuid | |
| variant_id | uuid null | null = whole item |
| price_paise | bigint null | Outlet price |
| is_hidden | boolean null | |
| sold_out | boolean | default false |
| sold_out_until | timestamptz null | null with sold_out = until turned back on; set to next business-day cutoff for "end of day" |
| updated_by | uuid | |
| unique | | (`outlet_id`, `item_id`, `variant_id`) (NULLS NOT DISTINCT) |

**outlet_option_overrides** — same idea for `modifier_option_id` (sold out per outlet).

**tax_groups** — `tenant_id`, `name` ("GST 5 %"), `is_default`.
**taxes** — `tenant_id`, `name` ("CGST"), `rate_bp` (250), `code` (`cgst`\|`sgst`\|`igst`\|`cess`\|`other`).
**tax_group_taxes** — `tax_group_id`, `tax_id`.

**menu_snapshots**
| Column | Type | Notes |
|--------|------|-------|
| tenant_id, outlet_id | uuid | |
| version | int | Monotonic per outlet; unique (`outlet_id`, `version`) |
| content | jsonb | Full snapshot (§9.3) |
| content_hash | text | SHA-256; identical content → no new version |
| size_bytes | int | |
| published_by | uuid | |
| published_at | timestamptz | |
| is_current | boolean | Partial unique index: one current per outlet |

**catalog_changes** — draft change journal for "3 unpublished changes" and diff review: `tenant_id`, `entity_type`, `entity_id`, `change` (`create`\|`update`\|`delete`), `before jsonb`, `after jsonb`, `changed_by`, `published_in_version null`.

**Draft vs published model**: the catalog tables are the **working copy** that admins edit. Guests never read them; they read `menu_snapshots` (immutable) plus the **availability overlay** (sold-out flags from `outlet_item_overrides`/`outlet_option_overrides`, served live — §9.4). Order validation and pricing use the **current published snapshot**, so guests are always charged what they saw.

**media_assets** — `tenant_id`, `kind` (`item_photo`\|`logo`\|`cover`\|`category`), `original_key`, `status` (`uploading`\|`processing`\|`ready`\|`failed`), `width`, `height`, `thumbhash`, `variants jsonb` (`{ "card_480": {"avif": key, "webp": key}, … }`), `bytes`, `error`, `uploaded_by`.

### 7.7 Ordering tables

**table_sessions**
| Column | Type | Notes |
|--------|------|-------|
| tenant_id, outlet_id | uuid | |
| status | text | `open` \| `bill_requested` \| `settled` \| `closed` \| `merged` |
| primary_spot_id | uuid | Spot shown as the session's name |
| opened_at | timestamptz | |
| last_activity_at | timestamptz | Drives idle timeout (GST-ORD-012) |
| bill_requested_at | timestamptz null | |
| closed_at | timestamptz null | |
| closed_reason | text null | `settled` \| `idle_timeout` \| `staff_cleared` \| `merged` |
| merged_into_id | uuid null | |
| business_date | date | Business date of opening |
| is_test | boolean | ADM-ONB-005 |
| version | int | |

**session_spots** — `session_id`, `spot_id`, `active boolean`, `attached_at`, `detached_at`. **Partial unique index** `(spot_id) WHERE active` guarantees **one open session per spot** — the database, not application code, prevents two parallel sessions at a table under concurrent first orders.

**outlet_counters** — `outlet_id`, `business_date`, `last_order_number int`; PK both. Order numbers are allocated with `INSERT … ON CONFLICT (outlet_id, business_date) DO UPDATE SET last_order_number = outlet_counters.last_order_number + 1 RETURNING last_order_number` inside the order transaction.

**orders**
| Column | Type | Notes |
|--------|------|-------|
| tenant_id, outlet_id | uuid | |
| session_id | uuid | |
| spot_id | uuid null | Spot where placed |
| order_number | int | Per outlet per business day (#27) |
| business_date | date | |
| source | text | `guest` \| `staff` |
| customer_id | uuid null | |
| guest_device_id | uuid null | |
| placed_by_user_id | uuid null | Staff source |
| guest_display_name | text null | First name shown to table and staff |
| status | text | §7.8 |
| notes | text null | ≤ 200 chars |
| snapshot_version | int | Menu version priced against |
| items_subtotal_paise | bigint | Sum of line totals |
| idempotency_key | text | Unique (`tenant_id`, `idempotency_key`) |
| request_hash | text | SHA-256 of normalised request body (detects key reuse with different body) |
| placed_at, confirmed_at, preparing_at, ready_at, served_at, cancelled_at | timestamptz null | |
| decline_reason / cancel_reason | text null | |
| cancelled_by_type | text null | `guest` \| `staff` \| `system` |
| is_test | boolean | |
| version | int | |

**order_items** (all names and prices are **snapshotted** — later menu edits never change past orders)
| Column | Type | Notes |
|--------|------|-------|
| tenant_id, order_id | uuid | |
| item_id, variant_id | uuid | References for analytics |
| item_name, item_name_hi, variant_name | text | Snapshot |
| unit_price_paise | bigint | Variant price at order time |
| modifiers_unit_paise | bigint | Sum of modifier prices per unit |
| quantity | smallint | 1–max |
| line_total_paise | bigint | (unit + modifiers) × qty |
| tax_group_snapshot | jsonb | `[{"name":"CGST","code":"cgst","rateBp":250}, …]` |
| station_id | uuid null | |
| instructions | text null | ≤ 140 chars |
| status | text | `pending` \| `preparing` \| `ready` \| `served` \| `cancelled` |
| cancelled_reason | text null | |
| dietary | text | Snapshot for KDS display |

**order_item_modifiers** — `order_item_id`, `group_name`, `option_id`, `option_name`, `option_name_hi`, `price_paise`, `quantity`.

**order_events** (append-only history; drives the timeline in A-12)
`tenant_id`, `order_id`, `order_item_id null`, `from_status`, `to_status`, `actor_type` (`guest`\|`staff`\|`device`\|`system`), `actor_id`, `reason`, `at`.

**guest_carts** (server mirror of device carts; enables cart recovery and future shared cart)
`tenant_id`, `outlet_id`, `guest_device_id`, `spot_id`, `content jsonb`, `client_updated_at`, `expires_at` (12 h). Unique (`guest_device_id`, `outlet_id`).

### 7.8 State machines

Implemented once in `packages/domain/state-machines` and enforced in services; invalid transitions throw `INVALID_STATE_TRANSITION`.

**Order**
| From | To | Who | Condition |
|------|----|-----|-----------|
| — | `placed` | guest/staff | Created |
| `placed` | `confirmed` | staff, or system (auto-accept) | |
| `placed` | `declined` | staff | Reason required |
| `placed`, `confirmed` | `cancelled` | guest | Within cancel window (GST-ORD-009) |
| `placed`, `confirmed`, `preparing`, `ready` | `cancelled` | staff with `orders.cancel` | Reason required |
| `confirmed` | `preparing` | staff/KDS | Also automatic when any line is started |
| `preparing` | `ready` | staff/KDS | Also automatic when all non-cancelled lines are ready |
| `ready` | `served` | staff | |
| `ready` | `preparing` | KDS "recall" | Within 10 minutes |

Auto-accept writes `placed → confirmed` in the same transaction as creation (two `order_events` rows, same timestamp).

**Order item**: `pending → preparing → ready → served`; `pending|preparing → cancelled` (staff, reason). Cancelling all lines cancels the order.

**Table session**
| From | To | Trigger |
|------|----|---------|
| — | `open` | First order at a spot with no active session |
| `open` | `bill_requested` | Guest "Request bill" / staff |
| `bill_requested` | `open` | New order placed (guest ordered more) |
| `open`, `bill_requested` | `settled` | Bill settled (due = 0) |
| `settled` | `closed` | Staff "clear" (or automatic after 2 min, setting) |
| `open` | `closed` | Idle timeout with **no orders** or all orders cancelled; staff clear with no billable lines |
| `open`, `bill_requested` | `merged` | Merged into another session |

A session with unpaid billable lines can never auto-close; idle timeout only raises a "needs attention" flag for staff.

**Bill**: `open → partially_paid → paid → settled`; `settled → voided`. `paid` means due = 0 but not yet confirmed by staff; `settled` assigns the bill number.

**Payment**: `pending → succeeded | failed | cancelled`; `succeeded → refunded | partially_refunded` (Phase 8). Counter payments are created directly as `succeeded`.

**Service request**: `open → acknowledged → done`; `open|acknowledged → cancelled`.

### 7.9 Billing tables and calculation

**bills**
| Column | Type | Notes |
|--------|------|-------|
| tenant_id, outlet_id | uuid | |
| session_id | uuid unique | One bill per session |
| status | text | §7.8 |
| price_display | text | Copied from settings when the bill opens (rule changes mid-meal don't alter it) |
| items_subtotal_paise | bigint | |
| discount_total_paise | bigint | |
| service_charge_bp | int | 0 if disabled |
| service_charge_removed | boolean | D-15 |
| service_charge_paise | bigint | |
| taxable_total_paise | bigint | |
| tax_total_paise | bigint | |
| round_off_paise | bigint | May be negative |
| grand_total_paise | bigint | |
| paid_total_paise | bigint | |
| tip_total_paise | bigint | Not revenue |
| bill_number | text null | Assigned at settlement |
| financial_year | text null | `25-26` |
| settled_at / settled_by | | |
| voided_at / voided_by / void_reason | | |
| calc_version | int | Version of the calculator used (for future rule changes) |
| version | int | |

Totals are **recomputed in full** by the calculator after every change (order placed/cancelled, discount, service charge toggle) — never incrementally patched.

**bill_taxes** — `bill_id`, `tax_code`, `tax_name`, `rate_bp`, `taxable_paise`, `tax_paise`.
**discounts** — `tenant_id`, `bill_id`, `scope` (`bill`\|`line`), `order_item_id null`, `kind` (`percent`\|`fixed`), `value` (bp or paise), `amount_paise` (computed), `reason`, `applied_by`, `coupon_id null`, `removed_at null`.
**payments** — `tenant_id`, `bill_id`, `method` (`cash`\|`card_terminal`\|`upi_counter`\|`room_charge`\|`online`\|`other`), `amount_paise`, `tip_paise`, `cash_tendered_paise null`, `status`, `reference`, `provider null`, `provider_payment_id null` (unique when not null), `idempotency_key`, `recorded_by`, `created_at`.
**bill_number_sequences** — `tenant_id`, `outlet_id`, `financial_year`, `prefix`, `last_value`; row-locked (`SELECT … FOR UPDATE`) during settlement, so numbers are **gapless and never reused** (GST requirement). Voided bills keep their number.
**day_closes** — `tenant_id`, `outlet_id`, `business_date` unique per outlet, `summary jsonb`, `expected_cash_paise`, `counted_cash_paise`, `difference_paise`, `closed_by`, `closed_at`, `notes`.

**Bill calculation algorithm** (`packages/domain/billing/calculate.ts`, pure function, identical on client and server)

Inputs: billable lines (non-cancelled order items with tax snapshots), discounts, price display mode, service charge settings, rounding mode.

1. **Line gross** = `(unit_price + modifiers_unit) × qty` for each line.
2. **Line discounts**: percent → `round_half_up(line_gross × bp / 10000)`; fixed → `min(value, line_gross)`.
3. **Bill-level discounts**: computed on the sum of post-line-discount amounts, then **allocated across lines proportionally** using the largest-remainder method so allocations sum exactly to the discount. (Needed because lines may have different tax rates.)
4. **Line net** = gross − line discount − allocated bill discount.
5. **Taxable value per line**:
   - Exclusive pricing: `taxable = line net`.
   - Inclusive pricing: `taxable = round_half_up(line_net × 10000 / (10000 + total_rate_bp))`; the tax is the difference.
6. **Service charge** (if enabled and not removed): `round_half_up(Σ taxable × sc_bp / 10000)`. In India service charge is taxable at the supply's GST rate, so it is allocated across lines by taxable value (largest remainder) and added to each line's taxable value.
7. **Taxes**: group by tax code + rate; `tax = round_half_up(Σ taxable_in_group × rate_bp / 10000)` — rounding happens **once per tax component at bill level**, not per line (avoids paise drift).
8. **Total before rounding** = Σ taxable + Σ taxes (exclusive) or Σ line net + service charge + service charge tax (inclusive).
9. **Round off** to the setting (nearest ₹1 / ₹0.50 / none), recorded as `round_off_paise`.
10. **Due** = grand total − Σ succeeded payments (tips excluded).

**Worked example** (exclusive pricing, GST 5 % = CGST 2.5 % + SGST 2.5 %, 10 % bill discount, 5 % service charge, round to ₹1):

| Step | Value |
|------|-------|
| Iced Latte ₹280 × 2 + Croissant ₹180 | 74,000 paise |
| 10 % discount | −7,400 → 66,600 |
| Service charge 5 % | +3,330 → taxable 69,930 |
| CGST 2.5 % of 69,930 = 1,748.25 | 1,748 |
| SGST 2.5 % | 1,748 |
| Total | 73,426 |
| Round off | −26 |
| **Grand total** | **73,400 paise = ₹734.00** |

Rounding function: `round_half_up(x) = floor(x + 0.5)` on exact integer math using `bigint` (numerator/denominator), never floats.

### 7.10 Customers tables

**customers**
| Column | Type | Notes |
|--------|------|-------|
| tenant_id | uuid | D-07: per tenant |
| phone_enc | bytea | AES-256-GCM encrypted `+91XXXXXXXXXX` (application-level, key in secrets manager) |
| phone_hash | text | HMAC-SHA-256(phone, tenant-specific pepper) for lookups; unique (`tenant_id`, `phone_hash`) where not anonymised |
| phone_last4 | char(4) | For masked display (`98•••••210` uses first 2 from enc on owner view) |
| name | text | D-17: only personal field |
| verified_at | timestamptz null | Null = unverified (OTP off) |
| marketing_consent | boolean | Default false |
| first_visit_at, last_visit_at | timestamptz | |
| visit_count | int | Settled sessions |
| total_spend_paise | bigint | |
| anonymised_at | timestamptz null | After deletion request |

**guest_devices** (global, no tenant — one browser cookie works across all tenants) — `id`, `created_at`, `last_seen_at`, `user_agent_hash`. The cookie holds a signed random token mapped to this ID.
**device_customer_links** — `tenant_id`, `guest_device_id`, `customer_id`, `linked_at`, `expires_at` (180 days, GST-ID-005), `revoked_at`.
**customer_visits** — `tenant_id`, `customer_id`, `outlet_id`, `session_id`, `visited_at`, `spend_paise`, `items_count`.
**customer_item_stats** — `tenant_id`, `customer_id`, `item_id`, `times_ordered`, `quantity_total`, `last_ordered_at`; PK (`customer_id`, `item_id`). Powers "Your usual".
**consent_records** — `tenant_id`, `customer_id`, `notice_version`, `purposes text[]` (`orders`, `history`, `marketing`), `granted_at`, `withdrawn_at`, `channel` (`guest_app`), `ip_hash`.
**customer_notes** — `tenant_id`, `customer_id`, `note` (≤ 280), `author_id`.
**data_requests** — `tenant_id`, `customer_id`, `type` (`deletion`\|`access`), `status`, `requested_at`, `completed_at`.
**otp_attempt_log** — `phone_hash`, `guest_device_id`, `ip_hash`, `event` (`sent`\|`verified`\|`failed`\|`blocked`), `at` (abuse analysis; 90-day retention). Active OTP codes live only in Redis (§14.5).

### 7.11 Service, feedback, promotions

**service_request_types** — `tenant_id`, `key`, `label`, `label_hi`, `icon`, `is_system` (`call_waiter`, `request_bill`), `enabled`, `sort_order`.
**service_requests** — `tenant_id`, `outlet_id`, `spot_id`, `session_id null`, `type_id`, `status`, `created_at`, `acknowledged_by/at`, `done_by/at`, `guest_device_id`.
**feedback** (P1) — `tenant_id`, `outlet_id`, `session_id`, `customer_id null`, `rating smallint` (1–5), `comment`; **feedback_items** — `feedback_id`, `item_id`, `thumbs` (−1/1).
**coupons** (P1) — `tenant_id`, `code null` (null = automatic offer), `kind`, `value`, `min_bill_paise`, `max_discount_paise`, `applies_to jsonb` (categories/items), `schedule jsonb`, `starts_at`, `ends_at`, `usage_limit`, `per_customer_limit`, `status`; **coupon_redemptions** — `coupon_id`, `bill_id`, `customer_id`, `amount_paise`.

### 7.12 Insights, audit, and infrastructure tables

**Aggregate tables** (written by the insights worker; read by dashboard and reports; all keyed by `tenant_id, outlet_id, business_date`):
- `sales_daily` — gross, discounts, net, service charge, tax, round off, tips, bills, orders, items sold, guests (sum of capacity or session count), cancellations, voids, avg bill, payment method breakdown (jsonb).
- `sales_hourly` — same core measures by `hour smallint` (outlet local time).
- `item_sales_daily` — `item_id`, `variant_id`, qty, gross, net, orders containing.
- `category_sales_daily` — `category_id`, qty, net.
- `customer_daily` — new customers, returning customers, verified share.
- `ops_daily` — average/p90 confirm time, prep time (confirmed → ready), table turnover (session open → settled), service request response time.

**insights** — `tenant_id`, `outlet_id null`, `rule_key`, `fingerprint` (unique while active), `severity` (`info`\|`opportunity`\|`warning`\|`critical`), `score numeric`, `title`, `body`, `data jsonb` (numbers used), `actions jsonb` (deep links), `period_start`, `period_end`, `status` (`active`\|`dismissed`\|`snoozed`\|`expired`), `snoozed_until`, `created_at`, `expires_at`.

**audit_log** (append-only; `UPDATE`/`DELETE` revoked from `app_rw`)
| Column | Type |
|--------|------|
| id | bigint identity |
| tenant_id | uuid |
| outlet_id | uuid null |
| actor_type | text (`staff`\|`device`\|`platform`\|`system`\|`guest`) |
| actor_id | uuid null |
| action | text (e.g. `item.price_changed`, `bill.voided`, `discount.applied`, `customer.exported`, `customer.viewed_phone`) |
| entity_type / entity_id | text / uuid |
| before / after | jsonb |
| ip_hash, user_agent | text |
| request_id | text |
| at | timestamptz |

Partitioned by month (`PARTITION BY RANGE (at)`), retained 3 years.

**outbox_events** — `id bigint identity`, `tenant_id`, `outlet_id`, `topic` (`order.placed`…), `aggregate_type`, `aggregate_id`, `aggregate_version`, `payload jsonb`, `created_at`, `published_at null`. Index on `(published_at) WHERE published_at IS NULL`.

**idempotency_keys** — `tenant_id`, `scope` (`order.create`, `payment.record`, `service_request.create`, …), `key`, `request_hash`, `status` (`in_progress`\|`completed`), `response_status`, `response_body jsonb`, `created_at`, `expires_at` (24 h). PK (`tenant_id`, `scope`, `key`).

**outlet_event_seq** — `outlet_id` PK, `last_seq bigint`; every outbox event for an outlet gets the next sequence number (used by realtime gap detection, §11.4).

### 7.13 Key indexes

| Table | Index | Serves |
|-------|-------|--------|
| qr_codes | unique (`token`) | QR resolution |
| session_spots | unique (`spot_id`) where `active` | One session per spot |
| orders | (`tenant_id`, `outlet_id`, `status`, `placed_at`) where status in active set | Orders board |
| orders | (`session_id`) | Session view |
| orders | unique (`tenant_id`, `idempotency_key`) | Exactly-once |
| orders | (`tenant_id`, `outlet_id`, `business_date`) | Day reports |
| orders | (`tenant_id`, `customer_id`, `placed_at desc`) | Guest history |
| order_items | (`order_id`), (`tenant_id`, `item_id`, `created_at`) | Lines, item analytics |
| bills | (`tenant_id`, `outlet_id`, `settled_at`) | Bill history |
| payments | unique (`provider`, `provider_payment_id`) where not null | Webhook dedupe |
| customers | unique (`tenant_id`, `phone_hash`) where `anonymised_at is null` | Lookup |
| customers | (`tenant_id`, `last_visit_at desc`) | Customer list |
| outlet_item_overrides | unique (`outlet_id`, `item_id`, `variant_id`) | Availability |
| menu_snapshots | unique (`outlet_id`) where `is_current` | Pointer |
| audit_log | (`tenant_id`, `at desc`), (`tenant_id`, `entity_type`, `entity_id`) | Audit screen |
| outbox_events | (`id`) where `published_at is null` | Publisher |

### 7.14 Data retention

| Data | Retention |
|------|-----------|
| Orders, bills, payments, bill taxes | 8 years (Indian GST record-keeping requires 72 months after the annual return; 8 years gives margin) |
| Audit log | 3 years |
| Customer personal data (name, phone) | Until deletion request, or 3 years after last visit (then anonymised automatically) |
| OTP attempt log | 90 days |
| Guest carts | 12 hours |
| Idempotency keys | 24 hours |
| Outbox events (published) | 7 days |
| Original photos | While referenced + 30 days |
| Exports | 7 days |

### 7.15 Migrations

- Forward-only SQL migrations in `packages/db/migrations`, generated by drizzle-kit and reviewed by hand.
- **Expand → migrate → contract** for breaking changes: add new column/table, dual-write, backfill in batches, switch reads, remove old column in a later release. No migration may lock a large table for more than a second (use `CREATE INDEX CONCURRENTLY`, add nullable columns, backfill in batches of 5,000).
- Seed data: built-in roles and permissions, service request types, tax presets (GST 5 %, 12 %, 18 %), allergen list, theme presets.

---

## 8. API design

### 8.1 Conventions

| Topic | Rule |
|-------|------|
| Style | REST, JSON, HTTPS only. Base path `/api/v1`. |
| Surfaces | `/api/v1/guest/*` (guest app), `/api/v1/admin/*` (admin, staff), `/api/v1/kds/*` (paired devices), `/api/v1/platform/*` (platform admin), `/api/v1/webhooks/*` (providers). Each surface has its own auth plugin; a token for one surface is rejected on the others. |
| Contracts | Every request and response is a Zod schema in `packages/contracts`; OpenAPI 3.1 is generated from them; `packages/sdk` is generated from OpenAPI. |
| Casing | camelCase fields. Money fields end in `Paise`. Times ISO 8601 UTC (`2026-10-21T14:06:00Z`); business dates `YYYY-MM-DD`. |
| IDs | UUID strings. |
| Request ID | Client may send `X-Request-Id`; server always returns `X-Request-Id` and includes it in errors and logs. |
| Idempotency | `Idempotency-Key` header (UUIDv4/v7) **required** on: create order, cancel order, record payment, settle bill, service request create, OTP send. Optional on other POSTs (§13.4). |
| Concurrency | Mutable resources return `version`; updates send `If-Match: <version>` (or `version` in body). Mismatch → `409 CONFLICT_STALE_VERSION`. |
| Pagination | Cursor-based: `?limit=50&cursor=<opaque>` → `{ data: [...], nextCursor }`. Max limit 200. |
| Filtering/sorting | `?status=placed,confirmed&sort=-placedAt`. |
| Partial responses | Lists return summaries; details endpoints return full objects. |
| Caching | `ETag` on GETs; `Cache-Control` as specified per endpoint (§9). |
| Compression | Brotli/gzip for responses > 1 KB. |
| Rate limits | `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset` headers; `429` with `Retry-After` (§14.6). |
| Errors | `application/problem+json` (§13.2). |
| Versioning | Additive changes only within `v1`. Breaking changes → `v2` path, with both served for ≥ 6 months. Clients send `X-Client-Version`; the API can return `426 CLIENT_UPGRADE_REQUIRED` for dangerously old guest/admin builds. |
| Timeouts | Server request timeout 10 s (exports and imports are async jobs). |

### 8.2 Guest API (`/api/v1/guest`)

Auth: signed `gd` cookie (guest device, httpOnly, Secure, SameSite=Lax, 1-year) set on first request; optional customer link via `device_customer_links`. Tenant/outlet/spot derived from the QR token or the targeted resource.

| Method & path | Purpose | Notes |
|---------------|---------|-------|
| `GET /q/{token}` | Resolve QR → outlet, spot, branding, status, current snapshot version & URL, ordering state, hours | Edge cached 30 s (§9.2) |
| `GET /outlets/{outletId}/menu/{version}` | Immutable menu snapshot | `Cache-Control: public, max-age=31536000, immutable` |
| `GET /outlets/{outletId}/availability` | Live sold-out overlay + ordering paused state + `snapshotVersion` | Edge cached 5 s; ETag |
| `PUT /carts/{outletId}` | Mirror device cart (best effort) | Debounced 2 s by client |
| `POST /carts/validate` | Validate cart lines vs current snapshot + availability + rules; returns priced lines, bill preview, and any `changes[]` | Read-only |
| `POST /orders` | Place order | Idempotency-Key required (§10) |
| `GET /orders/{orderId}` | Order with lines and status | Only for the device/customer that placed it or any device that has joined the session |
| `POST /orders/{orderId}/cancel` | Guest cancel | Idempotency-Key |
| `GET /sessions/current?spotToken=` | Current session at this spot: orders (first names), statuses, bill summary | Requires the QR token (proof of presence) |
| `GET /sessions/{sessionId}/bill` | Full bill | |
| `POST /sessions/{sessionId}/bill-request` | Request bill / pay at counter | Idempotent per session |
| `DELETE /sessions/{sessionId}/service-charge` | Remove service charge (D-15) | |
| `POST /service-requests` | Create request (`typeKey`, `spotToken`) | Idempotency-Key; 60 s dedupe |
| `GET /service-requests/{id}` | Status | |
| `POST /identity/otp` | Send OTP `{ phone, name, tenantId via spotToken, consent }` | Rate limited (§14.6) |
| `POST /identity/verify` | Verify OTP `{ phone, code }` → links device to customer | |
| `GET /me?tenant=` | Current customer at this tenant (name, masked phone), "your usual", last visit | |
| `GET /me/orders?tenant=` | Visit history | Paginated |
| `POST /me/reorder` | `{ sessionId or orderId }` → validated lines for cart | |
| `POST /me/logout` | Unlink device for this tenant | |
| `POST /me/delete` | Data deletion request | Creates `data_requests` row; immediate anonymisation (§14.8) |
| `POST /feedback` (P1) | Rating | One per session per device |
| `POST /beacon` | Fire-and-forget analytics (scan, menu viewed, item viewed, add to cart) | `navigator.sendBeacon`, batched, never blocks UI |

### 8.3 Admin API (`/api/v1/admin`)

Auth: session cookie (`sid`, httpOnly, Secure, SameSite=Lax) + CSRF token header for state-changing requests; `X-Outlet-Id` for outlet-scoped calls. Every route declares a permission (§14.3) and, where relevant, a feature flag.

| Area | Endpoints (summary) |
|------|---------------------|
| Auth | `POST /auth/login`, `POST /auth/otp`, `POST /auth/otp/verify`, `POST /auth/logout`, `POST /auth/password/forgot`, `POST /auth/password/reset`, `POST /auth/switch` (PIN quick switch), `POST /invites/{token}/accept` |
| Me | `GET /me` (user, memberships, current tenant, permissions, features, limits, outlets) |
| Tenant | `GET/PATCH /settings`, `GET/PATCH /branding`, `GET/PATCH /business-profile` |
| Outlets | `GET/POST /outlets`, `GET/PATCH/DELETE /outlets/{id}`, `PUT /outlets/{id}/hours`, `POST /outlets/{id}/closures`, `POST /outlets/{id}/pause-ordering`, `POST /outlets/{id}/resume-ordering`, `POST /outlets/{id}/go-live` |
| Areas & spots | `GET/POST /areas`, `PATCH/DELETE /areas/{id}`, `GET/POST /spots`, `POST /spots/bulk`, `PATCH/DELETE /spots/{id}`, `POST /spots/{id}/regenerate-qr`, `POST /qr-cards` (async PDF job → `jobId`) |
| Floor | `GET /floor` (spots with session state, totals, timers) |
| Catalog | `GET/POST /menus`, `PATCH/DELETE /menus/{id}`, `PUT /menus/{id}/schedules`, `PUT /menus/{id}/assignments`, `PUT /menus/{id}/categories` (order), `GET/POST /categories`, `PATCH/DELETE /categories/{id}`, `GET/POST /items`, `GET/PATCH/DELETE /items/{id}`, `POST /items/{id}/duplicate`, `PUT /items/{id}/variants`, `PUT /items/{id}/modifier-groups`, `PUT /items/{id}/media`, `GET/POST /modifier-groups`, `PATCH/DELETE /modifier-groups/{id}`, `GET/POST /tags`, `GET/POST /stations`, `POST /items/bulk-price` (preview with `dryRun=true`), `GET/POST /tax-groups`, `GET/POST /taxes` |
| Availability | `GET /availability`, `PUT /availability` (`[{itemId, variantId?, optionId?, soldOut, until}]`) — applies instantly |
| Publishing | `GET /publish/changes`, `POST /publish` (→ new snapshot version), `GET /menu-versions`, `POST /menu-versions/{v}/restore` |
| Media | `POST /media/uploads` (→ presigned PUT URL + `mediaId`), `POST /media/{id}/complete`, `GET /media/{id}` (status, variants) |
| Imports/exports | `POST /imports/menu` (file → job), `GET /jobs/{id}`, `POST /exports/{type}` |
| Orders | `GET /orders?status=&area=&station=` (board), `GET /orders/changes?since=<seq>` (resync), `GET /orders/{id}`, `POST /orders` (waiter order, Idempotency-Key), `POST /orders/{id}/confirm`, `POST /orders/{id}/decline`, `POST /orders/{id}/status` (`preparing`/`ready`/`served`), `POST /orders/{id}/cancel`, `POST /order-items/{id}/cancel` |
| Sessions | `GET /sessions/{id}`, `POST /sessions/{id}/move` (`toSpotId`), `POST /sessions/{id}/merge` (`intoSessionId`), `POST /sessions/{id}/clear` |
| Billing | `GET /bills/{id}`, `POST /bills/{id}/discounts`, `DELETE /bills/{id}/discounts/{discountId}`, `PATCH /bills/{id}/service-charge`, `POST /bills/{id}/payments` (Idempotency-Key), `POST /bills/{id}/settle` (Idempotency-Key), `POST /bills/{id}/void`, `GET /bills?from=&to=&status=`, `GET /bills/{id}/print` (receipt HTML) |
| Day close | `GET /day-close/preview?date=`, `POST /day-close` |
| Service requests | `GET /service-requests?status=open`, `POST /service-requests/{id}/ack`, `POST /service-requests/{id}/done` |
| Customers | `GET /customers?q=&filter=`, `GET /customers/{id}`, `POST /customers/{id}/reveal-phone` (audited), `POST /customers/{id}/notes`, `POST /customers/{id}/anonymise`, `POST /exports/customers` |
| Staff | `GET /staff`, `POST /staff/invite`, `PATCH /staff/{membershipId}` (role, outlets), `POST /staff/{id}/deactivate`, `PUT /staff/{id}/pin`, `GET/POST /roles`, `PATCH /roles/{id}` |
| Devices | `GET /devices`, `POST /devices/pair` (`code`, outlet, station, name), `POST /devices/{id}/revoke` |
| Dashboard & insights | `GET /dashboard/today`, `GET /dashboard?period=&outlet=`, `GET /insights`, `POST /insights/{id}/dismiss`, `POST /insights/{id}/snooze` |
| Reports | `GET /reports/{type}?from=&to=&outlet=` (`sales`, `items`, `categories`, `tax`, `discounts`, `payments`, `staff`, `hourly`) |
| Audit | `GET /audit?actor=&action=&from=&to=` |
| Plan | `GET /plan` (features, limits, usage) |

### 8.4 KDS API (`/api/v1/kds`)

Auth: device token (`Authorization: Device <token>`), bound to tenant + outlet + optional station.

| Method & path | Purpose |
|---------------|---------|
| `POST /pairing/start` | Unpaired device gets a 6-digit pairing code (valid 10 min) and polls |
| `GET /pairing/{code}/status` | Returns device token once a manager pairs it |
| `GET /tickets` | Active tickets for the device's station(s) |
| `GET /tickets/changes?since=<seq>` | Resync |
| `POST /order-items/{id}/status` | Strike line (`preparing`/`ready`) |
| `POST /orders/{id}/bump` | Ticket ready |
| `POST /orders/{id}/recall` | Undo bump (≤ 10 min) |
| `POST /heartbeat` | Every 30 s: device health, app version (shown in A-52) |

### 8.5 Platform API (`/api/v1/platform`)

Auth: platform session + mandatory TOTP. `GET/POST /tenants`, `GET/PATCH /tenants/{id}`, `POST /tenants/{id}/suspend`, `POST /tenants/{id}/reactivate`, `POST /tenants/{id}/schedule-deletion`, `POST /tenants/{id}/export`, `GET/POST /plans`, `PATCH /plans/{id}`, `PUT /plans/{id}/features`, `PUT /plans/{id}/limits`, `PUT /tenants/{id}/overrides`, `POST /tenants/{id}/support-sessions`, `GET /health`, `GET /audit`, `GET/POST /announcements` (P1), `GET/POST /tenants/{id}/invoices` (P1).

### 8.6 Webhooks (`/api/v1/webhooks`)

`POST /webhooks/payments/{provider}` (Phase 8, §16), `POST /webhooks/sms/{provider}` (delivery reports). Signature verified before parsing; always respond `200` quickly after durable storage; processing is async and idempotent.

### 8.7 Example: place order

```http
POST /api/v1/guest/orders
Idempotency-Key: 0192b6f4-6f1e-7c4a-9d2e-5a1b2c3d4e5f
Content-Type: application/json

{
  "spotToken": "7Hk2pX9aQm4Z",
  "snapshotVersion": 42,
  "clientCreatedAt": "2026-10-21T14:06:00Z",
  "guestName": "Riya",
  "notes": "Starters first please",
  "lines": [
    {
      "clientLineId": "l1",
      "itemId": "0192…a1",
      "variantId": "0192…b2",
      "quantity": 2,
      "modifiers": [{ "optionId": "0192…c3", "quantity": 1 }],
      "instructions": "Less ice",
      "expectedUnitTotalPaise": 32000
    }
  ]
}
```

```http
201 Created
{
  "order": {
    "id": "0192…f9", "orderNumber": 27, "status": "placed",
    "requiresConfirmation": true, "sessionId": "0192…e1",
    "lines": [ … priced lines … ], "itemsSubtotalPaise": 64000,
    "placedAt": "2026-10-21T14:06:01Z", "estimatedReadyMinutes": 15
  },
  "bill": { "grandTotalPaise": 70600, "duePaise": 70600, … },
  "realtime": { "room": "session:0192…e1", "token": "<short-lived JWT>" }
}
```

A retry with the same key and body returns the **same** `201` body (from the idempotency store) with header `Idempotent-Replayed: true`.

---

## 9. Menu delivery and caching

### 9.1 Why snapshots

A crowded venue means dozens of phones opening the same menu over weak Wi-Fi at the same moment. If each request hit the database, menu speed would depend on server load. Instead the whole menu for an outlet is pre-built into **one immutable JSON document per published version**, served from the CDN edge in Mumbai/Chennai/Delhi, and cached on the phone. The database is not touched when a guest browses.

### 9.2 Resolution and caching flow

```
Phone ──GET hotl.in/q/7Hk2pX9aQm4Z──▶ Edge
   Edge cache HIT (≤ 30 s old) ─────────────▶ HTML (menu rendered, theme inlined,
   MISS → Guest app SSR → API resolve_qr        snapshot v42 URL in <link rel=preload>)
            → pointer {outlet, spot, v42}
Phone ──GET /api/v1/guest/outlets/{o}/menu/42──▶ Edge (immutable, HIT almost always)
Phone ──GET /api/v1/guest/outlets/{o}/availability──▶ Edge (5 s) → API → Redis
```

| Resource | Cache policy | Invalidation |
|----------|--------------|--------------|
| `/q/{token}` HTML | `s-maxage=30, stale-while-revalidate=300`, `private` parts (cart, identity) are client-side only | Natural expiry (30 s). Revoked QR: ordering is blocked immediately by the API; the page itself stops working within 30 s. |
| Snapshot `/menu/{version}` | `public, max-age=31536000, immutable` | Never — new version = new URL |
| Availability | `public, s-maxage=5, stale-while-revalidate=30` + ETag | Natural expiry |
| Images `img.hotl.in/...` | `public, max-age=31536000, immutable` | New asset = new URL |
| Fonts, JS, CSS | Hashed filenames, immutable | Deploy |

The guest app's HTML must contain **no personal data** so it can be shared in the edge cache; identity, cart, and session details load client-side after paint.

### 9.3 Snapshot content

```jsonc
{
  "v": 42,
  "outletId": "…",
  "generatedAt": "2026-10-21T13:58:00Z",
  "currency": "INR",
  "priceDisplay": "exclusive",
  "menus": [
    { "id": "m1", "name": "All day", "nameHi": "पूरे दिन", "schedule": [{ "days": [0,1,2,3,4,5,6], "from": "08:00", "to": "23:00" }],
      "assign": { "areas": null, "spotTypes": null },
      "categories": ["c1", "c2"] }
  ],
  "categories": {
    "c1": { "name": "Coffee", "nameHi": "कॉफ़ी", "layout": "photo_list", "items": ["i1", "i2"] }
  },
  "items": {
    "i1": {
      "name": "Iced Latte", "nameHi": "आइस्ड लाटे",
      "desc": "Double shot over ice…", "descHi": null,
      "diet": "veg", "spice": 0, "badges": ["bestseller"], "tags": ["t1"], "allergens": ["dairy"],
      "img": { "id": "a9", "th": "<thumbhash base64>", "w": 1200, "h": 900 },
      "variants": [{ "id": "v1", "name": "Regular", "price": 28000, "default": true },
                   { "id": "v2", "name": "Large", "price": 32000 }],
      "groups": ["g1"], "prep": 6, "instr": true, "maxQty": 20, "window": null,
      "upsell": ["i7"]
    }
  },
  "groups": {
    "g1": { "name": "Milk", "sel": "single", "min": 1, "max": 1,
            "options": [{ "id": "o1", "name": "Regular", "price": 0, "default": true },
                        { "id": "o2", "name": "Oat", "price": 4000 }] }
  },
  "tags": { "t1": { "name": "Gluten free", "kind": "filter" } },
  "taxGroups": { "tg1": { "taxes": [{ "code": "cgst", "rateBp": 250 }, { "code": "sgst", "rateBp": 250 }] } },
  "settings": { "serviceChargeBp": 500, "rounding": "nearest_1", "maxQtyPerLine": 20, "maxItemsPerOrder": 50 }
}
```

- Short keys, IDs mapped to objects (no duplication), images as IDs (URLs built client-side from a template).
- Size budget: **≤ 60 KB gzip for a 200-item menu**. The build job fails loudly above 150 KB and warns the tenant.
- Search index is built client-side from the snapshot on idle (`requestIdleCallback`).

### 9.4 Availability overlay

```json
{ "snapshotVersion": 42, "etag": "W/\"a81f\"",
  "orderingOpen": true, "pausedUntil": null, "pausedMessage": null,
  "soldOut": { "items": ["i3"], "variants": ["v9"], "options": ["o2"] },
  "hidden": { "items": [] } }
```

- Stored in Redis (`t:{tenant}:avail:{outlet}`), rebuilt from `outlet_item_overrides` on any change, sub-millisecond reads.
- Guest app fetches it on load, every 60 s while the page is visible, on `visibilitychange`, when opening the cart, and before placing an order (GST-MENU-013).
- If `snapshotVersion` in the overlay is newer than the loaded snapshot, the client fetches the new snapshot in the background and swaps it in without disturbing the cart (cart lines referencing removed items are flagged at validation).

### 9.5 Snapshot build job

Triggered by `POST /admin/publish` or by instant-publish changes (price/visibility) debounced 3 s per outlet:
1. Load working-copy catalog + outlet overrides + menus assigned to the outlet.
2. Apply validation (ADM-MENU-014). Blocking errors abort with a list returned to the admin.
3. Serialise deterministically (sorted keys) → hash. If hash equals current, stop.
4. Insert `menu_snapshots` row (`version = current + 1`), set `is_current`, mark `catalog_changes.published_in_version`.
5. Update Redis pointer + availability; warm the edge by requesting the snapshot URL once from each region (optional).
6. Emit `menu.published` → admin toast "Live on the menu".

Time budget: < 2 s for 500 items.

---

## 10. Order pipeline

### 10.1 Client side (guest app)

1. Guest taps **Place order**.
2. Client creates `idempotencyKey` (UUIDv7) **once per order attempt** and stores `{key, payload, createdAt, state: "pending"}` in the IndexedDB **outbox** before sending anything.
3. Client calls `POST /carts/validate`. If `changes[]` is non-empty, show the change list (GST-CART-008) and stop; the outbox entry is discarded. If the network fails, skip validation — the order endpoint re-validates anyway.
4. Identity step if needed (GST-ORD-004). The outbox entry waits.
5. Send `POST /orders` with the key. Button shows progress ring.
6. Outcomes:
   - `201` → mark outbox entry `sent`, show status screen.
   - `409 CART_CHANGED` (prices/availability changed between validation and order) → show changes, discard entry.
   - `4xx` other → show error (§13.7), discard entry.
   - Network error / timeout (8 s) / `5xx` / `429` → keep entry, retry with **the same key** using backoff (§13.4). UI: "Sending your order…", then after 15 s "Still sending — your order is saved".
7. On app reload, the outbox is checked first; pending entries are resent with their original key. Entries older than 30 min are not auto-sent; the guest sees "This order wasn't sent. Send now?" (avoids surprising the kitchen with a stale order).
8. Only one pending order per outlet at a time; "Place order" is disabled while one is in flight.

### 10.2 Server side (`POST /guest/orders`)

```
 1. Rate-limit check (device, IP, spot)                         → 429
 2. Parse & validate body (Zod)                                  → 400 VALIDATION_FAILED
 3. Resolve spotToken → tenant, outlet, spot (resolve_qr)        → 404 QR_NOT_FOUND / 410 QR_REVOKED
 4. Idempotency lookup (tenant, "order.create", key):
      completed & same hash  → return stored response (replay)
      completed & diff hash  → 422 IDEMPOTENCY_KEY_REUSED
      in_progress            → 409 REQUEST_IN_PROGRESS (client retries after 1 s)
      none                   → insert in_progress row (unique constraint arbitrates races)
 5. Check tenant status, outlet live, opening hours, ordering paused, feature 'ordering'
                                                                  → 423 OUTLET_CLOSED / ORDERING_PAUSED
 6. Identity rule: require verified customer if tenant requires it → 401 IDENTITY_REQUIRED
 7. Load current snapshot (Redis/memory LRU) + availability
 8. Price every line on the server using snapshot; compare with expectedUnitTotalPaise
      any sold out / removed / price changed → 409 CART_CHANGED { changes[] }
 9. Business rules: max qty per line, max items, min/max modifiers, instructions allowed,
    item time windows, menu schedule at this spot              → 422 with specific code
10. BEGIN (SET LOCAL app.tenant_id)
      a. Find active session for spot: SELECT … FROM session_spots WHERE spot_id=$1 AND active FOR UPDATE
         none → INSERT table_sessions + session_spots (unique partial index; on conflict re-read)
      b. Allocate order_number (outlet_counters upsert)
      c. INSERT orders, order_items, order_item_modifiers, order_events
      d. Acceptance: auto / manual_all / manual_first (first = no confirmed orders in session)
      e. Recalculate bill (packages/domain/billing) → UPSERT bills, bill_taxes
      f. Update session last_activity_at; if status bill_requested → open
      g. INSERT outbox_events ('order.placed', 'bill.updated') with outlet seq numbers
      h. UPDATE idempotency_keys SET completed + response
    COMMIT
11. Return 201 (p95 target < 400 ms server time)
```

If anything fails before COMMIT, the transaction rolls back and the idempotency row is deleted (or marked failed) so the client's retry can succeed.

### 10.3 After commit

- **Outbox publisher** (worker, runs continuously): `SELECT … FROM outbox_events WHERE published_at IS NULL ORDER BY id LIMIT 500 FOR UPDATE SKIP LOCKED`, publishes to Redis channel `rt:outlet:{id}`, marks published. Typical lag < 100 ms; also triggered instantly via `LISTEN/NOTIFY` after commit.
- **Realtime gateway** pushes `order.placed` to `outlet:{id}:staff` and (if confirmed) `outlet:{id}:kds:*`; `order.status_changed` to `session:{id}`.
- **Insights** consumer increments live "today" counters in Redis (`ADM-DSH-001`).
- **Customer stats** consumer updates `customer_item_stats` (async; never blocks ordering).

### 10.4 Staff actions

Confirm / decline / status changes use the same pattern: permission check → state machine check → optimistic version check → update + `order_events` + outbox in one transaction. Double taps by staff are harmless: a second "confirm" on a confirmed order returns `200` with the current state (idempotent by target state).

### 10.5 Move and merge sessions

- **Move**: in one transaction, deactivate `session_spots(old)`, insert `session_spots(new)` (fails with `SPOT_OCCUPIED` if the target has an active session), update `primary_spot_id`, emit `session.moved`. Guests viewing the session get the event and update the label.
- **Merge**: lock both sessions in ID order (prevents deadlocks), move orders' `session_id` to the target, re-point `session_spots`, mark source `merged` with `merged_into_id`, recalc bill (target), void the empty source bill, emit `session.merged`.

---

## 11. Realtime

### 11.1 Gateway

- Separate process (`apps/realtime`) using Socket.IO with the Redis adapter; stateless, any number of instances behind the load balancer (sticky sessions required only for the long-polling fallback — enabled via cookie affinity).
- Transports: WebSocket first, automatic fallback to HTTP long-polling (corporate/hotel Wi-Fi and some mobile networks block WebSockets).
- Auth on connect: short-lived JWT (5 min, refreshed by REST) containing `sub`, `kind` (`staff`/`device`/`guest`), `tenantId`, `outletIds`, `sessionIds` (guest), `permissions` summary. The gateway verifies and joins rooms; clients can't choose rooms freely.

### 11.2 Rooms

| Room | Members | Receives |
|------|---------|----------|
| `outlet:{id}:staff` | Admin/staff with access to the outlet | All order, session, bill, service request, availability events |
| `outlet:{id}:kds:{stationId\|all}` | Paired KDS devices | Confirmed orders and line changes for the station |
| `session:{id}` | Guests who placed or joined the session | Order status, bill updates, session moved/merged/closed, service request ack |
| `tenant:{id}:admins` | Owners/managers | Insights, publish results, job completions, announcements |

### 11.3 Event catalogue

Every event has the envelope:
```json
{ "id": "evt_…", "type": "order.status_changed", "seq": 18234, "outletId": "…",
  "at": "2026-10-21T14:07:12Z", "entity": { "type": "order", "id": "…", "version": 4 },
  "data": { … } }
```

| Event | Data (summary) | Rooms |
|-------|----------------|-------|
| `order.placed` | Full order card (spot, number, lines, guest badge, requiresConfirmation) | staff, kds (if confirmed), session |
| `order.status_changed` | `{from, to, reason?}` | staff, kds, session |
| `order.item_status_changed` | `{orderItemId, from, to}` | staff, kds, session |
| `order.item_cancelled` | `{orderItemId, reason}` | staff, kds, session |
| `session.opened` / `session.closed` | spot, state | staff |
| `session.moved` / `session.merged` | from/to spot or session | staff, session |
| `bill.updated` | totals, due | staff, session |
| `bill.requested` | spot | staff |
| `bill.settled` | bill number, receipt token | staff, session |
| `payment.recorded` | method, amount | staff, session |
| `service_request.created` / `.updated` | type, spot, status | staff, session |
| `availability.changed` | sold-out deltas | staff (guests poll, §9.4) |
| `menu.published` | version | staff, tenant admins |
| `insight.created` | insight summary | tenant admins |
| `staff.deactivated` / `device.revoked` | — | forces disconnect of that user/device |

### 11.4 Delivery guarantees and resync

- Delivery is **at-least-once**; clients **deduplicate by event `id`** and ignore events whose `entity.version` is ≤ the version they already have.
- Each outlet has a monotonically increasing `seq`. Staff/KDS clients track `lastSeq`. If an event arrives with `seq > lastSeq + 1`, the client calls `GET /admin/orders/changes?since=lastSeq` (or the KDS equivalent) to fill the gap.
- On reconnect: always call `changes?since=lastSeq`. If `since` is older than the retention window (24 h) the API returns `410 RESYNC_REQUIRED` and the client reloads the full board.
- **Belt and braces for the kitchen**: KDS and the orders board poll `changes?since` every 30 s even while connected, and every 5 s while disconnected. A missed WebSocket message can delay a ticket by at most 30 s, never lose it.
- Guests: subscribe only after placing an order (keeps menu browsing light). Fallback polling of `GET /orders/{id}` every 5 s for 2 min, then 15 s, paused when the tab is hidden.
- Heartbeat: Socket.IO ping every 20 s, timeout 25 s. Connection indicator (KDS-005) turns amber on first missed ping, red after 10 s offline with an audible alert on KDS.

---

## 12. Performance, slow networks, and offline

### 12.1 Reference conditions

| Profile | Downlink / RTT / loss | Device |
|---------|----------------------|--------|
| Good 4G | 9 Mbps / 170 ms / 0 % | Mid-range Android (e.g. Samsung Galaxy A14, Redmi 12), iPhone 11 |
| Slow 3G | 400 Kbps / 400 ms / 0 % | Same |
| Crowded venue Wi-Fi | 1 Mbps / 300 ms / 3 % loss, frequent stalls | Same |
| Offline | — | Same |

### 12.2 Budgets (guest app)

| Metric | Budget (p75, good 4G, mid-range Android) | Slow 3G |
|--------|------------------------------------------|---------|
| Time to first byte (edge HIT) | < 200 ms | < 600 ms |
| Largest Contentful Paint (menu visible) | < 1.5 s | < 3.5 s |
| Interaction to Next Paint | < 100 ms | < 200 ms |
| Cumulative Layout Shift | < 0.05 | < 0.05 |
| Tap feedback | < 50 ms (CSS `:active`, no JS needed) | same |
| Animation frame rate | 60 fps (no long tasks > 50 ms during animations) | same |
| Initial JS (menu route, gzip) | ≤ 130 KB total, of which framework ≤ 90 KB, app ≤ 40 KB | — |
| Animation code | ≤ 18 KB gzip | — |
| CSS | ≤ 25 KB gzip, critical CSS inlined | — |
| Fonts | ≤ 90 KB (one pairing), +60 KB only in Hindi mode | — |
| Snapshot | ≤ 60 KB gzip (200 items) | — |
| Above-the-fold images | ≤ 150 KB total | ≤ 60 KB (save-data mode) |
| Requests before first paint | ≤ 4 (HTML, CSS inline, 1 font preload, cover image) | — |

Budgets are enforced in CI (bundle size check fails the build) and measured in production with real-user monitoring (web-vitals beacon).

### 12.3 Guest app techniques

1. **Server-rendered menu**: the menu list is a React Server Component; item cards are plain HTML/CSS. No client JavaScript is required to see the menu, scroll it, or read prices.
2. **Islands of interactivity**: only the add buttons, cart bar, item sheet, tabs scroll-spy, and search are client components. Identity (G-05/G-06), realtime client, order status, and bill views are **lazy-loaded** when first needed and **prefetched on idle** after the menu is interactive.
3. **Same-origin API** under `hotl.in/api` → no CORS preflight, one TLS connection reused (HTTP/2/3).
4. **Early hints & preload**: `103 Early Hints` / `<link rel=preload>` for CSS, the chosen font, the snapshot JSON, and the cover image.
5. **No layout shift**: every image has width/height from the snapshot; ThumbHash placeholder painted instantly.
6. **Client-side search** over the snapshot (works offline).
7. **Long lists**: `content-visibility: auto` on category sections so off-screen sections cost nothing to render; no virtualisation library needed under ~500 items.
8. **Main thread hygiene**: snapshot parsing and search-index building happen in `requestIdleCallback` chunks; no third-party scripts on the guest app.
9. **Save-Data / slow connection** (`navigator.connection.saveData` or `effectiveType` in `slow-2g`/`2g`/`3g`): serve smaller image sizes, skip "Popular here" images, defer beacons.
10. **Captive portals** (hotel Wi-Fi login pages): if an API response is HTML instead of JSON, the client shows "Your Wi-Fi may need you to sign in" instead of a generic error.

### 12.4 Image pipeline

1. Admin requests an upload URL → uploads the original directly to object storage (no API bandwidth), with progress.
2. `POST /media/{id}/complete` → job `media.process`:
   - Validate real type by magic bytes (JPEG/PNG/WebP/HEIC), max 20 MB, max 40 megapixels.
   - Auto-orient, strip all metadata (EXIF/GPS), convert to sRGB.
   - Produce variants: `thumb_160` (1:1), `card_480` & `card_960` (4:3), `grid_400` & `grid_800` (1:1), `hero_1080` (4:3) — each in **AVIF (q≈50) and WebP (q≈75)**.
   - Compute ThumbHash.
   - Mark `ready`; emit `media.ready` (admin UI swaps the processing spinner for the image).
3. Guest markup uses `<picture>` with AVIF → WebP sources and `srcset`/`sizes`, `loading="lazy"` below the fold, `fetchpriority="high"` for the first visible card image and the cover, `decoding="async"`.
4. Typical card image: 18–35 KB AVIF.

### 12.5 Offline and cart persistence (guest)

| Data | Storage | Strategy |
|------|---------|----------|
| App shell (JS/CSS/fonts) | Service worker precache | Cache-first, updated in background on deploy |
| Menu snapshot | SW runtime cache | Cache-first (immutable URL) |
| Images | SW runtime cache (max 200 entries, LRU) | Cache-first |
| `/q/{token}` HTML | SW runtime cache | Network-first with 3 s timeout → cached copy (so a re-scan offline still opens the menu) |
| Availability | Memory + SW cache | Network-first, 3 s timeout → last known (marked stale) |
| Cart | IndexedDB (`cart:{outletId}`) | Written on every change; mirrored to server debounced 2 s (best effort) |
| Order outbox | IndexedDB | §10.1 |
| Identity hints | IndexedDB (first name only) | Server cookie is the real identity |

- The service worker is registered after the first successful load (never blocks first paint).
- Online/offline detection combines `navigator.onLine` with real request outcomes (the browser's flag is often wrong on captive or congested Wi-Fi).

### 12.6 Admin, staff, and KDS performance

- TanStack Query with `staleTime` tuned per resource (board: realtime-driven; menu lists: 30 s; reports: 5 min).
- Optimistic updates for availability toggles, status changes, and reorder drags; automatic rollback with a toast on failure.
- Virtualised tables for long lists (customers, orders history, audit).
- Charts, image cropper, spreadsheet import, and PDF previews are lazy-loaded per route.
- KDS runs for a full shift: no memory growth (bounded ticket list; completed tickets removed after 10 min), Wake Lock re-acquired on visibility change, automatic soft reload at 4 AM business-day cutoff when idle.

### 12.7 Server performance

| Operation | p95 target (server time) |
|-----------|-------------------------|
| QR resolve (cache miss) | < 50 ms |
| Availability (Redis) | < 10 ms |
| Cart validate | < 120 ms |
| Place order | < 400 ms |
| Staff status change | < 150 ms |
| Board load (100 active orders) | < 200 ms |
| Dashboard today | < 150 ms (Redis counters + aggregates) |
| Report (30 days) | < 800 ms (aggregate tables, replica) |

Techniques: in-process LRU cache of parsed snapshots (keyed by outlet+version), Redis for hot data, prepared statements, no N+1 queries (repositories load aggregates in ≤ 3 queries), aggregates for reporting, replica for read-heavy admin screens, connection pooling.

---

## 13. Error handling

### 13.1 Principles

1. **Never lose a guest's work.** Carts and pending orders survive every error.
2. **Never lie about state.** "Confirmed" only after the server confirms. Pending states are shown honestly.
3. **Every error says what to do next** (retry, change something, call a waiter) — in the guest's language.
4. **Fail closed on security, fail open on experience.** A tenant check failure denies access; a failed analytics beacon or recommendation row is silently skipped.
5. **One error format** across the API; one error-to-message mapping per client.
6. **Every server error is traceable** by `requestId` from the guest's screen to the log line.

### 13.2 Error response format

All API errors use RFC 9457 `application/problem+json`:

```json
{
  "type": "https://docs.hotl.in/errors/CART_CHANGED",
  "title": "Cart changed",
  "status": 409,
  "code": "CART_CHANGED",
  "detail": "Some items changed since the cart was built.",
  "retryable": false,
  "requestId": "req_01J…",
  "fields": [ { "path": "lines[0].quantity", "code": "TOO_LARGE", "message": "Max 20" } ],
  "meta": { "changes": [ { "clientLineId": "l1", "kind": "price_changed", "oldPaise": 28000, "newPaise": 30000 } ] }
}
```

- `code` is stable and is what clients switch on; `title`/`detail` are English developer text, never shown to guests directly.
- `fields` present for validation errors; `meta` carries structured data for the client to render its own message.
- 5xx responses never include stack traces, SQL, or internal names.

### 13.3 Error code catalogue

| Code | HTTP | When | Retryable | Client behaviour |
|------|------|------|-----------|------------------|
| `VALIDATION_FAILED` | 400 | Body/query fails schema | No | Inline field errors |
| `UNAUTHENTICATED` | 401 | No/expired session | No | Admin: go to login (return URL kept). Guest: re-create device cookie silently |
| `IDENTITY_REQUIRED` | 401 | Tenant needs verified guest | No | Open identity step, then resume order |
| `OTP_INVALID` | 401 | Wrong code | No | "That code isn't right · n tries left" |
| `OTP_EXPIRED` | 401 | Code older than 5 min | No | Offer resend |
| `OTP_TOO_MANY_ATTEMPTS` | 429 | 5 wrong codes | After wait | Show wait time |
| `OTP_SEND_LIMIT` | 429 | Send limits (§14.6) | After wait | Show wait time |
| `OTP_DELIVERY_FAILED` | 502 | SMS provider error | Yes | "Couldn't send the code · Try again"; after 2 failures, offer to continue without OTP if tenant allows unverified |
| `PHONE_INVALID` | 400 | Not a valid Indian mobile | No | Inline |
| `FORBIDDEN` | 403 | Missing permission | No | "You don't have access to this. Ask your manager." |
| `FEATURE_NOT_IN_PLAN` | 403 | Feature flag off | No | Lock icon + upgrade message |
| `PLAN_LIMIT_REACHED` | 403 | Limit exceeded | No | "You've reached your plan's limit of {n} {things}. Contact us to upgrade." |
| `DISCOUNT_LIMIT_EXCEEDED` | 403 | Above role's max discount | No | Show allowed max; offer manager PIN override |
| `NOT_FOUND` | 404 | Unknown or other tenant's resource | No | Context message |
| `QR_NOT_FOUND` | 404 | Unknown token | No | G-E2 |
| `QR_REVOKED` | 410 | Regenerated/disabled QR | No | G-E2 |
| `TENANT_UNAVAILABLE` | 423 | Tenant suspended/cancelled | No | G-E3 |
| `OUTLET_CLOSED` | 423 | Outside hours / closure | No | Banner with next opening time; browsing allowed if enabled |
| `ORDERING_PAUSED` | 423 | Manual pause | No | Banner with message and resume time |
| `CART_CHANGED` | 409 | Price/availability/option change since snapshot | No | Show change list, update cart on confirm |
| `ITEM_UNAVAILABLE` | 409 | Item sold out/hidden (single-item actions) | No | "Just sold out" |
| `MENU_NOT_ACTIVE_AT_SPOT` | 422 | Item's menu not available at this spot/time | No | Remove line with explanation |
| `MODIFIER_RULES_FAILED` | 422 | Min/max selection unmet | No | Highlight group |
| `QUANTITY_LIMIT` | 422 | Over per-line or per-order max | No | Clamp + message |
| `ORDER_NOT_CANCELLABLE` | 409 | Already preparing | No | "Kitchen has started · Call waiter" |
| `INVALID_STATE_TRANSITION` | 409 | E.g. serve a cancelled order | No | Refresh entity, show current state |
| `CONFLICT_STALE_VERSION` | 409 | Optimistic lock failed | Auto | Refetch, re-apply if safe (status changes), else ask user |
| `SPOT_OCCUPIED` | 409 | Move to a spot with active session | No | Offer merge instead |
| `SESSION_CLOSED` | 409 | Ordering into a closed session | No | Start new session (automatic for guests on re-scan) |
| `BILL_ALREADY_SETTLED` | 409 | Payment/discount after settle | No | Show settled bill |
| `BILL_HAS_DUE` | 409 | Settle with due > 0 | No | Show due |
| `PAYMENT_EXCEEDS_DUE` | 422 | Non-cash overpayment | No | Prefill due amount |
| `IDEMPOTENCY_KEY_REUSED` | 422 | Same key, different body | No | Client bug → log to Sentry, regenerate key |
| `REQUEST_IN_PROGRESS` | 409 | Same key still processing | Yes (1 s) | Keep spinner, retry |
| `RATE_LIMITED` | 429 | Generic limit | After `Retry-After` | Back off; guest copy "Please wait a moment" |
| `SERVICE_REQUEST_COOLDOWN` | 429 | Same request < 60 s | No | "We've let them know" (treated as success in UI) |
| `UPLOAD_INVALID` | 422 | Wrong type/too big/corrupt | No | Explain limits |
| `IMPORT_INVALID` | 422 | Spreadsheet errors | No | Row-by-row report |
| `RESYNC_REQUIRED` | 410 | `since` too old | No | Full reload of board/KDS |
| `CLIENT_UPGRADE_REQUIRED` | 426 | Old client build | No | Reload app |
| `DEPENDENCY_UNAVAILABLE` | 503 | DB/Redis/provider down | Yes | Retry with backoff |
| `INTERNAL` | 500 | Unexpected | Yes (idempotent only) | Generic message + requestId |

### 13.4 Retries, timeouts, and idempotency

**Client retry policy** (built into `packages/sdk`):
- Retry only when the request is safe (GET) or carries an `Idempotency-Key`, and only on network errors, timeouts, `408`, `409 REQUEST_IN_PROGRESS`, `429`, `502`, `503`, `504`.
- Backoff: 0.5 s, 1 s, 2 s, 4 s, 8 s, then every 15 s, each with ±30 % jitter; honour `Retry-After`. Guest order outbox keeps retrying until success, a non-retryable error, or the 30-minute staleness limit (§10.1).
- Timeouts: reads 6 s, order placement 8 s, uploads per-chunk 30 s. A timeout is treated as "unknown outcome" — retry with the same key, never a new one.

**Idempotency store** (§7.12):
- Key scope: `(tenant, operation, key)`. Request hash = SHA-256 of canonical JSON body.
- `in_progress` rows older than 30 s are considered crashed and may be taken over.
- Responses stored for 24 h; replays return the identical status and body with `Idempotent-Replayed: true`.
- Natural idempotency is used where possible: staff status changes are idempotent by target state; bill request is idempotent per session; webhook events by provider event ID.

**Server-side retries**: database serialization failures and deadlocks (`40001`, `40P01`) are retried up to 3 times inside the API with 20–80 ms jitter before returning `503`.

### 13.5 Data integrity rules

- All multi-row writes in a single transaction; events via outbox in the same transaction.
- Database constraints are the final guard: unique partial indexes (one session per spot, idempotency keys, bill numbers), `CHECK` constraints on amounts and statuses, foreign keys.
- Optimistic locking (`version`) on orders, bills, sessions, items; pessimistic `FOR UPDATE` only for short critical sections (session lookup, counters, bill number allocation), always locking in a consistent order.
- Bills are recomputed from source lines every time (no drift); a nightly consistency job recomputes all bills settled that day and alerts on any mismatch.
- Snapshots of names/prices on order lines make history immutable.

### 13.6 Failure scenarios

| # | Scenario | System behaviour | Guest / staff sees |
|---|----------|------------------|--------------------|
| 1 | Guest's network drops while browsing | Snapshot and images already cached; cart in IndexedDB | Offline banner; everything keeps working |
| 2 | Network drops right after "Place order" (request may or may not have arrived) | Outbox retries with same key; server returns original response if it was created | "Sending your order…" → "Sent" — exactly one order |
| 3 | Guest double-taps "Place order" | Button disabled after first tap; same key anyway | One order |
| 4 | Guest closes the browser while order pending | On next open, outbox resends (< 30 min) | Order appears; or "This order wasn't sent. Send now?" if older |
| 5 | Item sells out between adding and ordering | Validation/`CART_CHANGED` | "Paneer Tikka just sold out" with option to remove or swap |
| 6 | Price changes while guest has item in cart | `CART_CHANGED` with old/new price | Must confirm new price; never silently charged |
| 7 | Two guests at an empty table order at the same instant | Unique partial index on `session_spots`; the loser re-reads and joins the session | Both orders on one bill |
| 8 | Prank order from a photographed QR at home | First order of session needs staff confirm (D-14); rate limits; staff can decline as "suspicious", which blocks that device for the outlet for 24 h | Staff sees "New" card with "First order — confirm"; guest sees "Declined" |
| 9 | Kitchen tablet loses Wi-Fi | KDS shows red light + alarm after 10 s; queues bump actions locally; resyncs on reconnect; manager board still shows orders | Staff alerted immediately; no tickets lost |
| 10 | Realtime gateway crashes | Clients reconnect to another instance; resync via `changes?since`; 30 s polling safety net | Brief "Reconnecting…" |
| 11 | API instance crashes mid-transaction | Transaction rolls back; idempotency row times out after 30 s; client retry succeeds | Slight delay in "Sent" |
| 12 | Redis unavailable | API degrades: availability falls back to DB query (cached in-process 5 s); rate limits fall back to in-memory per instance; queues pause and resume; realtime falls back to polling | Slightly slower; no errors on ordering |
| 13 | Primary database failover (managed, ~30–60 s) | API returns `503 DEPENDENCY_UNAVAILABLE`; clients retry; guest outbox holds orders; menu browsing unaffected (CDN) | "Sending…" a little longer |
| 14 | SMS provider slow/down | 10 s timeout; secondary provider tried (if configured); if tenant allows, continue as unverified | "Didn't get it? Resend" → fallback message |
| 15 | Image processing fails | Job retries 3× then marks `failed`; item keeps previous photo or text layout | Admin: "Photo couldn't be processed · Try another" |
| 16 | Snapshot build fails validation | Publish aborted, current version stays live | Admin: list of blocking problems with links |
| 17 | Admin and manager edit the same item at once | Version conflict on save | "Someone else changed this item. Review their changes?" with diff |
| 18 | Staff confirms an order that guest just cancelled | State machine rejects | Card updates to "Cancelled by guest" |
| 19 | Merge/move while another staff settles | Row locks + version checks; second action gets `409` | "This table was just settled" |
| 20 | Settle bill on two devices simultaneously | Bill row lock + `BILL_ALREADY_SETTLED` | Second device shows settled bill |
| 21 | Outlet's business day rolls over at 4 AM with open sessions | Sessions keep their opening `business_date`; new orders keep that date while the session is open | Reports stay correct |
| 22 | Clock skew on guest phone | Server time used for everything; `clientCreatedAt` only informational | — |
| 23 | Tenant suspended mid-service | Ordering blocked with `TENANT_UNAVAILABLE`; open sessions can still be settled by staff (read/settle-only mode for 24 h) | Guests: "Ordering is unavailable right now. Please ask the staff." |
| 24 | Old guest app version cached in a phone | API returns `426` for incompatible clients; SW updates and reloads at the next safe moment (never during checkout) | Seamless refresh |
| 25 | Captive portal Wi-Fi | Detected by non-JSON response | "Your Wi-Fi may need you to sign in" |

### 13.7 Guest error copy (English / Hindi)

All strings live in `packages/i18n/{en,hi}/errors.json`. Tone per doc 03 §8.

| Key | English | हिंदी |
|-----|---------|-------|
| `offline` | You're offline — your cart is saved. | आप ऑफ़लाइन हैं — आपका कार्ट सुरक्षित है। |
| `back_online` | Back online. | आप फिर से ऑनलाइन हैं। |
| `order_sending` | Sending your order… | आपका ऑर्डर भेजा जा रहा है… |
| `order_still_sending` | Still sending — your order is saved and we'll keep trying. | अभी भी भेज रहे हैं — आपका ऑर्डर सुरक्षित है, हम कोशिश करते रहेंगे। |
| `order_stale` | This order wasn't sent. Send it now? | यह ऑर्डर भेजा नहीं गया था। क्या अभी भेजें? |
| `cart_changed` | A few things in your cart have changed. | आपके कार्ट में कुछ चीज़ें बदल गई हैं। |
| `item_sold_out` | {item} just sold out. | {item} अभी-अभी ख़त्म हो गया। |
| `price_changed` | {item} is now ₹{new} (was ₹{old}). | {item} अब ₹{new} का है (पहले ₹{old} था)। |
| `ordering_paused` | Ordering is paused for a few minutes. You can keep browsing. | कुछ मिनटों के लिए ऑर्डर लेना रोका गया है। आप मेन्यू देखते रह सकते हैं। |
| `outlet_closed` | We're closed right now. We open at {time}. | अभी हम बंद हैं। हम {time} बजे खुलेंगे। |
| `qr_inactive` | This code is no longer active. Please ask the staff for help. | यह कोड अब सक्रिय नहीं है। कृपया स्टाफ़ से मदद लें। |
| `order_declined` | The restaurant couldn't accept this order: {reason} | रेस्टोरेंट यह ऑर्डर स्वीकार नहीं कर सका: {reason} |
| `not_cancellable` | The kitchen has already started on this. Ask your waiter if you need to change it. | किचन ने इसे बनाना शुरू कर दिया है। बदलाव के लिए अपने वेटर से कहें। |
| `phone_invalid` | Please enter a valid 10-digit mobile number. | कृपया सही 10 अंकों का मोबाइल नंबर डालें। |
| `otp_invalid` | That code isn't right. {n} tries left. | यह कोड सही नहीं है। {n} कोशिशें बाकी हैं। |
| `otp_expired` | This code has expired. Send a new one. | इस कोड की समय-सीमा ख़त्म हो गई है। नया कोड भेजें। |
| `otp_locked` | Too many tries. Please try again in {m} minutes. | बहुत ज़्यादा कोशिशें हो गईं। कृपया {m} मिनट बाद फिर कोशिश करें। |
| `otp_send_failed` | We couldn't send the code. Please try again. | कोड नहीं भेज पाए। कृपया फिर से कोशिश करें। |
| `request_sent` | We've let them know. | हमने स्टाफ़ को बता दिया है। |
| `session_closed` | This table's bill was settled. Scan again to start a new order. | इस टेबल का बिल चुका दिया गया है। नया ऑर्डर शुरू करने के लिए फिर से स्कैन करें। |
| `captive_portal` | Your Wi-Fi may need you to sign in. | आपके Wi-Fi में साइन इन करना पड़ सकता है। |
| `rate_limited` | Please wait a moment and try again. | कृपया थोड़ा रुककर फिर से कोशिश करें। |
| `generic` | Something went wrong. Please try again. (Ref: {requestId}) | कुछ गड़बड़ हो गई। कृपया फिर से कोशिश करें। (संदर्भ: {requestId}) |

Hindi copy must be reviewed by a native speaker before launch.

### 13.8 Where errors are shown

| Situation | Pattern |
|-----------|---------|
| Field problem | Inline under the field |
| Action failed, user can retry | Toast with "Retry" (stays until dismissed) |
| Something changed (cart) | Inline change list in context |
| Connection state | Thin persistent banner |
| Screen cannot load at all | Full-screen friendly state with Retry (and cached content if available) |
| Unexpected crash in a component | React error boundary per screen section; rest of the app keeps working; Sentry report |

### 13.9 Server and job error handling

- A single Fastify error handler maps `AppError` → problem JSON; unknown errors → `500 INTERNAL`, logged with stack and request context, reported to Sentry.
- Logs never contain OTP codes, full phone numbers, passwords, tokens, or card data (redaction list in the logger config).
- Background jobs: 3–5 retries with exponential backoff; then moved to a **dead-letter queue** with an alert; jobs are idempotent (safe to run twice).
- Graceful shutdown: stop accepting requests, finish in-flight (≤ 10 s), close sockets with a reconnect hint.

---

## 14. Security, access control, and privacy

### 14.1 Staff and platform authentication

| Aspect | Rule |
|--------|------|
| Passwords | argon2id (memory 64 MB, iterations 3), min 10 characters, checked against a breached-password list |
| Phone OTP login | Same OTP service as guests, separate rate limits |
| Sessions | Random 256-bit ID in `sid` cookie (httpOnly, Secure, SameSite=Lax); data in Redis; idle timeout 12 h, absolute 30 days; rotated on login and privilege change |
| CSRF | Double-submit token header on state-changing requests |
| Brute force | 5 failed logins per account per 15 min → 15 min lock + notification; per-IP limits |
| Quick switch | Personal PIN on shared devices; 5 wrong PINs → switch disabled for 5 min; PIN actions are attributed to that staff member |
| 2FA | TOTP optional for owners (P1, recommended); **mandatory** for platform admins |
| Deactivation | Deactivating staff deletes their Redis sessions and disconnects their sockets (≤ 60 s, ADM-STF-005) |
| Platform admins | Separate app/domain, separate session store namespace, TOTP required, optional IP allowlist; support access to a tenant only through a `support_sessions` record (PLT-005), shown to the owner |

### 14.2 Device pairing (KDS, counter tablets)

1. Unpaired device opens `/kds` → `POST /kds/pairing/start` → shows a 6-digit code (valid 10 min) and polls.
2. Manager (permission `devices.manage`) enters the code in A-52, chooses outlet, station, and name.
3. API issues a device token (random 256-bit, stored hashed), bound to tenant + outlet + station, scope `kds` or `counter`.
4. Device stores token in an httpOnly cookie; token never expires on its own but can be revoked instantly (A-52), which disconnects the device.
5. Device tokens can only call the KDS surface; counter devices additionally require a staff PIN for any billing action.

### 14.3 Roles and permissions

Built-in roles are seeded per tenant from templates; owners can create custom roles (P1, feature `custom_roles`). Outlet scope comes from the membership.

| Permission | Owner | Manager | Cashier | Waiter | Kitchen |
|------------|:-----:|:-------:|:-------:|:------:|:-------:|
| `dashboard.view` | ✓ | ✓ | — | — | — |
| `reports.view` | ✓ | ✓ | Day reports only | — | — |
| `reports.export` | ✓ | ✓ | — | — | — |
| `menu.view` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `menu.edit` (items, categories, photos, modifiers) | ✓ | ✓ | — | — | — |
| `menu.price_edit` | ✓ | ✓ | — | — | — |
| `menu.publish` | ✓ | ✓ | — | — | — |
| `availability.edit` (sold out) | ✓ | ✓ | ✓ | ✓ | ✓ |
| `spots.manage` / `qr.manage` | ✓ | ✓ | — | — | — |
| `branding.manage` | ✓ | ✓ | — | — | — |
| `orders.view` | ✓ | ✓ | ✓ | ✓ | ✓ (KDS) |
| `orders.create` (waiter order) | ✓ | ✓ | ✓ | ✓ | — |
| `orders.confirm` / decline | ✓ | ✓ | ✓ | ✓ | — |
| `orders.update_status` | ✓ | ✓ | ✓ | ✓ (served) | ✓ (preparing/ready) |
| `orders.cancel` (after confirm) | ✓ | ✓ | ✓ | — | — |
| `sessions.move_merge` | ✓ | ✓ | ✓ | ✓ | — |
| `bills.view` | ✓ | ✓ | ✓ | ✓ | — |
| `bills.discount` | ✓ (no limit) | ✓ (≤ 30 %) | ✓ (≤ 10 %) | — | — |
| `bills.remove_service_charge` | ✓ | ✓ | ✓ | ✓ | — |
| `payments.record` / `bills.settle` | ✓ | ✓ | ✓ | — | — |
| `bills.void` | ✓ | ✓ | — | — | — |
| `day_close.perform` | ✓ | ✓ | ✓ | — | — |
| `service_requests.handle` | ✓ | ✓ | ✓ | ✓ | — |
| `customers.view` (masked phone) | ✓ | ✓ | ✓ | ✓ | — |
| `customers.view_phone` (reveal, audited) | ✓ | ✓ | — | — | — |
| `customers.export` / `customers.delete` | ✓ | — | — | — | — |
| `staff.manage` | ✓ | ✓ (not owners/managers) | — | — | — |
| `roles.manage` | ✓ | — | — | — | — |
| `devices.manage` | ✓ | ✓ | — | — | — |
| `settings.manage` (taxes, ordering rules) | ✓ | ✓ (except legal/business profile) | — | — | — |
| `audit.view` | ✓ | ✓ (own outlet) | — | — | — |
| `plan.view` | ✓ | — | — | — | — |
| `kds.use` | ✓ | ✓ | — | — | ✓ |

Discount limits are stored in `roles.limits.max_discount_bp`; an over-limit discount can be approved on the spot by a manager entering their PIN (recorded as approved_by in the audit log).

### 14.4 Enforcement

1. Route declares `permission`, optional `feature`, and `outletScoped: true|false`.
2. Auth plugin loads membership (cached 60 s, invalidated on change) → checks permission → checks outlet in membership outlets → checks feature flag → sets tenant context (RLS).
3. Service-level policies check record-level rules (e.g. a manager can't deactivate an owner; a waiter can only mark `served`).
4. The admin app hides what a role can't do, but **the server is the only authority**.

### 14.5 Guest identity and OTP

- OTP: 6 random digits (CSPRNG), stored in Redis as `otp:{tenant}:{phoneHash}` → `{codeHash, attempts, expiresAt}`; TTL 5 min; compared in constant time; deleted on success.
- SMS template (DLT-registered) ends with `@hotl.in #123456` so Android's Web OTP API can auto-fill.
- On success: find or create `customers` row by `(tenant, phone_hash)`; record consent; link device for 180 days; rotate the guest device cookie (prevents session fixation).
- A guest can see their history only if the device is linked to that customer; anyone at the table sees the **current session** only with the spot token (proof they scanned the QR) and only first names.
- Unverified mode (OTP off): customer row created with `verified_at = null`; history visible only on the device that created it; admin sees "Unverified".

### 14.6 Rate limits

Token-bucket limits in Redis (sliding window), keyed as shown. Values are defaults, configurable by the platform.

| Action | Limit |
|--------|-------|
| OTP send | 3 per phone per 15 min; 10 per device per hour; 20 per IP per hour; 200 per tenant per hour (alert above) |
| OTP verify | 5 attempts per code; 10 per device per 15 min |
| Place order (guest) | 5 per device per 5 min; 20 per spot per 10 min |
| Service requests | 1 per type per spot per 60 s; 20 per device per hour |
| Cart validate | 60 per device per 5 min |
| Guest API general | 300 per device per 5 min; 1,000 per IP per 5 min (venues share IPs, so IP limits are generous) |
| Staff login | 5 failures per account per 15 min; 30 per IP per 15 min |
| Admin API general | 1,200 per user per 5 min |
| Exports | 10 per tenant per hour |

Suspicious patterns (many first-orders declined as "suspicious" from one device/IP, OTP pumping to premium numbers) raise platform alerts and can auto-block a device or IP range for an outlet.

### 14.7 Application security

| Area | Measure |
|------|---------|
| Transport | TLS 1.2+ only, HSTS (preload), HTTP→HTTPS redirect at the edge |
| Headers | Strict Content-Security-Policy (no inline scripts except hashed), `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` (camera only on admin upload pages), `frame-ancestors 'none'` |
| Input | Zod validation on every input; length limits on all strings; menu descriptions sanitised to the allowed markdown subset; React escapes output by default; no `dangerouslySetInnerHTML` except sanitised content |
| SQL | Parameterised queries only (Drizzle); RLS as backstop |
| Uploads | Direct-to-storage presigned URLs with size and content-type conditions; server-side magic-byte check; re-encode every image (removes malicious payloads and EXIF/GPS); originals never served publicly |
| Secrets | In the cloud provider's secrets manager; never in the repo; rotated yearly or on staff change |
| Encryption at rest | Managed database and storage encryption; additional application-level AES-256-GCM for customer phone numbers and TOTP secrets (envelope encryption, key ID stored with the ciphertext for rotation) |
| Dependencies | Lockfile, automated vulnerability alerts, monthly updates |
| WAF / bots | Edge WAF managed rules; bot challenge only on abusive patterns, never on normal guest flows |
| Audit | Sensitive actions (§7.12) logged with before/after; customer phone reveal and exports always logged |
| Backups | Managed point-in-time recovery (7–35 days) + daily logical backup stored in a separate account; restore rehearsed quarterly |

### 14.8 Privacy and India's DPDP Act 2023

**Roles under the Act**: each business is the **Data Fiduciary** for its customers' data; the platform is its **Data Processor**. The subscription terms include a data processing agreement. For staff accounts and platform operations, the platform is the Data Fiduciary.

| Requirement | Implementation |
|-------------|----------------|
| Notice | Shown before OTP (GST-ID-004): what is collected (name, phone, orders), purposes, the business's name, how to withdraw/delete. Available in English and Hindi. Versioned (`notice_version`). |
| Consent | Clear affirmative action (sending the OTP after the notice). Marketing is a separate, unticked opt-in. Stored in `consent_records`. |
| Purpose limitation | Data used only for ordering, order updates, history/recognition at that business, and (if opted-in) that business's marketing. Never shared across tenants; never sold. |
| Data minimisation | Only name and phone (D-17). Analytics beacons carry no personal data. |
| Access & correction | Guest profile shows the data; name editable. |
| Withdrawal & erasure | "Delete my data" (G-11) and admin anonymise (ADM-CUS-005): name → "Deleted guest", phone fields erased, device links revoked, consent withdrawn; order totals kept for tax law (no personal data). Completed immediately; confirmation shown. |
| Retention | §7.14; automatic anonymisation after 3 years of inactivity. |
| Security safeguards | §14.7. |
| Breach handling | Incident runbook: contain, assess, notify the Data Protection Board of India and affected Data Principals without delay (through the business as fiduciary), record the incident. |
| Children | The service is not directed at children; no age data collected. |
| Grievance | Each tenant sets a grievance contact (ADM-SET-009); the platform publishes its own. |

---

## 15. Insights engine

### 15.1 Data flow

```
order/bill/session events ─▶ live counters (Redis, today)      ─▶ "Today at a glance"
settled bills (bill.settled) ─▶ incremental aggregate job       ─▶ sales_daily / hourly / items …
nightly 05:00 per outlet     ─▶ full recompute of previous business day (corrects any drift)
hourly + nightly             ─▶ insight rules evaluate aggregates ─▶ insights table ─▶ dashboard cards
```

- Revenue is counted when a bill is **settled** (attributed to the session's `business_date`); "today" live tiles also show the value of open sessions separately ("₹12,400 settled · ₹3,200 open").
- Test orders and voided bills are excluded; voids appear in the discounts & voids report.

### 15.2 Metric definitions

| Metric | Definition |
|--------|-----------|
| Gross sales | Σ line gross (before discounts) of settled bills |
| Discounts | Σ discount amounts |
| Net sales | Gross − discounts (excl. tax, service charge, tips) |
| Service charge | Σ service charge collected |
| Taxes | Σ bill taxes by code and rate |
| Total collected | Σ succeeded payments − refunds |
| Orders | Count of non-cancelled orders |
| Bills | Count of settled, non-voided bills |
| Average bill value (ABV) | Net sales / bills |
| Items sold | Σ quantity of non-cancelled lines |
| Table turnover time | Median (settled_at − opened_at) for table sessions |
| Confirm time | Median (confirmed_at − placed_at) for manual-confirm orders |
| Prep time | Median (ready_at − confirmed_at) per order and per item |
| Cancellation rate | Cancelled orders / placed orders |
| New customers | Customers whose first visit is in the period |
| Returning rate | Bills with a customer who had a previous visit / bills with an identified customer |
| QR conversion | Sessions with an order / QR scans (per spot) |
| Peak hour | Hour with highest average net sales across the selected weekdays |

### 15.3 Insight rule format

Each rule is code in `apps/worker/src/insights/rules/*.ts` implementing:

```ts
interface InsightRule {
  key: string;                       // 'item.not_selling'
  schedule: 'hourly' | 'nightly' | 'weekly';
  minData: { days?: number; bills?: number };   // don't run on too little data
  evaluate(ctx: RuleContext): Promise<InsightCandidate[]>;
}
interface InsightCandidate {
  fingerprint: string;               // e.g. 'item.not_selling:<itemId>'
  severity: 'info' | 'opportunity' | 'warning' | 'critical';
  impactPaise?: number;              // estimated money at stake, used for ranking
  confidence: number;                // 0..1
  titleKey: string; bodyKey: string; // i18n keys with {placeholders}
  data: Record<string, unknown>;     // every number shown, for "Why am I seeing this?"
  actions: { labelKey: string; href: string }[];
  expiresAt: Date;
}
```

**Ranking**: `score = severityWeight × (1 + log10(1 + impact₹)) × confidence × freshness`. The dashboard shows the top 3–6 active insights, maximum 2 of the same rule. **Deduplication**: one active insight per fingerprint; a dismissed fingerprint can't reappear for 14 days (snooze: chosen duration) unless its severity increases.

### 15.4 Rule library (initial)

| Key | Trigger | Example card | Action |
|-----|---------|--------------|--------|
| `sales.trend_week` | Net sales last 7 days vs previous 7 days changes ≥ 10 % | "Revenue is up 18 % this week, mostly from weekend dinners." | View sales |
| `sales.day_vs_typical` | Yesterday's net sales ≥ 20 % below/above the same weekday's 4-week median | "Tuesday was 26 % quieter than a usual Tuesday." | View hourly |
| `item.top_seller` | Item with highest net sales in period, if share ≥ 10 % | "Iced Latte brought in 14 % of revenue this week." | Mark as Bestseller |
| `item.not_selling` | Visible, available item with 0 sales in 10+ days while outlet had ≥ 30 bills | "Cold Coffee hasn't sold in 12 days." | Add photo / hide item |
| `item.rising` | Item quantity up ≥ 50 % week over week (min 10 units) | "Paneer Wrap orders doubled this week." | Feature it |
| `item.falling` | Previously top-10 item down ≥ 40 % | "Masala Chai sales fell 45 % this week." | View item |
| `item.no_photo_low_views` | Item without photo with below-median add-to-cart rate | "Items with photos sell 2.1× more here. 8 items have no photo." | Add photos |
| `item.often_sold_out` | Marked sold out on ≥ 3 of last 7 days | "Butter Croissant sold out 4 times this week." | Review stock |
| `ops.peak_hours` | Weekly: top 2 hours by sales | "Your busiest time is 8–9 PM on Fri–Sat." | View heatmap |
| `ops.slow_prep` | Median prep time this week ≥ 25 % above 4-week median, or an item's prep ≥ 2× its set time | "Orders took 6 minutes longer to prepare this week." | View orders |
| `ops.slow_confirm` | Median confirm time > 3 min | "Guests waited 4 min on average for their first order to be confirmed." | Switch to auto-confirm |
| `ops.table_wait` | Spot with highest average wait / slowest turnover | "Table 4 has the longest wait for food." | View floor |
| `ops.service_requests` | Response time median > 3 min | "Waiter calls took 5 minutes on average to answer." | View requests |
| `customers.returning_share` | Returning rate change ≥ 5 points | "38 % of bills were from returning guests, up from 31 %." | View customers |
| `customers.at_risk_regulars` | Customers with ≥ 4 visits and no visit in 30 days, count ≥ 5 | "12 regulars haven't visited in a month." | View list |
| `customers.big_spenders` | Top 5 % customers by spend this month | "Your top 20 guests spent ₹1.2 lakh this month." | View list |
| `discounts.high` | Discounts > 8 % of gross sales, or one staff member's discounts ≥ 2× median | "Discounts were 11 % of sales this week, mostly by one cashier." | View discounts report |
| `cancellations.high` | Cancellation rate ≥ 5 % | "6 % of orders were cancelled, mostly 'item unavailable'." | Review availability |
| `menu.category_mix` | Category share shift ≥ 10 points | "Desserts are 4 % of sales — lower than similar cafés' typical 10 %." (benchmark only when ≥ 20 comparable tenants, anonymised) | View categories |
| `setup.checklist` | Setup incomplete (Hindi names missing for top items, no photos, QR never scanned on a spot) | "Add Hindi names to your 10 best sellers." | Open menu studio |

Every number in a card is clickable through to its source (ADM-DSH-005); "Why am I seeing this?" shows the rule condition and the data.

### 15.5 Future AI layer (D-12)

Optional later: a language model rewrites the top insights into a short daily summary and answers "Ask your data" questions. Guardrails: the model only receives pre-computed aggregates (no personal data), must cite the metric IDs it used, and every number in its output is verified against the data before display; if verification fails, the rule-based text is shown.

---

## 16. Payment plug-in point

The founder's existing payment integrator is attached in Phase 8 (D-03). Everything around it — bills, payments table, statuses, counter payments, realtime updates — is built earlier, so attaching it only needs an adapter and the guest "Pay online" screen.

### 16.1 Provider interface (`apps/api/src/modules/billing/payments/provider.ts`)

```ts
export interface PaymentProvider {
  readonly key: string;                          // 'founder_gateway'
  createPayment(input: {
    tenantId: string; billId: string; paymentId: string;  // our IDs
    amountPaise: number; currency: 'INR';
    customer?: { name: string; phoneE164: string };
    returnUrl: string; idempotencyKey: string;
  }): Promise<{ providerPaymentId: string; clientAction:
      | { type: 'redirect'; url: string }
      | { type: 'sdk'; payload: Record<string, unknown> }   // e.g. checkout options
      | { type: 'upi_intent'; url: string } }>;

  verifyWebhook(rawBody: Buffer, headers: Record<string, string>): boolean;
  parseWebhook(rawBody: Buffer): {
    providerEventId: string; providerPaymentId: string;
    status: 'succeeded' | 'failed' | 'pending'; amountPaise: number; raw: unknown;
  };
  fetchStatus(providerPaymentId: string): Promise<'succeeded' | 'failed' | 'pending'>;
  refund(input: { providerPaymentId: string; amountPaise: number; idempotencyKey: string })
    : Promise<{ providerRefundId: string; status: 'pending' | 'succeeded' | 'failed' }>;
}
```

Per-tenant gateway credentials (each business's own merchant account) are stored encrypted in `tenant_payment_configs` (`tenant_id`, `provider`, `credentials_enc`, `status`).

### 16.2 Flow

1. Guest taps **Pay online** → `POST /guest/sessions/{id}/payments` (Idempotency-Key) → API creates `payments` row (`pending`, amount = due) → `provider.createPayment` → returns `clientAction`.
2. Guest completes payment in the provider's UI.
3. Provider webhook → signature verified → stored by `providerEventId` (dedupe) → payment `succeeded`/`failed` → bill recalculated → `payment.recorded`, `bill.updated` events.
4. Guest app also polls `GET /guest/payments/{id}` after returning (webhooks can be late).
5. **Reconciliation job** every 5 min: payments `pending` for > 10 min → `fetchStatus`; mismatches alert.
6. If a payment succeeds after the bill was already settled at the counter (double payment), it's flagged for refund in the admin with one-click `refund`.

### 16.3 Rules

- Amount always comes from the server's bill due, never from the client.
- Webhook handlers are idempotent and order-independent (a `failed` after `succeeded` for the same payment is ignored and logged).
- Online payment success marks the bill `paid`; staff still "settle & clear" (or auto-settle if the tenant enables it).

---

## 17. Background jobs and notifications

| Queue | Jobs | Trigger | Retries |
|-------|------|---------|---------|
| `outbox` | Publish outbox events to Redis | Continuous + NOTIFY | Infinite (it's a loop) |
| `media` | Process uploaded images | Upload complete | 3, then failed |
| `snapshots` | Build menu snapshot | Publish / instant change (debounced 3 s) | 3 |
| `availability` | Rebuild Redis overlay; restore "sold out until end of day" at cutoff | Change / scheduled | 5 |
| `sessions` | Idle-timeout checks, auto-clear after settle | Every minute | — |
| `sms` | Send OTP / notifications via provider (primary → secondary) | API | 2 (fast) |
| `insights` | Incremental aggregates, hourly/nightly rules, nightly recompute | Events / cron per outlet time zone | 5 |
| `exports` | CSV/XLSX/PDF reports, customer exports, QR card PDFs, tenant export | Admin request | 3 |
| `imports` | Menu spreadsheet import | Admin request | 1 (report on failure) |
| `notifications` | Web push to staff (P1), daily owner summary (P1) | Events / cron | 3 |
| `maintenance` | Expire idempotency keys, carts, exports; anonymise inactive customers; partition creation; bill consistency check | Nightly | 3 |
| `payments` (Phase 8) | Reconciliation, webhook processing | Cron / webhook | 5 |

Each queue has its own concurrency and per-tenant fairness limit; failed jobs go to a dead-letter queue visible in the platform health screen.

---

## 18. Monitoring

| Signal | What we watch |
|--------|---------------|
| **Real-user metrics** (guest app) | LCP, INP, CLS, JS errors, offline rate, order outbox retry counts — by device class and network type |
| **API** | Request rate, p50/p95/p99 latency per route, error rate by code, idempotent replays |
| **Ordering health** | Orders per minute per outlet vs normal; order placement p95; time from order placed to visible on KDS (measured with an ack event from the KDS) |
| **Realtime** | Connected clients, reconnect rate, outbox lag |
| **Jobs** | Queue depth, job duration, failures, dead-letter count |
| **Dependencies** | DB CPU/connections/replication lag, Redis memory, SMS delivery rate and latency |
| **Business guards** | Tenants with zero orders during their usual service hours (possible outage at a venue), spike in declined/suspicious orders |

**Alerts** (to the founder's phone): order placement error rate > 1 % for 5 min; KDS delivery p95 > 5 s; outbox lag > 10 s; DB failover; SMS delivery < 90 %; dead-letter growth; any `500` spike. Logs are structured JSON with `requestId`, `tenantId`, `outletId`, retained 30 days. Traces via OpenTelemetry for order placement and publish paths.

---

## 19. Scaling and capacity

### 19.1 Targets (NFR-SCL-001)

| Dimension | Design target |
|-----------|---------------|
| Outlets | 5,000 |
| Concurrent guests (peak dinner hour, all India) | 100,000 |
| Order placement burst | 200 orders/s |
| Concurrent realtime connections | 60,000 (staff + KDS + guests with active orders) |
| Orders stored | ~1.5 M/day at full scale; ~550 M/year |

### 19.2 Load model

- Menu views: ~95 % served by the CDN (HTML 30 s cache per spot, snapshot immutable, images immutable). Origin load for menus at peak ≈ a few hundred requests/s.
- Availability polling: 100k guests ÷ 60 s ≈ 1,700 req/s, mostly absorbed by the 5 s edge cache (one origin request per outlet per 5 s ≈ 1,000 req/s at 5,000 outlets, each a Redis read < 1 ms).
- Order writes: 200/s × ~15 rows per order ≈ 3,000 row inserts/s — comfortable for a single well-sized PostgreSQL primary.

### 19.3 How each part scales

| Component | Strategy |
|-----------|----------|
| Guest/admin apps | Stateless; scale horizontally; CDN in front |
| API | Stateless; horizontal autoscaling on CPU and p95 latency |
| Realtime | Horizontal; Redis adapter fan-out; ~10k connections per instance |
| Worker | Horizontal per queue |
| PostgreSQL | Vertical first (largest gains, simplest); read replica for reports/admin lists; PgBouncer; monthly partitioning of `orders`, `order_items`, `order_events`, `audit_log` when they exceed ~100 M rows (partition key `created_at`, primary keys include it). Beyond that, shard by `tenant_id` (the schema already allows it because every query is tenant-scoped). |
| Redis | Separate instances for cache/rate-limit, queues, and pub/sub when load requires |
| Object storage / images | CDN; no origin scaling needed |

### 19.4 Hosting summary

Managed services in India (Mumbai region): managed PostgreSQL (primary + standby + read replica, automatic failover, point-in-time recovery), managed Redis, managed application hosting for the six apps with autoscaling, Cloudflare in front for CDN/WAF/TLS, Cloudflare R2 for storage. Three environments: **development** (local), **staging** (production-like, seeded demo tenants), **production**. Deploys are automatic from the main branch after checks pass, with instant rollback to the previous release.

---

## 20. Build phases

Each phase ends with something usable. Requirement IDs refer to doc 02; screen IDs to doc 04.

### Phase 0 — Foundations
**Scope**: monorepo (§4), design tokens and core components (doc 03), i18n setup (en/hi), database base schema (tenants, plans, users, memberships, roles, outlets), RLS (§5), auth for staff and platform (§14.1), error format (§13.2), SDK with retries/idempotency (§13.4), outbox + worker skeleton, platform admin: create tenant + invite owner (PLT-001, PLT-002).
**Exit**: founder creates a tenant; owner accepts invite and logs in; cross-tenant isolation check passes.

### Phase 1 — Menu, branding, and QR (digital menu)
**Scope**: ADM-ONB-001/002, ADM-BRD-*, ADM-TBL-001–005, ADM-MENU-001–015 (incl. Hindi fields), image pipeline (§12.4), snapshots + availability overlay (§9), guest G-01, G-02, G-03 (view only), G-13, G-E1–E3, Hindi switch (GST-MENU-011), search/filters (GST-MENU-005/006), service worker basics.
**Exit**: a real venue can publish its menu with photos, print QR cards, and guests browse it within the performance budgets (§12.2) in English and Hindi. *(This is already sellable as a premium digital menu.)*

### Phase 2 — Ordering and kitchen
**Scope**: GST-CART-001–009, GST-ORD-001–010, GST-ORD-012, GST-SRV-*, order pipeline (§10), sessions, realtime (§11), ADM-ORD-001–010, ADM-TBL-006/008, KDS-001–006, device pairing (§14.2), waiter ordering (A-16), outbox offline (§12.5).
**Exit**: a full dinner service runs on QR ordering + KDS + board with zero lost/duplicated orders through the scenarios in §13.6 (1–13).

### Phase 3 — Billing and counter settlement
**Scope**: bill calculator (§7.9), GST-PAY-001/002/004/006, ADM-BIL-001–008, ADM-BIL-010, taxes and settings (ADM-SET-001–006), bill numbers, receipts, day close.
**Exit**: bills match hand-calculated GST examples to the paisa; a day can be closed with a correct payment-method summary.

### Phase 4 — Guest identity and customers
**Scope**: GST-ID-001–009, GST-ID-012, OTP service (§14.5) with DLT SMS (Q-02), rate limits (§14.6), "Your usual"/reorder (GST-MENU-010, GST-ID-006/007), ADM-CUS-001–005, privacy flows (§14.8).
**Exit**: returning guests are recognised on the same and new devices; deletion works end to end.

### Phase 5 — Dashboard, reports, and insights
**Scope**: aggregates (§15.1), ADM-DSH-001–006, insight rules (§15.4), report exports.
**Exit**: owner sees accurate today tiles and at least 8 rule types producing sensible cards on pilot data.

### Phase 6 — Staff, roles, audit, and platform completion (first sellable full release)
**Scope**: ADM-STF-001–007 (PIN quick switch, audit log UI), permission matrix (§14.3), plan builder and feature/limit enforcement (PLT-003, §5.4), PLT-004/005/006/009, test mode & go-live (ADM-ONB-005), monitoring and alerts (§18).
**Exit**: the P0 acceptance checklist in doc 02 §11 is fully met.

### Phase 7 — Growth features (P1)
**Scope**: kitchen stations (ADM-ORD-011/012), coupons (ADM-BIL-009), feedback (GST-FBK-*), web push (ADM-NTF-002), menu import/export (ADM-ONB-003, ADM-MENU-017), upsells (GST-MENU-014, ADM-MENU-016), allergens display (GST-MENU-012), counter pickup (GST-QR-006, GST-ORD-013), room service extras (GST-ORD-014), tips (GST-PAY-005), floor layout editor (ADM-TBL-007), daily owner summary (ADM-DSH-007), admin in Hindi (ADM-SET-011), WhatsApp OTP (GST-ID-010), subscription invoices (PLT-007), announcements (PLT-008).

### Phase 8 — Online payments
**Scope**: attach the founder's existing gateway through §16 (Q-04), GST-PAY-003, reconciliation, refunds.
**Exit**: online and counter payments mix on one bill; reconciliation shows zero unexplained mismatches over a pilot week.

### Phase 9 — Advanced (P2)
**Scope**: thermal printing via print bridge (KDS-008, Q-05), shared table cart (GST-CART-010), all-day counts (KDS-007), AI helpers (ADM-ONB-004, ADM-MENU-018, ADM-DSH-008), customer segments and campaigns (ADM-CUS-006/007), Google review prompt (GST-FBK-003).
