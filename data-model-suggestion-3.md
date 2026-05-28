# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Marketplace Seller Management · Created: 2026-05-19

## Philosophy

This model uses standard relational tables for core operational data (sellers, orders, inventory, shipments) but leverages PostgreSQL JSONB columns for marketplace-specific, variable, and rapidly evolving attributes. The key insight is that marketplace seller management has a stable core (every marketplace has orders, listings, and inventory) surrounded by a wildly variable periphery (Amazon has 500+ product type definitions with different required attributes; eBay has item specifics; TikTok Shop has creator/affiliate metadata; Walmart has WFS-specific fields).

Rather than the EAV (Entity-Attribute-Value) anti-pattern — which creates join-heavy queries and poor performance — JSONB columns with GIN indexes provide the flexibility of a document store with the ACID guarantees and JOIN capability of PostgreSQL. Core fields that every marketplace shares (price, quantity, status, SKU) remain as typed relational columns for efficient indexing and aggregation. Marketplace-specific fields (Amazon bullet points, eBay item specifics, TikTok creator commission rates) live in JSONB columns with marketplace-aware JSON Schema validation at the application layer.

This approach is how Shopify, ChannelEngine, and modern PIM (Product Information Management) systems handle the attribute explosion problem. It is the fastest path to supporting a new marketplace without schema migrations: add a new JSON Schema definition, and the JSONB column handles the rest.

**Best for:** Teams that need to rapidly integrate new marketplaces, handle variable product attributes across categories, and want MVP speed without sacrificing the ability to run complex cross-marketplace analytics later.

**Trade-offs:**
- Pro: No DDL changes required when adding new marketplace integrations
- Pro: Handles the "attribute explosion" problem elegantly (Amazon alone has 500+ product types)
- Pro: Core relational columns provide fast aggregation and indexing for operational queries
- Pro: JSONB + GIN indexes support efficient containment queries on variable attributes
- Pro: Lower table count than fully normalized model; simpler migrations
- Pro: Natural fit for storing raw marketplace API responses alongside parsed fields
- Con: JSONB columns lack database-level type enforcement (validated at app layer)
- Con: JSONB queries can be slower than indexed relational columns for complex filtering
- Con: Risk of schema drift if JSON Schema validation is not consistently enforced
- Con: Reporting on JSONB fields requires more complex SQL (->>, @>, jsonb_path_query)
- Con: ORM support for JSONB varies; some frameworks handle it poorly

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | Relational `gtin` column on `product` table; also embedded in `marketplace_attributes` JSONB for marketplace-specific GTIN fields |
| GS1 Global Data Model | Attribute layering (Global Core = relational columns, Category-specific = JSONB) mirrors GS1 GDM's tiered attribute model |
| Amazon Product Type Definitions | Amazon's JSON Schema-based product type definitions stored in `marketplace_product_type.schema_definition` JSONB; validated against listing `marketplace_attributes` |
| ISO 4217 | Relational `currency_code` columns on monetary fields |
| ISO 3166-1 alpha-2 | Relational `country_code` columns; also in shipping address JSONB |
| JSON Schema 2020-12 | JSONB columns validated against JSON Schema definitions per marketplace and product type |
| Schema.org Product/Offer | Core listing fields (title, price, availability, condition) align with Schema.org vocabulary |
| OpenAPI 3.1 | Platform API documented with OAS 3.1; JSONB-backed fields documented as `additionalProperties` in OpenAPI schemas |
| RFC 7807 | Error responses for JSONB validation failures use Problem Details format |

---

## Core Tables

### Seller & Marketplace

