# E-Commerce Microservices Platform — Production System Architecture & Technical Specification

**Project:** Enterprise Cloud-Native E-Commerce Microservices Platform  
**Architecture Classification:** Distributed Event-Driven Microservices Architecture (CQRS + Saga Orchestration)  
**Tech Stack:** Java 21 / Kotlin 2.0 · Gradle (`build.gradle.kts`) · Spring Boot 3.3 / Micronaut · Spring Cloud Gateway / Envoy · Istio Service Mesh · Apache Kafka · PostgreSQL 16 · Redis 7 Cluster · OpenSearch 2.13 · HashiCorp Vault · Kubernetes (EKS/GKE)  
**Domain Modules (Bounded Contexts):** User & IAM, Product Catalog, Item Management & Inventory, Cart & Checkout Session, Order Processing, Payment & Settlement Ledger, Notification Engine, Offers & Promotions, Review & Ratings, Recommendations & Personalization, Serviceability & Logistics, Data Archival & Compliance  
**Document Version:** 2.0 (Production-Grade Enterprise Baseline)  
**Status:** Approved for Implementation  
**Ownership:** Principal Distributed Systems Architect & Lead Product Architect  

---

## LAYER 1: BUSINESS SPECIFICATION & VALUE REALIZATION

### 1.1 Executive Summary & Problem Statement

#### 1.1.1 Problem Statement
Legacy monolithic e-commerce platforms suffer from tight physical and logical coupling where changes to dynamic promotion or catalog schemas require risky, monolithic deployments. Peak flash-sale and seasonal traffic surges (e.g., 50x baseline) induce cascading failures across relational databases, causing lock contention in inventory and checkout tables that paralyzes the entire system. Vendor onboarding takes weeks due to monolithic schema migrations, while real-time personalization algorithms compete directly for CPU and database connection pools with mission-critical order-processing transactions.

#### 1.1.2 Market Gap
Mid-market and enterprise multi-vendor merchants require a platform that provides:
1. **Independent Elasticity:** Ability to scale read-heavy catalog searches (50,000+ QPS) independently from write-heavy transactional checkouts (1,000+ TPS).
2. **Deterministic Flash-Sale Performance:** Guaranteed atomic stock reservation without database deadlocks during high-contention traffic spikes.
3. **Decoupled Business Agility:** Zero-downtime releases allowing separate deployment cycles across 12 distinct functional domains.
4. **Resilient Eventual Consistency:** Fault-tolerant distributed transactions that eliminate double-charging, phantom stock reservations, and silent event loss.

#### 1.1.3 Business Objectives & Target Service Level Objectives (SLOs)
* **Scale Target:** Support 50,000,000 registered users, 5,000,000 Daily Active Users (DAU), and peak throughput of 1,000 completed checkout transactions per second (TPS) during flash-sale events.
* **Catalog Latency:** Sub-100ms P99 API response time for product catalog search and listing views.
* **Checkout Latency:** Sub-400ms P99 end-to-end response time for order placement and payment initiation.
* **High Availability:** 99.99% availability for critical path services (Order, Payment, Inventory, Gateway), equating to less than 4.38 minutes of unscheduled downtime per month.

---

### 1.2 Core Value Proposition Matrix

| Stakeholder | Pain Point in Legacy Architecture | Value Proposition in Target Platform | Key Enabling Microservices |
| :--- | :--- | :--- | :--- |
| **Buyer (Consumer)** | Stockouts discovered only at final payment step; slow page hydration; lost carts across device switches; payment failures with ambiguous states. | Sub-second faceted catalog search; real-time Available-to-Promise (ATP) stock visibility; persistent cross-device carts; transparent order tracking via SSE. | Product Catalog, Inventory, Cart, Order, Payment, Serviceability, Notification |
| **Merchant / Vendor** | Delayed inventory synchronization; manual price updates; inability to run targeted flash promos; opaque payout reconciliation. | Real-time multi-warehouse inventory feed; self-serve promotional campaign configuration; dynamic regional serviceability controls; transparent automated settlement ledgers. | Product, Inventory, Offers, Serviceability, Payment, Review & Ratings |
| **Platform Operator (SRE)** | Single point of failure; cascading service degradation; lack of distributed tracing; risky all-or-nothing deployments. | Zero-downtime canary deployments; circuit breaker isolation; automated horizontal pod autoscaling (HPA); unified OpenTelemetry distributed tracing and metrics. | API Gateway, Service Mesh, Config Server, Central IAM |
| **Platform Owner (Executive)** | Revenue leakages from uncollected commission fees; lack of unified merchant monetization; slow expansion to new geographies. | Automated multi-tiered take-rate deductions; merchant SaaS subscription tier enforcement; paid promotional placements; auditable compliance logs. | User & IAM, Offers, Payment, Recommendations, Archival |

---

### 1.3 Key Performance Indicators (KPIs) & Target SLAs

```
                               ┌───────────────────────────┐
                               │   PLATFORM HEALTH (99.99%)│
                               └─────────────┬─────────────┘
                                             │
             ┌───────────────────────────────┼───────────────────────────────┐
             ▼                               ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────┐     ┌─────────────────────────┐
│     CONVERSION FUNNEL   │     │    FULFILLMENT & SLA    │     │      FINANCIAL SLA      │
│ • Cart-to-Order: ≥ 70%  │     │ • Order-to-Ship: ≤ 4h   │     │ • Payment Success: ≥98% │
│ • Catalog P99: < 100ms  │     │ • Pincode Check: < 50ms │     │ • Reconciliation: 100%  │
│ • Search CTR: ≥ 18%     │     │ • Stock Drift: 0.00%    │     │ • Dispute Rate: < 0.1%  │
└─────────────────────────┘     └─────────────────────────┘     └─────────────────────────┘
```

| Domain Metric | Formal Definition | Target SLA / SLO | Measurement Boundary | Upstream / Downstream Services |
| :--- | :--- | :--- | :--- | :--- |
| **Cart-to-Order Conversion** | `(Completed Orders / Created Carts) * 100` | $\ge 70\%$ | Gateway to Order Confirmation | Cart $\rightarrow$ Inventory $\rightarrow$ Offers $\rightarrow$ Payment $\rightarrow$ Order |
| **Catalog Query P99 Latency** | Time elapsed for faceted search / detail queries | $\le 100\text{ ms}$ | API Gateway Entry to Client | API Gateway $\rightarrow$ Product Catalog $\rightarrow$ OpenSearch |
| **Checkout Flow P99 Latency** | Time to execute stock reservation and create payment intent | $\le 400\text{ ms}$ | POST `/checkout` to Client | Order $\rightarrow$ Inventory $\rightarrow$ Offers $\rightarrow$ Payment |
| **Serviceability SLA** | Roundtrip latency for pincode/hub delivery calculation | $\le 50\text{ ms}$ (P99) | Client request to Gateway | Serviceability $\rightarrow$ Logistics Geofence |
| **Payment Success Rate** | `(Authorized Payments / Attempted Payments) * 100` | $\ge 98.0\%$ | Payment Provider Webhook | Payment $\rightarrow$ External Gateway (Stripe/Adyen) |
| **Inventory Accuracy** | `1 - (Phantom Oversells / Total Orders)` | $100.00\%$ | Reservation Token Expiration | Inventory Service $\rightarrow$ Redis Atomic Tokens |
| **Offer Redemption Rate** | `(Redeemed Vouchers / Issued Vouchers) * 100` | $\ge 40\%$ | Cart Apply to Order Confirm | Offers Service $\rightarrow$ Cart |
| **Fulfillment Handover** | Time from order placement to warehouse dispatch | $\text{P95} \le 4.0\text{ hours}$ | Order Confirmed to Carrier Scan | Order $\rightarrow$ Item/Inventory $\rightarrow$ Logistics |

---

### 1.4 Monetization & Financial Flow Mechanics

1. **Transaction Commission (Take-Rate):**
   * Variable take-rate between $2.5\%$ and $8.0\%$ deducted automatically on every completed order based on merchant category.
   * Settled asynchronously via the Payment Service's double-entry ledger (`ledger_entries`) into escrow, disbursing net proceeds to vendors on a $T+2$ rolling basis.
2. **Merchant SaaS Subscription Tiers:**
   * **Starter ($49/month):** Up to 500 SKUs, standard catalog indexing, 5% commission.
   * **Growth ($199/month):** Up to 10,000 SKUs, priority search ranking, 3.5% commission, dedicated account manager.
   * **Enterprise ($799/month):** Unlimited SKUs, sub-minute inventory sync APIs, custom promo rules, 2.5% commission.
