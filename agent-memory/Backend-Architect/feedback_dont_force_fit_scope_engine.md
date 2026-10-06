---
name: feedback_dont_force_fit_scope_engine
description: Before reusing an existing role-based scope/authz engine (e.g. an all/branch/team/own matrix) for a new entity, verify the entity's data model actually has the columns the engine's levels need — a mismatched level silently degrades to fail-closed (denies everyone) or fail-open (over-broad) for roles the engine wasn't built to handle for this entity.
metadata:
  type: feedback
---

A repo may have a generic, well-designed scope/authorization engine (e.g. a role→level matrix like
`owner→all, branch_manager→branch, team_leader→team, sales_agent→own`, applied via a shared
predicate builder). The instinct "don't invent a new mechanism, reuse the existing one" is usually
right — but reusing it blindly for an entity whose data model doesn't carry the columns those
levels need produces silently wrong behavior, not a compile error:

- A "branch" level with no `branchCol` on the entity typically falls back to a fail-closed
  DENY_ALL sentinel — a role that should see SOMETHING (their own records, say) sees NOTHING.
- A "team" level with no team concept on the entity may fall back to a broader predicate (e.g.
  "anyone on my team", via an ownCols/ownJoin fallback) — silently WIDENING visibility beyond
  what was actually decided for that entity.

Both failure modes are worse than just not using the engine, because they look like reuse
(no new code, "we followed the existing pattern") while actually shipping behavior nobody asked
for and that a permission-key audit script won't catch (it's a runtime `SQL` predicate, not a
missing route guard).

**Why:** In Est8Core, the RBAC scope engine's `SCOPE_MATRIX` (`Core/Authz/Scope/ScopeLevel.ts`)
maps 4 roles to 4 levels, applied uniformly across tenant-schema entities via
`EntityScopeMeta`/`BuildReadPredicate`. Support Tickets live in a cross-tenant MASTER-db table with
no `branchId`/`teamId` columns at all — Sabry's actual decision for that entity was a strict binary
(owner/"*" sees all, everyone else sees only what THEY created), not the 4-level matrix. Plugging
Support into the matrix as-is would have DENY_ALL'd branch_manager/viewer and over-widened
team_leader to "everything my team ever opened" — both wrong, and both invisible without tracing
`BranchPredicate`'s fallback and `TeamPredicate`'s `HasOwn(Meta)` branch by hand.

**How to apply:** Before wiring an entity into an existing scope engine, read the engine's
predicate-building code (not just its public entry point) and check, for EVERY role the matrix
maps to a level, whether this entity's model has the column/join that level needs. If the actual
authorization spec for this entity is simpler than the engine's full matrix (e.g. a two-tier
"admin sees all / everyone else sees own" instead of a four-tier branch/team hierarchy), it's
usually correct to reuse only the SMALLEST shared primitive the engine itself is built on (e.g. the
`Permissions.includes("*")` owner-detection closure) rather than the whole matrix — and to say so
explicitly, with the specific fallback behavior that would have gone wrong, rather than silently
picking one or the other.
