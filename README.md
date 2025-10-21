# Contractor Proposals SaaS (Web + Mobile)

Multi-tenant SaaS for U.S. contractors to create, send, and manage project proposals with one-click client approval via SMS and email.

## Architecture Diagram

```mermaid
flowchart LR
  subgraph Client
    W[Next.js Web] -- REST --> API
    M[Expo Mobile] -- Deep Links --> Web
    ClientUser[Client Browser] -- Approval Link --> Web
  end

  subgraph Backend
    API[NestJS API] -- Prisma --> DB[(PostgreSQL)]
    API -- Twilio --> SMS
    API -- SendGrid/SES --> Email
  end

  Nginx[(Reverse Proxy)] --> W
  Nginx --> API
```

## Multi-Tenancy & Isolation

- Single database with strong row-level isolation.
- Tables include `tenantId` with PostgreSQL RLS policies (optional) and service-layer enforcement.
- RLS pattern: set `app.tenant_id` per request and add policies `tenant_id = current_setting('app.tenant_id')::uuid`.
- Roles: Contractor (admin), Client (proposal recipient; no login needed for approval links).

## Database Schema (Prisma)

- Tenants, Users (CONTRACTOR/CLIENT), PasswordResetToken
- Proposals (status: Pending, Approved, Rejected, In Progress, Completed)
- Attachments (Photo/Video/PDF/Audio URLs)
- ProposalToken (public approval link), AuditLog (actions)

See `apps/api/prisma/schema.prisma`.

## API Routes & Logic (NestJS)

- Auth: `POST /auth/register`, `POST /auth/login`, `POST /auth/forgot-password`, `POST /auth/reset-password`
- Proposals: `GET /proposals` (tenant-scoped), `POST /proposals` (create, sends notifications)
- Public: `GET /proposals/public/:token` (read), `POST /proposals/respond/:token?approve=true|false` (one-click)

Tenant context passed via JWT or `X-Tenant-Id` header (demo). In production, prefer JWT claims + subdomain mapping.

## UI/UX Layout (Web)

- Dashboard: list proposals with status, totals, created date.
- Create Proposal: client details, area dropdown, costs, description, attachments (URL-based placeholder).
- Public Proposal Page: visual layout, totals, Approve/Reject buttons.
- Mobile-first: Bootstrap 5 styling; simple, fast flows.

## Code Overview

- Monorepo with workspaces
  - `apps/api`: NestJS + Prisma, Twilio/SendGrid stubs
  - `apps/web`: Next.js app router, Bootstrap UI
  - `apps/mobile`: Expo RN placeholder (opens links)
  - `packages/shared`: (reserved for shared libs)

## System Requirements

- Ubuntu 22.04+
- Node.js 18+ (API and Web)
- Docker (optional but recommended) and Docker Compose
- PostgreSQL 15+ (if not using Docker)

## Install & Configure (Linux, step-by-step)

1) Install system packages

```bash
sudo apt update && sudo apt install -y ca-certificates curl gnupg git build-essential python3
```

2) Install Node.js LTS

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
```

3) Clone and prepare

```bash
git clone <your-repo-url> contractor-proposals
cd contractor-proposals
cp apps/api/.env.example apps/api/.env
cp apps/web/.env.example apps/web/.env
```

4) Configure environment variables

- `apps/api/.env`:
  - `DATABASE_URL=postgresql://USER:PASS@HOST:5432/DB?schema=public`
  - `JWT_SECRET=your-strong-secret`
  - `APP_URL=https://your.domain` (web)
  - `API_URL=https://api.your.domain`
  - Twilio: `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_FROM_NUMBER`
  - Email: `SENDGRID_API_KEY` or SES credentials

- `apps/web/.env`:
  - `NEXT_PUBLIC_API_URL=https://api.your.domain`

5) Run with Docker (recommended for prod-like dev)

```bash
docker compose up --build -d
# Run migrations inside API container (first time)
docker compose exec api npx prisma migrate deploy
docker compose exec api node -e "require('child_process').execSync('node -v');"
```

6) Bare-metal (without Docker)

```bash
# Postgres
sudo apt install -y postgresql postgresql-contrib
sudo -u postgres createuser -P app && sudo -u postgres createdb -O app app

# Install deps
npm install
npm --workspace apps/api run prisma:generate
npm --workspace apps/api run prisma:dev

# Start services (two terminals)
npm --workspace apps/api run start:dev
npm --workspace apps/web run dev
```

7) Migrations

```bash
# Development
npm -w apps/api run prisma:dev

# Production
npm -w apps/api run prisma:migrate
```

8) Start backend and frontend

```bash
npm run dev
# API: http://localhost:4000
# Web: http://localhost:3000
```

9) Production deployment

- Reverse proxy: Nginx or Caddy in front of API and Web
- Process manager: systemd or PM2

Example systemd for API:

```
[Unit]
Description=Contractor Proposals API
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/contractor/apps/api
Environment=NODE_ENV=production
Environment=PORT=4000
Environment=JWT_SECRET=change-me
Environment=DATABASE_URL=postgresql://app:app@127.0.0.1:5432/app
ExecStart=/usr/bin/node dist/main.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Nginx site (example):

```
server {
  listen 80;
  server_name app.your.domain;
  location /api/ { proxy_pass http://127.0.0.1:4000/; }
  location / { proxy_pass http://127.0.0.1:3000/; }
}
```

Use Certbot to enable HTTPS.

## Notifications

- SMS: Twilio credentials required; in dev falls back to console logs.
- Email: SendGrid via Nodemailer (or swap to SES transport).
- Triggered on proposal creation and approval/rejection.

## Analytics & Templates (Roadmap)

- Dashboard: revenue/approval rates; add daily/weekly aggregations.
- Proposal templates: duplicate past proposals to accelerate quoting.

## Stripe Payments (Placeholder)

- Reserve `payments` module; add `paymentIntentId` field to `Proposal` on approval.

## Troubleshooting

- Prisma cannot connect: verify `DATABASE_URL` and database is reachable.
- Twilio/Email not sending: ensure credentials; in dev messages are logged to console.
- CORS errors: confirm `NEXT_PUBLIC_API_URL` and proxy config.
- Docker build fails on Prisma: clear cache and rebuild `docker compose build --no-cache`.

