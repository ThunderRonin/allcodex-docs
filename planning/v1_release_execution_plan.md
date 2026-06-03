# AllCodex v1 Release Execution Plan

Date: 2026-05-06
Status: ready for implementation

This is the execution-ready companion to [v1_release_plan.md](./v1_release_plan.md). Use this file as the implementation checklist for v1. The source plan and audit remain in `v1_release_plan.md`; this document is the trimmed, decision-complete work plan.

For the final release-hit checklist, use [v1_final_hit_plan.md](./v1_final_hit_plan.md).

## Release Goal

Ship v1 with this service boundary:

```text
Browser -> Portal -> AllKnower -> Core
        -> Portal -> Core
```

Rules:

- The browser talks only to Portal.
- Portal is the only browser-facing BFF.
- Core and AllKnower are private backend services.
- Backend tokens are never returned to browser JavaScript.
- Core ETAPI tokens are integration credentials, not the app user model.
- AllKnower better-auth is the v1 identity/control-plane backend.
- Each signed-in user gets their own encrypted Core integration record.

Do not tag v1 until all release gates in this file pass.

## Phase 1: Stabilize Build and Destructive Routes

Goal: make the release branch safe to build and prevent dev-only data wipes from shipping.

Implementation:

- In `allcodex-portal/app/api/config/wipe/route.ts`, make the route return `404` unless `NODE_ENV !== "production"` and `ALLOW_DEV_WIPE === "true"`.
- Remove the import of non-exported `etapiFetch` from `@/lib/etapi-server`; either stop deleting Core lore from this route or use exported ETAPI helpers only after the dev gate passes.
- In `allknower/src/routes/config.ts`, gate `/config/wipe` with the same `NODE_ENV !== "production"` and `ALLOW_DEV_WIPE === "true"` condition before any LanceDB or Prisma deletion runs.
- Keep dirty-tree triage explicit in each submodule. Do not revert unrelated user changes.

Acceptance:

- `cd allcodex-portal && bun run build` no longer fails on `app/api/config/wipe/route.ts`.
- Production requests to Portal wipe route and AllKnower wipe route return `404`.
- Development wipe requires the explicit `ALLOW_DEV_WIPE=true` opt-in.

## Phase 2: Add Per-User Core Integration Storage

Goal: replace global Core credentials with encrypted user-scoped credentials in AllKnower.

Implementation:

- Add a Prisma model in `allknower/prisma/schema.prisma`:

```prisma
model UserIntegration {
  id             String   @id @default(cuid())
  userId         String
  provider       String
  baseUrl        String
  encryptedToken String
  tokenLast4     String?
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt

  user           User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([userId, provider])
  @@index([provider])
  @@map("user_integrations")
}
```

- Add `integrations UserIntegration[]` to `User`.
- Generate a Prisma migration.
- Add an encryption helper using AES-256-GCM with a random IV per encryption and auth tag storage.
- Add `INTEGRATION_CREDENTIALS_KEY` to `allknower/src/env.ts`.
- Production behavior: startup must fail if `INTEGRATION_CREDENTIALS_KEY` is missing or invalid.
- Test behavior: tests may provide an explicit deterministic test key.
- Never log plaintext tokens.
- Never return plaintext tokens from public status routes.

Acceptance:

- A saved Core ETAPI token is not present in plaintext in Postgres.
- Decrypt works only with the configured key.
- Missing production key fails startup before serving traffic.
- `cd allknower && bunx prisma generate` succeeds after the schema change.

## Phase 3: Add AllKnower Integration Service and Routes

Goal: make AllKnower the owner of per-user Core integration state.

Implementation:

- Add an integration service module in AllKnower that provides:
  - connect Core for authenticated user.
  - get safe Core connection status for authenticated user.
  - disconnect Core for authenticated user.
  - resolve decrypted Core credentials for authenticated user.
- Add authenticated public AllKnower routes:
  - `POST /integrations/allcodex/connect`
  - `GET /integrations/allcodex/status`
  - `DELETE /integrations/allcodex`
- Add an internal server-only route for Portal:
  - `POST /internal/integrations/allcodex/credentials`
- The internal route must require both:
  - `Authorization: Bearer <user session token>`
  - `X-Portal-Internal-Secret: <PORTAL_INTERNAL_SECRET>`
- The internal route returns plaintext `{ baseUrl, token }` only to Portal server code.
- Keep `/config/allcodex` only as a legacy/dev route or deprecate it. It must not power authenticated v1 user flows.

Acceptance:

- User A cannot read, overwrite, or disconnect User B's integration.
- Public status response includes connected state and safe metadata only.
- Public status response does not include plaintext token.
- Internal credential response requires the shared internal secret.
- Internal credential response requires the user's valid better-auth bearer token.

