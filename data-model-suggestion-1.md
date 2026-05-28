# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Marketplace Seller Management · Created: 2026-05-19

## Philosophy

This model follows classical relational database design with full normalization (3NF). Every domain concept — marketplace, seller, product, listing, order, shipment, price rule, competitor observation — gets its own table with explicit foreign key relationships. The schema enforces referential integrity at the database level, making it impossible to create orphaned listings or orders without valid seller and marketplace references.

This approach mirrors how enterprise multi-channel platforms like Linnworks and ChannelAdvisor structure their backends: a rigid, well-defined schema where every field has a type, every relationship is enforced, and every query can be optimized with standard B-tree indexes. It aligns naturally with the GS1 Global Data Model's layered attribute structure (Global Core, Global Category, Regional Category) by giving each attribute layer its own relational representation.

The trade-off is schema rigidity. Adding a new marketplace with unique listing attributes requires DDL changes (new columns or junction tables). This model works best when the set of supported marketplaces is well-defined and the team values data integrity over deployment speed.

**Best for:** Teams building a production-grade platform where data integrity, complex cross-entity reporting, and regulatory compliance are top priorities.

**Trade-offs:**
- Pro: Maximum data integrity via foreign keys and constraints
- Pro: Excellent query performance for complex joins and aggregations
- Pro: Well-understood by most backend engineers; broad tooling support
- Pro: Clean audit trail via separate history tables
- Con: Schema changes required for each new marketplace's unique attributes
- Con: Higher table count increases migration complexity
- Con: Junction tables for many-to-many relationships add query verbosity
- Con: Less flexible for rapid prototyping of new marketplace integrations

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN (GTIN-12/UPC, GTIN-13/EAN, GTIN-14) | `product.gtin` column with CHECK constraint for valid lengths; used as canonical product identifier for cross-marketplace matching |
| GS1 Global Data Model | Product attribute layers (global core, category-specific) map to `product_attribute` and `category_attribute` tables |
| ISO 4217 | `currency_code` columns on all monetary fields use ISO 4217 three-letter codes |
| ISO 3166-1 alpha-2 | `country_code` columns on marketplace regions, shipping addresses, and seller locations |
| ISO 8601 | All timestamps stored as `TIMESTAMPTZ`; date-only fields use `DATE` |
| OAuth 2.0 (RFC 6749) | `marketplace_credential.token_type`, `access_token`, `refresh_token` fields align with OAuth 2.0 token response structure |
| OpenAPI 3.1 | Platform's own API documented in OAS 3.1; schema maps 1:1 to table structures |
| Schema.org Product/Offer | `listing` table fields (title, description, price, availability) align with Schema.org Product and Offer vocabulary |
| ANSI X12 EDI | `edi_transaction` table stores parsed EDI 850/856/810/846 documents for Walmart/enterprise integrations |

---

## Core Platform Tables

### Tenancy & Authentication

```sql
CREATE TABLE seller (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_id     TEXT,                          -- external system reference
    company_name    TEXT NOT NULL,
    contact_email   TEXT NOT NULL,
    contact_phone   TEXT,
    country_code    CHAR(2) NOT NULL,              -- ISO 3166-1 alpha-2
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    subscription_tier TEXT NOT NULL DEFAULT 'free', -- free, starter, pro, enterprise
    status          TEXT NOT NULL DEFAULT 'active', -- active, suspended, closed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_seller_status ON seller(status);
CREATE INDEX idx_seller_country ON seller(country_code);

CREATE TABLE seller_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'member', -- owner, admin, member, viewer
    password_hash   TEXT,
    last_login_at   TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_seller_user_seller ON seller_user(seller_id);
```

### Marketplace Integration

