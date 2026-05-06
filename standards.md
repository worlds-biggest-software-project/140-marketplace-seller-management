# Standards & API Reference

> Project: Marketplace Seller Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 20022 — Financial Messaging Standard**
- URL: https://www.iso20022.org/iso-20022
- Relevance: Defines the universal data model for financial messages including payment initiation, order settlement, and cash reporting. Marketplace seller platforms that handle cross-border payments, invoicing, or financial reconciliation between seller and marketplace operator must align with ISO 20022 as major payment rails (SWIFT, FedNow, SEPA) migrate to this standard. Supports both XML and JSON syntaxes.

**GS1 GTIN / EAN / UPC — Global Trade Item Number Standards**
- URL: https://www.gs1.org/standards/get-barcodes ; https://www.gs1.org/services/verified-by-gs1
- Relevance: GTINs (GTIN-12/UPC, GTIN-13/EAN, GTIN-14) are mandatory product identifiers required for listing on Amazon, Walmart, eBay, and most major global marketplaces. Correct GTIN assignment determines which product detail page a seller's offer appears on. Marketplace seller management platforms must validate and map GTINs across catalogues. GS1's "Verified by GS1" lookup service provides a canonical check against the global registry.

**GS1 EDI / GS1 XML — Supply Chain Data Standards**
- URL: https://www.gs1.org/standards/edi
- Relevance: GS1 XML and EDI message standards (orders, advanced ship notices, invoices) underpin supply chain interoperability with large retail partners, drop-ship programmes, and Walmart/Target supplier requirements. Seller platforms targeting enterprise or wholesale/drop-ship channels must support these formats.

### W3C & IETF Standards

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Relevance: The foundation of third-party API authentication across all major marketplace platforms (Amazon SP-API, eBay, Walmart, TikTok Shop, Shopify). Marketplace seller management platforms must implement OAuth 2.0 flows to obtain and refresh access tokens for each integrated marketplace on behalf of sellers.

**RFC 6750 — OAuth 2.0 Bearer Token Usage**
- URL: https://datatracker.ietf.org/doc/html/rfc6750
- Relevance: Defines how access tokens are transmitted in API requests. Required for compliance with all major marketplace API authentication models.

**OpenID Connect Core 1.0 — Identity Layer over OAuth 2.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Relevance: Extends OAuth 2.0 to provide user identity and SSO. Relevant for platforms offering multi-seller account management, reseller portals, and agency access where individual sellers authenticate to a shared management platform.

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Relevance: Defines HTTP method semantics (GET, POST, PUT, DELETE, PATCH) and status codes underpinning all REST marketplace API integrations. Required reading for correct error handling and idempotency in repricing, order, and inventory API calls.

**RFC 7807 — Problem Details for HTTP APIs**
- URL: https://datatracker.ietf.org/doc/html/rfc7807
- Relevance: Standardises structured error payloads from REST APIs. Adopting this RFC for the seller management platform's own API layer improves integration partner experience and makes error handling consistent across all marketplace connectors.

**Schema.org — Product and Offer Vocabulary**
- URL: https://schema.org/Product ; https://schema.org/Offer ; https://schema.org/AggregateOffer
- Relevance: W3C community standard defining structured data vocabulary for products, offers, and aggregate offers (used when multiple sellers offer the same product). Required for Google Merchant Center, rich product search results, and increasingly used by AI agents navigating commerce surfaces. The `AggregateOffer` type is directly relevant to multi-seller marketplace scenarios.

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0.html ; https://swagger.io/specification/
- Relevance: The industry-standard format for documenting REST APIs. Amazon SP-API, Walmart Marketplace API, Zalando Merchant APIs, Mirakl Seller APIs, and ChannelEngine APIs all publish OpenAPI/Swagger specifications. A marketplace seller management platform should expose its own unified API using OAS 3.1 to enable code generation, SDK distribution, and automated contract testing. OAS 3.1 aligns fully with JSON Schema 2020-12.

**JSON Schema 2020-12**
- URL: https://json-schema.org/specification
- Relevance: Standard for validating JSON data structures. Used to define and validate product catalogue schemas, order payloads, and inventory records exchanged between seller management platforms and marketplace APIs. Supports complex conditional validation required for marketplace-specific listing attribute sets (e.g., apparel vs. electronics).

**ANSI ASC X12 EDI — Electronic Data Interchange**
- URL: https://x12.org/products/transaction-sets
- Relevance: The US retail EDI standard, required for integrations with Walmart Marketplace (supplier programme), Target Plus, and large wholesale buyers. Key transaction sets include 850 (Purchase Order), 856 (Advance Ship Notice), 810 (Invoice), and 846 (Inventory Inquiry/Advice). Seller platforms targeting enterprise or 1P-adjacent channels must support X12 over AS2 or SFTP.

