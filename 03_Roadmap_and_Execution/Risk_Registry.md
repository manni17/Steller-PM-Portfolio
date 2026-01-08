# Risk Registry & Mitigation Plan

**Project:** Steller Integration Platform
**Author:** Muhanad Abdelrahim
**Status:** Active Mitigation

---

## 🛠️ Risk Management Matrix

This document identifies potential threats to the Steller platform and defines the proactive strategies implemented to ensure system resilience and financial integrity.

| ID | Risk Category | Description | Impact | Probability | Mitigation Strategy (The "How") |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **R-01** | **Dependency** | Upstream provider (Bamboo API) downtime or severe latency. | High | Medium | **Asynchronous Queueing:** Orders are accepted and stored in RabbitMQ immediately. A 3-retry policy handles transient failures without user intervention. |
| **R-02** | **Financial** | Price Volatility: Supplier cost exceeds selling price before catalog sync. | High | Low | **Price Guardrail Service:** A validation check ensures `SupplierCost < SellingPrice` before order fulfillment. Failure triggers an automatic "Kill Switch" for the item. |
| **R-03** | **Security** | PII/Secret Leakage: Exposure of Card PINs or Partner API Keys via DB leak. | Critical | Low | **Encryption at Rest:** Implementation of "Blind Storage" policy. Sensitive fields are encrypted at the application layer before database persistence. |
| **R-04** | **Regulatory** | AML/Fraud: Exploitation of gift card liquidity for money laundering. | Critical | Low | **Velocity Limits:** Enforced rate limiting and volume caps per partner. Transactions exceeding $10k/day trigger manual audit flags. |
| **R-05** | **Operational** | Environment Drift: Discrepancies between local dev and production environments. | Medium | High | **Infrastructure as Code:** Unified Docker Compose deployment ensures dev/prod parity and eliminates "works on my machine" bottlenecks. |

---

## 🛡️ Mitigation Deep Dive

### 1. The "Price Guardrail" (Financial Risk)
To prevent revenue leakage, the system does not trust stale catalog data for real-time transactions. During the order fulfillment phase (Consumer API), the system performs a final validation of the current margin. If the margin is negative due to a supplier price hike, the transaction is halted, and a "Price Anomaly" alert is sent to the Admin Dashboard.

### 2. Transactional Reliability (Dependency Risk)
Leveraging the **Adapter Pattern** and **MassTransit/RabbitMQ**, Steller decouples the customer's request from the supplier's response. This "Store and Forward" mechanism ensures that even if Bamboo is undergoing maintenance, the customer's intent is captured and fulfilled automatically once connectivity is restored.

### 3. Security Hardening (Information Risk)
Moving away from the original prototype's plain-text storage, the platform now utilizes environment-injected secrets. All sensitive identifiers (PINs) are treated as "Black Box" data—Steller facilitates the delivery but does not "know" the PIN in a readable format within the persistence layer.
