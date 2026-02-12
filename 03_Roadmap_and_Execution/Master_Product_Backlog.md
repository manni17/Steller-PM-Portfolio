### 📝 The Corrected "Pro" Backlog (v2.2)

I have inserted the missing items (marked with `🆕`) to bridge the gap between "It compiles" and "It's a Simulation."

#### 🚀 ACTIVE SPRINT: The Data Layer & Simulation Prep

**Goal:** Ingest Catalog and ensure the "Mass Order" simulation doesn't crash the DB.

| ID | Feature | User Story | Priority | Status |
| --- | --- | --- | --- | --- |
| **IA-03** | **Ref Data Seeder** `🆕` | As System, ensure `TransactionTypes` (Credit/Debit) & `SystemUser` exist on startup. | **P0** | `[✅]` |
| **GC-01** | **Auto-Sync Service** | As System, poll Bamboo Mock (`GET /catalog`) and upsert to DB. | **P0** | `[✅]` |
| **PM-02** | **Revenue Share Config** | As Admin, method to assign `Product X` to `Partner Y` with `5% Margin`. | **P0** | `[✅]` |
| **QA-02** | **Load Generator** `🆕` | As QA, script to fire 500 simultaneous orders for Partner X. | **P1** | `[✅]` |

#### 🔮 ON DECK: The Partner Experience (Module D)

**Goal:** Visual Verification.

| ID | Feature | User Story | Priority | Status |
| --- | --- | --- | --- | --- |
| **UX-01** | **Partner Portal Grid** | As Partner, see my specific assigned products and prices. | **P0** | `[ ]` |
| **UX-02** | **Reactive Balance** | As Partner, balance updates immediately after Mass Order batch. | **P0** | `[ ]` |
| **IA-04** | **Config Cleanup** `🆕` | Move `1256` and `PartnerId` logic to `appsettings.json`. | **P2** | `[ ]` |

