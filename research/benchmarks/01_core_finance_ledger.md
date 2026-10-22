# Core Financials, Treasury & Statutory Ledger: Architectural Benchmark

## 1. Chart of Accounts (COA) & Hierarchical Segmentation

### 1.1 Comparative Functional Analysis

*   **SAP S/4HANA**: Employs a robust global Chart of Accounts (Operating COA) layered with Company Code-specific data. It heavily relies on the Universal Journal (ACDOCA) to unify FI (Financial Accounting) and CO (Controlling). Segmentation is deeply structured using standard entities like Cost Centers, Profit Centers, Functional Areas, and Segments. This architecture enforces strict validations but requires high upfront configuration.
*   **NetSuite**: Utilizes a single unified COA across subsidiaries (with subsidiary-specific mappings). Multi-dimensional reporting is driven by core segments (Subsidiary, Department, Class, Location) and unlimited Custom Segments. This provides high flexibility, decoupling natural accounts from organizational hierarchy.
*   **Dynamics 365 F&O**: Features a highly normalized Financial Dimensions framework. Users define Account Structures that dictate which dimensions are valid or mandatory for specific main accounts. Rather than storing flat strings, it dynamically generates and caches `DimensionAttributeValueCombination` (RecId) records for every unique account-dimension combination used in transactions.
*   **Odoo Accounting**: Maintains a relatively flat COA structure. For multi-dimensional reporting, it shifts the burden to "Analytic Accounting" (Analytic Plans and Analytic Accounts), which operate orthogonally to the standard GL accounts. This allows lightweight COA management while retaining deep cost analysis capabilities.
*   **Zoho Books**: Designed for SMB to mid-market, it features a straightforward flat COA. Dimensionality is achieved via "Reporting Tags," which act as key-value pairs applied at the transaction line level. It lacks the complex hierarchical dimension constraints found in enterprise systems.

### 1.2 Synthesized Canonical Blueprint

For a modern Business OS, the COA should strictly represent the **Natural Account** (Asset, Liability, Equity, Revenue, Expense) without embedding organizational structure (like departments or geographies) into the account code.

**Architectural Pattern: Dynamic Dimensional Tagging Engine**
*   **Base Accounts**: Define the "What" (e.g., 6000 - Travel Expense).
*   **Dimensions**: Define the "Who, Where, and Why" (e.g., Cost Center, Region, Project).
*   **Ledger Code Combinations (CCID)**: Similar to D365, dynamically hash and store unique valid combinations of Account + Dimensions as they are transacted. This allows massive scale in analytical queries without expanding the natural COA.
*   **Dimension Rule Engine**: Rules dictate whether specific dimensions are mandatory, optional, or restricted based on the Natural Account or Transaction Type.

### 1.3 Hybrid Techno-Functional Data Catalog

#### Canonical Entities & Field Specifications

| Entity | Attribute | Semantic Type | Constraints | Description |
| :--- | :--- | :--- | :--- | :--- |
| **Account** | `id` | UUID | PK | Unique identifier |
| | `code` | VARCHAR(50) | UNIQUE(tenant_id, code) | Natural account number (e.g., "1000") |
| | `name` | VARCHAR(255) | NOT NULL | Human-readable name |
| | `account_type` | ENUM | NOT NULL | ASSET, LIABILITY, EQUITY, REVENUE, EXPENSE |
| | `is_active` | BOOLEAN | DEFAULT TRUE | Soft delete flag |
| **Dimension** | `id` | UUID | PK | E.g., "Cost Center", "Project" |
| | `name` | VARCHAR(100) | NOT NULL | Dimension label |
| **DimensionValue**| `id` | UUID | PK | Specific value (e.g., "CC-Marketing") |
| | `dimension_id` | UUID | FK -> Dimension(id) | Parent dimension link |
| | `code` | VARCHAR(50) | NOT NULL | Unique code per dimension |
| **AccountCodeCombination**| `id` | UUID | PK | CCID used on GL Lines |
| | `account_id` | UUID | FK -> Account(id) | Natural Account |
| | `dimension_hash`| VARCHAR(256) | UNIQUE(tenant_id) | Deterministic hash of Account + sorted DimensionValues |

#### Real-World Reference Mappings

