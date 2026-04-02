# Plan: AllCodex DM-First Roadmap Implementation (Phases 0-3)

> **Status: \u2705 ALL PHASES COMPLETE** (April 2, 2026)
>
> Every step in this plan has been implemented and validated with clean builds.
> This document is retained as the historical implementation specification.
>
> | Phase | Name | Status |
> |-------|------|--------|
> | A | Portal Config + Sanitization + Brain Dump History | \u2705 Complete |
> | B | Wiki View + Navigation + Hierarchy | \u2705 Complete |
> | C | Brain Dump Review Mode + Commit Flow | \u2705 Complete |
> | D | Session Workspace + Quests + Timeline | \u2705 Complete |
> | E1 | Contract Normalization + History Detail | \u2705 Complete |
> | E2 | Session/Quest/Scene Templates + Canonical Schema | \u2705 Complete |
> | F | Statblock Library + System-Pack Import + RAG Tag Filter | \u2705 Complete |
> | G | Player-Safe Sharing + Shared Content Browser | \u2705 Complete |

Date: April 2, 2026

## TL;DR

Implement the full DM-first roadmap from Phase 0 through Phase 3, deferring Azgaar import and Phase 5 (rules/statblocks) to last. Use Stitch designs (project `9354722103712192017`) as reference for each screen. Several Phase 0 features are already shipped — focus on remaining gaps then build forward.

## Current State

