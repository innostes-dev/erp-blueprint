# Architectural Requirements: Multi-Tenant Org Hierarchies & Concurrency Edge Cases

This document outlines the architectural requirements and strategies for handling complex multi-tenant enterprise organizational structures, high-throughput concurrency, strict data isolation, and timezone reconciliation.

## 1. Complex Organization Trees

Enterprise organizations often consist of holding companies with multiple subsidiaries, branches, and legal entities operating across different currencies and tax jurisdictions.

### 1.1. Hierarchical Data Model
* **Requirement**: The system must support nested organizational structures of arbitrary depth.
* **Architecture**: Implement an Adjacency List model with materialized paths or Closure Tables to allow efficient querying of sub-trees and parent relationships.
* **Entity Resolution**: Each transaction or record must be strictly bound to a specific legal entity node within the hierarchy, inheriting configuration (currency, tax rules) from its node or overriding as necessary.

### 1.2. Multi-Currency and Multi-Jurisdiction
* **Requirement**: The system must handle transactions in multiple currencies and calculate taxes based on the specific jurisdiction of the transacting entity.
* **Architecture**:
  * Store all financial values using a `Money` pattern (value + currency).
  * Record exchange rates at the exact time of the transaction for historical accuracy.
  * Integrate with a specialized tax engine (e.g., Avalara, Stripe Tax) isolated by entity jurisdiction.

### 1.3. Inter-Company Transactions
* **Requirement**: When one subsidiary sells to another within the same holding company, the system must automatically generate corresponding sales and purchase records.
* **Architecture**: Implement an event-driven architecture. A `SalesInvoiceCreated` event emitted by the selling entity triggers a consumer that idempotently generates a `PurchaseInvoice` in the buying entity's context, maintaining a cryptographic linkage between the two records for auditability.

## 2. Concurrency and Race Conditions: Gapless Sequential Invoices

Generating gapless sequential invoice numbers at 1,000 requests/second is a significant bottleneck because traditional relational database sequence generation (e.g., `AUTOINCREMENT`, `SERIAL`) allows gaps if a transaction rolls back.

### 2.1. Gapless Sequence Generation
* **Requirement**: Invoice numbers must be sequential with absolutely no gaps, supporting 1,000+ requests/second.
* **Architecture**:
  * **Late Assignment / Commit-Time Generation**: Do not assign the invoice number when the draft is created or during the business logic processing. Assign it strictly at the final database commit phase.
  * **Dedicated Sequencer**: Use a specialized, highly available counter service (e.g., Redis `INCR` backed by AOF persistence, or a single-partition Kafka topic where the offset serves as the gapless sequence).
  * **Event Sourcing / Append-Only Log**: Write the completed invoice to an append-only ledger. The sequence number is derived from the contiguous row ID or log offset. If a failure occurs, it happens before writing to the ledger, preventing gaps.
  * **Fallback / Gap Reconciliation**: If a sequence number is pulled but the transaction fatally crashes before completion, a reconciliation worker must mark the missing sequence number as a "Voided" or "System Canceled" invoice to legally satisfy the gapless requirement in accounting systems.

## 3. Row-Level Data Isolation

Preventing cross-tenant data leaks and subsidiary-level permissions leaks is critical in a multi-tenant enterprise system.

### 3.1. Database Isolation
* **Requirement**: Strict isolation of tenant data at the storage layer.
* **Architecture**:
  * **Row-Level Security (RLS)**: Enforce PostgreSQL RLS (or equivalent) natively at the database level. Every query must pass a `tenant_id` context variable. RLS policies will reject any read or write that does not match the active tenant context.
  * **Tenant Pooling vs. Siloing**: For standard subsidiaries, pool data with RLS. For enterprise clients requiring extreme isolation, support deploying dedicated database instances or schemas (Silo model).

### 3.2. Search and Analytics Isolation
* **Requirement**: Search engines (e.g., Elasticsearch) and analytics data warehouses must not leak data across boundaries.
* **Architecture**:
  * **Index per Tenant / Routing**: Use tenant-specific routing in Elasticsearch to ensure queries only hit the tenant's specific shard.
  * **Injected Filters**: All search queries must be intercepted at the API gateway or middle tier, forcibly injecting the `tenant_id` filter into the abstract syntax tree of the query before it reaches the search engine.
  * **Analytics Views**: Expose analytics to tenants via strictly secured, parameterized views that require a tenant context, rather than giving direct table access.

## 4. Timezone Reconciliation

Global subsidiaries operate across disparate timezones and calendar/fiscal years, complicating reconciliation and reporting.

### 4.1. Timestamp Handling
* **Requirement**: Accurate chronological ordering and local-time reporting.
* **Architecture**:
  * **Standardized Persistence**: Store all timestamps in standard UTC (`TIMESTAMP WITH TIME ZONE` in PostgreSQL). Absolutely no local times should be persisted for transactional data without a timezone identifier.
  * **Contextual Metadata**: Alongside the UTC timestamp, store the originating timezone string (e.g., `America/New_York`) to accurately reconstruct the exact local wall-clock time as the user experienced it, which is vital for legal disputes.

### 4.2. Fiscal Calendar Management
* **Requirement**: Financial reporting must align with varying fiscal calendars across subsidiaries.
* **Architecture**:
  * **Application-Layer Resolution**: Do not handle fiscal years in the database schema. Resolve fiscal periods dynamically at the application or reporting layer using a Fiscal Calendar configuration table.
  * **Pre-calculated Dimensions**: For data warehouses, use a Time Dimension table that maps standard UTC dates to the respective fiscal periods for each subsidiary to enable high-performance analytics.