*   **SAP S/4HANA**:
    *   `SKA1` (G/L Account Master), `SKB1` (Company Code Data).
    *   Cost Centers (`CSKS`), Profit Centers (`CEPC`).
    *   Universal Journal (`ACDOCA`) unifies these natively as columns.
*   **NetSuite**:
    *   `Account` record.
    *   Core Segments: `Department`, `Classification`, `Location`.
    *   `CustomSegment` configuration engine.
*   **Dynamics 365 F&O**:
    *   `MainAccount` table.
    *   `DimensionAttribute` (Dimension definitions).
    *   `DimensionAttributeValueCombination` (Equivalent to our AccountCodeCombination).
*   **Odoo**:
    *   `account.account` (Natural COA).
    *   `account.analytic.plan`, `account.analytic.account` (Dimensions).
*   **Zoho Books**:
    *   `Chart of Accounts API` endpoints.
    *   `Reporting Tags` for dimensional mapping.

#### Mathematical Invariants & Business Validations

1.  **Code Uniqueness & Immutability**:
    *   `ASSERT UNIQUE(tenant_id, Account.code)`.
    *   Once an `Account` or `DimensionValue` is referenced in a posted GL Entry (e.g., `AccountCodeCombination` exists and is linked to a ledger line), it cannot be hard-deleted. Updates must toggle `is_active = FALSE`.
2.  **Dimension Rule Evaluation**:
    *   If `RuleEngine.isMandatory(Account.id, Dimension.id) == TRUE`, then transaction insertion `FAILS` if `DimensionValue` for that `Dimension` is `NULL`.
3.  **Deterministic Hash Invariant**:
    *   The `dimension_hash` must be a cryptographically consistent sort of dimension IDs to ensure that identical tag combinations resolve to the exact same `AccountCodeCombination.id` globally.

## 2. General Ledger Entry Schema

### 2.1 Comparative Functional Analysis

*   **SAP S/4HANA**: The Universal Journal (`ACDOCA`) is the single source of truth, merging FI and CO. It stores line items for all business processes (GL, AP, AR, Asset Accounting, Material Ledger). It handles up to 10 parallel currencies natively. FX revaluation creates specific delta postings.
*   **NetSuite**: Operates on a Header (`Transaction`) and Line (`TransactionLine`) model. Custom GL Lines allow scripted generation of complex accounting entries at posting time without modifying the source transaction. FX handles base vs. foreign currency with dedicated revaluation routines creating automated variance journals.
*   **Dynamics 365 F&O**: Strongly normalizes transactions. Source documents (like Invoices) post via the Subledger Accounting framework to the General Ledger. The GL structures split into `GeneralJournalEntry` (Header) and `GeneralJournalAccountEntry` (Line). Multi-currency includes Accounting Currency, Reporting Currency, and Transaction Currency.
*   **Odoo Accounting**: Uses `account.move` for both headers (Journals, Invoices) and `account.move.line` for the actual debits/credits. It enforces strict double-entry balancing at the `account.move` level. Multi-currency computes base currency equivalents instantly upon line creation.
*   **Zoho Books**: Simplified journal entry system. While it enforces double-entry rules, its multi-currency ledger computes realized gains/losses dynamically upon payment application rather than complex parallel ledger tracking found in S/4HANA.

### 2.2 Synthesized Canonical Blueprint

The GL Entry Schema must act as an **Immutable Event Log**. All downstream ledgers (AP, AR, Inventory) are simply sub-ledgers that feed the GL Engine.

**Architectural Pattern: Dual-Layer Ledger**
*   **Transaction Document (Header)**: The business event (e.g., Sales Invoice, Depreciation Run, Manual Journal).
*   **Journal Entry Lines**: The normalized, atomic debit/credit movements.
*   **Multi-Currency Core**: Every line must capture three tiers of currency:
    1.  *Transaction Currency*: The currency of the actual event (e.g., EUR).
    2.  *Base/Accounting Currency*: The functional currency of the legal entity (e.g., USD).
    3.  *Reporting Currency*: A consolidated group currency (e.g., GBP).

### 2.3 Hybrid Techno-Functional Data Catalog

#### Canonical Entities & Field Specifications

