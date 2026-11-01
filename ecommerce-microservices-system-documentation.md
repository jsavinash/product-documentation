# E-Commerce Microservices Platform — System Documentation

**Project:** Enterprise E-Commerce Microservices Platform  
**Tech Stack:** Java/Kotlin · Gradle (`build.gradle.kts`) · API Gateway · Central Config Server · Service Discovery  
**Domain Modules:** User, Product, Item Management, Cart, Order, Payment, Notification, Offers, Review & Ratings, Recommendations, Serviceability, Archival  
**Status:** Draft v1.0  
**Owner:** Principal Product Manager & Lead Distributed Systems Architect

---

## LAYER 1: BUSINESS SPECIFICATION

### 1.1 Business Objective & Problem Statement

**Problem Statement:**  
Legacy monolithic e-commerce platforms suffer from tightly coupled domains where a change to pricing logic forces redeployment of the entire application. Peak traffic events (flash sales, holiday promotions) cause cascading failures across inventory, cart, and checkout services because there is no isolation or independent scaling. Vendor onboarding takes weeks due to monolithic schema migrations, and real-time personalization is impractical because recommendation logic competes for database resources with transactional order processing.

**Market Gap:**  
Mid-market merchants lack an enterprise-grade, domain-driven e-commerce platform that offers independent scalability per domain, event-driven consistency across services, and real-time personalization without the operational overhead of building distributed systems from scratch.

**Business Objective:**  
Launch a cloud-native, microservices-based e-commerce platform capable of handling 10,000+ concurrent buyers during peak flash-sale events, with sub-200ms P99 API latency for product catalog reads, sub-500ms P99 for checkout flows, and 99.99% uptime for order and payment services. Enable independent deployment cadence per domain (User, Product, Cart, Order, Payment, Notification, Offers, Serviceability, Recommendations, Review & Ratings, Item Management, Archival) with zero-downtime releases.

### 1.2 Core Value Proposition

| Stakeholder | Value Proposition | Delivered By |
|-------------|-------------------|--------------|
| **Buyer** | Sub-second product search, real-time stock visibility, one-click checkout, personalized recommendations, instant order confirmation and delivery tracking | Product, Item, Cart, Order, Payment, Recommendations, Notification |
| **Vendor** | Independent catalog management, dynamic pricing/promotions, serviceability (delivery pincode checks), review management, archival of historical orders | Product, Offers, Serviceability, Review & Ratings, Archival |
| **Platform Operator** | Independent service scaling, centralized config, service discovery, circuit breaking, observability per domain | API Gateway, Config Server, Service Discovery |
| **Platform Owner** | Monetization via transaction fees, subscription tiers, promotional placement fees | Offers, Payment, Order |

### 1.3 Microservice Ecosystem KPIs

| KPI | Definition | Target | Mapped Services |
|-----|------------|--------|-----------------|
| **Cart-to-Order Conversion Rate** | % of carts that convert to completed orders | ≥ 68% | Cart → Order → Payment |
| **Order Fulfillment Time** | Time from order placement to "shipped" status | P95 ≤ 4 hours | Order → Item → Notification |
| **Offer Redemption Rate** | % of issued promo codes successfully redeemed | ≥ 45% | Offers → Cart → Order → Payment |
| **Serviceability Success Rate** | % of pincode serviceability checks returning within SLA | P99 ≤ 100ms, ≥ 99.5% | Serviceability → Item |
| **Platform Availability** | Uptime of Order and Payment services | 99.99% | Order, Payment, API Gateway |

### 1.4 Monetization Model

1. **Transaction Fee (Primary):** 2.5%–4.5% of gross merchandise value (GMV) per completed order, collected by the Payment service and settled to the platform owner daily.
2. **Vendor Subscription Tiers (Secondary):** Monthly SaaS fees for vendors: Starter ($49/mo), Growth ($199/mo), Enterprise ($799/mo). Managed by User service with entitlement checks in Product and Offers services.
3. **Promotional Placement (Tertiary):** Vendors pay for featured placement in Recommendations and Product search results. Managed by Offers and Recommendations services.
4. **Direct Sales (Platform-Owned Inventory):** Platform sells private-label goods directly; revenue recognized in Order service.

---

## LAYER 2: PRODUCT SENSE & USER EXPERIENCE

### 2.1 User Personas

#### Persona A: Buyer (Primary Consumer)
- **Profile:** Tech-savvy consumer, shops via mobile web/app, expects instant gratification
- **Goals:** Find products quickly, verify stock/delivery, checkout seamlessly, track orders
- **Pain Points:** Stockouts discovered at checkout, slow page loads, payment failures without clear retry paths, no delivery visibility
- **Key Services Touched:** Product, Item, Cart, Order, Payment, Notification, Recommendations, Serviceability, Review & Ratings

#### Persona B: Vendor / Seller (Secondary Operator)
- **Profile:** Small-to-medium business owner managing catalog, inventory, and promotions
- **Goals:** List products, set pricing, run promotions, manage reviews, fulfill orders
- **Pain Points:** Complex onboarding, no real-time inventory sync, delayed payout visibility, inability to run targeted offers
- **Key Services Touched:** Product, Item, Offers, Order, Review & Ratings, Archival, Payment (payouts)

#### Persona C: Platform Operator (Tertiary)
- **Profile:** DevOps / SRE team managing the platform infrastructure
- **Goals:** Deploy services independently, monitor per-domain SLAs, rollback failed deployments, manage config
- **Key Services Touched:** API Gateway, Config Server, Service Discovery, all domain services

### 2.2 End-to-End User Journeys

#### Journey 1: Product Search → Cart Addition

| Step | Actor | Service | Action | Data Flow |
|------|-------|---------|--------|-----------|
| 1 | Buyer | API Gateway | GET `/search?q=wireless+headphones&page=1&size=20` | Gateway routes to Product + Recommendations |
| 2 | Product | Product Service | Elasticsearch full-text search; returns 20 products | Product DB (Elasticsearch + PostgreSQL) |
| 3 | Product | Recommendations Service | Injects personalized "You may also like" items | User profile + collaborative filtering |
| 4 | Item | Item Service | Batch stock check for 20 product SKUs | Item DB (PostgreSQL + Redis cache) |
| 5 | API Gateway | Aggregation Layer | Merges product details + stock + recommendations into single response | Client receives unified product catalog |
| 6 | Buyer | UI | Selects product, clicks "Add to Cart" | POST `/cart/items` |
| 7 | Cart | Cart Service | Validates stock via Item service (circuit breaker protected); adds item | Cart DB (Redis) → Item Service (gRPC/REST) |
| 8 | Cart | Cart Service | Returns updated cart with line-item total | Client hydrates cart UI |

#### Journey 2: Cart Checkout → Payment → Notification

