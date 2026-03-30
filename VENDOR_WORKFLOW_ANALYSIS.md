# Vendor Workflow Analysis — Cethos Translation App

**Date:** 2026-03-30
**Database:** Supabase project `lmzoyezvsjgsxveoakdr` (Cethos_Translation_App)

---

## 1. Current Architecture Overview

### Workflow Templates (9 active)

| Template Code | Name | Steps | Default Service |
|---|---|---|---|
| `certified_translation` | Certified Translation | Translation → Customer Draft Review → PM Review & Certification | Certified Translation |
| `standard_tep` | Standard TEP | Translation → Editing → Proofreading → QA Review | Standard Translation |
| `translation_review` | Translation + Review | Translation → Review → QA Review | Standard Translation |
| `translation_only` | Translation Only | Translation → QA Review | Standard Translation |
| `mtpe_review` | MTPE + Review | MTPE → Review → QA Review | MTPE |
| `medical_back_translation` | Medical / Back Translation | Translation → Back Translation → Clinician Review → Post Clinician Review → Reconciliation | Medical Translation |
| `software_localization` | Localization | Translation → Editing → DTP/Engineering → LQA Testing → QA Review | Software Localization |
| `subtitling` | Subtitling | Transcription → Translation → Subtitling/Timing → QA Review | Subtitling |
| `transcription_translation` | Transcription + Translation | Transcription → Translation → Review → QA Review | Transcription |

### Vendor Database
- **1,463 vendors** (744 active, 719 applicants)
- **5,186 language pairs** configured
- **764 rate entries** across vendors/services
- **45 services** defined

### Edge Functions (54 vendor/workflow-related)

**Vendor Auth & Profile (17):**
`vendor-auth-otp-send`, `vendor-auth-otp-verify`, `vendor-auth-password`, `vendor-auth-session`, `vendor-auth-logout`, `vendor-set-password`, `vendor-auth-check`, `vendor-auth-activate`, `vendor-auth-invite`, `vendor-update-profile`, `vendor-verify-phone`, `vendor-get-profile`, `vendor-update-availability`, `vendor-update-payment-info`, `vendor-update-language-pairs`, `vendor-update-rates`, `vendor-upload-certification`

**Vendor Job Portal (7):**
`vendor-get-jobs`, `vendor-accept-job`, `vendor-decline-job`, `vendor-upload-delivery`, `vendor-get-source-files`, `vendor-get-job-detail`, `vendor-get-invoices`, `vendor-get-invoice-pdf`

**Workflow Engine (6):**
`assign-order-workflow` (v4), `get-order-workflow`, `update-workflow-step` (v12), `manage-workflow-templates`, `manage-order-workflow-steps` (v2), `staff-deliver-step`

**Offer & Negotiation (7):**
`vendor-accept-step`, `vendor-decline-step`, `vendor-deliver-step`, `vendor-counter-offer`, `admin-respond-counter-offer`, `vendor-accept-terms`, `expire-stale-offers`

**Vendor Management (8):**
`get-vendors-list`, `get-vendor-detail`, `update-vendor-rates`, `update-vendor-payment-info`, `find-matching-vendors`, `vendor-manage-rates`, `create-vendor`, `manage-vendor-payables`

**Notifications & Reminders (2):**
`send-workflow-notification`, `vendor-deadline-reminders`

**CVP Pipeline (2):**
`cvp-prescreen-application`, `cvp-submit-application`

**Import/Sync (5):**
`import-applicant-vendors`, `vendor-invitation-reminder`, `import-vendor-lang-rates`, `xtrf-sync-vendor-cache`, `xtrf-sync-vendors`, `xtrf-sync-vendor-lp`

---

## 2. Gap Analysis

### GAP 1: Workflow Adoption — Near-Zero (CRITICAL)

| Metric | Value |
|---|---|
| Total orders | 130 |
| Orders with workflows | 7 (5.4%) |
| Orders delivered with `work_status = pending` | 98 (75%) |
| Workflow steps progressed past pending | 0 |
| Steps with source files attached | 0 |
| Steps with delivered files | 0 |

**Root cause:** Workflows are not auto-created when orders are paid. Staff must manually assign workflows via the admin panel. 123 out of 130 orders have no workflow at all.

**Fix needed:** Trigger `assign-order-workflow` automatically when an order status changes to `paid` or `in_production`. Use the order's `service_id` to select the default template.

---

### GAP 2: CVP (Certified Vendor Pipeline) — Not Operational

