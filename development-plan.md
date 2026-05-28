# Marketplace Seller Management — Phased Development Plan

> Project: 140-marketplace-seller-management · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js 22 LTS) | API-heavy platform with real-time event processing (SP-API notifications, webhooks); strong ecosystem for marketplace API clients; shared types between API server and future frontend; excellent async/await for concurrent marketplace API calls |
| API framework | Fastify 5 | Fastest Node.js HTTP framework; built-in JSON Schema validation (aligns with marketplace product type schemas); OpenAPI 3.1 auto-generation via @fastify/swagger; first-class TypeScript support; plugin architecture maps cleanly to per-marketplace modules |
| Database | PostgreSQL 16 | JSONB columns with GIN indexes for marketplace-specific attributes (Data Model Suggestion 3); native partitioning for high-volume competitor observations and price change logs; NUMERIC type for precise monetary arithmetic; mature row-level security for multi-tenant isolation |
| ORM / query layer | Drizzle ORM | Type-safe SQL with zero runtime overhead; native PostgreSQL JSONB support; migration generation from schema definitions; avoids the query-builder overhead of Prisma while keeping full type safety |
| Task queue | BullMQ (Redis 7) | Handles async marketplace sync jobs, repricing cycles, and webhook processing; rate-limiting per queue maps to per-marketplace API rate limits; repeatable jobs for scheduled inventory syncs; Redis also serves as the cache layer |
| Cache | Redis 7 (shared with BullMQ) | Caches marketplace credentials (short TTL), competitor price snapshots, and Buy Box status; pub/sub for real-time repricing event propagation |
| Frontend | Next.js 15 (App Router) | React-based dashboard for seller management; server components for data-heavy analytics pages; API routes proxy to Fastify backend; Tailwind CSS + shadcn/ui for rapid UI development |
| Real-time updates | Server-Sent Events (SSE) | Lighter than WebSocket for unidirectional dashboard updates (order notifications, repricing events, inventory alerts); Fastify SSE plugin available; falls back to polling gracefully |
| Authentication | Lucia Auth + OAuth 2.0 | Lucia for session management and seller user auth; OAuth 2.0 client implementations per marketplace (Amazon LWA, eBay OAuth, TikTok HMAC, Walmart client credentials) |
| Containerisation | Docker + docker-compose | Multi-service architecture (API, worker, frontend, PostgreSQL, Redis); Dockerfile per service; docker-compose for local development |
| Testing | Vitest + Supertest + Playwright | Vitest for unit/integration tests (fastest TS test runner); Supertest for HTTP endpoint testing; Playwright for E2E dashboard tests; Testcontainers for PostgreSQL/Redis in integration tests |
| Code quality | Biome (lint + format) + tsc --noEmit | Biome replaces ESLint + Prettier with faster single tool; strict TypeScript with no implicit any; Husky + lint-staged for pre-commit hooks |
| Package manager | pnpm 9 | Workspace support for monorepo (api, worker, web, shared); strict dependency resolution; faster installs than npm/yarn |
| Key libraries | `@sp-api-sdk/*` (Amazon SP-API), `ebay-api` (eBay), `@sellercloud/walmart-api` or custom (Walmart), `ajv` (JSON Schema validation), `ioredis`, `zod` (runtime validation), `date-fns` (timezone-safe dates), `decimal.js` (monetary arithmetic) |
| Monorepo structure | pnpm workspaces | Packages: `@msm/api` (Fastify server), `@msm/worker` (BullMQ processors), `@msm/web` (Next.js), `@msm/shared` (types, schemas, utilities), `@msm/marketplace-connectors` (per-marketplace API clients) |

### Project Structure

```
marketplace-seller-management/
├── pnpm-workspace.yaml
├── package.json
├── tsconfig.base.json
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── .env.example
├── biome.json
├── drizzle.config.ts
├── packages/
│   ├── shared/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── types/
│   │       │   ├── seller.ts
│   │       │   ├── marketplace.ts
│   │       │   ├── product.ts
│   │       │   ├── listing.ts
│   │       │   ├── inventory.ts
│   │       │   ├── order.ts
│   │       │   ├── shipment.ts
│   │       │   ├── repricing.ts
│   │       │   └── analytics.ts
│   │       ├── schemas/
│   │       │   ├── product-schema.ts
│   │       │   ├── listing-schema.ts
│   │       │   └── order-schema.ts
│   │       ├── constants/
│   │       │   ├── marketplace-codes.ts
│   │       │   ├── order-statuses.ts
│   │       │   └── fulfillment-types.ts
│   │       └── utils/
│   │           ├── money.ts
│   │           ├── gtin.ts
│   │           └── rate-limiter.ts
│   ├── marketplace-connectors/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── connector-interface.ts
│   │       ├── amazon/
│   │       │   ├── client.ts
│   │       │   ├── listings.ts
│   │       │   ├── orders.ts
│   │       │   ├── pricing.ts
│   │       │   ├── notifications.ts
│   │       │   └── auth.ts
│   │       ├── ebay/
│   │       │   ├── client.ts
│   │       │   ├── listings.ts
│   │       │   ├── orders.ts
│   │       │   └── auth.ts
│   │       ├── walmart/
│   │       │   ├── client.ts
│   │       │   ├── listings.ts
│   │       │   ├── orders.ts
│   │       │   └── auth.ts
│   │       └── tiktok/
│   │           ├── client.ts
│   │           ├── listings.ts
│   │           ├── orders.ts
│   │           └── auth.ts
│   ├── api/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── server.ts
│   │       ├── plugins/
│   │       │   ├── auth.ts
│   │       │   ├── database.ts
│   │       │   └── swagger.ts
│   │       ├── routes/
│   │       │   ├── sellers.ts
│   │       │   ├── marketplaces.ts
│   │       │   ├── products.ts
│   │       │   ├── listings.ts
│   │       │   ├── inventory.ts
│   │       │   ├── orders.ts
│   │       │   ├── shipments.ts
│   │       │   ├── repricing.ts
│   │       │   ├── analytics.ts
│   │       │   ├── compliance.ts
│   │       │   └── webhooks.ts
│   │       ├── services/
│   │       │   ├── product-service.ts
│   │       │   ├── listing-service.ts
│   │       │   ├── inventory-service.ts
│   │       │   ├── order-service.ts
│   │       │   ├── repricing-service.ts
│   │       │   └── sync-service.ts
│   │       └── db/
│   │           ├── schema.ts
│   │           ├── migrations/
│   │           └── seed.ts
│   ├── worker/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts
│   │       ├── queues.ts
│   │       ├── processors/
│   │       │   ├── listing-sync.ts
│   │       │   ├── order-sync.ts
│   │       │   ├── inventory-sync.ts
│   │       │   ├── price-sync.ts
│   │       │   ├── repricing-engine.ts
│   │       │   ├── competitor-monitor.ts
│   │       │   └── analytics-aggregator.ts
│   │       └── schedulers/
│   │           ├── sync-scheduler.ts
│   │           └── analytics-scheduler.ts
│   └── web/
│       ├── package.json
│       ├── tsconfig.json
│       ├── next.config.ts
│       ├── tailwind.config.ts
│       └── src/
│           └── app/
│               ├── layout.tsx
│               ├── page.tsx
│               ├── (auth)/
│               │   ├── login/
│               │   └── register/
│               ├── (dashboard)/
│               │   ├── layout.tsx
│               │   ├── overview/
│               │   ├── products/
│               │   ├── listings/
│               │   ├── inventory/
│               │   ├── orders/
│               │   ├── repricing/
│               │   ├── analytics/
│               │   ├── compliance/
│               │   └── settings/
│               └── api/
│                   └── [...proxy]/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/
│       ├── amazon-order.json
│       ├── ebay-listing.json
│       ├── walmart-inventory.json
│       └── tiktok-product.json
└── scripts/
    ├── seed-marketplaces.ts
    ├── create-partition.ts
    └── migrate.ts
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose

Establish the monorepo, database schema, configuration system, and development toolchain. After this phase, every subsequent phase can run `pnpm dev` to start a working API server connected to PostgreSQL and Redis, with typed database queries, automated migrations, and a CI-ready test harness.

### Tasks

#### 1.1 — Monorepo Initialisation

**What**: Create the pnpm workspace with all five packages (`shared`, `marketplace-connectors`, `api`, `worker`, `web`), shared TypeScript configuration, Biome linting, and Docker Compose for local services.

**Design**:

```yaml
# pnpm-workspace.yaml
packages:
  - "packages/*"
```

```jsonc
// tsconfig.base.json
{
  "compilerOptions": {
    "target": "ES2024",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "dist",
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "skipLibCheck": true
  }
}
```

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: msm
      POSTGRES_USER: msm
      POSTGRES_PASSWORD: msm_dev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
volumes:
  pgdata:
```

```jsonc
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/2.0/schema.json",
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "rules": { "recommended": true }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  }
}
```

**Testing**:
- `Unit: pnpm install completes without errors across all workspace packages`
- `Unit: pnpm build compiles all packages with zero TypeScript errors`
- `Unit: pnpm lint runs Biome across all packages with no violations`
- `Integration: docker-compose up -d starts PostgreSQL and Redis, both accepting connections`

#### 1.2 — Database Schema & Migrations

**What**: Implement the Hybrid Relational + JSONB data model (Data Model Suggestion 3) using Drizzle ORM schema definitions, with initial migration and seed data for supported marketplaces.

**Design**:

Adopts Data Model Suggestion 3 because it balances relational integrity for core operational fields with JSONB flexibility for marketplace-specific attributes. This is the fastest path to supporting new marketplaces without schema migrations.

```typescript
// packages/api/src/db/schema.ts
import { pgTable, uuid, text, timestamp, numeric, integer, boolean, jsonb, char, unique, index } from "drizzle-orm/pg-core";

export const seller = pgTable("seller", {
  id: uuid("id").primaryKey().defaultRandom(),
  companyName: text("company_name").notNull(),
  contactEmail: text("contact_email").notNull(),
  contactPhone: text("contact_phone"),
  countryCode: char("country_code", { length: 2 }).notNull(),
  timezone: text("timezone").notNull().default("UTC"),
  subscriptionTier: text("subscription_tier").notNull().default("free"),
  status: text("status").notNull().default("active"),
  settings: jsonb("settings").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const sellerUser = pgTable("seller_user", {
  id: uuid("id").primaryKey().defaultRandom(),
  sellerId: uuid("seller_id").notNull().references(() => seller.id),
  email: text("email").notNull().unique(),
  displayName: text("display_name").notNull(),
  role: text("role").notNull().default("member"),
  passwordHash: text("password_hash"),
  lastLoginAt: timestamp("last_login_at", { withTimezone: true }),
  status: text("status").notNull().default("active"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const marketplace = pgTable("marketplace", {
  id: uuid("id").primaryKey().defaultRandom(),
  code: text("code").notNull().unique(),
  name: text("name").notNull(),
  region: text("region").notNull(),
  countryCode: char("country_code", { length: 2 }),
  currencyCode: char("currency_code", { length: 3 }).notNull(),
  apiType: text("api_type").notNull().default("rest"),
  apiConfig: jsonb("api_config").notNull().default({}),
  status: text("status").notNull().default("active"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

export const marketplaceCredential = pgTable("marketplace_credential", {
  id: uuid("id").primaryKey().defaultRandom(),
  sellerId: uuid("seller_id").notNull().references(() => seller.id),
  marketplaceId: uuid("marketplace_id").notNull().references(() => marketplace.id),
  authType: text("auth_type").notNull(),
  credentials: jsonb("credentials").notNull().default({}),
  status: text("status").notNull().default("active"),
  lastSyncedAt: timestamp("last_synced_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  unique().on(table.sellerId, table.marketplaceId),
]);

export const product = pgTable("product", {
  id: uuid("id").primaryKey().defaultRandom(),
  sellerId: uuid("seller_id").notNull().references(() => seller.id),
  sku: text("sku").notNull(),
  title: text("title").notNull(),
  description: text("description"),
  brand: text("brand"),
  gtin: text("gtin"),
  mpn: text("mpn"),
  category: text("category"),
  weightKg: numeric("weight_kg", { precision: 10, scale: 3 }),
  dimensions: jsonb("dimensions"),
  countryOfOrigin: char("country_of_origin", { length: 2 }),
  hsCode: text("hs_code"),
  images: jsonb("images").notNull().default([]),
  variants: jsonb("variants").notNull().default([]),
  customAttributes: jsonb("custom_attributes").notNull().default({}),
  status: text("status").notNull().default("active"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  unique().on(table.sellerId, table.sku),
]);

export const marketplaceProductType = pgTable("marketplace_product_type", {
  id: uuid("id").primaryKey().defaultRandom(),
  marketplaceId: uuid("marketplace_id").notNull().references(() => marketplace.id),
  productType: text("product_type").notNull(),
  displayName: text("display_name").notNull(),
  schemaDefinition: jsonb("schema_definition").notNull(),
  lastUpdatedAt: timestamp("last_updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  unique().on(table.marketplaceId, table.productType),
]);

export const listing = pgTable("listing", {
  id: uuid("id").primaryKey().defaultRandom(),
  sellerId: uuid("seller_id").notNull().references(() => seller.id),
  productId: uuid("product_id").notNull().references(() => product.id),
  marketplaceId: uuid("marketplace_id").notNull().references(() => marketplace.id),
  marketplaceListingId: text("marketplace_listing_id"),
  marketplaceSku: text("marketplace_sku"),
  productTypeId: uuid("product_type_id").references(() => marketplaceProductType.id),
  title: text("title").notNull(),
  price: numeric("price", { precision: 12, scale: 2 }).notNull(),
  currencyCode: char("currency_code", { length: 3 }).notNull(),
  quantityAvailable: integer("quantity_available").notNull().default(0),
  condition: text("condition").notNull().default("new"),
  fulfillmentType: text("fulfillment_type").notNull().default("merchant"),
  status: text("status").notNull().default("draft"),
  buyBoxOwned: boolean("buy_box_owned").default(false),
  marketplaceAttributes: jsonb("marketplace_attributes").notNull().default({}),
  syncStatus: text("sync_status").notNull().default("pending"),
  syncError: jsonb("sync_error"),
  aiContent: jsonb("ai_content"),
  lastSyncedAt: timestamp("last_synced_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  unique().on(table.sellerId, table.marketplaceId, table.marketplaceSku),
]);

// Remaining tables (warehouse, inventory, inventory_transaction,
// marketplace_order, shipment, repricing_rule, competitor_observation,
// price_change_log, daily_summary, compliance_alert, sync_job) follow
// the same pattern from Data Model Suggestion 3.
```

