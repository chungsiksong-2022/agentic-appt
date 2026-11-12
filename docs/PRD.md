# Appointment System — PRD & Technical Specification

Nov 12, 2026 · @Chungsik

## 1. Executive Summary & User Personas

Version 1 ships a self-service web app where a registered user books a 30–120 minute Planning, Training, or Consulting session with one of three specialists; double-booking is made impossible by a PostgreSQL exclusion constraint, not just application checks.

### Goals

- Self-serve registration and email/password login meeting OWASP ASVS Level 1 for authentication and session management.
- A user can book an appointment in under 60 seconds through one form with 4 inputs.
- Zero overlapping bookings per specialist and per user, guaranteed by the database.
- Keyless, reproducible delivery: every merge to `main` is built, migrated, and deployed to Cloud Run by GitHub Actions authenticating through Workload Identity Federation (no JSON keys).

### Non-goals (v1)

Cancellation and rescheduling UI, password reset, email/SMS notifications, admin console, specialist logins, payments, and external calendar sync are out of scope. The schema reserves an `appointments.status` column so cancellation can ship in v1.1 without a breaking migration.

### Tech stack

| Layer | Choice | Notes |
| --- | --- | --- |
| Runtime | Node.js 22 LTS | Same major version in CI, Docker, and local dev |
| Framework | Next.js (App Router) 15.x or later, TypeScript `strict: true` | `output: "standalone"` in `next.config.ts`; on Next.js 16+, `middleware.ts` is named `proxy.ts` |
| UI | React 19, Tailwind CSS 4, shadcn/ui (Radix primitives) | Accessible form controls out of the box |
| Validation | Zod | One schema module shared by client forms, Server Actions, and Route Handlers |
| Data access | Drizzle ORM + drizzle-kit, `pg` driver | SQL-first migrations; constraints Drizzle cannot express live in raw SQL migrations |
| Database | PostgreSQL 16 on Cloud SQL | Extension `btree_gist` for the exclusion constraints |
| Password hashing | Argon2id via `@node-rs/argon2` | m=19456 KiB, t=2, p=1 (OWASP minimum) |
| Sessions | Opaque random token in an HTTP-only cookie; SHA-256 hash stored in `sessions` | Revocable server-side; chosen over JWT so logout and forced expiry take effect immediately |
| Dates | `date-fns` 4 + `@date-fns/tz` | All storage in UTC `timestamptz` |
| Testing | Vitest (unit/integration against a Postgres service container), Playwright (E2E) |  |
| Infrastructure | Cloud Run, Cloud SQL, Artifact Registry, Secret Manager, GitHub Actions + WIF | Region `us-central1` unless changed |

### Architecture decisions

1. **Server Actions for UI mutations, Route Handlers for a versioned REST API.** Both call the same service layer (`src/server/services/*`), so business rules live in one place. The REST API (`/api/v1/*`) exists for integration tests, scripts, and future mobile clients.
2. **Database-backed sessions over stateless JWTs.** A stolen JWT stays valid until expiry; a DB session is deleted on logout and can be revoked in bulk.
3. **The database is the final arbiter of conflicts.** The app pre-checks availability for good error messages, but correctness under concurrent requests comes from `EXCLUDE USING gist` constraints.

### User personas

| Persona | Profile | Goals | Pain points today | v1 surface |
| --- | --- | --- | --- | --- |
| Jordan Ellis — Client | 34, operations manager at a 40-person logistics firm; books Planning and Consulting sessions monthly from a laptop, sometimes from a phone | See what is coming up at a glance; book without email back-and-forth | Scheduling by email takes days; time zones cause missed calls | Register, login, dashboard, booking |
| Aisha Bello — Returning learner | 27, analyst on a weekly Training cadence | Track sessions completed; rebook the same trainer quickly | Loses track of past sessions and durations | Dashboard, history, booking |
| Sam Ortiz — Operations admin (internal) | Maintains the specialist roster | Add or deactivate specialists safely | None in v1 | No UI in v1; roster changes ship as SQL migrations |

### Success metrics

| Metric | Target | Source |
| --- | --- | --- |
| Booking completion rate (submits ÷ visits to `/appointments/new`) | ≥ 85% | Structured logs |
| Booking conflict rate (409 responses ÷ booking submits) | < 5% | Structured logs |
| Dashboard server response time, warm instance | p95 < 300 ms | Cloud Run request latency |
| Overlapping `BOOKED` rows | 0, always | Enforced by constraint; nightly audit query |

## 2. User Stories & Acceptance Criteria

Each story below maps to one Playwright spec file under `e2e/` with the same feature name; every scenario is a required test. Shared definitions: the session cookie is `__Host-session`; "now" is server time in UTC; bookable hours and limits come from Section 6.

### Stories

| ID | As a… | I want… | So that… |
| --- | --- | --- | --- |
| AUTH-1 | visitor | to register with first name, last name, email, and password | I can book appointments |
| AUTH-2 | registered user | to log in with email and password | I can access my appointments |
| AUTH-3 | logged-in user | to log out, and be logged out after inactivity | nobody else can use my session |
| DASH-1 | logged-in user | to see my upcoming and in-progress appointments on landing | I know what is next |
| HIST-1 | logged-in user | to see my past appointments | I can track sessions completed |
| BOOK-1 | logged-in user | to book a type, specialist, duration, and start time | I get a confirmed slot |
| BOOK-2 | logged-in user | to be blocked from conflicting or invalid times | I never get a broken booking |

### Feature: Registration (AUTH-1)

```gherkin
Feature: Registration

  Scenario: Successful registration
    Given no account exists for "jordan@example.com"
    And I am on "/register"
    When I submit First Name "Jordan", Last Name "Ellis", Email "  Jordan@Example.COM ", Password "correct-horse-battery", Confirm Password "correct-horse-battery"
    Then a user row exists with email "jordan@example.com"
    And its password_hash starts with "$argon2id$"
    And the response sets cookie "__Host-session" with HttpOnly, Secure, SameSite=Lax, Path=/
    And I am redirected to "/dashboard"

  Scenario: Email already registered (case-insensitive)
    Given an account exists for "jordan@example.com"
    When I register with Email "JORDAN@example.com"
    Then no new user row is created
    And the Email field shows "An account with this email already exists."
    And the API responds 409 with error code "EMAIL_TAKEN"

  Scenario Outline: Field validation
    When I register with <field> set to <value>
    Then the <field> field shows <message>
    And the API responds 400 with error code "VALIDATION_ERROR"

    Examples:
      | field            | value                   | message                                     |
      | First Name       | ""                      | "First name is required."                   |
      | Last Name        | 101 characters          | "Last name must be 100 characters or fewer." |
      | Email            | "jordan@"               | "Enter a valid email address."              |
      | Password         | "short-pw1" (9 chars)   | "Password must be at least 12 characters."  |
      | Password         | 129 characters          | "Password must be 128 characters or fewer." |
      | Confirm Password | different from Password | "Passwords do not match."                   |

  Scenario: Logged-in user visits registration
    Given I am logged in
    When I visit "/register"
    Then I am redirected to "/dashboard"
```

### Feature: Login, logout, and session lifetime (AUTH-2, AUTH-3)

```gherkin
Feature: Login and session

  Background:
    Given an account exists for "jordan@example.com" with password "correct-horse-battery"

  Scenario: Successful login
    When I log in with "jordan@example.com" and "correct-horse-battery"
    Then a sessions row is created with expires_at = now + 7 days
    And the response sets cookie "__Host-session"
    And I am redirected to "/dashboard"

  Scenario: Login honours a safe return path
    Given I visited "/appointments/history" while logged out
    When I log in successfully
    Then I am redirected to "/appointments/history"

  Scenario: Open-redirect attempt is ignored
    When I log in successfully from "/login?next=https://evil.example"
    Then I am redirected to "/dashboard"

  Scenario Outline: Invalid credentials give one generic error
    When I log in with <email> and <password>
    Then the form shows "Invalid email or password."
    And the API responds 401 with error code "INVALID_CREDENTIALS"
    And no session cookie is set

    Examples:
      | email                  | password                |
      | "jordan@example.com"   | "wrong-password-123"    |
      | "nobody@example.com"   | "correct-horse-battery" |

  Scenario: Brute-force throttling
    Given 5 failed login attempts for "jordan@example.com" in the last 15 minutes
    When I log in with the correct password
    Then the API responds 429 with error code "RATE_LIMITED" and a Retry-After header in seconds
    And the form shows "Too many attempts. Try again in a few minutes."

  Scenario: Logout
    Given I am logged in
    When I click "Log out"
    Then my sessions row is deleted
    And the "__Host-session" cookie is cleared (Max-Age=0)
    And I am redirected to "/login"
    And replaying the old cookie on "/dashboard" redirects to "/login"

  Scenario: Idle session expiry
    Given my session's expires_at is in the past
    When I visit "/dashboard"
    Then I am redirected to "/login?next=%2Fdashboard&reason=expired"
    And the login page shows "Your session expired. Please log in again."

  Scenario: Sliding renewal
    Given my session expires in less than 3.5 days
    When I load any authenticated page
    Then expires_at is extended to now + 7 days, capped at created_at + 30 days
```

### Feature: Dashboard (DASH-1)

```gherkin
Feature: Dashboard

  Background:
    Given I am logged in as "jordan@example.com" with browser time zone "America/Chicago"

  Scenario: Upcoming appointments are listed soonest first
    Given I have BOOKED appointments starting in 2 days and in 1 day
    And I have an appointment that ended yesterday
    When I visit "/dashboard"
    Then I see 2 rows in "Current Active Appointments", the 1-day one first
    And each row shows Date & Time, Appointment Type, Assigned Specialist, Duration
    And Date & Time renders like "Tue, Mar 10, 2026 · 2:00 PM CDT"
    And Duration renders like "1 hr 30 min"
    And the appointment that ended yesterday is not shown

  Scenario: In-progress appointment
    Given I have an appointment with start_time 10 minutes ago and end_time in 20 minutes
    When I visit "/dashboard"
    Then it appears first with an "In progress" badge

  Scenario: Only my appointments
    Given another user has an upcoming appointment
    When I visit "/dashboard"
    Then I do not see that appointment

  Scenario: Empty state
    Given I have no upcoming or in-progress appointments
    When I visit "/dashboard"
    Then I see "No upcoming appointments" and a "Book an appointment" button linking to "/appointments/new"

  Scenario: Navigation
    When I visit "/dashboard"
    Then I see a primary "New Appointment" button linking to "/appointments/new"
    And a "Appointment History" link to "/appointments/history"

  Scenario: Unauthenticated access
    Given I am logged out
    When I visit "/dashboard"
    Then I am redirected to "/login?next=%2Fdashboard"
```

