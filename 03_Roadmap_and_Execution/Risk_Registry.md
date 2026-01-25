# Risk Registry & Mitigation Plan

**Project:** Steller Integration Platform
**Author:** Muhanad Abdelrahim
**Status:** Active Mitigation

---

## 🛠️ Risk Management Matrix

This document identifies potential threats to the Steller platform and defines the proactive strategies implemented to ensure system resilience and financial integrity.

| ID | Risk Category | Description | Impact | Probability | Mitigation Strategy (The "How") |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **R-01** | **Human Capital** | **Knowledge Silo / Bus Factor:** The platform was built by a single developer in a high-risk zone (Sudan). Loss of this resource would render the codebase unmaintainable. | Critical | High | **AI-Driven Knowledge Extraction:** Utilized AI Agents to reverse-engineer the codebase, generate system maps, and establish a "Local Twin" environment, decoupling business continuity from specific personnel. |
| **R-02** | **Dependency** | Upstream provider (Bamboo API) downtime or severe latency. | High | Medium | `[MITIGATED]` **1. Technical:** RabbitMQ "Store and Forward" for transient failures. <br> **2. Strategic:** Multi-sourcing strategy (Supplier B & C integration in progress). <br> **3. Operational:** Buffer Stock policy for top 2 high-volume SKUs to bypass API dependency during outages. |
| **R-03** | **Financial** | Price Volatility: Supplier cost exceeds selling price before catalog sync. | High | Low | **Price Guardrail Service:** A validation check ensures `SupplierCost < SellingPrice` before order fulfillment. Failure triggers an automatic "Kill Switch" for the item. |
| **R-04** | **Security** | PII/Secret Leakage: Exposure of Card PINs or Partner API Keys via DB leak. | Critical | Low | **Encryption at Rest:** Implementation of "Blind Storage" policy. Sensitive fields are encrypted at the application layer before database persistence. |
| **R-05** | **Regulatory** | AML/Fraud: Exploitation of gift card liquidity for money laundering. | Critical | Low | **Velocity Limits:** Enforced rate limiting and volume caps per partner. Transactions exceeding $10k/day trigger manual audit flags. |
| **R-06** | **Operational** | Environment Drift: Discrepancies between local dev and production environments. | Medium | High | `[MITIGATED]` **Infrastructure as Code:** Unified Docker Compose deployment ensures dev/prod parity and eliminates "works on my machine" bottlenecks. |
| **R-07** | **Operational** | **Infrastructure Drift:** Dockerfiles becoming stale due to code refactoring (e.g., moved projects). | Medium | High | **"Green Light" Path Check:** Explicit verification of `COPY` paths in Dockerfiles against the file system before build recommendations. |
| **R-08** | **Technical** | **Schema/Code Mismatch:** Code expecting columns that don't exist yet, causing runtime crashes. | High | Medium | **Schema-First Verification:** Explicit `dotnet ef migrations list` check before feature implementation to ensure data layer readiness. |

---

## 🛡️ Mitigation Deep Dive

### 1. Operational Resilience (Human Capital Risk)
Facing extreme volatility in the development team (due to geopolitical instability), we mitigated the "Bus Factor" by treating **AI as the Interim Tech Lead**.
*   **Reverse Engineering:** Used AI agents to inspect the live VPS and map the undocumented architecture.
*   **Local Replication:** Created a "Digital Twin" of the production environment on a local machine, ensuring the business retains ownership of the IP regardless of developer availability.
*   **Automated Documentation:** Generated comprehensive API schemas and System Diagrams to lower the barrier to entry for future hires from months to days.

### 2. The "Price Guardrail" (Financial Risk)
To prevent revenue leakage, the system does not trust stale catalog data for real-time transactions. During the order fulfillment phase (Consumer API), the system performs a final validation of the current margin. If the margin is negative due to a supplier price hike, the transaction is halted, and a "Price Anomaly" alert is sent to the Admin Dashboard.

### 3. Transactional Reliability (Dependency Risk)
Leveraging the **Adapter Pattern** and **MassTransit/RabbitMQ**, Steller decouples the customer's request from the supplier's response. This "Store and Forward" mechanism ensures that even if Bamboo is undergoing maintenance, the customer's intent is captured and fulfilled automatically once connectivity is restored.

### 4. Security Hardening (Information Risk)
Moving away from the original prototype's plain-text storage, the platform now utilizes environment-injected secrets. All sensitive identifiers (PINs) are treated as "Black Box" data—Steller facilitates the delivery but does not "know" the PIN in a readable format within the persistence layer.