| Entity | Attribute | Semantic Type | Constraints | Description |
| :--- | :--- | :--- | :--- | :--- |
| **JournalEntry** | `id` | UUID | PK | Header ID |
| | `posting_date` | DATE | NOT NULL | Accounting period date |
| | `document_type` | ENUM | NOT NULL | INVOICE, PAYMENT, ACCRUAL, MANUAL |
| | `status` | ENUM | NOT NULL | DRAFT, POSTED, VOIDED |
| **JournalEntryLine** | `id` | UUID | PK | Line ID |
| | `journal_entry_id` | UUID | FK -> JournalEntry(id) | Parent header |
| | `account_combination_id` | UUID | FK -> AccountCodeCombination(id) | The dynamic CCID |
| | `txn_currency` | CHAR(3) | NOT NULL | ISO 4217 (e.g., EUR) |
| | `txn_amount` | DECIMAL(18,4) | NOT NULL | Original amount (signed: +Debit, -Credit) |
| | `base_currency` | CHAR(3) | NOT NULL | ISO 4217 (e.g., USD) |
| | `base_amount` | DECIMAL(18,4) | NOT NULL | Converted to functional currency |
| | `exchange_rate` | DECIMAL(18,8) | NOT NULL | Rate used at posting_date |

#### Real-World Reference Mappings

*   **SAP S/4HANA**:
    *   Header: `BKPF` (Accounting Document Header).
    *   Line/Unified: `ACDOCA` (Universal Journal Entry Line Items).
*   **NetSuite**:
    *   Header: `Transaction`.
    *   Line: `TransactionLine`.
*   **Dynamics 365 F&O**:
    *   Header: `GeneralJournalEntry`.
    *   Line: `GeneralJournalAccountEntry`.
*   **Odoo**:
    *   Header: `account.move`.
    *   Line: `account.move.line`.
*   **Zoho Books**:
    *   Header: `Journals` (API).
    *   Line: `Journal_Lines` (API).

#### Mathematical Invariants & Business Validations

1.  **Double-Entry Balancing Constraint**:
    *   For a given `JournalEntry` where `status == POSTED`, the sum of `base_amount` across all its `JournalEntryLine` records MUST equal zero.
    *   $\sum_{i=1}^{n} \text{base\_amount}_{i} = 0$
2.  **Immutability of Posted Records**:
    *   If `JournalEntry.status == POSTED`, no updates to `txn_amount`, `base_amount`, `account_combination_id`, or `posting_date` are allowed. To correct, a reversing journal MUST be posted.
3.  **Foreign Exchange Revaluation & Gains/Losses**:
    *   *Realized Gain/Loss*: Calculated at settlement time (e.g., Payment applied to Invoice). $\text{Gain} = (\text{Invoice Txn Amount} \times \text{Rate}_{\text{Payment}}) - (\text{Invoice Txn Amount} \times \text{Rate}_{\text{Invoice}})$.
    *   *Unrealized Gain/Loss*: Calculated at period-end for open AP/AR balances using the period-end spot rate.

## 3. Period-End Closing Mechanisms

### 3.1 Comparative Functional Analysis

*   **SAP S/4HANA**: Features complex, multi-stage closing mechanisms governed by the Schedule Manager. It supports hard closes for statutory reporting and soft closes for internal management accounting. Retained earnings are updated automatically, but balance carryforward (`FAGLGVTR`) must be explicitly run to generate opening balances for the next fiscal year.
*   **NetSuite**: Utilizes a checklist-based Period Close Checklist. It supports locking specific sub-ledgers (AR/AP/Inventory) before locking the general ledger (a "soft close" phase). An automated process calculates retained earnings and rolls forward balances without requiring a discrete opening balance journal.
*   **Dynamics 365 F&O**: Year-end close requires running a specific routine that zeros out P&L accounts to retained earnings and creates opening balance transactions in the new fiscal year. It supports rollback/re-running of the year-end close to accommodate late-arriving audit adjustments.
*   **Odoo Accounting**: Uses "Lock Dates" rather than complex batch closing routines. Administrators set a "Non-advisers Lock Date" (soft close) and an "All Users Lock Date" (hard close). P&L balances are virtually rolled up to retained earnings for reporting, meaning a formal closing journal is often bypassed for a continuous close approach.
*   **Zoho Books**: Operates similarly to Odoo but geared toward simpler accounting. Administrators lock periods via a transaction lock date. Year-end is largely virtual for reporting purposes, though a formal manual closing entry can be made if requested by accountants.

### 3.2 Synthesized Canonical Blueprint