3. **Promotional Placement Fees (Sponsored Products):**
   * Cost-Per-Click (CPC) and Cost-Per-Impression (CPM) bidding engine managed by the Offers and Recommendations services, injecting sponsored catalog items into top search positions.
4. **Logistics & Delivery Convenience Margins:**
   * Dynamic delivery fee calculation by the Serviceability Service based on volumetric weight, distance band, and express fulfillment priority.

---

## LAYER 2: PRODUCT SENSE & USER EXPERIENCE SPECIFICATION

### 2.1 User Personas

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       TARGET PLATFORM PERSONAS                                         │
├────────────────────────────┬────────────────────────────┬─────────────────────────────┬────────────────┤
│ 1. Buyer (Consumer)        │ 2. Merchant / Seller       │ 3. Customer Operations      │ 4. Platform SRE│
│ • Intent: Frictionless buy │ • Intent: Catalog & orders │ • Intent: Issue resolution  │ • Intent: SLA  │
│ • Primary: Mobile/Web      │ • Primary: Merchant Portal │ • Primary: Internal Admin   │ • Primary: K8s │
│ • Latency Sensitive: High  │ • Batch/CRUD Heavy: High   │ • Audit & Traceability: High│ • Zero Downtime│
└────────────────────────────┴────────────────────────────┴─────────────────────────────┴────────────────┘
```

#### Persona 1: Consumer Buyer (Aisha)
* **Demographics:** Urban professional, mobile-first shopper, high disposable income, zero tolerance for payment friction.
* **Goals:** Instant search with autocomplete, real-time stock certainty, single-click checkout, instant payment confirmation, live delivery tracking.
* **Frustrations:** Products appearing in stock that fail at checkout; ambiguous payment processing spinners; delayed SMS/email confirmations.
* **Services Interacted:** API Gateway, Product Catalog, Cart, Inventory, Offers, Payment, Order, Notification, Recommendations.

#### Persona 2: Multi-Warehouse Merchant (Rajesh)
* **Demographics:** Mid-sized electronic retailer operating across 3 regional fulfillment warehouses.
* **Goals:** Bulk SKU updates via API/CSV; real-time inventory synchronization from local ERP; dynamic promo code creation; automated bi-weekly payout settlements.
* **Frustrations:** Delayed inventory updates causing overselling; opaque return processing fees; inability to restrict promotions to specific pincodes.
* **Services Interacted:** Product Catalog, Item Management, Offers, Serviceability, Order, Review & Ratings, Archival.

#### Persona 3: Customer Operations Specialist (Elena)
* **Demographics:** Platform support lead handling tier-2 escalations and chargebacks.
* **Goals:** Unified single-screen timeline of order state transitions, payment gateway transaction IDs, stock reservation logs, and notification delivery receipts.
* **Frustrations:** Disjointed database logs across multiple services; inability to issue partial refunds or manual stock re-allocations without engineering tickets.
* **Services Interacted:** Order, Payment, Inventory, Notification, User & IAM, Archival.

#### Persona 4: Platform SRE (Vikram)
* **Demographics:** Distributed systems engineer maintaining infrastructure stability and resilience.
* **Goals:** Zero-downtime deployments via canary traffic shifting; immediate detection of circuit breaker trip events; strict tenant rate-limiting.
* **Services Interacted:** API Gateway, Istio Service Mesh, HashiCorp Vault, Kafka Cluster, Prometheus/Grafana.

---

### 2.2 End-to-End User Journeys

#### Journey 1: Search Discovery, Real-Time ATP Verification & Atomic Cart Addition

```mermaid
sequenceDiagram
    autonumber
    actor Buyer as Buyer (Browser/App)
    participant GW as API Gateway (Envoy)
    participant Prod as Product Service
    participant OS as OpenSearch Cluster
    participant Item as Item / Inventory Service
    participant Redis as Redis Stock Cluster
    participant Cart as Cart Service

    Buyer->>GW: GET /api/v1/products/search?q=anc+headphones&pincode=110001
    GW->>Prod: Forward Search Request
    Prod->>OS: Execute Boolean Query (Text Match + Facets + Stock Filter)
    OS-->>Prod: Top 20 Product Summaries
    Prod->>Item: Batch Check Available-to-Promise (SKUs, Pincode)
    Item->>Redis: MGET stock:sku_101, stock:sku_102...
    Redis-->>Item: Live Stock Counters
    Item-->>Prod: Enriched Stock & Delivery Estimate Matrix
    Prod-->>GW: Enriched Search Response (JSON)
    GW-->>Buyer: Hydrate Product Grid (with Stock & SLAs)

    Buyer->>GW: POST /api/v1/cart/items (SKU, Qty=1, SessionID)
    GW->>Cart: Add Item Command
    Cart->>Item: Validate SKU Active & Max Per Order Limit
    Item-->>Cart: Validation Passed
    Cart->>Cart: Atomically Update Cart Aggregate in Redis
    Cart-->>GW: Updated Cart View (Items, Subtotal)
    GW-->>Buyer: 200 OK (Optimistic UI Confirmation)
```

#### Journey 2: Distributed Checkout, Saga Stock Reservation & Payment Capture

```mermaid
sequenceDiagram
    autonumber
    actor Buyer as Buyer
    participant GW as API Gateway
    participant Order as Order Service (Saga Orchestrator)
    participant Offers as Offers Service
    participant Item as Inventory Service
    participant Pay as Payment Service
    participant Stripe as External Gateway (Stripe)
    participant Kafka as Kafka Event Bus
    participant Notif as Notification Service

    Buyer->>GW: POST /api/v1/orders/checkout (CartID, AddressID, PaymentMethod)
    GW->>Order: Initialize Order Saga
    Order->>Offers: Validate & Lock Promo Code (DIWALI20)
    Offers-->>Order: Promo Locked (Discount: ₹1,000)
    Order->>Item: Reserve Stock (Order_123, SKU_101, Qty: 1, TTL: 600s)
    Item->>Item: Execute Atomic Lua Script in Redis (Decrement & Create Lease)
    Item-->>Order: Stock Reserved (Reservation_Token_789)
    Order->>Pay: Create Payment Intent (Order_123, Amount: ₹8,999)
    Pay->>Stripe: Initialize Client Secret (Stripe Elements)
    Stripe-->>Pay: Client Secret (pi_3N9...)
    Pay-->>Order: Payment Intent Ready
    Order-->>GW: 201 Created (Order Status: PENDING_PAYMENT, Client Secret)
    GW-->>Buyer: Render Payment Modal (3D Secure)

    Buyer->>Stripe: Complete 3DS Authentication
    Stripe->>Pay: Webhook: payment_intent.succeeded
    Pay->>Pay: Idempotent Ledger Entry Recorded (DB Commit)
    Pay->>Kafka: Publish PaymentCapturedEvent (Order_123)
    Kafka->>Order: Consume PaymentCapturedEvent
    Order->>Item: Commit Stock Reservation (Permanent Deduction)
    Item-->>Order: Stock Committed
    Order->>Order: Transition Order -> CONFIRMED (DB Commit)
    Order->>Kafka: Publish OrderConfirmedEvent (Transactional Outbox)
    Kafka->>Notif: Consume OrderConfirmedEvent
    Notif->>Buyer: Dispatch SMS & Email Confirmation
