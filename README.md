# Module 9 Tutorial B: Visualizing and Architectural Risk

This document summarizes the current microservice architecture of the JaStip Online Nasional (JSON) project and then expands it with proposed improvements and module-level analysis.

Working assumptions used in the diagrams:

- `group-preparation` is treated as the Tutorial B deliverable repository because it is the only documentation-only repository in the organization.
- `frontend` is treated as the active user interface because its README explicitly states it is the real Milestone 75 frontend. Service-local `frontend/` folders inside `Order`, `Wallet`, and `Voucher-Promo` are treated as legacy or service-scoped assets rather than separate production containers.
- The deployment view reflects the Cloud Run commands and runtime defaults documented in each service README. Where multiple database profiles exist, the currently documented demo profile is treated as the current architecture.
- No message broker is wired into the current milestone implementation. Service-to-service integration is synchronous HTTP REST.

## Current Architecture

### System Context Diagram

```mermaid
flowchart LR
    admin["Admin"]
    titiper["Titiper / Buyer"]
    jastiper["Jastiper / Traveler"]

    json["JaStip Online Nasional (JSON)<br/>Microservice-based jastip platform"]

    admin -->|"manage users, vouchers, and orders"| json
    titiper -->|"browse catalog, top up wallet, checkout, rate orders"| json
    jastiper -->|"manage products and fulfill orders"| json
```

Short explanation:
The current system serves three user roles from the project brief: Admin, Titiper, and Jastiper. At this level, the system behaves as one business platform that handles authentication, catalog browsing, wallet transactions, order checkout, and voucher usage.

### Container Diagram

```mermaid
flowchart LR
    users["Web users<br/>(Admin, Titiper, Jastiper)"]

    subgraph json["JSON platform"]
        direction LR
        frontend["Frontend SPA<br/>React + Vite + Nginx"]

        auth["Auth/Profile API<br/>Spring Boot"]
        inventory["Inventory API<br/>Spring Boot"]
        wallet["Wallet API<br/>Spring Boot"]
        order["Order API<br/>Spring Boot"]
        voucher["Voucher/Promo API<br/>Spring Boot"]

        authdb[("Auth data<br/>H2 file")]
        inventorydb[("Inventory data<br/>H2 file")]
        walletdb[("Wallet data<br/>H2 file")]
        orderdb[("Order data<br/>H2 file")]
        voucherdb[("Voucher data<br/>H2 in Cloud Run demo<br/>MySQL in local/cloudsql profile")]
    end

    users -->|"HTTPS"| frontend

    frontend -->|"HTTPS REST /auth/*<br/>JSON"| auth
    frontend -->|"HTTPS REST /api/products/*<br/>Bearer JWT"| inventory
    frontend -->|"HTTPS REST /wallet/*<br/>Bearer JWT"| wallet
    frontend -->|"HTTPS REST /orders/*<br/>Bearer JWT"| order
    frontend -->|"HTTPS REST /vouchers/active and /admin/vouchers/*<br/>public list or X-Admin-Token"| voucher

    order -->|"HTTP REST /api/products/inventory/*<br/>X-Internal-Token"| inventory
    order -->|"HTTP REST /wallet/deduct and /wallet/refund<br/>X-Internal-Token"| wallet
    order -->|"HTTP REST /vouchers/validate and /vouchers/claim<br/>X-Internal-Token"| voucher

    auth -->|"JPA / JDBC"| authdb
    inventory -->|"JPA / JDBC"| inventorydb
    wallet -->|"JPA / JDBC"| walletdb
    order -->|"JPA / JDBC"| orderdb
    voucher -->|"JPA / JDBC"| voucherdb
```

Short explanation:
The platform currently has one browser-based frontend container and five backend API containers: Auth/Profile, Inventory, Wallet, Order, and Voucher/Promo. The frontend calls each API directly over REST. The Order service acts as the checkout orchestrator and synchronously calls Inventory, Wallet, and Voucher/Promo with `X-Internal-Token` headers. Each service owns its own persistence, which follows the microservice data isolation principle even though most current demo deployments still use embedded H2 storage.

### Deployment Diagram

```mermaid
flowchart TB
    browser["User browser"]

    subgraph internet["Public internet"]
        browser
    end

    subgraph gcp["Google Cloud Run demo deployment (assumed from repo READMEs)"]
        frontendRun["Cloud Run service<br/>frontend"]
        authRun["Cloud Run service<br/>auth-profile-api"]
        inventoryRun["Cloud Run service<br/>inventory-api"]
        walletRun["Cloud Run service<br/>wallet-api"]
        orderRun["Cloud Run service<br/>order-api"]
        voucherRun["Cloud Run service<br/>voucher-promo-api"]
    end

    subgraph storage["Current service-local storage"]
        authStore["/tmp/auth-profile-db<br/>H2 file"]
        inventoryStore["/tmp/inventory-db<br/>H2 file"]
        walletStore["/tmp/wallet-db<br/>H2 file"]
        orderStore["/tmp/order-db or ./data/orderdb<br/>H2 file"]
        voucherStore["H2 in-memory in cloudrun profile<br/>or MySQL when cloudsql profile is activated"]
    end

    browser -->|"HTTPS"| frontendRun
    frontendRun -->|"HTTPS REST"| authRun
    frontendRun -->|"HTTPS REST"| inventoryRun
    frontendRun -->|"HTTPS REST"| walletRun
    frontendRun -->|"HTTPS REST"| orderRun
    frontendRun -->|"HTTPS REST"| voucherRun

    orderRun -->|"HTTP REST + X-Internal-Token"| inventoryRun
    orderRun -->|"HTTP REST + X-Internal-Token"| walletRun
    orderRun -->|"HTTP REST + X-Internal-Token"| voucherRun

    authRun --- authStore
    inventoryRun --- inventoryStore
    walletRun --- walletStore
    orderRun --- orderStore
    voucherRun --- voucherStore
```