Seed script inserts marketplace reference data:

```typescript
// scripts/seed-marketplaces.ts
const MARKETPLACES = [
  { code: "amazon_us", name: "Amazon US", region: "na", countryCode: "US", currencyCode: "USD", apiType: "rest" },
  { code: "amazon_uk", name: "Amazon UK", region: "eu", countryCode: "GB", currencyCode: "GBP", apiType: "rest" },
  { code: "amazon_de", name: "Amazon DE", region: "eu", countryCode: "DE", currencyCode: "EUR", apiType: "rest" },
  { code: "ebay_us", name: "eBay US", region: "na", countryCode: "US", currencyCode: "USD", apiType: "rest" },
  { code: "ebay_uk", name: "eBay UK", region: "eu", countryCode: "GB", currencyCode: "GBP", apiType: "rest" },
  { code: "walmart_us", name: "Walmart US", region: "na", countryCode: "US", currencyCode: "USD", apiType: "rest" },
  { code: "tiktok_us", name: "TikTok Shop US", region: "na", countryCode: "US", currencyCode: "USD", apiType: "rest" },
  { code: "tiktok_uk", name: "TikTok Shop UK", region: "eu", countryCode: "GB", currencyCode: "GBP", apiType: "rest" },
] as const;
```

**Testing**:
- `Unit: Drizzle schema compiles with zero type errors`
- `Integration: drizzle-kit generate produces valid SQL migration files`
- `Integration: drizzle-kit migrate runs against empty PostgreSQL database without errors`
- `Integration: seed script inserts all 8 marketplace rows; SELECT count(*) FROM marketplace returns 8`
- `Unit: GTIN column accepts valid UPC-12 ("012345678901") and EAN-13 ("0123456789012")`
- `Unit: currency_code column rejects values longer than 3 characters`
- `Integration: unique constraint on (seller_id, marketplace_id) in marketplace_credential prevents duplicate connections`

#### 1.3 — Fastify API Server Bootstrap

**What**: Create the Fastify server with health check, database connection plugin, OpenAPI 3.1 documentation, request logging, and error handling middleware.

**Design**:

```typescript
// packages/api/src/server.ts
import Fastify from "fastify";
import cors from "@fastify/cors";
import swagger from "@fastify/swagger";
import swaggerUi from "@fastify/swagger-ui";
import { dbPlugin } from "./plugins/database";
import { errorHandler } from "./plugins/error-handler";

export async function buildServer() {
  const app = Fastify({
    logger: {
      level: process.env.LOG_LEVEL ?? "info",
      transport: process.env.NODE_ENV === "development"
        ? { target: "pino-pretty" }
        : undefined,
    },
    genReqId: () => crypto.randomUUID(),
  });

  await app.register(cors, { origin: process.env.CORS_ORIGIN ?? "http://localhost:3000" });

  await app.register(swagger, {
    openapi: {
      openapi: "3.1.0",
      info: {
        title: "Marketplace Seller Management API",
        version: "0.1.0",
        description: "Multi-marketplace listing, order, inventory, and repricing management",
      },
      servers: [{ url: process.env.API_BASE_URL ?? "http://localhost:4000" }],
    },
  });

  await app.register(swaggerUi, { routePrefix: "/docs" });
  await app.register(dbPlugin);
  app.setErrorHandler(errorHandler);

  app.get("/health", {
    schema: {
      response: {
        200: {
          type: "object",
          properties: {
            status: { type: "string" },
            timestamp: { type: "string", format: "date-time" },
            version: { type: "string" },
          },
        },
      },
    },
  }, async () => ({
    status: "healthy",
    timestamp: new Date().toISOString(),
    version: process.env.npm_package_version ?? "0.1.0",
  }));

  return app;
}
```

Environment configuration:

```typescript
// packages/shared/src/config.ts
import { z } from "zod";

export const EnvSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url().default("redis://localhost:6379"),
  API_PORT: z.coerce.number().default(4000),
  API_BASE_URL: z.string().url().default("http://localhost:4000"),
  CORS_ORIGIN: z.string().default("http://localhost:3000"),
  LOG_LEVEL: z.enum(["fatal", "error", "warn", "info", "debug", "trace"]).default("info"),
  SESSION_SECRET: z.string().min(32),
});

export type Env = z.infer<typeof EnvSchema>;

export function loadEnv(): Env {
  return EnvSchema.parse(process.env);
}
```

**Testing**:
- `Unit: loadEnv() with valid .env returns typed Env object`
- `Unit: loadEnv() with missing DATABASE_URL throws ZodError with field name`
- `Unit: loadEnv() applies defaults for optional fields (API_PORT=4000, LOG_LEVEL=info)`
- `Integration: GET /health returns 200 with status "healthy" and ISO 8601 timestamp`
- `Integration: GET /docs returns Swagger UI HTML`
- `Integration: GET /docs/json returns valid OpenAPI 3.1 JSON document`
- `Integration: server connects to PostgreSQL on startup; logs "database connected"`
- `Integration: invalid DATABASE_URL causes server to fail fast with descriptive error`

#### 1.4 — Shared Types & Utilities

**What**: Define the core TypeScript types, Zod validation schemas, monetary arithmetic utilities, and GTIN validation for use across all packages.

**Design**:

```typescript
// packages/shared/src/types/marketplace.ts
export type MarketplaceCode =
  | "amazon_us" | "amazon_uk" | "amazon_de" | "amazon_fr" | "amazon_jp"
  | "ebay_us" | "ebay_uk" | "ebay_de"
  | "walmart_us"
  | "tiktok_us" | "tiktok_uk";

export type MarketplaceRegion = "na" | "eu" | "apac" | "latam";
export type ApiType = "rest" | "graphql" | "edi" | "soap";
export type AuthType = "oauth2" | "api_key" | "hmac" | "as2";

export interface Marketplace {
  id: string;
  code: MarketplaceCode;
  name: string;
  region: MarketplaceRegion;
  countryCode: string | null;
  currencyCode: string;
  apiType: ApiType;
  apiConfig: Record<string, unknown>;
  status: "active" | "inactive";
}

// packages/shared/src/types/listing.ts
export type ListingStatus = "draft" | "active" | "inactive" | "suppressed" | "error";
export type SyncStatus = "pending" | "synced" | "error";
export type FulfillmentType = "merchant" | "fba" | "wfs" | "marketplace";
export type ItemCondition = "new" | "used_like_new" | "used_good" | "refurbished";

export interface Listing {
  id: string;
  sellerId: string;
  productId: string;
  marketplaceId: string;
  marketplaceListingId: string | null;
  marketplaceSku: string | null;
  title: string;
  price: string; // NUMERIC as string for precision
  currencyCode: string;
  quantityAvailable: number;
  condition: ItemCondition;
  fulfillmentType: FulfillmentType;
  status: ListingStatus;
  buyBoxOwned: boolean;
  marketplaceAttributes: Record<string, unknown>;
  syncStatus: SyncStatus;
  syncError: Record<string, unknown> | null;
  aiContent: Record<string, unknown> | null;
  lastSyncedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

// packages/shared/src/types/order.ts
export type OrderStatus =
  | "pending" | "confirmed" | "processing"
  | "shipped" | "delivered"
  | "cancelled" | "returned" | "refunded";

export interface MarketplaceOrder {
  id: string;
  sellerId: string;
  marketplaceId: string;
  marketplaceOrderId: string;
  orderStatus: OrderStatus;
  orderDate: string;
  currencyCode: string;
  subtotal: string;
  shippingCost: string;
  taxAmount: string;
  marketplaceFees: string;
  totalAmount: string;
  fulfillmentType: FulfillmentType;
  itemCount: number;
  shippingAddress: ShippingAddress | null;
  items: OrderItem[];
  marketplaceData: Record<string, unknown>;
  rawApiResponse: Record<string, unknown> | null;
}

export interface ShippingAddress {
  name: string;
  line1: string;
  line2?: string;
  city: string;
  state: string;
  postalCode: string;
  countryCode: string;
  phone?: string;
}

export interface OrderItem {
  marketplaceItemId: string;
  sku: string;
  title: string;
  quantity: number;
  unitPrice: string;
  tax: string;
  discount: string;
  lineTotal: string;
  listingId?: string;
  productId?: string;
}
```

```typescript
// packages/shared/src/utils/money.ts
import Decimal from "decimal.js";

Decimal.set({ precision: 20, rounding: Decimal.ROUND_HALF_UP });

export class Money {
  private constructor(
    public readonly amount: Decimal,
    public readonly currency: string,
  ) {}

  static of(amount: string | number, currency: string): Money {
    return new Money(new Decimal(amount), currency.toUpperCase());
  }

  static zero(currency: string): Money {
    return new Money(new Decimal(0), currency.toUpperCase());
  }

  add(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this.amount.plus(other.amount), this.currency);
  }

  subtract(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this.amount.minus(other.amount), this.currency);
  }

  multiply(factor: number | string): Money {
    return new Money(this.amount.times(factor), this.currency);
  }

  toFixed(dp = 2): string {
    return this.amount.toFixed(dp);
  }

  isGreaterThan(other: Money): boolean {
    this.assertSameCurrency(other);
    return this.amount.greaterThan(other.amount);
  }

  private assertSameCurrency(other: Money): void {
    if (this.currency !== other.currency) {
      throw new Error(`Currency mismatch: ${this.currency} vs ${other.currency}`);
    }
  }
}
```

```typescript
// packages/shared/src/utils/gtin.ts
export function validateGtin(gtin: string): { valid: boolean; type: string | null; error?: string } {
  const cleaned = gtin.replace(/\s|-/g, "");
  if (!/^\d+$/.test(cleaned)) {
    return { valid: false, type: null, error: "GTIN must contain only digits" };
  }
  const validLengths: Record<number, string> = {
    8: "GTIN-8",
    12: "UPC-A/GTIN-12",
    13: "EAN-13/GTIN-13",
    14: "GTIN-14",
  };
  const type = validLengths[cleaned.length];
  if (!type) {
    return { valid: false, type: null, error: `Invalid GTIN length: ${cleaned.length}. Expected 8, 12, 13, or 14 digits` };
  }
  // Check digit validation (GS1 standard)
  const digits = cleaned.split("").map(Number);
  const checkDigit = digits.pop()!;
  const sum = digits.reduce((acc, d, i) => acc + d * ((digits.length - i) % 2 === 0 ? 1 : 3), 0);
  const expectedCheck = (10 - (sum % 10)) % 10;
  if (checkDigit !== expectedCheck) {
    return { valid: false, type, error: `Invalid check digit: expected ${expectedCheck}, got ${checkDigit}` };
  }
  return { valid: true, type };
}
```

**Testing**:
- `Unit: Money.of("29.99", "USD").add(Money.of("5.01", "USD")).toFixed() === "35.00"`
- `Unit: Money.of("10.00", "USD").subtract(Money.of("10.00", "EUR")) throws "Currency mismatch"`
- `Unit: Money.of("100.00", "USD").multiply("0.15").toFixed() === "15.00" (no floating-point drift)`
- `Unit: validateGtin("012345678905") returns { valid: true, type: "UPC-A/GTIN-12" }`
- `Unit: validateGtin("0123456789012") returns valid EAN-13 with correct check digit`
- `Unit: validateGtin("0123456789099") returns { valid: false, error: "Invalid check digit" }`
- `Unit: validateGtin("abc") returns { valid: false, error: "GTIN must contain only digits" }`
- `Unit: validateGtin("12345") returns { valid: false, error: "Invalid GTIN length: 5" }`
- `Unit: OrderStatus type accepts "pending" | "confirmed" | ... | "refunded" but rejects arbitrary strings at compile time`

---

## Phase 2: Seller & Marketplace Management

### Purpose

Implement multi-tenant seller registration, authentication, and marketplace credential management. After this phase, sellers can create accounts, log in, connect their marketplace accounts via OAuth 2.0, and manage their marketplace credentials through the API. This is the prerequisite for all data-syncing phases.

### Tasks

#### 2.1 — Seller Registration & Authentication

**What**: Implement seller signup, login, session management, and RBAC using Lucia Auth.

**Design**:

```typescript
// packages/api/src/routes/sellers.ts
// POST /api/sellers — register a new seller account
interface CreateSellerRequest {
  companyName: string;
  contactEmail: string;
  contactPhone?: string;
  countryCode: string; // ISO 3166-1 alpha-2
  timezone?: string;
  user: {
    email: string;
    displayName: string;
    password: string;
  };
}

interface CreateSellerResponse {
  seller: {
    id: string;
    companyName: string;
    contactEmail: string;
    countryCode: string;
    timezone: string;
    subscriptionTier: "free";
    status: "active";
    createdAt: string;
  };
  user: {
    id: string;
    email: string;
    displayName: string;
    role: "owner";
  };
  sessionToken: string;
}

// POST /api/auth/login
interface LoginRequest {
  email: string;
  password: string;
}

interface LoginResponse {
  sessionToken: string;
  user: { id: string; email: string; displayName: string; role: string };
  seller: { id: string; companyName: string };
}

// GET /api/sellers/:sellerId — get seller details (requires auth)
// PUT /api/sellers/:sellerId — update seller profile (requires owner/admin)
// GET /api/sellers/:sellerId/users — list seller users (requires owner/admin)
// POST /api/sellers/:sellerId/users — invite user (requires owner/admin)
```

Role hierarchy: `owner > admin > member > viewer`. Authorization middleware:

```typescript
// packages/api/src/plugins/auth.ts
type SellerRole = "owner" | "admin" | "member" | "viewer";

const ROLE_HIERARCHY: Record<SellerRole, number> = {
  owner: 40,
  admin: 30,
  member: 20,
  viewer: 10,
};

function requireRole(minimumRole: SellerRole) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const session = await validateSession(request);
    if (!session) return reply.status(401).send({ error: "Unauthorized" });
    if (ROLE_HIERARCHY[session.role] < ROLE_HIERARCHY[minimumRole]) {
      return reply.status(403).send({ error: "Insufficient permissions" });
    }
    request.session = session;
  };
}
```

Password hashing uses Argon2id (OWASP recommendation):

```typescript
import { hash, verify } from "@node-rs/argon2";

const ARGON2_OPTIONS = {
  memoryCost: 19456, // 19 MiB
  timeCost: 2,
  outputLen: 32,
  parallelism: 1,
};
```

**Testing**:
- `Unit: password hashing with Argon2id produces different hashes for same input (salt)`
- `Unit: password verification succeeds for correct password`
- `Unit: password verification fails for incorrect password`
- `Integration: POST /api/sellers creates seller + owner user, returns 201 with sessionToken`
- `Integration: POST /api/sellers with duplicate email returns 409 Conflict`
- `Integration: POST /api/auth/login with valid credentials returns 200 with session`
- `Integration: POST /api/auth/login with invalid password returns 401`
- `Integration: GET /api/sellers/:id without auth returns 401`
- `Integration: GET /api/sellers/:id with valid session returns seller details`
- `Integration: viewer role cannot POST /api/sellers/:id/users (returns 403)`
- `Integration: owner role can POST /api/sellers/:id/users (returns 201)`

#### 2.2 — Marketplace Credential Management

**What**: Implement CRUD for marketplace credentials with encrypted storage, OAuth 2.0 token refresh, and connection status tracking.

**Design**:

```typescript
// POST /api/sellers/:sellerId/marketplace-credentials
interface ConnectMarketplaceRequest {
  marketplaceCode: MarketplaceCode;
  authType: AuthType;
  credentials: {
    // OAuth2 (Amazon, eBay, TikTok)
    clientId?: string;
    clientSecret?: string;
    refreshToken?: string;
    // API Key (Walmart)
    apiKey?: string;
    apiSecret?: string;
    // Additional marketplace-specific fields
    [key: string]: unknown;
  };
}

interface ConnectMarketplaceResponse {
  id: string;
  marketplaceCode: MarketplaceCode;
  authType: AuthType;
  status: "active" | "expired" | "revoked";
  lastSyncedAt: string | null;
  createdAt: string;
}

// GET /api/sellers/:sellerId/marketplace-credentials — list all connected marketplaces
// DELETE /api/sellers/:sellerId/marketplace-credentials/:id — disconnect marketplace
// POST /api/sellers/:sellerId/marketplace-credentials/:id/test — test connection
// POST /api/sellers/:sellerId/marketplace-credentials/:id/refresh — force token refresh
```

Credential encryption at rest using AES-256-GCM:

```typescript
// packages/api/src/services/credential-encryption.ts
import { createCipheriv, createDecipheriv, randomBytes } from "node:crypto";

const ALGORITHM = "aes-256-gcm";
const IV_LENGTH = 16;
const TAG_LENGTH = 16;

export function encrypt(plaintext: string, key: Buffer): string {
  const iv = randomBytes(IV_LENGTH);
  const cipher = createCipheriv(ALGORITHM, key, iv);
  const encrypted = Buffer.concat([cipher.update(plaintext, "utf8"), cipher.final()]);
  const tag = cipher.getAuthTag();
  return Buffer.concat([iv, tag, encrypted]).toString("base64");
}

export function decrypt(ciphertext: string, key: Buffer): string {
  const data = Buffer.from(ciphertext, "base64");
  const iv = data.subarray(0, IV_LENGTH);
  const tag = data.subarray(IV_LENGTH, IV_LENGTH + TAG_LENGTH);
  const encrypted = data.subarray(IV_LENGTH + TAG_LENGTH);
  const decipher = createDecipheriv(ALGORITHM, key, iv);
  decipher.setAuthTag(tag);
  return decipher.update(encrypted) + decipher.final("utf8");
}
```

**Testing**:
- `Unit: encrypt then decrypt returns original plaintext`
- `Unit: decrypt with wrong key throws error`
- `Unit: two encryptions of same plaintext produce different ciphertexts (random IV)`
- `Integration: POST /api/sellers/:id/marketplace-credentials with valid Amazon credentials returns 201`
- `Integration: credentials stored in database are encrypted (raw DB query shows base64, not plaintext)`
- `Integration: GET /api/sellers/:id/marketplace-credentials returns list without exposing secrets`
- `Integration: POST .../test calls marketplace health endpoint and returns { connected: true }`
- `Integration: DELETE .../marketplace-credentials/:id soft-deletes and returns 204`
- `Integration: duplicate (seller, marketplace) returns 409`

#### 2.3 — OAuth 2.0 Token Lifecycle Management

**What**: Implement automatic token refresh for OAuth 2.0 marketplaces (Amazon LWA, eBay, TikTok) with configurable refresh-before-expiry, retry with exponential backoff, and credential status tracking.

**Design**:

```typescript
// packages/marketplace-connectors/src/connector-interface.ts
export interface MarketplaceAuthProvider {
  /** Exchange authorization code for access/refresh tokens */
  exchangeAuthCode(code: string, redirectUri: string): Promise<OAuthTokenSet>;
  /** Refresh an expired access token */
  refreshAccessToken(refreshToken: string): Promise<OAuthTokenSet>;
  /** Validate that current credentials can make API calls */
  testConnection(credentials: Record<string, unknown>): Promise<ConnectionTestResult>;
}

export interface OAuthTokenSet {
  accessToken: string;
  refreshToken: string;
  expiresInSeconds: number;
  tokenType: string;
  scope?: string;
}

export interface ConnectionTestResult {
  connected: boolean;
  sellerId?: string;
  sellerName?: string;
  error?: string;
  marketplace: MarketplaceCode;
}
```

Token refresh BullMQ job (runs every 5 minutes, refreshes tokens expiring within 10 minutes):

```typescript
// packages/worker/src/processors/token-refresh.ts
export async function processTokenRefresh(job: Job) {
  const expiringCredentials = await db
    .select()
    .from(marketplaceCredential)
    .where(
      and(
        eq(marketplaceCredential.status, "active"),
        eq(marketplaceCredential.authType, "oauth2"),
        // Refresh tokens expiring within 10 minutes
        lt(
          sql`(credentials->>'token_expires_at')::timestamptz`,
          sql`now() + interval '10 minutes'`
        ),
      )
    );

  for (const cred of expiringCredentials) {
    try {
      const provider = getAuthProvider(cred.marketplaceCode);
      const newTokens = await provider.refreshAccessToken(
        decrypt(cred.credentials.refresh_token, encryptionKey)
      );
      await updateCredentialTokens(cred.id, newTokens);
    } catch (error) {
      await markCredentialExpired(cred.id, error.message);
    }
  }
}
```

**Testing**:
- `Unit: Amazon LWA refreshAccessToken sends correct POST to api.amazon.com/auth/o2/token`
- `Unit: eBay refreshAccessToken sends correct POST to api.ebay.com/identity/v1/oauth2/token`
- `Integration (mocked API): token refresh with valid refresh token updates access_token and token_expires_at`
- `Integration (mocked API): token refresh with revoked refresh token marks credential as "expired"`
- `Integration: token refresh job processes 3 expiring credentials, skips 2 non-expiring ones`
- `Unit: exponential backoff retries 3 times on 429 rate limit responses`
- `Unit: testConnection for Amazon calls GET /sellers/v1/marketplaceParticipations`

---

## Phase 3: Product Catalogue & Listing Management

### Purpose