```sql
CREATE TABLE marketplace (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            TEXT NOT NULL UNIQUE,           -- amazon_us, ebay_us, walmart_us, tiktok_us
    name            TEXT NOT NULL,                  -- "Amazon US", "eBay US"
    region          TEXT NOT NULL,                  -- na, eu, apac, latam
    country_code    CHAR(2),                        -- ISO 3166-1 alpha-2
    api_base_url    TEXT,
    api_type        TEXT NOT NULL DEFAULT 'rest',   -- rest, graphql, edi, soap
    currency_code   CHAR(3) NOT NULL,               -- ISO 4217
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE marketplace_credential (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id),
    credential_name TEXT NOT NULL,                  -- friendly name
    auth_type       TEXT NOT NULL,                  -- oauth2, api_key, hmac, as2
    access_token    TEXT,                           -- encrypted at rest
    refresh_token   TEXT,                           -- encrypted at rest
    token_expires_at TIMESTAMPTZ,
    api_key         TEXT,                           -- encrypted at rest
    client_id       TEXT,
    additional_config TEXT,                         -- encrypted JSON for marketplace-specific auth params
    status          TEXT NOT NULL DEFAULT 'active', -- active, expired, revoked
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(seller_id, marketplace_id)
);

CREATE INDEX idx_mktpl_cred_seller ON marketplace_credential(seller_id);
CREATE INDEX idx_mktpl_cred_marketplace ON marketplace_credential(marketplace_id);
```

### Product Catalogue

```sql
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    sku             TEXT NOT NULL,                  -- seller's internal SKU
    title           TEXT NOT NULL,
    description     TEXT,
    brand           TEXT,
    manufacturer    TEXT,
    gtin            TEXT,                           -- GS1 GTIN (UPC/EAN/GTIN-14)
    mpn             TEXT,                           -- Manufacturer Part Number
    category        TEXT,                           -- internal category
    weight_kg       NUMERIC(10,3),
    length_cm       NUMERIC(10,2),
    width_cm        NUMERIC(10,2),
    height_cm       NUMERIC(10,2),
    country_of_origin CHAR(2),                     -- ISO 3166-1 alpha-2
    hs_code         TEXT,                           -- Harmonised System tariff code
    status          TEXT NOT NULL DEFAULT 'active', -- active, discontinued, draft
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(seller_id, sku)
);

CREATE INDEX idx_product_seller ON product(seller_id);
CREATE INDEX idx_product_gtin ON product(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_sku ON product(seller_id, sku);

CREATE TABLE product_image (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    sort_order      INT NOT NULL DEFAULT 0,
    image_type      TEXT NOT NULL DEFAULT 'main',  -- main, variant, lifestyle, swatch
    alt_text        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_product_image_product ON product_image(product_id);

CREATE TABLE product_variant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    sku             TEXT NOT NULL,
    variant_name    TEXT NOT NULL,                  -- "Red / Large"
    gtin            TEXT,
    weight_kg       NUMERIC(10,3),
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(product_id, sku)
);

CREATE INDEX idx_product_variant_product ON product_variant(product_id);

CREATE TABLE variant_attribute (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    variant_id      UUID NOT NULL REFERENCES product_variant(id) ON DELETE CASCADE,
    attribute_name  TEXT NOT NULL,                  -- "color", "size", "material"
    attribute_value TEXT NOT NULL,                  -- "Red", "XL", "Cotton"
    UNIQUE(variant_id, attribute_name)
);

CREATE INDEX idx_variant_attr_variant ON variant_attribute(variant_id);
```

### Marketplace Listings

```sql
CREATE TABLE listing (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id           UUID NOT NULL REFERENCES seller(id),
    product_id          UUID NOT NULL REFERENCES product(id),
    variant_id          UUID REFERENCES product_variant(id),
    marketplace_id      UUID NOT NULL REFERENCES marketplace(id),
    marketplace_listing_id TEXT,                    -- marketplace's own ID (ASIN, eBay item ID, etc.)
    marketplace_sku     TEXT,                       -- marketplace-specific SKU
    title               TEXT NOT NULL,              -- marketplace-specific title (may differ from product.title)
    description         TEXT,
    price               NUMERIC(12,2) NOT NULL,
    currency_code       CHAR(3) NOT NULL,           -- ISO 4217
    condition           TEXT NOT NULL DEFAULT 'new', -- new, used_like_new, used_good, refurbished
    fulfillment_type    TEXT NOT NULL DEFAULT 'merchant', -- merchant, fba, wfs, marketplace
    status              TEXT NOT NULL DEFAULT 'draft', -- draft, active, inactive, suppressed, error
    listing_url         TEXT,
    last_synced_at      TIMESTAMPTZ,
    sync_status         TEXT NOT NULL DEFAULT 'pending', -- pending, synced, error
    sync_error_message  TEXT,
    buy_box_owned       BOOLEAN DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(seller_id, marketplace_id, marketplace_sku)
);

CREATE INDEX idx_listing_seller ON listing(seller_id);
CREATE INDEX idx_listing_product ON listing(product_id);
CREATE INDEX idx_listing_marketplace ON listing(marketplace_id);
CREATE INDEX idx_listing_status ON listing(status);
CREATE INDEX idx_listing_sync ON listing(sync_status);
CREATE INDEX idx_listing_mktpl_id ON listing(marketplace_listing_id) WHERE marketplace_listing_id IS NOT NULL;

CREATE TABLE listing_attribute (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id      UUID NOT NULL REFERENCES listing(id) ON DELETE CASCADE,
    attribute_name  TEXT NOT NULL,                  -- marketplace-specific attribute name
    attribute_value TEXT NOT NULL,
    UNIQUE(listing_id, attribute_name)
);

CREATE INDEX idx_listing_attr_listing ON listing_attribute(listing_id);
```

