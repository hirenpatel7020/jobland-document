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

> **Connection to Role Module:** The `role` value embedded in the issued JWT (see §8.4) is one of the four roles defined in the [Role & Permission Module](./TRD-Jobland-Role-Module.md) — `superadmin`, `vendor`, `property_manager`, `user`. Login is shared across all roles; only the role claim differs.

Development is delivered in phases:
- **Phase 1 — Mock Setup:** Auth endpoints backed by in-memory / hardcoded mock data (no DB) to unblock frontend integration and validate the JWT flow.
- **Phase 2 — Real Login:** Email & password validated against the database with hashed passwords.
- **Phase 3 — Forgot Password:** Reset-token generation, delivery, and password reset.

**Post-login behavior:** When login succeeds and the JWT's `role = superadmin`, the app redirects the user to the **Superadmin Dashboard** (see §14). The dashboard shows 5 summary count cards and a "Todays Work" table.

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
After a successful login where `role = superadmin`, the app redirects to the **Superadmin Dashboard** at `/superadmin/dashboard`. The dashboard has two sections:
1. **5 summary count cards.**
2. **"Todays Work"** — a table of today's work records.

**Flow:** Login (§8.1) → JWT issued with `role = superadmin` → redirect to dashboard → load summary + today's work.

### 14.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-D1 | On login with `role = superadmin`, redirect to the Superadmin dashboard | High |
| FR-D2 | Dashboard accessible only to authenticated `superadmin`; others get 403/redirect | High |
| FR-D3 | Show 5 count cards, each with a title and a numeric count | High |
| FR-D4 | Cards: **Total Activity, Total Property, Total Client, Total Vendors, Total Work User** | High |
| FR-D5 | Card counts are fetched from the backend (not hardcoded) | High |
| FR-D6 | Show a **"Todays Work"** section with a data table | High |
| FR-D7 | Table has 4 columns: **Property Name, Worker Name, Shift, Status** | High |
| FR-D8 | "Todays Work" lists only records for the current date | High |
| FR-D9 | Show empty, loading, and error states | Medium |

### 14.3 UI Layout
```
+-----------------------------------------------------------------------+
|  Superadmin Dashboard                                    [ Superadmin ]|
+-----------------------------------------------------------------------+
|  +----------+ +----------+ +----------+ +----------+ +-----------+     |
|  |  Total   | |  Total   | |  Total   | |  Total   | |   Total   |     |
|  | Activity | | Property | |  Client  | | Vendors  | | Work User |     |
|  |  1,240   | |    86    | |   512    | |    47    | |    130    |     |
|  +----------+ +----------+ +----------+ +----------+ +-----------+     |
|                                                                       |
|  Todays Work                                                          |
|  +-----------------------------------------------------------------+  |
|  | Property Name | Worker Name | Shift    | Status                 |  |
|  +---------------+-------------+----------+------------------------+  |
|  | Sunrise Apts  | John Doe    | Morning  | Completed              |  |
|  | Palm Villa    | Jane Smith  | Evening  | In Progress            |  |
|  | Lake View     | Ravi Kumar  | Night    | Pending                |  |
|  +-----------------------------------------------------------------+  |
+-----------------------------------------------------------------------+
```

### 14.4 Count Cards
| Card Title | Meaning (data source) |
|------------|-----------------------|
| Total Activity | Count of all activity/work records |
| Total Property | Count of properties |
| Total Client | Count of clients |
| Total Vendors | Count of vendor accounts |
| Total Work User | Count of workers / work users |

### 14.5 Todays Work Table
| Column | Description | Example |
|--------|-------------|---------|
| Property Name | Property the work belongs to | "Sunrise Apts" |
| Worker Name | Assigned worker | "John Doe" |
| Shift | Work shift | Morning / Evening / Night |
| Status | Current status | Pending / In Progress / Completed |

### 14.6 Data Model (read models)
**Dashboard Summary (computed)**
| Field | Type | Notes |
|-------|------|-------|
| totalActivity | integer | Count of activities |
| totalProperty | integer | Count of properties |
| totalClient | integer | Count of clients |
| totalVendors | integer | Count of vendors |
| totalWorkUser | integer | Count of work users |

