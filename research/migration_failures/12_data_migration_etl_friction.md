# 12. Data Migration & ETL Friction Points: Moving off Legacy Systems

**Audience:** Lead Engineers, System Architects, and Staff Technical Product Managers (TPMs).
**Focus:** Technical specifications, ETL ingestion pipelines, data reconciliation proofs, conversion scripts, transformation edge cases, and schema translation for legacy ERP/Accounting migrations.

---

## 1. The Legacy ETL Friction Points

Migrating from legacy systems (like Tally, QuickBooks Online, SAP, NetSuite, or massive Excel spreadsheets) to a modern Business OS is fraught with data integrity and extraction challenges. Legacy systems rarely enforce the strict validation rules expected by modern ledgers, resulting in pervasive ETL friction points.

### 1.1 Inconsistent Field Formats & Data Quirks
Legacy data exports often contain deep structural inconsistencies:
* **Tally Prime:** Exports often rely on unstructured or deeply nested XML, featuring unconstrained parent-child ledger groups and flat inventory tagging which defy standard double-entry relationship modeling.
* **QuickBooks Online (QBO):** Enforces rigid, non-deletable default accounts (like "Uncategorized Asset") and maintains notoriously loose boundaries between "Customer" and "Vendor" entities, allowing cyclic or ambiguous relationships.
* **NetSuite / SAP:** Utilize complex, multi-segment Chart of Accounts (COA) dimensions and custom segment logic. Extracting transaction lines often requires resolving split lines, obscure clearing accounts, and proprietary custom fields that don't neatly map to a canonical state.

### 1.2 Dirty, Duplicate Customer Records
Without unified CRM/ERP checks, legacy systems are plagued by duplicated records (e.g., "Acme Corp", "Acme Corporation", "Acme Inc."). Attempting a direct import creates duplicate sub-ledgers and fragmented AP/AR agings.

### 1.3 Legacy COA Hierarchies
Older COAs often evolve organically, resulting in misaligned hierarchies that do not cleanly map to a modern, multi-dimensional ledger. The result is "mapping fatigue" where migration engineers spend weeks manually aligning old account codes to new ledger segments.

---

## 2. The Historical Data Dilemma

When moving to a new system, companies face a critical decision: migrate every historical transaction or start fresh. Attempting a full transactional ETL across 5–10 years of dirty, non-standard legacy ledgers consistently causes massive project delays, reconciliation discrepancies, and corrupted historic audit trails.

### 2.1 Recommended Architectural Best Practice: "Opening Balances Cutover + Searchable Read-Only Archive"
We strongly advocate for the **Opening Balances Cutover + Searchable Read-Only Archive** approach. This method avoids the quagmire of historical data transformation by migrating only the final closing balances of the legacy system as the opening balances in the new system. The historical data (the 5-10 years of transactions) is dumped into an indexed, searchable, read-only data lake or cold storage archive.

This path ensures fast, fail-safe customer adoption while maintaining full audit-readiness and compliance.

### 2.2 Alternative Options Matrix

| Approach | Description | Cost / Engineering Effort | Risk / Delays | Audit-Readiness |
| :--- | :--- | :--- | :--- | :--- |
| **Opening Balances + Read-Only Archive (Recommended)** | Migrate only opening balances. Store history in an indexed archive. | Low - Moderate | Low (Fastest time to value) | High (History is preserved immutably) |
| **Full Transactional Replay** | Re-creating every historical journal entry in the new system. | Very High (Requires massive transformation mapping) | Extremely High (Guaranteed imbalances, endless data cleaning) | Low (Replay often alters historical presentation) |
| **Phased Cutover (Hybrid)** | Migrate history for the current fiscal year only, archive the rest. | High | High (Year-to-date reconciliations are complex) | Moderate |

---

## 3. Field Mapping Schema & Transformation Pipeline

Automating the translation from legacy systems to a canonical Business OS requires a robust schema-reflection engine and a multi-stage transformation pipeline.

