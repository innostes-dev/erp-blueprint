# Statutory Rules, Tax Law Specifications & Compliance Master Data
## Exhaustive Regulatory Reference for India (Immediate) and US (Migration Architecture)

---

### Executive Summary & Architecture Invariant
This document serves as the regulatory data dictionary, state-machine matrix, and algorithmic baseline for the autonomous compliance bots. All bots reference the statutory thresholds, formulas, schemas, and penalty logic documented below.

---

## PART 1: INDIA STATUTORY COMPLIANCE SPECIFICATIONS (CURRENT)

### 1. Goods & Services Tax (GST) Architecture

#### 1.1 Tax Rates & HSN/SAC Master Structure
| Tax Slab | Typical Goods (HSN) | Typical Services (SAC) | Reverse Charge (RCM) Flag |
| :--- | :--- | :--- | :--- |
| **0% (Exempt)** | Unprocessed food, basic grains, books (HSN 01-10) | Public health, education (SAC 9992, 9993) | No |
| **5%** | Essential raw materials, transport (HSN 11-20) | Goods Transport Agency (GTA - SAC 9965) | Mandated RCM if GTA opts 5% |
| **12%** | Processed agro, computers, medical equipment | Hotel accommodation < INR 7,500/night | Optional depending on provider |
| **18%** | SaaS, IT services, enterprise software, industrial parts | Software, Consulting, Legal (SAC 9982, 9983) | Legal counsel mandated under RCM |
| **28%** | Luxury goods, motor vehicles, gambling | Online gaming, specified entertainment | No |
| **Compensation Cess** | Automobiles, aerated beverages, tobacco | N/A | Added on top of 28% base |

#### 1.2 Place of Supply & Inter vs. Intra-State Resolution Engine
The statutory bot resolves tax splits using Sections 10, 12, and 13 of the IGST Act:
* **Rule:** If `Supplier_State_Code == Place_Of_Supply_State_Code`:
  * Split into equal halves: `CGST = Tax_Rate / 2` and `SGST = Tax_Rate / 2`.
* **Rule:** If `Supplier_State_Code != Place_Of_Supply_State_Code`:
  * Single levy: `IGST = Tax_Rate`.
* **Goods Rule (Section 10):** Location of goods at the time delivery terminates for movement to the buyer.
* **Services Rule (Section 12):** 
  * If B2B (registered recipient): Location of recipient.
  * If immovable property (SAC 9972): Location where property is situated.

#### 1.3 Government E-Invoicing & IRN Generation Specifications
* **Statutory Mandate:** All B2B, B2G, and Export invoices from entities with aggregate annual turnover > INR 5 Crore.
* **Payload Invariants for NIC/GSTN API (Version 1.03):**
  * `Version`: `"1.1"`
  * `TranDtls`: `TaxSch` ("GST"), `SupTyp` ("B2B"/"B2G"/"EXPWP"/"EXPWOP"), `RegRev` ("Y"/"N" for RCM).
  * `DocDtls`: `Typ` ("INV"/"CRN"/"DBN"), `No` (unique within fiscal year, alphanumeric, max 16 chars), `Dt` ("DD/MM/YYYY").
  * `SellerDtls`: `Gstin`, `LglNm`, `TrdNm`, `Addr1`, `Loc`, `Pin`, `Stcd`.
  * `BuyerDtls`: `Gstin`, `LglNm`, `Pos` (Place of Supply state code).
  * `ValDtls`: `AssVal`, `CgstVal`, `SgstVal`, `IgstVal`, `CesVal`, `RndOffAmt`, `TotInvVal`.
* **IRN Cryptographic Hash Calculation:**
  $$\text{IRN} = \text{SHA-256}(\text{Supplier\_GSTIN} + \text{Fiscal\_Year} + \text{Doc\_Type} + \text{Doc\_Number})$$
* **Signed QR Code Verification:** Contains digital signature of NIC, Seller GSTIN, Buyer GSTIN, Invoice Number, Invoice Date, Total Value, Item Count, HSN of top item, and IRN.

---

### 2. Government Tax Credit Reconciliation (GSTR-2B vs. Books)

