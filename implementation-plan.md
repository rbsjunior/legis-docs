# LEGIS — Implementation Plan

*Documented: 2026-05-01 — Last updated: 2026-05-30 (session 13)*

---

## Critical Path Analysis

- **Schema design gates everything** — org structure and approval paths as data must be correct before building UI
- **Workflow state machine is the core** — parallel Administration paths converging at COO is the hardest design decision
- **Email and PDF are late-stage** — they don't block the main workflow

## Highest-Risk Items (de-risk early)

1. **Approval path data model** — storing zero or more parallel Admin paths + standalone bureaus converging at COO as DB records
2. **TipTap real-time collaboration** — Yjs + Hocuspocus WebSocket server on Railway; needs careful Railway deployment config
3. **Approval invalidation cascade** — any field edit after approval must void all downstream approvers correctly

---

## Phases

### Phase 1 — Foundation ✅
*Scaffold, auth, org structure — complete 2026-05-13*

- [x] Next.js project scaffold — Next.js 15.5.18, Prisma 7.8.0, Tailwind 4, TypeScript
- [x] Neon dev/prod branches wired — `DATABASE_URL` in `.env.local` (dev) and Railway env vars (prod); `prisma.config.ts` loads both `.env` and `.env.local` so migrations always hit dev branch locally
- [x] DB schema: `User`, `Role` enum (12 roles), org hierarchy (`Administration`, `Bureau`, `Division`, `Section`) — migrated to Neon dev branch, Prisma client generated to `app/generated/prisma/client`
- [x] NextAuth.js v5 — Credentials provider (username + password); `auth.config.ts` (Edge-safe, used by middleware) split from `auth.ts` (Node.js, has Prisma); login field is "User ID" (`username`), not email
- [x] Role-based route middleware (`middleware.ts`) — redirects unauthenticated users to `/login`
- [x] Layout shell and navigation (`app/(app)/layout.tsx`) — header with user name, role, sign-out
- [x] Login/logout pages (`app/(auth)/login/`)
- [x] Dev/prod environment setup — Railway deployment config (`railway.json`), `.env.example`, `postinstall: prisma generate`
- [x] Seed script (`scripts/seed.ts`) — 8 test users, one per key role, password: `password`

---

### Phase 2 — Bill Record ✅
*Data model + CRUD — complete 2026-05-14*

- [x] Full Prisma schema for all 14 sections — 6 enums + 6 models; `BillAnalysis` flat (all section columns), child tables for `BillReviewer`, `SuggestedChange`, `BillSme`, `BillApprovalPath`, `ApprovalDecision`; migrated to Neon dev branch (`BillSeniorDeputy` dropped — Sr. Deputies assigned exclusively via `BillApprovalPath.srDeputyId`)
- [x] Multi-bill package support (2026-05-26) — replaced single `billNumber: String` with `BillPackageBill` child table; one or more bill numbers per analysis; at least one required at creation; `createBill` action, `CreateBillForm` (multi-entry list), bill list (shows up to 2 joined with " / ", "+N more"), bill detail page query, `Section1` ("Bill Number" / "Bill Package" label), and `BillHeader` all updated; migration SQL migrates existing `billNumber` values before dropping the column; `relatedBillNumbers` retained for cross-referencing other LEGIS records
- [x] Bill list view (`/bills`) — scoped by role: LA roles see all; others see assigned only; status badges, priority, created-by, date columns
- [x] Create record (`/bills/new`) — Section 1 locked at creation; dynamic reviewer list; senior deputy checkboxes; APOC/LA Contact dropdowns; Server Action + Zod validation
- [x] Detail view (`/bills/[id]`) — full 13-section layout; per-section inline edit with save/cancel; executive lock enforced; Section 12 shown only when `workflowStatus === ENROLLED`
- [x] Inline section editing (Sections 2–13) — `useActionState` + Server Actions; `revalidatePath` refreshes server data after save; `successAt: number` timestamp triggers `useEffect` close on every save
- [x] Zod server-side validation in all Server Actions (not React Hook Form — see tech-stack note)
- [x] shadcn/ui installed — Nova preset (`base-nova`), uses `@base-ui/react` (Base UI from MUI); components: Button, Input, Textarea, Select, Checkbox, Badge, Card, Label, Table, Separator
- [x] TipTap rich text integration — complete (Phase 4, 2026-05-26); all Rich Text fields wired across Sections 2–10, 13; `RichTextEditor` + `RichTextContent` components; `SimpleTextSection` upgraded to use rich text by default

---