The modern ledger should support a **Continuous Close Architecture**, avoiding massive batch jobs to calculate retained earnings, relying instead on high-performance compute and virtual aggregations, protected by strict chronological access controls (Lock Dates).

**Architectural Pattern: Phased Chronological Ledger Locks**
*   **Sub-Ledger Soft Locks**: Prevent new AR/AP/Inventory transactions prior to Date X, but allow manual GL adjustments by controllers.
*   **Global Hard Locks**: Absolute freeze on any accounting event (no DML operations on `JournalEntry` where `posting_date <= lock_date`).
*   **Virtual Retained Earnings**: Instead of physical zeroing entries, query engines dynamically roll up Revenue/Expense accounts into a Retained Earnings aggregate based on the fiscal year definition.

### 3.3 Hybrid Techno-Functional Data Catalog

#### Canonical Entities & Field Specifications

| Entity | Attribute | Semantic Type | Constraints | Description |
| :--- | :--- | :--- | :--- | :--- |
| **FiscalPeriod** | `id` | UUID | PK | Period identifier |
| | `start_date` | DATE | NOT NULL | E.g., '2024-01-01' |
| | `end_date` | DATE | NOT NULL | E.g., '2024-01-31' |
| | `is_year_end` | BOOLEAN | DEFAULT FALSE | Flags period as end of fiscal year |
| **LedgerLock** | `id` | UUID | PK | Lock state identifier |
| | `module` | ENUM | NOT NULL | AR, AP, INVENTORY, GL_ALL |
| | `locked_through_date`| DATE | NOT NULL | Chronological boundary |

#### Real-World Reference Mappings

*   **SAP S/4HANA**:
    *   `OB52` (Posting Period Variant configuration).
    *   `FAGLGVTR` (Balance Carryforward program).
*   **NetSuite**:
    *   `AccountingPeriod` record (states: AP Locked, AR Locked, All Locked, Closed).
*   **Dynamics 365 F&O**:
    *   `LedgerPeriodModuleAccessControl`.
    *   Year-end close routines generating `GeneralJournalEntry`.
*   **Odoo**:
    *   `res.company` fields: `period_lock_date`, `fiscalyear_lock_date`.
*   **Zoho Books**:
    *   Transaction Lock settings in Company Profile.

#### Mathematical Invariants & Business Validations

1.  **Strict Chronological Enclosure**:
    *   `ASSERT new_transaction.posting_date > max(LedgerLock.locked_through_date)` for the relevant `module`.
2.  **Virtual Opening Balance Theorem**:
    *   Opening Balance for Asset/Liability at Year $Y$ = $\sum$ of all transactions for that account from the beginning of time up to $Y-1$.
    *   Opening Balance for Retained Earnings at Year $Y$ = $\sum$ (Revenue - Expense) from the beginning of time up to $Y-1$.
3.  **Rollback / Late Adjustment State Handling**:
    *   If a lock is reverted (e.g., to post an audit adjustment), the system must log a specialized "Late Adjustment" event and trigger an asynchronous cache invalidation for any pre-computed opening balance materializations.

## 4. Accounts Receivable (AR) Subledger

### 4.1 Comparative Functional Analysis

*   **SAP S/4HANA**: The FI-AR module is deeply integrated with SD (Sales & Distribution). It features a robust dunning program (`F150`) capable of calculating interest, fees, and generating tiered dunning letters. Write-offs and bad debt provisions are handled through complex batch evaluation runs. Aging is dynamically calculated using baseline dates and payment terms.
*   **NetSuite**: Drives AR through Invoice and Customer Payment records. Dunning is managed via automated workflows or the Dunning Letters suiteapp. Aging is calculated on-the-fly in reporting based on the `dueDate`. Write-offs can be processed via credit memos or dedicated write-off journals.
*   **Dynamics 365 F&O**: Utilizes Customer transactions linked to standard GL entries. It includes comprehensive Credit and Collections management, allowing for detailed credit limits, collection letters, and interest calculation. Write-offs are managed within the collections workspace.
*   **Odoo Accounting**: Manages AR natively through `account.move` (type: out_invoice). The Follow-up levels feature automates dunning. Because AR lines are simply GL lines tagged with a `partner_id`, aging is a direct SQL view over unreconciled `account.move.line` records.
*   **Zoho Books**: Simplified AR focusing on Invoices and Payments. It offers basic automated payment reminders based on due dates. Bad debt write-offs are a dedicated action on the invoice UI that generates a closing journal entry.