Short explanation:
The current deployment is best understood as a set of independently deployed Cloud Run services, one per repository, plus one public frontend service. This view is an explicit assumption based on the deploy commands documented in the READMEs. The important current characteristic is that the runtime remains fragile: most services keep state in H2 files or in-memory storage attached to the service runtime instead of durable managed databases, and public traffic reaches backend services directly rather than through a single API gateway.

## Future Architecture

### Future Container Diagram

```mermaid
flowchart LR
    users["Web users<br/>(Admin, Titiper, Jastiper)"]

    subgraph jsonFuture["JSON platform - proposed future state"]
        direction LR
        frontendFuture["Frontend SPA<br/>React + Vite + CDN"]
        gateway["API Gateway / BFF<br/>single public API boundary"]

        authFuture["Auth/Profile API"]
        inventoryFuture["Inventory API"]
        walletFuture["Wallet API"]
        orderFuture["Order API / Saga orchestrator"]
        voucherFuture["Voucher/Promo API"]

        bus["Message broker / event bus<br/>for order lifecycle, retries, and async side effects"]
        observability["Central observability<br/>logs, metrics, tracing, alerts"]

        authDbFuture[("Managed Auth DB")]
        inventoryDbFuture[("Managed Inventory DB")]
        walletDbFuture[("Managed Wallet DB")]
        orderDbFuture[("Managed Order DB")]
        voucherDbFuture[("Managed Voucher DB")]
        hotCache["Redis cache<br/>hot catalog and active voucher reads"]
    end

    users -->|"HTTPS"| frontendFuture
    frontendFuture -->|"HTTPS REST"| gateway

    gateway -->|"JWT auth, rate limit, routing"| authFuture
    gateway -->|"JWT auth, rate limit, routing"| inventoryFuture
    gateway -->|"JWT auth, rate limit, routing"| walletFuture
    gateway -->|"JWT auth, rate limit, routing"| orderFuture
    gateway -->|"JWT auth, rate limit, routing"| voucherFuture

    orderFuture -->|"synchronous commands for critical reservation/payment checks"| inventoryFuture
    orderFuture -->|"synchronous commands for critical reservation/payment checks"| walletFuture
    orderFuture -->|"synchronous commands for critical reservation/payment checks"| voucherFuture

    orderFuture -->|"publish domain events"| bus
    inventoryFuture -->|"consume and publish stock events"| bus
    walletFuture -->|"consume and publish payment events"| bus
    voucherFuture -->|"consume and publish voucher events"| bus

    inventoryFuture -->|"cache hot reads"| hotCache
    voucherFuture -->|"cache active vouchers"| hotCache

    authFuture -->|"JPA / JDBC"| authDbFuture
    inventoryFuture -->|"JPA / JDBC"| inventoryDbFuture
    walletFuture -->|"JPA / JDBC"| walletDbFuture
    orderFuture -->|"JPA / JDBC"| orderDbFuture
    voucherFuture -->|"JPA / JDBC"| voucherDbFuture

    observability -.-> gateway
    observability -.-> authFuture
    observability -.-> inventoryFuture
    observability -.-> walletFuture
    observability -.-> orderFuture
    observability -.-> voucherFuture
```

### Future Deployment Diagram

```mermaid
flowchart TB
    browser["User browser"]

    subgraph edge["Public edge"]
        cdn["CDN + HTTPS load balancer"]
        gatewayRun["API Gateway / BFF<br/>multiple instances"]
    end

    subgraph gcpFuture["Cloud Run / managed services production environment"]
        frontendRunFuture["Frontend service<br/>multiple instances"]

        subgraph privateNet["Private service network"]
            authRunFuture["Auth/Profile service<br/>autoscaled"]
            inventoryRunFuture["Inventory service<br/>autoscaled"]
            walletRunFuture["Wallet service<br/>autoscaled"]
            orderRunFuture["Order service<br/>autoscaled"]
            voucherRunFuture["Voucher service<br/>autoscaled"]
            busFuture["Managed message broker"]
            obsFuture["Managed logs, metrics, tracing"]
            redisFuture["Redis cache"]
        end

        authDbRun[("Managed Auth DB<br/>backup + PITR")]
        inventoryDbRun[("Managed Inventory DB<br/>backup + PITR")]
        walletDbRun[("Managed Wallet DB<br/>backup + PITR")]
        orderDbRun[("Managed Order DB<br/>backup + PITR")]
        voucherDbRun[("Managed Voucher DB<br/>backup + PITR")]
    end

    browser -->|"HTTPS"| cdn
    cdn -->|"static assets"| frontendRunFuture
    cdn -->|"API calls"| gatewayRun

    gatewayRun -->|"private HTTPS"| authRunFuture
    gatewayRun -->|"private HTTPS"| inventoryRunFuture
    gatewayRun -->|"private HTTPS"| walletRunFuture
    gatewayRun -->|"private HTTPS"| orderRunFuture
    gatewayRun -->|"private HTTPS"| voucherRunFuture

    orderRunFuture -->|"private HTTPS"| inventoryRunFuture
    orderRunFuture -->|"private HTTPS"| walletRunFuture
    orderRunFuture -->|"private HTTPS"| voucherRunFuture
    orderRunFuture -->|"events"| busFuture
    inventoryRunFuture -->|"events"| busFuture
    walletRunFuture -->|"events"| busFuture
    voucherRunFuture -->|"events"| busFuture

    inventoryRunFuture --> redisFuture
    voucherRunFuture --> redisFuture

    authRunFuture --- authDbRun
    inventoryRunFuture --- inventoryDbRun
    walletRunFuture --- walletDbRun
    orderRunFuture --- orderDbRun
    voucherRunFuture --- voucherDbRun

    obsFuture -.-> gatewayRun
    obsFuture -.-> authRunFuture
    obsFuture -.-> inventoryRunFuture
    obsFuture -.-> walletRunFuture
    obsFuture -.-> orderRunFuture
    obsFuture -.-> voucherRunFuture
```

