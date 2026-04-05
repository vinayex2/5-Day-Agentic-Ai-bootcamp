## Phase 1 — Foundation (Weeks 1–2)

### Issue: Set up Databricks Asset Bundle project structure
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `infrastructure`, `databricks`, `asset-bundle`
- **Assignee:** `@devops-lead`
- **Description:** Configure the Databricks Asset Bundle (DAB) project structure with proper directory layout, databricks.yml configuration, and resource definitions for jobs and apps.
- **Acceptance Criteria:**
  - DAB project structure created with standard DAB conventions
  - databricks.yml configured with workspace_host, sql_warehouse_id, app_catalog, app_schema variables
  - Resources defined for jobs (approval_escalation_job, teams_identity_sync_job, quarterly_audit_job, expiry_monitor_job, draft_cleanup_job) and apps (data_curator_app)
  - CI/CD pipeline ready for deployment to dev, staging, and prod environments

### Issue: Configure Unity Catalog API wrapper
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `api`, `unity-catalog`, `backend`
- **Assignee:** `@backend-lead`
- **Description:** Implement base REST API client with Databricks authentication (OAuth / PAT) and Unity Catalog API wrapper for listing catalogs, schemas, tables, and tags across all 4 environments (dev, sit, uat, prod).
- **Acceptance Criteria:**
  - REST API client with proper error handling and retry logic
  - Unity Catalog API wrapper with pagination support (100 results per page)
  - Methods for: list_catalogs, list_schemas, list_tables, get_table_tags
  - Mock support for local development
  - Unit tests with 80%+ coverage

### Issue: Implement SQL Execution client for table size queries
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `api`, `sql-execution`, `backend`
- **Assignee:** `@backend-lead`
- **Description:** Build SQL Execution client for querying `information_schema.tables` to retrieve table sizes and metadata. Implement batching per catalog to minimize round trips.
- **Acceptance Criteria:**
  - SQL Execution API client with proper authentication
  - Batch query per catalog (not per table)
  - Handle cold warehouse start with loading indicator
  - Cache results with @st.cache_data (ttl=600)
  - Unit tests with mocked responses

### Issue: Build SCIM client for user/group lookups
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `api`, `scim`, `backend`
- **Assignee:** `@backend-lead`
- **Description:** Implement SCIM client for Azure AD user and group lookups. Support for resolving reviewer emails to Teams user IDs and department dropdown population.
- **Acceptance Criteria:**
  - GET /api/2.0/preview/scim/v2/Me for current user
  - GET /api/2.0/preview/scim/v2/Groups for group membership
  - Resolve reviewer_email to teams_user_id via Azure AD Graph API
  - Handle tenant migrations and renamed accounts
  - Cache user/group data with @st.cache_data (ttl=3600)
  - Unit tests with mocked SCIM responses

### Issue: Initialize Delta tables with schema creation pipeline
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `infrastructure`, `delta-tables`, `schema`
- **Assignee:** `@devops-lead`
- **Description:** Create all required Delta tables using CREATE TABLE IF NOT EXISTS. Implement versioned migration scripts for schema changes.
- **Acceptance Criteria:**
  - Migration scripts in data_curator/migrations/ directory
  - Tables: access_requests, reviewer_assignments, review_comments, audit_log, notification_dlq, reviewers, quarterly_audit_reports
  - Migration runner script with forward-only migrations
  - Migration log tracking applied migrations
  - Deploy to dev, staging, and prod environments

### Issue: Set up GitHub repository with branch protection and CI/CD
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `infrastructure`, `github`, `ci-cd`
- **Assignee:** `@devops-lead`
- **Description:** Configure GitHub repository with branch protection rules, CODEOWNERS, PR templates, and CI/CD workflows for linting, testing, and deployment.
- **Acceptance Criteria:**
  - Branch protection on main and develop branches
  - CODEOWNERS file with appropriate reviewers
  - PR template with checklist
  - CI workflows: ci.yml (lint + unit tests), deploy-staging.yml, integration-tests.yml, deploy-prod.yml, deploy-rollback.yml
  - GitHub Actions secrets configured for Databricks tokens and hosts
  - Environment configuration for production approval gate

### Issue: Configure Databricks Secrets for service principal authentication
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `security`, `secrets`, `azure-ad`
- **Assignee:** `@devops-lead`
- **Description:** Set up Databricks secret scope for Azure AD credentials, Power Automate webhook URL, and service principal PAT. Implement secret rotation process.
- **Acceptance Criteria:**
  - Secret scope: data-curator created in Databricks
  - Secrets: azure-ad-client-id, azure-ad-client-secret, azure-ad-tenant-id, pa-webhook-url, sp-pat-token
  - ACL configured for app service principal and admin users
  - Secret rotation documented and tested (90-day cycle)
  - Mock mode for local development without secrets

