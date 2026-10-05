# Employee Attendance Management System — Technical Documentation

> **Version:** 2.0 **Date:** 2026-09-28
> **Stack:** Next.js · Node.js + Express · TypeScript · PostgreSQL · Prisma · JWT + Refresh Tokens · RBAC · Zod

Each section below says **what** the part is or does and **why** it was chosen, so you can use this document both as a coding blueprint and as a project submission.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Requirements](#2-system-requirements)
3. [User Roles & Permissions](#3-user-roles--permissions)
4. [Technology Stack & Why](#4-technology-stack--why)
5. [System Architecture](#5-system-architecture)
6. [Database Design](#6-database-design)
7. [Authentication & Authorization](#7-authentication--authorization)
8. [Attendance System & Rules](#8-attendance-system--rules)
9. [API Documentation](#9-api-documentation)
10. [Project Folder Structure](#10-project-folder-structure)
11. [UI Pages](#11-ui-pages)
12. [Security](#12-security)
13. [Error Handling](#13-error-handling)
14. [Testing Strategy](#14-testing-strategy)
15. [Deployment](#15-deployment)
16. [Development Roadmap](#16-development-roadmap)
17. [Future Improvements](#17-future-improvements)
18. [Glossary](#18-glossary)

---

## 1. Project Overview

### 1.1 Introduction

**What:** The Employee Attendance Management System (AMS) is a web application that lets a company record, track and report the working attendance of its employees. Employees clock in and clock out from their browser or phone, managers review their team's attendance and approve leave and corrections, and HR/administrators manage employees, departments, shifts, holidays and reports.

**Why:** Paper sign-in sheets, punch cards and spreadsheets are slow, easy to falsify ("buddy punching"), and make it hard to calculate working hours, lateness, overtime and leave balances for payroll. A central system records time accurately using the server clock, applies the company's rules automatically, and makes attendance instantly reportable.

### 1.2 Problem Statement

Manual attendance has these problems:

| Problem | Effect |
|---|---|
| Sign-in sheets / spreadsheets | Times can be written in wrongly or altered with no trace |
| Working hours and overtime calculated by hand | Slow and error-prone at month end, causes payroll disputes |
| No central view | HR cannot see who is in, late or absent across departments today |
| Employees can't check their own record | Disagreements about hours or leave balance are found only on payday |
| Leave handled by email or paper | Approvals get lost, balances drift, and leave doesn't link to attendance |
| Forgotten clock-outs fixed informally | No record of who changed the time and why |

### 1.3 Objectives

| Objective | Why it matters |
|---|---|
| Let an employee clock in or out in one click | If it is slow, people forget or skip it |
| Record times with the **server** clock, never the device clock | Employees can't fake a time by changing their phone clock |
| Store attendance safely in a relational database | Data must be permanent, consistent and queryable |
| Allow only the right people to see or change data (RBAC) | Attendance feeds payroll and is personal data |
| Calculate worked hours, lateness and overtime automatically | Removes manual errors and disputes |
| Handle leave requests and leave balances digitally | Approved leave is reflected in attendance and balances automatically |
| Handle forgotten punches through a correction request | Mistakes are fixed with approval, not silently |
| Provide daily, monthly (timesheet) and per-employee reports with CSV export | HR needs them for payroll and compliance |
| Keep an audit log of every change | Any edit can be traced to who did it and when |

### 1.4 Scope

**In scope (version 1):**

- Employee management (Admin/HR, Manager, Employee), CSV bulk import
- Departments, reporting lines (who is whose manager)
- Shifts (start/end time, grace period, break, working days)
- Company holidays
- Clock-in / clock-out from the web (desktop and mobile browser), multiple punches per day
- Automatic daily status: Present, Late, Half-day, Absent, On Leave
- Worked hours, late minutes and overtime calculation
- Attendance correction requests with manager approval
- Leave types, yearly leave balances, leave requests with approval workflow
- Nightly job that marks absences and closes forgotten clock-outs
- Dashboards for each role
- Reports and CSV export (timesheet for payroll)
- Audit logging

**Out of scope (possible later, see §17):**

- Biometric devices, face recognition, RFID/NFC card readers
- Native mobile apps
- Payroll calculation (the system only **exports** hours to payroll)
- Rotating/roster shift scheduling
- Multiple companies in one deployment (multi-tenant)

**Why define scope:** It stops the project from growing endlessly ("scope creep") and gives a clear finish line for version 1.

### 1.5 Target Users

| User | What they do |
|---|---|
| **Admin (HR)** | Sets up the system: employees, departments, shifts, holidays, leave types; edits any record; views all reports; exports payroll timesheets |
| **Manager** | Clocks in/out like any employee; views the team's attendance; approves or rejects the team's leave and correction requests |
| **Employee** | Clocks in/out; views own attendance, hours and leave balance; submits leave and correction requests |

**Note:** Managers and admins are employees too — they also have an employee profile and clock in. The role only adds extra permissions.

---

## 2. System Requirements

### 2.1 Functional Requirements

Functional requirements describe **what the system must do**.

| ID | Requirement | Why |
|---|---|---|
| FR-01 | Users can log in with email and password | Identify who is using the system |
| FR-02 | Users can log out and all their tokens are invalidated | Protect shared/public computers |
| FR-03 | Users can reset a forgotten password via email | Avoid HR handling every lost password |
| FR-04 | Admin can create, update and deactivate employees (single or CSV import) | Manage who has access |
| FR-05 | Admin can manage departments, shifts, holidays and leave types | Attendance rules need a structure to attach to |
| FR-06 | Admin can assign each employee a department, a manager and a shift | Decides the working hours and who approves requests |
| FR-07 | Employee can clock in and clock out; the server records the time | Core feature |
| FR-08 | The system prevents clocking in twice without clocking out | Keeps data correct |
| FR-09 | The system calculates the daily status, worked hours, late minutes and overtime | Removes manual calculation |
| FR-10 | A nightly job marks absent employees and closes forgotten clock-outs | Every working day has a result, even without a punch |
| FR-11 | Employee can request a correction for a missed or wrong punch (within 7 days) | Fix mistakes with approval |
| FR-12 | Manager approves/rejects correction requests of direct reports | Accountability for time changes |
| FR-13 | Admin can edit any attendance at any time (with a reason) | Handle disputes and special cases |
| FR-14 | Employee can view own attendance history, hours and leave balance | Transparency |
| FR-15 | Employee can submit a leave request; manager/admin approves or rejects | Digital leave workflow |
| FR-16 | Approved leave deducts the leave balance and marks the days as ON_LEAVE | No double work |
| FR-17 | Reports: daily, monthly timesheet, per employee, late arrivals, overtime, leave | Decision-making and payroll |
| FR-18 | Export reports to CSV | Import into payroll / Excel |
| FR-19 | Every create/update/delete of attendance, leave and users is written to an audit log | Accountability |

### 2.2 Non-Functional Requirements

Non-functional requirements describe **how well the system must work**.

| ID | Category | Requirement | Why |
|---|---|---|---|
| NFR-01 | Performance | API responds in < 300 ms for 95% of requests, including the 9:00 clock-in rush | Clock-in must feel instant even when everyone arrives at once |
| NFR-02 | Scalability | Supports 2,000 employees and 100 managers without redesign | Room to grow for a mid-size company |
| NFR-03 | Security | Passwords hashed with Argon2; HTTPS only; RBAC on every endpoint | Protect personal and payroll data |
| NFR-04 | Availability | 99.5% uptime during working hours | Employees can't clock in if it is down |
| NFR-05 | Reliability | Daily database backups, 30-day retention | Attendance feeds payroll; must be recoverable |
| NFR-06 | Usability | Works on mobile browsers; clock in/out with one tap from the dashboard | Many employees use phones |
| NFR-07 | Maintainability | TypeScript strict mode, layered modules, ≥ 70% test coverage on services | Easy to change safely |
| NFR-08 | Auditability | All attendance and leave changes are traceable | Labour-law and payroll records |
| NFR-09 | Privacy | Employees see only their own data; managers only their direct reports; location is captured only at punch time, never tracked | Legal and ethical requirement |
| NFR-10 | Correctness of time | All times stored in UTC; "work date" calculated in the company timezone | Avoids midnight/timezone bugs in hours and payroll |

### 2.3 Hardware & Software Requirements

**Development machine**

| Item | Minimum | Why |
|---|---|---|
| OS | Windows 10/11, macOS or Linux | Node runs on all |
| RAM | 8 GB | Run Node, PostgreSQL and browser together |
| Node.js | v20 LTS or newer | Long-term support, modern features |
| PostgreSQL | v15 or newer (local or Docker) | Database |
| Git | Any recent | Version control |
| Editor | VS Code | TypeScript/Prisma extensions |

**End users**

- Any modern browser (Chrome, Edge, Firefox, Safari) on desktop or mobile.
- Internet connection (optionally the office network, if IP restriction is enabled — see §8.4).

---

## 3. User Roles & Permissions

### 3.1 Why Role-Based Access Control (RBAC)

**What:** Each user has exactly one role. Each API endpoint states which roles may call it. Some endpoints also check **ownership / reporting line** (e.g. a manager may only see and approve requests of their direct reports).

**Why:** It is simple to understand, easy to enforce in one middleware, and matches a normal company structure. Checking only the role is not enough — a manager must not approve another team's leave, and nobody may approve their own requests — so we combine **role checks** with **reporting-line checks** in the service layer.

### 3.2 Roles

| Role | Description |
|---|---|
| `ADMIN` | HR / system administrator. Full control of the system |
| `MANAGER` | Employee who also manages a team of direct reports |
| `EMPLOYEE` | Clocks in/out, sees own data, submits requests |

### 3.3 Permission Matrix

| Action | Admin | Manager | Employee |
|---|:---:|:---:|:---:|
| Manage employees | ✅ | ❌ | ❌ |
| Manage departments / shifts / holidays / leave types | ✅ | ❌ | ❌ |
| Clock in / clock out (self) | ✅ | ✅ | ✅ |
| View attendance | ✅ all | ✅ own + direct reports | ✅ own only |
| Edit attendance directly | ✅ any time, with reason | ❌ | ❌ |
| Submit correction request | ✅ | ✅ | ✅ |
| Approve / reject correction | ✅ | ✅ direct reports only | ❌ |
| Submit leave request | ✅ | ✅ | ✅ |
| Approve / reject leave | ✅ | ✅ direct reports only | ❌ |
| Adjust leave balances | ✅ | ❌ | ❌ |
| View reports | ✅ all | ✅ own team | ✅ own only |
| Export CSV (timesheets) | ✅ | ✅ own team | ❌ |
| View audit logs | ✅ | ❌ | ❌ |

**Rule:** Nobody can approve their own request. A manager's own requests go to **their** manager, or to an admin if they have none.

---

## 4. Technology Stack & Why

| Layer | Choice | What it does | Why this choice |
|---|---|---|---|
| Frontend | **Next.js (React)** | Builds the user interface: pages, forms, dashboards | File-based routing, server components, fast builds, huge ecosystem, deploys perfectly on Vercel |
| Backend | **Node.js + Express** | Receives HTTP requests, runs business logic, talks to the DB | Minimal, well known, same language (TypeScript) as the frontend, lots of middleware |
| Language | **TypeScript** | Adds static types to JavaScript | Catches bugs at compile time, shares types between frontend and backend, better autocompletion |
| Database | **PostgreSQL** | Stores all data permanently | Relational data (employees ↔ departments ↔ shifts ↔ attendance) fits tables; strong constraints (unique, foreign keys, partial indexes), transactions, great for reports with SQL aggregations |
| ORM | **Prisma** | Maps TypeScript code to SQL queries; manages migrations | Type-safe queries generated from the schema, readable schema file, automatic migrations, prevents SQL injection by parameterising queries |
| Auth | **JWT access token + refresh token** | Proves who the user is on each request | Access token is stateless and fast; refresh token (stored hashed in DB) allows long sessions and real logout/revocation |
| Authorization | **RBAC** | Decides what a logged-in user may do | Matches company roles directly; easy to audit |
| Validation | **Zod** | Checks the shape and values of incoming data | One schema gives both runtime validation and TypeScript types; can be shared with the frontend forms |
| Password hashing | **Argon2id** (bcrypt as fallback) | Turns passwords into irreversible hashes | Argon2id is the current recommended algorithm (memory-hard, resists GPU cracking) |
| Dates & timezones | **date-fns + date-fns-tz** | Converts UTC timestamps to the company's local work date | Hours and "which day was this" must be correct around midnight |
| Scheduled jobs | **node-cron** | Runs the nightly attendance close-out | Simple, runs inside the API process, no extra infrastructure |
| API style | **REST** | Resource-based HTTP endpoints | Simple, cacheable, easy to test with Postman, fits CRUD-heavy apps |
| Styling | **Tailwind CSS + shadcn/ui** | UI design | Fast to build consistent, responsive UI |
| Data fetching | **TanStack Query** | Fetches, caches and refreshes API data in React | Handles loading/error states and cache invalidation for you |
| Forms | **React Hook Form + Zod** | Form state and validation | Reuses the same Zod schemas as the backend |
| Logging | **Pino** | Structured JSON logs | Fast and easy to search in production |
| Testing | **Vitest, Supertest, Playwright** | Unit, API and end-to-end tests | Fast, TypeScript-native |
| Hosting | **Vercel** (frontend), **Railway/Render** (API + PostgreSQL) | Runs the app on the internet | Free/cheap tiers, Git-based deploys, managed database with backups |

### 4.1 Why a separate Express backend instead of only Next.js API routes?

**What:** The frontend (Next.js) and backend (Express) are two apps in one repository (monorepo).

**Why:**
- A clear separation between UI and business logic, which is easier to explain and grade in a project.
- The API can later be reused by a mobile app or a biometric/kiosk device.
- The nightly close-out job and report generation run better on a normal long-running server than on serverless functions.
- It matches the planned deployment: Vercel for UI, Railway/Render for API + DB.

---

## 5. System Architecture

### 5.1 Overall Architecture

**What:** A three-tier architecture: Client (browser) → API server → Database, plus a scheduled job inside the API.

```mermaid
flowchart LR
    U[Employee Browser<br/>desktop / mobile] -->|HTTPS| FE[Next.js Frontend<br/>Vercel]
    FE -->|REST + JSON<br/>Bearer access token| API[Express API<br/>Railway/Render]
    API -->|Prisma| DB[(PostgreSQL)]
    CRON[Nightly job<br/>node-cron] --> API
    API -->|SMTP| MAIL[Email Service<br/>password reset, approvals]
    API --> LOG[Logs / Monitoring]
```

**Why:** Each tier has one job. The browser never talks to the database directly, so all rules and security checks happen in one place: the API. The API — not the browser — decides the punch time.

### 5.2 Backend Layered (Modular) Architecture

**What:** Every request passes through fixed layers. Code is grouped by **feature module** (auth, employees, organization, attendance, corrections, leave, reports), and each module has the same layers.

```mermaid
flowchart TD
    R[Route] --> M[Middleware<br/>auth · role · validate · rateLimit]
    M --> C[Controller]
    C --> S[Service]
    S --> RP[Repository / Prisma]
    RP --> DB[(PostgreSQL)]
```

| Layer | What it does | Why it exists |
|---|---|---|
| **Route** | Maps URL + HTTP method to a controller, attaches middleware | One place to see all endpoints of a module |
| **Middleware** | Authentication, role check, Zod validation, rate limiting | Reusable checks applied before any logic runs |
| **Controller** | Reads `req`, calls the service, sends `res` | Keeps HTTP details out of business logic |
| **Service** | Business rules (open punch check, status calculation, overtime, reporting line, leave balance) | The heart of the app; easy to unit test without HTTP |
| **Repository (Prisma)** | Reads/writes the database | Isolates database code so services stay clean |

**Why modular:** When you work on "leave requests", everything is in `modules/leave/`. Adding a feature means adding a module, not editing files everywhere.

### 5.3 Frontend Architecture

**What:**
- **App Router** (`app/`) — each folder is a URL route.
- **Route groups** — `(auth)` for login pages, `(dashboard)` for logged-in pages, with per-role subfolders.
- **API client** — one `fetch` wrapper that adds the access token and automatically refreshes it on `401`.
- **TanStack Query** — caches server data per page.
- **Auth context** — keeps the current user and access token in memory.
- **Middleware/guards** — redirect users who are not logged in or have the wrong role.

**Why:** Keeping all API calls in one client means token handling is written once. Keeping the access token in memory (not `localStorage`) protects it from XSS theft.

### 5.4 Request Flow Example — "Employee clocks in"

```mermaid
sequenceDiagram
    participant E as Employee Browser
    participant FE as Next.js
    participant API as Express API
    participant DB as PostgreSQL

    E->>FE: Opens dashboard and taps Clock in
    FE->>API: POST /api/attendance/clock-in with note and location
    API->>API: Authenticate, validate with Zod, and rate limit
    API->>API: Check server time, active status, IP allowed, and no open entry
    API->>API: Calculate work date in company timezone and load shift
    API->>API: Calculate late minutes from shift start plus grace period
    API->>DB: Transaction creates TimeEntry, upserts AttendanceDay, and writes audit log
    DB-->>API: OK
    API-->>FE: 201 Created with clockInAt, lateMinutes, and status
    FE-->>E: Clocked in at 09:04 - 4 min late
```

**Why a transaction:** The time entry, the daily summary and the audit entry are saved together. If any part fails, nothing is saved, so you never get a punch without a day record.

**Why the server time:** The request body contains **no timestamp**. Whatever the phone clock says is ignored, so employees can't clock in "at 08:59" from home at 09:30.

---

## 6. Database Design

### 6.1 Design Principles

| Principle | What | Why |
|---|---|---|
| Normalisation (3NF) | Each fact is stored once | Avoids inconsistent data |
| UUID primary keys | IDs like `c0a8…` instead of 1, 2, 3 | Can't be guessed or enumerated in URLs |
| Foreign keys | Records point to real parents | DB itself blocks orphan data |
| Unique constraints | e.g. one attendance day per employee per date | Prevents duplicates even if app code has a bug |
| Soft delete for employees (`isActive`) | Employees are deactivated when they leave, not deleted | Historical attendance and payroll stay valid |
| Timestamps in UTC | `clockInAt`, `createdAt`, … stored as UTC | One unambiguous time; converted to local only for display |
| Work date as `DATE` | `AttendanceDay.workDate` stores only the local calendar date | Reports group by day without timezone math |
| Enums for fixed values | Role, AttendanceStatus, RequestStatus | Invalid values are impossible |

### 6.2 Entity Overview

| Table | What it stores | Why it exists |
|---|---|---|
| `User` | Login identity: email, password hash, role | One table for authentication of all roles |
| `Employee` | Employee code, name, department, manager, shift, join date | Every person who works (including managers and admins); 1-to-1 with User |
| `Department` | e.g. "Engineering", "Sales" | Group employees for reports |
| `Shift` | Start/end time, grace minutes, break minutes, working weekdays | Defines when an employee is expected to work |
| `Holiday` | Company non-working days | No absence is recorded on holidays |
| `TimeEntry` | One clock-in / clock-out pair | Raw punches; several per day allowed (e.g. lunch out/in) |
| `AttendanceDay` | One employee's result for one work date: status, first in, last out, worked/late/overtime minutes | Fast reports without re-calculating from punches every time |
| `CorrectionRequest` | Employee's request to fix a punch, and its review | Controlled way to fix mistakes |
| `LeaveType` | e.g. Annual, Sick, Unpaid; yearly quota; paid or not | Different rules per leave kind |
| `LeaveBalance` | Days allocated and used per employee, leave type and year | Know how much leave is left |
| `LeaveRequest` | Leave application, dates, half-day flag, status | Digital leave workflow |
| `RefreshToken` | Hashed refresh tokens | Enables logout, token rotation, revocation |
| `PasswordResetToken` | Hashed one-time reset tokens | Secure "forgot password" |
| `AuditLog` | Who changed what, when, before/after values | Accountability |

### 6.3 ER Diagram

```mermaid
erDiagram
    User ||--|| Employee : has
    User ||--o{ RefreshToken : owns
    User ||--o{ PasswordResetToken : owns
    User ||--o{ AuditLog : performs

    Department ||--o{ Employee : contains
    Shift ||--o{ Employee : "assigned to"
    Employee ||--o{ Employee : manages

    Employee ||--o{ TimeEntry : punches
    Employee ||--o{ AttendanceDay : has
    AttendanceDay ||--o{ TimeEntry : groups

    Employee ||--o{ CorrectionRequest : submits
    Employee ||--o{ LeaveRequest : submits
    Employee ||--o{ LeaveBalance : has
    LeaveType ||--o{ LeaveBalance : "counted in"
    LeaveType ||--o{ LeaveRequest : "type of"
    User ||--o{ LeaveRequest : reviews
    User ||--o{ CorrectionRequest : reviews
```

### 6.4 Key Relationships & Constraints

| Constraint | What | Why |
|---|---|---|
| `User.email` unique | No two accounts with the same email | Login identifier |
| `Employee.userId` unique | 1-to-1 with User | A user is one employee |
| `Employee.employeeCode` unique | e.g. `EMP-0042` | Matches HR/payroll records |
| `Employee.managerId` → `Employee` | Self-relation | Defines the reporting line used for approvals |
| `AttendanceDay (employeeId, workDate)` unique | One result per employee per day | **Prevents duplicate days** |
| Partial unique index on `TimeEntry(employeeId) WHERE clockOutAt IS NULL` | At most one open punch per employee | **Prevents double clock-in**, even with two taps at the same moment |
| Check `clockOutAt > clockInAt` | Clock-out after clock-in | No negative hours |
| `LeaveBalance (employeeId, leaveTypeId, year)` unique | One balance row per type per year | Correct balance math |
| `onDelete: Cascade` AttendanceDay → TimeEntry | Deleting a day deletes its punches | No orphan punches |
| `onDelete: Restrict` Employee → attendance | Can't hard-delete an employee with attendance | History and payroll are preserved |
| Index on `AttendanceDay.workDate` and `(employeeId, workDate)` | Faster lookups | Reports filter by date and employee constantly |

### 6.5 Prisma Schema

File: `backend/prisma/schema.prisma`

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ---------- Enums ----------

enum Role {
  ADMIN
  MANAGER
  EMPLOYEE
}

enum AttendanceStatus {
  PRESENT
  LATE
  HALF_DAY
  ABSENT
  ON_LEAVE
}

enum RequestStatus {
  PENDING
  APPROVED
  REJECTED
  CANCELLED
}

enum HalfDay {
  NONE
  FIRST_HALF
  SECOND_HALF
}

enum PunchSource {
  WEB
  MOBILE_WEB
  ADMIN        // entered by admin
  CORRECTION   // created by an approved correction
  SYSTEM       // auto-closed by the nightly job
}

enum AuditAction {
  CREATE
  UPDATE
  DELETE
  LOGIN
  LOGOUT
}

// ---------- Users & organisation ----------

model User {
  id           String   @id @default(uuid())
  email        String   @unique
  passwordHash String
  role         Role
  isActive     Boolean  @default(true)
  lastLoginAt  DateTime?
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  employee            Employee?
  refreshTokens       RefreshToken[]
  passwordResetTokens PasswordResetToken[]
  auditLogs           AuditLog[]
  reviewedLeaves      LeaveRequest[]      @relation("LeaveReviewer")
  reviewedCorrections CorrectionRequest[] @relation("CorrectionReviewer")
}

model Employee {
  id           String    @id @default(uuid())
  userId       String    @unique
  employeeCode String    @unique          // "EMP-0042"
  firstName    String
  lastName     String
  designation  String?                    // "Software Engineer"
  phone        String?
  joinDate     DateTime  @db.Date
  exitDate     DateTime? @db.Date
  departmentId String
  managerId    String?
  shiftId      String
  createdAt    DateTime  @default(now())
  updatedAt    DateTime  @updatedAt

  user          User       @relation(fields: [userId], references: [id], onDelete: Cascade)
  department    Department @relation(fields: [departmentId], references: [id])
  shift         Shift      @relation(fields: [shiftId], references: [id])
  manager       Employee?  @relation("ReportingLine", fields: [managerId], references: [id])
  directReports Employee[] @relation("ReportingLine")

  timeEntries    TimeEntry[]
  attendanceDays AttendanceDay[]
  corrections    CorrectionRequest[]
  leaveRequests  LeaveRequest[]
  leaveBalances  LeaveBalance[]

  @@index([departmentId])
  @@index([managerId])
}

model Department {
  id        String   @id @default(uuid())
  name      String   @unique                // "Engineering"
  createdAt DateTime @default(now())

  employees Employee[]
}

model Shift {
  id           String @id @default(uuid())
  name         String @unique               // "General 09:00–18:00"
  startTime    String                       // "09:00" (company local time)
  endTime      String                       // "18:00"; earlier than start = overnight shift
  graceMinutes Int    @default(10)          // late only after start + grace
  breakMinutes Int    @default(60)          // unpaid break, subtracted from expected hours
  workDays     Int[]                        // ISO weekdays: 1=Mon … 7=Sun, e.g. [1,2,3,4,5]

  employees Employee[]
}

model Holiday {
  id   String   @id @default(uuid())
  date DateTime @unique @db.Date
  name String
}

// ---------- Attendance ----------

model AttendanceDay {
  id              String           @id @default(uuid())
  employeeId      String
  workDate        DateTime         @db.Date
  status          AttendanceStatus
  firstClockIn    DateTime?
  lastClockOut    DateTime?
  workedMinutes   Int              @default(0)
  lateMinutes     Int              @default(0)
  overtimeMinutes Int              @default(0)
  missedPunch     Boolean          @default(false) // auto-closed by nightly job
  remarks         String?
  createdAt       DateTime         @default(now())
  updatedAt       DateTime         @updatedAt

  employee    Employee    @relation(fields: [employeeId], references: [id], onDelete: Restrict)
  timeEntries TimeEntry[]

  @@unique([employeeId, workDate])
  @@index([workDate])
}

model TimeEntry {
  id              String      @id @default(uuid())
  employeeId      String
  attendanceDayId String
  clockInAt       DateTime                  // UTC, server time
  clockOutAt      DateTime?                 // null = currently clocked in
  inSource        PunchSource
  outSource       PunchSource?
  inIp            String?
  outIp           String?
  inLatitude      Float?
  inLongitude     Float?
  note            String?
  createdAt       DateTime    @default(now())
  updatedAt       DateTime    @updatedAt

  employee      Employee      @relation(fields: [employeeId], references: [id], onDelete: Restrict)
  attendanceDay AttendanceDay @relation(fields: [attendanceDayId], references: [id], onDelete: Cascade)

  @@index([employeeId, clockInAt])
  // + partial unique index "one open entry per employee" added in SQL migration (see below)
}

model CorrectionRequest {
  id                 String        @id @default(uuid())
  employeeId         String
  workDate           DateTime      @db.Date
  requestedClockIn   DateTime?
  requestedClockOut  DateTime?
  reason             String
  status             RequestStatus @default(PENDING)
  reviewedById       String?
  reviewNote         String?
  reviewedAt         DateTime?
  createdAt          DateTime      @default(now())

  employee   Employee @relation(fields: [employeeId], references: [id])
  reviewedBy User?    @relation("CorrectionReviewer", fields: [reviewedById], references: [id])

  @@index([employeeId, status])
}

// ---------- Leave ----------

model LeaveType {
  id          String  @id @default(uuid())
  code        String  @unique               // "ANNUAL", "SICK", "UNPAID"
  name        String
  yearlyQuota Decimal @db.Decimal(4, 1)     // 20.0 days; 0 = unlimited/unpaid
  isPaid      Boolean @default(true)

  balances LeaveBalance[]
  requests LeaveRequest[]
}

model LeaveBalance {
  id          String  @id @default(uuid())
  employeeId  String
  leaveTypeId String
  year        Int
  allocated   Decimal @db.Decimal(4, 1)
  used        Decimal @db.Decimal(4, 1) @default(0)

  employee  Employee  @relation(fields: [employeeId], references: [id])
  leaveType LeaveType @relation(fields: [leaveTypeId], references: [id])

  @@unique([employeeId, leaveTypeId, year])
}

model LeaveRequest {
  id           String        @id @default(uuid())
  employeeId   String
  leaveTypeId  String
  fromDate     DateTime      @db.Date
  toDate       DateTime      @db.Date
  halfDay      HalfDay       @default(NONE)  // only when fromDate = toDate
  days         Decimal       @db.Decimal(4, 1) // working days, excluding weekends/holidays
  reason       String
  status       RequestStatus @default(PENDING)
  reviewedById String?
  reviewNote   String?
  reviewedAt   DateTime?
  createdAt    DateTime      @default(now())

  employee   Employee  @relation(fields: [employeeId], references: [id])
  leaveType  LeaveType @relation(fields: [leaveTypeId], references: [id])
  reviewedBy User?     @relation("LeaveReviewer", fields: [reviewedById], references: [id])

  @@index([employeeId, status])
}

// ---------- Auth support ----------

model RefreshToken {
  id         String    @id @default(uuid())
  userId     String
  tokenHash  String    @unique
  familyId   String               // for reuse detection
  expiresAt  DateTime
  revokedAt  DateTime?
  createdAt  DateTime  @default(now())
  userAgent  String?
  ipAddress  String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
}

model PasswordResetToken {
  id        String    @id @default(uuid())
  userId    String
  tokenHash String    @unique
  expiresAt DateTime
  usedAt    DateTime?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
}

// ---------- Audit ----------

model AuditLog {
  id         String      @id @default(uuid())
  actorId    String?                   // null = system (nightly job)
  action     AuditAction
  entity     String                    // "TimeEntry", "LeaveRequest"
  entityId   String
  before     Json?
  after      Json?
  reason     String?
  ipAddress  String?
  createdAt  DateTime    @default(now())

  actor User? @relation(fields: [actorId], references: [id])

  @@index([entity, entityId])
  @@index([actorId])
}
```

Extra SQL added to the first migration (Prisma can't express these in the schema file):

```sql
-- At most one open punch per employee → prevents double clock-in, even under race conditions
CREATE UNIQUE INDEX "TimeEntry_one_open_per_employee"
  ON "TimeEntry" ("employeeId") WHERE "clockOutAt" IS NULL;

-- Clock-out must be after clock-in
ALTER TABLE "TimeEntry"
  ADD CONSTRAINT "TimeEntry_out_after_in" CHECK ("clockOutAt" IS NULL OR "clockOutAt" > "clockInAt");
```

**Why these specific choices:**
- **`TimeEntry` + `AttendanceDay`** — punches are the raw facts; the day row is a calculated summary. Reports read the summary (fast), while the punches remain as evidence for disputes.
- **Several `TimeEntry` rows per day** — supports going out for lunch or an errand; worked time is the sum of all entries.
- **`workDate` as `@db.Date`, punch times as UTC `DateTime`** — hours are computed from exact instants; the day is the local calendar date in the company timezone, so a clock-in at 00:30 local time doesn't become "yesterday" in UTC.
- **Shift times as `"HH:mm"` strings** — they are local wall-clock times, not instants; they are combined with the work date in the company timezone when needed.
- **`Decimal` for leave days** — half-days (0.5) must add up exactly; floats would give 9.999999.
- **`familyId` on RefreshToken** — every rotated token in one login chain shares a family; if an old one is reused, the whole family is revoked (see §7.5).
- **Hashed tokens** — if the database leaks, the attacker can't use stored refresh/reset tokens.
- **`before`/`after` JSON in AuditLog** — shows exactly what changed without a separate history table for each entity.

---

## 7. Authentication & Authorization

### 7.1 Concepts

| Term | What | Why |
|---|---|---|
| **Authentication** | Proving *who* you are (login) | Without it, anyone could clock in as anyone |
| **Authorization** | Deciding *what* you may do | An employee must not edit their own hours |
| **Access token (JWT)** | Short-lived (15 min) signed token sent in `Authorization: Bearer …` | Fast: the API verifies it without a DB lookup |
| **Refresh token** | Long-lived (7 days) random string in an `httpOnly` cookie, stored hashed in DB | Gets new access tokens without re-login; can be revoked |

**Why two tokens:** A JWT can't be "cancelled" before it expires. Keeping it short (15 min) limits damage if it's stolen. The refresh token lives in the database, so logout, "log out all devices" and deactivating a leaving employee really work.

### 7.2 Registration / User Creation

**What:** There is **no public sign-up**. HR (admin) creates employees (single or CSV bulk import). A temporary password is emailed, and the employee must change it at first login. When someone leaves the company, HR deactivates the account, which also revokes all of their refresh tokens.

**Why:** Only real employees should have accounts. Public registration would let anyone create an account and appear on the timesheet.

### 7.3 Login Flow

```mermaid
sequenceDiagram
    participant B as Browser
    participant API
    participant DB

    B->>API: POST /api/auth/login {email, password}
    API->>DB: Find user by email
    API->>API: argon2.verify(hash, password)
    alt invalid or inactive
        API-->>B: 401 "Invalid email or password"
    else valid
        API->>API: Sign access JWT {sub, role, employeeId} (15 min)
        API->>API: Generate random refresh token
        API->>DB: Store sha256(refresh token), expiresAt, familyId
        API-->>B: 200 {accessToken, user} + Set-Cookie refreshToken (httpOnly, Secure, SameSite=Strict)
    end
```

**Why the same error for "wrong email" and "wrong password":** It stops attackers from discovering which emails are registered.

**Why `employeeId` in the token:** Almost every request (clock-in, my attendance, my leave) needs it; putting it in the token saves a database lookup per request.

### 7.4 Password Hashing

**What:** `argon2id` with library defaults (memory ≈ 19 MB, iterations ≥ 2).

```ts
import argon2 from "argon2";
export const hashPassword = (pw: string) => argon2.hash(pw, { type: argon2.argon2id });
export const verifyPassword = (hash: string, pw: string) => argon2.verify(hash, pw);
```

**Why:** Hashing is one-way; even admins can't read passwords. Argon2id is slow and memory-heavy on purpose, which makes brute-force cracking expensive. Password rule: at least 8 characters, checked by Zod.

### 7.5 Refresh Token Rotation & Reuse Detection

**What:**
1. When the access token expires, the frontend calls `POST /api/auth/refresh` (cookie sent automatically).
2. The API finds the token hash, checks it is not expired or revoked, and that the user is still active.
3. It **revokes the old token** and issues a **new** refresh token (same `familyId`) and a new access token.
4. If a token that was **already revoked** is presented, someone copied it → the API revokes the **whole family** and forces re-login.

**Why:** Rotation means a stolen refresh token only works once, and reuse detection turns theft into an automatic lockout.

### 7.6 Authentication Middleware

```ts
// backend/src/middleware/authenticate.ts
import { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";
import { env } from "../config/env";
import { AppError } from "../utils/AppError";

export function authenticate(req: Request, _res: Response, next: NextFunction) {
  const header = req.headers.authorization;
  if (!header?.startsWith("Bearer ")) throw new AppError(401, "UNAUTHENTICATED", "Missing token");

  try {
    const payload = jwt.verify(header.slice(7), env.JWT_ACCESS_SECRET) as {
      sub: string; role: Role; employeeId: string;
    };
    req.user = { id: payload.sub, role: payload.role, employeeId: payload.employeeId };
    next();
  } catch {
    throw new AppError(401, "TOKEN_INVALID", "Invalid or expired token");
  }
}
```

**What it does:** Reads the Bearer token, verifies its signature and expiry, and attaches `req.user` for later layers.

### 7.7 Role Middleware (RBAC)

```ts
// backend/src/middleware/requireRole.ts
export const requireRole = (...roles: Role[]) =>
  (req: Request, _res: Response, next: NextFunction) => {
    if (!req.user || !roles.includes(req.user.role)) {
      throw new AppError(403, "FORBIDDEN", "You do not have permission for this action");
    }
    next();
  };

// usage
router.patch("/leave/:id/review", authenticate, requireRole("MANAGER", "ADMIN"), validate(reviewSchema), controller.review);
```

**What it does:** Blocks the request with `403` if the user's role isn't in the allowed list.
**Why 401 vs 403:** `401` = "we don't know who you are"; `403` = "we know who you are, but you're not allowed".

### 7.8 Ownership & Reporting-Line Checks

**What:** Inside services:
- `assertCanManage(actor, employeeId)` — passes if the actor is an admin, or is the employee's direct manager (`employee.managerId === actor.employeeId`).
- `assertNotSelf(actor, request)` — nobody reviews their own leave or correction.
- For "my" endpoints (`/me`, clock-in, clock-out), the employee ID is **always** taken from `req.user.employeeId`, never from the URL or body.

**Why:** Role checks alone would let any manager approve any team's leave. Reporting-line checks close that gap and prevent "IDOR" attacks (changing an ID in the URL to see someone else's hours or salary-relevant data).

### 7.9 Protected Routes (Frontend)

**What:** `middleware.ts` in Next.js checks for the refresh cookie and redirects to `/login` if missing. Each role's layout checks `user.role` and redirects to that user's own dashboard if it doesn't match.

**Why:** Better user experience (no flash of forbidden pages). **Note:** frontend guards are for UX only — the real security is always the backend check.

### 7.10 Logout

**What:** `POST /api/auth/logout` revokes the current refresh token and clears the cookie. `POST /api/auth/logout-all` revokes every refresh token of the user.

**Why:** Needed on shared computers and if a device is lost.

### 7.11 Password Reset

1. `POST /api/auth/forgot-password {email}` → always returns `200` (don't reveal whether the email exists).
2. If the user exists and is active, generate a random token, store its **hash** with 30-minute expiry, email a link `https://app/reset-password?token=…`.
3. `POST /api/auth/reset-password {token, newPassword}` → verify hash, not expired, not used → set new password, mark token used, **revoke all refresh tokens**.

**Why revoke sessions:** If the password was reset because the account was compromised, the attacker's sessions must end too.

---

## 8. Attendance System & Rules

### 8.1 Key Time Concepts

| Term | Meaning | Example (shift 09:00–18:00, grace 10, break 60) |
|---|---|---|
| **Work date** | Local calendar date (company timezone) the work belongs to | Clock-in 2026-09-25 09:04 JST → work date 2026-09-25 |
| **Expected minutes** | Shift length − break | 9h − 1h = **480 min** |
| **Worked minutes** | Sum of all (clock-out − clock-in) for the day, minus the break if the day is longer than 6h | 09:04–18:30 = 566 − 60 = **506 min** |
| **Late minutes** | First clock-in − shift start, only if after start + grace | 09:04 → within grace → **0**; 09:25 → **25** |
| **Overtime minutes** | Worked − expected, if positive | 506 − 480 = **26 min** |

**Company timezone:** One setting (`COMPANY_TIMEZONE`, e.g. `Asia/Tokyo`). Every timestamp is stored in UTC and converted with `date-fns-tz` for the work date and shift comparison.

**Overnight shifts** (e.g. 22:00–06:00): the work date is the date the shift **starts**. A clock-in within 4 hours before shift start or before shift end belongs to that shift.

### 8.2 Daily Attendance Status

| Status | Rule | Counts as present? | Why |
|---|---|---|---|
| `PRESENT` | Clocked in on or before shift start + grace, and worked ≥ half-day threshold | ✅ Yes | Normal case |
| `LATE` | First clock-in after shift start + grace, and worked ≥ half-day threshold | ✅ Yes (tracked separately) | Employee worked, but lateness is recorded for HR |
| `HALF_DAY` | Worked less than the half-day threshold (default 4h), or has an approved half-day leave | ½ day | Partial attendance |
| `ABSENT` | Working day, no punches, no approved leave | ❌ No | Unexcused absence (usually unpaid) |
| `ON_LEAVE` | Approved full-day leave | Excluded from total | Employee shouldn't be penalised for approved leave |

Weekends (days not in `shift.workDays`) and holidays get **no** `AttendanceDay` row unless the employee actually works; if they do, all worked minutes count as **overtime**.

**Status is recalculated** every time punches change (clock-out, correction, admin edit) by one function, `recalculateDay(employeeId, workDate)`, so the rule lives in one place.

### 8.3 Attendance Rate & Flags

```
working days   = shift work days in period − holidays − ON_LEAVE days
present days   = PRESENT + LATE + 0.5 × HALF_DAY
attendance %   = present days / working days × 100
punctuality %  = PRESENT / (PRESENT + LATE) × 100
```

**Example:** September, 22 working days, 1 day of leave → 21 countable: 17 Present, 2 Late, 1 Half-day, 1 Absent
→ present days = 19.5, attendance = **92.9%**, punctuality = **89.5%**

**Flags (configurable):**
- More than **3 late arrivals** in a month → flagged "Frequent lateness".
- Any **missed punch** not yet corrected → shown to the employee and the manager.
- Attendance below **90%** in a month → flagged in reports.

**Why exclude leave:** Approved leave is a right, not an absence; counting it would unfairly lower the rate.

### 8.4 Clock-In / Clock-Out Rules (Service Layer)

| Rule | Why |
|---|---|
| Time is always `new Date()` on the server; the request has no time field | Employees can't fake the time |
| Account must be active and the employee not past `exitDate` | Leavers can't punch |
| Clock-in is rejected if there is already an open entry → `409 ALREADY_CLOCKED_IN` | **Prevents double clock-in**; backed by the partial unique index |
| Clock-out is rejected if there is no open entry → `409 NOT_CLOCKED_IN` | Can't finish what wasn't started |
| Optional: request IP must be in `OFFICE_IP_ALLOWLIST` | Clock-in only from the office network (off when empty, e.g. for remote staff) |
| Optional: browser location within `GEOFENCE_RADIUS_M` of the office | Stops clocking in from home; location is stored only with the punch |
| Clock-in on a day with approved full-day leave → allowed, but the leave is flagged for review | Employee came in anyway; HR decides whether to restore the leave day |
| Rate limit: 10 punch requests per minute per user | Stops scripts and accidental double taps |

**Why check "already clocked in" twice (service + DB index):** The service gives a friendly error message; the partial unique index is the final guarantee even when two taps arrive at the same moment (race condition). Prisma error `P2002` is converted to `409`.

### 8.5 Nightly Close-Out Job

**What:** A `node-cron` job runs at **02:00 company time** for the previous work date:

1. **Auto-close open entries** — any entry still open more than `AUTO_CLOSE_AFTER_HOURS` (default 14h) after clock-in is closed at **shift end time**, source `SYSTEM`, and the day is marked `missedPunch = true`. The employee is emailed to submit a correction.
2. **Mark absences** — for every active employee whose shift includes that weekday, which is not a holiday, and who has no `AttendanceDay` and no approved leave → create an `ABSENT` day.
3. **Mark leave** — employees with approved leave on that date get an `ON_LEAVE` (or `HALF_DAY`) day.
4. Write one `AuditLog` row per change with `actorId = null` (system).

The job is **idempotent**: running it twice for the same date changes nothing the second time (it uses `upsert` on `(employeeId, workDate)`).

**Why:** Without it, absent employees would simply have "no data", and reports would count missing rows as neither present nor absent. Idempotency means a crashed or repeated run is safe.

### 8.6 Corrections (Forgot to Clock In/Out)

**What:** Employees **cannot edit their own time**. Instead they submit a `CorrectionRequest` with the date, the correct clock-in and/or clock-out time, and a reason. It must be submitted within **7 days** (`CORRECTION_WINDOW_DAYS`).

```mermaid
stateDiagram-v2
    [*] --> PENDING : Employee submits
    PENDING --> APPROVED : Manager/Admin approves
    PENDING --> REJECTED : Manager/Admin rejects
    PENDING --> CANCELLED : Employee cancels
    APPROVED --> [*]
    REJECTED --> [*]
    CANCELLED --> [*]
```

On **APPROVED** (in one transaction): the time entries for that day are created/updated with source `CORRECTION`, `recalculateDay()` runs, `missedPunch` is cleared, and the audit log stores the before/after values.

**Admin direct edit:** Admins can edit any punch at any time but must give a `reason`, which is stored in the audit log.

**Why:** Time directly affects pay. A second person approving every change stops employees from adding hours to their own timesheet, and the audit log shows exactly who changed what.

### 8.7 Leave Management

**Leave types** (seeded, editable by admin):

| Code | Name | Default yearly quota | Paid |
|---|---|---|---|
| `ANNUAL` | Annual / paid leave | 20 days | ✅ |
| `SICK` | Sick leave | 10 days | ✅ |
| `UNPAID` | Unpaid leave | Unlimited | ❌ |

**Balances:** At the start of each year (or on the join date, pro-rated), a `LeaveBalance` row is created per leave type. `remaining = allocated − used`.

**Request rules:**
- `days` is calculated by the server: working days between `fromDate` and `toDate`, excluding weekends and holidays; a half-day request counts as 0.5.
- Rejected if `days` > remaining balance (except unpaid leave).
- Rejected if it overlaps another pending or approved leave of the same employee.
- Can be submitted for past dates (e.g. sick leave) up to 7 days back.

Workflow is the same as corrections: `PENDING → APPROVED / REJECTED / CANCELLED`. An employee can also cancel an **approved** leave that hasn't started yet; the days are returned to the balance.

On **APPROVED** (in one transaction):
- `LeaveBalance.used += days`.
- Existing `ABSENT` days in the range become `ON_LEAVE` (or `HALF_DAY` for half-day leave), audit logged.
- Future days in the range will be created as `ON_LEAVE` by the nightly job.

**Why:** Leave is often approved after the absence (e.g. sick leave), so existing records must update automatically, and the balance must change in the same transaction so it can never drift from the approved requests.

### 8.8 Attendance History

**What:** Employees see a monthly calendar coloured by status, with first in / last out and hours per day. Managers see a team × date grid. Admins can filter by any department, employee or date range.

**Why:** Different users need different views of the same data.

### 8.9 Reports

| Report | What it shows | Who | Why |
|---|---|---|---|
| Daily report ("Who's in") | Per department: present, late, absent, on leave, not yet clocked in | Admin, Manager (team) | Know today's staffing |
| Monthly timesheet | Per employee: days present/late/half/absent/leave, total worked hours, overtime hours | Admin, Manager (team) | **Payroll input** |
| Employee report | One employee's full history, hours, leave taken and balance | All (employee: own only) | Reviews, self-check |
| Late arrivals | Employees and dates with late minutes, monthly count | Admin, Manager | Punctuality follow-up |
| Overtime report | Overtime hours per employee and department | Admin, Manager | Overtime approval and cost |
| Leave report | Leave taken and remaining per employee and type | Admin, Manager | Planning and year-end |
| Missed punches | Days auto-closed and still uncorrected | Admin, Manager | Clean data before payroll |
| CSV export | Any of the above as a `.csv` file | Admin, Manager (team) | Import into payroll / Excel |

**Why compute in SQL:** Totals use `GROUP BY` with `COUNT(*) FILTER (WHERE …)` and `SUM` in PostgreSQL. Doing the math in the database is far faster than loading thousands of rows into Node.

```sql
-- Monthly timesheet
SELECT e."employeeCode",
       e."firstName" || ' ' || e."lastName"                          AS name,
       COUNT(*) FILTER (WHERE d.status = 'PRESENT')                  AS present,
       COUNT(*) FILTER (WHERE d.status = 'LATE')                     AS late,
       COUNT(*) FILTER (WHERE d.status = 'HALF_DAY')                 AS half_day,
       COUNT(*) FILTER (WHERE d.status = 'ABSENT')                   AS absent,
       COUNT(*) FILTER (WHERE d.status = 'ON_LEAVE')                 AS on_leave,
       ROUND(SUM(d."workedMinutes")   / 60.0, 2)                     AS worked_hours,
       ROUND(SUM(d."overtimeMinutes") / 60.0, 2)                     AS overtime_hours
FROM "AttendanceDay" d
JOIN "Employee" e ON e.id = d."employeeId"
WHERE d."workDate" BETWEEN $1 AND $2
GROUP BY e.id
ORDER BY e."employeeCode";
```

---

## 9. API Documentation

### 9.1 Conventions

| Convention | What | Why |
|---|---|---|
| Base URL | `/api` (versioned later as `/api/v1`) | Clear separation from frontend routes |
| Format | JSON, `camelCase` fields | Matches TypeScript |
| Auth | `Authorization: Bearer <accessToken>` | Standard |
| Dates | `YYYY-MM-DD` for dates, ISO 8601 UTC for timestamps | Unambiguous; the frontend converts to local time for display |
| Pagination | `?page=1&limit=20` → response has `meta: { page, limit, total }` | Don't send thousands of rows at once |
| Filtering | Query params, e.g. `?departmentId=…&from=…&to=…` | Simple and cacheable |
| Success envelope | `{ "data": …, "meta": … }` | Frontend always knows where to look |
| Error envelope | `{ "error": { "code", "message", "details" } }` | Consistent error handling (§13) |

### 9.2 Status Codes

| Code | Meaning | When |
|---|---|---|
| 200 | OK | Successful read/update |
| 201 | Created | Resource created (e.g. clock-in) |
| 204 | No Content | Successful delete/logout |
| 400 | Bad Request | Validation failed |
| 401 | Unauthorized | Missing/invalid/expired token |
| 403 | Forbidden | Wrong role, not the manager, IP not allowed |
| 404 | Not Found | Resource doesn't exist (or user may not know it exists) |
| 409 | Conflict | Already clocked in / not clocked in / duplicate |
| 422 | Unprocessable | Business rule broken (e.g. insufficient leave balance, correction window passed) |
| 429 | Too Many Requests | Rate limit hit |
| 500 | Server Error | Unexpected bug |

### 9.3 Endpoint List

**Auth**

| Method | Endpoint | Roles | Description |
|---|---|---|---|
| POST | `/api/auth/login` | Public | Log in |
| POST | `/api/auth/refresh` | Public (cookie) | Get a new access token |
| POST | `/api/auth/logout` | Any | Revoke current refresh token |
| POST | `/api/auth/logout-all` | Any | Revoke all refresh tokens |
| POST | `/api/auth/forgot-password` | Public | Send reset email |
| POST | `/api/auth/reset-password` | Public | Set new password with token |
| POST | `/api/auth/change-password` | Any | Change password when logged in |
| GET | `/api/auth/me` | Any | Current user + employee profile |

**Employees**

| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET | `/api/employees?departmentId=&search=&page=` | Admin, Manager (direct reports) | List employees |
| POST | `/api/employees` | Admin | Create employee (+ user account) |
| POST | `/api/employees/import` | Admin | Bulk import from CSV |
| GET | `/api/employees/:id` | Admin, Manager (direct reports) | Employee detail |
| PATCH | `/api/employees/:id` | Admin | Update (department, manager, shift, …) |
| PATCH | `/api/employees/:id/status` | Admin | Activate/deactivate (revokes tokens) |

**Organisation**

| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET/POST/PATCH | `/api/departments` | Admin (GET: all) | Departments |
| GET/POST/PATCH | `/api/shifts` | Admin (GET: all) | Shifts |
| GET/POST/DELETE | `/api/holidays` | Admin (GET: all) | Holidays |
| GET/POST/PATCH | `/api/leave-types` | Admin (GET: all) | Leave types |

**Attendance**

| Method | Endpoint | Roles | Description |
|---|---|---|---|
| POST | `/api/attendance/clock-in` | Any | Clock in (server time) |
| POST | `/api/attendance/clock-out` | Any | Clock out (server time) |
| GET | `/api/attendance/today` | Any | Own current state: clocked in?, since when, today's hours |
| GET | `/api/attendance/me?from=&to=` | Any | Own attendance days + summary |
| GET | `/api/attendance?employeeId=&departmentId=&from=&to=` | Admin, Manager (team) | Attendance days of others |
| GET | `/api/attendance/days/:id` | Admin, Manager (team), owner | One day with its time entries |
| PATCH | `/api/attendance/days/:id` | Admin | Edit punches/status directly (reason required) |

**Corrections**

| Method | Endpoint | Roles | Description |
|---|---|---|---|
| POST | `/api/corrections` | Any | Submit correction request |
| GET | `/api/corrections?status=` | Own; Manager (team); Admin (all) | List |
| PATCH | `/api/corrections/:id/review` | Manager (team), Admin | Approve / reject |
| PATCH | `/api/corrections/:id/cancel` | Owner (pending only) | Cancel |

**Leave**

| Method | Endpoint | Roles | Description |
|---|---|---|---|
| POST | `/api/leave` | Any | Submit leave request |
| GET | `/api/leave?status=` | Own; Manager (team); Admin (all) | List |
| GET | `/api/leave/balance/me?year=` | Any | Own leave balances |
| GET | `/api/leave/balance?employeeId=&year=` | Admin, Manager (team) | Others' balances |
| PATCH | `/api/leave/balance/:id` | Admin | Adjust allocation (reason required) |
| PATCH | `/api/leave/:id/review` | Manager (team), Admin | Approve / reject |
| PATCH | `/api/leave/:id/cancel` | Owner (pending, or approved and not started) | Cancel |

**Reports & dashboard**

| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET | `/api/reports/daily?date=&departmentId=` | Admin, Manager | Who's in today |
| GET | `/api/reports/timesheet?month=2026-09&departmentId=` | Admin, Manager | Monthly timesheet |
| GET | `/api/reports/employee/:employeeId?from=&to=` | Admin, Manager (team), owner | Employee report |
| GET | `/api/reports/late?month=` | Admin, Manager | Late arrivals |
| GET | `/api/reports/overtime?month=` | Admin, Manager | Overtime |
| GET | `/api/reports/leave?year=` | Admin, Manager | Leave taken / remaining |
| GET | `/api/reports/missed-punches?from=&to=` | Admin, Manager | Uncorrected missed punches |
| GET | `/api/reports/export?type=timesheet&format=csv&…` | Admin, Manager (team) | CSV export |
| GET | `/api/dashboard` | Any | Role-specific summary |
| GET | `/api/audit-logs?entity=&actorId=` | Admin | Audit log |

### 9.4 Request / Response Examples

#### `POST /api/auth/login`

Request:
```json
{ "email": "aiko.tanaka@company.com", "password": "S3cure!pass" }
```
Response `200` (+ `Set-Cookie: refreshToken=…; HttpOnly; Secure; SameSite=Strict; Path=/api/auth`):
```json
{
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "user": {
      "id": "5b1c…", "email": "aiko.tanaka@company.com", "role": "EMPLOYEE",
      "employee": { "id": "e7a2…", "employeeCode": "EMP-0042", "firstName": "Aiko", "lastName": "Tanaka", "department": "Engineering" }
    }
  }
}
```
Response `401`:
```json
{ "error": { "code": "INVALID_CREDENTIALS", "message": "Invalid email or password" } }
```

#### `POST /api/attendance/clock-in`

Request (all fields optional — note there is **no time field**):
```json
{ "note": "Working from Osaka office", "latitude": 34.7025, "longitude": 135.4959 }
```
Zod schema:
```ts
export const clockSchema = z.object({
  note: z.string().max(200).optional(),
  latitude: z.number().min(-90).max(90).optional(),
  longitude: z.number().min(-180).max(180).optional(),
}).strict();   // unknown fields (e.g. "clockInAt") are rejected
```
Response `201`:
```json
{
  "data": {
    "timeEntryId": "c9d0…",
    "workDate": "2026-09-25",
    "clockInAt": "2026-09-25T00:25:12Z",
    "status": "LATE",
    "lateMinutes": 25,
    "shift": { "start": "09:00", "end": "18:00" }
  }
}
```
Response `409`:
```json
{
  "error": {
    "code": "ALREADY_CLOCKED_IN",
    "message": "You are already clocked in since 09:25",
    "details": { "timeEntryId": "c9d0…", "clockInAt": "2026-09-25T00:25:12Z" }
  }
}
```
Response `403` (IP restriction on):
```json
{ "error": { "code": "LOCATION_NOT_ALLOWED", "message": "Clock-in is only allowed from the office network" } }
```

#### `POST /api/attendance/clock-out`

Request:
```json
{}
```
Response `200`:
```json
{
  "data": {
    "timeEntryId": "c9d0…",
    "clockInAt": "2026-09-25T00:25:12Z",
    "clockOutAt": "2026-09-25T09:40:03Z",
    "day": { "status": "LATE", "workedMinutes": 495, "lateMinutes": 25, "overtimeMinutes": 15 }
  }
}
```

#### `GET /api/attendance/me?from=2026-09-01&to=2026-09-30`

Response `200`:
```json
{
  "data": {
    "summary": {
      "workingDays": 21, "present": 17, "late": 2, "halfDay": 1, "absent": 1, "onLeave": 1,
      "attendanceRate": 92.9, "punctualityRate": 89.5,
      "workedHours": 158.5, "overtimeHours": 6.25, "missedPunches": 0
    },
    "days": [
      { "workDate": "2026-09-25", "status": "LATE", "firstClockIn": "2026-09-25T00:25:12Z",
        "lastClockOut": "2026-09-25T09:40:03Z", "workedMinutes": 495, "lateMinutes": 25, "overtimeMinutes": 15 }
    ]
  },
  "meta": { "page": 1, "limit": 31, "total": 21 }
}
```

#### `POST /api/corrections`

Request:
```json
{ "workDate": "2026-09-24", "requestedClockOut": "2026-09-24T09:15:00Z", "reason": "Forgot to clock out, left at 18:15" }
```
Response `201`:
```json
{ "data": { "id": "a3…", "status": "PENDING", "workDate": "2026-09-24" } }
```
Response `422` (too old):
```json
{ "error": { "code": "CORRECTION_WINDOW_EXPIRED", "message": "Corrections must be requested within 7 days. Contact HR." } }
```

#### `POST /api/leave`

Request:
```json
{ "leaveTypeId": "lt-annual…", "fromDate": "2026-10-01", "toDate": "2026-10-03", "halfDay": "NONE", "reason": "Family trip" }
```
Response `201`:
```json
{ "data": { "id": "f1…", "status": "PENDING", "days": 3.0, "remainingAfterApproval": 12.0 } }
```
Response `422`:
```json
{ "error": { "code": "INSUFFICIENT_LEAVE_BALANCE", "message": "You have 2.0 days of Annual leave left, but requested 3.0" } }
```

#### `PATCH /api/leave/:id/review`

Request:
```json
{ "decision": "APPROVED", "note": "Enjoy your trip" }
```
Response `200`:
```json
{ "data": { "id": "f1…", "status": "APPROVED", "balance": { "allocated": 20.0, "used": 8.0, "remaining": 12.0 }, "updatedDays": 0 } }
```
Response `403` (reviewing your own request or someone outside your team):
```json
{ "error": { "code": "FORBIDDEN", "message": "You can only review requests from your direct reports" } }
```

---

## 10. Project Folder Structure

**What:** A monorepo with two apps and a shared package.

```
attendance-management/
├── frontend/                      # Next.js app (deploys to Vercel)
│   ├── src/
│   │   ├── app/                   # Routes (App Router)
│   │   │   ├── (auth)/
│   │   │   │   ├── login/page.tsx
│   │   │   │   ├── forgot-password/page.tsx
│   │   │   │   └── reset-password/page.tsx
│   │   │   ├── (dashboard)/
│   │   │   │   ├── layout.tsx     # sidebar + role guard
│   │   │   │   ├── admin/         # employees, departments, shifts, holidays, leave types, reports, audit
│   │   │   │   ├── manager/       # team attendance, approvals, team reports
│   │   │   │   └── me/            # clock in/out, my attendance, my leave, my corrections
│   │   │   ├── layout.tsx
│   │   │   └── page.tsx           # redirects to role dashboard
│   │   ├── components/
│   │   │   ├── ui/                # buttons, inputs, tables (shadcn)
│   │   │   ├── attendance/        # ClockButton, AttendanceCalendar, TeamGrid
│   │   │   ├── charts/
│   │   │   └── layout/            # Sidebar, Header
│   │   ├── features/              # hooks per feature (useClockIn, useLeaveBalance…)
│   │   ├── lib/
│   │   │   ├── api-client.ts      # fetch wrapper + auto refresh
│   │   │   ├── auth-context.tsx
│   │   │   └── query-client.ts
│   │   ├── types/
│   │   └── utils/
│   ├── middleware.ts              # redirect if not logged in
│   ├── public/
│   ├── .env.local
│   └── package.json
│
├── backend/                       # Express API (deploys to Railway/Render)
│   ├── src/
│   │   ├── modules/
│   │   │   ├── auth/
│   │   │   │   ├── auth.routes.ts
│   │   │   │   ├── auth.controller.ts
│   │   │   │   ├── auth.service.ts
│   │   │   │   ├── auth.schema.ts   # Zod
│   │   │   │   └── auth.test.ts
│   │   │   ├── employees/
│   │   │   ├── organization/      # departments, shifts, holidays, leave types
│   │   │   ├── attendance/        # clock-in/out, days, recalculateDay()
│   │   │   ├── corrections/
│   │   │   ├── leave/             # requests + balances
│   │   │   ├── reports/
│   │   │   └── audit/
│   │   ├── jobs/
│   │   │   ├── scheduler.ts       # registers node-cron jobs
│   │   │   ├── closeOutDay.ts     # nightly: auto-close, absences, leave days
│   │   │   └── yearlyLeaveBalances.ts
│   │   ├── middleware/
│   │   │   ├── authenticate.ts
│   │   │   ├── requireRole.ts
│   │   │   ├── validate.ts
│   │   │   ├── rateLimit.ts
│   │   │   ├── officeNetwork.ts   # optional IP allowlist for punches
│   │   │   └── errorHandler.ts
│   │   ├── lib/
│   │   │   ├── prisma.ts          # single PrismaClient
│   │   │   ├── logger.ts          # pino
│   │   │   ├── mailer.ts
│   │   │   └── tokens.ts          # sign/verify/hash
│   │   ├── config/
│   │   │   └── env.ts             # Zod-validated env vars
│   │   ├── utils/
│   │   │   ├── AppError.ts
│   │   │   ├── time.ts            # work date, shift window, minutes math (company timezone)
│   │   │   └── workingDays.ts     # counts working days excluding weekends/holidays
│   │   ├── types/express.d.ts     # adds req.user
│   │   ├── app.ts                 # builds the Express app
│   │   └── server.ts              # starts listening + scheduler
│   ├── prisma/
│   │   ├── schema.prisma
│   │   ├── migrations/
│   │   └── seed.ts
│   ├── tests/
│   │   ├── integration/
│   │   └── helpers/
│   ├── .env
│   └── package.json
│
├── packages/
│   └── shared/                    # Zod schemas + types used by both apps
│
├── e2e/                           # Playwright tests
├── .github/workflows/ci.yml
├── docker-compose.yml             # local PostgreSQL
├── .gitignore
└── README.md
```

**Why each part:**

| Folder / file | Why |
|---|---|
| `frontend/` + `backend/` | Separate deployments and responsibilities |
| `packages/shared/` | One Zod schema validates both the form and the API → they never disagree |
| `modules/<feature>/` | All code of a feature in one place (routes → controller → service → schema → test) |
| `jobs/` | Scheduled work kept apart from request handling; each job can be run by hand for a given date |
| `utils/time.ts` | All timezone and minutes math in one tested file — the most bug-prone part of the system |
| `middleware/` | Cross-cutting checks written once |
| `lib/prisma.ts` | One shared DB client; creating many clients exhausts DB connections |
| `config/env.ts` | App refuses to start if an env var is missing/wrong, instead of failing later |
| `app.ts` vs `server.ts` | Tests import `app` without opening a port or starting cron jobs |
| `prisma/seed.ts` | Creates demo admin, departments, shifts, leave types and employees |
| `docker-compose.yml` | Everyone gets the same PostgreSQL with one command |
| `.github/workflows/ci.yml` | Runs lint, type-check and tests on every push |

---

## 11. UI Pages

| Page | Route | Role | What it does | Why |
|---|---|---|---|---|
| Login | `/login` | Public | Email + password form | Entry point |
| Forgot / Reset password | `/forgot-password`, `/reset-password` | Public | Request and set new password | Self-service recovery |
| Punch in / out (start page) | `/` | All | Live clock (company timezone), large Punch in / Punch out button, today's first in, last out, worked, late and overtime, missed-punch warning | One shared page every role uses to punch |
| My dashboard | `/me` | All | Big **Clock in / Clock out** button, live timer since clock-in, today's shift, this month's hours, leave balance, missed-punch warnings | One tap to punch; start of everyone's day |
| My attendance | `/me/attendance` | All | Monthly calendar coloured by status, daily in/out and hours, "Request correction" per day | Self-check |
| My leave | `/me/leave` | All | Balances per type, submit request (with day count preview), track and cancel requests | Leave workflow |
| My corrections | `/me/corrections` | All | Submit and track correction requests | Fix missed punches |
| Manager dashboard | `/manager` | Manager | Team: who's in / late / absent / on leave right now, pending approvals count | Team view at a glance |
| Team attendance | `/manager/attendance` | Manager | Employee × date grid, click a day to see punches | Review team |
| Approvals | `/manager/approvals` | Manager | Pending leave and correction requests, approve/reject with note, shows remaining balance | Handle requests quickly |
| Team reports | `/manager/reports` | Manager | Timesheet, late, overtime for own team, CSV export | Team analysis |
| Admin dashboard | `/admin` | Admin | Company-wide who's in today, late count, absences, pending approvals, trend chart | Whole-company view |
| Employee management | `/admin/employees` | Admin | Table with search and filters, create/edit, assign department/manager/shift, CSV import, activate/deactivate | Manage staff |
| Organisation settings | `/admin/departments`, `/admin/shifts`, `/admin/holidays`, `/admin/leave-types` | Admin | CRUD for each | Configure rules |
| Attendance admin | `/admin/attendance` | Admin | Search any employee/day, edit punches with a reason | Disputes and special cases |
| Reports | `/admin/reports` | Admin | All reports, filters, charts, CSV/payroll export | Analysis and payroll |
| Audit log | `/admin/audit` | Admin | Filterable list of changes | Accountability |
| Profile / settings | `/profile` | All | View details, change password, log out all devices | Account control |

**UX rules and why:**
- **Mobile-first** layout — many employees clock in from their phone at the door.
- **Clock button shows state clearly** ("Clocked in since 09:04 — 3h 12m") — nobody wonders whether the punch worked.
- **Disable the button while the request is in flight** — prevents double taps (the server blocks them too).
- **Show times in the company timezone** with the zone label — avoids confusion for remote staff.
- **Colour + icon + text** for statuses (not colour only) — accessible to colour-blind users.
- **Loading skeletons and clear error messages** — users know what is happening.

---

## 12. Security

| Measure | What | Why |
|---|---|---|
| Password hashing | Argon2id | Stolen DB doesn't reveal passwords |
| HTTPS everywhere | TLS on Vercel/Railway; `Secure` cookies | Stops eavesdropping on tokens/passwords |
| Server-side timestamps | Punch time from the server clock only; `.strict()` Zod schema rejects any time field | Time can't be forged from the client |
| Input validation | Zod on every body, query and param | Rejects malformed/malicious input early |
| SQL injection protection | Prisma parameterised queries; `$queryRaw` only with tagged templates | User input is never concatenated into SQL |
| XSS protection | React escapes output; no `dangerouslySetInnerHTML`; access token kept in memory | Scripts can't be injected or steal tokens |
| CSRF protection | Refresh cookie is `SameSite=Strict` and scoped to `/api/auth`; other endpoints use Bearer header | Other sites can't make authenticated requests |
| Authentication middleware | On every non-public route | No accidental open endpoints |
| Authorization | Role + reporting-line checks; no self-approval | Least privilege, separation of duties |
| Office network / geofence (optional) | IP allowlist or location radius for punches | Reduces clocking in from home or for a colleague |
| Rate limiting | `express-rate-limit`: login 5/min per IP+email, punches 10/min per user, global 100/min | Stops brute-force and abuse |
| Account lockout | 10 failed logins → 15-min lock | Slows password guessing |
| Deactivation | Deactivating an employee revokes all refresh tokens | Leavers lose access immediately |
| CORS | Allow only the frontend origin, `credentials: true` | Other websites can't call the API from a browser |
| Security headers | `helmet` (CSP, HSTS, no-sniff, frameguard) | Browser-level protections |
| Environment variables | Secrets in `.env`, never committed; validated at startup | Secrets don't leak via Git |
| Secure tokens | Short JWT, rotated hashed refresh tokens, reuse detection | Limits damage of token theft |
| Audit logs | All punches edited, corrections, leave decisions, user changes, logins | Traceability and payroll disputes |
| Least-privilege DB user | App DB user can't drop tables | Limits damage of a compromise |
| Dependency scanning | `npm audit`, Dependabot | Known vulnerabilities get patched |
| Error messages | No stack traces to clients in production | Don't leak internals |
| Personal data | Only needed fields stored; location stored only at punch time; employees see only own data; retention policy for leavers | Privacy and labour-law compliance |

---

## 13. Error Handling

**What:** All errors go through one central Express error handler that returns a consistent JSON format.

```ts
// backend/src/utils/AppError.ts
export class AppError extends Error {
  constructor(public status: number, public code: string, message: string, public details?: unknown) {
    super(message);
  }
}
```

```ts
// backend/src/middleware/errorHandler.ts
export function errorHandler(err: unknown, req: Request, res: Response, _next: NextFunction) {
  if (err instanceof ZodError) {
    return res.status(400).json({ error: { code: "VALIDATION_ERROR", message: "Invalid input", details: err.flatten() } });
  }
  if (err instanceof Prisma.PrismaClientKnownRequestError) {
    if (err.code === "P2002") return res.status(409).json({ error: { code: "DUPLICATE", message: "Record already exists" } });
    if (err.code === "P2025") return res.status(404).json({ error: { code: "NOT_FOUND", message: "Record not found" } });
  }
  if (err instanceof AppError) {
    return res.status(err.status).json({ error: { code: err.code, message: err.message, details: err.details } });
  }
  logger.error({ err, path: req.path, userId: req.user?.id }, "Unhandled error");
  return res.status(500).json({ error: { code: "INTERNAL_ERROR", message: "Something went wrong" } });
}
```

**Why:**
- The frontend handles all errors the same way (show `message`, highlight fields from `details`).
- Unknown errors are logged in full on the server but hidden from users (security).
- The clock-in service catches `P2002` from the open-entry index itself and throws `ALREADY_CLOCKED_IN`, so the user gets a clear message instead of the generic `DUPLICATE`.
- Express 5 passes thrown errors from `async` handlers to this middleware automatically (on Express 4 use `express-async-errors`).

**Scheduled jobs:** The nightly job wraps each employee in its own try/catch, logs failures with the employee ID, and continues; a summary (`processed`, `failed`) is logged and sent to Sentry if `failed > 0`. Because the job is idempotent, it can simply be re-run for that date.

**Frontend:** The API client throws a typed `ApiError`; TanStack Query shows toast messages; a React error boundary shows a fallback page for crashes; `401` triggers one refresh attempt, then redirects to login.

---

## 14. Testing Strategy

| Type | Tool | What is tested | Why |
|---|---|---|---|
| Unit | Vitest | `utils/time.ts` (work date, late, overtime, overnight shifts, company timezone vs UTC), status rules, attendance %, leave day counting | Fast; this math decides pay |
| API / Integration | Vitest + Supertest + test PostgreSQL | Full request → DB: status codes, validation, RBAC | Proves layers work together |
| Authentication | Supertest | Login, wrong password, expired token, refresh rotation, reuse detection, logout, deactivated user | Security-critical |
| Authorization | Supertest | Each role against each endpoint (matrix from §3.3); manager vs other team; self-approval | Ensures no privilege leaks |
| Attendance rules | Supertest | Double clock-in → 409, clock-out without clock-in → 409, client-sent time rejected, IP not allowed → 403 | Core business rules |
| Leave & corrections | Supertest | Insufficient balance → 422, overlap → 422, approval updates balance and days, 8-day-old correction → 422 | Money-relevant rules |
| Nightly job | Vitest + test DB | Absences created, open entries closed, leave days marked, running twice changes nothing | Idempotency |
| Concurrency | Supertest | Two simultaneous clock-ins → one 201, one 409; two approvals of the same leave → balance deducted once | Race-condition protection |
| End-to-end | Playwright | Employee clocks in → out → requests leave → manager approves → balance updates | Real user journeys |
| Frontend components | Vitest + Testing Library | ClockButton states, leave form day preview, form errors | UI behaves correctly |

**Time in tests:** Tests use `vi.useFakeTimers()` / `vi.setSystemTime()` to control "now", so "late at 09:25" or "the day after" is deterministic.

**Key test cases (examples):**

```ts
it("marks LATE after shift start + grace", () => {
  const day = calcDay({ shift: general, entries: [entry("09:25", "18:30")] });
  expect(day).toMatchObject({ status: "LATE", lateMinutes: 25, workedMinutes: 485, overtimeMinutes: 5 });
});

it("returns 409 on double clock-in", async () => {
  await request(app).post("/api/attendance/clock-in").set(auth(employee)).send({}).expect(201);
  await request(app).post("/api/attendance/clock-in").set(auth(employee)).send({}).expect(409);
});

it("ignores a client-supplied clock-in time", async () => {
  await request(app).post("/api/attendance/clock-in").set(auth(employee))
    .send({ clockInAt: "2026-09-25T00:00:00Z" }).expect(400);
});

it("forbids a manager from approving another team's leave", async () => {
  await request(app).patch(`/api/leave/${otherTeamLeaveId}/review`).set(auth(manager))
    .send({ decision: "APPROVED" }).expect(403);
});

it("forbids approving your own leave", async () => {
  await request(app).patch(`/api/leave/${ownLeaveId}/review`).set(auth(manager))
    .send({ decision: "APPROVED" }).expect(403);
});
```

**Test database:** A separate `DATABASE_URL` for tests; migrations run before the suite; tables truncated between tests.
**Why:** Tests must never touch development or production data, and each test must start clean.

**Coverage goal:** ≥ 70% on services, ≥ 95% on `utils/time.ts`, 100% of the RBAC matrix.

---

## 15. Deployment

### 15.1 Environments

| Env | Purpose |
|---|---|
| Local | Development with Docker PostgreSQL |
| Staging (optional) | Test deployments before production |
| Production | Real users |

**Why:** Changes are tested somewhere safe before real users see them.

### 15.2 Environment Variables

`backend/.env`
```
NODE_ENV=production
PORT=4000
DATABASE_URL=postgresql://user:pass@host:5432/attendance
JWT_ACCESS_SECRET=<64+ random chars>
JWT_ACCESS_EXPIRES_IN=15m
REFRESH_TOKEN_EXPIRES_DAYS=7
CORS_ORIGIN=https://attendance.example.com
SMTP_HOST=...
SMTP_USER=...
SMTP_PASS=...
APP_URL=https://attendance.example.com

# Attendance rules
COMPANY_TIMEZONE=Asia/Tokyo
HALF_DAY_MIN_HOURS=4
AUTO_CLOSE_AFTER_HOURS=14
CORRECTION_WINDOW_DAYS=7
LATE_FLAG_PER_MONTH=3
ATTENDANCE_FLAG_THRESHOLD=90
OFFICE_IP_ALLOWLIST=            # comma-separated CIDRs; empty = allow anywhere
GEOFENCE_LAT=
GEOFENCE_LNG=
GEOFENCE_RADIUS_M=              # empty = geofence off
CLOSE_OUT_CRON=0 2 * * *        # 02:00 company time
```

`frontend/.env.local`
```
NEXT_PUBLIC_API_URL=https://api.attendance.example.com/api
```

**Why:** Secrets and per-environment settings stay out of code. Attendance rules are configurable per company without code changes. `NEXT_PUBLIC_` variables are visible in the browser, so **never** put secrets there.

### 15.3 Deployment Steps

1. **Database:** Create PostgreSQL on Railway/Render → copy `DATABASE_URL`.
2. **Backend:** Connect GitHub repo to Railway/Render, root = `backend/`.
   - Build: `npm ci && npx prisma generate && npm run build`
   - Release/pre-deploy: `npx prisma migrate deploy`
   - Start: `node dist/server.js`
   - Run **one** instance (the cron job runs inside it). If you scale to more instances later, move the job to a separate worker or use a DB advisory lock so it runs once.
3. **Seed** first admin, leave types and a default shift once: `npm run seed:prod`.
4. **Frontend:** Import repo in Vercel, root = `frontend/`, set `NEXT_PUBLIC_API_URL`.
5. **Domain & HTTPS:** Add custom domains; both platforms issue TLS certificates automatically.
6. **CORS:** Set `CORS_ORIGIN` to the final frontend URL.
7. **IP allowlist:** If used, make sure the API sees the real client IP behind the platform's proxy (`app.set("trust proxy", 1)`).

**Why `migrate deploy` (not `migrate dev`):** `deploy` only applies existing, reviewed migrations and never resets data. `dev` is for local use only.

**Cookie note:** For the refresh cookie to work across frontend and API, put them on the same site (e.g. `app.example.com` and `api.example.com`). If they are on different sites, use `SameSite=None; Secure` and add a CSRF token.

### 15.4 CI/CD

GitHub Actions on every push/PR: install → lint → type-check → unit + integration tests (with a PostgreSQL service container) → build. Vercel and Railway auto-deploy `main` after CI passes.

**Why:** Broken code is caught before it reaches users.

### 15.5 Backup

- Managed daily automatic backups (Railway/Render, 30-day retention) + weekly `pg_dump` to separate storage.
- Test restoring a backup at least once per quarter, and before each year-end leave reset.

**Why:** Attendance is payroll evidence. A backup you've never restored might not work.

### 15.6 Monitoring

| Tool | What | Why |
|---|---|---|
| `GET /api/health` | Returns DB connectivity status | Hosting platform restarts unhealthy containers |
| Pino logs | Structured request/error logs | Debugging production issues |
| Sentry | Error tracking for frontend, backend and the nightly job | Get alerted with stack traces |
| Uptime monitor (e.g. UptimeRobot) | Pings health endpoint, alerts before 08:00 | Know about outages before the morning clock-in rush |
| Job heartbeat | Nightly job pings a heartbeat URL when it finishes | Alert if absences were not processed |

---

## 16. Development Roadmap

Build in this order — each phase depends on the previous one.

| Phase | Tasks | Output | Why this order |
|---|---|---|---|
| **0. Setup** (Week 1) | Monorepo, TypeScript, ESLint/Prettier, Docker PostgreSQL, Express skeleton, Next.js skeleton, CI | Both apps run locally | Foundation for everything |
| **1. Database** (Week 1–2) | Prisma schema, first migration (+ partial index), seed script | Tables + demo data | Every feature reads/writes this data |
| **2. Authentication** (Week 2) | Login, refresh, logout, middleware, password hashing, frontend login + auth context | Users can log in | All other endpoints need `req.user` |
| **3. Authorization** (Week 2) | `requireRole`, reporting-line helpers, frontend route guards, RBAC tests | Roles enforced | Must exist before exposing data |
| **4. Employee & organisation management** (Week 3) | Employees CRUD, CSV import, departments, shifts, holidays, leave types; admin pages | HR can set up the company | Attendance needs employees and shifts |
| **5. Clock-in / clock-out** (Week 4) | `utils/time.ts`, clock-in/out, `recalculateDay()`, today/me endpoints, clock button UI | Core feature works | Main purpose of the system |
| **6. Nightly job & corrections** (Week 5) | Close-out job, missed punches, correction requests and approval, admin edit | Every day has a result; mistakes fixable | Depends on attendance days |
| **7. Leave** (Week 5–6) | Balances, requests, approval, balance deduction, days → ON_LEAVE; employee + manager pages | Leave workflow | Depends on attendance days and the job |
| **8. Reports & dashboards** (Week 6–7) | SQL aggregations, per-role dashboards, charts, timesheet CSV export | Insights and payroll export | Needs real attendance data |
| **9. Security hardening** (Week 7) | Rate limiting, helmet, lockout, IP allowlist/geofence, password reset email, security review | Production-ready security | Easier once all endpoints exist |
| **10. Testing** (ongoing, focus Week 7) | Fill coverage gaps, concurrency tests, E2E journeys | Confidence | Write tests with each feature, finish here |
| **11. Deployment** (Week 8) | Hosting, env vars, migrations, domain, monitoring, backups, job heartbeat | Live system | Last step once stable |
| **12. Documentation** (Week 8) | README, API docs (Swagger/OpenAPI), user guides for employees, managers and HR | Handover | Reflects the final system |

---

## 17. Future Improvements

| Idea | What | Why it's valuable |
|---|---|---|
| Kiosk / QR check-in | A tablet at the entrance shows a rotating QR code; employees scan it with their phone | Proves physical presence without special hardware |
| Biometric / RFID integration | Fingerprint or ID-card readers post punches to the same API | Stops buddy punching |
| Face verification | Selfie at clock-in compared to profile photo | Stronger identity check (needs explicit consent) |
| Shift scheduling / rosters | Weekly rotating shifts, shift swaps | Supports retail, factories, hospitals |
| Overtime pre-approval | Overtime counts only if approved by the manager | Controls overtime cost |
| Payroll integration | Push the monthly timesheet directly to payroll software | No manual CSV import |
| Notifications | Email/Slack/Teams reminders: "You haven't clocked in", "3 requests waiting for approval" | Fewer missed punches and slow approvals |
| Mobile app (React Native) | Reuses the same API | Push notifications, better location handling |
| Offline punches (PWA) | Store punch offline with a signed server nonce, sync later | Sites with poor connectivity |
| Multi-company (multi-tenant) | One deployment serves several companies | SaaS potential |
| Fine-grained permissions | Permissions table instead of fixed roles | E.g. "HR assistant", "Department head" |
| Analytics | Absence trends, lateness patterns, department comparisons | Earlier HR intervention |
| i18n | Multiple languages (e.g. English/Japanese) | Wider usability |
| OpenAPI + generated client | Auto-generated docs and typed frontend client | Less manual work, fewer mismatches |

---

## 18. Glossary

| Term | Meaning |
|---|---|
| **API** | Application Programming Interface — the URLs the frontend calls to get/change data |
| **REST** | A style of API where URLs represent resources and HTTP methods (GET, POST, PUT, PATCH, DELETE) represent actions |
| **JWT** | JSON Web Token — a signed token containing the user ID and role |
| **Refresh token** | Long-lived token used only to get new access tokens |
| **RBAC** | Role-Based Access Control — permissions based on user role |
| **IDOR** | Insecure Direct Object Reference — accessing someone else's data by changing an ID |
| **ORM** | Object-Relational Mapper — lets you query the DB with code instead of raw SQL |
| **Migration** | A versioned file that changes the database structure |
| **Seed** | Script that inserts initial/demo data |
| **Transaction** | A group of DB operations that all succeed or all fail together |
| **Soft delete** | Marking a row inactive instead of deleting it |
| **Clock-in / clock-out (punch)** | Recording the moment an employee starts or stops working |
| **Time entry** | One clock-in and its matching clock-out |
| **Work date** | The local calendar day a piece of work belongs to |
| **Shift** | The expected working hours and days of an employee |
| **Grace period** | Minutes after shift start before an employee counts as late |
| **Overtime** | Minutes worked beyond the expected shift hours |
| **Missed punch** | A clock-in without a clock-out, closed automatically by the system |
| **Correction request** | An employee's request to fix a wrong or missing punch, approved by a manager |
| **Leave balance** | Leave days allocated minus days used for a leave type in a year |
| **Timesheet** | Monthly summary of days and hours per employee, used for payroll |
| **Buddy punching** | One employee clocking in for another |
| **Partial unique index** | A unique rule that applies only to rows matching a condition (e.g. open entries) |
| **Idempotent** | Running an operation twice has the same effect as running it once |
| **Cron job** | A task that runs automatically on a schedule |
| **CORS** | Browser rule controlling which websites may call your API |
| **CSRF** | Attack that tricks a logged-in browser into sending unwanted requests |
| **XSS** | Attack that injects malicious scripts into a page |
| **Argon2id** | Modern, memory-hard password hashing algorithm |
| **Audit log** | Permanent record of who changed what and when |
