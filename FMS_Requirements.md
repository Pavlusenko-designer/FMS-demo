# Facility Management System (FMS) — Full Product Requirements

> **Audience:** This document is written for a coding agent. Every section is implementation-ready. Ambiguity is intentional only where flexibility is acceptable; otherwise requirements are prescriptive.

---

## 1. Project Overview

Build a fully interactive, browser-based prototype of a **Facility Management System**. The system manages the lifecycle of facility inspections, issues, repair jobs, escalations, and reporting — applicable to any physical facility type: bank branches, ATMs, shopping malls, retail stores, office buildings.

The prototype consists of **two web applications** that share data through `localStorage`, simulating a real production environment with live cross-app sync.

| File | Primary Users | Interface |
|---|---|---|
| `admin-console.html` | Super User, Facility Manager | Full-screen desktop web app |
| `mobile-app.html` | QA Specialist, Field Specialist | Mobile-first, device-frame on desktop |

### Design goals
- **Demo-ready:** The system opens with data already in motion — incidents in progress, reports under review, jobs pending. No blank-slate onboarding required.
- **Off-script capable:** Every entity is fully editable. Every flow can be entered from any direction. The demo script is a guide, not a constraint.
- **Impressive AI moments:** Three specific AI interactions (photo scan, workload-aware assignment, vendor scoring) must feel real and polished — not just text labels.
- **Enterprise credibility:** UI quality should match Salesforce, ServiceNow, or similar — not a student project.

---

## 2. Tech Stack

**No restrictions.** Use whatever produces the best result fastest. Recommended options:

- **Vanilla HTML + CSS + JS** — simplest, fully self-contained
- **React (via CDN / esm.sh)** — acceptable if it speeds up component reuse
- **Vue 3 (via CDN)** — acceptable for the same reason
- **Tailwind CSS (via CDN Play)** — acceptable for utility-first styling

**Hard constraints regardless of stack:**
- No build step, no npm, no bundler. Everything must run by opening the HTML file directly in a browser.
- No backend. No real API calls. All data in `localStorage`.
- All CDN libraries must load from `cdnjs.cloudflare.com`, `esm.sh`, `cdn.jsdelivr.net`, or `unpkg.com`.
- Both files must be in the same folder and reference no external local files.

**Recommended libraries (all via CDN):**
- **Chart.js** — analytics charts
- **Font Awesome 6** or **Lucide Icons** — iconography
- **Inter font** via Google Fonts — typography
- **Day.js** — date manipulation (lightweight)

---

## 3. Data Model

All data lives in `localStorage`. Initialise with seed data on first load (check for a `fms_initialised` key to avoid overwriting live demo data on reload).

### 3.1 Entity Definitions

#### User
```
id                    string (UUID)
firstName             string
lastName              string
phone                 string
email                 string
employeeNumber        string
country               string
department            string
position              string
userType              enum: SuperUser | FacilityManager | QASpecialist | FieldSpecialist | Vendor
facilityIds           string[]  — which facilities this user belongs to (empty = all, for SuperUser)
isQAReportReviewer    boolean
status                enum: Active | Inactive
mobileAuthMethod      enum: SMS_OTP | Biometric | None
desktopAuthMethod     enum: Email_OTP | None
avatarInitials        string    — derived from name, stored for display
```

#### Vendor
```
id                    string
companyName           string
country               string
representativeFirst   string
representativeLast    string
phone                 string
email                 string
specialisation        string    — e.g. "HVAC", "Electrical", "Security Systems"
rating                number    — 1–5, one decimal place
avgResolutionDays     number    — for AI vendor suggestion display
totalJobsCompleted    number
```

#### Facility
```
id                    string
name                  string    — e.g. "Dubai Main Branch"
type                  enum: Branch | ATM | Office | Mall | Retail
country               string
city                  string
address               string
managerId             string    — userId of assigned FacilityManager
```

#### Category
```
id                    string
name                  string    — e.g. "ATM", "Branch", "Office"
description           string
```

#### SubCategory
```
id                    string
categoryId            string
name                  string    — e.g. "ATM Vestibule", "Main Banking Hall"
country               string
description           string
locations             string[]  — optional location labels within the sub-category
```

#### Item
```
id                    string
name                  string    — e.g. "ATM Screen", "Emergency Exit Light"
defaultDepartment     string
description           string
categoryId            string
subCategoryId         string
subItems              {id: string, name: string}[]
```

#### Sample
```
id                    string
name                  string
categoryId            string
sections              {id: string, questionText: string, linkedItemId: string}[]
```

#### Checklist
```
id                    string
name                  string
sampleId              string
subCategoryId         string
country               string
```

#### Incident
```
id                    string    — format: INC-XXXX
name                  string
facilityId            string
country               string
location              string    — specific location within facility
categoryId            string
subCategoryId         string
itemId                string
description           string
creationMethod        enum: Manual | AutoFromInspection
dueDate               string (ISO date)
executorId            string    — assigned FieldSpecialist
department            string
qaId                  string    — assigned QASpecialist
workflowStatus        enum: Unassigned | AssignedToExecutor | AssignedToVendor | InProgress | UnderReview | Approved | Closed
slaStatus             enum: OnTrack | AtRisk | Overdue | NoDueDate
dateCreated           string (ISO)
jobIds                string[]
photos                {id: string, url: string, aiAnalysis: AIAnalysisResult}[]
historyLog            HistoryEntry[]
sourceReportId        string | null   — if created from inspection
sourceChecklistItemId string | null
```

#### Job
```
id                    string    — format: JOB-XXXX
relatedIncidentId     string
facilityId            string
description           string
itemId                string
categoryId            string
subCategoryId         string
priority              enum: Critical | High | Medium | Low
workflowStatus        enum: Unassigned | AssignedToExecutor | AssignedToVendor | InProgress | UnderReview | Approved | Closed
executorId            string
vendorId              string | null
qaId                  string
dueDate               string (ISO)
department            string
vendorAssistanceNeeded boolean
escalationNote        string
dateCreated           string (ISO)
photos                {id: string, url: string, aiAnalysis: AIAnalysisResult}[]
historyLog            HistoryEntry[]
```