### Architectural Improvements

The future architecture addresses the main weaknesses of the current implementation:

- A single API Gateway or BFF becomes the only public API boundary, so browsers no longer call every backend directly.
- Each service moves from embedded H2 or in-memory runtime storage to a managed database with backup and point-in-time recovery.
- The checkout path remains synchronous only where immediate consistency is necessary, while non-critical side effects move to an event bus to reduce coupling and improve recovery behavior.
- Observability becomes a first-class architecture concern through centralized logs, metrics, tracing, and alerting.
- Hot reads such as active vouchers and frequently viewed catalog items can be cached to reduce repetitive database pressure during flash-sale or war traffic.
- Private service-to-service networking reduces exposure of internal APIs and makes token management easier to control.

### Trade-offs

- Introducing an API gateway and message broker adds operational complexity and deployment cost.
- Saga-style coordination and asynchronous events improve resilience, but they also increase consistency design work and debugging difficulty.
- Managed databases and Redis improve durability and scale, but they require schema migration discipline, backup operations, and capacity management.
- Centralized observability and secret management improve supportability and security, but they add platform dependencies that the team must maintain.

## Risk Storming Explanation

Risk storming is a structured architecture review technique used to identify risks by looking at a system from several quality perspectives such as availability, scalability, security, operability, and maintainability. Instead of discussing architecture only from a feature perspective, the team evaluates what could fail if the project becomes successful and receives much higher traffic than the milestone demo.

We applied risk storming to this project because the current repositories already show several production-sensitive decisions:

- every business capability is split into separate services,
- checkout depends on multiple synchronous downstream calls,
- the current demo deployment still relies heavily on embedded H2 storage,
- frontend traffic reaches backend services directly,
- there is no dedicated observability or integration backbone.

### Risk Matrix

| Risk Area | Impact | Likelihood | Score | Level | Explanation | Mitigation |
|---|---:|---:|---:|---|---|---|
| Data durability and recovery | 3 | 3 | 9 | High | Auth, Inventory, Wallet, and Order default to service-local H2 storage, while Voucher uses H2 in the documented Cloud Run profile. Instance loss or bad rollout can lose state or make recovery difficult. | Move every service to managed databases with backup, recovery, and migration discipline. |
| Checkout path availability | 3 | 3 | 9 | High | `Order` synchronously calls `Inventory`, `Wallet`, and `Voucher/Promo`. A slow or failing downstream service can block checkout entirely. | Keep only critical synchronous checks, add retries/timeouts/circuit breakers, and publish non-critical follow-up work through a broker. |
| Edge security and trust boundary | 3 | 2 | 6 | High | The frontend directly calls multiple public backend URLs. JWT verification is distributed through shared secrets, and Voucher admin/internal access depends on header tokens and custom filters. | Add an API gateway, private service networking, centralized secret management, and stricter token validation boundaries. |
| Scalability during war traffic | 3 | 2 | 6 | High | Flash-sale behavior concentrates load on Order, Inventory, Wallet, and Voucher quota checks. Active voucher reads and stock checks can spike at the same time. | Add autoscaling, cache hot reads, and use event-driven buffering for asynchronous work. |
| Observability and incident response | 2 | 3 | 6 | High | No repository documents centralized tracing, correlation IDs, or cross-service alerting. Failures across multiple services will be hard to diagnose quickly. | Add centralized logs, metrics, tracing, dashboards, and alert rules. |
| Deployment and configuration fragility | 2 | 2 | 4 | Medium | The repos rely on per-service environment variables and independent deploy commands. Token drift, CORS drift, or base URL mismatch can break the system even when each service still works individually. | Standardize deployment templates, secret storage, configuration contracts, and release checks. |

Scoring used:

- Impact: `1` low, `2` medium, `3` high
- Likelihood: `1` low, `2` medium, `3` high
- Score = `Impact x Likelihood`
- Level mapping: `1-2` low, `3-4` medium, `6-9` high

### Identified Risks in the Current Architecture

The current architecture is already functionally split by bounded context, but it still behaves like a tightly coupled demo environment in several important ways:

- data durability depends on per-service local storage instead of managed persistence,
- order checkout has no isolation from downstream service outages,
- public traffic reaches backend services directly instead of passing through a unified edge layer,
- security and operational behavior are configured separately in each repo,
- war-style traffic would stress the exact services that already sit in the critical checkout path.

### Consensus Result

The group consensus is that the highest-priority risks are:

