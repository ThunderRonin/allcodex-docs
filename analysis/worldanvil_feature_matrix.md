# AllCodex vs World Anvil - Comprehensive Feature Matrix

> Implementation-verified audit of the World Anvil feature pages stored in `docs/worldanvil/`.
> Prefer this document over the older high-level gap note when making parity or roadmap decisions.

Date: April 2, 2026

> **Phase A–G updates applied.** Many items previously rated `half-assed` or `Can have` are now `comparable` or `better` after shipping: session workspace, quests, timeline, statblock library, system-pack import, player-safe sharing, and 15 AllCodex Core templates.

---

## Scope

This audit covers the attached World Anvil source pages:

- `- Dashboard  World Anvil.md`
- `- Dashboard  World Anvil(1).md`
- `- Dashboard  World Anvil(2).md`
- `- Dashboard  World Anvil(3).md`
- `Create A Wiki For Your Worldbuilding  World Anvil  World Anvil.md`
- `Fantasy Timeline Maker for DnD, RPG Games & Novels  World Anvil.md`
- `agile-worldbuilding.md`

The source bundle mixes three kinds of material:

1. Product features
2. Product marketing copy
3. Agile Worldbuilding methodology guidance

The repeated dashboard slogans such as "Design Your World", "Showcase Your Creations", and "Save Your Notes" are folded into the concrete feature rows below instead of being duplicated as separate items.

---

## How To Read This

### Current verdict labels

- `Already have - better`: implemented and materially stronger or more flexible than the World Anvil equivalent.
- `Already have - comparable`: implemented and roughly on par.
- `Already have - half-assed`: real feature exists, but the UX, exposure, or completeness is rough.
- `Can have`: feasible with the current architecture, but not implemented yet.
- `Skip for now`: technically possible, but not a good near-term priority.

### Effort scale

- `S`: days
- `M`: 1-2 weeks
- `L`: 3-6 weeks
- `XL`: multi-month or architecture-heavy work

---

## Executive Summary

AllCodex is already ahead of World Anvil in AI-assisted lore work: Brain Dump, semantic search, consistency checking, relationship suggestion, and gap detection are meaningful differentiators.

AllCodex is broadly competitive on core authoring: rich editing, `@` mentions, autolinking, template-driven creation, structured fields, lore browsing, and direct lore CRUD are all real.

The biggest gaps are not in note-taking or intelligence. They are in:

- maps, calendars, and interactive visualization
- per-user collaboration and audience segmentation
- VTT integrations and system-content packaging beyond basic import

---

## Feature Matrix

## 1. Core Wiki, Authoring, and Sharing

