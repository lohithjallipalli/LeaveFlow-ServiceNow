# LeaveFlow - Leave Management on ServiceNow (Brazil release)

## Problem and scope

LeaveFlow is a Leave Management application built on the ServiceNow Brazil (PDI) release, designed to replace basic form-based leave requests with a modern, responsive, card-driven portal. The project addresses the need for employee self-service leave submission, manager approval workflows, team visibility into team leave, and real-time balance tracking — all while aligning with ServiceNow's strategic Next Experience (UI Builder) framework. The goal is to deliver a professional, accessible, and mobile-first user experience that stands out from standard ServiceNow implementations while maintaining enterprise standards and compliance.

## Features by role

### Employee
* **Self-service leave portal** — Home page with personalized hero banner, 4 KPI tiles showing Annual/Sick/Casual/Compensatory leave balances, and quick action buttons to Apply for Leave or View Request History.
* **Smart leave request form** — Live validation of start/end dates, automatic total-days calculation, minimum 10-character reason counter, and real-time balance deduction meter.
* **Leave balance tracking** — Dynamic display of available/pent/used days calculated via Scripted Data Resources querying `x_snc_leaveflow_leave_request`.
* **Real-time notifications** — Bell icon with polling-based notifications for request status changes (approved/rejected).
* **Holiday integration** — Upcoming public holidays fetched via Nager.Date REST API (no API key required).
* **Export functionality** — One-click CSV/PDF export of leave history from the My Requests page.
* **Mobile-responsive design** — All pages auto-adapt from desktop (12-column grid) to tablet (768px–1024px) to mobile (<768px) without separate themes.

### Manager
* **Approval dashboard** — KPI tiles showing Pending Approvals count, Team Availability %, Requests This Month, and Average Response Time.
* **Pending approvals queue** — Data table of submitted leave requests with one-click Approve/Reject buttons.
* **Rejection modal** — Required reason field, calls Script Include `ManagerLeaveActions.rejectRequest()`, refreshes data after action.
* **Team leave calendar** — Monthly grid view (7-column) with color-coded cell counts (green/orange/red) showing how many team members are on leave each day; toggleable List view.
* **Team member details** — Click any calendar day to view employee names and leave types on leave that day.
* **Real-time updates** — Manager and employee browsers sync within 30 seconds of status changes via polling-based data resources.

### Admin
* **Full access** — All portal pages, configuration, and user management.
* **Role containment** — Hierarchy: `x_snc_leaveflow.admin` > `x_snc_leaveflow.manager` > `x_snc_leaveflow.employee`.
* **Script Include management** — `ManagerLeaveActions` (approveRequest/rejectRequest), `LeaveRequestUtils` (validateRange/hasOverlap), `HolidayService` (getHolidays via Nager.Date), `LeaveExporter` (CSV generation).
* **Test user seeding** — Pre-configured employee and manager accounts for test execution (T01–T40).

## Architecture diagram + data model (tables, key fields, relationships)

### Core table: `x_snc_leaveflow_leave_request`
| Field | Type | Description |
|-------|------|-------------|
| `number` | String | Auto-generated record identifier |
| `requested_by` | Reference | Employee initiating the request |
| `approved_by` | Reference | Manager who approved/rejected |
| `leave_type` | Choice | Annual, Sick, Casual, Compensatory |
| `start_date` | Date | Leave start (inclusive) |
| `end_date` | Date | Leave end (inclusive) |
| `total_days` | Integer | Calculated duration (working days + 1) |
| `status` | Choice | Draft, Submitted, Approved, Rejected |
| `reason` | String | Free-text reason for leave |
| `approved_date` | DateTime | Timestamp of approval action |
| `approver_comments` | String | Manager's rejection/approval reason |

### Supporting tables
* `x_snc_leaveflow_leave_type` — Choice configuration (days_allocated per type).
* `sp_user_preference` — Optional user layout/theme preferences (Service Portal legacy).
* Nager.Date public holiday feed (external REST, no authentication).

### Key relationships
* **requested_by → user** — Lookup to `sys_user` for employee display name/email.
* **approved_by → user** — Lookup to `sys_user` for manager display name.
* **leave_type → leave_type [choice]** — Validates against permitted leave types and allocated days.
* **Scripted Data Resources** — Server-side GlideRecord queries transform to JSON for UI Builder component binding (`@datresource.method`).

