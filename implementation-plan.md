# LEGIS — Implementation Plan

*Documented: 2026-05-01 — Last updated: 2026-05-22 (session 5)*

---

## Critical Path Analysis

- **Schema design gates everything** — org structure and approval paths as data must be correct before building UI
- **Workflow state machine is the core** — parallel Administration paths converging at COO is the hardest design decision
- **Email and PDF are late-stage** — they don't block the main workflow

## Highest-Risk Items (de-risk early)

1. **Approval path data model** — storing zero or more parallel Admin paths + standalone bureaus converging at COO as DB records
2. **TipTap attributed contributions** — per-contribution who/when attribution requires a custom extension
3. **Approval invalidation cascade** — any field edit after approval must void all downstream approvers correctly

---

## Phases

### Phase 1 — Foundation ✅
*Scaffold, auth, org structure — complete 2026-05-13*

- [x] Next.js project scaffold — Next.js 15.5.18, Prisma 7.8.0, Tailwind 4, TypeScript
- [x] Neon dev/prod branches wired — `DATABASE_URL` in `.env.local` (dev) and Railway env vars (prod); `prisma.config.ts` loads both `.env` and `.env.local` so migrations always hit dev branch locally
- [x] DB schema: `User`, `Role` enum (12 roles), org hierarchy (`Administration`, `Bureau`, `Division`, `Section`) — migrated to Neon dev branch, Prisma client generated to `app/generated/prisma/client`
- [x] NextAuth.js v5 — Credentials provider (email + password); `auth.config.ts` (Edge-safe, used by middleware) split from `auth.ts` (Node.js, has Prisma)
- [x] Role-based route middleware (`middleware.ts`) — redirects unauthenticated users to `/login`
- [x] Layout shell and navigation (`app/(app)/layout.tsx`) — header with user name, role, sign-out
- [x] Login/logout pages (`app/(auth)/login/`)
- [x] Dev/prod environment setup — Railway deployment config (`railway.json`), `.env.example`, `postinstall: prisma generate`
- [x] Seed script (`scripts/seed.ts`) — 8 test users, one per key role, password: `password`

---

### Phase 2 — Bill Record ✅
*Data model + CRUD — complete 2026-05-14*

- [x] Full Prisma schema for all 14 sections — 6 enums + 6 models; `BillAnalysis` flat (all section columns), child tables for `BillReviewer`, `SuggestedChange`, `BillSme`, `BillApprovalPath`, `ApprovalDecision`; migrated to Neon dev branch (`BillSeniorDeputy` dropped — Sr. Deputies assigned exclusively via `BillApprovalPath.srDeputyId`)
- [x] Bill list view (`/bills`) — scoped by role: LA roles see all; others see assigned only; status badges, priority, created-by, date columns
- [x] Create record (`/bills/new`) — Section 1 locked at creation; dynamic reviewer list; senior deputy checkboxes; APOC/LA Contact dropdowns; Server Action + Zod validation
- [x] Detail view (`/bills/[id]`) — full 13-section layout; per-section inline edit with save/cancel; executive lock enforced; Section 12 shown only when `workflowStatus === ENROLLED`
- [x] Inline section editing (Sections 2–13) — `useActionState` + Server Actions; `revalidatePath` refreshes server data after save; `successAt: number` timestamp triggers `useEffect` close on every save
- [x] Zod server-side validation in all Server Actions (not React Hook Form — see tech-stack note)
- [x] shadcn/ui installed — Nova preset (`base-nova`), uses `@base-ui/react` (Base UI from MUI); components: Button, Input, Textarea, Select, Checkbox, Badge, Card, Label, Table, Separator
- [ ] TipTap rich text integration — deferred to Phase 4 (currently plain `<textarea>`)

---

### Phase 3 — Workflow & Approval Engine ✅
*Core engine complete 2026-05-16; gates + Sr. Deputy edit fix 2026-05-21; proxy + exec selection + mid-workflow changes complete 2026-05-22; seed consolidated to real MDHHS staff + GitHub Actions CI added 2026-05-22*