| World Anvil feature | Current verdict | What we have now | How to reach parity or exceed it | Effort |
|---|---|---|---|---|
| Custom worldbuilding wiki / game wiki / book wiki / RPG wiki | Already have - comparable | This is the core AllCodex model: notes, attributes, relations, branches, share pages, and a dedicated Portal. | No parity work needed. Improve hierarchy and publishing UX. | - |
| Quick search across the whole wiki | Already have - better | Portal ships both semantic search and attribute/full-text search. | No parity work needed. | - |
| Categories / folder-like hierarchy / nested sub-categories | Already have - comparable | `LoreTree.tsx` renders a full nested branch hierarchy in the lore sidebar. Drag-reparent via ETAPI is supported. | Improve UX polish (expand/collapse memory, lazy-load deep branches). | S |
| Tags | Already have - comparable | Labels behave as tags; `LoreTree` groups by loreType; label editing in note edit view. | Add dedicated tag browsing if needed. | S |
| Link articles together while writing | Already have - better | `@` mentions and inline links already exist in the editor. | No parity work needed. | - |
| Quick-create a new linked article while writing | Can have | The editor and lore creation flow already exist, but mention search currently only links existing notes. | Add a "create missing note" action from mention search or autolinker misses. | M |
| Mentions system | Already have - better | `@` autocomplete is implemented with ETAPI search plus optional AI fallback. | No parity work needed once the autocomplete contract drift is fixed. | S |
| Autolinker | Already have - comparable | A document scan plus selective apply dialog already exists. | No parity work needed. | - |
| Autolinker control modes (automatic / partial / manual) | Already have - half-assed | Current flow is review-and-select, which covers manual or semi-manual use, but not clear one-click or graduated modes. | Add explicit `link all`, `review`, and `manual only` modes. | S |
| Re-run autolinker on older articles after new notes exist | Already have - comparable | Current autolinker can be run against any open note body. | No parity work needed. | - |
| Rich text editor / WYSIWYG authoring | Already have - better | Tiptap/Novel editor with slash commands, formatting, uploads, mentions, autolink, task lists, and bubble menu. | No parity work needed. | - |
| Template browser / guided article creation | Already have - comparable | `TemplatePicker` and `PromotedFields` already support guided entry creation. | No parity work needed. | - |
| Template-specific structured fields | Already have - comparable | Structured fields exist in the Portal and in AllKnower's template seeding. | Normalize field names so Portal and AllKnower use the same canonical keys. | S |
| Broad template coverage | Already have - comparable | 15 AllCodex Core templates (Character, Location, Faction, Creature, Event, Timeline, Manuscript, Statblock, Item, Spell, Building, Language, Session, Quest, Scene). AllKnower supports 21 entity types total (6 brain-dump-only: organization, race, myth, cosmology, deity, religion). | Expose the remaining 6 brain-dump-only types in the Portal picker. | S |
| Hoverable article previews / richer linked-entry context | Already have - half-assed | Mention suggestions show title and lore type, but there is no rich hover preview of note content. | Add lightweight hover cards fed by note summaries. | S-M |
| Theme the wiki to match the world | Already have - half-assed | Portal has one hardcoded grimoire theme. Core share theming is more flexible under the hood, but not surfaced in Portal. | Expose per-world theme configuration and share theme selection. | M |
| Build privately before publishing | Already have - better | Default behavior is private authoring. Public exposure is opt-in through share mechanisms. | No parity work needed. | - |
| Draft vs published lifecycle | Already have - comparable | `#draft` is real and Portal exposes a draft toggle. Shared output hides drafts. | No parity work needed. | - |
| Public wiki / publication surface | Already have - comparable | Core public share pages plus Portal `/shared` browser with share toggle, password, and preview links. | Polish publish workflow and discovery UX. | S-M |
| Per-entry public / private toggle | Already have - comparable | `ShareSettings.tsx` provides per-note share toggle, password, and visibility controls. `PreviewToggle.tsx` switches between GM and player views. | No parity work needed. | - |
| Advanced privacy settings | Already have - comparable | Core supports passworded shares, `gmOnly`, `draft`, `shareIndex`, and related share labels. Portal `ShareSettings` surfaces these. `PreviewToggle` allows GM vs. player preview. | Add per-user access control as a future enhancement. | M-L |
| Exclusive access for patrons / customers / beta readers | Can have | Core has the idea of public vs hidden content, but not true audience groups or ACL-driven share filtering. | Build group-based access control and share filtering. | L-XL |
| Co-authors / teams | Can have | Current auth and Portal flows are oriented around a single owner, not collaborative editing. | Introduce multi-user permissions, conflict rules, and invite flows. | XL |
| Revoke collaborator access later | Can have | Same dependency as co-authoring. | Same project as team collaboration and ACLs. | XL |
| Novel writing software integrated with lore | Already have - half-assed | Manuscript note type exists, but there is no dedicated manuscript workspace. | Build a manuscript/editor workflow optimized for writing rather than generic note editing. | M |
| Search and edit lore from the writing interface | Already have - half-assed | Search exists globally, but not as a dedicated split-pane or contextual manuscript tool. | Add manuscript-side lore search and quick edit panel. | M |

## 2. Worldbuilding Process, Structure, and Visualization