### Data flow
1. Employee submits leave request → UI Builder form → `createleaverecord` data resource → GlideRecord insert → Status defaults to `Submitted` → Trigger notifies manager.
2. Manager views dashboard → `getpendingapprovals` data resource → Scripted Table Query (`status=submitted^approver=gs.getUserID()`) → `now-record-list` component renders table.
3. Manager approves request → Client click → `ManagerLeaveActions.approveRequest(requestSysId)` → GlideRecord update (`status=approved, approved_date=now, approver_comments`) → Data resource refresh → Employee receives bell notification.

## Scripts and flows (one paragraph each: purpose, inputs, outputs)

### `ManagerLeaveActions` (Script Include, Client callable: YES)
* **Purpose**: Centralized approve/reject logic for leave requests with role verification.
* **Inputs**: `requestSysId` (String), `approverComment` / `rejectionReason` (String).
* **Outputs**: `{success: Boolean, message: String}` — returns error if user is not the approved, request not found, or reason is missing on rejection.

### `LeaveRequestUtils` (Script Include, Client callable: NO)
* **Purpose**: Reusable utility functions for date validation and overlap checking.
* **Inputs**: `startDate`, `endDate` (Date objects), `durationType` ('full_day' | 'half_day').
* **Outputs**: `totalDays` (Integer) — returns 0 if end_date < start_date; for half_day returns raw day diff.

### `HolidayService` (Script Include, Client callable: NO)
* **Purpose**: Fetch public holidays via Nager.Date REST API for display in portal hero/sidebar.
* **Inputs**: Optional country code (defaults to 'us'), year (defaults to current).
* **Outputs**: Array of `{date: 'YYYY-MM-DD', name: 'Holiday Name'}` — embeddable in KPI tiles or hero banner.

### `LeaveExporter` (Script Include, Client callable: YES via UI Action)
* **Purpose**: Generate export file (CSV) of leave history for a specific user.
* **Inputs**: `userId` (String).
* **Outputs**: CSV string with headers `Number, Leave Type, Start Date, End Date, Total Days, Status, Submitted On`.

### Client-side form flow (Apply Leave page)
* **Date change listeners** (`startDate$.subscribe`, `endDate$.subscribe`, `durationType$.subscribe`) → recalculate `totalDays` via `calculateTotalDays()` → update read-only Total Days field.
* **Submit button** → execute `Create Record` data resource → on success show toast + navigate to home → on error show error toast.
* **Cancel button** → navigate to home without saving.

### Real-time notification polling (Bell icon)
* **Data resource** `getnotifications` → Scripted script queries `x_snc_leaveflow_leave_request` for user's new approvals/rejections within last 30 seconds → returns `{count, notifications: [...]}` → badge updates every 30 seconds → clicking opens dropdown with list of changes.

## Security design (role containment, ACL table, field ACLs, query rule)

### Role containment hierarchy
`x_snc_leaveflow.admin` > `x_snc_leaveflow.manager` > `x_snc_leaveflow.employee`

* **admin**: Full access to all portal pages, configuration, user impersonation, and data export.
* **manager**: Access to manager-dashboard and apply-leave pages; can approve/reject requests assigned to them; cannot view other managers' queues.
* **employee**: Access to home, apply-leave, my-requests pages; can submit and view own requests; cannot access manager dashboard or other employees' records.

### Access Control Lists (ACLs)
* **Table-level ACL** on `x_snc_leaveflow_leave_request`:
  * **Read**: Role `x_snc_leaveflow.employee` with condition `requested_by = current user`; Role `x_snc_leaveflow.manager` with condition `approver = current user`; Role `x_snc_leaveflow.admin` unrestricted.
  * **Write**: Role `x_snc_leaveflow.manager` and `x_snc_leaveflow.admin` only; condition `status IN ('draft', 'submitted')` blocks edits after submission.
* **Field-level ACLs**:
  * `approved_by` — Read: manager/admin; Write: manager/admin on approved/rejected records.
  * `approved_date` — Write: system-only on status transition.
  * `approver_comments` — Write: manager/admin on rejection/approval.
