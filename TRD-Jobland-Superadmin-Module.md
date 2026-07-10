# Technical Requirements Document (TRD)
## Jobland — Superadmin Authentication Module

**Version:** 1.0
**Date:** 2026-07-07
**Author:** sandeep@secinverse.com
**Status:** Draft

### Related Documents
- **[TRD-Jobland-Role-Module.md](./TRD-Jobland-Role-Module.md)** — defines the roles (Superadmin, Vendor, Property Manager, User) and permissions. This Authentication module **issues the JWT** whose `role` claim comes from the Role module, and the Role module's authorization middleware **consumes that JWT** to enforce access. The two modules are directly connected: *Auth authenticates → Role authorizes.*

---

## 1. Overview
This module implements authentication for the **Superadmin** role of the Jobland platform. It covers secure login using **email & password** with **JWT (JSON Web Token)** based session handling, and a **forgot/reset password** flow using a time-bound reset token.

> **Connection to Role Module:** The `role` value embedded in the issued JWT (see §8.4) is one of the **six roles** defined in the [Role & Permission Module](./TRD-Jobland-Role-Module.md) — `superadmin`, `hotel_central_authority`, `property_manager`, `department_manager`, `vendor_manager`, `worker`. **Login is shared across all roles**; only the role claim differs, and the app redirects by role after login (see §1.2).

Development is delivered in phases:
- **Phase 1 — Mock Setup:** Auth endpoints backed by in-memory / hardcoded mock data (no DB) to unblock frontend integration and validate the JWT flow.
- **Phase 2 — Real Login:** Email & password validated against the database with hashed passwords.
- **Phase 3 — Forgot Password:** Reset-token generation, delivery, and password reset.

**Post-login behavior:** When login succeeds and the JWT's `role = superadmin`, the app redirects the user to the **Superadmin Dashboard** (see §14). The dashboard shows platform-wide statistics (properties, clients, vendors, workers, active shifts), a platform activity feed, and real-time attendance with overtime flags.

### 1.1 Superadmin Modules
The **Superadmin** role is organised into **10 functional modules**. Authentication (§1–13) is the entry point; after login, the Superadmin works across these modules. Each module has its **own actions and functions** — those will be added as they are shared.

| # | Module | Purpose (summary) | Actions & Functions | Detailed spec |
|---|--------|-------------------|---------------------|---------------|
| 1 | Dashboard | Landing overview: summary cards + today's work | ✅ Defined | §14 |
| 2 | Hotel Management | Manage hotel chains/properties, property managers, departments | ✅ Defined | §15 |
| 3 | Vendor Management | Approve/suspend vendors; view ratings, subscription, headcount | ✅ Defined | §16 |
| 4 | Worker Management | View/suspend workers; resolve agency conflicts & assignment overrides | ✅ Defined | §17 |
| 5 | Work Request & Bidding | Oversee/override work requests; view bids; escalate stalled | ✅ Defined | §18 |
| 6 | Subscription & Billing | View subscriptions; configure tiers; credits/refunds; export | ✅ Defined | §19 |
| 7 | Rating & Trust | Audit ratings; tune weightage; remove fraudulent; review flagged | ✅ Defined | §20 |
| 8 | Dispute Resolution | Review disputes/evidence; binding hours decisions; QR/geofence overrides | ✅ Defined | §21 |
| 9 | Report & Exports | Generate entity/date reports; export logs; platform analytics | ✅ Defined | §22 |
| 10 | Platform Setting | Weightage, notification templates, pricing config, feature flags | ✅ Defined | §23 |

> **Status legend:** ✅ Defined · ⏳ Awaiting the actions/functions you'll share.
>
> All 10 modules are defined (§14–§23). Each module section includes its actions/functions, FRs, data model, API, UI, phases, acceptance criteria, and open questions.

### 1.2 Shared Login & Role-Based Redirect (all roles)
All roles use the **same login page** and the **shared authentication flow** (§1–13). Credentials are validated once; the issued JWT carries the `role` claim. After a successful login the app **redirects by role** to that role's landing module — the login screen, endpoints, and JWT issuance are identical for everyone; only the redirect target differs.

| Role (`role` claim) | Post-login landing |
|---------------------|--------------------|
| `superadmin` | Superadmin Dashboard (§14) — `/superadmin/dashboard` |
| `hotel_central_authority` | Hotel Central Authority Dashboard (§25) — `/hca/dashboard` |
| `property_manager` | Property Manager Dashboard — `/property-manager/dashboard` |
| `department_manager` | Department Manager Dashboard — `/department-manager/dashboard` |
| `vendor_manager` | Vendor Dashboard — `/vendor/dashboard` |
| `worker` | Worker home (mobile app) |

> Landing **routes are provisional [CONFIRM]**. Flow: shared login → JWT issued with `role` → router reads `role` → redirect to that role's dashboard.

## 2. Scope
### In Scope
- Superadmin login via email & password.
- JWT access token issuance and validation.
- Protected-route middleware for superadmin-only APIs.
- Mock authentication provider for early development.
- Forgot password (request reset) and reset password (consume token) flows.
- Password hashing and secure token storage.

### Out of Scope
- Other user roles (employer, candidate) — separate modules.
- Social / OAuth login.
- Multi-factor authentication (MFA) — future phase.
- User registration/self sign-up (superadmin is provisioned, not self-registered).

## 3. Definitions & Acronyms
| Term | Definition |
|------|------------|
| JWT | JSON Web Token — signed token carrying auth claims |
| Access Token | Short-lived JWT used to authorize API requests |
| Refresh Token | Longer-lived token used to obtain a new access token |
| Reset Token | Single-use, time-bound token to reset a password |
| Bcrypt | One-way password hashing algorithm |
| Superadmin | Highest-privilege administrative user of Jobland |

## 4. Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1 | Superadmin can log in with email and password | High |
| FR-2 | On successful login, system returns a signed JWT access token | High |
| FR-3 | Invalid credentials return a generic error (no user enumeration) | High |
| FR-4 | JWT must carry `sub` (user id), `role`, `iat`, `exp` claims | High |
| FR-5 | Protected superadmin APIs reject requests without a valid JWT | High |
| FR-6 | A mock auth provider must serve login/JWT with hardcoded data (Phase 1) | High |
| FR-7 | Superadmin can request a password reset by submitting their email | High |
| FR-8 | System generates a single-use, expiring reset token on request | High |
| FR-9 | Reset link/token is delivered to the superadmin (email in real phase) | Medium |
| FR-10 | Superadmin can set a new password using a valid reset token | High |
| FR-11 | Used or expired reset tokens are rejected | High |
| FR-12 | Passwords are stored only as bcrypt hashes | High |
| FR-13 | (Optional) Refresh token endpoint to renew expired access tokens | Low |

## 5. Non-Functional Requirements
| ID | Requirement | Category |
|----|-------------|----------|
| NFR-1 | Login response returns within 500 ms (excluding email send) | Performance |
| NFR-2 | Passwords hashed with bcrypt (cost factor >= 10) | Security |
| NFR-3 | JWT signed with strong secret/RS key stored in env, never in code | Security |
| NFR-4 | Access token expiry: 15–60 min; reset token expiry: 15–30 min | Security |
| NFR-5 | Rate limiting on login and forgot-password endpoints | Security |
| NFR-6 | All auth traffic over HTTPS | Security |
| NFR-7 | Error messages must not leak whether an email exists | Security |

## 6. System Architecture
```
Client (Superadmin UI)
        |  email/password
        v
[POST /api/superadmin/auth/login]
        |
        v
 Auth Service --> (Phase 1) Mock Provider  --+
        |         (Phase 2) DB + Bcrypt       |
        v                                     |
   JWT Signer --------------------------------+
        |  access token
        v
Client stores token --> sends `Authorization: Bearer <jwt>`
        v
[Auth Middleware] verifies signature + exp + role --> Protected APIs
```

**Components**
- **Auth Controller** — HTTP handlers for login, forgot, reset.
- **Auth Service** — business logic; swappable data source (mock vs DB).
- **JWT Utility** — sign/verify access (and refresh) tokens.
- **Auth Middleware** — validates JWT and enforces `role = superadmin`.
- **Reset Token Store** — persists reset tokens with expiry & used flag.
- **Mailer (Phase 3)** — sends reset link.

## 7. Data Model
### Superadmin
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| name | string | Display name |
| email | string | Unique, lowercase |
| passwordHash | string | Bcrypt hash |
| role | string | Fixed: `superadmin` |
| isActive | boolean | Login blocked if false |
| createdAt | timestamp | |
| updatedAt | timestamp | |

### PasswordResetToken
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| userId | UUID | FK -> Superadmin.id |
| tokenHash | string | Hash of reset token (store hash, not raw) |
| expiresAt | timestamp | Single-use expiry |
| used | boolean | Set true after consumption |
| createdAt | timestamp | |

### Mock Data (Phase 1)
```json
{
  "id": "mock-superadmin-001",
  "name": "Super Admin",
  "email": "superadmin@jobland.test",
  "password": "Admin@123",
  "role": "superadmin"
}
```

## 8. API / Interface Design
Base path: `/api/superadmin/auth`

| Endpoint | Method | Description | Auth |
|----------|--------|-------------|------|
| `/login` | POST | Authenticate and return JWT | Public |
| `/forgot-password` | POST | Request a password reset token | Public |
| `/reset-password` | POST | Set new password via reset token | Public (token) |
| `/me` | GET | Return current superadmin profile | Bearer JWT |
| `/refresh` | POST | Issue new access token (optional) | Refresh token |

### 8.1 POST /login
**Request**
```json
{ "email": "superadmin@jobland.test", "password": "Admin@123" }
```
**Response 200**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "user": { "id": "mock-superadmin-001", "email": "superadmin@jobland.test", "role": "superadmin" }
}
```
**Response 401**
```json
{ "success": false, "message": "Invalid email or password" }
```

### 8.2 POST /forgot-password
**Request**
```json
{ "email": "superadmin@jobland.test" }
```
**Response 200** (always generic, to avoid enumeration)
```json
{ "success": true, "message": "If the account exists, a reset link has been sent." }
```

### 8.3 POST /reset-password
**Request**
```json
{ "token": "<reset-token>", "newPassword": "NewAdmin@456" }
```
**Response 200**
```json
{ "success": true, "message": "Password reset successful. Please log in." }
```
**Response 400**
```json
{ "success": false, "message": "Invalid or expired reset token" }
```

### 8.4 JWT Payload
```json
{
  "sub": "mock-superadmin-001",
  "email": "superadmin@jobland.test",
  "role": "superadmin",
  "iat": 1751875200,
  "exp": 1751878800
}
```

## 9. Implementation Phases
### Phase 1 — Mock Setup
- Hardcoded superadmin credentials in a mock provider.
- `/login` validates against mock data and returns a real signed JWT.
- Auth middleware fully functional against the JWT.
- Goal: frontend can integrate login + protected routes without a DB.

### Phase 2 — Real Email/Password Login
- Replace mock provider with DB lookup.
- Verify password with bcrypt against `passwordHash`.
- Enforce `isActive` and `role = superadmin`.

### Phase 3 — Forgot / Reset Password
- Generate cryptographically random reset token; store only its hash.
- Set expiry (15–30 min) and `used = false`.
- Deliver token via email (mock/console in dev).
- `/reset-password` verifies token hash, expiry, unused -> updates passwordHash, marks token used.

## 10. Dependencies
- JWT library (e.g. `jsonwebtoken`).
- Password hashing (e.g. `bcrypt`).
- Crypto for reset-token generation (`crypto.randomBytes`).
- Mailer (Phase 3) — SMTP / provider.
- `.env` for `JWT_SECRET`, `JWT_EXPIRES_IN`, `RESET_TOKEN_EXPIRES_IN`.

## 11. Assumptions & Constraints
- Superadmin accounts are provisioned by seed/migration, not self sign-up.
- Single superadmin role; no granular permissions in this module.
- HTTPS is terminated at gateway/load balancer.
- Email delivery is stubbed/logged in dev environments.

## 12. Acceptance Criteria
- Valid mock credentials return a verifiable JWT (Phase 1).
- Protected route returns 401 without/with invalid token, 200 with valid token.
- Invalid login returns generic 401 with no enumeration.
- Forgot-password always returns generic success.
- Reset-password succeeds once with a valid token and fails on reuse/expiry.
- Passwords never stored or logged in plaintext.

## 13. Open Questions
- Should refresh tokens be included in v1, or access-token only?
- Reset link base URL for the frontend?
- Email provider/service to use in production?
- Access token and reset token exact expiry values?

---

## 14. Superadmin Dashboard (Post-Login)

### 14.1 Overview
After a successful login where `role = superadmin`, the app redirects to the **Superadmin Dashboard** at `/superadmin/dashboard`. The dashboard is the landing view of the **Dashboard module** (§1.1 #1) and provides platform-wide visibility.

**Actions & Functions**
1. **View platform-wide statistics** — total properties, clients, vendors, workers, and active shifts.
2. **View platform-wide activity feed** — a chronological feed of key events across the platform.
3. **Monitor real-time attendance and overtime** — live view of today's shifts with attendance status and overtime flags.

**Flow:** Login (§8.1) → JWT issued with `role = superadmin` → redirect to dashboard → load statistics + activity feed + real-time attendance.

### 14.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-D1 | On login with `role = superadmin`, redirect to the Superadmin dashboard | High |
| FR-D2 | Dashboard accessible only to authenticated `superadmin`; others get 403/redirect | High |
| FR-D3 | Show platform-wide statistic cards, each with a title and a numeric count | High |
| FR-D4 | Stat cards: **Total Properties, Total Clients, Total Vendors, Total Workers, Active Shifts** | High |
| FR-D5 | Card counts are fetched from the backend (not hardcoded) | High |
| FR-D6 | Show a **platform-wide activity feed** of recent events (newest first) | High |
| FR-D7 | Each activity item shows actor, action, target, and time | Medium |
| FR-D8 | Show a **real-time attendance** view of today's shifts | High |
| FR-D9 | Attendance view **flags overtime** shifts (flag + hours) | High |
| FR-D10 | Attendance columns: **Property, Worker, Shift, Clock In, Clock Out, Status, Overtime** | High |
| FR-D11 | Attendance lists only records for the current date | High |
| FR-D12 | Statistics and attendance refresh in near real-time (poll/refresh) | Medium |
| FR-D13 | Show empty, loading, and error states for all three sections | Medium |

### 14.3 UI Layout
```
+-----------------------------------------------------------------------+
|  Superadmin Dashboard                                    [ Superadmin ]|
+-----------------------------------------------------------------------+
|  +----------+ +----------+ +----------+ +----------+ +-----------+     |
|  |  Total   | |  Total   | |  Total   | |  Total   | |  Active   |     |
|  | Property | |  Client  | | Vendors  | | Workers  | |  Shifts   |     |
|  |    86    | |   512    | |    47    | |   130    | |    23     |     |
|  +----------+ +----------+ +----------+ +----------+ +-----------+     |
|                                                                       |
|  +------------------------------+  +-------------------------------+  |
|  | Activity Feed                |  | Real-time Attendance          |  |
|  |------------------------------|  |-------------------------------|  |
|  | . PM added a worker    09:15 |  | Property | Worker  | Overtime |  |
|  | . Vendor bid on WR-102 09:02 |  | Sunrise  | J.Doe   | -        |  |
|  | . Shift started        08:50 |  | Palm     | J.Smith | ! 1.5h   |  |
|  | . Client onboarded     08:30 |  | LakeView | R.Kumar | -        |  |
|  +------------------------------+  +-------------------------------+  |
+-----------------------------------------------------------------------+
```

### 14.4 Platform Statistics (Cards)
| Card Title | Meaning (data source) |
|------------|-----------------------|
| Total Properties | Count of properties |
| Total Clients | Count of clients |
| Total Vendors | Count of vendor accounts |
| Total Workers | Count of workers / work users |
| Active Shifts | Count of shifts currently active (clocked in, not ended) |

### 14.5 Platform Activity Feed
A chronological list of key platform events (newest first).

| Field | Description | Example |
|-------|-------------|---------|
| actor | Who performed the action | "Property Manager – Palm Villa" |
| action | What happened | "added a worker" |
| target | Entity affected | "Jane Smith" |
| time | When it happened | "09:15 AM" |

Example event types: worker added, client onboarded, vendor bid placed, shift started/ended, dispute raised, subscription changed.

### 14.6 Real-time Attendance & Overtime
A live view of **today's** shifts with attendance status and overtime flags.

| Column | Description | Example |
|--------|-------------|---------|
| Property Name | Property the shift belongs to | "Sunrise Apts" |
| Worker Name | Assigned worker | "John Doe" |
| Shift | Work shift | Morning / Evening / Night |
| Clock In | Actual clock-in time | 08:02 |
| Clock Out | Actual clock-out time (blank if active) | — |
| Status | Attendance status | Present / Absent / In Progress / Completed |
| Overtime | Overtime flag + hours | ⚠ 1.5h / — |

**Overtime rule:** a shift is flagged as overtime when worked hours exceed the shift's scheduled duration. **[CONFIRM]** the exact overtime threshold/policy.

### 14.7 Data Model (read models)
**Dashboard Statistics (computed)**
| Field | Type | Notes |
|-------|------|-------|
| totalProperties | integer | Count of properties |
| totalClients | integer | Count of clients |
| totalVendors | integer | Count of vendors |
| totalWorkers | integer | Count of workers |
| activeShifts | integer | Count of currently active shifts |

**Activity Feed item**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Event id |
| actor | string | Who performed the action |
| action | string | Action description |
| target | string | Affected entity |
| createdAt | timestamp | Event time |

**Attendance item**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Shift/attendance record id |
| propertyName | string | Property name |
| workerName | string | Worker name |
| shift | enum | `morning` \| `evening` \| `night` |
| clockIn | timestamp | Actual clock-in |
| clockOut | timestamp | Actual clock-out (nullable) |
| status | enum | `present` \| `absent` \| `in_progress` \| `completed` |
| isOvertime | boolean | True if worked hours exceed scheduled |
| overtimeHours | number | Overtime amount (hours) |
| workDate | date | Filtered to today |

### 14.8 API / Interface Design
Base path: `/api/superadmin/dashboard` — requires `Authorization: Bearer <jwt>` with `role = superadmin`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/statistics` | GET | Returns the platform-wide statistic counts (cards) |
| `/activity-feed` | GET | Returns recent platform activity events (newest first) |
| `/attendance` | GET | Returns today's real-time attendance with overtime flags |

**GET /statistics — Response 200**
```json
{
  "totalProperties": 86,
  "totalClients": 512,
  "totalVendors": 47,
  "totalWorkers": 130,
  "activeShifts": 23
}
```

**GET /activity-feed — Response 200**
```json
{
  "items": [
    { "id": "a1", "actor": "PM – Palm Villa",     "action": "added a worker", "target": "Jane Smith", "createdAt": "2026-07-10T09:15:00Z" },
    { "id": "a2", "actor": "Vendor – BrightClean", "action": "placed a bid",   "target": "WR-102",     "createdAt": "2026-07-10T09:02:00Z" }
  ]
}
```

**GET /attendance — Response 200**
```json
{
  "date": "2026-07-10",
  "items": [
    { "id": "s1", "propertyName": "Sunrise Apts", "workerName": "John Doe",  "shift": "morning", "clockIn": "08:02", "clockOut": null,    "status": "in_progress", "isOvertime": false, "overtimeHours": 0 },
    { "id": "s2", "propertyName": "Palm Villa",   "workerName": "Jane Smith", "shift": "evening", "clockIn": "14:00", "clockOut": "22:30", "status": "completed",   "isOvertime": true,  "overtimeHours": 1.5 }
  ]
}
```

**Empty / Forbidden**
```json
{ "date": "2026-07-10", "items": [] }
```
```json
{ "success": false, "message": "Forbidden: superadmin only" }
```

### 14.9 Status Values
| Status | Label | Suggested Color |
|--------|-------|-----------------|
| present | Present | Green |
| absent | Absent | Red |
| in_progress | In Progress | Blue |
| completed | Completed | Green |
| overtime (flag) | Overtime | Amber ⚠ |

### 14.10 Implementation Phases
- **Phase 1 — Mock:** Return hardcoded statistics, a static activity feed, and a static attendance list; build the full UI (cards + feed + attendance) against the mock API.
- **Phase 2 — Real Data:** Replace with real aggregation counts, a live activity feed, and today's attendance computed from clock-in/out with overtime evaluation.

### 14.11 Acceptance Criteria
- Login as `superadmin` redirects to the dashboard.
- Non-superadmin cannot access the dashboard (403/redirect).
- All 5 statistic cards render with correct titles and live counts.
- Activity feed renders recent events, newest first.
- Attendance view renders today's shifts with clock in/out, status, and overtime flags.
- Empty, loading, and error states are handled for all three sections.

### 14.12 Open Questions
- Exact **overtime policy** — threshold/scheduled hours per shift and how overtime is calculated?
- Which **event types** should appear in the activity feed, and how far back?
- Refresh mechanism for "real-time" — polling interval, or websockets?
- Is "Active Shifts" defined as clocked-in-not-ended, or scheduled-for-now?

---

## 15. Hotel Management Module

### 15.1 Overview
**Module #2** of the Superadmin role (§1.1). It lets the Superadmin manage hotel accounts — both **hotel chains** (multiple properties under one brand/owner) and **single-property accounts** — the properties within them, the property managers assigned to each property, and visibility into department structures across the platform.

Route: `/superadmin/hotels`

**Actions & Functions**
1. **Create, edit, deactivate hotel chain or single-property accounts** — provision a chain (brand/owner) or a standalone property, edit its details, and deactivate/reactivate it.
2. **View all properties under any chain** — drill into a chain to see all its properties.
3. **Assign and remove property managers** — attach or detach a Property Manager to/from a property.
4. **View all department structures across the platform** — read-only view of departments per property, platform-wide.

> **Note:** the detailed account/property **forms and cards** (all fields, mock data, validation) will be added under this module per your flow. This section defines the module's structure, key data, and APIs; the full create/edit UI detail is **[TO BE ADDED]**.

### 15.2 Concepts / Account Types
| Type | Meaning |
|------|---------|
| Hotel Chain account | A brand/owner that owns **multiple** properties. |
| Single-property account | An owner with **one** property (no chain grouping). |
| Property | An individual hotel/location, belonging to a chain or standalone. |
| Property Manager | A user assigned to run one property (see Role module). |
| Department | An operational unit within a property (Housekeeping, F&B, …). |

### 15.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-H1 | Superadmin can create a **hotel chain** account | High |
| FR-H2 | Superadmin can create a **single-property** account | High |
| FR-H3 | Superadmin can edit a chain or property account | High |
| FR-H4 | Superadmin can **deactivate/reactivate** a chain or property account | High |
| FR-H5 | Deactivating a chain cascades to / blocks access for its properties **[CONFIRM]** | Medium |
| FR-H6 | Superadmin can view **all properties under any chain** | High |
| FR-H7 | Superadmin can view standalone (single) properties | High |
| FR-H8 | Superadmin can **assign** a Property Manager to a property | High |
| FR-H9 | Superadmin can **remove** a Property Manager from a property | High |
| FR-H10 | A property has at most one active Property Manager **[CONFIRM]** | Medium |
| FR-H11 | Superadmin can view the **department structure** for any property (read-only) | High |
| FR-H12 | Department view spans **all properties** across the platform | Medium |
| FR-H13 | All actions restricted to `role = superadmin` | High |

### 15.4 Data Model
**HotelChain / Account**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| accountType | enum | `chain` \| `single_property` |
| name | string | Chain/brand or account name |
| ownerName | string | Contact/owner name |
| email | string | Contact email |
| phone | string | Contact phone |
| status | enum | `active` \| `inactive` |
| createdAt | timestamp | |

> For a **single-property** account, one Property is created together with the account.

**Property** — key fields for this module (full property schema to be added with the Property flow):
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| propertyName | string | Property/hotel name |
| chainId | UUID (nullable) | FK -> HotelChain.id; `null` if standalone |
| propertyManagerId | UUID (nullable) | FK -> User (`role = property_manager`) |
| status | enum | `active` \| `inactive` |

**Department (read model)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| propertyId | UUID | FK -> Property.id |
| name | string | e.g. Housekeeping, Front Desk, F&B |
| managerId | UUID (nullable) | Department Manager assigned |

### 15.5 API / Interface Design
Base path: `/api/superadmin/hotels` — requires `Bearer <jwt>` with `role = superadmin`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/chains` | GET | List hotel chains / accounts |
| `/chains` | POST | Create a chain or single-property account |
| `/chains/:id` | PUT | Edit an account |
| `/chains/:id/status` | PATCH | Activate / deactivate an account |
| `/chains/:id/properties` | GET | View all properties under a chain |
| `/properties/:id/manager` | PUT | Assign a Property Manager to a property |
| `/properties/:id/manager` | DELETE | Remove a property's Property Manager |
| `/properties/:id/departments` | GET | View a property's department structure |
| `/departments` | GET | View department structures across the platform |

**POST /chains — Request**
```json
{
  "accountType": "chain",
  "name": "Sunrise Hotels Group",
  "ownerName": "John Carter",
  "email": "owner@sunrisehotels.com",
  "phone": "+44 7911 123456"
}
```
**POST /chains — Response 201**
```json
{ "success": true, "chain": { "id": "ch1", "accountType": "chain", "name": "Sunrise Hotels Group", "status": "active" } }
```

**PUT /properties/:id/manager — Request**
```json
{ "propertyManagerId": "pm-102" }
```
**PUT /properties/:id/manager — Response 200**
```json
{ "success": true, "propertyId": "p1", "propertyManagerId": "pm-102" }
```

**GET /chains/:id/properties — Response 200**
```json
{
  "chainId": "ch1",
  "items": [
    { "id": "p1", "propertyName": "Sunrise Apartments", "status": "active", "propertyManagerId": "pm-102" },
    { "id": "p7", "propertyName": "Sunrise Bay Resort",  "status": "active", "propertyManagerId": null }
  ]
}
```

### 15.6 UI Layout (brief)
- **Hotels list** (`/superadmin/hotels`): chains and single properties shown as cards; **"Add Hotel Account"** action lets the Superadmin choose *chain* or *single property*.
- **Chain detail**: lists the chain's properties; each property row shows its assigned Property Manager with **Assign / Remove** controls.
- **Department view**: property → departments tree (read-only), filterable across all properties.

### 15.7 Implementation Phases
- **Phase 1 — Mock:** Chains, properties, PM assignment, and department trees served from mock data; full UI built against the mock API.
- **Phase 2 — Real Data:** DB-backed accounts/properties; assign/remove PM persists; department structures read from real data.

### 15.8 Acceptance Criteria
- Superadmin can create a chain and a single-property account, then edit and deactivate them.
- Chain detail lists all properties under that chain.
- Superadmin can assign and remove a Property Manager on a property.
- Department structure is viewable per property and across the platform (read-only).
- All endpoints reject non-superadmin callers.

### 15.9 Open Questions
- Does deactivating a **chain** cascade to its properties (and their users)?
- Can a property have **more than one** Property Manager, or exactly one?
- Can a property be **moved** between chains?
- Is department **create/edit** part of this module, or view-only here (managed by Property/Department Managers)? *(Functions list says "view".)*

---

## 16. Vendor Management Module

### 16.1 Overview
**Module #3** of the Superadmin role (§1.1). It gives the Superadmin oversight of all **vendor organisations** on the platform — reviewing and approving registrations, suspending or deactivating accounts, and viewing each vendor's ratings, subscription status, and headcount.

Route: `/superadmin/vendors`

**Actions & Functions**
1. **View all vendors on the platform** — a list of every vendor organisation.
2. **Approve or reject vendor registrations** — moderate vendors that have signed up and are awaiting approval.
3. **Suspend or deactivate vendor accounts** — temporarily suspend or permanently deactivate an approved vendor.
4. **View vendor ratings, subscription status, and headcount** — per-vendor rating, current plan/subscription state, and number of workers.

### 16.2 Vendor Account Status
| Status | Meaning |
|--------|---------|
| pending | Registered, awaiting Superadmin approval |
| approved | Approved and active on the platform |
| rejected | Registration declined |
| suspended | Temporarily blocked (reversible) |
| deactivated | Permanently disabled |

**Transitions:** `pending → approved / rejected`; `approved → suspended / deactivated`; `suspended → approved (reinstate) / deactivated`.

### 16.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-V1 | Superadmin can view a list of all vendors | High |
| FR-V2 | Vendor list shows status, rating, subscription status, and headcount | High |
| FR-V3 | Superadmin can view a single vendor's details | High |
| FR-V4 | Superadmin can **approve** a pending vendor registration | High |
| FR-V5 | Superadmin can **reject** a pending vendor registration (with reason) | High |
| FR-V6 | Superadmin can **suspend** an approved vendor | High |
| FR-V7 | Superadmin can **deactivate** a vendor account | High |
| FR-V8 | Superadmin can **reinstate** a suspended vendor | Medium |
| FR-V9 | Vendor list is filterable by status | Medium |
| FR-V10 | Rating, subscription status, and headcount are read-only aggregates | Medium |
| FR-V11 | All actions restricted to `role = superadmin` | High |

### 16.4 Data Model
**Vendor**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| businessName | string | Vendor organisation name |
| ownerName | string | Vendor manager / contact |
| email | string | Contact email |
| phone | string | Contact phone |
| status | enum | `pending` \| `approved` \| `rejected` \| `suspended` \| `deactivated` |
| rating | number | Average rating (0–5), read-only aggregate |
| subscriptionStatus | enum | `active` \| `trial` \| `expired` \| `none` |
| subscriptionPlan | string (nullable) | Current plan name (from Subscription module) |
| headcount | integer | Number of workers under the vendor |
| rejectionReason | string (nullable) | Set when `status = rejected` |
| createdAt | timestamp | Registration date |

> `rating`, `subscriptionStatus`/`subscriptionPlan`, and `headcount` are **sourced from other modules** (Rating & Trust, Subscription & Billing, Worker Management) and shown here read-only.

### 16.5 API / Interface Design
Base path: `/api/superadmin/vendors` — requires `Bearer <jwt>` with `role = superadmin`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/vendors` | GET | List all vendors (filter by `?status=`) |
| `/vendors/:id` | GET | Get a single vendor's details |
| `/vendors/:id/approve` | PATCH | Approve a pending registration |
| `/vendors/:id/reject` | PATCH | Reject a pending registration (with reason) |
| `/vendors/:id/suspend` | PATCH | Suspend an approved vendor |
| `/vendors/:id/deactivate` | PATCH | Deactivate a vendor |
| `/vendors/:id/reinstate` | PATCH | Reinstate a suspended vendor |

**GET /vendors — Response 200**
```json
{
  "items": [
    { "id": "v1", "businessName": "BrightClean Services", "status": "approved",  "rating": 4.6, "subscriptionStatus": "active",  "headcount": 24 },
    { "id": "v2", "businessName": "GreenScape Facilities", "status": "pending",   "rating": 0,   "subscriptionStatus": "none",    "headcount": 0 },
    { "id": "v3", "businessName": "Summit Maintenance",    "status": "suspended", "rating": 3.9, "subscriptionStatus": "expired", "headcount": 11 }
  ]
}
```

**PATCH /vendors/:id/reject — Request**
```json
{ "reason": "Incomplete business documents" }
```
**PATCH /vendors/:id/approve — Response 200**
```json
{ "success": true, "vendorId": "v2", "status": "approved" }
```

### 16.6 UI Layout (brief)
- **Vendor list** (`/superadmin/vendors`): table/cards of vendors showing name, status badge, rating, subscription status, and headcount; status filter tabs (All / Pending / Approved / Suspended / Deactivated).
- **Pending vendors** show **Approve** / **Reject** actions; **approved** vendors show **Suspend** / **Deactivate**; **suspended** show **Reinstate** / **Deactivate**.
- **Vendor detail**: full profile with ratings, subscription, and headcount.

### 16.7 Implementation Phases
- **Phase 1 — Mock:** Vendor list and detail served from mock data; status actions update the mock record; rating/subscription/headcount shown as static values.
- **Phase 2 — Real Data:** DB-backed vendors; status transitions persist; rating/subscription/headcount pulled from their source modules.

### 16.8 Acceptance Criteria
- Superadmin can view all vendors with status, rating, subscription status, and headcount.
- Pending registrations can be approved or rejected (rejection captures a reason).
- Approved vendors can be suspended or deactivated; suspended vendors can be reinstated.
- List can be filtered by status.
- All endpoints reject non-superadmin callers.