| World Anvil feature | Current verdict | What we have now | How to reach parity or exceed it | Effort |
|---|---|---|---|---|
| Worldbuilding meta overview tool | Can have | There is no dedicated world-meta page or note type in Portal. | Create a world-meta schema, note type, or guided setup page. | M |
| Guided meta sections: scope, theme, drama, scene, people | Can have | No first-class UI yet. | Add a guided world-meta wizard or structured form. | M |
| Motivation tracking | Can have | Not first-class. | Add motivation fields to world-meta notes. | S |
| Inspiration repository | Can have | Not first-class, though attachments and notes can hold inspiration informally. | Add inspiration fields or media sections to world-meta. | S |
| Use meta to generate new ideas | Already have - half-assed | AllKnower can help indirectly, but there is no meta-driven ideation flow. | Add prompt actions that read world-meta and generate next-step ideas. | M |
| Agile Worldbuilding methodology support | Can have | The product supports flexible note-taking, but not an explicit Agile Worldbuilding guide or workflow. | Add a setup flow and docs-driven templates for Agile Worldbuilding. | S-M |
| One-page foundation setup | Can have | Not first-class. | Make world-meta the canonical one-page foundation note. | S-M |
| "Build in sentences and paragraphs" discipline | Can have | Notes can be short today, but nothing nudges creators toward lightweight, just-in-time lore. | Add template guidance, helper copy, or size heuristics. | S |
| Content trees / magic systems / guild hierarchies / tech trees | Already have - half-assed | Mermaid, relationship graphs, and note-map primitives exist, but there is no first-class content-tree feature. | Build a dedicated tree renderer/editor over note relations or Mermaid definitions. | M |
| Interactive family trees / bloodlines / dynasties | Already have - half-assed | Family relationships exist conceptually, but there is no genealogy-specific view. | Add a family-tree renderer over `family` relations. | M |
| Random loot tables / encounter tables / rollable tables | Can have | No dedicated roll-table system exists. | Add table schemas, weighted rows, dice evaluation, and inline rendering. | M |
| Link table results to statblocks | Can have | Depends on roll tables. | Resolve rows to note IDs and statblocks once roll tables exist. | M |
| Global worldbuilding to-do list | Already have - half-assed | Task list blocks exist in the editor, but there is no global task index or automatic "unfinished article" list. | Build a task dashboard and note-completion tracker. | M |
| Interactive maps | Can have | Core has note-map and geo-map primitives, but Portal has no interactive map UI. | Build a visual map layer in Portal using current map/note infrastructure. | L |
| Map popups / minor location blurbs | Can have | Not exposed in Portal. | Add marker popups tied to note excerpts and lightweight location notes. | M-L |
| Chronicles: combine maps and timelines | Can have | No combined visualization exists today. | Layer events onto maps after map + timeline foundations are built. | L |
| Visual timeline | Already have - comparable | `/timeline` page renders events chronologically with in-world dates and sort controls. | Add richer filtering and zoom controls. | S |
| Simple list-mode timeline | Already have - comparable | `/timeline` ships as a list view by default. | No parity work needed. | - |
| Fantasy calendar generator | Can have | No calendar subsystem exists. | Build a calendar model, renderer, and recurrence rules. | L |
| Custom months and days | Can have | Not present. | Add custom calendar schema fields. | M-L |
| Recurring festivals | Can have | Not present. | Add recurring event rules on top of calendar support. | M-L |
| Celestial bodies / moon cycles | Can have | Not present as a system feature. | Add astronomy cycle data to the calendar engine. | M-L |
| Secrets and subscriber groups | Can have | Secret content ideas exist through `gmOnly` and hidden text patterns, but not audience groups. | Build audience groups and filtered share rendering. | L-XL |
| GM notes section | Already have - half-assed | Core can hide `gmOnly` notes and strip `.gm-only` blocks in share rendering, but Portal authoring UX is rough. | Add explicit GM-only and hidden-section controls in the editor and note settings. | S-M |
| Separate player-facing flavor text / read-aloud text | Already have - half-assed | The editor can express this informally, but there is no dedicated read-aloud block type. | Add a first-class "read aloud" block or callout style. | S |

## 3. RPG, Campaign, and System Tooling

