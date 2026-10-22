# CRM, CPQ & Order-to-Cash (O2C) Benchmarks

This document evaluates and extracts the commercial order-to-cash capabilities of five major platforms: **Salesforce Sales Cloud**, **HubSpot**, **NetSuite CRM**, **Zoho CRM**, and **Odoo Sales**.

## 1. Lead and Opportunity Management

### Salesforce Sales Cloud
- **Multi-Pipeline Stages:** Fully customizable sales processes and record types to support multiple pipelines.
- **Stage Probabilities:** Each stage can be mapped to a default probability for pipeline forecasting.
- **Rotary Assignment Rules:** Robust lead assignment rules using round-robin (via custom Apex/Flow or AppExchange apps) or criteria-based routing.

### HubSpot
- **Multi-Pipeline Stages:** Supports multiple deal pipelines (available in Sales Hub Starter/Professional+).
- **Stage Probabilities:** Customizable win probabilities per deal stage for revenue forecasting.
- **Rotary Assignment Rules:** Native lead rotation (round-robin) available in workflows (Sales Hub Professional/Enterprise).

### NetSuite CRM
- **Multi-Pipeline Stages:** Highly customizable statuses and stages tied to the ERP backend.
- **Stage Probabilities:** Probability percentages attached to opportunity statuses for weighted forecasting.
- **Rotary Assignment Rules:** Built-in sales territory management and routing rules.

### Zoho CRM
- **Multi-Pipeline Stages:** Multiple page layouts and pipeline tracking out of the box.
- **Stage Probabilities:** Stage-probability mapping is standard and customizable.
- **Rotary Assignment Rules:** Native assignment rules including round-robin lead routing based on criteria.

### Odoo Sales (CRM module)
- **Multi-Pipeline Stages:** Kanban-based, drag-and-drop customizable pipelines per sales team.
- **Stage Probabilities:** Auto-calculates success rate based on historical data or allows manual probability setting.
- **Rotary Assignment Rules:** Assignment rules available (can auto-assign leads based on domain or round-robin using automated actions).

---

## 2. Configure, Price, Quote (CPQ)

### Salesforce (via Salesforce CPQ)
- **Multi-Currency Price Lists:** Advanced price book management supporting global currencies.
- **Volume Break Discounts:** Tiered pricing, block pricing, and discount schedules native to CPQ.
- **Promotional Coupon Engines:** Support for promo codes and partner discounts.
- **Contract Pricing:** Systematically handles contracted prices and renewal terms.
- **Bundle Rules:** Highly advanced product bundling with validation/selection rules.
- **Margin Floor Validations:** Advanced approval rules triggered by margin thresholds.

### HubSpot
- **Multi-Currency Price Lists:** Multi-currency support in product library (Professional/Enterprise).
- **Volume Break Discounts:** Basic line-item discounts; lacks native advanced tiering without integration.
- **Promotional Coupon Engines:** Requires integration (e.g., via Stripe) for advanced coupon handling.
- **Contract Pricing:** Handled manually via properties or custom objects; limited native contract pricing.
- **Bundle Rules:** Lacks advanced native bundling (requires third-party CPQ).
- **Margin Floor Validations:** Requires custom workflow approvals based on custom margin fields.

### NetSuite CRM (ERP Integrated)
- **Multi-Currency Price Lists:** Core feature; robust multi-currency and multi-subsidiary pricing.
- **Volume Break Discounts:** Quantity-based pricing schedules and price levels.
- **Promotional Coupon Engines:** SuitePromotions offers advanced coupon and discount management.
- **Contract Pricing:** Customer-specific negotiated pricing is standard.
- **Bundle Rules:** Native kit/package and assembly items support bundling.
- **Margin Floor Validations:** Workflow-based approvals based on estimated gross profit/margin.