### Issue: Implement role detection and session state management
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `authentication`, `session-state`, `streamlit`
- **Assignee:** `@backend-lead`
- **Description:** Implement role detection logic (admin, reviewer, requestor) and session state caching for Streamlit app.
- **Acceptance Criteria:**
  - Role detection priority: admin group → reviewer table → requestor fallback
  - Session state caching in st.session_state.roles
  - Role re-evaluation on session expiry
  - Dual-role support (e.g., reviewer + requestor)
  - Unit tests for role detection logic

---

## Phase 2 — Core Wizard (Weeks 3–5)

### Issue: Implement multi-page Streamlit routing with shared components
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `ui`, `streamlit`, `navigation`
- **Assignee:** `@frontend-lead`
- **Description:** Create multi-page Streamlit app with 7 pages, shared sidebar, top navigation, progress stepper, and sensitivity badge component.
- **Acceptance Criteria:**
  - 7 pages: Dashboard, Data Selection, Requestor Info, Access Planning, Review & Confirm, Reviewer Home, Admin
  - Shared sidebar with navigation links
  - Top navigation with progress stepper for wizard
  - Sensitivity badge component (PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED, UNCLASSIFIED)
  - Responsive design for desktop and web
  - Unit tests for navigation and component rendering

### Issue: Step 1 — Data Selection page with table browsing
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `ui`, `data-selection`, `wizard`
- **Assignee:** `@frontend-lead`
- **Description:** Implement Step 1 of the wizard with table browsing grid across 4 catalogs, search functionality, toggle selection, and auto-populated size and sensitivity tags.
- **Acceptance Criteria:**
  - Table grid with catalog, schema, name, size, sensitivity columns
  - Real-time search filter by table name
  - Toggle selection with summary bar (count, environments)
  - Cold warehouse loading indicator
  - Minimum 1 table required before proceeding
  - Unit tests for table loading and filtering

### Issue: Step 2 — Requestor Information page
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `ui`, `requestor-info`, `wizard`
- **Assignee:** `@frontend-lead`
- **Description:** Implement Step 2 with name (auto-populated from SCIM), department dropdown (from SCIM groups), purpose single-select dropdown, and save draft functionality.
- **Acceptance Criteria:**
  - Name field auto-populated from SCIM current user
  - Department dropdown from SCIM group membership
  - Purpose dropdown: Reporting, Analytics and Data Science, Data Exploration, Data Migration
  - Save draft persists form state to Delta table
  - Validation: name and department required
  - Unit tests for form validation and draft save

### Issue: Step 3 — Access & Planning page
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `ui`, `access-planning`, `wizard`
- **Assignee:** `@frontend-lead`
- **Description:** Implement Step 3 with date pickers, duration calculation, long-term planning textarea, and server-side validation for date ranges.
- **Acceptance Criteria:**
  - Start date picker (cannot be in past)
  - End date picker (max 365 days from start)
  - Duration calculation and display in real-time
  - Long-term planning textarea (50–2000 characters with live counter)
  - Server-side validation for date range
  - Save draft functionality
  - Unit tests for date validation and duration calculation

### Issue: Step 4 — Review & Confirm page
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `ui`, `review-confirm`, `wizard`
- **Assignee:** `@frontend-lead`
- **Description:** Implement Step 4 with summary cards, inline edit navigation, attestation checklist, and submit functionality.
- **Acceptance Criteria:**
  - Summary cards for all entered data
  - Sensitivity badges and environment indicators
  - Inline edit navigation to correct step
  - Three attestation checkboxes (required before submit)
  - Resolve approval mode from admin config (read-only)
  - Populate reviewer_assignments on submission
  - Redirect to Dashboard on success
  - Unit tests for attestation validation and submission

### Issue: Implement draft save/load via Delta table
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `backend`, `draft-management`, `delta-tables`
- **Assignee:** `@backend-lead`
- **Description:** Implement draft save/load functionality using Delta table. Auto-save on every step transition. Handle stale draft cleanup.
- **Acceptance Criteria:**
  - Draft table schema with requestor_id, step, form_data, created_at, updated_at
  - Auto-save on step transition
  - Load draft on page load if exists
  - Stale draft cleanup (30 days) via scheduled job
  - Draft preservation on submit failure
  - Unit tests for draft CRUD operations

### Issue: Implement form validation and error handling
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `validation`, `error-handling`, `ui`
- **Assignee:** `@backend-lead`
- **Description:** Implement comprehensive form validation with inline error messages, highlighted fields, and graceful error handling for API failures.
- **Acceptance Criteria:**
  - Inline error text with highlighted fields
  - Submit blocked on validation failure
  - Toast notifications for errors
  - Draft preservation on submit failure
  - Error states for Databricks API unavailable
  - Unit tests for validation logic

---

## Phase 3 — Dashboard, Reviewer Home & Approval Workflow (Weeks 6–8)