| World Anvil feature | Current verdict | What we have now | How to reach parity or exceed it | Effort |
|---|---|---|---|---|
| Campaign planner | Already have - comparable | Session workspace (`/session`), quest tracker (`/quests`), timeline (`/timeline`), and scene templates provide campaign planning infrastructure. | Add multi-campaign support and campaign-specific lenses. | M |
| Session planner / session organizer | Already have - comparable | `/session` workspace with recap, timer, statblock lookup, quick-create. Session template with date/players/status/hooks/gmNotes. | Add session scheduling and player invite flows. | S-M |
| Session reports | Already have - comparable | Brain Dump can summarize raw notes. Session template captures recap, hooks, and GM notes. `/brain-dump/[id]` shows full detail. | Add dedicated recap generation flow. | S |
| Digital DM screen | Already have - comparable | `/session` workspace: pinned recap, current scene, quick search, statblock lookup, quick-create NPCs/locations/items. | Add pinnable note panels and customizable layout. | S-M |
| Create NPCs during play | Already have - comparable | Quick-create in session workspace supports NPC, location, item, scene, and secret creation. Brain Dump also available. | No parity work needed. | - |
| Statblock library | Already have - comparable | `/statblocks` page with CR filter, search, and full D&D-style stat cards (`StatblockCard.tsx`). | Add sorting and advanced filtering. | S |
| Homebrew statblocks | Already have - comparable | Full statblock template with 20+ promoted attributes (crName, CR, type, size, alignment, AC, HP, speed, ability scores, immunities, resistances, vulnerabilities, abilities, actions, legendary actions). | Add visual statblock editor. | S-M |
| Homebrew spells, items, monsters, classes, races | Already have - half-assed | Many of these are supported as note/entity types, but not as polished RPG system content. | Extend template coverage and create system-focused views. | M |
| Search and pull statblocks during a session | Already have - comparable | Session workspace includes statblock lookup. `/statblocks` page provides browsing and search. | No parity work needed. | - |
| Roll statblocks or actions instantly at the table | Can have | No dice or combat runner exists. | Add a dice/roller layer and statblock action execution affordances. | S-M |
| PC sheets | Can have | No PC sheet subsystem exists. | Create character-sheet types and player/campaign associations. | M-L |
| Factions, timelines, and world details in campaign context | Already have - half-assed | Factions and timeline-capable note types exist, but not inside a campaign-specific organizer. | Add campaign-specific lenses and associations. | M |
| Player journeys | Can have | No dedicated player-journey model exists. | Add journey or arc entities linked to campaigns and sessions. | M |
| Add images, music, and interactive dungeon maps to session plans | Can have | Notes support rich content and images, but there is no session-planning shell. Music and interactive maps are not surfaced. | Build media-aware session planner once campaign tooling exists. | L |
| Dice rolling | Can have | No dice engine or inline roller exists. | Add a small dice engine and UI components. | S-M |
| D&D 5e SRD | Already have - comparable | System-pack import (`/import`) with preview, duplicate-skip, and result reporting. Statblock library provides browse/search. | Add more system packs beyond 5e SRD. | S-M |
| Pathfinder / PF2 support | Can have | System-pack architecture supports arbitrary JSON packs. | Create PF2 pack format and import flow. | M |
| Dozens of systems support | Already have - half-assed | System-pack import supports arbitrary JSON packs. The statblock schema includes a `system` field. Currently only D&D 5e SRD is available as a pack. | Create packs for more systems. | M |
| Foundry VTT integration | Can have | No VTT integration exists. | Ship export/sync APIs or a Foundry module. | L |
| Create your own RPG system | Already have - half-assed | Notes and share pages can describe a system, but there is no dedicated ruleset builder. | Add system definitions, templates, and publishing flows. | M-L |
| Tailor your own statblock templates | Can have | Backend template machinery exists, but Portal does not expose user-defined template editing. | Build template-builder UX on top of template notes and canonical fields. | M-L |
| Manage rulesets | Can have | No first-class ruleset subsystem exists. | Create ruleset entities and system-aware views. | M-L |
| Run play tests | Can have | No explicit playtest workflow exists. | Add campaign/session/test report flows after rulesets exist. | L |
| Share an SRD with the world | Already have - half-assed | Core share pages can publish content, but there is no polished SRD packaging workflow. | Add ruleset publishing UX over existing share infrastructure. | M |
| Secret info to some players but not others | Can have | No player-group ACL exists today. | Same work as audience segmentation and multi-user access. | L-XL |