```sql
CREATE TABLE seller (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_name    TEXT NOT NULL,
    contact_email   TEXT NOT NULL,
    contact_phone   TEXT,
    country_code    CHAR(2) NOT NULL,              -- ISO 3166-1
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    subscription_tier TEXT NOT NULL DEFAULT 'free',
    status          TEXT NOT NULL DEFAULT 'active',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_currency": "USD",
    --   "notification_preferences": {"email": true, "in_app": true},
    --   "repricing_defaults": {"min_margin_pct": 15, "max_reprice_frequency_minutes": 5},
    --   "gdpr_consent": {"data_processing": true, "marketing": false, "consent_date": "2026-05-19"}
    -- }
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
    api_config      JSONB NOT NULL DEFAULT '{}',
    -- api_config example:
    -- {
    --   "base_url": "https://sellingpartnerapi-na.amazon.com",
    --   "auth_url": "https://api.amazon.com/auth/o2/token",
    --   "rate_limits": {
    --     "listings": {"requests_per_second": 5, "burst": 10},
    --     "orders": {"requests_per_second": 0.0167, "burst": 20},
    --     "pricing": {"requests_per_second": 10, "burst": 20}
    --   },
    --   "webhook_events": ["ANY_OFFER_CHANGED", "ORDER_STATUS_CHANGE"],
    --   "supported_fulfillment": ["merchant", "fba"]
    -- }
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE marketplace_credential (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id),
    auth_type       TEXT NOT NULL,
    credentials     JSONB NOT NULL DEFAULT '{}',    -- encrypted at rest (pgcrypto or app-level)
    -- credentials example (Amazon SP-API):
    -- {
    --   "client_id": "amzn1.application-oa2-client.xxx",
    --   "client_secret": "encrypted:...",
    --   "refresh_token": "encrypted:...",
    --   "access_token": "encrypted:...",
    --   "token_expires_at": "2026-05-19T15:30:00Z",
    --   "aws_access_key": "encrypted:...",
    --   "aws_secret_key": "encrypted:...",
    --   "marketplace_ids": ["ATVPDKIKX0DER"]
    -- }
    status          TEXT NOT NULL DEFAULT 'active',
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(seller_id, marketplace_id)
);

CREATE INDEX idx_cred_seller ON marketplace_credential(seller_id);
```

### Product Catalogue with JSONB Attributes

```sql
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    sku             TEXT NOT NULL,
    title           TEXT NOT NULL,
    description     TEXT,
    brand           TEXT,
    gtin            TEXT,                          -- GS1 GTIN
    mpn             TEXT,
    category        TEXT,
    weight_kg       NUMERIC(10,3),
    dimensions      JSONB,
    -- dimensions example:
    -- {"length_cm": 30.5, "width_cm": 20.0, "height_cm": 10.0, "package_weight_kg": 1.2}
    country_of_origin CHAR(2),
    hs_code         TEXT,
    images          JSONB NOT NULL DEFAULT '[]',
    -- images example:
    -- [
    --   {"url": "https://cdn.example.com/img1.jpg", "type": "main", "sort_order": 0, "alt_text": "Front view"},
    --   {"url": "https://cdn.example.com/img2.jpg", "type": "lifestyle", "sort_order": 1}
    -- ]
    variants        JSONB NOT NULL DEFAULT '[]',
    -- variants example:
    -- [
    --   {"sku": "WIDGET-001-RED-L", "name": "Red / Large", "gtin": "012345678901",
    --    "attributes": {"color": "Red", "size": "Large"}, "weight_kg": 0.5},
    --   {"sku": "WIDGET-001-BLU-M", "name": "Blue / Medium", "gtin": "012345678902",
    --    "attributes": {"color": "Blue", "size": "Medium"}, "weight_kg": 0.45}
    -- ]
    custom_attributes JSONB NOT NULL DEFAULT '{}',
    -- custom_attributes example:
    -- {
    --   "material": "Cotton blend",
    --   "care_instructions": "Machine wash cold",
    --   "seasonal": true,
    --   "target_demographic": "adults"
    -- }
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(seller_id, sku)
);

CREATE INDEX idx_product_seller ON product(seller_id);
CREATE INDEX idx_product_gtin ON product(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_brand ON product(brand) WHERE brand IS NOT NULL;
CREATE INDEX idx_product_variants ON product USING GIN (variants jsonb_path_ops);
CREATE INDEX idx_product_custom_attrs ON product USING GIN (custom_attributes);
```

### Marketplace Product Type Definitions

