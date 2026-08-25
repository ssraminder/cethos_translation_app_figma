# QMS Audit — Welo Life Sciences Partner Audit Readiness

**Audit date:** 25 August 2026
**Prepared for:** Raminder Shah, Founder & CEO
**Trigger:** Welo Life Sciences (Welocalize) partner audit — Lorena Preduna, Partner Experience Manager, 24 Aug 2026
**Questionnaire:** CF-005.025 Supplier Audit Questionnaire — ClinRev / CogDeb — Cethos 2026
**Committed return date:** Monday 31 August 2026
**Scope:** Cognitive Debriefing and Clinician Review services; the whole QMS supporting them
**Reference standards named by Welo:** ISPOR; ISO 9001:2015; ISO 13485:2016; ISO 17100:2015

---

## 1. Method

This audit tested the QMS **against the live system of record**, not against the document set. QM-002 §1
names the Cethos portal SOP registry as "the single source of record", so every claim below was verified by
direct query of the portal database (Supabase project `lmzoyezvsjgsxveoakdr`) on 25 Aug 2026, and
cross-read against the controlled documents in SharePoint and the closed IQVIA audit file
(AUD-VEND-2026-000015).

Findings use IQVIA's severity taxonomy, because that is the language Cethos has already answered in and
Welo's questionnaire covers the same ground.

---

## 2. Verdict

The QMS is real, documented and, in most control areas, genuinely strong: 44 active SOPs, an immutable
document registry, row-level security on 386 of 387 tables, tested backup/restore, 4,151 vendor
e-signatures, and a closed IQVIA qualification audit with **no critical and no major findings**.

There is also a substantial staff and linguist training pipeline — 32 courses, 175 lessons, 175 quiz
questions and **2,740 recorded completions between 26 June and 24 August 2026**, 1,894 of them carrying
completion IP and user-agent metadata. It is live and running as recently as yesterday.

The exposure is narrower than the framework suggests: **the clinician cohort is the one population the
training pipeline never reached**, and demonstrable corrective action has no live register. Both touch
questions Welo asks directly, and the GCP element is an open IQVIA commitment that falls due this week.

| # | Finding | Severity | Welo question(s) hit |
|---|---|---|---|
| F-1 | Clinician roster does not meet Cethos's own "Qualified" bar in SOP-019 §6.2 | **Major** | 2.1 all six |
| F-2 | GCP training exists and runs, but has not reached staff or clinicians | Minor | General Q2, Q3, Q4 |
| F-3 | IQVIA CAPA closure statement overstates document-control coverage | **Major** | QMS Q6, Q7 |
| F-4 | SOP register has drifted from the live registry again | Minor | QMS Q6 |
| F-5 | Welo asks about focus groups; Cethos documents individual interviews | Minor | 2.2 Q2, Q6 |
| F-6 | No live CAPA / nonconformity register in the portal | Minor | QMS Q2, Q3 |
| F-7 | One public table without row-level security | Observation | IT Q4 |
| F-8 | Quality function is not independent of the CEO | Observation | QMS Q1 |
| F-9 | Two divergent training-completion stores, plus dead legacy schema | Minor | General Q2 |

---

## 3. Findings

### F-1 — Clinician and interviewer qualification evidence (Major)

SOP-019 §6.2 sets Cethos's own bar: a Clinical Validation roster member is **Qualified** only when the CV is
supported by **at least one verifiable evidence item plus GCP**. Until then the member is **Provisional**, and
§6.2 states plainly: *"A Provisional member is never presented to a client or auditor as 'qualified.'"*

Live roster, 25 Aug 2026:

| Measure | Value |
|---|---|
| Clinicians on roster (active) | 37 (36) |
| With a CV on file | 29 of 37 |
| **With `gcp_trained` recorded** | **0 of 37** |
| With at least one *verified* credential | 8 of 37 |
| Credentials recorded / verified | 88 / 22 |
| With independence declared | 0 of 37 |

**Consequence:** on the live data, **no roster member currently meets the SOP-019 §6.2 Qualified bar**,
because the GCP limb is unrecorded for everyone. Every clinician and interviewer is therefore Provisional,
and Provisional deployment is only defensible under §6.3 — 100% independent internal QA review of every
deliverable before release.

This is the single most important item. Welo's section 2.1 asks six consecutive questions about exactly
this, including *"Are CVs and qualifications kept for all Clinicians? Can these be shared with Welocalize
and/or the Clinical Trial Sponsor if necessary for audit purposes?"* Answering "yes, all our clinicians are
qualified" would contradict Cethos's own SOP if tested.