## Phase 4: Update Portal Auth and Core Connect Flow

Goal: make Portal the only browser-facing auth and integration surface.

Implementation:

- Add or normalize browser-safe Portal routes:
  - `POST /api/auth/register`
  - `POST /api/auth/login`
  - `POST /api/auth/logout`
  - `GET /api/auth/session`
  - `POST /api/integrations/allcodex/connect`
  - `GET /api/integrations/allcodex/status`
  - `DELETE /api/integrations/allcodex`
- Portal auth routes call AllKnower better-auth server-side and store the AllKnower session token in an HTTP-only cookie.
- Core connect route accepts Core URL and password, then calls `POST {coreUrl}/etapi/auth/login` server-side.
- Portal sends the generated Core ETAPI token to AllKnower `/integrations/allcodex/connect` for encrypted per-user storage.
- Portal must not store `allcodex_token` in browser cookies for v1.
- Settings UI should show:
  - signed-in state.
  - AllKnower session state.
  - Core connected/disconnected state.
  - safe Core URL if needed.
  - no raw backend tokens.

Acceptance:

- Browser-visible JSON responses never include AllKnower bearer token or Core ETAPI token.
- HTTP-only cookie may contain the AllKnower session token.
- No v1 user flow writes `allcodex_token` to browser cookies.
- Core password is never persisted.
- Disconnect removes the user's Core integration from AllKnower.

## Phase 5: Refactor Core Credential Resolution

Goal: ensure every user-facing Core read/write uses the current user's Core integration.

Implementation:

- In Portal, update `allcodex-portal/lib/get-creds.ts` so `getEtapiCreds()` resolves Core credentials from AllKnower's internal credential route using the current signed-in user's HTTP-only AllKnower session cookie.
- Keep environment fallback only for clearly marked local/dev flows if needed.
- In AllKnower, refactor `allknower/src/etapi/client.ts` to support explicit credentials instead of global `AppConfig` credential lookup for authenticated user flows.
- Update AllKnower AI/Core routes to resolve credentials by `session.user.id` before Core operations:
  - brain dump.
  - brain dump commit.
  - relationship apply.
  - gap detector.
  - consistency checker.
  - RAG reindex/indexer paths that fetch Core content.
  - import/setup paths if exposed to authenticated v1 users.
- Authenticated user flows must not fall back to `AppConfig` `allcodexUrl` or `allcodexToken`.

Acceptance:

- With no Core integration, user-facing Core-dependent routes fail clearly with a reconnect message.
- With User A connected and User B disconnected, User B cannot use User A's Core token.
- With User A and User B connected to different Core URLs, each user reaches only their own Core.
- All authenticated user Core operations can be traced to `session.user.id`.

## Phase 6: Fix Relationship Type Semantics

Goal: make relationship apply durable and semantically correct.

Implementation:

- Add one canonical relationship mapping module in AllKnower.
- Cover every canonical type:
  - `ally`
  - `enemy`
  - `rival`
  - `family`
  - `member_of`
  - `leader_of`
  - `serves`
  - `located_in`
  - `originates_from`
  - `participated_in`
  - `caused`
  - `created`
  - `owns`
  - `wields`
  - `worships`
  - `inhabits`
  - `related_to`
- Use stable Core relation names such as `relMemberOf`, `relLeaderOf`, `relServes`, and `relRelatedTo`.
- Reject unknown relationship types. Do not silently collapse to `relOther`.
- Check for existing equivalent relation before writing and report it as skipped.
- Return this shape from apply:

```ts
type ApplyRelationshipsResult = {
  applied: Array<{
    sourceNoteId: string;
    targetNoteId: string;
    relationshipType: string;
    relationName: string;
  }>;
  skipped: Array<{
    sourceNoteId: string;
    targetNoteId: string;
    relationshipType: string;
    reason: string;
  }>;
  failed: Array<{
    sourceNoteId: string;
    targetNoteId: string;
    relationshipType: string;
    error: string;
  }>;
};
```

- Update `RelationHistory` writes to store canonical type and Core relation name.
- Do not migrate existing `relOther` data for v1. Display it as `Related`.

Acceptance:

- Applying `serves` no longer creates `relOther`.
- Applying the same relationship twice returns a skipped result, not a duplicate write.
- Partial failure returns `failed` entries and does not look like full success.
- Unknown relationship type returns a validation error.

## Phase 7: Fix Relationship Graph UI State

Goal: make applied AI relationships appear applied after refresh and stop showing as unapplied suggestions.

Implementation:

- Update Portal's AllKnower apply result schema/client type to match `ApplyRelationshipsResult`.
- In `components/portal/RelationshipGraph.tsx`, invalidate both:
  - `["relationships", noteId]`
  - `["note", noteId]`