#### Question
```
id                    string    — format: QST-XXXX
questionNumber        number
relatedJobId          string | null
relatedIncidentId     string | null
relatedReportId       string | null
categoryId            string
subCategoryId         string
itemId                string
text                  string
responsibleExecutorId string
priority              enum: Critical | High | Medium | Low
workflowStatus        enum: Open | Assigned | Responded | UnderReview | Closed
dateCreated           string (ISO)
responses             {userId: string, text: string, photoUrl: string | null, timestamp: string}[]
historyLog            HistoryEntry[]
```

#### Report (Inspection Report)
```
id                    string    — format: RPT-XXXX
facilityId            string
country               string
location              string
categoryId            string
subCategoryId         string
checklistId           string
responsibleQAId       string
dateCreated           string (ISO)
dateSubmitted         string | null
status                enum: Draft | Submitted | UnderReview | Finalized | Rejected
rejectionComment      string | null
failedItemsCount      number
createdQuestionsCount number
inspectionScore       number    — 0–100
inspectionResults     InspectionResult[]
historyLog            HistoryEntry[]
```

#### InspectionResult
```
checklistItemId       string
questionText          string
itemId                string
answer                enum: Pass | Fail | NA
photo                 {url: string, aiAnalysis: AIAnalysisResult} | null
commentary            string
incidentId            string | null    — auto-created if answer = Fail
```

#### AIAnalysisResult
```
detectedObject        string    — e.g. "ATM Cash Acceptor"
condition             enum: Good | WornNormal | Damaged | Critical
conditionLabel        string    — human-readable
confidence            number    — 0–100
tags                  string[]  — e.g. ["mechanical", "requires-inspection"]
```

#### Notification
```
id                    string
userId                string
message               string
timestamp             string (ISO)
read                  boolean
entityType            enum: Incident | Job | Question | Report
entityId              string
actionRequired        boolean
```

#### HistoryEntry
```
timestamp             string (ISO)
userId                string
userName              string
action                string    — past-tense verb phrase, e.g. "Status changed to In Progress"
detail                string | null
```

---

### 3.2 localStorage Keys

```
fms_initialised       boolean
fms_users             User[]
fms_vendors           Vendor[]
fms_facilities        Facility[]
fms_categories        Category[]
fms_subcategories     SubCategory[]
fms_items             Item[]
fms_samples           Sample[]
fms_checklists        Checklist[]
fms_incidents         Incident[]
fms_jobs              Job[]
fms_questions         Question[]
fms_reports           Report[]
fms_notifications     Notification[]
fms_auth_settings     {userType: string, mobileAuth: string, desktopAuth: string}[]
fms_current_admin_user  string (userId)
fms_current_mobile_user string (userId)
```

Cross-tab sync: both apps listen to the `window` `storage` event and re-render any affected views when a key changes.

---

### 3.3 Seed Data

Pre-populate on first load. This data represents a world already in motion.

#### Facilities (3)
- **Dubai Main Branch** — Branch, UAE, managed by facility manager
- **Riyadh ATM Cluster — King Fahd Road** — ATM, Saudi Arabia, managed by same manager
- **London Canary Wharf Office** — Office, UK, managed by same manager

#### Users (8)
| Name | Role | Facility scope |
|---|---|---|
| Alex Morgan | SuperUser | All |
| Sara Hassan | FacilityManager | All 3 facilities |
| Omar Khalid | QASpecialist | Dubai + Riyadh |
| Priya Nair | QASpecialist | London |
| James Wright | FieldSpecialist | Dubai |
| Fatima Al-Amin | FieldSpecialist | Dubai + Riyadh |
| David Chen | FieldSpecialist | London |
| — | Vendor contact | n/a |

#### Vendors (3)
- **Gulf Tech Services** — Electrical, rating 4.8, avg 1.8 days, 47 jobs
- **SecurePro Systems** — Security/CCTV, rating 4.5, avg 2.4 days, 31 jobs
- **CoolAir HVAC** — HVAC, rating 4.2, avg 3.1 days, 22 jobs

#### Categories / SubCategories / Items
**ATM category:**
- SubCategory: ATM Vestibule → Items: Cash Acceptor, Card Reader, Screen Display, Receipt Printer, Door Lock Mechanism, Security Camera
- SubCategory: ATM Exterior → Items: Canopy Lighting, Signage, Privacy Screen

**Branch category:**
- SubCategory: Main Banking Hall → Items: Teller Counter, Queue Management System, CCTV Camera, Emergency Exit Light, Fire Extinguisher, Air Conditioning Unit
- SubCategory: Vault Area → Items: Vault Door, Biometric Scanner, CCTV Camera (Vault)
- SubCategory: Customer Waiting Area → Items: Seating, Digital Display Board, ATM Machine

**Office category:**
- SubCategory: Reception → Items: Access Control Terminal, Reception Desk, CCTV, Fire Alarm Panel
- SubCategory: Server Room → Items: Server Rack Cooling, UPS System, Fire Suppression, Access Control

#### Samples (2)
- **ATM Standard Inspection** — 8 questions covering all ATM items
- **Branch Full Inspection** — 12 questions covering banking hall + vault + waiting area

#### Checklists (2)
- **Dubai ATM Monthly Check** — links ATM Standard Inspection to ATM Vestibule, UAE
- **Dubai Branch Quarterly** — links Branch Full Inspection to Main Banking Hall, UAE