#### 2.1 The GSTR-2B Data Feed Structure
The autonomous reconciliation bot ingests the official government GSTR-2B auto-drafted ITC statement with the following record structure:

```json
{
  "inv": [
    {
      "ctin": "27AAACB1234F1Z1",
      "inum": "INV/2026/0491",
      "idt": "14-04-2026",
      "val": 118000.00,
      "pos": "27",
      "rchrg": "N",
      "inv_typ": "R",
      "itcavl": "Y",
      "rsn": "",
      "diffprcnt": 0.0,
      "items": [
        {
          "num": 1,
          "txval": 100000.00,
          "iamt": 0.00,
          "camt": 9000.00,
          "samt": 9000.00,
          "csamt": 0.00
        }
      ]
    }
  ]
}
```

#### 2.2 ITC Discrepancy Categorization & Escrow Hold Rules
1. **Match State `PERFECT_MATCH`:**
   * Tax ID + normalized invoice number + tax amounts match within INR 1.00 tolerance.
   * Action: Mark ITC as `LOCKED_CLAIMABLE`, release invoice for full payment.
2. **Match State `VENDOR_UNFILED`:**
   * Invoice exists in internal Purchase Register, but absent in GSTR-2B after supplier's filing deadline (11th or 13th of following month).
   * Action:
     * Base amount scheduled for payment.
     * Tax amount moved to `ESCROW_TAX_HOLD`.
     * Automated legal notice sent to vendor.
3. **Match State `INELIGIBLE_UNDER_17_5`:**
   * Invoices containing blocked credits under Section 17(5) (e.g., motor vehicles, food & beverages, personal consumption, club memberships).
   * Action: Force tax amount to be capitalized into expense/asset; prevent ITC ledger posting.

---

### 3. Statutory MSME Rule & Section 43B(h) Countdown Watchdog

#### 3.1 Classification Criteria (MSMED Act 2006)
| Category | Plant & Machinery / Equipment Investment | Annual Turnover Slab | Covered under Section 43B(h)? |
| :--- | :--- | :--- | :--- |
| **Micro Enterprise** | $\le \text{INR 1 Crore}$ | $\le \text{INR 5 Crore}$ | **YES** |
| **Small Enterprise** | $\le \text{INR 10 Crore}$ | $\le \text{INR 50 Crore}$ | **YES** |
| **Medium Enterprise**| $\le \text{INR 50 Crore}$ | $\le \text{INR 250 Crore}$ | **NO** (Exempt from 43B(h)) |
| **Traders / Retailers**| Retail & Wholesale Trade | Any | **NO** (Exempt per MSME OM 5/2/2021) |

#### 3.2 Statutory Payment Timelines & Clock Triggers
* **Agreement in Writing Exists:** Payment must be executed within **45 calendar days** from the Day of Acceptance.
* **No Written Agreement:** Payment must be executed within **15 calendar days** from the Day of Acceptance.
* **Day of Acceptance:** The actual physical delivery of goods or completion of service (GRN timestamp). If an objection is raised in writing within 15 days, the clock restarts when the dispute is resolved.

#### 3.3 Penalties for Delayed Settlement
1. **Tax Expense Disallowance (Income Tax Act Section 43B(h)):**
   * If not paid within the statutory window before fiscal year-end (March 31), the entire expense is disallowed from taxable deductions for that year.
   * Disallowance adds directly to corporate taxable income:
     $$\text{Immediate Tax Penalty} = \text{Invoice Amount} \times \text{Corporate Tax Rate (25.17\% or 34.94\%)}$$
2. **Compound Punitive Interest (Section 16 MSMED Act):**
   * Mandatory compound interest with monthly rests at **3 times the RBI Bank Rate** from the appointment date until full settlement.
   * Interest paid is strictly **non-deductible** under Income Tax law.

---

### 4. Withholding Tax (TDS / TCS) Statutory Tables

