# AllCodex DM-First Roadmap

> **Status: \ud83d\udce6 ARCHIVED** \u2014 Superseded by `docs/shared/ROADMAP.md` and `docs/shared/implementation_plan_phases_0_3.md`.
> All items in this file have been triaged into the canonical roadmap and implementation plan.
> Retained for historical reference only. **Do not update this file.**

---

## Product Objective

AllCodex should become the best AI-integrated DM notebook for:

- capturing ideas before setup friction kills them
- improvising during play without silently creating contradictions
- retrieving lore quickly in a readable wiki-like format
- letting AI handle the boring organization work
- presenting selected canon cleanly to players without turning the product into a social publishing platform

This roadmap assumes the DM-first scope is correct.

That means the roadmap favors:

- Brain Dump as the front door
- wiki readability and navigation over vanity presentation
- continuity and contradiction prevention over breadth
- player-safe read views over collaboration features
- exposing existing Core and AllKnower power before inventing large new systems

---

## Not In Near-Term Scope

These are explicitly de-prioritized unless the product direction changes:

- advanced fantasy calendars
- novelist and manuscript-first workflows
- public community and social features
- monetization and patron gating
- heavy multi-user collaboration
- full VTT integration
- full map creation tooling

Simple timeline support is in scope.

Huge calendar engines are not.

Lightweight map embeds and pins are in scope.

A full map editor is not.

---

## Success Criteria

The roadmap should produce a product where:

1. Brain Dump can be trusted as the fastest capture path, not just a demo feature.
2. The default lore view feels like a real wiki article instead of a raw note renderer.
3. A GM can run a live session without constantly leaving the product.
4. Improvised canon is checked against existing lore before it quietly causes continuity problems.
5. Players can read selected lore safely without exposing GM-only information.

---

## Delivery Order

| Phase | Focus | Why now |
|---|---|---|
| Phase 0 | Foundation and contract repair | Remove trust-breaking drift and data-shape bugs first |
| Phase 1 | Brain Dump becomes the front door | This is the core differentiator and fastest path to product identity |
| Phase 2 | Wiki article view and knowledge architecture | The product needs a strong read surface, not just AI generation |
| Phase 3 | Session runtime and continuity guardrails | Move from note system to actual DM tool |
| Phase 4 | Player-safe sharing, maps, and import | Expand outward only after the GM loop is solid |
| Phase 5 | Rules reference and system-content expansion | Useful, but secondary to core lore and continuity flows |

---

## Cross-Service Foundation

Before feature work spreads, the three services need a few shared assumptions.

### Shared contracts

Define and document canonical JSON shapes for:

- Brain Dump create response
- Brain Dump history list response
- Brain Dump history detail response
- autocomplete suggestions
- promoted attribute keys per lore type
- share-state metadata exposed to the Portal

The current goal is not a new shared package at any cost.

The goal is to stop Portal and AllKnower from drifting on response shapes and field names.

### Shared product conventions

Define and document canonical conventions for:

- lore type keys
- promoted attribute keys
- session, quest, timeline, and statblock note types
- GM-only and draft visibility behavior
- geolocation and map-pin metadata
- provenance metadata for notes created from Brain Dump or imports

### Contract example

The normalized Brain Dump history detail should look like this regardless of which service generated it:

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

## Phase 0: Foundation And Contract Repair

Priority: `P0`

Outcome: the existing product stops feeling inconsistent, fragile, or misleading before larger DM workflows are added.

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

### Cross-service exit criteria

- Brain Dump create, list, and detail views all use one documented contract
- autocomplete no longer silently fails because of shape mismatches
- promoted attribute keys are canonical and consistent
- new lore no longer falls into the wrong root by default

---

## Phase 1: Brain Dump Becomes The Front Door

Priority: `P1`

Outcome: raw thought becomes reliable structured canon with low friction and visible review points.

### Portal

- redesign Brain Dump page around three clear modes: auto-create, review-first, inbox-first
- show extracted entities as cards with type, confidence, warnings, and action taken
- let users approve, edit, merge, or skip extracted entities before writing canon in review-first mode
- show likely duplicates and near-duplicates before creation
- surface contradiction warnings and related lore directly in the Brain Dump UI
- show links to created and updated notes inline in the results view
- add quick follow-up actions such as `open note`, `apply relations`, `create session recap`, `send to timeline`

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

### Cross-service exit criteria

- Brain Dump is the fastest way to capture new canon without losing trust
- users can review and correct AI output before it becomes canon when desired
- created notes keep enough provenance to support history, recap, and cleanup flows later

---

## Phase 2: Wiki Article View And Knowledge Architecture

Priority: `P1`

Outcome: AllCodex lore becomes easy to scan, navigate, and trust during play.

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
- formalize read-aloud, GM-only, and related content block conventions if the current HTML patterns are too ad hoc

### Cross-service exit criteria

- article reading is fast enough for live-session scanning
- hierarchy is visible and navigable
- metadata and relations are obvious without reading the full page body
- linked notes have enough preview context to reduce tab churn

---

## Phase 3: Session Runtime And Continuity Guardrails

Priority: `P1` moving into `P2`

Outcome: the product becomes usable during a live session, not just before or after one.

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