### Phase 3 — Workflow & Approval Engine ✅
*Core engine complete 2026-05-16; gates + Sr. Deputy edit fix 2026-05-21; proxy + exec selection + mid-workflow changes complete 2026-05-22; seed consolidated to real MDHHS staff + GitHub Actions CI added 2026-05-22; bug fixes + seed additions 2026-05-23*

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
- [x] Seed script updated — real MDHHS staff with `@legis.test` domain; org units: Children's Services Administration (with Sr. Deputy), Juvenile Justice Bureau (under CSA), Bureau of Finance (standalone), Office of Legal Affairs (standalone division), Legislative Affairs (standalone division), LG010/LG020 sections; LG010 members hold triple-role `LAI + APOC + LA_CONTACT`
- [x] Seed additions (2026-05-23) — Financial Operations Administration (Amy Epkey, SENIOR_DEPUTY); Bureau of Budget (under FO Admin); Human Services Budget Division + Health Care & Aging Services Budget Division (under Bureau of Budget); Children's Services Administration Section (D3130) + Medical Services Administration Section (D3030); 5 new staff: Matthew Deaton, Jensine Garza (CSA Section), Tyler Bishop (Human Svc Div), Kelly Wilcox (Medical Svc Section) — all SME role
- [x] Multi-role support — `User.role Role` → `User.roles Role[]`; all role checks updated to `.some()` / `.includes()` across the array; auth JWT/session carry `roles[]`; `trustHost: true` added to fix Railway `UntrustedHost` error
- [x] LAI creation form simplified — Sr. Deputy assignment removed; LAI assigns APOC and LA Contact only; Sr. Deputies assigned exclusively by APOC via `ApprovalPathBuilder`
- [x] `BillSeniorDeputy` model dropped — schema migration applied; access control for Sr. Deputies derived from `ApprovalDecision` rows
- [x] `BillHeader` updated — Sr. Deputies sourced from `BillApprovalPath.srDeputy` (deduped); shown only after APOC builds approval paths
- [x] Admin route guard — `middleware.ts` blocks `/admin/*` for non-ADMIN sessions; `app/(app)/admin/layout.tsx` enforces server-side; all `app/actions/admin.ts` actions re-check `requireAdmin()` independently
- [x] Admin org management (`/admin/org`) — full CRUD for all four org levels: Administrations (with Sr. Deputy assignment), Bureaus (optional Administration parent), Divisions (optional Bureau parent — `bureauId: null` for standalone divisions like Legislative Affairs), Sections (Division parent via `divisionId` OR direct Bureau parent via `bureauId` — one required; admin page shows whichever parent exists); inline edit rows; delete blocked when org unit has referencing bill approval paths or assigned users
- [x] Admin user management (`/admin/users`) — create/edit users with name, email, User ID (username), password (optional on edit), multi-role checkboxes (all 12 roles including ADMIN), org unit assignment (Administration / Bureau / Division / Section / None); activate/deactivate toggle; duplicate email or username blocked; "User ID" column shown in user table
- [x] Session staleness behavior documented — role changes (including ADMIN grant) take effect on next sign-in due to JWT caching; affected user must sign out and back in
- [x] Sr. Deputy ordering gate — `recordDecision` blocks APPROVE for `SENIOR_DEPUTY` until all `BillSme` records on the same `BillApprovalPath` have APPROVED decisions; skipped when `srDeputyActsAsSme: true` (no sibling SMEs)
- [x] Conditional approver ordering gate — `recordDecision` blocks APPROVE for `COO` until all required conditional approvers (CAO/HSD/CME per bill flags) have non-voided APPROVED decisions
- [x] COO → Director serial gate — `recordDecision` blocks APPROVE for `DIRECTOR` until the non-voided COO decision is APPROVED
- [x] Sr. Deputy bill edit access — `editable` flag in `app/(app)/bills/[id]/page.tsx` now includes `bill.approvalPaths.some(p => p.srDeputy?.id === userId)`; previously Sr. Deputies were always non-editable because they are assigned via `BillApprovalPath.srDeputyId`, not as `BillSme` rows
- [x] Executive approver selection decoupled from LAI approval — `setExecApprovers` Server Action + `ExecApproverSelector` client component (`app/(app)/bills/[id]/_components/ExecApproverSelector.tsx`); APOC or LAI selects individual exec approvers and conditional flags (CAO/HSD/CME) during `LAI_REVIEW` as a distinct step; `laiApprove` validates selections are present before transitioning to `EXECUTIVE_REVIEW`
- [x] Proxy approver assignment — `BillProxyAssignment` model; `assignProxy` / `removeProxy` Server Actions; `ProxyAssignmentPanel` client component; APOC or LAI assigns a `PROXY_APPROVER` user to act on behalf of any pending approver on a specific bill; `recordDecision` updated to allow proxy submission and record `proxyApproverId` on the decision
- [x] Mid-workflow bill changes — `BillChangeNotice` model; `recordBillChange` Server Action; `BillChangeNoticeForm` client component; LAI records a required comment (≥10 chars) when bill text changes; all active non-voided `APPROVED` decisions are voided and replaced with fresh `PENDING` rows; change notices displayed as amber banner in `WorkflowPanel`; available up to and including `LAI_REVIEW`; auto-email to APOC/SMEs/Sr. Deputies is deferred to Phase 5
- [x] Bug fix (2026-05-23) — `WorkflowPanel.tsx`: `proxiedApproverIds` used before initialization; `billProxyAssignment.findMany` query moved above `myPendingDecision` computation; duplicate query block at old location removed

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