**Action before 31 Aug:** option (a) is more achievable than it first appears. A working GCP course
(`gcp-clinical-linguists`) already exists and 44 linguists have completed it since June — the clinicians were
simply never enrolled (F-2). Assign it to the clinicians working Welo projects, verify at least one evidence
item each, and a defensible subset becomes genuinely Qualified within the window. Failing that, (b) answer
honestly on the Provisional + 100%-QA compensating-control basis. The 8 clinicians with no CV on file should
be chased or made inactive either way.

Note one wiring defect behind this: `clinician_roster.gcp_trained` reads a manually-set boolean on
`clinician_profiles` and is **not connected to the training system at all**. Even once a clinician completes
GCP, the qualification record will not update unless someone sets the flag by hand. Fix the link, or the same
gap reappears at the next audit.

### F-2 — GCP training exists and runs, but has not reached staff or clinicians (Minor)

**This finding was originally recorded as a major "no training records exist". That was wrong** — it was
based on the legacy `training_modules` / `staff_training_progress` tables, which are empty dead schema. The
live pipeline is the CVP training system, and it is substantial:

| Measure | Value |
|---|---|
| Courses (18 staff-audience, 14 linguist-audience) | 32 |
| Lessons / quiz questions | 175 / 175 |
| Assignments | 3,240 (135 staff across 16 people; 3,105 vendor across 1,935) |
| **Completions recorded** | **2,740** (26 Jun – 24 Aug 2026, still running) |
| Completions with IP + user-agent captured | 1,894 |
| Completions with a quiz score (pass bar 80%) | 692 |

GCP training is not missing either. Two courses exist — `gcp-clinical-linguists` and
`gcp-clinical-project-staff` — and **44 linguists have completed GCP since 26 June, the most recent on
24 August**.

What is genuinely wrong is narrower and sharper:

1. **The staff GCP course is mis-audienced.** `gcp-clinical-project-staff` is titled for clinical project
   staff but carries `audience = 'linguist'`. Eight staff are assigned to it and **zero have completed it**.
   The voluntary ICH E6 (R3) commitment made to IQVIA is due **31 August** — six days away, with 0 of 8 done.
2. **No clinician has been assigned GCP at all.** Of the 44 vendors who completed GCP, **none is on the
   Clinical Validation roster**. The 37 clinicians were never enrolled. This is the direct cause of F-1.

**Action:** re-audience `gcp-clinical-project-staff` to `staff` and chase the 8 assigned completions before
31 Aug; assign `gcp-clinical-linguists` to the clinician roster. The course content, quiz, pass threshold and
completion-recording all already exist — this is enrolment and follow-through, not build work, which is why
it is achievable inside the window.

### F-3 — IQVIA CAPA closure statement overstates coverage (Major)

The completed CAPA form for AUD-VEND-2026-000015 states two things that the live system does not support
as written:

1. *"A post-completion verification confirmed that no active controlled document remains without a recorded
   reviewer and approver on its current version."*
   **Live:** 44 controlled documents. Sign-off is enforced (`requires_signoff = true`) on **3** — QM-001,
   QM-002, QP-001. **41 current file versions carry no recorded reviewer and no recorded approver**,
   including ORG-001, CSV-001, CSV-002, SOP-LV-001, JD-001, STMT-001/002 and every register.

2. *"Publication of an unsigned controlled document is thereby impossible by construction."*
   **Live:** true only for those 3 documents. Any other controlled document can be published without a
   signature chain.

Separately, the CAPA describes a chain "prepared, independently reviewed, and approved by **distinct**
signatories". On QM-001 v5.8 the submitter and the reviewer are **both Amrita Shah**.

**Why this matters more than the original finding:** the original was one blank signature block — a minor.
A CAPA that reads as broader than what was implemented is an *ineffective CAPA*, which auditors escalate.
Welo asks whether processes are documented and whether SOP training is carried out and documented; IQVIA
may also verify closure. Fix the system, or restate the scope precisely.

**Action:** set `requires_signoff` on the remaining controlled documents and run the sign-off chain, or
re-scope the closure statement to "the three QMS Core documents" and log the rest as a tracked action.
Correct the QM-001 v5.8 reviewer so preparer and reviewer are distinct.

### F-4 — SOP register drift (Minor)

QM-002 v6.7 states the QMS comprises **40 active SOPs and 1 draft**. The live registry holds **44 active
SOPs**. SOP-003 is `6.2` live against `6.1` in the register.

