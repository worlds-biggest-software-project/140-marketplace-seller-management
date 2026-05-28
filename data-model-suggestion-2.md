# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Marketplace Seller Management · Created: 2026-05-19

## Philosophy

This model treats every state change as an immutable event written to an append-only event store. The current state of any entity — a listing's price, an order's status, an inventory level — is derived by replaying its event stream. Separate read-optimized materialised views (projections) serve queries, while the event store remains the single source of truth. This is the CQRS (Command Query Responsibility Segregation) pattern.

This approach is inspired by how financial ledgers and Amazon SP-API's Notifications API work: every price change, order status transition, and inventory adjustment is a discrete, timestamped event. The repricing domain is a natural fit — the ability to answer "what was the Buy Box price at 14:32 on Tuesday?" or "how many times did we reprice this listing in the last hour?" comes for free when every mutation is an event. Audit trails, regulatory compliance, and ML training data are first-class outputs rather than afterthoughts.

The trade-off is complexity. Event replay for entities with thousands of events requires snapshotting. Read models must be maintained (eventually consistent). The engineering team needs experience with event-sourced architectures to avoid common pitfalls like event schema evolution and projection rebuild latency.

**Best for:** Teams building a platform where complete audit history, temporal queries, AI/ML training on behavioural data, and regulatory compliance are critical requirements.

**Trade-offs:**
- Pro: Complete, immutable audit trail — every change is preserved forever
- Pro: Temporal queries ("what was true at time T?") are trivial
- Pro: Event streams are ideal training data for ML repricing models
- Pro: Natural fit for real-time event processing (SP-API notifications, webhooks)
- Pro: Easy to add new read models (projections) without changing the write path
- Con: Higher implementation complexity — event replay, snapshotting, projection rebuilds
- Con: Eventually consistent read models — not instant after writes
- Con: Event schema evolution requires careful versioning
- Con: Higher storage requirements (every change stored, not just current state)
- Con: Debugging requires understanding event streams, not just current rows

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | Product identifier in `ProductRegistered` and `ListingCreated` events; immutable once set |
| ISO 4217 | Currency codes embedded in all monetary event payloads |
| ISO 3166-1 alpha-2 | Country codes in marketplace and shipping address events |
| ISO 8601 | All event timestamps in RFC 3339 / ISO 8601 format with timezone |
| Amazon SP-API Notifications | Inbound marketplace events (ANY_OFFER_CHANGED, ORDER_STATUS_CHANGE) map directly to domain events |
| CloudEvents v1.0 | Event envelope structure follows CloudEvents spec for interoperability |
| OCSF (Open Cybersecurity Schema Framework) | Audit event categorization borrows from OCSF activity/category taxonomy |
| RFC 6749 (OAuth 2.0) | Credential lifecycle events (token granted, refreshed, revoked) tracked in event store |

---

## Event Store

The event store is the single source of truth. All mutations are written here as immutable events.

```sql
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                 -- aggregate root ID (product, listing, order, etc.)
    stream_type     TEXT NOT NULL,                  -- 'Product', 'Listing', 'Order', 'Inventory', 'RepricingRule'
    event_type      TEXT NOT NULL,                  -- 'ListingPriceChanged', 'OrderPlaced', 'InventoryAdjusted'
    event_version   INT NOT NULL,                   -- sequential version within this stream
    payload         JSONB NOT NULL,                 -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',    -- correlation_id, causation_id, user_id, source
    seller_id       UUID NOT NULL,                  -- denormalized for partition/query efficiency
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, event_version)                -- ensures ordering within a stream
);

-- Primary query: replay events for an aggregate
CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);

-- Query by type for projections
CREATE INDEX idx_event_type ON event_store(event_type, created_at);

-- Tenant isolation
CREATE INDEX idx_event_seller ON event_store(seller_id, created_at);

-- Time-range queries for analytics
CREATE INDEX idx_event_created ON event_store(created_at);

-- Partition by month for scalability
-- ALTER TABLE event_store PARTITION BY RANGE (created_at);
```

### Event Payload Examples

