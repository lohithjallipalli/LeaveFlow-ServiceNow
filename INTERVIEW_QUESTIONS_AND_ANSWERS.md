# 📋 LeaveFlow - Interview Questions & Answers

A comprehensive list of interview questions and answers about the LeaveFlow ServiceNow project, covering all aspects from project overview to technical details.

---

## 📖 Section 1: Project Overview

### Q1: What is LeaveFlow and what problem does it solve?
**A:** LeaveFlow is a modern leave management portal built on ServiceNow Brazil (PDI) release that replaces basic form-based leave requests with a professional, responsive, card-driven portal. It addresses the need for employee self-service leave submission, manager approval workflows, team visibility into team leave, and real-time balance tracking — all while aligning with ServiceNow's strategic Next Experience (UI Builder) framework. The goal is to deliver a professional, accessible, and mobile-first user experience that stands out from standard ServiceNow implementations while maintaining enterprise standards and compliance.

### Q2: What release of ServiceNow is LeaveFlow built on?
**A:** LeaveFlow is built on the ServiceNow Brazil (PDI) release. UI Builder was introduced/finalized in the Brazil release, making it the strategic framework for new portal developments.

### Q3: Who are the primary users of LeaveFlow?
**A:** The primary users are:
- **Employees**: Submit leave requests, view balances, see request history
- **Managers**: Approve/reject requests, view team calendars, track team availability
- **HR Admins**: Full configuration, user management, reporting, and data export

---

## ✨ Section 2: Features by Role

### Q4: What are the key features for Employees?
**A:** Employees get:
- Self-service leave portal with personalized hero banner
- 4 KPI tiles showing Annual/Sick/Casual/Compensatory leave balances
- Smart leave request form with live date validation and automatic total-days calculation
- Minimum 10-character reason counter validation
- Real-time balance deduction meter
- Real-time notifications via bell icon (polling-based)
- Holiday integration showing upcoming public holidays via Nager.Date REST API
- One-click CSV/PDF export of leave history
- Fully mobile-responsive design adapting from desktop to mobile

### Q5: What features are available for Managers?
**A:** Managers have access to:
- KPI tiles showing Pending Approvals count, Team Availability %, Requests This Month, and Average Response Time
- Pending approvals queue with data table of submitted requests
- One-click Approve/Reject buttons with required reason field for rejections
- Team leave calendar view (monthly grid with color-coded cell counts)
- Toggleable List view for the calendar
- Click-to-view employee names and leave types on leave each day
- Real-time browser sync with employee browsers within 30 seconds
- Script Include `ManagerLeaveActions` for approve/reject logic

### Q6: What capabilities does the Admin role have?
**A:** Admins receive:
- Full access to all portal pages, configuration, and user management
- Role containment hierarchy: `x_snc_leaveflow.admin` > `x_snc_leaveflow.manager` > `x_snc_leaveflow.employee`
- Script Include management: `ManagerLeaveActions`, `LeaveRequestUtils`, `HolidayService`, `LeaveExporter`
- Choice list seeding: `x_snc_leaveflow_leave_type` (Annual, Sick, Casual, Compensatory with allocated days)
- Test user seeding for test execution (T01–T40)
- Full data export capabilities (no user filtering)
- Inline edit access for any record

---

## 🏗️ Section 3: Architecture and Data Model

### Q7: Describe the core table structure for LeaveFlow.
**A:** The core table is `x_snc_leaveflow_leave_request` with these key fields:
- `number` (String) — Auto-generated record identifier
- `requested_by` (Reference) — Employee initiating the request
- `approved_by` (Reference) — Manager who approved/rejected
- `leave_type` (Choice) — Annual, Sick, Casual, Compensatory
- `start_date` (Date) — Leave start (inclusive)
- `end_date` (Date) — Leave end (inclusive)
- `total_days` (Integer) — Auto-calculated duration (working days + 1)
- `status` (Choice) — Draft, Submitted, Approved, Rejected
- `reason` (String) — Free-text reason for leave
- `approved_date` (DateTime) — Timestamp of approval action
- `approver_comments` (String) — Manager's rejection/approval reason