This is the same failure mode as CAPA-2026-001 ("live SOP register had diverged from QM-002"), which was
closed in June. Recurrence within two months indicates the reconciliation is event-driven, not periodic.

**Action:** reconcile QM-002 to the live registry before sending anything to Welo — the SOP index is an
explicit questionnaire attachment, and a register that miscounts its own SOPs is the first thing an auditor
notices. Add a periodic reconciliation to SOP-001.

### F-5 — Focus groups vs individual interviews (Minor)

Welo's section 2.2 assumes focus groups: *"How are the focus groups organized? How many members are
included in each focus group? … Are the focus group meetings done face-to-face or remotely?"* and asks
about **Focus Group Moderators**.

SOP-008 documents Cethos's method as **individual structured cognitive interviews**, default **5–8
respondents per language**, each interview documented separately, with findings traced to interview
evidence. That is ISPOR-consistent and is the better answer — but it is not what the form assumes.
Meanwhile the one focus-group artefact that exists, **CTS-COA-CD-FG-001 "Cognitive Debriefing Focus Group
Moderator Instructions", is unpublished and uncontrolled** in the portal.

**Action:** answer 2.2 on Cethos's actual methodology and say why (ISPOR; individual interviews avoid group
effects contaminating comprehension data), noting capability to run group sessions where a sponsor protocol
requires. Either publish CTS-COA-CD-FG-001 as a controlled document or leave it out of the pack — do not
cite an uncontrolled document to an auditor.

### F-6 — No live CAPA / nonconformity register (Minor)

REG-AF-001 states open nonconformities are "tracked live in the QMS (/admin/quality)". The portal database
contains **no CAPA, nonconformity, deviation or complaint table**. Tracking is document-based, in a register
last updated 27 June 2026 — while NC-2026-00004's own corrective action was *"establish routine CAPA
recording."*

**This is the finding with the sharpest commercial edge.** Lorena's email pairs the audit with *"your team's
recent performance across our projects… trends we would like to review together."* The August mailbox shows
what those trends are: 2604_P2180 running four days past deadline with six languages still missing after
delivery had been confirmed, a Junction task left unclosed and blocking Welo's pipeline, and the
IQVIA/TransPerfect Ghana cognitive-debriefing work overdue with escalations. Welo's PM has already written
*"such updates should have been communicated before we have reached out to follow up on delivery."*

Welo will ask what corrective action was taken. The QMS answer to a delivery-performance question is a CAPA
record — and there is no live CAPA system to show one in.

**Action:** open a formal CAPA on on-time delivery and client communication **now**, with root cause,
containment and measurable actions, so that by the meeting there is a real, dated, in-progress record. This
turns the weakest part of the conversation into evidence that the corrective-action system works.

### F-7 — Table without row-level security (Observation)

386 of 387 public tables have RLS enabled. The exception is `_tm_reconcile_staging`, a staging table. IQVIA
was told "293/293". Drop it or enable RLS before an auditor runs the same query.

### F-8 — Independence of the quality function (Observation)

Welo asks: *"Do you have an independent Quality function within the company?"*

Position has improved — QM-002 v6.7 now shows Fayza Elbezzari as Quality Manager (preparer), Amrita Shah as
reviewer, Raminder Shah as approver, replacing the earlier "Acting QM = Raminder Shah". But Raminder remains
Founder/CEO **and** final approver on all three QMS Core documents, and REG-AF-001 was prepared and approved
by the same person.

**Action:** answer accurately — a designated Quality Manager independent of project delivery, with CEO
approval authority, proportionate to company size. Do not claim organisational independence Cethos does not
have; auditors accept proportionality, they do not accept overstatement.

### F-9 — Two divergent training-completion stores, plus dead legacy schema (Minor)

Training completion is recorded in two places that disagree. `cvp_training_assignments.completed_at` reports
**0** completions for both GCP courses, while `cvp_training_completions` reports **45**. Separately, the
legacy `training_modules`, `training_lessons`, `training_slides` and `staff_training_progress` tables are
present but entirely empty.

An auditor — or an internal reviewer preparing a response — who queries the wrong table concludes that no
training exists. That is exactly what happened in the first pass of this audit. Under ALCOA+ a record that
contradicts its own counterpart is a data-integrity issue in its own right, independent of which number is
correct.

**Action:** make the completions table the single source of truth and have assignments derive from it (or
sync on write), and drop the empty legacy tables so they cannot be mistaken for the system of record.