**Completed:**
- [x] `WorkflowStatus` state machine — DRAFT → SUBMITTED → SME_REVIEW → LAI_REVIEW → EXECUTIVE_REVIEW → APPROVED → ENROLLED; all transitions server-validated by role + current status
- [x] 6 Server Actions in `app/actions/workflow.ts`:
  - `submitBill` — LAI only; DRAFT → SUBMITTED
  - `createApprovalPaths` — APOC only; SUBMITTED → SME_REVIEW; creates `BillApprovalPath` + `BillSme` + `ApprovalDecision` rows for all SMEs and Sr. Deputies
  - `signalReviewComplete` — APOC only; SME_REVIEW → LAI_REVIEW; all non-voided admin decisions must be APPROVED first
  - `laiApprove` — LAI only; LAI_REVIEW → EXECUTIVE_REVIEW; auto-creates `ApprovalDecision` rows for COO + DIRECTOR (always) and CAO/HSD/CME (when flagged on bill)
  - `recordDecision` — any assigned approver; APPROVE/REJECT + comment; auto-advances to APPROVED when all exec decisions pass
  - `markEnrolled` — LAI only; APPROVED → ENROLLED
- [x] `WorkflowPanel` server component (`app/(app)/bills/[id]/_components/WorkflowPanel.tsx`) — self-fetching; role-gated action blocks; approval chain display with voided-decision filtering and Sr. Deputy / executive rows
- [x] `ApprovalPathBuilder` client component — APOC selects org units, assigns SMEs to parallel paths; hierarchical SME lookup (Administration → Bureau → Division levels)
- [x] `WorkflowActions` client component — Submit, Signal Complete, LAI Approve, Mark Enrolled buttons with `useTransition`
- [x] `DecisionForm` client component — Approve/Reject + optional comment via `useActionState`
- [x] Approval invalidation — `voidActiveApprovals()` in `app/actions/bills.ts`; voids old APPROVED decisions, creates new PENDING ones (preserves audit trail); called on every section save
- [x] Executive lock — `updateBill` action rejects edits from non-exec roles once status reaches EXECUTIVE_REVIEW/APPROVED/ENROLLED
- [x] Railway deployment — `railway.json` runs `npx prisma migrate deploy && npm start`; `engines: { node: ">=22.12.0" }` in `package.json`
- [x] GitHub Actions CI — `.github/workflows/db-migrate-and-deploy.yml` at git root (one level above `legis/`); triggers on push to `main`; runs `npx prisma migrate deploy` against `NEON_DEV_DATABASE_URL` secret; `defaults.run.working-directory: legis` for all run steps; `cache-dependency-path: legis/package-lock.json`
- [x] Seed script updated — 46 users total; all test emails use `@legis.test` domain (stale `@mdhhs.test` records deleted on each seed run via `deleteMany`); includes real Legislative Affairs staff (Chardae Burton, Jeffrey Spitzley, Marina Wyrzykowski, Mary Imre, Nicholas Rossow, Lesley Keyton, Caryn Shannon, Elaine Heckman, Nicole Nelson); org units: Children's Services Administration (with Sr. Deputy), Juvenile Justice Bureau (under CSA), Bureau of Finance (standalone), Office of Legal Affairs (standalone division), Legislative Affairs (standalone division, `bureauId: null`, under COO), LG010 Legislative Affairs Section, LG020 Legislative Constituent Services Section; LG010 members hold triple-role `LAI + APOC + LA_CONTACT`; org assignments wired to all users
- [x] Multi-role support — `User.role Role` → `User.roles Role[]`; all role checks updated to `.some()` / `.includes()` across the array; auth JWT/session carry `roles[]`; `trustHost: true` added to fix Railway `UntrustedHost` error
- [x] LAI creation form simplified — Sr. Deputy assignment removed; LAI assigns APOC and LA Contact only; Sr. Deputies assigned exclusively by APOC via `ApprovalPathBuilder`
- [x] `BillSeniorDeputy` model dropped — schema migration applied; access control for Sr. Deputies derived from `ApprovalDecision` rows
- [x] `BillHeader` updated — Sr. Deputies sourced from `BillApprovalPath.srDeputy` (deduped); shown only after APOC builds approval paths
- [x] Admin route guard — `middleware.ts` blocks `/admin/*` for non-ADMIN sessions; `app/(app)/admin/layout.tsx` enforces server-side; all `app/actions/admin.ts` actions re-check `requireAdmin()` independently
- [x] Admin org management (`/admin/org`) — full CRUD for all four org levels: Administrations (with Sr. Deputy assignment), Bureaus (optional Administration parent), Divisions (optional Bureau parent — `bureauId: null` for standalone divisions like Legislative Affairs), Sections (Division parent via `divisionId` OR direct Bureau parent via `bureauId` — one required; admin page shows whichever parent exists); inline edit rows; delete blocked when org unit has referencing bill approval paths or assigned users
- [x] Admin user management (`/admin/users`) — create/edit users with name, email, password (optional on edit), multi-role checkboxes (all 12 roles including ADMIN), org unit assignment (Administration / Bureau / Division / Section / None); activate/deactivate toggle; duplicate email blocked
- [x] Session staleness behavior documented — role changes (including ADMIN grant) take effect on next sign-in due to JWT caching; affected user must sign out and back in
- [x] Sr. Deputy ordering gate — `recordDecision` blocks APPROVE for `SENIOR_DEPUTY` until all `BillSme` records on the same `BillApprovalPath` have APPROVED decisions; skipped when `srDeputyActsAsSme: true` (no sibling SMEs)
- [x] Conditional approver ordering gate — `recordDecision` blocks APPROVE for `COO` until all required conditional approvers (CAO/HSD/CME per bill flags) have non-voided APPROVED decisions
- [x] COO → Director serial gate — `recordDecision` blocks APPROVE for `DIRECTOR` until the non-voided COO decision is APPROVED
- [x] Sr. Deputy bill edit access — `editable` flag in `app/(app)/bills/[id]/page.tsx` now includes `bill.approvalPaths.some(p => p.srDeputy?.id === userId)`; previously Sr. Deputies were always non-editable because they are assigned via `BillApprovalPath.srDeputyId`, not as `BillSme` rows
- [x] Executive approver selection decoupled from LAI approval — `setExecApprovers` Server Action + `ExecApproverSelector` client component (`app/(app)/bills/[id]/_components/ExecApproverSelector.tsx`); APOC or LAI selects individual exec approvers and conditional flags (CAO/HSD/CME) during `LAI_REVIEW` as a distinct step; `laiApprove` validates selections are present before transitioning to `EXECUTIVE_REVIEW`
- [x] Proxy approver assignment — `BillProxyAssignment` model; `assignProxy` / `removeProxy` Server Actions; `ProxyAssignmentPanel` client component; APOC or LAI assigns a `PROXY_APPROVER` user to act on behalf of any pending approver on a specific bill; `recordDecision` updated to allow proxy submission and record `proxyApproverId` on the decision
- [x] Mid-workflow bill changes — `BillChangeNotice` model; `recordBillChange` Server Action; `BillChangeNoticeForm` client component; LAI records a required comment (≥10 chars) when bill text changes; all active non-voided `APPROVED` decisions are voided and replaced with fresh `PENDING` rows; change notices displayed as amber banner in `WorkflowPanel`; available up to and including `LAI_REVIEW`; auto-email to APOC/SMEs/Sr. Deputies is deferred to Phase 5