### Zoho CRM (via Zoho CPQ / Finance Suite)
- **Multi-Currency Price Lists:** Standard multi-currency support in price books.
- **Volume Break Discounts:** Supported via Zoho Books/Inventory integrations.
- **Promotional Coupon Engines:** Basic discounting; coupon engines better handled via Zoho Commerce.
- **Contract Pricing:** Can be managed via custom price books linked to specific accounts.
- **Bundle Rules:** Product bundling supported (often enhanced with Zoho Inventory).
- **Margin Floor Validations:** Custom functions/workflows required for margin-based approvals.

### Odoo Sales
- **Multi-Currency Price Lists:** Advanced pricelist configurations natively support multi-currency.
- **Volume Break Discounts:** Pricelists support quantity-based minimums and complex formula pricing.
- **Promotional Coupon Engines:** Native "Coupons & Promotions" program available in Sales.
- **Contract Pricing:** Pricelists can be tied directly to specific customer records.
- **Bundle Rules:** Optional products and kit/bundle configuration via BoM (Bill of Materials) in Manufacturing/Inventory.
- **Margin Floor Validations:** Margin module provides visibility; automated floor validations require automated actions/customization.

---

## 3. Order-to-Cash Lifecycle

*Lifecycle Stages: Quote -> Sales Order -> Credit Hold Verification -> Stock Allocation/Reservation -> Pick List -> Pack Slip -> Shipping Label/BOL -> Delivery Proof -> Invoice -> Revenue Recognition schedule.*

| Stage | Salesforce (w/ CPQ & Billing) | HubSpot (Native + Payments) | NetSuite | Zoho (w/ Finance Plus) | Odoo (Sales + Inventory + Accounting) |
|---|---|---|---|---|---|
| **Quote** | Native / Adv. in CPQ | Native quotes | Native estimates | Native quotes | Native quotations |
| **Sales Order** | Standard object / CPQ Order | Not natively structured as SO | Core ERP record | Native in Zoho Inventory | Confirmed Quotation = SO |
| **Credit Hold** | Requires customization | Custom properties | Native credit limits/holds | Credit limits in Zoho Books | Native credit limit warnings |
| **Stock Allocation**| Requires ERP integration | N/A | Native automatic/manual | Native in Zoho Inventory | Native reservation upon SO |
| **Pick List** | Requires ERP integration | N/A | Native | Native in Zoho Inventory | Native delivery order/pick |
| **Pack Slip** | Requires ERP integration | N/A | Native | Native in Zoho Inventory | Native |
| **Shipping/BOL** | Requires AppExchange | N/A | Native / SuiteApp | Native carrier integrations | Native carrier integrations |
| **Delivery Proof**| Requires AppExchange | N/A | Requires SuiteApp / Custom | Via carrier tracking | Via carrier tracking |
| **Invoice** | Salesforce Billing | Native (HubSpot Invoices) | Native | Zoho Books | Native Accounting |
| **Rev Rec** | Salesforce Billing | Basic via reporting | Adv. Revenue Management (ARM) | Rev Rec in Zoho Books | Deferred Revenue rules native |

*(Note: Salesforce and HubSpot typically require third-party ERP integrations for physical fulfillment and advanced revenue recognition. NetSuite, Zoho, and Odoo have these capabilities embedded in their unified suites.)*

---

## 4. Customer Returns and RMA Workflows

### Salesforce
- Uses **Return Order** and **Return Order Line Item** objects.
- Workflows for RMAs usually require Service Cloud or custom flows. Restocking fee calculations require custom Apex or formula fields.

### HubSpot
- No native RMA object. Returns are typically handled via Service Hub **Tickets** and custom properties. Restock fees require manual entry or custom calculated properties.

### NetSuite
- Native **Return Authorization (RMA)** transaction types.
- Supports receiving goods, issuing credit memos, or refunding. Restocking fees are native (can be applied as a percentage or flat rate on the RMA line).

### Zoho
- RMA managed via **Zoho Inventory** (Sales Returns).
- Can process return receipts, issue credit notes, and manage refunds. Restocking fees can be added as line items on the credit note.