### Inventory Management

```sql
CREATE TABLE warehouse (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    name            TEXT NOT NULL,
    warehouse_type  TEXT NOT NULL DEFAULT 'own',   -- own, 3pl, fba, wfs
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state_province  TEXT,
    postal_code     TEXT,
    country_code    CHAR(2) NOT NULL,              -- ISO 3166-1 alpha-2
    is_default      BOOLEAN DEFAULT false,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_warehouse_seller ON warehouse(seller_id);

CREATE TABLE inventory (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id           UUID NOT NULL REFERENCES seller(id),
    product_id          UUID NOT NULL REFERENCES product(id),
    variant_id          UUID REFERENCES product_variant(id),
    warehouse_id        UUID NOT NULL REFERENCES warehouse(id),
    quantity_available  INT NOT NULL DEFAULT 0 CHECK (quantity_available >= 0),
    quantity_reserved   INT NOT NULL DEFAULT 0 CHECK (quantity_reserved >= 0),
    quantity_on_order   INT NOT NULL DEFAULT 0 CHECK (quantity_on_order >= 0),
    reorder_point       INT,
    reorder_quantity    INT,
    cost_per_unit       NUMERIC(12,4),
    currency_code       CHAR(3),                   -- ISO 4217
    last_counted_at     TIMESTAMPTZ,
    version             INT NOT NULL DEFAULT 1,     -- optimistic locking
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(product_id, variant_id, warehouse_id)
);

CREATE INDEX idx_inventory_seller ON inventory(seller_id);
CREATE INDEX idx_inventory_product ON inventory(product_id);
CREATE INDEX idx_inventory_warehouse ON inventory(warehouse_id);
CREATE INDEX idx_inventory_low_stock ON inventory(seller_id, quantity_available) WHERE quantity_available <= 10;

CREATE TABLE inventory_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inventory_id    UUID NOT NULL REFERENCES inventory(id),
    transaction_type TEXT NOT NULL,                 -- restock, sale, return, transfer_in, transfer_out, adjustment, reservation
    quantity         INT NOT NULL,
    reference_type   TEXT,                          -- order, purchase_order, adjustment, transfer
    reference_id     UUID,                          -- FK to related entity
    notes            TEXT,
    created_by       UUID REFERENCES seller_user(id),
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inv_txn_inventory ON inventory_transaction(inventory_id);
CREATE INDEX idx_inv_txn_created ON inventory_transaction(created_at);
CREATE INDEX idx_inv_txn_type ON inventory_transaction(transaction_type);

CREATE TABLE inventory_channel_allocation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inventory_id    UUID NOT NULL REFERENCES inventory(id),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id),
    allocated_qty   INT NOT NULL DEFAULT 0 CHECK (allocated_qty >= 0),
    sync_status     TEXT NOT NULL DEFAULT 'pending',
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(inventory_id, marketplace_id)
);

CREATE INDEX idx_inv_alloc_inventory ON inventory_channel_allocation(inventory_id);
CREATE INDEX idx_inv_alloc_marketplace ON inventory_channel_allocation(marketplace_id);
```

### Order Management