### Cross-service exit criteria

- a GM can run a live session with fewer context switches
- quick capture during play can be checked against canon before it hardens into contradiction
- sessions, hooks, and timeline events are first-class enough to support campaign continuity

---

## Phase 4: Player-Safe Sharing, Maps, And Import

Priority: `P2`

Outcome: selected lore can be safely shown to players, and maps become useful support surfaces without taking over the product.

### Portal

- add a proper share and publish settings UI for notes and worlds
- add GM preview versus player-safe preview
- expose draft, GM-only, share-root, share-index, and related share controls cleanly
- add map embeds to lore pages and session views
- add clickable map pins that resolve to lore notes
- add a lightweight map view for browsing linked places
- add Azgaar import UI with preview, select, import, and result reporting

### AllKnower

- add `POST /import/azgaar` plus import history and dry-run support
- map Azgaar entities into AllCodex location, faction, and optional river/event notes
- return import results in a Portal-friendly shape with created note links and warnings
- optionally generate follow-up summaries or hierarchy suggestions for imported geography

### AllCodex Core

- expose or improve APIs for share metadata, passwords, aliases, and filtered share rendering
- ensure GM-only and draft content are always stripped from player-safe output
- formalize geolocation and map-pin attribute conventions so imports and manual edits land in one format
- expose map and note-link data cleanly enough for Portal map views

### Cross-service exit criteria

- player-safe lore can be shared without leaking GM-only content
- maps help navigation and context instead of becoming a separate product pillar
- imported map data lands as usable lore, not just bulk-created clutter

---

## Phase 5: Rules Reference And System Content Expansion

Priority: `P3`

Outcome: AllCodex becomes more useful at the table without turning into a full rules platform too early.

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

### Cross-service exit criteria

- rules reference helps the GM at the table without consuming roadmap priority meant for lore and continuity

---

## Detailed Workstreams

This section lists the main backlog slices that cut across phases.

### Workstream A: Contract And Schema Stabilization

Priority: `P0`

- Brain Dump response normalization
- Brain Dump history detail endpoint and Portal detail view
- autocomplete shape normalization
- canonical promoted field registry
- share-state metadata contract
- contract tests between Portal proxy and AllKnower responses

### Workstream B: Low-Friction Lore Creation

Priority: `P0` to `P1`

- lore root note configuration
- quick-create from editor link misses
- create/edit parity for lore forms
- promoted attributes on create and edit
- quick-capture from session workspace
- duplicate detection before note creation

### Workstream C: Article View And Navigation

Priority: `P1`

- infobox and metadata panel
- hierarchy browser and breadcrumbs
- related notes and hover previews
- relationship graph integration
- table of contents
- tag and type browsing improvements

### Workstream D: Continuity And Contradiction Prevention

Priority: `P1`

- contradiction checks during Brain Dump review
- contradiction checks during live-session quick capture
- affected-entity and overlap warnings
- recap and timeline extraction from sessions and dumps

### Workstream E: Session Toolkit

Priority: `P1` to `P2`

- live session workspace
- quest and hook tracker
- session notes and recap archive
- fast note and statblock lookup
- pinned notes and current-scene context

### Workstream F: Player-Safe Publishing And Maps

Priority: `P2`

- GM/player preview
- share settings UI
- map embeds and clickable pins
- Azgaar import pipeline
- share-safe public article presentation

### Workstream G: Rules And System Content

Priority: `P3`

- statblock library
- rules pack import and browsing
- grounded rules-aware generation
- homebrew system-content views

---

## Dependency Notes

1. Phase 0 should ship before any large Brain Dump or sharing UX overhaul.
2. Brain Dump review flows depend on contract normalization and provenance metadata.
3. A good wiki article view depends on canonical promoted attributes and hierarchy APIs.
4. Live session tools should build on article view, quick capture, and contradiction checks, not bypass them.
5. Map import should wait until lightweight map display and geolocation conventions are decided.
6. Rules expansion should wait until the DM notebook loop is clearly stronger than it is today.

---

## Existing Backlog Items Mapped Into This Roadmap

The current Portal roadmap items fit into the new plan like this:

| Existing item | New phase | Notes |
|---|---|---|
| Lore root note config | Phase 0 | Keep as-is |
| Promoted attributes create/edit | Phase 0 | Keep as-is, but use canonical field registry |
| Edit page parity | Phase 0 | Keep as-is |
| Rendered lore content view | Phase 0 and Phase 2 | Ship sanitization first, then full article redesign |
| Brain Dump history detail | Phase 0 | Keep as-is and tie to contract normalization |
| Azgaar import | Phase 4 | Keep, but lower priority than core DM loop work |

---

## What To Build First

If only the next few roadmap slices can be funded, the highest-value order is:

1. contract and schema stabilization
2. lore root config and promoted attribute parity
3. Brain Dump history detail and trusted Brain Dump outputs
4. article view redesign with hierarchy and infoboxes
5. Brain Dump review-first workflow with contradiction and duplicate warnings
6. live session workspace with quick capture, recap, and quest tracking

That sequence follows the DM-first scope without overcommitting to maps, publishing, or rules-platform ambitions too early.