```sql
CREATE TABLE marketplace_product_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id),
    product_type    TEXT NOT NULL,                  -- Amazon product type code, eBay category ID
    display_name    TEXT NOT NULL,
    schema_definition JSONB NOT NULL,              -- JSON Schema defining required/optional attributes
    -- schema_definition example (simplified Amazon product type):
    -- {
    --   "$schema": "https://json-schema.org/draft/2020-12/schema",
    --   "type": "object",
    --   "required": ["item_name", "brand", "bullet_point", "product_description"],
    --   "properties": {
    --     "item_name": {"type": "string", "maxLength": 200},
    --     "brand": {"type": "string", "maxLength": 50},
    --     "bullet_point": {"type": "array", "items": {"type": "string", "maxLength": 500}, "maxItems": 5},
    --     "product_description": {"type": "string", "maxLength": 2000},
    --     "color": {"type": "string"},
    --     "size": {"type": "string"},
    --     "material_type": {"type": "string"},
    --     "target_gender": {"type": "string", "enum": ["male", "female", "unisex"]}
    --   }
    -- }
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(marketplace_id, product_type)
);

CREATE INDEX idx_mpt_marketplace ON marketplace_product_type(marketplace_id);
```

### Listings with Marketplace-Specific JSONB

```sql
CREATE TABLE listing (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id               UUID NOT NULL REFERENCES seller(id),
    product_id              UUID NOT NULL REFERENCES product(id),
    marketplace_id          UUID NOT NULL REFERENCES marketplace(id),
    marketplace_listing_id  TEXT,                   -- ASIN, eBay item ID, etc.
    marketplace_sku         TEXT,
    product_type_id         UUID REFERENCES marketplace_product_type(id),

    -- Core relational fields (shared across all marketplaces)
    title                   TEXT NOT NULL,
    price                   NUMERIC(12,2) NOT NULL,
    currency_code           CHAR(3) NOT NULL,
    quantity_available      INT NOT NULL DEFAULT 0,
    condition               TEXT NOT NULL DEFAULT 'new',
    fulfillment_type        TEXT NOT NULL DEFAULT 'merchant',
    status                  TEXT NOT NULL DEFAULT 'draft',
    buy_box_owned           BOOLEAN DEFAULT false,

    -- JSONB: marketplace-specific listing attributes
    marketplace_attributes  JSONB NOT NULL DEFAULT '{}',
    -- Amazon example:
    -- {
    --   "bullet_point": ["Durable cotton blend", "Machine washable", "Available in 5 colors"],
    --   "search_terms": "widget gadget tool accessory",
    --   "fulfillment_channel": "AMAZON_NA",
    --   "item_condition_note": null,
    --   "product_tax_code": "A_GEN_TAX",
    --   "max_order_quantity": 10,
    --   "a_plus_content_id": "B0XXXXX"
    -- }
    --
    -- eBay example:
    -- {
    --   "item_specifics": {"Brand": "Acme", "Color": "Red", "Size": "Large"},
    --   "listing_format": "FixedPrice",
    --   "listing_duration": "GTC",
    --   "return_policy_id": "rp-001",
    --   "payment_policy_id": "pp-001",
    --   "shipping_policy_id": "sp-001",
    --   "condition_id": 1000,
    --   "subtitle": "Premium quality widget"
    -- }
    --
    -- TikTok Shop example:
    -- {
    --   "category_id": "601234",
    --   "package_dimensions": {"length": 30, "width": 20, "height": 10, "unit": "cm"},
    --   "package_weight": {"value": 500, "unit": "g"},
    --   "is_cod_allowed": false,
    --   "affiliate_commission_rate": 10.0,
    --   "creator_marketplace_enabled": true,
    --   "video_ids": ["v001", "v002"]
    -- }

    -- Sync state
    sync_status             TEXT NOT NULL DEFAULT 'pending',
    sync_error              JSONB,
    -- sync_error example:
    -- {"code": "INVALID_ATTRIBUTE", "message": "bullet_point exceeds 500 chars", "timestamp": "2026-05-19T10:00:00Z"}
    last_synced_at          TIMESTAMPTZ,

    -- AI-generated content
    ai_content              JSONB,
    -- ai_content example:
    -- {
    --   "generated_title": "Premium Cotton Widget - Red, Large - Machine Washable",
    --   "generated_bullets": ["...", "...", "..."],
    --   "generated_description": "...",
    --   "seo_keywords": ["widget", "cotton", "premium"],
    --   "generation_model": "claude-opus-4-20250514",
    --   "generated_at": "2026-05-19T09:00:00Z",
    --   "approval_status": "pending"
    -- }

    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(seller_id, marketplace_id, marketplace_sku)
);

CREATE INDEX idx_listing_seller ON listing(seller_id);
CREATE INDEX idx_listing_product ON listing(product_id);
CREATE INDEX idx_listing_marketplace ON listing(marketplace_id);
CREATE INDEX idx_listing_status ON listing(status);
CREATE INDEX idx_listing_buybox ON listing(buy_box_owned) WHERE buy_box_owned = true;
CREATE INDEX idx_listing_sync ON listing(sync_status) WHERE sync_status != 'synced';
CREATE INDEX idx_listing_mktpl_attrs ON listing USING GIN (marketplace_attributes jsonb_path_ops);
CREATE INDEX idx_listing_mktpl_id ON listing(marketplace_listing_id) WHERE marketplace_listing_id IS NOT NULL;
```

