# Steller Platform - Product Management Showcase

**Project:** Steller Integration Middleware
**Role:** Technical Product Manager
**Owner:** Muhanad Abdelrahim

---

## 🚀 Executive Summary
Steller is a unified middleware platform that simplifies the global supply chain for digital gift cards. This portfolio showcases the **Technical Product Management** lifecycle, from identifying architectural bottlenecks to delivering a stable, scalable "One-Click" solution for B2B retailers.

**Business Goal:** Reduce partner onboarding time from **2 weeks** to **<24 hours**.

## 📂 Portfolio Artifacts

### 1. [Strategy & Vision](01_Strategy_and_Vision/Steller_PR_FAQ.md)
*   **The "Amazon" PR/FAQ:** A working backwards document defining the customer value proposition and answering difficult strategic questions before writing code.
*   **Key Insight:** Identifying that "Trust" (reliability) is more valuable to our customers than "Speed" (feature quantity).

### 2. [Technical Architecture & Trade-offs](02_Technical_Architecture/Strategic_Tradeoff_Memo.md)
*   **The "Ferrari vs. Toyota" Decision:** A strategic memo documenting the choice to move from a complex microservices architecture to a unified, reliable Docker-based deployment.
*   **Metric:** Reduced environment startup time by **85%** (15 mins -> 2 mins).

### 3. Roadmap & Execution (Coming Soon)
*   **User Stories:** Breakdown of "Happy Path" requirements for Admin and Partner personas.
*   **Risk Registry:** Mitigation strategies for Upstream API dependencies (Bamboo).

### 4. Launch & Metrics (Coming Soon)
*   **Quality Assurance:** Results from "Happy Path" validation testing.
*   **Launch Dashboard:** SQL queries used to monitor the "Time to First Transaction" KPI.

## 💻 Source Code
*   **Backend Middleware:** [manni17/steller-backend](https://github.com/manni17/steller-backend) - Core API, Event Bus, and Database.
*   **Consumer Service:** [manni17/steller-consumer](https://github.com/manni17/steller-consumer) - Order Fulfillment Microservice.

---

## 🛠️ Tech Stack & Context
*   **Core:** .NET 9, PostgreSQL 16, RabbitMQ (for async order processing).
*   **Frontend:** Vue.js (Admin Dashboard).
*   **Infrastructure:** Docker Compose (Infrastructure as Code).

## 💡 How to Read This Showcase
Start with the **PR/FAQ** to understand *Why* we built this. Then read the **Trade-off Memo** to see *How* we made difficult technical decisions to ensure launch success.