### Feature: Appointment History (HIST-1)

```gherkin
Feature: Appointment history

  Background:
    Given I am logged in

  Scenario: Past appointments, most recent first
    Given I have appointments that ended 1 day ago and 10 days ago, and one upcoming
    When I visit "/appointments/history"
    Then I see 2 rows, the 1-day-ago one first, with the same 4 columns as the dashboard
    And the upcoming appointment is not shown

  Scenario: Boundary — appointment ending exactly now
    Given an appointment whose end_time equals now
    Then it appears in history and not on the dashboard

  Scenario: Pagination
    Given I have 45 past appointments
    When I visit "/appointments/history"
    Then I see 20 rows and a "Load more" button
    When I click "Load more" twice
    Then I see 45 rows and no "Load more" button

  Scenario: Empty state
    Given I have no past appointments
    When I visit "/appointments/history"
    Then I see "No past appointments yet" and a link back to "/dashboard"
```

### Feature: Booking (BOOK-1, BOOK-2)

```gherkin
Feature: Book a new appointment

  Background:
    Given I am logged in with browser time zone "America/New_York"
    And the seeded specialists exist
    And now is "2026-03-09T13:05:00Z" (Monday 09:05 EDT)

  Scenario: Successful booking
    When I visit "/appointments/new"
    And I select Type "Planning"
    Then the Specialist list contains only "Dr. Elena Marquez — Senior Strategic Planner"
    When I select Duration "60 minutes", date "2026-03-10", and start time "10:00 AM"
    And I click "Confirm booking"
    Then an appointment row exists with start_time "2026-03-10T14:00:00Z", end_time "2026-03-10T15:00:00Z", status "BOOKED"
    And I am redirected to "/dashboard?booked=<id>"
    And a success toast reads "Appointment booked for Tue, Mar 10 · 10:00 AM EDT"

  Scenario: Duration options
    When I open the Duration selector
    Then the options are exactly "30 minutes", "60 minutes", "90 minutes", "120 minutes"
    And "60 minutes" is preselected

  Scenario: Server rejects an unsupported duration
    When I POST /api/v1/appointments with durationMinutes 45
    Then the API responds 400 with error code "VALIDATION_ERROR" on field "durationMinutes"

  Scenario: Past times are not offered and are rejected
    When I pick date "2026-03-09"
    Then no start time at or before 09:20 AM is offered
    When I POST a start_time of "2026-03-09T12:00:00Z"
    Then the API responds 422 with error code "START_IN_PAST"

  Scenario: Specialist overlap is rejected
    Given Dr. Elena Marquez has a BOOKED appointment 2026-03-10 10:00–11:00 EDT
    When I try to book her 2026-03-10 10:30 EDT for 60 minutes
    Then the API responds 409 with error code "SLOT_UNAVAILABLE"
    And the form keeps my selections, shows "That time was just taken. Pick another slot.", and refreshes available times

  Scenario: Back-to-back is allowed
    Given Dr. Elena Marquez has a BOOKED appointment 2026-03-10 10:00–11:00 EDT
    When I book her 2026-03-10 11:00 EDT for 30 minutes
    Then the booking succeeds

  Scenario: Concurrent requests for the same slot
    When two users submit the same specialist and start time within the same second
    Then exactly one request returns 201 and the other returns 409 "SLOT_UNAVAILABLE"

  Scenario: User cannot overlap themselves
    Given I have a BOOKED Training appointment 2026-03-10 10:00–11:00 EDT
    When I book Consulting 2026-03-10 10:30 EDT for 30 minutes
    Then the API responds 409 with error code "USER_CONFLICT"

  Scenario: Specialist does not offer the chosen type
    When I POST type "TRAINING" with Dr. Elena Marquez's specialistId
    Then the API responds 422 with error code "SPECIALIST_TYPE_MISMATCH"

  Scenario Outline: Outside bookable hours
    When I POST a start of <start> (specialist local time) for <minutes> minutes
    Then the API responds 422 with error code <code>

    Examples:
      | start                   | minutes | code                |
      | Sat 2026-03-14 10:00    | 30      | "OUTSIDE_HOURS"     |
      | Tue 2026-03-10 08:30    | 30      | "OUTSIDE_HOURS"     |
      | Tue 2026-03-10 16:30    | 60      | "OUTSIDE_HOURS"     |
      | Tue 2026-03-10 10:15    | 30      | "MISALIGNED_START"  |
      | 61 days from today      | 30      | "BEYOND_HORIZON"    |
```

## 3. Database Schema & Data Models

Five tables on PostgreSQL 16: `users`, `sessions`, `login_attempts`, `specialists`, `appointments`. Overlap prevention is two `EXCLUDE USING gist` constraints on `appointments`, and a composite foreign key guarantees an appointment's type matches its specialist's type. All timestamps are `timestamptz` stored in UTC.

### Relationships

- `users 1—N sessions` (cascade delete), `users 1—N appointments` (cascade delete).
- `specialists 1—N appointments` via the composite key `(specialist_id, appointment_type) → specialists(id, appointment_type)` with `ON DELETE RESTRICT`; specialists are deactivated with `is_active = false`, never deleted.
- `login_attempts` has no foreign key; it records attempts for unknown emails too.

### Migration layout

```text
drizzle/
  0000_init.sql               -- generated by drizzle-kit from schema.ts (tables, enums, FKs, checks, indexes)
  0001_constraints.sql        -- custom (drizzle-kit generate --custom): btree_gist, EXCLUDE constraints, updated_at triggers
  0002_seed_specialists.sql   -- custom: idempotent specialist seed
  meta/_journal.json
```

Migrations are applied by a dedicated Cloud Run Job (Section 7) using `drizzle-orm/node-postgres/migrator`, never at app start-up. Drizzle cannot express `EXCLUDE` constraints or triggers; drizzle-kit diffs against its own snapshots, so they are not dropped by later generated migrations, but every generated migration must be reviewed in the PR.

### Resulting DDL (source of truth)

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;  -- gist equality on uuid for EXCLUDE

CREATE TYPE appointment_type   AS ENUM ('PLANNING', 'TRAINING', 'CONSULTING');
CREATE TYPE appointment_status AS ENUM ('BOOKED', 'CANCELLED');

-- users -----------------------------------------------------------------
CREATE TABLE users (
  id            uuid         PRIMARY KEY DEFAULT gen_random_uuid(),
  first_name    varchar(100) NOT NULL CHECK (length(btrim(first_name)) > 0),
  last_name     varchar(100) NOT NULL CHECK (length(btrim(last_name)) > 0),
  email         varchar(254) NOT NULL CHECK (email = lower(btrim(email))),
  password_hash text         NOT NULL,                 -- Argon2id PHC string
  created_at    timestamptz  NOT NULL DEFAULT now(),
  updated_at    timestamptz  NOT NULL DEFAULT now(),
  CONSTRAINT users_email_key UNIQUE (email)
);

-- sessions --------------------------------------------------------------
CREATE TABLE sessions (
  id           char(64)     PRIMARY KEY,               -- hex SHA-256 of the cookie token; raw token never stored
  user_id      uuid         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at   timestamptz  NOT NULL DEFAULT now(),
  expires_at   timestamptz  NOT NULL,
  last_seen_at timestamptz  NOT NULL DEFAULT now(),
  ip_address   inet,
  user_agent   varchar(512)
);
CREATE INDEX sessions_user_id_idx    ON sessions (user_id);
CREATE INDEX sessions_expires_at_idx ON sessions (expires_at);

-- login_attempts (rate limiting; works across Cloud Run instances) ------
CREATE TABLE login_attempts (
  id           bigint       GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email        varchar(254) NOT NULL,                  -- normalized, may not exist in users
  ip_address   inet,
  succeeded    boolean      NOT NULL,
  attempted_at timestamptz  NOT NULL DEFAULT now()
);
CREATE INDEX login_attempts_email_time_idx ON login_attempts (email, attempted_at DESC);
CREATE INDEX login_attempts_ip_time_idx    ON login_attempts (ip_address, attempted_at DESC);

-- specialists -----------------------------------------------------------
CREATE TABLE specialists (
  id               uuid             PRIMARY KEY DEFAULT gen_random_uuid(),
  full_name        varchar(120)     NOT NULL,
  title            varchar(120)     NOT NULL,
  appointment_type appointment_type NOT NULL,
  bio              text             NOT NULL CHECK (length(bio) <= 1000),
  timezone         varchar(64)      NOT NULL DEFAULT 'America/New_York',  -- IANA name, validated in app
  is_active        boolean          NOT NULL DEFAULT true,
  created_at       timestamptz      NOT NULL DEFAULT now(),
  CONSTRAINT specialists_id_type_key UNIQUE (id, appointment_type)      -- target of composite FK
);
CREATE INDEX specialists_type_active_idx ON specialists (appointment_type) WHERE is_active;

-- appointments ----------------------------------------------------------
CREATE TABLE appointments (
  id               uuid               PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id          uuid               NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  specialist_id    uuid               NOT NULL,
  appointment_type appointment_type   NOT NULL,
  start_time       timestamptz        NOT NULL,
  end_time         timestamptz        NOT NULL,
  duration_minutes smallint           NOT NULL CHECK (duration_minutes IN (30, 60, 90, 120)),
  status           appointment_status NOT NULL DEFAULT 'BOOKED',
  cancelled_at     timestamptz,
  created_at       timestamptz        NOT NULL DEFAULT now(),
  updated_at       timestamptz        NOT NULL DEFAULT now(),

  CONSTRAINT appointments_specialist_type_fkey
    FOREIGN KEY (specialist_id, appointment_type)
    REFERENCES specialists (id, appointment_type) ON DELETE RESTRICT,

  -- end_time is stored (not generated) because timestamptz + interval is STABLE, not IMMUTABLE
  CONSTRAINT appointments_end_matches_duration
    CHECK (end_time - start_time = make_interval(mins => duration_minutes)),

  -- 30-minute grid, zero seconds; epoch arithmetic is independent of session TimeZone
  CONSTRAINT appointments_start_aligned
    CHECK (mod(extract(epoch FROM start_time), 1800) = 0),

  CONSTRAINT appointments_cancel_consistency
    CHECK ((status = 'CANCELLED') = (cancelled_at IS NOT NULL))
);