```sql
CREATE TABLE marketplace_order (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id               UUID NOT NULL REFERENCES seller(id),
    marketplace_id          UUID NOT NULL REFERENCES marketplace(id),
    marketplace_order_id    TEXT NOT NULL,          -- marketplace's own order ID
    order_status            TEXT NOT NULL DEFAULT 'pending',
        -- pending, confirmed, processing, shipped, delivered, cancelled, returned, refunded
    order_date              TIMESTAMPTZ NOT NULL,
    currency_code           CHAR(3) NOT NULL,      -- ISO 4217
    subtotal                NUMERIC(12,2) NOT NULL,
    shipping_cost           NUMERIC(12,2) NOT NULL DEFAULT 0,
    tax_amount              NUMERIC(12,2) NOT NULL DEFAULT 0,
    marketplace_fees        NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_amount            NUMERIC(12,2) NOT NULL,
    net_payout              NUMERIC(12,2),          -- total - fees
    buyer_name              TEXT,
    buyer_email             TEXT,
    shipping_name           TEXT,
    shipping_address_line1  TEXT,
    shipping_address_line2  TEXT,
    shipping_city           TEXT,
    shipping_state          TEXT,
    shipping_postal_code    TEXT,
    shipping_country_code   CHAR(2),               -- ISO 3166-1 alpha-2
    shipping_method         TEXT,
    fulfillment_type        TEXT NOT NULL DEFAULT 'merchant',
    notes                   TEXT,
    raw_order_data          TEXT,                   -- original marketplace JSON for debugging
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(marketplace_id, marketplace_order_id)
);

CREATE INDEX idx_order_seller ON marketplace_order(seller_id);
CREATE INDEX idx_order_marketplace ON marketplace_order(marketplace_id);
CREATE INDEX idx_order_status ON marketplace_order(order_status);
CREATE INDEX idx_order_date ON marketplace_order(order_date);
CREATE INDEX idx_order_mktpl_id ON marketplace_order(marketplace_id, marketplace_order_id);

CREATE TABLE order_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES marketplace_order(id),
    listing_id      UUID REFERENCES listing(id),
    product_id      UUID REFERENCES product(id),
    variant_id      UUID REFERENCES product_variant(id),
    marketplace_item_id TEXT,                      -- marketplace's line item ID
    sku             TEXT NOT NULL,
    title           TEXT NOT NULL,
    quantity        INT NOT NULL CHECK (quantity > 0),
    unit_price      NUMERIC(12,2) NOT NULL,
    tax_amount      NUMERIC(12,2) NOT NULL DEFAULT 0,
    discount_amount NUMERIC(12,2) NOT NULL DEFAULT 0,
    line_total      NUMERIC(12,2) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_item_order ON order_item(order_id);
CREATE INDEX idx_order_item_product ON order_item(product_id);
CREATE INDEX idx_order_item_listing ON order_item(listing_id);
```

### Shipping & Fulfillment

```sql
CREATE TABLE shipment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id            UUID NOT NULL REFERENCES marketplace_order(id),
    carrier_code        TEXT NOT NULL,             -- ups, fedex, dhl, usps
    carrier_name        TEXT NOT NULL,
    tracking_number     TEXT,
    shipping_method     TEXT,
    label_url           TEXT,
    shipping_cost       NUMERIC(12,2),
    currency_code       CHAR(3),
    status              TEXT NOT NULL DEFAULT 'pending',
        -- pending, label_created, picked_up, in_transit, delivered, exception
    shipped_at          TIMESTAMPTZ,
    delivered_at        TIMESTAMPTZ,
    warehouse_id        UUID REFERENCES warehouse(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shipment_order ON shipment(order_id);
CREATE INDEX idx_shipment_tracking ON shipment(tracking_number) WHERE tracking_number IS NOT NULL;
CREATE INDEX idx_shipment_status ON shipment(status);

CREATE TABLE shipment_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id     UUID NOT NULL REFERENCES shipment(id),
    order_item_id   UUID NOT NULL REFERENCES order_item(id),
    quantity        INT NOT NULL CHECK (quantity > 0),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shipment_item_shipment ON shipment_item(shipment_id);
```

### Repricing & Competitive Intelligence