**UN/EDIFACT — EDI for Administration, Commerce, and Transport**
- URL: https://unece.org/trade/uncefact/introducing-unedifact
- Relevance: European counterpart to X12 EDI, required for integration with European retail partners and marketplace operators. Used by Zalando supplier integrations and cross-border wholesale channels.

**AS2 Protocol — Applicability Statement 2 (RFC 4130)**
- URL: https://datatracker.ietf.org/doc/html/rfc4130
- Relevance: The dominant transport protocol for EDI messages in retail (mandated by Walmart and widely used by Amazon 1P vendors). AS2 provides encrypted, signed, and receipted HTTPS transmission of EDI documents. Seller management platforms serving enterprise clients must support AS2 alongside REST APIs.

### Security & Authentication Standards

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- Relevance: The definitive industry checklist for API security risks relevant to multi-marketplace seller platforms. Key risks include Broken Object-Level Authorization (critical for multi-seller/multi-account architectures), Broken Authentication (OAuth token management), Unrestricted Resource Consumption (marketplace API rate limiting and quota management), and Server-Side Request Forgery (relevant when proxying webhook callbacks). Compliance is essential for enterprise customers.

**PCI DSS v4.0 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/
- Relevance: Required for any marketplace seller management platform that processes, stores, or transmits payment card data. Platforms that handle seller payout data or facilitate direct checkout must comply with PCI DSS. Marketplaces that handle payments through marketplace operator (Amazon Pay, eBay Managed Payments) reduce PCI scope but platforms must still understand tokenization and data minimisation requirements.

**GDPR — General Data Protection Regulation (EU 2016/679)**
- URL: https://gdpr.eu/
- Relevance: Mandatory for platforms serving European sellers or handling EU buyer data. Seller management platforms process buyer shipping addresses, order details, and communication data, all of which constitute personal data under GDPR. Data Processing Agreements (DPAs) are required with all marketplace API providers; seller platforms must implement data retention limits, right-to-erasure workflows, and consent management.

### MCP Server Specifications

**Model Context Protocol (MCP) — Agentic Commerce Standard**
- URL: https://modelcontextprotocol.io/ ; https://www.essamamdani.com/blog/complete-guide-model-context-protocol-mcp-2026
- Relevance: MCP has emerged as the de facto standard for connecting AI agents to external tools and data sources, with 97M+ monthly SDK downloads and governance transferred to the Agentic AI Foundation (vendor-neutral). In e-commerce, MCP servers are being deployed to give AI agents real-time access to inventory, pricing, order management, and marketplace APIs. A marketplace seller management platform should expose an MCP server to allow AI agents (Claude, ChatGPT, Copilot) to perform seller operations programmatically — repricing, listing updates, order acknowledgement — via natural language instruction. Shopify, Commercetools, and Logicbroker have published MCP server integrations as of 2026.

---

## Similar Products — Developer Documentation & APIs

### Amazon Selling Partner API (SP-API)

- **Description:** Amazon's REST-based API replacing MWS (fully sunset March 2024). Provides programmatic access to listings, orders, inventory, pricing, fulfilment, advertising, and seller performance data for Amazon Marketplace sellers globally. Includes a real-time Notifications API delivering push events for competitive price changes, order updates, and feed processing results.
- **API Documentation:** https://developer-docs.amazon.com/sp-api/docs/welcome
- **SDKs/Libraries:** Python, Java, C#, PHP community SDKs; official Postman collections at https://www.postman.com/amazon-selling-partner-api/sp-api/
- **Developer Guide:** https://developer-docs.amazon.com/sp-api/docs/what-is-the-selling-partner-api
- **Key APIs for Seller Management:**
  - Product Pricing API — competitive pricing, Buy Box data, repricing automation: https://developer-docs.amazon.com/sp-api/docs/product-pricing-api
  - Listings Items API — create, update, delete listing items
  - Orders API — order retrieval and management
  - Notifications API — real-time webhooks (ANY_OFFER_CHANGED, PRICING_HEALTH, ORDER_STATUS_CHANGE)
  - Feeds API — bulk listing and inventory updates
  - Finances API — settlement and payment data
  - Catalog Items API — product catalogue data
- **Standards:** REST/JSON, OAuth 2.0 (Login with Amazon), AWS SigV4 for some endpoints
- **Authentication:** OAuth 2.0 (LWA — Login with Amazon) + AWS Signature Version 4

### eBay Developers Program APIs

