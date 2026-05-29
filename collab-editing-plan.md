# Collaborative Real-Time Editing — Implementation Plan

*Documented: 2026-05-29. Status: Pre-implementation — concerns must be resolved before building.*

---

## Context

All rich text fields in LEGIS (Sections 2–10, 13; ~15 fields total) use `RichTextEditor` — a TipTap client component that syncs editor JSON to a hidden input, submitting via a Server Action form. Save is explicit. Content is stored as TipTap JSON in `BillAnalysis` columns.

Real-time collaboration is the last remaining deferred Phase 4 feature.

---

## Agreed Architecture

### Three new pieces

**1. Hocuspocus server** — standalone Node.js WebSocket server deployed as a separate Railway service. Manages Yjs document rooms, awareness (cursor presence), and authentication. One room per field per bill:

```
room name: ${billId}__${fieldName}
```

**2. JWT token endpoint** — new Next.js API route `/api/collab/token`. Called by the client before connecting. Validates:
- User session
- Bill access (assigned LAI, APOC, LA Contact, SME, Sr. Deputy, proxy)
- Edit vs. read-only permission
- Executive lock state

Returns a short-lived JWT (5-minute expiry) containing `userId`, `billId`, `fieldName`, and a `readOnly` flag. Hocuspocus validates this in its `onAuthenticate` hook for every WebSocket connection.

**3. Updated `RichTextEditor`** — gains `HocuspocusProvider`, `Collaboration`, and `CollaborationCursor` TipTap extensions. Fetches token from `/api/collab/token`, connects to the Hocuspocus WebSocket URL (`NEXT_PUBLIC_COLLAB_WS_URL`), shows other users' cursors with name and a deterministic colour derived from `userId`.

### Monorepo structure

```
/                          ← git root
  legis/                   ← existing Next.js service (Railway service 1)
  hocuspocus/              ← new Hocuspocus service (Railway service 2)
    src/server.ts
    package.json
    railway.json
```

### Environment variables needed

| Service | Variable | Purpose |
|---|---|---|
| Next.js | `NEXT_PUBLIC_COLLAB_WS_URL` | Hocuspocus WebSocket URL |
| Next.js | `COLLAB_SECRET` | Signs collab JWTs |
| Hocuspocus | `COLLAB_SECRET` | Verifies collab JWTs |
| Hocuspocus | `PORT` | Injected by Railway |

---

## Persistence Strategy — Modified Option C

**Core principle: Yjs is a synchronization layer, not a storage layer. Content is stored in exactly one place: `BillAnalysis`.**

```
BillAnalysis (single source of truth)
        ↑  storeData (debounced write)
        |
   Hocuspocus (sync + awareness)
        |
       Yjs (CRDT, in-memory per room)
```

### On room open (`onLoadDocument` hook)
- Load existing TipTap JSON from `BillAnalysis` for the field
- Convert TipTap JSON → Yjs binary using `y-prosemirror` (`prosemirrorJSONToYDoc`)
- Initialize the Yjs document from that content
- Clients connect and begin collaborating from the current saved state

### During editing (`storeData` hook, debounced)
- Hocuspocus receives Yjs updates from connected clients
- After a debounce period (target: 5 seconds), convert Yjs → TipTap JSON (`yDocToProsemirrorJSON`)
- Persist TipTap JSON to `BillAnalysis` via an internal Next.js API route (see open concerns)

### On room close (`onDestroy` hook)
- Flush any pending debounced write before the room is destroyed
- Best-effort — not a hard guarantee (see open concerns)

### Why not Option A (ephemeral Yjs)
- Users expect collaborative edits to persist between sessions
- Browser crashes and tab closes lose all in-memory state
- Last user leaving a room destroys unsaved work

### Why not Option B (BillAnalysis JSON + Yjs binary)
- Two representations of the same content that can silently drift apart
- Violates single source of truth

---

## Save Button — Repurposed

| | Today | After collaboration |
|---|---|---|
| **Content persistence** | Save button | Auto-save (debounced, background) |
| **Save button action** | Persist content | Workflow action: void approvals + write audit log entry |

Content is persisted automatically in the background. The Save button becomes the "commit with consequences" action — the moment a user declares the field ready and triggers downstream effects.

**Scope of Save button post-collaboration:**
- Call `voidActiveApprovals` (invalidate downstream decisions)
- Write the final diff to `AuditLog` via `diffFields()`

Nothing else. "Create revision" and "mark draft ready" are out of scope for this implementation.

---

## Read-Only Collaborative Viewers

Users with bill access but without edit rights (SMEs viewing other sections, Sr. Deputies during executive lock, exec approvers) connect as read-only viewers — they see live content and cursors but cannot make changes.

This requires enforcement at **all three layers**:
1. `/api/collab/token` — sets `readOnly: true` in the JWT payload based on user role and bill state
2. Hocuspocus `onAuthenticate` — reads `readOnly` from JWT; rejects any update messages (Yjs document updates) from read-only connections at the server level
3. `RichTextEditor` — sets TipTap `editable: false` for read-only sessions; provider still connects for cursor/awareness

Client-side `editable: false` is not sufficient — the server must also reject updates from read-only connections.

---

## Open Concerns — Must Resolve Before Building

### 1. `voidActiveApprovals` timing (most critical)