-- 0001_constraints.sql: concurrency-safe overlap prevention, half-open ranges so 10:00–11:00 and 11:00–11:30 do not conflict
ALTER TABLE appointments ADD CONSTRAINT appointments_no_specialist_overlap
  EXCLUDE USING gist (specialist_id WITH =, tstzrange(start_time, end_time, '[)') WITH &&)
  WHERE (status = 'BOOKED');

ALTER TABLE appointments ADD CONSTRAINT appointments_no_user_overlap
  EXCLUDE USING gist (user_id WITH =, tstzrange(start_time, end_time, '[)') WITH &&)
  WHERE (status = 'BOOKED');

-- dashboard: user_id = $1 AND end_time > now(); history: user_id = $1 AND end_time <= now()
CREATE INDEX appointments_user_end_idx ON appointments (user_id, end_time) WHERE status = 'BOOKED';

CREATE FUNCTION set_updated_at() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN NEW.updated_at := now(); RETURN NEW; END $$;

CREATE TRIGGER users_set_updated_at        BEFORE UPDATE ON users        FOR EACH ROW EXECUTE FUNCTION set_updated_at();
CREATE TRIGGER appointments_set_updated_at BEFORE UPDATE ON appointments FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

The gist exclusion constraints also serve the specialist availability query (`specialist_id = $1 AND tstzrange(...) && $range`), so no separate specialist/time index is needed.

### Constraint → API error mapping

| PostgreSQL error | Constraint | API error code | HTTP |
| --- | --- | --- | --- |
| `23P01` exclusion\_violation | `appointments_no_specialist_overlap` | `SLOT_UNAVAILABLE` | 409 |
| `23P01` exclusion\_violation | `appointments_no_user_overlap` | `USER_CONFLICT` | 409 |
| `23505` unique\_violation | `users_email_key` | `EMAIL_TAKEN` | 409 |
| `23503` foreign\_key\_violation | `appointments_specialist_type_fkey` | `SPECIALIST_TYPE_MISMATCH` | 422 |
| `23514` check\_violation | any | `VALIDATION_ERROR` (logged as a bug: app validation should have caught it) | 400 |

Map by `err.code` and `err.constraint` from `pg`'s `DatabaseError`; never by message text.

### Drizzle schema (`src/server/db/schema.ts`)

```ts
import { sql } from "drizzle-orm";
import {
  pgTable, pgEnum, uuid, varchar, text, timestamp, boolean, smallint,
  char, inet, bigint, index, unique, foreignKey, check,
} from "drizzle-orm/pg-core";

export const appointmentType   = pgEnum("appointment_type", ["PLANNING", "TRAINING", "CONSULTING"]);
export const appointmentStatus = pgEnum("appointment_status", ["BOOKED", "CANCELLED"]);

const tstz = (name: string) => timestamp(name, { withTimezone: true, mode: "date" });

export const users = pgTable("users", {
  id:           uuid("id").primaryKey().defaultRandom(),
  firstName:    varchar("first_name", { length: 100 }).notNull(),
  lastName:     varchar("last_name", { length: 100 }).notNull(),
  email:        varchar("email", { length: 254 }).notNull(),
  passwordHash: text("password_hash").notNull(),
  createdAt:    tstz("created_at").notNull().defaultNow(),
  updatedAt:    tstz("updated_at").notNull().defaultNow(),
}, (t) => [
  unique("users_email_key").on(t.email),
  check("users_email_lower", sql`${t.email} = lower(btrim(${t.email}))`),
  check("users_first_name_nonblank", sql`length(btrim(${t.firstName})) > 0`),
  check("users_last_name_nonblank", sql`length(btrim(${t.lastName})) > 0`),
]);

export const sessions = pgTable("sessions", {
  id:         char("id", { length: 64 }).primaryKey(),
  userId:     uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  createdAt:  tstz("created_at").notNull().defaultNow(),
  expiresAt:  tstz("expires_at").notNull(),
  lastSeenAt: tstz("last_seen_at").notNull().defaultNow(),
  ipAddress:  inet("ip_address"),
  userAgent:  varchar("user_agent", { length: 512 }),
}, (t) => [
  index("sessions_user_id_idx").on(t.userId),
  index("sessions_expires_at_idx").on(t.expiresAt),
]);

export const loginAttempts = pgTable("login_attempts", {
  id:          bigint("id", { mode: "number" }).primaryKey().generatedAlwaysAsIdentity(),
  email:       varchar("email", { length: 254 }).notNull(),
  ipAddress:   inet("ip_address"),
  succeeded:   boolean("succeeded").notNull(),
  attemptedAt: tstz("attempted_at").notNull().defaultNow(),
}, (t) => [
  index("login_attempts_email_time_idx").on(t.email, t.attemptedAt.desc()),
  index("login_attempts_ip_time_idx").on(t.ipAddress, t.attemptedAt.desc()),
]);

export const specialists = pgTable("specialists", {
  id:              uuid("id").primaryKey().defaultRandom(),
  fullName:        varchar("full_name", { length: 120 }).notNull(),
  title:           varchar("title", { length: 120 }).notNull(),
  appointmentType: appointmentType("appointment_type").notNull(),
  bio:             text("bio").notNull(),
  timezone:        varchar("timezone", { length: 64 }).notNull().default("America/New_York"),
  isActive:        boolean("is_active").notNull().default(true),
  createdAt:       tstz("created_at").notNull().defaultNow(),
}, (t) => [
  unique("specialists_id_type_key").on(t.id, t.appointmentType),
  index("specialists_type_active_idx").on(t.appointmentType).where(sql`${t.isActive}`),
  check("specialists_bio_length", sql`length(${t.bio}) <= 1000`),
]);

export const appointments = pgTable("appointments", {
  id:              uuid("id").primaryKey().defaultRandom(),
  userId:          uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  specialistId:    uuid("specialist_id").notNull(),
  appointmentType: appointmentType("appointment_type").notNull(),
  startTime:       tstz("start_time").notNull(),
  endTime:         tstz("end_time").notNull(),
  durationMinutes: smallint("duration_minutes").notNull(),
  status:          appointmentStatus("status").notNull().default("BOOKED"),
  cancelledAt:     tstz("cancelled_at"),
  createdAt:       tstz("created_at").notNull().defaultNow(),
  updatedAt:       tstz("updated_at").notNull().defaultNow(),
}, (t) => [
  foreignKey({
    name: "appointments_specialist_type_fkey",
    columns: [t.specialistId, t.appointmentType],
    foreignColumns: [specialists.id, specialists.appointmentType],
  }).onDelete("restrict"),
  check("appointments_duration_allowed", sql`${t.durationMinutes} IN (30, 60, 90, 120)`),
  check("appointments_end_matches_duration",
    sql`${t.endTime} - ${t.startTime} = make_interval(mins => ${t.durationMinutes})`),
  check("appointments_start_aligned", sql`mod(extract(epoch FROM ${t.startTime}), 1800) = 0`),
  check("appointments_cancel_consistency",
    sql`(${t.status} = 'CANCELLED') = (${t.cancelledAt} IS NOT NULL)`),
  index("appointments_user_end_idx").on(t.userId, t.endTime).where(sql`${t.status} = 'BOOKED'`),
  // EXCLUDE constraints + triggers: drizzle/0001_constraints.sql (not expressible in Drizzle)
]);

export type User        = typeof users.$inferSelect;
export type Specialist  = typeof specialists.$inferSelect;
export type Appointment = typeof appointments.$inferSelect;
export type AppointmentType = (typeof appointmentType.enumValues)[number];
```

### Seed data: the three specialists (`drizzle/0002_seed_specialists.sql`)

Fixed UUIDs make the seed idempotent and let tests reference specialists by ID. One specialist per appointment type.

| ID | Name | Title | Type | Time zone |
| --- | --- | --- | --- | --- |
| `3f1c2a9e-8b4d-4c1e-9a7f-2d5e6b8c0a11` | Dr. Elena Marquez | Senior Strategic Planner | PLANNING | America/New\_York |
| `8a2d4f6b-1c3e-4f5a-8b7c-9d0e1f2a3b22` | Marcus Chen | Lead Learning & Development Trainer | TRAINING | America/New\_York |
| `c5e7a9b1-d3f5-4a7c-9e1b-3d5f7a9c1e33` | Priya Raman | Principal Management Consultant | CONSULTING | America/New\_York |

```sql
INSERT INTO specialists (id, full_name, title, appointment_type, bio, timezone) VALUES
(
  '3f1c2a9e-8b4d-4c1e-9a7f-2d5e6b8c0a11',
  'Dr. Elena Marquez',
  'Senior Strategic Planner',
  'PLANNING',
  'Fifteen years building multi-year operating plans for mid-market logistics and healthcare companies. Former director of strategy at a national distributor and holds a PhD in operations research. Focuses on quarterly roadmaps, capacity planning, and OKR design.',
  'America/New_York'
),
(
  '8a2d4f6b-1c3e-4f5a-8b7c-9d0e1f2a3b22',
  'Marcus Chen',
  'Lead Learning & Development Trainer',
  'TRAINING',
  'Designs hands-on workshops in data literacy, spreadsheet modelling, SQL, and project-management tooling. Has trained teams from 5 to 200 people over 10+ years and is an ATD Certified Professional in Talent Development (CPTD).',
  'America/New_York'
),
(
  'c5e7a9b1-d3f5-4a7c-9e1b-3d5f7a9c1e33',
  'Priya Raman',
  'Principal Management Consultant',
  'CONSULTING',
  'Former engagement manager at a global strategy consultancy with 12 years across retail, fintech, and the public sector. Advises on process redesign, vendor selection, and cost reduction programmes. MBA, and a certified Lean Six Sigma Black Belt.',
  'America/New_York'
)
ON CONFLICT (id) DO UPDATE SET
  full_name = EXCLUDED.full_name,
  title     = EXCLUDED.title,
  bio       = EXCLUDED.bio,
  timezone  = EXCLUDED.timezone;
```

`appointment_type` is deliberately excluded from the `DO UPDATE` list: changing a specialist's type would orphan the composite foreign key on existing appointments, so a type change requires a new specialist row.

## 4. API Endpoints & Server Actions

