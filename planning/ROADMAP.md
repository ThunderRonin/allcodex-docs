# AllCodex Roadmap

Date: March 31, 2026 · Updated: April 27, 2026

> **Single source of truth** for all AllCodex roadmap work across AllCodex Core, AllKnower, and AllCodex Portal.
> When roadmap priorities conflict with component-level backlogs, this document wins.

See also:

- `docs/shared/planning/allcodex_dm_first_scope.md` — product identity and scope boundary
- `docs/shared/analysis/worldanvil_feature_matrix.md` — implementation-verified World Anvil parity audit

---

## Product Objective

AllCodex should become the best AI-integrated DM notebook for:

- capturing ideas before setup friction kills them
- improvising during play without silently creating contradictions
- retrieving lore quickly in a readable wiki-like format
- letting AI handle the boring organization work
- presenting selected canon cleanly to players without turning the product into a social publishing platform

The roadmap favors:

- Brain Dump as the front door
- wiki readability and navigation over vanity presentation
- continuity and contradiction prevention over breadth
- player-safe read views over collaboration features
- exposing existing Core and AllKnower power before inventing large new systems

---

## Not In Near-Term Scope

Explicitly de-prioritized unless the product direction changes:

- advanced fantasy calendars
- novelist and manuscript-first workflows
- public community and social features
- monetization and patron gating
- heavy multi-user collaboration
- full VTT integration
- full map creation tooling

Simple timeline support is in scope. Huge calendar engines are not.
Lightweight map embeds and pins are in scope. A full map editor is not.

---

## Success Criteria

1. Brain Dump can be trusted as the fastest capture path, not just a demo feature.
2. The default lore view feels like a real wiki article instead of a raw note renderer.
3. A GM can run a live session without constantly leaving the product.
4. Improvised canon is checked against existing lore before it quietly causes continuity problems.
5. Players can read selected lore safely without exposing GM-only information.

---

## Delivery Order

| Phase | Focus | Priority | Status |
|---|---|---|---|
| Phase 0 | Foundation and contract repair | `P0` | ✅ Complete |
| Phase 1 | Brain Dump becomes the front door | `P1` | ✅ Complete |
| Phase 2 | Wiki article view and knowledge architecture | `P1` | ✅ Complete |
| Phase 3 | Session runtime and continuity guardrails | `P1→P2` | ✅ Complete |
| Phase 4 | Player-safe sharing, maps, and import | `P2` | ✅ Complete |
| Phase 5 | Rules reference and system-content expansion | `P3` | ⚠️ Partial (statblocks shipped, rules ref TBD) |

---

## Cross-Service Foundation

Before feature work spreads, the three services need shared assumptions pinned down.

### Canonical JSON contracts

Define and freeze shapes for:

- Brain Dump create response
- Brain Dump history list response
- Brain Dump history detail response
- autocomplete suggestions
- promoted attribute keys per lore type
- share-state metadata exposed to the Portal

Goal: stop Portal and AllKnower from drifting on response shapes.

### Shared product conventions

Define and document:

- lore type keys
- promoted attribute keys
- session, quest, timeline, and statblock note types
- GM-only and draft visibility behavior
- geolocation and map-pin metadata
- provenance metadata for notes created from Brain Dump or imports

### Brain Dump history detail contract

```json
{
  "id": "dump_123",
  "rawText": "The party reached Blackstone...",
  "summary": "The party reached Blackstone and met...",
  "entities": [
    {
      "title": "Blackstone",
      "loreType": "location",
      "action": "created",
      "noteId": "abc123",
      "confidence": 0.92,
      "warnings": ["Possible overlap with Blackstone Keep"]
    }
  ],
  "notesCreated": ["abc123"],
  "notesUpdated": [],
  "tokensUsed": 1422,
  "model": "...",
  "createdAt": "2026-03-31T12:34:56Z"
}
```

---

## Phase 0: Foundation And Contract Repair ✅