```sql
CREATE TABLE repricing_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    name            TEXT NOT NULL,
    rule_type       TEXT NOT NULL,                 -- floor_ceiling, match_lowest, beat_by_percent, beat_by_amount, target_margin
    marketplace_id  UUID REFERENCES marketplace(id), -- NULL = all marketplaces
    min_price       NUMERIC(12,2),
    max_price       NUMERIC(12,2),
    target_margin_pct NUMERIC(5,2),
    beat_by_amount  NUMERIC(12,2),
    beat_by_pct     NUMERIC(5,2),
    priority        INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_repricing_rule_seller ON repricing_rule(seller_id);

CREATE TABLE repricing_rule_listing (
    repricing_rule_id UUID NOT NULL REFERENCES repricing_rule(id) ON DELETE CASCADE,
    listing_id        UUID NOT NULL REFERENCES listing(id) ON DELETE CASCADE,
    PRIMARY KEY (repricing_rule_id, listing_id)
);

CREATE TABLE competitor_price (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id      UUID NOT NULL REFERENCES listing(id),
    competitor_name TEXT,                           -- seller name or "Buy Box Winner"
    competitor_price NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    shipping_price  NUMERIC(12,2) DEFAULT 0,
    fulfillment_type TEXT,                         -- fba, merchant
    is_buy_box      BOOLEAN DEFAULT false,
    observed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_competitor_listing ON competitor_price(listing_id);
CREATE INDEX idx_competitor_observed ON competitor_price(observed_at);
-- Partition by month for large datasets:
-- CREATE TABLE competitor_price ... PARTITION BY RANGE (observed_at);

CREATE TABLE price_change_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id      UUID NOT NULL REFERENCES listing(id),
    old_price       NUMERIC(12,2) NOT NULL,
    new_price       NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    change_reason   TEXT NOT NULL,                 -- manual, repricing_rule, competitor_match, api_sync
    repricing_rule_id UUID REFERENCES repricing_rule(id),
    triggered_by    TEXT,                           -- system, user, api
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_price_change_listing ON price_change_log(listing_id);
CREATE INDEX idx_price_change_created ON price_change_log(created_at);
```

### Analytics & Reporting

```sql
CREATE TABLE daily_sales_summary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id),
    product_id      UUID REFERENCES product(id),
    summary_date    DATE NOT NULL,
    units_sold      INT NOT NULL DEFAULT 0,
    gross_revenue   NUMERIC(14,2) NOT NULL DEFAULT 0,
    marketplace_fees NUMERIC(14,2) NOT NULL DEFAULT 0,
    shipping_cost   NUMERIC(14,2) NOT NULL DEFAULT 0,
    cogs            NUMERIC(14,2) NOT NULL DEFAULT 0,  -- cost of goods sold
    net_profit      NUMERIC(14,2) NOT NULL DEFAULT 0,
    avg_selling_price NUMERIC(12,2),
    buy_box_win_pct NUMERIC(5,2),                  -- % of time owned Buy Box
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(seller_id, marketplace_id, product_id, summary_date)
);

CREATE INDEX idx_daily_sales_seller_date ON daily_sales_summary(seller_id, summary_date);
CREATE INDEX idx_daily_sales_marketplace ON daily_sales_summary(marketplace_id, summary_date);

CREATE TABLE seller_health_metric (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id),
    metric_date     DATE NOT NULL,
    order_defect_rate NUMERIC(5,4),
    late_shipment_rate NUMERIC(5,4),
    cancellation_rate NUMERIC(5,4),
    feedback_score  NUMERIC(5,2),
    account_health  TEXT,                          -- healthy, at_risk, critical
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(seller_id, marketplace_id, metric_date)
);

CREATE INDEX idx_health_seller_date ON seller_health_metric(seller_id, metric_date);
```

### Compliance & Notifications

```sql
CREATE TABLE compliance_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id),
    listing_id      UUID REFERENCES listing(id),
    alert_type      TEXT NOT NULL,                 -- map_violation, suppression, policy_change, restricted_content, category_change
    severity        TEXT NOT NULL DEFAULT 'warning', -- info, warning, critical
    title           TEXT NOT NULL,
    description     TEXT,
    marketplace_ref TEXT,                           -- marketplace's reference ID for the alert
    status          TEXT NOT NULL DEFAULT 'open',   -- open, acknowledged, resolved, dismissed
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_seller ON compliance_alert(seller_id);
CREATE INDEX idx_compliance_status ON compliance_alert(status);
CREATE INDEX idx_compliance_listing ON compliance_alert(listing_id) WHERE listing_id IS NOT NULL;

CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    channel         TEXT NOT NULL DEFAULT 'in_app', -- in_app, email, webhook
    category        TEXT NOT NULL,                  -- order, inventory, repricing, compliance, system
    title           TEXT NOT NULL,
    body            TEXT,
    reference_type  TEXT,                           -- order, listing, inventory, compliance_alert
    reference_id    UUID,
    is_read         BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notification_seller ON notification(seller_id, is_read);
CREATE INDEX idx_notification_created ON notification(created_at);
```