Implement the product catalogue (seller's canonical product data) and the listing layer (marketplace-specific representations of those products). After this phase, sellers can create products, push them as listings to connected marketplaces, and sync listing status back from marketplaces. This is the foundation for inventory sync, order management, and repricing.

### Tasks

#### 3.1 — Product CRUD & Catalogue Management

**What**: Full CRUD for the seller's product catalogue with variant support, GTIN validation, image management, and search/filtering.

**Design**:

```typescript
// POST /api/sellers/:sellerId/products
interface CreateProductRequest {
  sku: string;
  title: string;
  description?: string;
  brand?: string;
  gtin?: string; // validated via validateGtin()
  mpn?: string;
  category?: string;
  weightKg?: number;
  dimensions?: { lengthCm: number; widthCm: number; heightCm: number };
  countryOfOrigin?: string;
  hsCode?: string;
  images?: Array<{ url: string; type: "main" | "variant" | "lifestyle" | "swatch"; altText?: string }>;
  variants?: Array<{
    sku: string;
    name: string;
    gtin?: string;
    attributes: Record<string, string>; // { color: "Red", size: "Large" }
    weightKg?: number;
  }>;
  customAttributes?: Record<string, unknown>;
}

// GET /api/sellers/:sellerId/products?page=1&limit=50&search=widget&brand=Acme&status=active
interface ProductListResponse {
  data: Product[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}

// GET /api/sellers/:sellerId/products/:productId
// PUT /api/sellers/:sellerId/products/:productId
// DELETE /api/sellers/:sellerId/products/:productId (soft delete)
// POST /api/sellers/:sellerId/products/import (bulk CSV/JSON import)
```

Product service with GTIN validation:

```typescript
// packages/api/src/services/product-service.ts
export class ProductService {
  async createProduct(sellerId: string, input: CreateProductRequest): Promise<Product> {
    if (input.gtin) {
      const gtinResult = validateGtin(input.gtin);
      if (!gtinResult.valid) {
        throw new ValidationError("gtin", gtinResult.error!);
      }
    }
    // Validate variant GTINs
    for (const variant of input.variants ?? []) {
      if (variant.gtin) {
        const result = validateGtin(variant.gtin);
        if (!result.valid) {
          throw new ValidationError(`variants[${variant.sku}].gtin`, result.error!);
        }
      }
    }
    // Validate unique variant SKUs
    const variantSkus = (input.variants ?? []).map(v => v.sku);
    if (new Set(variantSkus).size !== variantSkus.length) {
      throw new ValidationError("variants", "Duplicate variant SKUs");
    }

    return await this.db.insert(product).values({
      sellerId,
      ...input,
      images: input.images ?? [],
      variants: input.variants ?? [],
      customAttributes: input.customAttributes ?? {},
    }).returning();
  }
}
```

**Testing**:
- `Unit: createProduct with valid GTIN inserts product and returns it`
- `Unit: createProduct with invalid GTIN throws ValidationError with field name "gtin"`
- `Unit: createProduct with duplicate variant SKUs throws ValidationError`
- `Unit: createProduct without optional fields uses defaults (images=[], variants=[])`
- `Integration: POST /api/sellers/:id/products returns 201 with created product`
- `Integration: GET /api/sellers/:id/products?search=widget returns matching products`
- `Integration: GET /api/sellers/:id/products?brand=Acme filters by brand`
- `Integration: PUT /api/sellers/:id/products/:id updates only provided fields`
- `Integration: DELETE /api/sellers/:id/products/:id sets status="discontinued", returns 204`
- `Integration: POST /api/sellers/:id/products with duplicate SKU returns 409`
- `Fixture: import 100 products from tests/fixtures/product-import.csv; verify all 100 inserted`

#### 3.2 — Listing Creation & Marketplace Sync

**What**: Create listings from products for specific marketplaces, push listing data to marketplace APIs, and sync listing status (active, suppressed, error) back to the platform.

**Design**:

```typescript
// packages/marketplace-connectors/src/connector-interface.ts
export interface MarketplaceListingConnector {
  /** Push a listing to the marketplace (create or update) */
  pushListing(listing: ListingPushPayload): Promise<ListingPushResult>;
  /** Fetch current listing status from marketplace */
  getListingStatus(marketplaceListingId: string): Promise<MarketplaceListingStatus>;
  /** Bulk fetch listing statuses */
  getListingStatuses(ids: string[]): Promise<MarketplaceListingStatus[]>;
  /** Delete/deactivate a listing on the marketplace */
  deactivateListing(marketplaceListingId: string): Promise<void>;
}

export interface ListingPushPayload {
  sku: string;
  title: string;
  description: string;
  price: string;
  currencyCode: string;
  quantity: number;
  condition: ItemCondition;
  gtin?: string;
  images: Array<{ url: string; type: string }>;
  marketplaceAttributes: Record<string, unknown>;
}

export interface ListingPushResult {
  success: boolean;
  marketplaceListingId?: string;
  errors?: Array<{ code: string; message: string; field?: string }>;
  warnings?: Array<{ code: string; message: string }>;
}

// POST /api/sellers/:sellerId/listings
interface CreateListingRequest {
  productId: string;
  marketplaceCode: MarketplaceCode;
  marketplaceSku?: string; // defaults to product SKU
  title?: string; // defaults to product title
  price: string;
  condition?: ItemCondition;
  fulfillmentType?: FulfillmentType;
  marketplaceAttributes?: Record<string, unknown>;
}

// POST /api/sellers/:sellerId/listings/:listingId/sync — push to marketplace
// POST /api/sellers/:sellerId/listings/bulk-sync — push multiple listings
// GET /api/sellers/:sellerId/listings?marketplace=amazon_us&status=active
```

Listing sync BullMQ processor:

```typescript
// packages/worker/src/processors/listing-sync.ts
export async function processListingSync(job: Job<{ listingId: string }>) {
  const listingRow = await getListingWithProduct(job.data.listingId);
  const connector = getListingConnector(listingRow.marketplace.code);

  const payload: ListingPushPayload = {
    sku: listingRow.marketplaceSku ?? listingRow.product.sku,
    title: listingRow.title,
    description: listingRow.product.description ?? "",
    price: listingRow.price,
    currencyCode: listingRow.currencyCode,
    quantity: listingRow.quantityAvailable,
    condition: listingRow.condition,
    gtin: listingRow.product.gtin ?? undefined,
    images: listingRow.product.images as ProductImage[],
    marketplaceAttributes: listingRow.marketplaceAttributes as Record<string, unknown>,
  };

  const result = await connector.pushListing(payload);

  if (result.success) {
    await updateListing(listingRow.id, {
      marketplaceListingId: result.marketplaceListingId,
      syncStatus: "synced",
      status: "active",
      syncError: null,
      lastSyncedAt: new Date(),
    });
  } else {
    await updateListing(listingRow.id, {
      syncStatus: "error",
      status: "error",
      syncError: { errors: result.errors, timestamp: new Date().toISOString() },
    });
  }
}
```

**Testing**:
- `Integration: POST /api/sellers/:id/listings creates listing in "draft" status`
- `Integration: POST /api/sellers/:id/listings for unconnected marketplace returns 400`
- `Integration: POST .../listings/:id/sync enqueues listing-sync job`
- `Integration (mocked connector): listing sync with valid data sets syncStatus="synced", status="active"`
- `Integration (mocked connector): listing sync with marketplace validation error sets syncStatus="error" with error details`
- `Integration: GET /api/sellers/:id/listings?marketplace=amazon_us returns only Amazon listings`
- `Integration: GET /api/sellers/:id/listings?status=error returns listings with sync errors`
- `Unit: ListingPushPayload includes product GTIN when available, omits when null`
- `Unit: default marketplaceSku falls back to product SKU when not explicitly set`

#### 3.3 — Marketplace-Specific Attribute Validation

**What**: Validate listing `marketplace_attributes` JSONB against marketplace product type JSON Schema definitions before sync. Use the `marketplace_product_type` table to store schemas per marketplace and category.

**Design**:

```typescript
// packages/shared/src/schemas/attribute-validator.ts
import Ajv from "ajv/dist/2020";

const ajv = new Ajv({ allErrors: true, strict: false });

export interface ValidationResult {
  valid: boolean;
  errors: Array<{
    field: string;
    message: string;
    keyword: string;
  }>;
}

export function validateMarketplaceAttributes(
  attributes: Record<string, unknown>,
  schema: Record<string, unknown>,
): ValidationResult {
  const validate = ajv.compile(schema);
  const valid = validate(attributes);
  if (valid) return { valid: true, errors: [] };

  return {
    valid: false,
    errors: (validate.errors ?? []).map(err => ({
      field: err.instancePath.replace(/^\//, "").replace(/\//g, ".") || err.params?.missingProperty || "unknown",
      message: err.message ?? "Validation failed",
      keyword: err.keyword,
    })),
  };
}
```

Amazon product type schema example stored in `marketplace_product_type`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["item_name", "brand", "bullet_point"],
  "properties": {
    "item_name": { "type": "string", "maxLength": 200 },
    "brand": { "type": "string", "maxLength": 50 },
    "bullet_point": {
      "type": "array",
      "items": { "type": "string", "maxLength": 500 },
      "minItems": 1,
      "maxItems": 5
    },
    "product_description": { "type": "string", "maxLength": 2000 },
    "search_terms": { "type": "string", "maxLength": 250 },
    "product_tax_code": { "type": "string" }
  }
}
```

**Testing**:
- `Unit: valid Amazon attributes pass validation`
- `Unit: missing required "brand" field returns error with field="brand", keyword="required"`
- `Unit: bullet_point with 6 items returns error with keyword="maxItems"`
- `Unit: item_name exceeding 200 chars returns error with keyword="maxLength"`
- `Integration: POST .../listings/:id/sync validates attributes before pushing; returns 400 with errors if invalid`
- `Integration: listing without product_type_id skips attribute validation (graceful degradation)`
- `Fixture: tests/fixtures/amazon-apparel-attributes.json passes Amazon apparel schema`
- `Fixture: tests/fixtures/ebay-item-specifics.json passes eBay consumer electronics schema`

---

## Phase 4: Inventory Management

### Purpose

Implement real-time inventory tracking across warehouses and marketplace channel allocations, with optimistic locking to prevent overselling. After this phase, sellers can manage stock levels, allocate inventory per marketplace, and have inventory automatically decremented on order receipt and updated across all connected marketplaces.

### Tasks

#### 4.1 — Warehouse & Inventory CRUD

**What**: CRUD for warehouses and inventory records, with three-tier quantity tracking (available, reserved, on_order) and optimistic locking via version column.

**Design**:

```typescript
// POST /api/sellers/:sellerId/warehouses
interface CreateWarehouseRequest {
  name: string;
  warehouseType: "own" | "3pl" | "fba" | "wfs";
  address: {
    line1: string;
    line2?: string;
    city: string;
    state: string;
    postalCode: string;
    countryCode: string;
  };
  isDefault?: boolean;
}

// GET /api/sellers/:sellerId/inventory?warehouse=:warehouseId&lowStock=true&product=:productId
// POST /api/sellers/:sellerId/inventory
interface CreateInventoryRequest {
  productId: string;
  variantSku?: string;
  warehouseId: string;
  quantityAvailable: number;
  costPerUnit?: string;
  currencyCode?: string;
  reorderPoint?: number;
  reorderQuantity?: number;
  channelAllocations?: Record<MarketplaceCode, { allocatedQty: number }>;
}

// PUT /api/sellers/:sellerId/inventory/:inventoryId
interface UpdateInventoryRequest {
  quantityAvailable?: number;
  costPerUnit?: string;
  reorderPoint?: number;
  reorderQuantity?: number;
  channelAllocations?: Record<MarketplaceCode, { allocatedQty: number }>;
  version: number; // optimistic locking — must match current version
}

// POST /api/sellers/:sellerId/inventory/:inventoryId/adjust
interface InventoryAdjustmentRequest {
  adjustmentType: "restock" | "sale" | "return" | "transfer_in" | "transfer_out" | "adjustment" | "reservation";
  quantityChange: number; // positive for additions, negative for reductions
  reason?: string;
  referenceType?: "order" | "purchase_order" | "transfer";
  referenceId?: string;
}
```

Optimistic locking implementation:

```typescript
// packages/api/src/services/inventory-service.ts
export class InventoryService {
  async adjustInventory(inventoryId: string, adjustment: InventoryAdjustmentRequest): Promise<Inventory> {
    return await this.db.transaction(async (tx) => {
      const current = await tx.select().from(inventory).where(eq(inventory.id, inventoryId)).for("update");
      if (!current.length) throw new NotFoundError("Inventory record not found");

      const record = current[0];
      const newQuantity = record.quantityAvailable + adjustment.quantityChange;
      if (newQuantity < 0) throw new ValidationError("quantityChange", "Insufficient stock");

      const updated = await tx.update(inventory)
        .set({
          quantityAvailable: newQuantity,
          version: record.version + 1,
          updatedAt: new Date(),
        })
        .where(and(eq(inventory.id, inventoryId), eq(inventory.version, record.version)))
        .returning();

      if (!updated.length) throw new ConflictError("Inventory was modified concurrently. Retry.");

      await tx.insert(inventoryTransaction).values({
        inventoryId,
        transactionType: adjustment.adjustmentType,
        quantityChange: adjustment.quantityChange,
        quantityBefore: record.quantityAvailable,
        quantityAfter: newQuantity,
        referenceType: adjustment.referenceType,
        referenceId: adjustment.referenceId,
        metadata: { reason: adjustment.reason },
      });

      return updated[0];
    });
  }
}
```

**Testing**:
- `Integration: POST /api/sellers/:id/warehouses creates warehouse, returns 201`
- `Integration: POST /api/sellers/:id/inventory creates inventory record with version=1`
- `Integration: POST .../inventory/:id/adjust with quantityChange=-5 decrements available from 100 to 95`
- `Integration: POST .../inventory/:id/adjust with quantityChange=-200 on stock of 100 returns 400 "Insufficient stock"`
- `Integration: concurrent adjustments — two simultaneous -50 on stock of 80; one succeeds, one gets 409 "modified concurrently"`
- `Integration: adjustment creates inventory_transaction record with before/after quantities`
- `Integration: GET .../inventory?lowStock=true returns only records where quantityAvailable <= reorderPoint`
- `Unit: optimistic lock version increments on every update`
- `Integration: PUT .../inventory/:id with stale version returns 409`

#### 4.2 — Channel Allocation & Cross-Marketplace Sync

**What**: Manage per-marketplace inventory allocation from warehouse stock, and push inventory updates to connected marketplaces when allocations or stock levels change.

**Design**:

```typescript
// PUT /api/sellers/:sellerId/inventory/:inventoryId/allocations
interface UpdateAllocationsRequest {
  allocations: Record<MarketplaceCode, { allocatedQty: number }>;
  version: number;
}
// Validation: sum of allocatedQty across all channels must not exceed quantityAvailable

// Worker: inventory-sync processor
// Triggered when: inventory adjustment occurs, channel allocation changes
// For each marketplace allocation that changed:
//   1. Look up active listings for this product on this marketplace
//   2. Call connector.updateInventory(listingId, newQuantity)
//   3. Update allocation syncStatus and lastSyncedAt
```

```typescript
// packages/marketplace-connectors/src/connector-interface.ts
export interface MarketplaceInventoryConnector {
  /** Update inventory quantity for a listing */
  updateInventory(marketplaceListingId: string, quantity: number): Promise<InventorySyncResult>;
  /** Bulk update inventory for multiple listings */
  bulkUpdateInventory(updates: Array<{ marketplaceListingId: string; quantity: number }>): Promise<InventorySyncResult[]>;
}

export interface InventorySyncResult {
  success: boolean;
  marketplaceListingId: string;
  error?: string;
}
```

**Testing**:
- `Integration: PUT .../inventory/:id/allocations updates channel_allocations JSONB`
- `Integration: allocations summing to more than quantityAvailable returns 400`
- `Integration: changing Amazon allocation from 60 to 40 enqueues inventory-sync job for Amazon`
- `Integration (mocked connector): inventory sync calls connector.updateInventory with correct quantity`
- `Integration (mocked connector): failed inventory sync sets allocation syncStatus="error"`
- `Unit: inventory adjustment triggers allocation recalculation when total allocations exceed new available quantity`

---

## Phase 5: Order Management & Fulfilment

### Purpose

Implement order ingestion from marketplace APIs, unified order dashboard, order lifecycle management, and shipment tracking. After this phase, sellers can view all orders from all marketplaces in a single interface, process orders, create shipments with tracking numbers, and have inventory automatically adjusted when orders are placed.

### Tasks

#### 5.1 — Order Ingestion from Marketplaces

**What**: Periodic and webhook-driven order sync from connected marketplaces. Orders are normalised into the unified `marketplace_order` table with marketplace-specific data preserved in JSONB.

**Design**:

```typescript
// packages/marketplace-connectors/src/connector-interface.ts
export interface MarketplaceOrderConnector {
  /** Fetch new/updated orders since lastSyncedAt */
  fetchOrders(since: Date, options?: { statuses?: string[] }): Promise<RawMarketplaceOrder[]>;
  /** Acknowledge/confirm an order on the marketplace */
  acknowledgeOrder(marketplaceOrderId: string): Promise<void>;
  /** Submit shipment tracking to marketplace */
  submitShipment(marketplaceOrderId: string, shipment: ShipmentSubmission): Promise<void>;
}