> **Shipped.** Implementation plan: Phases A + E1. See `docs/shared/planning/implementation_plan_phases_0_3.md`.

**Outcome:** the existing product stops feeling inconsistent, fragile, or misleading before larger DM workflows are added.

### Portal

- add lore root note configuration in Settings and use it in lore creation
- ship promoted attributes on create and edit flows
- bring edit-page parity up to the level of create flow
- sanitize rendered lore HTML and improve article typography baseline
- make Brain Dump history entries clickable and viewable in detail
- normalize Portal-side types around canonical Brain Dump and autocomplete contracts

### AllKnower

- normalize `POST /brain-dump` response shape
- normalize `GET /brain-dump/history` list shape
- add or normalize `GET /brain-dump/history/:id` detail shape
- normalize autocomplete response shape so Portal fallback works reliably
- expose canonical promoted attribute keys in template or schema outputs
- add lightweight contract tests for Portal-facing routes

### AllCodex Core

- expose any missing ETAPI support needed for promoted attribute patching and note metadata reads
- document or formalize canonical labels and attributes used by Portal and AllKnower
- expose share-state metadata cleanly if current APIs make Portal share controls awkward

### Exit criteria

- Brain Dump create, list, and detail views all use one documented contract
- autocomplete no longer silently fails because of shape mismatches
- promoted attribute keys are canonical and consistent
- new lore no longer falls into the wrong root by default

---

## Phase 0 Implementation Detail (Portal)

These are the concrete Portal tasks for Phase 0, with exact file locations.

### Feature 1: Lore Root Note Configuration

**Problem:** `parentNoteId` defaults to `"root"` — everything created via the portal lands in the AllCodex root.

**Files changed:** 4 modified, 1 new

#### 1a. Settings — Portal Configuration card

**File:** `app/(portal)/settings/page.tsx`

Add a third card below the AllCodex/AllKnower connection cards:

- Title: "Portal Configuration"
- Field: **Lore Root Note ID** — text input, placeholder `"root"`, description `"The AllCodex note ID where new lore entries will be created."`
- Save button stores value as `lore_root_note_id` cookie

#### 1b. Portal config API route

**New file:** `app/api/config/portal/route.ts`

- `GET` — reads `lore_root_note_id` cookie, returns JSON
- `PUT` — sets `lore_root_note_id` cookie with same `COOKIE_OPTS` as connect route

#### 1c. Lore root helper

**File:** `lib/get-creds.ts`

```ts
export async function getLoreRootNoteId(): Promise<string>
// reads cookie lore_root_note_id → env LORE_ROOT_NOTE_ID → fallback "root"
```

#### 1d. Use config in lore creation

**File:** `app/api/lore/route.ts`

Replace hardcoded `"root"` with `await getLoreRootNoteId()`.

#### 1e. Pre-populate parent field in New Lore form

**File:** `app/(portal)/lore/new/page.tsx`

Fetch `/api/config/portal` on mount, pre-populate Parent Note ID. User can still override per-entry.

---

### Feature 2: Promoted Attributes (Create & Edit)

**Problem:** Template-defined promoted attributes (fullName, race, age, affiliation, etc.) are never surfaced on create or edit forms.

**Files changed:** 3 modified, 3 new

#### 2a. Promoted field schema

**New file:** `lib/lore-fields.ts`