```sql
-- ListingPriceChanged event payload:
-- {
--   "listing_id": "uuid",
--   "marketplace_code": "amazon_us",
--   "old_price": 29.99,
--   "new_price": 27.49,
--   "currency_code": "USD",
--   "reason": "repricing_rule",
--   "repricing_rule_id": "uuid",
--   "competitor_price": 27.99,
--   "buy_box_owned_before": false,
--   "buy_box_owned_after": true
-- }

-- OrderPlaced event payload:
-- {
--   "marketplace_code": "amazon_us",
--   "marketplace_order_id": "111-2222222-3333333",
--   "items": [
--     {"sku": "WIDGET-001", "quantity": 2, "unit_price": 27.49}
--   ],
--   "subtotal": 54.98,
--   "shipping_cost": 5.99,
--   "tax": 4.85,
--   "total": 65.82,
--   "buyer_name": "Jane Doe",
--   "shipping_address": {
--     "line1": "123 Main St",
--     "city": "Portland",
--     "state": "OR",
--     "postal_code": "97201",
--     "country_code": "US"
--   }
-- }

-- InventoryAdjusted event payload:
-- {
--   "product_id": "uuid",
--   "variant_id": "uuid",
--   "warehouse_id": "uuid",
--   "adjustment_type": "sale",
--   "quantity_change": -2,
--   "quantity_before": 150,
--   "quantity_after": 148,
--   "reference_type": "order",
--   "reference_id": "uuid"
-- }

-- CompetitorPriceObserved event payload:
-- {
--   "listing_id": "uuid",
--   "marketplace_code": "amazon_us",
--   "competitor_name": "Buy Box Winner",
--   "price": 27.99,
--   "shipping": 0.00,
--   "fulfillment_type": "fba",
--   "is_buy_box": true,
--   "source": "sp_api_notification"
-- }
```

## Snapshot Store

Snapshots prevent expensive full replays for entities with many events.

```sql
CREATE TABLE event_snapshot (
    snapshot_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    event_version   INT NOT NULL,                  -- snapshot taken at this version
    state           JSONB NOT NULL,                 -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_snapshot_stream_version ON event_snapshot(stream_id, event_version DESC);
```

### Snapshot Strategy

```sql
-- To load an aggregate:
-- 1. Find latest snapshot for stream_id
-- 2. Replay events after snapshot's event_version
-- 3. Return hydrated aggregate

-- Example: load listing aggregate
-- SELECT state, event_version FROM event_snapshot
--   WHERE stream_id = $1 ORDER BY event_version DESC LIMIT 1;
-- SELECT * FROM event_store
--   WHERE stream_id = $1 AND event_version > $snapshot_version
--   ORDER BY event_version;
```

---

## Materialised Read Models (Projections)

These tables are rebuilt from the event store. They are the query layer — optimized for reads, eventually consistent with writes.

### Listing Projection

```sql
CREATE TABLE v_listing (
    listing_id          UUID PRIMARY KEY,
    seller_id           UUID NOT NULL,
    product_id          UUID NOT NULL,
    variant_id          UUID,
    marketplace_id      UUID NOT NULL,
    marketplace_code    TEXT NOT NULL,
    marketplace_listing_id TEXT,
    sku                 TEXT NOT NULL,
    title               TEXT NOT NULL,
    description         TEXT,
    current_price       NUMERIC(12,2) NOT NULL,
    currency_code       CHAR(3) NOT NULL,
    condition           TEXT NOT NULL DEFAULT 'new',
    fulfillment_type    TEXT NOT NULL DEFAULT 'merchant',
    status              TEXT NOT NULL DEFAULT 'draft',
    buy_box_owned       BOOLEAN DEFAULT false,
    sync_status         TEXT NOT NULL DEFAULT 'pending',
    last_repriced_at    TIMESTAMPTZ,
    reprice_count_24h   INT DEFAULT 0,
    last_synced_at      TIMESTAMPTZ,
    event_version       INT NOT NULL,              -- tracks projection currency
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_listing_seller ON v_listing(seller_id);
CREATE INDEX idx_v_listing_marketplace ON v_listing(marketplace_id);
CREATE INDEX idx_v_listing_status ON v_listing(status);
CREATE INDEX idx_v_listing_buybox ON v_listing(buy_box_owned) WHERE buy_box_owned = true;
```

### Order Projection