export interface RawMarketplaceOrder {
  marketplaceOrderId: string;
  orderDate: string;
  status: string;
  items: Array<{
    marketplaceItemId: string;
    sku: string;
    title: string;
    quantity: number;
    unitPrice: string;
    tax: string;
  }>;
  subtotal: string;
  shippingCost: string;
  taxAmount: string;
  totalAmount: string;
  shippingAddress: Record<string, unknown>;
  marketplaceSpecificData: Record<string, unknown>;
  rawResponse: Record<string, unknown>;
}
```

Order sync worker:

```typescript
// packages/worker/src/processors/order-sync.ts
export async function processOrderSync(job: Job<{ sellerId: string; marketplaceCode: MarketplaceCode }>) {
  const { sellerId, marketplaceCode } = job.data;
  const credential = await getActiveCredential(sellerId, marketplaceCode);
  const connector = getOrderConnector(marketplaceCode, credential);
  const lastSync = credential.lastSyncedAt ?? new Date(Date.now() - 24 * 60 * 60 * 1000);

  const rawOrders = await connector.fetchOrders(lastSync);

  for (const raw of rawOrders) {
    const existing = await findOrderByMarketplaceId(marketplaceCode, raw.marketplaceOrderId);
    if (existing) {
      await updateOrderStatus(existing.id, mapOrderStatus(raw.status, marketplaceCode));
    } else {
      const order = await createOrder(sellerId, marketplaceCode, raw);
      // Decrement inventory for new orders
      for (const item of raw.items) {
        await inventoryService.adjustInventory(
          await findInventoryBySku(sellerId, item.sku),
          { adjustmentType: "sale", quantityChange: -item.quantity, referenceType: "order", referenceId: order.id }
        );
      }
    }
  }

  await updateCredentialLastSync(credential.id, new Date());
}
```

Marketplace-specific status mapping:

```typescript
// packages/shared/src/constants/order-status-maps.ts
export const AMAZON_STATUS_MAP: Record<string, OrderStatus> = {
  Pending: "pending",
  Unshipped: "confirmed",
  PartiallyShipped: "processing",
  Shipped: "shipped",
  Canceled: "cancelled",
  Unfulfillable: "cancelled",
};

export const EBAY_STATUS_MAP: Record<string, OrderStatus> = {
  ACTIVE: "confirmed",
  COMPLETED: "shipped",
  CANCELLED: "cancelled",
  INACTIVE: "cancelled",
};
```

**Testing**:
- `Integration (mocked connector): order sync fetches 5 new Amazon orders; all 5 inserted into marketplace_order`
- `Integration (mocked connector): order sync with existing order updates status only, does not duplicate`
- `Integration: new order decrements inventory by item quantities`
- `Integration: order with unknown SKU logs warning but does not fail sync`
- `Unit: Amazon status "Unshipped" maps to "confirmed"`
- `Unit: eBay status "COMPLETED" maps to "shipped"`
- `Fixture: tests/fixtures/amazon-order.json parses into valid MarketplaceOrder`
- `Integration: order sync updates credential.lastSyncedAt after completion`

#### 5.2 — Unified Order Dashboard API

**What**: API endpoints for querying, filtering, and managing orders across all marketplaces from a single interface.

**Design**:

```typescript
// GET /api/sellers/:sellerId/orders?marketplace=amazon_us&status=confirmed&dateFrom=2026-05-01&dateTo=2026-05-25&page=1&limit=50&sort=orderDate:desc
interface OrderListResponse {
  data: MarketplaceOrder[];
  pagination: { page: number; limit: number; total: number; totalPages: number };
  summary: {
    totalOrders: number;
    totalRevenue: string;
    byMarketplace: Record<MarketplaceCode, { count: number; revenue: string }>;
    byStatus: Record<OrderStatus, number>;
  };
}

// GET /api/sellers/:sellerId/orders/:orderId — full order details with items and shipments
// PUT /api/sellers/:sellerId/orders/:orderId/status — manually update order status
// POST /api/sellers/:sellerId/orders/:orderId/acknowledge — acknowledge order on marketplace
// GET /api/sellers/:sellerId/orders/export?format=csv&dateFrom=&dateTo= — export orders
```

**Testing**:
- `Integration: GET .../orders returns orders from all marketplaces sorted by date desc`
- `Integration: GET .../orders?marketplace=amazon_us returns only Amazon orders`
- `Integration: GET .../orders?status=confirmed&status=processing returns orders in either status`
- `Integration: GET .../orders?dateFrom=2026-05-01&dateTo=2026-05-15 returns orders in date range`
- `Integration: response includes summary.byMarketplace with correct counts and revenue`
- `Integration: GET .../orders/:id returns full order with items array and shipment data`
- `Integration: GET .../orders/export?format=csv returns valid CSV with order rows`

#### 5.3 — Shipment Creation & Tracking

**What**: Create shipments for orders, generate or upload tracking numbers, and push shipment data back to marketplaces.

**Design**:

```typescript
// POST /api/sellers/:sellerId/orders/:orderId/shipments
interface CreateShipmentRequest {
  carrierCode: "ups" | "fedex" | "dhl" | "usps" | string;
  trackingNumber: string;
  shippingMethod?: string;
  warehouseId?: string;
  items: Array<{ sku: string; quantity: number }>;
}

interface CreateShipmentResponse {
  id: string;
  orderId: string;
  carrierCode: string;
  trackingNumber: string;
  status: "pending";
  items: Array<{ sku: string; quantity: number }>;
  createdAt: string;
}

// Workflow:
// 1. Create shipment record
// 2. Enqueue marketplace-shipment-submit job
// 3. Worker calls connector.submitShipment() to push tracking to marketplace
// 4. Update order status to "shipped" if all items shipped
```

**Testing**:
- `Integration: POST .../orders/:id/shipments creates shipment and returns 201`
- `Integration: creating shipment enqueues marketplace-shipment-submit job`
- `Integration (mocked connector): shipment submit calls Amazon Orders API with tracking number`
- `Integration: shipment covering all order items updates order status to "shipped"`
- `Integration: partial shipment (not all items) keeps order status as "processing"`
- `Unit: shipment with quantity exceeding order item quantity returns 400`

---

## Phase 6: Repricing Engine

### Purpose

Implement the rule-based repricing engine with competitor price monitoring, Buy Box tracking, and automated price adjustments. After this phase, sellers can define repricing strategies (floor/ceiling, beat-by-percent, target-margin), monitor competitor prices, and have the engine automatically adjust listing prices within defined boundaries.

### Tasks

#### 6.1 — Repricing Rule CRUD

**What**: CRUD for repricing rules with JSONB-based strategy configuration, listing assignment, and scheduling.

**Design**:

```typescript
// POST /api/sellers/:sellerId/repricing-rules
interface CreateRepricingRuleRequest {
  name: string;
  marketplaceCode?: MarketplaceCode; // null = all marketplaces
  listingIds: string[];
  config: RepricingConfig;
  priority?: number;
}

type RepricingConfig =
  | FloorCeilingConfig
  | BeatLowestConfig
  | TargetMarginConfig
  | MatchBuyBoxConfig;

interface FloorCeilingConfig {
  strategy: "floor_ceiling";
  minPrice: string;
  maxPrice: string;
}

interface BeatLowestConfig {
  strategy: "beat_lowest_by_percent" | "beat_lowest_by_amount";
  beatByPct?: number;
  beatByAmount?: string;
  minPrice: string;
  maxPrice: string;
  minMarginPct?: number;
  onlyMatchFba?: boolean;
  excludeSellers?: string[];
  cooldownMinutes?: number;
  maxChangesPerDay?: number;
}

interface TargetMarginConfig {
  strategy: "target_margin";
  targetMarginPct: number;
  minPrice: string;
  maxPrice: string;
}

interface MatchBuyBoxConfig {
  strategy: "match_buy_box";
  beatByAmount?: string;
  minPrice: string;
  maxPrice: string;
  onlyWhenLosing?: boolean;
}

// GET /api/sellers/:sellerId/repricing-rules
// PUT /api/sellers/:sellerId/repricing-rules/:ruleId
// DELETE /api/sellers/:sellerId/repricing-rules/:ruleId
// POST /api/sellers/:sellerId/repricing-rules/:ruleId/activate
// POST /api/sellers/:sellerId/repricing-rules/:ruleId/deactivate
// GET /api/sellers/:sellerId/repricing-rules/:ruleId/history — price changes caused by this rule
```

**Testing**:
- `Integration: POST .../repricing-rules creates rule with config JSONB, returns 201`
- `Integration: rule with strategy "beat_lowest_by_percent" stores beatByPct in config`
- `Integration: rule with minPrice > maxPrice returns 400 validation error`
- `Integration: deactivated rule has isActive=false and is excluded from repricing engine`
- `Unit: RepricingConfig union type correctly discriminates on "strategy" field`
- `Integration: GET .../repricing-rules/:id/history returns price_change_log entries for this rule`

#### 6.2 — Competitor Price Monitoring

**What**: Periodic competitor price observation via marketplace APIs (Amazon SP-API Product Pricing API, eBay Finding API), stored in the partitioned `competitor_observation` table.

**Design**:

```typescript
// packages/worker/src/processors/competitor-monitor.ts
export async function processCompetitorMonitor(job: Job<{ sellerId: string; marketplaceCode: MarketplaceCode }>) {
  const listings = await getActiveListingsForMarketplace(job.data.sellerId, job.data.marketplaceCode);
  const connector = getPricingConnector(job.data.marketplaceCode);

  const batchSize = 20; // SP-API getCompetitivePricing supports up to 20 ASINs per call
  for (let i = 0; i < listings.length; i += batchSize) {
    const batch = listings.slice(i, i + batchSize);
    const observations = await connector.getCompetitivePricing(
      batch.map(l => l.marketplaceListingId!)
    );

    for (const obs of observations) {
      await insertCompetitorObservation({
        listingId: obs.listingId,
        observedAt: new Date(),
        observations: obs.competitors,
        buyBoxPrice: obs.buyBoxPrice,
        ourPrice: obs.ourPrice,
        weOwnBuybox: obs.weOwnBuybox,
      });

      // Update listing.buyBoxOwned
      await updateListing(obs.listingId, { buyBoxOwned: obs.weOwnBuybox });
    }
  }
}
```

```typescript
// packages/marketplace-connectors/src/connector-interface.ts
export interface MarketplacePricingConnector {
  /** Get competitive pricing for listings */
  getCompetitivePricing(marketplaceListingIds: string[]): Promise<CompetitivePricingResult[]>;
}

export interface CompetitivePricingResult {
  listingId: string;
  marketplaceListingId: string;
  ourPrice: string;
  buyBoxPrice: string | null;
  weOwnBuybox: boolean;
  competitors: {
    buyBoxWinner: {
      sellerName: string;
      price: string;
      shipping: string;
      fulfillmentType: string;
      isAmazon: boolean;
    } | null;
    otherSellers: Array<{
      sellerName: string;
      price: string;
      shipping: string;
      fulfillmentType: string;
    }>;
    totalOffers: number;
    source: string;
  };
}
```

**Testing**:
- `Integration (mocked connector): competitor monitor fetches pricing for 50 listings in 3 batches of 20/20/10`
- `Integration: competitor observations inserted into correct monthly partition`
- `Integration: listing.buyBoxOwned updated to true when we own Buy Box`
- `Integration: listing.buyBoxOwned updated to false when competitor wins Buy Box`
- `Unit: batch size respects SP-API limit of 20 ASINs per call`
- `Fixture: tests/fixtures/amazon-competitive-pricing.json parses into CompetitivePricingResult`

#### 6.3 — Repricing Engine Execution

**What**: BullMQ repeatable job that evaluates active repricing rules against current competitor prices and adjusts listing prices, respecting floor/ceiling limits, cooldown periods, and daily change caps.

**Design**:

```typescript
// packages/worker/src/processors/repricing-engine.ts
export async function processRepricingCycle(job: Job<{ sellerId: string }>) {
  const rules = await getActiveRepricingRules(job.data.sellerId);

  for (const rule of rules) {
    const config = rule.config as RepricingConfig;
    const listings = await getListingsForRule(rule.id);

    for (const listing of listings) {
      const latestObs = await getLatestCompetitorObservation(listing.id);
      if (!latestObs) continue;

      const currentPrice = Money.of(listing.price, listing.currencyCode);
      const newPrice = calculateNewPrice(config, currentPrice, latestObs, listing);

      if (newPrice && !newPrice.amount.equals(currentPrice.amount)) {
        // Enforce cooldown
        if (config.cooldownMinutes) {
          const lastChange = await getLastPriceChange(listing.id);
          if (lastChange && (Date.now() - lastChange.createdAt.getTime()) < config.cooldownMinutes * 60_000) {
            continue;
          }
        }
        // Enforce daily change cap
        if (config.maxChangesPerDay) {
          const changesToday = await countPriceChangesToday(listing.id);
          if (changesToday >= config.maxChangesPerDay) continue;
        }

        await updateListingPrice(listing.id, newPrice.toFixed());
        await insertPriceChangeLog({
          listingId: listing.id,
          oldPrice: currentPrice.toFixed(),
          newPrice: newPrice.toFixed(),
          currencyCode: listing.currencyCode,
          changeReason: "repricing_rule",
          ruleId: rule.id,
          context: {
            competitorPrice: latestObs.buyBoxPrice,
            strategy: config.strategy,
          },
        });
        // Enqueue price sync to marketplace
        await enqueuePriceSync(listing.id, newPrice.toFixed());
      }
    }
  }
}

