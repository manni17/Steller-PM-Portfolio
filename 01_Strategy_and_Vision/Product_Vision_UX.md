# Product Vision & UX Strategy: The B2B Partner Journey

**Context:** Bridging the gap between Customer Acquisition (Steller.org) and Technical Execution (API).
**Primary Persona:** The "Developer-Buyer" (CTOs and Lead Engineers at Digital Wallets).

---

## 1. The Acquisition Funnel (Steller.org)
**Objective:** High-intent lead generation.
*   **The Problem:** Competitors require weeks of sales calls just to see documentation.
*   **The Steller Solution:** "Developer-First" transparency.
*   **Key UX Elements:**
    *   **Live API Sandbox:** A mocked terminal on the landing page showing a real `200 OK` response.
    *   **Value Proposition:** "Connect once, sell 2,500+ brands."
    *   **Call to Action (CTA):** "Get API Keys Now" (Self-Service) vs. "Contact Sales" (Gatekeeper).

## 2. The "First Mile" Experience (Partner Portal)
**Objective:** Minimize "Time to First Transaction" (TTFT).
*   **Strategy:** The **Vue.js Consumer Dashboard** is designed as a "Cockpit" for developers.
*   **Critical User Flows:**
    1.  **Credential Generation:** One-click generation of `X-Api-Key` and `Secret`.
    2.  **Wallet Management:** Visual "Top-Up" interface to load prepaid funds.
    3.  **Traffic Monitoring:** Real-time graphs showing API latency and success rates (Transparency builds trust).

## 3. The Operational Surface (Internal Admin)
**Objective:** Efficiency at Scale.
*   **Strategy:** Empower support staff to solve issues without engineering intervention.
*   **Key Capabilities:**
    *   **Revenue Configuration:** A simple slider interface to set "Commission %" per partner (Visualizing the backend feature `PM-02`).
    *   **Catalog Override:** A "Kill Switch" toggle for specific brands during upstream outages (Visualizing `GC-03`).
    *   **Order Forensics:** A detailed "Trace View" showing the exact RabbitMQ message payload for debugging failed orders.

## 4. Design Principles
*   **Latency is the UX:** For an API product, speed *is* the interface. The dashboard must load in <500ms to reflect the speed of the API.
*   **Fail Gracefully:** If the Bamboo API is down, the dashboard must clearly say "Upstream Maintenance" rather than throwing a generic `500 Error`.
