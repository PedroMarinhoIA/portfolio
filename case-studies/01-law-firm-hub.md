# 1. Law Firm Operations Hub

**Client:** a Brazilian law firm with 30+ lawyers across 7 practice areas (labor, corporate, tax, civil, restructuring…).
**Status:** in production since April 2026, used daily. Private repository — 757 commits, 153 database migrations, 11 Edge Functions, ~200 unit/integration test files + Playwright E2E.
**My role:** sole engineer — discovery with the partners, architecture, implementation, security, CI/CD, operations and support.

## The problem

The firm ran on spreadsheets, Trello, a legacy case-management tool and WhatsApp. Leads were lost between partners, nobody had a reliable view of who was doing what, and deadlines lived in people's heads. Everything involved data protected by **LGPD** (Brazil's GDPR) and **attorney–client privilege** (OAB rules) — a leak is not a bug, it is a professional-ethics incident.

## What I built

One platform with three pillars, plus the modules the firm asked for after go-live:

- **Commercial** — sales funnel from lead to signed engagement.
- **Operational** — Kanban of matters and tasks with a per-role transition matrix, deadlines and notifications.
- **Dashboard** — KPIs per area and per partner.
- Room booking, purchase requests, internal announcements board, personal agenda, legal-case distribution.

## Architecture

```
React 18 + TypeScript (Vite, TanStack Query, shadcn/ui)
        │  all reads/writes through typed React Query hooks — no mock data (a test fails if one is reintroduced)
        ▼
Supabase
  ├─ Postgres  — business rules + Row Level Security on every table
  ├─ private schema — SECURITY DEFINER helpers used by policies (is_admin, get_my_area, …)
  ├─ Auth      — session + profile with role and capability flags
  └─ Edge Functions — privileged operations (user provisioning/deletion, notifications)
GitHub Actions — CI, daily backup + restore drill, daily "rules watchdog", schema-from-scratch build
Vercel — auto-deploy from main
```

## Engineering highlights

**Authorization model with 7 roles × 5 capability axes, enforced in Postgres.**
Roles (`partner`, `head`, `lawyer`, `reception`, `purchasing`, …) are not enough: a firm needs "the person who distributes cases" without making them a partner, or "the person who maintains the board" without making them a system admin. I modeled independent boolean capabilities plus derived ones, and implemented each gate *once* in SQL and mirrored it in the frontend from the same definition, so the UI and the database can't disagree. I also documented RLS anti-patterns (e.g. policies that filter on a mutable data field instead of identity) as review checklists.

**Destructive operations behind a hard protocol.** Any `DELETE`/`DROP`/mass `UPDATE` in production requires: the exact predicate plus a `count(*)` with the same predicate, proof of a safety net (audit trail or soft-delete), a `BEGIN … ROLLBACK` dry-run, and explicit written approval. Soft-delete and audit triggers are the default.

**Backups that prove they can be restored.** A daily workflow dumps `public` *and* `private` (restoring data without the security helpers would restore the data without the security). A separate "backup drill" runs on every PR touching the scripts and **must fail when given deliberately corrupted data** — otherwise a broken comparator would stay green forever.

**A daily watchdog for production drift.** Migrations self-check once when applied, but nothing stopped someone from undoing a rule later. I wrote a job that runs every night against the real database, asks the Postgres catalog whether each documented rule still holds (e.g. "no function reachable by anonymous users", "this table has no UPDATE policy"), and compares the migration ledger with `main`. It opens an issue only on *dangerous* drift — a red build every day trains people to ignore it.

**Schema reproducible from zero.** A workflow builds the whole database from migrations in a clean Postgres on every run, with no secrets — a cheap staging that catches migrations that only work on top of production state.

## Results

- Replaced four tools with one system; partners see the pipeline and workload per area in real time.
- Security audit run before each milestone, findings closed in tracked PRs and locked by regression tests.
- Deploys are routine because CI, backups and the watchdog make them boring.