```ts
export interface LoreField {
  name: string;       // AllCodex attribute key, e.g. "fullName"
  label: string;      // display label
  type: "text" | "number" | "textarea";
  placeholder?: string;
}

export const LORE_TYPE_FIELDS: Record<string, LoreField[]> = {
  character: [
    { name: "fullName",    label: "Full Name",    type: "text" },
    { name: "aliases",     label: "Aliases",      type: "text", placeholder: "Comma-separated" },
    { name: "age",         label: "Age",          type: "number" },
    { name: "race",        label: "Race",         type: "text" },
    { name: "gender",      label: "Gender",       type: "text" },
    { name: "affiliation", label: "Affiliation",  type: "text" },
    { name: "role",        label: "Role",         type: "text" },
    { name: "status",      label: "Status",       type: "text", placeholder: "alive, dead, unknown" },
  ],
  location: [
    { name: "locationType", label: "Type",       type: "text", placeholder: "City, fortress, forest…" },
    { name: "region",       label: "Region",     type: "text" },
    { name: "population",   label: "Population", type: "number" },
    { name: "ruler",        label: "Ruler",      type: "text" },
  ],
  faction: [
    { name: "factionType",  label: "Type",        type: "text", placeholder: "Guild, kingdom, order…" },
    { name: "leader",       label: "Leader",      type: "text" },
    { name: "foundingDate", label: "Founded",     type: "text" },
    { name: "goals",        label: "Goals",       type: "textarea" },
  ],
  creature: [
    { name: "creatureType", label: "Type",         type: "text", placeholder: "Beast, undead, dragon…" },
    { name: "habitat",      label: "Habitat",      type: "text" },
    { name: "diet",         label: "Diet",         type: "text" },
    { name: "dangerLevel",  label: "Danger Level", type: "number" },
    { name: "abilities",    label: "Abilities",    type: "textarea" },
  ],
  event: [
    { name: "inWorldDate",    label: "In-World Date", type: "text" },
    { name: "outcome",        label: "Outcome",       type: "text" },
    { name: "consequences",   label: "Consequences",  type: "textarea" },
  ],
  item: [
    { name: "itemType",   label: "Type",          type: "text", placeholder: "Weapon, artifact, potion…" },
    { name: "rarity",     label: "Rarity",        type: "text" },
    { name: "owner",      label: "Current Owner", type: "text" },
    { name: "properties", label: "Properties",    type: "textarea" },
  ],
};
```

#### 2b. PromotedFields component

**New file:** `components/portal/PromotedFields.tsx`

Props: `loreType`, `values`, `onChange`, `disabled?`

- looks up `LORE_TYPE_FIELDS[loreType]`
- renders 2-column grid of inputs/textareas
- returns nothing if type has no fields

#### 2c. Wire into New Lore page

**File:** `app/(portal)/lore/new/page.tsx`

- add `const [attributes, setAttributes] = useState<Record<string, string>>({})`
- render `<PromotedFields>` below type selector, reset on type change
- pass `attributes` in POST body

#### 2d. Wire into Edit Lore page

**File:** `app/(portal)/lore/[id]/edit/page.tsx`

- fetch note's existing attributes on load, pre-populate `<PromotedFields>`
- on save, PUT to `/api/lore/[id]/attributes`

#### 2e. API updates

**File:** `app/api/lore/route.ts` — destructure `attributes` from body, call `createAttribute()` for each after note creation.

**New file:** `app/api/lore/[id]/attributes/route.ts` — `PUT` accepts `{ attributes: Record<string, string> }`, diffs and creates/updates via ETAPI.

**File:** `lib/etapi-server.ts` — add `patchAttribute(creds, attrId, { value })`.

---

### Feature 3: Edit Page Parity

**Problem:** Edit page has bare title + raw HTML textarea. Should match create: lore type, promoted attributes, preview toggle.

**Files changed:** 1 heavily modified, depends on Feature 2

#### 3a. Lore type selector on edit

- read `loreType` attribute on load
- show `<Select>` matching create page
- on save, update `loreType` label attribute

#### 3b. Promoted fields on edit

- import `<PromotedFields>` from Feature 2
- pre-populate from existing note attributes
- on save, PUT to `/api/lore/[id]/attributes`

#### 3c. Edit / Preview tabs

- "Edit" tab: existing textarea
- "Preview" tab: `<div className="lore-content" dangerouslySetInnerHTML={{ __html: content }} />`
- same CSS as detail page guarantees WYSIWYG appearance

---

### Feature 4: Rendered Lore Content View