1. data loss or inconsistent recovery after deployment/runtime failure,
2. checkout unavailability caused by the synchronous dependency chain,
3. weak edge control caused by direct frontend-to-service traffic,
4. poor diagnosability when failures span several services.

These were treated as the key risks because they directly affect user trust, transaction success, and the team's ability to operate the system once traffic grows.

### Mitigation Strategy

The proposed mitigation strategy is:

- keep microservice boundaries, but strengthen the public edge with an API gateway,
- replace embedded demo persistence with managed service-owned databases,
- add event-driven communication for asynchronous and compensating work,
- introduce centralized observability and alerting,
- put internal traffic on private networking and manage secrets centrally,
- add cache support for read-heavy paths such as active vouchers and popular catalog items.

### How the Mitigations Influenced the Future Architecture

The future architecture section is a direct consequence of the risk storming discussion:

- the API gateway exists because the current public surface is too fragmented,
- the event bus exists because the current checkout path is too tightly coupled,
- managed databases and backup strategy exist because service-local H2 storage is too fragile,
- Redis is introduced because war traffic will repeatedly hit the same catalog and voucher reads,
- observability is elevated into its own platform concern because the current repositories do not provide cross-service visibility by default.

## Individual Architecture Work

My individual responsibility: **Voucher Promo**

### Component Diagram

```mermaid
flowchart LR
    checkoutUi["Frontend checkout flow<br/>(frontend/src/pages/CheckoutPage.jsx)"]
    adminUi["Frontend admin console<br/>(frontend/src/pages/AdminPage.jsx)"]
    orderService["Order service<br/>(order.integration.VoucherClient)"]

    publicController["VoucherController<br/>GET /vouchers/active<br/>POST /vouchers/validate<br/>POST /vouchers/claim"]
    adminController["AdminVoucherController<br/>POST /admin/vouchers<br/>GET /admin/vouchers<br/>PUT /admin/vouchers/{id}<br/>POST /admin/vouchers/{id}/disable"]
    healthController["HealthController<br/>GET /health"]

    internalFilter["InternalTokenFilter<br/>protects /vouchers/validate and /vouchers/claim"]
    adminFilter["AdminTokenFilter<br/>protects /admin/*"]

    voucherService["VoucherService"]
    voucherPolicy["VoucherPolicy"]
    voucherRepo["VoucherRepository"]
    redemptionRepo["VoucherRedemptionRepository"]

    voucherDb[("Voucher database")]

    checkoutUi -->|"GET /vouchers/active"| publicController
    adminUi -->|"voucher admin requests + X-Admin-Token"| adminFilter
    orderService -->|"validate/claim + X-Internal-Token"| internalFilter

    internalFilter --> publicController
    adminFilter --> adminController

    publicController --> voucherService
    adminController --> voucherService
    healthController -->|"checks DB connectivity"| voucherDb

    voucherService --> voucherPolicy
    voucherService --> voucherRepo
    voucherService --> redemptionRepo

    voucherRepo --> voucherDb
    redemptionRepo --> voucherDb
```

This component diagram expands the **Voucher/Promo API** container from the group container diagram. In the group view, Voucher/Promo appears as one backend service. In this zoomed-in view, that service is decomposed into public/admin controllers, security filters, application service logic, policy validation logic, repositories, and the persistence layer that stores voucher definitions and voucher claims.

### Code Diagram 1 - Main Class and Module Relationships

```mermaid
classDiagram
    class VoucherController {
        +getActiveVouchers()
        +validateVoucher(request)
        +claimVoucher(request)
    }

    class AdminVoucherController {
        +createVoucher(request)
        +listVouchers(status)
        +editVoucher(id, request)
        +disableVoucher(id)
    }

    class VoucherService {
        +getActiveVouchers()
        +getAdminVouchers(status)
        +validateVoucher(request)
        +claimVoucher(request)
        +createVoucher(request)
        +editVoucher(id, request)
        +disableVoucher(id)
    }

    class VoucherPolicy {
        +normalizeCode(code)
        +validateVoucherDefinition(...)
        +ensureVoucherEditable(...)
        +validateVoucherUsability(...)
        +calculateDiscount(...)
    }

    class VoucherRepository {
        +findByCode(code)
        +findByCodeForUpdate(code)
        +findAllByOrderByCreatedAtDesc()
        +findByStatusOrderByCreatedAtDesc(status)
        +markExpiredVouchers(...)
    }

    class VoucherRedemptionRepository {
        +findByVoucherIdAndOrderId(voucherId, orderId)
        +save(redemption)
    }

    class InternalTokenFilter
    class AdminTokenFilter

    VoucherController --> VoucherService
    AdminVoucherController --> VoucherService
    VoucherService --> VoucherPolicy
    VoucherService --> VoucherRepository
    VoucherService --> VoucherRedemptionRepository
    InternalTokenFilter --> VoucherController
    AdminTokenFilter --> AdminVoucherController
```

### Code Diagram 2 - Voucher Persistence Model

```mermaid
erDiagram
    VOUCHERS {
        bigint id PK
        string code UK
        string discount_type
        decimal discount_value
        datetime start_at
        datetime end_at
        decimal min_spend
        int quota_total
        int quota_remaining
        string status
        bigint version
    }

    VOUCHER_REDEMPTIONS {
        bigint id PK
        bigint voucher_id FK
        string order_id
        bigint buyer_id
        decimal order_amount
        decimal discount_applied
        timestamp claimed_at
    }

    VOUCHERS ||--o{ VOUCHER_REDEMPTIONS : records
```