### Q8: What supporting tables does LeaveFlow use?
**A:** Supporting tables include:
- `x_snc_leaveflow_leave_type` — Choice configuration with `days_allocated` per type (Annual: 20, Sick: 10, Casual: 5, Compensatory: 3)
- `sp_user_preference` — Optional user layout/theme preferences (Service Portal legacy)
- Nager.Date public holiday feed (external REST, no authentication)

### Q9: How are the key relationships defined in the data model?
**A:** Key relationships:
- `requested_by → user` — Lookup to `sys_user` for employee display name/email
- `approved_by → user` — Lookup to `sys_user` for manager display name
- `leave_type → leave_type [choice]` — Validates against permitted leave types and allocated days
- Scripted Data Resources — Server-side GlideRecord queries transform to JSON for UI Builder component binding (`@datresource.method`)

### Q10: Explain the data flow for a leave request submission.
**A:** The data flow is:
1. Employee submits leave request → UI Builder form → `createleaverecord` data resource → GlideRecord insert → Status defaults to `Submitted` → Trigger notifies manager
2. Manager views dashboard → `getpendingapprovals` data resource → Scripted Table Query (`status=submitted^approver=gs.getUserID()`) → `now-record-list` component renders table
3. Manager approves request → Client click → `ManagerLeaveActions.approveRequest(requestSysId)` → GlideRecord update (`status=approved, approved_date=now, approver_comments`) → Data resource refresh → Employee receives bell notification

---

## ⚙️ Section 4: Scripts and Automations

### Q11: What is the `ManagerLeaveActions` Script Include and how does it work?
**A:** `ManagerLeaveActions` is a Client callable Script Include with methods:
- `approveRequest(requestSysId, approverComment)` — Verifies approver matches `gs.getUserID()`, updates status to `approved`, sets `approved_date` and `approver_comments`, returns `{success: Boolean, message: String}`
- `rejectRequest(requestSysId, rejectionReason)` — Validates reason is not empty, verifies approver, updates status to `rejected`, sets `approved_date` and `approver_comments`, returns success/error response

### Q12: Describe `LeaveRequestUtils` and its purpose.
**A:** `LeaveRequestUtils` is a Server-only Script Include (not client callable) containing reusable utility functions:
- `calculateTotalDays(startDate, endDate, durationType)` — Returns total working days; returns 0 if end_date < start_date; for half_day returns raw day diff
- `hasOverlap(newStart, newEnd, existingRequests)` — Checks if new request overlaps with any approved existing requests
- These utilities are called from client scripts when date fields change to recalculate `totalDays`

### Q13: Explain the `HolidayService` Script Include.
**A:** `HolidayService` fetches public holidays via Nager.Date REST API:
- Endpoint: `https://nager.date-api.azure.net/v3/PublicHolidays/{year}/{countryCode}`
- Country code: `us` (configurable via Script Include parameter `holidayCountryCode`)
- No API key required — Azure Function-backed endpoint is publicly accessible
- Wraps fetch in try/catch; on error returns empty array `[]` — portal continues without holidays displayed
- Returns array of `{date: 'YYYY-MM-DD', name: 'Holiday Name'}`

### Q14: What does the `LeaveExporter` Script Include do?
**A:** `LeaveExporter` generates CSV export of leave history:
- Input: `userId` (String)
- Output: CSV string with headers `Number, Leave Type, Start Date, End Date, Total Days, Status, Submitted On`
- Queries `x_snc_leaveflow_leave_request` where `requested_by = userId`, orders by `syscreatedon` descending
- Marked `client_callable: YES` for UI Action execution

### Q15: How does client-side form validation work in the Apply Leave page?
**A:** Client-side validation includes:
- Date change listeners (`startDate$.subscribe`, `endDate$.subscribe`, `durationType$.subscribe`) → recalculate `totalDays` via `calculateTotalDays()` → update read-only Total Days field
- Minimum 10-character reason counter — error toast if less than 10 characters
- End date before start date — totalDays resets to 0, submit button may disable
- Past date detection — submit button disables if start_date is in the past
- Half-day duration type selection — affects totalDays calculation (raw day diff vs business-day logic)