### Phase 4 — Collaboration & Audit *(in progress)*

**Completed (2026-05-23):**
- [x] `updateBill` access guard — `app/actions/bills.ts` now fetches `apocId`, `laContactId`, `smes { userId }`, `approvalPaths { srDeputyId }` on the bill; rejects if caller is not an LA role (`ADMIN/LAI/APOC/LA_CONTACT`) and not explicitly assigned (as APOC, LA Contact, SME, or Sr. Deputy on a path); exec users in a locked bill bypass the assignment check — they already passed the exec-lock gate above it
- [x] TipTap installed — `@tiptap/react@3.23.6` + `@tiptap/starter-kit@3.23.6`; `immediatelyRender: false` for SSR safety
- [x] `RichTextEditor` component (`app/(app)/bills/[id]/_components/RichTextEditor.tsx`) — "use client"; Bold / Italic / ul / ol toolbar; syncs editor JSON to a hidden `<input>` so content submits with the existing Server Action form unchanged; `parseInitialContent()` handles both stored TipTap JSON and legacy plain-text strings
- [x] `RichTextContent` component (`app/(app)/bills/[id]/_components/RichTextContent.tsx`) — "use client"; `generateHTML()` from `@tiptap/core` for JSON content; HTML-escaped plain-text fallback for values stored before TipTap; `dangerouslySetInnerHTML` render
- [x] `.rich-text` CSS — added to `app/globals.css`; styles `p`, `ul`, `ol`, `li`, `strong`, `em`, `code` so editor and read view render consistently; no `@tailwindcss/typography` dependency
- [x] Section 5 narrative field wired — `suggestedChangesNarrative`: `<textarea>` → `<RichTextEditor>`; read-path `<p>` → `<RichTextContent>`; structured changes rows (page/line/instruction/explanation) are plain inputs and unchanged

**Completed (2026-05-26):**
- [x] TipTap rollout complete — `RichTextEditor` / `RichTextContent` wired to all remaining Rich Text fields: Section 2 (intentOfLegislation), 3 (summary), 4 (positionDetails), 6 (fiscalDeptBudgetary, fiscalDeptComments, fiscalStateBudgetary, fiscalLocalGovt), 7 (significantChanges), 8 (proArguments, conArguments — edit layout switched from grid-cols-2 to stacked for editor width), 9 (stakeholderPositions), 10 (background, problemBeingAddressed), 13 (finOpsReview); `SimpleTextSection` upgraded to use rich text by default (covers Sections 3, 7, 9, 13 in one edit); legalCitation (Section 11) kept as plain text per requirements
- [x] Global font/color improvements — `app/globals.css`: `html { font-size: 106.25% }` (17px base, scales all rem sizes); `--color-zinc-400` and `--color-zinc-500` remapped one step darker in `@theme`; `--muted-foreground` darkened from 55.6% to 40% lightness; affects all screens without component changes
- [x] Bill document attachments (Phase 6 pulled forward, 2026-05-26) — `BillDocument` model + migration applied; Railway Storage (S3-compatible, endpoint `t3.storageapi.dev`) via AWS SDK v3; accepted formats: PDF, HTML, plain text, Markdown, XML (up to 50 MB); file uploads via `POST /api/documents/upload` (Route Handler — no body size limit); signed download URLs via `GET /api/documents/[id]` (15-min expiry, access-gated by bill assignment); `removeDocument` + `setDocumentPrimary` Server Actions; `BillDocumentPanel` (Server Component) + `AddDocumentForm` (Client Component using `fetch`); first uploaded doc auto-becomes primary; LAI/APOC/ADMIN may add/remove/promote at any stage prior to executive lock; panel sits between BillHeader and WorkflowPanel on detail page; credentials in `.env.local` and Railway dashboard env vars
- [x] Audit trail (2026-05-26) — `AuditLog` model (`billAnalysisId`, `userId`, `section Int?`, `field`, `oldValue`, `newValue`, `createdAt`); `section = null` for document events, `section 2–13` for section field edits; `diffFields()` helper diffs old/new values per-field — only logs changes (no no-op noise); all Sections 2–13 Server Actions in `app/actions/bills.ts` emit `auditLog.createMany()` in same `$transaction` as `billAnalysis.update()`; document events (upload, remove, set-primary) emit `auditLog.create()` in same `$transaction` as the document operation; `AuditLogPanel` Server Component (self-fetching, 200 logs newest-first) — collapsible `<details>/<summary>`; TipTap JSON fields displayed as extracted plain text (120-char preview); document events formatted as "Added / Removed / Set primary" with filename, MIME, size; panel mounted above Section 14 on detail page; migration `20260526040000` makes `section` nullable

