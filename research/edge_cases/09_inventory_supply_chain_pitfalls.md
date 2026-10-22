# Inventory Discrepancy & Supply Chain Edge Cases

This document catalogs the operational failures and edge cases that corrupt inventory numbers and warehouse operations, detailing how to architect systems to handle them effectively.

## 1. Negative Inventory
**The Problem:**
Negative inventory typically occurs due to race conditions between the billing/sales system and physical dispatch. If the system does not properly reserve inventory when an order is placed, or if concurrent transactions attempt to deduct from the same pool of available stock without adequate locking, the recorded stock can drop below zero. This disconnects the system's ledger from the physical reality of the warehouse.

**The Solution:**
To prevent negative inventory, a strict state machine is required to track the lifecycle of goods. Inventory quantities should be divided into distinct buckets (e.g., `On Hand`, `Allocated/Reserved`, `Available`).

*State Machine Overview:*
1. **Available:** Goods are physically present and ready to be sold.
2. **Allocated/Reserved:** An order is placed, moving goods from `Available` to `Allocated`. The goods are still physically in the warehouse but cannot be sold to another customer.
3. **Picked/Packed:** Goods are being prepared for dispatch.
4. **Dispatched/Shipped:** Goods leave the warehouse, and the `On Hand` and `Allocated` quantities are decremented.

By enforcing that allocations can only happen if `Available >= requested_quantity`, negative inventory is fundamentally blocked at the order placement stage.

## 2. In-Transit Inventory
**The Problem:**
When goods are shipped between internal warehouses (e.g., Warehouse A to Warehouse B), there is a transport window where the goods have left A but have not yet arrived at B. If the system simply deducts from A and waits to add to B upon receipt, the goods effectively "vanish" from the company's valuation ledgers during transit, creating artificial dips in total asset valuation.

**The Solution:**
Introduce a virtual "In-Transit" warehouse or location.
* When goods are dispatched from Warehouse A, they are transferred to the `In-Transit` location.
* The company's total ledger still includes the value of goods at this `In-Transit` location.
* Upon arrival at Warehouse B, the goods are received from `In-Transit` into Warehouse B.
This ensures continuous financial visibility of assets throughout the supply chain.

## 3. Landed Cost Adjustments
**The Problem:**
Landed costs (freight, customs, insurance) often arrive via invoices weeks or months after the goods have been received, consumed, or sold. If these costs are simply expensed, the Cost of Goods Sold (COGS) and inventory valuation are inaccurate. Recalculating COGS retroactively is challenging once the original batch of goods has already moved through the supply chain.

**The Solution:**
The system must support retroactive landed cost allocation and COGS recalculation.
1. **Accruals (Estimates):** Upon receiving goods, the system should apply an estimated landed cost to the inventory valuation using an accrual account.
2. **Actuals and Variance:** When the actual invoice arrives, the variance between the accrued cost and the actual cost is calculated.
3. **Retroactive Adjustment:**
   - For goods still in stock, the inventory value is adjusted by the variance.
   - For goods already sold, the variance is posted as a retroactive adjustment to the COGS account for the period the goods were sold (or the current period, depending on accounting policies).

## 4. Concurrent Order Picking
**The Problem:**
In a busy warehouse, two pickers might receive instructions to pick the same specific serial number or batch simultaneously. If the system doesn't enforce strict concurrency control, both might scan the same item, leading to a race condition where one picker believes they have fulfilled their task, but the inventory is misallocated or double-counted.

**The Solution:**
To solve this, the system must employ explicit transactional locking when a picker claims an item.
* **Optimistic Locking:** Include a version number on the inventory record. When Picker A and Picker B read the record, they see version 1. When Picker A successfully picks the item, the version increments to 2. When Picker B attempts to submit the pick, the system detects the version mismatch and rejects the action, prompting Picker B to select a different serial/batch.
* **Pessimistic Locking:** Using `SELECT ... FOR UPDATE` (or database equivalent) when a picker attempts to claim the serial/batch. This locks the row, forcing the second picker's request to wait or fail immediately.

## 5. UOM Conversion Rounding
**The Problem:**
Supply chains often deal with multiple Units of Measure (UOM) for the same item (e.g., buying raw materials in tonnes, storing them in kilograms, and consuming/selling them in grams or milligrams). Converting between these units using floating-point arithmetic introduces rounding errors. Over thousands of transactions, these microscopic discrepancies compound, leading to "ghost" inventory—fractions of a unit that exist in the database but cannot be physically handled or cleared.

**The Solution:**
1. **Base Storage Unit:** Define the smallest possible indivisible unit as the base UOM (e.g., grams or milligrams) in the database.
2. **Fixed-Point Arithmetic:** Use fixed-point decimals (e.g., `DECIMAL(19,4)` or `DECIMAL(19,6)` in SQL) or integer types (representing the smallest unit, like cents for currency) instead of floats.
3. **Conversion Enforcement:** All transactions in alternate UOMs (e.g., purchasing in tonnes) must be immediately converted and stored in the base UOM. Rounding residuals must be handled explicitly—typically by applying the rounding difference to a specific designated variance account or item lot, rather than leaving infinitesimal fractions on the balance.

## 6. Data Locking & Transactional Isolation Mechanisms
Handling supply chain edge cases requires strict database constraints to guarantee ACID properties during high-concurrency operations.

**Transactional Isolation Levels:**
* **`READ COMMITTED`:** Generally sufficient for standard queries to prevent dirty reads, but insufficient for allocating inventory, as it does not prevent phantom reads or non-repeatable reads during complex state transitions.
* **`SERIALIZABLE` or `REPEATABLE READ`:** Essential for critical operations like inventory allocation and order picking. `SERIALIZABLE` ensures that concurrent transactions behave as if executed sequentially, eliminating race conditions that lead to negative inventory.

**Data Locking Strategies:**
* **Pessimistic Locking (`SELECT ... FOR UPDATE`):**
  Use this for exact serial/batch claiming during order picking or dispatch. It prevents other transactions from modifying the record until the current transaction commits.
  ```sql
  BEGIN TRANSACTION;
  SELECT quantity FROM inventory
  WHERE item_id = 123 AND location_id = 456
  FOR UPDATE;
  -- Deduct quantity and update state
  UPDATE inventory SET quantity = quantity - 1 WHERE item_id = 123;
  COMMIT;
  ```
* **Optimistic Concurrency Control (OCC):**
  Use this for high-throughput environments where conflicts are rare. It relies on a `version` or `updated_at` column.
  ```sql
  UPDATE inventory
  SET quantity = quantity - 1, version = version + 1
  WHERE item_id = 123 AND version = 5;
  ```
  If the `UPDATE` returns 0 rows affected, a concurrent modification occurred, and the application must retry or abort the transaction.

* **Constraint-Based Enforcement:**
  Enforce business rules directly in the schema. For example, a check constraint to inherently prevent negative physical stock:
  ```sql
  ALTER TABLE inventory ADD CONSTRAINT chk_positive_qty CHECK (quantity >= 0);
  ```
  This serves as the ultimate fail-safe against race conditions that might bypass application-level locks.