- **Description:** eBay's RESTful API suite for professional sellers covering listing management, order processing, inventory synchronisation, fulfilment, analytics, and account configuration. Replaced legacy SOAP Trading API for most new integrations.
- **API Documentation:** https://developer.ebay.com/api-docs/static/ebay-rest-landing.html
- **Developer Guide:** https://developer.ebay.com/api-docs/sell/static/dev-app.html
- **Key Sell APIs:**
  - Inventory API — manage inventory items and offers
  - Fulfillment API — order retrieval and shipment management
  - Account API — seller account configuration (return policies, payment policies)
  - Analytics API — seller performance metrics
  - Marketing API — promotions and campaign management
- **Standards:** REST/JSON, OpenAPI specifications published per API
- **Authentication:** OAuth 2.0 (User and Application tokens)

### Walmart Marketplace API

- **Description:** Walmart's REST API enabling marketplace sellers and approved solution providers to programmatically manage product listings, inventory, pricing, promotions, orders, and WFS (Walmart Fulfillment Services) operations on Walmart.com.
- **API Documentation:** https://developer.walmart.com/us-marketplace/docs/introduction-to-marketplace-apis
- **Developer Guide:** https://developer.walmart.com/us-marketplace/docs/integrate-with-marketplace-apis
- **Key Capabilities:** Item management (catalogue onboarding), inventory updates, order management, pricing and promotions, WFS integration, reporting feeds
- **Standards:** REST/JSON; bulk operations via asynchronous feed APIs; EDI/AS2 also supported for enterprise supplier programmes
- **Authentication:** OAuth 2.0 (Client ID + Client Secret, 15-minute token expiry)

### TikTok Shop Open Platform API

- **Description:** TikTok Shop's API platform enabling sellers, ISVs, and technology partners to integrate with TikTok Shop for listing management, order fulfilment, inventory synchronisation, logistics, and affiliate/creator programme management across all markets where TikTok Shop operates.
- **API Documentation:** https://partner.tiktokshop.com/docv2/page/tts-api-concepts-overview
- **Seller API Overview:** https://partner.tiktokshop.com/docv2/page/seller-api-overview
- **Developer Portal:** https://developers.tiktok.com/
- **Key APIs:** Products API, Orders API, Logistics API, Seller API, Affiliate API, Promotion API
- **Standards:** REST/JSON; HMAC-SHA256 request signature verification
- **Authentication:** OAuth 2.0; per-seller access tokens; HMAC signature for webhook validation

### Mirakl Marketplace Platform APIs

- **Description:** Mirakl powers enterprise operator-run marketplaces (B2B and B2C) for 450+ retailers globally. Its APIs support both marketplace operators and sellers — enabling catalogue management, offer listing, order processing, and accounting workflows for third-party sellers on any Mirakl-powered platform.
- **API Documentation:** https://developer.mirakl.com/
- **Seller API (OpenAPI 3.0):** https://developer.mirakl.com/content/product/mmp/rest/seller/openapi3
- **GitHub Documentation:** https://github.com/channelengine/api-docs (community reference)
- **Key Seller API Areas:** Shop information, Products & Offers (catalogue), Orders management & shipping, Accounting management (invoices)
- **Standards:** REST/JSON; OpenAPI 3.0 specifications published; HTTPS only
- **Authentication:** API Key in Authorization header (seller-scoped)

### Shopify Partner API & Storefront API

- **Description:** Shopify provides APIs for building marketplace-adjacent integrations: the Partner API for programme data and the Storefront API/Admin API for channel integrations. The Marketplace Kit enables third-party developers to build marketplaces on top of Shopify's commerce infrastructure.
- **API Documentation:** https://shopify.dev/docs/api
- **Partner API Reference:** https://shopify.dev/docs/api/partner/latest
- **Storefront GraphQL API:** https://shopify.dev/docs/api/storefront/latest
- **Admin REST/GraphQL API:** https://shopify.dev/docs/api/admin
- **Standards:** REST and GraphQL (SDL); OpenAPI specs available for REST endpoints; versioned quarterly (e.g., 2026-04)
- **Authentication:** OAuth 2.0 for app installs; access tokens per shop; API keys for Partner API

### ChannelEngine Merchant & Channel APIs

- **Description:** ChannelEngine is a multichannel commerce platform that bridges ERP/PIM/WMS systems with 950+ global marketplaces. Its Merchant API enables data synchronisation from existing commerce systems, while its Channel API enables marketplace operators to list on ChannelEngine as a sales channel.
- **API Documentation:** https://www.channelengine.com/developer-hub
- **GitHub API Docs:** https://github.com/channelengine/api-docs
- **Key APIs:** Merchant API (product/offer/order sync from ERP/webstore), Channel API (marketplace operator onboarding), Channel Management API (category and attribute export)
- **Standards:** REST/JSON; OpenAPI specification published (code generation supported); pre-built SDKs available
- **Authentication:** API Key per merchant account