**Problem:** `dangerouslySetInnerHTML` with no sanitization; basic typography.

**Files changed:** 2 modified, 1 dependency

#### 4a. HTML sanitization

**File:** `app/(portal)/lore/[id]/page.tsx`

Install `isomorphic-dompurify`. Sanitize before rendering — allow standard HTML elements, classes, and inline styles; strip scripts, event handlers, iframes.

#### 4b. Enhanced `.lore-content` CSS

**File:** `app/globals.css`

Add styles for: responsive images, grimoire-style `<hr>`, code/pre blocks, figure/figcaption captions, nested list indentation, CKEditor class compatibility (`.ck-content`, `.todo-list`).

#### 4c. Table of Contents (optional)

Parse rendered HTML for `h1`/`h2`/`h3`, generate a small floating ToC. Show only when content exceeds height threshold.

---

### Feature 5: Brain Dump History Detail

**Problem:** History entries are not clickable. `notesCreated`/`notesUpdated` are typed as `number` in the portal but stored as `string[]` in AllKnower — contract mismatch.

**Files changed:** 2 modified, 2–3 new

#### Root cause

`BrainDumpHistory` Prisma model stores `notesCreated: String[]`, `notesUpdated: String[]`, `parsedJson: Json`. The portal interface types them as `number`. Investigate whether AllKnower's history route is transforming arrays to counts, or whether the portal is silently mistyping arrays.

#### 5a. AllKnower history detail endpoint

If `GET /brain-dump/history/:id` does not exist, add it. Response shape must match the canonical contract in the Cross-Service Foundation section above.

#### 5b. Portal API route

**New file:** `app/api/brain-dump/history/[id]/route.ts`

`GET` — proxies to AllKnower's `GET /brain-dump/history/:id`.

#### 5c. AllKnower client function

**File:** `lib/allknower-server.ts`

```ts
export async function getBrainDumpEntry(creds, id): Promise<BrainDumpDetailEntry>
// BrainDumpDetailEntry uses notesCreated: string[], notesUpdated: string[], parsedJson
```

#### 5d. History detail page

**New file:** `app/(portal)/brain-dump/[id]/page.tsx`

- header: date, model, token count
- raw text in styled blockquote (full, not truncated)
- AI summary block
- entity cards grid: title, type, action taken, link to note
- back button

#### 5e. Make history entries clickable

**File:** `app/(portal)/brain-dump/page.tsx`

Wrap each history entry in `<Link href={/brain-dump/${entry.id}}>`, add hover affordance.

---

### Feature 6: Azgaar FMG Import ✅ SHIPPED

> **Status:** Fully implemented. AllKnower pipeline (`src/pipeline/azgaar.ts`) parses Azgaar `.map` JSON, creates location/faction/religion/race notes via ETAPI, supports duplicate-skip and preview mode. Portal parses uploaded `.map`/JSON files client-side and posts JSON to `/api/import/azgaar`; `?action=preview` forwards to the AllKnower preview route. Import page UI at `/import` supports drag-and-drop, preview, import options, and result reporting.

**Problem:** No bulk import exists for world geography. Scope: Portal triggers, AllKnower processes.

**Files changed:** 2 modified, 2–3 new (Portal) + AllKnower backend

#### Azgaar JSON structure

Relevant fields from `.map` export:

- `pack.burgs[]` — settlements with name, coordinates, state, capital, port, population, type
- `pack.states[]` — kingdoms/nations
- `pack.religions[]` — religions
- `pack.cultures[]` — cultures/species groups
- `notes[]` — map notes / points of interest

#### 6a. AllKnower import endpoint

**AllKnower:** `POST /import/azgaar`

- validate Azgaar JSON with `pack` entity arrays
- create notes via ETAPI with templates and labels: states → factions, burgs → locations, religions → religions, cultures → races, map notes → locations
- set promoted attributes plus `#importSource=azgaar` and optional `#geolocation`
- skip duplicates by title when enabled
- return `{ mapName, totals, states, burgs, religions, cultures, notes }` with per-entity created/skipped/error buckets

