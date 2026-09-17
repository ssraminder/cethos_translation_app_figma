# Portal Slowness — Incident Note, 2026-09-17

**Reported:** portal.cethos.com slow for several days, long load times.
**Database:** Supabase project `lmzoyezvsjgsxveoakdr` (Cethos_Translation_App)
**Vercel project:** `cethos-app-figma-design-v1` (prj_ZmRYFSKmtj82CSdEDCPL9g0itpKE)
**Status:** Root cause fixed and verified. Follow-ups below are open.

---

## 1. Summary

The slowness was entirely database-side. **83% of the database was garbage** —
2.6 GB of 3.17 GB was dead weight in two maintenance tables that are never
purged by default. The working set no longer fit in cache, so ordinary queries
went to disk, PostgREST could not load its schema cache within timeout, and the
portal received periodic HTTP 503 storms.

Purging and rewriting those two tables took the database from **3,167 MB to
586 MB** and cut p99 request latency from ~5–8 s to under 800 ms.

Vercel was not involved. The portal is a static Vite SPA on CDN with no
meaningful serverless runtime — zero runtime errors over 7 days, zero runtime
log lines, all production deploys `READY`.

---

## 2. Root cause

### 2.1 Unbounded maintenance tables

| Table | Size before | Live rows | Cause |
|---|---|---|---|
| `cron.job_run_details` | 1,458 MB | 1,956,622 | pg_cron run history, never purged since 2026-01-20 |
| `net._http_response` | 1,160 MB | 2,886 | pg_net response bloat — 413 KB/row of empty pages |

`net._http_response` had only 5 dead tuples, so autovacuum had nothing to
collect: the space was in empty pages that plain `VACUUM` cannot truncate.
Supabase's own performance advisor independently flagged it
("Table `net`.`_http_response` has excessive bloat").

### 2.2 Downstream effects

PostgREST could not complete schema-cache introspection (486 relations, 440
relationships, 293 functions) against a saturated database:

- `PGRST002 Could not query the database for the schema cache. Retrying.` — 280× in 24 h
- PostgREST restarted ~20× in 24 h
- HTTP 503 returned to portal.cethos.com in bursts (167 at 17:00, 92 at 23:00, 25 at 13:00)
- `Warp server error: Thread killed by timeout manager` — 217×
- Postgres `canceling statement due to statement timeout` throughout the day
- Simultaneous `cron job N job startup timeout` across ~10 jobs at 10:00 (background worker exhaustion)

Latency tracked load exactly — quiet hours avg 240 ms / p99 1.4 s, business
hours avg 400–620 ms / p99 5–8 s (peak 13.1 s).

---

## 3. What was changed

All changes were to the production database. No application code changed.

1. **Purged `cron.job_run_details`** to a 7-day window — ~1.88M rows deleted in
   six index-ranged batches on the `runid` primary key (avoids seq-scanning
   1.4 GB per batch). 77,639 rows retained.
2. **`VACUUM FULL (ANALYZE)`** on `cron.job_run_details` and `net._http_response`
   to return empty pages to the OS.
3. **Added retention job** `cron-job-run-details-purge-7d` (jobid 1872, daily at
   `30 4 * * *`), following the existing staggered 03:00–04:00 purge-job
   convention.

### Result

| | Before | After |
|---|---|---|
| `cron.job_run_details` | 1,458 MB | 33 MB |
| `net._http_response` | 1,160 MB | 3.5 MB |
| **Database total** | **3,167 MB** | **586 MB** |

Latency across the maintenance window:

| 5-min bucket | avg | p95 | p99 |
|---|---|---|---|
| 20:15 (before) | 894 ms | 3,987 ms | 8,903 ms |
| 20:30 (before) | 318 ms | 1,091 ms | 3,503 ms |
| 20:35 (after) | 138 ms | 314 ms | 768 ms |
| 20:40 (after) | 141 ms | 316 ms | 666 ms |
| 20:45 (after) | 146 ms | 329 ms | 756 ms |

pg_cron continued logging normally throughout (31 runs in the 3 minutes after
the rewrite). Post-change figures cover ~15 minutes against tapering evening
traffic — **confirm at the next business-hours peak.**

