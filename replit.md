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
│   └── depannage-auto/     # React + Vite frontend (main app)
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

## Application: ToiturePro

Full-stack web application for automotive breakdown service providers (prestataires de couverture et toiture).

### Apps
- **Admin Panel (`artifacts/admin-panel`): Interface de gestion à `/admin/`. Login : `admin@toiture-pro.fr` / `Admin123!``): Management interface at `/admin/`. Login: `admin@toiture-pro.fr` / `Admin123!`
- **Provider App (`artifacts/toiture-pro`): Application couvreur à `/`.`): Provider-facing app at `/`. Provider password: `Depannage123!`

### Features
- **Tableau de bord**: Dashboard with KPI cards, revenue charts, top providers, top cities by revenue, refusals by provider
- **Missions**: Full CRUD, status management (pending→confirmed→in_progress→completed/cancelled), vehicle type (citadine/berline/suv/moto_scooter/utilitaire/poids_lourd/autre), client payment method (especes/virement/carte_bancaire/cheque/compensation), price simulation (day/night rates by distance), GPS tracking, file attachments
- **Paiements**: Payment tracking with 20% commission auto-calculation, commission status management, summary cards
- **Prestataires**: CRUD with SIRET, raison sociale, TVA number, TVA status, specialties, storage management, password management
- **Refus de missions**: Providers can refuse missions with reason; mission goes back to pending; refusal count tracked per provider
- **WhatsApp**: Non-French phone numbers (+33/0033 excluded) show WhatsApp contact icon
- **Suivi GPS**: Real-time GPS tracking with shareable client link
- **Push Notifications**: Browser push notifications via Web Push API (VAPID) when mission assigned + 1h before scheduled time
- **Rapports mensuels**: CSV export for admin (all providers breakdown) and provider (personal stats)

### DB Schema
- `missions` - missions with client, vehicle (vehicleType, vehicleTypeOther), location, status, photos, clientPaymentMethod
- `providers` - providers with siret, companyName, tvaNumber, isSubjectToTva, refusalCount, towTruckPhoto, insuranceDocument, rating
- `payments` - payment records with commissionAmount (20%), commissionStatus, commissionProof
- `mission_refusals` - refusal history (providerId, missionId, reason)
- `availability` - prestataire availability settings
- `availability_slots` - blocked time periods
- `notifications` - in-app notifications (with providerId for provider-specific filtering)
- `push_subscriptions` - web push notification subscriptions (providerId, endpoint, keys)
- `mission_attachments` - file attachments for missions

### Key Routes (API)
- `GET/POST /api/missions` - List and create missions
- `GET/PATCH /api/missions/:id` - Get and update a mission
- `POST /api/missions/:id/confirm` - Confirm a mission
- `POST /api/missions/:id/complete` - Complete a mission (with clientPaymentMethod)
- `POST /api/missions/:id/refuse` - Refuse a mission (validates ownership, prevents duplicates)
- `GET /api/missions/:id/refusals` - Refusal history for a mission
- `POST /api/missions/simulate-price` - Price simulation (distanceKm, hour → price)
- `POST /api/missions/:id/assign` - Assign provider to mission
- `GET/POST /api/payments` - List and create payments (auto 20% commission)
- `PATCH /api/payments/:id` - Update payment/commission status
- `GET /api/payments/summary` - Payment summary with commission totals
- `GET/POST/PATCH/DELETE /api/providers` - Provider CRUD with SIRET/TVA fields
- `GET /api/providers/:id/refusals` - Provider refusal history
- `PUT /api/providers/:id/password` - Set provider password
- `GET /api/stats` - Statistics with topCities and topRefusals
- `GET /api/companies/search?q=...` - French company search (SIRENE API, returns siret/companyName/address/tvaNumber)
- `GET/POST /api/notifications` - Notifications management (filtered by providerId)
- `GET /api/push/vapid-key` - Get VAPID public key for push subscriptions
- `POST /api/push/subscribe` - Register push subscription for a provider
- `POST /api/push/unsubscribe` - Remove push subscription
- `GET /api/reports/admin/monthly?year=&month=` - CSV export of monthly admin report (all providers)
- `GET /api/reports/provider/:id/monthly?year=&month=` - CSV export of monthly provider report

### Price Simulation Logic
- Day (08h-20h): 0-7km→110€; 7-20km→110+(km-7)×20€; >20km→max(160, km×2.3)
- Night (20h-08h): 0-7km→140€; 7-20km→140+(km-7)×25€; >20km→max(200, km×2.8)

### Seed Data
Run `pnpm --filter @workspace/scripts run seed` to seed demo data (7 missions, 3 payments, 4 notifications, availability).

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
- App setup: `src/app.ts` — mounts CORS, JSON/urlencoded parsing, routes at `/api`
- Routes: `src/routes/index.ts` mounts sub-routers; `src/routes/health.ts` exposes `GET /healthz`
- Depends on: `@workspace/db`, `@workspace/api-zod`

### `artifacts/depannage-auto` (`@workspace/depannage-auto`)

React + Vite frontend for ToiturePro.

- Tech: React 19, Vite, TailwindCSS, shadcn/ui, React Query, Wouter, Recharts, Framer Motion
- Pages: Dashboard, Missions, Paiements, Historique, Disponibilités, Mon Profil
- Uses React Query hooks from `@workspace/api-client-react`

### `lib/db` (`@workspace/db`)

Database layer using Drizzle ORM with PostgreSQL. Exports a Drizzle client instance and schema models.

- `src/index.ts` — creates a `Pool` + Drizzle instance, exports schema
- `src/schema/index.ts` — barrel re-export of all models
- `drizzle.config.ts` — Drizzle Kit config (requires `DATABASE_URL`, automatically provided by Replit)
- Exports: `.` (pool, db, schema), `./schema` (schema only)

Production migrations are handled by Replit when publishing. In development, we just use `pnpm --filter @workspace/db run push`, and we fallback to `pnpm --filter @workspace/db run push-force`.

### `lib/api-spec` (`@workspace/api-spec`)

Owns the OpenAPI 3.1 spec (`openapi.yaml`) and the Orval config (`orval.config.ts`). Running codegen produces output into two sibling packages:

1. `lib/api-client-react/src/generated/` — React Query hooks + fetch client
2. `lib/api-zod/src/generated/` — Zod schemas

Run codegen: `pnpm --filter @workspace/api-spec run codegen`
