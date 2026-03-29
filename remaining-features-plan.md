# AllKnower: Remaining Features

**Status as of 2026-03-29:** One feature remaining — Azgaar map import. Features 1 (relation writing) and 2 (auto-apply relations) are fully shipped.

---

## ✅ Feature 1: Relation Writing Back to AllCodex — **SHIPPED**

`POST /suggest/relationships/apply` persists approved suggestions as AllCodex `relation` attributes (bidirectional by default). Applied relations are logged to `relation_history`. Implemented in `src/routes/suggest.ts` + `src/etapi/client.ts`.

---

## ✅ Feature 2: Auto-Applying Suggested Relations — **SHIPPED**

Brain dump pipeline (`src/pipeline/brain-dump.ts`) now calls `suggestRelationsForNote()` and `applyRelations()` from `src/pipeline/relations.ts` after each note is created. High-confidence suggestions are auto-applied. Medium/low are returned for manual review. Controlled by `autoRelate: boolean` request param (default `true`).

---

## Feature 3: Azgaar Fantasy Map Generator Import

**What:** A new endpoint that accepts an Azgaar FMG JSON export and bulk-creates Location (and optionally Faction) notes in AllCodex via ETAPI.

**Why:** Azgaar exports contain structured data for every burg (settlement), state (nation), river, and region. That can be hundreds of entries that would take hours to create by hand.

### Route

```
POST /import/azgaar
Content-Type: multipart/form-data
Body: file (the .json export from Azgaar FMG)
```

### Azgaar JSON Structure (relevant fields)

```json
{
  "info": { "mapName": "All Reach", ... },
  "burgs": [
    { "i": 1, "name": "Solara", "state": 3, "x": 540.2, "y": 312.8,
      "population": 28400, "capital": 1, "type": "Large City" }
  ],
  "states": [
    { "i": 3, "name": "Übermenschreich", "capital": 1, "color": "#4a7c59",
      "type": "Kingdom", "center": 8842, "diplomacy": [...] }
  ],
  "rivers": [
    { "i": 1, "name": "River Lume", "mouth": 4521, "source": 7832 }
  ],
  "cells": { "biome": [...], "culture": [...] }
}
```

### Implementation Steps

- [ ] **3a.** Create `src/pipeline/azgaar.ts`:
  - Define TypeScript types for the Azgaar JSON subset we consume (burgs, states, rivers — skip cells/biomes for now):
    - `AzgaarBurg` — `{ i, name, state, x, y, population, capital, type }`
    - `AzgaarState` — `{ i, name, capital, type, diplomacy }`
    - `AzgaarRiver` — `{ i, name, mouth, source }`
  - `parseAzgaarExport(json: unknown): { burgs, states, rivers }` — validates and extracts relevant arrays
  - `importBurgs(burgs, stateMap, loreRootNoteId)` — for each burg:
    1. `createNote({ parentNoteId, title: burg.name, type: "text", content: generated HTML })`
    2. `tagNote(noteId, "lore")` + `tagNote(noteId, "loreType", "location")`
    3. `createAttribute` for promoted fields: `locationType`, `region` (from state name), `population`, `ruler`
    4. `createAttribute({ type: "label", name: "geolocation", value: "${burg.x},${burg.y}" })` for map pins
    5. `setNoteTemplate(noteId, TEMPLATE_ID_MAP.location)` (best-effort, already try/catch'd)
  - `importStates(states, loreRootNoteId)` — for each state:
    1. Create as faction note
    2. Tag `loreType: faction`, set `factionType` to state type
    3. Link capital burg via relation (if burg note was already created)
  - `importRivers(rivers, loreRootNoteId)` — optional, creates location notes with `locationType: "river"`

- [ ] **3b.** Create `src/routes/import.ts`:
  - `POST /import/azgaar` accepts multipart form-data with a single JSON file field
  - Parse JSON, call `parseAzgaarExport`
  - Call `importBurgs`, `importStates`, and optionally `importRivers`
  - Return `{ burgsCreated, statesCreated, riversCreated, skipped, errors }`

- [ ] **3c.** Register route in `src/index.ts`.

- [ ] **3d.** Add `ImportHistory` model to Prisma schema:
  ```prisma
  model ImportHistory {
    id           String   @id @default(cuid())
    source       String   // "azgaar"
    fileName     String
    notesCreated String[]
    summary      Json
    createdAt    DateTime @default(now())
    @@map("import_history")
  }
  ```

- [ ] **3e.** Add `azgaarImportParentNoteId` to `AppConfig` — the AllCodex note under which imported locations land. Defaults to lore root.

- [ ] **3f.** Add a `dryRun: boolean` query param. When true, parses and validates the JSON but doesn't write anything to AllCodex. Returns what would be created.

### Generated HTML Template for Burgs

```html
<h2>{burg.name}</h2>
<p><strong>Type:</strong> {burg.type}</p>
<p><strong>State:</strong> {stateName}</p>
<p><strong>Population:</strong> {burg.population.toLocaleString()}</p>
<p>{burg.capital ? "Capital city of " + stateName : ""}</p>
```

### Files Touched

| File | Change |
|---|---|
| `src/pipeline/azgaar.ts` | New — parser + importers |
| `src/routes/import.ts` | New — `POST /import/azgaar` |
| `src/index.ts` | Register `importRoute` |
| `prisma/schema.prisma` | Add `ImportHistory` model |

---

## Implementation Order

Features 1 and 2 are complete. Only Feature 3 (Azgaar import) remains.

```
Feature 3: Azgaar import    (no dependencies — standalone)
```

---

## Prisma Migrations

Feature 3 adds one new model. Run after all schema changes:

```bash
bunx prisma migrate dev --name "add-import-history"
```

New model:
- `ImportHistory` for logging Azgaar imports (Feature 3)
