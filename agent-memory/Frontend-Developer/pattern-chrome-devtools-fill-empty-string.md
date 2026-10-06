---
name: pattern-chrome-devtools-fill-empty-string
description: chrome-devtools-mcp's fill() with value="" clears the DOM but does not fire React's onChange — controlled-input state silently stays stale during live-verify.
metadata:
  type: feedback
---

`mcp__plugin_chrome-devtools-mcp_chrome-devtools__fill(uid, value="")` visibly empties a
controlled `<input>`/`<textarea>` in the accessibility snapshot (no `value=` attribute shown), but
because there's nothing to "type," it does not dispatch the real input/keystroke events React's
`onChange` listens for. The component's own state (and therefore the app's dirty-tracking /
draft / diff logic) never updates — it silently keeps the OLD value, while the page visually looks
cleared. This produced a false negative while live-verifying Est8Core's publishing-settings
patch-diff logic: a field I "cleared" was absent from the save diff because the draft state never
actually changed.

**Why:** `fill` on empty string skips whatever keystroke-simulation path fires on non-empty text.

**How to apply:** to clear a controlled input during live browser verification, fill with a single
space `" "` (or any throwaway character) instead of `""` — this is a real character so the normal
typing path fires onChange, and if the app already does trim-to-null/empty normalization
server-side (e.g. `value.trim() ? value : null`), a lone space still resolves to the intended
empty/null result. Confirmed working end-to-end (PUT body carried the correctly-nulled field)
after switching from `fill(uid, "")` to `fill(uid, " ")`. Applies to any React-controlled-input
app, not just Est8Core — check this pattern first whenever a "clear the field" step in a live-verify
doesn't show up in a diff/dirty state that should have caught it.
