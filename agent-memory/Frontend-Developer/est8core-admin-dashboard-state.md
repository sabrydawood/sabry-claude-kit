---
name: est8core-admin-dashboard-state
description: Est8Core admin Dashboard (apps/admin/app/page.tsx) was already real-backend-wired before this session; only gaps were shared KpiCard/StatusBadge extraction + row navigation + relative-time.
metadata:
  type: project
---

Est8Core admin `/` Dashboard (`platform-master` repo, `apps/admin/app/page.tsx` + `PageContent.tsx` +
`app/_components/{KpiCards,RecentTenantsTable,SystemHealthCard,QuickActions,DashboardRefreshButton}.tsx`)
was **already rebuilt to the AGENTS.md standard in a prior session** — real `GET /admin/dashboard` wiring
(`lib/api/hooks/useDashboardStats.ts` → `resources/dashboard.ts`), React Query, next-intl namespace
(`messages/dashboard/{ar,en}.json`), no mock data. The task prompt describing it as needing a from-scratch
rebuild was stale relative to actual repo state — always `git status`/read the current files before assuming
a screen needs full reconstruction.

**Correctly deviated from the static mock on purpose (do not "fix" back to mock):**
- `SystemHealthCard` renders only `GetPoolStats()` fields (master/tenant pool + cachedSchemas) — the mock's
  redis/queue rows are illustrative fluff not returned by any real endpoint. Confirmed against
  `Server/Src/Config/Database.ts#GetPoolStats` + `Server/Src/Modules/Admin/Admin.Service.ts#GetDashboardStats`.
- KPI icon chip colors use theme-aware utilities (`bg-success/10 text-success` etc.) instead of the mock's
  inline `style="background:rgba(...)"` — see [[est8core-no-inline-hex-theme-aware]] memory in the main index.
- Plan NAME resolution: `GET /admin/tenants` (`AdminListTenants`) returns raw `planId` (UUID) only — no
  plan-name join. Resolve it **client-side** by cross-referencing the plan catalog from
  `GET /admin/billing/plans` (admin app is super_admin-authed, so use the admin billing endpoint, not the
  public `/api/v1/public/plans`). Do NOT show the raw UUID or invent a label.

**What this session added (2026-07-04):**
- `KpiCard` + `StatusBadge` promoted to `@est8/ui` (`packages/ui/components/`, exported from the *real* barrel
  `packages/ui/index.tsx` — NOT `packages/ui/components/index.ts`, which is dead/unreferenced code despite its
  own comment claiming otherwise; grep confirmed nothing imports it).
- `tenantStatusBadgeClass(status): string` → renamed `tenantStatusTone(status): StatusBadgeTone` (returns a
  tone enum for the new component instead of a raw class string). Both call sites (dashboard's
  `RecentTenantsTable` + `tenants/_components/TenantsTable`) updated together since it's a shared helper.
- `RecentTenantsTable` rows had `cursor-pointer` styling but **no actual click handler** — real gap vs the
  mock and vs the sibling `TenantsTable` (which already used `Link href="/tenant-detail?id=..."`). Fixed to
  match that established pattern.
- Added `useRelativeTime()` hook (`apps/admin/hooks/useRelativeTime.ts`) wrapping next-intl's
  `useFormatter().relativeTime` + `useNow({ updateInterval: 60_000 })` — replaces the mock's hand-rolled
  `timeAgo()` JS. Used for both the recent-tenants "registered" column and a new `DashboardUpdatedLabel`
  ("last updated: {time}") that reads `dataUpdatedAt` off the existing `useDashboardStats()` query (React
  Query dedupes the call — no extra fetch) instead of a fake `setTimeout`-driven `useState`.
- `apps/admin/lib/tenantAvatar.ts` (`tenantAvatarText`) factored out of duplicated `name.slice(0,2)` in two
  table components.

**Coordinator follow-up (same 2026-07-04 session) — plan-name resolution + mock-exact wording:**
- Plan-name resolution built as a **shared** stack in `lib/api/`: `types/plans.ts` (`PlanListItem`) +
  `resources/plans.ts` (`listPlans` → `GET /admin/billing/plans?page=1&limit=100`) + `hooks/usePlans.ts`
  (React Query, `staleTime` 5min — catalog rarely changes) + `hooks/usePlanNameResolver.ts` (builds a
  `Map<planId,name>` via `useMemo`, returns a stable `resolve(planId) => name | null`). The trial/dash
  fallback label is a **pure helper** `lib/tenantPlan.ts#tenantPlanLabel({planName,status,trialLabel,noneLabel})`
  — resolution is data (hook), the label is copy (i18n passed by caller). Both `RecentTenantsTable` (dashboard)
  and `tenants/_components/TenantsTable` consume the same resolver + helper → the Plan column now exists on
  BOTH tables identically. Reuse this exact stack for any future tenant-plan display.
- Backend verified: `Admin.Billing.Service#ListPlans` returns full `plans` rows (`id`,`name`,`code`,
  `priceMonthly`,`isActive`) in the `{ results, meta }` Paginated envelope; mounted at
  `${API_PREFIX}/admin/billing` (App.ts).
- Mock-exact Arabic wording is a hard "طبق الأصل" requirement — the dashboard `ar.json` copy must match the
  mock's `dashboard()` script verbatim (e.g. "منشآت نشطة" not "المنشآت النشطة", "إلغاءات الشهر", "الإيراد
  السنوي المتوقّع", "معدّل الإلغاء {rate}٪" with the **Arabic** percent glyph ٪ U+066A, quick-action
  "الفواتير المعلّقة"). churn rate denominator is **active** tenants (not total) — mock parity.
- System-health card chrome now matches mock: a `StatusBadge` in the card head (سليم/متدهور/غير سليم) whose
  status is DERIVED from real pool pressure via `app/_lib/systemHealthStatus.ts` (down = a pool saturated:
  waiting>0 AND idle==0; degraded = any waiting>0; else healthy) — never fabricated — plus a full-width ghost
  "عرض كل التنبيهات" `Link` to `/notifications`. Rows still the real GetPoolStats data only (no redis/queue).

**tenant-detail route reality:** `apps/admin/app/tenant-detail/page.tsx` is still the **unrebuilt legacy
mock** (`'use client'`, hardcoded `initialTenant`, old `AdminShell` import) — navigating there from the
dashboard/tenants tables lands on a page that ignores the `?id=` query param entirely. Rebuilding
`tenant-detail` to the real contract is a separate, not-yet-done task (per `apps/admin/AGENTS.md §12`
refactor backlog).