| Table | Rows |
|---|---|
| `cvp_applications` | 0 |
| `cvp_test_library` | 0 |
| `cvp_test_submissions` | 0 |
| `cvp_test_combinations` | 0 |
| `cvp_translators` | 0 |
| `cvp_jobs` | 0 |
| `cvp_payments` | 0 |

**Impact:** 719 vendors stuck in "applicant" status with no way to vet, test, or promote them. Edge functions `cvp-prescreen-application` and `cvp-submit-application` exist but have no test content to work with.

**Fix needed:**
1. Populate `cvp_test_library` with test translations per language pair/domain
2. Build application intake form connected to `cvp-submit-application`
3. Wire prescreening → test assignment → scoring → tier promotion pipeline

---

### GAP 3: Step Deliveries — Not Tracked

| Table | Rows |
|---|---|
| `step_deliveries` | 0 |

The `update-workflow-step` edge function (v11+) does write to `step_deliveries` on approve/revision_requested actions, but no deliveries have ever been recorded because no step has ever reached the `delivered` status.

**Fix needed:** Part of Gap 1 — once workflows are running end-to-end, deliveries will be tracked. Also need to ensure `vendor-upload-delivery` and `staff-deliver-step` create records in `step_deliveries`.

---

### GAP 4: Terms & Compliance — Empty

| Table | Rows |
|---|---|
| `vendor_terms_acceptances` | 0 |
| `service_terms` | 2 |

`vendor-accept-terms` edge function exists. But no vendor has ever accepted terms because the offer-to-acceptance pipeline rarely completes.

**Fix needed:** Require terms acceptance as part of the offer acceptance flow. Block step progression until terms are signed.

---

### GAP 5: Vendor Payment Info — Missing

| Table | Rows |
|---|---|
| `vendor_payment_info` | 0 |
| `vendor_payables` | 1 (cancelled) |

`vendor-update-payment-info` and `manage-vendor-payables` edge functions exist. No vendor has entered payout details.

**Fix needed:** Prompt vendors to complete payment info during onboarding/first job acceptance. Block invoice generation until payment info is on file.

---

### GAP 6: Notifications — Minimal

Only **22 notifications** ever sent across 5 event types:

| Event Type | Recipient | Count |
|---|---|---|
| `admin_new_order` | staff | 7 |
| `order_confirmation` | customer | 7 |
| `vendor_missed_deadline` | admin | 4 |
| `payment_reminder` | customer | 3 |
| `vendor_unassigned` | vendor | 1 |

**Missing notification types (supported in code but never triggered):**
- `offer_sent` — vendor receives job offer
- `offer_accepted` — admin notified of acceptance
- `offer_expired` — admin notified when offer expires
- `step_approved` — vendor notified of approval
- `revision_requested` — vendor notified to revise
- `deadline_reminder` — proactive warnings at 50%/75%/90%
- `step_completed` — next actor notified
- `payment_processed` — vendor notified of payment

---

### GAP 7: No Auto-Assignment

All 33 template steps use `assignment_mode = manual` with `auto_assign_rule = null`.

`find-matching-vendors` edge function exists but is never auto-triggered.

**Fix needed:**
1. Implement auto-assign rules: `best_rated`, `lowest_cost`, `fastest_response`, `round_robin`
2. Wire auto-assignment to trigger when a step becomes active
3. Fall back to manual if no suitable vendor found

---

### GAP 8: Deadline Management — NOW ACTIVATED

**Cron jobs were already scheduled (every 15 min):**
- `expire-stale-offers` (job 29)
- `vendor-deadline-reminders` (job 33)
- `check-missed-deadlines` (job 32)

**Upgrades deployed (2026-03-30):**

1. **`vendor-deadline-reminders` v3** — Multi-stage reminders:
   - 48h: "Due in less than 48 hours"
   - 24h: "Due in less than 24 hours"
   - 4h: "Due in less than 4 hours" (urgent styling)
   - 1h: "URGENT — due in less than 1 hour" (red styling)
   - Each stage fires only once per step (tracked via `metadata.reminder_stage`)
   - Supports both external vendors and internal staff

2. **`check-missed-deadlines` v2** — Now notifies both:
   - Admin: `vendor_missed_deadline` event (as before)
   - Vendor: `deadline_reminder` with `stage: overdue` and "OVERDUE" styling

3. **`send-workflow-notification` v2** — Enhanced deadline email template:
   - Urgency-based color coding (blue → amber → red)
   - Different subject prefixes: Reminder / URGENT / OVERDUE
   - Styled alert banner with stage-appropriate icons