* **Query Business Rule** on `x_snc_leaveflow_leave_request`:
  * Active: YES
  * Condition: `WHEN status = 'submitted' AND current.user.hasRole('x_snc_leaveflow.employee') THEN SET work_on = TRUE ELSE SET work_on = FALSE` — ensures employees only see their own records in list views.

### UI Builder page-level permissions
* Each variant has **Audiences** field restricting visibility by role.
* Dashboard widgets inherit portal role unless explicitly configured otherwise.
* Script Includes marked `Client callable: YES` only for actions explicitly intended for client execution (approve/cancel buttons, export).

### Negative test evidence (screenshots S1–S10)
* **(S1)** Form submission with empty reason → error toast.
* **(S2)** End date before start date → totalDays resets to 0.
* **(S3)** Employee attempting to access manager-dashboard → access denied.
* **(S4)** Manager approving another manager's request → error "You are not authorized".
* **(S5)** Submitting leave request overlapping approved period → warning toast "Overlap detected".
* **(S6)** Rejecting request without reason → error "Rejection reason is required".
* **(S7)** User without employee role viewing home page → redirected to login.
* **(S8)** Mobile viewport (375px) KPI tiles stack vertically.
* **(S9)** Past-end-date start selection → disabled submit button.
* **(S10)** CSV export of empty request list → file with headers only.

## Integration (Nager.Date: endpoint, failure behaviour, no secrets)

### Nager.Date public holiday integration
* **Endpoint**: `https://nager.date-api.azure.net/v3/PublicHolidays/{year}/{countryCode}`
* **Country code**: `us` (United States) — configurable via Script Include parameter `holidayCountryCode`.
* **No API key required**: Azure Function-backed endpoint is open to public internet without authentication.
* **Failure behaviour**: Script Include `HolidayService.getHolidays()` wraps fetch in `try / catch`; on network error or non-200 response returns empty array `[]` — portal continues to function without holidays displayed (no fallback mock data).
* **Caching**: Results not cached between requests (fresh fetch per portal load) — acceptable volume for < 1000 daily portal users; future enhancement: implement `sp_cache` or instance metadata cache.
* **Rate limits**: Azure Functions default limits allow > 1M monthly calls; well within scope for project usage.
* **Data mapped to UI**: Each holiday `{date, name}` rendered in small sidebar card on Home page; date formatted to `MMM dd, yyyy` (e.g., `Nov 28, 2026`).

## Setup (PDI request, import steps, seed Leave Types, create test users)

### Prerequisites
* ServiceNow Brazil (2024) or later PDI instance.
* UI Builder plugin `com.glide.ui_builder` activated (System Definition → Plugins).
* Nager.Date API access (no credentials needed — public endpoint).

### Update set import
1. Navigate to **`System Update Sets`** → select or create update set `LeaveFlow`.
2. **Import** the `.par` file committed to GitHub repository `LeaveFlow-ServiceNow`.
3. Verify imported components appear in Application Navigator:
   * **`LeaveFlow`** application module.
   * **`x_snc_leaveflow_leave_request`** table.
   * **Script Includes**: `ManagerLeaveActions`, `LeaveRequestUtils`, `HolidayService`, `LeaveExporter`.
   * **Choice list**: `x_snc_leaveflow_leave_type` (Annual, Sick, Casual, Compensatory).
   * **UI Builder Experience**: `LeaveFlow Portal` (url path: `leaveflow`).
   * **Dashboards**: `LeaveFlow Management Dashboard`.

### Seed Leave Types
1. Navigate to **`Lists` → `Choice`** → filter `Table` = `x_snc_leaveflow_leave_request`, `Field` = `leave_type`.
2. Add choice entries:
   * `Annual` — `days_allocated`: 20 (or org policy).
   * `Sick` — `days_allocated`: 10.
   * `Casual` — `days_allocated`: 5.
   * `Compensatory` — `days_allocated`: 3.
3. Click **Save**.

### Create test users
1. **Employee** (`x_snc_leaveflow.employee` role):
   * User record: `first_name` = `Test`, `last_name` = `Employee`, `email` = `test.employee@test.com`.
   * Ensure `user` has `x_snc_leaveflow.employee` role.
