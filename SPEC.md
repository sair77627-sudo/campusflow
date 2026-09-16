# CampusFlow — Smart Campus Operating System (v4)

You are a senior full-stack software architect and engineer.

Build CampusFlow, a production-quality full-stack web app acting as a
centralized operating system for a college campus. It replaces WhatsApp
groups, paper notices, Google Forms, spreadsheets, and disconnected portals.

**Tagline:** "One campus. One platform. Everything connected."

It must feel like a real SaaS product, not a tutorial CRUD app.

**Scope assumption:** this is a **single-institution** deployment (one
campus), not multi-tenant. If that assumption is wrong, say so before
Phase 0 — multi-tenancy changes the data model (an `Institution`/`Campus`
table everything else hangs off) and is much harder to retrofit later
than to design in from the start.

---

## HOW YOU SHOULD WORK (read before writing any code)

- Do NOT attempt all phases in one pass. Build ONLY the current phase,
  then stop and summarize what you built, what's tested, and what's next.
- Before writing code for a phase, restate its scope in 3-5 bullets and
  confirm it matches the phase definition below.
- Prioritize correctness, security, and UX over feature count.
- If a requirement conflicts with the chosen stack (e.g. a hosting
  constraint), flag the conflict and propose a resolution before coding —
  don't silently pick one.

---

## PHASE ROADMAP (build in this order; each phase = a working, demoable slice)

**Phase 0 — Foundation**
- Repo scaffold, Prisma schema for User/StudentProfile/FacultyProfile/
  Department/Course/Class, Auth.js (email+password), RBAC utility
  (`can(user, action)`), seed script with demo users for all 5 roles,
  minimal GitHub Actions CI (typecheck + lint + unit tests).
- *Done when:* can register/login as each role, see a role-aware empty
  dashboard shell, and a denied action returns 403 server-side even via
  a direct API call.

**Phase 1 — Academic Core**
- Timetable (read-only), Attendance (faculty marks, students view %),
  Assignments (create/submit/grade), deadline reminder job (see
  Background Jobs section).
- *Done when:* duplicate attendance records for the same
  student/class/date are rejected at the DB layer, not just the UI.

**Phase 2 — Complaints (flagship feature)**
- Full lifecycle: PENDING → ASSIGNED → IN_PROGRESS → RESOLVED → CLOSED
  (+ reopen), comments, evidence upload, audit log per status change,
  notification to the student on each change.
- *Done when:* status transitions are enforced server-side (no illegal
  jumps) and every transition writes an AuditLog row.

**Phase 3 — Notifications + Realtime layer**
- Build the notification service (persisted in Postgres, read/unread,
  preferences) with the realtime delivery abstraction wired to
  complaints from Phase 2 first.
- *Done when:* a complaint status change triggers a live in-app toast
  for the affected student without a page refresh.

**Phase 4 — Events + QR attendance**
- Club/event CRUD + approval lifecycle, registration, QR generation
  (`qrcode` npm package), scan-based check-in, "event starts in 30 min"
  reminder job.
- *Done when:* duplicate check-ins for the same registration are
  rejected via a unique constraint.

**Phase 5 — Resource Booking**
- Booking requests with overlap prevention via DB transaction or a
  Postgres exclusion constraint on the time range — never an
  application-level check-then-insert.

**Phase 6 — Lost & Found + Marketplace**
- Both reuse the same image-upload and search/filter patterns; build
  the shared components once (search, filters, pagination, upload) and
  apply to both. Lost & Found contact between finder and owner routes
  through the Messaging module (Phase 7) rather than a separate
  contact-reveal flow — no new PII-exposure surface.

**Phase 7 — Messaging**
- Conversations/messages over the realtime layer from Phase 3.

**Phase 8 — Admin Dashboard + Analytics**
- All metrics computed live from DB queries, charts via Recharts.

**Phase 9 — Hardening**
- Rate limiting, CSRF, secure headers, file-upload validation (size +
  real MIME sniffing, not just extension), error monitoring, test
  coverage for permission logic and booking-overlap logic specifically
  — these are the two places a silent bug becomes a security or
  data-integrity incident.

---

## TECHNOLOGY STACK

**Frontend:** Next.js (App Router) + React + TypeScript + Tailwind +
shadcn/ui + Recharts + React Hook Form + Zod

**Backend:** Next.js Route Handlers / Server Actions, TypeScript,
Prisma ORM. Keep business logic OUT of route handlers — route handler
calls a service in `src/server/services/`, service calls a repository
in `src/server/repositories/`. Route handlers should be thin.

**Database:** PostgreSQL via Neon or Supabase.