```

---

### 2.3 Comprehensive Functional Requirements (PRD Specification)

| Requirement ID | Module / Domain | Priority | Description | Acceptance Criteria |
| :--- | :--- | :--- | :--- | :--- |
| **FR-CAT-01** | Product Catalog | P0 | Full-text and faceted search with semantic understanding and spell check. | Query latency $\le 80\text{ ms}$ for 95% of queries over 10M SKUs; fuzzy matching handles 2-character typos. |
| **FR-INV-01** | Item / Inventory | P0 | Real-time Available-to-Promise (ATP) tracking per SKU and regional warehouse. | Stock reservations expire deterministically after 10 minutes if uncommitted; zero phantom overselling under 10k QPS. |
| **FR-CRT-01** | Cart Service | P0 | Distributed cart session supporting guest users and authenticated merge. | Guest cart automatically merged with user cart on login without dropping items; Redis persistence with 30-day TTL. |
| **FR-ORD-01** | Order Processing | P0 | Orchestrated Saga execution for multi-step order lifecycle state transitions. | State machine persists transitions in PostgreSQL with optimistic locking; automatic compensation rollback on failure. |
| **FR-PAY-01** | Payment Service | P0 | PCI DSS SAQ-A compliant multi-provider payment processing with double-entry ledger. | Zero plain-text card storage; idempotent webhook processing; double-entry ledger verification prevents imbalanced settlements. |
| **FR-OFF-01** | Offers Service | P1 | Dynamic rule-based discount evaluation with global usage ceilings. | Support cart-level, SKU-level, and tier-based discounts; atomic usage counters prevent coupon over-redemption. |
| **FR-SRV-01** | Serviceability | P1 | Real-time delivery availability, shipping charges, and SLA calculation per pincode. | Evaluates vendor warehouse proximity, carrier constraints (weight/hazmat), and cutoff times in $\le 50\text{ ms}$. |
| **FR-REV-01** | Review & Ratings | P1 | Verified-buyer review submissions with abuse filtering and Bayesian scoring. | Submissions restricted to users with `DELIVERED` status for that SKU; asynchronous score aggregation to Product Catalog. |
| **FR-REC-01** | Recommendations | P2 | Personalized item recommendations ("Frequently Bought Together", "Trending"). | Ingests clickstream and purchase events from Kafka; serves inference rankings in $\le 60\text{ ms}$ with heuristic fallback. |
| **FR-NOT-01** | Notification | P1 | Multi-channel delivery (Email, SMS, WhatsApp, WebPush) with priority queuing. | P0 notifications (OTP, Order Placed) delivered to vendor provider within 3 seconds of event ingestion. |
| **FR-ARC-01** | Data Archival | P2 | Automated cold-storage partitioning for orders and audits older than 90 days. | Regulatory 7-year audit retention compliance; support on-demand historical queries via Trino/Athena; GDPR data erasure. |
| **FR-USR-01** | User & IAM | P0 | Role-based identity management, OAuth2/OIDC token issuance, and address book. | Token validation via JWKS in sub-5ms at API Gateway; MFA enforcement for merchant and administrative accounts. |

---

### 2.4 Distributed Failure Modes & Edge Case Matrix

| Scenario / Failure Mode | Affected Microservices | Root Cause Trigger | System Mitigation & Recovery Strategy | Client / UX Manifestation |
| :--- | :--- | :--- | :--- | :--- |
| **Flash Sale Stock Exhaustion** | Cart $\rightarrow$ Inventory $\rightarrow$ Order | 50,000 users attempting to buy 1,000 units simultaneously. | Redis Lua script executes atomic test-and-decrement. When counter hits 0, subsequent requests are rejected at cache tier without hitting RDBMS. | Client displays "Sold Out" badge; button disabled; user invited to join "Waitlist" via Notification Service. |
| **Payment Success but Order Timeout** | Payment $\rightarrow$ Order Service | Network partition between Payment Webhook and Order Service after card authorization. | Idempotent webhook receiver writes to Payment DB and publishes to Kafka. Order Service reconciles missing state via polling consumer or CDC replay. | Buyer sees "Payment Received — Securing Your Order" screen with auto-polling SSE stream; no duplicate charge. |
| **Payment 3DS Abandonment** | Payment $\rightarrow$ Inventory $\rightarrow$ Order | Buyer closes browser during banking OTP / 3DS authentication. | Redis reservation token expires at 600s TTL. Scheduled Reaper job or Redis key-expiration hook returns units to ATP pool. Order marked `ABANDONED`. | Cart retains items with notification: "Stock reservation expired. Click checkout to re-verify availability." |
| **Serviceability Timeout** | API Gateway $\rightarrow$ Serviceability | Logistics database latency spike exceeds 100ms threshold. | Resilience4j circuit breaker trips to OPEN. Gateway executes fallback to regional pincode cache (stale-while-revalidate). | Client displays: "Estimated Delivery: 3-5 Days (Live verification pending)." |
| **Double Webhook Delivery** | Payment Gateway $\rightarrow$ Payment | External payment gateway (Stripe/Adyen) delivers duplicate webhooks due to slow ACK. | Unique constraint on `payment_transactions.provider_reference`. Subsequent webhook hits idempotent deduplication check and returns `200 OK` immediately. | Silent backend deduplication; user experiences zero impact or double-allocation. |
| **Notification Provider Outage** | Order $\rightarrow$ Notification Engine | Third-party SMS/Email gateway experiences 5xx failure. | Notification Service queues event in dead-letter retry topic with exponential jitter; falls back to secondary SMS carrier (e.g., Twilio $\rightarrow$ AWS SNS). | In-app notification center displays order confirmation immediately; SMS arrives with slight delay. |

---

## LAYER 3: PRODUCTION FRONTEND & CLIENT ARCHITECTURE

### 3.1 RADIO Framework Implementation for E-Commerce

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              RADIO FRAMEWORK ARCHITECTURE                              │
├─────────────────────┬─────────────────────┬────────────────────┬───────────────────────┤
│ R - Requirements    │ A - Architecture    │ D - Data Models    │ I - Interfaces / APIs │
│ • Core Web Vitals   │ • Next.js 14+ SSR   │ • Normalised Store │ • Typed REST / JSON   │
│ • SEO Maximisation  │ • Module Federation │ • Optimistic Carts │ • SSE Order Stream    │
│ • WCAG 2.1 AAA      │ • Edge Cloudflare   │ • Offline Queues   │ • GraphQL BFF Aggreg  │
└─────────────────────┴─────────────────────┴────────────────────┴───────────────────────┘
```

#### R — Requirements & Constraints
* **Performance Budget:** First Contentful Paint (FCP) $\le 1.2\text{s}$, Largest Contentful Paint (LCP) $\le 2.0\text{s}$, Cumulative Layout Shift (CLS) $\le 0.05$, Interaction to Next Paint (INP) $\le 150\text{ms}$.
* **Device Spectrum:** Responsive web (Mobile/Tablet/Desktop) + Native iOS/Android apps sharing core API contracts.
* **SEO Constraints:** 100% server-renderable HTML for all public catalog, brand, and category pages with dynamic OpenGraph meta tags and Schema.org microdata.

#### A — Architecture & Hybrid Rendering Strategy
* **Catalog & Landing Pages:** Server-Side Rendering (SSR) with Incremental Static Regeneration (ISR) using Next.js 14+ App Router. Catalog pages revalidate every 60 seconds.
* **Cart, Checkout & User Dashboard:** Client-Side Rendering (CSR) protected by authenticated sessions and edge-routed to nearest Cloudflare Points of Presence (PoP).
* **Micro-Frontend Modularization:** Core domains partitioned via Webpack Module Federation:
  * `catalog-mfe`: Search bar, filter facets, product display cards.
  * `checkout-mfe`: Cart drawer, promo code applicator, address selector, payment container.
  * `account-mfe`: Order history, tracking timeline, reviews, returns management.

#### D — Client State Architecture & Normalized Data Stores
* Local client cache partitioned into:
  1. **Server State (TanStack Query):** Product queries, search results, stock status, cached with strict TTLs and stale-while-revalidate policies.
  2. **Session / Client State (Zustand):** Local cart modifications, drawer toggles, active filter selections.
  3. **Offline Queue (IndexedDB):** Queued offline mutations (e.g., wishlist additions, unsent analytics events).

#### I — Interface Protocols & Aggregation
* **Primary Interconnect:** RESTful HTTPS JSON APIs mediated by Spring Cloud Gateway / Envoy.
* **Complex Page Aggregation:** Backend-for-Frontend (BFF) aggregation layer combining Product Details, Live Stock, Reviews, and Related Items into a single payload.
* **Live Streams:** Server-Sent Events (SSE) on `/api/v1/orders/{orderId}/timeline` for real-time order status tracking.

#### O — Edge & Client Optimizations
* **Image Transformation Pipeline:** Dynamic WebP/AVIF generation with `srcset` served from edge CDN with 30-day immutability headers.
* **Resource Prioritization:** Critical CSS inlined; non-critical script bundles deferred; DNS prefetch and preconnect for CDN and Payment Gateway origins.

---

### 3.2 6-State Component Design Contract

Every user-facing component adheres to a deterministic 6-state finite state machine in TypeScript:

