# Product Requirements Document
# Data Curator — Unity Catalog Access Request App

**Version:** 2.1  
**Status:** Draft — In Review  
**Last Updated:** 2026-04-03

---

## Table of Contents

1. [Product Overview](#1-product-overview)
2. [Screen Inventory & User Flow](#2-screen-inventory--user-flow)
3. [Data Model — User Inputs](#3-data-model--user-inputs)
4. [Technical Architecture](#4-technical-architecture)
5. [Streamlit App Structure](#5-streamlit-app-structure)
6. [Approval Workflow](#6-approval-workflow)
7. [Microsoft Teams Integration](#7-microsoft-teams-integration)
8. [Development Plan](#8-development-plan)
9. [Testing Strategy](#9-testing-strategy)
10. [Deployment Strategy](#10-deployment-strategy)
11. [Security & Compliance](#11-security--compliance)
12. [Caching & Performance](#12-caching--performance)
13. [Error Handling](#13-error-handling)
14. [Acceptance Criteria](#14-acceptance-criteria)
15. [Non-Functional Requirements](#15-non-functional-requirements)
16. [Known Limitations / Out of Scope](#16-known-limitations--out-of-scope)
17. [Glossary / Definitions](#17-glossary--definitions)
18. [Runbook / Operations](#18-runbook--operations)

---

## 1. Product Overview

**Data Curator** is a Streamlit-based Databricks app that enables users to request, track, and manage READ access to Unity Catalog tables through a governed, auditable workflow. The app replaces ad-hoc access requests with a structured 4-step wizard, surfacing catalog inventory across environments (Dev, SIT, UAT, Prod), auto-detecting data sensitivity from table tags, and routing approvals through a combination of fixed governance reviewers and dynamically resolved data owners.

### Primary Users

| Role | Description |
|---|---|
| Requestors | Data engineers, analysts, data scientists, and BI teams requesting access to Unity Catalog assets |
| Fixed Reviewers | Data Governance team members — pre-configured in a Delta table |
| Data Owner Reviewers | Table owners auto-resolved from Unity Catalog table tag metadata |
| Admins | Configure reviewer lists, approval modes, and audit reporting |

### Key Value Propositions

- Self-service READ access requests with built-in governance
- Real-time Unity Catalog inventory browsing across Dev, SIT, UAT, and Prod via Databricks REST API
- Auto-assigned reviewers combining fixed governance roles and dynamic data owners from table tags
- Full audit trail of requests, approvals, comments, and access lifecycles
- Microsoft Teams integration with Adaptive Cards and Power Automate for full approval orchestration
- Draft persistence and multi-step wizard with inline editing

---

## 2. Screen Inventory & User Flow

### 2.1 Screens

| # | Screen | Purpose |
|---|---|---|
| 1 | **Request Overview — Requestor Dashboard** | Landing page for requestors: status metrics (Pending / Approved / Expired), recent requests list, entry point for new requests |
| 2 | **Reviewer Home** | Landing page for reviewers: applications awaiting their review with status and priority indicators |
| 3 | **Data Selection (Step 1)** | Browse/search Unity Catalog tables across Dev, SIT, UAT, Prod; select datasets; view metadata (size via `information_schema`, sync status, data sensitivity from tags) |
| 4 | **Requestor Information (Step 2)** | Collect name, department, and purpose of access |
| 5 | **Access & Planning (Step 3)** | Set access start/end dates; document long-term planning and future project context |
| 6 | **Review & Confirm (Step 4)** | Summary of all entered data with inline edit links, compliance attestation checkboxes, and final submit action |
| 7 | **Request Detail / Review Page** | Shared view for requestors (read-only) and reviewers (approve / comment / request clarification) |
| 8 | **Admin — Reviewer Management** | Admin screen to manage the fixed reviewer list: add / deactivate / edit reviewers, configure sequence order, view Teams identity mapping |
| 9 | **Admin — All Requests View** | Admin dashboard showing all requests across all requestors with filters by status, environment, date range, and department |
| 10 | **Admin — Audit & Compliance Reports** | Admin screen for running audit queries, exporting compliance reports, and viewing the append-only audit log |

### 2.2 User Flow — Requestor

```
Requestor Dashboard
  └─> [New Contract Request]
        └─> Step 1: Data Selection
        │     Browse UC tables (Dev/SIT/UAT/Prod)
        │     Toggle-select tables, view metadata + sensitivity tags
        │
        └─> Step 2: Requestor Information
        │     Name, Department, Purpose
        │
        └─> Step 3: Access & Planning
        │     Start/End dates, Long-term planning notes
        │
        └─> Step 4: Review & Confirm
              Pre-submit compliance attestations
              Submit → status = "submitted"
                └─> Reviewers notified via Teams Adaptive Card
                      └─> Status visible on Requestor Dashboard
```

### 2.3 User Flow — Reviewer

```
Reviewer Home (pending items queue)
  └─> [Open Request]
        └─> Request Detail / Review Page
              ├─> [Approve] → marks reviewer's decision as approved
              ├─> [Request Clarification] → comment added; request moves to
              │     "Awaiting Requestor Response" state
              │       └─> Requestor resolves comment or approves exception
              │             └─> Reviewer notified to continue review
              └─> [Reject] → request closed with reason
```

### 2.4 User Flow — Access Renewal / Extension

When an approved request is nearing its `access_end` date, the requestor can extend access without re-entering all information:

```
Requestor Dashboard → [Approved Requests]
  └─> [Extend Access] (available for approved requests within 30 days of expiry)
        └─> Pre-filled form: original tables (read-only), original purpose (editable),
        │   new end date picker (must be after current end date, subject to max duration cap)
        │   renewal justification textarea (required)
        └─> [Submit Extension] → creates a new access_request with status = "submitted"
              parent_request_id links back to original request
              └─> Reviewers notified; approval workflow proceeds as normal
```

An extension is a **new request** linked to the original via `parent_request_id`, not a modification of the existing approved record. This preserves the audit trail and allows reviewers to evaluate the renewal independently.

### 2.5 User Flow — Request Cancellation by Requestor

A requestor may withdraw a request that is still in `submitted`, `under_review`, or `awaiting_response` state:

```
Requestor Dashboard → [Pending Requests]
  └─> [Cancel Request] (available for non-terminal requests)
        └─> Confirmation dialog: "Are you sure? This cannot be undone."
        └─> [Confirm Cancel] → status = "cancelled"
              └─> All pending reviewer_assignments marked as "cancelled"
              └─> Reviewers notified of cancellation via Teams
              └─> Audit log entry created
```

Terminal states (`approved`, `rejected`, `expired`, `revoked`, `cancelled`) cannot be cancelled. Cancelled requests are visible on the requestor dashboard with a `CANCELLED` badge but cannot be reopened or resubmitted — the requestor must create a new request.

### 2.6 User Flow — Rejection Response

When a reviewer rejects a request, the requestor has two options:

```
Requestor Dashboard → [Rejected Requests]
  └─> View rejection reason on Request Detail page
        ├─> [Create New Request] → pre-fills Step 1–3 from the rejected request
        │     requestor can modify any field before resubmitting
        │     new request_id; no link to rejected request
        │
        └─> [Appeal] → request status moves to "appealing"
              appeal_count incremented to 1
              original reviewer(s) notified to reconsider
              └─> status transitions to "under_review"
                    └─> Reviewer can: approve (overturn), reject again (re-affirm — terminal),
                          or add a clarification comment
```

**Policy decision:** The default flow is **create new request** (fresh start). The appeal mechanism is a secondary option that only re-engages the original reviewer(s) without resetting the full workflow. Appeals are limited to **one per rejected request** — a second rejection on the same request is terminal and requires a new request.

### 2.7 Cross-Cutting Capabilities

- **Save Draft** available at every wizard step
- **Back navigation** between steps with state preserved
- **Inline edit** from Review page navigates to the relevant step
- **Department-level filtering** on requestor dashboard
- **Status badges** on all request cards: DRAFT, SUBMITTED, UNDER REVIEW, AWAITING RESPONSE, APPEALING, APPROVED, REJECTED, EXPIRED, REVOKED, CANCELLED

---

## 3. Data Model — User Inputs

### 3.1 Step 1 — Data Selection

| Field | Type | Notes |
|---|---|---|
| Selected tables | Multi-select | `{catalog, schema, table_name}` per row; browsable across Dev, SIT, UAT, Prod catalogs |
| Table size (bytes) | Auto-populated | Queried from `information_schema.tables` via SQL Execution API — not entered by user |
| Data sensitivity | Auto-populated | Detected from Unity Catalog table tags — displayed to user, not entered |

### 3.2 Step 2 — Requestor Information

| Field | Type | Required | Constraints |
|---|---|---|---|
| Name | Text | Yes | Auto-populated from SCIM where possible |
| Department | Dropdown | Yes | Populated from SCIM groups |
| Purpose of Access | Single select | Yes | Options: `Reporting`, `Analytics and Data Science`, `Data Exploration`, `Data Migration` |

### 3.3 Step 3 — Access & Planning

| Field | Type | Required | Constraints |
|---|---|---|---|
| Access Start Date | Date picker | Yes | Cannot be in the past |
| Access End Date | Date picker | Yes | Must be after start date |
| Duration (days) | Calculated | N/A | Auto-calculated and displayed |
| Long-term Planning / Future Projects | Textarea | Yes | Free-form; 50–2000 characters; describes future use context and project roadmap |

> **Access Level** is always `READ` and is set automatically — it is not presented to the user.

### 3.3.1 Maximum Access Duration Policy

The organisation enforces a **maximum access duration cap of 365 days (12 months)** from the start date. This is a hard constraint:

- The end date picker UI will not allow selection beyond `start_date + 365 days`
- Server-side validation rejects any request where `access_end - access_start > 365`
- The duration cap is configured in `config.py` as `MAX_ACCESS_DURATION_DAYS = 365` and can be overridden per-environment via DAB variables if policy changes
- Requests needing access beyond 12 months must go through a separate long-term access governance process (out of scope for this app)

### 3.3.2 Multi-Environment Grant Model

When a requestor selects the same table from multiple environments (e.g., `dev.finance.transactions` and `prod.finance.transactions`), the following rules apply:

**One request, multiple line items, per-environment approval.** A single `access_request` contains all selected tables across all environments, stored as separate rows in `request_tables`. However, approval is scoped **per environment**:

- `reviewer_assignments` includes a `catalog_name` column, so reviewers can be assigned per-environment
- Fixed reviewers review all environments; data owner reviewers are resolved per-table (a table in Dev may have a different data_owner tag than the same-named table in Prod)
- A request is not fully `approved` until **all environments** have received unanimous approval from their respective reviewers
- On approval, READ permissions are granted **independently per environment** — if Prod approval is still pending but Dev is approved, Dev access is granted immediately

This model ensures that sensitive Prod tables require their own data owner sign-off even if the same table name exists in a lower environment.

### 3.4 Unity Catalog Table Tag Schema

The app relies on Unity Catalog table tags for sensitivity classification and data owner resolution. The following tag taxonomy is expected:

| Tag Key | Purpose | Expected Values | Usage |
|---|---|---|---|
| `sensitivity` | Data classification tier | `PUBLIC`, `INTERNAL`, `CONFIDENTIAL`, `RESTRICTED` | Auto-detected and displayed as sensitivity badge; controls highlight level in UI |
| `data_owner` | Identity of the table's business owner | Email address (e.g., `alice@example.com`) | Used to resolve dynamic data owner reviewers |
| `pii` | Personally identifiable information flag | `true`, `false` | Used as secondary sensitivity indicator; tables with `pii=true` are treated as at least `CONFIDENTIAL` |

**Tag resolution rules:**

- Tags are read from `GET /api/2.1/unity-catalog/tables/{full_name}` in the `properties` map
- If the `sensitivity` tag is missing, the table is classified as `UNCLASSIFIED` — the user can proceed but is warned
- If the `sensitivity` tag has an unexpected value (not in the four valid tiers), it is treated as `UNCLASSIFIED` and an admin alert is raised
- If a table has **both** `sensitivity=PUBLIC` and `pii=true`, the higher classification (`CONFIDENTIAL`) wins
- If the `data_owner` tag is missing or the email does not match any active record in the `reviewers` table, no data owner reviewer is assigned for that table; only fixed reviewers will review. An admin alert is raised for missing data owner.
- Tag key names are configurable via `config.py` constants: `SENSITIVITY_TAG_KEY = "sensitivity"`, `DATA_OWNER_TAG_KEY = "data_owner"`, `PII_TAG_KEY = "pii"`

### 3.5 Step 4 — Review & Confirm (Pre-Submit Attestations)

Users must check all three boxes before submission is enabled:

- ☐ I agree to the organisation's data privacy policy
- ☐ I will not share credentials or export data outside approved systems
- ☐ I understand that access will be automatically revoked on the access end date

---

## 4. Technical Architecture

### 4.1 Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Streamlit (Python) |
| Backend / Data | Databricks Unity Catalog |
| API Layer | Databricks REST API |
| Approval Orchestration | Microsoft Power Automate |
| Reviewer Notifications | Microsoft Teams Adaptive Cards |
| Packaging & Deployment | Databricks Asset Bundles (DAB) |
| CI/CD | GitHub Actions |
| Testing | pytest + pytest-cov |
| Linting | ruff |

### 4.2 Architecture Diagram

```
┌──────────────────────────────────────────────────────────┐
│                      GitHub Repo                          │
│  src/  tests/  databricks.yml  .github/workflows/         │
└──────────────────────┬───────────────────────────────────┘
                       │ GitHub Actions CI/CD
                       ▼
┌──────────────────────────────────────────────────────────┐
│                  Databricks Workspace                     │
│                                                           │
│  ┌────────────────┐     ┌──────────────────────────────┐ │
│  │ Streamlit App  │◄───►│    Databricks REST API        │ │
│  │ (App Hosting)  │     │                              │ │
│  └────────────────┘     │  - Unity Catalog API         │ │
│                         │  - Permissions API            │ │
│                         │  - SQL Execution API          │ │
│                         │  - SQL Warehouses API         │ │
│                         │  - SCIM / Workspace API       │ │
│                         └──────────────┬───────────────┘ │
│                                        │                  │
│                         ┌──────────────▼───────────────┐ │
│                         │       Unity Catalog           │ │
│                         │  Dev / SIT / UAT / Prod       │ │
│                         │  - Catalogs / Schemas         │ │
│                         │  - Tables + Tags              │ │
│                         │  - Permissions                │ │
│                         └──────────────────────────────┘ │
│                                                           │
│                         ┌──────────────────────────────┐ │
│                         │   App Delta Tables            │ │
│                         │  - access_requests            │ │
│                         │  - request_tables             │ │
│                         │  - reviewer_assignments       │ │
│                         │  - review_comments            │ │
│                         │  - approvals                  │ │
│                         │  - reviewers (config)         │ │
│                         │  - notification_dlq           │ │
│                         │  - quarterly_audit_reports    │ │
│                         │  - audit_log                  │ │
│                         └──────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│               Microsoft Ecosystem                         │
│                                                           │
│  ┌──────────────────┐    ┌──────────────────────────┐   │
│  │  Power Automate  │◄──►│  Teams Adaptive Cards    │   │
│  │  (Approval       │    │  (Inline approve/reject/ │   │
│  │   Orchestration) │    │   comment for reviewers) │   │
│  └──────────────────┘    └──────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

### 4.3 Databricks REST API Endpoints

| Feature | API | Endpoint |
|---|---|---|
| List catalogs | Unity Catalog | `GET /api/2.1/unity-catalog/catalogs` |
| List schemas | Unity Catalog | `GET /api/2.1/unity-catalog/schemas` |
| List tables | Unity Catalog | `GET /api/2.1/unity-catalog/tables` |
| Get table details + tags | Unity Catalog | `GET /api/2.1/unity-catalog/tables/{full_name}` |
| Table size metadata | SQL Execution | `POST /api/2.0/sql/statements` → `information_schema.tables` |
| Grant READ permission | Permissions | `POST /api/2.0/unity-catalog/permissions/{securable_type}/{full_name}` |
| List permissions | Permissions | `GET /api/2.0/unity-catalog/permissions/{securable_type}/{full_name}` |
| Current user | SCIM | `GET /api/2.0/preview/scim/v2/Me` |
| List groups (departments) | SCIM | `GET /api/2.0/preview/scim/v2/Groups` |
| List warehouses | SQL Warehouses | `GET /api/2.0/sql/warehouses` |

### 4.4 App Data Model (Delta Tables)

All app state is persisted as Delta tables within a dedicated schema (`catalog.data_curator_app`).

**`access_requests`**
```
request_id              STRING PK
parent_request_id       STRING NULL  FK -> access_requests
                                     (for renewals, links to the original approved request;
                                      NULL for first-time requests)
status                  ENUM [draft, submitted, under_review, awaiting_response, appealing,
                               approved, rejected, cancelled, revoked, expired]
                         -- must match state machine in Section 6.3
approval_mode           ENUM [parallel, sequential]  (set at submission from admin config)
requestor_name          STRING
requestor_dept          STRING
purpose                 ENUM [reporting, analytics_and_data_science,
                               data_exploration, data_migration]
long_term_planning      TEXT (50–2000 chars)
access_start            DATE
access_end              DATE
access_level            STRING DEFAULT 'READ'
appeal_count            INT DEFAULT 0   (max 1 — enforced at service layer; prevents repeat appeals)
created_at              TIMESTAMP
updated_at              TIMESTAMP
submitted_at            TIMESTAMP
approved_at             TIMESTAMP
cancelled_at            TIMESTAMP
revoked_at              TIMESTAMP
```

**`request_tables`**
```
request_id              FK -> access_requests
catalog_name            STRING   (dev | sit | uat | prod)
schema_name             STRING
table_name              STRING
table_full_name         STRING   (catalog.schema.table)
table_size_bytes        BIGINT   (from information_schema)
sensitivity_tags        MAP<STRING,STRING>  (from UC table tags)
selected_at             TIMESTAMP
```

**`reviewers`** *(admin-maintained config table)*
```
reviewer_id             STRING PK
reviewer_name           STRING
reviewer_email          STRING
reviewer_type           ENUM [fixed, data_owner]
teams_user_id           STRING
is_active               BOOLEAN
sequence_order          INT NULL     (for sequential mode ordering)
default_approval_mode   ENUM [parallel, sequential]  (admin-configured default)
deputy_reviewer_id      STRING NULL  (FK -> reviewers; fallback if primary unavailable)
updated_at              TIMESTAMP
```

**`reviewer_assignments`**
```
assignment_id           STRING PK
request_id              FK -> access_requests
reviewer_id             FK -> reviewers
reviewer_type           ENUM [fixed, data_owner]
catalog_name            STRING NULL  (environment scope: dev | sit | uat | prod)
assigned_table          STRING NULL  (for data_owner type)
sequence_order          INT NULL     (for sequential mode)
status                  ENUM [pending, approved, rejected, awaiting_response, cancelled]
assigned_at             TIMESTAMP
decided_at              TIMESTAMP
delegated_from          STRING NULL  (original reviewer_id if reassigned)
delegation_reason       STRING NULL
```

**`review_comments`**
```
comment_id              STRING PK
request_id              FK -> access_requests
reviewer_id             FK -> reviewers
comment_text            TEXT
resolution_status       ENUM [open, resolved, exception_approved]
resolution_note         TEXT NULL
resolved_by             STRING NULL
resolved_at             TIMESTAMP NULL
created_at              TIMESTAMP
```

**`approvals`**
```
approval_id             STRING PK
request_id              FK -> access_requests
reviewer_id             FK -> reviewers
decision                ENUM [approved, rejected]
comments                TEXT NULL
decided_at              TIMESTAMP
```

**`notification_dlq`** *(dead-letter queue for failed Teams/Power Automate notifications)*
```
dlq_id                  STRING PK
request_id              FK -> access_requests
reviewer_id             STRING NULL  FK -> reviewers
event_type              STRING       (e.g., notify_reviewer, notify_requestor, callback_write)
payload                 TEXT         (JSON — the original notification payload)
error_message           TEXT
retry_count             INT DEFAULT 0
last_retry_at           TIMESTAMP NULL
resolved                BOOLEAN DEFAULT FALSE
resolved_at             TIMESTAMP NULL
created_at              TIMESTAMP
```

**`quarterly_audit_reports`**
```
report_id               STRING PK
quarter                 STRING       (e.g., '2026-Q1')
generated_at            TIMESTAMP
total_submitted         INT
total_approved          INT
total_rejected          INT
total_cancelled         INT
avg_approval_days       DECIMAL(5,2)
sla_breach_count        INT
active_grants           INT
sensitivity_breakdown   MAP<STRING,INT>   (tier -> count)
reviewer_workload       MAP<STRING,INT>   (reviewer_id -> approval_count)
tag_coverage_pct        DECIMAL(5,2)
report_csv_path         STRING NULL  (path to exported CSV in DBFS)
```

**`audit_log`** *(append-only — MERGE prohibited)*
```
event_id                STRING PK
request_id              STRING
actor                   STRING
action                  STRING
timestamp               TIMESTAMP
details                 MAP<STRING,STRING>
```

---

## 5. Streamlit App Structure

### 5.1 Project Layout

```
data_curator/
├── src/
│   ├── app.py                           # Streamlit entry point + routing
│   ├── pages/
│   │   ├── 0_requestor_dashboard.py     # Request Overview (requestor home)
│   │   ├── 1_reviewer_home.py           # Reviewer pending queue
│   │   ├── 2_data_selection.py          # Step 1
│   │   ├── 3_requestor_info.py          # Step 2
│   │   ├── 4_access_planning.py         # Step 3
│   │   ├── 5_review_confirm.py          # Step 4 (requestor)
│   │   └── 6_request_detail.py          # Shared detail/review page
│   ├── components/
│   │   ├── sidebar.py
│   │   ├── top_nav.py
│   │   ├── progress_stepper.py
│   │   ├── status_card.py
│   │   ├── contract_card.py
│   │   ├── data_table_card.py
│   │   ├── sensitivity_badge.py
│   │   ├── form_inputs.py
│   │   ├── attestation_checklist.py
│   │   ├── review_section.py
│   │   └── comment_thread.py
│   ├── api/
│   │   ├── client.py                    # Base authenticated Databricks client
│   │   ├── unity_catalog.py             # Catalog / schema / table / tags
│   │   ├── permissions.py               # Grant / revoke / list permissions
│   │   ├── sql_execution.py             # information_schema + app queries
│   │   └── workspace.py                 # SCIM user + group lookups
│   ├── services/
│   │   ├── request_service.py           # CRUD for access_requests
│   │   ├── draft_service.py             # Draft save / load / cleanup
│   │   ├── reviewer_service.py          # Reviewer assignment logic
│   │   ├── approval_service.py          # Approval state machine
│   │   ├── comment_service.py           # Comment CRUD + resolution
│   │   ├── sensitivity_service.py       # Tag-based sensitivity detection
│   │   └── teams_service.py             # Power Automate / Teams webhook calls
│   ├── utils/
│   │   ├── auth.py
│   │   ├── config.py
│   │   └── formatters.py
│   └── styles/
│       └── design_system.css
├── tests/
│   ├── unit/
│   │   ├── test_unity_catalog_api.py
│   │   ├── test_permissions_api.py
│   │   ├── test_request_service.py
│   │   ├── test_draft_service.py
│   │   ├── test_reviewer_service.py
│   │   ├── test_approval_service.py
│   │   ├── test_comment_service.py
│   │   ├── test_sensitivity_service.py
│   │   ├── test_teams_service.py
│   │   ├── test_formatters.py
│   │   └── test_auth.py
│   └── conftest.py
├── databricks.yml
├── resources/
│   └── app_resource.yml
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                 # Lint + unit tests on PR
│   │   ├── deploy-staging.yml     # Auto-deploy to staging on merge to develop
│   │   ├── deploy-prod.yml        # Deploy to prod on merge to main (with UAT gate)
│   │   ├── deploy-rollback.yml    # Manual rollback to a tagged version
│   │   └── integration-tests.yml  # Integration tests on merge to develop
│   ├── CODEOWNERS                 # Auto-assign reviewers by path
│   ├── PULL_REQUEST_TEMPLATE.md   # PR checklist template
│   └── dependabot.yml             # Automated dependency updates
├── requirements.txt
└── pyproject.toml
```

### 5.2 Streamlit Session State Schema

```python
st.session_state.wizard = {
    "current_step": int,            # 1–4
    "selected_tables": list,        # [{catalog, schema, table, size_bytes, sensitivity_tags}]
    "requestor": {
        "name": str,
        "department": str,
        "purpose": str,             # enum value
    },
    "planning": {
        "start_date": date,
        "end_date": date,
        "duration_days": int,       # calculated
        "long_term_planning": str,
    },
    "attestations": {
        "privacy_policy": bool,
        "no_credential_sharing": bool,
        "revocation_awareness": bool,
    },
    "draft_id": str | None,
    "is_dirty": bool,
    "approval_mode": str,           # "parallel" | "sequential" — read-only, from admin config
}

# Resolved once per session on login; persists until session expiry
st.session_state.roles = list       # e.g. ["requestor"], ["reviewer"], ["requestor", "reviewer"], ["admin"]
st.session_state.current_user = {
    "email": str,
    "name": str,
    "department": str,
    "reviewer_id": str | None,      # set if user is a reviewer
    "is_admin": bool,
}
```

---

## 6. Approval Workflow

### 6.1 Reviewer Assignment

When a request is submitted, the system automatically assembles the reviewer list:

1. **Fixed reviewers** — all active records with `reviewer_type = 'fixed'` in the `reviewers` Delta table
2. **Data owner reviewers** — resolved from Unity Catalog table tags on the selected tables; matched to active records with `reviewer_type = 'data_owner'` in the `reviewers` table

The combined and deduplicated list is written to `reviewer_assignments`.

### 6.2 Approval Modes

| Mode | Behaviour |
|---|---|
| **Parallel** | All reviewers are notified simultaneously. Any reviewer can act at any time. Request is approved only when all reviewers have approved. |
| **Sequential** | Reviewers are notified one at a time in `sequence_order`. The next reviewer is only notified after the current one approves. |

**Approval mode selection:** The approval mode is configured by the **admin** in the reviewers config table (`reviewers.default_approval_mode` column). It is **not** selected by the requestor. The rationale is that governance policy — not individual requestors — determines whether reviewers act in parallel or sequence. The admin UI (Screen 8) allows overriding the default mode per-request if needed for special cases.

On the Review & Confirm page (Step 4), the resolved approval mode is displayed as a read-only field so the requestor understands how their request will be reviewed.

**Sequential order definition:** The `sequence_order` column on `reviewer_assignments` is set at submission time based on the `sequence_order` column in the `reviewers` config table. For fixed reviewers, the admin pre-configures the order. For data owner reviewers, they are appended after fixed reviewers in alphabetical order of their email unless the admin has specified a per-reviewer order. The sequence is visible to the requestor on the Request Detail page.

### 6.2.1 Partial Approval State Visibility

In **parallel mode**, the Request Detail page shows a reviewer progress panel:

```
Reviewer Status:
  ✅ Alice (Data Governance) — Approved on 2026-03-15
  ⏳ Bob (Data Owner, prod.finance.transactions) — Pending
  ✅ Carol (Compliance) — Approved on 2026-03-16
  ⏳ Dave (Security) — Pending
  Progress: 2/4 approved
```

In **sequential mode**, the panel shows the current active reviewer and the queue:

```
Sequential Review:
  ✅ Step 1: Alice (Data Governance) — Approved
  🔶 Step 2: Bob (Data Owner) — Current reviewer (notified 2026-03-15)
  ⬜ Step 3: Carol (Compliance) — Waiting
  ⬜ Step 4: Dave (Security) — Waiting
```

This visibility helps requestors understand progress and anticipate timelines.

### 6.2.2 Reviewer Reassignment / Delegation

When a fixed reviewer is unavailable (e.g., on leave), the following mechanisms apply:

1. **Admin manual reassignment:** The admin can reassign a pending reviewer assignment via the Admin — Reviewer Management screen. The admin selects a replacement reviewer from the active reviewers list and provides a reason. The `delegated_from` column on `reviewer_assignments` records the original reviewer.

2. **Deputy reviewer field:** Each reviewer in the `reviewers` config table has an optional `deputy_reviewer_id` column. If set and the primary reviewer has not acted within the escalation window (see 6.2.3), the system automatically reassigns to the deputy.

3. **Notification:** When reassignment occurs (manual or automatic), both the original and replacement reviewers are notified via Teams. The Request Detail page shows the delegation chain.

### 6.2.3 Approval SLA / Escalation

| SLA Metric | Threshold | Action |
|---|---|---|
| Reviewer action deadline | 3 business days from notification | Automated reminder via Teams Adaptive Card |
| Escalation | 5 business days with no action | Reassign to deputy (if configured) or escalate to admin |
| Admin escalation | 7 business days | Admin receives a Teams notification with request details and overdue reviewer |
| Overall request SLA | 10 business days from submission | Flagged on admin dashboard as "SLA Breached" |

Business days are calculated excluding weekends and configurable public holidays.

The SLA timers are tracked by the **Approval Escalation Job** (scheduled daily at 07:00 UTC) which:
1. Queries `reviewer_assignments` for records with `status = 'pending'` where `assigned_at` exceeds the SLA threshold
2. Sends reminder notifications via Power Automate
3. Escalates as per the table above
4. Logs all escalation events to `audit_log`

### 6.3 Approval State Machine

This is the canonical reference for all valid request states and transitions:

```
[draft]
  └─> submitted  (requestor submits)
        │
        ├─> cancelled  (requestor withdraws — terminal)
        │
        └─> under_review  (reviewers notified)
              │
              ├─> cancelled  (requestor withdraws while under review — terminal)
              │
              ├─> awaiting_response  (reviewer posts open comment)
              │     └─> under_review  (requestor resolves comment / approves exception)
              │           └─> cancelled  (requestor withdraws — terminal)
              │
              ├─> rejected  (any reviewer rejects — terminal)
              │     └─> appealing  (requestor appeals — ONE appeal per request)
              │           └─> under_review  (original reviewers re-notified)
              │                 ├─> approved  (reviewer overturns rejection)
              │                 └─> rejected  (reviewer re-affirms — terminal, no further appeal)
              │
              └─> approved  (all reviewers unanimous)
                    ├─> revoked  (manual revoke by admin — terminal)
                    └─> expired  (access_end date reached — terminal)
```

**Terminal states:** `cancelled`, `rejected` (after appeal exhausted), `revoked`, `expired`. No further transitions are possible from these states. A new request must be created.

**Status values used in code and Delta tables:**
`draft` | `submitted` | `under_review` | `awaiting_response` | `appealing` | `approved` | `rejected` | `cancelled` | `revoked` | `expired`

### 6.4 Comment Resolution Rules

A reviewer comment must reach one of two terminal resolution states before the overall approval can proceed:

- **`resolved`** — the requestor has addressed the comment satisfactorily and the reviewer marks it resolved
- **`exception_approved`** — the comment is acknowledged but waived; the reviewer approves with a documented exception note

No request can transition to `approved` while any comment has status `open`.

### 6.5 Reviewer Home Page

Reviewers land on a dedicated home page showing:

- Count of pending reviews awaiting their action
- List of requests in `pending` or `awaiting_response` state, sorted by submission date
- Per-request: requestor name, purpose, selected table count, sensitivity level, days pending
- Quick-link to the Request Detail / Review page

---

## 7. Microsoft Teams Integration

### 7.1 Scope

| Capability | Implementation |
|---|---|
| Reviewer notification on submission | Power Automate flow triggered by Databricks Job; sends Adaptive Card to each reviewer via Teams |
| Inline approve / reject / comment | Adaptive Card action buttons; response handled by Power Automate, which writes results to Delta table |
| Sequential reviewer handoff | Power Automate step triggers next reviewer card only after prior approval recorded |
| Requestor notification on status change | Power Automate sends Teams message to requestor on: under_review → awaiting_response, rejected, approved |
| Comment notification | Reviewer notified when requestor resolves or exception-approves a comment |

### 7.2 Authentication Between App and Power Automate

The Streamlit app and Power Automate authenticate via an **Azure AD App Registration** with a shared service principal:

1. **Azure AD App Registration:** An app registration is created in Azure AD with the following configuration:
   - API permissions: `User.Read` (for Teams user lookup), `TeamsAppInstallation.ReadWriteSelfForChat` (for Adaptive Card delivery)
   - A client secret is generated and stored in Databricks Secrets (see Section 8.4)
   - The app registration's `client_id` and `tenant_id` are referenced in both the Streamlit app config and the Power Automate flow

2. **Streamlit → Power Automate:** The Streamlit app triggers Power Automate via an **HTTP webhook trigger**. The request includes:
   - `Authorization: Bearer <signed JWT>` using the app registration's client credentials flow
   - Request body: JSON payload with `request_id`, `action` (e.g., `notify_reviewers`, `notify_requestor`), and relevant data

3. **Power Automate → Databricks (callback):** Power Automate writes results back by calling a **Databricks Job** (not the Streamlit app directly — see 7.3). Authentication uses a Databricks personal access token (PAT) stored in Power Automate's Azure Key Vault connection.

### 7.3 Callback Architecture (Critical Design Decision)

**Problem:** Streamlit apps cannot receive inbound HTTP callbacks — they are UI rendering frameworks, not REST API servers. Power Automate needs a way to write reviewer decisions back to the app's Delta tables.

**Solution: Databricks Job as Webhook Receiver + Delta Table as Message Queue**

```
Streamlit App                    Databricks Job (webhook receiver)
     │                                    │
     ├─ triggers ──► Power Automate        │
     │               ├─ sends Adaptive Card to reviewer
     │               ├─ reviewer clicks [Approve]
     │               ├─ Power Automate ──POST──► Databricks Job
     │               │                         (HTTP endpoint via
     │               │                          Databricks Jobs REST API
     │               │                          or Databricks SQL webhook)
     │               │                         │
     │               │                         ├─ writes decision to
     │               │                         │   review_comments /
     │               │                         │   reviewer_assignments
     │               │                         │   via SQL Execution API
     │               │                         │
     │               └─ (on next page load) ──┤
     │                                        │
     └─ polls Delta tables ◄──────────────────┘
         (reads updated state on
          each page interaction)
```

**Why this approach:**
- No custom API server to host and secure
- Leverages Databricks' existing infrastructure and authentication
- Delta tables serve as a durable message queue — the Streamlit app reads updated state on each interaction (page load, button click)
- The Databricks Job is triggered via the **Databricks Jobs Run Now API** called by Power Automate, or via a **SQL webhook** (if available in the workspace)

**Alternative considered and rejected:**
- FastAPI/Flask sidecar: Adds deployment complexity, requires separate hosting, and introduces a new authentication surface
- Polling-only (no webhook): Creates unacceptable latency for approval actions (reviewers expect near-instant feedback)

### 7.4 Power Automate Flow Design

```
Flow: "Data Curator — Reviewer Notification"
Trigger: HTTP Request (from Streamlit app via Databricks Job)
  │
  ├─ Parse request body: { request_id, action, reviewer_ids[], ... }
  ├─ Authenticate: validate JWT from Azure AD app registration
  │
  ├─ Switch: action type
  │   ├─ "notify_reviewers_parallel"
  │   │     └─ For each reviewer_id:
  │   │           ├─ Look up teams_user_id from reviewers Delta table
  │   │           ├─ Send Adaptive Card (approve/reject/comment)
  │   │           └─ Track delivery status
  │   │
  │   ├─ "notify_reviewer_sequential"
  │   │     └─ Look up next reviewer by sequence_order
  │   │           └─ Send Adaptive Card to single reviewer
  │   │
  │   ├─ "notify_requestor"
  │   │     └─ Send simple Teams message (status change notification)
  │   │
  │   └─ "reviewer_action_response"
  │         ├─ Parse action: approve | reject | comment
  │         ├─ Write result to Delta table via Databricks SQL Execution API
  │         ├─ If all approved → trigger "notify_requestor" (approved)
  │         ├─ If rejected → trigger "notify_requestor" (rejected)
  │         └─ If sequential + approved → trigger "notify_reviewer_sequential" (next)
  │
  └─ Error handling: on failure, write to dead_letter_queue Delta table
```

### 7.5 Adaptive Card Payload Specification

**Card sent to reviewer on assignment:**

```json
{
  "type": "AdaptiveCard",
  "body": [
    {
      "type": "TextBlock",
      "text": "Data Access Request — Approval Required",
      "weight": "Bolder",
      "size": "Medium"
    },
    {
      "type": "FactSet",
      "facts": [
        { "title": "Requestor", "value": "${requestor_name}" },
        { "title": "Department", "value": "${requestor_dept}" },
        { "title": "Purpose", "value": "${purpose}" },
        { "title": "Tables", "value": "${table_count} tables (${catalogs})" },
        { "title": "Sensitivity", "value": "${max_sensitivity}" },
        { "title": "Access Period", "value": "${access_start} to ${access_end}" },
        { "title": "Duration", "value": "${duration_days} days" }
      ]
    },
    {
      "type": "TextBlock",
      "text": "**Long-term Planning:** ${long_term_planning}",
      "wrap": true
    }
  ],
  "actions": [
    {
      "type": "Action.Execute",
      "title": "Approve",
      "verb": "approve",
      "data": { "request_id": "${request_id}", "reviewer_id": "${reviewer_id}" }
    },
    {
      "type": "Action.Execute",
      "title": "Request Clarification",
      "verb": "clarify",
      "data": { "request_id": "${request_id}", "reviewer_id": "${reviewer_id}" }
    },
    {
      "type": "Action.Execute",
      "title": "Reject",
      "verb": "reject",
      "data": { "request_id": "${request_id}", "reviewer_id": "${reviewer_id}" }
    }
  ],
  "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
  "version": "1.5"
}
```

**Clarification and Reject actions** include an `input` element for multi-line text:

```json
{
  "type": "Action.Execute",
  "title": "Reject",
  "verb": "reject",
  "data": { "request_id": "${request_id}", "reviewer_id": "${reviewer_id}" },
  "associatedInputs": "auto"
}
```

### 7.6 Teams Channel vs Direct Message

**Default: Direct Messages (DMs).** Adaptive Cards are sent as 1:1 chats to each reviewer's Teams user identity. Rationale:

- Reduces noise in shared channels
- Reviewer actions are private until recorded in the system
- Supports sequential mode where only one reviewer is active at a time
- Easier to track which notifications are pending per-reviewer

**Optional: Shared governance channel.** An admin-configurable flag (`TEAMS_NOTIFICATION_MODE` = `dm` | `channel` | `both`) allows copying a summary notification to a shared Teams channel (e.g., "Data Governance — Approvals") for visibility. The actionable Adaptive Card is always sent via DM regardless of this setting.

### 7.7 Power Automate Licensing

Power Automate **premium licenses** are required for:
- HTTP trigger (receiving webhooks from Databricks)
- HTTP action (calling Databricks REST APIs for callback)
- Custom connectors (if used instead of raw HTTP)

**Required licenses:**
- At minimum 1 Power Automate per-flow plan (covers the single orchestrating flow)
- Alternatively, per-user licenses for each admin who manages the flows
- Budget estimate: ~$15/user/month or ~$100/flow/month (verify with Microsoft licensing team)

This dependency must be confirmed with the organisation's Microsoft licensing administrator before Phase 4.

### 7.8 Failure Handling for Power Automate

| Failure Scenario | Detection | Recovery |
|---|---|---|
| Power Automate flow fails to trigger | Streamlit app checks HTTP response; logs error to `audit_log` | Retry with exponential backoff (3 attempts); if all fail, fall back to in-app review only |
| Adaptive Card delivery fails | Power Automate flow run status = Failed; error logged in PA run history | Dead-letter entry written to `notification_dlq` Delta table; admin alerted via email |
| Callback POST fails | Power Automate catches HTTP error from Databricks Job | Retry 3 times; on persistent failure, write decision to `notification_dlq` for manual reconciliation |
| Databricks Job for webhook fails | Job run status = Failed in Databricks workspace | Admin alert; manual re-trigger via Databricks UI; reviewer decision already captured in PA flow run |
| Partial fan-out (some cards sent, some fail) | Per-reviewer delivery tracked in PA flow | Failed deliveries retried; successful deliveries not re-sent |

**Dead-letter queue:** A `notification_dlq` Delta table stores failed notifications with `event_id`, `request_id`, `reviewer_id`, `payload`, `error_message`, `retry_count`, and `created_at`. An admin can inspect and manually re-trigger from the Admin dashboard.

### 7.9 Comment Delivery via Teams

When a reviewer clicks **[Request Clarification]** or **[Reject]** on the Adaptive Card:

1. A **multi-line text input** is shown inline in the Adaptive Card (using `Input.Text` with `isMultiline: true`)
2. The reviewer types their comment and submits
3. Power Automate captures the full text and POSTs it to the Databricks webhook Job
4. The Job writes the comment to `review_comments` with `resolution_status = 'open'`

For longer comments or complex responses, the Adaptive Card includes a **deep-link** to the in-app review page: `[Open in Data Curator]` button that navigates the reviewer to the Request Detail page in the Streamlit app where they can use the full comment thread interface.

### 7.10 Teams User Identity Mapping

The `reviewers.teams_user_id` column stores the Microsoft Teams user identifier (Azure AD Object ID) for each reviewer. This value is:

1. **Populated during reviewer onboarding:** When an admin adds a reviewer via the Admin UI, the app queries the Azure AD Graph API (`GET /users/{email}`) to resolve the Teams/Azure AD user ID from the reviewer's email address
2. **Kept in sync by:** A nightly **Teams Identity Sync Job** that:
   - Iterates all active reviewers
   - Queries Azure AD Graph API for each `reviewer_email`
   - Updates `teams_user_id` if changed (handles renamed accounts, tenant migrations)
   - Flags reviewers with no matching Azure AD identity for admin review
3. **Stored format:** Azure AD Object GUID (e.g., `a1b2c3d4-e5f6-7890-abcd-ef1234567890`)

---

## 8. Development Plan

### Phase 1 — Foundation (Weeks 1–2)

- Set up Databricks Asset Bundle project structure
- Configure `databricks.yml` with workspace, SQL warehouse, and Streamlit app resources
- Implement base REST API client with Databricks authentication (OAuth / PAT)
- Build Unity Catalog API wrapper (list catalogs, schemas, tables, tags across all 4 environments)
- Implement SQL Execution client for `information_schema.tables` size queries
- Build SCIM client for user/group lookups
- Initialise Delta tables (schema creation pipeline)
- Set up GitHub repo with branch protection, PR templates, and ruff linting config

### Phase 2 — Core Wizard (Weeks 3–5)

- Multi-page Streamlit routing (7 pages)
- Shared components: sidebar, top nav, progress stepper, sensitivity badge
- Step 1 — Data Selection: table browsing grid across 4 catalogs, search, toggle selection, auto-populated size and sensitivity tags
- Step 2 — Requestor Information: name, department dropdown, purpose single-select
- Step 3 — Access & Planning: date pickers, duration calculation, long-term planning textarea
- Step 4 — Review & Confirm: summary cards, inline edit navigation, attestation checklist, submit
- Draft save/load via Delta table writes

### Phase 3 — Dashboard, Reviewer Home & Approval Workflow (Weeks 6–8)

- Requestor dashboard: status metric cards, recent requests list, department filter
- Reviewer home page: pending queue with priority indicators
- Request Detail / Review page (shared; role-aware rendering)
- Reviewer assignment logic (fixed + data owner resolution from UC tags)
- Approval state machine: parallel/sequential modes, unanimous requirement, comment resolution
- Comment thread component with resolve / exception-approve actions

### Phase 4 — Teams Integration & Permissions (Weeks 9–10)

- Power Automate flow configuration (parallel/sequential fan-out, reviewer handoff)
- Teams Adaptive Card templates (approve / reject / comment actions)
- Databricks Job webhook receiver for Power Automate callbacks (see Section 7.3)
- Azure AD App Registration setup for Power Automate authentication
- Teams Identity Sync Job for reviewer `teams_user_id` resolution (nightly)
- Approval Escalation Job: SLA tracking, reminder notifications, deputy reassignment (weekday schedule)
- Automated READ permission grant via Permissions API on final approval
- Access expiry monitor job (scheduled daily; revokes expired grants)
- Notification dead-letter queue (`notification_dlq`) and admin retry UI

### Phase 5 — Admin UI & Renewal Flows (Weeks 11–12)

- Admin — Reviewer Management screen (add/edit/deactivate reviewers, configure sequence order)
- Admin — All Requests View (filter by status, environment, department, date range)
- Admin — Audit & Compliance Reports (audit log viewer, export)
- Access renewal / extension flow
- Request cancellation flow
- Rejection appeal flow

### Phase 6 — Polish, Hardening & UAT (Weeks 13–14)

- Error handling and user-friendly error states
- Loading states and skeleton screens for API calls
- Performance optimisation (`@st.cache_data` for UC lookups)
- Responsive design adjustments
- End-to-end testing and UAT with governance team
- Security review and penetration test

### 8.1 Delta Table Migration Strategy

The app uses a **versioned migration script** approach for Delta table schema changes:

```
data_curator/
├── migrations/
│   ├── 001_initial_schema.sql        # CREATE TABLE IF NOT EXISTS for all tables
│   ├── 002_add_parent_request_id.sql  # ALTER TABLE access_requests ADD COLUMN ...
│   ├── 003_add_delegation.sql         # ALTER TABLE reviewer_assignments ADD COLUMN ...
│   └── migration_log.sql              # Tracks applied migrations
```

**Migration runner:** A Databricks Job (`schema_migration_job`) runs on each deploy (before the app starts) and:
1. Queries `migration_log` to determine which migrations have been applied
2. Executes pending migrations in order
3. Records each applied migration with `migration_id`, `applied_at`, and `applied_by`
4. Fails the deploy if any migration errors (no partial applies)

**Schema conventions:**
- All migrations are **forward-only** — no rollback scripts (see Section 10.5 for rollback strategy)
- Column additions use `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`
- Breaking changes (column renames, type changes) require a multi-step migration: add new column → backfill → drop old column → rename

### 8.2 Local Development Setup

Developers can run the app locally against a dev Databricks workspace:

```bash
# 1. Clone and install
git clone <repo>
cd data_curator
pip install -r requirements.txt

# 2. Configure environment
cp .env.example .env
# Edit .env with:
#   DATABRICKS_HOST=https://<dev-workspace>.cloud.databricks.com
#   DATABRICKS_TOKEN=<dev-pat-token>
#   APP_CATALOG=dev
#   APP_SCHEMA=data_curator_app
#   SQL_WAREHOUSE_ID=<dev-warehouse-id>

# 3. Run migrations against dev workspace
python -m migrations.runner

# 4. Start Streamlit
streamlit run src/app.py
```

**Mock mode for offline development:** The app supports a `MOCK_MODE=true` environment variable that:
- Replaces all Databricks API calls with in-memory mock data
- Skips Power Automate webhook calls (logs to console instead)
- Pre-populates sample requests, reviewers, and comments
- Enables developers to work on UI components without a live Databricks workspace

### 8.3 Secret Management

Secrets are managed at two levels:

| Secret | Storage Location | Access Method |
|---|---|---|
| Databricks PAT (CI/CD) | GitHub Secrets | `${{ secrets.DATABRICKS_TOKEN }}` |
| Databricks workspace URL (CI/CD) | GitHub Secrets | `${{ secrets.DATABRICKS_HOST }}` |
| Azure AD client ID | Databricks Secrets (scope: `data-curator`) | `dbutils.secrets.get("data-curator", "azure-ad-client-id")` |
| Azure AD client secret | Databricks Secrets (scope: `data-curator`) | `dbutils.secrets.get("data-curator", "azure-ad-client-secret")` |
| Azure AD tenant ID | Databricks Secrets (scope: `data-curator`) | `dbutils.secrets.get("data-curator", "azure-ad-tenant-id")` |
| Power Automate webhook URL | Databricks Secrets (scope: `data-curator`) | `dbutils.secrets.get("data-curator", "pa-webhook-url")` |
| Service principal PAT | Databricks Secrets (scope: `data-curator`) | `dbutils.secrets.get("data-curator", "sp-pat-token")` |

**Databricks Secret Scope setup:**
```bash
databricks secrets create-scope data-curator
databricks secrets put-secret data-curator azure-ad-client-id --string-value <value>
databricks secrets put-secret data-curator azure-ad-client-secret --string-value <value>
# ... etc
```

**Access control:** The secret scope ACL grants read access only to:
- The Streamlit app's service principal
- The Databricks Jobs that need secrets (webhook receiver, expiry monitor)
- Admin users for troubleshooting

### 8.4 Service Principal Setup

A dedicated service principal (`data-curator-sp`) executes permission grants on behalf of the app:

1. **Provisioning:** Created via Databricks Account Console or Azure AD as a service principal, then added to the Databricks workspace
2. **Unity Catalog grants:**
   ```
   -- On each catalog the app manages (dev, sit, uat, prod):
   GRANT USE CATALOG ON CATALOG <catalog> TO `data-curator-sp`;
   GRANT USE SCHEMA ON CATALOG <catalog> TO `data-curator-sp`;
   GRANT SELECT ON TABLE <catalog>.<schema>.<table> TO `data-curator-sp`;
   
   -- For granting permissions to requestors:
   GRANT MANAGE ON TABLE <catalog>.<schema>.<table> TO `data-curator-sp`;
   ```
3. **Authentication:** Uses a PAT stored in Databricks Secrets (see 8.3); rotated every 90 days via a documented manual process
4. **Least privilege:** The SP can only grant READ permissions on tables within the managed catalogs; it cannot create/drop tables or modify schema
5. **Audit:** All permission grants executed by the SP are logged to `audit_log` with `actor = 'data-curator-sp'`

---

## 9. Testing Strategy

### 9.1 Unit Test Coverage

| Test File | Coverage |
|---|---|
| `test_unity_catalog_api.py` | Mocked UC API responses: list catalogs/schemas/tables, tag retrieval, error handling, pagination |
| `test_permissions_api.py` | Mocked grant/revoke/list READ permissions, error on invalid securable |
| `test_request_service.py` | CRUD operations, all status transitions (incl. `appealing`, `cancelled`), validation rules, `appeal_count` enforcement, `parent_request_id` linking for renewals |
| `test_draft_service.py` | Save/load/update/delete draft; stale draft cleanup |
| `test_reviewer_service.py` | Fixed reviewer list retrieval, data owner resolution from tags, deduplication, sequential ordering |
| `test_approval_service.py` | Parallel approval logic, sequential handoff, unanimous requirement, rejection terminal state, per-environment partial approval |
| `test_comment_service.py` | Comment creation, resolution, exception-approve, blocking approval while open comments exist |
| `test_sensitivity_service.py` | Tag-to-sensitivity-level mapping, missing tag handling, `pii=true` override logic, unexpected tag values |
| `test_escalation_service.py` | SLA threshold calculation (business days only), reminder trigger at 3 days, deputy reassignment at 5 days, admin escalation at 7 days, SLA breach flag at 10 days |
| `test_renewal_service.py` | Extension pre-fill from approved request, `parent_request_id` linking, max duration cap validation on renewal end date, renewal creates new request not modifying existing |
| `test_appeal_service.py` | Appeal creates `appealing` status, increments `appeal_count`, blocks second appeal when `appeal_count >= 1`, re-notifies original reviewers only |
| `test_teams_service.py` | Adaptive Card payload construction, webhook POST, Power Automate trigger, DLQ write on failure |
| `test_formatters.py` | Date formatting, byte-size formatting, status label mapping, duration calculation, business day calculation |
| `test_auth.py` | Token refresh, credential validation, expired token handling, role detection logic (admin group, reviewer table, fallback requestor) |

### 9.2 Testing Patterns

```python
# conftest.py — shared fixtures
@pytest.fixture
def mock_uc_client():
    with patch("src.api.unity_catalog.DatabricksClient") as mock:
        mock.return_value.get.return_value = {
            "tables": [
                {
                    "catalog_name": "prod",
                    "schema_name": "finance",
                    "name": "transactions",
                    "table_type": "MANAGED",
                    "properties": {"sensitivity": "high", "data_owner": "alice@example.com"}
                }
            ]
        }
        yield mock

# Example: reviewer assignment test
def test_data_owner_resolved_from_table_tags(mock_uc_client, mock_reviewer_table):
    service = ReviewerService(uc_client=mock_uc_client, db_client=mock_reviewer_table)
    assignments = service.resolve_reviewers(request_id="req-001", tables=["prod.finance.transactions"])
    data_owners = [a for a in assignments if a["reviewer_type"] == "data_owner"]
    assert any(a["reviewer_email"] == "alice@example.com" for a in data_owners)

# Example: approval blocked by open comment
def test_approval_blocked_when_open_comments_exist(mock_approval_service):
    with pytest.raises(ApprovalBlockedError, match="open comments"):
        mock_approval_service.approve(request_id="req-001", reviewer_id="rev-002")
```

### 9.3 Test Requirements

- Minimum **80% line coverage** across `src/api/` and `src/services/`
- All API client tests **must mock HTTP responses** — no live Databricks calls in unit tests
- Business logic tests must cover all status transition paths
- Approval service tests must cover parallel and sequential modes independently
- Comment service tests must cover all three resolution states

### 9.4 CI Test Execution

```bash
pytest tests/ --cov=src --cov-report=term-missing --cov-fail-under=80
```

Runs on every PR to `main` and `develop`. Blocks merge if coverage drops below threshold or any test fails.

### 9.5 Integration Tests

Integration tests run against a **dev Databricks workspace** (not mocked) and validate end-to-end API interactions:

| Test File | Coverage |
|---|---|
| `test_integration_uc.py` | Live Unity Catalog API calls: list real catalogs/schemas/tables in dev workspace, retrieve actual tags, verify pagination |
| `test_integration_permissions.py` | Grant and revoke READ on a test table in dev workspace; verify via list permissions |
| `test_integration_sql_execution.py` | Execute `information_schema.tables` query against dev warehouse; verify size metadata returned |
| `test_integration_scim.py` | Look up current user and groups via SCIM API |

**Execution:**
```bash
pytest tests/integration/ --databricks-host=$DATABRICKS_HOST --databricks-token=$DATABRICKS_TOKEN
```

Integration tests run:
- On merge to `develop` (pre-staging gate)
- Nightly against dev workspace to detect upstream API changes
- **Not** on every PR (to avoid rate limiting and flaky tests blocking development)

### 9.6 End-to-End / UI Tests

E2E tests use **Playwright** with Python to test Streamlit UI flows:

| Test | Coverage |
|---|---|
| `test_e2e_wizard_happy_path.py` | Full 4-step wizard: data selection → requestor info → planning → review & submit |
| `test_e2e_draft_save_load.py` | Save draft at each step, reload, verify state preserved |
| `test_e2e_reviewer_approve.py` | Reviewer opens request, approves, verify status changes |
| `test_e2e_reviewer_reject.py` | Reviewer rejects, requestor sees rejection, appeals |
| `test_e2e_admin_reviewer_mgmt.py` | Admin adds/edits/deactivates a reviewer |

**Framework:** `pytest-playwright` with headless Chromium
**Execution:** `pytest tests/e2e/ --browser chromium --headed=false`
**Runs:** On merge to `develop` only (not on every PR — too slow for fast feedback)

### 9.7 Teams / Power Automate Testing Approach

Testing the Teams integration is challenging because it involves external Microsoft services. The approach:

1. **Unit tests (mocked):** All Power Automate interactions are tested with mocked HTTP calls. The `teams_service.py` has a clean interface that can be fully mocked.

2. **Integration smoke test (dev environment):** A single Playwright-based test that:
   - Creates a test request via the Streamlit app
   - Verifies the Power Automate flow was triggered (check flow run history via PA API)
   - Verifies the Adaptive Card was delivered to a test Teams account
   - Simulates an approve action via the PA callback
   - Verifies the request status updated in the Delta table

3. **Manual QA checklist (pre-release):**
   - [ ] Adaptive Card renders correctly in Teams desktop and web
   - [ ] Approve button triggers callback and updates app state
   - [ ] Reject button with comment captures full text
   - [ ] Clarification request creates open comment in app
   - [ ] Sequential mode notifies next reviewer after approval
   - [ ] Requestor receives status change notifications

4. **Not tested in CI:** Full Adaptive Card rendering, Teams channel delivery, multi-user fan-out — these require live Microsoft infrastructure and are validated during UAT.

---

## 10. Deployment Strategy

### 10.1 Databricks Asset Bundle Configuration

```yaml
# databricks.yml
bundle:
  name: data-curator

variables:
  workspace_host:
    description: "Databricks workspace URL"
  sql_warehouse_id:
    description: "SQL Warehouse for app data queries"
  app_catalog:
    default: "main"
    description: "Unity Catalog for app Delta tables"
  app_schema:
    default: "data_curator_app"
    description: "Schema for app Delta tables"

resources:
  jobs:
    approval_escalation_job:
      name: "Approval Escalation Monitor"
      schedule:
        quartz_cron_expression: "0 0 7 ? * MON-FRI"   # 07:00 UTC, weekdays only
        timezone_id: "UTC"
      tasks:
        - task_key: "check_and_escalate"
          python_file: "src/jobs/approval_escalation.py"
          warehouse_id: ${var.sql_warehouse_id}

    teams_identity_sync_job:
      name: "Teams Identity Sync"
      schedule:
        quartz_cron_expression: "0 0 1 * * ?"   # Daily at 01:00 UTC
        timezone_id: "UTC"
      tasks:
        - task_key: "sync_reviewer_teams_ids"
          python_file: "src/jobs/teams_identity_sync.py"

    quarterly_audit_job:
      name: "Quarterly Audit Report"
      schedule:
        quartz_cron_expression: "0 0 8 1 1,4,7,10 ?"  # 08:00 on Jan/Apr/Jul/Oct 1
        timezone_id: "UTC"
      tasks:
        - task_key: "generate_quarterly_report"
          python_file: "src/jobs/quarterly_audit.py"
          warehouse_id: ${var.sql_warehouse_id}

    expiry_monitor_job:
      name: "Access Expiry Monitor"
      schedule:
        quartz_cron_expression: "0 0 6 * * ?"   # Daily at 06:00 UTC
        timezone_id: "UTC"
      tasks:
        - task_key: "check_and_revoke_expired"
          python_file: "src/jobs/expiry_monitor.py"
          warehouse_id: ${var.sql_warehouse_id}

    draft_cleanup_job:
      name: "Stale Draft Cleanup"
      schedule:
        quartz_cron_expression: "0 0 2 * * ?"   # Daily at 02:00 UTC
        timezone_id: "UTC"
      tasks:
        - task_key: "delete_stale_drafts"
          python_file: "src/jobs/draft_cleanup.py"
          warehouse_id: ${var.sql_warehouse_id}

  apps:
    data_curator_app:
      name: "Data Curator"
      source_code_path: ./src
      config:
        catalog: ${var.app_catalog}
        schema: ${var.app_schema}
        sql_warehouse_id: ${var.sql_warehouse_id}
```

### 10.2 Version Control & CI/CD

#### 10.2.1 Branch Strategy

The project uses a **GitHub Flow** variant with `main` as the production branch and `develop` as the staging integration branch:

```
main (production)     ─────────●─────────●─────────●──────►
                               ↑         ↑         ↑
                          (merge)    (merge)   (merge)
                               ↑         ↑         ↑
develop (staging)     ────●────●────●────●────●────●──────►
                         ↑    ↑    ↑    ↑
                    (merge)(merge)(merge)(merge)
                         ↑    ↑    ↑    ↑
feature/feat-name      ─●────●    ↑    ↑
                                  ↑    ↑
feature/feat-name-2    ──●───●────●    ↑
                                      ↑
bugfix/fix-name       ────●───●────────●
```

| Branch | Purpose | Protected | Auto-deploys to |
|---|---|---|---|
| `main` | Production-ready code | Yes | Production (with UAT gate) |
| `develop` | Integration branch for staging | Yes | Staging |
| `feature/*` | New features | No | — |
| `bugfix/*` | Bug fixes | No | — |
| `hotfix/*` | Urgent production fixes (branched from `main`) | No | — |

**Feature branch naming convention:** `feature/<short-description>` (e.g., `feature/data-selection-wizard`, `feature/teams-adaptive-cards`)

**Flow:**
1. Developer creates `feature/*` from `develop`
2. Developer opens PR → `develop` when feature is ready
3. PR passes CI checks, gets reviewed and merged
4. `develop` auto-deploys to staging for UAT
5. When staging is validated, `develop` is merged to `main` via PR
6. `main` triggers prod deploy (with manual UAT gate)

**Hotfix flow:**
1. Developer creates `hotfix/*` from `main`
2. Opens PR → `main` for urgent fix
3. After merge, `main` is also merged back to `develop` to keep branches in sync

#### 10.2.2 Branch Protection Rules

| Rule | `main` | `develop` |
|---|---|---|
| Require pull request before merging | Yes | Yes |
| Required reviewers | 1 (minimum) | 1 (minimum) |
| Dismiss stale reviews on new push | Yes | Yes |
| Required status checks | `ci-lint`, `ci-test` | `ci-lint`, `ci-test` |
| Require branches to be up to date | Yes | Yes |
| Allow force push | No | No |
| Allow deletions | No | No |
| Require linear history (squash merge only) | Yes | Yes |
| Admins are subject to rules | Yes | No |

**Merge strategy:** Squash merge only. This keeps `main` and `develop` history linear and makes rollbacks via `git revert` straightforward.

#### 10.2.3 PR Template

```markdown
## Description
<!-- What does this PR do? -->

## Type of Change
- [ ] Feature
- [ ] Bug fix
- [ ] Refactor
- [ ] Documentation
- [ ] CI/CD

## Checklist
- [ ] Code follows project style (ruff passes)
- [ ] Unit tests added/updated
- [ ] Integration tests considered
- [ ] No secrets or credentials in code
- [ ] PR title follows conventional commits format
```

#### 10.2.4 CODEOWNERS

```
# Default reviewers for all changes
*                       @data-governance-team

# API layer — requires backend reviewer
src/api/                @data-governance-team @backend-lead

# Services — requires architecture review
src/services/           @data-governance-team @tech-lead

# Deployment config — requires DevOps review
databricks.yml          @data-governance-team @devops-lead
.github/                @data-governance-team @devops-lead

# Tests — ensures test coverage maintained
tests/                  @data-governance-team
```

#### 10.2.5 CI Workflows

**`ci.yml` — Lint + Unit Tests (on every PR)**

```yaml
name: CI
on:
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -r requirements.txt
      - run: ruff check src/ tests/
      - run: ruff format --check src/ tests/

  test:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -r requirements.txt
      - run: pytest tests/unit/ --cov=src --cov-report=term-missing --cov-fail-under=80
```

**`deploy-staging.yml` — Auto-deploy to Staging (on merge to develop)**

```yaml
name: Deploy Staging
on:
  push:
    branches: [develop]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@v0.228
      - name: Run migrations
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_STAGING }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_STAGING }}
        run: databricks bundle deploy --target staging
      - name: Run schema migrations
        run: python -m migrations.runner --target staging
```

**`integration-tests.yml` — Integration Tests (on merge to develop)**

```yaml
name: Integration Tests
on:
  push:
    branches: [develop]

jobs:
  integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -r requirements.txt
      - run: pytest tests/integration/ --timeout=60
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_STAGING }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_STAGING }}
```

**`deploy-prod.yml` — Production Deploy (on merge to main, with UAT gate)**

```yaml
name: Deploy Production
on:
  push:
    branches: [main]

jobs:
  deploy-staging-final:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@v0.228
      - name: Deploy to Staging (final verification)
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_STAGING }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_STAGING }}
        run: databricks bundle deploy --target staging

  uat-gate:
    needs: deploy-staging-final
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval in GitHub
    steps:
      - run: echo "UAT approved — proceeding to prod deploy"

  deploy-prod:
    needs: uat-gate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@v0.228
      - name: Run migrations
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
        run: python -m migrations.runner --target prod
      - name: Deploy to Production
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
        run: databricks bundle deploy --target prod
      - name: Tag release
        run: |
          git tag "release-$(date +%Y%m%d-%H%M%S)"
          git push --tags
```

**`deploy-rollback.yml` — Manual Rollback**

```yaml
name: Rollback
on:
  workflow_dispatch:
    inputs:
      version_tag:
        description: 'Git tag to roll back to (e.g., release-20260402-120000)'
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.inputs.version_tag }}
      - uses: databricks/setup-cli@v0.228
      - name: Rollback to tagged version
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
        run: databricks bundle deploy --target prod
```

#### 10.2.6 Automated Testing Matrix Summary

| Test Type | Trigger | Blocks Merge? | Environment |
|---|---|---|---|
| Lint (ruff) | Every PR to `main`/`develop` | Yes | GitHub runner |
| Unit tests | Every PR to `main`/`develop` | Yes (if < 80% coverage or failure) | GitHub runner (mocked) |
| Integration tests | Merge to `develop` | No (post-merge validation) | Staging Databricks workspace |
| E2E (Playwright) | Merge to `develop` | No (post-merge validation) | Staging Databricks workspace |
| Manual UAT | Between staging and prod | Yes (manual approval gate) | Staging |

#### 10.2.7 Git Tagging / Versioning

- Every prod deploy creates a timestamped tag: `release-YYYYMMDD-HHMMSS`
- Tags are pushed automatically by `deploy-prod.yml`
- Rollback references these tags
- The last 10 tags are retained; older ones can be pruned manually
- No semantic versioning (semver) — timestamps are sufficient for a continuously deployed internal app

### 10.3 Deployment Environments

| Environment | Trigger | Purpose |
|---|---|---|
| `dev` | Manual / feature branch | Developer testing, feature validation |
| `staging` | Merge to `develop` | UAT, governance team review |
| `prod` | Merge to `main` | Production access for all users |

### 10.4 App Table Initialisation

On first deploy, a pipeline creates all required Delta tables using `CREATE TABLE IF NOT EXISTS`. Subsequent deploys apply migration scripts for schema changes. Tables are created in `{app_catalog}.{app_schema}` as configured in `databricks.yml`.

### 10.5 Rollback Strategy

**Streamlit app code rollback:**
- DAB maintains deployment history. Rollback is performed via `databricks bundle deploy --target prod` from the previous Git tag
- A `deploy_rollback.yml` GitHub Actions workflow accepts a version tag and deploys that version to prod
- Rollback is available for the last 5 deployed versions (tagged releases)

**Delta table schema rollback:**
- Schema migrations are **forward-only** — there are no automatic rollback scripts
- If a migration causes issues, the remediation is:
  1. Fix the migration in a new commit
  2. Deploy the fix forward
  3. If data corruption occurred, restore from Delta table time travel (`RESTORE TABLE ... TO VERSION AS OF ...`)
- Delta's time travel (default 30-day retention) provides a safety net for data-level rollback

**Power Automate flow rollback:**
- Power Automate flows are versioned in the Microsoft environment
- Rollback: deactivate the current flow version and reactivate the previous version
- Flow versions should be documented in a change log alongside Git releases

### 10.6 Staging-to-Prod Promotion Gate

The deploy pipeline includes a **manual UAT sign-off gate** between staging and prod. The `deploy-prod.yml` workflow (defined in Section 10.2.5) deploys to staging first, then pauses for manual approval before deploying to production.

The `production` environment in GitHub requires **at least one approval** from designated reviewers (governance team lead or tech lead) before the prod deploy proceeds. This is configured in GitHub repo Settings → Environments → `production` → Required reviewers.

---

## 11. Security & Compliance

### 11.1 Authentication & Authorisation

- App inherits Databricks workspace authentication (SSO / SCIM)
- Current user resolved via `GET /api/2.0/preview/scim/v2/Me`
- Department auto-populated from user's SCIM group membership
- Row-level filtering: requestors see only their own requests; reviewers see all pending items assigned to them; admins see everything

**Role detection:** A user's role within the app is determined at login using the following priority order:

1. **Admin** — user's SCIM group membership includes the configured admin group (e.g., `data-curator-admins`; set via `ADMIN_GROUP_NAME` in `config.py`)
2. **Reviewer** — user's email matches an active record in `reviewers` table with `is_active = TRUE`
3. **Requestor** — any authenticated Databricks workspace user who does not match the above

A user can hold multiple roles simultaneously — for example, a Data Governance team member may be both a fixed reviewer and a requestor. The app renders role-appropriate UI based on the active context (e.g., a reviewer who also has pending requests will see both the Reviewer Home page and the Requestor Dashboard in the sidebar navigation).

Role resolution is performed once per session and cached in `st.session_state.roles` (a list, e.g., `["requestor", "reviewer"]`). The roles list is re-evaluated on session expiry.

### 11.2 Permissions Model

| Role | Capabilities |
|---|---|
| Requestor | Create, edit drafts, submit, view own requests |
| Reviewer | View assigned requests, approve, reject, comment |
| Admin | Manage reviewer Delta table, view all requests, access audit log, run compliance reports |

READ permission grants on Unity Catalog tables are executed via a service principal with least-privilege scope (table-specific, time-bound to the approved access window).

### 11.3 Audit Trail

Every request action (create, submit, approve, reject, comment, resolve, revoke) is appended to `audit_log`. This table is append-only — `MERGE` is prohibited. Fields include `event_id`, `request_id`, `actor`, `action`, `timestamp`, and `details`.

### 11.4 Data Sensitivity

Sensitivity classification is **auto-detected from Unity Catalog table tags** and surfaced to the user as a read-only badge (e.g., `PUBLIC`, `INTERNAL`, `CONFIDENTIAL`, `RESTRICTED`). Users do not manually classify sensitivity. High-sensitivity tables are highlighted in the Data Selection step and the Review page.

### 11.5 Compliance Attestations

Users must explicitly acknowledge all three attestation checkboxes on the Review & Confirm page before the submit button is enabled. Attestation status is recorded in `audit_log` on submission.

### 11.6 Data Residency / Workspace Boundary

All four catalogs (Dev, SIT, UAT, Prod) are assumed to reside in the **same Databricks workspace** within the same Azure region. The app authenticates with a single workspace host and accesses all catalogs via Unity Catalog's cross-catalog permissions model.

**If catalogs span multiple workspaces** (e.g., Prod in a separate hardened workspace), the following implications apply:
- Each workspace requires its own authentication context (PAT or OAuth token)
- The app would need a multi-workspace API client that routes requests to the correct host based on catalog name
- Service principal credentials must be provisioned in each workspace
- This is a significant architectural change and should be validated before implementation

**Current assumption:** Single workspace, multiple catalogs. If multi-workspace is required, it should be treated as a Phase 2 enhancement.

### 11.7 Prod Access Request Gate

Any authenticated user can request access to Prod-environment tables, but the following controls apply:

1. **Enhanced review:** Prod table requests always require the full fixed reviewer set plus data owner reviewers — the approval mode for Prod requests is forced to `parallel` regardless of the admin's default setting
2. **Sensitivity highlighting:** `RESTRICTED` and `CONFIDENTIAL` tables are highlighted with a warning badge in the Data Selection step
3. **Manager attestation (optional):** An org-level config flag (`REQUIRE_MANAGER_APPROVAL_FOR_PROD`) can enable a manager approval step before the governance review begins. When enabled, the requestor must provide their manager's email, who receives an approval card before the governance reviewers are notified.
4. **No hard block:** There is no technical restriction preventing non-Prod users from requesting Prod access — the governance is handled through the reviewer workflow, not through user role gates.

### 11.8 Quarterly Audit / Compliance Reporting

A **Quarterly Audit Job** runs on the first business day of each quarter (Jan 1, Apr 1, Jul 1, Oct 1) and generates a compliance report:

**Report contents:**
- Total requests submitted, approved, rejected, cancelled in the quarter
- Average time-to-approval (business days)
- SLA breach count and details
- All active access grants (approved requests where `access_end >= today`)
- All expired and revoked grants in the quarter
- Sensitivity breakdown: count of requests by max sensitivity tier
- Reviewer workload: approvals per reviewer, average response time
- Data owner tag coverage: percentage of tables with valid `data_owner` tags

**Output format:** Delta table (`quarterly_audit_reports`) with one row per quarter, plus a downloadable CSV export from the Admin — Audit & Compliance Reports screen.

**Owner:** Data Governance team lead is responsible for reviewing the quarterly report within 5 business days of generation. The job is defined in the main `databricks.yml` (see Section 10.1).

---

## 12. Caching & Performance

| Strategy | Implementation |
|---|---|
| Table inventory cache | `@st.cache_data(ttl=300)` on Unity Catalog list calls |
| `information_schema` size cache | `@st.cache_data(ttl=600)` on SQL Execution size queries |
| User / group cache | `@st.cache_data(ttl=3600)` on SCIM user/group lookups |
| Dashboard metrics cache | `@st.cache_data(ttl=60)` on request count queries |
| API retry logic | Exponential backoff, max 3 retries on transient errors |
| Pagination | Unity Catalog table listing paginated at 100 results per page |

### 12.1 SQL Warehouse Cold Start

SQL Warehouse cold start (when the warehouse has been auto-stopped due to inactivity) can add **30–120 seconds** to the first query in a session. This directly impacts Step 1 (Data Selection), where `information_schema.tables` size queries are issued against the warehouse.

**Mitigation strategy:**

- The warehouse used by the app should be configured with **auto-stop of no less than 30 minutes idle** during business hours to reduce cold-start frequency
- A recommended configuration is to use a **scheduled warm-up job** (a no-op SQL query) that runs every 20 minutes during business hours (08:00–18:00, Mon–Fri) to keep the warehouse live
- If the warehouse is cold on page load, the app displays a loading spinner with an informational message: *"Data warehouse is starting up — this may take up to 2 minutes on first load"*
- The `information_schema` size query is batched per catalog (one query returns all table sizes for that catalog), not issued per-table, to minimise round trips once the warehouse is warm
- Cold-start latency is acknowledged in the NFR response time targets (see Section 15.2 footnote)

---

## 13. Error Handling

| Scenario | User Experience |
|---|---|
| Databricks API unavailable | Banner: "Data service temporarily unavailable. Please try again." Draft auto-saved. |
| No tables found in catalog | Informational empty state with guidance to check catalog permissions |
| Table tags missing (no sensitivity detected) | Badge shown as `UNCLASSIFIED`; user can proceed but is warned |
| Form validation failure | Inline error text with highlighted field; submit blocked |
| Submit failure | Toast notification with error details; request preserved as draft |
| Teams webhook failure | Approval workflow continues in-app; reviewer home page reflects pending state; admin alerted |
| Expired session | Redirect to login with "Session expired — please re-authenticate" message |
| Draft > 30 days old | Draft cleanup job deletes stale drafts; user notified via in-app banner on next login |

---

## 14. Acceptance Criteria

### Requestor Dashboard

- Displays accurate counts for Pending, Approved, and Expired requests
- Recent requests list shows last 10 requests sorted by `updated_at DESC`
- Status badges shown for all states: DRAFT, SUBMITTED, UNDER REVIEW, AWAITING RESPONSE, APPROVED, REJECTED, EXPIRED, REVOKED, CANCELLED
- Department filter correctly filters the request list
- "New Contract Request" navigates to Step 1 with clean session state
- Approved requests within 30 days of expiry show an "Extend Access" action
- Non-terminal requests show a "Cancel Request" action
- Rejected requests show "Create New Request" and "Appeal" actions; Appeal is disabled if `appeal_count >= 1`

### Data Selection (Step 1)

- Tables loaded from all four catalogs (Dev, SIT, UAT, Prod) via Unity Catalog API
- Table size populated per-catalog batch from `information_schema.tables` via SQL Execution API
- Sensitivity badges auto-populated from Unity Catalog table tags using defined tag taxonomy
- Tables with `pii=true` and `sensitivity=PUBLIC` are displayed at `CONFIDENTIAL` tier
- Tables with missing `sensitivity` tag display `UNCLASSIFIED` badge with a warning
- Search filters tables by name in real-time
- Toggle selection updates summary bar with count and environments represented
- At least 1 table must be selected before proceeding to Step 2
- Cold warehouse start surfaces a loading indicator with explanatory message

### Requestor Information (Step 2)

- Name and Department are required; Name auto-populated from SCIM where possible
- Department dropdown populated from SCIM groups
- Purpose dropdown contains exactly: Reporting, Analytics and Data Science, Data Exploration, Data Migration
- "Save Draft" persists form state to Delta table

### Access & Planning (Step 3)

- Start date cannot be in the past
- End date must be after start date; invalid range shows inline error
- End date picker does not allow selection beyond `start_date + 365 days`
- Server-side validation also enforces the 365-day cap and rejects non-compliant submissions
- Duration in days calculated and displayed in real-time
- Long-term planning textarea enforces 50–2000 character range with live counter
- Review & Confirm page (Step 4) displays the resolved approval mode as a read-only field
- "Save Draft" persists form state

### Review & Confirm (Step 4)

- All sections display entered data accurately including sensitivity badges and environments
- Resolved approval mode displayed as read-only (parallel or sequential)
- Edit buttons navigate to correct step with state preserved
- All three attestation checkboxes must be checked before submit is enabled
- "Confirm & Submit Request" creates record with `status = 'submitted'` and `approval_mode` set from admin config
- On success, redirects to Dashboard with success toast
- `reviewer_assignments` populated on submission: fixed reviewers from config table + data owners from UC table tags

### Access Renewal / Extension

- "Extend Access" visible only for approved requests within 30 days of `access_end`
- Extension form pre-fills original tables (read-only), original purpose (editable), and requires new end date and renewal justification
- New end date must be after current `access_end`; 365-day cap applies from new start date
- Submitting an extension creates a new `access_request` with `parent_request_id` linking to original
- Extension goes through full approval workflow; original access continues until new request resolves

### Request Cancellation

- "Cancel Request" available for requests in `submitted`, `under_review`, `awaiting_response` states only
- Confirmation dialog shown before cancellation is executed
- On cancellation: `status = 'cancelled'`, all pending `reviewer_assignments` set to `cancelled`, reviewers notified via Teams, event logged to `audit_log`
- Cancelled requests visible on dashboard with `CANCELLED` badge; no actions available

### Rejection Appeal

- "Appeal" action visible on rejected requests where `appeal_count = 0`
- Appeal is disabled (greyed out with tooltip) when `appeal_count >= 1`
- Submitting an appeal changes status to `appealing` and increments `appeal_count` to 1
- Original reviewers re-notified via Teams
- Reviewer can approve (overturning rejection), reject again (terminal), or add clarification comment
- A second rejection after appeal is terminal; "Appeal" action permanently disabled on that request

### Reviewer Home

- Displays count of pending reviews assigned to the current reviewer
- Requests in `pending` or `awaiting_response` state listed, sorted by submission date ascending (oldest first)
- Each item shows: requestor name, purpose, table count, max sensitivity, days pending, SLA status indicator (on-time / at-risk / breached)
- SLA indicator turns amber at 3 days, red at 5 days

### Request Detail / Review Page

- Renders in read-only mode for requestors; renders with action buttons for assigned reviewers
- Reviewer progress panel shows per-reviewer status (parallel mode) or sequential queue position
- In parallel mode: shows ✅ approved, ⏳ pending, ❌ rejected per reviewer
- In sequential mode: shows current active reviewer and remaining queue
- Per-environment approval status shown for multi-catalog requests
- Open comments block the approve action; button is disabled with tooltip explanation

### Approval Workflow

- Reviewer list correctly combines fixed reviewers + data owners resolved from UC table tags
- `reviewer_assignments.catalog_name` set correctly for per-environment scoping
- Parallel mode notifies all reviewers simultaneously via Teams
- Sequential mode notifies one reviewer at a time in configured `sequence_order`
- Request cannot transition to `approved` while any comment has `resolution_status = 'open'`
- Unanimous approval required: all reviewers across all environments must approve
- Any single rejection immediately moves request to `rejected`
- On final approval, READ permissions granted via Databricks Permissions API for approved access window
- For multi-environment requests: per-environment READ grants issued as each environment reaches unanimous approval (partial grants allowed)

### SLA Escalation

- Escalation job runs weekdays at 07:00 UTC
- At 3 business days: automated Teams reminder sent to reviewer
- At 5 business days: reassignment to `deputy_reviewer_id` if configured; admin notified
- At 7 business days: admin receives escalation Teams notification
- At 10 business days: request flagged as "SLA Breached" on admin dashboard
- SLA timer correctly excludes weekends and configured public holidays

### Admin Screens

- Reviewer Management: admin can add, edit, and deactivate reviewers; changes take effect for new requests only; `sequence_order` and `deputy_reviewer_id` configurable per reviewer
- All Requests View: filters by status, environment, department, date range; SLA breach flag visible
- Audit & Compliance Reports: audit log viewer with filters; downloadable CSV export; quarterly report viewer
- Dead-letter queue visible in admin dashboard; admin can manually re-trigger failed notifications

### Teams Integration

- Adaptive Card delivered to reviewer on assignment (DM by default)
- Inline approve / reject / comment actions captured and written to Delta tables via Power Automate callback
- Requestor notified via Teams on: `under_review` → `awaiting_response`, `rejected`, `approved`, `cancelled`
- Reviewer notified on: comment resolved, exception-approved, request cancelled, SLA reminder
- Failed notifications written to `notification_dlq`; admin alerted

### Cross-Cutting

- Draft auto-saves on every step transition
- Drafts older than 30 days deleted by scheduled cleanup job; user notified via in-app banner on next login
- All API calls handle 401 / 403 / 500 gracefully
- Role detection correctly identifies admin, reviewer, and requestor roles; dual-role users see both Reviewer Home and Requestor Dashboard in navigation
- Unit test coverage ≥ 80% across `src/api/` and `src/services/`
- CI pipeline passes on all PRs
- DAB deploy succeeds to dev, staging, and prod targets
- `deploy_rollback.yml` successfully redeploys a prior release tag to production

---

## 15. Non-Functional Requirements

### 15.1 Expected Load

| Metric | Estimate |
|---|---|
| Concurrent users | 50–100 (peak during quarterly access reviews) |
| Requests per month | 200–500 |
| Tables browsed per session | 50–200 (Step 1 browsing) |
| API calls per request | ~15–30 (UC catalog lookups, SCIM, SQL execution) |
| Reviewer notifications per day | 10–50 |

### 15.2 Response Time Targets

| Operation | Target |
|---|---|
| Page load (cached data) | < 2 seconds |
| Table catalog browsing (cached) | < 3 seconds |
| Table catalog browsing (uncached, 100 tables) | < 8 seconds |
| `information_schema` size query (warm warehouse) | < 5 seconds per catalog batch |
| `information_schema` size query (cold warehouse) | < 120 seconds *(see Section 12.1)* |
| Request submission | < 3 seconds |
| Approval action (in-app) | < 2 seconds |
| Dashboard metrics load | < 2 seconds |

> **Note:** Cold warehouse start times of up to 120 seconds are an accepted limitation. The app surfaces a loading indicator with an explanatory message in this case. Targets above assume a warm warehouse.

### 15.3 Availability

- Target: **99.5% uptime** during business hours (08:00–18:00 local time, Mon–Fri)
- The app depends on Databricks workspace availability — outages in Databricks directly impact the app
- Power Automate / Teams integration has a separate availability profile — app must degrade gracefully (in-app review only) if Teams is unavailable

### 15.4 Scalability Considerations

- Unity Catalog API pagination limits table browsing to manageable chunks (100/page)
- SQL Execution API queries for `information_schema` should be batched (one query per catalog, not per table)
- Delta table queries for dashboard metrics use partitioning by `created_at` for performance
- Streamlit session state is per-user and ephemeral — no shared state across users

---

## 16. Known Limitations / Out of Scope

The following capabilities are **intentionally excluded** from this version:

| Item | Rationale |
|---|---|
| WRITE / MODIFY access grants | App only handles READ access. Write access requires a separate, more rigorous governance process. |
| Cross-workspace requests | All catalogs assumed in a single workspace. Multi-workspace is a Phase 2 consideration. |
| Mobile support | Streamlit desktop/web only. Mobile-responsive layout is a best-effort, not a guaranteed UX. |
| Email notifications | Teams is the sole notification channel. Email fallback is not implemented. |
| Bulk access requests (CSV upload) | Requestors must select tables individually via the UI. Bulk upload is a future enhancement. |
| Access request templates | No pre-built templates for recurring access patterns. Users must fill out each request. |
| External user access | Only users with Databricks workspace SSO credentials can use the app. External/guest users are not supported. |
| Fine-grained row/column access | App manages table-level READ only. Row-level filtering and column masking are Unity Catalog features managed outside this app. |
| Automated table discovery / recommendation | No AI/ML-driven table recommendations based on user role or past requests. |
| Real-time webhook notifications (no polling) | The Streamlit app polls Delta tables for state changes; true real-time push notifications would require a WebSocket layer. |

---

## 17. Glossary / Definitions

| Term | Definition |
|---|---|
| **Access Request** | A formal submission by a requestor to obtain READ access to one or more Unity Catalog tables for a defined time period. |
| **Requestor** | Any authenticated Databricks user who submits an access request. |
| **Fixed Reviewer** | A pre-configured governance team member (e.g., Data Governance, Compliance, Security) who reviews all requests regardless of the tables involved. |
| **Data Owner Reviewer** | A reviewer dynamically resolved from the `data_owner` tag on a specific Unity Catalog table. Each table may have a different data owner. |
| **Admin** | A user with elevated privileges who manages the reviewer list, views all requests, and runs compliance reports. |
| **Sensitivity Tier** | A classification level assigned to a table via UC tags: `PUBLIC`, `INTERNAL`, `CONFIDENTIAL`, `RESTRICTED`. |
| **Contract Request** | Synonym for "access request" — originates from the business term for a formal data access agreement. |
| **Unity Catalog (UC)** | Databricks' unified governance solution for data and AI assets. |
| **Catalog Environment** | One of four named catalogs mapping to deployment environments: `dev` (Development), `sit` (System Integration Testing), `uat` (User Acceptance Testing), `prod` (Production). |
| **Power Automate** | Microsoft's cloud-based workflow automation platform, used here for Teams notification orchestration. |
| **Adaptive Card** | A platform-agnostic UI card format used in Teams to present interactive approval/reject/comment actions to reviewers. |
| **DAB (Databricks Asset Bundle)** | A packaging and deployment format for Databricks applications and jobs. |
| **SCIM** | System for Cross-domain Identity Management — the protocol used to sync users and groups from Azure AD to Databricks. |
| **Approval Mode** | Either `parallel` (all reviewers notified simultaneously) or `sequential` (reviewers act in a defined order). |
| **SLA** | Service Level Agreement — the maximum acceptable time for a reviewer to act on a pending request. |
| **Expiry Monitor** | A scheduled job that revokes READ permissions when the `access_end` date has passed. |

---

## 18. Runbook / Operations

### 18.1 Manually Revoking Access

**Scenario:** A user's access needs to be terminated before the `access_end` date (e.g., employee departure, policy violation).

1. Admin navigates to Admin — All Requests View
2. Filters to `status = 'approved'` and locates the request
3. Clicks **[Revoke Access]**
4. System:
   - Updates `access_requests.status = 'revoked'`, `revoked_at = NOW()`
   - Calls Permissions API to remove READ grant on all tables in the request
   - Logs `revoke` event to `audit_log`
   - Notifies requestor and reviewers via Teams

### 18.2 Recovering a Failed Expiry Monitor Job

**Scenario:** The daily expiry monitor job fails (e.g., warehouse down, permissions error).

1. Check job run history in Databricks workspace → Jobs → "Access Expiry Monitor"
2. Review error logs
3. Common fixes:
   - Warehouse is stopped → start the warehouse and re-run
   - Service principal permissions expired → rotate PAT (see 8.4)
   - Delta table lock → wait and re-run
4. Re-trigger manually: `databricks jobs run-now --job-id <expiry_monitor_job_id>`
5. If data inconsistency suspected: query `access_requests WHERE status = 'approved' AND access_end < CURRENT_DATE()` and manually revoke using the Admin UI

### 18.3 Re-triggering a Stuck Approval

**Scenario:** A reviewer's Adaptive Card was not delivered (Power Automate failure) or the reviewer is unresponsive beyond the SLA.

1. Check `reviewer_assignments` for the request — confirm `status = 'pending'`
2. Check Power Automate flow run history for delivery failures
3. If card was delivered but reviewer unresponsive:
   - Admin reassigns to a deputy via Admin UI (see 6.2.2)
   - Or admin manually records the approval/rejection on behalf of the reviewer (with audit note)
4. If card was not delivered:
   - Admin re-triggers the notification via Admin UI → "Resend Notification" button
   - System calls Power Automate with the same payload

### 18.4 Managing the Reviewer Delta Table

**Scenario:** Adding, removing, or updating a fixed reviewer.

Via Admin UI (preferred):
1. Admin navigates to Admin — Reviewer Management
2. Add/edit/deactivate reviewers
3. Changes take effect immediately for new requests; in-flight requests are not affected

Via SQL (emergency):
```sql
-- Add a new fixed reviewer
INSERT INTO main.data_curator_app.reviewers
  (reviewer_id, reviewer_name, reviewer_email, reviewer_type,
   teams_user_id, is_active, sequence_order, default_approval_mode,
   deputy_reviewer_id, updated_at)
VALUES
  ('rev-new', 'Jane Doe', 'jane@company.com', 'fixed',
   '<teams_azure_ad_object_id>', TRUE, 3, 'parallel',
   NULL, CURRENT_TIMESTAMP());

-- Deactivate a reviewer (preserves history; does not delete)
UPDATE main.data_curator_app.reviewers
SET is_active = FALSE, updated_at = CURRENT_TIMESTAMP()
WHERE reviewer_id = 'rev-old';

-- Update sequence order
UPDATE main.data_curator_app.reviewers
SET sequence_order = 2, updated_at = CURRENT_TIMESTAMP()
WHERE reviewer_id = 'rev-jane';

-- Assign a deputy reviewer
UPDATE main.data_curator_app.reviewers
SET deputy_reviewer_id = 'rev-backup', updated_at = CURRENT_TIMESTAMP()
WHERE reviewer_id = 'rev-primary';
```

### 18.5 Handling a Failed Deploy

**Scenario:** A production deploy introduces a bug.

1. Verify the issue on the Request Detail page or via admin dashboard
2. Rollback: trigger `deploy_rollback.yml` with the previous release tag
3. If schema migration caused the issue: use Delta time travel to restore affected tables
4. Post-incident: add regression test, update runbook

---

*End of Document*