| Step | Actor | Service | Action | Data Flow |
|------|-------|---------|--------|-----------|
| 1 | Buyer | API Gateway | POST `/checkout/initialize` | Gateway → Order Service |
| 2 | Order | Order Service | Creates order in `PENDING_PAYMENT` state; reserves stock via Item service (Saga: reserve) | Order DB (PostgreSQL) → Item Service |
| 3 | Order | Order Service | Calculates totals: subtotal + tax + shipping - discounts (from Offers) | Offers Service (discount validation) |
| 4 | Payment | Payment Service | Creates payment intent; redirects buyer to payment provider | Payment DB (PostgreSQL) |
| 5 | Buyer | Payment Provider | Completes payment (card/UPI/wallet) | External payment gateway |
| 6 | Payment | Payment Service | Webhook receives `payment.success`; updates payment status | Payment DB → Order Service (event) |
| 7 | Order | Order Service | Saga step: confirm order → `CONFIRMED`; release stock reservation → `CONFIRMED` | Item Service (release reservation) |
| 8 | Notification | Notification Service | Sends order confirmation email + SMS | Email/SMS provider |
| 9 | Order | Order Service | Emits `OrderConfirmed` event to Kafka | Archival Service consumes for historical storage |
| 10 | Buyer | UI | Redirects to order confirmation page with tracking ID | Polls Order service for status updates |

### 2.3 Functional Requirements (PRD Style)

| ID | Feature | Priority | Owner Service(s) | Description |
|----|---------|----------|------------------|-------------|
| FR-01 | Product search with filters | Must-Have | Product, Item | Full-text search, category, price range, brand, stock availability |
| FR-02 | Real-time stock verification | Must-Have | Item, Cart | Stock checked before cart addition and at checkout |
| FR-03 | Cart persistence across sessions | Must-Have | Cart | Redis-backed cart with 30-day TTL |
| FR-04 | Multi-payment methods | Must-Have | Payment | Card, UPI, wallet, EMI; webhook reconciliation |
| FR-05 | Order status tracking | Must-Have | Order, Notification | Buyer sees order state transitions; email/SMS on change |
| FR-06 | Dynamic promo codes | Should-Have | Offers, Cart | Percentage/fixed discounts, min-order constraints, expiry |
| FR-07 | Serviceability (pincode check) | Should-Have | Serviceability, Item | Delivery availability and estimated date per pincode |
| FR-08 | Product reviews & ratings | Should-Have | Review & Ratings, Product | 1-5 star ratings, text reviews, vendor response |
| FR-09 | Personalized recommendations | Should-Have | Recommendations, Product | Collaborative filtering + trending items injected into search |
| FR-10 | Order archival | Nice-to-Have | Archival, Order | Orders older than 90 days moved to cold storage; retrievable on demand |
| FR-11 | Vendor promo placement | Nice-to-Have | Offers, Recommendations | Paid featured slots in search and homepage |
| FR-12 | Subscription tier entitlements | Nice-to-Have | User, Offers | Tier-based discount caps, free shipping thresholds |

### 2.4 Cross-Service Edge Cases & Failure Modes

| Scenario | Services Involved | Failure Mode | UX Handling |
|----------|-------------------|--------------|-------------|
| Stock changes during checkout | Cart → Order → Item | Item reserved for cart expires before payment completes | Cart shows "Item availability changed" warning; buyer can re-verify stock; auto-refresh cart every 30s via SSE |
| Payment succeeds, Order creation times out | Payment → Order | Payment webhook received, Order service down | Idempotent payment webhook with retry (exponential backoff); Order service reconciles on recovery; buyer sees "Processing your order" with auto-poll |
| Recommendations service down | Product → Recommendations | Personalized fallback unavailable | Graceful degradation: show trending/popular items instead; circuit breaker open for 30s |
| Serviceability check times out | Cart → Serviceability | Pincode delivery unknown | Show "Delivery availability unknown — proceed at your own risk"; allow checkout with disclaimer |
| Notification service down | Order → Notification | Order confirmed but no confirmation email | Outbox pattern: event persisted in Order DB; Notification service consumes on recovery; buyer can "Resend confirmation" |
| Offers service rejects promo at checkout | Cart → Offers → Order | Promo code expired or invalid | Inline validation error on cart page; suggest alternative active promos from Offers service |
| API Gateway timeout (504) | All | Upstream service slow | Client shows retry button with exponential backoff; Skeleton UI for partial data |

---

## LAYER 3: UI ARCHITECTURE & CLIENT ENGAGEMENT

### 3.1 API Gateway Integration & Aggregation with RADIO Framework

The client application interacts with our microservices framework through the API Gateway using structured system design principles. We apply the **RADIO framework** (Requirements, Architecture, Data Model, Interface, Optimizations) to guide our integration strategy:

**R - Requirements Exploration:**
- Clarify user needs: product browsing, cart management, checkout, order tracking
- Define supported devices: mobile web, desktop web, native apps
- Establish constraints: sub-2s time-to-interactive, SEO requirements, accessibility (WCAG 2.1 AA)

**A - Architecture / High-Level Design:**
- **API Gateway Pattern:** Spring Cloud Gateway as single entry point at `https://api.platform.com/v1/{resource}`
- **BFF (Backend for Frontend):** Gateway routes aggregate data from Product, Recommendations, and Item services in parallel
- **Client-Server Architecture:** Stateless gateway routes to domain services; client manages session state via JWT
- **Render Modes:** 
  - SSR (Next.js) for product catalog pages (SEO critical)
  - CSR/SPA for cart, checkout, and account pages (user-specific, dynamic)
- **Global Load Balancing:** DNS-based routing to nearest data center; Cloudflare CDN at edge

**D - Data Model:**
- Unified product catalog view merging: product details + stock status + recommendations + reviews
- Faceted search metadata: brands, price ranges, categories, availability
- Cart state model: `{ items: LineItem[], promotions: [], totals: { subtotal, tax, shipping, grandTotal } }`
- User profile: `{ id, email, addresses: Address[], preferences: { theme, locale, notifications } }`

**I - Interface (API) Contracts:**
- **REST endpoints:** `GET /api/v1/products`, `POST /api/v1/cart/items`, `GET /api/v1/orders/{id}`
- **GraphQL (optional):** For complex detail pages requiring Product + Reviews + Recommendations in single query
- **SSE endpoints:** `GET /api/v1/orders/{orderId}/stream` for real-time order status
- **GraphQL schema excerpt:**
  ```graphql
  type Query {
    products(filters: ProductFilters!, pagination: PaginationInput!): ProductConnection!
    product(id: ID!): Product!
    cart(userId: ID!): Cart!
    order(id: ID!): Order!
  }
  
  type Product {
    id: ID!
    name: String!
    price: Decimal!
    images: [String!]!
    stockStatus: StockStatus!
    averageRating: Float!
    reviewCount: Int!
    isInStock: Boolean!
  }
  ```

**O - Optimizations & Performance:**
- **Response Caching:** Gateway caches non-user-specific responses (product list: 60s TTL, recommendations: 300s TTL)
- **Asset Optimization:** WebP/AVIF images with responsive `srcset`; critical CSS inlining
- **Connection Multiplexing:** HTTP/2 or gRPC for all inter-service and client-gateway communication
- **Edge Caching:** Cloudflare CDN for static assets (TTL 24h with purge on catalog update)
- **Request Deduplication:** In-flight request merging for same product detail fetches
- **Compression:** Brotli for JSON (> 200 bytes); 70% average reduction

**Example Aggregated Response (`GET /v1/products?query=headphones&page=1`):**
```json
{
  "products": [
    {
      "id": "prod_123",
      "name": "Wireless Headphones Pro",
      "price": 12999,
      "currency": "INR",
      "stockStatus": "IN_STOCK",
      "stockQuantity": 45,
      "serviceable": true,
      "estimatedDelivery": "2026-11-03",
      "averageRating": 4.5,
      "reviewCount": 128,
      "isRecommended": true,
      "images": ["https://cdn.platform.com/images/prod_123/1.jpg"],
      "vendorId": "vendor_456"
    }
  ],
  "totalPages": 5,
  "totalCount": 98,
  "recommendations": [
    {"id": "prod_789", "name": "USB-C Charging Cable", "price": 499}
  ],
  "facets": {
    "brands": ["Sony", "Bose", "JBL"],
    "priceRanges": [{"min": 0, "max": 5000}, {"min": 5001, "max": 15000}]
  }
}
```