---

## 4. Open follow-ups

### 4.1 Frontend polling storm (highest remaining impact)

Roughly **half of all Supabase requests** are a fixed-interval badge-count poll
from every open portal tab. Over 24 h:

| Endpoint | Requests |
|---|---|
| `/rest/v1/staff_users` | 19,314 |
| `/rest/v1/cvp_inbound_emails` | 13,688 |
| `/rest/v1/qms_vendor_status` (view) | 13,598 |
| `/rest/v1/cvp_test_combinations` | 13,596 |
| `/rest/v1/bug_reports` | 13,587 |
| `/rest/v1/cvp_frontdesk_escalations` | 13,414 |

~87k of 180k total requests. These are PostgREST `HEAD ... count=exact` calls,
so each forces a full count, and two of the six are views.

**Fix:** one RPC returning all six counts in a single round trip, with estimated
rather than exact counts. Lives in `cethos_app_figma_design_v1`.

### 4.2 Three cron jobs that have never succeeded

**0 successes across 3,363 runs** in all retained history. Four more broken jobs — the storage
retention purges — are covered in section 5.2, bringing the total to **seven**.

| Job | Schedule | Failure |
|---|---|---|
| 3 `check-sla-deadlines` | `*/15 * * * *` | `invalid input syntax for type json` |
| 5 `process-invoice-queue` | `*/5 * * * *` | `invalid input syntax for type json` |
| 904 `vendor-acceptance-reminders` | `*/15 * * * *` | `unrecognized configuration parameter "app.settings.supabase_url"` |

Jobs 3 and 5 call `net.http_post(url, body, 'application/json')` positionally,
but the third positional parameter is `params jsonb`, not a content type. They
fail while parsing arguments and never issue an HTTP request.

Job 5 additionally has a typo in its URL — `lmzoyezvsijgsxveoakdr` instead of
`lmzoyezvsjgsxveoakdr`. Latent: it would only surface once the JSON bug is fixed.

Job 904 is the only job referencing `app.settings.supabase_url`; every other job
hardcodes the URL. Fix by matching that convention.

> **Do not simply repair these jobs.** In each case the broken call is holding
> back a backlog, and fixing the transport without first handling the payload
> would cause real damage. Details below.

#### Job 5 — 378 unbilled orders (finance decision)

`process-invoice-queue` reads `invoice_generation_queue` (10 rows/run), calls
`create_invoice_from_order`, then `generate-invoice-pdf`.

| status | count | oldest | newest |
|---|---|---|---|
| pending | **378** | 2026-06-12 | 2026-09-17 |
| failed | 314 | 2026-02-13 | 2026-06-11 |
| completed | **1** | 2026-02-02 | 2026-02-02 |

The queue has produced one invoice, in February, and is still being fed daily.
Repairing the job would generate all 378 invoices in ~3 hours, including orders
delivered in June. The function creates invoices and PDFs but does not email
them, and no cron auto-sends customer invoices — so the blast radius is rows and
stored PDFs, not 378 customer emails. Still requires a finance decision on
back-dated invoicing before enabling.

#### Job 904 — 12 orphaned workflow steps (ops cleanup first)

`vendor-acceptance-reminders` sweeps `order_workflow_steps` where
`status='assigned'`, and emails the vendor at 1 h and the vendor + pm@cethoscorp.com
at 2 h, deduped on `(step_id, event_type)` in `notification_log`.

There are 12 such steps across 10 vendors, all older than 2 h, oldest assigned
2026-06-09 — and `notification_log` holds **0** acceptance-reminder dedup rows.
A first successful run would fire both tiers on all 12 at once: ~24 emails, 12
CC'ing the PM.

Critically, **all 12 belong to orders already `completed`, `delivered`, `paid` or
`balance_due`**; two have `work_status = completed`; two are test records
(`ORD-TRAIN-001`, vendor `DUMMY · EN→HI · #2`). None is live work awaiting
acceptance. These are orphaned rows where `status` was never advanced when the
work completed.

**Order of work:** close the 12 orphaned steps, then apply the URL fix.

