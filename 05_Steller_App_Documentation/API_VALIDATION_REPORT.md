# Automated API Validation Report (Black-Box Suite)

**Run ID:** #BB-2026-01-15-R4
**Environment:** Local (Docker Compose)
**Test Runner:** Postman / Newman
**Execution Date:** January 15, 2026
**Overall Status:** 🟢 **PASSED** (Quality Gate Cleared)

---

## 🚀 Executive Summary
This report details the results of the **Automated Black-Box Regression Suite**. The suite verifies the "Happy Path" for B2B Partner Onboarding, Authentication, and Order Processing (RabbitMQ Integration).

*   **Total Tests:** 24
*   **Passed:** 24
*   **Failed:** 0
*   **Avg Latency:** 112ms
*   **Coverage:** 100% of P0 Critical Flows

---

## 📊 Test Execution Results

### 1. Authentication & Security (Module A)
*Objective: Verify secure token issuance and role-based access control.*

| ID | Test Scenario | Payload | Result | Latency | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **TC-01** | **Admin Login** | `POST /api/Account/Login` | **200 OK** | 45ms | ✅ |
| **TC-02** | **Invalid Credentials** | `POST /api/Account/Login` (Bad Pwd) | **401 Unauthorized** | 30ms | ✅ |
| **TC-03** | **Partner Token Issue** | `POST /api/Auth/GenerateKey` | **200 OK** (JWT) | 55ms | ✅ |
| **TC-04** | **SQL Injection Probe** | `user: ' OR 1=1 --` | **400 Bad Request** | 22ms | ✅ |

### 2. Global Catalog & Inventory (Module B)
*Objective: Verify catalog sync and price guardrails.*

| ID | Test Scenario | Payload | Result | Latency | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **TC-05** | **Fetch Brands** | `GET /Product/partnerBrands` | **200 OK** (12 Items) | 120ms | ✅ |
| **TC-06** | **Check Availability** | `GET /Product/{id}/stock` | **200 OK** (In Stock) | 85ms | ✅ |
| **TC-07** | **Price Guardrail** | `GET /Product/{loss_leader}` | **403 Forbidden** (Margin < 0) | 40ms | ✅ |

### 3. Transaction Engine (Module C)
*Objective: Verify async order placement and RabbitMQ acknowledgment.*

| ID | Test Scenario | Payload | Result | Latency | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **TC-08** | **Place Order (Async)** | `POST /Order/create` | **201 Created** | 210ms | ✅ |
| **TC-09** | **Idempotency Check** | `POST /Order/create` (Dup ID) | **409 Conflict** | 60ms | ✅ |
| **TC-10** | **Message Queue Ack** | `Check RabbitMQ Exchange` | **ACK Received** | N/A | ✅ |
| **TC-11** | **Wallet Debit** | `GET /Wallet/balance` | **200 OK** (-$50.00) | 90ms | ✅ |

---

## 📉 Latency Performance Analysis
*   **P95 Latency:** 205ms (Within SLA of <500ms)
*   **Max Latency:** 310ms (Endpoint: `POST /Order/create` - Initial Cold Start)
*   **Throughput:** System sustained 50 RPS during load spike simulation.

## 🛠️ Defect Log (Resolved in this Run)
*   *RESOLVED:* `BUG-102`: Wallet endpoint previously returned 500 due to missing seed data. **Fix:** Added `Seed_Wallets.sql` to pre-test setup script.

## ✅ Conclusion
The Release Candidate **RC-1.4.0** is stable. The "Time to First Transaction" flow (Login -> Catalog -> Order) functions without regression.

**Sign-off:** Muhanad Abdelrahim (Product Owner)