### 16.9 Open Questions
- On **rejection**, is the vendor notified (email) and can they re-apply?
- Does **suspend/deactivate** cascade to the vendor's workers and active work?
- Is `headcount` all workers, or only **active** workers?
- Should there be an audit trail of status changes (who/when/why)?

---

## 17. Worker Management Module

### 17.1 Overview
**Module #4** of the Superadmin role (§1.1). It gives the Superadmin platform-wide oversight of all **workers** registered under any vendor/agency — suspending or deactivating accounts, and acting as the final authority to resolve **worker–agency association conflicts** and override **primary/secondary agency assignments** when they are in dispute.

Route: `/superadmin/workers`

**Actions & Functions**
1. **View all registered workers across all vendors** — every worker on the platform, regardless of agency.
2. **Deactivate or suspend worker accounts** — block a worker temporarily (suspend) or permanently (deactivate).
3. **Resolve worker–agency association conflicts** — adjudicate disputes where a worker's agency association is contested.
4. **Override primary/secondary agency assignments in dispute** — set/reassign a worker's primary and secondary agencies as the deciding authority.

> **Terminology:** an **agency** is a vendor organisation (see §16). A worker may be associated with **multiple agencies** — exactly one **primary** and zero-or-more **secondary**. **[CONFIRM]** that "agency" == "vendor".

### 17.2 Concepts
| Term | Meaning |
|------|---------|
| Worker | An individual registered under one or more agencies. |
| Agency | A vendor organisation the worker is associated with (§16). |
| Primary agency | The single main agency responsible for the worker. |
| Secondary agency | Additional agency the worker also works with. |
| Association conflict | A contested association (e.g. two agencies both claim the worker as primary). |

### 17.3 Worker Account Status
| Status | Meaning |
|--------|---------|
| active | Registered and able to work |
| suspended | Temporarily blocked (reversible) |
| deactivated | Permanently disabled |

### 17.4 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-WM1 | Superadmin can view all workers across all vendors/agencies | High |
| FR-WM2 | Worker list shows agencies (primary/secondary) and account status | High |
| FR-WM3 | Superadmin can view a single worker's details and associations | High |
| FR-WM4 | Superadmin can **suspend** a worker account | High |
| FR-WM5 | Superadmin can **deactivate** a worker account | High |
| FR-WM6 | Superadmin can **reinstate** a suspended worker | Medium |
| FR-WM7 | Superadmin can view a list of **association conflicts** | High |
| FR-WM8 | Superadmin can **resolve** an association conflict (with a decision/reason) | High |
| FR-WM9 | Superadmin can **override the primary agency** of a worker | High |
| FR-WM10 | Superadmin can **override secondary agency** assignments | High |
| FR-WM11 | Worker list is filterable by status, agency, and conflict flag | Medium |
| FR-WM12 | All actions restricted to `role = superadmin` | High |

### 17.5 Data Model
**Worker**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| name | string | Full name |
| email | string | Contact email |
| phone | string | Contact phone |
| status | enum | `active` \| `suspended` \| `deactivated` |
| primaryAgencyId | UUID (nullable) | FK -> Vendor.id (primary agency) |
| createdAt | timestamp | Registration date |

**WorkerAgencyAssociation**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| workerId | UUID | FK -> Worker.id |
| agencyId | UUID | FK -> Vendor.id |
| associationType | enum | `primary` \| `secondary` |
| status | enum | `active` \| `in_conflict` \| `pending` |
| createdAt | timestamp | |

**AssociationConflict**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| workerId | UUID | FK -> Worker.id |
| agencyIds | UUID[] | Agencies involved in the dispute |
| type | enum | e.g. `duplicate_primary` \| `ownership_dispute` |
| status | enum | `open` \| `resolved` |
| resolution | string (nullable) | Decision/notes |
| resolvedBy | UUID (nullable) | Superadmin id |
| resolvedAt | timestamp (nullable) | |
| createdAt | timestamp | |

### 17.6 API / Interface Design
Base path: `/api/superadmin/workers` — requires `Bearer <jwt>` with `role = superadmin`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/workers` | GET | List all workers (filter by `?status=`, `?agencyId=`, `?conflict=true`) |
| `/workers/:id` | GET | Get a worker's details and associations |
| `/workers/:id/suspend` | PATCH | Suspend a worker |
| `/workers/:id/deactivate` | PATCH | Deactivate a worker |
| `/workers/:id/reinstate` | PATCH | Reinstate a suspended worker |
| `/workers/:id/primary-agency` | PATCH | Override the worker's primary agency |
| `/workers/:id/associations` | PUT | Override/replace secondary agency assignments |
| `/conflicts` | GET | List association conflicts |
| `/conflicts/:id/resolve` | PATCH | Resolve a conflict (decision + reason) |

**GET /workers — Response 200**
```json
{
  "items": [
    { "id": "w1", "name": "John Doe",   "status": "active",    "primaryAgency": "BrightClean Services", "secondaryAgencies": ["Summit Maintenance"], "hasConflict": false },
    { "id": "w2", "name": "Ravi Kumar", "status": "active",    "primaryAgency": null,                    "secondaryAgencies": [],                     "hasConflict": true },
    { "id": "w3", "name": "Jane Smith", "status": "suspended", "primaryAgency": "GreenScape Facilities", "secondaryAgencies": [],                     "hasConflict": false }
  ]
}
```

**PATCH /workers/:id/primary-agency — Request**
```json
{ "agencyId": "v1" }
```

**PATCH /conflicts/:id/resolve — Request**
```json
{ "decision": "assign_primary", "agencyId": "v1", "reason": "Verified original registration by BrightClean" }
```
**PATCH /conflicts/:id/resolve — Response 200**
```json
{ "success": true, "conflictId": "cf1", "status": "resolved" }
```

### 17.7 UI Layout (brief)
- **Worker list** (`/superadmin/workers`): table/cards showing name, status, primary agency, secondary agencies, and a **conflict flag**; filters for status / agency / conflicts.
- **Worker detail**: profile, agency associations, and **Suspend / Deactivate / Reinstate** actions plus **override primary/secondary agency** controls.
- **Conflicts view**: list of open association conflicts with a **Resolve** action (choose the deciding agency + reason).

### 17.8 Implementation Phases
- **Phase 1 — Mock:** Workers, associations, and conflicts served from mock data; status changes, overrides, and conflict resolution update the mock records.
- **Phase 2 — Real Data:** DB-backed workers/associations/conflicts; transitions and overrides persist; conflicts recomputed from real association data.

### 17.9 Acceptance Criteria
- Superadmin can view all workers across all agencies with their primary/secondary agencies and status.
- Superadmin can suspend, deactivate, and reinstate a worker.
- Association conflicts are listed and can be resolved with a recorded decision and reason.
- Superadmin can override a worker's primary agency and secondary agency assignments.
- All endpoints reject non-superadmin callers.

### 17.10 Open Questions
- Is **"agency" the same as "vendor"** (§16), or a distinct entity? *(Assumed same.)*
- Can a worker have **multiple secondary** agencies, or just one?
- What events **create a conflict** — duplicate primary claims, overlapping active work, or a manual dispute filing?
- Are the affected **agencies notified** when the Superadmin overrides an assignment or resolves a conflict?
- Does suspend/deactivate here **cascade** to the worker's active shifts/work orders?

---

## 18. Work Request & Bidding Module

### 18.1 Overview
**Module #5** of the Superadmin role (§1.1). It gives the Superadmin full visibility and override authority over the **work request** and **bidding** lifecycle across the platform — viewing every request and its bids, overriding or cancelling any request, reviewing bidding history and vendor responses, and escalating or force-resolving requests that have stalled.

Route: `/superadmin/work-requests`

**Actions & Functions**
1. **View all work requests across all properties and vendors** — every request on the platform, regardless of owner.
2. **Override or cancel any work request** — edit a request's details/status or cancel it outright.
3. **View bidding history and vendor responses** — the full set of bids and vendor replies for any request.
4. **Escalate or force-resolve stalled requests** — flag/escalate a stuck request, or force it to a resolution (award/close/cancel).

### 18.2 Work Request Lifecycle
| Status | Meaning |
|--------|---------|
| open | Created, not yet accepting bids |
| bidding | Open for vendor bids |
| awarded | A vendor's bid has been accepted |
| in_progress | Work is underway |
| completed | Work finished |
| cancelled | Cancelled (by owner or Superadmin) |
| stalled | No progress within the expected window (e.g. no bids, or awarded but not started) |
| escalated | Flagged by Superadmin for attention |

**Bid status:** `submitted` → `accepted` / `rejected` / `withdrawn`.

### 18.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-WR1 | Superadmin can view all work requests across all properties and vendors | High |
| FR-WR2 | Work request list is filterable by property, vendor, and status | Medium |
| FR-WR3 | Superadmin can view a single work request's full detail | High |
| FR-WR4 | Superadmin can **override** a work request (edit fields/status) | High |
| FR-WR5 | Superadmin can **cancel** any work request | High |
| FR-WR6 | Superadmin can view the **bidding history** and vendor responses for a request | High |
| FR-WR7 | System **identifies stalled** requests (per a stall rule) | High |
| FR-WR8 | Superadmin can **escalate** a stalled request | High |
| FR-WR9 | Superadmin can **force-resolve** a stalled request (award / close / cancel) | High |
| FR-WR10 | Override/cancel/resolve actions are recorded (who/when/why) | Medium |
| FR-WR11 | All actions restricted to `role = superadmin` | High |

### 18.4 Data Model
**WorkRequest**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| propertyId | UUID | FK -> Property.id |
| propertyName | string | Denormalized property name |
| title | string | Short summary of the work |
| description | string | Details |
| status | enum | `open` \| `bidding` \| `awarded` \| `in_progress` \| `completed` \| `cancelled` \| `stalled` \| `escalated` |
| awardedVendorId | UUID (nullable) | Winning vendor, if awarded |
| createdBy | UUID | Requester (e.g. Property Manager) |
| createdAt | timestamp | |
| updatedAt | timestamp | |

**Bid**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| workRequestId | UUID | FK -> WorkRequest.id |
| vendorId | UUID | FK -> Vendor.id |
| vendorName | string | Denormalized vendor name |
| amount | number | Bid amount |
| message | string (nullable) | Vendor's response/note |
| status | enum | `submitted` \| `accepted` \| `rejected` \| `withdrawn` |
| createdAt | timestamp | |

### 18.5 API / Interface Design
Base path: `/api/superadmin/work-requests` — requires `Bearer <jwt>` with `role = superadmin`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/work-requests` | GET | List all requests (filter by `?propertyId=`, `?vendorId=`, `?status=`) |
| `/work-requests/:id` | GET | Get a request's full detail |
| `/work-requests/:id` | PATCH | Override a request (edit fields/status) |
| `/work-requests/:id/cancel` | PATCH | Cancel a request |
| `/work-requests/:id/bids` | GET | Bidding history and vendor responses |
| `/work-requests/:id/escalate` | PATCH | Escalate a stalled request |
| `/work-requests/:id/force-resolve` | PATCH | Force-resolve (award / close / cancel) |

**GET /work-requests — Response 200**
```json
{
  "items": [
    { "id": "wr1", "propertyName": "Sunrise Apartments", "title": "Deep clean lobby", "status": "bidding",  "awardedVendorId": null, "bidCount": 3 },
    { "id": "wr2", "propertyName": "Palm Villa",          "title": "AC repair",        "status": "stalled",  "awardedVendorId": null, "bidCount": 0 },
    { "id": "wr3", "propertyName": "Lake View Residency", "title": "Garden upkeep",    "status": "awarded",  "awardedVendorId": "v1", "bidCount": 5 }
  ]
}
```

**GET /work-requests/:id/bids — Response 200**
```json
{
  "workRequestId": "wr3",
  "bids": [
    { "id": "b1", "vendorName": "BrightClean Services",  "amount": 320, "message": "Can start Monday", "status": "accepted", "createdAt": "2026-07-08T10:00:00Z" },
    { "id": "b2", "vendorName": "GreenScape Facilities", "amount": 290, "message": "Weekly plan",      "status": "rejected", "createdAt": "2026-07-08T11:30:00Z" }
  ]
}
```

**PATCH /work-requests/:id/force-resolve — Request**
```json
{ "resolution": "award", "vendorId": "v1", "reason": "Stalled 5 days; awarding lowest valid bid" }
```
**PATCH /work-requests/:id/force-resolve — Response 200**
```json
{ "success": true, "workRequestId": "wr2", "status": "awarded" }
```

### 18.6 UI Layout (brief)
- **Work request list** (`/superadmin/work-requests`): table of requests with property, title, status badge, bid count; filters for property / vendor / status; **stalled** and **escalated** requests visually highlighted.
- **Request detail**: full request info, current status, and **Override / Cancel / Escalate / Force-resolve** actions.
- **Bidding history**: list of all bids with vendor, amount, message, and status.

### 18.7 Implementation Phases
- **Phase 1 — Mock:** Requests and bids served from mock data; override/cancel/escalate/force-resolve update the mock record; stalled flagged by a mock rule.
- **Phase 2 — Real Data:** DB-backed requests/bids; actions persist; stalled computed from real timestamps/state.

### 18.8 Acceptance Criteria
- Superadmin can view all work requests across every property and vendor.
- Superadmin can override and cancel any request.
- Bidding history and vendor responses are viewable per request.
- Stalled requests are identified and can be escalated or force-resolved (award/close/cancel).
- All endpoints reject non-superadmin callers.

### 18.9 Open Questions
- What exactly defines a **"stalled"** request — no bids after N hours, awarded-but-not-started, or a manual flag?
- **Force-resolve** options — auto-award (which bid?), close, or cancel? Which are allowed?
- On override/cancel/resolve, are the **property and bidding vendors notified**?
- Can the Superadmin place or edit **bids**, or only view them?
- Is there an SLA/time threshold configurable per property or platform-wide?

---

## 19. Subscription & Billing Module

### 19.1 Overview
**Module #6** of the Superadmin role (§1.1). It gives the Superadmin control over the platform's **subscriptions** and **billing** — viewing every vendor's active subscription, configuring pricing tiers (base and payroll), issuing credits/refunds/billing corrections, and exporting platform-wide billing reports.

Route: `/superadmin/billing`

**Actions & Functions**
1. **View all active subscriptions across all vendors** — every vendor's current plan and subscription state.
2. **Configure subscription pricing tiers (base and payroll)** — create/edit the **base** plan tiers and **payroll** pricing tiers.
3. **Issue credits, refunds, or billing corrections** — apply financial adjustments to a vendor's account.
4. **Export billing reports across the platform** — download platform-wide billing/revenue reports.

### 19.2 Concepts
| Term | Meaning |
|------|---------|
| Subscription | A vendor's active plan on the platform. |
| Base tier | Pricing tier for the core platform subscription (e.g. Basic / Pro / Enterprise). |
| Payroll tier | Pricing tier for payroll processing (e.g. per-worker / per-payrun). **[CONFIRM]** exact model. |
| Adjustment | A credit, refund, or billing correction applied to a vendor. |
| Billing report | A platform-wide export of subscriptions, charges, and adjustments. |

### 19.3 Subscription Status
| Status | Meaning |
|--------|---------|
| active | Paid and current |
| trial | In trial period |
| past_due | Payment overdue |
| expired | Lapsed |
| cancelled | Ended |

### 19.4 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-SB1 | Superadmin can view all active subscriptions across all vendors | High |
| FR-SB2 | Subscription list is filterable by vendor, status, and tier | Medium |
| FR-SB3 | Superadmin can view a single subscription's detail | High |
| FR-SB4 | Superadmin can create/edit **base** pricing tiers | High |
| FR-SB5 | Superadmin can create/edit **payroll** pricing tiers | High |
| FR-SB6 | Superadmin can activate/deactivate a pricing tier | Medium |
| FR-SB7 | Superadmin can issue a **credit** to a vendor | High |
| FR-SB8 | Superadmin can issue a **refund** to a vendor | High |
| FR-SB9 | Superadmin can issue a **billing correction** (adjust a charge) | High |
| FR-SB10 | Every adjustment records amount, reason, and issuer | High |
| FR-SB11 | Superadmin can **export** billing reports across the platform (CSV/PDF) | High |
| FR-SB12 | All actions restricted to `role = superadmin` | High |

### 19.5 Data Model
**Subscription**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| vendorId | UUID | FK -> Vendor.id |
| vendorName | string | Denormalized vendor name |
| baseTierId | UUID | FK -> PricingTier (type `base`) |
| payrollTierId | UUID (nullable) | FK -> PricingTier (type `payroll`) |
| status | enum | `active` \| `trial` \| `past_due` \| `expired` \| `cancelled` |
| startDate | date | |
| renewalDate | date | Next billing date |
| amount | number | Current recurring amount |
| createdAt | timestamp | |

**PricingTier**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| type | enum | `base` \| `payroll` |
| name | string | e.g. Basic / Pro / Enterprise (base), or payroll plan name |
| price | number | Tier price |
| billingCycle | enum | `monthly` \| `annual` \| `per_worker` \| `per_payrun` |
| features | string[] | Included features/limits |
| isActive | boolean | Whether offered to vendors |

**BillingAdjustment**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| vendorId | UUID | FK -> Vendor.id |
| type | enum | `credit` \| `refund` \| `correction` |
| amount | number | Positive (credit/refund) or signed (correction) |
| reason | string | Justification |
| issuedBy | UUID | Superadmin id |
| createdAt | timestamp | |

### 19.6 API / Interface Design
Base path: `/api/superadmin/billing` — requires `Bearer <jwt>` with `role = superadmin`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/subscriptions` | GET | List all subscriptions (filter by `?vendorId=`, `?status=`, `?tierId=`) |
| `/subscriptions/:id` | GET | Get a subscription's detail |
| `/pricing-tiers` | GET | List pricing tiers (filter by `?type=base\|payroll`) |
| `/pricing-tiers` | POST | Create a pricing tier |
| `/pricing-tiers/:id` | PUT | Edit a pricing tier |
| `/pricing-tiers/:id/status` | PATCH | Activate/deactivate a tier |
| `/adjustments` | POST | Issue a credit / refund / correction |
| `/reports/export` | GET | Export a billing report (`?format=csv\|pdf`, date range) |

**GET /subscriptions — Response 200**
```json
{
  "items": [
    { "id": "sub1", "vendorName": "BrightClean Services", "baseTier": "Pro",   "payrollTier": "Per-Worker", "status": "active",   "amount": 199, "renewalDate": "2026-08-01" },
    { "id": "sub2", "vendorName": "GreenScape Facilities", "baseTier": "Basic", "payrollTier": null,          "status": "past_due", "amount": 99,  "renewalDate": "2026-07-05" }
  ]
}
```

**POST /pricing-tiers — Request**
```json
{ "type": "base", "name": "Enterprise", "price": 499, "billingCycle": "monthly", "features": ["Unlimited properties", "Priority support"] }
```

**POST /adjustments — Request**
```json
{ "vendorId": "v2", "type": "refund", "amount": 99, "reason": "Service outage on 2026-07-02" }
```
**POST /adjustments — Response 201**
```json
{ "success": true, "adjustment": { "id": "adj1", "vendorId": "v2", "type": "refund", "amount": 99, "status": "issued" } }
```

**GET /reports/export — Response 200**
Returns a file download (CSV/PDF) of platform-wide subscriptions, charges, and adjustments for the selected period.

### 19.7 UI Layout (brief)
- **Subscriptions list** (`/superadmin/billing`): vendors with base tier, payroll tier, status, amount, renewal date; filters by status/tier.
- **Pricing tiers**: manage **Base** and **Payroll** tiers (create/edit/activate) in separate tabs.
- **Adjustments**: issue a credit/refund/correction against a vendor with amount + reason.
- **Reports**: date-range picker + **Export** (CSV/PDF).

### 19.8 Implementation Phases
- **Phase 1 — Mock:** Subscriptions, tiers, and adjustments from mock data; tier config and adjustments update mock records; export returns a sample file.
- **Phase 2 — Real Data:** DB-backed billing; tier changes and adjustments persist; refunds integrate with the payment provider; reports generated from real data.

### 19.9 Acceptance Criteria
- Superadmin can view all active subscriptions across all vendors with base and payroll tiers.
- Superadmin can create and edit base and payroll pricing tiers.
- Superadmin can issue credits, refunds, and corrections, each recording amount, reason, and issuer.
- Superadmin can export a platform-wide billing report (CSV/PDF).
- All endpoints reject non-superadmin callers.

### 19.10 Open Questions
- Exact **base vs payroll** pricing model — is payroll billed per worker, per pay run, or a flat add-on?
- **Payment provider** for refunds/charges (e.g. Stripe)? Are refunds processed there or recorded only?
- Do **tier changes** apply to existing subscriptions immediately or at next renewal?
- Report **formats** (CSV/PDF/XLSX) and required columns/date ranges?
- **Currency** handling — single currency or per-vendor?

---

## 20. Rating & Trust Module

### 20.1 Overview
**Module #7** of the Superadmin role (§1.1). It gives the Superadmin oversight of the platform's **rating and trust** system — auditing every rating across workers, vendors, and properties, tuning how ratings are weighted into trust scores, removing fraudulent or abusive ratings after review, and working a queue of flagged/disputed ratings.

Route: `/superadmin/ratings`

**Actions & Functions**
1. **View and audit all ratings across workers, vendors, properties** — every rating on the platform, by subject.
2. **Adjust rating weightage thresholds** — configure shift-count bands and their weight (e.g. 1–5 shifts = X%, 5–10 = Y%, …) used in trust-score calculation.
3. **Remove fraudulent or abusive ratings after review** — take down a rating with a recorded reason.
4. **View flagged or disputed ratings** — a moderation queue of ratings reported as problematic or under dispute.

### 20.2 Concepts
| Term | Meaning |
|------|---------|
| Rating | A 1–5 score (with optional comment) given about a worker, vendor, or property. |
| Subject | The entity being rated (`worker` \| `vendor` \| `property`). |
| Trust score | A derived score computed from a subject's weighted ratings. |
| Weightage threshold | A shift-count band that assigns a **weight %** to ratings (e.g. 1–5 shifts → 60%). |
| Flagged / disputed | A rating reported as fraudulent/abusive, or contested, awaiting review. |

### 20.3 Rating Status
| Status | Meaning |
|--------|---------|
| active | Counted toward trust score |
| flagged | Reported as fraudulent/abusive, under review |
| disputed | Contested by the subject, under review |
| removed | Taken down by Superadmin (excluded from trust score) |

### 20.4 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-RT1 | Superadmin can view/audit all ratings across workers, vendors, and properties | High |
| FR-RT2 | Ratings are filterable by subject type, subject, status, and score | Medium |
| FR-RT3 | Superadmin can view a rating's detail, author, and context (shift/work) | High |
| FR-RT4 | Superadmin can configure **weightage thresholds** (shift-count band → weight %) | High |
| FR-RT5 | Weightage bands are validated (no overlaps/gaps; weights within 0–100%) | Medium |
| FR-RT6 | Superadmin can **remove** a fraudulent/abusive rating after review (with reason) | High |
| FR-RT7 | Superadmin can view the **flagged/disputed** ratings queue | High |
| FR-RT8 | Removal and threshold changes are recorded (who/when/why) — audit trail | Medium |
| FR-RT9 | Removing a rating triggers **trust-score recalculation** for the subject | Medium |
| FR-RT10 | All actions restricted to `role = superadmin` | High |

### 20.5 Data Model
**Rating**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| subjectType | enum | `worker` \| `vendor` \| `property` |
| subjectId | UUID | Entity being rated |
| subjectName | string | Denormalized subject name |
| authorType | enum | Who rated (e.g. `property_manager` \| `vendor` \| `worker`) |
| authorId | UUID | Rater id |
| score | number | 1–5 |
| comment | string (nullable) | Review text |
| contextShiftId | UUID (nullable) | Shift/work this rating relates to |
| status | enum | `active` \| `flagged` \| `disputed` \| `removed` |
| flagReason | string (nullable) | Why flagged/disputed |
| removalReason | string (nullable) | Set when removed |
| createdAt | timestamp | |

**RatingWeightRule** (threshold band)
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| minShifts | integer | Band lower bound (inclusive) |
| maxShifts | integer (nullable) | Band upper bound (`null` = and above) |
| weightPercent | number | Weight applied to ratings in this band (0–100) |
| isActive | boolean | Whether the rule is applied |

> **[CONFIRM]** whether the band is based on the **subject's** shift count or the **rater's**, and whether changing thresholds recomputes trust scores retroactively.

### 20.6 API / Interface Design
Base path: `/api/superadmin/ratings` — requires `Bearer <jwt>` with `role = superadmin`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/ratings` | GET | List/audit all ratings (filter by `?subjectType=`, `?subjectId=`, `?status=`, `?score=`) |
| `/ratings/:id` | GET | Get a rating's detail and context |
| `/ratings/:id/remove` | PATCH | Remove a fraudulent/abusive rating (with reason) |
| `/ratings/flagged` | GET | List flagged/disputed ratings (moderation queue) |
| `/weight-rules` | GET | Get the current weightage threshold bands |
| `/weight-rules` | PUT | Replace/configure the weightage threshold bands |

**GET /ratings — Response 200**
```json
{
  "items": [
    { "id": "r1", "subjectType": "worker",   "subjectName": "John Doe",              "score": 5, "status": "active",   "authorType": "property_manager" },
    { "id": "r2", "subjectType": "vendor",   "subjectName": "BrightClean Services",  "score": 1, "status": "flagged",  "flagReason": "Suspected fake account" },
    { "id": "r3", "subjectType": "property", "subjectName": "Palm Villa",            "score": 2, "status": "disputed", "flagReason": "Vendor contests score" }
  ]
}
```

**PUT /weight-rules — Request**
```json
{
  "bands": [
    { "minShifts": 1,  "maxShifts": 5,    "weightPercent": 60 },
    { "minShifts": 6,  "maxShifts": 10,   "weightPercent": 80 },
    { "minShifts": 11, "maxShifts": null, "weightPercent": 100 }
  ]
}
```

**PATCH /ratings/:id/remove — Request**
```json
{ "reason": "Abusive language; violates content policy" }
```
**PATCH /ratings/:id/remove — Response 200**
```json
{ "success": true, "ratingId": "r2", "status": "removed" }
```

### 20.7 UI Layout (brief)
- **Ratings audit** (`/superadmin/ratings`): table of all ratings with subject type/name, score, status, author; filters by subject type / status / score.
- **Flagged queue**: flagged/disputed ratings with **Review → Remove / Keep** actions (reason captured on removal).
- **Weightage config**: editable table of shift-count bands → weight %, with validation for overlaps/gaps.

### 20.8 Implementation Phases
- **Phase 1 — Mock:** Ratings, flags, and weight rules from mock data; remove/keep and threshold edits update mock records.
- **Phase 2 — Real Data:** DB-backed ratings and rules; removals persist and recompute trust scores; flags sourced from real reports.

### 20.9 Acceptance Criteria
- Superadmin can view and audit all ratings across workers, vendors, and properties.
- Superadmin can configure shift-count weightage bands with valid, non-overlapping ranges.
- Superadmin can remove a fraudulent/abusive rating with a recorded reason.
- Flagged/disputed ratings are viewable in a moderation queue.
- All endpoints reject non-superadmin callers.

### 20.10 Open Questions
- Are weightage bands based on the **subject's** or the **rater's** shift count?
- Does changing thresholds **recompute trust scores retroactively**, or apply going forward?
- How are ratings **flagged** — automated abuse detection, user reports, or both?
- On removal/dispute resolution, is the **author or subject notified**?
- Is there a separate **"keep/dismiss"** action to clear a flag without removing the rating?

---

## 21. Dispute Resolution Module

### 21.1 Overview
**Module #8** of the Superadmin role (§1.1). It makes the Superadmin the **final arbiter** of disputes on the platform — reviewing raised disputes with their supporting evidence, making binding decisions on disputed hours, overriding QR/geofence attendance data proven to be erroneous, and closing or escalating disputes.

Route: `/superadmin/disputes`

**Actions & Functions**
1. **View all raised disputes with supporting evidence** — every dispute and its attached evidence (photos, documents, QR/geofence logs).
2. **Make final binding decisions on disputed hours** — set the authoritative approved hours for a contested shift.
3. **Override QR/geofence data where proven erroneous** — correct clock-in/out, QR, or geofence records when the evidence shows they are wrong.
4. **Close or escalate disputes** — resolve and close a dispute, or escalate it for further handling.