**Completed (2026-05-27, session 11):**
- [x] **Disable / re-enable SME** — `disabledAt DateTime?` added to `BillSme` (migration `20260527010000_bill_sme_disabled_at`); `disableSme` and `enableSme` Server Actions in `workflow.ts`; shared `checkSmeManagePermission` helper enforces stage-gated permissions (APOC/ADMIN pre-exec, LAI/ADMIN during EXECUTIVE_REVIEW, ADMIN-only post-approval); `SmeActions.tsx` client component wraps buttons in `useTransition` (avoids Next.js form-action type mismatch); disabled SMEs retain read access but lose edit access (`editable` flag updated to `&& !s.disabledAt`); Sr. Deputy ordering gate (Gate 1) updated to count only active SMEs
- [x] **Surgical per-path editing (Option B)** — replaces the destructive all-or-nothing `ApprovalPathBuilder` rebuild during live review; five new Server Actions in `workflow.ts`:
  - `addSmeToPath` — upserts `BillSme` to an existing path; creates fresh PENDING `ApprovalDecision` only if none exists; re-enables previously disabled SME if found; flips `srDeputyActsAsSme: false` when a real SME is added
  - `setSrDeputy` — voids outgoing Sr. Deputy decision (if any), updates `srDeputyId`, creates fresh PENDING `ApprovalDecision`
  - `removeSrDeputy` — blocked if any active SME exists on the path; nulls `srDeputyId`, voids decision
  - `addPath` — adds a single new `BillApprovalPath` without touching existing paths; order is `max(existing) + 1`; accepts full `PathInput` JSON (type, IDs, SMEs, Sr. Deputy)
  - `removePath` — transaction: disable active `BillSme` rows, disconnect all `BillSme` FK, void SME and Sr. Deputy decisions, delete path
- [x] `PathModifyControls.tsx` — per-path client component: Add SME dropdown (shows org-scoped candidates minus already-active), Sr. Deputy add/remove button (remove blocked when active SMEs present), Remove path with confirmation dialog; uses `useTransition` for pending state and inline error display
- [x] `AddPathPanel.tsx` — client component rendered once below the path list; toggle open/closed; org unit selector with optgroups (Administrations, Standalone Bureaus, Standalone Divisions) filtered by `usedOrgKeys`; SME checkbox picker scoped to the selected org unit; auto-includes org Sr. Deputy with informational note; calls `addPath`
- [x] `WorkflowPanel.tsx` updates — removed old `<details>` "Modify approval paths" rebuild block; `orgData` now fetched whenever `canManageSmes` (covers all stages, not just pre-EXECUTIVE_REVIEW + isApoc); `canManageSmes` flag controls visibility of Disable/Re-enable SME buttons, `PathModifyControls`, and `AddPathPanel`; `smesForPath` server-side helper computes org-scoped SME candidates per path (excludes org Sr. Deputy); `usedOrgKeys` excludes already-assigned org units from `AddPathPanel`; `createApprovalPaths` cleanup transaction updated to preserve disabled SME rows and their decisions on rebuild; upsert `update` clause sets `disabledAt: null` to auto-re-enable when APOC explicitly re-includes a previously disabled SME
- [x] **Proxy access bug fix** — `bills/page.tsx` `WHERE` clause now includes `{ proxyAssignments: { some: { proxyUserId: userId } } }`; `bills/[id]/page.tsx` query fetches `proxyAssignments`, `hasAccess` check includes `bill.proxyAssignments.some(p => p.proxyUserId === userId)`; proxy users can now view a bill immediately upon assignment, before submitting any decision
- [x] **ExecApproverSelector pre-population** — "Currently: [name]" label displayed above each exec approver dropdown during LAI_REVIEW; derived from existing `current.*ApproverId` props already passed to the component; display-only

