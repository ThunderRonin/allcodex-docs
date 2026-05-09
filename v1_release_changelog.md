# AllCodex v1 Release Summary

This document outlines the major architectural and functional changes accomplished for the AllCodex v1 release, highlighting the transition from the pre-v1 state to the stabilized v1 production state.

## 1. Architectural Boundary & Security
**Previous State:** The browser was storing the `allcodex_token` directly in cookies via legacy configuration endpoints, leaking backend credentials to the client.
**v1 State:** A strict integration boundary is now enforced: `Browser -> Portal -> AllKnower -> Core`. The Portal acts as the sole browser-facing BFF (Backend for Frontend). Core ETAPI tokens are generated server-side, passed to AllKnower for secure storage, and never reach browser JavaScript.

## 2. User-Scoped Credentials
**Previous State:** The application relied on a global `AppConfig` to store a single set of Core credentials, making multi-user support impossible and insecure.
**v1 State:** Introduced the `UserIntegration` Prisma model in AllKnower. Core credentials are now securely encrypted at rest and scoped per-user. All AI routes, ETAPI calls, RAG indexing, and background pipelines strictly resolve credentials using the authenticated `session.user.id`. 

## 3. Relationship Graph & Apply Fixes
**Previous State:** 
- Applying AI-suggested relationships would map unknown types to a generic `relOther`.
- The system allowed duplicate relations to be written.
- The Portal UI failed to refresh after applying, leaving processed suggestions visible as if they were unapplied.
- Missing `targetTitle` fields from the AI response caused the graph to crash (`undefined.replace`).
**v1 State:** 
- **Backend:** `applyRelations` now enforces canonical mapping (e.g., `member_of` → `relMemberOf`), rejects unknown types, and safely skips duplicates. Returns explicit `applied`, `skipped`, and `failed` arrays.
- **Frontend:** The graph sanitization logic handles missing titles gracefully. The UI immediately invalidates the relationship query cache upon application, transitions dashed AI edges to solid existing edges, and visually reports any skipped or failed relations.

## 4. Dev/Debug Database Management
**Previous State:** The database wipe functionality was incomplete. Dropping the LanceDB vector table left orphaned metadata in Prisma, causing the dashboard to report phantom "Indexed RAG entities."
**v1 State:** Integrated a dangerous-ops "Wipe DB Lore & RAG" card into the Portal Settings UI. The `/config/wipe` route now comprehensively drops the LanceDB table AND clears all associated tracking state from Prisma, including `LoreSession`, `LlmCallLog`, `RagIndexMeta`, `BrainDumpHistory`, and `RelationHistory`.

## 5. Code Quality & Release Gates
**Previous State:** The repository had several failing typechecks in `allcodex-portal` and `allcodex-core`, alongside broken test mocks in `allknower` due to evolving API signatures.
**v1 State:** 
- Successfully threaded the new `credentials?: EtapiCredentials` parameter through all necessary functions and test mocks.
- `bun run check` (TypeScript + 121 tests) passes flawlessly in `allknower`.
- `bun run check` (TypeScript + 140 tests) and `bun run build` pass in `allcodex-portal`.
- `pnpm typecheck` passes cleanly in `allcodex-core`.