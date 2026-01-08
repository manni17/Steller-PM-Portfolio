# Architectural Decision Record: "Ferrari vs. Toyota" Deployment Strategy

**Date:** 2026-01-07
**Status:** Accepted
**Author:** Muhanad Abdelrahim

## Context
The Steller platform was originally architected as a highly distributed set of microservices (Identity, Catalog, Ordering, Payment) designed for theoretical hyper-scale ("The Ferrari"). 

While powerful, this architecture introduced significant friction during the early stages of development and partner onboarding:
*   **High Cognitive Load:** Developers needed to run 5+ separate terminal commands to start the environment.
*   **Fragility:** "It works on my machine" issues were common due to version drift between local and dev environments.
*   **Operational Overhead:** Debugging a failed order required tracing logs across 4 different containers/services.

## Decision
We decided to pivot to a **"Toyota" Architecture**: A unified, reliable, and "boring" deployment strategy using a consolidated Docker Compose configuration.

### Key Changes
1.  **Unified Orchestration:** Replaced manual startup scripts with a single `docker-compose up` command that bootstraps the database, message broker (RabbitMQ), and all API services in dependency order.
2.  **Secret Injection:** Moved from hardcoded settings to environment variable injection (`.env`), ensuring no secrets are committed to version control.
3.  **Network Isolation:** Placed all services on a private Docker network (`steller_network`), exposing only the necessary ports (API Gateway) to the host.

## Consequences (Trade-offs)

| Trade-off | Description | Impact |
| :--- | :--- | :--- |
| **Sacrificed: Granular Scaling** | We can no longer scale the "User Service" independently of the "Order Service" easily. | **Low Risk:** Current volume (<10k TPS) does not require granular scaling. Vertical scaling of the Docker host is sufficient. |
| **Gained: Developer Velocity** | Environment startup time reduced from 15 minutes to 2 minutes. | **High Value:** Faster iteration loops for feature development. |
| **Gained: Reliability** | "Works on my machine" ensures it works in Production. | **High Value:** Eliminated deployment surprises during partner demos. |

## Conclusion
This decision prioritizes **Stability** and **Developer Experience** over **Theoretical Scalability**. We need a car that starts every time (Toyota) before we need a car that goes 200mph (Ferrari).