### 3.2 Core UI States & Data Hydration with Frontend System Design Patterns

**Six-State Component Contract (React/TypeScript):**  
Expanded from four states to include `optimistic` and `offline` states per frontend system design best practices:

```typescript
type DataState<T> =
  | { status: 'idle'; data: null; error: null }
  | { status: 'loading'; data: T | null; error: null }
  | { status: 'success'; data: T; error: null }
  | { status: 'error'; data: T | null; error: ApiError }
  | { status: 'optimistic'; data: T; error: null; pendingMutation: MutationId }
  | { status: 'offline'; data: T | null; error: null; queueId: string };
```

**State Transitions per Operation:**

| Operation | Initial | Loading Trigger | Success | Error | Optimistic | Offline |
|-----------|---------|-----------------|---------|-------|------------|---------|
| Product search | `idle` | Query typed (debounce 300ms) | 200 from Product | 4xx/5xx or timeout | N/A | Serve from cache |
| Real-time stock check | `idle` | User selects variant | Item: `IN_STOCK` | Circuit open | N/A | Show cached status |
| Cart addition | `idle` | User clicks "Add" | Cart confirms | Stock insufficient | `optimistic` → `success/error` | Queue → sync on reconnect |
| Checkout / Order | `idle` | User clicks "Place Order" | Order ID returned | Payment timeout | `optimistic` → `success/error` | Queue → sync on reconnect |
| Order status poll | `idle` | Page mount + 10s | `CONFIRMED` | 5 failures | N/A | Resume on reconnect |

**React Query / SWR Configuration:**
- **Stale-While-Revalidate:** Product catalog cached 5 min, revalidated in background
- **Cache Invalidation:** `cart.deltas` event invalidates cart cache; `order.events` invalidates order cache
- **Retry Logic:** 3 retries with exponential backoff (100ms → 1s → 5s); no retry on 4xx
- **Refetch Interval:** Order status pages poll every 10s; product pages refetch on window focus

**SSE for Real-Time Order Status:**
- Client opens `EventSource` to `GET /v1/orders/{orderId}/stream`
- Server pushes state transitions: PENDING → CONFIRMED → SHIPPED → DELIVERED
- Automatic reconnection on disconnect with `Last-Event-ID` header for resume

**Race Condition Handling:**
- **AbortController:** Cancel in-flight requests when component unmounts or user navigates away
- **Version Stamps:** Each cart operation includes `version` field; server rejects stale operations
- **Deduplication:** In-flight request dedup keyed by request signature (endpoint + params)

### 3.3 Client-Side Performance & Resiliency with Frontend System Design

**Rendering Model with Performance Optimization:**

| Page Type | Render Strategy | Rationale | Key Techniques |
|-----------|----------------|-----------|----------------|
| Product Catalog / Search | SSR (Next.js) | SEO + fast FCP | Critical CSS, image lazy loading, font subsetting |
| Product Detail | SSR + Hydration | SEO + interactivity | Streaming SSR, Suspense boundaries |
| Cart / Checkout | CSR/SPA | User-specific, dynamic | Optimistic updates, client-side validation |
| Account / Profile | CSR/SPA | Authenticated, private | Data prefetching, skeleton screens |
| Order Tracking | SSR + SSE | Public URL + real-time | SSE stream for live updates |

**Network Performance Optimization (per Chakde System Design):**

1. **Resource Prioritization:**
   - `rel="preload"` for critical assets (fonts, hero images)
   - `rel="preconnect"` for API gateway and CDN domains
   - `rel="prefetch"` for next-page resources on hover

2. **Connection Efficiency:**
   - HTTP/2 server push for critical resources
   - Keep-alive connections to gateway
   - TCP/TLS connection pooling

3. **Debounce & Throttle:**
   - Search input debounced at 300ms
   - Scroll event throttled at 100ms for lazy loading triggers
   - Stock check debounced at 500ms after variant selection

**Asset Optimization:**

| Asset Type | Technique | Benefit |
|------------|-----------|---------|
| Product Images | WebP/AVIF format, responsive `srcset`, CDN, lazy loading | 70% reduction in image payload |
| Fonts | WOFF2, font-display: swap, subsetting | Reduced CLS and FOUT |
| JavaScript | Code splitting by route, tree-shaking, minification | Smaller initial bundle |
| CSS | Critical CSS inline, non-critical deferred | Faster first paint |

**Bundle Splitting Strategy:**

```
vendor-bundle.js       → React, React Router, Redux (shared)
feature-catalog.js     → Product search, filters, facets
feature-checkout.js    → Cart, checkout, payment forms
feature-account.js     → User profile, orders, addresses
runtime-bundle.js      → Webpack runtime, manifest
```

**Bundle Size Targets:**
- Initial JS: < 150 KB gzipped
- CSS: < 30 KB gzipped
- Hero image: < 50 KB WebP
- Time to Interactive: < 2s on 4G, < 3s on 3G

**Local Caching Strategies:**

| Cache Type | Storage | TTL | Use Case |
|------------|---------|-----|----------|
| Product Catalog | IndexedDB | 30 min | Browse without re-fetch |
| Cart State | LocalStorage + Redis sync | 30 days | Persist across sessions |
| Auth Token | Memory (access) + httpOnly cookie (refresh) | 15 min / 7 days | XSS protection |
| Serviceability Results | sessionStorage | 15 min | Pincode check during session |
| User Preferences | LocalStorage | 1 year | Theme, locale, notifications |
| Offline Queue | IndexedDB | Until sync | Mutations during offline |

**Request Deduplication & In-Flight Request Management:**
- Single in-flight call per unique request signature
- Components subscribing to same data share the promise
- Cache key includes user ID, pagination, and filters
- Stale data served while revalidating in background

**Error Boundary & Fallback UI:**
- **React Error Boundaries:** Catch component tree errors, show retry UI
- **Skeleton Screens:** Shimmer placeholders during loading states
- **Empty States:** "No products found" with filter reset suggestion
- **Offline Indicator:** Toast notification when client goes offline; queue mutations

**Accessibility & Resiliency:**
- WCAG 2.1 AA compliance (contrast ratio 4.5:1, focus indicators)
- Keyboard navigation for all interactive elements
- Screen reader announcements for state changes (loading → success → error)
- Graceful degradation: basic functionality without JavaScript (progressive enhancement)

---

## LAYER 4: MACHINE CODING & LOW-LEVEL COMPONENT DESIGN

### 4.1 Distributed Design Patterns

#### Pattern 1: Saga Pattern (Choreography-Based) — Distributed Order Transaction

Used for: Order placement spanning Order → Item (stock reservation) → Payment → Order (confirmation).

**Execution Flow:**

```
Buyer → Order Service: POST /orders (PENDING_PAYMENT)
Order Service → Item Service: Reserve stock (Saga step 1)
Item Service → Order Service: Stock reserved event
Order Service → Payment Service: Create payment intent (Saga step 2)
Payment Service → Order Service: Payment success webhook
Order Service → Item Service: Confirm reservation / release (Saga step 3)
Order Service → Order Service: Transition order to CONFIRMED
Order Service → Notification Service: Send confirmation (Outbox pattern)
```