#### 6b. Portal API proxy

**New file:** `app/api/import/azgaar/route.ts`

`POST` — accepts JSON `{ mapData, parentNoteId?, options? }`, forwards to AllKnower, and returns the preview or import result. `?action=preview` forwards to `POST /import/azgaar/preview`.

#### 6c. Import UI page

**New file:** `app/(portal)/import/page.tsx`

- drag-and-drop zone for `.map`/JSON file
- client-side JSON parsing and automatic preview
- checkboxes to select entity types (states, burgs, religions, cultures, map notes) and duplicate-skip behavior
- "Import" button, progress display, result summary with created/skipped/error counts

#### 6d. Sidebar nav

**File:** `components/portal/AppSidebar.tsx`

Add to Studio group: `{ href: "/import", icon: Upload, label: "Import" }`

#### 6e. AllKnower proxy behavior

The Portal route calls AllKnower directly from `app/api/import/azgaar/route.ts`; there is no dedicated `lib/allknower-server.ts` helper for this path.

---

## Phase 0 Effort Estimates

| Feature | Complexity | Effort | Priority |
|---|---|---|---|
| 1. Lore Root Note Config | Low | 2–3h | `P0` |
| 2. Promoted Attributes (Create/Edit) | Medium | 4–6h | `P1` |
| 3. Edit Page Parity | Medium | 4–6h | `P1` — ship with Feature 2 |
| 4. Rendered Content View + sanitization | Low | 3–4h | `P2` |
| 5. Brain Dump History Detail | Medium | 4–5h + AllKnower fix | `P2` |
| 6. Azgaar FMG Import | High | 10–14h | `P3` — AllKnower backend required first |

**Recommended order:** 1 → 2+3 together → 4 → 5 → 6

---

## Phase 0 File Map

### New files

| File | Feature | Purpose |
|---|---|---|
| `lib/lore-fields.ts` | 2 | Promoted field definitions per lore type |
| `components/portal/PromotedFields.tsx` | 2 | Reusable promoted fields form component |
| `app/api/config/portal/route.ts` | 1 | GET/PUT portal-level config (lore root) |
| `app/api/lore/[id]/attributes/route.ts` | 2 | PUT promoted attributes on existing notes |
| `app/api/brain-dump/history/[id]/route.ts` | 5 | Proxy to AllKnower history detail |
| `app/(portal)/brain-dump/[id]/page.tsx` | 5 | Brain dump history detail page |
| `app/api/import/azgaar/route.ts` | 6 | Proxy Azgaar map data to AllKnower |
| `app/(portal)/import/page.tsx` | 6 | Azgaar import UI |

### Modified files

| File | Features | Changes |
|---|---|---|
| `app/(portal)/settings/page.tsx` | 1 | Add "Portal Configuration" card |
| `app/(portal)/lore/new/page.tsx` | 1, 2 | Pre-populate parent ID, add promoted fields |
| `app/(portal)/lore/[id]/edit/page.tsx` | 2, 3 | Add lore type, promoted fields, content preview |
| `app/(portal)/lore/[id]/page.tsx` | 4 | Sanitize HTML |
| `app/(portal)/brain-dump/page.tsx` | 5 | Make history entries clickable |
| `app/api/lore/route.ts` | 1, 2 | Use configured lore root, save attributes |
| `app/globals.css` | 4 | Enhanced `.lore-content` styles |
| `lib/get-creds.ts` | 1 | Add `getLoreRootNoteId()` |
| `lib/etapi-server.ts` | 2 | Add `patchAttribute()` |
| `lib/allknower-server.ts` | 5 | Add history detail function |
| `components/portal/AppSidebar.tsx` | 6 | Add Import nav entry |

### Dependencies to add

| Package | Feature | Purpose |
|---|---|---|
| `isomorphic-dompurify` | 4 | HTML sanitization for rendered lore content |

