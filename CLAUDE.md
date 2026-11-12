# CLAUDE.md — Appointment System (v1)

## Source of Truth & Scope
- Full specification lives in `docs/PRD.md`[cite: 1]. Always read the relevant section before implementing a slice.
- **v1 Scope:** Self-service registration/login, dashboard (upcoming + in-progress), appointment history, and booking 30/60/90/120-min `PLANNING`, `TRAINING`, or `CONSULTING` sessions with 3 seeded specialists[cite: 1, 14, 18].
- **Strict Non-Goals (Do NOT implement):** Cancellation/rescheduling UI, password reset, email/SMS notifications, admin console, specialist logins, payments, or external calendar sync[cite: 1]. (Reserve `appointments.status` and `cancelled_at` in schema only[cite: 1, 14]).

## Tech Stack & Versions
- **Runtime & Framework:** Node.js 22 LTS, Next.js App Router (15.x+; note `proxy.ts` instead of `middleware.ts` if on Next.js 16+), React 19, TypeScript (`strict: true`)[cite: 1, 2].
- **UI & Styling:** Tailwind CSS 4, `shadcn/ui` (Radix primitives)[cite: 2].
- **Validation:** Zod (`src/lib/validation.ts` shared across client forms, Server Actions, and Route Handlers)[cite: 2, 22].
- **Database & ORM:** PostgreSQL 16 (`btree_gist` extension), Drizzle ORM + `drizzle-kit`, `pg` driver[cite: 2, 12].
- **Auth & Security:** Argon2id via `@node-rs/argon2` (`m=19456`, `t=2`, `p=1`), opaque 32-byte random session tokens in `__Host-session` cookie (SHA-256 hex stored in `sessions.id`)[cite: 2, 39].
- **Dates/Timezones:** `date-fns` 4+ and `@date-fns/tz`[cite: 2].
- **Testing:** Vitest (unit + integration against real Postgres 16) and Playwright (E2E)[cite: 2].

## Commands & Verification Gates
Every vertical slice is incomplete until all checks pass:
- `npm run lint` — ESLint check
- `npm run typecheck` — `tsc --noEmit`
- `npm run db:migrate` — Apply Drizzle + custom SQL migrations (`drizzle/*.sql`)
- `npm test -- --run` — Run Vitest unit/integration suite
- `npm run build && npm run test:e2e` — Production standalone build + Playwright E2E tests

## Architecture & Directory Boundaries
```text
src/
├── middleware.ts (or proxy.ts)     # Cookie-presence redirect only; NO DB queries
├── app/
│   ├── (auth)/                     # Logged-out routes (/login, /register) + actions.ts
│   ├── (app)/                      # Authenticated shell (layout calls requireUser()) + actions.ts
│   └── api/
│       ├── v1/                     # Versioned REST API mirroring Server Actions
│       ├── healthz/route.ts        # Liveness (no DB call)
│       └── readyz/route.ts         # Readiness (SELECT 1 within 2s)
├── components/                     # UI components (Server by default; Client only where interactive)
├── lib/                            # Pure shared utilities (validation.ts, format.ts, time.ts, errors.ts)
└── server/                         # MUST start every file with `import "server-only";`
    ├── db/                         # client.ts (pg Pool with UTC), schema.ts
    ├── auth/                       # session.ts, password.ts, cookies.ts, rate-limit.ts, require-user.ts
    └── services/                   # Single source of truth for all business logic