### Inventory

```sql
CREATE TABLE warehouse (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    name            TEXT NOT NULL,
    warehouse_type  TEXT NOT NULL DEFAULT 'own',
    address         JSONB NOT NULL DEFAULT '{}',
    -- address example:
    -- {
    --   "line1": "123 Warehouse Ave",
    --   "line2": "Unit B",
    --   "city": "Dallas",
    --   "state": "TX",
    --   "postal_code": "75001",
    --   "country_code": "US"
    -- }
    is_default      BOOLEAN DEFAULT false,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_warehouse_seller ON warehouse(seller_id);

CREATE TABLE inventory (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id           UUID NOT NULL REFERENCES seller(id),
    product_id          UUID NOT NULL REFERENCES product(id),
    variant_sku         TEXT,                       -- NULL for non-variant products
    warehouse_id        UUID NOT NULL REFERENCES warehouse(id),
    quantity_available  INT NOT NULL DEFAULT 0 CHECK (quantity_available >= 0),
    quantity_reserved   INT NOT NULL DEFAULT 0 CHECK (quantity_reserved >= 0),
    quantity_on_order   INT NOT NULL DEFAULT 0,
    cost_per_unit       NUMERIC(12,4),
    currency_code       CHAR(3),
    reorder_point       INT,
    reorder_quantity    INT,
    channel_allocations JSONB NOT NULL DEFAULT '{}',
    -- channel_allocations example:
    -- {
    --   "amazon_us": {"allocated_qty": 60, "sync_status": "synced", "last_synced_at": "2026-05-19T10:00:00Z"},
    --   "ebay_us": {"allocated_qty": 30, "sync_status": "synced", "last_synced_at": "2026-05-19T10:01:00Z"},
    --   "walmart_us": {"allocated_qty": 10, "sync_status": "pending", "last_synced_at": null}
    -- }
    version             INT NOT NULL DEFAULT 1,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(product_id, variant_sku, warehouse_id)
);

CREATE INDEX idx_inventory_seller ON inventory(seller_id);
CREATE INDEX idx_inventory_product ON inventory(product_id);
CREATE INDEX idx_inventory_warehouse ON inventory(warehouse_id);
CREATE INDEX idx_inventory_low ON inventory(seller_id) WHERE quantity_available <= 10;
CREATE INDEX idx_inventory_allocations ON inventory USING GIN (channel_allocations);

CREATE TABLE inventory_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inventory_id    UUID NOT NULL REFERENCES inventory(id),
    transaction_type TEXT NOT NULL,
    quantity_change INT NOT NULL,
    quantity_before INT NOT NULL,
    quantity_after  INT NOT NULL,
    reference_type  TEXT,
    reference_id    UUID,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {"reason": "customer_return", "marketplace_order_id": "111-2222-3333", "condition": "like_new"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inv_txn_inventory ON inventory_transaction(inventory_id);
CREATE INDEX idx_inv_txn_created ON inventory_transaction(created_at);
```

