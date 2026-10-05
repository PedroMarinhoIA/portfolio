# 2. Cattle Ranch Management System

**Client:** a cattle ranching group in Brazil (owners, managers, administrative staff and field hands).
**Status:** in production since June 2026, tagged releases v1.0 → v1.6. Private repository — 571 commits, ~280 test files, 17 API route modules.
**My role:** sole engineer, end to end.

## The problem

The ranch tracked herds, pastures and animal movements on paper and in a NocoDB database edited by hand and by n8n automations. Field hands needed to record events from a phone in the field; owners needed a trustworthy picture of the herd. The browser was talking to the database with a token embedded in the bundle — anyone could read or change anything.

## What I built

A mobile-first web app for the whole operation: herd lots, pastures and rotational grazing (Voisin rest periods by season), social areas and feed stock, births, entries/exits, bulk movements, breeding season (IATF) records, user management and a full audit trail.

## Architecture

```
React SPA (TanStack Query, React Hook Form + Zod)
        │  /api/*  (httpOnly session cookie)
        ▼
Hono backend on Vercel Functions (Node runtime)
  ├─ auth: bcrypt + own JWT (HS256 pinned), httpOnly/Secure cookie, role guards
  ├─ Zod validation on every write; size limits on every list query
  ├─ audit_log row for every write (who, what, from where)
  ├─ in-memory route cache with bounded keyspace
  └─ Sentry with outbound redaction
        │  REST (service token never leaves the server)
        ▼
NocoDB (existing system of record, also written by n8n automations)
```

The key decision was a **backend-for-frontend pivot**: instead of migrating the client off NocoDB (which their automations depend on), I put a Hono API in front of it. Table IDs and the database token disappeared from the bundle, and every business operation became a named, validated, audited verb (`split-lot`, `register-entry`, `move-bulk`…) instead of a generic row update.

## Engineering highlights

**Login throttling designed against real attack math.**
Per-account bucket (IP + e-mail), per-user bucket on password change, and a global per-IP ceiling. The slot is *reserved in the same tick as the check*, before any `await`, and returned in `finally` if the outcome wasn't a wrong guess — otherwise a burst of concurrent requests could buy N guesses instead of 5. Buckets cut before bcrypt and respond `429` with the wait time. Because a per-IP ceiling let an attacker lock out the whole ranch with 20 fake e-mails, successful logins issue a "known device" cookie (no identity inside) that skips the IP ceiling but never the per-account one. Eviction never drops a blocked entry — an earlier version did, and an attacker could flush their own block by spraying invented e-mails.

```ts
// simplified excerpt — reserve the attempt before any await
if (throttle.blocked(key)) return tooManyRequests(c, throttle.retryAfter(key));
throttle.registerFailure(key);            // reserve the slot now
let wasWrongGuess = false;
try {
  const user = await findUserByEmail(email);
  wasWrongGuess = !user || !(await bcrypt.compare(password, user.hash));
  if (wasWrongGuess) return unauthorized(c);
  return issueSession(c, user);
} finally {
  if (!wasWrongGuess) throttle.giveBack(key); // success or server error: refund
}
```

**Audit sampling with logarithmic cost.** Requests refused by an already-blocked bucket are logged at the 1st, 2nd, 4th, 8th… attempt: a million-request attack leaves ~20 rows and the counter still tells you its real size, without flooding the single-threaded database or drowning the forensic signal.

**Telemetry redacted on the way out.** Sentry's default fetch instrumentation records outbound URLs as breadcrumbs and spans, which carried base IDs, table IDs, where-clauses and e-mails. Redaction runs on the final event (request, breadcrumbs, spans and exception message), not only at the source.

**Secrets with a strength floor, not just presence.** Boot fails if `JWT_SECRET` or the DB token is shorter than 32 chars or equals any literal committed in the repo (tests included). The test-mode exemption requires *two* independent runner signals, because a single `NODE_ENV=test` in production would have silently accepted a public secret.

**Tests that police the codebase.** Some tests scan the source tree: e.g. reading IP headers outside one module fails the build; a list route that parses `offset/sort/fields` by hand instead of the shared limits fails the build.

**Performance for 3G in the field.** Dashboard went from 21–22 requests to 6 (aggregated endpoints + cache warming via cron), and every flow was redesigned for one-handed phone use. Every dialog's error is announced to screen readers and bound to its field.

## Results

- 14 users across 5 roles using it daily, including field hands on phones.
- Every write is attributable (user, verb, IP) — the owners trust the numbers.
- Multiple security audit passes, with findings triaged and fixed in tracked PRs.
