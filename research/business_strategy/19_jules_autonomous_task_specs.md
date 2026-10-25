# Google Jules Autonomous Task Specifications & Execution Prompts
## Engineering & Commercial Execution Prompts for Asynchronous Agent Runs

This document contains production-ready, highly specified task prompts formatted for dispatching directly into **Google Jules** (or any autonomous agent orchestration system). Each task is structured with explicit context, source files, acceptance criteria, test assertions, and architectural boundaries.

---

### Task 1: Automated Zero-Downtime Migration Engine (Zoho & QuickBooks Importer)

```yaml
Task Title: "feat(migration): build automated zero-downtime ETL importer for Zoho Books and QuickBooks Online"
Labels: ["enhancement", "migration", "agent-task"]
Assigned Agent: "Jules"
```

#### Task Description & Context:
To execute the competitive displacement playbook detailed in `research/business_strategy/16_competitive_displacement_zoho_odoo_netsuite.md`, we must eliminate the friction of migrating off Zoho Books and QuickBooks Online. We need a modular ETL ingestion pipeline that connects via OAuth, streams historical financial and transactional data, normalizes disparate account taxonomies into our unified PostgreSQL General Ledger, and runs a parallel validation pass.

#### Required Deliverables:
1. **Connector Interfaces (`src/migration/connectors/`):**
   * `ZohoBooksConnector`: Implements pagination, rate-limit throttling (backoff on HTTP 429), and extraction of:
     * Chart of Accounts (COA)
     * Customers and Vendors (Entity directory)
     * Historical Journal Entries and Invoices with line-item detail
     * Item master with inventory valuation method (FIFO/Moving Average)
   * `QuickBooksConnector`: Ingests QBO CDC (Change Data Capture) or batch endpoints.
2. **Schema Normalization Layer (`src/migration/normalizers/`):**
   * Deterministic mapping engine translating Zoho/QBO account types into our standard GAAP/IFRS Ledger schema (`research/benchmarks/01_core_finance_ledger.md`).
3. **Shadow-Run Ledger Audit Verifier (`src/migration/verifier/`):**
   * Computes Trial Balance from source data vs destination PostgreSQL ledger.
   * Emits discrepancy report highlighting any variance > $0.00.

#### Acceptance Criteria & Invariants:
* Must achieve 100% mathematical parity on debit/credit sums for imported fiscal years.
* Must preserve historical transaction timestamps and original external reference UUIDs.
* Must include unit and integration tests with mock API responses for 10,000 journal lines.

---

### Task 2: Autonomous 3-Way Matching & Anomaly Reconciliation Agent

```yaml
Task Title: "feat(agents): implement autonomous 3-way invoice matching and ledger reconciliation worker"
Labels: ["ai-first", "accounting", "agent-task"]
Assigned Agent: "Jules"
```

#### Task Description & Context:
Reference `research/ai_first/13_ai_native_agentic_workflows.md` and `research/benchmarks/01_core_finance_ledger.md`. Manual invoice reconciliation is the #1 operational bottleneck in mid-market companies. We must implement an autonomous background worker that executes the "Action-as-a-Tool" paradigm to reconcile Purchase Orders (PO), Goods Receipt Notes (GRN), and incoming Vendor Invoices.

#### Required Deliverables:
1. **Tool Definitions (`src/agents/tools/reconciliation/`):**
   * `fetch_pending_receipts(vendor_id: string, po_id: string)`: Retrieves unbilled GRN items.
   * `calculate_landed_cost_variance(invoice_amount: number, po_amount: number)`: Determines price and quantity variance percentages.
   * `post_reconciliation_journal(discrepancy_treatment: enum, idempotency_key: string)`: Posts balanced double-entry adjustments for tax rounding, freight variance, or discounts.