### Orders with Marketplace-Specific JSONB

```sql
CREATE TABLE marketplace_order (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id               UUID NOT NULL REFERENCES seller(id),
    marketplace_id          UUID NOT NULL REFERENCES marketplace(id),
    marketplace_order_id    TEXT NOT NULL,
    order_status            TEXT NOT NULL DEFAULT 'pending',

    -- Core relational fields
    order_date              TIMESTAMPTZ NOT NULL,
    currency_code           CHAR(3) NOT NULL,
    subtotal                NUMERIC(12,2) NOT NULL,
    shipping_cost           NUMERIC(12,2) NOT NULL DEFAULT 0,
    tax_amount              NUMERIC(12,2) NOT NULL DEFAULT 0,
    marketplace_fees        NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_amount            NUMERIC(12,2) NOT NULL,
    fulfillment_type        TEXT NOT NULL DEFAULT 'merchant',
    item_count              INT NOT NULL DEFAULT 1,

    -- Shipping address as JSONB (varies by marketplace)
    shipping_address        JSONB,
    -- shipping_address example:
    -- {
    --   "name": "Jane Doe",
    --   "line1": "123 Main St",
    --   "line2": "Apt 4B",
    --   "city": "Portland",
    --   "state": "OR",
    --   "postal_code": "97201",
    --   "country_code": "US",
    --   "phone": "+15035551234"
    -- }

    -- Items stored as JSONB array (denormalized for read performance)
    items                   JSONB NOT NULL DEFAULT '[]',
    -- items example:
    -- [
    --   {
    --     "marketplace_item_id": "li-001",
    --     "sku": "WIDGET-001-RED-L",
    --     "title": "Premium Red Widget - Large",
    --     "quantity": 2,
    --     "unit_price": 27.49,
    --     "tax": 4.85,
    --     "discount": 0,
    --     "line_total": 54.98,
    --     "listing_id": "uuid",
    --     "product_id": "uuid"
    --   }
    -- ]

    -- Marketplace-specific order fields
    marketplace_data        JSONB NOT NULL DEFAULT '{}',
    -- Amazon example:
    -- {
    --   "amazon_order_id": "111-2222222-3333333",
    --   "purchase_date": "2026-05-18T14:30:00Z",
    --   "order_type": "StandardOrder",
    --   "fulfillment_channel": "MFN",
    --   "sales_channel": "Amazon.com",
    --   "is_premium_order": false,
    --   "is_business_order": false,
    --   "earliest_ship_date": "2026-05-19T07:00:00Z",
    --   "latest_ship_date": "2026-05-21T07:00:00Z"
    -- }
    --
    -- TikTok Shop example:
    -- {
    --   "tiktok_order_id": "TTS-123456",
    --   "payment_method": "ONLINE",
    --   "creator_username": "@influencer",
    --   "affiliate_commission": 5.50,
    --   "video_id": "v-789",
    --   "is_sample_order": false,
    --   "cancel_reason": null
    -- }

    -- Raw API response for debugging
    raw_api_response        JSONB,

    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(marketplace_id, marketplace_order_id)
);

CREATE INDEX idx_order_seller ON marketplace_order(seller_id);
CREATE INDEX idx_order_marketplace ON marketplace_order(marketplace_id);
CREATE INDEX idx_order_status ON marketplace_order(order_status);
CREATE INDEX idx_order_date ON marketplace_order(order_date);
CREATE INDEX idx_order_items ON marketplace_order USING GIN (items jsonb_path_ops);
CREATE INDEX idx_order_mktpl_data ON marketplace_order USING GIN (marketplace_data jsonb_path_ops);
```

### Shipping & Fulfillment

