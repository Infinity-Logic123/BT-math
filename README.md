# The BT Math — Educational Platform

A production-ready learning platform for **The BT Math** YouTube channel: every video lesson organized by class → subject → chapter → topic, with student accounts, progress tracking, courses, resources, announcements and a full admin portal.

---

## Table of contents

1. [Feature overview](#feature-overview)
2. [Tech stack](#tech-stack)
3. [Quick start](#quick-start)
4. [Environment variables](#environment-variables)
5. [Database & migrations](#database--migrations)
6. [YouTube API configuration](#youtube-api-configuration)
7. [Testing](#testing)
8. [Project structure](#project-structure)
9. [Security model](#security-model)
10. [Deployment](#deployment)
11. [Design decisions & assumptions](#design-decisions--assumptions)

---

## Feature overview

> **Access model:** the landing page is open to everyone; every content route (lessons, courses, watch pages, class pages, search, resources) requires authentication — anonymous visitors are redirected to the **login / sign-up screen** with a return URL, and anonymous API calls receive JSON 401s. After signing in, everything works: lessons, courses, resources, search and progress for students; the full admin console for admins.

### Public site
- Landing page with configurable hero, live stats, classes, subjects, featured/popular lessons, courses, announcements, CTA and footer
- SEO-friendly URLs (`/class/12/mathematics`, `/watch/[slug]`), dynamic metadata, Open Graph/Twitter cards, canonical URLs, `sitemap.xml`, `robots.txt`, JSON-LD (`EducationalOrganization`, `VideoObject`, `Course`, `BreadcrumbList`)
- Lesson library with filters (class, subject, chapter, difficulty), sorting and pagination
- Global search with type-grouped results + instant suggestions in the header
- Watch pages with privacy-enhanced YouTube embedding, related lessons, previous/next lesson, chapter progress, tags

### Student portal (`/dashboard`)
- Dashboard: continue watching, course progress rings, recommendations, announcements, activity calendar & learning streak
- Progress page: completed/in-progress lessons, streak stats
- My courses with per-course completion tracking
- Profile: edit name/class, change password, learning stats
- Announcements inbox with read tracking + notifications (bell badge)

### Admin portal (`/admin`)
- Dashboard: KPIs, registration/view charts, top videos, recent signups & activity
- Video manager: manual add (with YouTube oEmbed auto-fill), bulk import from the channel (**RSS — no API key needed** — or the official Data API), publish/feature toggles, taxonomy assignment with consistency validation, search/filter/pagination, soft delete
- Curriculum manager: classes → subjects → chapters → topics
- Course builder: ordered lesson lists, enrollments, publish control
- Student management: search/filter, view profiles & progress, activate/deactivate (kills sessions), promote/demote, revoke sessions, one-time temporary passwords
- Resources: notes/PDF/question papers with file upload (25 MB, type allow-list) or external links, download counters
- Announcements: target all students or a class, scheduling, lazy notification fan-out
- Analytics: 7/30/90-day trends (registrations, views, completions), subject distribution, course completion averages
- Settings: site identity, socials, contact, homepage copy, theme accent, YouTube channel, registration on/off — all DB-backed
- Audit log: every sensitive admin action recorded (actor, action, target, IP)

---

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Framework | Next.js 14 (App Router, TypeScript, strict) | SSR/RSC for speed & SEO, one deployable server, typed end-to-end |
| Styling | Tailwind CSS 3.4 + CSS-variable theming | Fast, consistent, themeable without a runtime |
| DB / ORM | SQLite + Prisma (PostgreSQL-ready) | Zero-config locally; one-line provider switch for production |
| Auth | Server-side sessions (SHA-256-hashed tokens in DB) + bcrypt (cost 12) | Revocable sessions, no plaintext, no JWT-in-localStorage pitfalls |
| Validation | Zod on every API boundary | One schema language, typed outputs |
| Email | Nodemailer (SMTP) with dev outbox fallback | Verification + password reset when SMTP configured |
| Charts | Hand-rolled server-rendered SVG | Zero chart-library JS; excellent Core Web Vitals |
| Icons | lucide-react (tree-shaken) | Consistent, accessible iconography |

**Live preview JS budget:** ~87 kB shared first-load JS.

---

## Quick start

```bash
# 1. Install dependencies
npm install

# 2. Create the database schema (SQLite at prisma/dev.db)
npm run db:push

# 3. Seed admin + demo curriculum
npm run db:seed

# 4. Run
npm run dev          # development (http://localhost:3000)
# or
npm run build && npm start   # production
```

### Seeded accounts (from `.env`)

| Role | Email | Password |
|---|---|---|
| Admin | `admin@thebtmath.com` | `ChangeMe!2026` |
| Demo student | `student@demo.com` | `Student@12345` |

> ⚠️ **Change the admin password immediately** (Admin → not needed: use *Forgot password* with SMTP configured, or set `ADMIN_PASSWORD` before seeding). Delete the demo student via **Admin → Students** for production.

### Live channel auto-sync

New uploads on the configured YouTube channel are pulled into the platform automatically:
- **Server/Docker/VPS** — the in-app scheduler checks the channel feed every `YOUTUBE_SYNC_INTERVAL_MINUTES` (default 15) and adds new uploads as **drafts** (Admin → Videos → publish after organizing). Disable with `YOUTUBE_AUTO_SYNC="false"`.
- **Serverless (Vercel)** — background timers don't persist, so set `CRON_SECRET` and let a platform cron call `POST /api/cron/youtube-sync` with the `x-cron-secret` header (see `docs/DEPLOYMENT.md`).
- Admins receive a notification whenever new videos are imported, and Admin → Settings shows the last sync time with a manual **Sync now** button.

### Changing the admin email / password

**Via the UI (recommended):** sign in as admin → **Admin → Settings → "Admin account"** → enter the new sign-in email and/or a new password, confirm with your **current password** → *Save*. All sessions are revoked and you sign in again with the new credentials. Every change is written to the audit log (`ADMIN_ACCOUNT_UPDATE`).

**Via the CLI (headless servers / automation):**
```bash
node scripts/set-admin.mjs --email you@example.com --password 'YourNewPass123'
# either flag alone works; password must be 8+ chars with letters and numbers
```

### First content setup (2 minutes)

> **This workspace is already set up:** the real channel (*The BT Math* — `UC2iqrFhsRNS5Vup0Uh7uA5A`) is configured in settings and **13 lessons are imported, organized and published**. To re-sync new uploads from the channel at any time:
>
> ```bash
> npx tsx scripts/import-thebtmath.ts   # idempotent; updates + imports latest uploads
> ```
>
> Or use the same flow by hand as described below (this is what runs on a fresh deployment).

1. Sign in at `/login` → you land on `/admin`
2. **Settings → YouTube channel**: paste your Channel ID (`UC…`) and channel URL → Save
3. **Videos → Import from YouTube**: choose *Channel RSS feed* (no API key needed) → Import
4. Assign imported videos to class/subject/chapter (they start as drafts), then publish

---

## Environment variables

Copy `.env.example` → `.env`. Only `DATABASE_URL` is required; everything else degrades gracefully.

| Variable | Required | Purpose |
|---|---|---|
| `DATABASE_URL` | ✅ | `file:./dev.db` (SQLite) or Postgres URL |
| `NEXT_PUBLIC_SITE_URL` | – | Canonical base URL; auto-detected from request headers when empty |
| `ADMIN_NAME` / `ADMIN_EMAIL` / `ADMIN_PASSWORD` | seed | Bootstrap admin created by `db:seed` |
| `SEED_DEMO_DATA` | seed | `"true"` seeds curriculum taxonomy + demo student; set `"false"` in production |
| `YOUTUBE_API_KEY` | – | YouTube Data API v3 key. **Server-only** — never sent to the browser |
| `YOUTUBE_CHANNEL_ID` | – | Fallback channel ID (also configurable in Admin → Settings) |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASS` / `SMTP_FROM` | – | Email delivery. **Without SMTP**: accounts auto-verify and password reset happens via Admin → Students → *Set temporary password* |

---

## Database & migrations

```bash
npm run db:migrate     # create/apply a migration in development
npm run db:deploy      # apply committed migrations (production)
npm run db:push        # quick sync during prototyping
npm run db:studio      # browse data
```

**Switching to PostgreSQL** (recommended for production):

1. In `prisma/schema.prisma`: `provider = "postgresql"`
2. `DATABASE_URL="postgresql://user:pass@host:5432/btmath"`
3. `npx prisma migrate dev --name init` (regenerates SQL), then `npm run db:deploy`

Schema highlights: soft deletion with slug tombstoning, unique constraints that play well with soft deletes, indexes on every hot path (taxonomy lists, progress lookups, view analytics), `ApiCache` table for YouTube quota conservation, `AuditLog` for sensitive actions. Full ERD in [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

---

## YouTube API configuration

The platform integrates with YouTube in **embed-only** fashion (`youtube-nocookie.com`) — nothing is downloaded or re-hosted.

| Capability | Needs API key? | Notes |
|---|---|---|
| Manual video add + oEmbed metadata lookup | ❌ | Public oEmbed endpoint returns title/author/thumbnail |
| Channel import (RSS) — latest uploads | ❌ | The channel's public uploads feed |
| Channel import (Data API v3) + durations | ✅ | Official API, quota-aware |
| Watch-time/position sync | – | Not implemented by design (YouTube TOS-friendly) |

**Getting an API key (optional):**
1. Google Cloud Console → create/select a project
2. Enable **YouTube Data API v3**
3. Create credentials → API key (restrict it to that API)
4. Set `YOUTUBE_API_KEY` in the server environment (never in client code)

Quota errors (`quotaExceeded`) surface as friendly admin messages; all API/RSS responses are cached in the `ApiCache` table (6 h TTL) to conserve quota.

---

## Testing

```bash
npm run typecheck   # strict TypeScript, zero errors
npm run lint        # ESLint (next/core-web-vitals)
npm test            # 34 vitest unit tests (auth utils, YouTube parsing, validation, streaks, rate limiter…)
npm run smoke       # 73-check end-to-end suite against a running server (see scripts/smoke-test.mjs)
```

The smoke suite covers: public pages, SEO files, full auth lifecycle, RBAC (anon/student/admin boundaries), IDOR guards, CSRF origin checks, content CRUD, taxonomy consistency, student management actions, progress tracking, view throttling, search, announcements, settings persistence, audit logging and login rate limiting.

---

## Project structure

```
├── prisma/                  # schema, migrations, seed
├── scripts/smoke-test.mjs   # e2e smoke suite
├── tests/                   # vitest unit tests
├── docs/                    # architecture, API reference, deployment, commit plan
└── src/
    ├── middleware.ts        # CSP (nonce), security headers, auth pre-check
    ├── lib/                 # db, auth, api kernel, youtube, settings, mailer,
    │                        # analytics, progress, validation, audit, rate-limit…
    ├── components/
    │   ├── ui/              # primitives, overlays (toast/modal), data table
    │   ├── admin/           # admin shell + feature clients
    │   ├── auth/            # auth forms
    │   └── …                # header/footer, cards, charts, search
    └── app/
        ├── (public)/        # landing, /videos, /class/*, /watch/*, /courses, /search
        ├── (auth)/          # login, register, forgot/reset password, verify email
        ├── (student)/       # /dashboard/*, /announcements
        ├── admin/           # admin portal (guarded layout)
        └── api/             # auth, public, student, admin REST APIs
```

---

## Security model

- **Passwords**: bcrypt cost 12; complexity enforced (8+ chars, letters + numbers); never logged
- **Sessions**: 256-bit random tokens; only SHA-256 hashes stored; httpOnly + SameSite=Lax + Secure (prod) cookies; 30-day expiry; server-side revocation (per-session or account-wide)
- **RBAC**: `ADMIN`/`STUDENT` role checked server-side in layouts **and** every API handler (`requireAdmin`/`requireUser`) — the UI never gates anything alone
- **CSRF**: SameSite cookies + Origin/Host verification on all state-changing requests
- **SQL injection**: Prisma parameterized queries exclusively
- **XSS**: React auto-escaping; strict CSP with per-request nonce + `strict-dynamic`; no `dangerouslySetInnerHTML` on user content (JSON-LD payloads are server-controlled data only)
- **IDOR**: every read/write is scoped to the session user; admin APIs validate entity existence and return uniform 403/404s
- **Rate limiting**: sliding-window limiter on login (per-IP and per-email), registration, password reset and view endpoints (in-memory; swap in Redis for multi-instance — see DEPLOYMENT)
- **Uploads**: extension allow-list, 25 MB cap, randomized storage names, path-traversal-safe download route, `nosniff`
- **Secrets**: API keys and SMTP credentials live only in env vars; the settings API returns capability booleans, never values
- **No backdoors**: no universal passwords or bypasses; bootstrap admin comes only from seeded credentials which you rotate
- **Login-first gate**: edge middleware redirects anonymous visitors to `/login` (with return URL) and returns JSON 401 for anonymous API calls; the `(public)` layout re-verifies the session server-side, and `robots.txt` allows only the auth screens

---

## Deployment

See **[docs/DEPLOYMENT.md](docs/DEPLOYMENT.md)** — covers Vercel + Postgres, Docker/VPS with Nginx, environment hardening, migrations and backups. A `Dockerfile` and `docker-compose.yml` are included.

---

## Design decisions & assumptions

- **Two roles, not a permission matrix** — the spec allows judgment: a full roles/permissions graph would add tables without a consumer. `User.role` + server-side guards + audit log covers real needs; extending later is additive.
- **"Lesson" = a YouTube video placed in the taxonomy** (`Video` rows). A separate `lessons` table would duplicate content; `Course` = ordered playlist via `CourseItem`. `course_progress` is *derived* (completed items ÷ items) rather than stored.
- **Channel not hard-coded**: the public channel ID for "The BT Math" could not be reliably resolved, so the platform ships with configurable channel settings + one-click import (this is exactly the "real configurable content" requirement). No fake videos are seeded.
- **Email optional by design**: without SMTP, verification is auto-granted and reset is admin-mediated — documented in-app so owners aren't surprised.
- **SQLite default, Postgres-ready**: keeps local setup to one command; production docs recommend Postgres.
- **Embed-only YouTube**: `youtube-nocookie.com`, no downloads, no position-scraping.