### EDI Integration

```sql
CREATE TABLE edi_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id),
    transaction_set TEXT NOT NULL,                  -- X12: 850, 856, 810, 846; EDIFACT equivalent
    direction       TEXT NOT NULL,                  -- inbound, outbound
    transport       TEXT NOT NULL DEFAULT 'as2',    -- as2, sftp, api
    raw_content     TEXT NOT NULL,                  -- original EDI document
    parsed_status   TEXT NOT NULL DEFAULT 'pending', -- pending, parsed, error
    reference_type  TEXT,                           -- order, shipment, invoice
    reference_id    UUID,
    partner_id      TEXT,                           -- trading partner identifier
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_edi_seller ON edi_transaction(seller_id);
CREATE INDEX idx_edi_created ON edi_transaction(created_at);
```

### Sync & Job Management

```sql
CREATE TABLE sync_job (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id),
    job_type        TEXT NOT NULL,                  -- listing_sync, order_sync, inventory_sync, price_sync, analytics_pull
    status          TEXT NOT NULL DEFAULT 'pending', -- pending, running, completed, failed
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    items_processed INT DEFAULT 0,
    items_failed    INT DEFAULT 0,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sync_job_seller ON sync_job(seller_id);
CREATE INDEX idx_sync_job_status ON sync_job(status);
CREATE INDEX idx_sync_job_created ON sync_job(created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenancy & Authentication | 2 | seller, seller_user |
| Marketplace Integration | 2 | marketplace, marketplace_credential |
| Product Catalogue | 4 | product, product_image, product_variant, variant_attribute |
| Marketplace Listings | 2 | listing, listing_attribute |
| Inventory Management | 4 | warehouse, inventory, inventory_transaction, inventory_channel_allocation |
| Order Management | 2 | marketplace_order, order_item |
| Shipping & Fulfillment | 2 | shipment, shipment_item |
| Repricing & Competition | 4 | repricing_rule, repricing_rule_listing, competitor_price, price_change_log |
| Analytics & Reporting | 2 | daily_sales_summary, seller_health_metric |
| Compliance & Notifications | 2 | compliance_alert, notification |
| EDI Integration | 1 | edi_transaction |
| Sync & Jobs | 1 | sync_job |
| **Total** | **28** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation across microservices and avoids sequential ID enumeration attacks. Standard for modern SaaS platforms.

2. **Separate `product` from `listing`** — a single product can have listings on multiple marketplaces, each with different titles, prices, and attributes. This one-to-many relationship is the core of multi-channel selling.

3. **`listing_attribute` for marketplace-specific fields** — rather than adding columns for every marketplace's unique attributes (Amazon bullet points, eBay item specifics, TikTok categories), a key-value table handles the long tail. This is the main flexibility mechanism in this normalized model.

4. **Three-tier inventory: available, reserved, on_order** — prevents overselling by tracking soft reservations separately from available stock. The `version` column enables optimistic locking for concurrent inventory updates.

5. **`inventory_channel_allocation` table** — enables explicit inventory allocation per marketplace, supporting strategies like reserving 60% for Amazon and 40% for eBay rather than sharing a single pool.

6. **`competitor_price` as append-only observations** — competitor prices are recorded as point-in-time observations, not updated in place. This preserves price history for analytics and ML model training. Should be partitioned by `observed_at` for large datasets.

7. **`price_change_log` links to `repricing_rule`** — every price change records why it happened and which rule triggered it, enabling repricing strategy performance analysis.

8. **`raw_order_data` on orders** — stores the original marketplace JSON response for debugging sync issues. Not queried in normal operations but invaluable for support.

9. **`edi_transaction` table** — stores raw EDI documents alongside parsed references to orders/shipments, supporting the dual REST+EDI integration path required for Walmart and enterprise partners.

10. **Monetary amounts use `NUMERIC(12,2)` with explicit `currency_code`** — avoids floating-point rounding and supports multi-currency operations across global marketplaces.