**Todays Work item**
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Work record id |
| propertyName | string | Property name |
| workerName | string | Worker name |
| shift | enum | `morning` \| `evening` \| `night` |
| status | enum | `pending` \| `in_progress` \| `completed` |
| workDate | date | Filtered to today |

### 14.7 API / Interface Design
Base path: `/api/superadmin/dashboard` — requires `Authorization: Bearer <jwt>` with `role = superadmin`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/summary` | GET | Returns the 5 counts for the cards |
| `/todays-work` | GET | Returns today's work list for the table |

**GET /summary — Response 200**
```json
{
  "totalActivity": 1240,
  "totalProperty": 86,
  "totalClient": 512,
  "totalVendors": 47,
  "totalWorkUser": 130
}
```

**GET /todays-work — Response 200**
```json
{
  "date": "2026-07-07",
  "items": [
    { "id": "w1", "propertyName": "Sunrise Apts", "workerName": "John Doe",  "shift": "morning", "status": "completed" },
    { "id": "w2", "propertyName": "Palm Villa",   "workerName": "Jane Smith", "shift": "evening", "status": "in_progress" },
    { "id": "w3", "propertyName": "Lake View",    "workerName": "Ravi Kumar", "shift": "night",   "status": "pending" }
  ]
}
```

**Empty / Forbidden**
```json
{ "date": "2026-07-07", "items": [] }
```
```json
{ "success": false, "message": "Forbidden: superadmin only" }
```

### 14.8 Status Values
| Status | Label | Suggested Color |
|--------|-------|-----------------|
| pending | Pending | Amber |
| in_progress | In Progress | Blue |
| completed | Completed | Green |

### 14.9 Implementation Phases
- **Phase 1 — Mock:** Return hardcoded counts and a static "Todays Work" list; build the full UI (cards + table) against the mock API.
- **Phase 2 — Real Data:** Replace with real aggregation counts and today's work filtered by `workDate = today`.

### 14.10 Acceptance Criteria
- Login as `superadmin` redirects to the dashboard.
- Non-superadmin cannot access the dashboard (403/redirect).
- All 5 cards render with correct titles and live counts.
- "Todays Work" table renders 4 columns and today's records only.
- Empty, loading, and error states are handled.

### 14.11 Open Questions
- Exact definition of "Total Activity" — all work records, or an activity log?
- Is "Total Client" a separate entity from the `user` role?
- Is "Total Work User" the same as "Worker"?
- Should the table support pagination/sorting in v1?

---

## 15. Client Management (Client Page)

### 15.1 Overview
A **Client** page accessible to the Superadmin that lists all clients as **cards**. Each card shows key client details. The page also has an **"Add Client"** action that opens a form to create a new client.

Route: `/superadmin/clients`

### 15.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-C1 | Client page displays all clients as cards | High |
| FR-C2 | Each card shows: Business Name, VAT No, Status, Property Logo, Client Name, Email, Phone No, Address | High |
| FR-C3 | Page loads 5–6 mock clients in Phase 1 | High |
| FR-C4 | An **"Add Client"** button/option is visible on the page | High |
| FR-C5 | Clicking "Add Client" opens a form (modal or page) | High |
| FR-C6 | Form fields: Business Name, VAT Number, First Name, Last Name, Email Address, Mobile No (+ country code), Full Address | High |
| FR-C7 | Country code is a selectable dropdown for the mobile number | High |
| FR-C8 | On submit, the new client is added to the list | High |
| FR-C9 | Required-field and email/phone validation on the form | Medium |
| FR-C10 | Accessible only to `superadmin` | High |

### 15.3 Client Card Layout
```
+---------------------------------------------------+
|  [Logo]   Business Name              [ Status ]   |
|           Client Name                             |
|---------------------------------------------------|
|  VAT No   : GB123456789                           |
|  Email    : contact@acme.com                      |
|  Phone    : +44 7911 123456                        |
|  Address  : 12 High St, London, UK                |
+---------------------------------------------------+
```

**Card fields**
| Field | Description |
|-------|-------------|
| Property Logo | Client / business logo image |
| Business Name | Company/business name |
| Client Name | Contact person's full name |
| VAT No | VAT registration number |
| Status | Active / Inactive |
| Email id | Contact email |
| Phone No | Contact phone (with country code) |
| Address | Business address |

### 15.4 Mock Data (Phase 1)
```json
[
  {
    "id": "c1", "logo": "/mock/logos/acme.png", "businessName": "Acme Facilities Ltd",
    "clientName": "John Carter", "vatNo": "GB123456789", "status": "active",
    "email": "john@acmefacilities.com", "phone": "+44 7911 123456",
    "address": "12 High Street, London, UK"
  },
  {
    "id": "c2", "logo": "/mock/logos/brightclean.png", "businessName": "BrightClean Services",
    "clientName": "Sarah Nolan", "vatNo": "GB987654321", "status": "active",
    "email": "sarah@brightclean.com", "phone": "+44 7822 654321",
    "address": "48 Queen Road, Manchester, UK"
  },
  {
    "id": "c3", "logo": "/mock/logos/greenscape.png", "businessName": "GreenScape Property",
    "clientName": "David Kim", "vatNo": "GB456789123", "status": "inactive",
    "email": "david@greenscape.com", "phone": "+44 7700 900123",
    "address": "9 Park Lane, Birmingham, UK"
  },
  {
    "id": "c4", "logo": "/mock/logos/urbannest.png", "businessName": "UrbanNest Living",
    "clientName": "Priya Sharma", "vatNo": "GB321654987", "status": "active",
    "email": "priya@urbannest.com", "phone": "+44 7500 111222",
    "address": "22 River View, Leeds, UK"
  },
  {
    "id": "c5", "logo": "/mock/logos/coastal.png", "businessName": "Coastal Estates",
    "clientName": "Michael Reed", "vatNo": "GB147258369", "status": "active",
    "email": "michael@coastalestates.com", "phone": "+44 7333 444555",
    "address": "5 Marine Drive, Brighton, UK"
  },
  {
    "id": "c6", "logo": "/mock/logos/summit.png", "businessName": "Summit Property Care",
    "clientName": "Emma Watson", "vatNo": "GB258369147", "status": "inactive",
    "email": "emma@summitcare.com", "phone": "+44 7444 555666",
    "address": "31 Hilltop Ave, Bristol, UK"
  }
]
```

### 15.5 Add Client Form
Opened via the **"Add Client"** button.

| Field | Type | Required | Notes |
|-------|------|:--------:|-------|
| Business Name | text | Yes | |
| VAT Number | text | Yes | |
| First Name | text | Yes | Combined into Client Name |
| Last Name | text | Yes | Combined into Client Name |
| Email Address | email | Yes | Valid email format |
| Country Code | select | Yes | Dropdown (e.g. +44, +1, +91) |
| Mobile No | tel | Yes | Digits only; paired with country code |
| Full Address | textarea | Yes | |

**Form layout**
```
+------------------- Add Client -------------------+
| Business Name  [__________________________]       |
| VAT Number     [__________________________]       |
| First Name     [_______________]  Last Name [___] |
| Email Address  [__________________________]       |
| Mobile No      [+44 v] [___________________]      |
| Full Address   [__________________________]       |
|                [__________________________]       |
|                                                   |
|              [ Cancel ]     [ Save Client ]       |
+---------------------------------------------------+
```

### 15.6 Data Model — Client
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| businessName | string | Required |
| vatNumber | string | Required |
| firstName | string | Required |
| lastName | string | Required |
| clientName | string | Derived: `firstName + " " + lastName` |
| email | string | Required, unique |
| countryCode | string | e.g. `+44` |
| mobileNo | string | Digits |
| phone | string | Derived: `countryCode + " " + mobileNo` |
| fullAddress | string | Required |
| logo | string | Logo URL/path (optional in v1) |
| status | enum | `active` \| `inactive` (default `active`) |
| createdAt | timestamp | |

### 15.7 API / Interface Design
Base path: `/api/superadmin/clients` — requires `Bearer <jwt>` with `role = superadmin`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/clients` | GET | List all clients (for cards) |
| `/clients` | POST | Create a new client (Add Client form) |
| `/clients/:id` | GET | Get a single client (optional) |