```sql
CREATE TABLE v_order (
    order_id                UUID PRIMARY KEY,
    seller_id               UUID NOT NULL,
    marketplace_id          UUID NOT NULL,
    marketplace_code        TEXT NOT NULL,
    marketplace_order_id    TEXT NOT NULL,
    order_status            TEXT NOT NULL,
    order_date              TIMESTAMPTZ NOT NULL,
    currency_code           CHAR(3) NOT NULL,
    subtotal                NUMERIC(12,2) NOT NULL,
    shipping_cost           NUMERIC(12,2) NOT NULL DEFAULT 0,
    tax_amount              NUMERIC(12,2) NOT NULL DEFAULT 0,
    marketplace_fees        NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_amount            NUMERIC(12,2) NOT NULL,
    net_payout              NUMERIC(12,2),
    item_count              INT NOT NULL DEFAULT 0,
    buyer_name              TEXT,
    shipping_country_code   CHAR(2),
    fulfillment_type        TEXT,
    tracking_number         TEXT,
    carrier_code            TEXT,
    shipped_at              TIMESTAMPTZ,
    delivered_at            TIMESTAMPTZ,
    event_version           INT NOT NULL,
    projected_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_order_seller ON v_order(seller_id);
CREATE INDEX idx_v_order_marketplace ON v_order(marketplace_id);
CREATE INDEX idx_v_order_status ON v_order(order_status);
CREATE INDEX idx_v_order_date ON v_order(order_date);
```

### Inventory Projection

```sql
CREATE TABLE v_inventory (
    inventory_id        UUID PRIMARY KEY,
    seller_id           UUID NOT NULL,
    product_id          UUID NOT NULL,
    variant_id          UUID,
    warehouse_id        UUID NOT NULL,
    sku                 TEXT NOT NULL,
    quantity_available  INT NOT NULL DEFAULT 0,
    quantity_reserved   INT NOT NULL DEFAULT 0,
    quantity_on_order   INT NOT NULL DEFAULT 0,
    cost_per_unit       NUMERIC(12,4),
    reorder_point       INT,
    is_low_stock        BOOLEAN DEFAULT false,
    last_counted_at     TIMESTAMPTZ,
    event_version       INT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_inventory_seller ON v_inventory(seller_id);
CREATE INDEX idx_v_inventory_product ON v_inventory(product_id);
CREATE INDEX idx_v_inventory_low ON v_inventory(is_low_stock) WHERE is_low_stock = true;
```

### Repricing Activity Projection

```sql
CREATE TABLE v_repricing_activity (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id          UUID NOT NULL,
    seller_id           UUID NOT NULL,
    marketplace_code    TEXT NOT NULL,
    sku                 TEXT NOT NULL,
    old_price           NUMERIC(12,2) NOT NULL,
    new_price           NUMERIC(12,2) NOT NULL,
    currency_code       CHAR(3) NOT NULL,
    price_direction     TEXT NOT NULL,              -- increase, decrease
    change_pct          NUMERIC(8,4) NOT NULL,
    reason              TEXT NOT NULL,
    repricing_rule_id   UUID,
    competitor_price    NUMERIC(12,2),
    buy_box_won         BOOLEAN,
    occurred_at         TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_reprice_listing ON v_repricing_activity(listing_id, occurred_at);
CREATE INDEX idx_v_reprice_seller ON v_repricing_activity(seller_id, occurred_at);
-- Partition by month for analytics queries
```

### Competitor Price History Projection

```sql
CREATE TABLE v_competitor_price_history (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id          UUID NOT NULL,
    marketplace_code    TEXT NOT NULL,
    competitor_name     TEXT,
    price               NUMERIC(12,2) NOT NULL,
    shipping_price      NUMERIC(12,2) DEFAULT 0,
    fulfillment_type    TEXT,
    is_buy_box          BOOLEAN DEFAULT false,
    observed_at         TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_comp_listing ON v_competitor_price_history(listing_id, observed_at);
-- Partition by month
```

### Daily Analytics Projection