### 21.2 Concepts
| Term | Meaning |
|------|---------|
| Dispute | A contested claim (typically about **hours worked**) tied to a shift/work order. |
| Evidence | Supporting material attached to a dispute (photo, document, QR log, geofence log, note). |
| QR data | Attendance verification captured via QR scan at the property. |
| Geofence data | Location-based attendance verification (worker inside the property's geofence). |
| Binding decision | The Superadmin's final, authoritative ruling on the disputed hours. |
| Attendance override | A correction to QR/geofence/clock data made as part of a ruling. |

### 21.3 Dispute Status
| Status | Meaning |
|--------|---------|
| open | Raised, not yet reviewed |
| under_review | Being reviewed by the Superadmin |
| resolved | A binding decision has been made |
| escalated | Escalated for further handling |
| closed | Finalised and closed |

### 21.4 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-DR1 | Superadmin can view all raised disputes | High |
| FR-DR2 | Superadmin can view a dispute's **supporting evidence** (incl. QR/geofence logs) | High |
| FR-DR3 | Disputes are filterable by status, type, property, and vendor | Medium |
| FR-DR4 | Superadmin can make a **final binding decision on disputed hours** (set approved hours) | High |
| FR-DR5 | The binding decision is recorded and marked final | High |
| FR-DR6 | Superadmin can **override QR/geofence/clock data** with a reason | High |
| FR-DR7 | Superadmin can **close** a dispute | High |
| FR-DR8 | Superadmin can **escalate** a dispute | High |
| FR-DR9 | Decisions and overrides are captured in an **audit trail** (who/when/why) | Medium |
| FR-DR10 | A binding decision on hours can feed **payroll/billing** recalculation | Medium |
| FR-DR11 | All actions restricted to `role = superadmin` | High |

### 21.5 Data Model
**Dispute**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| type | enum | `disputed_hours` \| `attendance` \| `payment` \| `other` |
| shiftId | UUID (nullable) | Related shift/work order |
| propertyName | string | Denormalized property |
| raisedByType | enum | `worker` \| `vendor` \| `property_manager` |
| raisedById | UUID | Who raised it |
| against | string | Counterparty |
| claimedHours | number (nullable) | Hours claimed by the raiser |
| recordedHours | number (nullable) | Hours per system (QR/geofence) |
| approvedHours | number (nullable) | Final binding decision |
| reason | string | Dispute description |
| status | enum | `open` \| `under_review` \| `resolved` \| `escalated` \| `closed` |
| decisionReason | string (nullable) | Justification for the ruling |
| decidedBy | UUID (nullable) | Superadmin id |
| decidedAt | timestamp (nullable) | |
| createdAt | timestamp | |

**DisputeEvidence**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| disputeId | UUID | FK -> Dispute.id |
| type | enum | `photo` \| `document` \| `qr_log` \| `geofence_log` \| `note` |
| url | string (nullable) | File/link (for photo/document/logs) |
| detail | string (nullable) | Text note or log summary |
| createdAt | timestamp | |

**AttendanceOverride**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| disputeId | UUID | FK -> Dispute.id |
| shiftId | UUID | Affected shift |
| field | enum | `clock_in` \| `clock_out` \| `qr` \| `geofence` |
| originalValue | string | Value before override |
| correctedValue | string | Corrected value |
| reason | string | Why the override was made |
| overriddenBy | UUID | Superadmin id |
| createdAt | timestamp | |

### 21.6 API / Interface Design
Base path: `/api/superadmin/disputes` — requires `Bearer <jwt>` with `role = superadmin`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/disputes` | GET | List all disputes (filter by `?status=`, `?type=`, `?propertyId=`, `?vendorId=`) |
| `/disputes/:id` | GET | Get a dispute's full detail |
| `/disputes/:id/evidence` | GET | List a dispute's supporting evidence |
| `/disputes/:id/decision` | PATCH | Make a binding decision on disputed hours |
| `/disputes/:id/attendance-override` | PATCH | Override QR/geofence/clock data |
| `/disputes/:id/close` | PATCH | Close a dispute |
| `/disputes/:id/escalate` | PATCH | Escalate a dispute |

**GET /disputes — Response 200**
```json
{
  "items": [
    { "id": "d1", "type": "disputed_hours", "propertyName": "Palm Villa", "raisedByType": "worker", "claimedHours": 9, "recordedHours": 7.5, "status": "open" },
    { "id": "d2", "type": "attendance",     "propertyName": "Sunrise Apts", "raisedByType": "vendor", "claimedHours": null, "recordedHours": null, "status": "under_review" }
  ]
}
```

**PATCH /disputes/:id/decision — Request**
```json
{ "approvedHours": 8, "reason": "Geofence log confirms on-site 08:00–16:00; QR miss at clock-out" }
```
**PATCH /disputes/:id/decision — Response 200**
```json
{ "success": true, "disputeId": "d1", "approvedHours": 8, "status": "resolved" }
```

**PATCH /disputes/:id/attendance-override — Request**
```json
{ "shiftId": "s2", "field": "clock_out", "correctedValue": "16:00", "reason": "QR scanner offline; CCTV confirms departure at 16:00" }
```

### 21.7 UI Layout (brief)
- **Disputes list** (`/superadmin/disputes`): table with type, property, raiser, claimed vs recorded hours, status; filters by status/type/property/vendor.
- **Dispute detail**: claim summary, **evidence gallery** (photos/documents/QR & geofence logs), and actions — **Decide Hours**, **Override Attendance**, **Close**, **Escalate**.
- **Decision panel**: enter approved hours + reason; override panel: pick field (clock in/out, QR, geofence) → corrected value + reason.

### 21.8 Implementation Phases
- **Phase 1 — Mock:** Disputes, evidence, and overrides from mock data; decisions/overrides/close/escalate update mock records.
- **Phase 2 — Real Data:** DB-backed disputes; evidence from storage; QR/geofence logs from the attendance system; decisions feed payroll/billing recalculation.

### 21.9 Acceptance Criteria
- Superadmin can view all disputes with their supporting evidence, including QR/geofence logs.
- Superadmin can set final approved hours on a disputed shift (binding, recorded).
- Superadmin can override QR/geofence/clock data with a captured reason.
- Superadmin can close or escalate a dispute.
- All endpoints reject non-superadmin callers.

### 21.10 Open Questions
- Who can **raise** a dispute (worker, vendor, property manager), and is there a time limit after the shift?
- Does a binding hours decision **automatically recalculate payroll/billing**, or just record the ruling?
- **Escalate to whom** — the Superadmin is top authority; is escalation to an external/legal process or a review board?
- What is the authoritative source when **QR and geofence disagree**?
- Can overrides be **reverted**, and are affected parties notified of decisions?

---

## 22. Report & Exports Module

### 22.1 Overview
**Module #9** of the Superadmin role (§1.1). It lets the Superadmin generate and export **reports and analytics** across the platform — building reports scoped to any entity (hotel, vendor, worker) over a date range, exporting operational logs (hours, payroll summaries, cancellations, disputes), and downloading platform-wide analytics.

Route: `/superadmin/reports`

**Actions & Functions**
1. **Generate reports across any entity** — filter by hotel, vendor, worker, and date range.
2. **Export hours, payroll summaries, cancellation logs, dispute logs** — download these report types as files.
3. **Download platform-wide analytics** — aggregate metrics across the whole platform.

### 22.2 Report Types
| Report | Contents |
|--------|----------|
| Hours | Worked hours by worker/shift within scope and date range. |
| Payroll Summary | Payroll totals/summaries (sourced from Subscription & Billing / payroll). |
| Cancellation Log | Cancelled work requests/shifts with reason and actor. |
| Dispute Log | Disputes and their outcomes (from Dispute Resolution, §21). |
| Platform Analytics | Aggregate KPIs across hotels, vendors, workers, and revenue. |

### 22.3 Report Scope & Filters
| Filter | Values |
|--------|--------|
| entityType | `hotel` \| `vendor` \| `worker` \| `platform` |
| entityId | Specific hotel/vendor/worker id (omit for platform-wide) |
| dateFrom / dateTo | Reporting period |
| format | `csv` \| `pdf` \| `xlsx` |

### 22.4 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-RE1 | Superadmin can generate a report scoped by entity (hotel/vendor/worker) and date range | High |
| FR-RE2 | Superadmin can export an **Hours** report | High |
| FR-RE3 | Superadmin can export a **Payroll Summary** report | High |
| FR-RE4 | Superadmin can export a **Cancellation Log** | High |
| FR-RE5 | Superadmin can export a **Dispute Log** | High |
| FR-RE6 | Superadmin can download **platform-wide analytics** | High |
| FR-RE7 | Reports export in CSV/PDF/XLSX | Medium |
| FR-RE8 | Large reports may be generated **asynchronously** (job + download when ready) | Low |
| FR-RE9 | All actions restricted to `role = superadmin` | High |

### 22.5 Data Model
**ReportRequest**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| reportType | enum | `hours` \| `payroll_summary` \| `cancellation_log` \| `dispute_log` \| `analytics` |
| entityType | enum | `hotel` \| `vendor` \| `worker` \| `platform` |
| entityId | UUID (nullable) | Target entity; `null` for platform-wide |
| dateFrom | date | Period start |
| dateTo | date | Period end |
| format | enum | `csv` \| `pdf` \| `xlsx` |
| status | enum | `ready` \| `processing` \| `failed` |
| fileUrl | string (nullable) | Download link when ready |
| requestedBy | UUID | Superadmin id |
| createdAt | timestamp | |

### 22.6 API / Interface Design
Base path: `/api/superadmin/reports` — requires `Bearer <jwt>` with `role = superadmin`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/generate` | POST | Generate/export a report (type + entity + date range + format) |
| `/hours` | GET | Export an hours report (query filters) |
| `/payroll-summary` | GET | Export a payroll summary |
| `/cancellations` | GET | Export a cancellation log |
| `/disputes` | GET | Export a dispute log |
| `/analytics` | GET | Download platform-wide analytics |

**POST /generate — Request**
```json
{
  "reportType": "hours",
  "entityType": "vendor",
  "entityId": "v1",
  "dateFrom": "2026-06-01",
  "dateTo": "2026-06-30",
  "format": "csv"
}
```
**POST /generate — Response 200 (ready)**
```json
{ "status": "ready", "fileUrl": "/downloads/reports/hours-v1-2026-06.csv" }
```
**POST /generate — Response 202 (async)**
```json
{ "status": "processing", "reportId": "rep1" }
```

**GET /analytics — Response 200**
```json
{
  "period": "2026-06",
  "totalHotels": 86,
  "totalVendors": 47,
  "totalWorkers": 130,
  "totalHours": 24850,
  "totalRevenue": 184300,
  "cancellations": 62,
  "disputes": 14
}
```

### 22.7 UI Layout (brief)
- **Report builder** (`/superadmin/reports`): pick **report type**, **entity** (hotel/vendor/worker or platform-wide), **date range**, and **format**, then **Generate / Download**.
- **Quick exports**: one-click export buttons for Hours, Payroll Summary, Cancellation Log, Dispute Log.
- **Analytics**: platform-wide KPI cards/charts with a **Download** action.

### 22.8 Implementation Phases
- **Phase 1 — Mock:** Report builder UI wired to mock generation returning sample files; analytics shown from mock aggregates.
- **Phase 2 — Real Data:** Reports generated from real data across modules (hours, payroll/billing, cancellations, disputes); async jobs for large exports.

### 22.9 Acceptance Criteria
- Superadmin can generate a report filtered by hotel/vendor/worker and date range.
- Superadmin can export hours, payroll summaries, cancellation logs, and dispute logs.
- Superadmin can download platform-wide analytics.
- Exports are produced in the selected format (CSV/PDF/XLSX).
- All endpoints reject non-superadmin callers.

### 22.10 Open Questions
- Which **export formats** are required (CSV/PDF/XLSX), and which is the default?
- Are large exports **synchronous downloads** or **async jobs** (with notification when ready)?
- Which **metrics** define "platform-wide analytics" (the §22.6 sample is a starting set)?
- Are **scheduled/recurring** reports (e.g. monthly email) needed?
- Does **payroll summary** come from this module or purely from Subscription & Billing (§19)?

---

## 23. Platform Setting Module

### 23.1 Overview
**Module #10** of the Superadmin role (§1.1). It is the central **configuration hub** for the platform — tuning rating weightage thresholds, managing platform-wide notification templates (SMS, WhatsApp, email), managing subscription pricing config, and controlling feature flags and platform-wide toggles.

Route: `/superadmin/settings`

**Actions & Functions**
1. **Configure rating weightage thresholds** — the shift-count → weight % bands used in trust scoring.
2. **Set platform-wide notification templates** — SMS, WhatsApp, and email templates.
3. **Manage subscription pricing config** — the base and payroll pricing tiers.
4. **Control feature flags and platform-wide toggles** — enable/disable features across the platform.

> **Shared configuration:** rating weightage (§20 Rating & Trust) and subscription pricing (§19 Subscription & Billing) are **defined in those modules and edited here** as the central settings surface. This module is the **single source of truth** for those configs. **[CONFIRM]** whether config is edited only here or also inline in §19/§20.

### 23.2 Concepts
| Term | Meaning |
|------|---------|
| Rating weightage threshold | Shift-count band → weight % (see §20.5 `RatingWeightRule`). |
| Notification template | A reusable message template for a channel (SMS/WhatsApp/email). |
| Pricing config | Base and payroll pricing tiers (see §19.5 `PricingTier`). |
| Feature flag | A named on/off switch controlling a platform feature. |

### 23.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-PS1 | Superadmin can configure **rating weightage thresholds** (bands → weight %) | High |
| FR-PS2 | Superadmin can create/edit **notification templates** for SMS, WhatsApp, and email | High |
| FR-PS3 | Templates support **placeholders/variables** (e.g. `{{workerName}}`) | High |
| FR-PS4 | Superadmin can activate/deactivate a template | Medium |
| FR-PS5 | Superadmin can manage **subscription pricing config** (base & payroll tiers) | High |
| FR-PS6 | Superadmin can view and toggle **feature flags** platform-wide | High |
| FR-PS7 | Toggling a feature flag takes effect **platform-wide** | High |
| FR-PS8 | All setting changes are recorded (who/when) — audit trail | Medium |
| FR-PS9 | All actions restricted to `role = superadmin` | High |

### 23.4 Data Model
**NotificationTemplate**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| channel | enum | `sms` \| `whatsapp` \| `email` |
| key | string | Template key (e.g. `shift_reminder`) |
| name | string | Display name |
| subject | string (nullable) | Email subject (email only) |
| body | string | Template body with placeholders |
| variables | string[] | Supported placeholders |
| isActive | boolean | Whether the template is in use |
| updatedAt | timestamp | |

**FeatureFlag**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| key | string | Flag key (e.g. `bidding_enabled`) |
| name | string | Display name |
| description | string | What the flag controls |
| enabled | boolean | On/off |
| updatedAt | timestamp | |

> **Rating weightage** uses `RatingWeightRule` (§20.5); **pricing** uses `PricingTier` (§19.5). This module reads/writes those same records.

### 23.5 API / Interface Design
Base path: `/api/superadmin/settings` — requires `Bearer <jwt>` with `role = superadmin`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/rating-weightage` | GET / PUT | Get/configure rating weightage bands (→ §20 `RatingWeightRule`) |
| `/pricing` | GET / PUT | Get/manage subscription pricing tiers (→ §19 `PricingTier`) |
| `/notification-templates` | GET | List notification templates (filter by `?channel=`) |
| `/notification-templates` | POST | Create a template |
| `/notification-templates/:id` | PUT | Edit a template |
| `/notification-templates/:id/status` | PATCH | Activate/deactivate a template |
| `/feature-flags` | GET | List all feature flags |
| `/feature-flags/:key` | PATCH | Enable/disable a feature flag |

**POST /notification-templates — Request**
```json
{
  "channel": "email",
  "key": "shift_reminder",
  "name": "Shift Reminder",
  "subject": "Your shift at {{propertyName}} starts soon",
  "body": "Hi {{workerName}}, your {{shift}} shift at {{propertyName}} starts at {{startTime}}.",
  "variables": ["workerName", "propertyName", "shift", "startTime"]
}
```

**GET /feature-flags — Response 200**
```json
{
  "flags": [
    { "key": "bidding_enabled",        "name": "Work Request Bidding", "enabled": true },
    { "key": "whatsapp_notifications", "name": "WhatsApp Notifications", "enabled": false },
    { "key": "geofence_attendance",    "name": "Geofence Attendance",  "enabled": true }
  ]
}
```

**PATCH /feature-flags/:key — Request**
```json
{ "enabled": true }
```

### 23.6 UI Layout (brief)
- **Settings** (`/superadmin/settings`) with tabs:
  - **Rating Weightage** — editable shift-count bands → weight % (shared with §20).
  - **Notification Templates** — SMS / WhatsApp / Email tabs; create/edit templates with placeholder helper.
  - **Pricing** — manage base and payroll tiers (shared with §19).
  - **Feature Flags** — list of toggles with on/off switches and descriptions.

### 23.7 Implementation Phases
- **Phase 1 — Mock:** All settings from mock config; edits/toggles update mock state.
- **Phase 2 — Real Data:** DB-backed settings; templates wired to the SMS/WhatsApp/email providers; feature flags read by the rest of the platform at runtime.

### 23.8 Acceptance Criteria
- Superadmin can configure rating weightage thresholds.
- Superadmin can create/edit SMS, WhatsApp, and email templates with placeholders.
- Superadmin can manage base and payroll subscription pricing.
- Superadmin can toggle feature flags that take effect platform-wide.
- All endpoints reject non-superadmin callers.

### 23.9 Open Questions
- **Providers** for each channel — SMS (e.g. Twilio), WhatsApp (Business API), email (Nodemailer per tech stack)?
- Do notification templates need **multi-language/localization**?
- Are feature flags **global only**, or targetable per vendor/property?
- Is settings config edited **only here**, or also inline in §19/§20 (source-of-truth confirm)?
- Should template edits be **previewable/testable** (send a test message) before saving?

---
---

# Hotel Central Authority Role

> **§1–23 above specify the Superadmin role.** The sections below (§24+) specify the **Hotel Central Authority** role. This document holds **all roles** of the Jobland platform.

## 24. Hotel Central Authority — Role Overview

### 24.1 Overview
The **Hotel Central Authority** sits at the top of a **hotel group / brand** and oversees **all properties** belonging to that group. Its scope is the **hotel group** (not the whole platform — that is the Superadmin's scope, §1–23).

**Authentication:** login is shared across roles (see §1–13). On successful login where the JWT `role = hotel_central_authority`, the app redirects to the **Hotel Central Authority Dashboard** (§25).

**Scope rule:** every action in this role is limited to the **authority's own hotel group** and its properties, vendors, workers, and work.

### 24.2 Hotel Central Authority Modules
The role is organised into **6 functional modules**. Each has its **own actions and functions** — those will be added as they are shared.

| # | Module | Purpose (summary) | Actions & Functions | Detailed spec |
|---|--------|-------------------|---------------------|---------------|
| 1 | Dashboard | Group-wide overview across the authority's properties | ✅ Defined | §25 |
| 2 | Property Management | Manage the group's properties (and property managers) | ✅ Defined | §26 |
| 3 | Vendor Management | Nominate/approve chain vendors; ratings & headcount; discover | ✅ Defined | §27 |
| 4 | Settings & Permission | Vendor-selection permission, buffer %, payment cycle, notifications | ✅ Defined | §28 |
| 5 | Reports | Hours/payroll reports; authenticated vendor payments; cancellation/dispute logs | ✅ Defined | §29 |
| 6 | Work Request | View-only oversight of all chain work requests (no raise/approve) | ✅ Defined | §30 |

> **Status legend:** ✅ Defined · ⏳ Awaiting the actions/functions you'll share.
>
> All 6 modules are defined (§25–§30). Each module section includes its actions/functions, FRs, data model, API, UI, phases, acceptance criteria, and open questions.

### 24.3 Open Questions
- One **hotel group per authority**, or can one authority span multiple groups?
- Does Vendor Management here see **only vendors engaged by the group**, or all platform vendors (read-only)?
- How does **Settings & Permission** relate to the Superadmin's Platform Setting (§23) — group-scoped subset?

---

## 25. Hotel Central Authority — Dashboard Module

### 25.1 Overview
**Module #1** of the Hotel Central Authority role (§24.2). It is the landing view after login and gives the authority a **group-wide (chain-wide)** overview across **all properties** in its hotel group — aggregate operational stats, vendor ratings and subscription status, and dispute/overtime alerts.

Route: `/hca/dashboard`

**Scope:** all figures are aggregated **only across the authority's own hotel group's properties**.

**Actions & Functions**
1. **View aggregate stats across all properties** — hours, active workers, open requests, and vendor performance.
2. **View chain-wide vendor ratings and subscription statuses** — per-vendor rating and subscription state across the group.
3. **See dispute and overtime alerts across all properties** — a consolidated alerts view for the group.

### 25.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-HD1 | On login with `role = hotel_central_authority`, redirect to the HCA dashboard | High |
| FR-HD2 | Dashboard accessible only to `hotel_central_authority`; scoped to the own group | High |
| FR-HD3 | Show aggregate stat cards: **Total Hours, Active Workers, Open Requests, Vendor Performance** | High |
| FR-HD4 | Stats aggregate across **all properties** in the group (not a single property) | High |
| FR-HD5 | Counts/metrics are fetched from the backend (not hardcoded) | High |
| FR-HD6 | Show **chain-wide vendors** with rating and subscription status | High |
| FR-HD7 | Show **dispute and overtime alerts** across all group properties | High |
| FR-HD8 | All data is limited to the authority's group scope (no cross-group data) | High |
| FR-HD9 | Show empty, loading, and error states | Medium |

### 25.3 UI Layout
```
+-----------------------------------------------------------------------+
|  Hotel Central Authority Dashboard              [ Sunrise Hotels Grp ] |
+-----------------------------------------------------------------------+
|  +-----------+ +-----------+ +-----------+ +---------------------+     |
|  |  Total    | |  Active   | |   Open    | |  Vendor Performance |     |
|  |  Hours    | |  Workers  | |  Requests | |       4.4 / 5       |     |
|  |  12,480   | |    210    | |    18     | |                     |     |
|  +-----------+ +-----------+ +-----------+ +---------------------+     |
|                                                                       |
|  +-------------------------------+  +------------------------------+  |
|  | Vendors (chain-wide)          |  | Alerts                       |  |
|  |-------------------------------|  |------------------------------|  |
|  | Vendor        | Rate | Sub    |  | ! Overtime  Palm Villa  1.5h |  |
|  | BrightClean   | 4.6  | active |  | ! Dispute   Sunrise  #d1     |  |
|  | GreenScape    | 3.9  | expired|  | ! Overtime  Lake View   2h   |  |
|  +-------------------------------+  +------------------------------+  |
+-----------------------------------------------------------------------+
```

### 25.4 Aggregate Stat Cards
| Card | Meaning (aggregated across group properties) |
|------|----------------------------------------------|
| Total Hours | Sum of worked hours across all properties (period) |
| Active Workers | Count of currently active/working workers |
| Open Requests | Count of open/unfulfilled work requests |
| Vendor Performance | Average vendor rating/performance score for the group |

### 25.5 Data Model (read models)
**Group Stats (computed)**
| Field | Type | Notes |
|-------|------|-------|
| totalHours | number | Sum of hours across group properties |
| activeWorkers | integer | Currently active workers in the group |
| openRequests | integer | Open work requests across the group |
| vendorPerformance | number | Avg vendor rating/performance (0–5) |
| period | string | Reporting period (e.g. current month) |

**Vendor Summary item**
| Field | Type | Notes |
|-------|------|-------|
| vendorId | UUID | Vendor id |
| vendorName | string | Vendor name |
| rating | number | Chain-wide rating (0–5) |
| subscriptionStatus | enum | `active` \| `trial` \| `past_due` \| `expired` \| `none` |

**Alert item**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Alert id |
| type | enum | `dispute` \| `overtime` |
| propertyName | string | Property the alert belongs to |
| detail | string | e.g. overtime hours, or dispute reference |
| severity | enum | `info` \| `warning` \| `critical` |
| createdAt | timestamp | |

### 25.6 API / Interface Design
Base path: `/api/hca/dashboard` — requires `Bearer <jwt>` with `role = hotel_central_authority`; all responses scoped to the caller's group.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/stats` | GET | Aggregate stats across the group's properties |
| `/vendors` | GET | Chain-wide vendors with rating and subscription status |
| `/alerts` | GET | Dispute and overtime alerts across the group (filter by `?type=`) |

**GET /stats — Response 200**
```json
{ "totalHours": 12480, "activeWorkers": 210, "openRequests": 18, "vendorPerformance": 4.4, "period": "2026-07" }
```

**GET /vendors — Response 200**
```json
{
  "items": [
    { "vendorId": "v1", "vendorName": "BrightClean Services",  "rating": 4.6, "subscriptionStatus": "active" },
    { "vendorId": "v2", "vendorName": "GreenScape Facilities", "rating": 3.9, "subscriptionStatus": "expired" }
  ]
}
```

**GET /alerts — Response 200**
```json
{
  "items": [
    { "id": "al1", "type": "overtime", "propertyName": "Palm Villa",         "detail": "1.5h overtime", "severity": "warning",  "createdAt": "2026-07-10T16:30:00Z" },
    { "id": "al2", "type": "dispute",  "propertyName": "Sunrise Apartments",  "detail": "Dispute #d1 open", "severity": "critical", "createdAt": "2026-07-10T09:15:00Z" }
  ]
}
```

### 25.7 Implementation Phases
- **Phase 1 — Mock:** Stats, vendors, and alerts served from mock data scoped to a mock group; full UI built against the mock API.
- **Phase 2 — Real Data:** Real aggregation across the group's properties; vendors/ratings/subscriptions and alerts from their source modules, filtered to the group.

### 25.8 Acceptance Criteria
- Login as `hotel_central_authority` redirects to the HCA dashboard.
- Stat cards show group-wide Total Hours, Active Workers, Open Requests, and Vendor Performance.
- Vendors list shows chain-wide rating and subscription status.
- Alerts show dispute and overtime items across all group properties.
- No data outside the authority's group is visible; empty/loading/error states handled.

### 25.9 Open Questions
- Is **Vendor Performance** the average vendor rating, or a composite (rating + on-time + disputes)?
- Reporting **period** for Total Hours — current day, month, or a selectable range?
- Should alerts be **actionable** here (click through to the dispute/attendance), or view-only?
- Does "Active Workers" count workers **currently clocked in**, or all active-status workers in the group?

---

## 26. Hotel Central Authority — Property Management Module

### 26.1 Overview
**Module #2** of the Hotel Central Authority role (§24.2). It lets the authority manage the **properties under its chain** — creating, editing, and deactivating them, assigning/removing a Property Manager per property, and setting **chain-wide default settings** that apply across all the group's properties.

Route: `/hca/properties`

**Scope:** limited to the authority's **own hotel group** — it can only manage properties belonging to its chain.

**Actions & Functions**
1. **Create, edit, deactivate properties under the chain** — full lifecycle for the group's properties.
2. **Assign and remove property managers per property** — attach/detach a Property Manager to each property.
3. **Set chain-wide default settings** — vendor selection rules, buffer tolerance, and payment cycle applied group-wide.

### 26.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-HP1 | Authority can create a property under its chain | High |
| FR-HP2 | Authority can edit a property in its chain | High |
| FR-HP3 | Authority can deactivate/reactivate a property in its chain | High |
| FR-HP4 | Authority can **assign** a Property Manager to a property | High |
| FR-HP5 | Authority can **remove** a Property Manager from a property | High |
| FR-HP6 | A property has at most one active Property Manager **[CONFIRM]** | Medium |
| FR-HP7 | Authority can set **chain-wide default settings**: vendor selection rules, buffer tolerance, payment cycle | High |
| FR-HP8 | Chain-wide defaults apply to all group properties (per-property override **[CONFIRM]**) | Medium |
| FR-HP9 | All actions are limited to the authority's **own group** (no cross-group properties) | High |
| FR-HP10 | Restricted to `role = hotel_central_authority` | High |

### 26.3 Data Model
**Property** (chain-scoped)
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| chainId | UUID | The authority's group (enforced) |
| propertyName | string | Property/hotel name |
| address | string | Property address |
| propertyManagerId | UUID (nullable) | FK -> User (`role = property_manager`) |
| status | enum | `active` \| `inactive` |
| createdAt | timestamp | |

**ChainDefaultSettings**
| Field | Type | Notes |
|-------|------|-------|
| chainId | UUID | The authority's group |
| vendorSelectionRule | enum | `manual` \| `auto_lowest_bid` \| `rating_based` \| `preferred_vendors` |
| bufferTolerance | integer | Attendance/clock tolerance in **minutes** (grace buffer) |
| paymentCycle | enum | `weekly` \| `biweekly` \| `monthly` |
| updatedAt | timestamp | |

> **Settings home:** these chain-wide defaults are configured in the **Settings & Permission module (§28)**, which is the source of truth. `ChainDefaultSettings` here reflects those values. Note buffer tolerance is specified as a **% in §28** (this section's *minutes* is superseded — **[CONFIRM]** unit), and whether a property can **override** the chain defaults (per-property override lives in §28.3).

### 26.4 API / Interface Design
Base path: `/api/hca/properties` — requires `Bearer <jwt>` with `role = hotel_central_authority`; all responses scoped to the caller's group.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/properties` | GET | List the chain's properties |
| `/properties` | POST | Create a property under the chain |
| `/properties/:id` | PUT | Edit a property |
| `/properties/:id/status` | PATCH | Activate/deactivate a property |
| `/properties/:id/manager` | PUT | Assign a Property Manager |
| `/properties/:id/manager` | DELETE | Remove a Property Manager |
| `/settings/defaults` | GET | Get chain-wide default settings |
| `/settings/defaults` | PUT | Set chain-wide default settings |

**POST /properties — Request**
```json
{ "propertyName": "Sunrise Bay Resort", "address": "5 Marine Drive, Brighton, UK" }
```
**POST /properties — Response 201**
```json
{ "success": true, "property": { "id": "p7", "propertyName": "Sunrise Bay Resort", "status": "active", "propertyManagerId": null } }
```

**PUT /properties/:id/manager — Request**
```json
{ "propertyManagerId": "pm-204" }
```

**PUT /settings/defaults — Request**
```json
{
  "vendorSelectionRule": "rating_based",
  "bufferTolerance": 15,
  "paymentCycle": "biweekly"
}
```
**PUT /settings/defaults — Response 200**
```json
{ "success": true, "chainId": "ch1", "vendorSelectionRule": "rating_based", "bufferTolerance": 15, "paymentCycle": "biweekly" }
```

### 26.5 UI Layout (brief)
- **Properties list** (`/hca/properties`): the chain's properties as cards/rows with status and assigned Property Manager; **Add Property** action.
- **Property row/detail**: **Edit**, **Activate/Deactivate**, and **Assign / Remove Property Manager** controls.
- **Chain settings**: a form for **vendor selection rule**, **buffer tolerance**, and **payment cycle** (chain-wide defaults).

### 26.6 Implementation Phases
- **Phase 1 — Mock:** Properties, PM assignment, and default settings from mock data scoped to a mock chain.
- **Phase 2 — Real Data:** DB-backed properties within the group; assign/remove PM and default settings persist and apply to the group.

### 26.7 Acceptance Criteria
- Authority can create, edit, and deactivate properties within its chain only.
- Authority can assign and remove a Property Manager per property.
- Authority can set chain-wide vendor selection rule, buffer tolerance, and payment cycle.
- No property outside the authority's group is accessible.
- All endpoints reject non-`hotel_central_authority` callers.

### 26.8 Open Questions
- Can a property **override** chain-wide defaults, or are they enforced group-wide?
- **Buffer tolerance** — is it a clock-in/out grace window (minutes) or a geofence radius buffer?
- Does creating a property here also **provision the property account**, or is that Superadmin-only (§15)?
- **Vendor selection rules** — what full set of options should be supported (auto lowest bid, rating-based, preferred list, manual)?

---

## 27. Hotel Central Authority — Vendor Management Module

### 27.1 Overview
**Module #3** of the Hotel Central Authority role (§24.2). It lets the authority build and manage the **chain's approved vendor list** — nominating vendors to onboard, approving or removing vendors from the approved list, viewing vendor ratings and headcount across all the group's properties, and discovering new vendors by service area.

Route: `/hca/vendors`

**Scope:** the authority manages vendors **for its own chain**. Platform-level vendor account approval is a Superadmin function (§16); this module governs which vendors are **approved for use within the chain**. **[CONFIRM]** the exact split between chain approval and platform approval.

**Actions & Functions**
1. **Nominate vendors to onboard** — enter name, business ID, mobile, and email to put a vendor forward.
2. **Approve or remove vendors from the chain's approved list** — control the chain's usable vendor pool.
3. **View vendor ratings and headcount across all properties** — chain-wide vendor rating and worker headcount.
4. **Discover new vendors by service area** — search the vendor directory by service area.

### 27.2 Vendor (chain) Status
| Status | Meaning |
|--------|---------|
| nominated | Put forward by the authority, onboarding pending |
| onboarding | Completing registration/verification |
| approved | On the chain's approved list (usable) |
| removed | Removed from the chain's approved list |
| rejected | Nomination declined |

### 27.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-HV1 | Authority can **nominate** a vendor by name, business ID, mobile, and email | High |
| FR-HV2 | A nominated vendor enters an **onboarding** flow | Medium |
| FR-HV3 | Authority can **approve** a vendor onto the chain's approved list | High |
| FR-HV4 | Authority can **remove** a vendor from the chain's approved list | High |
| FR-HV5 | Authority can view vendor **ratings** across all group properties | High |
| FR-HV6 | Authority can view vendor **headcount** across all group properties | High |
| FR-HV7 | Authority can **discover** new vendors filtered by **service area** | High |
| FR-HV8 | All vendor data/actions are scoped to the authority's **own chain** | High |
| FR-HV9 | Restricted to `role = hotel_central_authority` | High |

### 27.4 Data Model
**VendorNomination**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| businessName | string | Vendor business name |
| businessId | string | Business/registration ID |
| mobile | string | Contact mobile |
| email | string | Contact email |
| status | enum | `nominated` \| `onboarding` \| `approved` \| `removed` \| `rejected` |
| nominatedBy | UUID | Hotel Central Authority user id |
| createdAt | timestamp | |

**ChainApprovedVendor** (chain ↔ vendor link)
| Field | Type | Notes |
|-------|------|-------|
| chainId | UUID | The authority's group |
| vendorId | UUID | FK -> Vendor.id |
| status | enum | `approved` \| `removed` |
| addedAt | timestamp | |

**Vendor Summary (read model)**
| Field | Type | Notes |
|-------|------|-------|
| vendorId | UUID | Vendor id |
| vendorName | string | Vendor name |
| rating | number | Rating across group properties (0–5) |
| headcount | integer | Workers across group properties |
| serviceArea | string | Vendor's service area/region |

### 27.5 API / Interface Design
Base path: `/api/hca/vendors` — requires `Bearer <jwt>` with `role = hotel_central_authority`; all responses scoped to the caller's chain.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/vendors` | GET | List the chain's approved vendors (with rating & headcount) |
| `/vendors/nominate` | POST | Nominate a vendor to onboard |
| `/vendors/:id/approve` | PATCH | Approve a vendor onto the chain's approved list |
| `/vendors/:id/remove` | PATCH | Remove a vendor from the chain's approved list |
| `/vendors/discover` | GET | Discover vendors by service area (`?serviceArea=`) |

**POST /vendors/nominate — Request**
```json
{ "businessName": "SparkClean Ltd", "businessId": "GB556677889", "mobile": "+44 7912 445566", "email": "hello@sparkclean.com" }
```
**POST /vendors/nominate — Response 201**
```json
{ "success": true, "nomination": { "id": "nom1", "businessName": "SparkClean Ltd", "status": "nominated" } }
```

**GET /vendors — Response 200**
```json
{
  "items": [
    { "vendorId": "v1", "vendorName": "BrightClean Services",  "rating": 4.6, "headcount": 24, "serviceArea": "London" },
    { "vendorId": "v3", "vendorName": "Summit Maintenance",    "rating": 3.9, "headcount": 11, "serviceArea": "Bristol" }
  ]
}
```

**GET /vendors/discover?serviceArea=London — Response 200**
```json
{
  "serviceArea": "London",
  "items": [
    { "vendorId": "v9",  "vendorName": "CityFix Services", "rating": 4.3, "serviceArea": "London" },
    { "vendorId": "v12", "vendorName": "Metro Housekeeping", "rating": 4.7, "serviceArea": "London" }
  ]
}
```

### 27.6 UI Layout (brief)
- **Approved vendors** (`/hca/vendors`): list of the chain's approved vendors with rating and headcount; **Nominate Vendor** action (form: name, business ID, mobile, email).
- **Vendor row**: **Approve** (pending) / **Remove** (approved) actions.
- **Discover**: search by **service area** returning matching vendors, each with an **Approve/Nominate** action.

### 27.7 Implementation Phases
- **Phase 1 — Mock:** Nominations, approved list, ratings/headcount, and discovery from mock data.
- **Phase 2 — Real Data:** DB-backed nominations and chain approved list; ratings/headcount aggregated across group properties; discovery from the real vendor directory.

### 27.8 Acceptance Criteria
- Authority can nominate a vendor with name, business ID, mobile, and email.
- Authority can approve and remove vendors from the chain's approved list.
- Authority can view vendor rating and headcount across all group properties.
- Authority can discover vendors filtered by service area.
- All data/actions are scoped to the authority's chain; endpoints reject other roles.

### 27.9 Open Questions
- Does nomination require **Superadmin platform approval** (§16) before the vendor can be approved to a chain?
- Is **headcount** the vendor's workers deployed to this chain, or the vendor's total headcount?
- What defines a vendor's **service area** — city, region, radius, or postal zones?
- On **remove**, what happens to that vendor's **active work/requests** in the chain?

---

## 28. Hotel Central Authority — Settings & Permission Module

### 28.1 Overview
**Module #4** of the Hotel Central Authority role (§24.2). It is the **chain settings & permission** hub — controlling how department managers select vendors, the default attendance buffer tolerance, the chain's payment cycle, and notification channel preferences.

Route: `/hca/settings`

**Scope:** all settings apply to the authority's **own chain** (its properties).

> **Settings home:** the "chain-wide default settings" referenced in Property Management (§26) are configured **here**. This module is the single source of truth for chain settings. **[CONFIRM]** the **buffer tolerance unit** — §26 modelled it in *minutes*, whereas this module specifies a **percentage (%)**; this section uses **%** per the latest requirement.

**Actions & Functions**
1. **Set vendor-selection permission** — whether **department managers can select vendors directly** or **must route through the property manager**.
2. **Set default buffer tolerance %** — the chain default for all properties, **overridable per property**.
3. **Set payment cycle** — **7 / 15 / 30 days** for the chain.
4. **Set notification preferences** — SMS, WhatsApp, email channels.

### 28.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-HS1 | Authority can set whether **department managers select vendors directly** or **route through the property manager** | High |
| FR-HS2 | Authority can set a **default buffer tolerance %** applied to all properties | High |
| FR-HS3 | Buffer tolerance % is **overridable per property** | High |
| FR-HS4 | Authority can set the **payment cycle** to `7`, `15`, or `30` days | High |
| FR-HS5 | Authority can set **notification preferences** (SMS / WhatsApp / email on/off) | High |
| FR-HS6 | Settings apply **chain-wide** unless a per-property override exists | High |
| FR-HS7 | All settings are scoped to the authority's **own chain** | High |
| FR-HS8 | Restricted to `role = hotel_central_authority` | High |

### 28.3 Data Model
**ChainSettings**
| Field | Type | Notes |
|-------|------|-------|
| chainId | UUID | The authority's group |
| vendorSelectionMode | enum | `department_manager_direct` \| `via_property_manager` |
| defaultBufferTolerancePercent | number | Default buffer tolerance **%** for all properties |
| paymentCycleDays | enum | `7` \| `15` \| `30` |
| notifications | object | `{ sms: boolean, whatsapp: boolean, email: boolean }` |
| updatedAt | timestamp | |

**PropertyBufferOverride** (per-property override of the default)
| Field | Type | Notes |
|-------|------|-------|
| propertyId | UUID | FK -> Property.id (within the chain) |
| bufferTolerancePercent | number | Overrides the chain default for this property |
| updatedAt | timestamp | |

### 28.4 API / Interface Design
Base path: `/api/hca/settings` — requires `Bearer <jwt>` with `role = hotel_central_authority`; scoped to the caller's chain.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/settings` | GET | Get the chain settings |
| `/settings` | PUT | Update the chain settings |
| `/settings/property-buffer/:propertyId` | PUT | Set a per-property buffer tolerance override |
| `/settings/property-buffer/:propertyId` | DELETE | Remove the override (revert to chain default) |

**PUT /settings — Request**
```json
{
  "vendorSelectionMode": "via_property_manager",
  "defaultBufferTolerancePercent": 10,
  "paymentCycleDays": 15,
  "notifications": { "sms": true, "whatsapp": true, "email": true }
}
```
**PUT /settings — Response 200**
```json
{ "success": true, "chainId": "ch1", "vendorSelectionMode": "via_property_manager", "defaultBufferTolerancePercent": 10, "paymentCycleDays": 15, "notifications": { "sms": true, "whatsapp": true, "email": true } }
```

**PUT /settings/property-buffer/:propertyId — Request**
```json
{ "bufferTolerancePercent": 5 }
```

### 28.5 UI Layout (brief)
- **Settings** (`/hca/settings`) with sections:
  - **Vendor Selection Permission** — toggle/radio: *Department managers select directly* vs *Route through property manager*.
  - **Buffer Tolerance** — default % input, plus a per-property override list.
  - **Payment Cycle** — select 7 / 15 / 30 days.
  - **Notifications** — on/off switches for SMS, WhatsApp, email.

### 28.6 Implementation Phases
- **Phase 1 — Mock:** Settings and per-property overrides from mock state; edits update mock records.
- **Phase 2 — Real Data:** DB-backed chain settings; consumed by vendor-selection flow, attendance/overtime evaluation (buffer), payroll (payment cycle), and notification dispatch.

### 28.7 Acceptance Criteria
- Authority can set whether department managers select vendors directly or via the property manager.
- Authority can set a default buffer tolerance % and override it per property.
- Authority can set the payment cycle to 7, 15, or 30 days.
- Authority can enable/disable SMS, WhatsApp, and email notifications.
- All settings are chain-scoped; endpoints reject other roles.

### 28.8 Open Questions
- **Buffer tolerance unit** — confirm **%** here vs *minutes* in §26; what is the % relative to (shift duration)?
- Do notification preferences use the **templates** defined in the Superadmin Platform Setting (§23), or chain-specific templates?
- Does **payment cycle** here drive payroll/billing, or mirror the Subscription & Billing config (§19)?
- Are there additional **permission toggles** beyond vendor selection (e.g. who can approve work, raise disputes)?

---

## 29. Hotel Central Authority — Reports Module

### 29.1 Overview
**Module #5** of the Hotel Central Authority role (§24.2). It gives the authority **chain-wide reporting** — hours and payroll summaries across all properties, exportable authenticated payment reports for vendors, and cancellation/dispute logs across the chain.

Route: `/hca/reports`

**Scope:** all reports cover **only the authority's own chain** (its properties and vendors).

**Actions & Functions**
1. **View hours and payroll summary reports across all properties** — aggregated hours and payroll for the chain.
2. **Export authenticated payment reports for all vendors** — verified/authenticated vendor payment reports.
3. **View cancellation and dispute logs across the chain** — chain-wide cancellation and dispute records.

### 29.2 Report Types
| Report | Contents |
|--------|----------|
| Hours | Worked hours across all chain properties (by property/worker). |
| Payroll Summary | Payroll totals/summaries across the chain. |
| Vendor Payment (authenticated) | Verified/authenticated payment reports per vendor, exportable. |
| Cancellation Log | Cancelled requests/shifts across the chain (reason, actor). |
| Dispute Log | Disputes and outcomes across the chain (from Dispute Resolution). |

### 29.3 Scope & Filters
| Filter | Values |
|--------|--------|
| propertyId | A chain property (omit for all properties) |
| vendorId | A chain vendor (for vendor payment reports) |
| dateFrom / dateTo | Reporting period |
| format | `csv` \| `pdf` \| `xlsx` |

### 29.4 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-HR1 | Authority can view **hours** reports across all chain properties | High |
| FR-HR2 | Authority can view **payroll summary** reports across all chain properties | High |
| FR-HR3 | Authority can **export authenticated payment reports** for all vendors | High |
| FR-HR4 | Authority can view **cancellation logs** across the chain | High |
| FR-HR5 | Authority can view **dispute logs** across the chain | High |
| FR-HR6 | Reports are filterable by property, vendor, and date range | Medium |
| FR-HR7 | Reports export in CSV/PDF/XLSX | Medium |
| FR-HR8 | All reports are scoped to the authority's **own chain** | High |
| FR-HR9 | Restricted to `role = hotel_central_authority` | High |

### 29.5 Data Model
**ReportRequest** (chain-scoped)
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| chainId | UUID | The authority's group (enforced) |
| reportType | enum | `hours` \| `payroll_summary` \| `vendor_payment` \| `cancellation_log` \| `dispute_log` |
| propertyId | UUID (nullable) | Filter to a property; `null` = all chain properties |
| vendorId | UUID (nullable) | For vendor payment reports |
| dateFrom | date | Period start |
| dateTo | date | Period end |
| format | enum | `csv` \| `pdf` \| `xlsx` |
| status | enum | `ready` \| `processing` \| `failed` |
| fileUrl | string (nullable) | Download link when ready |
| authenticated | boolean | For vendor payment reports — verified/signed |
| createdAt | timestamp | |

> **[CONFIRM]** what "**authenticated** payment report" means — verified against payroll records, digitally signed, or approval-stamped.

### 29.6 API / Interface Design
Base path: `/api/hca/reports` — requires `Bearer <jwt>` with `role = hotel_central_authority`; all data scoped to the caller's chain.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/hours` | GET | Hours report across chain properties |
| `/payroll-summary` | GET | Payroll summary across the chain |
| `/vendor-payments` | GET | Export authenticated vendor payment reports (`?vendorId=`, `?format=`) |
| `/cancellations` | GET | Cancellation log across the chain |
| `/disputes` | GET | Dispute log across the chain |
| `/generate` | POST | Generate/export a report (type + filters + format) |

**GET /payroll-summary — Response 200**
```json
{
  "chainId": "ch1",
  "period": "2026-06",
  "totalHours": 12480,
  "totalPayroll": 156200,
  "byProperty": [
    { "propertyName": "Sunrise Apartments", "hours": 5200, "payroll": 65000 },
    { "propertyName": "Palm Villa",         "hours": 7280, "payroll": 91200 }
  ]
}
```

**GET /vendor-payments — Response 200**
```json
{
  "format": "pdf",
  "authenticated": true,
  "items": [
    { "vendorId": "v1", "vendorName": "BrightClean Services",  "period": "2026-06", "amount": 42000, "fileUrl": "/downloads/hca/payments/v1-2026-06.pdf" },
    { "vendorId": "v3", "vendorName": "Summit Maintenance",    "period": "2026-06", "amount": 18500, "fileUrl": "/downloads/hca/payments/v3-2026-06.pdf" }
  ]
}
```

### 29.7 UI Layout (brief)
- **Reports** (`/hca/reports`) with tabs/sections: **Hours**, **Payroll Summary**, **Vendor Payments** (export), **Cancellations**, **Disputes**.
- Each with property/vendor/date-range filters and **View / Export** actions.
- Vendor Payments shows an **Authenticated** badge and an **Export** button per vendor (and export-all).

### 29.8 Implementation Phases
- **Phase 1 — Mock:** Reports and exports from mock data scoped to a mock chain; export returns sample files.
- **Phase 2 — Real Data:** Real aggregation across the chain's properties/vendors; authenticated payment reports generated from payroll records.

### 29.9 Acceptance Criteria
- Authority can view hours and payroll summary reports across all chain properties.
- Authority can export authenticated payment reports for vendors.
- Authority can view cancellation and dispute logs across the chain.
- All reports are chain-scoped; endpoints reject other roles.

### 29.10 Open Questions
- Definition of **"authenticated"** payment report (verified vs signed vs approved)?
- Do payroll/payment figures come from **Subscription & Billing (§19)** / a payroll system, or are they computed here?
- Are large exports **synchronous** or **async jobs**?
- Should dispute/cancellation logs **link through** to the underlying records?

---

## 30. Hotel Central Authority — Work Request Module

### 30.1 Overview
**Module #6** of the Hotel Central Authority role (§24.2). It gives the authority **read-only oversight** of work requests across all its chain's properties. The authority **cannot raise or approve** requests — that is the **Property Manager's** role; this module is for visibility and monitoring only.

Route: `/hca/work-requests`

**Scope:** view-only, limited to the authority's **own chain** properties.

**Actions & Functions**
1. **View all work requests across all properties** — every request in the chain, with its status and bidding.
2. **No create / approve** — the authority **cannot raise or approve** work requests (Property Manager responsibility). This module is strictly **read-only**.

### 30.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-HW1 | Authority can view all work requests across all chain properties | High |
| FR-HW2 | Work requests are filterable by property, vendor, and status | Medium |
| FR-HW3 | Authority can view a single work request's detail, including bidding history | High |
| FR-HW4 | Authority **cannot create/raise** a work request | High |
| FR-HW5 | Authority **cannot approve/award/act on** a work request | High |
| FR-HW6 | Module is **read-only**; no write endpoints are exposed to this role | High |
| FR-HW7 | All data is scoped to the authority's **own chain** | High |
| FR-HW8 | Restricted to `role = hotel_central_authority` | High |

### 30.3 Data Model (read model)
**WorkRequest (read-only view)** — see the full model in §18.4.
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Work request id |
| propertyName | string | Chain property |
| title | string | Work summary |
| status | enum | `open` \| `bidding` \| `awarded` \| `in_progress` \| `completed` \| `cancelled` \| `stalled` \| `escalated` |
| awardedVendorName | string (nullable) | Winning vendor, if awarded |
| bidCount | integer | Number of bids |
| createdBy | string | Requester (e.g. Property Manager) |
| createdAt | timestamp | |

### 30.4 API / Interface Design
Base path: `/api/hca/work-requests` — requires `Bearer <jwt>` with `role = hotel_central_authority`; **read-only**, scoped to the caller's chain.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/work-requests` | GET | List all chain work requests (filter by `?propertyId=`, `?vendorId=`, `?status=`) |
| `/work-requests/:id` | GET | View a work request's detail |
| `/work-requests/:id/bids` | GET | View bidding history (read-only) |

> No `POST`/`PUT`/`PATCH`/`DELETE` endpoints — raising and approving requests is the Property Manager's role.

**GET /work-requests — Response 200**
```json
{
  "items": [
    { "id": "wr1", "propertyName": "Sunrise Apartments", "title": "Deep clean lobby", "status": "bidding",  "awardedVendorName": null,                 "bidCount": 3 },
    { "id": "wr3", "propertyName": "Palm Villa",          "title": "Garden upkeep",    "status": "awarded",  "awardedVendorName": "BrightClean Services", "bidCount": 5 }
  ]
}
```

### 30.5 UI Layout (brief)
- **Work requests** (`/hca/work-requests`): read-only table of all chain requests with property, title, status, awarded vendor, bid count; filters for property / vendor / status.
- **Request detail**: full info and bidding history — **no action buttons** (view-only).

### 30.6 Implementation Phases
- **Phase 1 — Mock:** Read-only list/detail served from mock data scoped to a mock chain.
- **Phase 2 — Real Data:** Read-only aggregation across the chain's properties; write operations remain with the Property Manager role.

### 30.7 Acceptance Criteria
- Authority can view all work requests across the chain with status and bidding.
- Authority has **no** create/approve/act controls in this module.
- Only read endpoints are available; write attempts are rejected/absent.
- All data is chain-scoped; endpoints reject other roles.

### 30.8 Open Questions
- Should the authority be able to **filter/sort** by stalled/escalated to spot problems (view-only)?
- Does the authority get **notified** of stalled/escalated requests (links to Dashboard alerts, §25)?
- Any **read-only drill-through** to disputes/attendance from a request?

---
---

# Property Manager Role

> **§1–23 specify Superadmin; §24–30 specify Hotel Central Authority.** The sections below (§31+) specify the **Property Manager** role.

## 31. Property Manager — Role Overview

### 31.1 Overview
The **Property Manager** runs the day-to-day operations of a **single property** within a hotel group. It provisions and oversees the property's departments, workers, and vendors, and is the role that **raises and approves work requests** (the Hotel Central Authority only observes — §30).

**Authentication:** login is shared across roles (see §1–13). On successful login where the JWT `role = property_manager`, the app redirects to the **Property Manager Dashboard** (§32).

**Scope rule:** every action in this role is limited to the **manager's own property** — its departments, workers, vendors, work requests, attendance, and reports.

### 31.2 Property Manager Modules
The role is organised into **7 functional modules**. Each has its **own actions and functions** — those will be added as they are shared.

| # | Module | Purpose (summary) | Actions & Functions | Detailed spec |
|---|--------|-------------------|---------------------|---------------|
| 1 | Dashboard | Property-level overview and alerts | ✅ Defined | §32 |
| 2 | Work Request Management | Raise/approve/edit requests; select vendors; cancel per policy | ✅ Defined | §33 |
| 3 | Vendor Management | View property vendors; nominate (if permitted); subscription/ratings/workers | ✅ Defined | §34 |
| 4 | Worker Management | View workers, live attendance/overtime; raise disputes; rate workers/vendors | ✅ Defined | §35 |
| 5 | Attendance & Hours | QR/geofence attendance logs; overtime pre-approve/query; hours report | ✅ Defined | §36 |
| 6 | Settings | Override chain defaults (buffer/payment/notifications); DM vendor-selection toggle | ✅ Defined | §37 |
| 7 | Reports | Hours/payroll; authenticated vendor invoice cross-check; dispute/cancellation history | ✅ Defined | §38 |

> **Status legend:** ✅ Defined · ⏳ Awaiting the actions/functions you'll share.
>
> All 7 modules are defined (§32–§38). Each module section includes its actions/functions, FRs, data model, API, UI, phases, acceptance criteria, and open questions.

### 31.3 Open Questions
- Can a Property Manager be assigned to **more than one property**, or exactly one?
- Which chain-wide defaults (from §28) are **inherited vs overridable** at the property level?
- Does Work Request Management here **create** requests that the §30 authority observes read-only? (Expected: yes.)

---

## 32. Property Manager — Dashboard Module

### 32.1 Overview
**Module #1** of the Property Manager role (§31.2). It is the landing view after login and gives the manager a **property-level** overview — worked hours, active users, open work requests by status, and overtime/attendance alerts for the property.

Route: `/property-manager/dashboard`

**Scope:** all figures are for the manager's **own property** only.

**Actions & Functions**
1. **View today's work hours, current month hours, active property users** — headline stat cards.
2. **View open work requests with status** — New, Pending, In Progress, Accepted, Declined.
3. **View overtime flags and attendance alerts** — for the property.

### 32.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-PD1 | On login with `role = property_manager`, redirect to the Property Manager dashboard | High |
| FR-PD2 | Dashboard accessible only to `property_manager`; scoped to the own property | High |
| FR-PD3 | Show stat cards: **Today's Hours, Current Month Hours, Active Property Users** | High |
| FR-PD4 | Metrics are fetched from the backend (not hardcoded) | High |
| FR-PD5 | Show **open work requests** with status: New, Pending, In Progress, Accepted, Declined | High |
| FR-PD6 | Show **overtime flags** and **attendance alerts** for the property | High |
| FR-PD7 | All data is limited to the manager's property (no other properties) | High |
| FR-PD8 | Show empty, loading, and error states | Medium |

### 32.3 UI Layout
```
+-----------------------------------------------------------------------+
|  Property Manager Dashboard                          [ Sunrise Apts ]  |
+-----------------------------------------------------------------------+
|  +---------------+ +-------------------+ +-----------------------+     |
|  | Today's Hours | | This Month Hours  | | Active Property Users |     |
|  |     186       | |      4,120        | |          42           |     |
|  +---------------+ +-------------------+ +-----------------------+     |
|                                                                       |
|  +-------------------------------+  +------------------------------+  |
|  | Open Work Requests            |  | Overtime / Attendance Alerts |  |
|  |-------------------------------|  |------------------------------|  |
|  | Deep clean lobby   New        |  | ! Overtime  J.Smith   1.5h   |  |
|  | AC repair          In Progress|  | ! Absent    R.Kumar   AM     |  |
|  | Garden upkeep      Accepted   |  | ! Late      J.Doe     08:20  |  |
|  | Window cleaning    Declined   |  |                              |  |
|  +-------------------------------+  +------------------------------+  |
+-----------------------------------------------------------------------+
```

### 32.4 Stat Cards
| Card | Meaning (property-scoped) |
|------|---------------------------|
| Today's Hours | Sum of worked hours today at the property |
| Current Month Hours | Sum of worked hours this month at the property |
| Active Property Users | Count of active users (workers/staff) at the property |

### 32.5 Work Request Status (dashboard view)
| Status | Meaning |
|--------|---------|
| New | Just raised, not yet actioned |
| Pending | Awaiting response/bids |
| In Progress | Work underway |
| Accepted | Request accepted/awarded |
| Declined | Request declined/rejected |

> **[CONFIRM]** how these map to the platform work-request lifecycle in §18.2 (`open`/`bidding`/`awarded`/`in_progress`/`completed`/`cancelled`).

### 32.6 Data Model (read models)
**Property Stats (computed)**
| Field | Type | Notes |
|-------|------|-------|
| todayHours | number | Worked hours today |
| monthHours | number | Worked hours this month |
| activeUsers | integer | Active property users |
| period | string | Current month reference |

**Open Work Request item**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Work request id |
| title | string | Work summary |
| status | enum | `new` \| `pending` \| `in_progress` \| `accepted` \| `declined` |
| createdAt | timestamp | |

**Alert item**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Alert id |
| type | enum | `overtime` \| `attendance` |
| workerName | string | Worker involved |
| detail | string | e.g. overtime hours, absent, late |
| severity | enum | `info` \| `warning` \| `critical` |
| createdAt | timestamp | |

### 32.7 API / Interface Design
Base path: `/api/pm/dashboard` — requires `Bearer <jwt>` with `role = property_manager`; all responses scoped to the manager's property.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/stats` | GET | Today's hours, month hours, and active users |
| `/work-requests` | GET | Open work requests with status |
| `/alerts` | GET | Overtime and attendance alerts (filter by `?type=`) |

**GET /stats — Response 200**
```json
{ "todayHours": 186, "monthHours": 4120, "activeUsers": 42, "period": "2026-07" }
```

**GET /work-requests — Response 200**
```json
{
  "items": [
    { "id": "wr1", "title": "Deep clean lobby", "status": "new",         "createdAt": "2026-07-10T08:00:00Z" },
    { "id": "wr2", "title": "AC repair",        "status": "in_progress", "createdAt": "2026-07-09T14:00:00Z" },
    { "id": "wr3", "title": "Garden upkeep",    "status": "accepted",    "createdAt": "2026-07-08T10:00:00Z" }
  ]
}
```

**GET /alerts — Response 200**
```json
{
  "items": [
    { "id": "al1", "type": "overtime",   "workerName": "Jane Smith", "detail": "1.5h overtime", "severity": "warning",  "createdAt": "2026-07-10T16:30:00Z" },
    { "id": "al2", "type": "attendance", "workerName": "Ravi Kumar", "detail": "Absent (AM)",   "severity": "critical", "createdAt": "2026-07-10T09:05:00Z" }
  ]
}
```

### 32.8 Implementation Phases
- **Phase 1 — Mock:** Stats, work requests, and alerts from mock data scoped to a mock property; full UI built against the mock API.
- **Phase 2 — Real Data:** Real property aggregation; work requests and alerts from their source modules, filtered to the property.

### 32.9 Acceptance Criteria
- Login as `property_manager` redirects to the Property Manager dashboard.
- Stat cards show today's hours, current month hours, and active property users.
- Open work requests list shows the five statuses (New/Pending/In Progress/Accepted/Declined).
- Overtime flags and attendance alerts for the property are shown.
- No data outside the manager's property is visible; empty/loading/error states handled.

### 32.10 Open Questions
- Mapping of dashboard statuses to the platform lifecycle (§18.2) — confirm New/Pending/Accepted/Declined semantics.
- "Active Property Users" — currently clocked-in workers, or all active-status users at the property?
- Should alerts be **actionable** (click through to attendance/hours §36), or view-only on the dashboard?
- Reporting period for "Current Month Hours" — calendar month, or rolling 30 days?

---

## 33. Property Manager — Work Request Management Module

### 33.1 Overview
**Module #2** of the Property Manager role (§31.2). It is the core workflow module: the manager **raises** work requests, **approves/rejects** requests raised by department managers, **edits** requests before sending, **selects vendors** (from the chain's approved list or by discovery), reviews vendor rating/headcount before selecting, and **cancels** requests per policy.

Route: `/property-manager/work-requests`

**Scope:** the manager's **own property** only.

**Actions & Functions**
1. **Raise new work requests** — start/end date, shift, designation, headcount, buffer %.
2. **Approve or reject work requests raised by department managers.**
3. **Edit requests before sending to vendors** — change headcount, dates, vendor routing.
4. **Select vendors** — from the existing approved list, or discover new vendors by area.
5. **View vendor ratings and headcount before selection.**
6. **Cancel work requests** — subject to the cancellation policy.

### 33.2 Work Request Lifecycle (PM view)
| Status | Meaning |
|--------|---------|
| draft | Being prepared by the PM |
| pending_approval | Raised by a Department Manager, awaiting PM approval |
| approved | Approved by the PM (ready to send) |
| rejected | Rejected by the PM (DM-raised) |
| sent_to_vendor | Sent to the selected vendor(s) |
| accepted | Vendor accepted |
| in_progress | Work underway |
| completed | Work finished |
| cancelled | Cancelled per policy |

> **[CONFIRM]** alignment with the platform lifecycle (§18.2) and the dashboard view (§32.5).

### 33.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-PW1 | PM can **raise** a work request with start/end date, shift, designation, headcount, and buffer % | High |
| FR-PW2 | PM can **approve** a work request raised by a department manager | High |
| FR-PW3 | PM can **reject** a department-manager request (with reason) | High |
| FR-PW4 | PM can **edit** a request before sending (headcount, dates, vendor routing) | High |
| FR-PW5 | PM can **select vendors from the approved list** | High |
| FR-PW6 | PM can **discover new vendors by area** and select them | High |
| FR-PW7 | PM can **view vendor rating and headcount** before selection | High |
| FR-PW8 | PM can **send** the request to the selected vendor(s) | High |
| FR-PW9 | PM can **cancel** a request, subject to the **cancellation policy** | High |
| FR-PW10 | All requests/actions are scoped to the manager's **property** | High |
| FR-PW11 | Restricted to `role = property_manager` | High |

### 33.4 Data Model
**WorkRequest** (PM-managed)
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| propertyId | UUID | The manager's property (enforced) |
| designation | string | Job role/designation needed (e.g. Housekeeper, Electrician) |
| startDate | date | Work start |
| endDate | date | Work end |
| shift | enum | `morning` \| `evening` \| `night` |
| headcount | integer | Number of workers required |
| bufferPercent | number | Buffer tolerance % (defaults from §28) |
| vendorRouting | enum | `direct` \| `broadcast` \| `preferred` (how the request reaches vendors) |
| raisedByType | enum | `property_manager` \| `department_manager` |
| raisedById | UUID | Who raised it |
| approvalStatus | enum | `not_required` \| `pending` \| `approved` \| `rejected` (for DM-raised) |
| selectedVendorIds | UUID[] | Chosen vendor(s) |
| status | enum | `draft` \| `pending_approval` \| `approved` \| `rejected` \| `sent_to_vendor` \| `accepted` \| `in_progress` \| `completed` \| `cancelled` |
| rejectionReason | string (nullable) | If rejected |
| cancellationReason | string (nullable) | If cancelled |
| createdAt | timestamp | |

**Vendor option (selection read model)** — from the chain approved list (§27) / discovery:
| Field | Type | Notes |
|-------|------|-------|
| vendorId | UUID | Vendor id |
| vendorName | string | Vendor name |
| rating | number | Rating (0–5) |
| headcount | integer | Available/total workers |
| serviceArea | string | Service area |
| source | enum | `approved_list` \| `discovered` |

### 33.5 API / Interface Design
Base path: `/api/pm/work-requests` — requires `Bearer <jwt>` with `role = property_manager`; scoped to the manager's property.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/work-requests` | GET | List the property's work requests (filter by `?status=`) |
| `/work-requests` | POST | Raise a new work request |
| `/work-requests/:id` | PUT | Edit a request before sending (headcount, dates, routing) |
| `/work-requests/:id/approve` | PATCH | Approve a department-manager request |
| `/work-requests/:id/reject` | PATCH | Reject a department-manager request (with reason) |
| `/work-requests/:id/select-vendor` | PATCH | Select vendor(s) and send the request |
| `/work-requests/:id/cancel` | PATCH | Cancel a request (policy-checked) |
| `/vendors` | GET | Approved vendors with rating & headcount (for selection) |
| `/vendors/discover` | GET | Discover vendors by area (`?area=`) with rating & headcount |

**POST /work-requests — Request**
```json
{
  "designation": "Housekeeper",
  "startDate": "2026-07-15",
  "endDate": "2026-07-20",
  "shift": "morning",
  "headcount": 4,
  "bufferPercent": 10,
  "vendorRouting": "preferred"
}
```
**POST /work-requests — Response 201**
```json
{ "success": true, "workRequest": { "id": "wr10", "designation": "Housekeeper", "headcount": 4, "status": "draft" } }
```

**PATCH /work-requests/:id/select-vendor — Request**
```json
{ "vendorIds": ["v1"] }
```
**PATCH /work-requests/:id/select-vendor — Response 200**
```json
{ "success": true, "workRequestId": "wr10", "selectedVendorIds": ["v1"], "status": "sent_to_vendor" }
```

**GET /vendors — Response 200**
```json
{
  "items": [
    { "vendorId": "v1", "vendorName": "BrightClean Services", "rating": 4.6, "headcount": 24, "serviceArea": "London", "source": "approved_list" },
    { "vendorId": "v3", "vendorName": "Summit Maintenance",   "rating": 3.9, "headcount": 11, "serviceArea": "London", "source": "approved_list" }
  ]
}
```

**PATCH /work-requests/:id/cancel — Request**
```json
{ "reason": "Booking postponed by client" }
```
> Cancellation is validated against the **cancellation policy** (e.g. notice window, penalties) — **[CONFIRM]** the policy rules.

### 33.6 UI Layout (brief)
- **Work requests list** (`/property-manager/work-requests`): the property's requests with designation, dates, shift, headcount, status; **Raise Request** action; DM-raised items show **Approve / Reject**.
- **Raise/Edit form**: start/end date, shift, designation, headcount, buffer %, vendor routing.
- **Vendor selection**: tabs for **Approved list** and **Discover by area**, each showing **rating** and **headcount**; select vendor(s) → **Send**.
- **Cancel**: action with reason, gated by the cancellation policy.

### 33.7 Implementation Phases
- **Phase 1 — Mock:** Requests, approvals, vendor selection, and cancellation from mock data; policy checks stubbed.
- **Phase 2 — Real Data:** DB-backed requests; DM-approval flow; vendor selection from the chain approved list/discovery; cancellation policy enforced.

### 33.8 Acceptance Criteria
- PM can raise a work request with all specified fields (dates, shift, designation, headcount, buffer %).
- PM can approve/reject department-manager requests (rejection captures a reason).
- PM can edit a request before sending (headcount, dates, vendor routing).
- PM can select vendors from the approved list or by discovery, seeing rating and headcount.
- PM can cancel a request subject to the cancellation policy.
- All actions are property-scoped; endpoints reject other roles.

### 33.9 Open Questions
- **Cancellation policy** — notice window, penalties/fees, and which statuses are cancellable?
- **Vendor routing** options — direct to one vendor, broadcast to many, or preferred-order? Full set?
- Does DM-raised approval also allow the PM to **edit before approving**, or approve/reject only?
- Can multiple vendors be selected (split headcount), or exactly one per request?
- Buffer % default — inherited from chain settings (§28) and editable per request?

---

## 34. Property Manager — Vendor Management Module

### 34.1 Overview
**Module #3** of the Property Manager role (§31.2). It lets the manager see the vendors associated with the property, nominate new vendors (when chain settings permit), and review each vendor's subscription status, ratings, and active workers.

Route: `/property-manager/vendors`

**Scope:** vendors associated with the manager's **own property**.

**Actions & Functions**
1. **View all vendors associated with the property.**
2. **Nominate new vendors to the property** — only **if permitted by chain settings** (§28).
3. **View vendor subscription status, ratings, and active workers.**

### 34.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-PV1 | PM can view all vendors associated with the property | High |
| FR-PV2 | PM can **nominate** a new vendor to the property **if permitted by chain settings** | High |
| FR-PV3 | The nominate action is hidden/blocked when chain settings disallow it | High |
| FR-PV4 | PM can view each vendor's **subscription status**, **rating**, and **active workers** | High |
| FR-PV5 | All vendor data/actions are scoped to the manager's **property** | High |
| FR-PV6 | Restricted to `role = property_manager` | High |

> **Permission gate:** whether a PM may nominate vendors is controlled by the chain's Settings & Permission (§28). **[CONFIRM]** the exact chain setting that governs PM vendor nomination.

### 34.3 Data Model
**Vendor (property association, read model)**
| Field | Type | Notes |
|-------|------|-------|
| vendorId | UUID | Vendor id |
| vendorName | string | Vendor name |
| subscriptionStatus | enum | `active` \| `trial` \| `past_due` \| `expired` \| `none` |
| rating | number | Rating at the property (0–5) |
| activeWorkers | integer | Vendor's workers currently active at the property |
| status | enum | `active` \| `inactive` (association state) |

**VendorNomination (property-level)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| propertyId | UUID | The manager's property |
| businessName | string | Vendor business name |
| businessId | string | Business/registration ID |
| mobile | string | Contact mobile |
| email | string | Contact email |
| status | enum | `nominated` \| `onboarding` \| `approved` \| `rejected` |
| nominatedBy | UUID | Property Manager user id |
| createdAt | timestamp | |

### 34.4 API / Interface Design
Base path: `/api/pm/vendors` — requires `Bearer <jwt>` with `role = property_manager`; scoped to the manager's property.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/vendors` | GET | List vendors associated with the property (subscription, rating, active workers) |
| `/vendors/:id` | GET | Get a vendor's detail |
| `/vendors/nominate` | POST | Nominate a new vendor to the property (**403 if chain settings disallow**) |

**GET /vendors — Response 200**
```json
{
  "items": [
    { "vendorId": "v1", "vendorName": "BrightClean Services", "subscriptionStatus": "active",  "rating": 4.6, "activeWorkers": 8 },
    { "vendorId": "v3", "vendorName": "Summit Maintenance",   "subscriptionStatus": "expired", "rating": 3.9, "activeWorkers": 0 }
  ]
}
```

**POST /vendors/nominate — Request**
```json
{ "businessName": "SparkClean Ltd", "businessId": "GB556677889", "mobile": "+44 7912 445566", "email": "hello@sparkclean.com" }
```
**POST /vendors/nominate — Response 201 / 403**
```json
{ "success": true, "nomination": { "id": "nom5", "businessName": "SparkClean Ltd", "status": "nominated" } }
```
```json
{ "success": false, "message": "Vendor nomination is disabled by chain settings" }
```

### 34.5 UI Layout (brief)
- **Vendors** (`/property-manager/vendors`): list of the property's vendors with subscription badge, rating, and active-worker count.
- **Nominate Vendor** action (form: name, business ID, mobile, email) — **shown only when chain settings permit**.
- **Vendor detail**: subscription, rating, and active workers at the property.

### 34.6 Implementation Phases
- **Phase 1 — Mock:** Property vendors and nominations from mock data; nomination permission read from a mock chain setting.
- **Phase 2 — Real Data:** DB-backed property↔vendor associations; nomination gated by the real chain setting (§28); subscription/rating/active-workers from source modules.

### 34.7 Acceptance Criteria
- PM can view all vendors associated with the property with subscription status, rating, and active workers.
- PM can nominate a new vendor **only when chain settings permit**; the action is blocked otherwise.
- All vendor data/actions are property-scoped; endpoints reject other roles.

### 34.8 Open Questions
- Which chain setting (§28) governs **PM vendor nomination** — is it distinct from the department-manager vendor-selection permission?
- Does a PM nomination require **chain (HCA) or Superadmin approval** before the vendor is usable?
- Is **active workers** counted at the property only, or the vendor's total active workers?
- Should the PM be able to **remove/deactivate** a vendor association, or only view/nominate?

---

## 35. Property Manager — Worker Management Module

### 35.1 Overview
**Module #4** of the Property Manager role (§31.2). It lets the manager oversee workers assigned to the property — viewing them, seeing live check-in/out and overtime, raising hour disputes with evidence, and rating workers and vendor agencies.

Route: `/property-manager/workers`

**Scope:** workers assigned to the manager's **own property**.

**Cross-module links:** disputes raised here feed **Dispute Resolution (§21)**; ratings feed **Rating & Trust (§20)**.

**Actions & Functions**
1. **View all workers assigned to the property.**
2. **See worker check-in/check-out status and live overtime.**
3. **Raise a dispute against specific hours for a worker** — with evidence upload.
4. **Rate individual workers at the end of each shift.**
5. **Rate the vendor agency at the end of the assignment.**

### 35.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-PWK1 | PM can view all workers assigned to the property | High |
| FR-PWK2 | PM can see each worker's **check-in/check-out status** and **live overtime** | High |
| FR-PWK3 | PM can **raise a dispute** against specific hours for a worker | High |
| FR-PWK4 | Dispute supports **evidence upload** (photos/documents) | High |
| FR-PWK5 | PM can **rate a worker** at the end of each shift (1–5 + comment) | High |
| FR-PWK6 | PM can **rate the vendor agency** at the end of the assignment (1–5 + comment) | High |
| FR-PWK7 | All data/actions are scoped to the manager's **property** | High |
| FR-PWK8 | Restricted to `role = property_manager` | High |

### 35.3 Data Model
**Worker (property assignment, read model)**
| Field | Type | Notes |
|-------|------|-------|
| workerId | UUID | Worker id |
| name | string | Worker name |
| vendorName | string | Vendor/agency the worker belongs to |
| shift | enum | `morning` \| `evening` \| `night` |
| attendanceStatus | enum | `checked_in` \| `checked_out` \| `absent` \| `not_started` |
| checkInTime | timestamp (nullable) | Actual check-in |
| checkOutTime | timestamp (nullable) | Actual check-out (null if active) |
| liveOvertimeHours | number | Overtime accrued so far (live) |

**Dispute (raised here → §21)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| workerId | UUID | Worker in dispute |
| shiftId | UUID | Shift/hours being disputed |
| claimedHours | number | Hours the PM asserts |
| recordedHours | number | System-recorded hours |
| reason | string | Description |
| evidence | string[] | Uploaded file URLs |
| status | enum | `open` (then handled per §21) |
| raisedBy | UUID | Property Manager id |
| createdAt | timestamp | |

**WorkerRating / VendorRating (→ §20)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| subjectType | enum | `worker` \| `vendor` |
| subjectId | UUID | Worker or vendor id |
| shiftId | UUID (nullable) | For worker (per-shift) ratings |
| assignmentId | UUID (nullable) | For vendor (per-assignment) ratings |
| score | number | 1–5 |
| comment | string (nullable) | Review text |
| ratedBy | UUID | Property Manager id |
| createdAt | timestamp | |

### 35.4 API / Interface Design
Base path: `/api/pm/workers` — requires `Bearer <jwt>` with `role = property_manager`; scoped to the manager's property.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/workers` | GET | List workers assigned to the property (with attendance & live overtime) |
| `/workers/:id` | GET | Get a worker's detail |
| `/workers/:id/dispute` | POST | Raise an hours dispute (multipart; evidence upload) |
| `/workers/:id/rating` | POST | Rate a worker for a shift |
| `/vendors/:id/rating` | POST | Rate a vendor agency for an assignment |

**GET /workers — Response 200**
```json
{
  "items": [
    { "workerId": "w1", "name": "John Doe",  "vendorName": "BrightClean Services", "shift": "morning", "attendanceStatus": "checked_in",  "checkInTime": "08:02", "checkOutTime": null,    "liveOvertimeHours": 0 },
    { "workerId": "w2", "name": "Jane Smith", "vendorName": "BrightClean Services", "shift": "evening", "attendanceStatus": "checked_out", "checkInTime": "14:00", "checkOutTime": "22:30", "liveOvertimeHours": 1.5 }
  ]
}
```

**POST /workers/:id/dispute — Request** (`multipart/form-data`)
```
shiftId: s2
claimedHours: 7.5
recordedHours: 9
reason: Worker left early; recorded hours overstated
evidence: <file1>, <file2>
```
**POST /workers/:id/dispute — Response 201**
```json
{ "success": true, "dispute": { "id": "d10", "workerId": "w2", "shiftId": "s2", "status": "open" } }
```

**POST /workers/:id/rating — Request**
```json
{ "shiftId": "s1", "score": 5, "comment": "Punctual and thorough" }
```

**POST /vendors/:id/rating — Request**
```json
{ "assignmentId": "as4", "score": 4, "comment": "Reliable staffing, minor delays" }
```

### 35.5 UI Layout (brief)
- **Workers** (`/property-manager/workers`): list of assigned workers with vendor, shift, attendance status, live overtime; live-updating.
- **Worker detail/actions**: **Raise Dispute** (hours + reason + evidence upload), **Rate Worker** (end of shift).
- **Vendor rating**: **Rate Agency** action at end of assignment.

### 35.6 Implementation Phases
- **Phase 1 — Mock:** Workers, attendance/overtime, disputes, and ratings from mock data; evidence upload stubbed.
- **Phase 2 — Real Data:** Live attendance/overtime from the attendance system; disputes created into §21; ratings written to §20; evidence stored.

### 35.7 Acceptance Criteria
- PM can view all workers assigned to the property with check-in/out status and live overtime.
- PM can raise an hours dispute for a worker with evidence upload.
- PM can rate a worker at the end of each shift and rate the vendor agency at end of assignment.
- All data/actions are property-scoped; endpoints reject other roles.

### 35.8 Open Questions
- Is worker rating **mandatory** at shift end, and is there a rating window/deadline?
- Can the PM rate a **vendor** multiple times (per assignment) or once per period?
- Evidence upload — allowed **file types/size limits**, and multiple files?
- Does raising a dispute here **pause payroll** for the disputed hours until §21 resolves it?

---

## 36. Property Manager — Attendance & Hours Module

### 36.1 Overview
**Module #5** of the Property Manager role (§31.2). It gives the manager the attendance and hours record for the property — QR-based attendance logs per worker per shift, geofence backup data when a QR scan is missing, overtime pre-approval/query handling, and downloadable hours reports.

Route: `/property-manager/attendance`

**Scope:** attendance/hours for the manager's **own property**. QR/geofence data here is the same source used by Dispute Resolution (§21).

**Actions & Functions**
1. **View QR-based attendance logs** — per worker, per shift.
2. **View geofence backup data** — where a QR scan is absent.
3. **Handle overtime** — flag overtime as **pre-approved**, or **query** overtime that was not pre-approved.
4. **Download hours report** — for any period.

### 36.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-PA1 | PM can view **QR-based attendance logs** per worker per shift | High |
| FR-PA2 | PM can view **geofence backup data** where the QR scan is absent | High |
| FR-PA3 | PM can flag overtime as **pre-approved** | High |
| FR-PA4 | PM can **query** overtime that was not pre-approved | High |
| FR-PA5 | PM can **download an hours report** for any period (date range) | High |
| FR-PA6 | Hours report exports in CSV/PDF/XLSX | Medium |
| FR-PA7 | All data/actions are scoped to the manager's **property** | High |
| FR-PA8 | Restricted to `role = property_manager` | High |

### 36.3 Overtime Status
| Status | Meaning |
|--------|---------|
| none | No overtime on the shift |
| pending | Overtime accrued, not yet actioned |
| pre_approved | Overtime approved by the PM |
| queried | PM has queried/challenged the overtime |

> Querying overtime may open or feed a dispute (§21). **[CONFIRM]** whether "query" creates a dispute or is a lightweight flag.

### 36.4 Data Model
**AttendanceLog** (per worker per shift)
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| workerId | UUID | Worker id |
| workerName | string | Worker name |
| shiftId | UUID | Shift id |
| date | date | Shift date |
| shift | enum | `morning` \| `evening` \| `night` |
| qrCheckIn | timestamp (nullable) | QR scan-in |
| qrCheckOut | timestamp (nullable) | QR scan-out |
| geofenceCheckIn | timestamp (nullable) | Geofence backup in (if QR absent) |
| geofenceCheckOut | timestamp (nullable) | Geofence backup out (if QR absent) |
| source | enum | `qr` \| `geofence` (which provided the record) |
| totalHours | number | Computed worked hours |
| overtimeHours | number | Overtime portion |
| overtimeStatus | enum | `none` \| `pending` \| `pre_approved` \| `queried` |

### 36.5 API / Interface Design
Base path: `/api/pm/attendance` — requires `Bearer <jwt>` with `role = property_manager`; scoped to the manager's property.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/logs` | GET | Attendance logs per worker/shift (filter by `?workerId=`, `?date=`, `?shift=`) |
| `/logs/:id` | GET | Get a single attendance log (QR + geofence detail) |
| `/logs/:id/overtime/pre-approve` | PATCH | Flag the shift's overtime as pre-approved |
| `/logs/:id/overtime/query` | PATCH | Query overtime that was not pre-approved |
| `/hours-report` | GET | Download hours report for a period (`?from=`, `?to=`, `?format=`) |

**GET /logs — Response 200**
```json
{
  "items": [
    { "id": "at1", "workerName": "John Doe",  "shift": "morning", "date": "2026-07-10", "qrCheckIn": "08:02", "qrCheckOut": "16:05", "source": "qr",       "totalHours": 8.05, "overtimeHours": 0,   "overtimeStatus": "none" },
    { "id": "at2", "workerName": "Jane Smith", "shift": "evening", "date": "2026-07-10", "qrCheckIn": "14:00", "qrCheckOut": null,     "geofenceCheckOut": "22:30", "source": "geofence", "totalHours": 8.5, "overtimeHours": 1.5, "overtimeStatus": "pending" }
  ]
}
```

**PATCH /logs/:id/overtime/pre-approve — Response 200**
```json
{ "success": true, "logId": "at2", "overtimeStatus": "pre_approved" }
```

**PATCH /logs/:id/overtime/query — Request**
```json
{ "reason": "Overtime not pre-authorised; verifying with vendor" }
```

**GET /hours-report?from=2026-06-01&to=2026-06-30&format=csv — Response 200**
Returns a downloadable file of hours for the property over the period.

### 36.6 UI Layout (brief)
- **Attendance** (`/property-manager/attendance`): table per worker/shift with QR check-in/out; rows using **geofence backup** are badged; overtime shown with status.
- **Overtime actions**: **Pre-approve** or **Query** on overtime rows.
- **Hours report**: date-range picker + **Download** (CSV/PDF/XLSX).

### 36.7 Implementation Phases
- **Phase 1 — Mock:** Attendance logs (QR + geofence) and overtime status from mock data; report download returns a sample file.
- **Phase 2 — Real Data:** Logs from the QR/geofence attendance system; overtime status persists and feeds payroll; reports generated from real data.

### 36.8 Acceptance Criteria
- PM can view QR-based attendance logs per worker per shift.
- Geofence backup data is shown where the QR scan is absent.
- PM can pre-approve or query overtime.
- PM can download an hours report for any selected period.
- All data/actions are property-scoped; endpoints reject other roles.

### 36.9 Open Questions
- Does **"query" overtime** create a dispute (§21), or just flag it for follow-up?
- When **both QR and geofence** exist, which is authoritative for hours?
- Does **pre-approved overtime** get paid while **queried/pending** is withheld until resolved?
- Hours report **columns/format** and whether it matches the Reports module (§38) output.

---

## 37. Property Manager — Settings Module

### 37.1 Overview
**Module #6** of the Property Manager role (§31.2). It lets the manager **override chain-default settings** at the property level and control whether department managers may select vendors directly.

Route: `/property-manager/settings`

**Scope:** overrides apply to the manager's **own property**; anything not overridden **inherits the chain defaults** (§28).

> **Relationship to §28:** the Hotel Central Authority sets chain-wide defaults (§28); this module records **property-level overrides**. §28.3 modelled a per-property override for **buffer only** — this module extends that to **payment cycle** and **notification preferences** too. **[CONFIRM]** which chain defaults are property-overridable.

**Actions & Functions**
1. **Override chain-default settings at property level** — buffer %, payment cycle, notification preferences.
2. **Enable or disable department manager permission to select vendors directly.**

### 37.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-PMS1 | PM can override the **buffer %** at property level | High |
| FR-PMS2 | PM can override the **payment cycle** at property level | High |
| FR-PMS3 | PM can override **notification preferences** (SMS/WhatsApp/email) at property level | High |
| FR-PMS4 | PM can **enable/disable** department manager permission to **select vendors directly** | High |
| FR-PMS5 | Any setting not overridden **inherits the chain default** (§28) | High |
| FR-PMS6 | All settings are scoped to the manager's **property** | High |
| FR-PMS7 | Restricted to `role = property_manager` | High |

### 37.3 Data Model
**PropertySettings** (override of chain defaults; `null` = inherit)
| Field | Type | Notes |
|-------|------|-------|
| propertyId | UUID | The manager's property |
| bufferTolerancePercent | number (nullable) | Overrides chain default; `null` = inherit |
| paymentCycleDays | enum (nullable) | `7` \| `15` \| `30`; `null` = inherit |
| notifications | object (nullable) | `{ sms, whatsapp, email }`; `null` = inherit |
| departmentManagerVendorSelection | boolean | Whether DMs may select vendors directly at this property |
| updatedAt | timestamp | |

> `departmentManagerVendorSelection` at property level refines the chain's `vendorSelectionMode` (§28.3). **[CONFIRM]** precedence when chain and property settings differ.

### 37.4 API / Interface Design
Base path: `/api/pm/settings` — requires `Bearer <jwt>` with `role = property_manager`; scoped to the manager's property.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/settings` | GET | Get effective settings (chain defaults + property overrides) |
| `/settings` | PUT | Set property-level overrides |
| `/settings/dm-vendor-selection` | PATCH | Enable/disable DM direct vendor selection |

**GET /settings — Response 200** (effective view)
```json
{
  "propertyId": "p1",
  "bufferTolerancePercent": { "value": 5,  "source": "property_override" },
  "paymentCycleDays":       { "value": 15, "source": "chain_default" },
  "notifications":          { "value": { "sms": true, "whatsapp": false, "email": true }, "source": "property_override" },
  "departmentManagerVendorSelection": false
}
```

**PUT /settings — Request**
```json
{
  "bufferTolerancePercent": 5,
  "paymentCycleDays": null,
  "notifications": { "sms": true, "whatsapp": false, "email": true }
}
```

**PATCH /settings/dm-vendor-selection — Request**
```json
{ "enabled": true }
```

### 37.5 UI Layout (brief)
- **Settings** (`/property-manager/settings`) with sections:
  - **Buffer Tolerance** — property override % (with an "inherit chain default" option).
  - **Payment Cycle** — 7/15/30 override (or inherit).
  - **Notifications** — SMS/WhatsApp/email toggles (or inherit).
  - **Department Manager Vendor Selection** — enable/disable toggle.
- Each overridable setting shows whether it is **inherited** or **overridden**.

### 37.6 Implementation Phases
- **Phase 1 — Mock:** Effective settings computed from mock chain defaults + mock overrides.
- **Phase 2 — Real Data:** DB-backed property overrides; effective settings resolved against chain defaults (§28) and consumed by the relevant flows.

### 37.7 Acceptance Criteria
- PM can override buffer %, payment cycle, and notification preferences at the property.
- Non-overridden settings inherit the chain defaults.
- PM can enable/disable department manager direct vendor selection.
- All settings are property-scoped; endpoints reject other roles.

### 37.8 Open Questions
- Which chain defaults are **allowed** to be overridden per property (all, or a restricted set)?
- **Precedence** — if chain sets "via property manager" but the property enables DM direct selection, which wins?
- Should property overrides be **auditable/visible** to the Hotel Central Authority?

---

## 38. Property Manager — Reports Module

### 38.1 Overview
**Module #7** of the Property Manager role (§31.2). It gives the manager **property-level reporting** — hours worked and payroll summary, an exportable authenticated report for cross-checking vendor invoices, and dispute/cancellation history.

Route: `/property-manager/reports`

**Scope:** all reports cover the manager's **own property** only.

**Actions & Functions**
1. **View hours worked and payroll summary for the property.**
2. **Export authenticated report for vendor invoicing cross-check.**
3. **View dispute and cancellation history.**

### 38.2 Report Types
| Report | Contents |
|--------|----------|
| Hours & Payroll Summary | Hours worked and payroll totals for the property. |
| Vendor Invoice Cross-check (authenticated) | Verified/authenticated report to reconcile against a vendor's invoice. |
| Dispute History | Disputes raised at the property and their outcomes (from §21). |
| Cancellation History | Cancelled work requests/shifts at the property (reason, actor). |

### 38.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-PR1 | PM can view **hours worked** for the property | High |
| FR-PR2 | PM can view the **payroll summary** for the property | High |
| FR-PR3 | PM can **export an authenticated report** for vendor invoicing cross-check | High |
| FR-PR4 | PM can view **dispute history** for the property | High |
| FR-PR5 | PM can view **cancellation history** for the property | High |
| FR-PR6 | Reports are filterable by date range (and vendor where relevant) | Medium |
| FR-PR7 | Reports export in CSV/PDF/XLSX | Medium |
| FR-PR8 | All reports are scoped to the manager's **property** | High |
| FR-PR9 | Restricted to `role = property_manager` | High |

### 38.4 Data Model
**ReportRequest** (property-scoped)
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| propertyId | UUID | The manager's property (enforced) |
| reportType | enum | `hours_payroll` \| `vendor_invoice_crosscheck` \| `dispute_history` \| `cancellation_history` |
| vendorId | UUID (nullable) | For vendor invoice cross-check |
| dateFrom | date | Period start |
| dateTo | date | Period end |
| format | enum | `csv` \| `pdf` \| `xlsx` |
| authenticated | boolean | For the vendor invoice cross-check report |
| status | enum | `ready` \| `processing` \| `failed` |
| fileUrl | string (nullable) | Download link when ready |
| createdAt | timestamp | |

> **[CONFIRM]** what "authenticated" means for the invoice cross-check report (verified against attendance/hours, signed/stamped) — consistent with §29.5.

### 38.5 API / Interface Design
Base path: `/api/pm/reports` — requires `Bearer <jwt>` with `role = property_manager`; all data scoped to the manager's property.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/hours-payroll` | GET | Hours worked and payroll summary for the property |
| `/vendor-invoice-crosscheck` | GET | Export an authenticated cross-check report (`?vendorId=`, `?format=`) |
| `/disputes` | GET | Dispute history for the property |
| `/cancellations` | GET | Cancellation history for the property |

**GET /hours-payroll — Response 200**
```json
{
  "propertyId": "p1",
  "period": "2026-06",
  "totalHours": 5200,
  "totalPayroll": 65000,
  "byVendor": [
    { "vendorName": "BrightClean Services", "hours": 3200, "payroll": 40000 },
    { "vendorName": "Summit Maintenance",   "hours": 2000, "payroll": 25000 }
  ]
}
```

**GET /vendor-invoice-crosscheck?vendorId=v1&format=pdf — Response 200**
```json
{
  "vendorId": "v1",
  "vendorName": "BrightClean Services",
  "period": "2026-06",
  "authenticated": true,
  "recordedHours": 3200,
  "amount": 40000,
  "fileUrl": "/downloads/pm/crosscheck/v1-2026-06.pdf"
}
```

### 38.6 UI Layout (brief)
- **Reports** (`/property-manager/reports`) with sections: **Hours & Payroll**, **Vendor Invoice Cross-check** (export), **Disputes**, **Cancellations**.
- Date-range (and vendor) filters with **View / Export** actions.
- The cross-check report shows an **Authenticated** badge and **Export** button.

### 38.7 Implementation Phases
- **Phase 1 — Mock:** Reports and exports from mock data scoped to a mock property.
- **Phase 2 — Real Data:** Real aggregation for the property; cross-check report generated from attendance/hours (§36); dispute/cancellation history from source modules.

### 38.8 Acceptance Criteria
- PM can view hours worked and payroll summary for the property.
- PM can export an authenticated report for vendor invoice cross-check.
- PM can view dispute and cancellation history for the property.
- All reports are property-scoped; endpoints reject other roles.

### 38.9 Open Questions
- Definition of **"authenticated"** cross-check report (verified against §36 hours, signed, or approval-stamped)?
- Should the cross-check report support **entering the vendor's invoice figure** to show a delta?
- Do payroll figures come from a **payroll/billing** source (§19) or are they computed here?
- Should this reconcile with the Hotel Central Authority Reports (§29) — same underlying data?

---
---

# Department Manager Role

> **§1–23 Superadmin · §24–30 Hotel Central Authority · §31–38 Property Manager.** The sections below (§39+) specify the **Department Manager** role.

## 39. Department Manager — Role Overview

### 39.1 Overview
The **Department Manager** runs a **single department** within a property (e.g. Housekeeping, Front Desk, Maintenance, F&B). It raises work requests for its department (which the **Property Manager approves** — see §33), has visibility of its department's workers, and views department-level reports.

**Authentication:** login is shared across roles (see §1–13). On successful login where the JWT `role = department_manager`, the app redirects to the Department Manager landing view. There is no separate Dashboard module for this role — the landing is the **Work Request** module (§40). **[CONFIRM]** the preferred landing.

**Scope rule:** every action is limited to the manager's **own department** within its property.

### 39.2 Department Manager Modules
The role is organised into **3 functional modules**. Each has its **own actions and functions** — those will be added as they are shared.

| # | Module | Purpose (summary) | Actions & Functions | Detailed spec |
|---|--------|-------------------|---------------------|---------------|
| 1 | Work Request | Raise dept requests; optional vendor select; track status (no self-approval) | ✅ Defined | §40 |
| 2 | Worker Visibility | View dept workers/attendance for active shift; rate supervised workers | ✅ Defined | §41 |
| 3 | Reports | Department-only hours & attendance (no cross-department/property data) | ✅ Defined | §42 |

> **Status legend:** ✅ Defined · ⏳ Awaiting the actions/functions you'll share.
>
> All 3 modules are defined (§40–§42). Each module section includes its actions/functions, FRs, data model, API, UI, phases, acceptance criteria, and open questions.

### 39.3 Open Questions
- Can a Department Manager raise requests that go **directly to vendors**, or always **via Property Manager approval** (§33)? Governed by chain/property settings (§28/§37).
- Is Worker Visibility strictly **read-only**, or can the DM take any worker actions?
- Landing view for this role (no Dashboard module) — Work Request list, or a simple summary?

---

## 40. Department Manager — Work Request Module

### 40.1 Overview
**Module #1** of the Department Manager role (§39.2). It lets the manager raise work requests for its department, optionally select a vendor (when chain/property settings permit), and track the status of its own requests. The manager **cannot approve** its own requests — approval is always the **Property Manager's** action (§33).

Route: `/department-manager/work-requests`

**Scope:** the manager's **own department** within its property.

**Actions & Functions**
1. **Raise a work request for the department** — start/end date, shift, designation, headcount.
2. **Select a vendor** — if chain/property settings permit; otherwise **request without vendor selection**.
3. **View status of own raised requests** — Pending Approval, Approved, Sent to Vendor.
4. **No self-approval** — the manager cannot approve its own requests (Property Manager approves).

### 40.2 Request Status (DM view)
| Status | Meaning |
|--------|---------|
| pending_approval | Raised by the DM, awaiting Property Manager approval |
| approved | Approved by the Property Manager |
| sent_to_vendor | Sent to the vendor(s) after approval |
| rejected | Rejected by the Property Manager (with reason) |

### 40.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-DW1 | DM can **raise** a work request with start/end date, shift, designation, headcount | High |
| FR-DW2 | DM can **select a vendor** only if chain/property settings permit direct selection | High |
| FR-DW3 | DM can **raise a request without vendor selection** (routed for PM/vendor handling) | High |
| FR-DW4 | Vendor selection is hidden/blocked when settings disallow DM direct selection (§28/§37) | High |
| FR-DW5 | DM can **view the status** of its own requests (Pending Approval / Approved / Sent to Vendor) | High |
| FR-DW6 | DM **cannot approve** any request — no approve action is available to this role | High |
| FR-DW7 | All requests/actions are scoped to the manager's **department** | High |
| FR-DW8 | Restricted to `role = department_manager` | High |

### 40.4 Data Model
**WorkRequest** (DM-raised) — see the full model in §33.4.
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| propertyId | UUID | The department's property |
| departmentId | UUID | The manager's department (enforced) |
| designation | string | Job role/designation needed |
| startDate | date | Work start |
| endDate | date | Work end |
| shift | enum | `morning` \| `evening` \| `night` |
| headcount | integer | Number of workers required |
| selectedVendorId | UUID (nullable) | Chosen vendor, if permitted; `null` = no vendor selected |
| raisedByType | enum | Fixed: `department_manager` |
| raisedById | UUID | The DM user id |
| approvalStatus | enum | `pending` \| `approved` \| `rejected` (PM decision) |
| status | enum | `pending_approval` \| `approved` \| `sent_to_vendor` \| `rejected` |
| createdAt | timestamp | |

### 40.5 API / Interface Design
Base path: `/api/dm/work-requests` — requires `Bearer <jwt>` with `role = department_manager`; scoped to the manager's department.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/work-requests` | GET | List the department's own raised requests (with status) |
| `/work-requests/:id` | GET | Get a request's detail/status |
| `/work-requests` | POST | Raise a new work request (vendor optional/permitted) |
| `/vendors` | GET | Approved vendors for selection (**only if permitted**; else 403/empty) |

> No approve/reject endpoints — approval is the Property Manager's action (§33).

**POST /work-requests — Request**
```json
{
  "designation": "Housekeeper",
  "startDate": "2026-07-15",
  "endDate": "2026-07-18",
  "shift": "morning",
  "headcount": 3,
  "selectedVendorId": null
}
```
**POST /work-requests — Response 201**
```json
{ "success": true, "workRequest": { "id": "wr20", "designation": "Housekeeper", "headcount": 3, "status": "pending_approval" } }
```

**GET /work-requests — Response 200**
```json
{
  "items": [
    { "id": "wr20", "designation": "Housekeeper", "shift": "morning", "headcount": 3, "status": "pending_approval" },
    { "id": "wr18", "designation": "Electrician", "shift": "evening", "headcount": 1, "status": "approved" },
    { "id": "wr15", "designation": "Cleaner",     "shift": "night",   "headcount": 2, "status": "sent_to_vendor" }
  ]
}
```

### 40.6 UI Layout (brief)
- **Work requests** (`/department-manager/work-requests`): the department's own requests with designation, dates, shift, headcount, status; **Raise Request** action.
- **Raise form**: start/end date, shift, designation, headcount, and a **vendor select** (shown only when settings permit; otherwise "no vendor — routed for approval").
- **No approve/reject controls** anywhere in this module.

### 40.7 Implementation Phases
- **Phase 1 — Mock:** Requests raised into mock data; vendor selection availability read from a mock setting; statuses simulated.
- **Phase 2 — Real Data:** Requests created for PM approval (§33); vendor selection gated by real settings (§28/§37); statuses reflect the real approval/send flow.

### 40.8 Acceptance Criteria
- DM can raise a work request with start/end date, shift, designation, and headcount.
- DM can select a vendor only when settings permit; otherwise can raise without vendor selection.
- DM can view its own requests' status (Pending Approval / Approved / Sent to Vendor).
- DM has no approve/reject capability anywhere.
- All requests/actions are department-scoped; endpoints reject other roles.

### 40.9 Open Questions
- When a DM selects a vendor, does the PM still **approve** before it's sent, or does selection route it directly? (Expected: PM approval always precedes send.)
- Should the DM see the **rejection reason** when a request is rejected?
- Can the DM **edit or cancel** its own request while `pending_approval`?
- Does "request without vendor selection" mean the **PM selects the vendor**, or it broadcasts to the vendor pool?

---

## 41. Department Manager — Worker Visibility Module

### 41.1 Overview
**Module #2** of the Department Manager role (§39.2). It gives the manager visibility of the workers in its department for active shifts, their check-in/out status, and the ability to rate workers on shifts it directly supervises.

Route: `/department-manager/workers`

**Scope:** workers assigned to the manager's **own department**. This module is **view-only** except for worker rating.

**Cross-module link:** ratings feed **Rating & Trust (§20)**.

**Actions & Functions**
1. **View workers assigned to the department for an active shift.**
2. **See check-in / check-out status** for workers in the department.
3. **Rate individual workers** at the end of each shift they **directly supervise**.

### 41.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-DV1 | DM can view workers assigned to its department for an **active shift** | High |
| FR-DV2 | DM can see each worker's **check-in / check-out status** | High |
| FR-DV3 | DM can **rate a worker** at the end of a shift it directly supervises (1–5 + comment) | High |
| FR-DV4 | Rating is allowed only for shifts the DM **directly supervises** | Medium |
| FR-DV5 | Module is **view-only** apart from rating (no other worker actions) | High |
| FR-DV6 | All data/actions are scoped to the manager's **department** | High |
| FR-DV7 | Restricted to `role = department_manager` | High |

### 41.3 Data Model
**Worker (department assignment, read model)**
| Field | Type | Notes |
|-------|------|-------|
| workerId | UUID | Worker id |
| name | string | Worker name |
| vendorName | string | Vendor/agency the worker belongs to |
| shift | enum | `morning` \| `evening` \| `night` |
| attendanceStatus | enum | `checked_in` \| `checked_out` \| `absent` \| `not_started` |
| checkInTime | timestamp (nullable) | Actual check-in |
| checkOutTime | timestamp (nullable) | Actual check-out (null if active) |
| supervisedByMe | boolean | Whether the DM directly supervises this shift (rating gate) |

**WorkerRating (→ §20)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| subjectType | enum | Fixed: `worker` |
| subjectId | UUID | Worker id |
| shiftId | UUID | Shift being rated |
| score | number | 1–5 |
| comment | string (nullable) | Review text |
| ratedBy | UUID | Department Manager id |
| createdAt | timestamp | |

### 41.4 API / Interface Design
Base path: `/api/dm/workers` — requires `Bearer <jwt>` with `role = department_manager`; scoped to the manager's department.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/workers` | GET | List department workers for the active shift (with attendance status) |
| `/workers/:id` | GET | Get a worker's detail |
| `/workers/:id/rating` | POST | Rate a worker for a supervised shift (**403 if not supervised**) |

> No suspend/deactivate/dispute actions — this module is view + rate only.

**GET /workers — Response 200**
```json
{
  "shift": "morning",
  "items": [
    { "workerId": "w1", "name": "John Doe",   "vendorName": "BrightClean Services", "attendanceStatus": "checked_in",  "checkInTime": "08:02", "checkOutTime": null,    "supervisedByMe": true },
    { "workerId": "w4", "name": "Amit Verma",  "vendorName": "Summit Maintenance",   "attendanceStatus": "checked_out", "checkInTime": "08:00", "checkOutTime": "16:00", "supervisedByMe": true }
  ]
}
```

**POST /workers/:id/rating — Request**
```json
{ "shiftId": "s1", "score": 4, "comment": "Good work, arrived on time" }
```
**POST /workers/:id/rating — Response 201 / 403**
```json
{ "success": true, "rating": { "id": "rt10", "subjectId": "w1", "shiftId": "s1", "score": 4 } }
```
```json
{ "success": false, "message": "You can only rate shifts you directly supervise" }
```

### 41.5 UI Layout (brief)
- **Workers** (`/department-manager/workers`): list of department workers for the active shift with vendor, attendance status, and check-in/out times.
- **Rate Worker** action available at end of shift, **only for supervised shifts**.
- No suspend/dispute/other write actions.

### 41.6 Implementation Phases
- **Phase 1 — Mock:** Department workers and attendance from mock data; ratings appended to mock records.
- **Phase 2 — Real Data:** Workers/attendance from the attendance system, filtered to the department; ratings written to §20; supervision gate enforced.

### 41.7 Acceptance Criteria
- DM can view department workers for an active shift with check-in/out status.
- DM can rate workers only for shifts it directly supervises.
- No other worker actions are available (view + rate only).
- All data/actions are department-scoped; endpoints reject other roles.

### 41.8 Open Questions
- How is **"directly supervises"** determined — department + shift assignment, or an explicit supervisor link?
- Is rating **mandatory** at shift end, and is there a rating window/deadline?
- Should the DM see **historical shifts/workers**, or only the active shift?
- Can a worker be rated by **both** the DM (§41) and the PM (§35) for the same shift?

---

## 42. Department Manager — Reports Module

### 42.1 Overview
**Module #3** of the Department Manager role (§39.2). It gives the manager **department-only** reporting — hours and attendance data for its own department. It **cannot** view data across other departments or the full property.

Route: `/department-manager/reports`

**Scope:** strictly the manager's **own department**. Cross-department and full-property data are **not accessible**.

**Actions & Functions**
1. **View hours and attendance data for their department only.**
2. **No cross-department or full-property visibility** — hard scope boundary.

### 42.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-DMR1 | DM can view **hours** data for its own department | High |
| FR-DMR2 | DM can view **attendance** data for its own department | High |
| FR-DMR3 | DM **cannot** view data for **other departments** | High |
| FR-DMR4 | DM **cannot** view **full-property** aggregate data | High |
| FR-DMR5 | Reports are filterable by date range (within the department) | Medium |
| FR-DMR6 | Department hours/attendance may be **downloaded** for a period | Medium |
| FR-DMR7 | Department scope is **server-enforced** (from the JWT/user), not client-supplied | High |
| FR-DMR8 | Restricted to `role = department_manager` | High |

### 42.3 Report Types
| Report | Contents |
|--------|----------|
| Department Hours | Worked hours for the department (by worker/shift) within a period. |
| Department Attendance | Attendance records (check-in/out, absences) for the department. |

### 42.4 Data Model
**ReportRequest** (department-scoped)
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| departmentId | UUID | The manager's department (server-enforced) |
| reportType | enum | `department_hours` \| `department_attendance` |
| dateFrom | date | Period start |
| dateTo | date | Period end |
| format | enum | `csv` \| `pdf` \| `xlsx` |
| status | enum | `ready` \| `processing` \| `failed` |
| fileUrl | string (nullable) | Download link when ready |
| createdAt | timestamp | |

> **Scope enforcement:** `departmentId` is derived from the authenticated user — it is **never accepted from the request**, preventing access to other departments.

### 42.5 API / Interface Design
Base path: `/api/dm/reports` — requires `Bearer <jwt>` with `role = department_manager`; all data scoped to the manager's department (server-side).

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/hours` | GET | Department hours (filter by date range) |
| `/attendance` | GET | Department attendance (filter by date range) |
| `/export` | GET | Download a department hours/attendance report (`?type=`, `?from=`, `?to=`, `?format=`) |

> No `propertyId` or cross-department parameters are accepted; requests targeting another department/the whole property return **403**.

**GET /hours — Response 200**
```json
{
  "departmentId": "dep-housekeeping",
  "period": "2026-06",
  "totalHours": 1840,
  "byWorker": [
    { "workerName": "John Doe",  "hours": 168 },
    { "workerName": "Amit Verma", "hours": 152 }
  ]
}
```

**GET /attendance — Response 200**
```json
{
  "departmentId": "dep-housekeeping",
  "date": "2026-07-10",
  "items": [
    { "workerName": "John Doe",   "shift": "morning", "checkIn": "08:02", "checkOut": "16:05", "status": "present" },
    { "workerName": "Amit Verma", "shift": "morning", "checkIn": null,    "checkOut": null,    "status": "absent" }
  ]
}
```

### 42.6 UI Layout (brief)
- **Reports** (`/department-manager/reports`) with **Hours** and **Attendance** sections for the department, a date-range filter, and a **Download** action.
- No department selector and no property-wide view — the department is fixed to the manager's own.

### 42.7 Implementation Phases
- **Phase 1 — Mock:** Department hours/attendance from mock data scoped to a mock department.
- **Phase 2 — Real Data:** Real hours/attendance filtered to the department by server-enforced scope; downloads generated from real data.

### 42.8 Acceptance Criteria
- DM can view hours and attendance data for its own department.
- DM cannot access other departments' or full-property data (403).
- Department scope is enforced server-side, not from request parameters.
- All endpoints reject other roles.

### 42.9 Open Questions
- Should the DM be able to **export** (download) department reports, or **view-only**?
- Reporting **period** options — day, month, or custom range?
- Do department hours reconcile with the property-level hours in §36/§38 (same source)?

---
---

# Vendor Manager Role

> **§1–23 Superadmin · §24–30 Hotel Central Authority · §31–38 Property Manager · §39–42 Department Manager.** The sections below (§43+) specify the **Vendor Manager** role.

## 43. Vendor Manager — Role Overview

### 43.1 Overview
The **Vendor Manager** runs an **external vendor organisation** on the platform. It manages the vendor's subscription, bids on work requests, manages and assigns the vendor's workers, tracks their attendance/hours, views ratings, handles reports/payroll, and manages disputes.

**Authentication:** login is shared across roles (see §1–13). On successful login where the JWT `role = vendor_manager`, the app redirects to the **Vendor Dashboard** (§44).

**Scope rule:** every action is limited to the manager's **own vendor organisation** — its subscription, workers, bids, assignments, attendance, ratings, reports, and disputes.

### 43.2 Vendor Manager Modules
The role is organised into **9 functional modules**. Each has its **own actions and functions** — those will be added as they are shared.

| # | Module | Purpose (summary) | Actions & Functions | Detailed spec |
|---|--------|-------------------|---------------------|---------------|
| 1 | Dashboard | Subscription & headcount, today's assignments, alerts | ✅ Defined | §44 |
| 2 | Subscription Management | Purchase seats (£3.5/£5); headcount; invoices; IRIS; payroll export | ✅ Defined | §45 |
| 3 | Work Request / Bidding | View incoming requests; headcount-checked bids; decline; bid history | ✅ Defined | §46 |
| 4 | Worker Management | Add/accept workers; designation tags; primary/secondary; activate/remove | ✅ Defined | §47 |
| 5 | Worker Assignment | Primary+backup selection; response countdown; reassign; live geofence | ✅ Defined | §48 |
| 6 | Attendance & Hours | Check-in/out logs; overtime flags; invoicing hours report; geofence backup | ✅ Defined | §49 |
| 7 | Ratings | Agency rating + shift context; worker ratings; rate property/managers | ✅ Defined | §50 |
| 8 | Reports & Payroll | Payroll (hours×rate) & margin summaries; payroll export; hotel payment confirmations | ✅ Defined | §51 |
| 9 | Dispute Management | Raise/respond to hours disputes; auto QR/geofence counter-evidence | ✅ Defined | §52 |

> **Status legend:** ✅ Defined · ⏳ Awaiting the actions/functions you'll share.
>
> All 9 modules are defined (§44–§52). Each module section includes its actions/functions, FRs, data model, API, UI, phases, acceptance criteria, and open questions.

### 43.3 Open Questions
- Is a Vendor Manager tied to **one vendor organisation**, or can it manage multiple?
- Are the vendor's **workers** the same `worker` role (§Worker role, pending), managed here?
- Does **Reports & Payroll** here pay workers, and how does it relate to Subscription & Billing (§19)?

---

## 44. Vendor Manager — Dashboard Module

### 44.1 Overview
**Module #1** of the Vendor Manager role (§43.2). It is the landing view after login and gives the vendor a **vendor-wide** overview — subscription and headcount, today's assignments and active shifts, and key alerts.

Route: `/vendor/dashboard`

**Scope:** the manager's **own vendor organisation** only.

**Actions & Functions**
1. **View active subscriptions, committed vs available headcount.**
2. **View today's worker assignments and active shifts.**
3. **See alerts** — no-show risk, subscription expiring, pending bid responses.

### 44.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-VD1 | On login with `role = vendor_manager`, redirect to the Vendor dashboard | High |
| FR-VD2 | Dashboard accessible only to `vendor_manager`; scoped to the own vendor | High |
| FR-VD3 | Show **active subscription** status and **committed vs available headcount** | High |
| FR-VD4 | Show **today's worker assignments** and **active shifts** | High |
| FR-VD5 | Show **alerts**: no-show risk, subscription expiring, pending bid responses | High |
| FR-VD6 | Metrics/data are fetched from the backend (not hardcoded) | High |
| FR-VD7 | All data is limited to the manager's vendor (no other vendors) | High |
| FR-VD8 | Show empty, loading, and error states | Medium |

### 44.3 UI Layout
```
+-----------------------------------------------------------------------+
|  Vendor Dashboard                                  [ BrightClean Ltd ] |
+-----------------------------------------------------------------------+
|  +---------------------+ +---------------------------------------+     |
|  | Subscription        | | Headcount                             |     |
|  |  Pro — active        | |  Committed 18   Available 6   Total 24 |     |
|  |  renews 2026-08-01  | |                                       |     |
|  +---------------------+ +---------------------------------------+     |
|                                                                       |
|  +-------------------------------+  +------------------------------+  |
|  | Today's Assignments           |  | Alerts                       |  |
|  |-------------------------------|  |------------------------------|  |
|  | J.Doe   Sunrise   Morning  in |  | ! No-show risk  R.Kumar      |  |
|  | J.Smith Palm      Evening  -- |  | ! Subscription expiring 5d   |  |
|  | A.Verma Lake View Morning  in |  | ! 2 pending bid responses    |  |
|  +-------------------------------+  +------------------------------+  |
+-----------------------------------------------------------------------+
```

### 44.4 Data Model (read models)
**Vendor Overview (computed)**
| Field | Type | Notes |
|-------|------|-------|
| subscriptionPlan | string | Current plan name |
| subscriptionStatus | enum | `active` \| `trial` \| `past_due` \| `expired` \| `none` |
| renewalDate | date (nullable) | Next renewal |
| committedHeadcount | integer | Workers currently committed to assignments |
| availableHeadcount | integer | Workers free to assign |
| totalHeadcount | integer | Total workers |

**Today's Assignment item**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Assignment id |
| workerName | string | Assigned worker |
| propertyName | string | Property/site |
| shift | enum | `morning` \| `evening` \| `night` |
| status | enum | `scheduled` \| `in_progress` \| `completed` \| `no_show` |

**Alert item**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Alert id |
| type | enum | `no_show_risk` \| `subscription_expiring` \| `pending_bid` |
| detail | string | e.g. worker name, days to expiry, bid count |
| severity | enum | `info` \| `warning` \| `critical` |
| createdAt | timestamp | |

### 44.5 API / Interface Design
Base path: `/api/vendor/dashboard` — requires `Bearer <jwt>` with `role = vendor_manager`; all responses scoped to the caller's vendor.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/overview` | GET | Subscription status and committed vs available headcount |
| `/today-assignments` | GET | Today's worker assignments and active shifts |
| `/alerts` | GET | Alerts (filter by `?type=`) |

**GET /overview — Response 200**
```json
{
  "subscriptionPlan": "Pro",
  "subscriptionStatus": "active",
  "renewalDate": "2026-08-01",
  "committedHeadcount": 18,
  "availableHeadcount": 6,
  "totalHeadcount": 24
}
```

**GET /today-assignments — Response 200**
```json
{
  "date": "2026-07-10",
  "items": [
    { "id": "as1", "workerName": "John Doe",   "propertyName": "Sunrise Apts", "shift": "morning", "status": "in_progress" },
    { "id": "as2", "workerName": "Jane Smith",  "propertyName": "Palm Villa",   "shift": "evening", "status": "scheduled" }
  ]
}
```

**GET /alerts — Response 200**
```json
{
  "items": [
    { "id": "al1", "type": "no_show_risk",         "detail": "Ravi Kumar — late check-in risk", "severity": "warning",  "createdAt": "2026-07-10T08:40:00Z" },
    { "id": "al2", "type": "subscription_expiring", "detail": "Subscription expires in 5 days",  "severity": "critical", "createdAt": "2026-07-10T00:00:00Z" },
    { "id": "al3", "type": "pending_bid",           "detail": "2 pending bid responses",         "severity": "info",     "createdAt": "2026-07-10T07:00:00Z" }
  ]
}
```

### 44.6 Implementation Phases
- **Phase 1 — Mock:** Overview, assignments, and alerts from mock data scoped to a mock vendor; full UI built against the mock API.
- **Phase 2 — Real Data:** Real subscription/headcount, today's assignments, and alerts computed for the vendor.

### 44.7 Acceptance Criteria
- Login as `vendor_manager` redirects to the Vendor dashboard.
- Overview shows subscription status and committed vs available headcount.
- Today's assignments and active shifts are shown.
- Alerts show no-show risk, subscription expiring, and pending bid responses.
- No data outside the manager's vendor is visible; empty/loading/error states handled.

### 44.8 Open Questions
- Definition of **"committed" headcount** — assigned for today, or across all active assignments?
- **No-show risk** trigger — late check-in window, past no-show rate, or both?
- Should alerts be **actionable** (click to bids §46 / subscription §45 / worker), or view-only?
- Can a vendor hold **multiple active subscriptions**, or exactly one at a time?

---

## 45. Vendor Manager — Subscription Management Module

### 45.1 Overview
**Module #2** of the Vendor Manager role (§43.2). It lets the vendor purchase and activate **per-user (seat-based) subscriptions**, track active vs available headcount, manage billing and invoices, opt workers in/out of **IRIS payroll integration**, and export payroll data to third-party accounting software.

Route: `/vendor/subscriptions`

**Scope:** the manager's **own vendor organisation**. Pricing tiers are defined by the Superadmin (§19) — this module consumes them.

**Actions & Functions**
1. **Purchase and activate user subscriptions** — **Base £3.5/user/month** or **Payroll-enabled £5/user/month**.
2. **View active vs available headcount** across all subscriptions.
3. **Manage billing cycle and view invoices.**
4. **Opt in/out of IRIS payroll integration** — per worker.
5. **Export payroll data** to third-party software — CSV/XML for **Sage, Xero, BrightPay, QuickBooks**.

### 45.2 Subscription Tiers
| Tier | Price | Includes |
|------|-------|----------|
| Base | **£3.5 / user / month** | Standard platform access per seated user |
| Payroll-enabled | **£5 / user / month** | Base + payroll features (incl. IRIS integration eligibility) |

> Prices are per **seat/user per month** and are set by the Superadmin pricing config (§19). **[CONFIRM]** currency (GBP) and whether annual billing is offered.

### 45.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-VS1 | Vendor can **purchase & activate** a subscription (Base £3.5 or Payroll-enabled £5 per user/month) | High |
| FR-VS2 | Vendor can choose the **number of seats** and see the total cost | High |
| FR-VS3 | Vendor can view **active vs available headcount** across all subscriptions | High |
| FR-VS4 | Vendor can **manage the billing cycle** | High |
| FR-VS5 | Vendor can **view/download invoices** | High |
| FR-VS6 | Vendor can **opt a worker in/out of IRIS payroll integration** | High |
| FR-VS7 | IRIS opt-in is available only for **payroll-enabled** seats **[CONFIRM]** | Medium |
| FR-VS8 | Vendor can **export payroll data** as CSV/XML for Sage, Xero, BrightPay, QuickBooks | High |
| FR-VS9 | All data/actions are scoped to the manager's **vendor** | High |
| FR-VS10 | Restricted to `role = vendor_manager` | High |

### 45.4 Data Model
**Subscription** (seat-based)
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| vendorId | UUID | The manager's vendor |
| tier | enum | `base` \| `payroll_enabled` |
| unitPrice | number | £3.5 (base) or £5 (payroll-enabled) per user/month |
| seats | integer | Purchased seats |
| activeSeats | integer | Seats currently assigned to workers |
| availableSeats | integer | `seats - activeSeats` |
| billingCycle | enum | `monthly` \| `annual` |
| status | enum | `active` \| `trial` \| `past_due` \| `expired` \| `cancelled` |
| startDate | date | |
| renewalDate | date | |

**Invoice**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| number | string | Invoice number |
| amount | number | Amount (GBP) |
| period | string | Billing period |
| status | enum | `paid` \| `due` \| `overdue` |
| issuedAt | timestamp | |
| fileUrl | string | Downloadable invoice |

**WorkerPayrollIntegration**
| Field | Type | Notes |
|-------|------|-------|
| workerId | UUID | Worker id |
| irisEnabled | boolean | IRIS payroll integration opt-in |
| updatedAt | timestamp | |

### 45.5 API / Interface Design
Base path: `/api/vendor/subscriptions` — requires `Bearer <jwt>` with `role = vendor_manager`; scoped to the caller's vendor.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/subscriptions` | GET | List the vendor's subscriptions |
| `/subscriptions/purchase` | POST | Purchase & activate a subscription (tier + seats) |
| `/headcount` | GET | Active vs available headcount across all subscriptions |
| `/billing-cycle` | PUT | Manage the billing cycle |
| `/invoices` | GET | List invoices |
| `/invoices/:id` | GET | Get/download an invoice |
| `/workers/:id/iris` | PATCH | Opt a worker in/out of IRIS payroll integration |
| `/payroll/export` | GET | Export payroll data (`?format=csv\|xml`, `?target=sage\|xero\|brightpay\|quickbooks`, period) |

**POST /subscriptions/purchase — Request**
```json
{ "tier": "payroll_enabled", "seats": 10, "billingCycle": "monthly" }
```
**POST /subscriptions/purchase — Response 201**
```json
{ "success": true, "subscription": { "id": "vsub1", "tier": "payroll_enabled", "unitPrice": 5, "seats": 10, "monthlyTotal": 50, "status": "active" } }
```

**GET /headcount — Response 200**
```json
{ "totalSeats": 30, "activeSeats": 24, "availableSeats": 6, "byTier": { "base": { "seats": 20, "active": 16 }, "payroll_enabled": { "seats": 10, "active": 8 } } }
```

**PATCH /workers/:id/iris — Request**
```json
{ "irisEnabled": true }
```

**GET /payroll/export?format=csv&target=xero&period=2026-06 — Response 200**
Returns a Xero-compatible CSV of payroll data for the period.

### 45.6 UI Layout (brief)
- **Subscriptions** (`/vendor/subscriptions`): current subscriptions, a **Purchase** flow (choose **Base £3.5** vs **Payroll-enabled £5**, seat count → shows total), and a headcount summary (active vs available).
- **Billing & Invoices**: billing cycle control + invoice list with **Download**.
- **Payroll**: per-worker **IRIS** opt-in toggle; **Export** to Sage / Xero / BrightPay / QuickBooks (CSV/XML).

### 45.7 Implementation Phases
- **Phase 1 — Mock:** Subscriptions, headcount, invoices, IRIS flags, and exports from mock data; export returns sample files.
- **Phase 2 — Real Data:** Real purchase/activation via the payment provider; invoices generated; IRIS integration wired; payroll exports generated in each target's format.

### 45.8 Acceptance Criteria
- Vendor can purchase and activate Base (£3.5) or Payroll-enabled (£5) per-user subscriptions.
- Vendor can view active vs available headcount across all subscriptions.
- Vendor can manage the billing cycle and view/download invoices.
- Vendor can opt workers in/out of IRIS payroll integration.
- Vendor can export payroll data as CSV/XML for Sage, Xero, BrightPay, and QuickBooks.
- All data/actions are vendor-scoped; endpoints reject other roles.

### 45.9 Open Questions
- Is **IRIS opt-in** restricted to payroll-enabled seats only?
- **Payment provider** for purchase/renewal (e.g. Stripe)? Are invoices generated there or here?
- Exact **export field mappings** required per target (Sage/Xero/BrightPay/QuickBooks) and CSV vs XML per target?
- Can a vendor **mix tiers** (some base, some payroll-enabled seats) — assumed yes per §45.5 headcount?
- What happens to **active workers** if seats are reduced below activeSeats?

---

## 46. Vendor Manager — Work Request / Bidding Module

### 46.1 Overview
**Module #3** of the Vendor Manager role (§43.2). It lets the vendor see incoming work requests (from hotels that nominated or discovered it), have the system auto-validate subscription headcount before bidding, submit bids / confirm availability, decline with a reason, and review bid history with won/lost status.

Route: `/vendor/work-requests`

**Scope:** the manager's **own vendor**. Requests here are those routed to the vendor by properties (see §33); the Superadmin observes the same bidding platform-wide (§18).

**Actions & Functions**
1. **View incoming work requests** — from nominated or discovered hotels.
2. **Validate subscription headcount before bidding** — the system auto-checks available seats (§45).
3. **Submit a bid or confirm availability** for a request.
4. **Decline a request with a reason.**
5. **View bid history and won/lost status.**

### 46.2 Bid Status
| Status | Meaning |
|--------|---------|
| submitted | Bid placed / availability confirmed |
| won | Bid accepted (request awarded to this vendor) |
| lost | Another vendor was awarded |
| declined | Vendor declined the request |
| withdrawn | Vendor withdrew the bid |

### 46.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-VW1 | Vendor can view incoming work requests from nominated or discovered hotels | High |
| FR-VW2 | Requests are filterable by source (nominated/discovered) and status | Medium |
| FR-VW3 | System **auto-validates available headcount** (§45) before allowing a bid | High |
| FR-VW4 | Vendor is blocked/warned if available headcount < requested headcount | High |
| FR-VW5 | Vendor can **submit a bid** or **confirm availability** for a request | High |
| FR-VW6 | Vendor can **decline** a request with a reason | High |
| FR-VW7 | Vendor can view **bid history** with won/lost status | High |
| FR-VW8 | All data/actions are scoped to the manager's **vendor** | High |
| FR-VW9 | Restricted to `role = vendor_manager` | High |

### 46.4 Data Model
**IncomingWorkRequest (read model)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Work request id |
| propertyName | string | Requesting property/hotel |
| designation | string | Job role/designation needed |
| startDate | date | Work start |
| endDate | date | Work end |
| shift | enum | `morning` \| `evening` \| `night` |
| headcount | integer | Workers required |
| source | enum | `nominated` \| `discovered` |
| status | enum | `open` \| `bidding` \| `awarded` \| `closed` |
| bidDeadline | timestamp (nullable) | Deadline to respond |

**Bid**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| workRequestId | UUID | FK -> work request |
| vendorId | UUID | The manager's vendor |
| amount | number (nullable) | Bid amount (if priced) |
| headcountOffered | integer | Workers the vendor can provide |
| availabilityConfirmed | boolean | Availability confirmation |
| message | string (nullable) | Note to the property |
| status | enum | `submitted` \| `won` \| `lost` \| `declined` \| `withdrawn` |
| declineReason | string (nullable) | If declined |
| createdAt | timestamp | |

### 46.5 API / Interface Design
Base path: `/api/vendor/work-requests` — requires `Bearer <jwt>` with `role = vendor_manager`; scoped to the caller's vendor.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/work-requests` | GET | List incoming requests (filter by `?source=`, `?status=`) |
| `/work-requests/:id` | GET | Get a request's detail (incl. a headcount-availability check) |
| `/work-requests/:id/bid` | POST | Submit a bid / confirm availability (**auto headcount check**) |
| `/work-requests/:id/decline` | POST | Decline a request with a reason |
| `/bids` | GET | Bid history with won/lost status |

**GET /work-requests/:id — Response 200** (with availability check)
```json
{
  "id": "wr30", "propertyName": "Sunrise Apartments", "designation": "Housekeeper",
  "startDate": "2026-07-15", "endDate": "2026-07-18", "shift": "morning",
  "headcount": 4, "source": "nominated", "status": "bidding",
  "availability": { "requiredHeadcount": 4, "availableHeadcount": 6, "canBid": true }
}
```

**POST /work-requests/:id/bid — Request**
```json
{ "headcountOffered": 4, "availabilityConfirmed": true, "amount": 320, "message": "Available from Monday" }
```
**POST /work-requests/:id/bid — Response 201 / 409**
```json
{ "success": true, "bid": { "id": "b30", "workRequestId": "wr30", "status": "submitted" } }
```
```json
{ "success": false, "message": "Insufficient available headcount (need 4, available 2)" }
```

**POST /work-requests/:id/decline — Request**
```json
{ "reason": "No available workers for these dates" }
```

**GET /bids — Response 200**
```json
{
  "items": [
    { "id": "b30", "propertyName": "Sunrise Apartments", "designation": "Housekeeper", "status": "submitted" },
    { "id": "b27", "propertyName": "Palm Villa",         "designation": "Electrician", "status": "won" },
    { "id": "b22", "propertyName": "Lake View",          "designation": "Cleaner",     "status": "lost" }
  ]
}
```

### 46.6 UI Layout (brief)
- **Incoming requests** (`/vendor/work-requests`): list with property, designation, dates, shift, headcount, source badge (Nominated/Discovered), and a **headcount availability** indicator.
- **Request detail**: **Submit Bid / Confirm Availability** (blocked with a message if headcount insufficient) and **Decline** (reason).
- **Bid history**: list with won/lost/submitted status.

### 46.7 Implementation Phases
- **Phase 1 — Mock:** Incoming requests and bids from mock data; headcount check against a mock available count.
- **Phase 2 — Real Data:** Requests routed from properties (§33); headcount validated against real subscription seats (§45); bids feed the platform bidding (§18) and property selection.

### 46.8 Acceptance Criteria
- Vendor can view incoming requests from nominated and discovered hotels.
- System auto-validates available headcount before allowing a bid; insufficient headcount is blocked with a message.
- Vendor can submit a bid / confirm availability and decline with a reason.
- Vendor can view bid history with won/lost status.
- All data/actions are vendor-scoped; endpoints reject other roles.

### 46.9 Open Questions
- Does headcount validation count **available seats** (§45), **available workers**, or both?
- Is bidding **priced** (amount) or **availability-only** — depends on the property's routing (§33)?
- Can a vendor **withdraw** a submitted bid before award?
- On **won**, does the flow auto-advance to Worker Assignment (§48)?

---

## 47. Vendor Manager — Worker Management Module

### 47.1 Overview
**Module #4** of the Vendor Manager role (§43.2). It lets the vendor build and manage its workforce — adding workers, accepting self-nominations, tagging designations, viewing each worker's primary/secondary agency status, activating/deactivating workers against subscription headcount, and removing workers.

Route: `/vendor/workers`

**Scope:** the manager's **own vendor/agency**.

**Cross-module links:** activating/deactivating consumes/frees **subscription seats (§45)**; primary/secondary agency status is governed platform-wide by **Worker Management (§17)**.

**Actions & Functions**
1. **Add workers to the agency** — enter name, mobile, email; the **worker completes registration**.
2. **Accept worker self-nominations** into the agency.
3. **Assign designation tags** — Chef, Waiter, Front Desk, etc.
4. **View worker primary/secondary agency status.**
5. **Activate or deactivate workers** from active subscription headcount.
6. **Remove a worker** from the agency — headcount reduces accordingly.

### 47.2 Worker Registration & Status
| Field | Values | Meaning |
|-------|--------|---------|
| registrationStatus | `invited` \| `registered` | Whether the worker has completed self-registration |
| status | `active` \| `inactive` | Active consumes a subscription seat; inactive frees it |
| agencyRole (this agency) | `primary` \| `secondary` | This agency's relationship to the worker (see §17) |

### 47.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-VWM1 | Vendor can **add a worker** (name, mobile, email); the worker completes registration | High |
| FR-VWM2 | Vendor can **accept worker self-nominations** into the agency | High |
| FR-VWM3 | Vendor can **reject** a self-nomination | Medium |
| FR-VWM4 | Vendor can **assign designation tags** to workers (Chef, Waiter, Front Desk, …) | High |
| FR-VWM5 | Vendor can view a worker's **primary/secondary agency status** (§17) | High |
| FR-VWM6 | Vendor can **activate/deactivate** a worker against active subscription headcount | High |
| FR-VWM7 | Activating a worker requires an **available seat** (§45); else blocked | High |
| FR-VWM8 | Vendor can **remove a worker** from the agency; **headcount reduces** accordingly | High |
| FR-VWM9 | All data/actions are scoped to the manager's **vendor** | High |
| FR-VWM10 | Restricted to `role = vendor_manager` | High |

### 47.4 Data Model
**Worker (agency view)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Worker id |
| name | string | Full name |
| mobile | string | Contact mobile |
| email | string | Contact email |
| registrationStatus | enum | `invited` \| `registered` |
| designations | string[] | Tags: `chef`, `waiter`, `front_desk`, … |
| agencyRole | enum | `primary` \| `secondary` (this agency, per §17) |
| status | enum | `active` \| `inactive` |
| seatConsumed | boolean | True while active (consumes a subscription seat) |
| createdAt | timestamp | |

**WorkerNomination (self-nomination)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| workerId | UUID (nullable) | Existing worker, if applicable |
| name | string | Worker name |
| mobile | string | Contact mobile |
| email | string | Contact email |
| status | enum | `pending` \| `accepted` \| `rejected` |
| createdAt | timestamp | |

### 47.5 API / Interface Design
Base path: `/api/vendor/workers` — requires `Bearer <jwt>` with `role = vendor_manager`; scoped to the caller's vendor.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/workers` | GET | List the agency's workers (status, designations, agency role) |
| `/workers/:id` | GET | Get a worker's detail (incl. primary/secondary status) |
| `/workers` | POST | Add a worker (name, mobile, email) → sends registration invite |
| `/workers/:id/designations` | PUT | Assign/update designation tags |
| `/workers/:id/activate` | PATCH | Activate (consumes a seat; **409 if none available**) |
| `/workers/:id/deactivate` | PATCH | Deactivate (frees the seat) |
| `/workers/:id` | DELETE | Remove worker from the agency (headcount reduces) |
| `/nominations` | GET | List worker self-nominations |
| `/nominations/:id/accept` | PATCH | Accept a self-nomination |
| `/nominations/:id/reject` | PATCH | Reject a self-nomination |

**POST /workers — Request**
```json
{ "name": "Alex Turner", "mobile": "+44 7911 223344", "email": "alex.turner@work.com" }
```
**POST /workers — Response 201**
```json
{ "success": true, "worker": { "id": "w40", "name": "Alex Turner", "registrationStatus": "invited", "status": "inactive" } }
```

**PUT /workers/:id/designations — Request**
```json
{ "designations": ["chef", "front_desk"] }
```

**PATCH /workers/:id/activate — Response 200 / 409**
```json
{ "success": true, "workerId": "w40", "status": "active", "seatConsumed": true }
```
```json
{ "success": false, "message": "No available subscription seats (upgrade in §45)" }
```

**GET /workers/:id — Response 200** (primary/secondary)
```json
{ "id": "w1", "name": "John Doe", "agencyRole": "primary", "designations": ["waiter"], "status": "active", "otherAgencies": [{ "vendorName": "Summit Maintenance", "role": "secondary" }] }
```

### 47.6 UI Layout (brief)
- **Workers** (`/vendor/workers`): list with name, registration status, designations, agency role (primary/secondary), and active/inactive toggle.
- **Add Worker** (name, mobile, email → invite) and **Self-nominations** queue with **Accept / Reject**.
- **Designation tags** editor per worker; **Activate/Deactivate** (seat-aware) and **Remove**.

### 47.7 Implementation Phases
- **Phase 1 — Mock:** Workers, nominations, designations, and status from mock data; seat checks against a mock available count.
- **Phase 2 — Real Data:** Registration invites; self-nomination flow; activation validated against subscription seats (§45); primary/secondary status from §17; removal adjusts headcount.

### 47.8 Acceptance Criteria
- Vendor can add a worker (name/mobile/email) and the worker can complete registration.
- Vendor can accept/reject worker self-nominations.
- Vendor can assign designation tags and view primary/secondary agency status.
- Vendor can activate/deactivate workers against available seats and remove workers (headcount reduces).
- All data/actions are vendor-scoped; endpoints reject other roles.

### 47.9 Open Questions
- Is the **designation** list a fixed platform taxonomy or vendor-defined tags?
- When a worker is **primary** at another agency, can this agency still activate them (secondary)? (See §17 conflicts.)
- Does **removing** a worker require them to have **no active assignments** first?
- Does adding a worker **auto-activate** (consume a seat) once they register, or stay inactive until activated?

---

## 48. Vendor Manager — Worker Assignment Module

### 48.1 Overview
**Module #5** of the Vendor Manager role (§43.2). After winning a bid (§46), it lets the vendor staff the job — selecting primary and backup workers, tracking each worker's acceptance/decline within a response window, reassigning from the backup pool when a primary declines, and viewing worker geofence location during active shifts.

Route: `/vendor/assignments`

**Scope:** the manager's **own vendor**. Assignments originate from **won bids (§46)** and draw from the vendor's **active workers (§47)**.

**Actions & Functions**
1. **Select specific workers for a won bid** — primary selection + backup selection.
2. **View worker acceptance/decline status** and the **response window countdown**.
3. **Reassign from the backup pool** if a primary worker declines.
4. **View worker location via geofence data** during active shifts.

### 48.2 Assignment & Response Status
| Field | Values | Meaning |
|-------|--------|---------|
| assignmentRole | `primary` \| `backup` | Whether the worker is a primary pick or backup |
| responseStatus | `pending` \| `accepted` \| `declined` \| `expired` | Worker's response (expired = no response in window) |
| assignmentStatus | `staffing` \| `confirmed` \| `in_progress` \| `completed` | Overall assignment state |

### 48.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-VA1 | Vendor can **select primary and backup workers** for a won bid | High |
| FR-VA2 | Selection is limited to **active** workers with matching designation/availability | High |
| FR-VA3 | Vendor can view each worker's **acceptance/decline status** | High |
| FR-VA4 | Vendor can see a **response window countdown** per worker | High |
| FR-VA5 | When a primary **declines/expires**, vendor can **reassign from the backup pool** | High |
| FR-VA6 | Vendor can view worker **geofence location** during active shifts | High |
| FR-VA7 | All data/actions are scoped to the manager's **vendor** | High |
| FR-VA8 | Restricted to `role = vendor_manager` | High |

### 48.4 Data Model
**Assignment**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| workRequestId | UUID | The won request (§46) |
| propertyName | string | Site |
| shift | enum | `morning` \| `evening` \| `night` |
| requiredHeadcount | integer | Workers needed |
| status | enum | `staffing` \| `confirmed` \| `in_progress` \| `completed` |
| createdAt | timestamp | |

**AssignmentWorker**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| assignmentId | UUID | FK -> Assignment.id |
| workerId | UUID | FK -> Worker.id |
| workerName | string | Denormalized |
| assignmentRole | enum | `primary` \| `backup` |
| responseStatus | enum | `pending` \| `accepted` \| `declined` \| `expired` |
| responseDeadline | timestamp | Response window end (countdown) |
| respondedAt | timestamp (nullable) | When the worker responded |

**Worker Location (read model, active shift)**
| Field | Type | Notes |
|-------|------|-------|
| workerId | UUID | Worker id |
| latitude | number | Current latitude |
| longitude | number | Current longitude |
| insideGeofence | boolean | Whether inside the property's geofence |
| updatedAt | timestamp | Last location update |

### 48.5 API / Interface Design
Base path: `/api/vendor/assignments` — requires `Bearer <jwt>` with `role = vendor_manager`; scoped to the caller's vendor.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/assignments` | GET | List assignments (won bids to staff) |
| `/assignments/:id` | GET | Get an assignment's detail |
| `/assignments/:id/select` | POST | Select primary + backup workers |
| `/assignments/:id/responses` | GET | Worker acceptance/decline + response countdown |
| `/assignments/:id/reassign` | PATCH | Promote a backup when a primary declines/expires |
| `/assignments/:id/locations` | GET | Worker geofence locations during the active shift |

**POST /assignments/:id/select — Request**
```json
{ "primaryWorkerIds": ["w1", "w4"], "backupWorkerIds": ["w7"] }
```

**GET /assignments/:id/responses — Response 200**
```json
{
  "assignmentId": "as10",
  "workers": [
    { "workerId": "w1", "workerName": "John Doe",   "assignmentRole": "primary", "responseStatus": "accepted", "responseDeadline": "2026-07-14T18:00:00Z", "secondsRemaining": 0 },
    { "workerId": "w4", "workerName": "Amit Verma",  "assignmentRole": "primary", "responseStatus": "pending",  "responseDeadline": "2026-07-14T18:00:00Z", "secondsRemaining": 5400 },
    { "workerId": "w7", "workerName": "Sara Ali",    "assignmentRole": "backup",  "responseStatus": "pending",  "responseDeadline": null, "secondsRemaining": null }
  ]
}
```

**PATCH /assignments/:id/reassign — Request**
```json
{ "declinedWorkerId": "w4", "backupWorkerId": "w7" }
```
**PATCH /assignments/:id/reassign — Response 200**
```json
{ "success": true, "promoted": "w7", "assignmentRole": "primary", "responseStatus": "pending" }
```

**GET /assignments/:id/locations — Response 200**
```json
{
  "assignmentId": "as10",
  "workers": [
    { "workerId": "w1", "latitude": 51.5074, "longitude": -0.1278, "insideGeofence": true,  "updatedAt": "2026-07-15T09:10:00Z" }
  ]
}
```

### 48.6 UI Layout (brief)
- **Assignments** (`/vendor/assignments`): won bids awaiting staffing; per assignment, **Select Primary** and **Select Backup** from active workers.
- **Responses**: list of selected workers with accept/decline status and a **countdown timer**; declined/expired primaries show a **Reassign from backup** action.
- **Live map**: worker geofence positions during active shifts, with inside/outside-geofence indicator.

### 48.7 Implementation Phases
- **Phase 1 — Mock:** Assignments, responses, countdowns, and locations from mock data.
- **Phase 2 — Real Data:** Assignments from won bids (§46); worker responses and countdowns real-time; reassignment updates roles; live geofence from the attendance/location system.

### 48.8 Acceptance Criteria
- Vendor can select primary and backup workers for a won bid.
- Vendor can view acceptance/decline status and a response-window countdown.
- Vendor can reassign from the backup pool when a primary declines/expires.
- Vendor can view worker geofence location during active shifts.
- All data/actions are vendor-scoped; endpoints reject other roles.

### 48.9 Open Questions
- Default **response window** length, and what happens on **expiry** (auto-promote backup?).
- Can the vendor select **multiple backups** ranked, and is promotion automatic or manual?
- Is live location shown **only during active shifts** (privacy), and at what refresh rate?
- Does an accepted assignment feed the **Attendance & Hours** module (§49) for the shift?

---

## 49. Vendor Manager — Attendance & Hours Module

### 49.1 Overview
**Module #6** of the Vendor Manager role (§43.2). It gives the vendor the attendance and hours record for **its assigned workers** — check-in/out logs, per-worker overtime flags, geofence backup data where a scan-out is missing, and a downloadable hours report for client invoicing.

Route: `/vendor/attendance`

**Scope:** the manager's **own vendor's** assigned workers (across the properties they serve). Uses the same QR/geofence attendance source as the property view (§36).

**Actions & Functions**
1. **View check-in/check-out logs** for all assigned workers.
2. **View overtime flags** per worker.
3. **Download hours report** for client invoicing.
4. **View geofence backup data** for workers who forgot to scan out.

### 49.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-VAH1 | Vendor can view **check-in/check-out logs** for all assigned workers | High |
| FR-VAH2 | Vendor can view **overtime flags** per worker | High |
| FR-VAH3 | Vendor can **download an hours report** for client invoicing | High |
| FR-VAH4 | Hours report exports in CSV/PDF/XLSX | Medium |
| FR-VAH5 | Vendor can view **geofence backup data** where a QR scan-out is missing | High |
| FR-VAH6 | Logs are filterable by worker, property, date, and shift | Medium |
| FR-VAH7 | All data is scoped to the manager's **vendor** | High |
| FR-VAH8 | Restricted to `role = vendor_manager` | High |

### 49.3 Data Model
**AttendanceLog (vendor view)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| workerId | UUID | Worker id |
| workerName | string | Worker name |
| propertyName | string | Site served |
| shiftId | UUID | Shift id |
| date | date | Shift date |
| shift | enum | `morning` \| `evening` \| `night` |
| qrCheckIn | timestamp (nullable) | QR scan-in |
| qrCheckOut | timestamp (nullable) | QR scan-out (null if forgot to scan) |
| geofenceCheckOut | timestamp (nullable) | Geofence backup out (used when QR scan-out missing) |
| source | enum | `qr` \| `geofence` (which provided the record) |
| totalHours | number | Computed worked hours |
| overtimeHours | number | Overtime portion |
| overtimeFlag | boolean | True if overtime present |
| missingScanOut | boolean | True if QR scan-out absent (geofence used) |

### 49.4 API / Interface Design
Base path: `/api/vendor/attendance` — requires `Bearer <jwt>` with `role = vendor_manager`; scoped to the caller's vendor.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/logs` | GET | Check-in/out logs for assigned workers (filter by `?workerId=`, `?propertyId=`, `?date=`, `?shift=`) |
| `/logs/:id` | GET | Get a single attendance log (QR + geofence detail) |
| `/overtime` | GET | Overtime flags per worker |
| `/hours-report` | GET | Download hours report for client invoicing (`?from=`, `?to=`, `?format=`) |

**GET /logs — Response 200**
```json
{
  "items": [
    { "id": "at10", "workerName": "John Doe",  "propertyName": "Sunrise Apts", "shift": "morning", "date": "2026-07-10", "qrCheckIn": "08:02", "qrCheckOut": "16:05", "source": "qr",       "totalHours": 8.05, "overtimeFlag": false, "missingScanOut": false },
    { "id": "at11", "workerName": "Amit Verma", "propertyName": "Palm Villa",   "shift": "evening", "date": "2026-07-10", "qrCheckIn": "14:00", "qrCheckOut": null,     "geofenceCheckOut": "22:30", "source": "geofence", "totalHours": 8.5, "overtimeHours": 0.5, "overtimeFlag": true, "missingScanOut": true }
  ]
}
```

**GET /overtime — Response 200**
```json
{
  "items": [
    { "workerName": "Amit Verma", "propertyName": "Palm Villa", "date": "2026-07-10", "overtimeHours": 0.5 }
  ]
}
```

**GET /hours-report?from=2026-06-01&to=2026-06-30&format=csv — Response 200**
Returns a downloadable hours report suitable for client invoicing.

### 49.5 UI Layout (brief)
- **Attendance** (`/vendor/attendance`): table of assigned workers' logs with QR check-in/out; rows using **geofence backup** (missing scan-out) badged; overtime flagged.
- **Overtime**: per-worker overtime list.
- **Hours report**: date-range picker + **Download** (CSV/PDF/XLSX) for client invoicing.

### 49.6 Implementation Phases
- **Phase 1 — Mock:** Attendance logs (QR + geofence) and overtime from mock data; report download returns a sample file.
- **Phase 2 — Real Data:** Logs from the QR/geofence attendance system filtered to the vendor's assigned workers; reports generated from real data.

### 49.7 Acceptance Criteria
- Vendor can view check-in/out logs for all assigned workers.
- Vendor can view overtime flags per worker.
- Vendor can download an hours report for client invoicing.
- Geofence backup data is shown for workers who missed a scan-out.
- All data is vendor-scoped; endpoints reject other roles.

### 49.8 Open Questions
- Does the vendor's hours report need to **match the property's** cross-check report (§38) exactly (same figures)?
- When a scan-out is missing, is the **geofence-derived** time authoritative for invoicing, or does it require confirmation?
- Should overtime here reflect the property's **pre-approved/queried** status (§36), or raw overtime?
- Invoicing report — include **rates/amounts**, or hours only?

---

## 50. Vendor Manager — Ratings Module

### 50.1 Overview
**Module #7** of the Vendor Manager role (§43.2). It shows the agency's own rating (with shift-count context), its individual workers' ratings, and lets the vendor rate the hotel property and its managers at the end of an assignment.

Route: `/vendor/ratings`

**Scope:** the manager's **own vendor**. Ratings and weightage are governed by **Rating & Trust (§20)**; the shift-count context uses the weightage bands from §20.5.

**Actions & Functions**
1. **View agency's own rating and shift-count context.**
2. **View individual worker ratings.**
3. **Rate hotel property and managers** at the end of an assignment.

### 50.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-VR1 | Vendor can view its **agency rating** with **shift-count context** (shifts + weight band) | High |
| FR-VR2 | Vendor can view **individual worker ratings** for its workers | High |
| FR-VR3 | Vendor can **rate the hotel property** at the end of an assignment (1–5 + comment) | High |
| FR-VR4 | Vendor can **rate property managers** at the end of an assignment (1–5 + comment) | High |
| FR-VR5 | Rating is available only after an assignment **completes** **[CONFIRM]** | Medium |
| FR-VR6 | All data/actions are scoped to the manager's **vendor** | High |
| FR-VR7 | Restricted to `role = vendor_manager` | High |

### 50.3 Data Model
**Agency Rating (read model)**
| Field | Type | Notes |
|-------|------|-------|
| vendorId | UUID | The manager's vendor |
| overallRating | number | Weighted rating (0–5) |
| totalShifts | integer | Shifts informing the rating |
| weightBand | string | Applied band from §20.5 (e.g. "11+ shifts") |
| weightPercent | number | Weight % for the band |

**Worker Rating (read model)**
| Field | Type | Notes |
|-------|------|-------|
| workerId | UUID | Worker id |
| workerName | string | Worker name |
| rating | number | Rating (0–5) |
| shiftCount | integer | Shifts informing the rating |

**PropertyRating (write → §20)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| subjectType | enum | `property` \| `property_manager` |
| subjectId | UUID | Property or manager id |
| assignmentId | UUID | The completed assignment being rated |
| score | number | 1–5 |
| comment | string (nullable) | Review text |
| ratedBy | UUID | Vendor Manager id |
| createdAt | timestamp | |

> **[CONFIRM]** with §20 whether `property_manager` is a valid rating subject (§20.5 currently lists `worker`/`vendor`/`property`).

### 50.4 API / Interface Design
Base path: `/api/vendor/ratings` — requires `Bearer <jwt>` with `role = vendor_manager`; scoped to the caller's vendor.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/agency` | GET | Agency's own rating with shift-count context |
| `/workers` | GET | Individual worker ratings |
| `/property-rating` | POST | Rate a hotel property or manager for a completed assignment |

**GET /agency — Response 200**
```json
{ "vendorId": "v1", "overallRating": 4.6, "totalShifts": 128, "weightBand": "11+ shifts", "weightPercent": 100 }
```

**GET /workers — Response 200**
```json
{
  "items": [
    { "workerId": "w1", "workerName": "John Doe",  "rating": 4.8, "shiftCount": 42 },
    { "workerId": "w4", "workerName": "Amit Verma", "rating": 4.1, "shiftCount": 7 }
  ]
}
```

**POST /property-rating — Request**
```json
{ "subjectType": "property", "subjectId": "p1", "assignmentId": "as10", "score": 5, "comment": "Well-organised site, clear briefs" }
```
**POST /property-rating — Response 201**
```json
{ "success": true, "rating": { "id": "rt50", "subjectType": "property", "subjectId": "p1", "score": 5 } }
```

### 50.5 UI Layout (brief)
- **Ratings** (`/vendor/ratings`): agency rating card with **shift-count context** (shifts + weight band); list of workers with their ratings and shift counts.
- **Rate Property / Manager**: available on completed assignments — pick property or manager, 1–5 + comment.

### 50.6 Implementation Phases
- **Phase 1 — Mock:** Agency/worker ratings and shift context from mock data; property/manager ratings appended to mock records.
- **Phase 2 — Real Data:** Ratings and weighted scores from §20; shift counts from attendance; property/manager ratings written to §20.

### 50.7 Acceptance Criteria
- Vendor can view its agency rating with shift-count context (shifts + weight band).
- Vendor can view individual worker ratings.
- Vendor can rate the hotel property and managers at the end of an assignment.
- All data/actions are vendor-scoped; endpoints reject other roles.

### 50.8 Open Questions
- Is **property_manager** a valid rating subject in §20, or should manager ratings map to the property?
- Can the vendor rate a property/manager **once per assignment**, or multiple times?
- Should the agency rating show a **breakdown by band** (per §20.5), not just the applied band?
- Are worker ratings here **read-only** aggregates (the vendor doesn't rate its own workers — PM/DM do, §35/§41)?

---

## 51. Vendor Manager — Reports & Payroll Module

### 51.1 Overview
**Module #8** of the Vendor Manager role (§43.2). It gives the vendor payroll and margin reporting — per-worker payroll summaries (hours × rate), margin summaries (charge rate − pay rate), payroll-file exports for external systems, and payment confirmation reports received from hotels.

Route: `/vendor/reports-payroll`

**Scope:** the manager's **own vendor**.

> **Relationship to §45:** the payroll **export** capability also appears in Subscription Management (§45.5). This module is the reporting/payroll home; §45 exposes export as part of the payroll-enabled subscription. **[CONFIRM]** the single source (targets here: **IRIS, Sage, Xero, BrightPay, manual**; §45 listed Sage/Xero/BrightPay/QuickBooks).

**Actions & Functions**
1. **View payroll summary per worker** — hours × rate.
2. **View margin summary** — charge rate − pay rate per worker.
3. **Export payroll file** — IRIS, Sage, Xero, BrightPay, or manual payroll.
4. **View payment confirmation reports** received from hotels.

### 51.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-VRP1 | Vendor can view **payroll summary per worker** (hours × pay rate = gross) | High |
| FR-VRP2 | Vendor can view **margin summary** per worker (charge rate − pay rate) | High |
| FR-VRP3 | Vendor can **export a payroll file** for IRIS, Sage, Xero, BrightPay, or manual | High |
| FR-VRP4 | Vendor can **view payment confirmation reports** received from hotels | High |
| FR-VRP5 | Reports are filterable by worker and date range | Medium |
| FR-VRP6 | All data/actions are scoped to the manager's **vendor** | High |
| FR-VRP7 | Restricted to `role = vendor_manager` | High |

### 51.3 Data Model
**PayrollSummary (per worker)**
| Field | Type | Notes |
|-------|------|-------|
| workerId | UUID | Worker id |
| workerName | string | Worker name |
| period | string | Payroll period |
| hours | number | Worked hours |
| payRate | number | Pay rate (per hour) |
| grossPay | number | `hours × payRate` |

**MarginSummary (per worker)**
| Field | Type | Notes |
|-------|------|-------|
| workerId | UUID | Worker id |
| workerName | string | Worker name |
| hours | number | Worked hours |
| chargeRate | number | Rate charged to the hotel (per hour) |
| payRate | number | Rate paid to the worker (per hour) |
| marginPerHour | number | `chargeRate − payRate` |
| totalMargin | number | `marginPerHour × hours` |

**PaymentConfirmation (from hotels)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| propertyName | string | Paying hotel/property |
| period | string | Period covered |
| amount | number | Confirmed amount |
| status | enum | `received` \| `pending` |
| confirmedAt | timestamp (nullable) | When confirmed |

### 51.4 API / Interface Design
Base path: `/api/vendor/payroll` — requires `Bearer <jwt>` with `role = vendor_manager`; scoped to the caller's vendor.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/summary` | GET | Payroll summary per worker (hours × rate) |
| `/margin` | GET | Margin summary per worker (charge − pay) |
| `/export` | GET | Export payroll file (`?target=iris\|sage\|xero\|brightpay\|manual`, `?format=`, period) |
| `/payment-confirmations` | GET | Payment confirmation reports received from hotels |

**GET /summary — Response 200**
```json
{
  "period": "2026-06",
  "items": [
    { "workerId": "w1", "workerName": "John Doe",  "hours": 168, "payRate": 12.5, "grossPay": 2100 },
    { "workerId": "w4", "workerName": "Amit Verma", "hours": 152, "payRate": 11.0, "grossPay": 1672 }
  ],
  "totalGross": 3772
}
```

**GET /margin — Response 200**
```json
{
  "period": "2026-06",
  "items": [
    { "workerId": "w1", "workerName": "John Doe", "hours": 168, "chargeRate": 16.0, "payRate": 12.5, "marginPerHour": 3.5, "totalMargin": 588 }
  ],
  "totalMargin": 588
}
```

**GET /export?target=iris&format=xml&period=2026-06 — Response 200**
Returns an IRIS-compatible payroll file for the period.

**GET /payment-confirmations — Response 200**
```json
{
  "items": [
    { "id": "pc1", "propertyName": "Sunrise Apartments", "period": "2026-06", "amount": 42000, "status": "received", "confirmedAt": "2026-07-05T10:00:00Z" },
    { "id": "pc2", "propertyName": "Palm Villa",         "period": "2026-06", "amount": 18500, "status": "pending",  "confirmedAt": null }
  ]
}
```

### 51.5 UI Layout (brief)
- **Payroll** (`/vendor/reports-payroll`): per-worker payroll summary (hours × rate, gross) with totals.
- **Margin**: per-worker charge vs pay with margin and totals.
- **Export**: choose target (IRIS / Sage / Xero / BrightPay / manual) + format → **Download**.
- **Payment Confirmations**: list of hotel payment confirmations (received/pending).

### 51.6 Implementation Phases
- **Phase 1 — Mock:** Payroll, margin, exports, and payment confirmations from mock data; export returns sample files.
- **Phase 2 — Real Data:** Payroll from real hours (§49) × rates; margin from charge vs pay rates; exports in each target's format; payment confirmations from hotel/billing records.

### 51.7 Acceptance Criteria
- Vendor can view payroll summary per worker (hours × rate).
- Vendor can view margin summary (charge − pay per worker).
- Vendor can export a payroll file for IRIS, Sage, Xero, BrightPay, or manual.
- Vendor can view payment confirmation reports received from hotels.
- All data/actions are vendor-scoped; endpoints reject other roles.

### 51.8 Open Questions
- Where are **pay rate** and **charge rate** set — per worker, per assignment, or per designation?
- Consolidate **payroll export** with §45 — single set of targets/formats (IRIS/Sage/Xero/BrightPay/QuickBooks/manual)?
- Are **payment confirmations** entered by hotels, or derived from the billing/payment system?
- Should margin be visible to **anyone else** (e.g. Superadmin analytics), or vendor-only?

---

## 52. Vendor Manager — Dispute Management Module

### 52.1 Overview
**Module #9** of the Vendor Manager role (§43.2). It is the vendor side of disputes — raising disputes against hotel-reported hours with evidence, tracking status and the Superadmin's resolution, and responding to hotel-raised disputes with counter-evidence (QR/geofence data auto-presented).

Route: `/vendor/disputes`

**Scope:** the manager's **own vendor**. Disputes are adjudicated by the Superadmin in **Dispute Resolution (§21)**.

**Actions & Functions**
1. **Raise a dispute against hotel-reported hours** — with supporting evidence.
2. **View dispute status and Superadmin resolution.**
3. **Respond to hotel-raised disputes** with counter-evidence — QR/geofence data is presented automatically.

### 52.2 Dispute Status (vendor view)
| Status | Meaning |
|--------|---------|
| open | Raised, awaiting review |
| under_review | Being reviewed by the Superadmin (§21) |
| awaiting_vendor_response | A hotel-raised dispute needs the vendor's counter-evidence |
| resolved | Superadmin made a binding decision |
| closed | Finalised |

### 52.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-VDM1 | Vendor can **raise a dispute** against hotel-reported hours with **evidence upload** | High |
| FR-VDM2 | Vendor can **view dispute status** and the **Superadmin resolution** | High |
| FR-VDM3 | Vendor can **respond to hotel-raised disputes** with counter-evidence | High |
| FR-VDM4 | QR/geofence data for the disputed shift is **auto-presented/attached** on response | High |
| FR-VDM5 | Vendor can view all disputes involving its workers/assignments | High |
| FR-VDM6 | All data/actions are scoped to the manager's **vendor** | High |
| FR-VDM7 | Restricted to `role = vendor_manager` | High |

### 52.4 Data Model
**Dispute (vendor view)** — adjudicated in §21.
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| shiftId | UUID | Disputed shift |
| workerName | string | Worker involved |
| propertyName | string | Hotel/property |
| raisedByType | enum | `vendor` \| `property` (who raised it) |
| claimedHours | number | Hours asserted by the raiser |
| recordedHours | number | System-recorded hours |
| status | enum | `open` \| `under_review` \| `awaiting_vendor_response` \| `resolved` \| `closed` |
| resolution | string (nullable) | Superadmin's binding decision (§21) |
| approvedHours | number (nullable) | Final decided hours |
| evidence | string[] | Uploaded evidence URLs |
| createdAt | timestamp | |

**DisputeResponse (counter-evidence)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| disputeId | UUID | FK -> Dispute.id |
| note | string | Vendor's statement |
| counterEvidence | string[] | Uploaded files |
| autoEvidence | object | Auto-attached QR/geofence data for the shift |
| createdAt | timestamp | |

### 52.5 API / Interface Design
Base path: `/api/vendor/disputes` — requires `Bearer <jwt>` with `role = vendor_manager`; scoped to the caller's vendor.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/disputes` | GET | List disputes (filter by `?status=`, `?raisedBy=`) |
| `/disputes/:id` | GET | Dispute detail, status, Superadmin resolution, and auto QR/geofence data |
| `/disputes` | POST | Raise a dispute against hotel-reported hours (multipart; evidence) |
| `/disputes/:id/respond` | POST | Respond to a hotel-raised dispute with counter-evidence |

**POST /disputes — Request** (`multipart/form-data`)
```
shiftId: s2
claimedHours: 8
recordedHours: 6.5
reason: Hotel under-reported hours; worker on-site until 16:00
evidence: <file1>
```
**POST /disputes — Response 201**
```json
{ "success": true, "dispute": { "id": "d20", "shiftId": "s2", "raisedByType": "vendor", "status": "open" } }
```

**GET /disputes/:id — Response 200** (auto QR/geofence presented)
```json
{
  "id": "d21", "shiftId": "s5", "workerName": "Amit Verma", "propertyName": "Palm Villa",
  "raisedByType": "property", "claimedHours": 7, "recordedHours": 8.5,
  "status": "awaiting_vendor_response",
  "autoEvidence": { "qrCheckIn": "14:00", "qrCheckOut": null, "geofenceCheckOut": "22:30", "insideGeofenceUntil": "22:28" }
}
```

**POST /disputes/:id/respond — Request** (`multipart/form-data`)
```
note: Geofence confirms on-site until 22:30; QR scan-out missed
counterEvidence: <file1>
```
**POST /disputes/:id/respond — Response 200**
```json
{ "success": true, "disputeId": "d21", "status": "under_review" }
```

### 52.6 UI Layout (brief)
- **Disputes** (`/vendor/disputes`): list of vendor- and hotel-raised disputes with status and (when resolved) the Superadmin decision.
- **Raise Dispute**: against hotel-reported hours (claimed vs recorded) with evidence upload.
- **Respond**: for hotel-raised disputes — a panel that **auto-shows the shift's QR/geofence data**, plus counter-evidence upload.

### 52.7 Implementation Phases
- **Phase 1 — Mock:** Disputes, responses, and auto QR/geofence data from mock data.
- **Phase 2 — Real Data:** Disputes created into §21; auto-evidence pulled from the attendance system (§49); status/resolution reflect the Superadmin decision.

### 52.8 Acceptance Criteria
- Vendor can raise a dispute against hotel-reported hours with evidence.
- Vendor can view dispute status and the Superadmin's resolution.
- Vendor can respond to hotel-raised disputes with counter-evidence, with QR/geofence data auto-presented.
- All data/actions are vendor-scoped; endpoints reject other roles.

### 52.9 Open Questions
- Is there a **time limit** to raise a dispute or respond to a hotel-raised one?
- If the vendor **doesn't respond** in time, does the hotel's reported hours stand?
- Does raising/responding **pause payment** for the disputed hours until §21 resolves?
- Should the vendor see the **hotel's evidence** as well as the auto QR/geofence data?

---
---

# Worker Role

> **§1–23 Superadmin · §24–30 Hotel Central Authority · §31–38 Property Manager · §39–42 Department Manager · §43–52 Vendor Manager.** The sections below (§53+) specify the **Worker** role.

## 53. Worker — Role Overview

### 53.1 Overview
The **Worker** is the individual who performs the work (shifts / work orders). This role is primarily delivered via the **mobile application** (React Native / Expo — see the technology stack), and covers registration, receiving and responding to job offers, checking in/out of shifts (QR / geofence), viewing earnings and ratings, and reviewing shift history.

**Authentication:** login is shared across roles (see §1–13). On successful login where the JWT `role = worker`, the worker lands in the **mobile app home**. **[CONFIRM]** the exact landing view (Job Offers or a simple home).

**Scope rule:** every action is limited to the worker's **own profile, offers, shifts, earnings, and ratings**.

### 53.2 Worker Modules
The role is organised into **6 functional modules**. Each has its **own actions and functions** — those will be added as they are shared.

| # | Module | Purpose (summary) | Actions & Functions | Detailed spec |
|---|--------|-------------------|---------------------|---------------|
| 1 | Registration & Profile | OTP self-register; profile/docs; designations; agency + 30-day lock | ✅ Defined | §54 |
| 2 | Job Offers | Multi-channel offers; tiered response window (48/12/4h); auto-decline | ✅ Defined | §55 |
| 3 | Attendance (Check In/Out) | QR check-in/out; geofence auto-close; real-time shift status | ✅ Defined | §56 |
| 4 | Earnings | Today's & historical earnings; per-shift breakdown (no margin/others) | ✅ Defined | §57 |
| 5 | Ratings | View own rating + shift context (read-only; no rating others) | ✅ Defined | §58 |
| 6 | Shift History | Past & upcoming shifts; check-in/out timestamps; overtime records | ✅ Defined | §59 |

> **Status legend:** ✅ Defined · ⏳ Awaiting the actions/functions you'll share.
>
> All 6 modules are defined (§54–§59). Each module section includes its actions/functions, FRs, data model, API, UI, phases, acceptance criteria, and open questions.

### 53.3 Open Questions
- Is the Worker experience **mobile-only**, or also available on web?
- Does registration begin from an **agency invite** (§47) or **self-registration/self-nomination**?
- Can a worker belong to **multiple agencies** (primary/secondary, §17) and see offers from each?

---

## 54. Worker — Registration & Profile Module

### 54.1 Overview
**Module #1** of the Worker role (§53.2). It covers self-registration with OTP verification, profile completion, designation tags, and agency association — including accepting/declining agency nominations, self-nominating, and setting a primary and secondary agency with a **30-day lock** after a switch.

Route (mobile): Registration & Profile screens.

**Scope:** the worker's **own** account. Agency association is governed platform-wide by **Worker Management (§17)** and reflects agency-side actions (§47).

**Actions & Functions**
1. **Self-register** — mobile number, email, OTP verification.
2. **Complete profile** — name, photo, documents.
3. **Add designation tags** — Chef, Waiter, Front Desk, etc.
4. **Accept or decline an agency nomination**, or **self-nominate** to an agency.
5. **Set primary and secondary agency** — a **30-day lock** applies after a switch.
6. **View and update correspondence details** — correspondence address and contact details.

### 54.2 Registration & Agency Rules
| Rule | Detail |
|------|--------|
| OTP verification | Mobile and email verified via OTP before profile completion |
| Primary agency | Exactly one primary agency at a time |
| Secondary agency | Optional additional agency (secondary) |
| **30-day lock** | After switching primary/secondary agency, the worker **cannot switch again for 30 days** |

### 54.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-WRP1 | Worker can **self-register** with mobile number and email | High |
| FR-WRP2 | Mobile and email are verified via **OTP** | High |
| FR-WRP3 | Worker can **complete profile** — name, photo, documents | High |
| FR-WRP4 | Worker can **add designation tags** (Chef, Waiter, Front Desk, …) | High |
| FR-WRP5 | Worker can **accept or decline** an agency nomination | High |
| FR-WRP6 | Worker can **self-nominate** to an agency | High |
| FR-WRP7 | Worker can **set primary and secondary** agency | High |
| FR-WRP8 | A **30-day lock** is enforced after switching primary/secondary agency | High |
| FR-WRP9 | Switching before the lock expires is **blocked** (with the unlock date) | High |
| FR-WRP10 | Worker can **view and update correspondence details** (correspondence address, contact) | High |
| FR-WRP11 | All data/actions are scoped to the worker's **own** account | High |
| FR-WRP12 | Restricted to `role = worker` | High |

### 54.4 Data Model
**WorkerProfile**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Worker id |
| name | string | Full name |
| mobile | string | Mobile number |
| email | string | Email |
| mobileVerified | boolean | OTP verified |
| emailVerified | boolean | OTP verified |
| photo | string (nullable) | Profile photo URL |
| documents | string[] | Uploaded document URLs |
| designations | string[] | Tags: `chef`, `waiter`, `front_desk`, … |
| registrationStatus | enum | `pending` \| `otp_verified` \| `profile_complete` |
| createdAt | timestamp | |

**CorrespondenceDetails**
| Field | Type | Notes |
|-------|------|-------|
| workerId | UUID | FK -> WorkerProfile.id |
| addressLine1 | string | Correspondence address line 1 |
| addressLine2 | string (nullable) | Line 2 |
| city | string | City/town |
| postcode | string | Postal code |
| country | string | Country |
| contactMobile | string (nullable) | Correspondence contact number (if different) |
| contactEmail | string (nullable) | Correspondence email (if different) |
| updatedAt | timestamp | |

**AgencyAssociation (worker view)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| agencyId | UUID | Vendor/agency id |
| agencyName | string | Agency name |
| role | enum | `primary` \| `secondary` |
| status | enum | `nominated` \| `active` \| `declined` |
| nominationSource | enum | `agency` (agency nominated) \| `self` (worker self-nominated) |
| lockedUntil | timestamp (nullable) | 30-day lock expiry after a switch |
| updatedAt | timestamp | |

### 54.5 API / Interface Design
Base path: `/api/worker/profile` — registration endpoints are public (pre-auth); the rest require `Bearer <jwt>` with `role = worker`.

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/register` | POST | Public | Self-register (mobile, email) → sends OTP |
| `/verify-otp` | POST | Public | Verify mobile/email OTP |
| `/profile` | PUT | Bearer | Complete/update profile (name, photo, documents) |
| `/correspondence` | GET | Bearer | View correspondence details |
| `/correspondence` | PUT | Bearer | Update correspondence details |
| `/designations` | PUT | Bearer | Add/update designation tags |
| `/agencies` | GET | Bearer | List nominations and current associations |
| `/agencies/:id/accept` | PATCH | Bearer | Accept an agency nomination |
| `/agencies/:id/decline` | PATCH | Bearer | Decline an agency nomination |
| `/agencies/nominate` | POST | Bearer | Self-nominate to an agency |
| `/agencies/primary` | PUT | Bearer | Set primary agency (**30-day lock**) |
| `/agencies/secondary` | PUT | Bearer | Set secondary agency (**30-day lock**) |

**POST /register — Request**
```json
{ "mobile": "+44 7911 223344", "email": "alex.turner@work.com" }
```
**POST /verify-otp — Request**
```json
{ "mobile": "+44 7911 223344", "otp": "482913" }
```

**PUT /correspondence — Request**
```json
{
  "addressLine1": "12 High Street",
  "addressLine2": "Flat 4",
  "city": "London",
  "postcode": "SW1A 1AA",
  "country": "UK",
  "contactMobile": "+44 7911 223344",
  "contactEmail": "alex.turner@work.com"
}
```

**PUT /agencies/primary — Request**
```json
{ "agencyId": "v1" }
```
**PUT /agencies/primary — Response 200 / 423 (locked)**
```json
{ "success": true, "agencyId": "v1", "role": "primary", "lockedUntil": "2026-08-09T00:00:00Z" }
```
```json
{ "success": false, "message": "Primary agency is locked until 2026-08-09 (30-day lock)" }
```

### 54.6 UI Layout (brief, mobile)
- **Register**: mobile + email → OTP screens (verify both).
- **Profile**: name, photo upload, document upload; **designation tags** selector.
- **Correspondence**: view/edit correspondence address and contact details.
- **Agencies**: nominations with **Accept / Decline**; **Self-nominate** search; **Primary / Secondary** selectors showing the **30-day lock** state and unlock date.

### 54.7 Implementation Phases
- **Phase 1 — Mock:** Registration/OTP stubbed; profile, designations, and agency associations from mock data; lock simulated.
- **Phase 2 — Real Data:** Real OTP via SMS/email; profile/documents stored; agency associations synced with §17/§47; 30-day lock enforced server-side.

### 54.8 Acceptance Criteria
- Worker can self-register and verify mobile and email via OTP.
- Worker can complete profile (name, photo, documents) and add designation tags.
- Worker can view and update correspondence details (address and contact).
- Worker can accept/decline an agency nomination and self-nominate.
- Worker can set primary and secondary agency; switching is blocked for 30 days after a change.
- All data/actions are scoped to the worker; endpoints reject other roles.

### 54.9 Open Questions
- Does the **30-day lock** apply to both primary and secondary, or primary only?
- Which **documents** are required (ID, right-to-work, certifications) and are they verified/approved?
- Is designation a **fixed taxonomy** (shared with §47) or free tags?
- On self-nomination, does the **agency must accept** (§47) before the association becomes active?
- Can a worker **leave** an agency, and how does that interact with the 30-day lock?

---

## 55. Worker — Job Offers Module

### 55.1 Overview
**Module #2** of the Worker role (§53.2). It delivers job offers to the worker across multiple channels, shows offer details, and lets the worker accept/decline within a **tiered response window** based on lead time — offers that aren't answered in time **auto-decline**.

Route (mobile): Job Offers screens.

**Scope:** the worker's **own** offers. Offers originate from the vendor's **Worker Assignment (§48)**; accepting feeds attendance/shift (§56).

**Actions & Functions**
1. **Receive notifications** — push, SMS, WhatsApp, and email for new job offers.
2. **View offer details** — hotel, location, dates, shift times, designation.
3. **Accept or decline within the tiered response window** — 48h / 12h / 4h depending on lead time.
4. **Auto-decline** — offers not answered in time expire and auto-decline.

### 55.2 Tiered Response Window
The response window is derived from the **lead time** (time between the offer and the shift start):
| Lead time (offer → shift start) | Response window |
|---------------------------------|-----------------|
| Long lead time | **48h** |
| Medium lead time | **12h** |
| Short lead time | **4h** |

> **[CONFIRM]** the exact lead-time thresholds for each tier (e.g. >7 days → 48h, 2–7 days → 12h, <2 days → 4h). The window never extends past the shift start.

### 55.3 Offer Status
| Status | Meaning |
|--------|---------|
| pending | Awaiting the worker's response (within the window) |
| accepted | Worker accepted |
| declined | Worker declined |
| expired | Not answered in time → **auto-declined** |

### 55.4 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-WJO1 | Worker receives a new-offer notification via **push, SMS, WhatsApp, and email** | High |
| FR-WJO2 | Worker can **view offer details**: hotel, location, dates, shift times, designation | High |
| FR-WJO3 | Worker can **accept or decline** an offer | High |
| FR-WJO4 | The **response window** is set by lead time (**48h / 12h / 4h**) | High |
| FR-WJO5 | A **countdown** to the response deadline is shown | High |
| FR-WJO6 | Offers not answered by the deadline are **auto-declined** (`expired`) | High |
| FR-WJO7 | All data/actions are scoped to the worker's **own** offers | High |
| FR-WJO8 | Restricted to `role = worker` | High |

### 55.5 Data Model
**JobOffer**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| workerId | UUID | The recipient worker |
| assignmentId | UUID | Source assignment (§48) |
| hotelName | string | Hotel/property |
| location | string | Site location |
| startDate | date | Work start |
| endDate | date | Work end |
| shiftStart | string | Shift start time |
| shiftEnd | string | Shift end time |
| designation | string | Role required (e.g. Chef, Waiter) |
| leadTimeHours | number | Hours between offer and shift start |
| responseWindowHours | enum | `48` \| `12` \| `4` (derived from lead time) |
| offeredAt | timestamp | When the offer was sent |
| responseDeadline | timestamp | Auto-decline deadline |
| status | enum | `pending` \| `accepted` \| `declined` \| `expired` |
| respondedAt | timestamp (nullable) | When the worker responded |

**NotificationDispatch (per offer)**
| Field | Type | Notes |
|-------|------|-------|
| offerId | UUID | FK -> JobOffer.id |
| channels | string[] | `push`, `sms`, `whatsapp`, `email` |
| sentAt | timestamp | When notifications were sent |

### 55.6 API / Interface Design
Base path: `/api/worker/offers` — requires `Bearer <jwt>` with `role = worker`; scoped to the worker's own offers.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/offers` | GET | List offers (filter by `?status=`) with countdown |
| `/offers/:id` | GET | Get an offer's full detail |
| `/offers/:id/accept` | PATCH | Accept an offer (rejected if past deadline) |
| `/offers/:id/decline` | PATCH | Decline an offer |

> Auto-decline is performed by a background job when `now > responseDeadline` (status → `expired`).

**GET /offers — Response 200**
```json
{
  "items": [
    { "id": "of1", "hotelName": "Sunrise Apartments", "designation": "Housekeeper", "startDate": "2026-07-20", "shiftStart": "08:00", "responseWindowHours": 48, "responseDeadline": "2026-07-18T10:00:00Z", "secondsRemaining": 84000, "status": "pending" },
    { "id": "of2", "hotelName": "Palm Villa",          "designation": "Waiter",     "startDate": "2026-07-11", "shiftStart": "14:00", "responseWindowHours": 4,  "responseDeadline": "2026-07-10T18:00:00Z", "secondsRemaining": 3600,  "status": "pending" }
  ]
}
```

**GET /offers/:id — Response 200**
```json
{
  "id": "of1", "hotelName": "Sunrise Apartments", "location": "12 High Street, London",
  "startDate": "2026-07-20", "endDate": "2026-07-22", "shiftStart": "08:00", "shiftEnd": "16:00",
  "designation": "Housekeeper", "leadTimeHours": 240, "responseWindowHours": 48,
  "responseDeadline": "2026-07-18T10:00:00Z", "status": "pending"
}
```

**PATCH /offers/:id/accept — Response 200 / 410 (expired)**
```json
{ "success": true, "offerId": "of1", "status": "accepted" }
```
```json
{ "success": false, "message": "Offer expired and was auto-declined" }
```

### 55.7 UI Layout (brief, mobile)
- **Offers list**: cards with hotel, designation, dates, shift times, and a **countdown** to the response deadline.
- **Offer detail**: hotel, location, dates, shift times, designation; **Accept / Decline** buttons with the remaining time.
- **Notifications**: push/SMS/WhatsApp/email deep-link into the offer.

### 55.8 Implementation Phases
- **Phase 1 — Mock:** Offers and countdowns from mock data; notifications stubbed; accept/decline updates mock state.
- **Phase 2 — Real Data:** Offers from §48 assignments; notifications sent via push/SMS/WhatsApp/email providers; response window computed from lead time; background job auto-declines on expiry.

### 55.9 Acceptance Criteria
- Worker receives new-offer notifications via push, SMS, WhatsApp, and email.
- Worker can view offer details (hotel, location, dates, shift times, designation).
- Worker can accept/decline within a 48h/12h/4h window based on lead time.
- Unanswered offers auto-decline (`expired`) at the deadline.
- All data/actions are worker-scoped; endpoints reject other roles.

### 55.10 Open Questions
- Exact **lead-time thresholds** mapping to 48h / 12h / 4h.
- Which channels are **mandatory** vs. per the worker's notification preferences?
- On accept, does it **confirm** the assignment (§48) immediately, or is it provisional until the vendor confirms?
- Does repeated **auto-decline/decline** affect the worker's rating or offer priority?

---

## 56. Worker — Attendance (Check In/Out) Module

### 56.1 Overview
**Module #3** of the Worker role (§53.2). It lets the worker check in and out of shifts by scanning a QR code at the property, auto-closes a shift from geofence last-seen data when check-out is missed, and shows real-time shift status.

Route (mobile): Attendance / active-shift screen.

**Scope:** the worker's **own** shifts. The QR/geofence records produced here are the source consumed by the property (§36), vendor (§49), and dispute (§21/§52) modules.

**Actions & Functions**
1. **Scan QR code to check in** at the start of a shift.
2. **Scan QR code to check out** at the end of a shift.
3. **Auto-close shift** using **geofence last-seen** data if check-out is missed.
4. **View real-time shift status** — active, overtime, checked out.

### 56.2 Shift Status
| Status | Meaning |
|--------|---------|
| scheduled | Assigned, not yet started |
| active | Checked in, shift in progress |
| overtime | Active beyond scheduled end |
| checked_out | Checked out via QR |
| auto_closed | Closed by the system from geofence last-seen (missed check-out) |

### 56.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-WA1 | Worker can **scan a QR code** at the property to **check in** at shift start | High |
| FR-WA2 | Worker can **scan a QR code** to **check out** at shift end | High |
| FR-WA3 | Check-in/out captures **timestamp and geofence** position | High |
| FR-WA4 | If check-out is missed, the system **auto-closes** the shift using **geofence last-seen** data | High |
| FR-WA5 | Worker can view **real-time shift status** (active / overtime / checked out) | High |
| FR-WA6 | QR scan is validated against the **property's** code (rejects wrong/expired QR) | High |
| FR-WA7 | All data/actions are scoped to the worker's **own** shifts | High |
| FR-WA8 | Restricted to `role = worker` | High |

### 56.4 Data Model
**Shift (worker view)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Shift id |
| assignmentId | UUID | Source assignment (§48) |
| propertyName | string | Property/site |
| date | date | Shift date |
| shiftStart | string | Scheduled start time |
| shiftEnd | string | Scheduled end time |
| qrCheckIn | timestamp (nullable) | QR check-in time |
| qrCheckOut | timestamp (nullable) | QR check-out time |
| geofenceLastSeen | timestamp (nullable) | Last geofence presence (for auto-close) |
| status | enum | `scheduled` \| `active` \| `overtime` \| `checked_out` \| `auto_closed` |
| overtimeHours | number | Overtime accrued |

**CheckEvent** (scan record)
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| shiftId | UUID | FK -> Shift.id |
| type | enum | `check_in` \| `check_out` |
| method | enum | `qr` \| `geofence_auto` |
| timestamp | timestamp | When it occurred |
| latitude | number | Location at scan |
| longitude | number | Location at scan |
| insideGeofence | boolean | Whether inside the property geofence |

### 56.5 API / Interface Design
Base path: `/api/worker/attendance` — requires `Bearer <jwt>` with `role = worker`; scoped to the worker's own shifts.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/shifts/current` | GET | Current/active shift with real-time status |
| `/shifts/:id` | GET | Get a shift's detail |
| `/shifts/:id/check-in` | POST | Check in by scanning the property QR |
| `/shifts/:id/check-out` | POST | Check out by scanning the property QR |

> Auto-close is a background job: when a shift passes its end without check-out, the system closes it using `geofenceLastSeen` (status → `auto_closed`).

**POST /shifts/:id/check-in — Request**
```json
{ "qrToken": "prop-p1-shift-s1-abc123", "latitude": 51.5074, "longitude": -0.1278 }
```
**POST /shifts/:id/check-in — Response 200 / 400 (invalid QR)**
```json
{ "success": true, "shiftId": "s1", "status": "active", "checkInTime": "08:02" }
```
```json
{ "success": false, "message": "Invalid or expired QR code for this property" }
```

**GET /shifts/current — Response 200**
```json
{
  "id": "s1", "propertyName": "Sunrise Apartments", "shiftStart": "08:00", "shiftEnd": "16:00",
  "qrCheckIn": "08:02", "qrCheckOut": null, "status": "active", "overtimeHours": 0,
  "elapsed": "05:12", "insideGeofence": true
}
```

### 56.6 UI Layout (brief, mobile)
- **Active shift**: large **Scan to Check In** / **Scan to Check Out** button (opens QR scanner); real-time status badge (Active / Overtime / Checked out) and elapsed time.
- **Geofence indicator**: shows whether the worker is inside the property geofence.
- **Auto-close note**: if check-out is missed, the shift shows **Auto-closed (geofence)** with the last-seen time.

### 56.7 Implementation Phases
- **Phase 1 — Mock:** QR scan and status simulated; check-in/out updates mock shift; auto-close simulated.
- **Phase 2 — Real Data:** Real QR validation against property codes; geofence capture; background auto-close from last-seen; records feed §36/§49/§21.

### 56.8 Acceptance Criteria
- Worker can check in and out by scanning the property QR code.
- Invalid/expired QR codes are rejected.
- A missed check-out auto-closes the shift from geofence last-seen data.
- Worker can view real-time shift status (active / overtime / checked out).
- All data/actions are worker-scoped; endpoints reject other roles.

### 56.9 Open Questions
- Must the worker be **inside the geofence** for a QR check-in to be accepted?
- Is the QR code **static per property** or **rotating per shift** (anti-fraud)?
- For auto-close, is the shift end taken as **geofenceLastSeen** or the **scheduled end** time?
- Should the worker get a **reminder** to check out before auto-close triggers?
- Offline handling — can scans be **queued** when there's no connectivity?

---

## 57. Worker — Earnings Module

### 57.1 Overview
**Module #4** of the Worker role (§53.2). It shows the worker's own earnings — today's hours and pay, earnings history over a date range, and a per-shift breakdown. It **cannot** show other workers' earnings or the agency's margin.

Route (mobile): Earnings screens.

**Scope:** strictly the worker's **own** earnings. **Agency margin** (charge − pay, §51) and **other workers' earnings** are never exposed to this role.

**Actions & Functions**
1. **View today's hours and today's pay.**
2. **View earnings history** with a date-range filter.
3. **View per-shift breakdown** — base hours, overtime, rate.
4. **No cross-worker / margin visibility** — cannot see other workers' earnings or agency margin.

### 57.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-WE1 | Worker can view **today's hours and pay** | High |
| FR-WE2 | Worker can view **earnings history** filtered by date range | High |
| FR-WE3 | Worker can view a **per-shift breakdown** (base hours, overtime, rate) | High |
| FR-WE4 | Worker **cannot** view **other workers'** earnings | High |
| FR-WE5 | Worker **cannot** view **agency margin** (charge − pay, §51) | High |
| FR-WE6 | Worker scope is **server-enforced** (from the JWT), not client-supplied | High |
| FR-WE7 | Restricted to `role = worker` | High |

### 57.3 Data Model
**Today's Earnings (read model)**
| Field | Type | Notes |
|-------|------|-------|
| date | date | Today |
| hours | number | Hours worked today |
| pay | number | Pay earned today |

**ShiftEarning (per-shift breakdown)**
| Field | Type | Notes |
|-------|------|-------|
| shiftId | UUID | Shift id |
| date | date | Shift date |
| propertyName | string | Site |
| baseHours | number | Base (non-overtime) hours |
| overtimeHours | number | Overtime hours |
| rate | number | Base hourly rate (**pay rate only**) |
| overtimeRate | number (nullable) | Overtime rate, if different |
| basePay | number | `baseHours × rate` |
| overtimePay | number | Overtime pay |
| totalPay | number | Base + overtime |

> The model exposes **pay rate only** — no `chargeRate` or `margin` fields are returned to this role.

### 57.4 API / Interface Design
Base path: `/api/worker/earnings` — requires `Bearer <jwt>` with `role = worker`; scoped server-side to the worker.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/today` | GET | Today's hours and pay |
| `/history` | GET | Earnings history (`?from=`, `?to=`) |
| `/shifts/:id` | GET | Per-shift earning breakdown |

> No `workerId` parameter is accepted — the worker is always taken from the token, so other workers' earnings are unreachable.

**GET /today — Response 200**
```json
{ "date": "2026-07-10", "hours": 8.05, "pay": 100.63 }
```

**GET /history?from=2026-06-01&to=2026-06-30 — Response 200**
```json
{
  "period": { "from": "2026-06-01", "to": "2026-06-30" },
  "totalHours": 168,
  "totalPay": 2100,
  "items": [
    { "shiftId": "s1", "date": "2026-06-02", "propertyName": "Sunrise Apts", "baseHours": 8, "overtimeHours": 0, "rate": 12.5, "totalPay": 100 }
  ]
}
```

**GET /shifts/:id — Response 200**
```json
{
  "shiftId": "s2", "date": "2026-06-05", "propertyName": "Palm Villa",
  "baseHours": 8, "overtimeHours": 1.5, "rate": 12.5, "overtimeRate": 18.75,
  "basePay": 100, "overtimePay": 28.13, "totalPay": 128.13
}
```

### 57.5 UI Layout (brief, mobile)
- **Today**: hours and pay tiles for the current day.
- **History**: date-range filter with a list of shifts and totals.
- **Shift breakdown**: base hours, overtime, rate, and total pay per shift.
- No margin, charge rate, or other-worker data anywhere.

### 57.6 Implementation Phases
- **Phase 1 — Mock:** Today's/history earnings and breakdowns from mock data.
- **Phase 2 — Real Data:** Earnings computed from real hours (§56) × the worker's pay rate; server-enforced scope; margin/charge excluded from all payloads.

### 57.7 Acceptance Criteria
- Worker can view today's hours and pay.
- Worker can view earnings history filtered by date range.
- Worker can view a per-shift breakdown (base hours, overtime, rate).
- No agency margin or other workers' earnings are ever returned.
- Scope is server-enforced; endpoints reject other roles.

### 57.8 Open Questions
- Is "today's pay" **estimated** (from hours so far) or finalised at shift end?
- Are earnings shown **gross** only, or also net (post-deductions) once payroll runs (§51)?
- Does earnings reflect **disputed/queried** hours (§36/§52) as pending until resolved?
- Where is the worker's **pay rate** defined (per §51.8) and can the worker see it?

---

## 58. Worker — Ratings Module

### 58.1 Overview
**Module #5** of the Worker role (§53.2). It shows the worker their **own** rating with shift-count context. The worker **cannot** see other workers' ratings and **cannot rate** hotels or agencies — only agencies and hotels rate workers (§35/§41).

Route (mobile): Ratings screen.

**Scope:** strictly the worker's **own** rating. Ratings and weightage are governed by **Rating & Trust (§20)**; shift-count context uses the weightage bands from §20.5.

**Actions & Functions**
1. **View own rating and shift-count context.**
2. **No visibility of other workers' ratings.**
3. **No rating actions** — the worker cannot rate hotels or agencies.

### 58.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-WRT1 | Worker can view its **own rating** with **shift-count context** (shifts + weight band) | High |
| FR-WRT2 | Worker **cannot** view **other workers'** ratings | High |
| FR-WRT3 | Worker **cannot rate** hotels or agencies (no rating actions exist for this role) | High |
| FR-WRT4 | Worker scope is **server-enforced** (from the JWT), not client-supplied | High |
| FR-WRT5 | Restricted to `role = worker` | High |

### 58.3 Data Model
**Own Rating (read model)**
| Field | Type | Notes |
|-------|------|-------|
| workerId | UUID | The worker (from token) |
| overallRating | number | Weighted rating (0–5) |
| totalShifts | integer | Shifts informing the rating |
| weightBand | string | Applied band from §20.5 (e.g. "11+ shifts") |
| weightPercent | number | Weight % for the band |

> Read-only. No rating-submission fields and no other-worker identifiers are exposed to this role.

### 58.4 API / Interface Design
Base path: `/api/worker/ratings` — requires `Bearer <jwt>` with `role = worker`; scoped server-side to the worker.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/me` | GET | The worker's own rating with shift-count context |

> No rating-submission endpoints (worker cannot rate) and no endpoints accepting another `workerId`.

**GET /me — Response 200**
```json
{ "workerId": "w1", "overallRating": 4.7, "totalShifts": 63, "weightBand": "11+ shifts", "weightPercent": 100 }
```

### 58.5 UI Layout (brief, mobile)
- **Rating**: a single card showing the worker's overall rating, total shifts, and the applied weight band.
- No other-worker data and **no rating controls** anywhere.

### 58.6 Implementation Phases
- **Phase 1 — Mock:** Own rating and shift context from mock data.
- **Phase 2 — Real Data:** Rating and weighted score from §20; shift count from attendance (§56); read-only and self-only.

### 58.7 Acceptance Criteria
- Worker can view its own rating with shift-count context.
- Worker cannot view other workers' ratings (no such endpoint/param).
- Worker has no ability to rate hotels or agencies.
- Scope is server-enforced; endpoints reject other roles.

### 58.8 Open Questions
- Should the worker see a **breakdown by band** or rating history (per §20.5), or just the current figure?
- Are the **comments** behind ratings visible to the worker, or only the numeric score?
- Should the worker see **who** rated them (hotel/agency), or anonymised?

---

## 59. Worker — Shift History Module

### 59.1 Overview
**Module #6** of the Worker role (§53.2). It gives the worker a record of all their shifts — past and upcoming — with check-in/check-out timestamps and overtime records per shift.

Route (mobile): Shift History screen.

**Scope:** the worker's **own** shifts only. Data reflects the attendance records captured in §56.

**Actions & Functions**
1. **View all past and upcoming assigned shifts.**
2. **View check-in/check-out timestamps** per shift.
3. **View overtime records** per shift.

### 59.2 Shift History Status
| Status | Meaning |
|--------|---------|
| upcoming | Assigned, not yet started |
| completed | Worked and checked out |
| auto_closed | Closed by the system from geofence (missed check-out, §56) |
| missed | Did not attend / no-show |

### 59.3 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-WSH1 | Worker can view **all past and upcoming** assigned shifts | High |
| FR-WSH2 | Worker can view **check-in/check-out timestamps** per shift | High |
| FR-WSH3 | Worker can view **overtime records** per shift | High |
| FR-WSH4 | Shifts are filterable by **date range** and past/upcoming | Medium |
| FR-WSH5 | Worker scope is **server-enforced** (from the JWT), not client-supplied | High |
| FR-WSH6 | Restricted to `role = worker` | High |

### 59.4 Data Model
**ShiftHistoryItem (read model)**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Shift id |
| propertyName | string | Property/site |
| date | date | Shift date |
| shiftStart | string | Scheduled start |
| shiftEnd | string | Scheduled end |
| checkIn | timestamp (nullable) | Actual check-in |
| checkOut | timestamp (nullable) | Actual check-out |
| source | enum | `qr` \| `geofence` (which provided the record) |
| overtimeHours | number | Overtime for the shift |
| status | enum | `upcoming` \| `completed` \| `auto_closed` \| `missed` |

### 59.5 API / Interface Design
Base path: `/api/worker/shifts` — requires `Bearer <jwt>` with `role = worker`; scoped server-side to the worker.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/shifts` | GET | List past & upcoming shifts (filter by `?scope=past\|upcoming`, `?from=`, `?to=`) |
| `/shifts/:id` | GET | Get a shift's detail (check-in/out timestamps, overtime) |

> No `workerId` parameter is accepted — the worker is always taken from the token.

**GET /shifts?scope=past — Response 200**
```json
{
  "items": [
    { "id": "s1", "propertyName": "Sunrise Apts", "date": "2026-07-08", "shiftStart": "08:00", "shiftEnd": "16:00", "checkIn": "08:02", "checkOut": "16:05", "source": "qr",       "overtimeHours": 0,   "status": "completed" },
    { "id": "s2", "propertyName": "Palm Villa",   "date": "2026-07-09", "shiftStart": "14:00", "shiftEnd": "22:00", "checkIn": "14:00", "checkOut": "22:30", "source": "geofence", "overtimeHours": 0.5, "status": "auto_closed" }
  ]
}
```

**GET /shifts/:id — Response 200**
```json
{
  "id": "s2", "propertyName": "Palm Villa", "date": "2026-07-09",
  "shiftStart": "14:00", "shiftEnd": "22:00", "checkIn": "14:00", "checkOut": "22:30",
  "source": "geofence", "overtimeHours": 0.5, "status": "auto_closed"
}
```

### 59.6 UI Layout (brief, mobile)
- **Shift History**: tabs for **Upcoming** and **Past**, with a date-range filter.
- Each shift shows property, date, scheduled times, **check-in/out timestamps**, and **overtime**; a badge marks auto-closed/missed shifts.

### 59.7 Implementation Phases
- **Phase 1 — Mock:** Past/upcoming shifts and timestamps from mock data.
- **Phase 2 — Real Data:** Shifts and attendance from §56 records; overtime from computed hours; read-only and self-only.

### 59.8 Acceptance Criteria
- Worker can view all past and upcoming assigned shifts.
- Worker can view check-in/check-out timestamps per shift.
- Worker can view overtime records per shift.
- Scope is server-enforced; endpoints reject other roles.

### 59.9 Open Questions
- Should shift history link to **earnings** (§57) and **ratings** (§58) for the same shift?
- How far back should **past** history be retained/shown?
- Should **disputed** shifts (§52) be flagged in the history view?
