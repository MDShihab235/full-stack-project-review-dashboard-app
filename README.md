# Project Submission & Review Dashboard
## Frontend live link: https://project-review-frontend-pi.vercel.app/
## Backend live link: https://project-review-backend-2sje.vercel.app/

Students submit projects; mentors/admins review them. Backend and frontend are fully separate apps.

```
project-review-dashboard/
├── docker-compose.yml        # local PostgreSQL
├── backend/                  # Express + Prisma + JWT (TypeScript)
│   ├── prisma/               # schema.prisma, seed.ts
│   └── src/
│       ├── config/           # validated env
│       ├── lib/              # prisma client, jwt, url normaliser, errors, helpers
│       ├── middleware/       # auth/RBAC, validate (zod), rate limits, error handler
│       └── modules/
│           ├── auth/         # schema, controller, routes
│           └── projects/     # schema, service, controller, routes
└── frontend/                 # Next.js 14 (App Router) + Tailwind + ShadCN UI
    └── src/
        ├── app/              # /login /signup /student /dashboard
        ├── components/       # feature components + ui/ (ShadCN primitives)
        ├── hooks/            # useAuth, useDebounce
        └── lib/              # api client, types, zod validators
```

## Quick start

```bash
# 1. Database
docker compose up -d

# 2. Backend  (http://localhost:4000)
cd backend
cp .env.example .env            # set a real JWT_SECRET (32+ chars)
npm install
npx prisma migrate dev --name init
npm run seed                    # admin + mentor (+ demo student & 25 projects)
npm run dev

# 3. Frontend  (http://localhost:3000)
cd ../frontend
cp .env.example .env.local
npm install
npm run dev
```

Seeded accounts (change in `backend/.env`):

| Role    | Email               | Password       |
|---------|---------------------|----------------|
| ADMIN   | admin@example.com   | Admin@12345    |
| MENTOR  | mentor@example.com  | Mentor@12345   |
| STUDENT | student@example.com | Student@12345  |

## API (`/api`)

| Method | Path                    | Role            | Notes |
|--------|-------------------------|-----------------|-------|
| POST   | /auth/signup            | public          | always creates a STUDENT |
| POST   | /auth/login             | public          | sets httpOnly JWT cookie |
| POST   | /auth/logout            | any             | |
| GET    | /auth/me                | any             | |
| POST   | /projects               | STUDENT         | validated, duplicate-checked, rate-limited |
| GET    | /projects/mine          | STUDENT         | paginated |
| PATCH  | /projects/:id           | STUDENT         | own, non-approved; returns it to PENDING |
| DELETE | /projects/:id           | STUDENT         | own, non-approved |
| GET    | /projects               | MENTOR, ADMIN   | `page, limit, status, search, sortBy, order` (server-side) |
| GET    | /projects/stats         | MENTOR, ADMIN   | total / pending / approved / rejected / approval rate / last 7 days |
| GET    | /projects/export.csv    | MENTOR, ADMIN   | honours `status` and `search` |
| PATCH  | /projects/:id/review    | MENTOR, ADMIN   | `{status: APPROVED\|REJECTED, feedback}`; feedback required on reject |

Responses: `{ success: true, data }` or `{ success: false, message, errors? }`.

## Decisions on the ambiguous parts

- **Roles**: `STUDENT`, `MENTOR`, `ADMIN`. Mentors and admins share the review dashboard. Public signup can only create students.
- **Duplicates**: URLs are normalised (https, no `www.`, no trailing slash / `.git` / query, case-insensitive repo path) and stored in `@unique` columns, so the DB enforces it even under race conditions. Both repo URL and live link are unique.
- **Rate limiting**: 300 req/15 min per IP globally; 20 failed auth attempts/15 min; 5 submissions or edits per 10 min per user; plus a cap of 5 projects pending review per student.
- **Validation**: Zod on both sides. Repo URL must be GitHub/GitLab/Bitbucket `owner/repo`; live link must be a public https URL (no localhost / private IPs).
- **Review rules**: only `PENDING` projects can be reviewed, done with an atomic `updateMany` so two mentors can't overwrite each other (second gets 409).
- **Auth**: JWT in an httpOnly cookie (not readable by JS, so safer than localStorage); `Authorization: Bearer` also accepted. For a frontend and API on different sites, set `COOKIE_SAME_SITE=none` (HTTPS required).

## Extra features beyond the brief

- Mentor feedback on rejection (required), visible to the student
- Students can edit/resubmit rejected or pending projects, or delete them
- Search + sorting on the review table, all server-side
- CSV export of the current filter (spreadsheet-formula-injection safe)
- Extra stats: approval rate and submissions in the last 7 days
- Per-student pending cap, login timing-attack mitigation, Helmet, body size limit, graceful shutdown
- Demo seed data for testing pagination