**POST /clients — Request**
```json
{
  "businessName": "New Client Ltd",
  "vatNumber": "GB111222333",
  "firstName": "Alex",
  "lastName": "Turner",
  "email": "alex@newclient.com",
  "countryCode": "+44",
  "mobileNo": "7911223344",
  "fullAddress": "10 New Road, London, UK"
}
```
**POST /clients — Response 201**
```json
{
  "success": true,
  "client": {
    "id": "c7", "businessName": "New Client Ltd", "vatNo": "GB111222333",
    "clientName": "Alex Turner", "email": "alex@newclient.com",
    "phone": "+44 7911223344", "address": "10 New Road, London, UK",
    "status": "active"
  }
}
```

### 15.8 Implementation Phases
- **Phase 1 — Mock:** Render 5–6 mock client cards; "Add Client" form appends to the local/mock list.
- **Phase 2 — Real Data:** `GET /clients` from DB; `POST /clients` persists and refreshes the list.

### 15.9 Acceptance Criteria
- Client page shows all clients as cards with the specified details.
- 5–6 mock clients appear in Phase 1.
- "Add Client" opens a form with all listed fields, including a country-code select.
- Submitting a valid form adds a new client card.
- Invalid/empty required fields are blocked with validation messages.

### 15.10 Open Questions
- Is "Property Logo" the client's business logo, or a logo of a property? (Assumed business/client logo.)
- Should status be editable from the card (activate/deactivate)?
- Is logo upload required in v1, or a placeholder for now?
- Which country codes should the dropdown include?