---

## 🔒 Section 5: Security Design

### Q16: Describe the role containment hierarchy in LeaveFlow.
**A:** The role containment hierarchy is:
`x_snc_leaveflow.admin` > `x_snc_leaveflow.manager` > `x_snc_leaveflow.employee`

- **admin**: Full access to all portal pages, configuration, user impersonation, and data export
- **manager**: Access to manager-dashboard and apply-leave pages; can approve/reject requests assigned to them; cannot view other managers' queues
- **employee**: Access to home, apply-leave, my-requests pages; can submit and view own requests; cannot access manager dashboard or other employees' records

### Q17: What ACLs are configured on the `x_snc_leaveflow_leave_request` table?
**A:** Table-level ACLs:
- **Read**: Role `x_snc_leaveflow.employee` with condition `requested_by = current user`; Role `x_snc_leaveflow.manager` with condition `approver = current user`; Role `x_snc_leaveflow.admin` unrestricted
- **Write**: Role `x_snc_leaveflow.manager` and `x_snc_leaveflow.admin` only; condition `status IN ('draft', 'submitted')` blocks edits after submission

### Q18: What field-level ACLs exist?
**A:** Field-level ACLs:
- `approved_by` — Read: manager/admin; Write: manager/admin on approved/rejected records
- `approved_date` — Write: system-only on status transition
- `approver_comments` — Write: manager/admin on rejection/approval

### Q19: What is the Query Business Rule and how does it work?
**A:** Active Query Business Rule on `x_snc_leaveflow_leave_request`:
- Condition: `WHEN status = 'submitted' AND current.user.hasRole('x_snc_leaveflow.employee') THEN SET work_on = TRUE ELSE SET work_on = FALSE`
- Ensures employees only see their own records in list views by automatically adding condition `requested_by = current user`

### Q20: How are UI Builder page-level permissions handled?
**A:** Each variant has an **Audiences** field restricting visibility by role. Dashboard widgets inherit portal role unless explicitly configured otherwise. Script Includes marked `Client callable: YES` only for actions explicitly intended for client execution (approve/cancel buttons, export).

---

## ⚙️ Section 6: Setup and Configuration

### Q21: What are the prerequisites for implementing LeaveFlow?
**A:** Prerequisites:
- ServiceNow Brazil (2024) or later PDI instance
- UI Builder plugin `com.glide.ui_builder` activated (System Definition → Plugins)
- Nager.Date API access (public endpoint, no credentials needed)

### Q22: How is the update set imported?
**A:** Update set import steps:
1. Navigate to **System Update Sets** → select or create update set `LeaveFlow`
2. **Import** the `.par` file committed to GitHub repository `LeaveFlow-ServiceNow`
3. Verify imported components appear in Application Navigator:
   - `LeaveFlow` application module
   - `x_snc_leaveflow_leave_request` table
   - Script Includes: `ManagerLeaveActions`, `LeaveRequestUtils`, `HolidayService`, `LeaveExporter`
   - Choice list: `x_snc_leaveflow_leave_type` (Annual, Sick, Casual, Compensatory)
   - UI Builder Experience: `LeaveFlow Portal` (url path: `leaveflow`)
   - Dashboards: `LeaveFlow Management Dashboard`

### Q23: How are Leave Types seeded?
**A:** Leave type seeding steps:
1. Navigate to **Lists → Choice** → filter `Table` = `x_snc_leaveflow_leave_request`, `Field` = `leave_type`
2. Add choice entries:
   - `Annual` — `days_allocated`: 20 (or org policy)
   - `Sick` — `days_allocated`: 10
   - `Casual` — `days_allocated`: 5
   - `Compensatory` — `days_allocated`: 3
3. Click **Save**

### Q24: How are test users created?
**A:** Test user creation:
- **Employee**: `test.employee@test.com` with `x_snc_leaveflow.employee` role
- **Manager**: `test.manager@test.com` with `x_snc_leaveflow.manager` + `x_snc_leaveflow.employee` roles (inheritance)
- **HR Admin**: `test.hr@test.com` with all three roles: `admin`, `manager`, `employee`