---

### GAP 9: Quality Scoring — No Feedback Loop

- `vendors.rating` field exists but is never updated
- No per-job quality scoring after QA Review step
- No mechanism to update vendor tier based on performance

**Fix needed:**
1. Add quality scoring form to the QA Review step
2. Aggregate scores into `vendors.rating` after each completed job
3. Factor rating into auto-assignment ranking

---

## 3. Console Error: `update-workflow-step` 404

The edge function is deployed as **ACTIVE** (v12, last updated 2026-03-25). Possible causes:
1. **Cold start timeout** — function may have hit a cold start limit
2. **Redeployment needed** — try redeploying the function
3. **Supabase edge runtime issue** — transient 404s can occur during function updates

**Recommended action:** Redeploy `update-workflow-step` via Supabase dashboard or CLI.

---

## 4. Recommended Implementation Roadmap

### Phase 1: Make Existing Workflow Engine Operational (Priority: CRITICAL)

1. **Auto-create workflow on order payment** — database trigger or webhook on `orders` status change
2. **Auto-attach source files** — copy `quote_files` to step 1 `source_file_paths`
3. **Redeploy `update-workflow-step`** — fix the 404
4. **Schedule `expire-stale-offers`** — cron every 5 min
5. **Schedule `vendor-deadline-reminders`** — cron every 15 min

### Phase 2: Vendor Portal Activation (Priority: HIGH)

6. **Vendor invitation campaign** — use `vendor-auth-invite` for 744 active vendors
7. **Vendor onboarding flow** — profile completion, payment info, terms acceptance
8. **Offer notification emails** — ensure `send-workflow-notification` sends real emails
9. **Vendor job dashboard** — wire up `vendor-get-jobs`, `vendor-get-job-detail`

### Phase 3: Smart Assignment (Priority: MEDIUM)

10. **Implement auto-assign rules** in `update-workflow-step`
11. **Bulk offer broadcasting** — use `offer_multiple` action
12. **Vendor availability tracking** — use `vendor-update-availability`
13. **Rate lookup integration** — pre-populate offers from `vendor_rates`

### Phase 4: CVP Pipeline (Priority: MEDIUM)

14. **Create test translation library** — populate `cvp_test_library`
15. **Application intake form** — public-facing, feeds `cvp_applications`
16. **AI prescreening** — activate `cvp-prescreen-application`
17. **Test → Score → Promote pipeline**

### Phase 5: Quality & Payment Loop (Priority: LOWER)

18. **QA scoring per step** — add scoring UI to QA Review
19. **Vendor rating aggregation** — rolling average updates
20. **Payable lifecycle automation** — pending → approved → invoiced → paid
21. **Vendor self-service invoicing**

---

## 5. Summary Scorecard

| Area | Schema | Edge Functions | Data | Working? |
|---|---|---|---|---|
| Workflow Templates | Yes (9) | Yes (6) | Yes | Partially |
| Order → Workflow Binding | Yes | Yes (`assign-order-workflow`) | 5.4% | No |
| Step Progression | Yes | Yes (`update-workflow-step` v12) | All pending | No |
| Vendor Offers & Negotiation | Yes | Yes (7 functions) | 6 offers | Partially |
| File Delivery per Step | Yes | Yes (`vendor-deliver-step`, `staff-deliver-step`) | 0 rows | No |
| Vendor Payables | Yes | Yes (`manage-vendor-payables`) | 1 (cancelled) | No |
| CVP / Vendor Vetting | Yes (7 tables) | Yes (2 functions) | All empty | No |
| Terms & Compliance | Yes | Yes (`vendor-accept-terms`) | 0 rows | No |
| Vendor Auth & Portal | Yes | Yes (17 functions) | 1 auth, 10 sessions | Barely |
| Notifications | Partial | Yes (`send-workflow-notification`) | 22 sent | Barely |
| Auto-Assignment | Schema ready | `find-matching-vendors` exists | Not configured | No |
| Quality Scoring | Field exists | Not built | Empty | No |
| Deadline Management | Schema ready | 2 functions exist | Reactive only | Barely |
| XTRF Sync | Yes | Yes (4 functions) | 760 vendors synced | Working |

**Overall Assessment:** The database schema and edge functions are ~80% complete. The system is well-architected but only ~5% operational. The #1 priority is auto-creating workflows on order payment and getting the first end-to-end workflow running.