### 3.1 Transformation Pipeline
The ETL pipeline must follow strict staging boundaries to ensure data integrity:
`Ingest (CSV/Excel/SQL Dump) -> Validate Data Types -> Deduplicate -> Transform & Normalise -> Dry-Run Staging Table -> Production Ledger Commit`

### 3.2 Schema-Reflection & Heuristic Mapping
To map legacy fields (e.g., Tally's `LEDGERNAME`, QBO's `TxnDate`, SAP's `KUNNR`) to canonical OS fields (`party_id`, `posting_date`, `customer_code`), the mapping engine employs heuristic algorithms:
1. **String Similarity:** Calculate Levenshtein distance or Jaro-Winkler similarity to auto-suggest mappings between source and target headers.
2. **Token Similarity & LLM Fallback:** Use embedding similarity for obscurely named fields (e.g., mapping `USR_DEF_DT_4` to `expected_delivery_date`).

*Algorithmic Pseudo-code for Field Mapping:*
```python
def map_legacy_field(source_field, canonical_schema, threshold=0.85):
    best_match = None
    highest_score = 0.0

    # 1. Exact or Fuzzy String Matching
    for target_field in canonical_schema:
        score = jaro_winkler(source_field.lower(), target_field.name.lower())
        if score > highest_score:
            highest_score = score
            best_match = target_field

    # 2. LLM Embedding Fallback if below threshold
    if highest_score < threshold:
        source_embedding = get_embedding(source_field)
        best_match = find_closest_schema_embedding(source_embedding, canonical_schema)

    return best_match
```

### 3.3 JSON Declarative Mapping Specification
A robust pipeline uses a declarative mapping configuration to translate source data into the target schema dynamically.

```json
{
  "source_system": "QuickBooksOnline",
  "entities": {
    "Customer": {
      "target_model": "Party",
      "mapping": [
        { "source_field": "DisplayName", "target_field": "party_name", "transform": "trim_whitespace" },
        { "source_field": "PrimaryEmailAddr.Address", "target_field": "contact_email", "transform": "lowercase" }
      ],
      "deduplication_keys": ["contact_email", "party_name"]
    },
    "JournalEntry": {
      "target_model": "Transaction",
      "mapping": [
        { "source_field": "TxnDate", "target_field": "posting_date", "transform": "parse_iso8601" },
        { "source_field": "Line.Amount", "target_field": "amount", "transform": "to_decimal" },
        { "source_field": "Line.DetailType", "target_field": "entry_type", "transform": "map_debit_credit" }
      ]
    }
  }
}
```

---

## 4. Migration Dry-Run & Reconciliation Engine

The final point of friction is building trust. Customers will not cut over unless they can mathematically prove that the new ledger perfectly reflects their historical financial state.

### 4.1 Automated Dry-Run Verification
Before the final commit, the ETL engine must run a "Dry-Run" into a staging environment. An Automated Reconciliation Engine then runs mathematical checks against the legacy system's closing Trial Balance, AR Aging, and AP Aging reports.

**Core Reconciliation Proofs:**

1. **Trial Balance Parity:** The sum of all debits and credits in the migrated opening balances must perfectly match the legacy system's closing state.
   $$\sum \text{Legacy Debit} \equiv \sum \text{New Debit} \quad \text{and} \quad \sum \text{Legacy Credit} \equiv \sum \text{New Credit}$$

2. **Sub-Ledger Integrity (AR/AP):** The open invoices migrated must match the customer's open balance in the legacy aging report.
   $$\text{Legacy AR Aging (Customer } X\text{)} \equiv \text{Imported AR Open Invoices (Customer } X\text{)}$$

Real-world migration failures—such as opening trial balance imbalances, customer balance mismatches post-cutover, or orphan invoices without purchase orders—are aggressively caught and flagged in the Dry-Run stage, forcing resolution before the production ledger is ever touched.
