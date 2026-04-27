# AllCodex DM-First Product Scope

Date: March 31, 2026

This document turns the recent World Anvil comparison and product-discovery answers into a practical scope boundary for AllCodex.

It is intentionally opinionated.

The goal is not to describe every feature the platform could support. The goal is to decide what AllCodex should optimize for if it remains a smart AI-integrated DM notebook instead of drifting into a general-purpose writer platform, social wiki, or full VTT.

See also:

- `docs/shared/planning/ROADMAP.md` for the canonical cross-service roadmap
- `docs/shared/analysis/worldanvil_feature_matrix.md` for the implementation-verified parity audit
- `docs/shared/analysis/gap_analysis_vs_worldanvil.md` for the older high-level gap note

---

## Product Identity

AllCodex should be the best tool for Dungeon Masters who need to:

- capture ideas before the setup friction kills them
- improvise during play without silently creating contradictions
- retrieve lore quickly in a readable wiki-like format
- let AI do the boring organizing work after or during capture

This is **not** primarily a novel-writing product.

This is **not** primarily a public community publishing platform.

This is **not** primarily a collaboration-first workspace.

The right mental model is:

> Fandom-style readability plus AI-assisted DM memory and campaign organization.

---

## Core User And Use Cases

### Primary user

- Solo GM

### Secondary users

- Small GM team
- Players as read-only consumers of selected lore

### Most important real-world moments

1. Running a live session
2. Preparing sessions and campaigns
3. Capturing ideas fast before they evaporate
4. Checking whether new canon contradicts existing lore
5. Browsing the world in a wiki view that is fast to scan

### Primary pain points gathered from discovery

1. Improvisation creates contradictions because recall is weaker than improvisation speed.
2. Capturing a new idea has too much organizational overhead, so the idea dies before it becomes a note.
3. Existing lore presentation in AllCodex is not motivating or scannable enough compared to wiki-style tools.

---

## Design Principles

These principles should decide feature priority when scope gets fuzzy.

1. `Brain Dump first`
   If a feature does not improve the path from raw thought to usable canon, it should justify itself very hard.

2. `Reduce chore load`
   AI should absorb note type selection, relation extraction, metadata filling, tag/attribute creation, linking, and duplicate detection wherever possible.

3. `Support improvisation safely`
   The system should help GMs add canon quickly while warning about likely contradictions and surfacing related lore.

4. `Wiki view for scanning, not vanity`
   The wiki-like article view matters because it helps the GM read, navigate, and trust the world state faster, not because public publishing is the core business.

5. `DM utility beats writer utility`
   When a feature benefits prose authors more than active GMs, it is probably the wrong near-term investment.

6. `Read-only player access beats collaboration`
   Players mostly need selective visibility into canon, not editing power.

7. `Prefer exposing existing backend power before inventing new systems`
   AllCodex Core and AllKnower already have more capability than the Portal currently surfaces.

---

## Priority Shape

The current discovery suggests this order of importance:

1. Consistency and contradiction prevention
2. Deep world structure and relationship clarity
3. Beautiful, readable wiki presentation
4. Speed during play
5. Fast capture with low setup
6. Public sharing

This ordering has one important implication:

AllCodex should not chase "pretty wiki" as an end in itself. The visual/wiki layer should serve comprehension, scanning, and continuity management.

---

## Keep For The DM-First Product

These are aligned with the product identity and should stay in scope.

### Core capture and AI

- Brain Dump inbox and Brain Dump to structured lore
- AI-assisted note organization
- relationship suggestions
- consistency checking
- gap detection
- NPC and location generation
- session recap generation
- autolinking and mention linking

### Core wiki and lore reading

- readable article layout closer to a fandom/wiki experience
- clear infobox or metadata block
- better navigation between related pages
- stronger category hierarchy and browsing
- cleaner relationship display
- GM-only hidden knowledge in shared or player-facing views

### Core DM content model

- characters and NPCs
- locations
- factions and organizations
- events and incidents
- quests and hooks
- sessions and recaps
- secrets and clues
- scenes and encounters
- items and artifacts
- creatures and monsters
- timelines and eras
- rules references and statblocks

### Core campaign support

- quest and hook tracking
- session recap system
- timeline view
- relationship graph view
- rules and statblock viewing

### Player-facing surface

- player-facing wiki pages in some form
- selective stripping of GM-only content

---

## Maybe Later

These are useful, but they should not distort the product core.

- campaign dashboard or DM screen as a dedicated surface
- scratchpad or fleeting notes as a formal system
- lore Q&A chat as a standalone product surface
- public share links as a polished publishing workflow

Important nuance:

Some of these may still exist in v1 in lightweight form. The point is that they should not dominate roadmap decisions ahead of the core capture, wiki readability, and contradiction-management flows.

---

## Leave Off Or De-Prioritize

These are currently the wrong bets for a DM-first roadmap.

