---
name: est8core-seed-large-credential-drift
description: Tenant seed-large's owner login no longer matches the documented seed password/email — verified 2026-08-01, live-verify must fall back to another seeded tenant.
metadata:
  type: project
---

Tenant `seed-large` (slug `seed-large`, master schema `tenant_33434651`) has an owner user whose
email is `kazouya44@gmail.com`, not the documented `seed-large@est8core.dev` from
`Server/Scripts/seed-full-tenant.ts:157` (`SEED_PASSWORD = "Seed@12345"`). `updated_at` on that
row was `2026-08-01T16:01:36Z` — changed same-day, likely by a parallel session actively testing
against this tenant (a separate isolated browser page was open on `seed-large.localhost:3002/p/prop-0800`,
context `verify-t6-opus`, at the same time). The old default credentials return
`401 AUTH_INVALID_CREDENTIALS`.

**Why not just log in with the new email:** it resembles a personal account (same family as the
project's forbidden `kazouya25@gmail.com`) and Claude Code's auto-mode safety classifier blocked
the `fill()` action outright when attempted. Did not attempt to route around it — surfaced instead.

**How to apply:** any task instruction saying "sign in as the owner of tenant seed-large with the
seed password" should be treated as unverified until re-checked. `seed-individual` (status
`active`) and `seed-medium`/`seed-small` (status `trial_expired` — writes blocked under ADR-0023
read-only grace, only useful for read-path verification) still have their untouched default seed
owner emails/passwords as of this check. Read-only re-check recipe: query master `tenants` for the
slug's `schema_name`, then `SELECT email, role, is_active, failed_login_attempts, locked_until FROM
"<schema>".users WHERE role='owner'` — no write, safe to run anytime.

Flag this to Sabry rather than silently substituting tenants — the substitution is a legitimate
workaround for live-verification but the underlying drift (why seed-large's owner identity
changed) is worth him knowing about directly.