#### Incidents (6 — in various states)
| ID | Facility | Item | Status | SLA | Priority |
|---|---|---|---|---|---|
| INC-0001 | Dubai Branch | ATM Screen Display | InProgress | AtRisk | High |
| INC-0002 | Dubai Branch | Emergency Exit Light | UnderReview | OnTrack | Critical |
| INC-0003 | Riyadh ATM | Cash Acceptor | Unassigned | Overdue | Critical |
| INC-0004 | Dubai Branch | Air Conditioning Unit | Approved | OnTrack | Medium |
| INC-0005 | London Office | Fire Alarm Panel | InProgress | OnTrack | High |
| INC-0006 | Riyadh ATM | Door Lock Mechanism | AssignedToExecutor | OnTrack | Medium |

Each incident has 2–4 history log entries showing realistic state transitions.

#### Jobs (8)
At least 2 jobs per active incident. Mix of: one assigned to executor, one pending vendor, one completed. Priority distribution: 2 Critical, 3 High, 2 Medium, 1 Low.

#### Reports (3)
- **RPT-0001** — Dubai Branch Quarterly, status: UnderReview, score: 72%, 3 failed items, submitted 2 days ago
- **RPT-0002** — Dubai ATM Monthly, status: Finalized, score: 91%, 1 failed item, approved
- **RPT-0003** — Dubai Branch (newest), status: Draft, score: 0% (in progress), QA has completed 4 of 12 items

#### Questions (4)
Mix of Open, Responded, and Closed states linked to above jobs/incidents.

#### Notifications (8, mix of read/unread)
Cover: new job assigned, report approved, escalation received, question responded, SLA warning.

---

## 4. Admin Console (`admin-console.html`)

### 4.1 Layout

**Top bar (fixed):**
- Left: hamburger toggle for sidebar, app logo/wordmark "FMS Admin"
- Centre: **Facility Selector** — dropdown/chip strip showing facilities the logged-in user can access. SuperUser sees all + "All Facilities" option. FacilityManager sees only their assigned facilities. Switching facility scopes all data on screen in real time.
- Right: Demo Mode toggle button | Bell icon with unread count badge | User avatar chip (name + role) with logout

**Sidebar (collapsible, 240px expanded / 60px icon-only collapsed):**
Sidebar background: dark navy (`#0F172A`) or deep slate. Active item has accent highlight.

Navigation items in order:
1. 🏠 Dashboard (Situation Board — default landing)
2. 👥 Users & Authentication
3. 🏢 Vendors
4. 📋 Reports
5. ⚠️ Incidents
6. 💼 Jobs
7. ❓ Questions
8. 📁 Samples & Checklists
9. 🗂 Categories & Items
10. 📊 Analytics

**Simulated login:** On load, show a brief login screen — select from SuperUser / FacilityManager dropdown (no password). Store selection in `fms_current_admin_user`. After selecting, render the full app.

---

### 4.2 Page 1 — Dashboard (Situation Board)

This is the default landing page. It must immediately communicate facility health and require no navigation to be useful.

#### KPI strip (4 cards, top of page)
Show numbers scoped to selected facility:
- **Open Incidents** — count, with delta vs last week (e.g. "+2 this week")
- **Overdue Jobs** — count, red if > 0
- **Pending Escalations** — jobs with vendorAssistanceNeeded = true and no vendor assigned
- **Avg Inspection Score** — last 30 days, colour-coded (green ≥ 85, amber 70–84, red < 70)

#### Active Incidents panel
Top 5 most urgent incidents (sorted by SLA status then priority). Each row shows: ID, location, item, assigned executor (avatar chip), SLA badge, status badge, "View" button opening incident detail panel.

#### Pending Actions panel
Items requiring the manager's attention right now:
- Reports awaiting approval (UnderReview status)
- Jobs awaiting vendor assignment
- Overdue incidents with no executor
Each item is a card with a direct action button (Approve Report / Assign Vendor / Assign Executor).

#### Recent Activity feed
Last 10 history log entries across all entities, newest first. Each entry: avatar, action text, entity link, relative timestamp (e.g. "3 min ago").

#### Inspection Score trend
Small Chart.js line chart showing last 6 inspection scores for the selected facility. If no facility selected (SuperUser + All), show average across all.

---

### 4.3 Page 2 — Users & Authentication

**Default tab:** Users

#### Users tab
- Table columns: Avatar+Name, Employee #, Country, Role, Department, Position, Phone, Status (toggle), Actions
- Search: by name or employee number
- Filters: Country (dropdown), Role (dropdown), Department (dropdown), Status (dropdown)
- "+ Add User" button — opens Add/Edit modal
- Per-row actions: Edit (pencil), Deactivate toggle, Delete (with confirmation dialog)

**Add/Edit User modal:**

Section 1 — Personal Information:
- First Name*, Last Name*, Phone*, Email*

Section 2 — Employee Details:
- User Type* (SuperUser / FacilityManager / QASpecialist / FieldSpecialist)
- Assigned Facilities (multi-select; hidden if SuperUser selected)
- Country*, Department*, Position*
- Checkbox: "Can review QA Reports"

Section 3 — Profile Settings:
- Employee Number*, Status toggle (Active/Inactive)

On save: write to `fms_users`, show success toast, close modal, refresh table.

#### Authentication tab
Grid of role cards (one per user type: QASpecialist, FieldSpecialist, FacilityManager, Vendor).

Each card:
- Role name + icon
- Mobile Auth dropdown: SMS OTP | Biometric | No 2FA
- Desktop Auth dropdown: Email OTP | No 2FA
- Description of what changes when settings are updated
- "Save Changes" button per card (or auto-save with visual confirmation)

Changes persist to `fms_auth_settings`.

---

### 4.4 Page 3 — Vendors

Table columns: Company, Specialisation, Representative, Country, Phone, Email, Rating (star display), Jobs Completed, Actions

Search: by company name or specialisation
Filters: Country, Specialisation