**Auth:** Auth.js, credentials provider, bcrypt/argon2 password
hashing, database session strategy (not pure JWT) so role changes take
effect without forcing re-login. Never store or log plaintext
passwords; never return password hashes in any API response.

**Realtime:** Use **Supabase Realtime** (Postgres change feeds +
broadcast/presence) rather than a raw Socket.IO server. Reason:
Vercel's serverless functions don't hold persistent WebSocket
connections, so a standard Socket.IO server won't stay alive there.
Supabase Realtime reads from the same Postgres you're already using
and works natively with serverless hosting.
- *Alternative:* if you specifically want Socket.IO, deploy the app
  (or just the realtime service) on a persistent-server host (Railway,
  Render, Fly.io) instead of Vercel — don't mix Socket.IO with pure
  serverless hosting.
- Either way, isolate realtime behind a service abstraction
  (`src/lib/realtime/`) so the concrete provider can be swapped
  without touching feature code.

**Background jobs:** Vercel Cron triggering a route handler for simple
periodic checks (assignment deadlines, event-starting-soon reminders).
If job complexity grows beyond "scan the DB on a schedule," move to
Inngest or Trigger.dev instead of hand-rolling a queue.

**Rate limiting:** Upstash Redis + `@upstash/ratelimit` (sliding
window), enforced in Next.js Edge Middleware for `/api/auth/*` and
other write-heavy endpoints. Chosen because standard Redis needs
persistent TCP connections that serverless functions can't hold —
Upstash is HTTP-based and stateless per request, which fits Vercel.

**Error monitoring:** Sentry (or equivalent) on both client and
server, plus structured logging at the service layer. Don't rely on
users reporting bugs as your only signal.

**Notifications:** In-app is real-time via Supabase Realtime, and a
defined subset also sends email via Resend (or similar) — at minimum,
assignment deadlines and event reminders. Decide and document which
notification types are in-app-only vs. in-app+email; don't leave this
implicit.

**Storage:** Cloudinary or S3-compatible, using **signed, direct-to-storage
upload URLs** generated by the server (not proxying file bytes through
the Next.js server). Never store files in Postgres.

**Search:** Postgres full-text search (`tsvector` column + GIN index)
per searchable model (Events, Announcements, Clubs, MarketplaceListing,
LostItem/FoundItem) — not plain `ILIKE '%query%'`, which won't perform
once there's real data volume. This is already in your stack, so no
new service is needed at this scale.

**Pagination:** cursor-based (keyset, on `id`/`createdAt`) for feeds
users scroll — Marketplace listings and Notifications specifically,
since offset pagination (`LIMIT/OFFSET`) skips or repeats rows under
concurrent inserts. Offset pagination is fine for admin tables where
that's not a concern.

**Concurrency control:** optimistic concurrency (compare `updatedAt` on
write, reject with a "this record changed, reload" error rather than
silently overwriting) for Complaint and Booking mutations specifically
— these are the two models most likely to have two people editing the
same row at once, beyond the overlap constraint already specified for
bookings.

**Images:** generate thumbnails on upload via Cloudinary/S3
transformations (profile pictures, listing/event images) rather than
serving full-resolution images everywhere; use `next/image` on the
frontend for responsive delivery.

**Rich text sanitization:** if announcements or messages ever get a
rich-text editor, sanitize server-side before persisting (e.g.
DOMPurify) — sanitizing only on render leaves stored XSS in the
database that hits every future viewer.

**Migrations:** `prisma migrate dev` locally, `prisma migrate deploy`
in CI/deploy. Do not use `prisma db push` beyond early prototyping —
it can silently drop columns and isn't safe once there's real data.

**Deployment:** App → Vercel (or Railway/Render if using self-hosted
Socket.IO). DB → Neon or Supabase. Storage → Cloudinary or S3.

---

## USER ROLES

STUDENT, FACULTY, CLUB_ADMIN, MAINTENANCE, ADMIN — permissions as
originally scoped (dashboard/timetable/attendance/assignments for
students; class management + grading for faculty; club/event
management for club admins; complaint queue for maintenance; full
platform control for admin).

**Rule:** never rely on frontend checks alone. Every protected backend
operation verifies permissions server-side via the centralized `can()`
utility — never scattered `if (user.role === 'ADMIN')` checks inline.
Never trust client-submitted user IDs, role claims, or ownership
fields; always derive identity from the authenticated session.

**Row-level scoping (not just action-level):** `can()` answers "may
this role perform this action" — it does not answer "may this specific
user see *this specific row*." A faculty member who can read attendance
in general must still be blocked from another faculty member's class if
they pass its ID directly in a request. Repositories for Attendance,
Submission, Complaint (assigned-to), and Booking must filter queries by
the requesting user's own/assigned records at the query level — never
trust an ID from the request to select the right row without that
filter. This is the standard IDOR (insecure direct object reference)
failure mode and it is not covered by role checks alone.