---

## Phase 1: Brain Dump Becomes The Front Door ✅

> **Shipped.** Implementation plan: Phase C. See `docs/shared/planning/implementation_plan_phases_0_3.md`.

**Outcome:** raw thought becomes reliable structured canon with low friction and visible review points.

### Portal

- redesign Brain Dump page around three clear modes: auto-create, review-first, inbox-first
- show extracted entities as cards with type, confidence, warnings, and action taken
- let users approve, edit, merge, or skip extracted entities before writing canon in review-first mode
- show likely duplicates and near-duplicates before creation
- surface contradiction warnings and related lore directly in the Brain Dump UI
- show links to created and updated notes inline in the results view
- add quick follow-up actions: `open note`, `apply relations`, `create session recap`, `send to timeline`

### AllKnower

- support explicit Brain Dump modes instead of a single create/update path
- return confidence scores per extracted entity
- add duplicate and near-duplicate detection against existing lore
- add contradiction and overlap checks during extraction
- return related lore suggestions alongside extracted entities
- generate summary, session recap, and candidate timeline events from the same dump
- preserve provenance metadata so downstream surfaces know which notes came from which dump

### AllCodex Core

- support or formalize provenance metadata on notes created from Brain Dump
- add or formalize templates for session, quest, scene, and event notes if current templates are too generic
- ensure notes created by Brain Dump can be grouped or browsed by source session or import batch

### Exit criteria

- Brain Dump is the fastest way to capture new canon without losing trust
- users can review and correct AI output before it becomes canon when desired
- created notes keep enough provenance to support history, recap, and cleanup flows later

**Estimated effort:** ~13–24h (stub mode to full contradiction detection)

---

## Phase 2: Wiki Article View And Knowledge Architecture ✅

> **Shipped.** Implementation plan: Phase B. See `docs/shared/planning/implementation_plan_phases_0_3.md`.

**Outcome:** AllCodex lore becomes easy to scan, navigate, and trust during play.

### Portal

- redesign lore detail pages to feel like real wiki articles instead of raw note renderers
- add a strong infobox or metadata panel driven by promoted attributes
- add breadcrumbs, tree position, and child-page navigation
- build a real hierarchy browser instead of a type-filter pretending to be a tree
- show outgoing and incoming relationships clearly
- improve relationship graph view and link it tightly to article reading
- add hover previews for linked notes based on summaries and metadata
- add a `create missing note` path from mention search and autolinker misses
- add table of contents support for long articles
- improve type, tag, and relation browsing from the main lore index

### AllKnower

- generate and refresh note summaries for hover cards and related-note panels
- suggest adjacent or missing related notes where article navigation feels sparse
- support lightweight summary regeneration when note content changes significantly

### AllCodex Core

- expose or improve APIs for breadcrumbs, branch trees, parent/child traversal, and backlinks if current APIs are not Portal-friendly
- expose share-safe rendered content and metadata cleanly enough for Portal article views
- formalize read-aloud, GM-only, and related content block conventions if current HTML patterns are too ad hoc

### Exit criteria

- article reading is fast enough for live-session scanning
- hierarchy is visible and navigable
- metadata and relations are obvious without reading the full page body
- linked notes have enough preview context to reduce tab churn

**Estimated effort:** ~16–22h

---

## Phase 3: Session Runtime And Continuity Guardrails ✅

> **Shipped.** Implementation plan: Phases D + E2 + F. See `docs/shared/planning/implementation_plan_phases_0_3.md`.

**Outcome:** the product becomes usable during a live session, not just before or after one.

### Portal

- add a live session workspace with pinned notes, current scene, quick search, and recent recap
- add quick-create actions for NPCs, locations, items, scenes, and secrets from inside the session workspace
- add a quest and hook tracker that is visible from both lore pages and session views
- add session notes, session recap views, and links between sessions and impacted lore
- add a lightweight timeline view with list mode first and richer filters later
- embed contradiction and related-lore prompts into quick capture flows