Auto-save writes TipTap JSON to `BillAnalysis` every 5 seconds without calling `voidActiveApprovals`. This creates a window where bill content has changed in the database but approvals have not been voided — an approver could hold an APPROVED decision on content that no longer reflects what they approved.

**Three candidate approaches — one must be chosen:**

**A.** `voidActiveApprovals` fires on the first auto-save after a field changes, then is suppressed for subsequent debounce ticks of the same session. Requires tracking "has content changed since last void" state in the Hocuspocus room.

**B.** `voidActiveApprovals` fires when the last user leaves the room (room close / `onDestroy`). Clean boundary but relies on the flush-on-close mechanism (see Concern 2).

**C.** `voidActiveApprovals` remains exclusively tied to the Save button click. Auto-save content is considered "in progress" until Save is clicked — but this means the background persistence guarantee is weakened (content is saved, but approvals are stale until explicit Save).

Option C is the simplest and preserves the existing behaviour most faithfully. It does mean approvals can be stale during an active editing session — which was already true today (edits are not saved at all until Save is clicked). May be acceptable.

---

### 2. Audit log impact of auto-save

The existing audit trail writes a `diffFields()` entry to `AuditLog` in the same transaction as every `BillAnalysis` update. Auto-save bypasses this path.

**Two options — one must be chosen:**

**A.** Auto-save skips audit log entries entirely. The Save button (or room close flush) writes a single final diff entry. This keeps the audit log clean but creates a gap: intermediate collaborative edits within a session are not individually logged.

**B.** Auto-save writes audit entries. The audit log becomes extremely noisy (potentially dozens of entries per editing session per field) and `AuditLogPanel` becomes hard to use.

Option A is strongly preferred. The audit log should record intentional save events, not every debounce tick.

---

### 3. Hocuspocus → BillAnalysis write mechanism

The critique says "persist to BillAnalysis" but doesn't specify how. Hocuspocus is a separate process and cannot call Next.js Server Actions directly.

**Two options — one must be chosen:**

**A. Internal Next.js API route** — `POST /api/collab/store`, authenticated with `COLLAB_SECRET`. Hocuspocus calls this on each debounced write. Keeps all DB writes inside Next.js where existing transaction patterns live. Adds one network hop per auto-save.

**B. Direct DB access from Hocuspocus** — Hocuspocus connects to Neon via `DATABASE_URL` directly using `pg` or Prisma. Faster, but Hocuspocus has full DB write access broader than needed, and schema changes in `BillAnalysis` must be reflected in the Hocuspocus service.

Option A is preferred. All BillAnalysis writes should go through Next.js to maintain transactional consistency with audit log and approval void logic.

---

### 4. Flush-on-room-close reliability

The plan describes "flush pending updates on room close" as a clean step. In practice, Hocuspocus's `onDestroy` fires for graceful disconnections only. It does not fire if:
- The Railway Hocuspocus service is restarted (deploy, crash, OOM kill)
- A client's network drops without a clean WebSocket close frame

**The debounced write is the real reliability mechanism.** Maximum potential data loss = one debounce interval (target: 5 seconds). Room-close flush is best-effort, not a guarantee.

This should be documented in code comments and communicated to users if data-loss risk is a compliance concern.

---

## Implementation Order

*After all four concerns above are resolved.*

1. Create `hocuspocus/` service — bare-bones Node.js server; `onAuthenticate` JWT verification; room lifecycle hooks; no persistence yet
2. Add `/api/collab/token` to Next.js — session check, bill access check, edit vs. read-only flag, JWT sign with `COLLAB_SECRET`
3. Update `RichTextEditor` — `HocuspocusProvider`, `Collaboration`, `CollaborationCursor` extensions; token fetch; cursor colour from `userId` hash; `editable` prop controls TipTap editability
4. Wire `onLoadDocument` — load TipTap JSON from BillAnalysis, convert to Yjs via `y-prosemirror`
5. Wire `storeData` — debounced write; Yjs → TipTap JSON conversion; POST to `/api/collab/store`
6. Add `/api/collab/store` internal route — validate `COLLAB_SECRET`, write TipTap JSON to `BillAnalysis`, skip audit log
7. Repurpose Save button — `voidActiveApprovals` + single audit log diff entry; content persistence removed from this path
8. Read-only enforcement — server-side update rejection in `onAuthenticate` for `readOnly` connections
9. Deploy Hocuspocus to Railway — second service, env vars, WebSocket routing verified
10. Smoke test — two browser sessions on the same bill field; verify cursors, sync, and persistence

---

## Fields in Scope

All TipTap rich text fields across Sections 2–10, 13:

| Section | Field(s) |
|---|---|
| 2 | `intentOfLegislation` |
| 3 | `summary` |
| 4 | `positionDetails` |
| 5 | `suggestedChangesNarrative` |
| 6 | `fiscalDeptBudgetary`, `fiscalDeptComments`, `fiscalStateBudgetary`, `fiscalLocalGovt` |
| 7 | `significantChanges` |
| 8 | `proArguments`, `conArguments` |
| 9 | `stakeholderPositions` |
| 10 | `background`, `problemBeingAddressed` |
| 13 | `finOpsReview` |

**Not in scope:** `legalCitation` (Section 11) — plain text textarea, excluded per requirements.