---

## 16. Property Management (Property Page)

### 16.1 Overview
A **Property** page accessible to the Superadmin that lists all properties as **cards**. Each card shows key property details including its geo-location (latitude/longitude).

Route: `/superadmin/properties`

### 16.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-P1 | Property page displays all properties as cards | High |
| FR-P2 | Each card shows: Property Image, Property Name, Address, VAT No, Client Name, Role, Email, Mobile No, Latitude, Longitude | High |
| FR-P3 | Page loads mock properties in Phase 1 | High |
| FR-P4 | Property data is fetched from the backend (not hardcoded) in Phase 2 | High |
| FR-P5 | Accessible only to `superadmin` | High |
| FR-P6 | Show empty, loading, and error states | Medium |

### 16.3 Property Card Layout
```
+---------------------------------------------------+
|  [ Property Image ]                                |
|  Property Name                                     |
|---------------------------------------------------|
|  Address    : 12 High Street, London, UK          |
|  VAT No     : GB123456789                          |
|  Client     : John Carter   (Role: Client)        |
|  Email      : john@acmefacilities.com             |
|  Mobile     : +44 7911 123456                      |
|  Lat / Lng  : 51.5074, -0.1278                     |
+---------------------------------------------------+
```

**Card fields**
| Field | Description |
|-------|-------------|
| Property Image | Photo/image of the property |
| Property Name | Name/title of the property |
| Address | Property address |
| VAT No | VAT number (of the associated client) |
| Client Name | Client the property belongs to |
| Role | Role of the associated user (e.g. `client` / `property_manager`) |
| Email | Contact email |
| Mobile No | Contact mobile (with country code) |
| Latitude | Geo latitude |
| Longitude | Geo longitude |