### Odoo
- Native return workflows from the Delivery Order (Reverse Transfer).
- Creates a return picking. Once received, generates a draft Credit Note. Restocking fees can be manually added to the credit note or automated via custom module.

---

## 5. Document Schema and Field Mapping (Lead to Revenue)

This section maps the typical conceptual schema and required fields to transition from a Lead through to Revenue.

### Step 1: Lead $\rightarrow$ Opportunity
- **Source Object:** `Lead`
- **Target Object:** `Opportunity` / `Account` / `Contact`
- **Required Fields for Transition:**
  - `Lead Status` (Must be Qualified)
  - `Company Name` $\rightarrow$ Maps to `Account.Name`
  - `First/Last Name`, `Email` $\rightarrow$ Maps to `Contact.Name`, `Contact.Email`
  - `Estimated Value` $\rightarrow$ Maps to `Opportunity.Amount`
  - `Lead Owner` $\rightarrow$ Maps to `Opportunity.OwnerId`

### Step 2: Opportunity $\rightarrow$ Quote
- **Source Object:** `Opportunity`
- **Target Object:** `Quote` (and `Quote Line Items`)
- **Required Fields for Transition:**
  - `Opportunity ID` $\rightarrow$ Maps to `Quote.OpportunityId`
  - `Price Book ID` $\rightarrow$ Maps to `Quote.Pricebook2Id`
  - `Opportunity Line Items` $\rightarrow$ Maps to `Quote Line Items` (Product, Quantity, Sales Price, Discount)
  - `Expiration Date` $\rightarrow$ `Quote.ExpirationDate`

### Step 3: Quote $\rightarrow$ Sales Order
- **Source Object:** `Quote`
- **Target Object:** `Sales Order` (or equivalent `Order` object)
- **Required Fields for Transition:**
  - `Quote Status` (Must be Accepted)
  - `Account ID`, `Contact ID` $\rightarrow$ `Order.AccountId`, `Order.BillToContactId`
  - `Billing/Shipping Address` $\rightarrow$ `Order.BillingAddress`, `Order.ShippingAddress`
  - `Quote Line Items` $\rightarrow$ `Order Line Items`

### Step 4: Sales Order $\rightarrow$ Fulfillment (Stock Allocation $\rightarrow$ Pack $\rightarrow$ Ship)
- **Source Object:** `Sales Order`
- **Target Objects:** `Inventory Commitment`, `Pick List`, `Pack Slip`, `Shipment`
- **Required Fields for Transition:**
  - `Order Status` (Must be Approved / No Credit Hold)
  - `Product ID` & `Quantity` $\rightarrow$ `StockReservation.Quantity`
  - `Warehouse Location` $\rightarrow$ `Fulfillment.LocationId`
  - `Tracking Number` (generated post-shipment) $\rightarrow$ `Shipment.TrackingNumber`

### Step 5: Fulfillment/Sales Order $\rightarrow$ Invoice
- **Source Object:** `Sales Order` (or `Shipment`)
- **Target Object:** `Invoice`
- **Required Fields for Transition:**
  - `Proof of Delivery` or `Fulfillment Status` (Shipped)
  - `Order Line Items` $\rightarrow$ `Invoice Line Items` (Quantity Shipped, Unit Price, Tax, Total)
  - `Payment Terms` (e.g., Net 30) $\rightarrow$ `Invoice.DueDate`
  - `Tax Rate` $\rightarrow$ `Invoice.TaxAmount`

### Step 6: Invoice $\rightarrow$ Revenue Recognition
- **Source Object:** `Invoice` (and `Payment`)
- **Target Object:** `Revenue Schedule` / `General Ledger`
- **Required Fields for Transition:**
  - `Invoice Status` (Posted/Paid)
  - `Service Start Date`, `Service End Date` $\rightarrow$ Defines amortization period.
  - `Total Amount` $\rightarrow$ `RevenueSchedule.TotalAmount`
  - `Deferred Revenue Account`, `Recognized Revenue Account` $\rightarrow$ Mapped from Product/Item master.