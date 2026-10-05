# 3. Multi-tenant Workplace Mental-Health Platform

**Client:** a consultancy that sells companies a workplace emotional-health program aligned with Brazil's **NR-1** regulation (psychosocial risk management, mandatory for employers).
**Status:** built from scratch starting June 2026, in production. Private repository — 1,411 commits in ~4 months, 90 migrations, 16 Edge Functions, 60+ test files plus dedicated RLS, behavior, E2E and UAT suites.
**My role:** sole engineer, from functional spec to deploy.

## The problem

One platform has to serve three very different audiences with conflicting interests:

- **Employees** of client companies — book individual psychological sessions (monthly quota per company), RSVP to live monthly sessions, follow e-learning tracks, answer surveys, file conduct reports.
- **Company managers** — need evidence for labor inspections (who was invited, participation rates, aggregated risk indicators) but **must never see an individual's health data**.
- **The consultancy** — operates everything across all tenants.

## Architecture

```
React 18 + TypeScript strict (Vite, TanStack Router file-based typed routes, TanStack Query, RHF + Zod)
        ▼
Supabase (multi-tenant by company_id)
  ├─ Postgres + RLS — tenant isolation + role isolation, RESTRICTIVE policies for inactive users
  ├─ SECURITY DEFINER RPCs as the only write doors for sensitive flows
  ├─ pg_cron — sweeps, retention, outbox draining
  └─ Edge Functions (Deno):
       provision-user / provision-users-batch / definir-senha / trocar-senha
       notifications-worker (outbox) · resend-webhook (delivery status)
       twygo-sync / twygo-worker / twygo-catalog (LMS integration)
       nr1-report (compliance report) · export-my-data · anonymize-account · retention-worker
```

## Engineering highlights

**Privacy enforced by structure, not discipline.** The core invariant: *a manager never reads individual health data*. Aggregates exposed to managers go through k-anonymity thresholds. When the conduct-report channel was added (the first place a manager reads individual records), I made the invariant hold *structurally*: the reports table has no foreign key to appointments, attendance, e-learning progress or surveys, and must never get one — the failure mode is a JOIN in a screen, which no per-role policy test would catch.

**Anonymity guaranteed by the absence of columns.** Anonymous reports have no `ip`, `user_agent` or `route` columns — a column that doesn't exist can't be filled by accident later. `is_anonymous` is its own column instead of `reporter_id IS NULL`, because account deletion also nulls `reporter_id` and would silently turn an identified report into an "anonymous" one. Only *failed* lookups are logged; logging successful ones would re-identify a reporter by correlation.

**Transactional outbox for every notification.** Features only `INSERT` into `notifications`; a pg_cron-driven worker claims rows (`FOR UPDATE SKIP LOCKED`), sends via the e-mail provider and records delivery through a webhook. Permanent failures are split by cause: missing configuration is an *environment* error (500 before claiming, no state change), an undeliverable address is a *row* failure — conflating them would push the whole outbox into a terminal state. Terminal states strip one-time credentials from the payload; the retry state deliberately doesn't, because the retry re-reads the payload.

**An unauthenticated write door, made safe.** "I can't log in" is the case that most needs a support button, so the feedback RPC is the only write reachable without a JWT. The door is the *function*, not the table (no insert grant, so the caller can't choose columns); identity is stamped from the database, never taken from parameters. Rate limiting has two axes (per IP and a global anonymous ceiling whose predicate never references the IP column — a missing IP header yields `NULL`, and `NULL` never matches `NULL`, which would silently make the IP limit unlimited). The count→insert window is serialized with `pg_advisory_xact_lock`.

**LGPD lifecycle as code.** Self-service data export, account anonymization, retention workers, and an allowlist that defines exactly which notification templates a manager may see delivery status for (employer acts like invitations — yes; anything about care — no). Adding a template to that list is documented as *moving the product's LGPD boundary*.

**Bulk onboarding.** CSV import of employees (Papa Parse + server-side validation) with batched provisioning and invitation links that let the employee set their own password.

## Results

- One system replacing manual scheduling, spreadsheets and e-mail for the consultancy and its client companies.
- Labor-inspection-ready NR-1 reporting without exposing any individual's health data to the employer.
- Every security-relevant decision recorded with its rationale in an architecture decision log, so the next change doesn't undo it.