**Completed (2026-05-27):**
- [x] Completion % calculation — `calcCompletionPct()` in `app/lib/completion.ts`; checks 4 required fields (intentOfLegislation, summary, leadAgencyPosition, positionDetails); called on every section save across all 13 sections and on clone; displayed in Section 14
- [x] Previous Department Review Reference — `previousReviewId` self-FK on `BillAnalysis`; `versionNumber Int?` auto-assigned at creation; `PreviousReviewPanel` server component shown below Section 1; `CloneFromPreviousReview` client component (one-time clone with confirmation prompt, sections selectable); pre-fill at creation time (bill numbers, topic, sponsors, committee, bipartisan support, related bill numbers); version badge on bill header and list
- [x] Seed email addresses updated — `xxx@legis.test` → `srbsjunior+xxx@gmail.com` across `scripts/seed-mdhhs.ts`
- [x] Username login system (2026-05-27) — `User.username String? @unique`; migration `20260527000000_add_username`; login changed from email to username (`<lastname><firstinitial>`, clash resolution `<lastname><firstname><number>`); `makeUsername()` helper in seed scripts (handles titles: Dr., Mr., etc.); all 45 seed users assigned usernames; `auth.ts` Credentials provider switched to `username` field; login form label changed to "User ID"; admin user management shows/edits username; `update-emails-prod.ts` updated to also set usernames; `db:migrate` now runs `prisma generate` automatically after applying migrations; `db:reset` fixed to use `--full-reset` (drops entire schema via `DROP SCHEMA public CASCADE`) instead of broken `--clean` flag
- [x] `scripts/migrate.ts` improvements — `extractPgCode()` recursively walks the Prisma 7 `P2010 → meta.driverAdapterError → cause` chain to find the raw PG error code (fixes idempotency for `42710 duplicate_object`, `42703 undefined_column`, etc.); partial-row cleanup (deletes stuck `finished_at IS NULL` rows before re-applying); `--full-reset` flag added

**Remaining:**
- [ ] **Collaborative real-time editing** — approach confirmed: Yjs + Hocuspocus WebSocket server (not attribution marks); requires `@tiptap/extension-collaboration`, `@tiptap/extension-collaboration-cursor`, Hocuspocus server deployed on Railway as a separate service; document room keyed by `billId + fieldName`

---

### Phase 5 — Email Notifications *(5 of 6 triggers complete; "Email Artifacts" remaining)*

**Tech:** Nodemailer + Gmail SMTP (`smtp.gmail.com:587`, STARTTLS) — no Inngest or Resend; fire-and-forget `void` calls in Server Actions; email failures never block workflow transitions.

**Env vars:** `GMAIL_USER` (sending address), `GOOGLE_APP_PASSWORD` (Gmail App Password from myaccount.google.com/apppasswords), `AUTH_URL` (used as base URL for record links).

**Files:**
- `app/lib/email.ts` — Nodemailer transporter + `sendEmail()` helper; skips silently if env vars unset
- `app/lib/emails/templates.ts` — HTML template functions for all implemented triggers; `BillEmailData` type
- `app/lib/emails/notifications.ts` — per-trigger helpers (`notifyBillSubmitted`, etc.); fetch bill data + send; each wrapped in try/catch + console.error
- `scripts/test-email.ts` — SMTP connection verify + test delivery; run with `npm run email:test [recipient]`

**Completed (2026-05-28):**

| Trigger | Action | Recipients |
|---|---|---|
| LAI submits record | `submitBill` | APOC, LA Contact |
| APOC creates approval paths | `createApprovalPaths` (initial only, not rebuild) | APOC, LAI |
| APOC signals review complete | `signalReviewComplete` | APOC, LAI |
| LAI approves | `laiApprove` | LA Contact |
| Director approves | `recordDecision` (DIRECTOR + APPROVED) | LAI (creator) |
| Bill text changed mid-workflow | `recordBillChange` | APOC + all active SMEs (disabledAt IS NULL) + Sr. Deputies across all approval paths; deduplicated; amber "Change notice" banner with LAI's comment |

**Remaining:**
- [ ] "Email Artifacts" button — sends PDF analysis + routing sheet; both PDFs are done; this is the last unimplemented trigger

---

### Phase 6 — PDF Generation

~~**Bill document attachments** — pulled into Phase 4; complete as of 2026-05-26~~

