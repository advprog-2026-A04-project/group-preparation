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
