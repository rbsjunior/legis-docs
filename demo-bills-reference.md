# Demo Bills Reference

## Scripts

| Command | Target | Effect |
|---------|--------|--------|
| `npm run db:seed:demo` | Dev (`DATABASE_URL`) | Seed — skips bills that already exist |
| `npm run db:seed:demo:reset` | Dev (`DATABASE_URL`) | Delete existing demo bills, then re-seed |
| `npm run db:seed:demo:prod` | Prod (`DATABASE_URL_PROD`) | Seed — skips bills that already exist |
| `npm run db:seed:demo:prod:reset` | Prod (`DATABASE_URL_PROD`) | Delete existing demo bills, then re-seed |

All commands run from the `legis/` directory. The prod commands read `DATABASE_URL_PROD` from `.env.local` and will fail fast if it is not set.

---

## Bills by Workflow Stage

| Stage | Bill(s) | Topic | PA | Sponsor(s) |
|-------|---------|-------|----|------------|
| DRAFT (URGENT) | HB 4726 | Health Facilities: County Medical Care Facilities MOE Reimbursement Sunset Extension | 45/2025 | Rep. Matt Bierlein |
| SUBMITTED | SB 0136 | Health: Dense Breast Tissue Notification & Radiation Machine Amendments | 63/2025 | Sen. Sarah Anthony |
| SME_REVIEW | SB 0096 · SB 0097 · SB 0098 | Construction/Child Care: Temporary Locking Devices in Child Care Centers | 60–62/2025 | Sen. Jeremy Moss · Sen. Roger Hauck · Sen. Mallory McMorrow |
| LAI_REVIEW | HB 4077 · HB 4078 | Health: Medical Examiner Death Certification & Investigation Requirements | 3–4/2026 | Rep. Julie Rogers · Rep. Mike Mueller |
| EXECUTIVE_REVIEW | HB 5455 | Health Occupations: Interstate Medical Licensure Compact | 6/2026 | Rep. Rylee Linting |
| APPROVED | HB 4002 · SB 0008 | Labor: Earned Sick Time and Minimum Wage Modifications | 2–1/2025 | Rep. Jay DeBoyer · Sen. Kevin Hertel |
| ENROLLED | HB 4694 · HB 4695 · HB 4798 | Local Government: Recreational Authorities Act Revisions | 12–14/2026 | Rep. Gregory Markkanen |

---

## LAI Role Assignments

| Bill(s) | LAI | APOC | LA Contact |
|---------|-----|------|------------|
| HB 4726 | Chardae Burton (`burtonc`) | Jeffrey Spitzley (`spitzleyj`) | Jackie McKee (`mckeej`) |
| SB 0136 | Jeffrey Spitzley (`spitzleyj`) | Marina Wyrzykowski (`wyrzykowskim`) | Lesley Keyton (`keytonl`) |
| SB 0096–0098 | Marina Wyrzykowski (`wyrzykowskim`) | Mary Imre (`imrem`) | Caryn Shannon (`shannonc`) |
| HB 4077–4078 | Mary Imre (`imrem`) | Nicholas Rossow (`rossown`) | Elaine Heckman (`heckmane`) |
| HB 5455 | Nicholas Rossow (`rossown`) | Chardae Burton (`burtonc`) | Nicole Nelson (`nelsonn`) |
| HB 4002 · SB 0008 | Chardae Burton (`burtonc`) | Jeffrey Spitzley (`spitzleyj`) | Jackie McKee (`mckeej`) |
| HB 4694–4798 | Jeffrey Spitzley (`spitzleyj`) | Chardae Burton (`burtonc`) | Lesley Keyton (`keytonl`) |

---

## Approval Paths

| Bill(s) | Stage detail | Org unit | SME | Sr. Deputy | COO | Director |
|---------|-------------|----------|-----|------------|-----|----------|
| HB 4726 | No paths yet (DRAFT) | — | — | — | — | — |
| SB 0136 | No paths yet (SUBMITTED) | — | — | — | — | — |
| SB 0096–0098 | Path 1: SIA — SME approved, Sr. Deputy pending | SIA | Tony Weber (`webert`) — APPROVED | Sudhakar Ramaswamy (`ramaswamys`) — PENDING | — | — |
| SB 0096–0098 | Path 2: BEPC — SME pending | BEPC | Jay Fiedler (`fiedlerj`) — PENDING | — | — | — |
| HB 4077–4078 | Path: SIA — all approved | SIA | Thomas Nighswander (`nighswandert`) — APPROVED | Sudhakar Ramaswamy (`ramaswamys`) — APPROVED | — | — |
| HB 5455 | Path: PPOS — all approved; exec pending | PPOS | Amber Myers (`myersa`) — APPROVED | Elizabeth Nagel (`nagele`) — APPROVED | David Knezek (`knezekd`) — PENDING | Elizabeth Hertel (`hertele`) — PENDING |
| HB 4002 · SB 0008 | Path: BCS — all approved | BCS | Monica Bowman (`bowmanm`) — APPROVED | — | David Knezek (`knezekd`) — APPROVED | Elizabeth Hertel (`hertele`) — APPROVED |
| HB 4694–4798 | Path: PPOS — all approved | PPOS | Ninah Sasy (`sasyn`) — APPROVED | Elizabeth Nagel (`nagele`) — APPROVED | David Knezek (`knezekd`) — APPROVED | Elizabeth Hertel (`hertele`) — APPROVED |

---

## Department Positions

| Bill(s) | Position | Notes |
|---------|----------|-------|
| HB 4726 | SUPPORT | Extends Medicaid reimbursement eligibility for county medical care facilities |
| SB 0136 | OPPOSE AS WRITTEN | Recommends retaining state notification requirement; elimination removes independent enforcement |
| SB 0096–0098 | SUPPORT | SB 0098 directly amends Child Care Organizations Act administered by DHHS |
| HB 4077–4078 | SUPPORT | Department worked with sponsors; improves vital records data quality |
| HB 5455 | SUPPORT | Addresses physician shortages in rural/underserved areas; no new appropriation needed |
| HB 4002 · SB 0008 | SUPPORT | DHHS as employer (~50K employees) and Medicaid Home Help reimbursement implications |
| HB 4694–4798 | NEUTRAL | No intersection with departmental statutes or programs |