**Compensation Transactions (Rollback):**
- If Payment fails: Order Service sends `ReleaseStock` to Item Service, marks order `CANCELLED`.
- If Item Service unavailable during confirmation: Order retries with exponential backoff; after max retries, marks order `PAYMENT_RECEIVED_STOCK_PENDING` for manual ops review.

#### Pattern 2: Outbox Pattern — Reliable Event Publishing

Used for: Order Service emitting `OrderConfirmed` to Notification, Archival, and Recommendations services without distributed transactions.

**Execution:**
1. Order Service writes `Order` row and `OutboxEvent` row in the **same local transaction** (PostgreSQL).
2. A Change Data Capture (Debezium) or polling relay reads `OutboxEvent` table and publishes to Kafka topic `order.events`.
3. Notification, Archival, and Recommendations services consume from Kafka independently.
4. After successful consumption, OutboxEvent is marked `PROCESSED` and purged after 7 days.

**Benefit:** No dual-write problem. If Kafka is down, events accumulate in Outbox table and are published on recovery.

#### Pattern 3: Circuit Breaker — Resilient Inter-Service Calls

Used for: Cart Service calling Item Service for stock verification; Order Service calling Offers Service for discount validation.

**States:** CLOSED (normal) → OPEN (failure threshold reached, e.g., 50% failures over 10s) → HALF-OPEN (trial request after cool-down).

**Fallback:** Cart Service returns cached stock status or allows cart addition with "Stock verification pending" warning.

---

### 4.2 Inter-Service API Contracts

#### Contract A: Order → Payment (Payment Initiation)

**Request:** `POST https://payment.platform.com/v1/payments/intents`
```json
{
  "orderId": "ord_987654321",
  "customerId": "cust_555666777",
  "amount": 12999,
  "currency": "INR",
  "paymentMethod": "card",
  "returnUrl": "https://platform.com/orders/ord_987654321/return",
  "webhookUrl": "https://order.platform.com/v1/webhooks/payment",
  "metadata": {
    "cartId": "cart_111222333",
    "promoCode": "DIWALI20"
  }
}
```

**Response (201 Created):**
```json
{
  "paymentId": "pay_888999000",
  "orderId": "ord_987654321",
  "status": "PENDING",
  "amount": 12999,
  "currency": "INR",
  "clientSecret": "pi_3QxYz_secret_12345",
  "expiresAt": "2026-11-01T08:30:44Z",
  "createdAt": "2026-11-01T07:30:44Z"
}
```

**Response (400 Bad Request - Invalid promo):**
```json
{
  "error": "INVALID_PROMO",
  "message": "Promo code DIWALI20 has expired or is not applicable to this order",
  "code": 400
}
```

#### Contract B: Order → Item/Inventory (Stock Reservation & Release)

**Request A: Reserve Stock** `POST https://item.platform.com/v1/inventory/reserve`
```json
{
  "orderId": "ord_987654321",
  "items": [
    {
      "sku": "SKU-WHP-001",
      "quantity": 1,
      "reservationTtlSeconds": 900
    }
  ],
  "reservationId": "res_777888999"
}
```

**Response A (200 OK):**
```json
{
  "reservationId": "res_777888999",
  "orderId": "ord_987654321",
  "status": "RESERVED",
  "items": [
    {
      "sku": "SKU-WHP-001",
      "quantity": 1,
      "reservedAt": "2026-11-01T07:31:00Z",
      "expiresAt": "2026-11-01T07:46:00Z"
    }
  ]
}
```

**Response A (409 Conflict - Insufficient stock):**
```json
{
  "error": "INSUFFICIENT_STOCK",
  "message": "Only 2 units available for SKU-WHP-001, requested 1",
  "availableQuantity": 2,
  "code": 409
}
```

**Request B: Confirm/Release Stock** `POST https://item.platform.com/v1/inventory/confirm`
```json
{
  "reservationId": "res_777888999",
  "action": "CONFIRM"
}
```

**Request C: Release Stock** `POST https://item.platform.com/v1/inventory/release`
```json
{
  "reservationId": "res_777888999",
  "action": "RELEASE",
  "reason": "ORDER_CANCELLED"
}
```

**Response (200 OK):**
```json
{
  "reservationId": "res_777888999",
  "action": "CONFIRMED",
  "items": [
    {
      "sku": "SKU-WHP-001",
      "quantity": 1,
      "status": "CONFIRMED"
    }
  ]
}
```

---

### 4.3 Data Models & Domain Interfaces

#### Kotlin — Order Domain Entity

```kotlin
package com.platform.order.domain

import java.time.Instant
import java.util.UUID

enum class OrderStatus {
    PENDING_PAYMENT,
    PAYMENT_RECEIVED,
    CONFIRMED,
    SHIPPED,
    DELIVERED,
    CANCELLED,
    REFUNDED
}

data class OrderItem(
    val sku: String,
    val productId: String,
    val quantity: Int,
    val unitPrice: BigDecimal,
    val totalPrice: BigDecimal
)

data class Order(
    val id: String = UUID.randomUUID().toString(),
    val customerId: String,
    val status: OrderStatus,
    val items: List<OrderItem>,
    val subtotal: BigDecimal,
    val taxAmount: BigDecimal,
    val shippingAmount: BigDecimal,
    val discountAmount: BigDecimal,
    val grandTotal: BigDecimal,
    val currency: String = "INR",
    val paymentId: String? = null,
    val createdAt: Instant = Instant.now(),
    val updatedAt: Instant = Instant.now(),
    val version: Long = 0  // Optimistic locking
)
```

#### TypeScript — User Domain Interface

```typescript
export interface User {
  id: string;
  email: string;
  phone: string;
  firstName: string;
  lastName: string;
  role: 'CUSTOMER' | 'VENDOR' | 'ADMIN';
  subscriptionTier: 'FREE' | 'STARTER' | 'GROWTH' | 'ENTERPRISE';
  addresses: Address[];
  defaultAddressId: string | null;
  emailVerified: boolean;
  phoneVerified: boolean;
  createdAt: string;
  updatedAt: string;
}

export interface Address {
  id: string;
  type: 'HOME' | 'WORK' | 'OTHER';
  street: string;
  city: string;
  state: string;
  pincode: string;
  country: string;
  isDefault: boolean;
}

export interface UserSession {
  accessToken: string;
  refreshToken: string;
  expiresAt: string;
  user: User;
}
```

#### Kotlin — User Domain Entity

```kotlin
package com.platform.user.domain

import java.time.Instant
import java.util.UUID

enum class UserRole { CUSTOMER, VENDOR, ADMIN }
enum class SubscriptionTier { FREE, STARTER, GROWTH, ENTERPRISE }

data class Address(
    val id: String = UUID.randomUUID().toString(),
    val type: String,
    val street: String,
    val city: String,
    val state: String,
    val pincode: String,
    val country: String = "IN",
    val isDefault: Boolean = false
)

data class User(
    val id: String = UUID.randomUUID().toString(),
    val email: String,
    val phone: String,
    val firstName: String,
    val lastName: String,
    val role: UserRole = UserRole.CUSTOMER,
    val subscriptionTier: SubscriptionTier = SubscriptionTier.FREE,
    val addresses: List<Address> = emptyList(),
    val defaultAddressId: String? = null,
    val emailVerified: Boolean = false,
    val phoneVerified: Boolean = false,
    val createdAt: Instant = Instant.now(),
    val updatedAt: Instant = Instant.now()
)
```

