---
name: feedback_authz_fix_read_and_write_same_predicate
description: When closing an IDOR/authorization gap, the same predicate must guard every write path (reply/update/close/delete), not just list/get, and must run BEFORE any business-state short-circuit that would leak data or confirm existence.
metadata:
  type: feedback
---

When fixing a "user A can read user B's record" (IDOR) bug, do not stop at the read/list
endpoints. Enumerate every mutating endpoint on the same entity (reply, update, close, delete) and
apply the IDENTICAL ownership/scope predicate to each — a guard that exists on GET but not on
POST/PATCH is the same vulnerability in a narrower shape (the attacker can't list the record, but
can still guess/replay its id and act on it).

Two closely-related ordering bugs to check for on every write path:
1. The ownership check must run BEFORE any early-return driven by business state (e.g. "already
   closed → no-op idempotent return", "ticket is closed → 409"). An early-return that fires before
   the ownership check either (a) confirms the record exists via a distinguishable status code, or
   worse (b) echoes the full record back to the caller as a "successful" idempotent response —
   leaking the victim's data even though the caller "wasn't authorized to see it" via the normal
   read path.
2. Prefer 404 over 403 for out-of-scope records when the codebase already uses 404 for a sibling
   check (e.g. cross-tenant mismatch). A 403 confirms the id exists at all (enumeration oracle);
   matching the existing 404 convention for "wrong tenant" with "wrong owner" keeps both
   indistinguishable from "doesn't exist".

**Why:** Found this exact pattern in Est8Core's Support Tickets module (96/F-03 == 99/T7): List
and Get had no per-user scoping at all (any tenant user could read any other user's tickets), and
even after adding an ownership check to Get, Reply and Close each independently re-fetched the row
and needed their OWN copy of the same check — Close in particular had an idempotent
`if (status==="closed") return Ok(Ticket)` that would have leaked the full ticket row as a 200 if
the ownership check were placed after it instead of before.

**How to apply:** On any authorization-gap fix, write down the full list of endpoints touching the
entity (read AND write) before writing code, derive ONE shared predicate/helper function, and
verify each endpoint calls it BEFORE any status-based early return. Prove it empirically per
function, not just "I copy-pasted the same block four times" — read the final text of each path
independently.