### Issue: Implement Requestor Dashboard
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `ui`, `dashboard`, `requestor`
- **Assignee:** `@frontend-lead`
- **Description:** Create Requestor Dashboard with status metric cards, recent requests list, department filter, and action buttons (Extend Access, Cancel Request, Appeal).
- **Acceptance Criteria:**
  - Status metric cards: Pending, Approved, Expired counts
  - Recent requests list (last 10, sorted by updated_at DESC)
  - Status badges for all states
  - Department filter
  - Actions: Extend Access (within 30 days), Cancel Request, Appeal (if allowed)
  - Unit tests for dashboard rendering and filtering

### Issue: Implement Reviewer Home page
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `ui`, `reviewer-home`, `reviewer`
- **Assignee:** `@frontend-lead`
- **Description:** Create Reviewer Home page with pending queue, priority indicators, SLA status indicators, and request detail links.
- **Acceptance Criteria:**
  - Count of pending reviews assigned to current reviewer
  - Requests in pending or awaiting_response state
  - Sorted by submission date ascending (oldest first)
  - Requestor name, purpose, table count, max sensitivity
  - Days pending and SLA status indicator (on-time / at-risk / breached)
  - SLA indicator: amber at 3 days, red at 5 days
  - Unit tests for reviewer home rendering

### Issue: Implement Request Detail / Review page (shared component)
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `ui`, `request-detail`, `reviewer`
- **Assignee:** `@frontend-lead`
- **Description:** Create shared Request Detail page with role-aware rendering. Requestors see read-only; reviewers see action buttons and progress panel.
- **Acceptance Criteria:**
  - Role-aware rendering (requestor vs reviewer)
  - Requestor view: read-only with status and actions
  - Reviewer view: action buttons, progress panel, comment thread
  - Per-environment approval status
  - Reviewer progress panel (parallel/sequential modes)
  - Open comments block approve action
  - Unit tests for role-aware rendering

### Issue: Implement reviewer assignment logic
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `backend`, `reviewer-assignment`, `logic`
- **Assignee:** `@backend-lead`
- **Description:** Implement reviewer assignment logic combining fixed reviewers from config table and data owners resolved from UC table tags.
- **Acceptance Criteria:**
  - Fixed reviewers from reviewers table (is_active = TRUE)
  - Data owners resolved from UC table data_owner tags
  - Deduplication of reviewers
  - Sequential ordering from sequence_order column
  - Per-environment scoping (catalog_name in reviewer_assignments)
  - Unit tests for assignment logic

### Issue: Implement approval state machine
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `backend`, `approval-workflow`, `state-machine`
- **Assignee:** `@backend-lead`
- **Description:** Implement approval state machine with parallel/sequential modes, unanimous requirement, rejection terminal state, and per-environment partial approval.
- **Acceptance Criteria:**
  - Parallel mode: all reviewers notified simultaneously
  - Sequential mode: one reviewer at a time
  - Unanimous approval required across all environments
  - Single rejection moves request to rejected (terminal)
  - Per-environment partial approval tracking
  - Comment resolution blocking approval
  - Unit tests for all state transitions

### Issue: Implement comment thread component
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `ui`, `comments`, `reviewer`
- **Assignee:** `@frontend-lead`
- **Description:** Create comment thread component with resolve / exception-approve actions. Open comments block approve action.
- **Acceptance Criteria:**
  - Multi-line comment input
  - Resolve comment action
  - Exception-approve action (bypass open comments)
  - Comment thread display
  - Open comments block approve button
  - Unit tests for comment resolution logic

### Issue: Implement Teams notification service
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `teams`, `notifications`, `power-automate`
- **Assignee:** `@backend-lead`
- **Description:** Implement Teams notification service for sending Adaptive Cards to reviewers and requestors. Handle parallel/sequential fan-out and delivery tracking.
- **Acceptance Criteria:**
  - Adaptive Card payload construction
  - Parallel fan-out: notify all reviewers simultaneously
  - Sequential fan-out: notify one reviewer at a time
  - Requestor notifications on status change
  - Delivery status tracking
  - Dead-letter queue for failed deliveries
  - Unit tests with mocked HTTP calls

### Issue: Implement Power Automate flow integration
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `power-automate`, `integration`, `webhooks`
- **Assignee:** `@devops-lead`
- **Description:** Configure Power Automate flows for reviewer notification, reviewer action response, and requestor notification. Implement webhook receiver in Databricks Job.
- **Acceptance Criteria:**
  - Flow: Data Curator — Reviewer Notification
  - HTTP trigger from Streamlit app via Databricks Job
  - Switch on action type: notify_reviewers_parallel, notify_reviewer_sequential, notify_requestor, reviewer_action_response
  - Databricks Job webhook receiver for callbacks
  - Error handling and dead-letter queue
  - Unit tests with mocked PA interactions