### Rithum (ChannelAdvisor) API

- **Description:** Rithum (formerly ChannelAdvisor) is an enterprise multichannel commerce platform integrating with 420+ global marketplaces. Its REST API enables programmatic management of listings, inventory, pricing, orders, and advertising across all integrated channels.
- **API Documentation:** https://developer.channeladvisor.com/
- **Standards:** REST (primary for new integrations); SOAP V7 (legacy); JSON
- **Authentication:** Developer account required; OAuth-style application credentials
- **Note:** Requires formal developer account application; primarily targeting ISV integrators and enterprise platform partners.

### Zalando Merchant APIs (European Marketplace)

- **Description:** Zalando operates one of Europe's largest fashion and lifestyle marketplaces, with seller/merchant APIs covering product onboarding, offer and inventory management, order fulfilment, and logistics integration for its 17-market European presence.
- **API Documentation:** https://developers.merchants.zalando.com/
- **OpenAPI Specifications:**
  - Orders API: https://developers.merchants.zalando.com/docs/openapi/orders.html
  - Products API: https://developers.merchants.zalando.com/docs/openapi/products.html
- **FCI API Reference:** https://docs.partner-solutions.zalan.do/
- **Standards:** REST/JSON; OpenAPI specifications published; OAuth 2.0 authentication
- **Authentication:** OAuth 2.0

### Lazada Open Platform API (Southeast Asia)

- **Description:** Lazada is a leading Southeast Asian marketplace operating across six regional markets (Indonesia, Malaysia, Philippines, Singapore, Thailand, Vietnam). Its Seller Center API provides RESTful access to product, order, logistics, and finance management for third-party sellers.
- **API Documentation:** https://open.lazada.com/apps/doc/api
- **Seller Center API Overview:** https://lazada-sellercenter.readme.io/docs/introduction-to-the-sellercenter-api
- **Standards:** REST/JSON; HMAC-SHA256 request signing
- **Authentication:** App key + HMAC-SHA256 signature; OAuth for seller access token

### Shopee Open Platform API (Southeast Asia)

- **Description:** Shopee is the dominant marketplace across Southeast Asia and Taiwan, with an Open Platform API enabling developers to build integrations for product listing updates, order management, and real-time stock synchronisation for the seller ecosystem.
- **API Documentation:** https://open.shopee.com/documents
- **Standards:** REST/JSON; HMAC-SHA256 request signing
- **Authentication:** Partner ID + HMAC-SHA256 signature; per-shop access tokens via OAuth-like flow

### Linnworks API

- **Description:** Linnworks is a multichannel inventory and order management platform with an API enabling custom integrations with its order, inventory, listing, and shipping workflows. Primarily used by ISVs building on top of the Linnworks platform for SMB and mid-market sellers.
- **API Documentation:** https://docs.linnworks.com/ (requires Linnworks account)
- **Basic API Info:** https://help.linnworks.com/support/solutions/articles/7000071691-basic-api-info
- **Standards:** REST/JSON; authentication via Linnworks application tokens
- **Authentication:** Application tokens (generated in Linnworks developer settings)

---

## Notes

**Emerging MCP Commerce Ecosystem (2026):** Multiple marketplace and commerce platforms are now publishing MCP servers as a complement to their REST APIs. As AI-agent-driven commerce workflows mature, MCP server exposure may become a de facto expectation alongside OpenAPI documentation for seller management platforms. Logicbroker, Commercetools, and Shopify have published MCP integrations. Platform architecture should accommodate MCP server exposure from the outset.

**Regional Marketplace Gaps:** This standards survey covers major global and regional platforms. Significant regional APIs not covered but relevant for international expansion include Mercado Libre (Latin America), Allegro (Poland/CEE), Cdiscount (France), Ozon (Russia), and Coupang (South Korea). Each has its own developer portal and API authentication model.

**EDI vs. REST Coexistence:** Enterprise seller management platforms must maintain parallel EDI (X12/EDIFACT over AS2) and REST API integration paths, as large retail partners and Walmart's supplier programme continue to require EDI for order and fulfilment workflows despite the broader industry trend toward REST APIs.

**Rate Limiting and Quota Management:** All marketplace APIs enforce rate limits and usage quotas that vary by API, seller tier, and selling plan. Seller management platforms must implement per-marketplace rate limit tracking, exponential backoff, and quota-aware batch scheduling to avoid API suspension — a critical operational concern not addressed by any published standard.