### Code Diagram 3 - Voucher Claim Flow

```mermaid
flowchart TD
    request["ClaimVoucherRequest"] --> controller["VoucherController.claimVoucher()"]
    controller --> service["VoucherService.claimVoucher()"]
    service --> normalize["VoucherPolicy.normalizeCode()"]
    service --> lock["VoucherRepository.findByCodeForUpdate()"]
    service --> existing["VoucherRedemptionRepository.findByVoucherIdAndOrderId()"]

    existing -->|"already exists"| idem["Return idempotent ClaimVoucherResponse"]
    existing -->|"not found"| validate["VoucherPolicy.validateVoucherUsability()"]

    validate -->|"invalid"| reject["Return failed ClaimVoucherResponse"]
    validate -->|"valid"| calc["VoucherPolicy.calculateDiscount()"]
    calc --> save["VoucherRedemptionRepository.save()"]
    save --> quota["voucher.setQuotaRemaining(quotaRemaining - 1)"]
    quota --> success["Return success ClaimVoucherResponse"]
```

### Code Diagram 4 - Security-Relevant Entry Paths

```mermaid
flowchart LR
    publicRead["frontend CheckoutPage"] -->|"GET /vouchers/active"| publicCtrl["VoucherController"]
    internalCaller["Order.integration.VoucherClient"] -->|"POST /vouchers/validate and /vouchers/claim<br/>X-Internal-Token"| internalGuard["InternalTokenFilter"]
    adminCaller["frontend AdminPage"] -->|"GET/POST/PUT /admin/vouchers*<br/>X-Admin-Token"| adminGuard["AdminTokenFilter"]

    internalGuard --> publicCtrl
    adminGuard --> adminCtrl["AdminVoucherController"]
```

These code diagrams map directly to the Voucher Promo source code:

- `VoucherController`, `AdminVoucherController`, `VoucherService`, `VoucherPolicy`, `VoucherRepository`, and `VoucherRedemptionRepository` are defined under `Voucher-Promo/backend/src/main/java/com/example/demo/voucher/...`
- `InternalTokenFilter` and `AdminTokenFilter` are defined under `Voucher-Promo/backend/src/main/java/com/example/demo/security/...`
- the `Voucher` and `VoucherRedemption` tables come from the JPA entities and Flyway migrations in `Voucher-Promo/backend/src/main/resources/db/migration/...`
- the external admin and checkout callers shown in the diagrams map to `frontend/src/pages/AdminPage.jsx`, `frontend/src/pages/CheckoutPage.jsx`, and `Order/backend/src/main/java/id/ac/ui/cs/advprog/order/integration/VoucherClient.java`

Together, these diagrams show that my individual work is centered on voucher validation, voucher claiming, quota protection, admin voucher lifecycle management, and the persistence rules needed to keep voucher usage correct under repeated or concurrent checkout requests.

### Bonus Additional Module View - Inventory

The following bonus diagrams expand the Inventory service as an additional architecture view. This module is a useful bonus subject because it sits directly on the checkout path and it already contains explicit stock-protection logic for war-like purchase contention.

#### Inventory Component Diagram

```mermaid
flowchart LR
    buyerUi["Frontend catalog and product detail pages"]
    jastiperUi["Frontend jastiper product management flow"]
    adminUiInv["Admin monitoring flow"]
    orderSvcInv["Order service<br/>(InventoryClient)"]

    productController["ProductController"]
    jwtFilterInv["JwtAuthenticationFilter"]
    internalFilterInv["InternalTokenAuthenticationFilter"]

    productService["ProductService"]
    productMapper["ProductMutationMapper"]
    productRepository["ProductRepository"]
    inventoryDb["Inventory database"]

    buyerUi -->|"GET /api/products/search<br/>GET /api/products/{id}"| jwtFilterInv
    jastiperUi -->|"POST/PUT/DELETE /api/products*<br/>GET /api/products/me"| jwtFilterInv
    adminUiInv -->|"GET /api/products<br/>PUT/DELETE /api/products/admin/*"| jwtFilterInv
    orderSvcInv -->|"GET /api/products/inventory/{id}<br/>PATCH reduce/restore-stock<br/>X-Internal-Token"| internalFilterInv

    jwtFilterInv --> productController
    internalFilterInv --> productController

    productController --> productService
    productService --> productMapper
    productService --> productRepository
    productRepository --> inventoryDb
```

This bonus component diagram expands the **Inventory API** container from the group-level container diagram. The important architectural point is that Inventory is not only a catalog CRUD service. It also acts as a guarded stock authority for checkout, with one access path for browser traffic and another path for internal service-to-service stock mutation requested by `Order`.

#### Inventory Code Diagram 1 - Main Class Relationships