UI mutations use four Server Actions; reads in Server Components call the service layer directly; a versioned REST API under `/api/v1` exposes the same services for tests and integrations. All three surfaces share one Zod module and one service layer, so validation and error codes are identical everywhere.

### Conventions

- JSON bodies, camelCase keys, `Content-Type: application/json`.
- Timestamps in responses are ISO 8601 UTC with `Z` (`2026-03-10T14:00:00.000Z`). Requests accept ISO 8601 with an explicit offset (`2026-03-10T10:00:00-04:00`); timestamps without an offset are rejected.
- Authentication: the `__Host-session` cookie. Every endpoint except `register`, `login`, `healthz`, and `readyz` returns 401 `UNAUTHENTICATED` without a valid session.
- CSRF: the cookie is `SameSite=Lax`, and every non-GET Route Handler rejects requests whose `Origin` header is absent or differs from `APP_ORIGIN` with 403 `CSRF_REJECTED`. Server Actions get Next.js's built-in Origin/Host check.
- Client IP for rate limiting comes from `X-Forwarded-For`, counting `TRUSTED_PROXY_HOPS` entries from the right (1 when served directly on `run.app`, 2 behind an external Application Load Balancer).
- Resources owned by another user return 404, never 403, so IDs cannot be probed.

### Error envelope

```ts
type ApiError = {
  error: {
    code: ErrorCode;                          // stable, machine-readable
    message: string;                          // safe to show the user
    fieldErrors?: Record<string, string[]>;   // only for VALIDATION_ERROR
    requestId: string;                        // echoes X-Cloud-Trace-Context trace id
  };
};
```

| Code | HTTP | Raised when | User-facing message |
| --- | --- | --- | --- |
| `VALIDATION_ERROR` | 400 | Body or query fails the Zod schema | Per-field messages |
| `UNAUTHENTICATED` | 401 | Missing, unknown, or expired session | "Please log in to continue." |
| `INVALID_CREDENTIALS` | 401 | Unknown email or wrong password | "Invalid email or password." |
| `CSRF_REJECTED` | 403 | Origin check fails | "Request blocked. Refresh the page and try again." |
| `NOT_FOUND` | 404 | Appointment not found or not owned | "Appointment not found." |
| `SPECIALIST_NOT_FOUND` | 404 | Specialist missing or `is_active = false` | "That specialist is no longer available." |
| `EMAIL_TAKEN` | 409 | `users_email_key` violated | "An account with this email already exists." |
| `SLOT_UNAVAILABLE` | 409 | `appointments_no_specialist_overlap` violated | "That time was just taken. Pick another slot." |
| `USER_CONFLICT` | 409 | `appointments_no_user_overlap` violated | "You already have an appointment at that time." |
| `START_IN_PAST` | 422 | `startTime < now + BOOKING_MIN_LEAD_MINUTES` | "Choose a time in the future." |
| `MISALIGNED_START` | 422 | Start not on a :00 or :30 boundary | "Start times must be on the hour or half hour." |
| `OUTSIDE_HOURS` | 422 | Outside the specialist's bookable hours | "That time is outside the specialist's hours." |
| `BEYOND_HORIZON` | 422 | Start more than `BOOKING_HORIZON_DAYS` ahead | "You can book up to 60 days ahead." |
| `SPECIALIST_TYPE_MISMATCH` | 422 | Specialist does not offer the chosen type | "This specialist doesn't offer that appointment type." |
| `RATE_LIMITED` | 429 | Login/registration throttle hit; sets `Retry-After` | "Too many attempts. Try again in a few minutes." |
| `INTERNAL` | 500 | Unhandled error (logged with stack, never returned) | "Something went wrong. Please try again." |

### Shared validation (`src/lib/validation.ts`)

```ts
import { z } from "zod";

export const APPOINTMENT_TYPES = ["PLANNING", "TRAINING", "CONSULTING"] as const;
export const DURATIONS = [30, 60, 90, 120] as const;

const name = (label: string) =>
  z.string().trim()
    .min(1, `${label} is required.`)
    .max(100, `${label} must be 100 characters or fewer.`);

export const emailSchema = z.string().trim().toLowerCase()
  .max(254, "Email must be 254 characters or fewer.")
  .email("Enter a valid email address.");

export const registerSchema = z.object({
  firstName: name("First name"),
  lastName:  name("Last name"),
  email:     emailSchema,
  password:  z.string()                         // never trimmed
    .min(12, "Password must be at least 12 characters.")
    .max(128, "Password must be 128 characters or fewer."),
  confirmPassword: z.string(),
}).refine((v) => v.password === v.confirmPassword, {
  path: ["confirmPassword"], message: "Passwords do not match.",
}).refine((v) => v.password.toLowerCase() !== v.email, {
  path: ["password"], message: "Password must not be your email address.",
});

export const loginSchema = z.object({
  email:    emailSchema,
  password: z.string().min(1, "Password is required.").max(128),
  next:     z.string().optional(),               // sanitized by safeRedirect()
});

export const createAppointmentSchema = z.object({
  appointmentType: z.enum(APPOINTMENT_TYPES),
  specialistId:    z.string().uuid(),
  startTime:       z.string().datetime({ offset: true }),
  durationMinutes: z.coerce.number().int()
    .refine((n): n is (typeof DURATIONS)[number] => (DURATIONS as readonly number[]).includes(n),
      "Duration must be 30, 60, 90, or 120 minutes."),
});

export const listAppointmentsQuery = z.object({
  scope:  z.enum(["upcoming", "past"]),
  limit:  z.coerce.number().int().min(1).max(50).default(20),
  cursor: z.string().max(200).optional(),
});

export const availabilityQuery = z.object({
  date:            z.string().regex(/^\d{4}-\d{2}-\d{2}$/),  // calendar date in `tz`
  durationMinutes: createAppointmentSchema.shape.durationMinutes,
  tz:              z.string().max(64).refine(isValidIanaZone, "Unknown time zone."),
});
```

`safeRedirect(next)` returns `next` only if it starts with `/`, does not start with `//` or `/\`, and parses as same-origin; otherwise `/dashboard`.

### Response DTOs

```ts
type UserDTO = { id: string; firstName: string; lastName: string; email: string };

type SpecialistDTO = {
  id: string; fullName: string; title: string;
  appointmentType: "PLANNING" | "TRAINING" | "CONSULTING";
  bio: string; timezone: string;                // IANA
};

type AppointmentDTO = {
  id: string;
  appointmentType: "PLANNING" | "TRAINING" | "CONSULTING";
  startTime: string;                            // ISO UTC
  endTime: string;                              // ISO UTC
  durationMinutes: 30 | 60 | 90 | 120;
  status: "BOOKED" | "CANCELLED";
  phase: "UPCOMING" | "IN_PROGRESS" | "PAST";   // computed server-side at response time
  specialist: { id: string; fullName: string; title: string };
};

type SlotDTO = { startTime: string; endTime: string; available: boolean };
```

### REST endpoints

| Method | Path | Auth | Success | Errors |
| --- | --- | --- | --- | --- |
| POST | `/api/v1/auth/register` | none | 201 `{ user: UserDTO }` + `Set-Cookie` | 400, 403, 409 `EMAIL_TAKEN`, 429 |
| POST | `/api/v1/auth/login` | none | 200 `{ user: UserDTO }` + `Set-Cookie` | 400, 401 `INVALID_CREDENTIALS`, 403, 429 |
| POST | `/api/v1/auth/logout` | optional | 204, cookie cleared (idempotent) | 403 |
| GET | `/api/v1/auth/session` | session | 200 `{ user: UserDTO, expiresAt }` | 401 |
| GET | `/api/v1/specialists?type=PLANNING` | session | 200 `{ specialists: SpecialistDTO[] }` (active only; `type` optional) | 400, 401 |
| GET | `/api/v1/specialists/{id}/availability?date=2026-03-10&durationMinutes=60&tz=America/New_York` | session | 200 `{ specialistId, date, tz, durationMinutes, slots: SlotDTO[] }` | 400, 401, 404 `SPECIALIST_NOT_FOUND` |
| GET | `/api/v1/appointments?scope=upcoming\|past&limit=20&cursor=…` | session | 200 `{ appointments: AppointmentDTO[], nextCursor: string \| null }` | 400, 401 |
| GET | `/api/v1/appointments/{id}` | session | 200 `{ appointment: AppointmentDTO }` | 401, 404 |
| POST | `/api/v1/appointments` | session | 201 `{ appointment: AppointmentDTO }` + `Location` | 400, 401, 403, 404, 409, 422 |
| GET | `/api/healthz` | none | 200 `{ status: "ok" }` (no DB call; liveness) | — |
| GET | `/api/readyz` | none | 200 `{ status: "ok" }` after `SELECT 1` within 2 s | 503 `{ status: "unavailable" }` |

### Endpoint details

**POST `/api/v1/auth/register`**

```json
// request
{ "firstName": "Jordan", "lastName": "Ellis", "email": "Jordan@Example.com",
  "password": "correct-horse-battery", "confirmPassword": "correct-horse-battery" }
// 201
{ "user": { "id": "0f7c…", "firstName": "Jordan", "lastName": "Ellis", "email": "jordan@example.com" } }
```

Hashes with Argon2id, inserts the user, creates a session, and sets the cookie in one request. Throttled to 10 registrations per IP per hour.

**POST `/api/v1/auth/login`**

```json
// request
{ "email": "jordan@example.com", "password": "correct-horse-battery" }
// 200
{ "user": { "id": "0f7c…", "firstName": "Jordan", "lastName": "Ellis", "email": "jordan@example.com" } }
// 401
{ "error": { "code": "INVALID_CREDENTIALS", "message": "Invalid email or password.", "requestId": "4bf92f35…" } }
```

Order of operations: (1) count failed `login_attempts` in the last 15 minutes for the email (limit 5) and for the IP (limit 20) → 429 with `Retry-After`; (2) look up the user; if absent, verify against a constant dummy Argon2id hash so timing matches; (3) record the attempt; (4) on success, rotate: create a new session (never reuse a pre-login token).

**Set-Cookie (register and login)**

```text
Set-Cookie: __Host-session=<43-char base64url of 32 random bytes>; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=2592000
```

**GET `/api/v1/specialists/{id}/availability`**

```json
// 200 — date and tz are the viewer's; slots are generated in the specialist's zone and filtered to that local date
{
  "specialistId": "3f1c2a9e-8b4d-4c1e-9a7f-2d5e6b8c0a11",
  "date": "2026-03-10", "tz": "America/New_York", "durationMinutes": 60,
  "slots": [
    { "startTime": "2026-03-10T13:00:00.000Z", "endTime": "2026-03-10T14:00:00.000Z", "available": true },
    { "startTime": "2026-03-10T13:30:00.000Z", "endTime": "2026-03-10T14:30:00.000Z", "available": false }
  ]
}
```

A slot is listed when it satisfies every rule in Section 6 except conflicts; `available` is false when it overlaps a `BOOKED` appointment of the specialist or of the requesting user. Response header `Cache-Control: no-store`.

**GET `/api/v1/appointments`**

| `scope` | Filter (`status = 'BOOKED'`, `user_id = session user`) | Order | Cursor (opaque base64url JSON) |
| --- | --- | --- | --- |
| `upcoming` | `end_time > now()` (includes in-progress) | `start_time ASC, id ASC` | `(start_time, id) > (:t, :id)` |
| `past` | `end_time <= now()` | `end_time DESC, id DESC` | `(end_time, id) < (:t, :id)` |

Each row joins `specialists` for `fullName` and `title`. A malformed cursor returns 400.

**POST `/api/v1/appointments`**

```json
// request
{ "appointmentType": "PLANNING", "specialistId": "3f1c2a9e-8b4d-4c1e-9a7f-2d5e6b8c0a11",
  "startTime": "2026-03-10T10:00:00-04:00", "durationMinutes": 60 }