This also matters beyond the reminder job — steps reading `assigned` on delivered
orders will skew any reporting built on `order_workflow_steps.status`.

#### Job 3 — source unavailable

The Supabase API returns `Failed to retrieve function bundle` for
`check-sla-deadlines` (retried twice), so its behaviour was not verified. It is
**not** what sends deadline reminders — those are alive from a different job
(`deadline_overdue_admin` 795, `deadline_reminder_6h` 345, `deadline_overdue` 265,
`deadline_reminder_24h` 236, most recent 2026-09-16). Read the source in the repo
before enabling.

Separately, job 2 `check-hitl-sla-breaches` is plain SQL and is running — 56 open
`hitl_reviews`, 31 flagged breached.

### 4.3 Autovacuum tuning — blocked

Per-table autovacuum tuning on the two bloat tables could not be applied: both
are owned by `supabase_admin` and the MCP connection is `postgres`.

```
ALTER TABLE cron.job_run_details SET (autovacuum_vacuum_scale_factor = 0.05);
ALTER TABLE net._http_response  SET (autovacuum_vacuum_scale_factor = 0.02);
```

Requires Supabase support. Lower priority now that the retention job keeps
`job_run_details` small enough for default thresholds to fire, but
`net._http_response` bloated once and could again.

### 4.4 Schema performance debt

From `get_advisors(type: performance)`:

| Finding | Count | Level |
|---|---|---|
| Multiple Permissive Policies | 634 | WARN |
| Auth RLS Initialization Plan | 168 | WARN |
| Unindexed foreign keys | 458 | INFO |
| Unused Index | 304 | INFO |
| Duplicate Index | 23 | WARN |

The RLS initplan warnings mean `auth.uid()` is re-evaluated per row rather than
wrapped as `(select auth.uid())`. Together with the duplicated permissive
policies, these multiply per-row cost on every query. `orders` carries 26 indexes
over 1,277 rows.

Also flagged: Auth is on absolute connection allocation (max 10), so increasing
instance size will not improve Auth throughput without switching to percentage-based
allocation.

---

## 5. Storage and backup

Investigated separately on the same day, prompted by the question of whether file storage was also
full and whether old files could be purged. **Storage was not a contributor to the slowdown** — object
bytes live in S3, not in the Postgres instance, so storage size has no effect on database or portal
performance.

### 5.1 Current state

**28 GB across 25,853 objects in 47 buckets.** Not close to full; Supabase Pro includes 100 GB.

Largest buckets: `quote-files` 10 GB (8,149 objects), `order-files` 3,786 MB, `ocr-uploads` 2,761 MB,
`careers-videos` 1,924 MB, `public-submissions` 1,778 MB, `transcription-uploads` 1,620 MB.

Objects older than 6 months total **2,594 MB / 2,092 objects — about 9% of storage**:

| Bucket | Objects >6mo | Size |
|---|---|---|
| `quote-files` | 1,107 | 1,653 MB |
| `ocr-uploads` | 202 | 686 MB |
| `blog-post-images` | 84 | 165 MB |
| `cethosweb-quote-files` | 39 | 30 MB |
| `message-attachments` | 22 | 19 MB |
| `quote-reference-files` | 23 | 17 MB |
| `logos` | 527 | 11 MB |
| others | 88 | ~13 MB |

### 5.2 The storage retention policy has never run

Four purge jobs fail every day — 7/7 runs in retained history:

| Job | Schedule | Target |
|---|---|---|
| 44 `pdf-to-word-purge-120d` | `0 3 * * *` | `pdf-to-word`, 120 days |
| 45 `public-submissions-purge-180d` | `15 3 * * *` | `public-submissions`, 180 days |
| 46 `public-submissions-quarantine-purge-180d` | `30 3 * * *` | quarantine, 180 days |
| 47 `customer-files-purge-365d` | `45 3 * * *` | `customer-files`, 365 days |

```
ERROR: Direct deletion from storage tables is not allowed. Use the Storage API instead.
```

`public.purge_storage_bucket(p_bucket_id, p_age_days)` issues a raw
`DELETE FROM storage.objects`, which Supabase now blocks. So the documented retention policy is not
being enforced on any bucket.

