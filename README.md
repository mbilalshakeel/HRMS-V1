# White-Label HRMS SaaS

A multi-tenant HR management platform: geofenced attendance, leave management, role-based dashboards, in-app notifications, per-company white-label branding, and custom-domain support. Built from the attached PRD.

- **Frontend:** React 18, Vite, Tailwind CSS, React Router
- **Backend:** Node.js, Express, Prisma ORM
- **Database:** PostgreSQL
- **Auth:** JWT with role-based access control (Employee, Manager, Admin)
- **Geofencing:** Haversine distance check against per-branch GPS coordinates and radius

---

## What is implemented

Every module in the PRD is built and runnable:

1. **Multi-tenancy** — single database, `company_id` on every table, tenant resolved per request (custom domain → `x-company-slug` header → subdomain). Auth middleware asserts the JWT user belongs to the resolved tenant on every call.
2. **Roles and permissions** — Employee, Manager, Admin enforced by RBAC middleware (`requireRole`, `requireAtLeast`).
3. **Geofenced check-in/out** — browser geolocation sent to the API, validated against the employee's assigned branch within its radius (default 500 m). Out-of-range attempts are rejected and the response includes how far away the user is.
4. **Multi-branch** — each branch carries its own latitude, longitude, and radius.
5. **Site visit mode** — employee-submitted, no geofence, no approval, shown in its own calendar color.
6. **Color-coded calendar** — On Time (green), Late (yellow), Absent (red), Site Visit (blue), On Leave (grey).
7. **Leave management** — configurable leave types, request flow, manager/admin approval, per-type yearly balances, approved leave written onto the attendance calendar as On Leave.
8. **Notifications** — in-app notifications for late check-in, marked absent, leave decision, and leave request received. The notification service has email/SMS channel stubs so other channels can be added without touching callers.
9. **White-label branding** — per-company logo, theme color, and name, applied live in the UI via a CSS variable.
10. **Custom domain support** — add a domain, receive CNAME + TXT verification instructions, verify ownership via a real DNS TXT lookup. See "Custom domains and SSL" below for the ops boundary.
11. **Reports** — attendance is filterable by branch, status, and date; managers see their team, admins see the whole company.

### Scope boundary (read this)

This is a complete, runnable foundation, not a hardened commercial release. The application layer for every module is real. The following are deliberately left as the ops/edge layer and documented rather than faked in app code:

- **Live SSL certificate issuance** for custom domains. The data model, verification token, DNS TXT check, and status tracking are implemented. Actual certificate provisioning belongs at your reverse proxy / ACME layer (see below).
- **Payments and billing.** The PRD's pricing section is informational; no payment processor is wired in.
- **Email/SMS delivery.** In-app notifications are fully implemented; email and SMS are stubbed channels.

---

## Prerequisites

- Node.js 18+ and npm
- PostgreSQL 14+ (or use the included `docker-compose.yml`)

---

## Quick start

### 1. Start PostgreSQL

Using Docker:

```bash
docker compose up -d db
```

This exposes Postgres on `localhost:5432` with user `hrms`, password `hrms`, database `hrms`. If you use your own Postgres instead, just point `DATABASE_URL` at it in the next step.

### 2. Backend

```bash
cd backend
cp .env.example .env
# If you used docker compose above, set:
# DATABASE_URL="postgresql://hrms:hrms@localhost:5432/hrms?schema=public"
# Set a real JWT_SECRET.

npm install
npm run prisma:generate
npm run prisma:migrate      # creates tables (name the migration e.g. "init")
npm run db:seed             # loads demo tenants, users, branches, attendance
npm run dev                 # API on http://localhost:4000
```

Health check: `GET http://localhost:4000/api/health`.

### 3. Frontend

```bash
cd frontend
cp .env.example .env
npm install
npm run dev                 # app on http://localhost:5173
```

The Vite dev server proxies `/api` to `http://localhost:4000`, so no CORS setup is needed in development.

---

## Demo credentials

All seeded users share the password **`Password123!`**. Two tenants are seeded to demonstrate isolation.

**Acme** (slug `acme`, blue theme):

| Email | Role |
| --- | --- |
| admin@acme.com | Admin |
| manager@acme.com | Manager |
| employee@acme.com | Employee (Emma, has sample attendance) |
| noah@acme.com | Employee |

**Globex** (slug `globex`, green theme):

| Email | Role |
| --- | --- |
| admin@globex.com | Admin |

In local dev the active tenant is set by `VITE_COMPANY_SLUG` in `frontend/.env` (defaults to `acme`). The login page themes itself from that tenant's branding. Change the slug to `globex` and restart Vite to see the other tenant's branding and confirm data isolation.

