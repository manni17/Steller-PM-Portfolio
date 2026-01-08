# Steller Launch PR/FAQ

## Press Release
**FOR IMMEDIATE RELEASE**

**Steller Unveils "One-Click" Integration Platform, Reducing Global Gift Card Retail Onboarding from Weeks to Minutes**

**TORONTO, ON — March 1, 2026** — Steller, the new digital middleware platform for the fintech supply chain, today announced the launch of its unified integration engine. Steller empowers digital retailers to instantly access and sell thousands of global gift card brands without building complex, custom API connections.

For years, digital wallets and fintech apps struggled to offer diverse rewards catalogs. Integrating with global providers like Bamboo required weeks of engineering effort, complex legal compliance, and managing fragile data synchronization.

"Retailers told us they wanted to sell products, not manage infrastructure," said Muhanad Abdelrahim, Product Lead at Steller. "With our new platform, we've abstracted the complexity of cross-border supply chains. A retailer can now connect to Steller and instantly access a curated, synchronized catalog of 2,500+ global brands. We handle the inventory, the compliance, and the connectivity."

The platform introduces a "Lean Architecture" design that prioritizes stability and data integrity. By containerizing the entire supply chain logic, Steller ensures that retailers experience zero downtime during high-traffic events like Black Friday.

"Before Steller, adding a new region's gift cards took us three sprints," said Sarah Chen, CTO of PayGlobal (a beta customer). "Now, we just toggle a switch in the Steller Admin Dashboard, and the products appear in our app instantly. It’s the difference between building a bridge and just driving across one."

Steller is available today for enterprise partners. For more information, visit [dashboard link].

### ###

## Internal FAQ (Frequently Asked Questions)

### Strategic Questions

**Q: Why did we build a middleware layer instead of letting retailers connect directly to providers like Bamboo?**
Direct integration puts the burden of maintenance on the retailer. Every time Bamboo changes their API, every retailer breaks. Steller acts as an "Adapter Pattern" for the industry. We absorb the complexity of upstream changes so our customers don't have to. This allows us to aggregate volume and negotiate better rates (Economies of Scale).

**Q: How do we measure success for this launch?**
Our primary metric is **"Time to First Transaction" (TTFT)**. We aim to reduce the time from a partner signing up to selling their first card from 14 days to <24 hours. Secondary metrics include **API Error Rate** (target <0.1%) and **Catalog Sync Latency** (target <5 mins).

### Technical Questions

**Q: The system uses RabbitMQ for order processing. Is that over-engineering for an MVP?**
A: Initially, yes. However, we retained it to ensure **Transactional Reliability**. When a customer pays for a card, we cannot risk losing that order if the upstream provider (Bamboo) is down. RabbitMQ allows us to queue orders and retry them asynchronously ("Store and Forward"), guaranteeing that no customer loses money due to network blips. This builds trust, which is our core product.

**Q: You mentioned "Lean Architecture." What tradeoff did you make?**
A: We consolidated the deployment strategy. The initial prototype was a fragmented set of microservices that required complex orchestration. We refactored this into a unified Docker Compose architecture.
*   **Tradeoff:** We sacrificed the theoretical ability to scale the "User Service" independently of the "Order Service" (micro-scaling).
*   **Benefit:** We gained massive operational simplicity. A single engineer can now deploy the entire stack in 5 minutes, eliminating "Environment Drift" bugs.

**Q: How are we handling security for sensitive data like PINs?**
A: We identified that early versions stored PINs in plain text. For launch, we have implemented a "Blind Storage" policy where sensitive fields are encrypted at the application layer before touching the database. Additionally, we have removed all hardcoded secrets from the codebase, replacing them with runtime environment injection.

**Q: What is the biggest risk to the launch?**
A: **Dependency Risk.** We rely entirely on the Bamboo API for inventory. If they blacklist our IP or change their schema without notice, our catalog goes dark. To mitigate this, we are building a "Cache & Serve" layer that keeps the last known good state of the catalog available even if the upstream connection is temporarily lost.