# LEGIS — Roles & Permissions

*Documented: 2026-05-27 (session 10) — Updated: 2026-05-27 (session 11)*

---

## Roles

| Role | Code | Division | Purpose |
|---|---|---|---|
| Administrator | `ADMIN` | Legislative Affairs | System administration; overrides all assignment restrictions |
| Legislative Affairs Initiator | `LAI` | Legislative Affairs | Creates and submits bill analyses; approves for executive review |
| Administrative Point of Contact | `APOC` | Legislative Affairs | Coordinates review; builds approval paths; signals review complete |
| Legislative Affairs Contact | `LA_CONTACT` | Legislative Affairs | First-level approval; full read/edit on all bills |
| Subject Matter Expert | `SME` | Any Bureau / Division / Administration | Expert analysis and review; scoped to assigned bills |
| Senior Deputy Director | `SENIOR_DEPUTY` | Administration | Administration-level approval; scoped to assigned bills |
| Chief Operating Officer | `COO` | Executive | Required approver on all analyses |
| Director | `DIRECTOR` | Executive | Final approval authority — required on all analyses |
| CAO | `CAO` | Executive | Conditional executive approver |
| Health Services Director | `HSD` | Executive | Conditional executive approver |
| Chief Medical Executive | `CME` | Executive | Conditional executive approver |
| Proxy Approver | `PROXY_APPROVER` | Any | Temporary approval authority assigned per-bill; no other access |

A user may hold multiple roles simultaneously. Access rights are the union of all held roles.

---

## Bill List (`/bills`)

| Roles | What they see |
|---|---|
| ADMIN, LAI, APOC, LA_CONTACT | All bill analyses |
| SME, SENIOR_DEPUTY, COO, DIRECTOR, CAO, HSD, CME, PROXY_APPROVER | Only bills they are assigned to (as APOC, LA Contact, active or disabled SME, have an `ApprovalDecision` row, or are listed in `BillProxyAssignment`) |

---

## Bill Creation (`/bills/new`)

Restricted to: **ADMIN, LAI, APOC, LA_CONTACT**

Enforced in three places: `bills/page.tsx` (button visibility), `bills/new/page.tsx` (redirect guard), `app/actions/bills.ts` (server action check).

---

## Bill Detail — Read Access

| Roles | Access |
|---|---|
| ADMIN, LAI, APOC, LA_CONTACT | All bills |
| All other roles | Only bills they are assigned to (same scoping as bill list) |

Non-LA users who are not assigned to a bill receive a 404.

---

## Bill Detail — Edit Access (Sections 2–13)

A user may edit when **both** conditions are met:

**Condition A — Assignment:**
- Holds an LA role (ADMIN / LAI / APOC / LA_CONTACT), OR
- Is the assigned APOC, LA Contact, **active** SME (`disabledAt IS NULL`), or Sr. Deputy on the bill

> Disabled SMEs (`disabledAt IS NOT NULL`) retain **read** access but lose edit access. Their `BillSme` row is preserved for history.

**Condition B — Status lock:**
- Bill status is DRAFT, SUBMITTED, SME_REVIEW, or LAI_REVIEW → any user satisfying Condition A may edit
- Bill status is EXECUTIVE_REVIEW → only exec roles (COO, DIRECTOR, CAO, HSD, CME) may edit
- Bill status is APPROVED or ENROLLED → **ADMIN only**

---

## Workflow Actions

| Action | Permitted roles | Additional restriction |
|---|---|---|
| Submit bill (DRAFT → SUBMITTED) | LAI, ADMIN | Must be the bill creator (ADMIN bypasses) |
| Create approval paths (SUBMITTED → SME_REVIEW) | APOC, ADMIN | Must be the assigned APOC (ADMIN bypasses) |
| Signal review complete (SME_REVIEW → LAI_REVIEW) | APOC, ADMIN | Must be the assigned APOC (ADMIN bypasses) |
| Set executive approvers | APOC, LAI, ADMIN | Must be the assigned APOC or creator LAI (ADMIN bypasses) |
| LAI approve (LAI_REVIEW → EXECUTIVE_REVIEW) | LAI, ADMIN | Must be the bill creator (ADMIN bypasses) |
| Record decision (APPROVE / REJECT) | Assigned approver only | Proxy approver may submit on behalf of their assigned principal |
| Mark enrolled (APPROVED → ENROLLED) | LAI, ADMIN | — |

### Per-path editing actions (surgical — do not reset other paths)

These actions modify individual paths without touching the rest of the approval chain. All are blocked at APPROVED/ENROLLED.

| Action | Permitted roles |
|---|---|
| Add SME to an existing path | DRAFT–LAI_REVIEW: APOC, ADMIN · EXECUTIVE_REVIEW: LAI, ADMIN |
| Remove Sr. Deputy from a path | Same as above · Blocked if any active SME exists on the path |
| Add Sr. Deputy to a path | Same as above |
| Add a new path | Same as above |
| Remove a path | Same as above · Disables path's SMEs, voids all path decisions |
| Disable / re-enable an SME | Same as above (see SME Association section below) |

### Approval ordering gates (enforced in `recordDecision`)

1. **SENIOR_DEPUTY** cannot approve until all **active** (`disabledAt IS NULL`) SMEs on the same path have APPROVED — disabled SMEs are excluded from this gate
2. **COO** cannot approve until all required conditional approvers (CAO / HSD / CME) have APPROVED
3. **DIRECTOR** cannot approve until the COO has APPROVED