function calculateNewPrice(
  config: RepricingConfig,
  currentPrice: Money,
  observation: CompetitorObservation,
  listing: Listing,
): Money | null {
  const minPrice = Money.of(config.minPrice, listing.currencyCode);
  const maxPrice = Money.of(config.maxPrice, listing.currencyCode);

  let targetPrice: Money;

  switch (config.strategy) {
    case "beat_lowest_by_percent": {
      const lowestCompetitor = Money.of(observation.buyBoxPrice ?? currentPrice.toFixed(), listing.currencyCode);
      targetPrice = lowestCompetitor.multiply(1 - (config.beatByPct! / 100));
      break;
    }
    case "beat_lowest_by_amount": {
      const lowestCompetitor = Money.of(observation.buyBoxPrice ?? currentPrice.toFixed(), listing.currencyCode);
      targetPrice = lowestCompetitor.subtract(Money.of(config.beatByAmount!, listing.currencyCode));
      break;
    }
    case "match_buy_box": {
      if (!observation.buyBoxPrice) return null;
      if (config.onlyWhenLosing && observation.weOwnBuybox) return null;
      targetPrice = Money.of(observation.buyBoxPrice, listing.currencyCode);
      if (config.beatByAmount) {
        targetPrice = targetPrice.subtract(Money.of(config.beatByAmount, listing.currencyCode));
      }
      break;
    }
    case "target_margin": {
      // Requires cost_per_unit from inventory
      return null; // implemented after inventory cost data is available
    }
    case "floor_ceiling": {
      return null; // floor/ceiling only constrains, doesn't calculate
    }
    default:
      return null;
  }

  // Clamp to floor/ceiling
  if (targetPrice.isGreaterThan(maxPrice)) targetPrice = maxPrice;
  if (minPrice.isGreaterThan(targetPrice)) targetPrice = minPrice;

  return targetPrice;
}
```

**Testing**:
- `Unit: beat_lowest_by_percent(2%) on competitor price $28.00 calculates $27.44`
- `Unit: beat_lowest_by_amount($0.50) on competitor price $28.00 calculates $27.50`
- `Unit: calculated price below minPrice gets clamped to minPrice`
- `Unit: calculated price above maxPrice gets clamped to maxPrice`
- `Unit: match_buy_box with onlyWhenLosing=true and weOwnBuybox=true returns null (no change)`
- `Unit: match_buy_box with onlyWhenLosing=true and weOwnBuybox=false returns Buy Box price`
- `Integration: repricing cycle with 3 active rules processes all associated listings`
- `Integration: price change creates price_change_log entry with rule reference`
- `Integration: price change enqueues price-sync job to push to marketplace`
- `Integration: cooldown of 15 minutes prevents reprice within 15 minutes of last change`
- `Integration: maxChangesPerDay=24 prevents 25th price change in a day`

---

## Phase 7: Amazon SP-API Real-Time Integration

### Purpose

Implement deep Amazon SP-API integration including the real-time Notifications API for instant competitor price changes and order updates, the Listings Items API for listing management, and the Feeds API for bulk operations. This phase differentiates the platform from competitors by enabling sub-second repricing responses to Buy Box changes.

### Tasks

#### 7.1 — SP-API Client & Authentication

**What**: Full Amazon Selling Partner API client with Login with Amazon (LWA) OAuth 2.0, AWS SigV4 signing (for restricted operations), and automatic rate limit management.

**Design**:

```typescript
// packages/marketplace-connectors/src/amazon/client.ts
export class AmazonSpApiClient {
  private rateLimiter: PerEndpointRateLimiter;

  constructor(
    private credentials: AmazonCredentials,
    private marketplace: AmazonMarketplace,
  ) {
    this.rateLimiter = new PerEndpointRateLimiter({
      "listingsItems": { requestsPerSecond: 5, burst: 10 },
      "orders": { requestsPerSecond: 0.0167, burst: 20 },
      "productPricing": { requestsPerSecond: 10, burst: 20 },
      "feeds": { requestsPerSecond: 0.0222, burst: 10 },
    });
  }

  async request<T>(endpoint: string, options: RequestOptions): Promise<T> {
    await this.rateLimiter.acquire(endpoint);
    const accessToken = await this.ensureValidToken();

    const response = await fetch(this.marketplace.baseUrl + options.path, {
      method: options.method,
      headers: {
        "x-amz-access-token": accessToken,
        "Content-Type": "application/json",
        ...options.headers,
      },
      body: options.body ? JSON.stringify(options.body) : undefined,
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("x-amzn-RateLimit-Limit")) || 1;
      await sleep(retryAfter * 1000);
      return this.request(endpoint, options); // retry once
    }

    if (!response.ok) {
      const error = await response.json();
      throw new AmazonApiError(response.status, error);
    }

    return response.json() as T;
  }
}

interface AmazonCredentials {
  clientId: string;
  clientSecret: string;
  refreshToken: string;
  accessToken?: string;
  tokenExpiresAt?: Date;
  sellingPartnerId: string;
  marketplaceIds: string[];
}
```

```typescript
// packages/shared/src/utils/rate-limiter.ts
export class PerEndpointRateLimiter {
  private buckets: Map<string, TokenBucket> = new Map();

  constructor(limits: Record<string, { requestsPerSecond: number; burst: number }>) {
    for (const [endpoint, config] of Object.entries(limits)) {
      this.buckets.set(endpoint, new TokenBucket(config.requestsPerSecond, config.burst));
    }
  }

  async acquire(endpoint: string): Promise<void> {
    const bucket = this.buckets.get(endpoint);
    if (!bucket) return;
    await bucket.acquire();
  }
}
```

**Testing**:
- `Unit: rate limiter allows burst requests up to burst limit`
- `Unit: rate limiter blocks requests beyond burst and releases at requestsPerSecond rate`
- `Unit: 429 response triggers retry with backoff`
- `Integration (mocked): LWA token refresh sends correct POST to api.amazon.com/auth/o2/token`
- `Unit: expired access token triggers automatic refresh before request`

#### 7.2 — SP-API Notifications (Real-Time Events)

**What**: Subscribe to and process Amazon SP-API Notifications (ANY_OFFER_CHANGED, ORDER_STATUS_CHANGE, PRICING_HEALTH) via SQS or webhook destination for real-time Buy Box and order events.

**Design**:

```typescript
// packages/marketplace-connectors/src/amazon/notifications.ts
export interface AnyOfferChangedNotification {
  NotificationType: "ANY_OFFER_CHANGED";
  Payload: {
    AnyOfferChangedNotification: {
      OfferChangeTrigger: {
        MarketplaceId: string;
        ASIN: string;
        ItemCondition: string;
        TimeOfOfferChange: string;
      };
      Summary: {
        NumberOfOffers: Array<{ Condition: string; FulfillmentChannel: string; OfferCount: number }>;
        BuyBoxPrices: Array<{
          Condition: string;
          LandedPrice: { CurrencyCode: string; Amount: string };
          ListingPrice: { CurrencyCode: string; Amount: string };
          Shipping: { CurrencyCode: string; Amount: string };
        }>;
        BuyBoxEligibleOffers: Array<{ Condition: string; FulfillmentChannel: string; OfferCount: number }>;
      };
      Offers: Array<{
        SellerId: string;
        SubCondition: string;
        ListingPrice: { CurrencyCode: string; Amount: string };
        Shipping: { CurrencyCode: string; Amount: string };
        IsBuyBoxWinner: boolean;
        IsFulfilledByAmazon: boolean;
      }>;
    };
  };
}

// POST /api/webhooks/amazon/notifications — receives SP-API notification events
export async function handleAmazonNotification(request: FastifyRequest) {
  const notification = request.body as SpApiNotification;

  switch (notification.NotificationType) {
    case "ANY_OFFER_CHANGED":
      await processOfferChange(notification.Payload.AnyOfferChangedNotification);
      break;
    case "ORDER_STATUS_CHANGE":
      await processOrderStatusChange(notification.Payload);
      break;
    case "PRICING_HEALTH":
      await processPricingHealth(notification.Payload);
      break;
  }
}

async function processOfferChange(data: AnyOfferChangedNotification["Payload"]["AnyOfferChangedNotification"]) {
  const asin = data.OfferChangeTrigger.ASIN;
  const listing = await findListingByMarketplaceId("amazon_us", asin);
  if (!listing) return;

  // Record competitor observation
  await insertCompetitorObservation({
    listingId: listing.id,
    observedAt: new Date(data.OfferChangeTrigger.TimeOfOfferChange),
    observations: {
      buyBoxWinner: data.Offers.find(o => o.IsBuyBoxWinner) ?? null,
      otherSellers: data.Offers.filter(o => !o.IsBuyBoxWinner),
      totalOffers: data.Offers.length,
      source: "sp_api_notification",
    },
    buyBoxPrice: data.Summary.BuyBoxPrices[0]?.LandedPrice.Amount ?? null,
    ourPrice: listing.price,
    weOwnBuybox: data.Offers.some(o => o.SellerId === listing.sellingPartnerId && o.IsBuyBoxWinner),
  });

  // Trigger immediate repricing for this listing
  await enqueueRepricingForListing(listing.id);
}
```

**Testing**:
- `Integration: POST /api/webhooks/amazon/notifications with ANY_OFFER_CHANGED creates competitor observation`
- `Integration: notification where we lose Buy Box triggers immediate repricing job`
- `Integration: notification for unknown ASIN is ignored gracefully (no error)`
- `Unit: AnyOfferChangedNotification payload correctly extracts Buy Box price`
- `Unit: IsBuyBoxWinner=false for our seller triggers repricing; IsBuyBoxWinner=true does not`
- `Fixture: tests/fixtures/sp-api-any-offer-changed.json parses and processes correctly`

#### 7.3 — SP-API Feeds for Bulk Operations

**What**: Implement the Feeds API for bulk listing creation, price updates, and inventory updates (more efficient than individual API calls for large catalogues).

**Design**:

```typescript
// packages/marketplace-connectors/src/amazon/feeds.ts
export class AmazonFeedsClient {
  /** Submit a JSON_LISTINGS_FEED for bulk listing updates */
  async submitListingsFeed(items: ListingFeedItem[]): Promise<FeedSubmissionResult> {
    const feedDocument = await this.createFeedDocument("application/json");
    await this.uploadFeedContent(feedDocument.url, items);
    const submission = await this.createFeed("JSON_LISTINGS_FEED", feedDocument.feedDocumentId);
    return submission;
  }

  /** Submit POST_FLAT_FILE_PRICEANDQUANTITYONLY for bulk price+quantity updates */
  async submitPriceQuantityFeed(updates: PriceQuantityUpdate[]): Promise<FeedSubmissionResult> {
    const tsvContent = this.generatePriceQuantityTsv(updates);
    const feedDocument = await this.createFeedDocument("text/tab-separated-values");
    await this.uploadFeedContent(feedDocument.url, tsvContent);
    return this.createFeed("POST_FLAT_FILE_PRICEANDQUANTITYONLY_UPDATE_DATA", feedDocument.feedDocumentId);
  }

  /** Poll feed processing status */
  async getFeedStatus(feedId: string): Promise<FeedProcessingStatus> {
    return this.client.request("feeds", {
      method: "GET",
      path: `/feeds/2021-06-30/feeds/${feedId}`,
    });
  }
}

interface PriceQuantityUpdate {
  sku: string;
  price: string;
  quantity: number;
}

interface FeedProcessingStatus {
  feedId: string;
  processingStatus: "CANCELLED" | "DONE" | "FATAL" | "IN_PROGRESS" | "IN_QUEUE";
  resultFeedDocumentId?: string;
}
```

**Testing**:
- `Integration (mocked): submitPriceQuantityFeed sends TSV content to feed document URL`
- `Integration (mocked): getFeedStatus returns processing status for submitted feed`
- `Unit: generatePriceQuantityTsv produces valid TSV with sku, price, quantity columns`
- `Unit: feed with 1000 items splits into appropriate batch sizes if needed`

---

## Phase 8: eBay, Walmart & TikTok Shop Connectors

### Purpose

Implement marketplace-specific connectors for eBay, Walmart, and TikTok Shop, following the same `MarketplaceConnector` interface established in Phase 3. After this phase, the platform supports the four primary marketplaces end-to-end.

### Tasks

#### 8.1 — eBay Connector

**What**: eBay REST API connector implementing listing sync, order fetch, inventory update, and fulfillment submission using eBay's OAuth 2.0 authentication.

**Design**:

```typescript
// packages/marketplace-connectors/src/ebay/client.ts
export class EbayClient implements MarketplaceListingConnector, MarketplaceOrderConnector, MarketplaceInventoryConnector {
  private baseUrl = "https://api.ebay.com";

  async pushListing(payload: ListingPushPayload): Promise<ListingPushResult> {
    // Uses eBay Inventory API: PUT /sell/inventory/v1/inventory_item/{sku}
    // Then creates/updates an offer: POST /sell/inventory/v1/offer
    // Then publishes: POST /sell/inventory/v1/offer/{offerId}/publish
  }

  async fetchOrders(since: Date): Promise<RawMarketplaceOrder[]> {
    // Uses eBay Fulfillment API: GET /sell/fulfillment/v1/order?filter=creationdate:[{since}..NOW]
  }

  async updateInventory(marketplaceListingId: string, quantity: number): Promise<InventorySyncResult> {
    // Uses eBay Inventory API: PUT /sell/inventory/v1/inventory_item/{sku}
    // Updates availability.shipToLocationAvailability.quantity
  }

  async submitShipment(orderId: string, shipment: ShipmentSubmission): Promise<void> {
    // Uses eBay Fulfillment API: POST /sell/fulfillment/v1/order/{orderId}/shipping_fulfillment
  }
}
```

**Testing**:
- `Integration (mocked): pushListing creates inventory item, offer, and publishes`
- `Integration (mocked): fetchOrders returns eBay orders mapped to unified format`
- `Integration (mocked): eBay order status "COMPLETED" maps to "shipped"`
- `Unit: eBay item specifics mapped from marketplace_attributes JSONB`
- `Fixture: tests/fixtures/ebay-listing.json round-trips through push/fetch`

#### 8.2 — Walmart Connector

**What**: Walmart Marketplace API connector with OAuth 2.0 client credentials flow (15-minute token expiry), item management, order management, and WFS integration.

**Design**:

```typescript
// packages/marketplace-connectors/src/walmart/client.ts
export class WalmartClient implements MarketplaceListingConnector, MarketplaceOrderConnector, MarketplaceInventoryConnector {
  private baseUrl = "https://marketplace.walmartapis.com/v3";
  // Note: Walmart tokens expire every 15 minutes — aggressive refresh needed

