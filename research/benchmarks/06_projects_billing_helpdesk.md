# Projects, Professional Services Automation (PSA) & Helpdesk

## 1. Benchmark Overview
This document analyzes the capabilities of **Jira/Linear**, **FinancialForce/Certinia**, **Zendesk**, **Zoho Projects**, and **Odoo Desk** across project management, resource allocation, billing, and customer support lifecycles.

---

## 2. Project Work Breakdown Structure (WBS)

A robust WBS organizes work into manageable sections, providing a clear roadmap for project execution.

### Key Capabilities
*   **Hierarchy:** Epics (large initiatives) -> Tasks (specific deliverables) -> Sub-tasks (granular work items).
*   **Dependencies:**
    *   **Finish-to-Start (FS):** Task B cannot start until Task A finishes (Most common).
    *   **Start-to-Start (SS):** Task B cannot start until Task A starts.
    *   **Finish-to-Finish (FF):** Task B cannot finish until Task A finishes.
    *   **Start-to-Finish (SF):** Task B cannot finish until Task A starts (Rare).
*   **Milestones:** Zero-duration tasks signifying major project achievements or phases.

### Platform Analysis
*   **Jira & Linear:** Highly optimized for Agile development. They excel at Epic/Task/Sub-task hierarchies and sprint planning. Dependencies are often treated as "blocks" or "relates to" rather than strict Gantt-style CPM (Critical Path Method) dependencies.
*   **Zoho Projects:** Provides a traditional, robust WBS. It includes native Gantt charts that fully support FS, SS, FF, and SF dependencies, alongside strict milestone tracking.

---

## 3. Time Tracking & Resource Allocation

Effective professional services rely on accurately capturing time spent and optimizing how resources are deployed.

### Key Capabilities
*   **Timesheet Logging:** Capturing hours against specific tasks, projects, or non-billable administrative buckets.
*   **Billing Rates:** Multi-tiered rate cards (e.g., per role, per specific resource, or negotiated customer rates).
*   **Utilization Percentages:** Tracking billable vs. non-billable hours against a resource's total capacity to determine efficiency (e.g., Target 80% billable utilization).

### Platform Analysis
*   **FinancialForce/Certinia:** Built natively on Salesforce, it offers enterprise-grade resource management. It handles complex role-based and customer-specific rate cards and provides deep analytics on forecast vs. actual utilization.
*   **Zoho Projects & Odoo:** Both offer built-in timers and manual timesheet entry directly tied to tasks, flowing seamlessly into project profitability reports.

---

## 4. PSA Billing Models

Professional Services Automation (PSA) requires flexible billing mechanisms that integrate directly with the general ledger and Accounts Receivable (AR).

### Key Capabilities
*   **Time & Materials (T&M):** Billing customers for the actual hours worked and expenses incurred, typically on a recurring schedule (e.g., monthly).
*   **Fixed-Bid / Milestone Billing:** Invoicing a predetermined amount upon reaching specific project milestones, regardless of the hours logged.
*   **Retainer Billing:** Customers pay a recurring fee for a set amount of hours or availability. Overages may be billed at T&M rates.
*   **AR Integration:** Approved timesheets and milestones directly trigger the creation of AR Invoices without manual data entry.

### Platform Analysis
*   **FinancialForce/Certinia:** The gold standard for PSA billing. It automates revenue recognition and handles all billing models (T&M, Fixed, Retainer) by directly generating AR invoices within the Salesforce ecosystem.
*   **Odoo:** Outstanding integrated flow. Timesheets logged on a task automatically update the delivered quantities on the linked Sales Order, allowing the finance team to trigger an invoice with one click.

---

## 5. Customer Support Lifecycle

The helpdesk bridges the gap between post-sale support and ongoing customer success.

### Key Capabilities
*   **Multi-channel Capture:** Consolidating tickets from email, web portals, live chat, telephony, and social media.
*   **SLA Timers:** Tracking metrics like First Response Time (FRT) and Total Resolution Time against customer entitlements.
*   **Escalation Rules:** Automated workflows that notify management or re-assign tickets if SLAs are at risk of being breached.
*   **Categorization & Linking:** Tagging tickets by product, issue type, and seamlessly linking them to backend ERP entities (e.g., Sales Orders, Return Merchandise Authorizations - RMAs).