```sql
CREATE TABLE shipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES marketplace_order(id),
    carrier_code    TEXT NOT NULL,
    tracking_number TEXT,
    shipping_method TEXT,
    status          TEXT NOT NULL DEFAULT 'pending',
    shipping_cost   NUMERIC(12,2),
    currency_code   CHAR(3),
    warehouse_id    UUID REFERENCES warehouse(id),
    label_data      JSONB,
    -- label_data example:
    -- {
    --   "label_url": "https://labels.example.com/ship-001.pdf",
    --   "label_format": "PDF",
    --   "service_type": "GROUND",
    --   "estimated_delivery": "2026-05-23",
    --   "insurance_amount": 100.00,
    --   "dimensions": {"length": 12, "width": 8, "height": 6, "unit": "in"},
    --   "weight": {"value": 2.5, "unit": "lb"}
    -- }
    items           JSONB NOT NULL DEFAULT '[]',
    -- items example:
    -- [{"sku": "WIDGET-001-RED-L", "quantity": 2}]
    shipped_at      TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shipment_order ON shipment(order_id);
CREATE INDEX idx_shipment_tracking ON shipment(tracking_number) WHERE tracking_number IS NOT NULL;
CREATE INDEX idx_shipment_status ON shipment(status);
```

### Repricing & Competitive Intelligence

```sql
CREATE TABLE repricing_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    name            TEXT NOT NULL,
    marketplace_id  UUID REFERENCES marketplace(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    priority        INT NOT NULL DEFAULT 0,
    config          JSONB NOT NULL,
    -- config example (floor/ceiling with competitor matching):
    -- {
    --   "strategy": "beat_lowest_by_percent",
    --   "beat_by_pct": 2.0,
    --   "min_price": 15.00,
    --   "max_price": 50.00,
    --   "min_margin_pct": 10.0,
    --   "only_match_fba": true,
    --   "exclude_sellers": ["SELLER-XYZ"],
    --   "schedule": {"active_hours": "09:00-21:00", "timezone": "America/New_York"},
    --   "max_changes_per_day": 24,
    --   "cooldown_minutes": 15
    -- }
    listing_ids     UUID[] NOT NULL DEFAULT '{}',  -- array of listing IDs this rule applies to
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_repricing_seller ON repricing_rule(seller_id);
CREATE INDEX idx_repricing_active ON repricing_rule(is_active) WHERE is_active = true;
CREATE INDEX idx_repricing_listings ON repricing_rule USING GIN (listing_ids);

CREATE TABLE competitor_observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id      UUID NOT NULL REFERENCES listing(id),
    observed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    observations    JSONB NOT NULL,
    -- observations example:
    -- {
    --   "buy_box_winner": {
    --     "seller_name": "CompetitorA",
    --     "price": 27.99,
    --     "shipping": 0.00,
    --     "fulfillment": "fba",
    --     "is_amazon": false
    --   },
    --   "other_sellers": [
    --     {"seller_name": "CompetitorB", "price": 28.50, "shipping": 3.99, "fulfillment": "merchant"},
    --     {"seller_name": "CompetitorC", "price": 29.99, "shipping": 0.00, "fulfillment": "fba"}
    --   ],
    --   "total_offers": 5,
    --   "source": "sp_api_notification"
    -- }
    buy_box_price   NUMERIC(12,2),                 -- denormalized for fast queries
    our_price       NUMERIC(12,2),
    we_own_buybox   BOOLEAN DEFAULT false
) PARTITION BY RANGE (observed_at);

-- Create monthly partitions
CREATE TABLE competitor_observation_2026_05 PARTITION OF competitor_observation
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE competitor_observation_2026_06 PARTITION OF competitor_observation
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_comp_obs_listing ON competitor_observation(listing_id, observed_at);

CREATE TABLE price_change_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id      UUID NOT NULL REFERENCES listing(id),
    old_price       NUMERIC(12,2) NOT NULL,
    new_price       NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    change_reason   TEXT NOT NULL,
    rule_id         UUID REFERENCES repricing_rule(id),
    context         JSONB NOT NULL DEFAULT '{}',
    -- context example:
    -- {
    --   "competitor_price": 27.99,
    --   "buy_box_before": false,
    --   "buy_box_after": true,
    --   "margin_pct": 22.5,
    --   "cost_per_unit": 18.50,
    --   "model_confidence": 0.87
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE price_change_log_2026_05 PARTITION OF price_change_log
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_price_log_listing ON price_change_log(listing_id, created_at);
CREATE INDEX idx_price_log_reason ON price_change_log(change_reason);
```

