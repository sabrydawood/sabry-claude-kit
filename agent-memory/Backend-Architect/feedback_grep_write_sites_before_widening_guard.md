---
name: feedback_grep_write_sites_before_widening_guard
description: Before widening a guard condition from "specific value" to "existence" (or any broader check), grep every write site of the field to confirm the equivalence actually holds — don't infer it from one function's local logic.
type: feedback
---

When a guard comment argues "checking X is equivalent to checking existence, because the only writer
of this field always sets it to X" — that claim is a statement about the ENTIRE codebase's write
graph, not just the one function you're looking at. Verify it by grepping every place the field is
ever set (`.values({…})`, `.set({…})`, raw SQL UPDATE/INSERT), not by reading the one insert call
that happens to be nearby.

**Why:** Caught on Est8Core's F-29 fix. A staleness guard in `Billing.StripeWebhooks.Service.ts`
started as `ExistingInvoice?.status === "paid"`, then got widened to bare existence
(`if (ExistingInvoice)`) on the reasoning that the only writer of `stripeInvoiceId` sets
`status: "paid"` at the same time. The advisor flagged this as needing verification before shipping:
if any OTHER write path ever inserted/updated a row with `stripeInvoiceId` set while `status` stayed
`open`, the widened guard would silently swallow a genuine first-time failure on that invoice — the
exact "breaks legitimate downgrades" failure mode the task explicitly required a test for, and the
existing tests (which only exercised a stripeInvoiceId with NO local row at all) would not catch it.
`grep -n "stripeInvoiceId"` across `Server/Src` found exactly one real-value writer, confirming the
premise — the widening shipped as-is.

**How to apply:** Whenever a fix relies on "field F implies condition C" as a shortcut for a more
specific check, grep every write site of F before trusting the shortcut. If a second writer exists
that breaks the equivalence, either revert to the specific/narrower check or add a test that exercises
that second writer's shape. This is a five-minute check that prevents a plausible-sounding comment
from becoming a silent correctness bug.