```typescript
export type AsyncState<TData, TError = ApiError> =
  | { status: 'IDLE'; data: null; error: null }
  | { status: 'LOADING'; data: TData | null; error: null }
  | { status: 'SUCCESS'; data: TData; error: null }
  | { status: 'ERROR'; data: TData | null; error: TError }
  | { status: 'OPTIMISTIC'; data: TData; error: null; pendingMutationId: string }
  | { status: 'OFFLINE_QUEUED'; data: TData | null; error: null; queueReference: string };

export interface ApiError {
  code: string;
  message: string;
  fieldErrors?: Record<string, string>;
  requestId: string;
  timestamp: string;
}
```

#### Deterministic State Transition Matrix for Cart Modification

| Trigger / User Action | Current State | Target State | UI Behavior & DOM Updates | Network & Storage Action |
| :--- | :--- | :--- | :--- | :--- |
| Initial Page Load | `IDLE` | `LOADING` | Skeleton loader rendered in cart sidebar. | Fetch `GET /api/v1/cart` via TanStack Query. |
| Fetch Successful | `LOADING` | `SUCCESS` | Cart items, prices, and checkout button activated. | Hydrate in-memory Zustand store and localStorage. |
| Network Disconnected | `SUCCESS` | `OFFLINE_QUEUED` | Offline indicator shown; optimistic item update retained. | Mutation stored in IndexedDB offline write-queue. |
| User Clicks "Add Item" | `SUCCESS` | `OPTIMISTIC` | Badge counter increments immediately; subtotal updates. | Dispatches `POST /api/v1/cart/items` with rollback snapshot. |
| Server Returns 200 OK | `OPTIMISTIC` | `SUCCESS` | Subtle checkmark animation; pending mutation cleared. | Overwrite local state with authoritative server cart payload. |
| Server Returns 409 Conflict | `OPTIMISTIC` | `ERROR` | Revert badge counter; toast: "Insufficient stock for SKU". | Rollback to pre-mutation snapshot; invalidate query cache. |

---

### 3.3 Production Frontend Technical Requirements Matrix

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND TECHNICAL CAPABILITIES                           │
├────────────────────────────┬─────────────────────────────┬─────────────────────────────┤
│ Performance & Rendering    │ Security & Resiliency       │ Accessibility & Inclusivity │
│ • Next.js App Router (SSR) │ • CSP Level 3 Strict        │ • WCAG 2.1 AAA Compliant    │
│ • WebP / AVIF Responsive   │ • Anti-CSRF Double-Submit   │ • ARIA Live Regions         │
│ • Brotli Compression       │ • Subresource Integrity SRI │ • Full Keyboard Navigability│
│ • Suspense Streaming       │ • Error Boundary Sandboxing │ • Screen Reader Friendly    │
└────────────────────────────┴─────────────────────────────┴─────────────────────────────┘
```

* **Core Web Vitals Enforcement:** Synthetic Lighthouse CI integrated into GitHub Actions rejecting PRs with performance scores $< 90$.
* **Accessibility (a11y):** Full semantic HTML5; ARIA live regions for cart updates (`aria-live="polite"`); focus trapping within modals; color contrast ratio $\ge 7:1$.
* **Progressive Web App (PWA):** Workbox Service Worker caching static shells; background sync for offline analytics; Web App Manifest enabling standalone mobile installation.
* **Security Measures:** Strict Content Security Policy (`default-src 'self'`); anti-CSRF tokens embedded in mutating requests; XSS sanitization via DOMPurify.

---

## LAYER 4: MACHINE CODING & LOW-LEVEL COMPONENT DESIGN

### 4.1 Core Distributed Design Patterns

#### 4.1.1 Orchestrated Saga Pattern — Order Fulfillment State Machine
Choreographed sagas become unmaintainable as domain complexity grows. This platform uses an **Orchestrated Saga Pattern** where the `Order Service` serves as the central stateful orchestrator.

```mermaid
stateDiagram-v2
    [*] --> CREATED
    CREATED --> INVENTORY_RESERVED: Step 1: Reserve Stock (Item Service)
    CREATED --> FAILED: Stock Unavailable
    
    INVENTORY_RESERVED --> PROMO_APPLIED: Step 2: Apply Promo (Offers Service)
    INVENTORY_RESERVED --> CANCELLED: Compensation: Release Stock
    
    PROMO_APPLIED --> PAYMENT_PENDING: Step 3: Create Payment Intent (Payment Service)
    
    PAYMENT_PENDING --> ORDER_CONFIRMED: Step 4: Capture Payment (Payment Webhook)
    PAYMENT_PENDING --> PAYMENT_FAILED: Payment Declined / Timeout
    
    PAYMENT_FAILED --> CANCELLED: Compensation: Release Promo & Release Stock
    
    ORDER_CONFIRMED --> ALLOCATED: Step 5: Warehouse Allocation
    ALLOCATED --> SHIPPED: Step 6: Carrier Handover
    SHIPPED --> DELIVERED: Step 7: Final Delivery
    
    ORDER_CONFIRMED --> REFUNDED: Post-Confirm Cancellation
    REFUNDED --> CANCELLED: Compensation: Reverse Payment Ledger & Return Stock
```

* **Compensation Logic Guarantee:** Every forward step has an idempotent compensating action:
  * `ReserveStock` $\rightarrow$ `Compensate: ReleaseStockReservation`
  * `ApplyPromo` $\rightarrow$ `Compensate: UnlockPromoCode`
  * `AuthorizePayment` $\rightarrow$ `Compensate: VoidPaymentAuthorization / IssueRefund`

#### 4.1.2 Transactional Outbox Pattern with Debezium CDC
To prevent distributed dual-write inconsistencies between PostgreSQL and Apache Kafka:
1. Every domain service writes its domain entity state change and an outbox event record within the **same local ACID database transaction**.
2. Debezium Change Data Capture (CDC) monitors the PostgreSQL Write-Ahead Log (WAL) and publishes events to Kafka with zero application-level dual-write vulnerability.

```sql
-- Outbox Table Schema in PostgreSQL (Partitioned by Month)
CREATE TABLE transactional_outbox (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(64) NOT NULL,
    aggregate_id VARCHAR(64) NOT NULL,
    event_type VARCHAR(64) NOT NULL,
    payload JSONB NOT NULL,
    trace_context JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMPTZ NULL
) PARTITION BY RANGE (created_at);
```

#### 4.1.3 High-Concurrency Atomic Flash-Sale Stock Engine
To achieve sub-5ms stock reservations under extreme concurrency without relational row deadlocks, stock counters are decremented atomically using a Redis Lua script:

```lua
-- Atomic Stock Reservation Lua Script (Executed on Redis Shard)
-- KEYS[1]: stock:counter:{sku_id}
-- KEYS[2]: stock:leases:{order_id}
-- ARGV[1]: requested_quantity
-- ARGV[2]: order_id
-- ARGV[3]: lease_ttl_seconds

local current_stock = redis.call('GET', KEYS[1])
if not current_stock then
    return -1 -- SKU not found
end

local available = tonumber(current_stock)
local required = tonumber(ARGV[1])

if available >= required then
    -- Atomically decrement stock counter
    redis.call('DECRBY', KEYS[1], required)
    -- Record reservation lease with TTL
    redis.call('HSET', KEYS[2], 'sku', KEYS[1], 'qty', required, 'reserved_at', redis.call('TIME')[1])
    redis.call('EXPIRE', KEYS[2], tonumber(ARGV[3]))
    return 1 -- Reservation Successful
else
    return 0 -- Insufficient Stock
end
```

---

### 4.2 Protobuf & Inter-Service gRPC Contracts

For latency-critical communication between the Order Service and Inventory Service, binary gRPC over HTTP/2 is utilized in place of REST:

```protobuf
syntax = "proto3";

package com.platform.inventory.v1;

option java_multiple_files = true;
option java_package = "com.platform.inventory.v1";

service InventoryService {
  rpc ReserveStock (ReserveStockRequest) returns (ReserveStockResponse);
  rpc CommitReservation (CommitReservationRequest) returns (CommitReservationResponse);
  rpc ReleaseReservation (ReleaseReservationRequest) returns (ReleaseReservationResponse);
  rpc CheckAvailability (CheckAvailabilityRequest) returns (CheckAvailabilityResponse);
}

message StockItemRequest {
  string sku = 1;
  int32 quantity = 2;
  string warehouse_id = 3;
}

