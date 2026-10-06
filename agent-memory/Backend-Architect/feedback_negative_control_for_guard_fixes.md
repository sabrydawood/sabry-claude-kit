---
name: feedback_negative_control_for_guard_fixes
description: When adding a staleness/race guard, prove it fires by temporarily disabling it and watching the adversarial test fail — green tests alone don't prove the guard does anything.
type: feedback
---

When fixing a race/staleness/ordering bug by adding a precondition guard (e.g. "only downgrade if
status is still X"), writing a passing adversarial test is NOT sufficient evidence the guard works.
A test can pass for the wrong reason (fixture setup accident, an unrelated earlier assertion masking
the real check, a condition that's vacuously true in the test's specific data shape).

**Why:** Caught on Est8Core's F-29 fix (`Billing.Webhooks.Service.ts` / `Billing.StripeWebhooks.Service.ts`
— a late `invoice_failed` webhook downgrading a tenant that had already paid). The advisor flagged
that green tests weren't proof the guard fired, and required temporarily disabling each guard
(`if (false && Condition)`) and re-running the specific adversarial test to confirm it fails with
the expected wrong values, before restoring the guard. It did — cleanly, with the exact expected
wrong values in the diff — which is what turned "should work" into "does work."

**How to apply:** For any guard added to close a race/staleness/ordering bug:
1. Write the adversarial test (the specific bad sequence, not the happy path).
2. Confirm it passes with the guard in place.
3. Temporarily neutralize the guard in the source (comment out the condition or force it false)
   and re-run ONLY that test — it must fail, and fail on the specific field/value the bug produces.
4. Restore the guard immediately, re-run to confirm green again, and grep the diff for any leftover
   disable markers before reporting done.
5. Also write the counterpart test proving the guard doesn't swallow the LEGITIMATE case (a guard
   that blocks everything is as broken as no guard, and is invisible without this second test).

This is cheap (a few extra tool calls) relative to the cost of shipping a guard that silently never
fires. Treat it as mandatory for any fix framed as "closes a race condition" or "prevents stale data
from overwriting fresh data," not just for this project.