**Workflow ordering summary (from legis_workflow_v4.svg):**

Correct executive approval sequence within `EXECUTIVE_REVIEW`:
```
[CAO] [HSD] [CME]  — parallel, dashed (only if required), all must complete
          ↓
    COO reviews & approves  [PA allowed]
          ↓
  Director reviews & approves  [PA allowed]
          ↓
       auto-advance → APPROVED
```

Correct SME/Sr. Deputy ordering within `SME_REVIEW` (Path A):
```
Admin SME(s)  — all must approve
      ↓
Sr. Deputy Director  — approves last for their path
      ↓
  path complete (converges with other paths at APOC signal)
```

---

### Phase 4 — Collaboration & Audit

- **TipTap rich text editor** — replace all plain `<textarea>` fields across Sections 2–13 with TipTap; affects 10+ Rich Text fields; store content as JSON (TipTap ProseMirror format) or HTML; migrate existing plain-text values on first save
- **Collaborative field attribution** — TipTap custom extension: per-contribution who/when attribution; multiple stakeholders may contribute to the same field; APOC or LAI consolidates; attribution metadata stored alongside content (not a separate table — embedded in TipTap document JSON)
- **Audit trail** — `AuditLog` model required (does not yet exist in schema); log every field write with: `billAnalysisId`, `userId`, `field` (string), `oldValue`, `newValue`, `editedAt`; note: the existing `voidedAt` pattern on `ApprovalDecision` is approval history only, not a general audit trail; `lastModifiedById` on `BillAnalysis` is last-editor only, not a history
- **`updateBill` access guard** — security gap: the `updateBill` Server Action (`app/actions/bills.ts`) validates workflow status and exec lock but does NOT verify the calling user is actually assigned to the bill; any authenticated user who knows a bill ID can POST edits; fix: load the bill with assignment fields and reject if user is not LAI, APOC, LA Contact, assigned SME, or Sr. Deputy on an approval path
- **Completion % calculation** — system-calculated from required field fill rate; currently hardcoded to `0` on all records
- **Previous Department Review Reference** — FK on `BillAnalysis` pointing to a prior LEGIS record (same or predecessor bill); set by LAI at creation or any time during drafting:
  - Schema: `previousReviewId String?` FK → `BillAnalysis.id`; self-relation; display in Section 1 header (currently missing from schema and UI)
  - Clone action — LAI triggers "Copy from previous review" to pre-populate selected sections (Summary, Intent, Lead Agency Position, Position Details, Suggested Changes, Fiscal Impact, Background, Stakeholder Positions) from the referenced record; cloned content is a plain editable draft; Section 1 and system/workflow fields are never cloned; can only be performed once per record; confirmation prompt required before overwriting existing content
  - "View Prior Review" panel — shown on any record that has a previous review linked; side-by-side comparison of key fields (Bill Number, Topic, Lead Agency Position, Summary, date); link to navigate to the prior record; full chain traversable if the prior record itself has a previous review