### Platform Analysis
*   **Zendesk:** The industry leader in pure-play helpdesk. It excels at multi-channel omnichannel routing, granular SLA policies, and complex macro/automation escalation rules.
*   **Odoo Desk / Helpdesk:** While offering solid SLA and ticketing features, its main differentiator is native ERP integration. A support agent can easily view the customer's original Sales Order or trigger an RMA/Refund directly from the ticket view.

---

## 6. Schema Definitions

Below are normalized schema definitions incorporating best practices across these platforms.

### 6.1. Projects Table

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `project_id` | UUID (PK) | Unique identifier for the project. |
| `name` | VARCHAR | Name of the project. |
| `client_id` | UUID (FK) | Reference to the Customer/Account. |
| `status` | ENUM | e.g., Draft, Active, On Hold, Completed, Cancelled. |
| `billing_type` | ENUM | T&M, Fixed-Bid, Retainer, Internal/Non-Billable. |
| `budget_amount` | DECIMAL | Total budget (if fixed bid or capped). |
| `start_date` | DATE | Projected or actual start date. |
| `end_date` | DATE | Projected or actual end date. |
| `manager_id` | UUID (FK) | Reference to the Employee managing the project. |

### 6.2. Tasks Table (WBS)

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `task_id` | UUID (PK) | Unique identifier for the task. |
| `project_id` | UUID (FK) | Project this task belongs to. |
| `parent_task_id` | UUID (FK) | Self-referencing FK for sub-tasks/epics. |
| `title` | VARCHAR | Title of the task. |
| `type` | ENUM | Epic, Task, Sub-task, Milestone. |
| `status` | ENUM | To Do, In Progress, Review, Done. |
| `assignee_id` | UUID (FK) | Reference to the Employee assigned. |
| `estimated_hours` | DECIMAL | Planned effort in hours. |
| `start_date` | DATETIME | Scheduled start time. |
| `due_date` | DATETIME | Scheduled completion time. |
| `dependency_type` | ENUM | FS, SS, FF, SF (applies if related to another task). |
| `depends_on_task_id`| UUID (FK) | The predecessor task this task depends on. |

### 6.3. Timesheets Table

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `timesheet_id` | UUID (PK) | Unique identifier for the time entry. |
| `employee_id` | UUID (FK) | The resource logging the time. |
| `task_id` | UUID (FK) | The task the time was spent on. |
| `project_id` | UUID (FK) | (Optional/Denormalized) For easier querying. |
| `date_logged` | DATE | Date the work was performed. |
| `hours` | DECIMAL | Number of hours logged. |
| `is_billable` | BOOLEAN | Whether this time is chargeable to the client. |
| `billing_rate` | DECIMAL | The resolved hourly rate for this entry. |
| `status` | ENUM | Draft, Submitted, Approved, Invoiced. |
| `invoice_id` | UUID (FK) | Link to the AR Invoice if billed. |

### 6.4. Tickets Table (Helpdesk)

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `ticket_id` | UUID (PK) | Unique identifier for the ticket. |
| `subject` | VARCHAR | Short description/subject line. |
| `description` | TEXT | Full details of the customer issue. |
| `customer_id` | UUID (FK) | Reference to the Customer/Account. |
| `contact_id` | UUID (FK) | Specific person who reported the issue. |
| `channel` | ENUM | Email, Web, Chat, Phone, Social. |
| `status` | ENUM | New, Open, Pending, Resolved, Closed. |
| `priority` | ENUM | Low, Normal, High, Urgent. |
| `category` | VARCHAR | E.g., Billing, Technical Support, Returns. |
| `assignee_id` | UUID (FK) | Support agent assigned. |
| `first_response_due`| DATETIME | SLA timer for initial response. |
| `resolution_due` | DATETIME | SLA timer for final resolution. |
| `sales_order_id` | UUID (FK) | Link to related ERP entity (if applicable). |
| `rma_id` | UUID (FK) | Link to related Return authorization (if applicable). |
