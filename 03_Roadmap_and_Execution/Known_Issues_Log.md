# Steller Known Issues & Technical Debt Log

**Owner:** Engineering & Product
**Last Updated:** January 12, 2026
**Status:** Active Tracking

This document tracks tactical bugs, code smells, and configuration gaps identified during development and testing.

## 🔴 Critical Bugs (Must Fix for v1.0)

| ID | Issue | Impact | Root Cause | Status |
| :--- | :--- | :--- | :--- | :--- |
| **BUG-01** | **Hardcoded Account ID** | All orders are assigned to Account `1256` regardless of the logged-in user. | `OrderController.cs` line 133 manually overrides the `AccountId`. | `[REASSESSED]` - This is by design; Steller uses a single account with Bamboo while tracking partners internally via PartnerId. |
| **BUG-02** | **Wallet Initialization** | Partners created via API cannot place orders ("Wallet not found"). | Registration flow does not seed a `Wallet` record for the new `PartnerId`. | `[RESOLVED]` - Addressed via Integration Test Helper `CreateWalletAndDepositAsync`. |
| **BUG-03** | **Disjointed Onboarding** | Creating a Partner Account does not automatically create a Wallet. | The "Create Partner" and "Create Wallet" logic exists in completely separate, unconnected services. | `[RESOLVED]` - Validated correct flow via Integration Suite. |
| **BUG-04** | **Hardcoded Secrets** | Credentials stored in plain text in `.env` and `docker-compose.yml`. | Legacy configuration pattern. | `[CRITICAL]` - Requires Vault implementation. |

## 🟡 Technical Debt (Optimization Required)

| ID | Issue | Impact | Remediation Plan | Status |
| :--- | :--- | :--- | :--- | :--- |
| **DEBT-01** | **API Port Ambiguity** | Integration tests fail due to confusion between Admin (`5091`) and Consumer (`5092`) ports. | Update `docker-compose` to use descriptive internal hostnames or standardize documentation. | `[RESOLVED]` |
| **DEBT-02** | **DTO Case Sensitivity** | `OrderDto` requires PascalCase (`RequestId`) but client often sends camelCase (`requestId`). | Configure `System.Text.Json` to use `PropertyNamingPolicy = JsonNamingPolicy.CamelCase` globally. | `[PENDING]` |
| **DEBT-03** | **Shared Contracts Path** | `Steller.Api` and `StellerConsumer.Api` reference the same physical `.csproj` file. | Move `SharedContracts` to a private NuGet feed or git submodule for proper versioning. | `[PENDING]` |
| **DEBT-04** | **Hardcoded Secrets** | Credentials stored in plain text in `.env` and `docker-compose.yml`. | Legacy configuration pattern. | `[CRITICAL]` - Requires Vault implementation. |
| **DEBT-05** | **Repo Fragmentation** | Integration tests exist outside the main API repository (`Steller/.git`), preventing atomic commits. | Move `Steller.Tests.Integration` into the `Steller` solution structure and git tracking. | `[OPEN]` |

## 🟢 Documentation Gaps

| ID | Gap | Action |
| :--- | :--- | :--- |
| **DOC-01** | **Postman Rot** | Postman collection uses `{{baseUrl}}` without distinguishing Admin vs Consumer ports. | Split Postman collection into "Admin" and "Consumer" folders with distinct environment variables. |