message ReserveStockRequest {
  string order_id = 1;
  string customer_id = 2;
  repeated StockItemRequest items = 3;
  int32 reservation_ttl_seconds = 4;
  string idempotency_key = 5;
}

message ReserveStockResponse {
  enum ReservationStatus {
    RESERVATION_STATUS_UNSPECIFIED = 0;
    SUCCESS = 1;
    INSUFFICIENT_STOCK = 2;
    SKU_NOT_FOUND = 3;
    DUPLICATE_IDEMPOTENT_REQUEST = 4;
  }
  ReservationStatus status = 1;
  string reservation_id = 2;
  int64 expires_at_epoch_ms = 3;
  string error_message = 4;
}

message CommitReservationRequest {
  string reservation_id = 1;
  string order_id = 2;
}

message CommitReservationResponse {
  bool committed = 1;
  int64 committed_at_epoch_ms = 2;
}

message ReleaseReservationRequest {
  string reservation_id = 1;
  string order_id = 2;
  string cancellation_reason = 3;
}

message ReleaseReservationResponse {
  bool released = 1;
  int32 restored_quantity = 2;
}

message CheckAvailabilityRequest {
  repeated string skus = 1;
  string destination_pincode = 2;
}

message StockAvailability {
  string sku = 1;
  int32 available_to_promise = 2;
  bool is_serviceable = 3;
  int32 estimated_delivery_days = 4;
}

message CheckAvailabilityResponse {
  repeated StockAvailability items = 1;
}
```

---

### 4.3 Kafka CloudEvents Specification

All asynchronous events adhere to the CNCF CloudEvents standard formatted in JSON:

#### `OrderConfirmedEvent` Specification (`order.events` topic)

```json
{
  "specversion": "1.0",
  "type": "com.platform.order.confirmed.v1",
  "source": "/services/order-service/cluster-prod-01",
  "id": "evt_9c8b7a6d-4e3f-4a1b-8c2d-1e0f9a8b7c6d",
  "time": "2026-11-01T08:30:00Z",
  "datacontenttype": "application/json",
  "dataschema": "https://schema.platform.com/events/order-confirmed-v1.json",
  "data": {
    "orderId": "ord_8829104",
    "customerId": "cust_440192",
    "orderStatus": "CONFIRMED",
    "currency": "INR",
    "financialSummary": {
      "subtotal": 12999.00,
      "discountAmount": 1000.00,
      "shippingFee": 0.00,
      "taxAmount": 2159.82,
      "grandTotal": 14158.82
    },
    "paymentDetails": {
      "paymentTransactionId": "txn_pay_7721049",
      "paymentMethod": "UPI",
      "gatewayProvider": "RAZORPAY"
    },
    "lineItems": [
      {
        "sku": "SKU-HEADPHONE-PRO-BLK",
        "quantity": 1,
        "unitPrice": 12999.00,
        "vendorId": "vnd_tech_direct",
        "warehouseId": "wh_delhi_hub_01"
      }
    ],
    "shippingAddress": {
      "addressId": "addr_991823",
      "pincode": "110001",
      "city": "New Delhi",
      "state": "Delhi",
      "country": "IN"
    }
  }
}
```

---

### 4.4 Production Kotlin Domain Entities

#### Order Domain Aggregate Root (Kotlin + Spring Data JPA)

```kotlin
package com.platform.order.domain

import jakarta.persistence.*
import org.hibernate.annotations.JdbcTypeCode
import org.hibernate.type.SqlTypes
import java.math.BigDecimal
import java.time.Instant
import java.util.UUID

enum class OrderStatus {
    CREATED,
    INVENTORY_RESERVED,
    PAYMENT_PENDING,
    CONFIRMED,
    ALLOCATED,
    SHIPPED,
    DELIVERED,
    CANCELLED,
    REFUNDED
}

@Entity
@Table(
    name = "orders",
    indexes = [
        Index(name = "idx_orders_customer", columnList = "customer_id"),
        Index(name = "idx_orders_status", columnList = "order_status"),
        Index(name = "idx_orders_created_at", columnList = "created_at")
    ]
)
class Order(
    @Id
    @Column(name = "order_id", length = 64, nullable = false, updatable = false)
    val id: String = "ord_${UUID.randomUUID()}",

    @Column(name = "customer_id", length = 64, nullable = false)
    val customerId: String,

    @Enumerated(EnumType.STRING)
    @Column(name = "order_status", length = 32, nullable = false)
    var status: OrderStatus = OrderStatus.CREATED,

    @OneToMany(cascade = [CascadeType.ALL], orphanRemoval = true, fetch = FetchType.EAGER)
    @JoinColumn(name = "order_id", nullable = false)
    val lineItems: MutableList<OrderLineItem> = mutableListOf(),

    @Embedded
    var financialSummary: OrderFinancialSummary,

    @Column(name = "reservation_id", length = 64)
    var reservationId: String? = null,

    @Column(name = "payment_intent_id", length = 64)
    var paymentIntentId: String? = null,

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(name = "shipping_snapshot", columnDefinition = "jsonb", nullable = false)
    val shippingSnapshot: ShippingAddressSnapshot,

    @Version
    @Column(name = "aggregate_version", nullable = false)
    var version: Long = 0,

    @Column(name = "created_at", nullable = false, updatable = false)
    val createdAt: Instant = Instant.now(),

    @Column(name = "updated_at", nullable = false)
    var updatedAt: Instant = Instant.now()
) {
    fun confirmPayment(transactionId: String) {
        check(this.status == OrderStatus.PAYMENT_PENDING || this.status == OrderStatus.INVENTORY_RESERVED) {
            "Cannot transition order $id to CONFIRMED from state ${this.status}"
        }
        this.paymentIntentId = transactionId
        this.status = OrderStatus.CONFIRMED
        this.updatedAt = Instant.now()
    }

    fun markCancelled(reason: String) {
        check(this.status != OrderStatus.DELIVERED && this.status != OrderStatus.SHIPPED) {
            "Cannot cancel order $id when already in state ${this.status}"
        }
        this.status = OrderStatus.CANCELLED
        this.updatedAt = Instant.now()
    }
}

@Entity
@Table(name = "order_line_items")
class OrderLineItem(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(name = "sku", length = 64, nullable = false)
    val sku: String,

    @Column(name = "product_name", length = 255, nullable = false)
    val productName: String,

    @Column(name = "quantity", nullable = false)
    val quantity: Int,

    @Column(name = "unit_price", precision = 12, scale = 2, nullable = false)
    val unitPrice: BigDecimal,

    @Column(name = "total_price", precision = 12, scale = 2, nullable = false)
    val totalPrice: BigDecimal,

    @Column(name = "vendor_id", length = 64, nullable = false)
    val vendorId: String
)

@Embeddable
data class OrderFinancialSummary(
    @Column(name = "subtotal_amount", precision = 12, scale = 2, nullable = false)
    val subtotal: BigDecimal,

    @Column(name = "discount_amount", precision = 12, scale = 2, nullable = false)
    val discount: BigDecimal,

    @Column(name = "tax_amount", precision = 12, scale = 2, nullable = false)
    val tax: BigDecimal,

    @Column(name = "shipping_fee", precision = 12, scale = 2, nullable = false)
    val shippingFee: BigDecimal,

    @Column(name = "grand_total", precision = 12, scale = 2, nullable = false)
    val grandTotal: BigDecimal,

    @Column(name = "currency", length = 3, nullable = false)
    val currency: String = "INR"
)