---

## How tenancy is resolved

On every `/api` request (except `/api/health` and `/api/platform/*`), the tenant middleware resolves the company in this order:

1. **Custom domain** — the request `Host` matches a verified domain row.
2. **`x-company-slug` header** — used by the frontend in development.
3. **Subdomain** — e.g. `acme.yourplatform.com`.

The resolved company is attached to the request. After authentication, the JWT's user is checked to belong to that same company, so a token from one tenant cannot be used against another.

---

## Custom domains and SSL

The flow that lives in this codebase:

1. Admin adds a domain (e.g. `hr.acme.com`). The API returns two DNS records to create: a **CNAME** pointing at your platform ingress (`PLATFORM_CNAME_TARGET`) and a **TXT** record containing a per-domain verification token.
2. Admin creates those records at their DNS provider, then clicks Verify.
3. The API performs a real `dns.resolveTxt` lookup and marks the domain Verified when the token matches.

What belongs at your infrastructure layer (not in app code): once a domain is verified, your reverse proxy or platform issues the TLS certificate. The standard approach is on-demand TLS via Caddy, an ACME client (Certbot/lego/acme.sh), or a managed option (Cloudflare for SaaS, AWS ACM). The `Domain` model already tracks `sslStatus` so your provisioning hook can update it. Faking certificate issuance inside the Node app would not produce trusted certificates, so it is intentionally left to that layer.

In development you can mark a domain verified without real DNS by sending `{ "force": true }` to the verify endpoint (only honored when `NODE_ENV=development`).

---

## API overview

Base path: `/api`. All routes except `/api/health` and `/api/auth/login` require a `Bearer` token. In development also send `x-company-slug: acme`.

| Area | Method & path | Role |
| --- | --- | --- |
| Auth | `POST /auth/login`, `GET /auth/me` | public / any |
| Attendance | `POST /attendance/check-in`, `POST /attendance/check-out`, `POST /attendance/site-visit` | Employee+ |
| Attendance | `GET /attendance/me/calendar?month=YYYY-MM`, `GET /attendance/me/today` | Employee+ |
| Attendance | `GET /attendance` (team for Manager, all for Admin; filters: branch, status, date) | Manager+ |
| Leave | `POST /leave`, `GET /leave/me`, `GET /leave/me/balances` | Employee+ |
| Leave | `GET /leave/pending`, `POST /leave/:id/decision` | Manager+ |
| Leave | `PATCH /leave/policy` | Admin |
| Users | `GET /users`, `POST /users`, `PATCH /users/:id`, `DELETE /users/:id` | Admin |
| Branches | `GET/POST/PATCH/DELETE /branches` | Admin (read Manager+) |
| Notifications | `GET /notifications`, `POST /notifications/:id/read`, `POST /notifications/read-all` | any |
| Branding | `GET /branding`, `PATCH /branding` | read any / write Admin |
| Domains | `GET/POST /domains`, `POST /domains/:id/verify`, `DELETE /domains/:id` | Admin |

---

## Tests

Backend logic tests (Haversine distance, geofence boundary, check-in status classification):

```bash
cd backend
npm test
```

---

## Project structure

```
hrms-saas/
├── backend/
│   ├── prisma/
│   │   ├── schema.prisma      # 8 models, multi-tenant, enums
│   │   └── seed.js            # two demo tenants
│   ├── src/
│   │   ├── config/            # env, prisma client
│   │   ├── middleware/        # tenant, auth, rbac, error
│   │   ├── services/          # attendance status, notifications
│   │   ├── controllers/       # one per resource
│   │   ├── routes/            # one per resource + index
│   │   ├── utils/             # jwt, password, geo, ApiError
│   │   ├── app.js
│   │   └── server.js
│   └── test/
├── frontend/
│   ├── src/
│   │   ├── api/               # fetch client (token + tenant header)
│   │   ├── context/           # Auth, Branding
│   │   ├── components/        # Layout, calendar, notification bell, guards
│   │   ├── pages/             # dashboards, leave, users, branches, branding, domains
│   │   ├── App.jsx
│   │   └── main.jsx
│   └── vite.config.js
├── docker-compose.yml
└── package.json               # workspace convenience scripts
```

## Production notes

- Set a long random `JWT_SECRET`, `NODE_ENV=production`, and a real `DATABASE_URL`.
- Run `npm run prisma:deploy` (instead of `migrate dev`) to apply migrations.
- Build the frontend with `npm run build` and serve `dist/` from your CDN or static host.
- Put the API behind a reverse proxy that terminates TLS and handles the custom-domain certificate issuance described above.