### 16.4 Mock Data (Phase 1)
```json
[
  {
    "id": "p1", "image": "/mock/properties/sunrise.jpg", "propertyName": "Sunrise Apartments",
    "address": "12 High Street, London, UK", "vatNo": "GB123456789",
    "clientName": "John Carter", "role": "client",
    "email": "john@acmefacilities.com", "mobileNo": "+44 7911 123456",
    "latitude": 51.5074, "longitude": -0.1278
  },
  {
    "id": "p2", "image": "/mock/properties/palm.jpg", "propertyName": "Palm Villa",
    "address": "48 Queen Road, Manchester, UK", "vatNo": "GB987654321",
    "clientName": "Sarah Nolan", "role": "property_manager",
    "email": "sarah@brightclean.com", "mobileNo": "+44 7822 654321",
    "latitude": 53.4808, "longitude": -2.2426
  },
  {
    "id": "p3", "image": "/mock/properties/lakeview.jpg", "propertyName": "Lake View Residency",
    "address": "9 Park Lane, Birmingham, UK", "vatNo": "GB456789123",
    "clientName": "David Kim", "role": "client",
    "email": "david@greenscape.com", "mobileNo": "+44 7700 900123",
    "latitude": 52.4862, "longitude": -1.8904
  },
  {
    "id": "p4", "image": "/mock/properties/urbannest.jpg", "propertyName": "UrbanNest Towers",
    "address": "22 River View, Leeds, UK", "vatNo": "GB321654987",
    "clientName": "Priya Sharma", "role": "property_manager",
    "email": "priya@urbannest.com", "mobileNo": "+44 7500 111222",
    "latitude": 53.8008, "longitude": -1.5491
  },
  {
    "id": "p5", "image": "/mock/properties/coastal.jpg", "propertyName": "Coastal Heights",
    "address": "5 Marine Drive, Brighton, UK", "vatNo": "GB147258369",
    "clientName": "Michael Reed", "role": "client",
    "email": "michael@coastalestates.com", "mobileNo": "+44 7333 444555",
    "latitude": 50.8225, "longitude": -0.1372
  },
  {
    "id": "p6", "image": "/mock/properties/summit.jpg", "propertyName": "Summit Court",
    "address": "31 Hilltop Ave, Bristol, UK", "vatNo": "GB258369147",
    "clientName": "Emma Watson", "role": "client",
    "email": "emma@summitcare.com", "mobileNo": "+44 7444 555666",
    "latitude": 51.4545, "longitude": -2.5879
  }
]
```

### 16.5 Data Model — Property
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| image | string | Property image URL/path |
| propertyName | string | Required |
| address | string | Required |
| vatNo | string | VAT number of associated client |
| clientId | UUID | FK -> Client.id |
| clientName | string | Denormalized client name |
| role | string | Role of associated user (`client` / `property_manager`) |
| email | string | Contact email |
| mobileNo | string | Contact mobile (with country code) |
| latitude | number | Geo latitude |
| longitude | number | Geo longitude |
| createdAt | timestamp | |