---

### 4.4 Complex Logic Breakdown

#### Offers Microservice — Discount Calculation Logic

**Pseudocode:**

```
FUNCTION calculateDiscount(cartId, promoCode, cartItems, subtotal, userTier):
    // 1. Validate promo code existence
    promo = offersRepository.findByCode(promoCode)
    IF promo IS NULL:
        RETURN Error("PROMO_NOT_FOUND")
    
    // 2. Check validity window
    now = currentTimestamp()
    IF now < promo.startDate OR now > promo.endDate:
        RETURN Error("PROMO_EXPIRED")
    
    // 3. Check usage limits
    IF promo.maxUses IS NOT NULL AND promo.usedCount >= promo.maxUses:
        RETURN Error("PROMO_USAGE_LIMIT_REACHED")
    
    // 4. Check per-user usage limit
    userUses = offerUsageRepository.countByUserAndPromo(currentUser.id, promoCode)
    IF promo.maxUsesPerUser IS NOT NULL AND userUses >= promo.maxUsesPerUser:
        RETURN Error("PROMO_PER_USER_LIMIT_REACHED")
    
    // 5. Check minimum order value
    IF subtotal < promo.minOrderValue:
        RETURN Error("MIN_ORDER_VALUE_NOT_MET", minRequired: promo.minOrderValue)
    
    // 6. Check applicable categories (if restricted)
    IF promo.applicableCategories IS NOT NULL:
        cartCategories = cartItems.map(item -> item.category).toSet()
        IF NOT promo.applicableCategories.containsAny(cartCategories):
            RETURN Error("PROMO_NOT_APPLICABLE_TO_CATEGORY")
    
    // 7. Calculate discount amount
    discountAmount = 0
    IF promo.type == 'PERCENTAGE':
        discountAmount = subtotal * (promo.value / 100)
        IF promo.maxDiscountAmount IS NOT NULL:
            discountAmount = min(discountAmount, promo.maxDiscountAmount)
    ELSE IF promo.type == 'FIXED_AMOUNT':
        discountAmount = min(promo.value, subtotal)
    
    // 8. Apply tier-based multiplier (loyalty bonus)
    tierMultiplier = getTierMultiplier(userTier)  // FREE=1.0, STARTER=1.0, GROWTH=1.1, ENTERPRISE=1.2
    discountAmount = discountAmount * tierMultiplier
    
    // 9. Round to currency precision
    discountAmount = round(discountAmount, 2)
    
    RETURN Success(discountAmount, promoId: promo.id)
```

**Example Calculation:**
- Subtotal: ₹5,000
- Promo: `DIWALI20` — 20% off, max discount ₹1,000, min order ₹2,000
- User Tier: GROWTH (1.1x multiplier)
- Base discount: ₹5,000 × 0.20 = ₹1,000 → capped at ₹1,000
- Tier bonus: ₹1,000 × 1.1 = ₹1,100 → re-capped at promo max ₹1,000
- Final discount: ₹1,000
- Payable: ₹4,000

#### Serviceability Microservice — Pincode Evaluation Logic

**Pseudocode:**

```
FUNCTION checkServiceability(pincode, items, preferredDeliveryDate):
    // 1. Validate pincode format (6 digits, India)
    IF NOT pincode.matches(/^[1-9][0-9]{5}$/):
        RETURN Error("INVALID_PINCODE")
    
    // 2. Check serviceability coverage
    coverage = serviceabilityRepository.findByPincode(pincode)
    IF coverage IS NULL OR coverage.status != 'SERVICEABLE':
        RETURN ServiceabilityResult(
            serviceable: false,
            reason: "PINCODE_NOT_SERVICEABLE",
            pincode: pincode
        )
    
    // 3. Check item-level serviceability (vendor may not ship to all pincodes)
    itemResults = []
    FOR item IN items:
        vendorCoverage = itemVendorRepository.findVendorPincode(item.vendorId, pincode)
        IF vendorCoverage IS NULL OR vendorCoverage.status != 'ACTIVE':
            itemResults.add(ItemServiceability(
                sku: item.sku,
                serviceable: false,
                reason: "VENDOR_DOES_NOT_SHIP_TO_PINCODE"
            ))
        ELSE:
            estimatedDelivery = calculateDeliveryDate(
                pincode, 
                vendorCoverage.warehouseId, 
                preferredDeliveryDate
            )
            itemResults.add(ItemServiceability(
                sku: item.sku,
                serviceable: true,
                estimatedDelivery: estimatedDelivery,
                shippingCharges: vendorCoverage.shippingCharges,
                codAvailable: vendorCoverage.codAvailable
            ))
    
    // 4. Overall serviceability
    allServiceable = itemResults.all { it.serviceable }
    overallDeliveryDate = itemResults
        .filter { it.serviceable }
        .map { it.estimatedDelivery }
        .max()  // Latest delivery date among all items
    
    RETURN ServiceabilityResult(
        serviceable: allServiceable,
        pincode: pincode,
        estimatedDelivery: overallDeliveryDate,
        items: itemResults,
        totalShippingCharges: itemResults.sumOf { it.shippingCharges },
        codAvailable: itemResults.all { it.codAvailable }
    )
```

**Example:**
- Pincode: `110001` (Delhi)
- Items: Headphones (Vendor A, ships to 110001), Cable (Vendor B, does NOT ship to 110001)
- Result: `serviceable: false`, Cable marked non-serviceable with reason, Headphones marked serviceable with delivery date 2026-11-03

---

## LAYER 5: BACKEND & DISTRIBUTED SYSTEM ARCHITECTURE

### 5.1 High-Level Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CLIENT (Browser / Mobile App)                       │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ HTTPS / WSS (SSE)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         API GATEWAY (Spring Cloud Gateway)                  │
│  • SSL Termination • Rate Limiting • OAuth2/JWT Validation • Request Routing│
│  • Request/Response Aggregation • Circuit Breaker • Observability           │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ gRPC / REST
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
│  Service Discovery  │ │   Config Server     │ │   Domain Services   │
│  (Eureka / Consul)  │ │ (Spring Cloud Config│ │ • User • Product   │
│  • Service registry │ │  / Git backend)     │ │ • Item • Cart      │
│  • Health checks    │ │ • Environment boot- │ │ • Order • Payment  │
│  • Dynamic routing  │ │  strapping          │ │ • Notification ... │
└─────────────────────┘ └─────────────────────┘ └───────────┬─────────┘
                                                            │ gRPC / REST / Kafka
                                              ┌─────────────┴─────────────┐
                                              ▼                           ▼
                                    ┌────────────────┐         ┌────────────────┐
                                    │  Relational    │         │  Document /    │
                                    │  DB (PostgreSQL│         │  Search (ES)   │
                                    │  / MySQL)      │         │  (Redis / ES)  │
                                    └────────────────┘         └────────────────┘
                                              │
                                              ▼
                                    ┌────────────────┐
                                    │   Kafka        │
                                    │ (Event Stream) │
                                    └────────────────┘
```

**Request Lifecycle (Example: Product Search):**

1. **Config Server Bootstrap:** On service startup, each microservice contacts Config Server (`http://config-server:8888/{service-name}/{profile}`) to fetch environment-specific properties (DB URLs, Kafka brokers, feature flags). Properties are loaded into Spring `Environment` before application context refresh.

