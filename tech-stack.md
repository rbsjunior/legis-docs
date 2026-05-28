# LEGIS — Technical Stack

*Documented: 2026-05-01 — Updated: 2026-05-28 (session 12)*

---

## Infrastructure (decided)

| Layer | Choice | Notes |
|---|---|---|
| Hosting | Railway | All services deployed here |
| Database | Neon (managed PostgreSQL) | Pooled connection URL via `DATABASE_URL` |
| Auth | NextAuth.js v5 — Credentials provider | Username + password; login field is "User ID"; Entra ID deferred; `trustHost: true` set for Railway |
| File Storage | Railway Storage (S3-compatible) | Bill draft uploads: PDF, HTML, markup |

---

## Core Framework

**Next.js 15 (App Router + Server Actions)**

- Natural pairing with NextAuth.js v5
- Server Actions handle form-heavy mutations without a separate API layer
- Single repo covers frontend + backend

---

## Data & ORM

**Prisma 7** on **Neon PostgreSQL**

> **Prisma 7 breaking changes from training data:**
> - Datasource URL lives in `prisma/prisma.config.ts` (not `schema.prisma`).
> - `PrismaClient` requires the `@prisma/adapter-pg` driver adapter — instantiate with `new PrismaPg({ connectionString })`.
> - Generated client path: `app/generated/prisma/client` (not `@prisma/client` directly — import from that path).
> - Node.js ≥ 22.12.0 required (set via `engines` in `package.json`; Railway: `NIXPACKS_NODE_VERSION=22`).

- Enums map directly to `DraftStatus`, `Priority`, `LeadAgencyPosition`, `ApprovalStatus`, `WorkflowStatus`, `ApprovalPathType`
- **Multi-role:** `User.roles Role[]` (PostgreSQL array) — a user may hold multiple roles simultaneously; all role checks use `.some()` / `.includes()` against the array; JWT and session carry `roles: Role[]`
- **Username login:** `User.username String? @unique` — login identifier is `<lastname><firstinitial>` (e.g. `burtonc`); clash resolved as `<lastname><firstname>` then `<lastname><firstname>2` etc.; `makeUsername()` helper in seed scripts handles titles (Dr., Mr., etc.); username set at seed time and editable in Admin → Users; login form field is labelled "User ID"
- **Session staleness:** JWT tokens are cached for the session lifetime; role changes (e.g. granting ADMIN) do not take effect until the affected user signs out and back in
- `BillSeniorDeputy` table removed — Sr. Deputies assigned exclusively by APOC via `BillApprovalPath.srDeputyId`; Sr. Deputy access derived from `ApprovalDecision` rows
- Full org hierarchy: `Administration → Bureau → Division → Section`; users may be assigned to any single level; `User.sectionId` is nullable (Section is optional for all users); Section has `divisionId` (normal path) OR `bureauId` (directly under a bureau with no intermediate division); Division has `bureauId: null` for standalone divisions (e.g. Legislative Affairs reports to COO, not a bureau)
- Access scoping enforced in Server Actions (LA roles see all; SME/approver roles see only assigned bills)
- Use Neon's pooled connection string in production; dev branch URL in `.env.local`
- Migrations run at deploy time: `npx prisma migrate deploy` in `railway.json` start command

---

## UI

| Package | Purpose |
|---|---|
| shadcn/ui + Tailwind CSS v4 | Accessible, composable components — well-suited to data-dense government forms |
| TipTap | Rich text editor for the 10+ `Rich Text` fields; supports per-contribution attribution via custom extensions |
| Zod | Server-side schema validation in Server Actions |

### shadcn/ui — Nova Preset

shadcn is installed with the **`base-nova`** preset, which uses **`@base-ui/react`** (Base UI from MUI) as the primitive layer — **not Radix UI**. This affects component APIs:

- `Select` is a custom dropdown (`SelectPrimitive.Root / Trigger / Popup / Positioner / Item`), not a native `<select>`. Supports a `name` prop for form participation via a hidden input.
- `Checkbox` uses `@base-ui/react/checkbox` with `data-checked` attribute and `onCheckedChange` (not `onChange`).
- `Button` uses `@base-ui/react/button`.
- Tailwind CSS v4: CSS-first config — no `tailwind.config.js`; theme defined via `@theme inline` in `globals.css`; colors in oklch format.
- `eslint-config-next` requires `.js` extension on ESM imports (`eslint-config-next/core-web-vitals.js`, not `eslint-config-next/core-web-vitals`).

### URL-param driven filtering

For read-only filtering/search (no mutation), URL search params are used instead of Server Actions:

- A `"use client"` component reads current params via `useSearchParams()` and writes updates via `router.replace()` (not `router.push()` — avoids polluting browser history with every keystroke)
- Text search is debounced 300 ms in a `useEffect` before the URL update fires
- The server page receives `searchParams: Promise<{ [key: string]: string | undefined }>` (Next.js 15 — must be `await`ed before reading values)
- Filter values are validated server-side against allow-lists before being passed to Prisma
- Implemented: Bill list (`/bills`) — text search (bill number + topic icontains), status, priority

