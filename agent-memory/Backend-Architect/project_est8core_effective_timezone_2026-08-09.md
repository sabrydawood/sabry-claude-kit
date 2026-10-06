---
name: project_est8core_effective_timezone_2026-08-09
description: Est8Core CRM's effective-timezone model (branch → tenant → UTC) — where the single resolver lives and what already existed vs what was built 2026-08-09.
metadata:
  type: project
---

Est8Core CRM (D:\Work\1-Nodejs\CRM\Est8Core\New\Server) resolves a caller's EFFECTIVE IANA timezone
per this chain (Sabry, 2026-08-09): `role === "owner"` → tenant's effective timezone (role rule,
never "has a branch") → everyone else → their ACTIVE branch's timezone → tenant's → "UTC".

**The single resolver**: `Core/Utils/Timezone.Util.ts`'s `ResolveEffectiveTimezone` (async, dynamic-
imports `Modules/Branches/Branches.Service.ts`'s cached `GetBranchTimezone` to dodge a Core→Module
static cycle) + `CombineEffectiveTimezone` (pure combinator, unit-tested with no DB). Consumers:
- Live requests: `Core/Middleware/Auth.Middleware.ts` computes it right after decoding `User` and
  stashes it as `c.set("EffectiveTimezone", ...)`; `Core/Authz/Scope/ScopeContext.ts`'s
  `BuildScopeCtx` reads it back — deliberately kept SYNCHRONOUS (it's called from ~250 controller
  sites) rather than doing the DB lookup inline.
- No-live-request paths (emails, scheduled jobs): `Modules/Export/Export.Fetch.CallerScope.ts`
  (split out of `Export.Fetch.Service.ts`, re-exported from there for backward compat) has
  `ResolveCallerScope` (full IScopeCtx, used for ROW SCOPING — its BranchId stays on the coarser
  `ResolveEffectiveBranchId`, not the active-branch-aware one, deliberately) and
  `ResolveRecipientTimezone` (JUST the timezone, built on `FetchSessionIdentity` for the ACTIVE-
  branch-aware answer a multi-branch manager's mailbox needs — these two intentionally use
  DIFFERENT branch resolvers for different reasons, see the file's own doc comments).

**What already existed before this task** (do not rebuild): `branches.timezone` (nullable) and
`tenants.timezone` columns; `Tenant.Middleware.ts`'s `settings.app.company_timezone → tenants.
timezone → "UTC"` chain (`ResolveEffectiveTenantTimezone` in the same Timezone.Util.ts file);
`Export.Format.ts`'s per-cell tenant-wall-clock date rendering. The gap was specifically: nothing
read `branches.timezone` anywhere, and no email/no-request path was branch-aware.

**Export column headers** now disclose the timezone for `type: "date"` columns (e.g. "تاريخ الإنشاء
(توقيت مصر)") via `Export.Format.ts`'s `AppendTimezoneToHeader`, injected at the SINGLE header-
resolution chokepoint `Export.Generator.ts`'s `ResolveHeader` — reaches every CSV/XLSX generator
(sync + streaming) with one change, no per-call-site duplication. Uses `Intl.DateTimeFormat`'s
`timeZoneName: "shortGeneric"` with a `longGeneric` fallback when the short form is a bare Latin
abbreviation (Dubai/Muscat's shortGeneric is "GST" even in the `ar` locale — falls back to "توقيت
الخليج").

**Frontend** (`est8core-frontend/platform-master/apps/crm`): server-side `timeZone` for next-intl
is NOT derivable from the client Zustand auth store (RSC has no access to it) — mirrors the
existing `NEXT_LOCALE` cookie pattern exactly: `i18n/timezone.ts`'s `getTimezone()` reads a
`NEXT_TIMEZONE` cookie, `i18n/setTimezone.ts` is a Server Action that writes it, called from
`AuthHydrator.tsx` (after `/auth/me`) and `useBranchSwitch.ts`'s `onSuccess` (immediately, using
the switch response's own `timezone` field — doesn't wait for an `auth.me` refetch).

See [[feedback_est8core_timezone_advisor_corrections]] for the mistakes the advisor caught before
I wrote any code.