data class ShippingAddressSnapshot(
    val recipientName: String,
    val street: String,
    val city: String,
    val state: String,
    val pincode: String,
    val phone: String
)
```

---

### 4.5 Production Logic Algorithms

#### 4.5.1 Offers Engine: Multi-Factor Promo Validation & Atomic Reservation
This algorithm validates coupon applicability against cart contents, user tier caps, category exclusions, and enforces distributed usage limits:

```
FUNCTION evaluateAndLockPromo(userId, userTier, cartAggregate, promoCode):
    // Step 1: Fetch promo definition from Read-Through Cache
    promo = OffersCache.get("promo:def:" + promoCode)
    IF promo IS NULL:
        promo = OffersDatabase.findByCode(promoCode)
        IF promo IS NULL:
            RETURN Error(CODE="PROMO_NOT_FOUND", MSG="Invalid promo code")
        OffersCache.set("promo:def:" + promoCode, promo, TTL=300s)

    // Step 2: Validate temporal window
    currentTime = System.currentTimeUtc()
    IF currentTime < promo.validFrom OR currentTime > promo.validUntil:
        RETURN Error(CODE="PROMO_EXPIRED", MSG="Offer has expired")

    // Step 3: Check global & user redemption ceiling atomically in Redis
    userUsageKey = "promo:user:" + promoCode + ":" + userId
    globalUsageKey = "promo:global:" + promoCode

    // Atomic evaluation script via Redis
    redisLockResult = Redis.executeScript("""
        local globalCount = redis.call('GET', KEYS[1]) or 0
        local userCount = redis.call('GET', KEYS[2]) or 0
        if tonumber(globalCount) >= tonumber(ARGV[1]) then
            return -1 -- Global limit exceeded
        end
        if tonumber(userCount) >= tonumber(ARGV[2]) then
            return -2 -- User limit exceeded
        end
        redis.call('INCR', KEYS[1])
        redis.call('INCR', KEYS[2])
        return 1
    """, keys=[globalUsageKey, userUsageKey], args=[promo.maxGlobalRedemptions, promo.maxPerUserRedemptions])

    IF redisLockResult == -1:
        RETURN Error(CODE="PROMO_EXHAUSTED", MSG="Global redemption limit reached")
    IF redisLockResult == -2:
        RETURN Error(CODE="USER_LIMIT_EXCEEDED", MSG="You have already redeemed this promo")

    // Step 4: Validate cart minimum order value (MOV)
    IF cartAggregate.subtotal < promo.minimumOrderValue:
        // Compensate: release the incremented redis counters
        Redis.decr(globalUsageKey)
        Redis.decr(userUsageKey)
        RETURN Error(CODE="MOV_NOT_MET", MSG="Minimum order value of ₹" + promo.minimumOrderValue + " required")

    // Step 5: Filter applicable category items
    applicableSubtotal = 0
    FOR item IN cartAggregate.items:
        IF promo.restrictedCategories IS EMPTY OR promo.restrictedCategories.contains(item.categoryId):
            applicableSubtotal = applicableSubtotal + item.totalPrice

    IF applicableSubtotal == 0:
        Redis.decr(globalUsageKey)
        Redis.decr(userUsageKey)
        RETURN Error(CODE="CATEGORY_MISMATCH", MSG="No eligible items in cart for this offer")

    // Step 6: Compute base discount
    baseDiscount = 0
    IF promo.discountType == "PERCENTAGE":
        baseDiscount = applicableSubtotal * (promo.discountValue / 100.0)
        IF promo.maxDiscountCap IS NOT NULL:
            baseDiscount = MIN(baseDiscount, promo.maxDiscountCap)
    ELSE IF promo.discountType == "FIXED_AMOUNT":
        baseDiscount = MIN(promo.discountValue, applicableSubtotal)

    // Step 7: Apply User Loyalty Tier Multiplier
    tierMultiplier = 1.00
    SWITCH userTier:
        CASE "ENTERPRISE": tierMultiplier = 1.25
        CASE "GROWTH":     tierMultiplier = 1.15
        CASE "STARTER":    tierMultiplier = 1.05
        DEFAULT:           tierMultiplier = 1.00
    
    finalDiscount = ROUND(baseDiscount * tierMultiplier, 2)
    IF promo.maxDiscountCap IS NOT NULL:
        finalDiscount = MIN(finalDiscount, promo.maxDiscountCap)

    RETURN Success(
        promoId = promo.id,
        code = promoCode,
        discountApplied = finalDiscount,
        reservationLockId = UUID.randomUUID()
    )
```

#### 4.5.2 Serviceability Engine: Geo-Proximity & Carrier Routing SLA
Evaluates whether a multi-warehouse shipment can be fulfilled to a target destination pincode within SLA:

```
FUNCTION evaluateServiceability(destinationPincode, cartItems, requestedPriority):
    // Step 1: Validate destination postal format
    IF NOT destinationPincode.matches("^[1-9][0-9]{5}$"):
        RETURN ServiceabilityResult(isServiceable=false, reason="INVALID_PINCODE_FORMAT")

    // Step 2: Fetch destination geographical hub metadata
    destHub = ServiceabilityCache.getHubByPincode(destinationPincode)
    IF destHub IS NULL OR destHub.isActive == false:
        RETURN ServiceabilityResult(isServiceable=false, reason="NON_SERVICEABLE_ZONE")

    warehouseShipmentPlans = []
    overallServiceable = true

    // Step 3: Iterate each SKU and resolve nearest vendor warehouse holding stock
    FOR item IN cartItems:
        candidateWarehouses = InventoryService.findWarehousesWithStock(item.sku, item.quantity)
        IF candidateWarehouses IS EMPTY:
            RETURN ServiceabilityResult(isServiceable=false, reason="SKU_OUT_OF_STOCK_GLOBALLY", sku=item.sku)

        // Select warehouse with lowest geodesic / road distance
        selectedWarehouse = NULL
        minDistanceKm = INFINITY
        FOR wh IN candidateWarehouses:
            distance = calculateHaversineDistance(wh.latitude, wh.longitude, destHub.latitude, destHub.longitude)
            IF distance < minDistanceKm:
                minDistanceKm = distance
                selectedWarehouse = wh

        // Step 4: Check courier operational capability (e.g. Hazardous, Cold-Chain, Heavy)
        carrierRoute = RoutingMatrix.findOptimalCarrier(
            originHubId = selectedWarehouse.hubId,
            destHubId = destHub.hubId,
            weightGrams = item.weightGrams,
            isHazardous = item.isHazardous
        )

        IF carrierRoute IS NULL:
            overallServiceable = false
            BREAK

        // Step 5: Compute SLA considering warehouse daily cutoff time
        currentTime = System.currentTimeUtc()
        dispatchDayOffset = 0
        IF currentTime.toLocalTime() > selectedWarehouse.dailyCutoffTime:
            dispatchDayOffset = 1 // Orders past 4 PM dispatch next morning

        calculatedDeliveryDate = currentTime.plusDays(dispatchDayOffset + carrierRoute.transitDays)

        warehouseShipmentPlans.add(ShipmentPlan(
            sku = item.sku,
            originWarehouseId = selectedWarehouse.id,
            carrierId = carrierRoute.carrierId,
            transitDays = carrierRoute.transitDays,
            estimatedDelivery = calculatedDeliveryDate,
            shippingFee = carrierRoute.baseCost + (item.weightGrams / 1000.0 * carrierRoute.ratePerKg)
        ))

    IF NOT overallServiceable:
        RETURN ServiceabilityResult(isServiceable=false, reason="NO_CARRIER_SERVICE_FOR_WEIGHT_OR_ZONE")

    // Aggregate overall metrics (latest delivery date determines overall SLA)
    maxDeliveryDate = warehouseShipmentPlans.map(p -> p.estimatedDelivery).max()
    totalShippingFee = warehouseShipmentPlans.map(p -> p.shippingFee).sum()

    RETURN ServiceabilityResult(
        isServiceable = true,
        destinationPincode = destinationPincode,
        latestEstimatedDelivery = maxDeliveryDate,
        totalShippingFee = ROUND(totalShippingFee, 2),
        splitShipmentRequired = (warehouseShipmentPlans.map(p -> p.originWarehouseId).distinct().size() > 1),
        shipmentPlans = warehouseShipmentPlans
    )