"+ Add Vendor" button — modal with:
- Company Name*, Country*, Specialisation*
- Representative First Name*, Last Name*, Phone*, Email*
- Rating (optional), Avg Resolution Days (optional)

Row actions: View (expand details panel), Edit, Delete (with confirmation)

---

### 4.5 Page 4 — Reports

Table columns: ID, Facility, Category, Sub-Category, Responsible QA, Checklist, Date, Score (coloured), Status badge

Search: by facility name, category, or QA name
Filters: Status (multi-select), Sub-Category, QA, Date range, Facility

Clicking any row opens **Report Preview** (right-side panel or full modal):

**Report Preview:**

Header with: Report ID, Facility name, status badge, action buttons:
- **Approve** (only if status = UnderReview) → changes to Finalized, adds history entry, sends notification to QA
- **Decline** (only if status = UnderReview) → requires rejection comment in a small inline text area, then changes to Rejected, sends notification
- **Export PDF** → calls `window.print()` on a print-specific stylesheet version

Overview section:
- Score (large, colour-coded), Failed Items count, Questions Created count
- Facility, Category, Sub-Category, Checklist used, Date Created, Date Submitted, QA Name

Inspection Results section:
List of all checklist items, each showing:
- Question text
- Item name (linked)
- Answer badge: Pass (green) | Fail (red) | N/A (grey)
- If photo exists: thumbnail with **AI Analysis badge** overlay. Hovering or tapping badge expands a small card: detected object, condition, confidence bar, tags. Badge colour = condition severity (green/amber/red).
- Commentary text
- If Fail: link to auto-created incident (INC-XXXX)

---

### 4.6 Page 5 — Incidents

**Default tab:** Incidents list

Table columns: ID, Facility, Location, Category, Sub-Category, Creation Method (badge), Assigned Executor (avatar+name), Jobs (e.g. "2/3 done"), Due Date, Workflow Status (badge), SLA Status (badge+icon)

KPI strip above table: Total Open | At Risk | Overdue | Unassigned (all scoped to facility filter)

Search: by location, item, or incident ID
Filters: Facility, Category, Workflow Status, SLA Status, Creation Method, Assigned Executor, Due Date range

"+ Create Incident" button — opens full Create Incident form (modal or slide-over):
- Incident Name*, Facility* (dropdown), Location*, Category*, Sub-Category*, Item* (cascades), Description*, Due Date
- Assign Executor (dropdown with **workload chips** — see AI section)
- Assign QA, Department
- Photo upload (multiple)

Clicking a row opens **Incident Detail Panel** (right slide-over, ~480px wide):

**Incident Detail Panel sections:**

1. Header: ID, Name, status badge, SLA badge, action buttons
2. Main Information: Facility, Location, Category, Sub-Category, Item, Creation Method, Date Created, Due Date
3. Assignment (all editable by SuperUser/Manager):
   - Assigned Executor (dropdown with workload chips + AI recommendation chip showing "AI recommends Fatima Al-Amin — workload: low")
   - Assigned QA
   - Department
   - Due Date
   - "Send Email Notification" button (mock — shows toast "Notification sent to [executor name]")
4. Jobs section: card for each linked job (ID, description, priority, status, assigned executor). Each card is clickable → opens Job Detail Panel
5. Photos gallery: grid of uploaded photos with AI analysis badges
6. History Log: timeline of all state changes, newest first
7. Action buttons: Change Status (dropdown of valid next statuses), Add Question, Close Incident

---

### 4.7 Page 6 — Jobs

Table columns: ID, Incident ID (link), Facility, Item, Priority (badge), Workflow Status (badge), Assigned Executor, Vendor (if any), Department, Due Date, SLA Status

KPI strip: Total Open | Critical | Vendor Pending | Overdue

Search: by Job ID or Incident ID
Filters: Facility, Workflow Status, Priority, Executor, Department, Vendor, Due Date

Clicking a row opens **Job Detail Panel:**

1. Header: ID, priority badge, status badge, SLA badge, action buttons
2. Main Information: Facility, Category, Sub-Category, Item, Related Incident (link), Date Created, Due Date
3. Description: full job description text
4. Assignment: Executor (with workload chips + AI recommendation), QA, Department, Due Date
5. Vendor Assistance:
   - Toggle: "Vendor Assistance Needed" — when turned on, shows vendor dropdown
   - Vendor dropdown: each option shows company name, specialisation, rating, avg resolution days, a "**AI Recommended**" badge on the best match
   - Escalation note text area
6. Photos / Evidence gallery with AI analysis per photo
7. History Log
8. Actions: Change Status, Add Question, Mark Complete, Approve (for manager)

---

### 4.8 Page 7 — Questions

Table columns: ID, Question #, Related Job (link), Related Incident (link), Category, Sub-Category, Item, Responsible Executor, Priority, Status

Search: by ID, Job ID, or Incident ID
Filters: Workflow Status, Priority, Executor, Item, Category

Clicking opens **Question Detail Modal:**
- Full question text
- Related entities (links to incident, job, report)
- All responses listed (author avatar, text, photo if attached, timestamp)
- "Add Response" area: text input + optional photo upload
- Status change buttons: Assign | Close | Re-open

---

### 4.9 Page 8 — Samples & Checklists

**Default tab:** Samples

#### Samples tab
Table: Name, Category, Questions Count, Linked Checklists Count, Actions

"+ Add Sample" modal:
- Sample Name*, Category* (dropdown)
- Questionnaire builder: drag-reorderable list of question rows. Each row: question text field + Linked Item dropdown. "Add Question" button adds a new row. Trash icon removes.

Edit opens same form pre-populated. Delete requires confirmation.

#### Checklists tab
Table: Name, Country, Sub-Category, Sample (name), Actions

"+ Add Checklist" modal:
- Name*, Country*, Sub-Category* (searchable dropdown), Sample* (dropdown)
- Read-only preview of the inherited questions from the selected Sample appears below the dropdowns

