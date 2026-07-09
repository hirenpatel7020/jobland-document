# Jobland — Technology Stack

> Status: Draft • Last updated: 2026-07-09
>
> This document lists every module of the Jobland platform along with its
> **Technology**, **Database**, and **Server**.
>
> Related documents:
> - [TRD — Superadmin Authentication Module](./TRD-Jobland-Superadmin-Module.md)
> - [TRD — Role & Permission Module](./TRD-Jobland-Role-Module.md)
> - [Design System](./JobLand_Design_System_Combined.md)

---

## Modules Overview

| # | Module | Technology | Database | Server |
|---|--------|-----------|----------|--------|
| 1 | Website | Next.js, TypeScript, Tailwind CSS | PostgreSQL (Prisma ORM) | — |
| 2 | Super Admin | Next.js, TypeScript, Tailwind CSS, Shadcn UI | PostgreSQL (Prisma ORM) | — |
| 3 | Backend API | Next.js, TypeScript, Prisma ORM, JWT, bcrypt, Zod | PostgreSQL (Prisma ORM) | Node.js (Next.js) |
| 4 | Mobile Application | React Native (Expo), TypeScript, Expo Router, React Navigation | PostgreSQL (Prisma ORM) | — |
| 5 | Database | Prisma ORM | PostgreSQL | — |
| 6 | Authentication & Security | JWT, Refresh Token, RBAC, bcrypt, Environment Variables | PostgreSQL (Prisma ORM) | — |
| 7 | Third-Party Services | Firebase Cloud Messaging, Nodemailer | — | — |
| 8 | Deployment | Docker, Nginx, PM2, SSL | — | VPS |

---

## 1. Website

**Technology**
- **Framework:** Next.js
- **Language:** TypeScript
- **Styling / CSS:** Tailwind CSS
- **API calls:** Axios
- **State management:** TanStack Query
- **Validation:** Zod

**Database**
- PostgreSQL (via Prisma ORM)

**Server**
- _TBD_

---

## 2. Super Admin

**Technology**
- **Framework:** Next.js
- **Language:** TypeScript
- **Styling / CSS:** Tailwind CSS
- **UI components:** Shadcn UI
- **API calls:** Axios
- **State management:** TanStack Query
- **Local/component state:** React Hooks
- **Validation:** Zod

**Database**
- PostgreSQL (via Prisma ORM)

**Server**
- _TBD_

---

## 3. Backend API

**Technology**
- **Framework:** Next.js
- **Language:** TypeScript
- **ORM / data access:** Prisma ORM
- **Authentication / tokens:** JWT (JSON Web Token)
- **Password encryption:** bcrypt
- **Validation:** Zod
- **Email:** Nodemailer
- **Push / messaging:** Firebase Cloud Messaging (FCM)

**Database**
- PostgreSQL (via Prisma ORM)

**Server**
- Node.js (Next.js API routes)

---

## 4. Mobile Application

**Technology**
- **Framework:** React Native (Expo)
- **Language:** TypeScript
- **Routing:** Expo Router
- **Navigation:** React Navigation
- **API calls:** Axios
- **State management:** React Hooks
- **Local storage:** Async Storage
- **Push notifications:** Expo Notifications
- **Secure storage (sensitive data):** Expo Secure Store

**Database**
- PostgreSQL (via Prisma ORM)

**Server**
- _TBD_

---

## 5. Database

**Technology**
- **Database:** PostgreSQL
- **ORM / data access:** Prisma ORM

**Database**
- PostgreSQL (via Prisma ORM)

**Server**
- _TBD_

---

## 6. Authentication & Security

**Technology**
- **Authentication:** JWT (JSON Web Token) Authentication
- **Session continuity:** Refresh Token
- **Authorization:** Role-Based Access Control (RBAC)
- **Password hashing:** bcrypt
- **Secrets / config:** Environment Variables

**Database**
- PostgreSQL (via Prisma ORM)

**Server**
- _TBD_

---

## 7. Third-Party Services

**Technology**
- **Push notifications / messaging:** Firebase Cloud Messaging (FCM)
- **Email:** Nodemailer

**Database**
- —

**Server**
- —

---

## 8. Deployment

**Technology**
- **Containerization:** Docker
- **Reverse proxy / web server:** Nginx
- **Process manager:** PM2
- **Security / encryption in transit:** SSL (HTTPS)

**Database**
- —

**Server**
- VPS (Virtual Private Server)