**Comparison & PDF:**
- [x] **Bill draft comparison — Draftable (2026-05-28)** — `app/lib/draftable.ts`: `createDraftableComparison()` POSTs to `https://api.draftable.com/v1/comparisons` via native `fetch` + `FormData`; `signedViewerUrl()` signs viewer URL with HMAC-SHA256 using Node.js `crypto` (no SDK dependency); `getDraftableFileType()` maps MIME type or URL extension to Draftable file type (PDF, Word, PPT, TXT); `draftableConfigured()` gates the UI on env var presence. `createComparison` Server Action in `app/actions/documents.ts`: access-checks, resolves Railway Storage presigned URLs or direct publicUrls, calls Draftable, returns 2-hour signed viewer URL. `CompareDocumentsForm` client component: two selects (left/right), Compare ↗ button opens viewer in new tab, inline error display. `BillDocumentPanel` renders compare form when ≥2 comparable docs exist. Env vars: `DRAFTABLE_ACCOUNT_ID` / `DRAFTABLE_AUTH_TOKEN` (prod); falls back to `DRAFTABLE_ACCOUNT_ID_TEST` / `DRAFTABLE_AUTH_TOKEN_TEST` (dev). No npm packages added — native fetch + crypto only.
- [x] **Bill analysis PDF — React PDF (2026-05-28)** — `@react-pdf/renderer` (pure JS, no Chromium, Railway-friendly); added to `serverExternalPackages` in `next.config.ts`. Template in `app/lib/pdf/bill-analysis.tsx` matches MDHHS Word doc layout: Times-Roman/Times-Bold, US Letter, 1" margins; Section 1 header block (Date, Bill Number/Package, Topic, Sponsors, Bipartisan Support, Committee, Analysis Done By, Reviewed By bullet list); Sections 2–11 in document order (Intent, Summary, Lead Agency Position with enum label + details, Suggested Changes narrative + structured page/line items, Fiscal Impact with Department/State/Local Gov sub-sections, Significant Changes, Summary of Arguments Pro/Con, Stakeholder Positions, Background, Executive Power checklist with `[ ]`/`[X]` indicators); Section 12 POST ENROLLMENT on a separate `<Page>` rendered only when `workflowStatus === ENROLLED` (Vote Summary, Admin Rules Impact, Technical Issues, Director's Signature). `RichText` component walks TipTap AST — handles paragraphs, bullet/ordered lists, bold/italic inline marks; falls back to plain-text for pre-TipTap content. API route `GET /api/pdf/bill/[id]` — auth + scoped access (same rules as bill detail page), `renderToBuffer` → `Uint8Array` → `application/pdf` response with `Content-Disposition: attachment` filename from first bill number. `PDF ↓` link added to each row in the bill list (`/bills`).
- PDF routing sheet generation — remaining
- "Email Artifacts" button — sends both PDFs; trigger implemented in Phase 5 once PDFs exist
- AI-assisted comparison *(low priority / future)*: generative AI produces plain-language synopsis of changes between two bill versions; surfaced as optional "Summarize Changes" alongside comparison link; depends on third-party tool maturity and LLM API selection

---

### Phase 7 — Polish

- ~~Admin UI: user management, role assignment, org structure management~~ — completed in Phase 3
- ~~Section 12 conditional display (post-enrollment only)~~ — completed in Phase 2
- ~~`OrgAdmin.tsx` Section type bug~~ — fixed 2026-05-23; `bureau` and `division` typed as `| null` to match Prisma query result (was blocking builds)
- ~~Section 1 Priority field display bug~~ — fixed 2026-05-26; `Section1.tsx` now reads `bill.priority` and displays "Normal" / "Urgent"; `priority` added to `Section1Props` interface
- ~~Global font/color improvements~~ — completed 2026-05-26 (see Phase 4)
- [x] **Dashboard** — three sections: "Awaiting Your Action" (role-scoped pending items: LAI drafts/LAI_REVIEW, APOC submitted, approver/proxy pending decisions; deduped by bill ID), "Bill Analyses" status count grid (7 statuses, scoped by role, each tile links to /bills), "Recent Activity" feed (capped at 5 entries per bill, up to 20 total; fetches 100 newest AuditLog rows then filters in JS to prevent one active bill from flooding the feed); all queries run in parallel; pure server component
- [x] **Emergency override controls (ADMIN role, 2026-05-28)** — `AdminOverridePanel` client component in `WorkflowPanel` (amber collapsible, ADMIN-only); three independent sections each with two-step confirm flow; three new Server Actions in `workflow.ts`:
  - `adminForceStatus(billId, newStatus, reason)` — backward-only status rollback (forward blocked; use normal workflow actions to advance); voids all active decisions; recreates fresh PENDING decisions for target stage (SME/Sr. Deputy rows for `SME_REVIEW`, exec approver rows for `EXECUTIVE_REVIEW`); resets `draftStatus` when rolling back to DRAFT
  - `adminReassignApoc(billId, newApocId, reason)` — replaces `apocId` on the bill; validates new user is active and holds APOC role; prevents re-assigning current APOC
  - `adminVoidDecision(billId, decisionId, reason)` — voids one specific non-voided decision (PENDING / APPROVED / REJECTED) and creates a fresh PENDING for the same approver+role without touching other decisions or bill status; surgical alternative to full rollback when only one rejection needs clearing
  - All three actions write two `AuditLog` rows: the field change (old → new value) + the admin reason; APOC query fetched conditionally (ADMIN sessions only) in `WorkflowPanel`