```mermaid
classDiagram
    class ProductController {
        +createProduct(request, authentication)
        +updateOwnProduct(productId, request, authentication)
        +deleteOwnProduct(productId, authentication)
        +listMyProducts(authentication)
        +searchByProduct(keyword)
        +searchByJastiper(jastiperId)
        +getProductById(productId)
        +monitorAllProducts()
        +adminUpdateProduct(productId, request)
        +adminDeleteProduct(productId)
        +reserveStock(productId, request)
        +reduceStock(request)
        +restoreStock(request)
    }

    class ProductService {
        +create(request, jastiperId)
        +listOwnedBy(jastiperId)
        +searchByProductName(keyword)
        +listByJastiper(jastiperId)
        +listAll()
        +updateOwnedProduct(productId, request, actorId)
        +deleteOwnedProduct(productId, actorId)
        +adminUpdateProduct(productId, request)
        +adminDeleteProduct(productId)
        +reserveStock(productId, quantity)
        +getById(productId)
        +restoreStock(productId, quantity)
    }

    class ProductMutationMapper {
        +fromCreateRequest(request, jastiperId)
        +applyUpdate(product, request)
    }

    class ProductRepository {
        +findAllByJastiperId(jastiperId)
        +searchByName(keyword)
        +findByIdForUpdate(id)
        +save(product)
        +saveAndFlush(product)
    }

    class Product

    ProductController --> ProductService
    ProductService --> ProductMutationMapper
    ProductService --> ProductRepository
    ProductRepository --> Product
```

#### Inventory Code Diagram 2 - Stock Reservation and War-Protection Flow

```mermaid
flowchart TD
    requestInv["ReserveStockRequest"] --> controllerInv["ProductController.reduceStock() or reserveStock()"]
    controllerInv --> serviceInv["ProductService.reserveStock()"]
    serviceInv --> validateQty["reject quantity <= 0"]
    serviceInv --> lockRow["ProductRepository.findByIdForUpdate()"]
    lockRow --> checkStock["compare requested quantity with available stock"]
    checkStock -->|"insufficient"| insufficient["throw InsufficientStockException"]
    checkStock -->|"enough"| decrement["product.setStock(available - quantity)"]
    decrement --> flush["productRepository.saveAndFlush(product)"]
    flush -->|"optimistic locking failure"| warConflict["throw WarConflictException"]
    flush -->|"success"| successInv["return updated Product"]
```

These Inventory code diagrams map directly to the current source files:

- `Inventory/src/main/java/id/ac/ui/cs/advprog/inventory/controller/ProductController.java`
- `Inventory/src/main/java/id/ac/ui/cs/advprog/inventory/service/ProductService.java`
- `Inventory/src/main/java/id/ac/ui/cs/advprog/inventory/service/ProductMutationMapper.java`
- `Inventory/src/main/java/id/ac/ui/cs/advprog/inventory/repository/ProductRepository.java`
- `Inventory/src/main/java/id/ac/ui/cs/advprog/inventory/model/Product.java`

Architecturally, this bonus expansion is valuable for four reasons:

1. It shows the exact place where overselling protection is implemented: `findByIdForUpdate()` combined with guarded stock mutation inside `reserveStock()`.
2. It makes the security split visible: browser-originated requests enter through JWT-based role checks, while checkout-originated requests enter through the internal-token path.
3. It shows that Inventory owns both product metadata and stock consistency, so it is a business-critical state holder rather than a passive data service.
4. It exposes the current consistency trade-off clearly: stock mutation is still synchronous in the request path, which is straightforward for correctness but keeps Inventory on the critical latency path for each checkout.

#### Inventory Architectural Interpretation

From an architecture-review perspective, the Inventory module reinforces several conclusions from the risk storming section:

- **Availability sensitivity:** if Inventory becomes unavailable, both catalog browsing and checkout stock reservation degrade immediately.
- **Scalability sensitivity:** war traffic can produce concentrated lock contention around a small number of hot products.
- **Coupling sensitivity:** `Order` depends on Inventory synchronously for product snapshots and stock mutation, so Inventory failures propagate into checkout failures.
- **Data integrity sensitivity:** the service contains the main business rule that prevents negative stock, making it one of the core correctness boundaries in the current system.

For that reason, Inventory is not only a supporting module. In the present architecture it behaves as one of the core reliability boundaries of the whole platform.

## Individual Architecture Work
 
My individual responsibility: **Order**
 
### Component Diagram
 
```mermaid
flowchart LR
    checkoutUi["Frontend checkout flow\n(frontend/src/pages/CheckoutPage.jsx)"]
    ordersUi["Frontend order history\n(frontend/src/pages/OrdersPage.jsx)"]
    jastiperUi["Frontend jastiper view\n(frontend/src/pages/JastiperOrdersPage.jsx)"]
    adminUi["Frontend admin monitor\n(frontend/src/pages/AdminPage.jsx)"]
 
    orderController["OrderController\nPOST /orders/checkout\nGET /orders/my\nGET /orders/my/active\nGET /orders/{id}\nPATCH /orders/{id}/status\nPOST /orders/{id}/cancel\nPOST /orders/{id}/rating\nGET /orders/jastiper\nGET /orders/admin"]
 
    jwtFilter["JwtAuthenticationFilter\nvalidates Bearer JWT\nenforces ROLE_TITIPER / ROLE_JASTIPER / ROLE_ADMIN"]
 
    orderService["OrderService\ncheckout orchestration\nlifecycle transitions\ncancel with refund\nrating"]
 
    prepService["CheckoutPreparationService\nvalidate request\nfetch product snapshots\nvalidate voucher\ncalculate totals"]
 
    compService["CheckoutCompensationService\nrefund wallet\nrestore stock"]
 
    inventoryClient["InventoryClient\nGET /api/products/inventory/{id}\nPATCH reduce-stock\nPATCH restore-stock"]
    walletClient["WalletClient\nPOST /wallet/balance\nPOST /wallet/deduct\nPOST /wallet/refund"]
    voucherClient["VoucherClient\nPOST /vouchers/validate\nPOST /vouchers/claim"]
 
    orderRepo["OrderRepository"]
    orderItemRepo["OrderItemRepository"]
    ratingRepo["RatingRepository"]
    idemRepo["IdempotencyRecordRepository"]
 
    orderDb[("Order database\norders\norder_items\nratings\nidempotency_records")]
 
    checkoutUi -->|"POST /orders/checkout\nIdempotency-Key header"| jwtFilter
    ordersUi -->|"GET /orders/my\nGET /orders/my/active\nGET /orders/{id}"| jwtFilter
    jastiperUi -->|"GET /orders/jastiper\nPATCH /{id}/status\nPOST /{id}/cancel"| jwtFilter
    adminUi -->|"GET /orders/admin\nPATCH /{id}/status\nPOST /{id}/cancel"| jwtFilter
 
    jwtFilter --> orderController
    orderController --> orderService
 
    orderService --> prepService
    orderService --> compService
    orderService --> orderRepo
    orderService --> orderItemRepo
    orderService --> ratingRepo
    orderService --> idemRepo
 
    prepService --> inventoryClient
    prepService --> voucherClient
    compService --> walletClient
    compService --> inventoryClient
    orderService --> walletClient
 
    orderRepo --> orderDb
    orderItemRepo --> orderDb
    ratingRepo --> orderDb
    idemRepo --> orderDb
```
 