### AllKnower

- add an improvisation check endpoint that evaluates new canon text against existing lore
- return likely contradictions, overlaps, and affected entities quickly enough to support session use
- support `related lore for current scene` retrieval
- generate session recaps from raw notes or Brain Dumps
- generate NPC, location, scene, and consequence suggestions tuned for improvisation
- extract timeline events from session notes and dumps

### AllCodex Core

- add or formalize templates for sessions, quests, scenes, timeline events, and statblocks
- support branch or relation conventions that let Portal connect sessions to impacted notes
- add any missing ordering or branch-metadata helpers needed for session planning surfaces

### Exit criteria

- a GM can run a live session with fewer context switches
- quick capture during play can be checked against canon before it hardens into contradiction
- sessions, hooks, and timeline events are first-class enough to support campaign continuity

**Estimated effort:** ~14–36h (stub workspace to full session runtime)

---

## Phase 4: Player-Safe Sharing, Maps, And Import ✅

> **Shipped** (sharing, system-pack import, and Azgaar import). Implementation plan: Phase G + Phase F (system-pack import) + Feature 6 (Azgaar).
> Maps (embeds, pins, visual map viewer) are **deferred** to a future phase.

**Outcome:** selected lore can be safely shown to players; Azgaar Fantasy Map Generator JSON can be imported to seed location/faction/religion notes.

### Portal

- add a proper share and publish settings UI for notes and worlds
- add GM preview versus player-safe preview
- expose draft, GM-only, share-root, share-index, and related share controls cleanly
- ~~add Azgaar import UI with preview, select, import, and result reporting~~ ✅ shipped
- add map embeds to lore pages and session views *(deferred)*
- add clickable map pins that resolve to lore notes *(deferred)*
- add a lightweight map view for browsing linked places *(deferred)*

### AllKnower

- ~~add `POST /import/azgaar` plus import history and dry-run support~~ ✅ shipped
- ~~map Azgaar entities into AllCodex location, faction, and optional river/event notes~~ ✅ shipped
- ~~return import results in a Portal-friendly shape with created note links and warnings~~ ✅ shipped
- optionally generate follow-up summaries or hierarchy suggestions for imported geography *(deferred)*

### AllCodex Core

- expose or improve APIs for share metadata, passwords, aliases, and filtered share rendering
- ensure GM-only and draft content are always stripped from player-safe output
- formalize geolocation and map-pin attribute conventions so imports and manual edits land in one format
- expose map and note-link data cleanly enough for Portal map views

### Exit criteria

- player-safe lore can be shared without leaking GM-only content
- maps help navigation and context instead of becoming a separate product pillar
- imported map data lands as usable lore, not just bulk-created clutter

**Estimated effort:** ~8–23h

---

## Phase 5: Rules Reference And System Content Expansion ⚠️ Partial

> **Partially shipped.** Statblock library, system-pack import, and session-workspace statblock search are live.
> Homebrew editing, rules-aware retrieval, and expanded template coverage remain TODO.

**Outcome:** AllCodex becomes more useful at the table without turning into a full rules platform.

### Portal

- add a statblock library and fast statblock viewer
- make statblocks searchable from the session workspace
- add lightweight system-pack browsing for imported rules content
- add homebrew statblock editing only after reading and lookup are solid

### AllKnower

- support rules-aware retrieval for imported SRD or system-pack content
- ground NPC or monster generation to the selected system when relevant
- support timeline and lore reasoning that can reference system content when needed

### AllCodex Core

- expand template coverage for statblocks, items, creatures, spells, and related system entities
- support user-defined statblock templates only after the reader and library experience stabilizes

### Exit criteria

- rules reference helps the GM at the table without consuming roadmap priority meant for lore and continuity

**Estimated effort:** ~5–16h

---

## Workstreams

Cross-phase slices that coordinate work across services.

### Workstream A: Contract And Schema Stabilization — `P0`