### Analytics & Compliance

```sql
CREATE TABLE daily_summary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id),
    summary_date    DATE NOT NULL,
    metrics         JSONB NOT NULL,
    -- metrics example:
    -- {
    --   "orders": 45,
    --   "units_sold": 128,
    --   "gross_revenue": 3420.50,
    --   "marketplace_fees": 513.08,
    --   "shipping_costs": 285.00,
    --   "cogs": 1540.00,
    --   "net_profit": 1082.42,
    --   "avg_selling_price": 26.72,
    --   "buy_box_win_pct": 78.5,
    --   "reprice_count": 156,
    --   "top_products": [
    --     {"sku": "WIDGET-001", "units": 42, "revenue": 1154.58},
    --     {"sku": "GADGET-002", "units": 31, "revenue": 868.00}
    --   ],
    --   "seller_health": {
    --     "order_defect_rate": 0.002,
    --     "late_shipment_rate": 0.01,
    --     "cancellation_rate": 0.005,
    --     "feedback_score": 4.8,
    --     "account_health": "healthy"
    --   }
    -- }
    UNIQUE(seller_id, marketplace_id, summary_date)
);

CREATE INDEX idx_summary_seller_date ON daily_summary(seller_id, summary_date);

CREATE TABLE compliance_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id),
    listing_id      UUID REFERENCES listing(id),
    alert_type      TEXT NOT NULL,
    severity        TEXT NOT NULL DEFAULT 'warning',
    status          TEXT NOT NULL DEFAULT 'open',
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "policy_name": "Restricted Products",
    --   "violation_description": "Listing contains restricted keyword 'miracle'",
    --   "marketplace_reference": "ASIN-B0XXXXX",
    --   "recommended_action": "Remove restricted keyword from title",
    --   "deadline": "2026-05-25T00:00:00Z",
    --   "previous_violations": 0
    -- }
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_seller ON compliance_alert(seller_id, status);
CREATE INDEX idx_alert_listing ON compliance_alert(listing_id) WHERE listing_id IS NOT NULL;
```

### Sync & Background Jobs

```sql
CREATE TABLE sync_job (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id       UUID NOT NULL REFERENCES seller(id),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id),
    job_type        TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    progress        JSONB NOT NULL DEFAULT '{}',
    -- progress example:
    -- {
    --   "total_items": 500,
    --   "processed": 342,
    --   "succeeded": 338,
    --   "failed": 4,
    --   "failed_items": [
    --     {"sku": "WIDGET-ERR", "error": "GTIN mismatch", "marketplace_error_code": "INVALID_GTIN"}
    --   ]
    -- }
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sync_seller ON sync_job(seller_id);
CREATE INDEX idx_sync_status ON sync_job(status) WHERE status IN ('pending', 'running');
```

---

## Example JSONB Queries