```sql
CREATE TABLE v_daily_analytics (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id           UUID NOT NULL,
    marketplace_code    TEXT NOT NULL,
    product_sku         TEXT,
    summary_date        DATE NOT NULL,
    units_sold          INT NOT NULL DEFAULT 0,
    gross_revenue       NUMERIC(14,2) NOT NULL DEFAULT 0,
    marketplace_fees    NUMERIC(14,2) NOT NULL DEFAULT 0,
    shipping_cost       NUMERIC(14,2) NOT NULL DEFAULT 0,
    net_profit          NUMERIC(14,2) NOT NULL DEFAULT 0,
    reprice_count       INT NOT NULL DEFAULT 0,
    avg_price           NUMERIC(12,2),
    buy_box_win_pct     NUMERIC(5,2),
    UNIQUE(seller_id, marketplace_code, product_sku, summary_date)
);

CREATE INDEX idx_v_analytics_seller_date ON v_daily_analytics(seller_id, summary_date);
```

---

## Reference Data Tables

These are not event-sourced — they are stable reference data used by projections and commands.

```sql
CREATE TABLE seller (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_name    TEXT NOT NULL,
    contact_email   TEXT NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    subscription_tier TEXT NOT NULL DEFAULT 'free',
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE marketplace (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            TEXT NOT NULL UNIQUE,
    name            TEXT NOT NULL,
    region          TEXT NOT NULL,
    country_code    CHAR(2),
    currency_code   CHAR(3) NOT NULL,
    api_type        TEXT NOT NULL DEFAULT 'rest',
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE warehouse (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    name            TEXT NOT NULL,
    warehouse_type  TEXT NOT NULL DEFAULT 'own',
    country_code    CHAR(2) NOT NULL,
    is_default      BOOLEAN DEFAULT false,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Type Catalogue

| Stream Type | Event Type | Description |
|-------------|-----------|-------------|
| Product | ProductRegistered | New product added to catalogue |
| Product | ProductUpdated | Product details changed (title, description, weight) |
| Product | ProductDiscontinued | Product marked as discontinued |
| Product | VariantAdded | New variant added to product |
| Product | VariantUpdated | Variant attributes changed |
| Listing | ListingCreated | New listing drafted for a marketplace |
| Listing | ListingPublished | Listing pushed to marketplace and confirmed active |
| Listing | ListingPriceChanged | Price updated (manual or repricing engine) |
| Listing | ListingSuppressed | Marketplace suppressed the listing |
| Listing | ListingSyncFailed | Sync to marketplace failed with error |
| Listing | BuyBoxWon | Seller won the Buy Box |
| Listing | BuyBoxLost | Seller lost the Buy Box |
| Listing | CompetitorPriceObserved | Competitor price captured from marketplace |
| Order | OrderPlaced | New order received from marketplace |
| Order | OrderConfirmed | Order confirmed and ready for fulfillment |
| Order | OrderShipped | Shipment created with tracking |
| Order | OrderDelivered | Delivery confirmed |
| Order | OrderCancelled | Order cancelled (by buyer or seller) |
| Order | OrderRefunded | Full or partial refund issued |
| Order | OrderReturnInitiated | Return request received |
| Inventory | InventoryInitialized | Initial stock count set for product/warehouse |
| Inventory | InventoryAdjusted | Stock adjusted (sale, return, restock, transfer, count) |
| Inventory | InventoryReserved | Stock reserved for pending order |
| Inventory | ReservationReleased | Reserved stock released (cancellation) |
| Inventory | LowStockAlertTriggered | Available quantity dropped below reorder point |
| Inventory | ChannelAllocationChanged | Inventory allocation per marketplace changed |
| Repricing | RepricingRuleCreated | New repricing rule defined |
| Repricing | RepricingRuleUpdated | Repricing rule parameters changed |
| Repricing | RepricingRuleDeactivated | Repricing rule turned off |
| Repricing | RepricingTriggered | Repricing engine fired and changed a price |
| Credential | CredentialLinked | Marketplace credential connected |
| Credential | TokenRefreshed | OAuth token refreshed |
| Credential | CredentialRevoked | Marketplace credential disconnected |
| Compliance | ComplianceAlertRaised | Policy violation or suppression risk detected |
| Compliance | ComplianceAlertResolved | Alert resolved or dismissed |

---

## Example Temporal Queries

```sql
-- What was the price of listing X at a specific point in time?
SELECT payload->>'new_price' AS price, created_at
FROM event_store
WHERE stream_id = $listing_id
  AND event_type = 'ListingPriceChanged'
  AND created_at <= '2026-05-15 14:32:00+00'
ORDER BY event_version DESC
LIMIT 1;