2. **Service Discovery Registration:** Each microservice registers with Eureka/Consul on startup, sending heartbeats every 30s. The API Gateway queries the discovery service to resolve service instances dynamically.

3. **Client Request:** Buyer sends `GET /api/v1/products?query=headphones` to API Gateway.

4. **Gateway Authentication:** Gateway extracts JWT from `Authorization` header, validates signature against the auth service's JWKS endpoint, extracts `userId` and `roles`.

5. **Gateway Routing:** Gateway routes to Product Service instances (round-robin from discovery cache). For aggregation, gateway may also call Recommendations and Item services in parallel via `WebClient`.

6. **Domain Service Processing:** Product Service queries Elasticsearch for full-text search, enriches with product details from PostgreSQL, merges recommendations, checks stock via Item Service (gRPC).

7. **Database Read:** PostgreSQL for structured product data; Elasticsearch for search; Redis for hot stock cache.

8. **Response:** Gateway returns aggregated JSON to client with appropriate cache headers.

---

### 5.2 Database Topography & Storage with Enhanced Dropbox-Inspired Patterns

| Microservice | Database Type | Technology | Rationale | Data Boundaries |
|-------------|--------------|------------|-----------|-----------------|
| **User** | Relational | PostgreSQL | ACID for auth, profiles, addresses; strong consistency required | Owns `users`, `addresses`, `user_entitlements` tables |
| **Product** | Relational + Search + Chunk Service | PostgreSQL + Elasticsearch + Redis | ES for full-text search/facets; PG for inventory metadata; Redis for product image chunks and metadata | Owns `products`, `categories`, `product_variants`, `product_chunks` |
| **Item Management** | Relational + Cache + Block Sync | PostgreSQL + Redis | Stock levels need consistency; Redis for hot stock reads; block-level sync for inventory updates | Owns `skus`, `stock_levels`, `reservations`, `inventory_blocks` |
| **Cart** | NoSQL (Key-Value) + Delta Sync | Redis | High write volume, ephemeral data, TTL-based expiration; delta sync for cross-device consistency | Owns `cart:{userId}` hash with TTL 30 days, `cart_deltas` |
| **Order** | Relational + Metadata Store | PostgreSQL + Redis | ACID for financial transactions; Redis metadata store for order status tracking and conflict resolution | Owns `orders`, `order_items`, `order_outbox`, `order_metadata` |
| **Payment** | Relational + Blob Store | PostgreSQL + S3 | ACID for payment records; S3 for payment proof documents and receipts | Owns `payments`, `payment_webhooks`, `refunds`, `payment_blobs` |
| **Notification** | Relational + Stream + Pub/Sub | PostgreSQL + Kafka | Message durability; Kafka for fan-out to email/SMS providers; pub/sub for real-time notifications | Owns `notifications`, `notification_templates`, `notification_subscriptions` |
| **Offers** | Relational + Sequencer | PostgreSQL + Redis | Promo code rules, usage tracking, validity windows; Redis sequencer for unique promo code generation | Owns `promo_codes`, `offer_usage`, `discount_rules`, `promo_sequencer` |
| **Review & Ratings** | Document + Sequencer | MongoDB + Redis | Flexible schema for reviews, ratings, vendor responses; Redis sequencer for review IDs | Owns `reviews`, `ratings`, `review_reactions` |
| **Recommendations** | NoSQL + Cache + Sequencer | Redis + PostgreSQL + Redis | Collaborative filtering data in PG; hot recommendations in Redis; Redis sequencer for recommendation IDs | Owns `recommendations`, `user_behavior_events`, `rec_seq_counter` |
| **Serviceability** | Relational + Sequencer | PostgreSQL + Redis | Pincode coverage, warehouse mapping, delivery SLAs; Redis sequencer for serviceability check IDs | Owns `serviceable_pincodes`, `warehouses`, `delivery_slas`, `service_checks` |
| **Archival** | Object Storage + Cold DB + Sequencer | S3 + PostgreSQL (index) + Redis | Historical orders moved to S3 (Parquet/JSON), PG index for retrieval; Redis sequencer for archival IDs | Owns `archived_orders`, `order_archives`, `arch_seq_counter` |

**Enhanced Data Boundary Enforcement (Dropbox Patterns):**

1. **Metadata Service Pattern (Dropbox Pattern):**
   - Central Metadata Service manages metadata across all microservices
   - Contains file mappings, ownership, access controls, version history
   - Reduces cross-service coupling by centralizing metadata access

2. **Chunk-Based Storage (Dropbox Pattern):**
   - Large files (product images, order documents) split into 4MB-10MB chunks
   - Each chunk uniquely identified by SHA-256 hash
   - Content-based addressing eliminates duplication (e.g., same product images across catalogs)
   - Implements resumable uploads and partial sync

3. **Block-Level Sync Algorithm (Dropbox Pattern):**
   - Cart and order state changes tracked at block level (micro-level updates)
   - Only modified blocks transmitted during synchronization between devices/services
   - Enables efficient conflict detection and resolution
   - Used for real-time cart updates across multiple user devices

4. **Delta Synchronization (Dropbox Pattern):**
   - For cart state, stores only the differences (`delta`) between versions
   - Reduces network bandwidth and improves sync performance
   - Applied to user preferences, recommendations, and serviceability cache

**Resource Estimation (Back-of-Envelope Calculations):**

Assuming 50 million active users (MVU), average 2TB storage per user:

**Storage Estimation:**
- Total storage required: 50M users × 2TB = 100PB (petabytes)
- With 3x replication across 3 data centers: 300PB physical storage
- With compression (avg 2x compression ratio): 150PB physical storage
- With content deduplication (avg 30% reduction): 105PB physical storage

**Bandwidth Estimation:**
- Average data per user per month: 50GB
- Total monthly bandwidth: 50M users × 50GB = 2.5PB/month
- Peak bandwidth (10x average): 25PB/month = ~833GB/s
- Distributed across 3 data centers: ~278GB/s per data center

**Server Estimation:**
- Assuming 64-core servers with 1PB storage and 10Gbps network:
  - Storage servers: 105PB ÷ 1PB = 105 servers
  - Network load balancers: 6 per data center × 3 data centers = 18 servers
  - Application servers (average 10k RPS per server): 4.2M RPS ÷ 10k = 420 servers
  - Metadata service servers: 50 servers
  - Total estimated infrastructure: ~548 servers

---

### 5.3 Scalability, Caching & System Bottlenecks with Dropbox-Inspired Patterns

**Caching Strategy with Block-Level & Delta Sync:**

| Service | Cache Layer | Technology | TTL | Eviction Policy | Pattern |
|---------|------------|------------|-----|-----------------|---------|
| Product | Product details, search results | Redis | 60s (details), 30s (search) | LRU | Cache-Aside |
| Item | Stock levels (hot SKUs) + blocks | Redis | 10s | LRU | Read-Through + Block Cache |
| Cart | Active carts + deltas | Redis | 30 days | TTL-based | Cache-Aside + Delta Sync |
| Recommendations | Personalized recommendations | Redis | 300s | LRU | Write-Through + Populate Ahead |
| Serviceability | Pincode coverage + blocks | Redis | 900s (15 min) | LRU | Read-Through |
| API Gateway | Aggregated responses (non-user-specific) | Redis | 60s | LRU | Cache-Aside |
| Metadata Service | Global file metadata, version tags | Redis | 120s | LRU | Pub/Sub notification + Invalidation |