```sql
-- Find all Amazon listings with bullet points containing "waterproof"
SELECT id, title, marketplace_attributes->'bullet_point' AS bullets
FROM listing
WHERE marketplace_id = $amazon_id
  AND marketplace_attributes @> '{"bullet_point": ["waterproof"]}';

-- Better: use jsonb_path_query for flexible text search in arrays
SELECT id, title
FROM listing
WHERE marketplace_id = $amazon_id
  AND marketplace_attributes @? '$.bullet_point[*] ? (@ like_regex "waterproof" flag "i")';

-- Find all TikTok Shop listings with affiliate commission > 8%
SELECT id, title,
       (marketplace_attributes->>'affiliate_commission_rate')::numeric AS commission
FROM listing
WHERE marketplace_id = $tiktok_id
  AND (marketplace_attributes->>'affiliate_commission_rate')::numeric > 8.0;

-- Cross-marketplace revenue by product
SELECT p.sku, p.title,
       m.code AS marketplace,
       (ds.metrics->>'gross_revenue')::numeric AS revenue,
       (ds.metrics->>'units_sold')::int AS units
FROM daily_summary ds
JOIN marketplace m ON m.id = ds.marketplace_id
JOIN listing l ON l.marketplace_id = ds.marketplace_id AND l.seller_id = ds.seller_id
JOIN product p ON p.id = l.product_id
WHERE ds.seller_id = $seller_id
  AND ds.summary_date = '2026-05-18';

-- Find inventory with pending channel allocations
SELECT i.*, p.sku, p.title
FROM inventory i
JOIN product p ON p.id = i.product_id
WHERE i.seller_id = $seller_id
  AND i.channel_allocations @> '{"amazon_us": {"sync_status": "pending"}}';

-- Order items: extract from JSONB array
SELECT
    mo.marketplace_order_id,
    item->>'sku' AS sku,
    (item->>'quantity')::int AS qty,
    (item->>'unit_price')::numeric AS price
FROM marketplace_order mo,
     jsonb_array_elements(mo.items) AS item
WHERE mo.seller_id = $seller_id
  AND mo.order_date >= '2026-05-01';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Seller & Marketplace | 3 | seller, marketplace, marketplace_credential |
| Product Catalogue | 2 | product, marketplace_product_type |
| Listings | 1 | listing (with JSONB for marketplace-specific attributes) |
| Inventory | 3 | warehouse, inventory, inventory_transaction |
| Orders | 1 | marketplace_order (items denormalized in JSONB) |
| Shipping | 1 | shipment |
| Repricing | 3 | repricing_rule, competitor_observation (partitioned), price_change_log (partitioned) |
| Analytics & Compliance | 2 | daily_summary, compliance_alert |
| Sync | 1 | sync_job |
| **Total** | **17** | Significantly fewer than normalized model |

---

## Key Design Decisions

1. **JSONB for marketplace-specific listing attributes** — the single most impactful decision. Amazon alone has 500+ product type definitions with different required fields. Rather than creating columns or EAV rows for each, the `marketplace_attributes` JSONB column holds whatever the marketplace requires. JSON Schema definitions in `marketplace_product_type` provide validation at the application layer.

2. **Order items as JSONB array, not a separate table** — orders rarely need item-level JOINs in this domain (orders come pre-assembled from marketplace APIs). Storing items as a JSONB array eliminates the `order_item` table and the N+1 queries it creates. For the rare case of item-level analytics, `jsonb_array_elements()` provides row-level access.

3. **Channel allocations as JSONB on inventory** — rather than a separate `inventory_channel_allocation` junction table, marketplace allocations are stored as a JSONB object keyed by marketplace code. This reduces a three-table JOIN to a single row read for the most common operation: "how much stock is allocated to Amazon?"

4. **Product variants as JSONB array** — for sellers with simple variant structures (color/size), embedding variants in the product JSONB avoids separate tables. Sellers with complex variant hierarchies can still use the `variant_sku` field on inventory for warehouse-level tracking.

5. **Competitor observations partitioned by month** — competitor price data grows fastest of any table. Monthly partitions on `observed_at` keep queries fast and enable dropping old partitions to cold storage after ML model training.

6. **`marketplace_product_type` stores JSON Schema definitions** — when adding a new marketplace or product type, the team creates a JSON Schema definition rather than DDL migrations. The application validates `listing.marketplace_attributes` against the schema before syncing to the marketplace.

7. **`raw_api_response` on orders as JSONB** — storing the original marketplace API response as JSONB (not TEXT) enables querying raw fields for debugging without parsing. This has saved countless support hours at scale.

8. **Repricing rule config as JSONB** — different repricing strategies need different parameters (floor/ceiling rules need min/max prices; competitor-matching rules need beat-by percentages; margin-targeting rules need cost data). A JSONB `config` column with a discriminator `strategy` field avoids a separate table per strategy type.

9. **GIN indexes on all JSONB columns used in WHERE clauses** — the `jsonb_path_ops` GIN index class is used for containment queries (`@>`, `@?`), providing sub-millisecond performance on JSONB filtering. Regular GIN is used for `?` (key existence) queries.

10. **Hybrid approach for monetary fields** — all monetary values (price, subtotal, fees) remain as typed `NUMERIC` relational columns despite other fields being JSONB. This ensures aggregation queries (SUM, AVG) run at full relational speed without JSONB casting overhead.