## 4. Publication, Monetization, Community, and Growth

| World Anvil feature | Current verdict | What we have now | How to reach parity or exceed it | Effort |
|---|---|---|---|---|
| Publication features | Already have - half-assed | Core public sharing exists, but Portal publication UX is thin. | Build publish/share workflows and note/world-level publication settings. | M |
| Monetization | Can have | No billing, entitlements, or gated audience groups exist. | Add payments plus entitlement-aware sharing only if it becomes a real business priority. | XL |
| Patron / customer exclusive content | Can have | Depends on monetization plus audience groups. | Same project as ACL-driven share segmentation. | XL |
| Creative / GM community | Can have | No social/community surface exists. | Build only if community becomes a deliberate product pillar. | XL |
| Challenges / creator discovery | Skip for now | This is a separate social product surface, not a core worldbuilding requirement. | Defer until core product direction is stable. | - |
| Guided onboarding / writer-vs-GM workflows / lessons | Can have | No onboarding wizard exists. | Add a getting-started flow once core UX stabilizes. | S-M |

---

## Where AllCodex Is Already Ahead

These are not parity items; they are current differentiators:

- **Brain Dump**: raw prose to structured lore entities
- **Semantic search**: concept search, not just title/tag search
- **Consistency checking**: contradiction and continuity analysis
- **Relationship suggestion**: AI-assisted lore graph building
- **Gap detection**: coverage analysis for underdeveloped areas

This matters because the main World Anvil gap is not intelligence. It is productized authoring, visualization, and collaboration UX.

---

## Investigation Findings And Recommendations

These are recommendations that surfaced while tracing the implementation, not just while reading the World Anvil marketing pages.

| Priority | Finding | Why it matters | Recommendation | Effort |
|---|---|---|---|---|
| ~~P0~~ | ~~Brain Dump `POST` response contract is drifting~~ | **Resolved in Phase A.** Contract normalized in Portal proxy with shared response types. | ~~Define one shared Brain Dump response contract.~~ | - |
| ~~P0~~ | ~~Brain Dump history contract is wrong today~~ | **Resolved in Phase A.** `/brain-dump/history/:id` route added; Portal history UI updated. | ~~Add history-detail route.~~ | - |
| P0 | AI autocomplete contract is drifting | AllKnower autocomplete returns `suggestions`, while the Portal AllKnower client reads `results`. This likely drops AI autocomplete fallback on the floor. | Normalize the autocomplete response shape and cover it with a tiny integration test. | S |
| P1 | Promoted attribute naming is inconsistent | The Portal currently uses human-facing labels as attribute names in some paths, while AllKnower template seeding and backend lore schemas use canonical lowercase field names. This will create drift across notes. | Introduce a single canonical field registry shared by Portal and AllKnower. UI labels should map to canonical keys, not become keys themselves. | S |
| P1 | The Portal docs overstate hierarchy parity | The existing high-level gap doc described `half-assed` hierarchy, but `LoreTree` now renders genuine nested branches. | Docs updated to reflect actual shipped behavior. | - |
| ~~P1~~ | ~~Share and publish capabilities are underexposed~~ | **Resolved in Phase G.** `ShareSettings.tsx`, `PreviewToggle.tsx`, `/shared` browser, and settings share configuration card now surface Core share primitives. | ~~Surface share-state in Portal.~~ | - |
| P1 | The backend already supports more lore types than the Portal exposes | AllKnower defines 21 entity types; AllCodex Core has 15 templates; Portal picker exposes 15. Remaining 6 (organization, race, myth, cosmology, deity, religion) are brain-dump-only. | Expose the 6 remaining types in the Portal picker. | S |
| P2 | Collaboration features need an explicit auth decision first | AllKnower auth is explicitly documented as single-owner oriented. Adding co-authors, player groups, or patron tiers without a deliberate auth/ACL model will create rework. | Decide whether the product remains single-owner with public sharing or evolves into true multi-user collaboration. Do this before building co-authoring or subscriber groups. | M-L |
| P2 | Roadmap choice should be explicit: writer-first or GM-first | The next large investments differ sharply. Writer-first favors manuscript UX, world meta, and publication. GM-first favors maps, timelines, and DM tools. | Pick a near-term product identity before taking on maps, campaign tooling, or monetization. | S |
| P2 | Documentation drift is already appearing | The older gap analysis now overstates some shipped features and understates some implementation mismatches. | Treat implementation-verified audits like this one as the source of truth, and re-audit docs when major features land. | S |

