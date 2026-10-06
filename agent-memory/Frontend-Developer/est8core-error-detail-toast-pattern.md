---
name: est8core-error-detail-toast-pattern
description: Est8Core CRM's established way to add a richer error toast (e.g. ApiError.details breakdown) on top of the automatic global one, without suppressing it — and the one-modal-state-machine convention for a detail page's dialogs.
metadata:
  type: project
---

Est8Core CRM (`apps/crm`) already has a global error-toast pipeline: `lib/api/queryClient.ts`'s
`MutationCache.onError` calls `showApiErrorToast(error)` for EVERY mutation error automatically,
using `getApiErrorMessage()` (`lib/api/apiErrorMessage.ts`) which already reads the server's
localized `error.message`. Codes only get silenced via a hardcoded `SILENT_ERROR_CODES` Set when
there's a dedicated replacement UI (e.g. `ALREADY_PAID`).

**When a specific error code needs to show MORE than the generic message** (e.g. a 422 whose
`error.details` carries numbers worth surfacing), do NOT suppress the global toast. The documented,
established pattern (`lib/api/mutationFeedback.ts`'s own doc comment on its `onError` param) is:
let the global toast fire for the base message, and ADD a second, narrower toast/callback that says
"something the generic message cannot." Two flows for this, both legitimate here:
- `withMutationFeedback(mutateAsync, successMsg, onError)` — the dominant house style (~80+ call
  sites), used when the caller also wants a success toast.
- Plain `.mutate(variables, { onError })` (TanStack Query v5's call-time options) — used when the
  mutation must stay fire-and-forget with NO success toast (matches `useDeals.ts`'s
  `deleteMutation.mutate(id, { onSuccess: ... })` sibling call shape). Confirmed valid and precedented
  even though no prior call site used the `onError` variant specifically.

`ApiError.details` (`packages/api-core/ApiError.ts`) is typed `unknown` — every consumer narrows it
with a local inline cast keyed off `error.code` (see `getPlanDowngradeViolations`,
`useUserActions.ts`'s branch-move-blocked handler, `TargetDistributeModal.tsx`). No shared generic
"details" type exists; don't invent one for a single call site.

**Deal-detail page (`app/deals/[id]/`) dialog convention**: `useDealDetailModals.ts` +
`DealDetailModalsHost.tsx` run ALL of a detail page's modals through one `modal: Kind` union +
one host component that inlines every `<Modal>` directly (no per-dialog component files, unlike
list-page deletes which DO get their own file, e.g. `DealDeleteModal.tsx`). Adding a new dialog to
an existing detail page means adding a new `Kind` member + inlining its `<Modal>` block in the SAME
host, not creating a new component or reaching into a sibling feature's private `_hooks`/`_components`
(e.g. `app/settings/_hooks/useConfirmDialog.ts` + its `ConfirmDialogModal.tsx` look like a reusable
confirm-dialog component but are page-scoped by Next.js App Router convention — importing them into
another route segment would be a layering violation even though they're the closest "existing
confirm dialog" by DRY search). The right reuse target is the underlying primitives (`Modal` +
`Button variant="danger"` from `@est8/ui`), matching what `DealDeleteModal.tsx`/`ConfirmDialogModal.tsx`
are themselves built from.

**Sharing one form between "add" and "edit" modes** (to avoid a second near-identical `<Modal>` block):
track a `editingXId: string | null` alongside the existing form state. The existing `openAdd*()` must
be upgraded to explicitly reset the form on open once a second entry point (`openEdit*()`) starts
seeding real data into the same state — otherwise a cancelled edit leaks into the next "add".