### 4.2 Synthesized Canonical Blueprint

The AR Subledger should not exist as a structurally isolated database; it should be a specialized view and state machine layered over the core Ledger Lines, enriched with Customer (Party) and Terms data.

**Architectural Pattern: Open-Item State Machine**
*   **Receivable Initialization**: An invoice posts a debit to the AR Trade account, tagged with a `party_id` (Customer). This creates an "Open Item".
*   **Aging Engine**: A stateless compute layer evaluates the delta between `CURRENT_DATE` and `due_date` across all Open Items, bucketing them dynamically (0-30, 31-60, etc.).
*   **Automated Dunning & Write-Offs**: A scheduled rule-engine queries Open Items by age and trigger conditions (e.g., age > 90 days), emitting events to send emails, halt credit, or generate a system-proposed write-off Journal Entry.

### 4.3 Hybrid Techno-Functional Data Catalog

#### Canonical Entities & Field Specifications

| Entity | Attribute | Semantic Type | Constraints | Description |
| :--- | :--- | :--- | :--- | :--- |
| **Invoice (Header)** | `id` | UUID | PK | Document ID |
| | `party_id` | UUID | FK -> Party(id) | Customer reference |
| | `terms_id` | UUID | FK -> PaymentTerms(id)| Determines due date |
| | `due_date` | DATE | NOT NULL | Computed from terms |
| **OpenItem (Materialized View / State)** | `ledger_line_id`| UUID | FK -> JournalEntryLine(id)| The specific AR debit line |
| | `original_amount` | DECIMAL(18,4) | NOT NULL | Amount billed |
| | `cleared_amount` | DECIMAL(18,4) | NOT NULL | Amount paid/credited |
| | `status` | ENUM | NOT NULL | OPEN, PARTIAL, CLEARED |

#### Real-World Reference Mappings

*   **SAP S/4HANA**:
    *   `BSIK` / `BSID` (Secondary index for open items).
    *   Dunning tables (`MHND`).
*   **NetSuite**:
    *   `Invoice` record.
    *   `CustomerPayment` record.
*   **Dynamics 365 F&O**:
    *   `CustTrans` (Customer transactions).
    *   `CustTransOpen` (Unsettled amounts).
*   **Odoo**:
    *   `account.move.line` where `account_type == 'asset_receivable'` and `reconciled == False`.
*   **Zoho Books**:
    *   `Invoices` API (status = overdue, unpaid).

#### Mathematical Invariants & Business Validations

1.  **Open Item Settlement Integrity**:
    *   `ASSERT cleared_amount <= original_amount`. Overpayments must create a distinct credit-balance Open Item (Unapplied Payment), rather than pushing `cleared_amount` above `original_amount`.
2.  **Aging Bucketing Logic**:
    *   Bucket index = `FLOOR((CURRENT_DATE - due_date) / 30)` for standard 30-day tranches.
3.  **Write-Off Constraint**:
    *   A write-off event must post a balanced `JournalEntry` (Debit Bad Debt Expense, Credit AR) and explicitly link to the `ledger_line_id` of the target Open Item to increment its `cleared_amount` to exactly equal `original_amount`.

## 5. Accounts Payable (AP) Subledger & Procurement Core

### 5.1 Comparative Functional Analysis

*   **SAP S/4HANA**: FI-AP connects directly with MM (Materials Management). Employs strict 3-way matching (PO, Goods Receipt (GR), Invoice Receipt (IR)). The GR/IR clearing account mechanism is a cornerstone of SAP's accrual accounting. It handles complex payment runs (`F110`) and intricate withholding tax (TDS) calculations at the time of invoice or payment.
*   **NetSuite**: Supports 3-way matching via Vendor Bill routing and Purchase Order receiving. Tolerances can be configured to hold bills that exceed PO amounts. Payment runs (Electronic Bank Payments) batch bills for clearing.
*   **Dynamics 365 F&O**: Very strong AP automation and invoice matching policies (2-way and 3-way). Tolerance parameters can be set at the legal entity, vendor, or item level. Payment proposals automate the selection of invoices due for payment, optimizing for early payment discounts.
*   **Odoo Accounting**: Manages AP via `account.move` (type: in_invoice). 3-way matching is handled by comparing PO lines, received quantities in the Inventory app, and billed quantities. Withholding taxes are configured as negative taxes on the invoice line.
*   **Zoho Books**: Focuses on Vendor Bills and Purchase Orders. 3-way matching is manual or requires Zoho Inventory. Basic payment batching and straightforward TDS handling (configured as a tax authority deduction).