- Brain Dump response normalization
- Brain Dump history detail endpoint and Portal detail view
- autocomplete shape normalization
- canonical promoted field registry
- share-state metadata contract
- contract tests between Portal proxy and AllKnower responses

### Workstream B: Low-Friction Lore Creation — `P0→P1`

- lore root note configuration
- quick-create from editor link misses
- create/edit parity for lore forms
- promoted attributes on create and edit
- quick-capture from session workspace
- duplicate detection before note creation

### Workstream C: Article View And Navigation — `P1`

- infobox and metadata panel
- hierarchy browser and breadcrumbs
- related notes and hover previews
- relationship graph integration
- table of contents
- tag and type browsing improvements

### Workstream D: Continuity And Contradiction Prevention — `P1`

- contradiction checks during Brain Dump review
- contradiction checks during live-session quick capture
- affected-entity and overlap warnings
- recap and timeline extraction from sessions and dumps

### Workstream E: Session Toolkit — `P1→P2`

- live session workspace
- quest and hook tracker
- session notes and recap archive
- fast note and statblock lookup
- pinned notes and current-scene context

### Workstream F: Player-Safe Publishing And Maps — `P2`

- GM/player preview
- share settings UI
- ~~map embeds and clickable pins~~ *(deferred)*
- ~~Azgaar import pipeline~~ ✅ shipped
- share-safe public article presentation

### Workstream G: Rules And System Content — `P3`

- statblock library
- rules pack import and browsing
- grounded rules-aware generation
- homebrew system-content views

### Cross-Cutting: Context Compaction (AllKnower infrastructure)

Infrastructure to keep RAG and LLM context usage efficient as lore databases grow.

| Tier | Description | Status |
|---|---|---|
| Tier 0 | Baseline RAG budget enforcement | ✅ Shipped |
| Tier 1 | Chunk deduplication (0.85 cosine threshold) — `rag/chunk-dedup.ts` | ✅ Shipped |
| Tier 2 | Chunk summarization — `rag/chunk-compactor.ts` | ✅ Shipped |
| Tier 3 | Session compaction — `pipeline/session-compactor.ts`, `lore_session` + `lore_session_message` Prisma tables | 🔧 Prepared (DB tables + compactor code exist, not wired to routes) |

See [analysis/context_compaction_plan.md](../analysis/context_compaction_plan.md) for the full design.

---

## Dependency Notes

1. Phase 0 ships before any large Brain Dump or sharing UX overhaul.
2. Brain Dump review flows depend on contract normalization and provenance metadata.
3. A good wiki article view depends on canonical promoted attributes and hierarchy APIs.
4. Live session tools build on article view, quick capture, and contradiction checks — not bypass them.
5. Map import waits until lightweight map display and geolocation conventions are decided.
6. Rules expansion waits until the DM notebook loop is clearly stronger.

---

## Overall Effort Summary

| Phase | Stub mode | Full implementation |
|---|---|---|
| Phase 0 (Foundation) | ~17h | ~24h |
| Phase 1 (Brain Dump) | ~13h | ~24h |
| Phase 2 (Wiki View) | ~16h | ~22h |
| Phase 3 (Session Runtime) | ~14h | ~36h |
| Phase 4 (Sharing/Maps) | ~8h | ~23h |
| Phase 5 (Rules) | ~5h | ~16h |
| Shared Components | ~21h | ~30h |
| **Total** | **~94h** | **~175h** |

Stub everything for visual fidelity: ~4 focused weeks solo.
Full implementation with real AI endpoints and session runtime: ~10–12 weeks solo.

---

## What To Build First

Highest-value sequence if only a few slices can run:

1. contract and schema stabilization
2. lore root config and promoted attribute parity
3. Brain Dump history detail and trusted Brain Dump outputs
4. article view redesign with hierarchy and infoboxes
5. Brain Dump review-first workflow with contradiction and duplicate warnings
6. live session workspace with quick capture, recap, and quest tracking