---

## 4. Open commitments already made to IQVIA

These are live promises. Welo is a different client, but an auditor who finds a missed commitment to another
sponsor draws conclusions.

| Commitment | Source | Due | Status 25 Aug |
|---|---|---|---|
| ICH E6 (R3) GCP training module for COA staff | Obs. 1, voluntary | **31 Aug 2026** | Course built; 0 of 8 assigned staff complete (F-2) |
| SOP-001 v2.1 — release checklist requires signature verification | Preventive action 1 | 30 Sep 2026 | Not evidenced |
| Document-control training refresh on GDP signatures | Preventive action 3 | 30 Sep 2026 | Not evidenced as a course |
| Full Cethos Portal validated to 21 CFR Part 11 | Obs. 2, voluntary | 30 Oct 2026 | On plan (COA module already compliant) |

NC-2026-00005 ("staff training completions not recorded", due 15 Aug) should be reassessed — 106 staff
completions are now recorded across 16 people, so the corrective action appears substantially delivered even
though the register was never updated to say so.

Also carried over from REG-AF-001 and still open past due date:
NC-2026-00006 (competence basis, due 31 Aug), NC-2026-00009 (assignment gate in warn mode, due 31 Jul).

---

## 5. What is genuinely strong — lead with these

Verified live, and worth putting in front of Welo:

- **Document control.** Immutable versioned registry, 44 active SOPs, append-only `sop_audit_log`, three-
  signature electronic sign-off with Approval Certificates on the QMS Core documents.
- **Access and data security.** RLS on 386/387 tables, three-tier roles, OTP authentication, no shared
  accounts (SOP-014).
- **Data integrity.** Append-only hash-chained order communication log with `verify_order_comm_log_integrity()`;
  UPDATE/DELETE blocked at the database.
- **Electronic signatures.** 4,151 vendor NDA e-signatures with full metadata and signed snapshots;
  CSV-002 evidences the COA module meeting 21 CFR Part 11 — confirmed by IQVIA's own auditor.
- **Backup and recovery.** PITR plus independent AWS S3 replica, restore-tested three ways
  (CTS-REC-RST-002/003/004), plus a BCDR tabletop exercise record.
- **Training pipeline.** 32 courses across staff and linguist audiences, 175 lessons and 175 quiz questions,
  2,740 recorded completions since June with IP and user-agent captured on 1,894 of them and quiz scores on
  692 against an 80% pass bar. This is a genuine, attributable training record system — it is simply not yet
  reaching the clinician roster.
- **Internal audit and management review.** SOP-012/SOP-013 with a real audit programme (IA-2026-001/002/004)
  and IAP-2026-001 / MRS-2026-001.
- **A closed sponsor audit.** IQVIA qualification audit, 29–30 June 2026: zero critical, zero major, one
  minor (corrected 6 Aug), two observations. Recommendation for approval submitted 14 Aug 2026.

---

## 6. Recommended sequence before 31 August

| Priority | Action | Owner | Effort |
|---|---|---|---|
| 1 | Open the on-time-delivery CAPA (F-6) | QM | 2h |
| 2 | Record GCP status + verify one evidence item for clinicians on Welo work (F-1) | Clinical Validation Dir. | 1–2d |
| 3 | Re-audience the staff GCP course, chase 8 completions, enrol clinicians (F-2, discharges IQVIA commitment) | QM | 1d + chase |
| 4 | Reconcile QM-002 to the live registry and re-issue (F-4) | QM | 3h |
| 5 | Extend `requires_signoff` to remaining controlled docs, or re-scope the IQVIA closure statement (F-3) | QM / IT | 1d |
| 6 | Fix QM-001 v5.8 reviewer so preparer ≠ reviewer (F-3) | QM | 30m |
| 7 | Publish or withdraw CTS-COA-CD-FG-001 (F-5) | Clinical Validation Dir. | 1h |
| 8 | Enable RLS on or drop `_tm_reconcile_staging` (F-7) | IT | 15m |
| 9 | Wire `clinician_profiles.gcp_trained` to training completions (F-1, F-2) | IT | 2h |
| 10 | Reconcile the two completion stores; drop the empty legacy tables (F-9) | IT | 3h |

Items 1–4 are the ones that change what Cethos can truthfully write on the questionnaire. Items 5–8 are
what an auditor finds if they look past the answers.

---

*Prepared 25 August 2026. All live-system figures verified by direct query on that date.*