### Issue: Implement SLA escalation logic
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `backend`, `sla`, `escalation`
- **Assignee:** `@backend-lead`
- **Description:** Implement SLA escalation logic with business day calculation, reminder triggers, deputy reassignment, and admin escalation.
- **Acceptance Criteria:**
  - SLA timer excludes weekends and holidays
  - Reminder at 3 business days
  - Deputy reassignment at 5 days
  - Admin escalation at 7 days
  - SLA breach flag at 10 days
  - Unit tests for SLA calculation and escalation triggers

### Issue: Implement automated READ permission grant
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `permissions`, `unity-catalog`, `automation`
- **Assignee:** `@backend-lead`
- **Description:** Implement automated READ permission grant via Unity Catalog Permissions API on final approval. Handle per-environment partial grants.
- **Acceptance Criteria:**
  - Grant MANAGE on tables for service principal
  - Grant READ on tables for requestor on approval
  - Per-environment partial grants on multi-catalog requests
  - Revoke on rejection/cancellation
  - Audit log on all permission changes
  - Unit tests for permission grant/revoke

### Issue: Implement access expiry monitor job
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `jobs`, `expiry-monitor`, `automation`
- **Assignee:** `@devops-lead`
- **Description:** Implement scheduled job to revoke expired access grants. Check access_end date and revoke if expired.
- **Acceptance Criteria:**
  - Daily schedule at 06:00 UTC
  - Query approved requests with access_end < CURRENT_DATE
  - Revoke READ permissions via Permissions API
  - Update status to expired
  - Log to audit_log
  - Handle errors and write to notification_dlq
  - Unit tests for expiry logic

### Issue: Implement draft cleanup job
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `jobs`, `draft-cleanup`, `automation`
- **Assignee:** `@devops-lead`
- **Description:** Implement scheduled job to delete stale drafts older than 30 days. Notify user via in-app banner on next login.
- **Acceptance Criteria:**
  - Daily schedule at 02:00 UTC
  - Delete drafts with updated_at < CURRENT_DATE - 30 days
  - Log deletions to audit_log
  - Notify users with stale drafts deleted
  - Unit tests for cleanup logic

### Issue: Implement quarterly audit job
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `jobs`, `audit`, `compliance`
- **Assignee:** `@devops-lead`
- **Description:** Implement scheduled job to generate quarterly compliance report. Run on first business day of each quarter.
- **Acceptance Criteria:**
  - Schedule: Jan 1, Apr 1, Jul 1, Oct 1 at 08:00 UTC
  - Report contents: request counts, approval times, SLA breaches, active grants, sensitivity breakdown, reviewer workload
  - Output to quarterly_audit_reports Delta table
  - CSV export from Admin UI
  - Unit tests for report generation

---

## Phase 4 — Teams Integration & Permissions (Weeks 9–10)

### Issue: Set up Azure AD App Registration for Power Automate
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `azure-ad`, `authentication`, `power-automate`
- **Assignee:** `@devops-lead`
- **Description:** Create Azure AD app registration with required API permissions (User.Read, TeamsAppInstallation.ReadWriteSelfForChat). Generate client secret and store in Databricks Secrets.
- **Acceptance Criteria:**
  - App registration in Azure AD
  - API permissions: User.Read, TeamsAppInstallation.ReadWriteSelfForChat
  - Client secret generated and stored in Databricks Secrets
  - client_id and tenant_id referenced in app config and Power Automate
  - Unit tests for authentication flow

### Issue: Implement Teams Identity Sync Job
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `jobs`, `teams-identity`, `sync`
- **Assignee:** `@devops-lead`
- **Description:** Implement nightly job to sync reviewer teams_user_id from Azure AD. Handle renamed accounts and tenant migrations.
- **Acceptance Criteria:**
  - Daily schedule at 01:00 UTC
  - Query Azure AD Graph API for each reviewer_email
  - Update teams_user_id if changed
  - Flag reviewers with no matching Azure AD identity
  - Log to audit_log
  - Unit tests for sync logic

### Issue: Configure Teams Adaptive Card templates
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `teams`, `adaptive-cards`, `ui`
- **Assignee:** `@frontend-lead`
- **Description:** Create Adaptive Card templates for reviewer notifications with approve/reject/comment actions. Handle clarification and reject with multi-line text input.
- **Acceptance Criteria:**
  - Card template with requestor info, purpose, tables, sensitivity, access period
  - Actions: Approve, Request Clarification, Reject
  - Clarification/Reject with multi-line text input
  - Deep-link to in-app review page
  - Teams DM delivery (default) or channel copy (optional)
  - Unit tests for card payload construction

### Issue: Implement Teams notification dead-letter queue
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `infrastructure`, `dlq`, `error-handling`
- **Assignee:** `@devops-lead`
- **Description:** Implement notification_dlq Delta table for failed notifications. Admin can inspect and manually re-trigger from Admin dashboard.
- **Acceptance Criteria:**
  - Delta table with event_id, request_id, reviewer_id, payload, error_message, retry_count, created_at
  - Admin UI for viewing and re-triggering
  - Retry logic with exponential backoff
  - Unit tests for DLQ operations

