# Google Jules Research Tasks: 3rd-Party Startup Integrations & FinOps Engine
## Production Engineering Prompts for Jules to Research and Finalize Integration Specs

This document defines four focused research and architecture tasks formatted for **Google Jules** to research, benchmark, and finalize the exact API contracts, webhook listeners, and data pipelines for our 3rd-party startup integrations.

---

### Task 1: Research & Benchmark Identity & SaaS Seat Utilization APIs

```yaml
Task Title: "research(integrations): benchmark google workspace, slack, and github apis for saas seat optimization"
Labels: ["research", "integrations", "identity", "saas-optimization", "agent-task"]
Assigned Agent: "Jules"
```

#### Context & Objective:
Reference `research/integrations/23_startup_third_party_tool_landscape.md`. Startups waste thousands of dollars per month on unused SaaS seats. We need a detailed technical architecture report exploring how our platform can connect to Google Workspace Admin SDK, Slack SCIM APIs, and GitHub Organization APIs to audit user activity, detect zombie licenses, and automate offboarding.

#### Deliverables & Scope:
1. **API Feasibility & Permission Matrix (`research/integrations/specs/identity_apis.md`):**
   * Document exact OAuth 2.0 scopes, rate limits, and service-account requirements for:
     * Google Workspace (Directory API & Reports API - User Usage Report).
     * Slack (SCIM 2.0 API, `admin.users.list`, and Audit Logs API).
     * GitHub (REST & GraphQL Organization Member API, SAML SSO mapping).
2. **Activity & Inactivity Heuristics:**
   * Define algorithms to determine "Idle/Zombie" status (e.g. no login for >45 days, no Git commits for >60 days).
3. **Automated Deprovisioning State Machine:**
   * Design the sequence diagram for when an employee is marked `TERMINATED` in the HRMS, triggering sequential webhook revocations across all linked systems.

#### Acceptance Criteria:
* Must produce concrete JSON request/response payloads for each API.
* Must include rate-limit handling strategies (e.g., token bucket backoff).

---

### Task 2: Research & Benchmark Startup Banking & Spend Management APIs

```yaml
Task Title: "research(integrations): benchmark mercury, brex, and razorpayx apis for real-time spend control"
Labels: ["research", "integrations", "fintech", "banking", "agent-task"]
Assigned Agent: "Jules"
```

#### Context & Objective:
Reference `research/integrations/23_startup_third_party_tool_landscape.md`. Tech startups use modern banks (Mercury, Brex in the US; RazorpayX in India). We need to architect real-time transaction streaming and virtual card management that locks spend per SaaS vendor.

#### Deliverables & Scope:
1. **Fintech Banking API Specifications (`research/integrations/specs/banking_apis.md`):**
   * Document webhook architectures and event schemas for:
     * Mercury API (`transaction.created`, `transaction.cleared`).
     * Brex Spend Management API (Card creation, transaction webhooks, receipt upload endpoint).
     * RazorpayX Payouts & Corporate Expense Cards API (India).
2. **Vendor-Locked Virtual Card Architecture:**
   * Design the schema and lifecycle for issuing programmatic virtual cards tied to specific vendors (e.g., "Figma-Only Card" with fixed monthly limit and expiration date).
3. **Automated Receipt Matching Pipeline:**
   * Workflow connecting incoming SMS/email card alerts to automated OCR parsing and GL journal creation.

#### Acceptance Criteria:
* Document multi-currency ledger implications (USD, INR, EUR) including automated FX fee posting.
* Guarantee zero-polling webhook latency: all events processed in <2 seconds.

---

### Task 3: Research & Benchmark Cloud Infrastructure & AI Compute FinOps Ingestion

```yaml
Task Title: "research(integrations): benchmark aws cur, gcp bigquery, and openai api usage pipelines"
Labels: ["research", "integrations", "finops", "cloud-billing", "agent-task"]
Assigned Agent: "Jules"
```

#### Context & Objective:
Reference `research/integrations/23_startup_third_party_tool_landscape.md`. Cloud infrastructure and AI token consumption are a startup's biggest variable cost. We must architect automated ingestion pipelines to bring daily infrastructure costs into the general ledger.

#### Deliverables & Scope:
1. **Cloud Billing Pipeline Specification (`research/integrations/specs/cloud_finops_apis.md`):**
   * Document data ingestion pipelines for:
     * AWS Cost & Usage Reports (CUR) in Amazon S3 (Parquet/Gzip format).
     * Google Cloud Billing export to BigQuery.
     * OpenAI Organization Usage & Cost APIs (daily token breakdown by model and API key).
2. **Runaway Compute Anomaly Detector:**
   * Statistical algorithm to detect abnormal compute surges (>25% day-over-day deviation) to prevent $10,000+ accidental cloud bills.
3. **Customer Unit-Margin Allocation Model:**
   * Formula and data model to map cloud/token costs to individual customer accounts, giving founders their real Gross Margin per client.

#### Acceptance Criteria:
* High-volume throughput: must support parsing 500,000 AWS CUR lines in <10 seconds.
* Include data retention and compression strategy for long-term historical cost analysis.

---

### Task 4: Research & Benchmark Payment Gateway & Contractor Payout Engines

```yaml
Task Title: "research(integrations): benchmark stripe, razorpay, and deel apis for automated revenue & payroll sync"
Labels: ["research", "integrations", "payments", "revenue", "agent-task"]
Assigned Agent: "Jules"
```

#### Context & Objective:
Reference `research/integrations/23_startup_third_party_tool_landscape.md`. Tech startups collect subscription revenue globally via Stripe (or Razorpay in India) and pay global engineers via Deel. We need to specify the real-time sync pipelines to maintain an audit-proof ledger.

#### Deliverables & Scope:
1. **Revenue Gateway Pipeline Specification (`research/integrations/specs/payment_apis.md`):**
   * Ingestion of Stripe webhooks: `invoice.paid`, `charge.dispute.created`, `payout.paid`.
   * Unbundling of Stripe merchant processing fees (`balance_transaction` breakdown) as operational expenses.
   * Domestic Indian UPI and NetBanking reconciliation via Razorpay Webhooks.
2. **Global Contractor & Deel Ingestion:**
   * Deel API endpoints for fetching approved contractor invoices, W-8BEN compliance documents, and monthly contractor payroll summaries.
3. **Double-Entry Mapping Matrix:**
   * Automated mapping table translating Stripe/Razorpay/Deel events into balanced debit/credit journal entries.

#### Acceptance Criteria:
* Idempotency guarantee: duplicate webhook events must never produce duplicate general ledger entries.
* Exact mathematical reconciliation down to the penny/paisa across all payout batches.
