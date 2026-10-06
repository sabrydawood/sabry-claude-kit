---
name: feedback_dual_layer_authz_grant_can_be_inert
description: Before granting a newly-split permission key to a role, check whether a SERVICE-layer literal-role check sits underneath the ROUTE-layer permission check — granting the permission alone can be a no-op if a separate, independent role gate still blocks it.
metadata:
  type: feedback
---

When decomposing a bundled permission into a narrower key (e.g. splitting a catalog-management
action out of a general `<entity>.update`), the route-level `RequirePermission(...)` middleware is
not always the only gate. Some services carry their OWN hardcoded role check
(`SOME_ROLE_SET.has(Ctx.Role)` or equivalent) inside the business logic itself, added earlier for a
narrower reason (e.g. "only a manager may edit the tenant's catalog") and never removed when a
permission-based route guard was later added on top. These two gates are independent sources of
truth and can silently drift: adding the new permission key to a role's preset makes the route
guard pass, but the caller still gets 403 from the service layer because their literal role string
isn't in the hardcoded set — a permission that LOOKS grantable but confers no actual capability.

**Why:** Found this while decomposing three bundled permissions in Est8Core (Server/Src). Two of
three splits were clean route-level-only gates. The third — deal-stage-catalog CRUD — had a
pre-existing `MANAGER_ROLES.has(Ctx.Role)` check (`["owner","branch_manager"]`) inside the SERVICE
functions themselves, underneath the route's permission check. The natural derivation ("grant the
new key to every role that already holds the bundle it was split from") would have granted the key
to `team_leader`, which holds the bundle but is NOT in that literal role set — reproducing, in a
brand-new migration, the EXACT anti-pattern a prior migration in the same codebase had already
found and reverted for a different permission (`targets.manage` granted to `team_leader`, but the
write path's scope-level check silently failed closed for that role 100% of the time — "the
permission was a label with no capability behind it"). The sibling permission split in the same
task (property-type-catalog CRUD) had NO such service-layer gate, so the naive "grant to every
bundle-holder" rule was correct there — the two splits looked identical from the route file alone
but required opposite population rules once the service layer was read.

**How to apply:** Before finalizing WHO gets a new/split permission key, grep the service/business
logic (not just the route file) for any independent role check — `Role ===`, `.has(Ctx.Role)`,
`RequireRole(...)`, a hardcoded role-set constant — that already sits on the same write paths the
new permission gates. If one exists, the actually-grantable population is the INTERSECTION of "who
holds the old bundle" and "who passes the hardcoded role check," not the bundle-holders alone —
even if that produces a narrower or differently-shaped population than a sibling split in the same
task. Verify empirically, not just by reading: a targeted integration test asserting a role that
HOLDS the new permission but sits outside the hardcoded set still gets 403 is the only thing that
reliably catches this class of bug (a route-level "wrong role → 403" test does not distinguish
"blocked because no permission" from "would still be blocked even with the permission").