### Forms

React Hook Form is **not used**. All mutations go through **React Server Actions** with `useActionState<State, FormData>()`:

- Form elements use native `name` attributes; Server Actions read `formData.get()` / `formData.getAll()`.
- Zod validates on the server inside each action.
- `revalidatePath()` refreshes server component data after successful mutations.
- `successAt: number` (timestamp) pattern lets `useEffect` close the edit panel on every save, including repeat saves of the same section.

---

## Workflow & Approvals

**DB-driven state machine** (Prisma enums + typed server-side transition functions)

The approval path logic — while branching — follows predictable, well-defined rules. No external state machine library needed. Approach:

- `ApprovalStatus` enum tracks each approver's decision per bill
- Transition functions validate allowed moves server-side before committing
- Parallel Administration paths modeled as sibling records converging at COO
- Approval invalidation on edit: a DB trigger or Prisma middleware voids downstream approvals on field write

---

## Admin

**ADMIN role — triple-enforced**

Access to `/admin/*` is restricted to users holding the `ADMIN` role, enforced at three independent layers:

1. `middleware.ts` — Edge-side redirect before the page renders
2. `app/(app)/admin/layout.tsx` — Server-side session check; redirects to `/dashboard`
3. Every function in `app/actions/admin.ts` — `requireAdmin()` re-checks before any DB write

The `ADMIN` role can be assigned to any user via the Users admin page (`/admin/users`). Because sessions are JWT-based, the change takes effect on the affected user's next sign-in.

**Admin pages:**

| Page | Path | Actions |
|---|---|---|
| Org Structure | `/admin/org` | CRUD for Administrations, Bureaus, Divisions, Sections; delete blocked when referencing records exist |
| Users | `/admin/users` | Create/edit users; multi-role checkboxes; org unit assignment (any level); activate/deactivate toggle |

---

## Email

**Nodemailer** + **Gmail SMTP** — implemented Phase 5

- `nodemailer` v7 with `smtp.gmail.com:587` (STARTTLS); auth via Gmail App Password (`GOOGLE_APP_PASSWORD`)
- `GMAIL_USER` = sending Gmail address; display name set in Gmail account settings (Settings → Accounts → Send mail as)
- `nodemailer` added to `serverExternalPackages` in `next.config.ts` to prevent webpack bundling
- Fire-and-forget `void` pattern in Server Actions — email failures never block workflow transitions
- HTML templates as pure TypeScript functions in `app/lib/emails/templates.ts`; no React Email dependency
- `scripts/test-email.ts` verifies SMTP connection and sends a test message (`npm run email:test`)
- Seed user emails use Gmail plus-addressing: `srbsjunior+{role}@gmail.com`

> **Not used:** Resend, React Email, Inngest — Nodemailer + Gmail is sufficient for this internal low-volume app.

---

## File Storage & PDF

| Need | Tool |
|---|---|
| Upload/retrieve bill drafts (PDF, HTML, markup) | Railway Storage via **AWS SDK v3** (S3-compatible API) — **implemented Phase 4** |
| Generate PDF bill analysis + routing sheet | **Puppeteer** — renders Next.js templates server-side to PDF |
| Extract text from uploaded PDFs for diffing | **pdf-parse** |
| Bill version comparison | Link out to external comparison tool (per requirements); integration point within the record |

**Railway Storage implementation details (Phase 4):**
- Endpoint: `https://t3.storageapi.dev` (Tigris-backed Railway Object Storage)
- Files stored under `bills/{billAnalysisId}/{uuid}.{ext}`; keys held in `BillDocument.storageKey`
- Uploads: `POST /api/documents/upload` (Route Handler — bypasses Next.js Server Action 1 MB body limit; handles up to 50 MB)
- Downloads: `GET /api/documents/[id]` (generates 15-min presigned S3 URL, redirects browser; access-gated by bill assignment)
- `app/lib/storage.ts` wraps `uploadFile` / `getDownloadUrl` / `deleteFile`
- Accepted MIME types: `application/pdf`, `text/html`, `text/plain`, `text/markdown`, `application/xml`, `text/xml`

> **Note on Puppeteer:** Memory-intensive per render. If PDF generation volume grows, migrate to **React PDF** (pure JS, no headless browser) — requires building layouts manually.

---

## Summary

```
Next.js 15 (App Router)
  ├── Prisma 7 + @prisma/adapter-pg → Neon PostgreSQL
  ├── NextAuth.js v5 (Credentials; Entra ID later)
  ├── shadcn/ui (Nova preset / @base-ui/react) + Tailwind CSS v4
  ├── Server Actions + useActionState + Zod (forms + validation — no React Hook Form)
  ├── TipTap (rich text — Phase 4)
  ├── Nodemailer + Gmail SMTP (transactional email — Phase 5)
  ├── Railway Storage + AWS SDK v3 (file uploads — implemented Phase 4)
  └── Puppeteer (server-side PDF generation — Phase 6)
```