2. **Manager** (`x_snc_leaveflow.manager` role):
   * User record: `first_name` = `Test`, `last_name` = `Manager`, `email` = `test.manager@test.com`.
   * Ensure `user` has `x_snc_leaveflow.manager` role AND `x_snc_leaveflow.employee` role (inheritance).
3. **HR Admin** (`x_snc_leaveflow.hr_admin` role):
   * User record: `first_name` = `Test`, `last_name` = `HR`, `email` = `test.hr@test.com`.
   * Ensure `user` has all three roles: `admin`, `manager`, `employee`.

### Verify UI Builder experience
1. Navigate to `https://<instance>.service-now.com/now/leaveflow/home`.
2. Confirm home page loads with hero banner, 4 KPI cards, and quick action buttons.
3. Test `apply-leave` route: submit form with valid dates → verify toast + redirect.
4. Test `manager-dashboard` as manager: KPI tiles populate, pending queue renders.

## Test results (table T01–T40 with real Pass/Fail)

| Test ID | Title | User Role | Steps | Expected Result | Actual Result | Pass/Fail |
|---------|-------|-----------|-------|----------------|---------------|----------|
| T01 | Employee portal access | Employee | 1. Navigate to `/leaveflow-portal/home` 2. Verify hero banner displays name 3. Verify KPI tiles load | Hero banner shows `@session.userdisplayname`; 4 KPI tiles display (may show 0 if no data) | — | — |
| T02 | Leave application flow | Employee | 1. Click Apply for Leave 2. Select leave type 3. Enter start/end dates 4. Verify total days calculates 5. Enter reason (≥10 chars) 6. Click Submit | Success toast appears; form clears; redirect to home; new request appears in Recent Requests table | — | — |
| T03 | Date validation (end before start) | Employee | 1. Set end_date before start_date 2. Observe totalDays field | totalDays displays 0; submit button may be disabled | — | — |
| T04 | Minimum reason length validation | Employee | 1. Enter 9-character reason 2. Attempt submit | Error toast: "Reason must be at least 10 characters." | — | — |
| T05 | Manager dashboard access | Manager | 1. Impersonate manager 2. Navigate to `/leaveflow-portal/manager-dashboard` | Dashboard loads; KPI tiles show team data; pending approvals table visible | — | — |
| T06 | Approve leave request | Manager | 1. In pending queue, click Approve on a request 2. Verify success | Success toast: "Leave request approved successfully"; status changes to Approved; data refreshes | — | — |
| T07 | Reject leave request with reason | Manager | 1. Click Reject on a request 2. Enter reason in modal 3. Confirm | Success toast: "Leave request rejected"; status changes to Rejected; modal closes | — | — |
| T08 | Reject without reason | Manager | 1. Click Reject 2. Leave reason field empty 3. Confirm | Error toast: "Rejection reason is required." | — | — |
| T09 | Mobile responsiveness (home) | Employee | 1. Resize browser to 375px or open on mobile 2. Navigate all pages | KPI tiles stack 1-column; hero banner reduces padding; buttons become full-width; tables scroll horizontally | — | — |
| T10 | Real-time notification update | Employee + Manager (separate browsers) | 1. Employee submits request 2. Manager refreshes within 30 seconds 3. Notification badge updates | Manager sees bell badge increment; clicking shows "New leave request submitted"; employee sees approval notification | — | — |
| T11 | Holiday display on home page | Employee | 1. Load home page 2. Observe sidebar holidays card | Upcoming 3 public holidays displayed with date and name (e.g., "Thanksgiving - Nov 28, 2026") | — | — |
| T12 | CSV export from My Requests | Employee | 1. Navigate to `/leaveflow-portal/my-requests` 2. Click Export to PDF/CSV button | Download begins; file contains headers `Number, Leave Type, Start Date, End Date, Total Days, Status, Submitted On` + rows for each request | — | — |
| T13 | Leave type choice validation | Employee | 1. In apply form, select leave type 2. Confirm valid choice | Dropdown only permits: Annual, Sick, Casual, Compensatory (enforced by choice list). | — | — |
| T14 | Balance calculation correctness | Employee | 1. Employee with 20 allocated annual days 2. Submit 5-day request 3. View balances | Available Days KPI displays 15 (20 - 5); Used Days KPI displays 5. | — | — |
| T15 | Overlap detection (new request overlaps approved) | Employee | 1. Have approved request for Nov 1–5 2. Attempt new request for Nov 3–7 3. Attempt submit | Warning toast: "Overlap with existing approved request (Nov 3–5). Please adjust dates." | — | — |
| T16 | Employee cannot view other employees' requests | Employee | 1. Log in as employee 2. Attempt to view another employee's my-requests (impersonate) | System denies access or filters to current user only (ACL condition `requested_by = current user`). | — | — |
| T17 | Manager can view team requests only | Manager | 1. Log in as manager 2. View pending approvals queue | Table only shows requests where `approver = current user`; does not show requests assigned to other managers. | — | — |
| T18 | Admin full access | Admin | 1. Log in as hr_admin 2. Attempt to access all pages, edit any record, export all data | All pages load; inline edit available; export generates file with all records (no user filtering). | — | — |
| T19 | Scripted Data Resource error handling | Employee | 1. Corrupt a data resource script (introduce syntax error) 2. Load dependent page | Page displays "Loading..." indefinitely or error message; console shows JavaScript error—does not crash entire portal. | — | — |
| T20 | Form cancellation navigation | Employee | 1. In apply-leave page, click Cancel 2. Observe navigation | User navigates back to home page without record creation. | — | — |
| T21 | KPI tile color correctness | Employee | 1. Verify color coding of KPI tiles on home page | Available Days: success green (`#00C853`); Pending Requests: warning orange (`#FFB300`); Used Days: info blue (`#00B8D4`). | — | — |
| T22 | Donut chart status distribution | Any | 1. View LeaveFlow - Requests by Status report 2. Observe slices | Four slices: submitted, approved, rejected, draft—colors match status color palette; legend maps labels to colors. | — | — |
| T23 | Monthly trends chart accuracy | Any | 1. View LeaveFlow - Monthly Leave Trends report 2. Compare to actual request dates | Bar heights correspond to total_days per month; X-axis shows month names (Jan–Dec). | — | — |
| T24 | Choice list seed persistence | New instance | 1. Fresh Brazil PDI instance 2. Import update set 3. Verify leave_type choices seeded | Four choices (Annual/Sick/Casual/Compensatory) present with correct `days_allocated` values. | — | — |
| T25 | User role inheritance | Manager | 1. Create user with `x_snc_leaveflow.manager` role only 2. Test employee-level access | User retains manager-specific access (dashboard, approvals) but also inherits employee-level portal access (apply-leave, my-requests). | — | — |
| T26 | ACL blocks unauthorized field edit | Employee | 1. View leave request detail as employee 2. Attempt to edit `approved_by` field | Field is read-only; inline edit control hidden or disabled. | — | — |
| T27 | Query rule filters employee records | Employee | 1. Log in as employee 2. Run custom query for `SELECT * FROM leave_request` | System automatically adds condition `requested_by = current user`; returns only employee's records. | — | — |
| T28 | Past date disable on form | Employee | 1. Set start_date to a past date 2. Attempt submit | Submit button disabled; error toast: "Start date cannot be in the past." | — | — |
| T29 | Half-day duration calculation | Employee | 1. In apply form, select duration type = Half Day 2. Set dates spanning 3 calendar days 3. Observe totalDays | totalDays displays 3 (raw day diff); note: business-day logic may vary by company policy. | — | — |
| T30 | Bell badge sync between browsers | Employee + Manager | 1. Employee submits request 2. Manager refreshes page 3. Badge updates | Manager bell badge increments from 0 to 1 within 30 seconds; clicking reveals notification detail. | — | — |
| T31 | Export empty request list | Employee | 1. Employee with no requests 2. Click Export CSV 3. Observe output | CSV file downloads with headers only: `Number, Leave Type, Start Date, End Date, Total Days, Status, Submitted On`. | — | — |
| T32 | Data resource test in UI Builder | Developer | 1. Open data resource `getemployeeleave_balance` 2. Click Test 3. Observe output | Returns JSON `{available_days: N, pending_requests: M, used_days: K, total_allocation: 20}` or error message in logs. | — | — |
| T33 | UI Builder variant role restriction | Employee/Manager | 1. View home page as employee 2. Impersonate manager 3. Refresh | Home page either hides manager-specific KPIs or displays different variant (configured via Audiences). | — | — |
| T34 | Page load performance | Any | 1. Measure time from URL navigation to fully rendered home page 2. Observe time | Page loads within 3 seconds on standard broadband; subsequent loads cached by browser. | — | — |
| T35 | Responsive column layout (manager dashboard) | Manager | 1. Resize dashboard viewport from 1440px to 375px 2. Observe KPI grid | 4-column desktop grid → 2-column tablet → 1-column mobile; no overflow or horizontal scrolling of content. | — | — |
| T36 | Calendar color legend accuracy | Manager | 1. View team calendar 2. Hover over day with 2 employees on leave 3. Observe color | Cell background orange (3-4 people on leave threshold); legend below calendar confirms: Yellow = 1-2, Orange = 3-4. | — | — |
| T37 | Nager.Date holiday fetch reliability | Employee | 1. Load home page multiple times 2. Observe holiday card consistency | Holidays display consistently (or absent if API unreachable); never shows partial/error data. | — | — |
| T38 | Export includes all required columns | Admin | 1. Generate CSV with multiple requests 2. Open in Excel/Sheets 3. Verify column count | CSV has exactly 7 columns matching headers; no extra/missing fields. | — | — |
| T39 | Script include client-callable flag | Developer | 1. Navigate to Script Include `ManagerLeaveActions` 2. Verify `client_callable = true` on approved actions only | Only `approveRequest` and `rejectRequest` marked client-callable; other utilities (`LeaveRequestUtils`) marked `false`. | — | — |
| T40 | Final sign-off checklist | Project Lead | 1. Review all T01–T39 results 2. Confirm zero critical blockers 3. Approve release | All tests Pass or Fail with documented workaround; project ready for production deployment on Brazil PDI. | — | — |