**Already built (don't re-build):**
- `TemplatePicker` (12 templates) in `components/editor/TemplatePicker.tsx`
- `PromotedFields` in `components/editor/PromotedFields.tsx`
- `LoreEditor` (Novel/Tiptap) in `components/editor/LoreEditor.tsx`
- Edit page: full attribute CRUD, draft toggle, template selection
- New page: template picker + promoted fields + content editor + parent ID
- Lore API `POST /api/lore` handles attributes, loreType, templateId
- Attribute API (POST + DELETE) at `app/api/lore/[id]/attributes/route.ts`

**Still TODO:**
- Portal Configuration in Settings (lore root note ID)
- `getLoreRootNoteId` helper in `lib/get-creds.ts`
- HTML sanitization (no DOMPurify installed)
- Enhanced `.lore-content` CSS
- Brain Dump history detail page (`/brain-dump/[id]`)
- Brain Dump contract normalization (Portal types notesCreated as number, AllKnower returns string[])

## Stitch Design Map

| Screen | Phase | Screen ID |
|---|---|---|
| Settings Page | A | `69e67453fc3d49dcb71292f6f9ec6d2c` |
| Lore Create & Edit | A (mostly done) | `14df200fa9fc4cd88e97911e2371a18a` |
| Brain Dump History Detail + Contradiction | A + C | `ea47483253894075b82d8fa2ae4197b8` |
| AllCodex Wiki View | B | `3524032d9c684b0f8dc022a5ba730f9a` |
| Components Sheet | Shared | `34951011d4f94a618aba0155a459aced` |
| Brain Dump Review Results | C | `78424d662279486a8f1f7e0ed95e45de` |
| Quest & Hook Tracker | D | `c36de168613f4c94afe886eac6d4f0eb` |
| Statblock Library | Deferred | `5941eae4cecb414b8493a18f9fd8e18d` |
| Azgaar Import Wizard | Deferred | `04268b09f57f4acd8111188123ae84eb` |

---

## Phase A: Remaining Foundation (Roadmap Phase 0)

All 4 steps run **in parallel** — no interdependencies.

### Step 1: Portal Config + Lore Root — `P0`
*No dependencies. Parallel with steps 2-3.*

1a. **New file:** `app/api/config/portal/route.ts`
- GET reads `lore_root_note_id` cookie → returns JSON
- PUT sets cookie with same COOKIE_OPTS as connect route (httpOnly, sameSite lax, path /, secure in prod, maxAge 30d)

1b. **Modify:** `lib/get-creds.ts`
- Add `getLoreRootNoteId()` — cookie → env LORE_ROOT_NOTE_ID → "root"

1c. **Modify:** `app/api/lore/route.ts`
- Import getLoreRootNoteId, replace hardcoded "root"

1d. **Modify:** `app/(portal)/settings/page.tsx`
- Add "Portal Configuration" card below service cards
- Lore Root Note ID text input + save/load via GET/PUT /api/config/portal
- Reference Stitch screen `69e67453fc3d49dcb71292f6f9ec6d2c`

1e. **Modify:** `app/(portal)/lore/new/page.tsx`
- Fetch lore root on mount, pre-populate Parent Note ID

**Files:** 1 new, 4 modified

### Step 2: HTML Sanitization — `P0`
*No dependencies. Parallel with steps 1, 3.*

2a. Install `isomorphic-dompurify`

2b. **New file:** `lib/sanitize.ts`
- `sanitizeLoreHtml(html)` — allow standard elements/classes/styles, strip scripts/event handlers/iframes

2c. **Modify:** `app/(portal)/lore/[id]/page.tsx`
- Sanitize before dangerouslySetInnerHTML

2d. **Modify:** `app/(portal)/lore/[id]/edit/page.tsx`
- Sanitize in any preview rendering

**Files:** 1 new, 1 dep, 2 modified

### Step 3: Enhanced `.lore-content` CSS — `P0`
*Parallel with steps 1-2.*

**Modify:** `app/globals.css`
- Responsive images, grimoire hr, code/pre, figure/figcaption, nested lists, CKEditor compat
- Reference Stitch "AllCodex Wiki View" for typography

**Files:** 1 modified

### Step 4: Brain Dump History Detail — `P0`
*No dependencies on 1-3. Parallel.*

4a. Investigate AllKnower history endpoint. `GET /brain-dump/history` returns last 20 with `notesCreated: string[]`, `notesUpdated: string[]`. Portal `HistoryEntry` type already has `entities` array + summary. Need to check if there's a single-entry endpoint or if we filter on portal side.

4b. **Modify:** `lib/allknower-server.ts`
- Add `getBrainDumpEntry(creds, id)` — call AllKnower, return single entry with full parsedJson

4c. **New file:** `app/api/brain-dump/history/[id]/route.ts`
- GET proxies to AllKnower

4d. **New file:** `app/(portal)/brain-dump/[id]/page.tsx`
- Header: date, model, tokens
- Full raw text in blockquote
- AI summary
- Entity cards: title, type, action, link to note
- Back button
- Reference Stitch screen `ea47483253894075b82d8fa2ae4197b8`

4e. **Modify:** `app/(portal)/brain-dump/page.tsx`
- Wrap history entries in `<Link href={/brain-dump/${entry.id}}>`
- Add hover affordance

**Files:** 2 new, 2 modified

### Phase A Verification
1. Settings → Portal Config card saves/loads lore root cookie
2. New lore entries default to configured root not "root"
3. Lore detail with `<script>` in content → no XSS
4. Click brain dump history → detail page with entities

---

## Phase B: Wiki Article View (Roadmap Phase 2)

*Depends on Phase A (sanitization + CSS)*

Reference Stitch: `3524032d9c684b0f8dc022a5ba730f9a`

### Step 5: Infobox + Article Redesign
*Depends on Phase A*

**Modify:** `app/(portal)/lore/[id]/page.tsx`
- Restructure sidebar into wiki-style infobox:
  - Title bar + lore type badge + template icon
  - Promoted attributes in labeled rows
  - Visual separator
- Improve main content area typography

### Step 6: Breadcrumbs + ToC
*Parallel with step 5*

6a. **New:** `components/portal/Breadcrumbs.tsx` — fetch note branch parents, render as nav chain

6b. **New:** `app/api/lore/[id]/breadcrumbs/route.ts` — return ancestor array via ETAPI branch traversal

6c. **New:** `components/portal/TableOfContents.tsx` — parse h1-h3, floating ToC for long articles

6d. **Modify:** `app/(portal)/lore/[id]/page.tsx` — add breadcrumbs above title, ToC in sidebar/gutter

### Step 7: Backlinks + Relationship Enhancement
*Parallel with steps 5-6*

7a. **New:** `app/api/lore/[id]/backlinks/route.ts` — ETAPI search for notes with relations TO this note

7b. **Modify:** `app/(portal)/lore/[id]/page.tsx` — show incoming + outgoing relations, grouped by type

### Step 8: Hierarchy Browser Upgrade
*Parallel with steps 5-7*

**Modify:** `components/portal/LoreTree.tsx` — replace flat type-filter with real tree using react-arborist with ETAPI branch data

**Modify:** `app/(portal)/lore/page.tsx` — wire hierarchy tree

### Step 9: Hover Previews
*Depends on step 5*

**New:** `components/portal/NotePreview.tsx` — hover card with title, type, promoted attrs, content snippet

**Modify:** `app/(portal)/lore/[id]/page.tsx` — wrap relation/content links with hover preview

### Phase B Verification
1. Lore detail has wiki infobox with structured attributes
2. Breadcrumbs show correct ancestry
3. Long articles have ToC
4. Backlinks visible
5. Hierarchy tree with expand/collapse
6. Hover previews on links

---

## Phase C: Brain Dump Enhancement (Roadmap Phase 1)

*Depends on Phase A step 4*

Reference Stitch: `78424d662279486a8f1f7e0ed95e45de`, `ea47483253894075b82d8fa2ae4197b8`

### Step 10: Entity Result Cards
*Depends on Phase A*

**Modify:** `app/(portal)/brain-dump/page.tsx`
- After dump: show entity cards with title, type badge, action, confidence, lore link
- Replace simple list with card grid

### Step 11: Brain Dump Mode Selector
*Depends on step 10*

**Modify:** `app/(portal)/brain-dump/page.tsx`
- Tabs: "Auto-Create" | "Review First" | "Inbox"
- Review: AllKnower processes without writing — user approves each entity

**AllKnower change required:** `POST /brain-dump` must accept `mode` param ("auto" | "review" | "inbox")

### Step 12: Contradiction Warnings
*Depends on step 10*

**Modify:** `app/(portal)/brain-dump/page.tsx`
- After dump, auto-run `checkConsistency` on created/updated noteIds
- Show contradiction panel: severity, description, affected notes

### Step 13: Duplicate Detection
*Depends on step 10*

- AllKnower pipeline returns `possibleDuplicates` per entity
- Portal shows "merge / skip / create" choices

### Phase C Verification
1. Entity cards with type badges after brain dump
2. Review-first mode shows entities without writing
3. Contradiction panel appears
4. Duplicate detection surfaces

---

## Phase D: Session Runtime (Roadmap Phase 3)

*Depends on Phases B+C*

Reference Stitch: `c36de168613f4c94afe886eac6d4f0eb`

### Step 14: Session Workspace

**New:** `app/(portal)/session/page.tsx` — split-pane: pinned notes + scene + recap + quick capture

**New:** `app/(portal)/session/layout.tsx` — session-specific layout

**Modify:** `components/portal/AppSidebar.tsx` — add Session nav entry

### Step 15: Quest & Hook Tracker
*Parallel with step 14*

**New:** `app/(portal)/quests/page.tsx` — list quests/hooks with status, linked notes

**New:** `app/api/quests/route.ts` — ETAPI search for `#quest` labels

**Modify:** sidebar

### Step 16: Timeline View
*Parallel with steps 14-15*

**New:** `app/(portal)/timeline/page.tsx` — list events by inWorldDate

**New:** `app/api/timeline/route.ts` — search for `#event`/`#timeline` labels

**Modify:** sidebar

### Step 17: Session Recap + Quick Capture
*Depends on step 14*

- Quick capture → brain dump auto-create
- Generate session recap summary
- Link to session note

### Phase D Verification
1. Session workspace with pinned notes, quick search
2. Quest tracker shows quests with note links
3. Timeline sorted by in-world date
4. Session recaps from brain dumps

---

## Phase E: AllKnower Contract Normalization + AllCodex Core APIs

**Ordering:** Subphase E1 runs **parallel with Phase A**. Subphase E2 runs **parallel with Phase B/D**.

### Subphase E1: AllKnower Contract Normalization — `P0`

*Runs alongside Phase A. Phase C steps 11-13 are blocked without this.*

#### Step 18: Fix Autocomplete Contract Mismatch

**Bug:** Portal's `akFetchAutocomplete()` in `allcodex-portal/lib/allknower-server.ts` reads `data.results` but AllKnower's `GET /suggest/autocomplete` returns `{ suggestions: [...] }`. Autocomplete silently returns empty array.

**Fix (Portal-side, no AllKnower deploy):**

**Modify:** `allcodex-portal/lib/allknower-server.ts`
- Change `data.results` → `data.suggestions` in `akFetchAutocomplete()`

**Files:** 1 modified

#### Step 19: Brain Dump History Detail Endpoint

AllKnower has `GET /brain-dump/history` (last 20) but **no single-entry endpoint** `GET /brain-dump/history/:id`.

**Modify:** `allknower/src/routes/brain-dump.ts`
- Add `GET /brain-dump/history/:id` route
- Query `prisma.brainDumpHistory.findUnique({ where: { id } })`
- Return full entry: `id`, `rawText`, `parsedJson` (entities + summary), `notesCreated: string[]`, `notesUpdated: string[]`, `model`, `tokensUsed`, `createdAt`
- Return 404 if not found
- Use Elysia `t` schema for validation

**Files:** 1 modified

#### Step 20: Brain Dump Response Shape Normalization

The `POST /brain-dump` route returns the raw `BrainDumpResult` from the pipeline (`summary`, `created`, `updated`, `skipped`, `relations`). The history endpoint returns `notesCreated: string[]`, `notesUpdated: string[]`. These two shapes don't match.

**Modify:** `allknower/src/routes/brain-dump.ts`
- Normalize the POST response to also include `notesCreated: string[]` and `notesUpdated: string[]` alongside `created`/`updated` arrays
- Or document that `POST` returns rich objects and `GET /history` returns ID arrays — either is fine, Portal just needs to know

**Modify:** `allknower/src/types/lore.ts`
- Ensure `BrainDumpResult` schema includes all fields the Portal expects
- Add `BrainDumpHistoryEntry` schema for history detail responses

**Files:** 2 modified

#### Step 21: AllKnower Contract Tests

Add lightweight contract tests for Portal-facing routes to prevent future drift.

**New file:** `allknower/test/integration/portal-contracts.test.ts`
- Test `POST /brain-dump` response matches `BrainDumpResult` schema
- Test `GET /brain-dump/history` returns array with correct fields
- Test `GET /brain-dump/history/:id` returns full entry
- Test `GET /suggest/autocomplete?q=test` returns `{ suggestions: [...] }`
- Test `GET /health` returns expected shape
- Use `bun:test` with mock.module for ETAPI/LLM dependencies

**Files:** 1 new

#### Subphase E1 Verification
1. Portal autocomplete works end-to-end (no more silent empty results)
2. `GET /brain-dump/history/:id` returns full parsedJson, notesCreated arrays, and summary
3. Contract tests pass for all Portal-facing routes
4. Brain dump detail page (Phase A step 4) works with the new endpoint

---

### Subphase E2: AllCodex Core API + Template Improvements

*Runs alongside Phase B and Phase D. These are incremental improvements to existing ETAPI endpoints.*

#### Step 22: Note Breadcrumb / Ancestor API

The Portal needs breadcrumbs (Phase B step 6). ETAPI has `GET /etapi/branches/:branchId` but no way to get a note's full ancestor chain in one call. Portal would need N+1 requests to walk up the tree.

**Option A (recommended): Portal-side traversal** — fetch note → get branches → walk parentNoteId chain. AllCodex Core branches table has `noteId` + `parentNoteId`. Portal can loop via existing ETAPI endpoints.

**Option B: Core convenience endpoint** — add `GET /etapi/notes/:noteId/ancestors` that returns `[{ noteId, title, branchId }]` from leaf to root.

Recommendation: Start with Option A (no Core changes needed). If it's too slow, add Option B later.

**If Option A:**
**Modify:** `allcodex-portal/lib/etapi-server.ts`
- Add `getNoteAncestors(creds, noteId): Promise<Array<{ noteId: string; title: string }>>` — loops via `GET /etapi/notes/:noteId` reading parentNoteIds from branches

**If Option B:**
**Modify:** `allcodex-core/apps/server/src/etapi/notes.ts`
- Add `GET /etapi/notes/:noteId/ancestors` route

**Files:** 1 modified (either Portal or Core)

#### Step 23: Backlinks Search Pattern

The Portal needs incoming relations (Phase B step 7). ETAPI has search: `GET /etapi/notes?search=...`. AllCodex search supports `~relationName=noteId` syntax via Becca.

**Modify:** `allcodex-portal/lib/etapi-server.ts`
- Add `getNoteBacklinks(creds, noteId): Promise<Array<{ noteId: string; title: string; relationType: string }>>` — search for `~*=noteId` or iterate known relation types

**Files:** 1 modified

#### Step 24: Template Formalization for Session/Quest/Timeline Notes

AllCodex Core already has 12+ lore templates in `hidden_subtree_templates.ts`. AllKnower's `TEMPLATE_ID_MAP` references 18 types. Phase D (session runtime) needs session, quest, and scene note types.

**Check if these exist already:**
- `_template_event` ✅ (exists — inWorldDate, outcome, consequences, secrets)
- `_template_timeline` ✅ (exists — calendarSystem)
- Templates for session, quest, scene → **do NOT exist**

**Modify:** `allcodex-core/apps/server/src/services/hidden_subtree_templates.ts`
- Add `_template_session` — sessionDate, players, recap, hooks
- Add `_template_quest` — questStatus (active/completed/failed/deferred), questGiver, reward, hooks, consequences
- Add `_template_scene` — sessionId, location, participants, outcome

**Modify:** `allknower/src/types/lore.ts`
- Add `session`, `quest`, `scene` to `LoreEntityType` enum
- Add to `TEMPLATE_ID_MAP`: `session: "_template_session"`, `quest: "_template_quest"`, `scene: "_template_scene"`

**Files:** 2 modified (1 Core, 1 AllKnower)

#### Step 25: Document Canonical Labels and Attributes

Currently the lore label conventions are scattered across AllKnower's `TEMPLATE_ID_MAP`, AllCodex Core's template definitions, and Portal's `TemplatePicker`. There's no single reference.

**New file:** `docs/shared/canonical-lore-schema.md`
- Document all lore type keys (character, location, faction, etc.)
- Document promoted attribute keys per type (aligned with Core templates)
- Document canonical labels: `#lore`, `#loreType`, `#draft`, `#gmOnly`, `#quest`, `#session`, `#timeline`
- Document relation types used by AllKnower's auto-relate pipeline
- Document provenance labels: `#brainDumpId`, `#importSource`

**Files:** 1 new

#### Subphase E2 Verification
1. Portal breadcrumb API returns correct ancestor chain
2. Portal backlinks search returns notes with relations to current note
3. AllCodex Core has session, quest, scene templates (check via ETAPI `GET /etapi/notes/_template_session`)
4. AllKnower TEMPLATE_ID_MAP includes all 21 types (18 existing + 3 new)
5. `canonical-lore-schema.md` documents all label/attribute conventions

---

## Phase F: Rules Reference + System Content (Roadmap Phase 5)

*Depends on Phase D (session workspace) and Phase E (template formalization).*

Reference Stitch screen: `5941eae4cecb414b8493a18f9fd8e18d` ("Statblock Library & Viewer")

### Step 26: Statblock Template Enhancement

AllCodex Core already has `_template_statblock` with basic D&D-style fields (crName, crLevel, ac, hp, ability scores, abilities). Enhance for browsing.

**Modify:** `allcodex-core/apps/server/src/services/hidden_subtree_templates.ts`
- Add promoted attributes to statblock: `challengeRating`, `type` (beast/humanoid/undead...), `size`, `alignment`, `speed`, `immunities`, `resistances`, `vulnerabilities`, `actions`, `legendaryActions`
- Ensure statblock template has `#statblock` label for easy search

**Files:** 1 modified

#### Step 27: Statblock Library Page

**New file:** `allcodex-portal/app/(portal)/statblocks/page.tsx`
- Search/filter statblocks by name, CR, type
- Card grid view: name, CR badge, type badge, HP, AC at-a-glance
- Click → detail view (same lore detail page, but statblock has its own infobox layout)
- Reference Stitch screen `5941eae4cecb414b8493a18f9fd8e18d`

**New file:** `allcodex-portal/app/api/statblocks/route.ts`
- `GET` — ETAPI search for notes with `#statblock` label
- Return with promoted attributes for card rendering

**Modify:** `allcodex-portal/components/portal/AppSidebar.tsx`
- Add "Statblocks" to Compendium/Reference group

**Files:** 2 new, 1 modified

#### Step 28: Statblock Viewer Component

**New file:** `allcodex-portal/components/portal/StatblockCard.tsx`
- Renders a single statblock in the classic two-column D&D stat block format
- Shows: name, size/type/alignment, AC, HP, speed, ability scores grid, features, actions
- Reusable from both the statblock library and the session workspace (Phase D)

**Files:** 1 new

#### Step 29: Session Workspace Statblock Search

*Depends on steps 14 (session workspace) + 27-28 (statblock library)*

**Modify:** `allcodex-portal/app/(portal)/session/page.tsx`
- Add "Quick Statblock" panel or command palette action
- Search statblocks by name from within session view
- Display `StatblockCard` inline without leaving session

**Files:** 1 modified

#### Step 30: Rules-Aware RAG Retrieval

**Modify:** `allknower/src/rag/lancedb.ts` (or new module)
- When querying RAG for NPC generation or session context, include statblock and rules notes in the retrieval pool
- Add tag filter: `queryLore(text, limit, { includeTags: ["statblock", "rules"] })`

**Modify:** `allknower/src/pipeline/brain-dump.ts`
- When brain dump mentions an NPC or creature, ground the generated content against matching statblocks if available
- Use RAG context to suggest "This NPC matches the statblock for X" in the response

**Files:** 2 modified

#### Step 31: System-Pack Import (Lightweight)

Allow importing SRD-compatible JSON packs of monsters, spells, items as statblock notes.

**New file:** `allknower/src/routes/import.ts`
- `POST /import/system-pack` — accepts JSON array of statblock objects
- Creates notes via ETAPI with `#statblock` template
- Sets promoted attributes from JSON fields
- Returns `{ created, skipped, errors }`

**New file:** `allcodex-portal/app/api/import/system-pack/route.ts`
- Proxy to AllKnower

**New file:** `allcodex-portal/app/(portal)/import/page.tsx`
- Import UI: upload JSON, preview entities, select/deselect, import
- Shared with future Azgaar import (same page, different tab)

**Modify:** `allcodex-portal/components/portal/AppSidebar.tsx`
- Add "Import" to sidebar (if not already added for Azgaar)

**Files:** 3 new, 1 modified

### Phase F Verification
1. Statblock library shows all `#statblock` notes with CR/type/HP badges
2. Click statblock → renders classic D&D stat block format
3. Session workspace → quick statblock search returns matching statblocks inline
4. System-pack import creates statblock notes with correct template and attributes
5. Brain dump mentioning an NPC references matching statblocks when available

---

## Phase G: Player-Safe Sharing (Roadmap Phase 4 — partial)

*Depends on Phase B (wiki article view) and Phase E2 step 25 (canonical labels).*
*Scheduling: parallel with Phase D/F once B completes.*

**What already exists in AllCodex Core (no changes needed):**
- Full share system via Shaca (read-only cache of `_share` subtree)
- `#gmOnly` label → entire note hidden from shared output
- `#draft` label → entire note hidden from shared output
- `.gm-only` CSS class → elements stripped from rendered HTML
- `#shareRoot` / `#shareAlias` / `#shareCredentials` labels for share configuration
- `#shareHiddenFromTree` / `#shareIndex` / `#shareDisallowRobotIndexing` labels
- Share routes at `/share/` with Basic Auth, alias resolution, recursive rendering
- World variable expansion (`{{placeholder}}` → JSON values)

### Step 32: Share Settings Panel on Lore Detail
*Depends on Phase B step 5 (infobox redesign) — P1*

**New file:** `allcodex-portal/components/portal/ShareSettings.tsx`
- Props: `noteId`, `attributes` (existing note labels)
- Shows current share state: shared / not shared / draft / GM-only
- Toggle buttons: `#draft`, `#gmOnly` labels (via existing attribute POST/DELETE API)
- Computed share URL display: `/share/{shareAlias || noteId}`

**Modify:** `allcodex-portal/app/(portal)/lore/[id]/page.tsx`
- Add "Sharing" section in the sidebar/infobox with the ShareSettings component

**Files:** 1 new, 1 modified

### Step 33: Share Root Configuration
*Parallel with step 32 — P1*

**Modify:** `allcodex-portal/app/(portal)/settings/page.tsx`
- Add "Share Configuration" card (alongside Portal Configuration from step 1)
- Field: **Share Root Note ID** — the note under which all shared content lives
- Info text: notes under this root with `#shareRoot` become publicly accessible at `/share/`
- Set `#shareRoot` label on the designated note via ETAPI

**New file:** `allcodex-portal/app/api/share/route.ts`
- `GET` — fetch share root info: find note with `#shareRoot` label, return its ID, title, alias
- `PUT` — set `#shareRoot` label on a note (add label via ETAPI `POST /etapi/attributes`)

**Files:** 1 new, 1 modified

### Step 34: GM Preview vs Player Preview
*Depends on step 32 — P1*

**New file:** `allcodex-portal/components/portal/PreviewToggle.tsx`
- Toggle on lore detail page: "GM View" | "Player View"
- GM View (default): shows everything including `#gmOnly` and `#draft` content
- Player View: fetches content from AllCodex Core's `/share/api/notes/:noteId` endpoint — returns already-filtered content with gmOnly/draft stripped

**New file:** `allcodex-portal/app/api/lore/[id]/preview/route.ts`
- `GET ?mode=player` — proxy to AllCodex Core `/share/api/notes/:noteId`
- `GET ?mode=gm` (or no param) — proxy to ETAPI as usual

**Modify:** `allcodex-portal/app/(portal)/lore/[id]/page.tsx`
- Add `PreviewToggle` component
- When "Player View" active, swap content source to preview API
- Visually indicate player mode (different border color, "Player View" badge)

**Files:** 2 new, 1 modified

### Step 35: Share Alias Management
*Parallel with steps 32-34 — P2*

**Modify:** `allcodex-portal/components/portal/ShareSettings.tsx`
- Add "Custom URL" field for `#shareAlias` label
- Input: slug field, e.g. "blackstone-keep" → accessible at `/share/blackstone-keep`
- Validation: alphanumeric + hyphens only
- Save: add/update `#shareAlias` attribute via ETAPI

**Files:** 1 modified

### Step 36: Share Credentials (Basic Auth)
*Parallel with steps 32-35 — P2*

**Modify:** `allcodex-portal/components/portal/ShareSettings.tsx`
- Add "Password Protection" section
- Toggle: enable/disable password
- When enabled: username + password fields → stored as `#shareCredentials` label value `"user:password"`
- Warning: "Credentials are stored as a note label. Use a unique password."

**Files:** 1 modified

### Step 37: Shared Content Browser
*Depends on steps 32-33 — P2*

**New file:** `allcodex-portal/app/(portal)/shared/page.tsx`
- Browse all notes under the share root
- Tree or list view showing: title, share status, alias, draft/gmOnly badges
- Quick toggle draft/gmOnly from the list
- Link to each note's detail page and its public share URL

**New file:** `allcodex-portal/app/api/share/tree/route.ts`
- `GET` — fetch all notes under share root via ETAPI search for children of `_share`
- Return with share-relevant attributes (shareAlias, gmOnly, draft, shareCredentials presence)

**Modify:** `allcodex-portal/components/portal/AppSidebar.tsx`
- Add "Shared" entry to sidebar navigation

**Files:** 2 new, 1 modified

### Phase G Verification
1. Lore detail page shows share status badges + draft/gmOnly toggles
2. Toggling `#gmOnly` on a note → note disappears from `/share/` output
3. Toggling `#draft` → same behavior
4. "Player View" toggle → shows content with gmOnly sections stripped
5. Set share alias → note accessible at `/share/custom-alias`
6. Set credentials → share page requires Basic Auth
7. Shared content browser lists all shared notes with status
8. Share root configurable from Settings

---

## Deferred

- **Azgaar FMG Import** (Phase 4) — deferred to last per user request
- **Full map view**
- **Homebrew statblock editing** (after reader stabilizes)

---

## Key Decisions

1. **TemplatePicker + PromotedFields already exist** — no `lib/lore-fields.ts` needed; 12 templates in `TemplatePicker.tsx` are the canonical source
2. **Attribute API already has POST + DELETE** — edit page uses delete+create pattern, no PATCH needed
3. **Brain Dump modes (Phase C) require AllKnower endpoint changes** — cross-service work
4. **Azgaar import deferred** — per user request, including the Stitch screen for it
5. **Stitch designs are layout/UX reference** — implementation uses shadcn/ui + Arcane Slate palette
6. **Autocomplete fix is Portal-only** — change `data.results` → `data.suggestions`, no AllKnower deploy needed
7. **Breadcrumbs use Portal-side traversal first** — avoid Core changes unless too slow
8. **Session/quest/scene templates added to Core** — AllKnower's TEMPLATE_ID_MAP updated to match
9. **Canonical lore schema doc** — single source of truth for all label/attribute conventions across 3 services
10. **Player-safe sharing is Portal-only** — AllCodex Core share system already complete (Shaca cache, `#gmOnly`/`#draft` filtering, `.gm-only` HTML stripping, Basic Auth, aliases). Player View proxies Core's share routes rather than re-implementing filtering in the Portal

---

## File Impact Summary

### Phase A (Foundation)
| Type | Files |
|---|---|
| New | `app/api/config/portal/route.ts`, `lib/sanitize.ts`, `app/api/brain-dump/history/[id]/route.ts`, `app/(portal)/brain-dump/[id]/page.tsx` |
| Modified | `lib/get-creds.ts`, `app/api/lore/route.ts`, `app/(portal)/settings/page.tsx`, `app/(portal)/lore/new/page.tsx`, `app/(portal)/lore/[id]/page.tsx`, `app/(portal)/lore/[id]/edit/page.tsx`, `app/globals.css`, `lib/allknower-server.ts`, `app/(portal)/brain-dump/page.tsx` |
| Dependencies | `isomorphic-dompurify` |

### Phase B (Wiki View)
| Type | Files |
|---|---|
| New | `components/portal/Breadcrumbs.tsx`, `app/api/lore/[id]/breadcrumbs/route.ts`, `components/portal/TableOfContents.tsx`, `app/api/lore/[id]/backlinks/route.ts`, `components/portal/NotePreview.tsx` |
| Modified | `app/(portal)/lore/[id]/page.tsx`, `components/portal/LoreTree.tsx`, `app/(portal)/lore/page.tsx` |

### Phase C (Brain Dump Enhancement)
| Type | Files |
|---|---|
| Modified (Portal) | `app/(portal)/brain-dump/page.tsx` |
| Modified (AllKnower) | `src/routes/brain-dump.ts`, `src/pipeline/brain-dump.ts` |

### Phase D (Session Runtime)
| Type | Files |
|---|---|
| New | `app/(portal)/session/page.tsx`, `app/(portal)/session/layout.tsx`, `app/(portal)/quests/page.tsx`, `app/api/quests/route.ts`, `app/(portal)/timeline/page.tsx`, `app/api/timeline/route.ts` |
| Modified | `components/portal/AppSidebar.tsx` |

### Phase E (Contracts + Core APIs)
| Type | Files |
|---|---|
| New | `allknower/test/integration/portal-contracts.test.ts`, `docs/shared/canonical-lore-schema.md` |
| Modified (Portal) | `allcodex-portal/lib/allknower-server.ts`, `allcodex-portal/lib/etapi-server.ts` |
| Modified (AllKnower) | `allknower/src/routes/brain-dump.ts`, `allknower/src/types/lore.ts` |
| Modified (Core) | `allcodex-core/apps/server/src/services/hidden_subtree_templates.ts` |

### Phase F (Rules + Statblocks)
| Type | Files |
|---|---|
| New | `allcodex-portal/app/(portal)/statblocks/page.tsx`, `allcodex-portal/app/api/statblocks/route.ts`, `allcodex-portal/components/portal/StatblockCard.tsx`, `allknower/src/routes/import.ts`, `allcodex-portal/app/api/import/system-pack/route.ts`, `allcodex-portal/app/(portal)/import/page.tsx` |
| Modified | `allcodex-portal/components/portal/AppSidebar.tsx`, `allcodex-portal/app/(portal)/session/page.tsx`, `allcodex-core/apps/server/src/services/hidden_subtree_templates.ts`, `allknower/src/rag/lancedb.ts`, `allknower/src/pipeline/brain-dump.ts` |

### Phase G (Player-Safe Sharing)
| Type | Files |
|---|---|
| New | `allcodex-portal/components/portal/ShareSettings.tsx`, `allcodex-portal/components/portal/PreviewToggle.tsx`, `allcodex-portal/app/api/share/route.ts`, `allcodex-portal/app/api/share/tree/route.ts`, `allcodex-portal/app/api/lore/[id]/preview/route.ts`, `allcodex-portal/app/(portal)/shared/page.tsx` |
| Modified | `allcodex-portal/app/(portal)/lore/[id]/page.tsx`, `allcodex-portal/app/(portal)/settings/page.tsx`, `allcodex-portal/components/portal/AppSidebar.tsx` |
| Core | **None** — share system already complete |