  async pushListing(payload: ListingPushPayload): Promise<ListingPushResult> {
    // Uses Items API: POST /v3/feeds?feedType=item
    // Walmart uses feed-based ingestion, not individual item APIs
  }

  async fetchOrders(since: Date): Promise<RawMarketplaceOrder[]> {
    // Uses Orders API: GET /v3/orders?createdStartDate={since}
  }

  async updateInventory(sku: string, quantity: number): Promise<InventorySyncResult> {
    // Uses Inventory API: PUT /v3/inventory?sku={sku}
  }
}
```

**Testing**:
- `Integration (mocked): Walmart token refresh happens every 15 minutes (not 60)`
- `Integration (mocked): pushListing submits item feed and returns feed ID`
- `Integration (mocked): fetchOrders maps Walmart order statuses to unified statuses`
- `Unit: Walmart authentication uses client_id + client_secret (not refresh token)`

#### 8.3 — TikTok Shop Connector

**What**: TikTok Shop Open Platform API connector with HMAC-SHA256 request signing, product management, order management, and affiliate/creator metadata handling.

**Design**:

```typescript
// packages/marketplace-connectors/src/tiktok/client.ts
export class TikTokShopClient implements MarketplaceListingConnector, MarketplaceOrderConnector {
  private baseUrl = "https://open-api.tiktokglobalshop.com";

  private signRequest(path: string, params: Record<string, string>, body?: string): string {
    const sortedParams = Object.entries(params).sort(([a], [b]) => a.localeCompare(b));
    const baseString = path + sortedParams.map(([k, v]) => k + v).join("") + (body ?? "");
    return createHmac("sha256", this.appSecret).update(baseString).digest("hex");
  }

  async pushListing(payload: ListingPushPayload): Promise<ListingPushResult> {
    // Uses Products API: POST /api/products
    // TikTok requires category_id and package dimensions
  }

  async fetchOrders(since: Date): Promise<RawMarketplaceOrder[]> {
    // Uses Orders API: POST /api/orders/search
    // TikTok orders include creator/affiliate data in marketplace_data
  }
}
```

**Testing**:
- `Unit: HMAC-SHA256 signature generation matches TikTok's documented test vector`
- `Integration (mocked): pushListing sends category_id from marketplace_attributes`
- `Integration (mocked): fetchOrders includes affiliate_commission in marketplace_data`
- `Unit: TikTok order with creator_username populates marketplace_data.creator_username`

---

## Phase 9: Analytics & Seller Dashboard

### Purpose

Implement the analytics aggregation pipeline and the Next.js seller dashboard UI. After this phase, sellers have a full web interface with real-time order monitoring, inventory status, repricing activity, sales analytics by marketplace and product, and seller health metrics.

### Tasks

#### 9.1 — Analytics Aggregation Pipeline

**What**: BullMQ scheduled job that aggregates daily sales, revenue, fees, and profitability data from orders and price change logs into the `daily_summary` table.

**Design**:

```typescript
// packages/worker/src/processors/analytics-aggregator.ts
export async function processAnalyticsAggregation(job: Job<{ sellerId: string; date: string }>) {
  const { sellerId, date } = job.data;
  const summaryDate = new Date(date);

  const marketplaces = await getConnectedMarketplaces(sellerId);
  for (const mp of marketplaces) {
    const orders = await getOrdersForDate(sellerId, mp.id, summaryDate);
    const priceChanges = await countPriceChangesForDate(sellerId, mp.id, summaryDate);

    const metrics = {
      orders: orders.length,
      unitsSold: orders.reduce((sum, o) => sum + o.itemCount, 0),
      grossRevenue: orders.reduce((sum, o) => sum + Number(o.totalAmount), 0).toFixed(2),
      marketplaceFees: orders.reduce((sum, o) => sum + Number(o.marketplaceFees), 0).toFixed(2),
      shippingCosts: orders.reduce((sum, o) => sum + Number(o.shippingCost), 0).toFixed(2),
      netProfit: "0.00", // calculated after COGS available
      repriceCount: priceChanges,
      avgSellingPrice: orders.length > 0
        ? (orders.reduce((sum, o) => sum + Number(o.totalAmount), 0) / orders.length).toFixed(2)
        : null,
    };

    await upsertDailySummary(sellerId, mp.id, summaryDate, metrics);
  }
}
```

Scheduled to run daily at 02:00 UTC via BullMQ repeatable job:

```typescript
await analyticsQueue.add("daily-aggregation", {}, {
  repeat: { pattern: "0 2 * * *" }, // cron: 2 AM UTC daily
});
```

**Testing**:
- `Integration: analytics aggregation for a day with 10 orders produces correct totals`
- `Integration: aggregation with no orders inserts summary with all zeros`
- `Integration: re-running aggregation for same date updates (upserts) rather than duplicating`
- `Unit: revenue calculation uses NUMERIC precision (no floating-point drift on $99.99 * 100)`

#### 9.2 — Next.js Dashboard Foundation

**What**: Next.js 15 App Router setup with Tailwind CSS, shadcn/ui component library, authentication flow (login/register), and layout with sidebar navigation.

**Design**:

Dashboard layout with marketplace-aware navigation:

```typescript
// packages/web/src/app/(dashboard)/layout.tsx
interface DashboardLayoutProps { children: React.ReactNode }

// Sidebar navigation items:
const NAV_ITEMS = [
  { label: "Overview", href: "/overview", icon: "LayoutDashboard" },
  { label: "Products", href: "/products", icon: "Package" },
  { label: "Listings", href: "/listings", icon: "Store" },
  { label: "Inventory", href: "/inventory", icon: "Warehouse" },
  { label: "Orders", href: "/orders", icon: "ShoppingCart" },
  { label: "Repricing", href: "/repricing", icon: "TrendingDown" },
  { label: "Analytics", href: "/analytics", icon: "BarChart3" },
  { label: "Compliance", href: "/compliance", icon: "Shield" },
  { label: "Settings", href: "/settings", icon: "Settings" },
] as const;
```

API proxy from Next.js to Fastify:

```typescript
// packages/web/src/app/api/[...proxy]/route.ts
export async function GET(request: NextRequest) {
  const backendUrl = process.env.API_BACKEND_URL ?? "http://localhost:4000";
  const path = request.nextUrl.pathname.replace("/api", "");
  const response = await fetch(`${backendUrl}/api${path}${request.nextUrl.search}`, {
    headers: { cookie: request.headers.get("cookie") ?? "" },
  });
  return new Response(response.body, {
    status: response.status,
    headers: response.headers,
  });
}
```

**Testing**:
- `E2E (Playwright): login page renders email and password fields`
- `E2E (Playwright): valid login redirects to /overview`
- `E2E (Playwright): invalid login shows error message`
- `E2E (Playwright): sidebar navigation shows all 9 menu items`
- `E2E (Playwright): clicking "Orders" navigates to /orders`

#### 9.3 — Dashboard Pages: Overview, Orders, Inventory, Repricing

**What**: Implement the four primary dashboard pages with data tables, filters, real-time status indicators, and summary cards.

**Design**:

Overview page components:

```typescript
// packages/web/src/app/(dashboard)/overview/page.tsx
// Summary cards:
// - Total Revenue (today / this week / this month)
// - Active Orders (pending + confirmed + processing)
// - Inventory Alerts (low stock count)
// - Buy Box Win Rate (percentage across all Amazon listings)
// - Active Listings (count by marketplace)

// Recent activity feed: last 20 events (orders, repricing, compliance alerts)
// Marketplace health status: connected/disconnected per marketplace
```

Orders page:

```typescript
// packages/web/src/app/(dashboard)/orders/page.tsx
// Filters: marketplace, status, date range, search (order ID, SKU)
// Data table: order ID, marketplace, date, items, total, status, tracking
// Expandable rows: order items detail, shipping address, marketplace-specific data
// Bulk actions: acknowledge, export CSV
```

**Testing**:
- `E2E (Playwright): overview page shows summary cards with correct values from API`
- `E2E (Playwright): orders page loads data table with 50 orders per page`
- `E2E (Playwright): filtering by marketplace=amazon_us updates table to show only Amazon orders`
- `E2E (Playwright): inventory page highlights rows where quantityAvailable <= reorderPoint in red`
- `E2E (Playwright): repricing page shows active rules with last-triggered timestamp`

---

## Phase 10: Compliance Monitoring & Notifications

### Purpose

Implement automated marketplace compliance monitoring (listing suppression detection, policy change alerts, MAP violation detection) and the notification system (in-app, email, webhook). After this phase, sellers receive proactive alerts about compliance risks before they lead to listing removals or account health issues.

### Tasks

#### 10.1 — Compliance Scanning Engine

**What**: Periodic job that checks listing health across marketplaces, detects suppressions, policy violations, and restricted content issues, and creates compliance alerts.

**Design**:

```typescript
// packages/worker/src/processors/compliance-scanner.ts
export async function processComplianceScan(job: Job<{ sellerId: string; marketplaceCode: MarketplaceCode }>) {
  const listings = await getActiveListings(job.data.sellerId, job.data.marketplaceCode);
  const connector = getComplianceConnector(job.data.marketplaceCode);

  for (const listing of listings) {
    const issues = await connector.checkListingHealth(listing.marketplaceListingId!);

    for (const issue of issues) {
      const existingAlert = await findOpenAlert(listing.id, issue.type);
      if (!existingAlert) {
        await createComplianceAlert({
          sellerId: job.data.sellerId,
          marketplaceId: listing.marketplaceId,
          listingId: listing.id,
          alertType: issue.type, // map_violation, suppression, policy_change, restricted_content
          severity: issue.severity,
          details: {
            policyName: issue.policyName,
            violationDescription: issue.description,
            marketplaceReference: issue.reference,
            recommendedAction: issue.recommendation,
            deadline: issue.deadline,
          },
        });
        await enqueueNotification(job.data.sellerId, "compliance", {
          title: `Compliance Alert: ${issue.type}`,
          body: issue.description,
          referenceType: "compliance_alert",
        });
      }
    }
  }
}
```

**Testing**:
- `Integration (mocked): compliance scan detects suppressed listing and creates alert`
- `Integration: duplicate alert for same listing+type is not created if existing alert is open`
- `Integration: resolved alert allows new alert of same type to be created`
- `Unit: compliance alert with severity "critical" triggers immediate notification`

#### 10.2 — Notification System

**What**: Multi-channel notification delivery (in-app, email, webhook) with per-seller preferences, notification history, and read/unread tracking.

**Design**:

```typescript
// packages/api/src/routes/notifications.ts
// GET /api/sellers/:sellerId/notifications?channel=in_app&category=compliance&unreadOnly=true
// PUT /api/sellers/:sellerId/notifications/:notificationId/read
// PUT /api/sellers/:sellerId/notifications/mark-all-read
// POST /api/sellers/:sellerId/notification-preferences
interface NotificationPreferences {
  channels: {
    inApp: boolean;
    email: boolean;
    webhook: boolean;
  };
  categories: {
    order: { inApp: boolean; email: boolean };
    inventory: { inApp: boolean; email: boolean };
    repricing: { inApp: boolean; email: boolean };
    compliance: { inApp: boolean; email: boolean };
    system: { inApp: boolean; email: boolean };
  };
  webhookUrl?: string;
}
```

SSE endpoint for real-time in-app notifications:

```typescript
// GET /api/sellers/:sellerId/notifications/stream (SSE)
export async function notificationStream(request: FastifyRequest, reply: FastifyReply) {
  reply.raw.writeHead(200, {
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache",
    Connection: "keep-alive",
  });

  const subscriber = await redis.subscribe(`notifications:${request.session.sellerId}`);
  subscriber.on("message", (channel, message) => {
    reply.raw.write(`data: ${message}\n\n`);
  });

  request.raw.on("close", () => subscriber.unsubscribe());
}
```

**Testing**:
- `Integration: creating a compliance alert sends in-app notification`
- `Integration: seller with email notifications enabled receives email (mocked SMTP)`
- `Integration: seller with webhookUrl receives POST to webhook endpoint`
- `Integration: GET .../notifications?unreadOnly=true returns only unread notifications`
- `Integration: PUT .../notifications/:id/read marks notification as read`
- `Integration: SSE stream receives real-time notification within 1 second of creation`
- `Unit: notification preferences filter channels per category correctly`

---

## Phase 11: Inventory Forecasting & Low-Stock Alerts

### Purpose

Implement inventory forecasting based on historical sales velocity, configurable low-stock alerts, and reorder suggestions. After this phase, sellers can see predicted stockout dates and receive proactive alerts before running out of inventory on any marketplace.

### Tasks

#### 11.1 — Sales Velocity Calculation & Stockout Prediction

**What**: Calculate rolling sales velocity per product per marketplace, predict stockout dates based on current inventory and velocity, and surface forecasting data through the API.

**Design**:

```typescript
// packages/worker/src/processors/inventory-forecaster.ts
export interface InventoryForecast {
  productId: string;
  sku: string;
  warehouseId: string;
  currentQuantity: number;
  avgDailySalesVelocity: number; // units/day (rolling 30-day average)
  daysOfStockRemaining: number | null; // null if velocity is 0
  predictedStockoutDate: string | null; // ISO 8601
  reorderSuggestion: {
    reorderQuantity: number;
    reorderBy: string; // date by which to reorder to avoid stockout (accounting for lead time)
  } | null;
}

export async function calculateForecast(
  sellerId: string,
  productId: string,
  warehouseId: string,
  leadTimeDays: number = 7,
): Promise<InventoryForecast> {
  const thirtyDaySales = await getUnitsSoldLast30Days(sellerId, productId);
  const velocity = thirtyDaySales / 30;
  const inventory = await getInventoryRecord(productId, warehouseId);
  const daysRemaining = velocity > 0 ? Math.floor(inventory.quantityAvailable / velocity) : null;
  const stockoutDate = daysRemaining !== null
    ? new Date(Date.now() + daysRemaining * 86_400_000).toISOString()
    : null;

  return {
    productId,
    sku: inventory.productSku,
    warehouseId,
    currentQuantity: inventory.quantityAvailable,
    avgDailySalesVelocity: Math.round(velocity * 100) / 100,
    daysOfStockRemaining: daysRemaining,
    predictedStockoutDate: stockoutDate,
    reorderSuggestion: velocity > 0 ? {
      reorderQuantity: Math.ceil(velocity * 30), // 30-day supply
      reorderBy: new Date(Date.now() + (daysRemaining! - leadTimeDays) * 86_400_000).toISOString(),
    } : null,
  };
}

// GET /api/sellers/:sellerId/inventory/forecasts?daysRemaining[lt]=14&sort=daysRemaining:asc
```

**Testing**:
- `Unit: product selling 3 units/day with 42 in stock has daysRemaining=14`
- `Unit: product with zero sales velocity has daysRemaining=null`
- `Unit: reorderBy date accounts for 7-day lead time (stockout in 14 days => reorder in 7 days)`
- `Unit: reorderQuantity suggests 30 days of supply based on velocity`
- `Integration: GET .../inventory/forecasts?daysRemaining[lt]=14 returns only at-risk products`
- `Integration: low stock triggers notification when daysRemaining drops below seller's configured threshold`

---

## Phase 12: AI-Powered Features & MCP Server

### Purpose

Implement the AI-native differentiators: LLM-powered listing content generation, ML-based Buy Box win probability scoring, and an MCP server exposing seller operations to AI agents. These features position the platform ahead of rule-based competitors.

### Tasks

#### 12.1 — LLM-Powered Listing Content Generation

**What**: Generate marketplace-optimised titles, bullet points, descriptions, and search terms using Claude, with per-marketplace prompt tuning and A/B test tracking.

**Design**:

```typescript
// packages/api/src/services/ai-content-service.ts
export class AiContentService {
  async generateListingContent(
    listing: Listing,
    product: Product,
    marketplace: Marketplace,
  ): Promise<GeneratedContent> {
    const systemPrompt = `You are a marketplace listing optimisation expert.
Generate listing content optimised for ${marketplace.name}'s search algorithm (A10/Cassini/equivalent).
Follow these marketplace-specific guidelines:
- Title: max ${MARKETPLACE_TITLE_LIMITS[marketplace.code]} characters
- Bullet points: max ${MARKETPLACE_BULLET_LIMITS[marketplace.code]} bullets, each max 500 characters
- Description: max ${MARKETPLACE_DESC_LIMITS[marketplace.code]} characters
- Include high-value keywords naturally; avoid keyword stuffing
- Match the product category's expected vocabulary`;

    const userPrompt = `Product: ${product.title}
Brand: ${product.brand ?? "Unbranded"}
Category: ${product.category ?? "General"}
SKU: ${product.sku}
Description: ${product.description ?? "No description"}
Attributes: ${JSON.stringify(product.customAttributes)}

Generate optimised listing content for ${marketplace.name}.
Return JSON with: title, bulletPoints (array), description, searchTerms (comma-separated).`;

    const response = await this.anthropic.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 2000,
      system: systemPrompt,
      messages: [{ role: "user", content: userPrompt }],
    });

    const content = JSON.parse(response.content[0].text) as GeneratedContent;

    // Store in listing.aiContent JSONB
    await updateListing(listing.id, {
      aiContent: {
        ...content,
        generationModel: "claude-sonnet-4-20250514",
        generatedAt: new Date().toISOString(),
        approvalStatus: "pending",
      },
    });

    return content;
  }
}

interface GeneratedContent {
  title: string;
  bulletPoints: string[];
  description: string;
  searchTerms: string;
}

// POST /api/sellers/:sellerId/listings/:listingId/generate-content
// POST /api/sellers/:sellerId/listings/:listingId/approve-content — applies AI content to listing
// POST /api/sellers/:sellerId/listings/:listingId/reject-content — clears AI content
```

**Testing**:
- `Integration (mocked LLM): generate-content returns structured JSON with title, bullets, description`
- `Integration: generated content stored in listing.aiContent with approvalStatus="pending"`
- `Integration: approve-content copies AI-generated title/description to listing fields`
- `Unit: Amazon title limited to 200 characters in prompt`
- `Unit: eBay bullet points limited to correct marketplace-specific count`
- `Integration: reject-content clears aiContent and sets approvalStatus="rejected"`

#### 12.2 — Buy Box Win Probability Scoring

**What**: Heuristic scoring model that estimates Buy Box win probability based on price, fulfillment type, seller metrics, and competitor landscape. Provides a probability score alongside repricing recommendations.

**Design**:

```typescript
// packages/shared/src/utils/buybox-scorer.ts
export interface BuyBoxScore {
  winProbability: number; // 0.0 to 1.0
  factors: {
    priceFactor: number; // 0-30 points
    fulfillmentFactor: number; // 0-25 points
    sellerMetricsFactor: number; // 0-25 points
    stockFactor: number; // 0-10 points
    competitorFactor: number; // 0-10 points
  };
  totalScore: number; // 0-100
  recommendation: "maintain" | "lower_price" | "improve_fulfillment" | "no_action";
}

export function calculateBuyBoxScore(
  ourPrice: string,
  competitorPrices: Array<{ price: string; fulfillment: string; isAmazon: boolean }>,
  fulfillmentType: FulfillmentType,
  sellerMetrics: { orderDefectRate: number; lateShipmentRate: number; feedbackScore: number },
  stockLevel: number,
): BuyBoxScore {
  // Price factor: 30 points max (lowest price gets full points, scaled by distance from lowest)
  const prices = competitorPrices.map(c => Number(c.price));
  const lowestPrice = Math.min(...prices);
  const ourPriceNum = Number(ourPrice);
  const priceFactor = ourPriceNum <= lowestPrice ? 30
    : Math.max(0, 30 - ((ourPriceNum - lowestPrice) / lowestPrice) * 100);

  // Fulfillment factor: 25 points (FBA=25, WFS=20, merchant with fast shipping=15, merchant=10)
  const fulfillmentScores: Record<FulfillmentType, number> = {
    fba: 25, wfs: 20, marketplace: 15, merchant: 10,
  };
  const fulfillmentFactor = fulfillmentScores[fulfillmentType] ?? 10;

  // Seller metrics: 25 points
  const defectScore = sellerMetrics.orderDefectRate < 0.01 ? 10 : sellerMetrics.orderDefectRate < 0.02 ? 5 : 0;
  const lateScore = sellerMetrics.lateShipmentRate < 0.04 ? 8 : sellerMetrics.lateShipmentRate < 0.08 ? 4 : 0;
  const feedbackScore = sellerMetrics.feedbackScore >= 4.5 ? 7 : sellerMetrics.feedbackScore >= 4.0 ? 4 : 0;
  const sellerMetricsFactor = defectScore + lateScore + feedbackScore;

  // Stock factor: 10 points (in-stock = 10, low stock = 5, out of stock = 0)
  const stockFactor = stockLevel > 10 ? 10 : stockLevel > 0 ? 5 : 0;

  // Competitor factor: 10 points (fewer competitors = higher score)
  const competitorFactor = competitorPrices.length <= 2 ? 10 : competitorPrices.length <= 5 ? 7 : 3;

  const totalScore = priceFactor + fulfillmentFactor + sellerMetricsFactor + stockFactor + competitorFactor;
  const winProbability = Math.min(1, totalScore / 85); // normalize to 0-1

  return {
    winProbability: Math.round(winProbability * 100) / 100,
    factors: { priceFactor, fulfillmentFactor, sellerMetricsFactor, stockFactor, competitorFactor },
    totalScore,
    recommendation: winProbability >= 0.7 ? "maintain"
      : priceFactor < 20 ? "lower_price"
      : fulfillmentFactor < 15 ? "improve_fulfillment"
      : "no_action",
  };
}

// GET /api/sellers/:sellerId/listings/:listingId/buybox-score
```

**Testing**:
- `Unit: lowest-priced FBA listing with perfect metrics scores above 0.9`
- `Unit: highest-priced merchant listing with poor metrics scores below 0.3`
- `Unit: FBA fulfillment scores 25 points; merchant scores 10 points`
- `Unit: recommendation is "lower_price" when price factor is the weakest component`
- `Unit: recommendation is "maintain" when win probability >= 0.7`
- `Integration: GET .../listings/:id/buybox-score returns score with factor breakdown`

#### 12.3 — MCP Server

**What**: Expose an MCP (Model Context Protocol) server that allows AI agents (Claude, ChatGPT, Copilot) to query listings, check inventory, trigger repricing, and manage orders via natural language tool calls.

**Design**:

```typescript
// packages/api/src/mcp/server.ts
import { McpServer, ResourceTemplate, ToolDefinition } from "@modelcontextprotocol/sdk/server";

const server = new McpServer({
  name: "marketplace-seller-management",
  version: "1.0.0",
});

// Tools exposed to AI agents:
const tools: ToolDefinition[] = [
  {
    name: "list_products",
    description: "List products in the seller's catalogue with optional filtering",
    inputSchema: {
      type: "object",
      properties: {
        search: { type: "string", description: "Search products by title or SKU" },
        brand: { type: "string" },
        status: { type: "string", enum: ["active", "discontinued", "draft"] },
        limit: { type: "number", default: 20 },
      },
    },
  },
  {
    name: "get_inventory_status",
    description: "Get current inventory levels and stockout predictions for a product",
    inputSchema: {
      type: "object",
      properties: {
        productId: { type: "string" },
        sku: { type: "string" },
      },
    },
  },
  {
    name: "reprice_listing",
    description: "Manually set a new price for a listing on a marketplace",
    inputSchema: {
      type: "object",
      properties: {
        listingId: { type: "string" },
        newPrice: { type: "string", description: "New price as decimal string e.g. '27.49'" },
        reason: { type: "string", description: "Why the price is being changed" },
      },
      required: ["listingId", "newPrice"],
    },
  },
  {
    name: "get_order_summary",
    description: "Get summary of recent orders across all marketplaces",
    inputSchema: {
      type: "object",
      properties: {
        daysBack: { type: "number", default: 7 },
        marketplace: { type: "string" },
      },
    },
  },
  {
    name: "check_compliance_alerts",
    description: "List open compliance alerts that need attention",
    inputSchema: {
      type: "object",
      properties: {
        severity: { type: "string", enum: ["info", "warning", "critical"] },
      },
    },
  },
];
```

**Testing**:
- `Integration: MCP server responds to list_products tool call with product data`
- `Integration: MCP server responds to reprice_listing by updating price and creating price_change_log`
- `Integration: MCP server responds to get_inventory_status with forecast data`
- `Integration: MCP server rejects requests without valid seller authentication`
- `Unit: all MCP tool definitions have valid JSON Schema inputSchema`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Scaffolding          ─── required by everything
    │
Phase 2: Seller & Marketplace Management  ─── requires Phase 1
    │
Phase 3: Product Catalogue & Listings     ─── requires Phase 2
    │
    ├── Phase 4: Inventory Management      ─── requires Phase 3
    │       │
    │       └── Phase 11: Inventory Forecasting ─── requires Phase 4 + Phase 9
    │
    ├── Phase 5: Order Management          ─── requires Phase 3, can parallel with Phase 4
    │
    └── Phase 6: Repricing Engine          ─── requires Phase 3
            │
            └── Phase 7: Amazon SP-API     ─── requires Phase 6
                    │
                    └── Phase 8: eBay/Walmart/TikTok ─── requires Phase 3, can parallel with Phase 7
    │
    Phase 9: Analytics & Dashboard         ─── requires Phases 4, 5, 6
    │
    Phase 10: Compliance & Notifications   ─── requires Phase 3 + Phase 9
    │
    Phase 12: AI Features & MCP            ─── requires Phases 6, 9

Parallelism opportunities:
  - Phases 4 (Inventory) and 5 (Orders) can be developed concurrently after Phase 3
  - Phase 8 (eBay/Walmart/TikTok) can proceed in parallel with Phase 7 (Amazon SP-API)
  - Phases 10 (Compliance) and 11 (Forecasting) can be developed concurrently after Phase 9
  - Phase 12 (AI) can begin once Phase 6 and Phase 9 are complete
```

---

## Definition of Done (per phase)

1. All tasks within the phase are implemented.
2. All unit tests pass (`pnpm test:unit`).
3. All integration tests pass (`pnpm test:integration`) — including tests with Testcontainers for PostgreSQL and Redis.
4. Biome linting passes with zero errors (`pnpm lint`).
5. TypeScript strict mode compilation succeeds with zero errors (`pnpm typecheck`).
6. Docker build succeeds for all modified services (`docker-compose build`).
7. The feature works end-to-end when running `docker-compose up` locally.
8. New API endpoints appear in auto-generated OpenAPI 3.1 spec at `/docs`.
9. Database migrations created and applied without errors (`pnpm db:migrate`).
10. New configuration options documented in `.env.example` with defaults.
11. No credentials, secrets, or API keys committed to source control.
12. Rate-limiting respects per-marketplace API quotas (no API suspension risk).