---

### 4.10 Page 9 — Categories & Items

**Default tab:** Categories

#### Categories tab
Table: Name, Description, Sub-Categories count (clickable to expand)

Clicking Sub-Categories count expands an inline row below the category showing all its sub-categories as a nested table (Name, Country, Description, Delete button).

"+ Add Category" modal:
- Category Name*, Description
- Sub-Categories builder: list of sub-category rows (Name, Country, Description, + Add Location button that appends a location tag inline). Add/remove rows.

#### Sub-Categories tab
Table: Name, Country, Parent Category, Description, Items Count
Search by name.
"+ Add Sub-Category" button — modal with Category dropdown (searchable) + sub-category fields.

#### Items tab
Table: Item Name, Category, Default Department, Sub-Items count, Description
Search by name.

"+ Add Item" modal:
- Item Name*, Category*, Sub-Category*, Default Department*, Description
- Sub-Items section: list of sub-item name inputs, add/remove

---

### 4.11 Page 10 — Analytics

Period controls (shared across all tabs): Day | Month | Year toggle + date range pickers. Facility filter (scoped by login role). Category filter.

Toggle for all tabs: **Table View** ↔ **Chart View**

#### Incidents tab
Table view columns: Period | Auto-created | Manual | Unassigned | Assigned Executor | Assigned Vendor | In Progress | Under Review | Approved | Closed | **Total**
Chart view: Stacked bar chart (by status) using Chart.js

#### Jobs tab
Table: Period | Unassigned | Assigned Executor | Assigned Vendor | In Progress | Under Review | Approved | Closed | Critical | High | Medium | Low | **Total**
Chart: Grouped bar (status vs priority)

#### Questions tab
Table: Period | Open | Assigned | Responded | Under Review | Closed | **Total**
Chart: Line chart showing trend over time

#### Reports tab
Table: Period | Draft | Submitted | Under Review | Finalized | Rejected | Avg Score | **Total**
Chart: Bar chart of scores per facility

---

### 4.12 Demo Mode

A toggle button in the top bar activates Demo Mode. When active:

- A persistent banner appears at the top of the content area: "Demo Mode — Guided Scenario"
- A floating side panel (collapsible) shows the scenario guide:

```
Guided Scenario: ATM Screen Fault — End to End
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
① [✓] QA inspects ATM, marks screen as Fail        → view in Reports
② [→] Incident INC-0001 auto-created               → view in Incidents  
③ [  ] Manager assigns James Wright to JOB-0001    → do now
④ [  ] Specialist marks job In Progress             → switch to Mobile
⑤ [  ] Specialist uploads evidence, submits to QA  → switch to Mobile
⑥ [  ] Manager approves report                     → view in Reports
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[← Prev]  Step 3 of 6  [Next →]
```

- Each "view" link navigates to that section with the relevant item highlighted (pulsing border or scroll-into-view)
- Demo Mode does not disable any functionality — users can click anywhere at any time
- Step completion is tracked automatically based on data state (e.g. step 3 checks off when JOB-0001 gets an executor assigned)

---

## 5. Mobile App (`mobile-app.html`)

### 5.1 Visual Design

On desktop: centred device frame (390×844px — iPhone 14 proportions) with rounded corners (50px), subtle device shadow, and a static status bar at the top (time: "09:41", signal bars, battery icon). The frame background is white/light. Scrolling happens inside the frame, not the page.

On mobile viewport (≤480px): frame disappears, app fills the full screen.

Font, spacing, and touch targets must feel native — minimum 44px tap targets.

### 5.2 Login Screen

Clean, minimal screen inside the device frame:
- FMS logo / wordmark
- "Select your role:" — two large tap cards: **QA Specialist** | **Field Specialist**
- After selecting role: "Select user:" — dropdown populated with users of that type
- "Sign In" button
- Stores selection in `fms_current_mobile_user`

### 5.3 Common Elements

**Bottom tab bar** (always visible, 5 tabs):
- Home (role-specific label)
- Jobs (FieldSpecialist) / Inspections (QASpecialist)
- Questions
- Notifications (badge with unread count)
- Profile

**Top bar per screen:** Back button (when drilling into detail) | Screen title | Optional action (+ or filter icon)

**Toast notifications:** Slide in from top, 3-second auto-dismiss, swipe-to-dismiss. Used for all action confirmations.

**Global floating chatbot button** — bottom right, above tab bar, semi-transparent. Opens chat overlay.

---

### 5.4 QA Specialist — Screens

#### Home tab: My Inspections
List of reports assigned to this QA user. Each card:
- Report ID, Facility name, Checklist name
- Status badge (Draft / Submitted / Under Review / Finalized / Rejected)
- Score (if started) or "Not started"
- Date
- Chevron

Top of list: "**+ Start New Inspection**" button (primary, full width)

#### Start New Inspection flow