---

## APPLICATION STRUCTURE

```
src/
├── app/            (route groups: public, auth, dashboard, and one
│                    per feature area)
├── components/     (ui, layout, forms, tables, charts, shared)
├── lib/            (auth, db, validation, permissions, notifications,
│                    storage, realtime, utils)
├── server/         (services, repositories, jobs)
├── types/
└── prisma/schema.prisma
```

Never put business logic directly in a route handler or a giant file.
Separate UI / validation / DB access / business logic / authorization /
external services.

---

## DATABASE

Normalized Postgres schema via Prisma. Core models: User,
StudentProfile, FacultyProfile, Department, Course, Class, Enrollment,
TimetableEntry, Attendance, Assignment, Submission, Announcement,
Notification, Club, ClubMembership, Event, EventRegistration,
EventAttendance, Complaint, ComplaintComment, ComplaintAttachment,
LostItem, FoundItem, MarketplaceListing, MarketplaceFavorite,
Conversation, Message, Resource, Booking, AuditLog.

Include PKs/FKs, unique constraints (one Attendance row per
student+class+date; one EventAttendance row per registration), indexes
on frequently filtered columns (complaint category/status/location,
announcement audience/department), created/updated timestamps, soft
deletion where records must be recoverable, enums for all status
fields.

**Timezone handling:** store all timestamps in UTC; convert to
campus-local time only at the display layer. This matters most for
booking overlap checks, attendance dates, and event start times — a
timezone mismatch there produces subtle off-by-one-hour bugs in
exactly the logic that needs to be most correct.

**Audit log retention:** define a retention period up front (e.g. 12
months) and whether admins can export AuditLog data — it will contain
who-viewed/changed-what and grows indefinitely, which matters for a
college's real compliance obligations, not just as a nice-to-have.

---

## SECURITY REQUIREMENTS

Password hashing, server-side authorization on every mutation, Zod
input validation, output shaping (never leak hashes/internal IDs
unnecessarily), rate limiting via Upstash on auth + write endpoints,
secure/httpOnly cookies, CSRF protection on state-changing routes,
secure headers (CSP, HSTS, X-Frame-Options), file upload size limits +
real MIME sniffing (not just extension checks), Prisma parameterization
for SQL injection protection, output escaping for XSS, audit logging
on all sensitive admin/status-change actions. Never expose stack
traces or raw DB errors to the client — map to generic, logged, coded
error responses.

---

## API DESIGN

Consistent REST resources (GET/POST/PATCH/DELETE on plural nouns,
nested actions like `/api/complaints/:id/comments`), consistent JSON
response envelope, consistent HTTP status codes, centralized error
handler that maps internal errors to safe client-facing messages.

---

## UX REQUIREMENTS

Clean, minimal, responsive, accessible (WCAG AA color contrast +
keyboard navigation on all interactive elements), fast, consistent,
mobile-friendly. Sidebar nav on desktop, responsive nav on mobile.
Cards, data tables, dialogs, drawers, toast notifications, skeleton
loaders, empty states, confirmation dialogs on destructive actions.

Every async operation needs all four states: loading, success, error,
empty. No bare spinners with no empty/error fallback.

---

## TESTING & DEMO DATA

- Unit tests are mandatory for: the `can()` permission matrix, booking
  overlap logic, and complaint status-transition rules — these are the
  places where a silent bug becomes a security or data-integrity
  issue.
- Integration tests for at least one full flow per phase (e.g. student
  submits complaint → admin assigns → maintenance resolves → student
  notified).
- CI runs typecheck, lint, and unit tests on every push (GitHub
  Actions) — catch regressions before they reach Vercel.
- Seed script must produce: 1 admin, 2-3 faculty, ~15 students across
  2 departments, sample courses/classes/timetable, a few complaints in
  different states, a couple of published events, and 2-3 marketplace
  listings — enough to demo every module without manual data entry.

---

## ENVIRONMENT / CONFIG

Provide a `.env.example` covering: `DATABASE_URL`, Auth.js
secret/providers, storage credentials (Cloudinary or S3), Supabase
Realtime URL/key, Upstash Redis URL/token, Sentry DSN, email provider
key (Resend or similar). Never commit real secrets; document required
env vars in the README alongside local setup steps (install, migrate,
seed, run).

---

**Start with Phase 0 only** and confirm the schema before moving
further — that's the change most likely to produce a working app
instead of a pile of half-finished code.