- Filter AI suggestions that match existing applied relationships by target and canonical type.
- Display canonical labels instead of raw ETAPI names like `relOther`.
- Render dashed AI edges only for unapplied suggestions.
- Render solid existing edges for applied relationships.
- Do not dedupe away AI suggestions merely because another existing relation points to the same target with a different type.
- Show apply failures inline and keep failed suggestions actionable.

Acceptance:

- Click Apply, refresh page, relationship remains visible as existing.
- Applied suggestion disappears from the AI Suggestions list.
- Graph line style matches source: solid existing, dashed unapplied AI.
- Multiple relation types to the same target can be represented without hiding one incorrectly.
- Failed apply shows an error instead of an Applied state.

## Phase 8: Close UI and Accessibility Release Blockers

Goal: fix the highest-risk issues from the forensic UI audit.

Implementation:

- Ensure each Portal page has one `main` landmark.
- Fix duplicate React keys in relation/lore rendering.
- Remove raw label/relation metadata from lore-card accessible names.
- Keep lore-card visible content clean:
  - title.
  - type/status metadata.
  - short readable excerpt.
  - at most two or three useful chips.
- Raise all interactive controls to a minimum `44x44` hit target.
- Update theme-song parsing so a user can paste either a raw Spotify URL or iframe HTML. For iframe HTML, extract and sanitize the `src` URL and store only the URL.
- Preserve sanitization: raw lore HTML continues through `sanitizeLoreHtml()`.

Acceptance:

- No nested `main` landmarks on audited pages.
- No duplicate key warnings in browser console for lore detail/index flows.
- Screen reader labels for lore cards do not include storage syntax like `relationNote`, `themeSongUrl`, or raw promoted label strings.
- Touch targets meet minimum size on mobile viewport.
- Theme song iframe input stores and renders the extracted Spotify embed URL.

## Phase 9: Apply Minimum Design Token and Contrast Cleanup

Goal: remove release-blocking contrast and polish failures without redesigning the whole product.

Implementation:

- Keep the gold-on-black identity.
- Standardize type sizes to:
  - `12`
  - `14`
  - `16`
  - `20`
  - `28`
  - `40`
  - `56`
- Standardize spacing to:
  - `4`
  - `8`
  - `12`
  - `16`
  - `24`
  - `32`
  - `48`
- Standardize radii to:
  - `8`
  - `12`
  - `16`
  - `9999`
- Raise muted text contrast in dark theme where axe flagged failures.
- Rebuild light-mode gold/amber/warning/chip colors so text reaches WCAG AA.
- Reserve gold for primary actions and major display headings; move secondary labels and utility icons to neutral or semantic tokens.

Acceptance:

- Axe `color-contrast` passes on:
  - `/`
  - `/lore`
  - one lore detail page.
  - one lore edit page.
- Both dark and light themes pass the audited contrast surfaces.
- Dashboard and lore cards preserve product identity but no longer rely on low-contrast accent text.

## Required Validation

Run these before calling v1 ready:

```bash
cd allknower
bun run check
```

```bash
cd allcodex-portal
bun run check
bun run build
```

```bash
cd allcodex-core
pnpm typecheck
```

Live smoke with all services running:

1. Open Portal.
2. Register or sign in.
3. Connect Core from Portal settings.
4. Verify browser-visible responses and cookies do not expose Core ETAPI token.
5. Verify Portal status shows AllKnower authenticated and Core connected.
6. Open a lore note.
7. Apply an AI relationship.
8. Refresh the page.
9. Verify the relationship remains applied.
10. Verify the applied suggestion no longer appears as unapplied.
11. Verify graph label and line style are correct.
12. Run Gap Detector and Consistency Checker.
13. Confirm their helper text and results match their distinct purposes.
14. Confirm wipe routes are unavailable in production env.

## Release Gates

Do not tag v1 until:

- Browser cannot directly use or see backend service tokens.
- Core credentials are scoped per signed-in user.
- AllKnower no longer uses global Core credentials for authenticated user flows.
- Relationship apply is durable after refresh.
- Applied suggestions no longer appear as unapplied.
- Relation types do not collapse to `relOther`.
- Portal `bun run check` passes.
- Portal `bun run build` passes.
- AllKnower `bun run check` passes.
- Core `pnpm typecheck` passes.
- Destructive wipe routes are unavailable in production.
- Required env vars are documented.
- Root and submodule dirty trees are separated into intentional changes before tagging.

## Deferred After v1

- External identity provider.
- Fine-grained Core authorization.
- Shared workspace/root-note isolation UI.
- Audit logs for every AI/Core write.
- Token rotation UI and integration disconnect history.
- Migration of old `relOther` relationship records.