**Scalability Patterns with Chunk-Based Architecture:**

1. **Product Service (Read-Heavy + Chunk Service):**
   - Read replicas: 3x PostgreSQL read replicas for product catalog queries
   - Elasticsearch cluster: 3 nodes with 2 replicas per shard
   - Redis cache: 90% cache hit rate target for product detail page
   - **Chunk Service:** 4MB chunk storage for product images; SHA-256 hash-based deduplication; 30% storage reduction
   - **Content Delivery:** CDN (CloudFront/AWS CloudFront) for static product images, signed URLs for temporary access

2. **Cart Service (Write-Heavy + Delta Sync):**
   - Redis cluster with sharding by `userId` hash
   - Optimistic locking via Redis `WATCH`/`MULTI`/`EXEC` for concurrent cart modifications
   - **Delta Sync:** Cart state deltas only transmitted on changes (not full cart snapshot)
   - **Hot Key Mitigation:** Consistent hashing with 512 virtual nodes per physical node
   - **Cache Warm-up:** Pre-populate cart data on user login from Archival metadata

3. **Order Service (Write-Heavy, ACID + Version Control):**
   - PostgreSQL primary with 2 read replicas (for order history queries)
   - Table partitioning by `created_at` (monthly partitions) for orders table
   - **Optimistic Locking:** Version column in orders table (`version: Long = 0`)
   - **Conflict Resolution:** Last-write-wins with notification; conflicted copy auto-created for editor collisions
   - **Connection pooling (HikariCP, max 25 connections per instance)**

4. **Payment Service (Write-Heavy, ACID + Idempotency):**
   - PostgreSQL with synchronous replication to standby
   - **Idempotency Keys:** All payment endpoints require `Idempotency-Key` header (UUID v4)
   - **Write-Ahead Log (WAL) archiving** for point-in-time recovery
   - **Chunking for large attachments:** Payment receipts > 10MB split into 4MB chunks stored in S3

5. **Metadata Service (Scalability Pattern):**
   - **Service Discovery with Metadata Caching:** Eureka/Consul + local Redis metadata cache
   - **Partitioned Metadata:** Metadata sharded by user ID hash across 8 metadata nodes
   - **Metadata Version Vectors:** Vector clocks for distributed metadata consistency
   - **Pub/Sub Metadata Updates:** Real-time metadata propagation via Kafka topics

**Event-Driven Streaming (Kafka) with Block-Level Updates:**

| Topic | Producer | Consumers | Purpose | Block-Level |
|-------|----------|-----------|---------|-------------|
| `order.events` | Order Service | Notification, Archival, Recommendations | Order state changes | Full event |
| `payment.events` | Payment Service | Order, Notification | Payment success/failure | Full event |
| `inventory.events` | Item Service | Order, Cart | Stock reservation/release | Full event |
| `user.events` | User Service | Recommendations, Notification | Profile updates | **User preference blocks only** |
| `review.events` | Review & Ratings | Product, Recommendations | New reviews affect search ranking | Full review chunk |
| `cart.deltas` | Cart Service | All services | Cart state changes across devices | **Only modified line items** |
| `sync.blocks` | All Services | Sync Engine | File/product block updates | **Chunk hashes and locations** |

**Bottleneck Mitigation with Chunk-Based & Dropbox Patterns:**

1. **Flash Sale (Item Service Overload):**
   - Circuit breakers on Item Service with fallback to "manual stock verification"
   - **Queue-based order creation:** Orders accepted into Kafka topic `flash-sale-orders`, processed asynchronously with rate limiting (max 5k orders/sec)
   - **Redis distributed locking (Redlock)** for stock reservation on high-contention products (≤ 5 second lock TTL)
   - **Chunk-based order processing:** Large orders (> 20 line items) split into chunks, processed independently

2. **Hot SKUs (Stock Reservation Contention):**
   - **Redlock with 3-node quorum** for stock reservation
   - **Optimistic locking:** Version-based stock check; retry with exponential backoff (100ms → 1s → 5s)
   - **Pre-reservation:** Items pre-reserved in Redis cache for "Add to Cart" actions
   - **Graceful degradation:** Show "Limited stock available" instead of hard errors

3. **Database Connection Exhaustion:**
   - **HikariCP** with timeout=30s and leak-detection-threshold=60s
   - **Read/Write splitting:** All read queries go to replicas; writes to primary only
   - **Connection pooling per tenant:** Multi-tenant isolation with separate connection pools
   - **Graceful degradation:** Auto-switch to "read-only mode" during extreme load

4. **Metadata Service Bottleneck:**
   - **Local Redis cache** per microservice for frequently accessed metadata
   - **Metadata version vectors** to detect stale caches
   - **Background metadata refresh:** Periodic sync (every 30s) from authoritative source
   - **Rate limiting:** 1000 metadata requests/min per service

5. **Cross-Data Center Replication (3 DC Architecture):**
   - **Active-Passive primary DC** with automatic failover
   - **Cross-DC Kafka MirrorMaker 2.** for geo-replication
   - **Chunk replication:** All product/image chunks replicated across all 3 data centers
   - **Metadata conflict resolution:** Last-write-wins with Lamport timestamps for distributed consistency

**Performance Optimization (Dropbox-Inspired):**

1. **Resumable Upload/Download:** All file/product uploads support byte-range requests; clients can resume from last successful chunk
2. **Compression Pipeline:** Gzip/Brotli compression for all JSON payloads (> 200 bytes); 70% average reduction
3. **Connection Multiplexing:** HTTP/2 or gRPC for all inter-service communication; multiplexed streams over single TCP connection
4. **Edge Caching:** CDN edge locations for product images and static assets; TTL 24h with purge on catalog update
5. **Cold Start Optimization:** Service instances warmed up with pre-loaded metadata cache on startup

---

### 5.4 Security & Compliance with Dropbox-Inspired Patterns

**Authentication & Authorization (OAuth2 + mTLS + Service Mesh):**

1. **API Gateway (Entry Point) - Enhanced:**
   - **OAuth2 Resource Server** validates JWT access tokens with JWKS rotation every 24h
   - **JWKS endpoint** at `https://auth.platform.com/.well-known/jwks.json` with automatic key rotation
   - **Token scope validation:** `cart:write`, `order:read`, `payment:write`, `admin:*`, `metadata:read`
   - **Rate limiting per scope:** 1000 req/min for `cart:write`, 500 req/min for `order:write`, 2000 req/min for read scopes
   - **IP reputation filtering** and bot detection at gateway layer (Cloudflare/ModSecurity)

2. **JWT Structure with Enhanced Claims:**
```json
{
  "sub": "cust_555666777",
  "iss": "https://auth.platform.com",
  "aud": "api.platform.com",
  "roles": ["CUSTOMER"],
  "scope": ["cart:write", "order:read", "metadata:read"],
  "permissions": ["cart:add", "cart:remove", "order:create"],
  "tenant_id": "tenant_001",
  "device_id": "device_abc123",
  "session_version": 14,
  "exp": 1793572244,
  "iat": 1793568644
}
```