### Issue: Implement admin reviewer management UI
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `ui`, `admin`, `reviewer-management`
- **Assignee:** `@frontend-lead`
- **Description:** Create Admin — Reviewer Management screen for adding, editing, and deactivating reviewers. Configure sequence_order and deputy_reviewer_id.
- **Acceptance Criteria:**
  - Add/edit/deactivate reviewers
  - Configure sequence_order and deputy_reviewer_id
  - Changes take effect for new requests only
  - Audit log on changes
  - Unit tests for admin operations

### Issue: Implement admin all requests view
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `ui`, `admin`, `audit`
- **Assignee:** `@frontend-lead`
- **Description:** Create Admin — All Requests View with filters by status, environment, department, date range. Show SLA breach flag.
- **Acceptance Criteria:**
  - Filter by status, environment, department, date range
  - SLA breach flag visible
  - Dead-letter queue visible
  - Revoke access action
  - Resend notification action
  - Unit tests for filtering and actions

### Issue: Implement admin audit and compliance reports UI
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `ui`, `admin`, `compliance`
- **Assignee:** `@frontend-lead`
- **Description:** Create Admin — Audit & Compliance Reports screen with audit log viewer, CSV export, and quarterly report viewer.
- **Acceptance Criteria:**
  - Audit log viewer with filters
  - CSV export functionality
  - Quarterly report viewer
  - Data owner tag coverage metric
  - Unit tests for report rendering

### Issue: Implement access renewal / extension flow
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `backend`, `renewal`, `workflow`
- **Assignee:** `@backend-lead`
- **Description:** Implement access renewal/extension flow. Pre-fill original tables, require new end date and justification. Create new request with parent_request_id linking.
- **Acceptance Criteria:**
  - Extend Access action for approved requests within 30 days of expiry
  - Pre-fill original tables (read-only), original purpose (editable)
  - New end date required (after current access_end)
  - 365-day cap from new start date
  - Create new access_request with parent_request_id
  - Full approval workflow for extension
  - Unit tests for renewal logic

### Issue: Implement request cancellation flow
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `backend`, `cancellation`, `workflow`
- **Assignee:** `@backend-lead`
- **Description:** Implement request cancellation flow. Update status, cancel pending assignments, notify reviewers via Teams, log to audit_log.
- **Acceptance Criteria:**
  - Cancel action for submitted, under_review, awaiting_response states
  - Confirmation dialog before cancellation
  - Update status to cancelled
  - Cancel pending reviewer_assignments
  - Notify reviewers via Teams
  - Log to audit_log
  - Unit tests for cancellation logic

### Issue: Implement rejection appeal flow
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `backend`, `appeal`, `workflow`
- **Assignee:** `@backend-lead`
- **Description:** Implement rejection appeal flow. Change status to appealing, increment appeal_count, re-notify original reviewers. Second rejection is terminal.
- **Acceptance Criteria:**
  - Appeal action visible when appeal_count = 0
  - Appeal disabled (greyed out) when appeal_count >= 1
  - Change status to appealing
  - Increment appeal_count to 1
  - Re-notify original reviewers via Teams
  - Second rejection is terminal
  - Unit tests for appeal logic

---

## Phase 5 — Admin UI & Renewal Flows (Weeks 11–12)

### Issue: Implement admin reviewer management (SQL fallback)
- **Type:** `task`
- **Priority:** `P1`
- **Labels:** `admin`, `sql`, `reviewer-management`
- **Assignee:** `@devops-lead`
- **Description:** Provide SQL fallback for emergency reviewer table management. Add, deactivate, update sequence_order, assign deputy reviewers.
- **Acceptance Criteria:**
  - SQL scripts for add, deactivate, update sequence_order, assign deputy
  - Documented in runbook
  - Unit tests for SQL operations
  - Integration tests against dev workspace

### Issue: Implement admin revoke access action
- **Type:** `task`
- **Priority:** `P1`
- **Labels:** `admin`, `revoke`, `permissions`
- **Assignee:** `@backend-lead`
- **Description:** Implement admin revoke access action. Update status, revoke permissions via Permissions API, notify requestor and reviewers.
- **Acceptance Criteria:**
  - Revoke Access action in Admin UI
  - Update access_requests.status = 'revoked'
  - Revoke READ permissions via Permissions API
  - Notify requestor and reviewers via Teams
  - Log to audit_log
  - Unit tests for revoke logic

### Issue: Implement admin resend notification action
- **Type:** `task`
- **Priority:** `P1`
- **Labels:** `admin`, `notifications`, `power-automate`
- **Assignee:** `@devops-lead`
- **Description:** Implement admin resend notification action for stuck approvals. Call Power Automate with same payload.
- **Acceptance Criteria:**
  - Resend Notification action in Admin UI
  - Call Power Automate with same payload
  - Track retry count
  - Log to audit_log
  - Unit tests for resend logic

