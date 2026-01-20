# Master Product Backlog

**Project:** Steller Integration Platform
**Owner:** Muhanad Abdelrahim
**Status:** Live Tracking

---

## 🟢 Module A: Partner Management (B2B Core)
**Goal:** Enable secure onboarding and revenue configuration for B2B partners (Digital Wallets).

| ID | Feature | User Story | Priority | Status |
| :--- | :--- | :--- | :--- | :--- |
| **PM-01** | **Partner Onboarding** | As an Admin, I want to create a new Partner entity with logo, contact details, and API credentials. | **P0** | `[ ]` |
| **PM-02** | **Revenue Share Config** | As an Admin, I want to configure a **Commission Percentage** (e.g., 2%) for each partner wallet to automate revenue recognition. | **P0** | `[ ]` |
| **PM-03** | **Wallet Ledger** | As a Partner, I want to view my transaction history and current credit balance to reconcile my accounts. | **P1** | `[ ]` |
| **PM-04** | **API Key Management** | As a Partner, I want to generate/rotate my `X-Api-Key` to maintain security compliance. | **P1** | `[ ]` |

## 🔵 Module B: Global Catalog & Risk Control
**Goal:** Synchronize inventory with upstream providers while preventing revenue leakage.

| ID | Feature | User Story | Priority | Status |
| :--- | :--- | :--- | :--- | :--- |
| **GC-01** | **Auto-Sync Service** | As a System, I want to poll the Bamboo API every 30 mins to update product availability and face values. | **P0** | `[ ]` |
| **GC-02** | **Price Guardrails** | As a Product Owner, I want the system to **automatically disable** products if the Supplier Cost > Selling Price (Negative Margin Protection). | **P0** | `[ ]` |
| **GC-03** | **Manual Overrides** | As an Admin, I want to manually hide specific brands or denominations from the catalog during maintenance windows. | **P2** | `[ ]` |
| **GC-04** | **FX Rate Buffer** | As an Admin, I want to set a currency conversion buffer (e.g., +1%) to protect against intraday volatility. | **P1** | `[ ]` |

## 🟠 Module C: Order Management (Transaction Engine)
**Goal:** Ensure high-reliability processing of financial transactions.

| ID | Feature | User Story | Priority | Status |
| :--- | :--- | :--- | :--- | :--- |
| **OM-01** | **API Order Placement** | As a Partner System, I want to POST an order and receive an immediate `201 Created` (Async) response. | **P0** | `[🟡]` |
| **OM-02** | **Resilient Processing** | As a System, I want to retry failed upstream orders (3x at 30s intervals) via RabbitMQ to handle transient network failures. | **P0** | `[ ]` |
| **OM-03** | **Blind PIN Storage** | As a Security Engineer, I want to ensure sensitive Card PINs are never stored in plain text in the database (Encryption at Rest). | **P0** | `[ ]` |
| **OM-04** | **Order Status Webhook** | As a Partner, I want to receive a webhook notification when an Async order transitions to `COMPLETED`. | **P2** | `[ ]` |

## ⚪ Module E: Quality & Reliability (Systemic Debt)
**Goal:** Ensure long-term stability via automated regression and environment parity.

| ID | Feature | User Story | Priority | Status |
| :--- | :--- | :--- | :--- | :--- |
| **QA-01** | **Black-Box Suite** | As a PM, I want a scriptable test runner to verify core flows (Login/Order) without manual interaction. | **P0** | `[✅]` |
| **QA-02** | **Service Mocks** | As a System, I want to simulate external dependencies (Bamboo API) to ensure deterministic test results. | **P1** | `[ ]` |
| **QA-03** | **Data Seeding** | As a Developer, I want automated "Fresh Data" injection before each test run to prevent state pollution. | **P1** | `[ ]` |

---

## 📅 Roadmap Phases

### Phase 1: MVP & Reliability (Launch Candidate)
*   Focus: PM-01, OM-01, QA-01, IA-01.
*   Goal: Stabilized environment with automated "Happy Path" verification.

### Phase 2: Revenue & Risk (Optimization)
*   Focus: PM-02, GC-02, GC-01.
*   Goal: Automate margin protection and catalog updates.

### Phase 3: Enterprise Scale
*   Focus: OM-04, IA-02.
*   Goal: Support high-volume partners with webhooks and RBAC.
