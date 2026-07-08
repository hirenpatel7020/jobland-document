# Technical Requirements Document (TRD)
## Jobland — Role & Permission Module

**Version:** 1.0
**Date:** 2026-07-07
**Author:** sandeep@secinverse.com
**Status:** Draft

### Related Documents
- **[TRD-Jobland-Superadmin-Module.md](./TRD-Jobland-Superadmin-Module.md)** — the Authentication module. It **issues the JWT** at login; the `role` claim in that token is one of the roles defined here. This Role module's authorization middleware **reads that JWT** to enforce permissions. Relationship: *Auth authenticates the user → Role authorizes the request.*

---

## 1. Overview
This module defines the **role-based access control (RBAC)** system for the Jobland platform. It establishes the set of user roles, the permissions attached to each role, how roles are assigned to users, and how the system enforces role-based authorization across APIs.

Jobland supports four roles:

| Role | Description |
|------|-------------|
| **Superadmin** | Full platform control; manages all users, vendors, properties, and system settings. |
| **Vendor** | Business/service provider account; manages own listings, services, and profile. |
| **Property Manager** | Manages properties, tenants/units, and property-related operations. |
| **User** | Standard end user; browses, applies, books, or requests services. |

## 2. Scope
### In Scope
- Definition of the four roles and their permission sets.
- Role assignment to users (single primary role per user).
- Authorization middleware to enforce role/permission on protected routes.
- Seeding of default roles and permissions.
- Role-permission data model.

### Out of Scope
- Authentication/login (covered in the Superadmin Authentication TRD).
- Multi-tenant org hierarchies.
- Custom user-defined roles (only the four predefined roles in v1).
- Fine-grained record-level (row-level) sharing rules.

## 3. Definitions & Acronyms
| Term | Definition |
|------|------------|
| RBAC | Role-Based Access Control |
| Role | Named collection of permissions assigned to a user |
| Permission | Discrete right to perform an action on a resource (e.g. `property:create`) |
| Resource | An entity the action applies to (user, property, vendor, service) |
| Superadmin | Highest-privilege role with full access |

## 4. Roles & Responsibilities
| Role | Key Responsibilities |
|------|----------------------|
| Superadmin | Manage all users & roles, approve/suspend vendors & managers, manage properties, view all data, configure platform settings. |
| Vendor | Create/manage own services or listings, manage own profile, respond to user requests/bookings. |
| Property Manager | Create/manage properties & units, manage tenants/occupancy, handle property requests, view own property analytics. |
| User | Browse listings/properties, apply/book/request services, manage own profile and requests. |

## 5. Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1 | System supports four roles: Superadmin, Vendor, Property Manager, User | High |
| FR-2 | Each user is assigned exactly one primary role | High |
| FR-3 | Each role maps to a defined set of permissions | High |
| FR-4 | Superadmin has full (all) permissions | High |
| FR-5 | Superadmin can assign or change a user's role | High |
| FR-6 | Authorization middleware enforces role/permission per route | High |
| FR-7 | Requests lacking the required role/permission return 403 Forbidden | High |
| FR-8 | Default roles & permissions are seeded on setup | High |
| FR-9 | Role of a user is included in the JWT for enforcement | High |
| FR-10 | List available roles via API | Medium |
| FR-11 | Vendor & Property Manager accounts may require Superadmin approval | Medium |

## 6. Non-Functional Requirements
| ID | Requirement | Category |
|----|-------------|----------|
| NFR-1 | Authorization check adds < 10 ms overhead per request | Performance |
| NFR-2 | Permission checks are deny-by-default | Security |
| NFR-3 | Role/permission definitions are centralized and version-controlled | Maintainability |
| NFR-4 | Role changes take effect on the user's next token issuance | Consistency |

## 7. Permission Matrix
Legend: ✔ = allowed, ✘ = not allowed. (Own = only records owned by the user.)