### 5.2 Synthesized Canonical Blueprint

The AP engine must act as an automated gatekeeper, preventing cash leakage through strict matching tolerances, while optimizing outbound cashflow via intelligent payment batching.

**Architectural Pattern: Accrual Clearing & Match Engine**
*   **GR/IR Pattern**: Goods Receipt instantly debits Inventory/Expense and credits a GR/IR Accrual account. The subsequent Vendor Invoice debits GR/IR and credits AP. This decouples physical receipt from financial obligation.
*   **3-Way Tolerance Engine**: Intercepts the Vendor Invoice post attempt. It calculates the variance between (PO Price $\times$ GR Qty) and (Invoice Price $\times$ Invoice Qty).
*   **Payment Factory**: A state machine that selects Open AP Items based on `due_date`, applies available vendor credits, calculates Withholding Tax deductions, and generates a structured Payment Batch payload (e.g., ISO 20022 PAIN.001).

### 5.3 Hybrid Techno-Functional Data Catalog

#### Canonical Entities & Field Specifications

| Entity | Attribute | Semantic Type | Constraints | Description |
| :--- | :--- | :--- | :--- | :--- |
| **VendorBill (Header)**| `id` | UUID | PK | Invoice ID |
| | `party_id` | UUID | FK -> Party(id) | Vendor reference |
| | `po_id` | UUID | FK -> PurchaseOrder(id)| Link to PO |
| | `status` | ENUM | NOT NULL | DRAFT, PENDING_APPROVAL, POSTED |
| **MatchTolerance** | `id` | UUID | PK | Rule configuration |
| | `max_qty_variance` | DECIMAL(5,4) | | Percentage allowed (e.g., 0.05 for 5%) |
| | `max_price_variance`| DECIMAL(5,4) | | Percentage allowed |
| **PaymentRun** | `id` | UUID | PK | Batch identifier |
| | `total_amount` | DECIMAL(18,4) | NOT NULL | Sum of payments |
| | `status` | ENUM | NOT NULL | GENERATED, TRANSMITTED, REJECTED, CLEARED |

#### Real-World Reference Mappings

*   **SAP S/4HANA**:
    *   `BSIK` / `BSAK` (Open/Cleared AP items).
    *   `EKKO`/`EKPO` (PO), `RBKP`/`RSEG` (Invoice).
*   **NetSuite**:
    *   `VendorBill` record.
    *   `VendorPayment` record.
*   **Dynamics 365 F&O**:
    *   `VendTrans`, `VendTransOpen`.
    *   `VendInvoiceInfoTable` (Pending invoices).
*   **Odoo**:
    *   `account.move` (in_invoice).
    *   `account.payment.register` (Payment runs).
*   **Zoho Books**:
    *   `Bills` API.

#### Mathematical Invariants & Business Validations

1.  **3-Way Match Invariant**:
    *   Let $V_q = \text{Invoice Qty}$, $V_p = \text{Invoice Price}$.
    *   Let $R_q = \text{Received Qty}$, $P_p = \text{PO Price}$.
    *   `IF ABS(1 - (V_q / R_q)) > MatchTolerance.max_qty_variance THEN BLOCK_POSTING()`.
    *   `IF ABS(1 - (V_p / P_p)) > MatchTolerance.max_price_variance THEN BLOCK_POSTING()`.
2.  **GR/IR Zero-Sum Integrity**:
    *   For a fully invoiced PO line, the balance of the `AccountCodeCombination` representing the GR/IR accrual for that specific PO MUST equal exactly 0.00.
3.  **Withholding Tax (TDS) Computation**:
    *   Payment Amount = Invoice Total - Computed TDS. TDS must simultaneously generate a posted liability journal to the Tax Authority account.

## 6. Bank Reconciliation Algorithms & Treasury

### 6.1 Comparative Functional Analysis

