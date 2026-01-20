# Steller QA Platform Specification: The "Black Box" Environment

**Status:** Draft
**Owner:** Product Engineering
**Date:** January 12, 2026

## 1. Executive Summary
To ensure the stability of the Steller Platform without relying on manual regression or unstable external dependencies, we are establishing a comprehensive **Automated QA Environment**. This system will utilize Service Virtualization (Mocking) and Test Data Management (TDM) to execute deterministic, end-to-end user workflows.

## 2. Testing Architecture

### The "Toyota" Test Stack
1.  **Test Runner (The Driver):** `Steller.Tests.Integration` (xUnit).
2.  **System Under Test (The Car):** `Steller.Api` & `StellerConsumer.Api` running in Docker.
3.  **Virtual Services (The Road):** 
    *   **Bamboo Mock:** A WireMock container simulating the external Gift Card provider.
    *   **Postgres (Test DB):** A transient database instance for data isolation.

### Data Flow
`[xUnit]` -> `[API Gateway]` -> `[RabbitMQ]` -> `[Consumer Service]` -> `[WireMock (Bamboo)]`

## 3. Test Data Management (TDM) Strategy

### A. Synthetic User Generation
*   **Problem:** Hardcoded users (like `ahmed.khtim`) cause state collisions (e.g., "Wallet Empty" failures).
*   **Solution:** Every test run generates a unique "Ephemeral User."
    *   *Pattern:* `testuser_{guid}@auto.steller.com`
    *   *Lifecycle:* Register -> Deposit Funds -> Execute Test -> Teardown (optional).

### B. Credential Persistence (Artifact Logging)
*   **Requirement:** Developers must be able to debug failed tests using the exact user account that failed.
*   **Implementation:** The Test Runner will append created credentials to a local artifact file: `test_artifacts/session_{date}_users.json`.
    ```json
    {
      "test_case": "Order_HappyPath",
      "user_email": "testuser_8f9a2@auto.steller.com",
      "password": "Password123!",
      "token": "eyJ..."
    }
    ```

## 4. Service Virtualization (Mocking Strategy)

### Mock Bamboo API
We will deploy a `wiremock/wiremock` container to replace the real Bamboo API. It must strictly validate incoming requests against the Bamboo Schema.

| Endpoint | Behavior | Logic |
| :--- | :--- | :--- |
| `POST /api/v3/orders` | **200 OK** | Returns success if payload matches schema and ProductId exists. |
| `POST /api/v3/orders` | **400 Bad Request** | Returns error if `RequestId` is duplicated (Idempotency check). |
| `POST /api/v3/orders` | **503 Service Unavailable** | Simulates upstream outage to test RabbitMQ retry logic. |
| `GET /api/v3/catalog` | **200 OK** | Returns a static, deterministic list of products for testing. |

## 5. Scenario Matrix (Scope)

### Core Workflows (P0)
1.  **Identity:** Register New User -> Login -> Verify Token Claims.
2.  **Wallet:** Add Funds (Admin) -> Check Balance (User) -> Verify Update.
3.  **Commerce:** Place Order (Sufficient Funds) -> Mock Bamboo Success -> Deduct Wallet -> Verify Status "Completed".

### Resilience Workflows (P1)
1.  **Insufficient Funds:** Place Order -> Fail Validation -> Verify Balance Unchanged.
2.  **Upstream Failure:** Place Order -> Mock Bamboo 500 -> Verify Order Status "Queued/Retrying" (Not Failed).
3.  **Double Spend:** Send duplicate `RequestId` -> Verify only one transaction recorded.

## 6. Implementation Plan (Next Steps)
1.  **Infrastructure:** Add `wiremock` service to `docker-compose.yml`.
2.  **Code:** Create `DataSeedingFixture.cs` in the Test Project.
3.  **Config:** Point Steller API `BambooUrl` to the WireMock container in `.env`.
