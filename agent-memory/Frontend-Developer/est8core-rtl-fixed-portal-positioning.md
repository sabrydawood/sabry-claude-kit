---
name: est8core-rtl-fixed-portal-positioning
description: Fixed-position portalled menus/panels in Est8Core's RTL admin app must anchor with physical `left` math, never a runtime `insetInlineEnd`/logical inline style — verified live, not just build-green.
metadata:
  type: feedback
---

When building a `position:fixed` portal (row-action menu, dropdown panel, tooltip) anchored
off a trigger's `getBoundingClientRect()` in the Est8Core admin app (`platform-master`,
`dir="rtl"` by default for `ar`), **do not** set the inline style using the logical property
`insetInlineEnd` computed as `window.innerWidth - rect.right`. This looks correct by analogy
with Tailwind's `end-*` utility classes, but a **runtime inline style** `inset-inline-end` is
resolved by the browser against the portalled element's own writing direction, and — worse —
the arithmetic itself breaks the moment the trigger sits near the *opposite* edge from what you
assumed (e.g. an "Actions" column that is the last `<th>` in DOM order renders on the visual
**left** in RTL, so `window.innerWidth - rect.right` is huge, not small, and the menu gets
pushed off-screen).

**Why:** Discovered building the admin Tenants list row-action menu (`packages/ui/components/RowActionMenu.tsx`).
`next build` was green and the accessibility snapshot even showed the menu with the right menu
items — everything *looked* done. Only a live Playwright check with `getBoundingClientRect()` +
a screenshot revealed the menu was rendering at `x: -120` (fully off-screen). This is the same
class of failure as the standing `est8core-frontend-verify-live` memory rule ("build green ≠
page works") but sharper: even an accessibility-tree snapshot passed, because the menu *was* in
the DOM with the right text — only its visual position was wrong.

**How to apply:** For any `position:fixed` portal anchored to a trigger rect:
1. Use **physical** `left`/`top` (or `right`/`top`), computed directly from `getBoundingClientRect()`
   — mirror `SmartSelect.tsx`'s proven approach (`left: r.left`), not a logical CSS property.
2. If aligning the panel's far edge to the trigger's far edge (e.g. right-align a menu under a
   button), compute `left = triggerRect.right - panelWidth`, then **clamp** to
   `[8, viewportWidth - panelWidth - 8]` so it can never overflow either edge — this clamp is
   what actually fixes the RTL "last-column trigger sits near the left" case.
3. Verify live: click the trigger via Playwright, `getBoundingClientRect()` the portalled
   element, and *assert the box is within the viewport* — don't trust an accessibility snapshot
   or a green build alone for anything involving computed screen position.
4. This app is bidirectional (`ar`=RTL default, `en`=LTR via `NEXT_LOCALE` cookie) — any
   position math must be verified in **both**, not just the default locale.

See also [[est8core-frontend-verify-live]] in the main index (same root cause, different failure mode).