- [x] **Bill list search & filter** — `BillFilters` client component (debounced text search, status select, priority select); URL-param driven so filters survive navigation and are shareable; `BillFilters` requires `<Suspense>` in Next.js 15 via the parent page; server-side Prisma `AND` combines access-scoping filter + `topic`/`billNumber` icontains search + status + priority; result count shown; empty state distinguishes "no bills" from "no match" (with clear-filters link); input values validated server-side before being passed to Prisma
- [x] **Accessibility pass (2026-05-28)** — HIGH: `BillChangeNoticeForm` textarea now has `id`/`htmlFor` link, `aria-describedby` on error, `aria-busy` on submit; `DecisionForm` same treatment + focus rings on Approve/Reject buttons; `RichTextEditor` toolbar buttons get `aria-label`, `aria-pressed`, and `focus-visible:ring`; optional `ariaLabel` prop passed through to TipTap `contenteditable`. MEDIUM: `SimpleTextSection` adds `aria-live="polite"` success announcement after save, `ariaLabel` passed to editor; `bills/page.tsx` table headers all get `scope="col"`, empty Downloads `<th>` gets `sr-only` label.
- [x] **Bug fixes (2026-05-29):**
  - Dashboard "Awaiting Your Action" — `approverDecisions` query was gated on `isApprover` (role-based); any user can be added to an approval path regardless of role (e.g. ADMIN as SME), so gate removed; query now runs for all users unconditionally
  - Dashboard — APOC missing "Signal SME review complete" action item; added `apocSmeReview` query for `SME_REVIEW` bills where `apocId === userId`
  - Dashboard — LAI missing "Email artifacts to LA Contact" action item after Director approves; added `laiApproved` query for `APPROVED` bills where `createdById === userId`
  - `signalReviewComplete` — gate counted disabled SME decisions as required approvals; now filters to active SMEs only (`BillSme.disabledAt IS NULL`) before checking all-approved; error message updated to "All active reviewers must approve"
  - `adminDecisions` in `WorkflowPanel` — also included disabled SME decisions, causing `ReviewProgress` to show wrong count (e.g. "1 of 3" instead of "1 of 1"); now excludes decisions for disabled SMEs via `disabledSmeIds` set
  - `ExecApproverSelector` — incorrectly shown to APOC during LAI_REVIEW; exec approver selection is LAI responsibility only; UI condition and `setExecApprovers` Server Action both restricted to LAI + ADMIN
  - `BillHeader` — LAI (bill creator) was not shown in the header people row; added `createdBy` to `BillHeaderProps` and renders as "LAI:" first in the `<dl>`
  - COO bill access — verified working (2026-05-29); COO gains access when `laiApprove` creates their `ApprovalDecision` row on transition to EXECUTIVE_REVIEW; no code change needed

---

## Known Gaps & Bugs (not yet scheduled)

### ~~Bug — Proxy users cannot see a bill before submitting their first decision~~ ✅ Fixed (session 11)

Both `bills/page.tsx` and `bills/[id]/page.tsx` now include `{ proxyAssignments: { some: { proxyUserId: userId } } }` in the access check. `roles-permissions.md` bill list table updated to reflect correct scoping.

---

### Gap — Mark Enrolled action and post-enrollment behavior needs end-to-end review *(flagged 2026-05-29)*

`markEnrolled` exists (APPROVED → ENROLLED) and Section 12 conditional display is implemented, but the full post-enrollment flow has not been tested end-to-end. Review checklist:

- `markEnrolled` Server Action in `app/actions/workflow.ts`
- Section 12 fields (Vote Summary Senate/House, Admin Rules Impact, Technical Issues, Director Signature + Date) — editable only after ENROLLED
- Whether any email notification is warranted on enrollment
- Bill list, dashboard, and audit trail correctly reflect ENROLLED status
- Email Artifacts button is available at ENROLLED — verify it works from that status too

---