---

## Admin Panel (`/admin/*`)

Restricted to: **ADMIN** only

Triple-enforced: `middleware.ts` (Edge), `app/(app)/admin/layout.tsx` (server), every action in `app/actions/admin.ts`.

| Page | Actions |
|---|---|
| `/admin/org` | CRUD for Administrations, Bureaus, Divisions, Sections |
| `/admin/users` | Create / edit users; multi-role assignment; org unit assignment; activate / deactivate |

---

## Admin Override Controls

Accessible via the amber "Admin override" collapsible in the WorkflowPanel on any bill detail page. Visible only to **ADMIN** users. All actions require a reason (≥10 characters) and a two-step confirmation. Every action writes two `AuditLog` rows: the field change and the reason.

### Roll back status

Moves a bill to any **earlier** workflow status, bypassing all gates. Forward moves are intentionally blocked — use normal workflow actions (which already bypass assignment checks for ADMIN) to advance a bill.

| Target status | Additional behavior |
|---|---|
| Any earlier status | All non-voided decisions are voided |
| `SME_REVIEW` | Fresh PENDING decisions created for all active SMEs and Sr. Deputies on existing approval paths |
| `EXECUTIVE_REVIEW` | Fresh PENDING decisions created for all assigned executive approvers |
| `DRAFT` | `draftStatus` also reset to DRAFT |

Common use cases: Director rejected → roll back to `LAI_REVIEW` to restart exec review; wrong paths built → roll back to `SUBMITTED` so APOC can rebuild; accidental progression.

### Reassign APOC

Replaces the `apocId` FK on the bill. Validates that the new user is active and holds the APOC role. Available at any workflow stage — the only way to change APOC after a bill is created.

### Void a specific decision

Voids one non-voided decision (PENDING, APPROVED, or REJECTED) and immediately creates a fresh PENDING for the same approver and role. Does **not** change the bill's workflow status. More surgical than a full rollback when only one decision needs clearing (e.g. Director rejected due to a misunderstanding resolved the same day).

---

## Document Uploads

Add / remove / promote documents: **LAI, APOC, ADMIN**, at any stage prior to the executive lock (`EXECUTIVE_REVIEW` / `APPROVED` / `ENROLLED`).

---

## SME Association — Disable / Re-enable

Disabling an SME preserves their edit history, approval decisions, and connection to the bill review without deleting the `BillSme` row. A `disabledAt` timestamp marks the association inactive.

| Stage | Who can disable / re-enable |
|---|---|
| DRAFT, SUBMITTED, SME_REVIEW, LAI_REVIEW | APOC (assigned to this bill), ADMIN |
| EXECUTIVE_REVIEW | LAI, ADMIN — LAI retains full authority until all approvals are final |
| APPROVED, ENROLLED | ADMIN only |

- Disabled SMEs retain **read** access to the bill (their userId still appears in `BillSme`).
- Disabled SMEs lose **edit** access.
- Disabled SMEs are excluded from the Sr. Deputy ordering gate — the gate counts only active (`disabledAt IS NULL`) SMEs on the path.
- Any existing `BillProxyAssignment` for a disabled SME is treated as inactive; proxy submission on their behalf is rejected.

---

## Proxy Approver — Scope and Assignment

`PROXY_APPROVER` is a system role assigned in the admin panel. Any user may hold it alongside other roles.

**Proxy scope: decisions only.** The proxy system covers `recordDecision` (approve/reject). It does **not** cover LAI or APOC workflow transition actions (submit, create paths, signal review complete, set exec approvers, LAI approve, mark enrolled). Those transitions require the named role holder or ADMIN.

**Who can assign a proxy:** APOC (assigned to the bill), LAI (bill creator), or ADMIN — per-bill, per-original-approver. One proxy per original approver per bill; assigning a second proxy replaces the first.

**What a proxy can do:** Submit an `ApprovalDecision` on behalf of any pending approver (SME, Sr. Deputy, COO, Director, CAO, HSD, CME). The decision records `approverId` = original and `proxyApproverId` = proxy (audit trail).

**LAI / APOC absence:** Covered by the ADMIN bypass, not the proxy system. Extending proxy to workflow transitions creates a circular assignment problem (the unavailable party cannot assign their own proxy) and requires changes to every transition action separately.

---

## Historical Association Coverage

| Source | What it reliably captures |
|---|---|
| `BillAnalysis.createdById` | Original LAI creator — never changes |
| `BillSme` (incl. disabled rows) | All SMEs ever assigned to the bill |
| `ApprovalDecision.approverId` | Everyone who ever had a formal decision requested (SMEs, Sr. Deputies, exec approvers, LAI) — rows are never deleted |
| `ApprovalDecision.proxyApproverId` | Everyone who ever acted as proxy |
| `AuditLog.userId` | Everyone who ever edited any field |

**Gap:** If a single-person FK (APOC, LA Contact, exec approver) is replaced **before** that person generates any `ApprovalDecision` row, the only record of their earlier assignment is the `AuditLog` (indirect, edit-based). A `BillAssociationHistory` log table would close this gap cleanly.

---

## Implementation notes

- Role checks in the UI (button visibility, page redirects) are a convenience — **every sensitive action is re-checked server-side** in the relevant Server Action.
- `ADMIN` bypasses all per-bill assignment restrictions (assigned APOC check, creator LAI check, etc.) but not status locks — exec lock still applies.
- Session staleness: role changes take effect on the affected user's next sign-in (JWT-cached sessions).
