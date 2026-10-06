---
name: est8core-workspace-page-patterns
description: Est8Core CRM /workspace build — reusable patterns for strict next-intl dynamic labels, AppShell embed handling being free, and delegating cross-route actions via a window instead of duplicating owned UI.
metadata:
  type: project
---

Built `apps/crm/app/workspace/**` (.Pm/112 "مساحتي") in the Est8Core frontend monorepo
(`est8core-frontend/platform-master`). Three patterns worth reusing on future CRM/admin work in
this repo:

**1. Strict next-intl namespace + dynamic value labels — no widening needed.**
`i18n/messages.d.ts` widens 4 namespaces (`nav`/`notifications`/`profile`/`targets`) to
`Record<string, any>` because their call sites need `t(`prefix.${runtimeValue}`)`, which fails
literal-key strictness. The correct fix for a NEW namespace facing the same need is NOT to widen
it too — type the translator precisely as `ReturnType<typeof useTranslations<"ns">>` (exact
pattern already in `components/CrmListPagination.tsx`'s `TeamT`), pass it into small helper
functions, and resolve every dynamic value through an exhaustive `switch` with literal string
arguments to `t(...)`. Zero widening, zero `tsc` errors, full typo-safety preserved. Verified
clean on `npx tsc --noEmit` across ~25 new files using this pattern for lead/deal/activity status
labels, activity type labels, and focus-queue source labels.
**Why:** widening trades away key-typo safety repo-wide for that namespace; the switch pattern
keeps it while still handling runtime-determined keys.
**How to apply:** any new i18n namespace with status enums / dynamic-source labels — reach for
`ReturnType<typeof useTranslations<"ns">>` + switch helpers before reaching for the widened-type
escape hatch.

**2. `?embed=1` chrome-stripping is centralized in `components/shell/AppShell.tsx`, not per-page.**
Every route's `page.tsx` just wraps `<CrmShell activeNav="x"><PageContent/></CrmShell>` unconditionally
(see `app/leads/page.tsx`, `app/deals/[id]/page.tsx`). `AppShell` itself reads
`useSearchParams().get("embed")` and renders bare content (no sidebar/topbar) when `"1"`. A new
page needing floating-window support needs ZERO embed-specific code of its own — verify this by
reading `AppShell.tsx` before assuming you need to branch on `embed` in a new page's own
`PageContent.tsx`.

**3. When a route is forbidden from touching another route's files but needs its create/export/bulk
UI, the established app convention is delegation-by-window, not reimplementation.** Even the
topbar's own `components/shell/CreateMenu.tsx` doesn't open a create modal directly — it navigates
to `/leads` (etc.) and lets that page's own create button do the rest. `useOpenLeadWindow.ts`'s
`openWindow({key, src: "/x?embed=1", title, full: "/x"})` contract is the reusable shape; when you
can't import that hook (cross-route `_hooks` import is against convention — see `useLeads.ts`'s
`asArray` header, small logic gets duplicated locally, not imported cross-route), reimplement the
~10-line `openWindow` call locally rather than building a competing create form. `openWindow`
refuses past `MAX_OPEN_WINDOWS` and the caller must show `useTranslations("window")`'s
`capReached` toast on refusal.

Also confirmed: `refetchOnWindowFocus: false` is the GLOBAL default in
`lib/api/queryClient.ts` — any feature relying on "the window you return focus to re-reads fresh
data" as its cross-iframe sync bridge (two copies of a page open as separate windows, each its own
QueryClient/realm) must override `refetchOnWindowFocus: true` on that SPECIFIC query, not assume
the global default already does it.