-- How many times was listing X repriced in the last 24 hours?
SELECT COUNT(*) AS reprice_count
FROM event_store
WHERE stream_id = $listing_id
  AND event_type IN ('ListingPriceChanged', 'RepricingTriggered')
  AND created_at >= now() - INTERVAL '24 hours';

-- Full order lifecycle timeline
SELECT event_type, payload, created_at
FROM event_store
WHERE stream_id = $order_id
  AND stream_type = 'Order'
ORDER BY event_version;

-- Inventory movement history for a product across all warehouses
SELECT
    payload->>'warehouse_id' AS warehouse,
    payload->>'adjustment_type' AS type,
    (payload->>'quantity_change')::int AS qty_change,
    (payload->>'quantity_after')::int AS qty_after,
    created_at
FROM event_store
WHERE stream_type = 'Inventory'
  AND payload->>'product_id' = $product_id
  AND created_at BETWEEN '2026-05-01' AND '2026-05-19'
ORDER BY created_at;

-- Repricing effectiveness: price changes that resulted in Buy Box wins
SELECT
    payload->>'listing_id' AS listing,
    (payload->>'old_price')::numeric AS old_price,
    (payload->>'new_price')::numeric AS new_price,
    payload->>'reason' AS reason,
    (payload->>'buy_box_owned_after')::boolean AS won_buybox,
    created_at
FROM event_store
WHERE event_type = 'ListingPriceChanged'
  AND seller_id = $seller_id
  AND (payload->>'buy_box_owned_after')::boolean = true
  AND created_at >= now() - INTERVAL '7 days'
ORDER BY created_at DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 2 | event_store, event_snapshot |
| Reference Data | 3 | seller, marketplace, warehouse |
| Listing Projections | 2 | v_listing, v_repricing_activity |
| Order Projections | 1 | v_order |
| Inventory Projections | 1 | v_inventory |
| Competitive Intelligence Projections | 1 | v_competitor_price_history |
| Analytics Projections | 1 | v_daily_analytics |
| **Total** | **11** | Plus any additional projections added over time |

---

## Key Design Decisions

1. **Single event store table with `stream_type` discriminator** — rather than separate event tables per aggregate, a single table simplifies infrastructure and enables cross-aggregate queries. The `stream_id` + `event_version` unique constraint guarantees ordering within each aggregate. Partitioning by `created_at` handles growth.

2. **JSONB payloads with typed event names** — event payloads are schemaless JSONB, but each `event_type` has a documented schema. This provides flexibility for event evolution while keeping the event store table structure stable. New event types require no DDL changes.

3. **Metadata field for correlation** — the `metadata` JSONB column carries `correlation_id` (which user action or API call triggered this chain), `causation_id` (which prior event caused this one), and `user_id`. This enables end-to-end request tracing through event chains.

4. **Projections are disposable and rebuildable** — all `v_*` tables can be dropped and rebuilt from the event store. Each projection tracks its `event_version` to know how current it is. This makes adding new read models (e.g., a new analytics dashboard) safe and reversible.

5. **Snapshot strategy at every 100 events** — for aggregates with high event velocity (listings being repriced every few minutes), snapshots are taken every 100 events to keep replay time under 50ms. Snapshots are stored as JSONB in `event_snapshot`.

6. **Marketplace events map to domain events** — inbound SP-API notifications (ANY_OFFER_CHANGED), eBay webhook events, and TikTok Shop callbacks are translated into domain events and stored in the same event store. This provides a unified timeline of both internal actions and external marketplace signals.

7. **Competitor observations as events, not separate tables** — rather than a dedicated competitor_price table in the write model, competitor observations are `CompetitorPriceObserved` events. The `v_competitor_price_history` projection serves read queries. This keeps the write path uniform.

8. **Event type catalogue as living documentation** — the event type table above serves as the contract between write-side command handlers and read-side projection builders. Adding a new event type is the primary extension mechanism.

9. **No foreign keys between event store and projections** — projections reference aggregate IDs but have no FK constraints to the event store. This decoupling allows projections to be rebuilt, replaced, or run on different database instances.

10. **Time-partitioned event store for operational manageability** — monthly partitions on `created_at` enable efficient archival of old events to cold storage, partition-level VACUUM, and time-range analytics queries without full table scans.