---

## Recommended Near-Term Roadmap

If the goal is the fastest, most credible World Anvil parity story, the best sequence is:

1. **Stabilize the contracts**
   - Fix Brain Dump response/history shape drift.
   - Fix autocomplete response drift.
   - Normalize promoted attribute keys.

2. **Expose what already exists underneath**
   - Real hierarchy/tree UX in Portal.
   - Publish/share controls for existing AllCodex Core share primitives.
   - Missing lore types already supported by backend schemas.

3. **Pick one major product lane**
   - **Writer-first**: manuscript workspace, world meta, publication polish.
   - **GM-first**: timeline, map layer, campaign/session workspace, statblock library.

4. **Only then take on the heavy systems**
   - collaboration and audience groups
   - calendars
   - campaign management
   - VTT integrations
   - monetization

---

## Key Implementation Anchors

These were the most important files used in this audit:

- `allcodex-portal/components/editor/LoreEditor.tsx`
- `allcodex-portal/components/editor/mention-extension.tsx`
- `allcodex-portal/components/editor/AutolinkerDialog.tsx`
- `allcodex-portal/components/editor/TemplatePicker.tsx`
- `allcodex-portal/components/editor/PromotedFields.tsx`
- `allcodex-portal/app/(portal)/lore/page.tsx`
- `allcodex-portal/app/(portal)/lore/new/page.tsx`
- `allcodex-portal/app/(portal)/lore/[id]/page.tsx`
- `allcodex-portal/app/(portal)/lore/[id]/edit/page.tsx`
- `allcodex-portal/app/(portal)/search/page.tsx`
- `allcodex-portal/app/(portal)/brain-dump/page.tsx`
- `allcodex-portal/components/portal/LoreTree.tsx`
- `allcodex-portal/components/portal/RelationshipGraph.tsx`
- `allcodex-portal/lib/allknower-server.ts`
- `allcodex-portal/lib/etapi-server.ts`
- `allcodex-portal/app/api/brain-dump/route.ts`
- `allcodex-portal/app/api/brain-dump/history/route.ts`
- `allcodex-portal/app/api/lore/route.ts`
- `allcodex-portal/app/api/lore/mention-search/route.ts`
- `allcodex-portal/app/api/lore/autolink/route.ts`
- `allcodex-portal/app/api/lore/move/route.ts`
- `allknower/src/pipeline/brain-dump.ts`
- `allknower/src/routes/brain-dump.ts`
- `allknower/src/routes/suggest.ts`
- `allknower/src/routes/consistency.ts`
- `allknower/src/routes/setup.ts`
- `allknower/src/types/lore.ts`
- `allknower/src/auth/index.ts`
- `allcodex-core/apps/server/src/share/content_renderer.ts`
- `allcodex-core/apps/server/src/share/routes.ts`
- `allcodex-core/apps/server/src/routes/api/note_map.ts`
- `allcodex-core/apps/server/src/routes/api/relation-map.ts`

---

## Bottom Line

AllCodex does **not** need a brand-new foundation to compete with World Anvil.

It already has the note model, AI layer, share engine, relation model, and rich editor. The biggest wins now come from:

- fixing contract drift
- surfacing latent platform capabilities
- choosing a product lane before building expensive subsystems

That is a much better position than needing to invent the core product from scratch.