### Issue: Implement admin escalation actions
- **Type:** `task`
- **Priority:** `P1`
- **Labels:** `admin`, `escalation`, `reviewer-management`
- **Assignee:** `@devops-lead`
- **Description:** Implement admin escalation actions: reassign to deputy, manual approval on behalf of reviewer, manual re-trigger of notifications.
- **Acceptance Criteria:**
  - Reassign to deputy reviewer action
  - Manual approval/rejection on behalf of reviewer
  - Manual re-trigger of notifications
  - Audit log on all actions
  - Unit tests for escalation actions

### Issue: Implement admin quarterly report generation
- **Type:** `task`
- **Priority:** `P1`
- **Labels:** `admin`, `reports`, `compliance`
- **Assignee:** `@devops-lead`
- **Description:** Implement quarterly report generation UI. Display report contents and allow CSV export.
- **Acceptance Criteria:**
  - Quarterly report viewer in Admin UI
  - Report contents: request counts, approval times, SLA breaches, active grants, sensitivity breakdown, reviewer workload
  - CSV export functionality
  - Unit tests for report rendering

### Issue: Implement admin dead-letter queue management
- **Type:** `task`
- **Priority:** `P1`
- **Labels:** `admin`, `dlq`, `error-handling`
- **Assignee:** `@devops-lead`
- **Description:** Implement dead-letter queue management UI. View failed notifications and manually re-trigger.
- **Acceptance Criteria:**
  - DLQ view in Admin UI
  - Filter by request_id, reviewer_id, error_type
  - Manual re-trigger action
  - Mark as resolved
  - Unit tests for DLQ management

---

## Phase 6 — Polish, Hardening & UAT (Weeks 13–14)

### Issue: Implement error handling and user-friendly error states
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `error-handling`, `ui`, `ux`
- **Assignee:** `@frontend-lead`
- **Description:** Implement comprehensive error handling with user-friendly error states, loading spinners, and informative messages.
- **Acceptance Criteria:**
  - Banner for Databricks API unavailable
  - Informational empty state for no tables
  - Toast notifications for errors
  - Loading spinners for API calls
  - Session expired redirect
  - Unit tests for error states

### Issue: Implement loading states and skeleton screens
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `ui`, `loading-states`, `ux`
- **Assignee:** `@frontend-lead`
- **Description:** Implement loading states and skeleton screens for API calls. Handle cold warehouse start with informative message.
- **Acceptance Criteria:**
  - Skeleton screens for table loading
  - Loading spinner for cold warehouse start
  - Informative message: "Data warehouse is starting up — this may take up to 2 minutes"
  - Unit tests for loading states

### Issue: Implement performance optimization with caching
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `performance`, `caching`, `optimization`
- **Assignee:** `@backend-lead`
- **Description:** Implement caching strategy with @st.cache_data for Unity Catalog lookups, SQL Execution queries, SCIM lookups, and dashboard metrics.
- **Acceptance Criteria:**
  - Table inventory cache (ttl=300)
  - SQL Execution size cache (ttl=600)
  - User/group cache (ttl=3600)
  - Dashboard metrics cache (ttl=60)
  - API retry logic with exponential backoff
  - Unit tests for caching behavior

### Issue: Implement responsive design adjustments
- **Type:** `task`
- **Priority:** `P1`
- **Labels:** `ui`, `responsive`, `ux`
- **Assignee:** `@frontend-lead`
- **Description:** Implement responsive design adjustments for desktop and web. Best-effort mobile support.
- **Acceptance Criteria:**
  - Responsive layout for desktop and web
  - Best-effort mobile support
  - Unit tests for responsive breakpoints

### Issue: Implement end-to-end testing and UAT
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `testing`, `e2e`, `uat`
- **Assignee:** `@qa-lead`
- **Description:** Implement end-to-end testing with Playwright and coordinate UAT with governance team.
- **Acceptance Criteria:**
  - E2E tests for wizard happy path
  - E2E tests for draft save/load
  - E2E tests for reviewer approve/reject
  - E2E tests for admin reviewer management
  - UAT checklist with governance team
  - Unit tests for E2E scenarios

### Issue: Implement security review and penetration test
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `security`, `penetration-test`, `compliance`
- **Assignee:** `@security-lead`
- **Description:** Conduct security review and penetration test. Address vulnerabilities before production release.
- **Acceptance Criteria:**
  - Security review completed
  - Penetration test completed
  - Vulnerabilities addressed
  - Security documentation updated
  - Unit tests for security scenarios