3. **Inter-Service Security (Zero Trust + mTLS):**
   - **mTLS:** All service-to-service communication over mutual TLS (Istio service mesh with Envoy sidecars)
   - **Service Mesh Policies:** `DestinationRule` and `PeerAuthentication` enforcing STRICT mTLS
   - **Service Tokens:** Short-lived JWTs (5-minute TTL) for service-to-service calls, signed by SPIFFE/SPIRE
   - **Header Propagation:** `X-Request-ID`, `X-User-ID`, `X-Correlation-ID`, `X-B3-TraceID` propagated across all services
   - **Authorization Policy:** Fine-grained RBAC at mesh level (e.g., `cart-service` can only call `item-service:reserve`, not `item-service:delete`)

4. **Metadata Service Security (Dropbox Pattern):**
   - **Access Control Lists (ACLs):** Per-resource ACLs stored in metadata service
   - **Permission Matrix:** Owner, Editor, Viewer roles for shared carts/orders
   - **Temporary Access Tokens:** Time-limited (1hr-7day) signed URLs for shared orders/carts
   - **Audit Trail:** All metadata access logged with user, action, timestamp, and IP

**Configuration Security with Vault & Secret Rotation:**

1. **Central Config Server (Spring Cloud Config + HashiCorp Vault):**
   - **Git backend** encrypted with SOPS/Mozilla SOPS for at-rest encryption
   - **Dynamic secrets** from Vault (database credentials rotated every 24h)
   - **`/actuator` endpoints** restricted to internal network with mTLS
   - **TLS between Config Server and client services** with certificate pinning

2. **Secret Management (Vault + Dynamic Secrets):**
   - **Database credentials:** Dynamic Vault roles with 1-hour TTL, auto-rotated
   - **API keys:** Static secrets in Vault KV v2 with versioning
   - **Signing keys:** JWKS keys managed by Vault Transit engine
   - **No secrets in `build.gradle.kts` or source control** - all injected at runtime via sidecar

3. **Encryption at Rest & In Transit (Dropbox Pattern):**
   - **AES-256-GCM** for all data at rest (databases, S3, Redis, Kafka)
   - **TLS 1.3** for all data in transit (inter-service, client-to-gateway, gateway-to-service)
   - **Chunk-level encryption:** Each 4MB chunk individually encrypted with unique key derived from master key
   - **Key Hierarchy:** Master key in HSM → Data encryption keys per service → Chunk encryption keys per object
   - **Key Rotation:** Annual master key rotation; 90-day DEK rotation; per-chunk keys not rotated (re-encrypted on access)

**Compliance with Dropbox-Level Standards:**

| Requirement | Implementation | Owner Service | Pattern |
|-------------|---------------|---------------|---------|
| **PCI DSS Level 1** | Payment service tokenization; card data never hits platform; encrypted DB fields; HSM for key management | Payment | Tokenization + HSM |
| **GDPR** | Right-to-erasure: Archival service hard-deletes user data on request; consent tracking in User service; data portability API | User, Archival | Erasure + Portability |
| **SOC 2 Type II** | Audit logs (who, what, when) in Order and Payment services; quarterly access reviews; automated compliance dashboards | All services | Audit Logging |
| **RBI (India)** | Data localization: payment and order data stored in India region; 2FA for card payments; encryption at rest in IN data centers | Payment, Order | Geo-fencing |
| **HIPAA (if health products)** | Encryption of PHI; Business Associate Agreements; access logging; breach notification procedures | User, Order, Payment | PHI Protection |
| **ISO 27001** | ISMS with risk assessments; incident response procedures; security awareness training | Platform | ISMS |

**Observability with Distributed Systems Patterns (Dropbox-Enhanced):**

1. **Distributed Tracing (OpenTelemetry + Jaeger + Tempo):**
   - **Trace Context Propagation:** W3C TraceContext (`traceparent`, `tracestate`) headers
   - **Service Graph:** Auto-generated service dependency map with latency distributions
   - **Trace Sampling:** 100% for errors; 10% for success; adaptive sampling based on throughput
   - **Correlation IDs:** `X-Correlation-ID` links business transactions (e.g., order checkout) across services

2. **Metrics (Prometheus + Grafana + Mimir):**
   - **RED Metrics per service:** Rate, Errors, Duration (P50, P95, P99)
   - **USE Metrics per resource:** Utilization, Saturation, Errors (CPU, Memory, Disk, Network)
   - **Business Metrics:** Cart-to-Order conversion, Offer redemption, Serviceability success rate
   - **SLO/SLI Dashboard:** Service Level Objectives with error budget tracking
   - **Long-term Storage:** Grafana Mimir for 13-month metric retention

3. **Structured Logging (ELK + OpenTelemetry Collector):**
   - **JSON Logs:** All services emit structured JSON with `timestamp`, `level`, `service`, `trace_id`, `span_id`, `message`, `fields`
   - **Log Levels:** ERROR (actionable), WARN (degraded), INFO (business events), DEBUG (diagnostic)
   - **Log Sampling:** 100% ERROR/WARN; 10% INFO; 1% DEBUG
   - **Centralized:** OpenTelemetry Collector → Elasticsearch → Kibana

4. **Health Checks (Spring Actuator + Kubernetes Probes):**
   - **Liveness Probe:** `/actuator/health/liveness` - process alive, no dependencies
   - **Readiness Probe:** `/actuator/health/readiness` - dependencies reachable (DB, Kafka, Redis)
   - **Startup Probe:** `/actuator/health/startup` - for slow-starting services (migrations)
   - **Health Indicators:** DB connectivity, Kafka lag, Redis cluster state, disk space, memory pressure

5. **Alerting (Prometheus Alertmanager + PagerDuty + Slack):**
   - **Critical:** Service down > 1min; Error rate > 5% for 2min; P99 latency > 5s
   - **Warning:** Error rate > 1% for 5min; P99 latency > 2s; Disk > 80%; Kafka lag > 10k
   - **Info:** New deployment; Config change; Certificate expiry < 30 days
   - **Runbooks:** Every alert links to runbook with diagnosis steps and remediation procedures

6. **Chaos Engineering (LitmusChaos / Chaos Mesh):**
   - **Weekly experiments:** Pod kill, network latency, CPU stress, DNS failure
   - **Game Days:** Monthly cross-team chaos exercises simulating flash sale + DC failure
   - **Automated Remediation:** Failed experiments auto-create JIRA tickets with findings

**Disaster Recovery (Multi-Region with RPO/RTO):**

| Tier | Services | RPO | RTO | Strategy |
|------|----------|-----|-----|----------|
| **Tier 0 (Critical)** | Order, Payment, User | < 1s | < 5min | Synchronous replication; active-active across 2 IN regions |
| **Tier 1 (Important)** | Cart, Item, Notification | < 1min | < 30min | Async Kafka replication; read replicas in DR region |
| **Tier 2 (Standard)** | Product, Recommendations, Offers, Serviceability | < 1hr | < 4hr | Nightly snapshots; cross-region S3 replication |
| **Tier 3 (Batch)** | Archival, Review & Ratings | < 24hr | < 24hr | Weekly exports; cold storage in Glacier |

**Data Governance:**
- **Data Lineage:** Automated lineage tracking from source (User service) to destination (Archival) via OpenLineage
- **Data Quality:** Great Expectations validation on all critical pipelines (order completeness, payment reconciliation)
- **Data Catalog:** Amundsen/DataHub for service data discovery, ownership, and sensitivity tagging

---

**Document End.**  
**Version:** 1.1 (Dropbox-Inspired Patterns Integrated)  
**Next Review:** After Phase 1 implementation (MVP launch)
