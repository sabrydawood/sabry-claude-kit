---
name: user_sabry_security_task_style
description: Sabry (Est8Core CTO) hands off security-fix tasks pre-scoped with exact file:line evidence, explicit design rules, hard git/test constraints, and an empirical-proof requirement — matches his broader CTO+PM working style (decisive, no guessing, verify before "done").
metadata:
  type: user
---

Sabry runs Est8Core CRM (multi-tenant SaaS, Bun/Hono/TypeScript/Postgres/Drizzle) as a solo
CTO+PM working through Claude agents. When he (or his orchestrating agent) hands off a security
fix, the brief is unusually complete and should be trusted as accurate but still verified, not
re-derived from scratch:

- Comes with exact `file.ts:line` evidence of the bug, already independently confirmed.
- States the fix DECISION explicitly (e.g. "owner sees all, everyone else sees own — implement,
  do not redesign") — this is a closed design question, not one to re-litigate.
- Names concrete hard constraints: forbidden git subcommands (add/commit/push/stash/reset/
  clean/restore/rebase/rm — status/log/diff/show only), never run tests under paths that hit a
  shared LIVE database, never mint/print tokens or secrets, and explicit file-ownership boundaries
  when another agent is concurrently editing a nearby file (e.g. "don't touch Routes.ts, another
  agent owns it right now — report if you think it needs a change instead of making it").
- Requires PROOF, not a claim of done: typecheck clean, project-specific lint/authz/docs gates,
  AND an empirical demonstration (e.g. a throwaway script under `Scratch/` that calls the actual
  shipped function and prints real output) before declaring completion — "I read the code and it
  looks right" is not accepted.
- Wants the report to state WHICH existing pattern/mechanism was reused and WHY (not just "I fixed
  it") when multiple existing mechanisms could plausibly apply — see
  [[feedback_dont_force_fit_scope_engine]].

**How to apply:** Treat these briefs as authoritative on WHAT to build and the constraints, but
still independently verify the evidence (re-read the cited lines) and the design fit (does the
suggested existing pattern actually match this entity's data model) before writing code — the
brief tells you the destination and the guardrails, not a blank check to skip verification.
