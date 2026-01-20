# Bamboo Integration - Mock TODO List & Quality Plan

## 1. Test Scenarios (Core Workflows)

### A. Authentication Handshake ("Login")
*   **Goal:** Verify Steller can successfully authenticate with Bamboo using Basic Auth.
*   **Method:** Perform a low-risk GET request (e.g., `GetAccounts`).
*   **Success Criteria:** Returns HTTP 200 and a list of accounts.
*   **Failure Criteria:** Returns HTTP 401 (Unauthorized) or 403 (Forbidden).

### B. Order Fulfillment Cycle ("Order")
*   **Goal:** Validate the end-to-end flow of purchasing a digital voucher.
*   **Steps:**
    1.  **Pre-check:** `GetAccounts` to retrieve a valid `AccountId` (Balance Check).
    2.  **Action:** `PlaceOrder` with a unique `RequestId`.
    3.  **Verification:** `GetOrder` using the `RequestId` to confirm status is `Created`, `Processing`, or `Succeeded`.
    4.  **Idempotency:** Retry `PlaceOrder` with the same `RequestId` to ensure it returns the *same* result (not a duplicate charge).

## 2. Critical API Endpoints (Integration Surface)

| Operation | Method | Endpoint | Use Case |
| :--- | :--- | :--- | :--- |
| **Get Accounts** | `GET` | `/api/integration/v1.0/accounts` | Retrieve `AccountId` and Balance. Required before ordering. |
| **Get Catalog (V2)** | `GET` | `/api/integration/v2.0/catalog` | Sync Products/Brands. (Low frequency). |
| **Place Order** | `POST` | `/api/integration/v1.0/orders/checkout` | Purchase Assets. High Criticality. |
| **Get Order** | `GET` | `/api/integration/v1.0/orders/checkout/{requestId}` | Polling for status updates. |

## 3. Automated Test Suite Plan (xUnit)

### Structure
*   **Project:** `Steller.Tests.Integration`
*   **Framework:** xUnit + FluentAssertions + RestSharp (for real HTTP calls).
*   **Configuration:** `appsettings.test.json` (Holds Sandbox Credentials).

### Test Cases
1.  `Authentication_Returns_200_With_Valid_Credentials`
2.  `GetAccounts_Returns_NonEmpty_List`
3.  `PlaceOrder_Creates_New_Order_Successfully`
4.  `GetOrder_Returns_Correct_Status_For_Existing_Order`

## 4. Implementation Checklist
- [ ] Create `BambooClientTests.cs` in `Steller.Tests.Integration`.
- [ ] Add `RestSharp` client setup with Basic Auth.
- [ ] Implement `GetAccounts` test.
- [ ] Implement `PlaceOrder` test (using random RequestId).
- [ ] Implement `GetOrder` assertion logic.
- [ ] CI/CD Pipeline Config (Pending Phase 3).
