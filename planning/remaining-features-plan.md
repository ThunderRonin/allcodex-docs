# AllKnower: Remaining Features

**Status as of 2026-04-03:** All three features are shipped. Feature 3 (Azgaar map import) was the last to complete — the pipeline, route, and Portal proxy are all live.

> **Infrastructure note:** `src/routes/import.ts` contains both `POST /import/system-pack` and `POST /import/azgaar` (with preview support). `src/pipeline/azgaar.ts` handles parsing and note creation.

---

## ✅ Feature 1: Relation Writing Back to AllCodex — **SHIPPED**

`POST /suggest/relationships/apply` persists approved suggestions as AllCodex `relation` attributes (bidirectional by default). Applied relations are logged to `relation_history`. Implemented in `src/routes/suggest.ts` + `src/etapi/client.ts`.

---

## ✅ Feature 2: Auto-Applying Suggested Relations — **SHIPPED**

Brain dump pipeline (`src/pipeline/brain-dump.ts`) now calls `suggestRelationsForNote()` and `applyRelations()` from `src/pipeline/relations.ts` after each note is created. High-confidence suggestions are auto-applied. Medium/low are returned for manual review. Controlled by `autoRelate: boolean` request param (default `true`).

---

## ✅ Feature 3: Azgaar Fantasy Map Generator Import — **SHIPPED**

Implemented in `src/pipeline/azgaar.ts` (parser + note creators) and `src/routes/import.ts` (`POST /import/azgaar`, `POST /import/azgaar/preview`, plus a URL-preview stub). Portal proxy at `app/api/import/azgaar/route.ts`. Import page UI supports drag-and-drop, client-side parsing, entity preview, duplicate-skip, import options, and result reporting.

**What:** An endpoint that accepts parsed Azgaar FMG JSON and bulk-creates AllCodex lore notes via ETAPI.

**Why:** Azgaar exports contain structured data for settlements, nations, religions, cultures, and map notes. That can be hundreds of entries that would take hours to create by hand.

### Route Contracts

```http
POST /import/azgaar/preview
Content-Type: application/json
```

```json
{ "mapData": { "info": { "mapName": "All Reach" }, "pack": { "burgs": [], "states": [] } } }
```

```http
POST /import/azgaar
Content-Type: application/json
```

```json
{
  "mapData": { "info": { "mapName": "All Reach" }, "pack": { "burgs": [], "states": [] } },
  "parentNoteId": "root",
  "options": {
    "importStates": true,
    "importBurgs": true,
    "importReligions": true,
    "importCultures": true,
    "importNotes": true,
    "skipDuplicates": true
  }
}
```

The Portal accepts `.map`/JSON files in the browser, parses them client-side, and sends the parsed JSON to `/api/import/azgaar`. The Portal route does not send multipart data to AllKnower.

### Current Entity Mapping

| Azgaar data | AllCodex note type | Template |
|---|---|---|
| `pack.states[]` | faction | `_template_faction` |
| `pack.burgs[]` | location | `_template_location` |
| `pack.religions[]` | religion | `_template_religion` |
| `pack.cultures[]` | race | `_template_race` |
| `notes[]` | location | `_template_location` |

Each created note gets `#lore`, `#loreType`, and `#importSource=azgaar`. Location-like entries include `#geolocation` when coordinates are available. Duplicate detection is title-based when `skipDuplicates` is enabled.

### Result Shape

```json
{
  "mapName": "All Reach",
  "totals": { "created": 12, "skipped": 3, "errors": 0 },
  "states": { "created": [], "skipped": [], "errors": [] },
  "burgs": { "created": [], "skipped": [], "errors": [] },
  "religions": { "created": [], "skipped": [], "errors": [] },
  "cultures": { "created": [], "skipped": [], "errors": [] },
  "notes": { "created": [], "skipped": [], "errors": [] }
}
```

### Files Touched

| File | Change |
|---|---|
| `src/pipeline/azgaar.ts` | Parser, preview, entity mapping, note creation |
| `src/routes/import.ts` | `POST /import/azgaar`, `POST /import/azgaar/preview`, URL-preview stub |
| `app/api/import/azgaar/route.ts` | Portal proxy for preview and full import |
| `app/(portal)/import/page.tsx` | Upload, preview, options, and result UI |

---

## Implementation Order

All three features are complete.

```
Feature 1: Relation Writing    ✅ shipped
Feature 2: Auto-Apply          ✅ shipped
Feature 3: Azgaar import       ✅ shipped
```

---

## Persistence Notes

Feature 3 did not ship an `ImportHistory` model or Prisma migration. Import result reporting is request-scoped: AllKnower returns the created/skipped/error buckets to the Portal, and the Portal displays them immediately. Persistent import history remains future work if it becomes useful.