// 201, Location: /api/v1/appointments/9d1e…
{ "appointment": {
    "id": "9d1e…", "appointmentType": "PLANNING",
    "startTime": "2026-03-10T14:00:00.000Z", "endTime": "2026-03-10T15:00:00.000Z",
    "durationMinutes": 60, "status": "BOOKED", "phase": "UPCOMING",
    "specialist": { "id": "3f1c2a9e-…", "fullName": "Dr. Elena Marquez", "title": "Senior Strategic Planner" } } }
// 409
{ "error": { "code": "SLOT_UNAVAILABLE", "message": "That time was just taken. Pick another slot.", "requestId": "4bf92f35…" } }
```

Service algorithm (`createAppointment(userId, input, now)`):

1. Parse with `createAppointmentSchema` → 400.
2. Load the specialist `WHERE id = $1 AND is_active` → 404 `SPECIALIST_NOT_FOUND`.
3. `specialist.appointmentType !== input.appointmentType` → 422 `SPECIALIST_TYPE_MISMATCH`.
4. Convert `startTime` to a UTC `Date`; compute `endTime = start + durationMinutes`.
5. Business rules in this order, first failure wins: `START_IN_PAST`, `MISALIGNED_START`, `BEYOND_HORIZON`, `OUTSIDE_HOURS` (Section 6).
6. Single `INSERT … RETURNING` with no prior SELECT; the exclusion constraints decide conflicts atomically. Map `23P01` by constraint name (Section 3).
7. Duplicate-submit replay: on `USER_CONFLICT`, if the conflicting row has the same user, specialist, start, and duration and was created within the last 10 seconds, return 201 with that row instead of an error.

### Server Actions

All actions live in `src/app/**/actions.ts` with `"use server"`, take `(prevState: ActionState, formData: FormData)` for use with React's `useActionState`, re-validate the session themselves (never trust the page that rendered the form), and call the same services as the REST handlers.

```ts
type ActionState =
  | { status: "idle" }
  | { status: "error"; code: ErrorCode; message: string;
      fieldErrors?: Record<string, string[]>;
      values?: Record<string, string> };       // echoed back, passwords excluded
```

| Action | File | Input fields | On success | On failure |
| --- | --- | --- | --- | --- |
| `registerAction` | `src/app/(auth)/register/actions.ts` | `firstName`, `lastName`, `email`, `password`, `confirmPassword` | Set cookie, `redirect("/dashboard")` | `ActionState` error |
| `loginAction` | `src/app/(auth)/login/actions.ts` | `email`, `password`, `next` | Set cookie, `redirect(safeRedirect(next))` | `ActionState` error |
| `logoutAction` | `src/app/(app)/actions.ts` | none | Delete session row, clear cookie, `redirect("/login")` | Always succeeds |
| `createAppointmentAction` | `src/app/(app)/appointments/new/actions.ts` | `appointmentType`, `specialistId`, `startTime`, `durationMinutes` | `revalidatePath("/dashboard")`, `redirect("/dashboard?booked=<id>")` | `ActionState` error; on 409 the client refetches availability |

### Service layer (`src/server/services`)

```ts
// auth.ts
register(input: RegisterInput, ctx: RequestCtx): Promise<{ user: UserDTO; token: string }>;
login(input: LoginInput, ctx: RequestCtx): Promise<{ user: UserDTO; token: string }>;
logout(token: string): Promise<void>;
getSession(token: string | undefined, now: Date): Promise<{ user: UserDTO; expiresAt: Date } | null>;

// specialists.ts
listSpecialists(type?: AppointmentType): Promise<SpecialistDTO[]>;
getAvailability(userId: string, specialistId: string, q: AvailabilityQuery, now: Date): Promise<SlotDTO[]>;

// appointments.ts
listAppointments(userId: string, q: ListAppointmentsQuery, now: Date):
  Promise<{ appointments: AppointmentDTO[]; nextCursor: string | null }>;
getAppointment(userId: string, id: string, now: Date): Promise<AppointmentDTO>;   // throws NOT_FOUND
createAppointment(userId: string, input: CreateAppointmentInput, now: Date): Promise<AppointmentDTO>;

// errors.ts
class DomainError extends Error { constructor(public code: ErrorCode, public status: number, public fieldErrors?: Record<string, string[]>) { super(code); } }
```

`now` is injected so tests can pin time; production passes `new Date()`. Files under `src/server/` start with `import "server-only";`.

## 5. UI/UX Wireframes & Component Hierarchy

Six page routes in two route groups: `(auth)` for logged-out pages and `(app)` for the authenticated shell. Lists render as Server Components; only the forms, the booking slot picker, and "Load more" are Client Components.

### Page routes

| Route | File | Auth | Behaviour |
| --- | --- | --- | --- |
| `/` | `app/page.tsx` | any | Redirects to `/dashboard` if a session exists, else `/login` |
| `/login` | `app/(auth)/login/page.tsx` | logged out | Reads `next` and `reason` search params; logged-in users redirect to `/dashboard` |
| `/register` | `app/(auth)/register/page.tsx` | logged out | Logged-in users redirect to `/dashboard` |
| `/dashboard` | `app/(app)/dashboard/page.tsx` | required | Upcoming + in-progress appointments; reads `booked` param for the success toast |
| `/appointments/history` | `app/(app)/appointments/history/page.tsx` | required | Past appointments, 20 per page |
| `/appointments/new` | `app/(app)/appointments/new/page.tsx` | required | Booking form |

Route protection is two layers. `middleware.ts` (`proxy.ts` on Next.js 16+) matches `/dashboard` and `/appointments/:path*` and redirects to `/login?next=<path>` when the cookie is absent; it does not touch the database. `app/(app)/layout.tsx` calls `requireUser()`, which validates the session against the database and redirects to `/login?next=<path>&reason=expired` when the cookie exists but the session is invalid. Every Server Action and Route Handler validates the session again.

### Wireframe: Login and Register

Centered card, max width 420 px. Login: Email, Password, "Log in" button, link "Create an account". Register: First Name and Last Name side by side (stacked under 640 px), Email, Password with a show/hide toggle and the hint "At least 12 characters", Confirm Password, "Create account" button, link "Already have an account? Log in". A form-level error banner sits above the first field; field errors sit under each field. `reason=expired` shows an info banner: "Your session expired. Please log in again."

### Wireframe: Dashboard

```text
┌───────────────────────────────────────────────────────────────────────────────┐
│ ◆ Appointments      Dashboard   Appointment History             Jordan Ellis ▾ │
├───────────────────────────────────────────────────────────────────────────────┤
│ Current Active Appointments                              [ + New Appointment ] │
│ Times shown in CDT (America/Chicago)                                           │
│ ┌─────────────────────────────────┬────────────┬──────────────────┬───────────┐│
│ │ Date & Time                     │ Type       │ Specialist       │ Duration  ││
│ ├─────────────────────────────────┼────────────┼──────────────────┼───────────┤│
│ │ Mon, Mar 9 · 8:00–9:00 AM       │ Training   │ Marcus Chen      │ 1 hr      ││
│ │ ● In progress                   │            │ L&D Trainer      │           ││
│ ├─────────────────────────────────┼────────────┼──────────────────┼───────────┤│
│ │ Tue, Mar 10 · 1:00–2:30 PM      │ Planning   │ Dr. Elena Marquez│ 1 hr 30min││
│ │                                 │            │ Strategic Planner│           ││
│ └─────────────────────────────────┴────────────┴──────────────────┴───────────┘│
│                                               View appointment history →       │
└───────────────────────────────────────────────────────────────────────────────┘
```

Under 640 px the table becomes stacked cards (date/time as the card title, type badge, specialist, duration). Empty state: an illustration-free panel reading "No upcoming appointments" with the sub-line "Book a Planning, Training, or Consulting session." and a primary "Book an appointment" button. The header "New Appointment" button stays visible in every state.

### Wireframe: Appointment History

Same shell and the same four-column table, titled "Appointment History", sorted most recent first, with no phase badge. A "Load more" button under the table appends 20 rows and disappears when `nextCursor` is null. Empty state: "No past appointments yet" with a link "Back to dashboard".

### Wireframe: New Appointment

```text
┌ Book an appointment ─────────────────────────────┬ Summary ───────────────┐
│ 1  Appointment type                              │ Planning               │
│    ( Planning )  ( Training )  ( Consulting )    │ Dr. Elena Marquez      │
│ 2  Specialist                                    │ Tue, Mar 10, 2026      │
│    ◉ Dr. Elena Marquez — Senior Strategic Planner │ 10:00–11:00 AM EDT     │
│      Fifteen years building multi-year…  [more]  │ 1 hr                   │
│ 3  Duration    [ 30 ][ 60 ][ 90 ][ 120 ] min     │                        │
│ 4  Date        ‹ March 2026 ›  (weekends, past   │ [  Confirm booking  ]  │
│                 and >60 days ahead disabled)     │                        │
│ 5  Start time  [9:00][9:30][10:00][10:30]…       │                        │
│                booked slots disabled + "Booked"  │                        │
└──────────────────────────────────────────────────┴────────────────────────┘
```

The form asks for type, specialist, and duration before date and time, because available start times depend on all three; this is a deliberate reordering of the four inputs in the requirements. Under 1024 px the summary moves below the form and the confirm button becomes sticky at the bottom of the viewport.

Field behaviour:

1. **Type** — radio cards, no default. Changing it resets specialist and start time.
2. **Specialist** — radio cards filtered to the chosen type, each with name, title, and a 2-line bio with "more". Auto-selected when exactly one specialist matches (true for the v1 seed).
3. **Duration** — segmented control with exactly 30, 60, 90, 120; default 60. Changing it refetches slots and clears the start time if it is no longer available.
4. **Date** — calendar; disables dates before today, Saturdays, Sundays, and dates beyond the 60-day horizon. Default: the first date with at least one available slot.
5. **Start time** — grid of slot buttons from the availability endpoint, labelled in the viewer's zone; unavailable slots are disabled and labelled "Booked". States: loading (6 skeleton chips), empty ("No times left on this day. Try another date."), error ("Couldn't load times." + Retry).

Confirm is disabled until all five are set and while submitting. On 409 the form keeps all selections, clears only the start time, shows the error above the slot grid, and refetches slots.

### Component hierarchy

```text
src/
├─ middleware.ts                      # cookie-presence redirect (proxy.ts on Next 16+)
├─ app/
│  ├─ layout.tsx                      # <html>, fonts, <Toaster/>, <TimezoneCookie/>
│  ├─ page.tsx                        # redirect to /dashboard or /login
│  ├─ (auth)/
│  │  ├─ layout.tsx                   # <CenteredCard>; redirect if logged in
│  │  ├─ login/{page.tsx, actions.ts}       → <LoginForm next reason/>
│  │  └─ register/{page.tsx, actions.ts}    → <RegisterForm/>
│  ├─ (app)/
│  │  ├─ layout.tsx                   # requireUser() → <AppShell user>{children}</AppShell>
│  │  ├─ actions.ts                   # logoutAction
│  │  ├─ error.tsx                    # <ErrorPanel reset/>
│  │  ├─ dashboard/{page.tsx, loading.tsx}
│  │  │    → <PageHeader title cta/> <BookedToast/> <AppointmentTable variant="upcoming"/> | <EmptyState/>
│  │  └─ appointments/
│  │     ├─ history/{page.tsx, loading.tsx}
│  │     │    → <PageHeader/> <AppointmentTable variant="past"/> <LoadMore cursor/> | <EmptyState/>
│  │     └─ new/{page.tsx, actions.ts}
│  │          → <BookingForm specialists/>
│  └─ api/
│     ├─ v1/auth/{register,login,logout,session}/route.ts
│     ├─ v1/specialists/route.ts, v1/specialists/[id]/availability/route.ts
│     ├─ v1/appointments/route.ts, v1/appointments/[id]/route.ts
│     └─ healthz/route.ts, readyz/route.ts
├─ components/
│  ├─ app-shell.tsx                   # <NavBar/> <UserMenu/> <LogoutButton/>
│  ├─ appointments/
│  │  ├─ appointment-table.tsx        # server; <AppointmentRow/> ≥640px, <AppointmentCard/> <640px
│  │  ├─ phase-badge.tsx, type-badge.tsx
│  │  ├─ empty-state.tsx
│  │  ├─ load-more.tsx                # client
│  │  └─ booked-toast.tsx             # client; reads ?booked, then router.replace("/dashboard")
│  ├─ booking/                        # all client
│  │  ├─ booking-form.tsx             # useReducer + useActionState
│  │  ├─ type-selector.tsx, specialist-picker.tsx, duration-selector.tsx
│  │  ├─ date-picker.tsx, time-slot-grid.tsx, booking-summary.tsx
│  │  └─ use-availability.ts          # fetch + AbortController
│  ├─ auth/{login-form.tsx, register-form.tsx, password-field.tsx}
│  ├─ timezone-cookie.tsx             # client; sets `tz` cookie, router.refresh() once if it changed
│  └─ ui/                             # shadcn/ui primitives
├─ lib/{validation.ts, format.ts, time.ts, safe-redirect.ts, errors.ts}
└─ server/
   ├─ db/{client.ts, schema.ts}
   ├─ auth/{session.ts, password.ts, cookies.ts, rate-limit.ts, require-user.ts}
   └─ services/{auth.ts, specialists.ts, appointments.ts, availability.ts}
