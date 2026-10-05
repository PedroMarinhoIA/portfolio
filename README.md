# Pedro Marinho — Engineering Portfolio

I build and run production software for small and mid-sized businesses in Brazil: internal platforms that replace spreadsheets, WhatsApp threads and paper with systems that real people use every day. I own the whole lifecycle — requirements with the client, data modeling, security, implementation, CI/CD, deploy, monitoring and support.

The source code of these systems belongs to my clients and lives in **private repositories**. This repo documents the three most complex ones as case studies: what the problem was, how the system is designed, and the engineering decisions that matter. I am happy to walk through the real code live during the interview.

## Case studies

| | Project | Domain | Core stack | Scale |
|---|---|---|---|---|
| 1 | [**Law Firm Operations Hub**](case-studies/01-law-firm-hub.md) | Legal (LGPD + attorney–client privilege) | React · TypeScript · Supabase (Postgres RLS, Edge Functions) | In production since Apr/2026 · 30+ lawyers · 757 commits · 153 migrations |
| 2 | [**Cattle Ranch Management System**](case-studies/02-ranch-management.md) | Agribusiness / field operations | React · Hono (Vercel Functions) · NocoDB · Zod | In production since Jun/2026 · v1.6 · 571 commits · ~280 test files |
| 3 | [**Multi-tenant Workplace Mental-Health Platform**](case-studies/03-mental-health-platform.md) | HealthTech / B2B SaaS (NR-1 compliance) | React · TanStack Router/Query · Supabase · Deno Edge Functions | In production · 1,411 commits in 4 months · 90 migrations · 16 Edge Functions |

## What these projects have in common

- **Security as a delivery gate, not an afterthought.** Major releases go through a static + dynamic security audit (injection, authz, RLS, CSRF, SSRF, secrets, logging leaks). Findings are tracked with IDs, fixed in dedicated PRs, and locked in place by regression tests.
- **Authorization lives in the database.** Row Level Security policies backed by `SECURITY DEFINER` helpers, with tests per role — the frontend hides buttons, the database enforces the rule.
- **Sensitive data by design.** Legal case data, health data and employee reports under LGPD (Brazil's GDPR): data minimization, audit trails, anonymization, retention jobs, redaction of telemetry before it leaves the server.
- **Operational discipline.** Branch → PR → CI → review → merge → auto-deploy. Daily database backups with a restore drill that must *fail* on deliberately corrupted data, daily jobs that check production against documented rules, and decision logs that record *why*, not just *what*.
- **AI-assisted engineering, done rigorously.** I work heavily with Claude Code and agentic workflows (spec → plan → execute → verify), with explicit project rules, measured context budgets and tests that keep the agent honest. The AI accelerates; the tests and the reviews decide.

## Stack I use day to day

**Frontend:** React 18/19, TypeScript (strict), Vite, TanStack Query & Router, React Hook Form + Zod, Tailwind + shadcn/ui, Recharts
**Backend:** Supabase (Postgres, RLS, Auth, Edge Functions, pg_cron), Hono on Vercel Functions, Node.js, Deno, SQL/PLpgSQL
**Quality & Ops:** Vitest, Testing Library, Playwright (E2E), pgTAP-style SQL specs, GitHub Actions, Vercel, Sentry
**Automation:** n8n, Python (scraping, data recovery, reporting), REST/webhook integrations (LMS, WhatsApp, e-mail providers)

## Contact

- GitHub: [@PedroMarinhoIA](https://github.com/PedroMarinhoIA)
- E-mail: available on request / in my application