This component diagram expands the **Order API** container from the group container diagram. The Order service acts as the checkout orchestrator for the whole platform. It is responsible for coordinating product snapshot reads from Inventory, wallet balance checks and deductions from Wallet, voucher validation and quota claims from Voucher/Promo, and persisting the resulting order and its line items. It is the only service in the system that calls three other services in a single request path.
 
The service also manages the full order lifecycle after checkout: status transitions driven by jastiper and buyer roles, cancel with automatic wallet refund and stock restoration, and buyer rating submission after order completion. An `IdempotencyRecordRepository` is wired into the checkout path to prevent duplicate orders on network retry or accidental double-submit from the frontend.

### Code Diagram 1 — Main Class and Module Relationships
 
```mermaid
classDiagram
    class OrderController {
        +checkout(auth, idempotencyKey, request)
        +myOrders(auth)
        +myActiveOrders(auth)
        +jastiperOrders(auth)
        +adminOrders(auth)
        +detail(auth, orderId)
        +updateStatus(auth, orderId, request)
        +cancel(auth, orderId)
        +rating(auth, orderId, request)
    }
 
    class OrderService {
        +checkout(buyerId, idempotencyKey, request)
        +listMyOrders(buyerId)
        +listActiveOrders(buyerId)
        +listJastiperOrders(jastiperId)
        +listAdminOrders()
        +getDetail(orderId, actorId, isAdmin)
        +updateStatus(orderId, actorId, isAdmin, isJastiper, nextStatus)
        +cancel(orderId, actorId, isAdmin, isJastiper)
        +rate(orderId, buyerId, request)
    }
 
    class CheckoutPreparationService {
        +prepare(request)
        +claimVoucher(code, orderId, subtotal, buyerId)
    }
 
    class CheckoutCompensationService {
        +compensate(order, buyerId, totalPaid, walletDeducted, reducedItems)
    }
 
    class OrderRepository {
        +findByBuyerIdOrderByCreatedAtDesc(buyerId)
        +findByJastiperIdOrderByCreatedAtDesc(jastiperId)
        +findByBuyerIdAndStatusNotInOrderByCreatedAtDesc(buyerId, excluded)
        +findAllByOrderByCreatedAtDesc()
    }
 
    class OrderItemRepository {
        +findByOrderId(orderId)
    }
 
    class RatingRepository {
        +findByOrderId(orderId)
    }
 
    class IdempotencyRecordRepository {
        +findByIdemKey(idemKey)
    }
 
    class InventoryClient {
        +getProduct(productId)
        +reduceStock(productId, quantity)
        +restoreStock(productId, quantity)
    }
 
    class WalletClient {
        +getBalance(userId)
        +deduct(userId, orderId, amount)
        +refund(userId, orderId, amount)
    }
 
    class VoucherClient {
        +validate(code, subtotal)
        +claim(code, orderId, subtotal, buyerId)
    }
 
    OrderController --> OrderService
    OrderService --> CheckoutPreparationService
    OrderService --> CheckoutCompensationService
    OrderService --> OrderRepository
    OrderService --> OrderItemRepository
    OrderService --> RatingRepository
    OrderService --> IdempotencyRecordRepository
    OrderService --> WalletClient
    CheckoutPreparationService --> InventoryClient
    CheckoutPreparationService --> VoucherClient
    CheckoutCompensationService --> WalletClient
    CheckoutCompensationService --> InventoryClient
```
### Code Diagram 2 — Order Persistence Model
 
```mermaid
erDiagram
    ORDERS {
        bigint id PK
        bigint buyer_id
        bigint jastiper_id
        string status
        string shipping_address
        decimal subtotal
        decimal discount_total
        decimal total_paid
        string voucher_code
        string failure_reason
        boolean refund_done
        timestamp created_at
        timestamp updated_at
    }
 
    ORDER_ITEMS {
        bigint id PK
        bigint order_id FK
        string product_id
        string product_name_snapshot
        decimal unit_price_snapshot
        int qty
        decimal line_total
    }
 
    RATINGS {
        bigint id PK
        bigint order_id FK
        bigint buyer_id
        int product_rating
        int jastiper_rating
        string comment
        timestamp created_at
    }
 
    IDEMPOTENCY_RECORDS {
        bigint id PK
        string idem_key UK
        bigint buyer_id
        string endpoint
        string request_hash
        bigint order_id
        timestamp created_at
    }
 
    ORDERS ||--o{ ORDER_ITEMS : contains
    ORDERS ||--o| RATINGS : has
    ORDERS ||--o| IDEMPOTENCY_RECORDS : tracks
```
### Code Diagram 3 — Checkout Flow with Idempotency
 