```

### State handling

| Concern | Owner | Mechanism |
| --- | --- | --- |
| Current user | Server | `getCurrentUser()` wrapped in React `cache()` so layout and page share one DB lookup per request; passed to `<AppShell>` as props |
| Auth and booking form submission | Client | `useActionState(action, { status: "idle" })`; `useFormStatus().pending` disables submit and shows a spinner |
| Field validation | Both | Same Zod schema validates on blur and submit in the browser; the server result is authoritative and returned as `fieldErrors` |
| Booking selections | Client | `useReducer` over `{ type, specialistId, durationMinutes, date, startTime }` with the reset rules above |
| Available slots | Client | `useAvailability(specialistId, date, duration, tz)` fetches the availability endpoint, aborts stale requests, refetches on window focus and after a 409 |
| Dashboard and history lists | Server | Server Components call services directly; pages are dynamic because they read cookies; `createAppointmentAction` calls `revalidatePath("/dashboard")` |
| History pagination | Client | `<LoadMore>` fetches `/api/v1/appointments?scope=past&cursor=…` and appends rows |
| Booking success feedback | URL | `?booked=<id>` triggers a toast, then the param is stripped with `router.replace` |
| Display time zone | Cookie `tz` | Set by `<TimezoneCookie>` from `Intl.DateTimeFormat().resolvedOptions().timeZone`; server falls back to `APP_DEFAULT_TZ` and always prints the zone abbreviation |

### Formatting and accessibility rules

- Date & Time: `Tue, Mar 10, 2026 · 10:00–11:00 AM EDT` via `Intl.DateTimeFormat` with an explicit `timeZone`; never `toLocaleString()` without one.
- Duration: 30 → "30 min", 60 → "1 hr", 90 → "1 hr 30 min", 120 → "2 hr".
- Type badges: Planning, Training, Consulting in Title Case, each with a distinct colour and an icon so colour is not the only cue.
- WCAG 2.2 AA: every input has a visible `<label>`; errors are linked with `aria-describedby` and announced through an `aria-live="polite"` region; focus moves to the first invalid field on failed submit; the slot grid is a `role="radiogroup"` navigable with arrow keys; tables have a `<caption>` and `<th scope="col">`; minimum target size 24×24 px.

## 6. Edge Cases & Business Logic Rules

All times are stored in UTC, bookable hours are evaluated in the specialist's time zone, and everything is displayed in the viewer's zone with its abbreviation. The rules below are the complete set the service layer enforces; anything not listed is allowed.

### Booking rules (configurable)

| Rule | v1 value | Env var | Error |
| --- | --- | --- | --- |
| Allowed durations | 30, 60, 90, 120 min | fixed in code | `VALIDATION_ERROR` |
| Start-time grid | Every 30 min, aligned to UTC epoch (:00/:30 in any whole- or half-hour zone) | fixed | `MISALIGNED_START` |
| Minimum lead time | Start ≥ now + 15 min | `BOOKING_MIN_LEAD_MINUTES=15` | `START_IN_PAST` |
| Booking horizon | Start ≤ now + 60 days | `BOOKING_HORIZON_DAYS=60` | `BEYOND_HORIZON` |
| Bookable days | Monday–Friday, specialist local | `BOOKING_DAYS=1,2,3,4,5` (ISO weekday) | `OUTSIDE_HOURS` |
| Bookable hours | Local start ≥ 09:00 and local end ≤ 17:00, same local day | `BOOKING_DAY_START=09:00`, `BOOKING_DAY_END=17:00` | `OUTSIDE_HOURS` |
| Specialist conflict | No overlap with the specialist's `BOOKED` rows | DB constraint | `SLOT_UNAVAILABLE` |
| User conflict | No overlap with the user's own `BOOKED` rows, any specialist | DB constraint | `USER_CONFLICT` |
| Type match | Specialist's type equals the chosen type | DB composite FK + service check | `SPECIALIST_TYPE_MISMATCH` |

The user-conflict rule is an addition to the stated requirements: a person cannot attend two sessions at once, and it also absorbs accidental double submits.

### Time classification

| Phase | Condition (`now` = server time) | Shown on |
| --- | --- | --- |
| `UPCOMING` | `start_time > now` | Dashboard |
| `IN_PROGRESS` | `start_time <= now < end_time` | Dashboard, first, with "In progress" badge |
| `PAST` | `end_time <= now` | History |

The requirements define the dashboard as `start_time > now` and history as `end_time < now`, which leaves in-progress appointments on neither page and an appointment ending exactly at `now` on neither. This spec closes both gaps: dashboard = `end_time > now`, history = `end_time <= now`. Every appointment is on exactly one page at any instant.

One `now` value is taken per request in the service layer and passed as a SQL parameter, so the list filter and the computed `phase` never disagree. The `pg` pool connects with `options: "-c TimeZone=UTC"`.

### Time zones and DST

- **Input:** the API accepts ISO 8601 with an explicit offset and rejects naive timestamps. The booking form submits the slot's UTC `startTime` exactly as the availability endpoint returned it, so the browser never does time-zone arithmetic.
- **Slot generation:** iterate local wall-clock times 09:00, 09:30 … (17:00 − duration) on each bookable day in the specialist's zone with `TZDate` from `@date-fns/tz`, then convert to UTC. A 09:00 slot stays 09:00 local across DST changes. Local times that do not exist (spring-forward gap) are skipped; ambiguous local times (fall-back) use the earlier instant. With 09:00–17:00 hours neither case occurs, but the generator must handle both.
- **Viewer vs specialist zone:** the availability `date` is a calendar date in the viewer's `tz`. The server computes the UTC range of that local day (23, 24, or 25 hours long), generates specialist slots for every specialist-local day overlapping it, and returns slots whose start falls inside the range. A Tokyo viewer looking at a New York specialist therefore sees evening and night-time starts on the following local date; that is correct, not a bug.
- **Specialist zones:** must be valid IANA names with offsets that are multiples of 30 minutes, so local :00/:30 starts satisfy the UTC-aligned grid check. Validated whenever a specialist is added.
- **Display:** every rendered time includes the zone abbreviation (`EDT`, `CDT`, `GMT+9`). The `tz` cookie falls back to `APP_DEFAULT_TZ=America/New_York` until the browser sets it.
- **Client clock:** the browser's clock is used only to pick the calendar's initial month; the server decides what is past.

### Overlapping slots and concurrency

- Ranges are half-open `[start, end)`: 10:00–11:00 and 11:00–11:30 do not overlap; partial, containing, contained, and identical ranges all do.
- Concurrent inserts for the same specialist and time are serialized by the exclusion constraint under the default `READ COMMITTED` isolation; exactly one succeeds and the other gets `23P01`. No `SELECT … FOR UPDATE` or advisory lock is used.
- Availability is advisory. A slot shown as available can be taken before submit; the user then sees `SLOT_UNAVAILABLE`, keeps their selections, and gets fresh slots.
- A double-click or network retry that produces the exact same booking within 10 seconds returns the original appointment (Section 4, step 7). The submit button is also disabled while pending.
- After a successful booking the action redirects (Post/Redirect/Get), so Back or Refresh cannot resubmit.
- Rows with `status = 'CANCELLED'` are excluded from both constraints, so cancellation in v1.1 frees the slot with no schema change.
- A specialist deactivated between page load and submit returns `SPECIALIST_NOT_FOUND`; their existing appointments keep displaying because lists join without the `is_active` filter.

### Session expiration

| Parameter | Value |
| --- | --- |
| Idle lifetime (`sessions.expires_at`) | 7 days, sliding |
| Renewal | When less than 3.5 days remain, `expires_at = min(now + 7 d, created_at + 30 d)`; one `UPDATE` per request at most |
| Absolute lifetime | 30 days from `created_at`, then re-login |
| Cookie `Max-Age` | 30 days; the database row, not the cookie, decides validity, so renewal never needs to rewrite the cookie (Server Components cannot set cookies) |
| Token | 32 bytes from `crypto.randomBytes`, base64url in the cookie; `sessions.id` stores its SHA-256 hex |
| Cleanup | Daily maintenance job deletes expired sessions and `login_attempts` older than 24 hours |

Behaviour on expiry:

- Page navigation with an invalid session → redirect to `/login?next=<path>&reason=expired`.
- Server Action or `fetch` returning `UNAUTHENTICATED` → the client redirects the same way. The booking form saves its selections to `sessionStorage` (`booking-draft`, no personal data) before redirecting and restores them after login if the slot is still in the future.
- Logout deletes only the current session row; other devices stay logged in.
- Login always creates a new session token; an existing cookie value is never promoted (prevents session fixation).

### Empty, loading, and failure states

| Where | Condition | What the user sees |
| --- | --- | --- |
| Dashboard | No upcoming or in-progress appointments | "No upcoming appointments" + "Book an appointment" button |
| History | No past appointments | "No past appointments yet" + link to dashboard |
| History | "Load more" request fails | Inline "Couldn't load more." + Retry; existing rows stay |
| Booking: specialist | No active specialist for the chosen type | "No specialists offer this type right now."; later steps disabled |
| Booking: slots | No available slot on the chosen date | "No times left on this day. Try another date." |
| Booking: slots | Availability request fails | "Couldn't load times." + Retry |
| Any page | Database unreachable | `error.tsx` panel "Something went wrong." + Try again; `/api/readyz` returns 503 |
| Any list | Loading | `loading.tsx` with 3 skeleton rows |

### Input and security edge cases

- Email is trimmed and lowercased before validation, storage, lookup, and rate limiting. Names are trimmed and Unicode-normalized to NFC; non-Latin names are allowed.
- Passwords are never trimmed or logged. The 128-character cap bounds Argon2id cost per request.
- Registration reveals whether an email exists (`EMAIL_TAKEN`). This is accepted for v1, which has no email verification, and is mitigated by the per-IP registration limit.
- Request bodies are capped at 16 KB for Route Handlers and via `serverActions.bodySizeLimit` for Server Actions.
- Logs never contain passwords, session tokens, or full cookies; emails are logged only as a SHA-256 prefix.

### Open questions

- Should public holidays block booking, and from which calendar?
- Should there be a cap on future appointments per user (for example 20) to stop one account reserving every slot?
- Should each specialist have their own working hours instead of the shared 09:00–17:00 Monday–Friday?

## 7. DevOps & GCP Deployment Architecture

Each merge to `main` authenticates to GCP through Workload Identity Federation with no stored keys, builds two images from one multi-stage `Dockerfile`, runs migrations as a Cloud Run Job, and only then deploys a new Cloud Run revision. The app reaches Cloud SQL exclusively through the Unix socket that Cloud Run's built-in Cloud SQL connection (a managed Cloud SQL Auth Proxy) mounts at `/cloudsql/<INSTANCE_CONNECTION_NAME>`.

&#91;embedded content: deployment architecture · CI/CD path and runtime\]

The deploy job's OIDC token is exchanged for the deployer identity, both images land in Artifact Registry, the migrate job runs against Cloud SQL, and only then does the service get a new revision; every runtime component reaches the database through the socket.

### GCP resources

| Resource | Name | Configuration |
| --- | --- | --- |
| Project | `appointments-prod` (repeat for `appointments-staging`) | Region `us-central1` |
| Artifact Registry | `appointments` (Docker) | Images `web` and `migrator`, tagged with the Git SHA; cleanup policy keeps the newest 20 |
| Cloud SQL | `appointments-pg` | PostgreSQL 16, `db-custom-1-3840` (1 vCPU, 3.75 GB), 20 GB SSD with auto-grow, daily backups kept 7 days, point-in-time recovery, maintenance Sunday 07:00 UTC, database `appointments`, user `app`. Public IP enabled with zero authorized networks, so only the IAM-authenticated Auth Proxy can connect |
| Cloud Run service | `appointments-web` | Port 8080, 1 vCPU, 512 MiB, concurrency 80, min instances 1 (prod) / 0 (staging), max 10, request timeout 30 s, execution environment gen2, startup CPU boost |
| Cloud Run job | `appointments-migrate` | `migrator` image, 1 task, 0 retries, 300 s timeout |
| Cloud Run job | `appointments-maintenance` | `migrator` image with args `scripts/maintenance.js`; triggered daily at 03:00 UTC by Cloud Scheduler |
| Secret Manager | `db-password` | Mounted as env var `DB_PASSWORD` |
| Service accounts | `github-deployer`, `appointments-runtime`, `scheduler-invoker` | Roles in the IAM table below |

Hardening option for later: switch Cloud SQL to private IP and give Cloud Run Direct VPC egress; the app code does not change because it still connects through `INSTANCE_CONNECTION_NAME`.

### Database connection (`src/server/db/client.ts`)

```ts
import "server-only";
import { Pool, type PoolConfig } from "pg";
import { drizzle } from "drizzle-orm/node-postgres";
import * as schema from "./schema";