### 16.6 API / Interface Design
Base path: `/api/superadmin/properties` — requires `Bearer <jwt>` with `role = superadmin`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/properties` | GET | List all properties (for cards) |
| `/properties/:id` | GET | Get a single property (optional) |

**GET /properties — Response 200**
```json
{
  "items": [
    {
      "id": "p1", "image": "/mock/properties/sunrise.jpg", "propertyName": "Sunrise Apartments",
      "address": "12 High Street, London, UK", "vatNo": "GB123456789",
      "clientName": "John Carter", "role": "client",
      "email": "john@acmefacilities.com", "mobileNo": "+44 7911 123456",
      "latitude": 51.5074, "longitude": -0.1278
    }
  ]
}
```

### 16.7 Implementation Phases
- **Phase 1 — Mock:** Render mock property cards from the static list.
- **Phase 2 — Real Data:** `GET /properties` from DB, joined with client info.

### 16.8 Acceptance Criteria
- Property page shows all properties as cards with the specified details.
- Mock properties appear in Phase 1.
- Latitude and longitude are displayed on each card.
- Empty, loading, and error states are handled.

### 16.9 Open Questions
- Should the card show a map preview using the lat/lng, or just the coordinates?
- Is "Role" the associated user's role (client/property_manager) or a property-specific role?
- Should there be an "Add Property" form (like Add Client) in v1?

---

## 17. Work User Management (Work User Page)

### 17.1 Overview
A **Work User** page accessible to the Superadmin that lists all work users (workers) as **cards**. Each card shows key details plus a **"View All Documents"** button. The page has an **"Add Work User"** action that opens a form; on submit, the app redirects back to the Work User list page.

Route: `/superadmin/work-users`

### 17.2 Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-W1 | Work User page displays all work users as cards | High |
| FR-W2 | Each card shows: Time, User Name, Email, Phone No, Gender, DOB, Education | High |
| FR-W3 | Each card has a **"View All Documents"** button | High |
| FR-W4 | Page loads mock work users in Phase 1 | High |
| FR-W5 | An **"Add Work User"** button/option is visible | High |
| FR-W6 | Clicking "Add Work User" opens a form (modal or page) | High |
| FR-W7 | Form fields: First Name, Last Name, Email, Mobile No, Gender (select), DOB, Education, Shift (select), Upload Document | High |
| FR-W8 | On submit, save the work user and **redirect to the Work User list page** | High |
| FR-W9 | Required-field, email, and file-type validation on the form | Medium |
| FR-W10 | Accessible only to `superadmin` | High |

### 17.3 Work User Card Layout
```
+---------------------------------------------------+
|  User Name                          Time: 09:15 AM |
|---------------------------------------------------|
|  Email      : john.doe@work.com                   |
|  Phone      : +44 7911 123456                      |
|  Gender     : Male                                 |
|  DOB        : 1994-05-12                           |
|  Education  : B.Sc Facilities Management           |
|                                                   |
|              [ View All Documents ]                |
+---------------------------------------------------+
```

**Card fields**
| Field | Description |
|-------|-------------|
| Time | Check-in / created / shift time |
| User Name | Full name of the work user |
| Email | Contact email |
| Phone No | Contact phone (with country code) |
| Gender | Male / Female / Other |
| DOB | Date of birth |
| Education | Highest education / qualification |
| View All Documents (button) | Opens the user's uploaded documents |

### 17.4 Mock Data (Phase 1)
```json
[
  {
    "id": "wu1", "time": "09:15 AM", "userName": "John Doe",
    "email": "john.doe@work.com", "phoneNo": "+44 7911 123456",
    "gender": "male", "dob": "1994-05-12", "education": "B.Sc Facilities Management",
    "shift": "morning",
    "documents": ["/mock/docs/john-id.pdf", "/mock/docs/john-cert.pdf"]
  },
  {
    "id": "wu2", "time": "11:40 AM", "userName": "Jane Smith",
    "email": "jane.smith@work.com", "phoneNo": "+44 7822 654321",
    "gender": "female", "dob": "1996-09-03", "education": "Diploma in Housekeeping",
    "shift": "evening",
    "documents": ["/mock/docs/jane-id.pdf"]
  },
  {
    "id": "wu3", "time": "08:00 AM", "userName": "Ravi Kumar",
    "email": "ravi.kumar@work.com", "phoneNo": "+44 7700 900123",
    "gender": "male", "dob": "1992-01-27", "education": "ITI Electrician",
    "shift": "night",
    "documents": ["/mock/docs/ravi-id.pdf", "/mock/docs/ravi-license.pdf"]
  },
  {
    "id": "wu4", "time": "02:30 PM", "userName": "Emma Watson",
    "email": "emma.w@work.com", "phoneNo": "+44 7500 111222",
    "gender": "female", "dob": "1998-11-19", "education": "B.A Hospitality",
    "shift": "morning",
    "documents": []
  },
  {
    "id": "wu5", "time": "06:45 PM", "userName": "Michael Reed",
    "email": "michael.reed@work.com", "phoneNo": "+44 7333 444555",
    "gender": "male", "dob": "1990-07-08", "education": "Certified Plumber",
    "shift": "evening",
    "documents": ["/mock/docs/michael-cert.pdf"]
  }
]
```

### 17.5 Add Work User Form
Opened via the **"Add Work User"** button.

| Field | Type | Required | Notes |
|-------|------|:--------:|-------|
| First Name | text | Yes | Combined into User Name |
| Last Name | text | Yes | Combined into User Name |
| Email | email | Yes | Valid email format |
| Mobile No | tel | Yes | (country code optional as in Client form) |
| Gender | select | Yes | Male / Female / Other |
| DOB | date | Yes | Date of birth |
| Education | text | Yes | Qualification |
| Shift | select | Yes | Morning / Evening / Night |
| Upload Document | file | Yes | One or more files (PDF/JPG/PNG) |

**Form layout**
```
+----------------- Add Work User ------------------+
| First Name   [_______________] Last Name [_____] |
| Email        [__________________________]        |
| Mobile No    [__________________________]        |
| Gender       [ Male v ]                          |
| DOB          [ 1994-05-12 ]                       |
| Education    [__________________________]        |
| Shift        [ Morning v ]                        |
| Documents    [ Upload File(s)... ]                |
|                                                   |
|            [ Cancel ]     [ Save & Continue ]     |
+---------------------------------------------------+
```
On successful submit -> **redirect to `/superadmin/work-users`** (list view).

### 17.6 Data Model — Work User
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| firstName | string | Required |
| lastName | string | Required |
| userName | string | Derived: `firstName + " " + lastName` |
| email | string | Required, unique |
| mobileNo | string | Contact mobile |
| gender | enum | `male` \| `female` \| `other` |
| dob | date | Date of birth |
| education | string | Qualification |
| shift | enum | `morning` \| `evening` \| `night` |
| time | string | Check-in / created time (display) |
| documents | string[] | Uploaded document URLs/paths |
| createdAt | timestamp | |

### 17.7 API / Interface Design
Base path: `/api/superadmin/work-users` — requires `Bearer <jwt>` with `role = superadmin`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/work-users` | GET | List all work users (for cards) |
| `/work-users` | POST | Create a work user (Add form; multipart for documents) |
| `/work-users/:id/documents` | GET | List a user's documents ("View All Documents") |