### Gap — No complete single-query history of everyone ever associated with a bill
`ApprovalDecision` rows are never deleted and capture everyone who had a formal decision requested. Combined with `BillSme` (including disabled rows after the disable-SME feature) and `AuditLog`, a near-complete picture is achievable. The remaining blind spot: if a single-person FK (APOC, LA Contact, exec approver) is swapped **before** that person generates any `ApprovalDecision` row, the only evidence of their prior assignment is in `AuditLog` — indirect and edit-based.

Clean fix when needed: a `BillAssociationHistory` log table that records every FK swap event (role, old user, new user, changed by, timestamp). Not urgent while APOC/exec approver changes remain rare.

---

### Phase 8 — Testing *(in progress)*

**Stack:** Vitest 4.x (integration), Playwright (E2E — not started)

**Infrastructure (2026-05-30):**
- [x] `vitest.config.ts` — node environment, `fileParallelism: false` (sequential), `clearMocks: true`, `@/*` alias wired
- [x] `tests/global-setup.ts` — loads `.env` + `.env.local` before workers start so Prisma gets `DATABASE_URL`
- [x] `tests/helpers/session.ts` — `makeSession()` returns the session shape actions read from `auth()`
- [x] `tests/helpers/factory.ts` — `createWorld()` creates 10 users + org units with cleanup; `buildBillAt(world, status)` advances a bill to any workflow state via direct DB writes (bypasses auth checks)

**Workflow integration tests (2026-05-30) — 46 tests, all passing:**
- [x] `tests/workflow/lifecycle.test.ts` — 10 tests walking full DRAFT → ENROLLED happy path; verifies DB state at every step including auto-advance to APPROVED
- [x] `tests/workflow/gates.test.ts` — 9 tests: Gate 1 (Sr. Deputy ordering), Gate 2 (COO/conditional approver ordering), Gate 3 (Director/COO ordering), `signalReviewComplete` blocking, disabled-SME exclusion from all gates
- [x] `tests/workflow/permissions.test.ts` — 27 tests: role enforcement (unauthenticated, wrong role, wrong assignee, wrong status) across `submitBill`, `createApprovalPaths`, `signalReviewComplete`, `setExecApprovers`, `laiApprove`, `recordDecision`, `markEnrolled`, `recordBillChange`

**Run:** `npm test` (single run) · `npm run test:watch` (watch mode)

**Mocking pattern:** `auth`, `next/cache`, all email notifications mocked per test file via `vi.mock`. Each test sets session explicitly: `vi.mocked(auth).mockResolvedValue(makeSession(user) as any)`. Real Neon dev DB — no Prisma mocking.

**Remaining within workflow integration tests (category 1):**
- [ ] Proxy assignment + proxied approval (`assignProxy`, proxy submitting via `recordDecision`)
- [ ] Audit log entry verification on section saves and workflow actions
- [ ] Admin override actions (`adminForceStatus`, `adminReassignApoc`, `adminVoidDecision`)
- [ ] Surgical path editing (`addSmeToPath`, `setSrDeputy`, `removeSrDeputy`, `addPath`, `removePath`)
- [ ] Disable/re-enable SME (`disableSme`, `enableSme`) beyond incidental gate coverage
- [ ] Bill section saves (`updateBill` — approval invalidation on edit, completion %)
- [ ] Bill creation (`createBill`) — package bills, previous review linking, version numbers

**Remaining other categories:**
- [ ] E2E critical role paths (Playwright) — role-based UI differences, multi-step approval flow through browser
- [ ] Admin action integration tests (`actions/admin.ts`) — user CRUD, org hierarchy, role assignment
- [ ] API route integration tests — `/api/documents/upload`, `/api/documents/[id]`, `/api/pdf/bill/[id]`
- [ ] Zod schema unit tests, auth/middleware unit tests, staging smoke tests

---

## Sequencing

```
Phase 1 ✅ → Phase 2 ✅ → Phase 3 ✅ → Phase 4 ✅ (real-time collab deferred)
                                                      ↓
                                              Phase 5 ⬤ (5/6 triggers done; "Email Artifacts" waits on Phase 6 PDFs)
                                                      ↓
                                              Phase 6 (PDF generation + comparison links + "Email Artifacts" trigger)
                                                      ↓
                                              Phase 7 ✅ (polish + dashboard + bug fixes)
                                                      ↓
                                              Phase 8 ⬤ (testing — workflow integration done; E2E + admin + API routes remaining)
```

**Phase 4 deferred:** Real-time collaborative editing (Yjs + Hocuspocus on Railway) — highest risk, moved to after Phase 6.

> **Key recommendation:** Spend extra design time on the approval path schema in Phase 2 before writing any Phase 3 UI. A wrong data model at that layer is expensive to unwind.