| Permission / Action | Superadmin | Vendor | Property Manager | User |
|---------------------|:----------:|:------:|:----------------:|:----:|
| Manage users & roles | ✔ | ✘ | ✘ | ✘ |
| Manage platform settings | ✔ | ✘ | ✘ | ✘ |
| Approve/suspend accounts | ✔ | ✘ | ✘ | ✘ |
| View all data | ✔ | ✘ | ✘ | ✘ |
| Create/manage properties | ✔ | ✘ | ✔ (Own) | ✘ |
| Manage units/tenants | ✔ | ✘ | ✔ (Own) | ✘ |
| Create/manage vendor services | ✔ | ✔ (Own) | ✘ | ✘ |
| Respond to bookings/requests | ✔ | ✔ (Own) | ✔ (Own) | ✘ |
| Browse listings/properties | ✔ | ✔ | ✔ | ✔ |
| Book/apply/request service | ✔ | ✘ | ✘ | ✔ |
| Manage own profile | ✔ | ✔ | ✔ | ✔ |

## 8. Data Model
### Role
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| name | string | Unique: `superadmin`, `vendor`, `property_manager`, `user` |
| displayName | string | e.g. "Property Manager" |
| description | string | |
| isSystem | boolean | True for the four built-in roles |
| createdAt | timestamp | |

### Permission
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| key | string | e.g. `property:create`, `user:manage` |
| description | string | |

### RolePermission (join)
| Field | Type | Notes |
|-------|------|-------|
| roleId | UUID | FK -> Role.id |
| permissionId | UUID | FK -> Permission.id |

### User (relevant fields)
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| email | string | Unique |
| roleId | UUID | FK -> Role.id (single primary role) |

## 9. Role Enum (implementation reference)
```json
{
  "SUPERADMIN": "superadmin",
  "VENDOR": "vendor",
  "PROPERTY_MANAGER": "property_manager",
  "USER": "user"
}
```

## 10. API / Interface Design
Base path: `/api/roles`

| Endpoint | Method | Description | Auth |
|----------|--------|-------------|------|
| `/roles` | GET | List all roles | Superadmin |
| `/roles/:id` | GET | Get a role and its permissions | Superadmin |
| `/users/:id/role` | PUT | Assign/change a user's role | Superadmin |
| `/me/permissions` | GET | Get current user's role & permissions | Any authenticated |

### 10.1 PUT /users/:id/role
**Request**
```json
{ "role": "property_manager" }
```
**Response 200**
```json
{ "success": true, "userId": "...", "role": "property_manager" }
```
**Response 403**
```json
{ "success": false, "message": "Forbidden: insufficient permissions" }
```

### 10.2 Authorization usage (pseudo)
```
router.post('/properties',
  authenticate,                 // verifies JWT
  authorize('property_manager'),// or authorize permission 'property:create'
  createPropertyHandler
)
```

## 11. Dependencies
- Authentication module (JWT carries `role`).
- Authorization middleware (`authorize(role | permission)`).
- Seed/migration to create the four roles and their permissions.

## 12. Assumptions & Constraints
- v1 uses four fixed, system-defined roles (no custom roles).
- A user holds exactly one role at a time.
- Role is embedded in the JWT; changing a role requires a new token to take effect.
- "Own" scope enforcement (record ownership) is handled in service logic.

## 13. Acceptance Criteria
- All four roles are seeded with correct permissions.
- Superadmin can change a user's role via API.
- A route protected for one role returns 403 for other roles.
- JWT contains the user's role and middleware enforces it.
- Deny-by-default: unknown/missing role cannot access protected routes.

## 14. Open Questions
- Should Vendor and Property Manager sign-ups require Superadmin approval before activation?
- Do we need multiple roles per user in the future (e.g., a user who is also a vendor)?
- Should permissions be enforced at the granular `permission` level or coarse `role` level in v1?
- Any read-only "support/staff" role needed beyond these four?