**POST /work-users — Request** (`multipart/form-data`)
```
firstName: Alex
lastName: Turner
email: alex.turner@work.com
mobileNo: +44 7911223344
gender: male
dob: 1995-03-15
education: B.Tech Mechanical
shift: morning
documents: <file1>, <file2>
```
**POST /work-users — Response 201**
```json
{
  "success": true,
  "workUser": {
    "id": "wu6", "userName": "Alex Turner", "email": "alex.turner@work.com",
    "phoneNo": "+44 7911223344", "gender": "male", "dob": "1995-03-15",
    "education": "B.Tech Mechanical", "shift": "morning",
    "documents": ["/uploads/wu6/id.pdf"]
  },
  "redirect": "/superadmin/work-users"
}
```

**GET /work-users/:id/documents — Response 200**
```json
{ "userId": "wu1", "documents": ["/mock/docs/john-id.pdf", "/mock/docs/john-cert.pdf"] }
```

### 17.8 Implementation Phases
- **Phase 1 — Mock:** Render mock work-user cards; "Add Work User" form appends to the local/mock list and redirects to the list view; "View All Documents" shows mock document links.
- **Phase 2 — Real Data:** `GET /work-users` from DB; `POST /work-users` persists with document upload; documents served from storage.

### 17.9 Acceptance Criteria
- Work User page shows all work users as cards with the specified details.
- Mock work users appear in Phase 1.
- Each card has a working "View All Documents" button.
- "Add Work User" opens a form with all listed fields, including Gender/Shift selects and document upload.
- Submitting a valid form saves the user and redirects to the list view.
- Invalid/empty required fields are blocked with validation messages.

### 17.10 Open Questions
- What exactly does "Time" represent — created time, shift start time, or check-in time?
- Should document upload allow multiple files, and which formats/size limits?
- Should Mobile No use the country-code select like the Client form?
- Any approval step before a new work user becomes active?
