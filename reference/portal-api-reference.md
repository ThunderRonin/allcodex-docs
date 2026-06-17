# AllCodex Portal — API Route Reference

> Complete reference for all Portal Next.js API routes.
> The Portal acts as a proxy; all backend calls happen server-side.
> The browser does not receive AllCodex ETAPI tokens or AllKnower Bearer tokens.

Date: April 27, 2026

---

## Table of Contents

- [Credential Flow](#credential-flow)
- [Lore CRUD](#lore-crud)
- [Note Attributes & Relationships](#note-attributes--relationships)
- [Note Content & Metadata](#note-content--metadata)
- [Lore Tree Operations](#lore-tree-operations)
- [Search & Discovery](#search--discovery)
- [Brain Dump](#brain-dump)
- [AI Tools](#ai-tools)
- [RAG](#rag)
- [Content Collections](#content-collections)
- [Share Settings](#share-settings)
- [Configuration & Auth](#configuration--auth)
- [Import](#import)
- [Notifications](#notifications)
- [Usage & Budgets](#usage--budgets)
- [Key Patterns](#key-patterns)

---

## Credential Flow

All backend calls resolve credentials from HTTP-only cookies via `getEtapiCreds()` and `getAkCreds()` (defined in `lib/get-creds.ts`). If cookies are not set, the route falls back to environment variables in `.env.local`. The browser does not receive tokens.

```
Browser  →  POST /api/auth/login or /api/config/allknower-login  (AllKnower: email/password → Bearer token)
         →  HTTP-only cookies set (allknower_url, allknower_token)
         →  POST /api/integrations/allcodex/connect  (Core: password → ETAPI token,
              stored as the user's encrypted UserIntegration inside AllKnower)
         →  Subsequent /api/* calls read the AllKnower token from cookies; Core creds
              are resolved server-side per-user by AllKnower
```

> **Note:** In zero-login/dev deployments, the auto-provisioning middleware sets the AllKnower token cookie, and Core credentials are bootstrapped into the default user's `UserIntegration`. Neither step is required for the first user. `getEtapiCreds()` resolves Core credentials per-user via AllKnower (`resolveAllCodexCredentials`, using the AllKnower token + `PORTAL_INTERNAL_SECRET`); in non-production it falls back to the `ALLCODEX_URL` / `ALLCODEX_ETAPI_TOKEN` env vars. There is no `allcodex_token` browser cookie in the v1 flow.

---

## Lore CRUD

| Method | Path | Description | Backend |
|--------|------|-------------|---------|
| `GET` | `/api/lore` | Search lore notes. Default: `?q=#lore` (all lore). Supports full ETAPI search syntax. | AllCodex ETAPI |
| `POST` | `/api/lore` | Create a new lore note. | AllCodex ETAPI |
| `GET` | `/api/lore/[id]` | Get note metadata + attributes. | AllCodex ETAPI |
| `PATCH` | `/api/lore/[id]` | Update note properties (title, type, mime). | AllCodex ETAPI |
| `DELETE` | `/api/lore/[id]` | Delete a note. Returns 204. | AllCodex ETAPI |

### POST `/api/lore` — Create Note

```json
{
  "title": "Lord Ashvane",
  "type": "text",
  "mime": "text/html",
  "loreType": "character",
  "templateId": "_template_character",
  "parentNoteId": "root",
  "attributes": { "fullName": "Lord Ashvane of Solara", "race": "Human" }
}
```

---

## Note Attributes & Relationships

| Method | Path | Description | Backend |
|--------|------|-------------|---------|
| `POST` | `/api/lore/[id]/attributes` | Create a label or relation attribute. | AllCodex ETAPI |
| `DELETE` | `/api/lore/[id]/attributes?attrId=<ID>` | Delete an attribute by ID. Returns 204. | AllCodex ETAPI |
| `POST` | `/api/lore/[id]/relationships` | Get existing relations + automatically suggested connections for a note. | AllKnower + AllCodex ETAPI |

### POST `/api/lore/[id]/attributes` — Create Attribute

```json
{ "type": "label", "name": "fullName", "value": "Lord Ashvane" }
```

```json
{ "type": "relation", "name": "~belongsTo", "value": "<targetNoteId>" }
```

---

## Note Content & Metadata

| Method | Path | Description | Backend |
|--------|------|-------------|---------|
| `GET` | `/api/lore/[id]/content` | Get the HTML content of a note. Returns `text/html`. | AllCodex ETAPI |
| `PUT` | `/api/lore/[id]/content` | Update the HTML content of a note. Body: raw HTML string. Returns 204. | AllCodex ETAPI |
| `GET` | `/api/lore/[id]/preview?mode=gm\|player` | Get sanitized HTML content. `gm` mode uses `sanitizeLoreHtml()`, `player` mode uses `sanitizePlayerView()` (strips `gmOnly` content). | AllCodex ETAPI |
| `GET` | `/api/lore/[id]/image` | Proxy image binary from AllCodex. Cached 1 day. | AllCodex ETAPI |
| `GET` | `/api/images/[id]/[filename]` | Redirect compatibility route for editor image URLs. Forwards to `/api/lore/[id]/image`. | None (local redirect) |
| `GET` | `/api/lore/[id]/backlinks` | Get notes that link to this note (inbound relations + content links). | AllCodex ETAPI |
| `GET` | `/api/lore/[id]/breadcrumbs` | Get the ancestor branch path for breadcrumb navigation. | AllCodex ETAPI |
| `GET` | `/api/lore/[id]/revisions` | List a note's revision history. | AllCodex ETAPI |
| `GET` | `/api/lore/[id]/revisions/[revId]/content` | Get the content of a specific revision. | AllCodex ETAPI |
| `GET` | `/api/lore/[id]/graph` | Get the relationship graph (existing relations + traversal) for a note. | AllKnower |
| `GET` | `/api/lore/[id]/relationship-history` | Get the applied-relationship history for a note. | AllKnower |
| `GET` | `/api/lore/[id]/map` | Build map data for a note: map image URL, dimensions, and geolocation pins derived from child notes. | AllCodex ETAPI |
| `POST` | `/api/lore/[id]/map/upload` | Upload a map image for a note. | AllCodex ETAPI |

---

## Lore Tree Operations

| Method | Path | Description | Backend |
|--------|------|-------------|---------|
| `POST` | `/api/lore/move` | Move a note to a different parent branch. | AllCodex ETAPI |
| `POST` | `/api/lore/upload-image` | Upload an image file and create an AllCodex image note. Returns `{ url, noteId }`. | AllCodex ETAPI |

### POST `/api/lore/move`

```json
{ "noteId": "<noteId>", "newParentId": "<parentId>", "index": 0 }
```

### POST `/api/lore/upload-image`

Binary image body. Set `x-vercel-filename` header for the file name.

---

## Search & Discovery

| Method | Path | Description | Backend |
|--------|------|-------------|---------|
| `GET` | `/api/lore/mention-search?q=<text>` | Autocomplete for `@`-mentions. Min 2 chars. Returns up to 8 results. | AllCodex ETAPI + AllKnower |
| `GET` | `/api/lore/note-search?q=<text>&type=<type>` | Title/type note search for picker-style UI. Min 2 chars. Returns up to 12 results. | AllCodex ETAPI |
| `POST` | `/api/lore/autolink` | Scan a text block for lore title matches. Cached 60s. | AllCodex ETAPI |
| `GET` | `/api/search?q=<query>&mode=etapi\|rag` | Dual-mode search. `etapi` = ETAPI full-text/attribute, `rag` = AllKnower semantic search. | AllCodex ETAPI or AllKnower |

### POST `/api/lore/autolink`

```json
{ "text": "Lord Ashvane marched to Solara with the Blackthorn Company." }
```

Returns:
```json
{ "matches": [
  { "term": "Lord Ashvane", "noteId": "abc123", "title": "Lord Ashvane" },
  { "term": "Solara", "noteId": "def456", "title": "Solara" }
] }
```

---

## Brain Dump

| Method | Path | Description | Backend |
|--------|------|-------------|---------|
| `POST` | `/api/brain-dump` | Process raw text through the extraction pipeline. | AllKnower |
| `POST` | `/api/brain-dump/stream` | SSE-streamed auto brain dump (progressive entity cards, tokens, done). | AllKnower |
| `POST` | `/api/brain-dump/commit` | Commit reviewed entities from review mode. | AllKnower |
| `GET` | `/api/brain-dump/history` | List the 20 most recent brain dumps. | AllKnower |
| `GET` | `/api/brain-dump/history/[id]` | Get full detail for a single brain dump entry. | AllKnower |
| `GET` | `/api/brain-dump/[id]/diffs` | Per-note content diffs (revision history) produced by a brain dump. | AllKnower |
| `POST` | `/api/brain-dump/batch` | Submit a batch (bulk) of brain dumps for background processing. | AllKnower |
| `GET` | `/api/brain-dump/batch/[batchId]` | Batch processing status. | AllKnower |
| `DELETE` | `/api/brain-dump/batch/[batchId]` | Cancel queued jobs in a batch. | AllKnower |

### POST `/api/brain-dump`

```json
{
  "rawText": "Lord Ashvane is a human noble who rules Solara...",
  "mode": "auto"
}
```

**Modes:**
- `auto` — extract and write entities immediately
- `review` — extract entities and return as proposals for user approval
- `inbox` — queue the raw text for later processing

### POST `/api/brain-dump/commit`

```json
{
  "rawText": "Lord Ashvane is a human noble...",
  "approvedEntities": [
    { "title": "Lord Ashvane", "type": "character", "action": "create", "content": "<p>...</p>" }
  ]
}
```

---

## AI Tools

| Method | Path | Description | Backend |
|--------|------|-------------|---------|
| `POST` | `/api/ai/consistency` | Run a RAG-augmented consistency scan. | AllKnower |
| `GET\|POST` | `/api/ai/gaps` | Detect underdeveloped lore areas. POST preferred (avoids caching). | AllKnower |
| `POST` | `/api/ai/relationships` | Get relationship suggestions for text/note. | AllKnower |
| `PUT` | `/api/ai/relationships` | Apply suggested relationships (persists as AllCodex relation attributes). | AllKnower |
| `POST` | `/api/lore/[id]/copilot/chat` | Article copilot turn — sends conversation transcript + current note context, returns assistant message + optional proposal. | AllKnower |
| `POST` | `/api/lore/[id]/copilot/stream` | SSE-streamed article copilot turn (status, token, reasoning, result, done). | AllKnower |
| `POST` | `/api/lore/[id]/copilot/apply` | Apply an approved copilot proposal (create/update notes, labels, relations). Returns `{ updatedNoteIds, createdNoteIds, skipped, failed }`. | AllCodex ETAPI |

### POST `/api/ai/consistency`

```json
{ "noteIds": ["abc123", "def456"] }
```

Optional `noteIds` — if omitted, AllKnower uses semantic sampling mode.

### PUT `/api/ai/relationships` — Apply

```json
{
  "sourceNoteId": "abc123",
  "relations": [
    { "targetNoteId": "def456", "type": "rulerOf", "description": "Lord Ashvane rules Solara" }
  ],
  "bidirectional": true
}
```

---

## RAG

| Method | Path | Description | Backend |
|--------|------|-------------|---------|
| `GET` | `/api/rag?text=<query>&topK=10` | Semantic similarity search against the lore vector index. | AllKnower |
| `POST` | `/api/rag` | Same, via POST body. | AllKnower |

---

## Content Collections

These routes query AllCodex ETAPI for notes with specific labels.

| Method | Path | Description | Filter |
|--------|------|-------------|--------|
| `GET` | `/api/quests` | List all quest notes. | `#quest` label |
| `GET` | `/api/statblocks` | List all statblock notes. | `#statblock` label |
| `GET` | `/api/timeline` | List events and timelines. | `#loreType=event` OR `#loreType=timeline` |

---

## Share Settings

| Method | Path | Description | Backend |
|--------|------|-------------|---------|
| `GET` | `/api/share` | Get the current share root configuration. | AllCodex ETAPI |
| `PUT` | `/api/share` | Set a note as the share root (public entry point). | AllCodex ETAPI |
| `GET` | `/api/share/tree` | List all lore notes with share visibility flags. | AllCodex ETAPI |

### GET `/api/share/tree` — Response

Each note includes: `noteId`, `title`, `isDraft`, `isGmOnly`, `shareAlias`, `isProtected`, `isShared`.

---

## Configuration & Auth

| Method | Path | Description | Backend |
|--------|------|-------------|---------|
| `POST` | `/api/auth/login` | AllKnower email/password login → sets Bearer token cookie. | AllKnower (better-auth) |
| `POST` | `/api/auth/register` | Register AllKnower account → sets token cookie. | AllKnower (better-auth) |
| `POST` | `/api/auth/logout` | Sign out, clear AllKnower session cookie. | AllKnower (better-auth) |
| `GET` | `/api/auth/session` | Verify AllKnower session, return user. | AllKnower (better-auth) |
| `POST` | `/api/config/allknower-login` | AllKnower email/password login → sets Bearer token cookie. | AllKnower (better-auth) |
| `POST` | `/api/config/allknower-register` | Register AllKnower account → sets token cookie. | AllKnower (better-auth) |
| `POST` | `/api/integrations/allcodex/connect` | Core password (or pre-acquired token) → obtains ETAPI token → stores it as the user's encrypted `UserIntegration` in AllKnower (no `allcodex_token` browser cookie). | AllCodex ETAPI + AllKnower |
| `GET` | `/api/integrations/allcodex/status` | Whether a Core integration is connected for the user. | AllKnower |
| `DELETE` | `/api/integrations/allcodex` | Remove the user's stored Core integration. | AllKnower |
| `DELETE` | `/api/config/disconnect?service=allcodex\|allknower\|all` | Clear stored credentials. | None (local) |
| `GET` | `/api/config/models` | List configured/available parser models per task. | AllKnower |
| `POST` | `/api/config/wipe` | Dangerous: wipe the user's lore + RAG state (LanceDB table + Prisma tracking). | AllKnower |
| `GET` | `/api/config/portal` | Get portal config (lore root note ID). | None (local) |
| `PUT` | `/api/config/portal` | Set portal config. | None (local) |
| `GET` | `/api/config/status` | Check connectivity to AllCodex and AllKnower. | Both |

### POST `/api/integrations/allcodex/connect`

```json
{ "baseUrl": "http://localhost:8080", "password": "your-password" }
```

### POST `/api/auth/login`

```json
{ "url": "http://localhost:3001", "email": "gm@example.com", "password": "secret" }
```

---

## Import

| Method | Path | Description | Backend |
|--------|------|-------------|---------|
| `POST` | `/api/import/system-pack` | Import a system pack (SRD, monster manual, etc.) | AllKnower |
| `POST` | `/api/import/azgaar` | Import parsed Azgaar Fantasy Map Generator JSON. Body: `{ mapData, parentNoteId?, options? }`. Use `?action=preview` to preview entities without creating notes; omit the parameter to execute the full import. | AllKnower |

The system pack request body is the full system pack JSON. AllKnower creates statblock notes with `#statblock` labels, skips duplicates by title, and returns created/skipped counts.

The Azgaar import page accepts `.map`/JSON files in the browser, parses them client-side, and sends the parsed JSON to this route. Preview mode returns a summary of entities that would be created without writing anything. Full import creates location, faction, religion, race, and map-note entries, skips duplicates by title when enabled, and returns per-entity created/skipped/error buckets.

---

## Notifications

Web-push notifications (e.g. brain dump completion).

| Method | Path | Description | Backend |
|--------|------|-------------|---------|
| `GET` | `/api/notifications/vapid-public-key` | Get the VAPID public key for push subscription. | AllKnower |
| `POST` | `/api/notifications/subscribe` | Register a browser push subscription. | AllKnower |
| `DELETE` | `/api/notifications/unsubscribe` | Remove a push subscription. | AllKnower |

---

## Usage & Budgets

Token/cost usage dashboard and per-user token budgets.

| Method | Path | Description | Backend |
|--------|------|-------------|---------|
| `GET` | `/api/usage/summary` | Token/cost usage summary (per day, task, model). | AllKnower |
| `GET` | `/api/usage/budgets` | Get the user's token-budget thresholds. | AllKnower |
| `PUT` | `/api/usage/budgets` | Set the user's token-budget thresholds. | AllKnower |
| `GET` | `/api/usage/alert-status` | Whether the user is over budget / nearing the alert threshold. | AllKnower |

---

## Key Patterns

1. **Proxy architecture**: Every `/api/*` route is a thin proxy. No domain logic lives in the Portal.

2. **Error handling**: Routes call `notConfigured()` when backend credentials are missing and `handleRouteError()` for exceptions. The `ServiceError` class wraps upstream errors.

3. **HTML sanitization**: Content routes use `sanitizeLoreHtml()` (DOMPurify) for GM view and `sanitizePlayerView()` for player-safe output.

4. **Caching**: Image proxy caches for 1 day. Mention search caches the title list for 60 seconds.

5. **Cookie-only secrets**: All tokens are stored in HTTP-only cookies with `secure=true` in production. Browser JavaScript does not have access to backend credentials.

6. **Dual backend**: Routes proxy to either AllCodex ETAPI (lore CRUD, search, content) or AllKnower (parsing features, brain dump, RAG, import).
