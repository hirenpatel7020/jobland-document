# Technical Requirements Document (TRD)
## Jobland — Role Module

**Version:** 1.0
**Date:** 2026-07-10
**Author:** sandeep@secinverse.com
**Status:** Draft

### Related Documents
- **[TRD-Jobland-Superadmin-Module.md](./TRD-Jobland-Superadmin-Module.md)** — the Authentication module. It **issues the JWT** at login; the `role` claim in that token is one of the roles defined here. Relationship: *Auth authenticates the user → roles determine what they can access.*
- **Permission matrix** — *maintained in a separate document* (to be linked). This file defines **only the roles**; per-role permissions are set there.
- **[jobland-technology-stack.md](./jobland-technology-stack.md)** — platform technology stack.

---

> ⚠️ **Assumptions to confirm.** The hierarchy and vendor model below are drafted from the role names and are **placeholders for your review**. Items needing confirmation are tagged **[CONFIRM]**.

---

## 1. Overview
This document defines the **user roles** for the Jobland platform — a **hotel operations** domain in which a central authority oversees one or more properties (hotels), each property runs departments, work is carried out by workers, and vendors supply external services/labour.

The platform defines **six** roles. Per-role **permissions are defined in a separate document**; this file lists and describes the roles and their relationships only.

## 2. Roles
| # | Role | Key (JWT `role`) | Scope | One-line summary |
|---|------|------------------|-------|------------------|
| 1 | Superadmin | `superadmin` | Platform | Owns and operates the Jobland platform itself. |
| 2 | Hotel Central Authority | `hotel_central_authority` | Hotel group | Oversees all properties belonging to a hotel group/brand. |
| 3 | Property Manager | `property_manager` | Single property | Runs the day-to-day operations of one property (hotel). |
| 4 | Department Manager | `department_manager` | Department | Manages a department within a property (e.g. Housekeeping, F&B). |
| 5 | Vendor Manager | `vendor_manager` | Vendor org | Manages an external vendor organisation and its workforce. |
| 6 | Worker | `worker` | Individual | Executes assigned tasks / work orders. |

## 3. Role Definitions

**1. Superadmin** — Highest-privilege user of the platform. Provisions and manages Hotel Central Authority accounts, oversees all tenants, and configures platform-wide settings. Not tied to any single hotel group. *(See the Superadmin Authentication TRD for login.)*

**2. Hotel Central Authority** — The top of a **hotel group / brand**. Owns multiple properties, provisions Property Managers, and sees consolidated reporting across all its properties. **[CONFIRM]** whether one Central Authority = one brand with many hotels.

**3. Property Manager** — Runs a **single property**. Provisions Department Managers, oversees all departments in that property, and manages property-level staff and vendors. **[CONFIRM]** whether a Property Manager can be assigned to more than one property.

**4. Department Manager** — Manages **one department** inside a property (e.g. Housekeeping, Front Desk, Maintenance, F&B). Assigns and reviews tasks for Workers in that department.

**5. Vendor Manager** — Manages an **external vendor organisation** on the platform. Onboards/manages that vendor's own Workers and fulfils work assigned to the vendor. **[CONFIRM]** the exact vendor model: is a Vendor an external company whose Vendor Manager manages that company's own workers, and are those workers the same `worker` role or a separate vendor-worker type?

**6. Worker** — An individual who **performs the work** (tasks / work orders). May be a hotel/department employee or a vendor's employee. **[CONFIRM]** whether Workers belong to a department, a vendor, or either.

## 4. Hierarchy (draft — **[CONFIRM]**)
```
Superadmin  (platform)
   └── Hotel Central Authority  (hotel group / brand)
          └── Property Manager  (one property)
                 ├── Department Manager  (one department)
                 │        └── Worker  (department staff)
                 └── Vendor Manager  (vendor org)
                          └── Worker  (vendor staff)
```
> This tree is a best-guess. Please describe the real reporting lines and I'll redraw it.

## 5. Definitions & Acronyms
| Term | Definition |
|------|------------|
| RBAC | Role-Based Access Control — permissions granted by role |
| JWT | JSON Web Token — signed token carrying the `role` claim |
| Scope | The data boundary a role can act within (platform / group / property / department / vendor / self) |
| Tenant | A hotel group and all data beneath it |
| Property | A single hotel/location under a hotel group |
| Department | An operational unit within a property |
| Vendor | An external organisation supplying services/labour |
| Worker | An individual who executes tasks / work orders |

## 6. Open Questions / To Confirm
1. **Hierarchy** — real reporting lines between the six roles (§4 is a guess).
2. **Vendor model** — is a Vendor an external company; does the Vendor Manager manage that company's own Workers; are those Workers the `worker` role or a separate type? (§3 #5)
3. **Worker ownership** — do Workers belong to a department, a vendor, or both? (§3 #6)
4. **Multi-assignment** — can a Property Manager run multiple properties? Can a Worker belong to multiple departments/vendors?
5. **Central Authority = brand?** — one Hotel Central Authority per hotel group/brand?