---

### Phase 5 — Email Notifications

- Resend + React Email setup; Inngest job queue on Railway
- All 6 auto-email triggers wired to workflow events:

| Trigger | Recipients |
|---|---|
| LAI submits record | APOC, Senior Deputies, LA Contact |
| APOC creates approval path | APOC, LAI |
| APOC triggers message to LAI | APOC, LAI |
| LAI approves | Legislative Affairs Manager (routing sheet) |
| Director approves | LAI |
| LAI clicks "Email Artifacts" | TBD |

---

### Phase 6 — Files & PDF

**Bill document attachments:**
- File upload to Railway Storage (S3-compatible) via AWS SDK v3; accepted formats: PDF (primary), HTML, plain text, Markdown, XML; at least one of file upload or public URL required per document entry
- Each document entry stores: label/filename, upload date, uploader (FK → User), optional public URL (e.g. Michigan Legislature link), Railway Storage key for uploaded file
- Document list UI: label/filename, upload date, uploader name, download link (if file uploaded), open URL link (if URL provided)
- Management: LAI and APOC may add or remove documents at any workflow stage prior to executive lock
- Mid-workflow primary document promotion: when LAI records a bill change (Phase 3), the new upload becomes current primary; prior versions demoted but retained and visible in document history

**Comparison & PDF:**
- Bill draft comparison link: when two or more documents are present, construct comparison link using their public URLs (user-supplied URL or Railway Storage download URL); load comparison tool in new tab or embedded iframe (integration details TBD)
- PDF bill analysis generation (Puppeteer — renders Next.js template server-side)
- PDF routing sheet generation (Puppeteer)
- "Email Artifacts" button — sends both PDFs via Inngest job
- AI-assisted comparison *(low priority / future)*: generative AI produces plain-language synopsis of changes between two bill versions; surfaced as optional "Summarize Changes" alongside comparison link; depends on third-party tool maturity and LLM API selection

---

### Phase 7 — Polish

- ~~Admin UI: user management, role assignment, org structure management~~ — completed in Phase 3
- ~~Section 12 conditional display (post-enrollment only)~~ — completed in Phase 2
- **Dashboard** — currently a stub (shows username + roles only); build: bills awaiting the current user's action (pending decisions, awaiting APOC path setup, awaiting LAI review), counts by workflow status, recent activity
- Emergency override controls (ADMIN role)
- Bill list: search, filter by status / priority / assignee
- **Section 1 Priority field display** — minor bug: `Section1.tsx` line 35 hardcodes `—` for Priority; should read `bill.priority` (field exists in schema and is set at creation)
- Accessibility and responsive pass

---

## Sequencing

```
Phase 1 ✅ → Phase 2 ✅ → Phase 3 ✅
                                    ↓
                              Phase 4 (TipTap + audit trail + access guard + previous review reference)
                                    ↓
                              Phase 5 (email notifications)
                                    ↓
                              Phase 6 (file attachments + PDF)
                                    ↓
                              Phase 7 (polish + dashboard + minor bugs)
```

Phases 5 and 6 can begin in parallel once Phase 4 is stable — email requires workflow events; PDF/file upload requires form data; neither blocks the other.

**Phase 4 recommended order:** fix `updateBill` access guard first (security), then TipTap integration (touches all sections — do before audit trail so log captures rich text format from the start), then audit trail schema + logging, then completion %, then previous review reference (needs schema migration).

> **Key recommendation:** Spend extra design time on the approval path schema in Phase 2 before writing any Phase 3 UI. A wrong data model at that layer is expensive to unwind.