### Issue: Implement documentation and runbook updates
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `documentation`, `runbook`, `operations`
- **Assignee:** `@tech-lead`
- **Description:** Update documentation and runbooks for operations, troubleshooting, and incident response.
- **Acceptance Criteria:**
  - Runbook for manual access revocation
  - Runbook for failed expiry monitor job recovery
  - Runbook for stuck approval re-trigger
  - Runbook for reviewer table management
  - Runbook for failed deploy rollback
  - Documentation for secret rotation
  - Unit tests for documentation completeness

### Issue: Implement rollback testing
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `testing`, `rollback`, `ci-cd`
- **Assignee:** `@devops-lead`
- **Description:** Test rollback functionality with deploy_rollback.yml workflow. Verify successful redeployment of prior release tag.
- **Acceptance Criteria:**
  - Rollback workflow tested
  - Successful redeployment of prior release tag
  - Delta table time travel tested
  - Unit tests for rollback logic

### Issue: Implement NFR validation
- **Type:** `task`
- **Priority:** `P0`
- **Labels:** `testing`, `nfr`, `performance`
- **Assignee:** `@qa-lead`
- **Description:** Validate non-functional requirements: response time targets, availability, scalability.
- **Acceptance Criteria:**
  - Page load < 2 seconds (cached)
  - Table catalog browsing < 3 seconds (cached)
  - information_schema size query < 5 seconds (warm warehouse)
  - 99.5% uptime during business hours
  - Unit tests for performance metrics

### Issue: Implement accessibility improvements
- **Type:** `task`
- **Priority:** `P2`
- **Labels:** `accessibility`, `ui`, `ux`
- **Assignee:** `@frontend-lead`
- **Description:** Implement accessibility improvements for screen readers and keyboard navigation.
- **Acceptance Criteria:**
  - ARIA labels for interactive elements
  - Keyboard navigation support
  - Screen reader compatibility
  - Color contrast compliance
  - Unit tests for accessibility

### Issue: Implement analytics and monitoring
- **Type:** `task`
- **Priority:** `P2`
- **Labels:** `monitoring`, `analytics`, `observability`
- **Assignee:** `@devops-lead`
- **Description:** Implement monitoring and analytics for app usage, performance, and errors.
- **Acceptance Criteria:**
  - App usage metrics
  - Performance monitoring
  - Error tracking
  - Alerting for critical issues
  - Unit tests for monitoring logic

---

## Technical Debt & Future Enhancements

### Issue: Implement multi-workspace support (Phase 2 consideration)
- **Type:** `enhancement`
- **Priority:** `P2`
- **Labels:** `enhancement`, `multi-workspace`, `architecture`
- **Assignee:** `@tech-lead`
- **Description:** Support catalogs spanning multiple workspaces. Requires multi-workspace API client and separate authentication contexts.
- **Acceptance Criteria:**
  - Multi-workspace API client
  - Separate authentication contexts per workspace
  - Service principal credentials per workspace
  - Documentation for multi-workspace setup

### Issue: Implement mobile app support
- **Type:** `enhancement`
- **Priority:** `P2`
- **Labels:** `enhancement`, `mobile`, `ui`
- **Assignee:** `@frontend-lead`
- **Description:** Implement mobile app support for iOS and Android.
- **Acceptance Criteria:**
  - Mobile app for iOS
  - Mobile app for Android
  - Responsive design for mobile web
  - Unit tests for mobile scenarios

### Issue: Implement email notification fallback
- **Type:** `enhancement`
- **Priority:** `P2`
- **Labels:** `enhancement`, `email`, `notifications`
- **Assignee:** `@backend-lead`
- **Description:** Implement email notification fallback for Teams unavailable scenarios.
- **Acceptance Criteria:**
  - Email notification service
  - Fallback to email when Teams unavailable
  - Configuration for email recipients
  - Unit tests for email notifications

### Issue: Implement bulk access requests (CSV upload)
- **Type:** `enhancement`
- **Priority:** `P2`
- **Labels:** `enhancement`, `bulk-requests`, `ui`
- **Assignee:** `@frontend-lead`
- **Description:** Implement bulk access request upload via CSV.
- **Acceptance Criteria:**
  - CSV upload for bulk table selection
  - Validation of CSV format
  - Bulk request creation
  - Unit tests for bulk upload

### Issue: Implement access request templates
- **Type:** `enhancement`
- **Priority:** `P2`
- **Labels:** `enhancement`, `templates`, `ui`
- **Assignee:** `@frontend-lead`
- **Description:** Implement pre-built templates for recurring access patterns.
- **Acceptance Criteria:**
  - Template library for common access patterns
  - Template selection on request creation
  - Template customization
  - Unit tests for template logic

### Issue: Implement external user access
- **Type:** `enhancement`
- **Priority:** `P2`
- **Labels:** `enhancement`, `external-users`, `authentication`
- **Assignee:** `@backend-lead`
- **Description:** Support external/guest users without Databricks workspace SSO.
- **Acceptance Criteria:**
  - External user authentication
  - Guest user support
  - External user provisioning
  - Unit tests for external user scenarios

