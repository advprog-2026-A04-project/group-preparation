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
