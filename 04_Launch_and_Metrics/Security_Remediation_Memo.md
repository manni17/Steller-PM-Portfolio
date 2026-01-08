# Security Remediation Roadmap & Information Assurance Protocol

**To:** Engineering Leadership & Stakeholders
**From:** Muhanad Abdelrahim, Technical Product Manager
**Date:** January 7, 2026
**Subject:** Remediation Strategy for Critical Security Vulnerabilities (Audit Response)

---

## 1. Executive Summary
Following our internal security audit of the Steller v1.0 Release Candidate, we have identified **3 Critical** and **2 High** severity vulnerabilities. This memo outlines our immediate remediation plan to achieve compliance with Fintech Information Assurance standards before the production rollout.

**Strategic Goal:** Transition from "Functional" security to a **"Defense in Depth"** architecture.

## 2. Vulnerability Impact Analysis & Remediation Plan

### 🚨 Critical Severity: Plain-Text PIN Storage
*   **Observation:** Card PINs and Serial Numbers are currently stored in plain text in the `OrderCards` PostgreSQL table.
*   **Risk:** A database breach would result in immediate, irrevocable financial loss and massive reputational damage.
*   **Remediation:** Implement **Column-Level Encryption** (AES-256) at the application layer. The database will only store the encrypted ciphertext. Keys will be managed via a dedicated Key Management Service (KMS).
    *   *Status:* 🟡 In Development (Sprint 4).

### 🚨 Critical: Hardcoded Secrets in Configuration
*   **Observation:** API Keys and Database Credentials were found in `.env` files or hardcoded in configuration providers.
*   **Risk:** Credential harvesting by bad actors or accidental exposure in source control history.
*   **Remediation:** Transition to **Runtime Secret Injection**. All secrets are now pulled from a secure vault at container startup. Git history has been purged of legacy secrets.
    *   *Status:* ✅ **Completed** (Docker Config Updated).

### ⚠️ High Severity: Message Broker Credential Exposure
*   **Observation:** RabbitMQ is running with default `guest/guest` credentials on an exposed port.
*   **Risk:** Unauthorized access to the transaction queue, allowing an attacker to inject fraudulent orders or disrupt the fulfillment service.
*   **Remediation:** Disable the default `guest` user. Create specific service accounts (e.g., `steller_producer`, `steller_consumer`) with least-privilege permissions scoped to specific exchanges/queues.
    *   *Status:* 📅 Scheduled for Pre-Launch Hardening.

## 3. Implementation Timeline

| Phase | Action Item | Target Date | Priority |
| :--- | :--- | :--- | :--- |
| **Phase 1 (Immediate)** | Clean Git History & Purge Secrets | Jan 10 | P0 |
| **Phase 1 (Immediate)** | Enable "Blind Storage" (AES-256 PIN Encryption) | Jan 12 | P0 |
| **Phase 2 (Pre-Launch)** | Hardening RabbitMQ & Postgres Service Accounts | Jan 15 | P1 |
| **Phase 3 (Post-Launch)** | Implement Internal mTLS for Microservice Comms | Feb 01 | P2 |

## 4. Conclusion
We are prioritizing **Confidentiality** and **Integrity** over feature velocity for the current sprint. The "Go Live" approval is contingent upon the successful validation of the Phase 1 and 2 remediation items.