Screen 1: Setup
- Facility* (dropdown, scoped to QA's facilities)
- Checklist* (filtered by facility's category)
- "Start Inspection" button → creates Draft report in localStorage, navigates to checklist

Screen 2: Checklist Walkthrough — **one item per screen**

Layout:
- Progress bar at top (e.g. "Item 4 of 12")
- Item name (large, 20px bold)
- Question text (16px)
- Item category/location label

**Photo capture area:**
- Dashed border box with camera icon and "Tap to photograph this item"
- When photo selected: thumbnail appears. After 400–700ms delay, show animated scan:
  - Sweeping horizontal line animation across the image (CSS animation)
  - "Analysing..." spinner text
  - Then: AI Analysis result card appears below image (see Section 7.2)

**Answer buttons (large, full-width, stacked):**
- ✅ Pass (green)
- ❌ Fail (red)
- — N/A (grey)

**Commentary field:** Optional text input "Add observation note..."

**Navigation:** "← Previous" and "Next →" buttons at bottom. On last item: "Next →" becomes "**Review & Submit**"

**If Fail is tapped:**
- Button turns solid red with checkmark
- A small banner appears: "An incident will be auto-created for this item"
- Commentary field becomes recommended (not required)

Screen 3: Review Summary
- Table of all items: Question | Answer | Photo? | Has note?
- Failed items highlighted in red rows
- Inspection Score displayed prominently (calculated as Pass / (Pass+Fail) × 100)
- Summary: X passed, Y failed, Z skipped
- "**Submit for Review**" button (primary) → changes report status to Submitted, creates incidents+jobs for all Fail items, sends notification to FacilityManager, navigates to report detail
- "Back to Edit" button

#### Report Detail Screen (from list tap)
- Header: Report ID, Facility, status badge
- Score (large ring chart or number) + stat chips: Failed Items, Questions, Date
- Inspection Results: scrollable list matching admin Report Preview but read-only
- Photo thumbnails with AI analysis badge
- If Finalized: green checkmark banner "Approved by [Manager Name]"
- If Rejected: red banner with rejection comment
- "Export / Share" button → `window.print()` with print stylesheet

#### Questions tab (QA view)
List of questions linked to this QA's reports.
Each card: Question ID, related report/incident, question text (truncated), status badge

Tap → Question Detail:
- Full question text
- Related entities (links — tapping opens that entity's detail screen within mobile app)
- Response thread
- "Add Response" — text area + optional photo upload + "Submit" button

#### Notifications tab
List of notifications newest first. Each item: icon (coloured by type), message, relative timestamp, read/unread dot.
Tap → navigates to relevant entity. "Mark all read" button in header.

---

### 5.5 Field Specialist — Screens

#### Home tab: My Jobs
List of jobs assigned to this specialist. Sorted by: Critical first, then due date.

Each card:
- Job ID, priority badge (colour-coded), SLA badge
- Item name + Incident ID
- Workflow status badge
- Due date (red if overdue)
- Chevron

Filter icon in header: filter by status, priority

Tap → **Job Detail Screen**

#### Job Detail Screen

Tabs within screen: **Details | Evidence | History**

**Details tab:**
- Job ID, Priority badge, Status badge, SLA badge
- Facility, Location, Category, Sub-Category, Item
- Job description (full text)
- Related Incident (chip that opens Incident Detail)
- Assigned by (manager name)
- Due Date

**Action buttons (contextual — show based on current status):**
- "Accept Job" → status: AssignedToExecutor (if Unassigned)
- "Start Work" → status: InProgress (adds history entry)
- "Submit for Review" → status: UnderReview, sends notification to QA+Manager
- "Request Escalation" → opens escalation flow (below)

**Escalation flow** (modal/bottom sheet):
- "Escalation Reason" text area (required)
- Vendor selection list — each vendor shown as a card:
  - Company name + specialisation
  - Star rating (e.g. ⭐ 4.8)
  - "Avg resolution: 1.8 days"
  - Jobs at this facility type: "47 completed"
  - **"AI Recommended" badge** on the best-match vendor (highest rating × lowest avg days for this category)
- "Submit Escalation" button → sets vendorAssistanceNeeded = true, stores escalationNote + vendorId, changes status to AssignedToVendor, sends notification to Manager

**Evidence tab:**
- Grid of uploaded photos, each with AI analysis card below thumbnail
- "Upload Evidence" button — file input styled as a camera shutter button with upload icon
- On photo select: animated scan → AI analysis card (see Section 7.2)
- Photos persist to job.photos in localStorage

**History tab:**
- Timeline of all status changes, photo uploads, assignments. Newest first.
- Each entry: icon, action text, user name, relative timestamp

#### Incidents tab (Field Specialist)
Read-only list of incidents linked to this specialist's jobs, for context.

Also: "**+ Report New Issue**" button — full create-incident form:
- Incident Name, Facility (pre-selected to specialist's primary), Location, Category, Sub-Category, Item, Description, Due Date, Photo upload

On save: creates incident (method: Manual) + optional job in localStorage.

#### Questions tab (Field Specialist)
Same structure as QA Questions tab. Shows questions where responsibleExecutorId = this user.

#### Profile tab
- Name, role, facility assignments display
- Current auth method settings (read-only)
- "Switch User" → returns to login screen (clears `fms_current_mobile_user`)
- App version

---

## 6. Cross-App Sync

Both apps write to the same `localStorage` keys. Each uses `window.addEventListener('storage', handler)` to detect changes made in another tab.

### Sync behaviour by trigger

| Action (originator) | What updates automatically |
|---|---|
| QA marks item Fail, submits report | Admin: new Incident + Job appear in lists; Notification bell increments |
| QA submits report | Admin: Report status changes to Submitted; Pending Actions panel updates |
| Specialist accepts/starts/completes job | Admin: Job status badge updates in real time |
| Specialist submits escalation | Admin: Job shows vendor pending badge; Notification sent to Manager |
| Specialist uploads photo | Admin: Photo appears in Job Detail evidence tab |
| Manager approves/rejects report | Mobile: QA sees notification; Report status badge updates |
| Manager assigns executor | Mobile: Field Specialist sees new job in Jobs tab; notification |
| Manager assigns vendor | Mobile: Specialist sees vendor info on job detail |

When a `storage` event fires, the receiving app should:
1. Parse the changed key
2. Re-fetch that entity from localStorage
3. Update only the affected components (not full re-render if possible)
4. Show a subtle "Updated" flash on changed list rows

---

## 7. AI Features (Simulated, Client-Side Only)

### 7.1 Chatbot Assistant

Floating circular button (bottom right, 56px). Opens a chat overlay (bottom sheet on mobile, floating panel 380×520px on desktop).

**Behaviour:**
- Input field at bottom, send button, message thread above
- Bot messages have an avatar chip ("FMS AI")
- 400–800ms simulated "typing" indicator (3 animated dots) before each response
- Suggested prompt chips inside empty chat state (3 chips, tappable):
  - "Show overdue jobs"
  - "Which facility has the lowest score?"
  - "What's pending vendor assignment?"

**Intent handlers (keyword matching):**

| Keywords | Action |
|---|---|
| "overdue", "overdue jobs", "sla" | Navigates to Jobs page filtered to Overdue; responds with count |
| "create incident", "new incident", "report issue" | Opens Create Incident form |
| "inspect", "start inspection", "new inspection" | (Mobile) opens Start New Inspection flow |
| "incident [ID]", "INC-" | Responds with incident status, SLA, assigned executor |
| "job [ID]", "JOB-" | Responds with job status, priority, due date |
| "report [ID]", "RPT-" | Responds with report score, status |
| "score", "inspection score", "lowest score" | Responds with list of facilities + last inspection score |
| "vendor", "pending vendor" | Responds with count of jobs awaiting vendor + list |
| "help", "commands", "what can you do" | Lists all available commands with examples |
| "assign", "who should I assign" | Gives AI recommendation for current context (if on job/incident page) |

Unrecognised input: "I didn't quite get that. Type **help** to see what I can do."

All responses use data from localStorage — not hardcoded.

### 7.2 Image Analysis Simulation

Called whenever a photo is selected in: inspection checklist, job evidence upload, incident photo upload.

**Visual sequence:**
1. Photo thumbnail renders immediately
2. After 50ms: scanning animation begins — a semi-transparent blue bar sweeps top-to-bottom over the image (2-second CSS animation, repeat: 1)
3. Simultaneously: "AI Analysing..." text with spinner appears below image
4. After 400–700ms (random): animation stops, result card fades in

**Result card layout:**
```
┌─────────────────────────────────────────┐
│ 🔍 AI Analysis                    94%  │
│ ─────────────────────────────────────── │
│ Detected: ATM Cash Acceptor             │
│ Condition: ⚠ Worn — Normal wear visible │
│ Tags: mechanical · maintenance-due      │
└─────────────────────────────────────────┘
```

Condition colour-coding:
- Good → green badge
- Worn Normal → amber badge
- Damaged → red badge
- Critical → dark red badge + pulsing border

**`analyzeImage(file, context)` function:**
- `context` = {categoryId, itemId} — used to pick a realistic detected object
- Returns a mock `AIAnalysisResult` based on item category
- Object/condition pool:

| Category | Detected Objects | Possible Conditions |
|---|---|---|
| ATM | Cash Acceptor, Card Reader, Screen, Receipt Printer | Good, Worn Normal, Damaged |
| Branch | CCTV Camera, Emergency Light, Fire Extinguisher, AC Unit | Good, Worn Normal, Damaged, Critical |
| Office | Access Terminal, Fire Alarm Panel, UPS System | Good, Worn Normal |

Confidence: random 78–97%.

Results stored in `photo.aiAnalysis` in localStorage. The same analysis is shown consistently when the photo is viewed again (not re-randomised).

### 7.3 Smart Workload Assignment

Used in all executor assignment dropdowns (admin + mobile).

Each FieldSpecialist in the dropdown shows:
- Name + avatar
- A small workload bar (coloured pill): number of active jobs / capacity
  - 0–2 open jobs → green "Low"
  - 3–4 open jobs → amber "Moderate"
  - 5+ open jobs → red "Heavy"
- One specialist has an **"AI Recommended"** badge — selected as the one with lowest workload who has handled this category before

Recommendation logic (simulated):
```
score = (maxJobs - openJobs) × 2 + (hasHandledThisCategory ? 3 : 0)
recommend = specialist with highest score
```

### 7.4 Smart Vendor Suggestion

On escalation flows (admin Job Detail + mobile escalation modal):

Each vendor card shows:
- Company name, specialisation, star rating (e.g. ⭐ 4.8)
- "Avg resolution: 1.8 days · 47 jobs completed"
- Small history line: "12 jobs at ATM facilities"
- **"AI Recommended"** badge on best match

Recommendation logic (simulated):
```
score = rating × 20 + (10 / avgResolutionDays) + (hasThisSpecialisation ? 15 : 0)
recommend = vendor with highest score
```

---

## 8. Notifications System

Notifications are stored in `fms_notifications` and scoped by `userId`.

**Bell icon (admin):** Shows unread count badge. Clicking opens a dropdown (max 6 items, "View all" link navigates to a full notifications page or clears). Each item: icon by type, message, relative time, entity link, read indicator. "Mark all read" button.

**Notifications tab (mobile):** Full list, same structure.

**Notification triggers (auto-created in localStorage):**

| Event | Recipient(s) | Message template |
|---|---|---|
| Report submitted by QA | FacilityManager | "RPT-XXXX submitted by [QA name] — awaiting your review" |
| Report approved | QA | "Your report RPT-XXXX has been approved ✓" |
| Report rejected | QA | "RPT-XXXX was declined — [rejection comment]" |
| Job assigned to specialist | FieldSpecialist | "New job JOB-XXXX assigned to you — [priority] priority" |
| Escalation submitted | FacilityManager | "JOB-XXXX escalated to vendor by [specialist name]" |
| Vendor assigned to job | FieldSpecialist | "[Vendor name] assigned to JOB-XXXX" |
| SLA overdue | FacilityManager | "INC-XXXX is now overdue — [item name] at [facility]" |
| Question responded | Originator | "[Executor name] responded to your question QST-XXXX" |

---

## 9. UI & Visual Design Standards

### Colour palette
- Primary brand: `#2563EB` (blue-600)
- Primary dark: `#1E40AF` (blue-800)
- Sidebar bg: `#0F172A` (slate-900)
- Surface: `#FFFFFF`
- Background: `#F8FAFC` (slate-50)
- Border: `#E2E8F0` (slate-200)
- Text primary: `#0F172A`
- Text secondary: `#64748B`

Status colours:
- Success/green: `#16A34A`
- Warning/amber: `#D97706`
- Danger/red: `#DC2626`
- Info/blue: `#2563EB`

### Status badges
All status values rendered as small pills (`border-radius: 20px`, `padding: 2px 10px`, `font-size: 12px`, `font-weight: 500`):

| Status | Background | Text |
|---|---|---|
| Unassigned | `#F1F5F9` | `#475569` |
| In Progress | `#DBEAFE` | `#1D4ED8` |
| Under Review | `#FEF3C7` | `#92400E` |
| Approved / Finalized | `#DCFCE7` | `#166534` |
| Closed | `#F1F5F9` | `#475569` |
| Critical priority | `#FEE2E2` | `#991B1B` |
| Overdue SLA | `#FEE2E2` | `#991B1B` |
| At Risk SLA | `#FEF3C7` | `#92400E` |
| On Track SLA | `#DCFCE7` | `#166534` |

### Typography
- Font: Inter (Google Fonts) with system-ui fallback
- Page headings: 20px / 500
- Section headings: 16px / 500
- Body: 14px / 400 / line-height 1.6
- Labels: 12px / 500 / uppercase / letter-spacing 0.05em
- Table data: 14px / 400

### Spacing & components
- Card border-radius: 12px
- Card shadow: `0 1px 3px rgba(0,0,0,0.08), 0 1px 2px rgba(0,0,0,0.04)`
- Table rows: 52px height, alternating row bg (`#F8FAFC` on even)
- Inputs: 36px height, 8px border-radius, `#E2E8F0` border, focus ring `#2563EB` 2px
- Primary button: `#2563EB` bg, white text, 8px radius, hover darken 10%
- Modals: max-width 560px, centred, backdrop `rgba(0,0,0,0.4)`, 16px border-radius
- Slide-over panels: 480px wide, right-anchored, full-height

### Micro-interactions
- Button press: `transform: scale(0.97)` on active
- Status badge change: brief background colour flash (CSS transition 200ms)
- Table row addition: fade+slide in from top
- Notification: slide from top, auto-dismiss after 3s
- Photo analysis card: fade-in with upward translate (200ms)
- Mobile: all tap targets minimum 44px, active state: slight bg darken

---

## 10. Error States & Edge Cases

Handle the following gracefully (no crashes, clear user feedback):

- **Empty states:** Each list/table must have an empty state (icon + heading + subtext) when no data matches current filters. E.g. "No overdue incidents — all SLAs on track ✓"
- **Filter with no results:** Show empty state within the table, don't hide the filter controls
- **Failed photo (no file API):** Show a placeholder image with a generated AI result card so the demo works without real photos
- **localStorage full:** Catch QuotaExceededError, show toast "Storage limit reached — clear old data to continue"
- **Circular navigation:** If a user is on a detail panel and clicks a linked entity, push to that entity's detail without losing back navigation
- **Unsaved changes:** If a user opens an Edit modal and clicks away, show "Unsaved changes — leave anyway?" confirmation
- **Concurrent edit (different tabs):** When a `storage` event fires for an entity that's currently open in a detail panel, show a subtle banner "This record was updated in another window" with a "Refresh" button

---

## 11. Implementation Notes for Coding Agent

1. **Initialise data once.** Check `localStorage.getItem('fms_initialised')` at startup. If null: write all seed data, set key to `'true'`. Never overwrite existing data on reload.

2. **ID generation.** Use `crypto.randomUUID()` for new entity IDs. Seed data IDs are hardcoded strings.

3. **Date handling.** All dates stored as ISO 8601 strings. Display as relative ("3 days ago") or formatted ("15 May 2025") using Day.js or manual formatting. Never display raw ISO strings.

4. **Image handling.** Since no backend exists, use `URL.createObjectURL(file)` for in-session image display. Store the object URL string in localStorage. On page reload these URLs break — this is acceptable for demo purposes; show a placeholder on broken URLs.

5. **Print export.** For "Export PDF" buttons, use `window.print()`. Include a `@media print` stylesheet that hides sidebar/nav, shows the report content in a clean A4 layout.

6. **Cascade deletes.** When deleting a Category, warn if SubCategories or Items exist. When deleting a User, warn if assigned to open incidents/jobs. In the demo, allow deletion anyway after confirmation.

7. **Chart.js.** Load lazily only when Analytics page is first visited. Destroy and recreate chart instances when filter changes to avoid canvas reuse errors.

8. **Mobile device frame.** Implement as a CSS wrapper div: `width: 390px; height: 844px; border-radius: 50px; overflow: hidden; box-shadow: ...`. All mobile content renders inside with `overflow-y: auto`. On `@media (max-width: 480px)`: remove frame, app fills full viewport.

9. **Demo Mode state.** Store in a non-persisted JS variable (not localStorage) so it resets on reload. Demo Mode should not affect any data or functionality — it is a pure presentation layer.

10. **Code organisation.** Suggested structure within each HTML file:
    ```
    <head> — styles, CDN imports
    <body> — HTML shell (sidebar, topbar, main area placeholders)
    <script>
      // 1. Constants & config
      // 2. Data layer (localStorage read/write functions)
      // 3. Seed data
      // 4. UI render functions (one per page/component)
      // 5. Event handlers
      // 6. AI simulation functions
      // 7. Cross-tab sync listener
      // 8. Router / navigation
      // 9. Init (run on DOMContentLoaded)
    </script>
    ```

11. **No placeholder "Coming Soon" screens.** Every navigation item must render something — even if it's a read-only list. The demo must survive a curious client clicking every menu item.

12. **Performance.** Avoid rendering > 200 table rows at once. Implement simple client-side pagination (25 rows per page) on Incidents, Jobs, Questions lists.