const common: PoolConfig = {
  max: Number(process.env.DB_POOL_MAX ?? 5),
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
  options: "-c TimeZone=UTC -c statement_timeout=5000",
};

const config: PoolConfig = process.env.INSTANCE_CONNECTION_NAME
  ? { ...common,                                        // Cloud Run: Unix socket
      host: `/cloudsql/${process.env.INSTANCE_CONNECTION_NAME}`,
      user: process.env.DB_USER, password: process.env.DB_PASSWORD, database: process.env.DB_NAME }
  : { ...common, connectionString: process.env.DATABASE_URL };  // local dev and CI

const g = globalThis as unknown as { pgPool?: Pool };
export const pool = g.pgPool ?? new Pool(config);
if (process.env.NODE_ENV !== "production") g.pgPool = pool;   // survive dev hot reload
export const db = drizzle(pool, { schema });
```

Connection budget: 10 instances × 5 connections + 1 migrate + 1 maintenance = 52, under the default `max_connections` of the chosen tier. Raise `max-instances` only together with `DB_POOL_MAX` and the instance tier.

### Runtime configuration

| Variable | Source | Example |
| --- | --- | --- |
| `NODE_ENV` | Dockerfile | `production` |
| `PORT`, `HOSTNAME` | Cloud Run / Dockerfile | `8080`, `0.0.0.0` |
| `APP_ORIGIN` | env var | `https://appointments.example.com` |
| `APP_DEFAULT_TZ` | env var | `America/New_York` |
| `INSTANCE_CONNECTION_NAME` | env var | `appointments-prod:us-central1:appointments-pg` |
| `DB_NAME`, `DB_USER` | env var | `appointments`, `app` |
| `DB_PASSWORD` | Secret Manager `db-password:latest` | — |
| `DB_POOL_MAX` | env var | `5` |
| `TRUSTED_PROXY_HOPS` | env var | `1` |
| `BOOKING_MIN_LEAD_MINUTES`, `BOOKING_HORIZON_DAYS`, `BOOKING_DAYS`, `BOOKING_DAY_START`, `BOOKING_DAY_END` | env var | `15`, `60`, `1,2,3,4,5`, `09:00`, `17:00` |
| `DATABASE_URL` | local `.env` and CI only | `postgres://postgres:postgres@localhost:5432/appointments` |

All variables are parsed once at start-up by a Zod schema in `src/server/env.ts`; the process exits on an invalid value.

### `next.config.ts`

```ts
import type { NextConfig } from "next";

const securityHeaders = [
  { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  { key: "X-Frame-Options", value: "DENY" },
  { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
];

const config: NextConfig = {
  output: "standalone",
  poweredByHeader: false,
  serverExternalPackages: ["@node-rs/argon2"],          // native binary; keep out of the bundle
  experimental: { serverActions: { bodySizeLimit: "16kb" } },
  async headers() { return [{ source: "/(.*)", headers: securityHeaders }]; },
};
export default config;
```

### Multi-stage `Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1.7
# Builder and runner are both Debian/glibc so @node-rs/argon2's prebuilt binary matches.
# Pin both base images by digest in the real file.

# ---- 1. base --------------------------------------------------------------
FROM node:22-bookworm-slim AS base
WORKDIR /app
ENV NEXT_TELEMETRY_DISABLED=1

# ---- 2. deps: install from lockfile only (cached unless package*.json changes)
FROM base AS deps
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci

# ---- 3. builder: Next.js standalone build + bundled scripts -----------------
FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build \
 && npx esbuild scripts/migrate.ts scripts/maintenance.ts \
      --bundle --platform=node --target=node22 --external:pg-native --outdir=dist/scripts

