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

## 5. Zero-Login Auto-Provisioning
**Previous State:** First-time users had to manually register an AllKnower account, log in, then separately connect AllCodex Core credentials in Settings before any AI features would work.
**v1 State:** A three-stage bootstrap chain eliminates all manual setup:
1. **AllCodex Core** auto-sets its password from the `ALLCODEX_PASSWORD` env var at startup.
2. **AllKnower** runs a bootstrap sequence on startup: creates a default user via `better-auth` sign-up, obtains an ETAPI token from Core via password auth, and stores it as an encrypted `UserIntegration` for that user.
3. **Portal middleware** detects the absence of an `allknower_token` cookie on each request and calls `POST /internal/auto-provision` on AllKnower to obtain a session for the first user, setting it as an HTTP-only cookie.

The result is zero-click operation: the first browser visit auto-provisions all credentials. Manual login/register flows remain available for multi-user scenarios.

## 6. Code Quality & Release Gates
**Previous State:** The repository had several failing typechecks in `allcodex-portal` and `allcodex-core`, alongside broken test mocks in `allknower` due to evolving API signatures.
**v1 State:** 
- Successfully threaded the new `credentials?: EtapiCredentials` parameter through all necessary functions and test mocks.
- `bun run check` (TypeScript + 121 tests) passes flawlessly in `allknower`.
- `bun run check` (TypeScript + 258 tests) and `bun run build` pass in `allcodex-portal`.
- `pnpm typecheck` and `pnpm test:all` (1028 tests) pass in `allcodex-core`.

## 7. V1 Code Review Remediation
**35 findings** from a full-stack code review were audited and resolved across all three services. 32 were already fixed during v1 development; 3 required new commits:

- **AllKnower:** Deleted dead credential modules (`core.ts`, `crypto.ts`, migration script — 187 lines removed). Scoped `/config/wipe` deleteMany calls by `session.user.id` instead of wiping all users' data. Threaded per-user credentials through the setup/seed-templates route.
- **Portal:** Fixed `putNoteContent` Content-Type from `text/html` to `text/plain` — Core's `express.text()` only parses `text/plain`; the wrong header caused `req.body = null` and 500 errors on note saves.
- **Core:** E2E test suite overhauled — 18 stale Trilium client UI specs deleted, 7 new ETAPI tests added (11 total, all passing). Share index now filters `#draft`/`#gmOnly`/protected notes. `shaca_mocking.ts` wires SBranch for parent-child test assertions.

## 8. Testing Infrastructure
**Previous State:** Core had 59 e2e specs, 56 of which were stale Trilium client UI tests that failed on every run. Unit test coverage was not tracked. Portal had ~140 unit tests.
**v1 State:**
- **Core:** 11 e2e tests (all ETAPI-based, no browser UI), 1028 unit tests across 87 files. Coverage baseline: 45.43% statements, 39.84% branches, 50.24% functions.
- **AllKnower:** 121+ tests across 5 test groups (run per-directory to avoid mock contamination).
- **Portal:** 258 unit tests across 45 files, plus ~150 Playwright e2e tests across 28 specs.