```

---

## LAYER 5: BACKEND & DISTRIBUTED SYSTEMS ARCHITECTURE

### 5.1 Global High-Level System Architecture

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT TIER (Web, iOS, Android)                           │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │ Anycast DNS / HTTPS (TLS 1.3)
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│               GLOBAL EDGE NETWORK (Cloudflare Enterprise CDN & WAF)                     │
│  • DDoS Shield • Bot Mitigation • Static Asset Cache (Images/JS) • SSL Termination     │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │ HTTP/2 & HTTP/3
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│             API GATEWAY & SECURITY PERIMETER (Spring Cloud Gateway / Envoy)            │
│  • Distributed Token-Bucket Rate Limiter (Redis) • OAuth2 / JWKS Token Verification     │
│  • Request Header Sanitization • W3C Trace Context Injection • Circuit Breaker Routing │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │ Internal Service Mesh (Istio STRICT mTLS)
        ┌───────────────────────────────────┼───────────────────────────────────┐
        ▼                                   ▼                                   ▼
┌───────────────────────┐       ┌───────────────────────┐       ┌───────────────────────┐
│ PRODUCT CATALOG SVC   │       │ CART & SESSION SVC    │       │ ORDER SAGA ORCHESTR.  │
│ OpenSearch + Mongo/PG │       │ Redis 7 Cluster       │       │ PostgreSQL 16 (ACID)  │
└───────────┬───────────┘       └───────────┬───────────┘       └───────────┬───────────┘
            │                               │                               │
            │                               │ gRPC Binary                   │ Debezium CDC
            ▼                               ▼                               ▼
┌───────────────────────┐       ┌───────────────────────┐       ┌───────────────────────┐
│ ITEM & INVENTORY SVC  │       │ OFFERS & PROMO SVC    │       │ PAYMENT & LEDGER SVC  │
│ Redis Lua + PostgreS  │       │ Redis + PostgreS      │       │ PostgreSQL Primary/Stby│
└───────────┬───────────┘       └───────────┬───────────┘       └───────────┬───────────┘
            │                               │                               │
            └───────────────────────────────┼───────────────────────────────┘
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│               ENTERPRISE EVENT BUS (Apache Kafka Multi-AZ Cluster)                     │
│   Topics: order.events · payment.events · inventory.events · notification.events        │
└───────┬───────────────────────────┬───────────────────────────┬────────────────────────┘
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│ NOTIFICATION  │           │ RECOMMENDATION│           │ ARCHIVAL &    │
│ ENGINE        │           │ ENGINE (Flink)│           │ COMPLIANCE    │
│ SendGrid/Twil │           │ Redis + Python│           │ S3 / Parquet  │
└───────────────┘           └───────────────┘           └───────────────┘
```

---

### 5.2 Polyglot Persistence & Database Topography

Under the **Database-per-Service** architectural pattern, each microservice encapsulates its own persistent store with zero cross-database queries:

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              POLYGLOT PERSISTENCE TOPOLOGY                              │
├────────────────────┬────────────────────┬────────────────────┬──────────────────────────┤
│ Microservice       │ Primary Store      │ Secondary / Cache  │ Rationale & SLA          │
├────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Product Catalog    │ PostgreSQL 16      │ OpenSearch 2.13    │ CQRS: Writes to RDBMS;   │
│                    │ (Master Catalog)   │ (Search & Facets)  │ CDC to OpenSearch index  │
├────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Item & Inventory   │ PostgreSQL 16      │ Redis 7 Cluster    │ Redis Lua for sub-5ms    │
│                    │ (Ledger History)   │ (Atomic Counters)  │ flash reservation; RDBMS │
├────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Cart & Session     │ Redis 7 Cluster    │ RocksDB / AWS S3   │ In-memory hash sets with │
│                    │ (Sharded by User)  │ (Abandoned Carts)  │ 30-day TTL; sub-2ms R/W  │
├────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Order Processing   │ PostgreSQL 16      │ Redis              │ Strict ACID; table       │
│                    │ (Multi-AZ Replica) │ (Read-Through)     │ partition by created_at  │
├────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Payment & Ledger   │ PostgreSQL 16      │ None (Strict ACID) │ Double-entry bookkeeping │
│                    │ (Sync Standby)     │                    │ Financial auditability   │
├────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ User & IAM         │ PostgreSQL 16      │ Redis (JWKS & Auth)│ OAuth token revocation   │
├────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Offers & Promos    │ PostgreSQL 16      │ Redis (Rate Caps)  │ Atomic usage counters    │
├────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Serviceability     │ PostgreSQL PostGIS │ Redis (Geo Index)  │ H3 hexagonal geospatial  │
├────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Review & Ratings   │ MongoDB 7.0        │ Redis (Hot Reviews)│ Flexible review schema   │
├────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Data Archival      │ Apache Iceberg     │ AWS S3 / Glacier   │ Parquet cold partitions  │
└────────────────────┴────────────────────┴────────────────────┴──────────────────────────┘
```

---

### 5.3 Realistic E-Commerce Capacity Planning & System Sizing

#### 5.3.1 Traffic & Scale Parameters
* **Registered Accounts:** $50,000,000$ users.
* **Daily Active Users (DAU):** $5,000,000$ users.
* **Peak Flash-Sale Concurrent Users:** $100,000$ active connections.
* **Average Daily Completed Orders:** $500,000$ orders/day.
* **Peak Order Velocity (Flash-Sale):** $1,000\text{ Orders/sec}$ ($1,000\text{ TPS}$).
* **Browse-to-Buy Ratio:** $50 : 1$ (50 search/catalog reads per 1 order checkout).
* **Peak Catalog Read Traffic:** $50,000\text{ QPS}$ at API Gateway.

#### 5.3.2 Storage Calculations
1. **Order Transactions:**
   * Average order record size (including JSON snapshots and line items): $4\text{ KB}$.
   * Daily storage consumption: $500,000 \times 4\text{ KB} = 2.0\text{ GB/day}$.
   * Annual order storage: $2.0\text{ GB} \times 365 = 730\text{ GB/year}$.
   * 5-Year active transactional retention (PostgreSQL): $\approx 3.65\text{ TB}$ (trivially managed via monthly table partitioning).
2. **Product Catalog & Media Assets:**
   * $10,000,000$ active catalog SKUs $\times 2\text{ KB}$ metadata $= 20\text{ GB}$ raw RDBMS storage.
   * OpenSearch Search Indices (with inverted text indices, vectors, and facets): $\approx 120\text{ GB}$.
   * Product Images: $10,000,000\text{ SKUs} \times 4\text{ images} \times 400\text{ KB (WebP)} = 16\text{ TB}$ stored in AWS S3 / Google Cloud Storage, with $98\%$ served directly from Cloudflare Edge Caches.
3. **Cart In-Memory Footprint (Redis):**
   * $5,000,000\text{ active carts} \times 1.5\text{ KB (JSON structure)} = 7.5\text{ GB}$ active memory.
   * Scaled with $3\times$ replication factor across 6 Redis shards: $22.5\text{ GB}$ total RAM required.

#### 5.3.3 Compute & Network Throughput
* **Edge CDN Ingress/Egress:** Peak catalog egress at $50,000\text{ QPS} \times 15\text{ KB (compressed JSON)} = 750\text{ MB/s} = 6.0\text{ Gbps}$.
* **API Gateway Sizing:** Envoy instances processing $50,000\text{ QPS}$; at $5,000\text{ QPS}$ per 8-core CPU node $\rightarrow 10$ active Gateway instances (autoscaling to 20 during flash sales).
* **Kafka Cluster Throughput:** Peak write volume of $25,000\text{ events/sec} \times 2\text{ KB} = 50\text{ MB/s}$ ingress. A 5-broker Kafka cluster with NVMe SSDs handles this with $< 20\%$ disk I/O saturation.

---

### 5.4 Flash-Sale Hardening & Concurrency Control

```
                              ┌────────────────────────┐
                              │  50,000 CONCURRENT USERS│
                              └───────────┬────────────┘
                                          │
                                          ▼
                              ┌────────────────────────┐
                              │ CLOUDFLARE WAITING ROOM│ (Smooths excess spikes)
                              └───────────┬────────────┘
                                          │
                                          ▼
                              ┌────────────────────────┐
                              │ API GATEWAY RATE LIMIT │ (Token-Bucket per User)
                              └───────────┬────────────┘
                                          │
                  ┌───────────────────────┴───────────────────────┐
                  │                                               │
                  ▼                                               ▼
      ┌───────────────────────┐                       ┌───────────────────────┐
      │   HOT SKU (FLASH SALE)│                       │     STANDARD CATALOG  │
      └───────────┬───────────┘                       └───────────┬───────────┘
                  │                                               │
                  ▼                                               ▼
      ┌───────────────────────┐                       ┌───────────────────────┐
      │ REDIS LUA ATOMIC LEASE│ (Sub-5ms Lockless)    │ POSTGRES READ-REPLICA │
      └───────────┬───────────┘                       └───────────────────────┘
                  │
                  ▼
      ┌───────────────────────┐
      │ ASYNC KAFKA QUEUE     │ (Drives Saga DB writes)
      └───────────────────────┘