## Known limitations and future work (Phase 2 list)

### Current limitations
* **Nager.Date API dependency** — Public endpoint with no authentication; if Azure Functions region goes offline, holidays card disappears until restored.
* **No offline support** — UI Builder portal requires active internet connection; no local data persistence or cached fallback.
* **Limited user preference persistence** — Dashboard layouts, filter selections, and KPI preferences are not saved between sessions (unlike Service Portal's `sp_user_preference` table).
* **Calendar color thresholds hardcoded** — Legend categories (Yellow: 1–2, Orange: 3–4, Red: 5+) are fixed; organizations with different team sizes may want configurable thresholds.
* **CSV export lacks recurrence** — One-time leave requests only; recurring leave (e.g., every Friday) exports as individual records, not as pattern.
* **Mobile form factor** — Half-day duration logic uses raw day difference, not business-day calendar; organizations with complex shift schedules may need custom calculation.

### Phase 2 — Future enhancements
* **Dark mode support** — Add `now-color` CSS variable overrides for light/dark theme switching; update color palette for contrast compliance.
* **Calendar recurrence support** — Enable "Repeat" selector (every Monday, every Friday, custom pattern) in leave request form; store recurrence rule in `reason` or new `repeat_rule` field.
* **Slack/Teams notifications** — Replace polling-based bell notifications with real-time webhook integrations; post approval/rejection events to configured channels.
* **Google Calendar / Outlook sync** — Add export feature that generates .ics files for user's personal calendar; map `start_date`/`end_date`/ `reason` to calendar event details.
* **Advanced analytics** — Phase 2 dashboard: trend visualizations (leave accrual over year), team utilization percentages, prediction models for upcoming absences based on historical patterns.
* **Delegation during manager absence** — Allow employee to designate a substitute manager; approve/reject actions routed to stand-in with audit trail.
* **KB article integration** — Link leave type choices to internal knowledge base articles (e.g., "Sick leave policy," "Bereavement procedures") within the request form.
* **API endpoint for external systems** — Public REST API (`/api/x_snc_leaveflow/leave_request`) enabling HRIS integration, payroll sync, or third-party absence management tools.
* **Accessibility WCAG AA enhancements** — Focus management, ARIA labels on dynamic components, keyboard navigation order optimization, color contrast ratio verification across all components.