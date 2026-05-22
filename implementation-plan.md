# LEGIS — Implementation Plan

*Documented: 2026-05-01 — Last updated: 2026-05-21 (session 3)*

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

### Phase 3 — Workflow & Approval Engine
*Core engine complete 2026-05-16; gates + Sr. Deputy edit fix 2026-05-21*

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
- [x] Seed script updated — 16 test users (added CAO, HSD, CME executives; added `lai-apoc@legis.test` dual-role test user); org units: Children's Services Administration (with Sr. Deputy), Juvenile Justice Bureau (under CSA), Bureau of Finance (standalone), Office of Legal Affairs (standalone division); org assignments wired to all dept users
- [x] Multi-role support — `User.role Role` → `User.roles Role[]`; all role checks updated to `.some()` / `.includes()` across the array; auth JWT/session carry `roles[]`; `trustHost: true` added to fix Railway `UntrustedHost` error
- [x] LAI creation form simplified — Sr. Deputy assignment removed; LAI assigns APOC and LA Contact only; Sr. Deputies assigned exclusively by APOC via `ApprovalPathBuilder`
- [x] `BillSeniorDeputy` model dropped — schema migration applied; access control for Sr. Deputies derived from `ApprovalDecision` rows
- [x] `BillHeader` updated — Sr. Deputies sourced from `BillApprovalPath.srDeputy` (deduped); shown only after APOC builds approval paths
- [x] Admin route guard — `middleware.ts` blocks `/admin/*` for non-ADMIN sessions; `app/(app)/admin/layout.tsx` enforces server-side; all `app/actions/admin.ts` actions re-check `requireAdmin()` independently
- [x] Admin org management (`/admin/org`) — full CRUD for all four org levels: Administrations (with Sr. Deputy assignment), Bureaus (optional Administration parent), Divisions (optional Bureau parent), Sections (required Division parent); inline edit rows; delete blocked when org unit has referencing bill approval paths or assigned users
- [x] Admin user management (`/admin/users`) — create/edit users with name, email, password (optional on edit), multi-role checkboxes (all 12 roles including ADMIN), org unit assignment (Administration / Bureau / Division / Section / None); activate/deactivate toggle; duplicate email blocked
- [x] Session staleness behavior documented — role changes (including ADMIN grant) take effect on next sign-in due to JWT caching; affected user must sign out and back in
- [x] Sr. Deputy ordering gate — `recordDecision` blocks APPROVE for `SENIOR_DEPUTY` until all `BillSme` records on the same `BillApprovalPath` have APPROVED decisions; skipped when `srDeputyActsAsSme: true` (no sibling SMEs)
- [x] Conditional approver ordering gate — `recordDecision` blocks APPROVE for `COO` until all required conditional approvers (CAO/HSD/CME per bill flags) have non-voided APPROVED decisions
- [x] COO → Director serial gate — `recordDecision` blocks APPROVE for `DIRECTOR` until the non-voided COO decision is APPROVED
- [x] Sr. Deputy bill edit access — `editable` flag in `app/(app)/bills/[id]/page.tsx` now includes `bill.approvalPaths.some(p => p.srDeputy?.id === userId)`; previously Sr. Deputies were always non-editable because they are assigned via `BillApprovalPath.srDeputyId`, not as `BillSme` rows

**Pending:**
- [ ] Proxy approver assignment — UI to designate a `PROXY_APPROVER` user for a given approver role on a specific bill
- [ ] Executive approver selection decoupled from LAI approval — v4 workflow (and requirements step 14) require exec approvers to be selected during `LAI_REVIEW` as a distinct step before the LAI approval action; currently `laiApprove` atomically creates exec decisions and transitions status in one action; needs to be split into (a) an APOC/LAI-accessible form to confirm exec approvers during `LAI_REVIEW` and (b) a separate LAI approve action that transitions to `EXECUTIVE_REVIEW`
- [ ] Mid-workflow bill changes — LAI uploads an updated bill document (old versions retained) and enters a required comment describing the change; system promotes new doc to current primary, records a prominent "bill change notice" visible to all users with record access, voids all active non-voided `APPROVED` decisions (consistent with existing approval invalidation), and sends auto-email to APOC/assigned SMEs/Sr. Deputies; available only up to and including `LAI_REVIEW` — LAI must contact ADMIN for an override once status reaches `EXECUTIVE_REVIEW` or later; upload is strongly recommended but not required (comment is required)

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

- Attributed contributions on collaborative rich text fields (TipTap custom extension: who/when per contribution block)
- Audit trail: all field writes logged with user + timestamp
- Completion % calculation (system-calculated from required field fill rate)
- **Previous Department Review Reference** — FK on `BillAnalysis` pointing to a prior LEGIS record (same or predecessor bill); set by LAI at creation or any time during drafting:
  - Schema: `previousReviewId String? @db.VarChar` FK → `BillAnalysis.id`; self-relation; add to Section 1 header
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
- Emergency override controls (ADMIN role)
- Bill list: search, filter by status / priority / assignee
- Section 12 conditional display (post-enrollment only)
- Accessibility and responsive pass

---

## Sequencing

```
Phase 1 → Phase 2 → Phase 3 (finish gates + mid-workflow bill changes)
                                    ↓
                              Phase 4 (TipTap + audit + previous review reference)
                                    ↓
                              Phase 5 (email notifications)
                                    ↓
                              Phase 6 (file attachments + PDF)
                                    ↓
                              Phase 7 (polish)
```

Phases 5 and 6 can begin in parallel once Phase 4 is stable — email requires workflow events; PDF/file upload requires form data; neither blocks the other.

**Phase 3 completion order:** implement the three `recordDecision` ordering gates first (all in one file, well-specified), then decouple executive approver selection from `laiApprove`, then mid-workflow bill changes (touches approval invalidation already built), then proxy approver assignment.

> **Key recommendation:** Spend extra design time on the approval path schema in Phase 2 before writing any Phase 3 UI. A wrong data model at that layer is expensive to unwind.