```mermaid
flowchart TD
    req["CheckoutRequest + Idempotency-Key header"] --> controller["OrderController.checkout()"]
    controller --> idemCheck["IdempotencyRecordRepository.findByIdemKey()"]
 
    idemCheck -->|"key exists, orderId not null"| returnExisting["return existing OrderDetailResponse\nno re-processing"]
    idemCheck -->|"key exists, orderId null"| conflict["throw CONFLICT\nCheckout already in progress"]
    idemCheck -->|"key not found"| saveIdem["save IdempotencyRecord with orderId = null"]
 
    saveIdem --> prepare["CheckoutPreparationService.prepare()\nfetch product snapshot from Inventory\nvalidate stock\nvalidate voucher with VoucherClient\ncalculate subtotal, discount, totalPaid"]
 
    prepare --> balanceCheck["WalletClient.getBalance()\ncompare with totalPaid"]
 
    balanceCheck -->|"insufficient"| rejectWallet["throw WALLET_INSUFFICIENT"]
    balanceCheck -->|"sufficient"| persist["save Order as PENDING\nsave OrderItems"]
 
    persist --> deduct["WalletClient.deduct()"]
    deduct --> reduce["InventoryClient.reduceStock() for each item"]
    reduce --> claim["CheckoutPreparationService.claimVoucher()\nif voucherCode present"]
 
    claim -->|"claim failed"| compensate["CheckoutCompensationService.compensate()\nrefund Wallet\nrestore stock\nmark Order as FAILED"]
    claim -->|"claim success"| paid["mark Order as PAID\nupdate IdempotencyRecord orderId"]
 
    paid --> response["return OrderDetailResponse"]
```

### Code Diagram 4 — Order Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> PENDING : checkout initiated

    PENDING --> PAID : wallet deducted\nvoucher claimed\nstock reduced
    PENDING --> FAILED : any step fails\ncompensation applied

    PAID --> PURCHASED : JASTIPER or ADMIN advances
    PAID --> CANCELLED : JASTIPER or ADMIN cancels\nwallet refunded\nstock restored

    PURCHASED --> SHIPPED : JASTIPER or ADMIN advances
    PURCHASED --> CANCELLED : JASTIPER or ADMIN cancels\nno refund at this stage

    SHIPPED --> COMPLETED : TITIPER confirms receipt\nor ADMIN advances

    COMPLETED --> COMPLETED : TITIPER submits rating\n(product + jastiper, 1-5)

    CANCELLED --> CANCELLED : cancel call is idempotent\nno re-processing
    FAILED --> FAILED : terminal state
    COMPLETED --> COMPLETED : terminal state
```
### Code Diagram 5 — Role-Based Access per Endpoint

```mermaid
flowchart LR
    titiper["ROLE_TITIPER"]
    jastiper["ROLE_JASTIPER"]
    admin["ROLE_ADMIN"]

    checkout["POST /orders/checkout"]
    myOrders["GET /orders/my"]
    myActive["GET /orders/my/active"]
    detail["GET /orders/{id}"]
    jastiperOrders["GET /orders/jastiper"]
    adminOrders["GET /orders/admin"]
    status["PATCH /orders/{id}/status"]
    cancel["POST /orders/{id}/cancel"]
    rating["POST /orders/{id}/rating"]

    titiper --> checkout
    titiper --> myOrders
    titiper --> myActive
    titiper --> detail
    titiper --> status
    titiper --> rating

    jastiper --> jastiperOrders
    jastiper --> detail
    jastiper --> status
    jastiper --> cancel

    admin --> adminOrders
    admin --> detail
    admin --> status
    admin --> cancel
```

These code diagrams map directly to the Order service source files:

- `OrderController` is defined at `Order/backend/src/main/java/id/ac/ui/cs/advprog/order/controller/OrderController.java`
- `OrderService`, `CheckoutPreparationService`, and `CheckoutCompensationService` are at `Order/backend/src/main/java/id/ac/ui/cs/advprog/order/service/`
- `InventoryClient`, `WalletClient`, and `VoucherClient` are at `Order/backend/src/main/java/id/ac/ui/cs/advprog/order/integration/`
- `Order`, `OrderItem`, `Rating`, and `IdempotencyRecord` entities are at `Order/backend/src/main/java/id/ac/ui/cs/advprog/order/entity/`
- All four repositories are at `Order/backend/src/main/java/id/ac/ui/cs/advprog/order/repository/`
- The external callers shown in the component diagram map to `frontend/src/pages/CheckoutPage.jsx`, `frontend/src/pages/OrdersPage.jsx`, `frontend/src/pages/JastiperOrdersPage.jsx`, and `frontend/src/pages/AdminPage.jsx`

Together these diagrams show that my individual work is centered on checkout orchestration across three downstream services, full order lifecycle management, cancel with idempotent refund, dual-dimension buyer rating, and retry-safe checkout through an idempotency key mechanism.