```

1. **Virtual Waiting Room (Queue-It / Cloudflare Waiting Room):** Activated when checkout ingress exceeds $2,500\text{ requests/sec}$, metering traffic into the API Gateway deterministically.
2. **Distributed Rate Limiting:** Enforced at the API Gateway via Redis Token-Bucket:
   * Anonymous IP: Max $100\text{ requests/min}$.
   * Authenticated Buyer: Max $1,000\text{ requests/min}$.
   * Checkout Endpoint: Max $5\text{ attempts/min}$ per `userId`.
3. **Cache Stampede Prevention:**
   * **Probabilistic Early Expiration (XFetch Algorithm):** Recomputes catalog cache keys before they expire based on read frequency and background worker calculation time.
   * **Singleflight Request Coalescing:** Gateway merges concurrent in-flight requests for identical product detail fetches, preventing backend query surges.
4. **Hot SKU Locking with Redlock & Fencing Tokens:**
   * High-value limited items utilize Redlock across 3 independent Redis nodes with monotonically increasing fencing tokens to prevent split-brain double-reservation.

---

### 5.5 Zero-Trust Security, PCI DSS & Regulatory Compliance

#### 5.5.1 Authentication, Authorization & Identity Management
* **OAuth 2.0 & OpenID Connect (OIDC):** Microservices act as OAuth 2.0 Resource Servers validating JWT tokens issued by the User & IAM Service (powered by Keycloak / Ory Hydra).
* **JWKS Cryptographic Rotation:** API Gateway verifies token signatures using public keys fetched from `https://auth.platform.com/.well-known/jwks.json`, with automated key rotation every 30 days.
* **Workload Identity (SPIFFE/SPIRE):** Every Kubernetes pod receives a short-lived X.509 certificate validating its workload identity, establishing **STRICT mutual TLS (mTLS)** across all service-to-service communication via Istio.
* **Fine-Grained RBAC / ABAC Permissions:** Enforced at Envoy sidecars via Open Policy Agent (OPA) filters:
  ```json
  {
    "sub": "usr_9921049",
    "iss": "https://auth.platform.com",
    "roles": ["CUSTOMER"],
    "scope": "openid profile cart:write order:create payment:initiate",
    "exp": 1793572244,
    "jti": "jwt_token_unique_id"
  }
  ```

#### 5.5.2 PCI DSS Level 1 SAQ-A Compliance Boundary
* **Zero Primary Account Number (PAN) Footprint:** No credit card numbers, CVVs, or expiration dates ever touch or transit platform servers.
* **Tokenized Client-Side Capture:** Buyer enters card details directly into hosted iframes / SDKs provided by certified payment processors (Stripe Elements / Adyen Drop-in / Razorpay Checkout).
* The Payment Service processes only opaque tokens (e.g., `tok_1N4m...`), satisfying **PCI DSS SAQ-A** eligibility and dramatically lowering compliance overhead.

#### 5.5.3 Regulatory Framework Enforcement
* **RBI Data Localization (India):** All transactional order, user PII, and financial settlement databases reside exclusively within AWS/GCP regions inside India (`ap-south-1` Mumbai / `ap-south-2` Hyderabad).
* **GDPR Compliance (Right to Erasure / Portability):**
  * When a user exercises their Right to be Forgotten, User Service publishes `UserAccountDeletedEvent`.
  * The Archival Service executes a cryptographic erase or hard-delete of personal identifiers across cold tables, preserving anonymized order records for statutory accounting obligations.

---

### 5.6 Enterprise Observability, SRE & Disaster Recovery

#### 5.6.1 OpenTelemetry Distributed Tracing & W3C Standards
Every inbound request at the API Gateway is tagged with a standard **W3C TraceContext** header:
* `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`
* Injected into downstream HTTP headers, gRPC metadata, and Kafka message headers.
* Traces ingested into OpenTelemetry Collector and exported to **Grafana Tempo / Jaeger**, guaranteeing end-to-end request visibility across all 12 domains.

#### 5.6.2 Metrics & Golden Signals (Prometheus / Alertmanager)
* **RED Metrics:** Rate (requests/sec), Errors (5xx status count), Duration (P50, P95, P99 latency) per microservice.
* **USE Metrics:** Utilization, Saturation, and Errors for CPU, memory, and database connection pools.
* **Core Production Alerts:**
  * **P0 - Critical (PagerDuty):** Order Service 5xx error rate $> 1.0\%$ for 2 consecutive minutes; Payment webhook failure rate $> 0.5\%$; Inventory Redis cluster CPU saturation $> 85\%$.
  * **P1 - Warning (Slack):** P99 catalog search latency $> 250\text{ ms}$; Kafka consumer group lag $> 10,000\text{ messages}$; Free disk space $< 20\%$.

#### 5.6.3 Disaster Recovery Tiering & Multi-Region Topology

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              DISASTER RECOVERY ARCHITECTURE                            │
├────────────────────┬────────────────────┬──────────┬──────────┬────────────────────────┤
│ Recovery Tier      │ Domains Covered    │ RPO      │ RTO      │ Failover Topology      │
├────────────────────┼────────────────────┼──────────┼──────────┼────────────────────────┤
│ **Tier 0**         │ Order, Payment,    │ $< 1\text{ sec}$ │ $< 3\text{ min}$ │ Active-Active Hot Standby│
│ (Mission Critical) │ Inventory, User    │          │          │ Multi-AZ Sync Repl     │
├────────────────────┼────────────────────┼──────────┼──────────┼────────────────────────┤
│ **Tier 1**         │ Cart, Offers,      │ $< 30\text{ s}$│ $< 15\text{ min}$│ Cross-Region Asymmetric│
│ (Business Critical)│ Serviceability     │          │          │ Kafka MirrorMaker 2.0  │
├────────────────────┼────────────────────┼──────────┼──────────┼────────────────────────┤
│ **Tier 2**         │ Product Catalog,   │ $< 1\text{ hr}$  │ $< 2\text{ hr}$  │ Read Replicas + Hourly │
│ (Operational)      │ Reviews, Notif     │          │          │ Snapshots to S3        │
├────────────────────┼────────────────────┼──────────┼──────────┼────────────────────────┤
│ **Tier 3**         │ Recommendations,   │ $< 24\text{ hr}$ │ $< 8\text{ hr}$  │ Batch Rehydration from │
│ (Analytical)       │ Archival Analytics │          │          │ Cold Parquet Tables    │
└────────────────────┴────────────────────┴──────────┴──────────┴────────────────────────┘
```

* **Recovery Point Objective (RPO):** Maximum acceptable data loss window during catastrophic regional failure.
* **Recovery Time Objective (RTO):** Maximum duration to restore services following a disaster declaration.
* **Automated Chaos Testing:** Chaos Mesh scheduled weekly in pre-production simulating pod evictions, network latency injection, and Kafka broker kills to continuously validate self-healing capabilities.

---

## 6. SYSTEM IMPLEMENTATION ROADMAP

```mermaid
gantt
    title E-Commerce Microservices Production Rollout Plan
    dateFormat  YYYY-MM-DD
    section Phase 1: Core Foundation
    API Gateway & Mesh Setup           :2026-11-01, 30d
    User & IAM Service                 :2026-11-15, 30d
    Product Catalog & OpenSearch       :2026-11-20, 45d
    section Phase 2: Transaction Engine
    Item & Inventory (Redis Lua)       :2026-12-15, 45d
    Cart & Session Microservice        :2027-01-01, 30d
    Order Saga Orchestrator            :2027-01-15, 45d
    Payment & Settlement Ledger        :2027-02-01, 45d
    section Phase 3: Experience & Scale
    Offers & Promotions Engine         :2027-03-01, 30d
    Serviceability & Logistics         :2027-03-15, 30d
    Notification Multi-Provider        :2027-04-01, 30d
    Review & Recommendations           :2027-04-15, 45d
    section Phase 4: Production Hardening
    Chaos Engineering & Load Testing   :2027-05-15, 30d
    Production Canary Cutover          :2027-06-15, 15d
```

---

**Document Approval & Sign-Off:**

| Role | Name | Status | Timestamp |
| :--- | :--- | :--- | :--- |
| **Lead Distributed Systems Architect** | Distributed Systems Architecture Review Board | **APPROVED** | 2026-11-01T14:00:00Z |
| **Head of Infrastructure & SRE** | Site Reliability Engineering Council | **APPROVED** | 2026-11-01T14:15:00Z |
| **Principal Product Architect** | E-Commerce Product Engineering Committee | **APPROVED** | 2026-11-01T14:30:00Z |