### Q25: How do you verify the UI Builder experience works?
**A:** Verification steps:
1. Navigate to `https://<instance>.service-now.com/now/leaveflow/home`
2. Confirm home page loads with hero banner, 4 KPI cards, and quick action buttons
3. Test `apply-leave` route: submit form with valid dates → verify toast + redirect
4. Test `manager-dashboard` as manager: KPI tiles populate, pending queue renders

---

## 📊 Section 7: Testing

### Q26: How many test cases are in the test plan and what do they cover?
**A:** The test plan includes T01–T40 covering:
- Employee portal access and UI verification (T01, T09, T11, T33)
- Leave application flow and validation (T02, T04, T13, T15, T28, T29)
- Manager dashboard access and approval/rejection (T05, T06, T07, T08)
- Mobile responsiveness (T09, T35)
- Real-time notifications (T10, T30)
- Holiday display (T11)
- CSV export functionality (T12, T31)
- Leave type choice validation (T13)
- Balance calculation correctness (T14)
- Overlap detection (T15)
- Role-based access control (T16, T17, T18, T25, T26, T27)
- Script error handling (T19)
- Form cancellation (T20)
- KPI tile color correctness (T21)
- Donut chart verification (T22)
- Monthly trends accuracy (T23)
- Choice list seed persistence (T24)
- Various edge cases and performance tests (T25–T40)

### Q27: What does test T09 specifically verify?
**A:** Test T09 verifies mobile responsiveness on the home page:
1. Resize browser to 375px or open on mobile
2. Navigate all pages
3. Verify KPI tiles stack 1-column
4. Hero banner reduces padding
5. Buttons become full-width
6. Tables scroll horizontally

### Q28: How does test T15 (overlap detection) work?
**A:** Test T15 verifies overlap detection:
1. Employee has approved request for Nov 1–5
2. Attempt new request for Nov 3–7
3. Attempt submit
4. Expected: Warning toast "Overlap with existing approved request (Nov 3–5). Please adjust dates."

### Q29: What does test T24 verify?
**A:** Test T24 verifies choice list seed persistence:
1. Fresh Brazil PDI instance
2. Import update set
3. Verify leave_type choices seeded
4. Expected: Four choices (Annual/Sick/Casual/Compensatory) present with correct `days_allocated` values

### Q30: How is real-time notification sync tested?
**A:** Test T30 verifies bell badge sync:
1. Employee submits request in Browser 1
2. Manager refreshes page in Browser 2 within 30 seconds
3. Expected: Manager bell badge increments from 0 to 1; clicking reveals "New leave request submitted" notification detail

---

## 🛠️ Section 8: Troubleshooting and Limitations

### Q31: What are the known limitations of LeaveFlow?
**A:** Current limitations:
- **Nager.Date API dependency** — Public endpoint; if Azure Functions region goes offline, holidays card disappears until restored
- **No offline support** — UI Builder portal requires active internet connection; no local data persistence or cached fallback
- **Limited user preference persistence** — Dashboard layouts, filter selections, and KPI preferences are not saved between sessions
- **Calendar color thresholds hardcoded** — Legend categories (Yellow: 1–2, Orange: 3–4, Red: 5+) are fixed; organizations with different team sizes may want configurable thresholds
- **CSV export lacks recurrence** — One-time leave requests only; recurring leave exports as individual records
- **Mobile form factor** — Half-day duration logic uses raw day difference, not business-day calendar

### Q32: What Phase 2 enhancements are planned?
**A:** Phase 2 future enhancements:
- **Dark mode support** — Add `now-color` CSS variable overrides for light/dark theme switching
- **Calendar recurrence support** — Enable "Repeat" selector in leave request form; store recurrence rule
- **Slack/Teams notifications** — Replace polling-based bell notifications with real-time webhook integrations
- **Google Calendar / Outlook sync** — Generate .ics files for user's personal calendar
- **Advanced analytics** — Trend visualizations, team utilization percentages, prediction models
- **Delegation during manager absence** — Allow employee to designate substitute manager with audit trail
- **KB article integration** — Link leave type choices to internal knowledge base articles
- **API endpoint for external systems** — Public REST API enabling HRIS integration, payroll sync
- **WCAG AA accessibility enhancements** — Focus management, ARIA labels, keyboard navigation, color contrast

