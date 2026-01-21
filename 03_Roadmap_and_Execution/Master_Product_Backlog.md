# Master Product Backlog

**Project:** Steller Integration Platform
**Owner:** Muhanad Abdelrahim
**Status:** Live Tracking
**Versioning:** `v1.0-RC` (Release Candidate) | `v1.1` (Growth)

---

## 🟣 Module Z: DevOps & Stability (Architecture)
**Goal:** Achieve "One-Command Start" and eliminate configuration fragility.

| ID | Feature | User Story | Priority | Release | Status | Done |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **IA-01** | **Self-Initializing Container** | As a Developer, I want the API container to automatically wait for the DB and apply migrations on startup so that I don't need manual host commands. | **P0** | `v1.0-RC` | `[✅]` | 2026-01-19 |
| **IA-02** | **Config Normalization** | As a DevOps Engineer, I want a single source of truth for configuration (Env Vars) to eliminate conflicts between `appsettings` and `docker-compose`. | **P0** | `v1.0-RC` | `[✅]` | 2026-01-19 |
| **BUG-01** | **Migration Bundle** | Fix "Missing Tables" crash by generating a self-contained EF Core migration bundle during the Docker build process. | **P0** | `v1.0-RC` | `[✅]` | 2026-01-19 |
| **BUG-02** | **JWT Key Validator** | Fix recurrent "HS512 Key Size" crash by implementing startup validation or increasing default key size in all environment templates. | **P1** | `v1.0-RC` | `[✅]` | 2026-01-19 |
| **BUG-03** | **Response Wrapper** | Fix "Wallet Not Found" and "Null Token" errors in integration tests by aligning client-side DTOs with the API's global `ServiceResponse` wrapper. | **P0** | `v1.0-RC` | `[✅]` | 2026-01-19 |
| **BUG-04** | **Test Data Seeding** | Implement `SeedOrder` endpoint to allow integration tests to inject "Pending" orders for fulfillment testing without depending on the Consumer API. | **P0** | `v1.0-RC` | `[✅]` | 2026-01-19 |

## 🟡 Module I: Integration & Verification (The "Last Mile")
**Goal:** Verify end-to-end wiring between Consumer, Queue, and Backend.

| ID | Feature | User Story | Priority | Release | Status | Dependencies |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **SP-01** | **End-to-End Verification** | **SPIKE:** Establish a successful "Hello World" flow from Frontend -> StellerConsumer -> RabbitMQ -> Steller.Api to verify "Last Mile" wiring. | **P0** | `v1.0-RC` | `[✅]` | 2026-01-21 | Blocked by: None |
| **OM-05** | **Consumer API Ingestion** | As a Partner System, I want to hit the `StellerConsumer` API endpoint and have it successfully publish a message to RabbitMQ. | **P0** | `v1.0-RC` | `[✅]` | 2026-01-21 | Blocked by: SP-01 |

## 🟢 Module A: Partner Management (B2B Core)
**Goal:** Enable secure onboarding and revenue configuration for B2B partners (Digital Wallets).

| ID | Feature | User Story | Priority | Release | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **PM-01** | **Partner Onboarding** | As an Admin, I want to create a new Partner entity with logo, contact details, and API credentials. | **P0** | `v1.0` | `[ ]` |
| **PM-02** | **Revenue Share Config** | As an Admin, I want to configure a **Commission Percentage** (e.g., 2%) for each partner wallet to automate revenue recognition. | **P0** | `v1.0` | `[ ]` |
| **PM-03** | **Wallet Ledger** | As a Partner, I want to view my transaction history and current credit balance to reconcile my accounts. | **P1** | `v1.0` | `[ ]` |
| **PM-04** | **API Key Management** | As a Partner, I want to generate/rotate my `X-Api-Key` to maintain security compliance. | **P1** | `v1.1` | `[ ]` |

## 🔵 Module B: Global Catalog & Risk Control
**Goal:** Synchronize inventory with upstream providers while preventing revenue leakage.

| ID | Feature | User Story | Priority | Release | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **GC-01** | **Auto-Sync Service** | As a System, I want to poll the Bamboo API every 30 mins to update product availability and face values. | **P0** | `v1.0` | `[ ]` |
| **GC-02** | **Price Guardrails** | As a Product Owner, I want the system to **automatically disable** products if the Supplier Cost > Selling Price (Negative Margin Protection). | **P0** | `v1.0` | `[ ]` |
| **GC-03** | **Manual Overrides** | As an Admin, I want to manually hide specific brands or denominations from the catalog during maintenance windows. | **P2** | `v1.1` | `[ ]` |
| **GC-04** | **FX Rate Buffer** | As an Admin, I want to set a currency conversion buffer (e.g., +1%) to protect against intraday volatility. | **P1** | `v1.1` | `[ ]` |

## 🟠 Module C: Order Management (Transaction Engine)
**Goal:** Ensure high-reliability processing of financial transactions.

| ID | Feature | User Story | Priority | Release | Status | Done |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **OM-01** | **Backend Fulfillment Engine** | As a System, I want to process a "Pending" order from the DB and successfully capture funds via the Bamboo API. | **P0** | `v1.0-RC` | `[✅]` | 2026-01-19 |
| **OM-02** | **Resilient Processing** | As a System, I want to retry failed upstream orders (3x at 30s intervals) via RabbitMQ to handle transient network failures. | **P0** | `v1.0-RC` | `[✅]` | 2026-01-20 |
| **OM-03** | **Blind PIN Storage** | As a Security Engineer, I want to ensure sensitive Card PINs are never stored in plain text in the database (Encryption at Rest). | **P0** | `v1.0` | `[ ]` | |
| **OM-04** | **Order Status Webhook** | As a Partner, I want to receive a webhook notification when an Async order transitions to `COMPLETED`. | **P2** | `v1.1` | `[ ]` | |

## ⚪ Module E: Quality & Reliability (Systemic Debt)
**Goal:** Ensure long-term stability via automated regression and environment parity.

| ID | Feature | User Story | Priority | Release | Status | Done |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **QA-01** | **Black-Box Suite** | As a PM, I want a scriptable test runner to verify core flows (Login/Order) without manual interaction. | **P0** | `v1.0-RC` | `[✅]` | 2026-01-19 |
| **QA-02** | **Service Mocks** | As a System, I want to simulate external dependencies (Bamboo API) to ensure deterministic test results. | **P1** | `v1.0-RC` | `[✅]` | 2026-01-20 |
| **QA-03** | **Data Seeding** | As a Developer, I want automated "Fresh Data" injection before each test run to prevent state pollution. | **P1** | `v1.0-RC` | `[✅]` | 2026-01-19 |

---

## 📅 Roadmap Phases

### Phase 1: Reliability & Engine Verification (v1.0-RC)
*   **Status:** ✅ **COMPLETED**
*   **Deliverables:** Self-Healing Docker, 100% Test Pass Rate (Backend), Idempotency Fix.

### Phase 2: Integration & The "Last Mile" (v1.0-RC2)
*   **Status:** ✅ **COMPLETED**
*   **Focus:** SP-01, OM-05.
*   **Goal:** Verify "Front Door" connectivity (Consumer -> RabbitMQ -> Backend).

### Phase 3: MVP Feature Complete (v1.0-Launch)
*   **Focus:** PM-01, GC-01, OM-03.
*   **Goal:** Security Hardening (Blind PINs) and Catalog Sync.

### Phase 4: Enterprise Growth (v1.1)
*   **Focus:** OM-04 (Webhooks), GC-04 (FX Buffer).
*   **Goal:** High-volume scale and partner autonomy.