# CampusFlow — Smart Campus Operating System

A production-quality full-stack web application serving as a centralized operating system for college campuses.

**Tagline:** "One campus. One platform. Everything connected."

## Tech Stack

- **Frontend:** Next.js 15 (App Router) + React 19 + TypeScript + Tailwind CSS
- **Backend:** Next.js Route Handlers + NextAuth.js + Prisma ORM
- **Database:** PostgreSQL (Supabase or Neon)
- **Auth:** NextAuth.js with credentials provider + bcrypt
- **Storage:** Cloudinary (or S3-compatible)
- **Realtime:** Supabase Realtime (Postgres change feeds)
- **Rate Limiting:** Upstash Redis
- **Email:** Resend
- **Error Monitoring:** Sentry

## Project Structure

```
src/
├── app/                     # Next.js App Router
│   ├── (public)/            # Public pages (login, register)
│   ├── (auth)/              # Auth pages
│   └── (dashboard)/         # Protected dashboard routes
├── components/              # React components
│   ├── ui/                  # Base UI components (button, card, etc.)
│   ├── layout/              # Layout components (navbar, sidebar)
│   ├── forms/               # Form components
│   └── shared/              # Shared components
├── lib/                     # Utilities & helpers
│   ├── auth/                # Auth utilities
│   ├── db/                  # Database helpers
│   ├── permissions/         # RBAC & permission checking
│   ├── validation/          # Zod schemas
│   └── utils/               # General utilities
├── server/                  # Backend logic
│   ├── services/            # Business logic services
│   ├── repositories/        # Data access layer
│   └── jobs/                # Background jobs
├── types/                   # TypeScript types & interfaces
└── prisma/
    ├── schema.prisma        # Prisma schema
    └── seed.ts              # Database seed script
```

## Getting Started

### Prerequisites

- Node.js 18+ and npm/yarn
- PostgreSQL database (Supabase, Neon, or local)
- Supabase account (optional, for realtime features)

### Setup

1. **Clone & install dependencies**
   ```bash
   npm install
   ```

2. **Configure environment**
   ```bash
   cp .env.example .env.local
   # Edit .env.local with your database URL, auth secret, etc.
   ```

3. **Set up database**
   ```bash
   npx prisma migrate dev --name init
   npx prisma generate
   ```

4. **Seed demo data**
   ```bash
   npx prisma db seed
   ```

5. **Start development server**
   ```bash
   npm run dev
   ```

   Open [http://localhost:3000](http://localhost:3000) in your browser.

## Demo Users

After seeding, use these credentials to test each role:

- **Admin:** admin@campus.local / password
- **Faculty:** faculty@campus.local / password
- **Student:** student@campus.local / password
- **Club Admin:** clubadmin@campus.local / password
- **Maintenance:** maintenance@campus.local / password

## Available Scripts

- `npm run dev` — Start development server
- `npm run build` — Build for production
- `npm start` — Start production server
- `npm run lint` — Run ESLint
- `npm run typecheck` — Run TypeScript type checker
- `npm run prisma:migrate` — Run Prisma migrations
- `npm run prisma:seed` — Seed the database
- `npm run prisma:studio` — Open Prisma Studio

## Phase Roadmap

See [SPEC.md](./SPEC.md) for the complete roadmap.

- **Phase 0 (Current):** Foundation, auth, RBAC, empty dashboard
- **Phase 1:** Academic Core (timetable, attendance, assignments)
- **Phase 2:** Complaints (flagship feature)
- **Phase 3:** Notifications + Realtime
- **Phase 4:** Events + QR attendance
- **Phase 5:** Resource Booking
- **Phase 6:** Lost & Found + Marketplace
- **Phase 7:** Messaging
- **Phase 8:** Admin Dashboard + Analytics
- **Phase 9:** Hardening & security

## Security

- All authentication handled via NextAuth.js with secure session storage
- Password hashing via bcrypt
- Server-side permission checks on every mutation (RBAC)
- Row-level scoping at the repository level to prevent IDOR vulnerabilities
- All secrets stored in environment variables (never committed)
- CSRF protection via NextAuth.js
- Input validation via Zod

## License

MIT