### Q33: How would you troubleshoot if the UI Builder experience doesn't load?
**A:** Troubleshooting steps:
1. Verify plugin is activated: System Definition → Plugins → `com.glide.ui_builder`
2. Check user has `ui_builder` role
3. Clear browser cache and retry
4. Open System Logs → Script Log Statements for JavaScript errors
5. Verify table permissions (ACLs) for the leave request table
6. Test data resource in isolation: open data resource, click "Test", check output
7. Check browser console (F12) for errors
8. Verify Now Design System is loaded
9. Test with user impersonation: Profile icon → Impersonate User → select test user → verify access

### Q34: What would you do if Nager.Date API returns an error?
**A:** If Nager.Date API fails:
1. `HolidayService.getHolidays()` wraps fetch in try/catch
2. On network error or non-200 response, returns empty array `[]`
3. Portal continues to function without holidays displayed (no fallback mock data)
4. Check Azure Functions region status
5. Consider implementing `sp_cache` or instance metadata cache for future resilience
6. Monitor error logs in System Logs for recurring failure patterns

### Q35: How would you handle a scenario where a manager cannot approve requests?
**A:** Troubleshooting steps:
1. Verify manager has `x_snc_leaveflow.manager` role AND `x_snc_leaveflow.employee` role
2. Check ACL on `x_snc_leaveflow_leave_request` — condition `approver = current user` for read access
3. Check Query Business Rule — ensures employees only see their own records
4. Verify Script Include `ManagerLeaveActions` is `client_callable: YES` for approval button
5. Check data resource `getpendingapprovals` filter: `status=submitted^approver=gs.getUserID()`
6. Review System Logs for JavaScript errors on approval button click
7. Test with role inheritance: ensure manager user has both roles explicitly assigned

---

## 🎯 Section 9: Quick Fire Questions

### Q36: What is the URL path for the LeaveFlow portal?
**A:** `/now/leaveflow/home` (the Experience URL path suffix is `leaveflow`)

### Q37: What choice list table and field store leave types?
**A:** `x_snc_leaveflow_leave_request`, field `leave_type`

### Q38: What are the four leave types and their default allocated days?
**A:** Annual (20), Sick (10), Casual (5), Compensatory (3)

### Q39: What is the data resource name for getting employee leave balance?
**A:** `getemployeeleave_balance`

### Q40: What Script Include method calculates total days when dates change?
**A:** `calculateTotalDays(startDate, endDate, durationType)` in `LeaveRequestUtils`

### Q41: What is the bell icon data resource name?
**A:** `getnotifications`

### Q42: What is the CSV export data resource name?
**A:** `LeaveExporter.exportToPDF()` (or CSV generation method)

### Q43: What is the Nager.Date API endpoint format?
**A:** `https://nager.date-api.azure.net/v3/PublicHolidays/{year}/{countryCode}`

### Q44: What is the minimum reason length validation?
**A:** 10 characters (form validates and shows error toast if less than 10)

### Q45: What is the role hierarchy order (highest to lowest)?
**A:** `x_snc_leaveflow.admin` > `x_snc_leaveflow.manager` > `x_snc_leaveflow.employee`

### Q46: What happens when a manager tries to approve another manager's request?
**A:** Error toast: "You are not authorized to approve this request" (returned from `ManagerLeaveActions.approveRequest()` role verification)

### Q47: What is the total_days calculation formula?
**A:** `Math.ceil((end.getTime() - start.getTime()) / (1000 * 3600 * 24)) + 1` for full_day; for half_day returns raw day diff

### Q48: What is the Nager.Date country code default?
**A:** `us` (configurable via Script Include parameter `holidayCountryCode`)

### Q49: What is the success toast message for approve request?
**A:** "Leave request approved successfully"

### Q50: What is the error message if rejection reason is missing?
**A:** "Rejection reason is required"