**Do not simply repair this function.** Had the raw DELETE succeeded it would have removed metadata
rows while leaving the bytes in S3 — still billed, permanently unreferenced, unrecoverable. Any
replacement must go through the Storage API. `transcription-cleanup` already demonstrates the correct
pattern (`admin.storage.from(...).remove(...)`), though note it deletes permanently with no archive step.

### 5.3 Backup: an AWS S3 replica does exist

It is **not configured inside the Supabase project**, which is why an in-project search finds nothing:
no `wrappers`/`aws_s3` extension, no foreign servers, no AWS credentials among the 24 vault secrets,
no analytics or external buckets, and no AWS reference in any cron job. Cross-region S3 replication is
configured at the bucket/platform layer, invisible from Postgres and edge functions.

**SOP-017 "Business Continuity and Disaster Recovery" v3 (active)** specifies it:

> Object/file storage replicated daily to a separate region (versioned, 90-day retention;
> file-recovery tested).

> **S2 Storage loss** — restore objects from the versioned replica (pre-incident version); verify a
> known file. RTO <= 24 h.

Last verified **27 Jun 2026** — an S3 cross-region replica sample-restore, byte-identical, recorded in
**CTS-REC-RST-004**, alongside an isolated-branch DR restore with re-verified hash-chain. BC tabletop
script recorded as CTS-REC-BCT-001.

SOP-017 names **Cital Enterprises** as contracted IT operations — the likely owner of the replica
configuration, and where to obtain the bucket, lifecycle rule and current sync status.

### 5.4 Why a flat 6-month purge is not advisable

1. **90-day retention is disaster recovery, not archive.** The replica exists to restore after loss or
   a region outage. Purging files older than 6 months would leave the replica as the only copy, expiring
   90 days later — after which those files are gone everywhere.
2. **Records retention is the binding constraint, not disk space.** SOP-017 records a 7-year retain
   policy on Microsoft 365 / Google Drive. Deleting client source and target files at 6 months may
   conflict with ISO 17100 record requirements or Welo/IQVIA contractual commitments. This is a QM
   decision, not a storage one.
3. **The reclaim is small.** 2.5 GB of 28 GB, against a 100 GB allowance. Nothing forces the decision.

### 5.5 Dropbox sync covers only post-May-2026 files

Separate from the S3 replica, `dropbox_file_syncs` records a Dropbox copy driven by `dropbox-sync`,
`dropbox-team-sync` and `qms-dropbox-sync` (jobid 1850, weekly Sundays 03:30).

Coverage does not extend to the old cohort: of the 2,092 objects older than 6 months, **2 have a
successful sync record**. The earliest sync record of any kind is **2026-05-23** — the sync was switched
on in late May, so everything in the >6-month cohort predates it.

Current sync health also has gaps: **624 failed syncs** (548 team, 76 legacy) and 69 pending.

### 5.6 Storage follow-ups

1. Confirm with Cital that cross-region replication is still running — the last evidence is 27 Jun,
   roughly three months old, and nothing inside Supabase monitors it. Given that four purge jobs and
   three other crons have been failing silently for months, this blind spot is the pattern, not an
   outlier.
2. Have QM rule on retention obligations per document class before any deletion.
3. Replace `purge_storage_bucket()` with a Storage API implementation, gated on per-bucket retention
   that reflects those rules rather than a flat cutoff.
4. Fix the 624 failed Dropbox syncs.
5. Correct SOP-017 at next review: it states "Production database ~1.4 GB". It had grown to 3.17 GB
   before this incident and is 586 MB after. That figure feeds the stated RTO/RPO.

---

## 6. Monitoring

Watch these; each was a leading indicator this time:

- `pg_total_relation_size('cron.job_run_details')` and `net._http_response` — alert above ~200 MB
- `PGRST002` count in `postgrest_logs` — should be zero
- PostgREST restart count (`Connection Pool initialized` in `postgrest_logs`) — should be rare
- edge_logs p99 via `log_attributes['response.origin_time']` — was 5–8 s, should stay under ~1 s

Note that the Supabase log API caps queries at a 24-hour window, so trend
analysis beyond a day needs an external sink.