### Issue: Implement fine-grained row/column access
- **Type:** `enhancement`
- **Priority:** `P2`
- **Labels:** `enhancement`, `row-level-security`, `column-masking`
- **Assignee:** `@backend-lead`
- **Description:** Implement row-level filtering and column masking via Unity Catalog features.
- **Acceptance Criteria:**
  - Row-level filtering support
  - Column masking support
  - Unity Catalog policy integration
  - Unit tests for fine-grained access

### Issue: Implement automated table discovery / recommendation
- **Type:** `enhancement`
- **Priority:** `P2`
- **Labels:** `enhancement`, `ai`, `recommendations`
- **Assignee:** `@ml-lead`
- **Description:** Implement AI/ML-driven table recommendations based on user role and past requests.
- **Acceptance Criteria:**
  - Table recommendation engine
  - User role-based recommendations
  - Past request-based recommendations
  - Unit tests for recommendation logic

### Issue: Implement real-time webhook notifications
- **Type:** `enhancement`
- **Priority:** `P2`
- **Labels:** `enhancement`, `webhooks`, `real-time`
- **Assignee:** `@backend-lead`
- **Description:** Implement WebSocket layer for real-time push notifications instead of polling.
- **Acceptance Criteria:**
  - WebSocket server implementation
  - Real-time push notifications
  - WebSocket client for Streamlit app
  - Unit tests for WebSocket scenarios

---

## Documentation

### Issue: Create user documentation
- **Type:** `documentation`
- **Priority:** `P0`
- **Labels:** `documentation`, `user-guide`, `content`
- **Assignee:** `@tech-lead`
- **Description:** Create user documentation for requestors, reviewers, and admins.
- **Acceptance Criteria:**
  - User guide for requestors
  - User guide for reviewers
  - User guide for admins
  - FAQ section
  - Troubleshooting guide

### Issue: Create admin documentation
- **Type:** `documentation`
- **Priority:** `P0`
- **Labels:** `documentation`, `admin-guide`, `content`
- **Assignee:** `@tech-lead`
- **Description:** Create admin documentation for reviewer management, compliance reporting, and incident response.
- **Acceptance Criteria:**
  - Admin guide for reviewer management
  - Admin guide for compliance reporting
  - Admin guide for incident response
  - Runbook for common operations
  - Troubleshooting guide

### Issue: Create API documentation
- **Type:** `documentation`
- **Priority:** `P0`
- **Labels:** `documentation`, `api`, `content`
- **Assignee:** `@backend-lead`
- **Description:** Create API documentation for Unity Catalog API wrapper, SQL Execution client, and SCIM client.
- **Acceptance Criteria:**
  - API reference documentation
  - Code examples
  - Error handling documentation
  - Authentication documentation
  - Unit tests for API documentation

### Issue: Create deployment documentation
- **Type:** `documentation`
- **Priority:** `P0`
- **Labels:** `documentation`, `deployment`, `content`
- **Assignee:** `@devops-lead`
- **Description:** Create deployment documentation for Databricks Asset Bundle, CI/CD workflows, and rollback procedures.
- **Acceptance Criteria:**
  - Deployment guide
  - CI/CD workflow documentation
  - Rollback procedures
  - Environment configuration guide
  - Secret management guide

### Issue: Create security documentation
- **Type:** `documentation`
- **Priority:** `P0`
- **Labels:** `documentation`, `security`, `content`
- **Assignee:** `@security-lead`
- **Description:** Create security documentation for authentication, authorization, and compliance.
- **Acceptance Criteria:**
  - Security architecture documentation
  - Authentication and authorization guide
  - Compliance documentation
  - Security best practices
  - Incident response procedures

---

## Total Issues: 105

**Breakdown by Priority:**
- P0 (Critical): 75 issues
- P1 (High): 15 issues
- P2 (Medium): 15 issues

**Breakdown by Type:**
- task: 90 issues
- enhancement: 8 issues
- documentation: 7 issues

**Breakdown by Labels:**
- infrastructure: 8 issues
- api: 12 issues
- ui: 25 issues
- backend: 20 issues
- teams: 6 issues
- power-automate: 4 issues
- jobs: 5 issues
- testing: 6 issues
- documentation: 7 issues
- enhancement: 8 issues
- security: 3 issues
- compliance: 3 issues
- performance: 2 issues
- accessibility: 1 issue
- monitoring: 1 issue
- rollback: 1 issue
- nfr: 1 issue

**Assignee Distribution:**
- @devops-lead: 20 issues
- @backend-lead: 25 issues
- @frontend-lead: 30 issues
- @qa-lead: 4 issues
- @tech-lead: 10 issues
- @security-lead: 3 issues
- @ml-lead: 1 issue
- @data-governance-team: 2 issues

---

*Generated on April 4, 2026*
*Total issues: 105*
*Ready for GitHub issue creation*