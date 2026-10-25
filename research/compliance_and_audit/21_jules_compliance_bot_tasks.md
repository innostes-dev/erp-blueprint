# Google Jules Tasks: Autonomous Compliance & Audit Bots Engine
## Engineering Prompts for Jules to Implement the Compliance & Tax Moat

This document outlines four production-ready task specifications formatted for **Google Jules** to build the autonomous compliance, audit, and tax-law enforcement engines.

---

### Task 1: Autonomous Government Tax Portal Reconciliation & Vendor Hold Engine

```yaml
Task Title: "feat(compliance): build government tax portal auto-reconciliation engine and vendor payment hold bot"
Labels: ["compliance", "tax", "reconciliation", "agent-task"]
Assigned Agent: "Jules"
```

#### Task Description & Context:
Reference `research/compliance_and_audit/20_autonomous_compliance_bots_and_statutory_watchdogs.md`. In modern tax regimes (such as India's GST GSTR-2B or European VAT verification), companies lose millions when vendors fail to upload invoices to government portals. We must implement an autonomous service that crawls/ingests government tax data, reconciles it against internal purchase bills, and places automated payment holds on tax amounts for non-compliant suppliers.

#### Required Deliverables:
1. **Government Portal Ingestion Pipeline (`src/compliance/tax/ingestion/`):**
   * Schema parser for government tax credit feeds (JSON/CSV) matching portal fields (`ctin`, `inum`, `idt`, `val`, `txval`, `iamt`, `camt`, `samt`).
2. **Fuzzy Reconciliation Matcher (`src/compliance/tax/matcher/`):**
   * Multi-pass matching algorithm:
     * *Pass 1 (Exact):* Exact match on `Vendor Tax ID` + `Invoice Number` + `Taxable Amount`.
     * *Pass 2 (Fuzzy):* Normalized invoice number (strips slashes, dashes, leading zeroes) with a 0.5% tolerance on tax rounding differences.
3. **Autonomous Vendor Payment Hold Mechanism (`src/compliance/tax/holds/`):**
   * If an internal invoice has no matching record in the government feed within 30 days of posting:
     * Split the payable into `Base Payable` and `Tax Hold Escrow`.
     * Flag invoice status as `STATUTORY_TAX_HOLD`.
     * Trigger notification dispatcher to generate vendor email notice with missing invoice details.

#### Acceptance Criteria & Invariants:
* Must process 10,000 tax lines in <5 seconds.
* Zero false holds: if a vendor uploads their tax filing subsequently, the bot must automatically release the hold within the next sync cycle.
* Unit test coverage must include edge cases: invoices with special characters, credit note offsets, and multi-rate tax items.

---

### Task 2: Statutory Prompt-Payment & MSME Rule Watchdog (Section 43B(h))

```yaml
Task Title: "feat(compliance): implement statutory 45-day payment countdown and tax disallowance forecaster"
Labels: ["compliance", "accounts-payable", "msme", "agent-task"]
Assigned Agent: "Jules"
```

#### Task Description & Context:
Statutory compliance laws (such as Section 43B(h) in India, UK Late Payment Acts, and US Prompt Payment Regulations) mandate that payments to registered small/micro enterprises must be completed within 15–45 days. Delayed payments cannot be claimed as tax-deductible expenses, triggering severe corporate tax penalties.

#### Required Deliverables:
1. **Vendor Statutory Classification Directory (`src/compliance/msme/directory/`):**
   * Extend Vendor entity with statutory metadata: `enterprise_type` (`MICRO`, `SMALL`, `MEDIUM`, `LARGE`), `registration_number`, `verification_date`, and `annual_turnover_slab`.
2. **Dynamic Statutory Clock Engine (`src/compliance/msme/timer/`):**
   * Clock starts upon Goods Receipt Note (GRN) acceptance timestamp.
   * Calculate statutory deadline: 15 days (without agreement) or max 45 days (with written contract).
3. **Treasury Prioritization Queue & Disallowance Forecaster (`src/compliance/msme/forecaster/`):**
   * Autonomous scheduler running daily at 00:00 UTC.
   * Categorizes bills into urgency brackets: `SAFE` (>15 days left), `WARNING` (5-15 days left), `CRITICAL` (<5 days left), `OVERDUE` (Disallowed).
   * Generates projected tax liability surge: `Disallowance Amount * Corporate Tax Rate (e.g. 25%)`.

#### Acceptance Criteria & Invariants:
* Invoices approaching statutory limits (<5 days) must be automatically placed at the front of payment batch export queues.
* Must provide clear audit reports exportable as CSV/PDF for tax auditors detailing all on-time and delayed vendor settlements.

---

### Task 3: Continuous Audit & Segregation of Duties (SoD) Toxic Combination Guard

```yaml
Task Title: "feat(audit): implement continuous transaction auditor, SoD matrix enforcement, and tamper-proof hash chain"
Labels: ["audit", "security", "internal-controls", "agent-task"]
Assigned Agent: "Jules"
```

#### Task Description & Context:
Traditional ERPs allow administrative overrides that violate internal audit controls and SOX compliance. We need a real-time audit watchdog that enforces a strict Segregation of Duties (SoD) matrix and maintains a tamper-proof cryptographic ledger chain.

#### Required Deliverables:
1. **SoD Conflict Matrix Interceptor (`src/audit/sod/`):**
   * Enforce conflicting role restrictions:
     * *Conflict 1:* `Vendor Create/Edit` and `Payment Disbursement Signoff`.
     * *Conflict 2:* `Journal Post` and `Journal Audit Signoff`.
     * *Conflict 3:* `Purchase Order Approval` and `Goods Receipt Note Logging`.
   * Intercept any attempt by a single user ID to execute both steps in a single business cycle; reject transaction with `SOD_TOXIC_COMBINATION_DETECTED`.
2. **Smurfing & Structuring Pattern Detector (`src/audit/patterns/`):**
   * Autonomous worker scanning transactions for approval evasion:
     * Flag if user posts 3 or more payments to the same vendor within 72 hours where individual amounts are just below an approval ceiling (e.g. $9,950 when threshold is $10,000).
3. **Cryptographic Ledger Hash Chain (`src/audit/ledger_chain/`):**
   * Each journal entry record computes `current_hash = SHA256(previous_hash + entry_uuid + timestamp + debit_sum + credit_sum + posted_by_id)`.
   * Background verification worker checks ledger chain integrity every hour. Any manual database tampering breaks the chain immediately.

#### Acceptance Criteria & Invariants:
* No bypass permitted, even by users with `SUPER_ADMIN` role: any emergency override requires dual authorization (two distinct cryptographic signatures).
* Verification worker must successfully detect intentionally corrupted test hashes in unit tests.

---

### Task 4: Autonomous ASC 606 & IFRS 15 Revenue Recognition Schedule Engine

```yaml
Task Title: "feat(accounting): automated multi-element contract decomposition and deferred revenue amortization"
Labels: ["accounting", "compliance", "asc606", "agent-task"]
Assigned Agent: "Jules"
```

#### Task Description & Context:
Under GAAP ASC 606 and IFRS 15, businesses cannot recognize revenue upon invoicing if performance obligations are delivered over time. We need an autonomous revenue recognition engine that unpacks sales contracts, builds deferred revenue amortization schedules, and automatically posts monthly recognition journals.

#### Required Deliverables:
1. **Contract Performance Obligation Parser (`src/accounting/revenue/parser/`):**
   * Splits contract line items into:
     * `Point-in-Time` (e.g. hardware delivery, setup fee).
     * `Over-Time` (e.g. 12-month software subscription, annual maintenance).
2. **Amortization Schedule Generator (`src/accounting/revenue/schedules/`):**
   * Computes monthly recognized amounts using daily proration (`exact_days_in_month / total_contract_days`).
   * Handles contract modifications, mid-term upgrades, and early cancellations with prospective or retrospective adjustments.
3. **Autonomous Monthly Posting Worker (`src/accounting/revenue/worker/`):**
   * Runs on the first day of each month to post recognition entries:
     * `DR: 2200 - Deferred Revenue (Current Liability)`
     * `CR: 4000 - Subscription Revenue (Income)`

#### Acceptance Criteria & Invariants:
* The sum of all monthly amortization entries must exactly equal the total contract consideration to the penny.
* Premature recognition must be mathematically impossible: status remains `UNEARNED` until the schedule effective date arrives.