# ---- 4. runner: the web app (default target) --------------------------------
FROM gcr.io/distroless/nodejs22-debian12:nonroot AS runner
WORKDIR /app
ENV NODE_ENV=production PORT=8080 HOSTNAME=0.0.0.0
COPY --from=builder --chown=nonroot:nonroot /app/.next/standalone ./
COPY --from=builder --chown=nonroot:nonroot /app/.next/static ./.next/static
COPY --from=builder --chown=nonroot:nonroot /app/public ./public
USER nonroot
EXPOSE 8080
CMD ["server.js"]                      # distroless entrypoint is node

# ---- 5. migrator: migrations + maintenance, no Next.js -------------------
FROM gcr.io/distroless/nodejs22-debian12:nonroot AS migrator
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder --chown=nonroot:nonroot /app/dist/scripts ./scripts
COPY --from=builder --chown=nonroot:nonroot /app/drizzle ./drizzle
USER nonroot
CMD ["scripts/migrate.js"]
```

`.dockerignore` must list `node_modules`, `.next`, `.git`, `.env*`, `coverage`, `e2e/`, and `playwright-report/`. The `runner` image contains no shell, no package manager, and no source code. `scripts/migrate.ts` calls `migrate(db, { migrationsFolder: "./drizzle" })` and exits non-zero on failure; `scripts/maintenance.ts` deletes expired sessions and `login_attempts` older than 24 hours.

### One-time GCP bootstrap

Run once per project by an owner. Replace `acme/appointments` with the GitHub `owner/repo`.

```bash
PROJECT_ID=appointments-prod
PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')
REGION=us-central1
GH_REPO=acme/appointments
DEPLOYER=github-deployer@$PROJECT_ID.iam.gserviceaccount.com
RUNTIME=appointments-runtime@$PROJECT_ID.iam.gserviceaccount.com

gcloud services enable run.googleapis.com sqladmin.googleapis.com artifactregistry.googleapis.com \
  secretmanager.googleapis.com iamcredentials.googleapis.com sts.googleapis.com cloudscheduler.googleapis.com

gcloud artifacts repositories create appointments --repository-format=docker --location="$REGION"
gcloud iam service-accounts create github-deployer
gcloud iam service-accounts create appointments-runtime
gcloud iam service-accounts create scheduler-invoker

# Workload Identity Federation: pool + GitHub OIDC provider, locked to one repo and branch
gcloud iam workload-identity-pools create github --location=global --display-name="GitHub Actions"
gcloud iam workload-identity-pools providers create-oidc github-oidc \
  --location=global --workload-identity-pool=github \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.ref=assertion.ref" \
  --attribute-condition="assertion.repository == '$GH_REPO' && assertion.ref == 'refs/heads/main'"

# Only that repo may impersonate the deployer
gcloud iam service-accounts add-iam-policy-binding "$DEPLOYER" \
  --role=roles/iam.workloadIdentityUser \
  --member="principalSet://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github/attribute.repository/$GH_REPO"

# Deployer permissions
gcloud artifacts repositories add-iam-policy-binding appointments --location="$REGION" \
  --member="serviceAccount:$DEPLOYER" --role=roles/artifactregistry.writer
gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$DEPLOYER" --role=roles/run.developer
gcloud iam service-accounts add-iam-policy-binding "$RUNTIME" \
  --member="serviceAccount:$DEPLOYER" --role=roles/iam.serviceAccountUser

# Runtime permissions
gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$RUNTIME" --role=roles/cloudsql.client
printf '%s' "$(openssl rand -base64 32)" | gcloud secrets create db-password --data-file=-
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:$RUNTIME" --role=roles/secretmanager.secretAccessor
# Then: create the Cloud SQL instance, database `appointments`, and user `app` with that password.
# After the first deploy, make the service public once (CI's run.developer role cannot set IAM policy):
#   gcloud run services add-iam-policy-binding appointments-web --region=$REGION --member=allUsers --role=roles/run.invoker
```

### IAM (least privilege)

| Principal | Role | Scope | Why |
| --- | --- | --- | --- |
| GitHub `principalSet` for `acme/appointments` | `roles/iam.workloadIdentityUser` | `github-deployer` SA | Federate OIDC tokens into the deployer identity |
| `github-deployer` | `roles/artifactregistry.writer` | Repo `appointments` | Push images |
| `github-deployer` | `roles/run.developer` | Project | Deploy the service, create/update and execute jobs |
| `github-deployer` | `roles/iam.serviceAccountUser` | `appointments-runtime` SA only | Attach the runtime identity to the service and jobs |
| `appointments-runtime` | `roles/cloudsql.client` | Project | Connect through the Cloud SQL socket |
| `appointments-runtime` | `roles/secretmanager.secretAccessor` | Secret `db-password` only | Read the DB password |
| `scheduler-invoker` | `roles/run.invoker` | Job `appointments-maintenance` | Trigger the daily cleanup |
| `allUsers` | `roles/run.invoker` | Service `appointments-web` | Public website (set once, manually) |

No service-account keys exist anywhere. The GitHub repository stores only two non-secret variables: the provider resource name and the deployer email.

### GitHub Actions workflow (`.github/workflows/ci-cd.yml`)

```yaml
name: ci-cd
on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ci-cd-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

env:
  PROJECT_ID: appointments-prod
  REGION: us-central1
  REGISTRY: us-central1-docker.pkg.dev/appointments-prod/appointments
  SERVICE: appointments-web
  SQL_INSTANCE: appointments-prod:us-central1:appointments-pg
  RUNTIME_SA: appointments-runtime@appointments-prod.iam.gserviceaccount.com

jobs:
  test:
    runs-on: ubuntu-24.04
    permissions: { contents: read }
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: postgres, POSTGRES_DB: appointments_test }
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U postgres" --health-interval 5s --health-timeout 5s --health-retries 10
    env:
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/appointments_test
      APP_ORIGIN: http://localhost:3000
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run lint && npm run typecheck
      - run: npm run db:migrate              # applies drizzle/*.sql incl. EXCLUDE constraints + seed
      - run: npm test -- --run               # Vitest unit + integration (real Postgres)
      - run: npx playwright install --with-deps chromium
      - run: npm run build && npm run test:e2e

  deploy:
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-24.04
    environment: production
    permissions:
      contents: read
      id-token: write                        # required to mint the OIDC token
    steps:
      - uses: actions/checkout@v4

      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.GCP_WIF_PROVIDER }}   # projects/<num>/locations/global/workloadIdentityPools/github/providers/github-oidc
          service_account: ${{ vars.GCP_DEPLOYER_SA }}

      - uses: google-github-actions/setup-gcloud@v2
      - run: gcloud auth configure-docker us-central1-docker.pkg.dev --quiet
      - uses: docker/setup-buildx-action@v3

      - name: Build and push web image
        uses: docker/build-push-action@v6
        with:
          context: .
          target: runner
          push: true
          tags: ${{ env.REGISTRY }}/web:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build and push migrator image
        uses: docker/build-push-action@v6
        with:
          context: .
          target: migrator
          push: true
          tags: ${{ env.REGISTRY }}/migrator:${{ github.sha }}
          cache-from: type=gha

      - name: Run database migrations
        run: |
          gcloud run jobs deploy appointments-migrate \
            --image="$REGISTRY/migrator:$GITHUB_SHA" --region="$REGION" \
            --service-account="$RUNTIME_SA" \
            --set-cloudsql-instances="$SQL_INSTANCE" \
            --set-env-vars="INSTANCE_CONNECTION_NAME=$SQL_INSTANCE,DB_NAME=appointments,DB_USER=app" \
            --set-secrets="DB_PASSWORD=db-password:latest" \
            --tasks=1 --max-retries=0 --task-timeout=300s
          gcloud run jobs execute appointments-migrate --region="$REGION" --wait

      - name: Deploy Cloud Run service
        id: deploy
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: ${{ env.SERVICE }}
          region: ${{ env.REGION }}
          image: ${{ env.REGISTRY }}/web:${{ github.sha }}
          env_vars: |-
            APP_ORIGIN=https://appointments.example.com
            APP_DEFAULT_TZ=America/New_York
            INSTANCE_CONNECTION_NAME=${{ env.SQL_INSTANCE }}
            DB_NAME=appointments
            DB_USER=app
            DB_POOL_MAX=5
            TRUSTED_PROXY_HOPS=1
          secrets: |-
            DB_PASSWORD=db-password:latest
          flags: >-
            --service-account=${{ env.RUNTIME_SA }}
            --set-cloudsql-instances=${{ env.SQL_INSTANCE }}
            --port=8080 --cpu=1 --memory=512Mi --concurrency=80
            --min-instances=1 --max-instances=10 --timeout=30s
            --execution-environment=gen2 --cpu-boost

      - name: Smoke test
        run: curl --fail --retry 5 --retry-delay 3 "${{ steps.deploy.outputs.url }}/api/readyz"
```

Pin third-party actions to full commit SHAs in the real file. Store `GCP_WIF_PROVIDER` and `GCP_DEPLOYER_SA` as repository variables (not secrets; neither is sensitive). Protect the `production` environment with required reviewers if deploys need a human gate.

### Release, rollback, and migration rules

1. Migrations run before the new revision takes traffic, so every migration must work with the currently serving code (expand/contract): add nullable columns or new tables first, ship code that uses them, drop old columns in a later release.
2. Migrations are forward-only. Rollback is a traffic shift to the previous revision: `gcloud run services update-traffic appointments-web --region=us-central1 --to-revisions=<previous-revision>=100`.
3. A failed migration job fails the workflow before the deploy step, leaving the old revision serving.
4. Database recovery uses Cloud SQL point-in-time recovery into a new instance; the runbook lives in `docs/runbooks/db-restore.md`.

### Observability

- Logs: one JSON object per line to stdout with `severity`, `message`, `requestId`, and `logging.googleapis.com/trace` (from `X-Cloud-Trace-Context`), so Cloud Logging groups them by request.
- Uptime check on `/api/healthz` every 1 minute from 3 regions.
- Alerts: 5xx ratio above 2% for 5 minutes; p95 latency above 1 s for 10 minutes; Cloud SQL CPU above 80% or connections above 80 for 10 minutes; any failed `appointments-migrate` or `appointments-maintenance` execution.
- Nightly audit query (in the maintenance job) asserts zero overlapping `BOOKED` rows and logs at `ERROR` if any exist.