- advanced fantasy calendar systems
- novel-writing-first workflows
- public community/social features
- monetization and patron gating
- heavy collaboration and multi-user editing
- full VTT integration as a near-term priority
- map creation tooling if it grows into a separate product pillar

Important nuance:

"Leave off" does not mean "never support a basic version."

For example, a simple timeline is still valuable. A giant custom calendar engine is not.

---

## What The Wiki View Is Really For

The desired wiki view is not just cosmetic. It serves six jobs:

1. Make lore fast to scan during play
2. Make metadata and relations obvious at a glance
3. Make the world feel coherent and tangible
4. Make the product motivating enough to use regularly
5. Present selected canon cleanly to players
6. Reduce the cognitive load of switching between notes, references, and mental context

This means the target is closer to:

- Fandom-style article readability
- clear infoboxes
- obvious links to adjacent entities
- visible hierarchy and related pages
- strong separation between player-safe and GM-only information

And less like:

- authoring for long-form prose manuscripts
- blog-like public publishing workflows
- creator community profile pages

---

## Suggested V1 Product Surfaces

If AllCodex had to win with only three major surfaces, these are the strongest candidates based on the current discovery.

### 1. Brain Dump Inbox

The raw-thought capture point.

It should accept messy text quickly and let AI:

- detect note/entity types
- suggest structure
- extract relationships
- fill infobox metadata
- link to existing canon
- identify possible duplicates
- flag likely contradictions

### 2. Wiki Article View

The default way to consume lore.

It should emphasize:

- readable article body
- strong infobox/metadata panel
- visible outgoing and incoming relationships
- clear hierarchy breadcrumbs or tree position
- player-safe vs GM-only visibility cues

### 3. Live Session Workspace

The runtime DM surface.

It does not need to be a huge dashboard on day one, but it should optimize:

- fast lookup of NPCs, locations, hooks, and statblocks
- quick capture of new canon mid-session
- contradiction warnings or related-lore prompts
- recent recap and ongoing session context

---

## Concrete Feature Decisions From Current Discovery

These are the most defensible calls using the information gathered so far.

| Feature area | Decision | Why |
|---|---|---|
| Brain Dump | Keep and center the product around it | This is the original product spark and the clearest differentiator |
| Rich wiki article view | Keep and improve aggressively | It directly supports GM scanning, motivation, and player-safe presentation |
| Relationship graph | Keep | It strengthens mental model and continuity |
| Timeline | Keep, but favor a simple useful version first | Chronology matters; giant calendar engines do not |
| Map embeds | Keep in lightweight form | Useful for context and navigation, but should not become a map-editor product by default |
| Advanced calendars | Leave off | Too much complexity for too little DM value relative to simpler timeline support |
| Quest / hook tracking | Keep | Strong DM workflow value |
| Session recap | Keep | Directly supports continuity and session prep |
| Rules / statblock viewing | Keep | Important table utility, but likely secondary to lore and continuity |
| NPC generation | Keep | Direct support for improvisation |
| Scratchpad system | Maybe later | Valuable, but lightweight capture may already cover most of the need |
| Standalone lore Q&A chat | Maybe later | Useful, but should probably be embedded into search and Brain Dump before becoming its own surface |
| Public share links | Maybe later | Nice to have, but not central unless player consumption becomes a bigger priority |
| Campaign dashboard / DM screen | Maybe later, likely after the core three surfaces stabilize | Useful, but easy to overbuild too early |

---

## Product Risks To Avoid

These are the main ways AllCodex could become impressive but misaligned.

1. Spending too much effort on writer or novelist workflows
2. Building polished public publishing features before fixing DM friction
3. Overbuilding calendars, ACLs, monetization, or collaboration before the solo-GM loop feels excellent
4. Letting AI produce more data without improving retrieval and trust
5. Treating the wiki redesign as merely visual instead of information architecture work

---

## Immediate Roadmap Implications

If this DM-first framing is correct, the near-term roadmap should bias toward:

1. Brain Dump contract and UX stabilization
2. Better article/read view and metadata presentation
3. Real hierarchy and relationship navigation
4. Faster quick-capture and quick-create flows
5. Contradiction detection integrated into capture and session workflows
6. Session recap, quest tracking, and lightweight timeline support
7. Basic player-safe sharing after the GM loop is strong

And it should de-prioritize:

1. advanced calendars
2. novelist/manuscript-centric tooling
3. community/social product features
4. monetization systems
5. deep collaboration/ACL work

---

## Open Questions Still Worth Answering Later

The current discovery is already enough to guide roadmap decisions, but these details still affect execution:

1. Whether player-facing pages should be the same GM pages with secrets stripped or a separate presentation mode
2. Whether maps should stop at embeds and pins or grow into interactive map navigation
3. Whether rules/statblocks are just a fast reference layer or should become editable/importable system content
4. Whether contradiction prevention should happen live, after save, or in a session-end review loop
5. Whether lore Q&A belongs as a standalone chat surface or as search/Brain Dump augmentation only

These questions change implementation details, but they do not change the high-level product identity above.
