---
name: project_imtithal_visitor_analytics
description: Imtithal (Server/) Visitor Analytics schema (migration 0012) — DB review findings from 2026-08-25, empirically verified with a local temp PostgreSQL 17 database.
metadata:
  type: project
---

Project: Imtithal e-commerce platform, `d:\Work\1-Nodejs\IsoEccomerce\Server`. Feature:
Visitor Analytics (plan at `d:\Work\1-Nodejs\IsoEccomerce\Plans\Visitor-Analytics-Plan.md`).
Tables: `VisitorProfiles`, `VisitorSessions`, `VisitorPageViews` (RANGE-partitioned monthly
on `CreatedAt`, composite PK `(Id, CreatedAt)`), `VisitorDailyStats`. Migration:
`Src/Database/Migrations/0012_lyrical_diamondback.sql` (drizzle-kit output hand-edited to add
partitioning — drizzle-orm cannot express `PARTITION BY`).

**Why:** Reviewed on 2026-08-25 at the T2 (schema-only) stage — no resolvers/services/cron
scripts existed yet querying these tables. Verified everything empirically against a scratch
PostgreSQL 17 DB (`/d/laragon/bin/postgresql/postgresql-17.10-1/bin`, local, no password),
seeded to realistic volume (~50K profiles, 150K sessions, ~1.5M page views across 3 monthly
partitions), rather than reasoning about EXPLAIN output theoretically.

**Findings (for future re-review or once resolvers/cron land):**

1. **Missing index — HIGH severity, confirmed via EXPLAIN.** `VisitorSessions` has no index
   covering `LastActivityAt`. An "active sessions now" widget (`WHERE IsBot=false AND
   LastActivityAt >= now() - interval '30 min'`) forces a full Parallel Seq Scan of the whole
   table — measured 80ms/150K rows even at high selectivity (~40 matching rows), and
   `VisitorSessions` is kept forever (never deleted), so this only gets worse. Adding
   `(IsBot, LastActivityAt)` dropped it to 0.2ms (Index Only Scan) — ~380x. This index does
   **not** exist in migration 0012 as shipped; flag if not added before the "active visitors"
   dashboard widget ships.

2. **Partition pruning works correctly** — verified with EXPLAIN: date-bounded queries only
   touch relevant partitions (compile-time static pruning), Index Only Scans on `IX_VPV_CreatedAt`.

3. **The `+00` UTC partition-boundary fix (documented at length in the migration's own Arabic
   comment) is real and correctly implemented** — reproduced the Cairo-session-timezone trap
   the comment describes, confirmed boundaries render correctly under `SET TIME ZONE 'UTC'`.

4. **The default-partition trap is real** — reproduced exactly: a stray row in
   `VisitorPageViews_default` blocks `CREATE TABLE ... PARTITION OF ... FOR VALUES FROM (...)`
   for that range with Postgres's own "would be violated by some row" error. Confirms the
   13-months-pre-created mitigation is load-bearing; whatever partition-creation cron ships
   later (referenced as "AnalyticsPartitionCron" / S16 in the plan, not written yet as of this
   review) MUST hard-code `+00` and never fall behind.

5. **New finding not yet documented anywhere: "all-time" (non-date-bounded) queries against
   `VisitorPageViews` don't benefit from partition pruning and their planning time scales with
   partition count.** A `WHERE EntityType='Pack' AND EntityId=X` query with no `CreatedAt`
   filter produces an N-way Append (one subplan per partition, including empty future ones);
   at just 13 partitions, Planning Time (60-198ms) already exceeded Execution Time (5-6ms).
   Same root cause applies to any cross-partition query lacking a date bound — `SessionId`
   lookups are naturally bounded (a session can't span months) but `VisitorProfileId`
   ("this visitor's full history") and `EntityId` ("all-time views for this pack") are not.
   Since the table is kept forever with no partition retirement mentioned in the plan, this
   degrades slowly but permanently across years. Worth a note in the eventual
   resolver/service layer: any "all-time" aggregate should either accept the Append cost or
   read from a rollup table instead of `VisitorPageViews` directly.

6. `IX_VPV_Path_CreatedAt` and `IX_VPV_EntityType_EntityId` cost ~8-27% extra bulk-insert time
   (measured 200K-row bulk insert: 11.3s with vs 8.2s without both) but deliver ~100-150x read
   speedups for their actual use case — single-Path / single-Entity point queries, NOT the
   "top N in a date range" aggregate (Postgres correctly prefers seq-scanning the pruned
   partition for that low-selectivity shape). Judged as a justified trade-off given the
   dashboard explicitly wants per-page trend and per-entity view counts, but the bulk-insert
   number likely overstates real single-row-insert overhead (fixed per-statement/WAL cost
   dominates in production, unlike in a bulk-insert benchmark).

7. FK `VisitorPageViews.SessionId → VisitorSessions.Id` costs ~7-8% of bulk-insert time in
   isolation (measured by dropping/re-adding the constraint) — bounded, O(log n) via
   `VisitorSessions`'s PK btree, not a concern. `DETACH PARTITION` empirically confirmed to
   correctly preserve the per-partition FK constraint on the now-standalone table (PG12+
   behavior working as documented in the model file's comment).

8. `VisitorDailyStats`'s `'*'` sentinel + `UQ_VisitorDailyStats_Dimensions` unique index:
   verified idempotent via an actual `ON CONFLICT ... DO UPDATE` test (two upserts of the same
   dimension key correctly accumulated into one row, no duplicate). No trap found — matches
   the file's own claims.

9. Types, naming, `BaseColumns` compliance, 63-char identifier limit, soft-delete/UTC
   conventions: all compliant on inspection — no findings.

**How to apply:** If re-reviewing this schema later (once resolvers/services/
AnalyticsPartitionCron are written), re-check whether finding #1 (`LastActivityAt` index) was
addressed, and whether any new query pattern hits the all-time/no-date-bound issue (#5) —
especially a "lifetime views" or "returning visitor history" feature.