| Section | Nature of Payment | Recipient Type | Threshold | Statutory TDS Rate | Higher Rate (Sec 206AB Non-Filer) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **194C** | Contractor / Sub-contractor payments | Individual / HUF | INR 30k single / 100k aggregate | 1.0% | 5.0% |
| **194C** | Contractor / Sub-contractor payments | Company / Partnership Firm | INR 30k single / 100k aggregate | 2.0% | 5.0% |
| **194J(a)**| Technical services / Call center | Any resident | INR 30,000 / year | 2.0% | 5.0% |
| **194J(b)**| Professional fees (Legal, CA, Architect) | Any resident | INR 30,000 / year | 10.0% | 20.0% |
| **194I(a)**| Rent on Plant, Machinery, Equipment | Any resident | INR 240,000 / year | 2.0% | 5.0% |
| **194I(b)**| Rent on Land, Building, Furniture | Any resident | INR 240,000 / year | 10.0% | 20.0% |
| **194Q** | Purchase of Goods (Buyer Turnover > 10 Cr) | Any resident vendor | Spend > INR 50 Lakhs in FY | 0.1% on excess | 5.0% |
| **206C(1H)**| TCS on Sale of Goods | Any buyer | Receipts > INR 50 Lakhs in FY | 0.1% on excess | 5.0% |

---

## PART 2: UNITED STATES COMPLIANCE & MIGRATION SPECIFICATIONS (NEXT)

### 1. US Sales & Use Tax Nexus Architecture

#### 1.1 Economic Nexus Thresholds (Post-South Dakota v. Wayfair)
* **General Standard:** Any business exceeding **$100,000 in gross sales** or **200 discrete transactions** in a state within the current or preceding calendar year triggers economic nexus.
* **Sourcing Rules:**
  * **Destination-Based (Majority of States):** Tax calculated based on buyer delivery address (State + County + City + Special District).
  * **Origin-Based (e.g., Texas, California in-state, Pennsylvania):** Tax calculated based on seller location.
* **Exemption Certificates:** Validates Multistate Tax Commission (MTC) or Streamlined Sales Tax (SST) resale certificates to prevent illegal tax exemption.

#### 1.2 Form 1099 Withholding & IRS Information Returns
* **Form 1099-NEC (Non-Employee Compensation):**
  * Mandated for any unincorporated service contractor paid $\ge \text{\$600}$ in a calendar year.
  * Direct W-9 ingestion and IRS TIN Matching API validation.
* **Backup Withholding (IRC Section 3406):**
  * If vendor fails to provide a verified TIN/EIN, the bot automatically institutes **24% Backup Withholding** and remits directly to the IRS.

### 2. US Prompt Payment Act (31 U.S.C. Chapter 39)
* **Statutory Requirement:** Mandates payment to federal/state contractors within **30 days** of receiving a proper invoice or acceptance of goods.
* **Penal Interest Rate:** Calculates Prompt Payment Interest based on the rate published semiannually by the Secretary of the Treasury under the Contract Disputes Act.

### 3. Sarbanes-Oxley (SOX 404) & Internal Financial Controls (IFC)
* **Segregation of Duties (SoD) Conflict Matrix:**
  * Level 1: Role Conflict (`AP Clerk` cannot approve `Disbursement Batch`).
  * Level 2: Entity Conflict (`Vendor Creator` cannot approve `Purchase Order`).
  * Level 3: Dual Authorization required on disbursements $> \text{\$25,000}$.
* **Cryptographic Journal Integrity:** Every journal line is sealed into an append-only Merkle DAG / SHA-256 hash sequence.

---

## PART 3: ASC 606 & IND AS 115 REVENUE RECOGNITION SPECIFICATION

Both US GAAP (ASC 606) and Indian Accounting Standards (Ind AS 115) follow the unified 5-step framework:
1. **Step 1: Identify Contract:** Multi-party agreement with enforceable rights and commercial substance.
2. **Step 2: Identify Performance Obligations (POBs):** Distinct goods or services promised.
3. **Step 3: Determine Transaction Price:** Fixed price plus variable consideration (discounts, rebates).
4. **Step 4: Allocate Price to POBs:** Relative Standalone Selling Price (SSP) method.
5. **Step 5: Recognize Revenue:** 
   * *Point-in-Time:* When customer obtains control (delivery/sign-off).
   * *Over-Time:* Input method (hours expended) or output method (milestones achieved).

$$\text{Monthly Recognized Revenue} = \frac{\text{Eligible Days in Period}}{\text{Total Performance Period Days}} \times \text{Allocated POB Price}$$