*   **SAP S/4HANA**: The Electronic Bank Statement (EBS) engine parses BAI2, MT940, or CAMT.053 formats. Posting rules automatically create sub-ledger clearing entries based on interpretation algorithms (e.g., matching reference numbers, amounts, and dates). High precision but requires rigorous configuration of search strings and posting rules.
*   **NetSuite**: The Bank Feeds SuiteApp imports data natively. The Intelligent Transaction Matching engine uses automated rules (exact match, date/amount thresholds) to reconcile statements against system transactions. Unmatched items trigger manual reconciliation or automated journal creation.
*   **Dynamics 365 F&O**: Advanced Bank Reconciliation supports complex matching rules spanning 1-to-1, 1-to-many, and many-to-many relationships. It handles chargebacks and NSF reversals by generating automated correction journals against the source sub-ledger items.
*   **Odoo Accounting**: Extremely user-friendly reconciliation widget. It uses a fuzzy matching algorithm (Levenshtein distance on text, exact amount match) to suggest reconciliations. It excels at rapid "reconciliation models" that auto-generate bank fees, write-offs, or multi-line allocations directly from the bank feed screen.
*   **Zoho Books**: Integrates via Yodlee or Plaid for direct feeds. Uses basic exact amount + date proximity rules to suggest matches. It allows quick categorization of unmatched lines into expense accounts.

### 6.2 Synthesized Canonical Blueprint

Bank Reconciliation should be treated as an **Asynchronous Stream Processing** problem, ingesting external bank events and probabilistically mapping them to internal ledger events.

**Architectural Pattern: Probabilistic Reconciliation Engine**
*   **Ingestion Pipeline**: Normalizes ISO 20022 CAMT / BAI2 / Plaid JSON into a standard `BankStatementLine` entity.
*   **Deterministic Rule Pass**: Exact matches on (Amount + Date + Reference ID). Automatically clears the internal GL line.
*   **Fuzzy Rule Pass**: (Amount + Proximity Date + Levenshtein distance on Description vs. Vendor Name). Flags for user review (Confidence Score > 80%).
*   **Exception Handlers**: Unmatched lines matching specific regex (e.g., "WIRE FEE") automatically post to expense accounts via predefined Auto-Journal rules.

### 6.3 Hybrid Techno-Functional Data Catalog

#### Canonical Entities & Field Specifications

| Entity | Attribute | Semantic Type | Constraints | Description |
| :--- | :--- | :--- | :--- | :--- |
| **BankStatement**| `id` | UUID | PK | External feed envelope |
| | `account_id` | UUID | FK -> Account(id) | The Bank GL Account |
| | `closing_balance`| DECIMAL(18,4) | NOT NULL | Reported by bank |
| **BankStatementLine**| `id` | UUID | PK | Individual bank event |
| | `statement_id` | UUID | FK -> BankStatement(id)| Link to envelope |
| | `amount` | DECIMAL(18,4) | NOT NULL | Transaction amount |
| | `memo` | TEXT | | Raw text from bank |
| **ReconciliationRule**| `id` | UUID | PK | Logic configuration |
| | `regex_pattern` | VARCHAR(255) | | E.g., `(?i).*FEE.*` |
| | `target_account_id`| UUID | FK -> Account(id) | Auto-post destination |

#### Real-World Reference Mappings

*   **SAP S/4HANA**:
    *   `FEBKO` (Header), `FEBEP` (Line items).
*   **NetSuite**:
    *   `BankStatement`, `BankStatementLine`.
*   **Dynamics 365 F&O**:
    *   `BankAccountStatement`, `BankAccountStatementLine`.
*   **Odoo**:
    *   `account.bank.statement`, `account.bank.statement.line`.
*   **Zoho Books**:
    *   `Bank_Transactions` API.

#### Mathematical Invariants & Business Validations

1.  **Statement Integrity Check**:
    *   `ASSERT BankStatement.closing_balance == BankStatement.opening_balance + SUM(BankStatementLine.amount)`.
2.  **Reconciliation State Lock**:
    *   Let $L$ be a `JournalEntryLine` representing a cash movement.
    *   If $L$ is linked to a `BankStatementLine`, $L$ is permanently locked. Reversals must spawn a new compensating $L'$ and a new $B'$ from the bank.
3.  **Fuzzy Match Scoring Algorithm**:
    *   $S = W_1(\text{AmountMatch}) + W_2(\Delta\text{Date}) + W_3(1 - \frac{\text{Levenshtein}(B_{memo}, L_{reference})}{\max(|B_{memo}|, |L_{reference}|)})$.
    *   If $S > \text{AutoClearThreshold}$, execute match.