2. **Deterministic State Machine & Fallback Rules:**
   * If Price Variance ≤ 1.5% and Absolute Dollar Variance ≤ $25.00: Auto-approve and post variance to `Purchase Price Variance (PPV)` expense account.
   * If Quantity Variance > 0: Flag for human supervisor with side-by-side diff.
3. **Cryptographic Audit Trail:**
   * Store agent reasoning traces, confidence score, and LLM input/output tokens in `agent_audit_logs` table.

#### Acceptance Criteria & Invariants:
* No hallucinated ledger accounts: all posting tools must strictly validate against existing active COA.
* Idempotency enforced: duplicate webhook payloads must return existing journal entry ID without re-executing state transitions.

---

### Task 3: Multi-Channel Inventory Webhook Streamer & Dynamic Safety Stock

```yaml
Task Title: "feat(inventory): high-throughput multi-channel inventory sync and dynamic safety stock calculator"
Labels: ["inventory", "scm", "agent-task"]
Assigned Agent: "Jules"
```

#### Task Description & Context:
Reference `research/benchmarks/02_procurement_scm_inventory.md` and `research/edge_cases/09_inventory_supply_chain_pitfalls.md`. Growing D2C brands churn from Zoho because high-volume multi-channel order spikes create race conditions and out-of-stock overselling. We must build a concurrent, race-condition-free inventory allocation service.

#### Required Deliverables:
1. **Concurrency Control Engine (`src/inventory/allocator/`):**
   * Implement pessimistic row-locking (`SELECT ... FOR UPDATE`) or atomic Redis decrement for real-time SKU reservations.
   * Decouple "Physical On-Hand" from "Available to Promise (ATP)" across multi-location warehouses.
2. **Dynamic Reorder Point (ROP) Service (`src/inventory/forecasting/`):**
   * Mathematical calculation: `ROP = (Average Daily Usage * Lead Time) + Safety Stock`.
   * Dynamic safety stock algorithm accounting for supplier lead-time variance and demand seasonality.
3. **Webhook Dispatcher:**
   * Instant broadcast of updated ATP levels to Shopify, Amazon SP-API, and WooCommerce endpoints upon order confirmation.

#### Acceptance Criteria & Invariants:
* Concurrency test: 500 simultaneous checkout requests on a single SKU with only 10 units in stock must result in exactly 10 successful reservations and 490 out-of-stock rejections with zero overselling.

---

### Task 4: Open-Core Licensing Guard & Enterprise Governance RBAC

```yaml
Task Title: "feat(governance): implement row-level security RBAC and BSL enterprise module guard"
Labels: ["security", "governance", "licensing", "agent-task"]
Assigned Agent: "Jules"
```

#### Task Description & Context:
Reference `research/business_strategy/17_monetization_open_core_cloud_economics.md`. To execute our commercial open-core model, we must establish a clean boundary between the open-source MIT core and enterprise commercial modules (SAML/SSO, granular Row-Level Security, multi-entity currency consolidation).

#### Required Deliverables:
1. **PostgreSQL Row-Level Security (RLS) Policies:**
   * Multi-tenant data isolation enforced directly at the database engine level via tenant session variables (`app.current_tenant_id`).
   * Role-based permissions: Auditor (Read-only), Warehouse Clerk (Inventory only), Controller (Ledger post), Super Admin.
2. **Commercial License Key Validator (`src/enterprise/license/`):**
   * Cryptographically signed JWT license key verification for enterprise features.
   * Graceful degradation: if a license expires, the system remains 100% operational in read-only/community mode without corrupting existing data.
3. **Audit Log Streamer (`src/security/audit/`):**
   * Immutable, append-only log capturing all mutations, IP addresses, user IDs, and agent reasoning traces compliant with SOC2 Type II requirements.

#### Acceptance Criteria & Invariants:
* No query can leak records across tenant IDs, even in the event of an unhandled application-level error.
* Full test suite confirming that the core community engine boots and passes all ledger tests with zero enterprise dependencies loaded.
