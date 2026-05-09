# AllCodex v1 Release Plan

Date: 2026-05-06

## Goal

Ship v1 with the correct service boundary:

```text
Browser -> Portal -> AllKnower -> Core
        -> Portal -> Core
```

The browser must never talk directly to Core or AllKnower. Portal is the user-facing BFF. Core and AllKnower are private backend services.

The main v1 blockers are:

- Relationship apply appears successful but writes the wrong relation type and leaves suggestions looking unapplied.
- User/backend credential handling is not yet correct for multi-user v1.
- Release checks are not green across Portal, AllKnower, and Core.
- Destructive/dev-only wipe routes must not ship unguarded.

## Architecture Decision

Correct v1 architecture:

- Browser receives only Portal session cookies.
- Portal owns the browser-facing session and API surface.
- Backend tokens are never returned to browser JavaScript.
- AllKnower is the AI/control-plane backend and may use better-auth for user identity.
- Core remains a private legacy data service accessed only through backend-to-backend ETAPI.
- Core ETAPI tokens are integration credentials, not a full app user model.
- AllKnower stores each user's Core integration encrypted and resolves it by authenticated user.

This avoids treating Core or AllKnower as public APIs. Users interact with Portal only. Portal brokers auth, token creation, token lookup, and backend calls.

## Relationship Apply Bug

### Current Root Causes

The relationship UI can say an apply request succeeded while the visible state stays wrong because of multiple issues:

- AllKnower maps unknown relationship types to `relOther`.
- Canonical AI types such as `serves`, `member_of`, and `leader_of` are not mapped to stable Core relation names.
- Portal does not refetch the `["relationships", noteId]` query after apply.
- Portal marks a suggestion locally as applied, but the source relationship query remains stale.
- The graph deduplicates by target note and prefers an existing edge over an AI edge, so dashed AI edges disappear when any existing relation to the same target exists.
- Existing edges display raw ETAPI attribute names like `relOther` instead of canonical labels.
- Suggestions are not filtered out after an equivalent relationship exists.

### Required Fix

Keep relationship writes owned by the backend, not the browser:

```text
Browser
  -> Portal API route
  -> AllKnower authenticated apply route
  -> Core ETAPI using current user's stored Core credentials
```

Implementation requirements:

- Add complete canonical relation mapping in AllKnower.
- Reject unknown relationship types instead of silently mapping them to `relOther`.
- Return explicit apply results: applied, skipped, failed.
- Treat partial failure as visible UI failure, not success.
- Invalidate/refetch the relationship query after apply.
- Hide suggestions that now exist as real relationships.
- Show canonical labels in the graph.
- Show dashed AI edges only for unapplied suggestions.

## Gap Detector vs Consistency Checker

Gap Detector answers:

> What important lore is missing, thin, or underdeveloped?

Use it to find missing worldbuilding, weak factions, absent locations, unresolved hooks, thin cultures, missing relationships, and shallow timeline/geography coverage.

Consistency Checker answers:

> What existing lore contradicts itself or breaks continuity?

Use it to find contradictions, timeline conflicts, orphaned references, naming mismatches, logic issues, and power-scaling problems.

Same note can appear in both tools, but the action is different:

- Gap Detector means add or flesh out content.
- Consistency Checker means reconcile or fix existing content.

## User Identity and Credential Model

### Correct v1 Model

Use AllKnower better-auth as the backend identity/control-plane identity for v1, brokered through Portal.

Portal should expose browser-facing routes for:

- Register.
- Login.
- Logout.
- Session status.
- Connect Core.
- Get Core connection status.
- Disconnect Core.

The browser calls only Portal routes. Portal calls AllKnower and Core server-side.

### Core Token Creation

Core currently supports ETAPI token creation through password-based login:

```text
POST /etapi/auth/login
```

This returns an ETAPI token. Portal should perform this server-side, then store the resulting token through AllKnower in the authenticated user's encrypted integration record.

The Core password must not be persisted unless a future explicit feature requires it. Store the generated ETAPI token, not the password.

### AllKnower Storage

Replace global `AppConfig` Core credentials with per-user encrypted integration storage.

Add a model conceptually like:

```prisma
model UserIntegration {
  id             String   @id @default(cuid())
  userId         String
  provider       String
  baseUrl        String
  encryptedToken String
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt

  user           User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([userId, provider])
}
```

Exact naming can vary, but behavior must not:

- Credentials are scoped by user.
- Tokens are encrypted at rest.
- Credential lookup requires an authenticated session.
- No AI/Core route may fall back to another user's token.
- Global `allcodexUrl` and `allcodexToken` AppConfig must not be used for authenticated user flows.

### Encryption Requirements

Add an environment variable for the encryption key, for example:

```text
INTEGRATION_CREDENTIALS_KEY=
```

Behavior:

- Production must fail startup or disable Core integration if the key is missing.
- Development may use an explicit dev fallback only if clearly named and logged.
- Never log plaintext tokens.
- Never return plaintext tokens from status endpoints.

## API Changes

### Portal Routes

Portal should expose browser-safe routes:

- `POST /api/auth/register`
- `POST /api/auth/login`
- `POST /api/auth/logout`
- `GET /api/auth/session`
- `POST /api/integrations/allcodex/connect`
- `GET /api/integrations/allcodex/status`
- `DELETE /api/integrations/allcodex`

These routes should:

- Use HTTP-only cookies.
- Never return backend bearer tokens.
- Normalize backend errors into actionable Portal errors.
- Avoid leaking Core URL/token details beyond safe connection status.

### AllKnower Routes

AllKnower should expose authenticated backend routes:

- `POST /integrations/allcodex/connect`
- `GET /integrations/allcodex/status`
- `DELETE /integrations/allcodex`
- Existing AI routes should resolve Core credentials by `session.user.id`.

Deprecate or rewrite:

- `POST /config/allcodex`

It currently writes global Core URL/token into `AppConfig`, which is not acceptable for multi-user v1.

### Relationship Apply Route

Keep the existing relationship apply behavior conceptually:

```text
POST /suggest/relationships/apply
```

But update the contract:

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

Portal must show failures and refetch on successful or partially successful apply.

## Implementation Plan

### 1. Stabilize Release Branch

- Review current dirty tree in root and submodules.
- Separate intentional changes from experiments.
- Do not revert user changes without confirmation.
- Remove or guard untracked/destructive wipe routes before v1.
- Fix current Portal typecheck failure from `app/api/config/wipe/route.ts`.

### 2. Add Per-User Core Integration in AllKnower

- Add Prisma model for per-user integrations.
- Add migration.
- Add encryption/decryption helpers.
- Add integration service functions:
  - connect Core for user.
  - get Core status for user.
  - delete Core integration for user.
  - resolve Core credentials for authenticated user.
- Replace global credential cache with user-scoped credential resolution.
- Ensure all Core-writing and Core-reading AI routes use user-scoped credentials.

### 3. Update Portal Auth and Settings Flow

- Keep browser calls limited to Portal routes.
- Portal handles AllKnower login/register server-side.
- Portal handles Core ETAPI login server-side.
- Portal sends generated Core token to AllKnower for encrypted per-user storage.
- Settings page shows safe status:
  - signed into Portal/AllKnower identity.
  - Core connected/disconnected.
  - no raw tokens displayed.

### 4. Fix Relationship Type Semantics

- Define canonical relation mapping in one AllKnower module.
- Map every relationship type from the shared schema.
- Use stable Core relation names.
- Reject invalid relationship types.
- Update relation history to store canonical type and Core relation name.
- Add tests for all canonical types.

### 5. Fix Relationship Graph UI State

- Invalidate/refetch `["relationships", noteId]` after apply.
- Filter suggestions that match existing applied relationships.
- Display canonical labels instead of raw `relOther` names.
- Show AI dashed edges only for unapplied suggestions.
- Show apply errors when AllKnower returns failures.

### 6. Clarify Gap and Consistency UX

- Add concise helper text in Portal near both tools.
- Gap Detector copy: "Finds missing or underdeveloped lore."
- Consistency Checker copy: "Finds contradictions and continuity problems in existing lore."
- Avoid implying they are interchangeable.

### 7. Release Gate Cleanup

- Portal must pass `bun run check`.
- AllKnower must pass `bun run check`.
- Core must pass `pnpm typecheck`.
- Fix missing Core package tsconfig references or restore missing packages.
- Confirm no destructive route is reachable in production.
- Confirm docs and env examples mention required integration encryption key.

## Test Plan

### AllKnower

- Per-user Core integration create/update/delete.
- Encrypted token is stored, plaintext token is not stored.
- Credential lookup returns only the current user's Core token.
- User A cannot use User B's Core integration.
- AI routes fail clearly when Core is not connected.
- Relationship apply maps all canonical relationship types correctly.
- Unknown relationship types are rejected.
- Partial relationship apply failures are returned explicitly.

Run:

```bash
cd allknower
bun run check
```

### Portal

- Auth routes never return backend tokens.
- Core connect route logs into Core server-side and stores token through AllKnower.
- Status route reports connected/disconnected safely.
- Relationship apply invalidates and refetches relationship data.
- Applied suggestions disappear after refresh.
- Failed apply shows an actionable error.
- Gap/Consistency helper text appears in the correct tools.

Run:

```bash
cd allcodex-portal
bun run check
```

### Core

- ETAPI auth/token flow still works.
- Typecheck passes after package reference cleanup.

Run:

```bash
cd allcodex-core
pnpm typecheck
```

### Live Smoke

Run with all services up:

1. Open Portal.
2. Register or sign in.
3. Connect Core from Portal settings.
4. Verify Portal status shows AllKnower and Core connected.
5. Open a lore note.
6. Apply an AI relationship.
7. Refresh the page.
8. Verify the relationship remains applied.
9. Verify suggestion no longer shows as unapplied.
10. Verify the graph label and line style are correct.
11. Run Gap Detector and Consistency Checker and confirm their outputs match their stated purpose.

## Known v1 Blockers Found

Portal:

- `bun run check` currently fails because `app/api/config/wipe/route.ts` imports `etapiFetch` from `@/lib/etapi-server`, but that symbol is not exported.
- The wipe route is dangerous for v1 unless production-gated or removed.

AllKnower:

- `bun run check` passed in the latest sweep.
- Current Core credential storage is global via `AppConfig`, which is a v1 multi-user blocker.
- Auth integration testing has a known TODO around deterministic better-auth sign-in setup.

Core:

- `pnpm typecheck` currently fails because root `tsconfig.json` references missing package tsconfigs:
  - `packages/highlightjs/tsconfig.json`
  - `packages/share-theme/tsconfig.json`
  - `packages/pdfjs-viewer/tsconfig.json`

Repository:

- Root and submodules have existing dirty/untracked files.
- v1 tag should wait until intentional changes are separated from experiments.

## Release Criteria

Do not tag v1 until:

- Browser cannot directly use or see backend service tokens.
- Core credentials are scoped per signed-in user.
- AllKnower no longer uses global Core credentials for authenticated user flows.
- Relationship apply is durable after refresh.
- Applied suggestions no longer appear as unapplied.
- Relation types do not collapse to `relOther`.
- Portal check passes.
- AllKnower check passes.
- Core typecheck passes.
- Destructive wipe routes are unavailable in production.
- Required env variables are documented.

## Deferred After v1

- Replace better-auth identity with an external IdP if needed.
- Add fine-grained Core authorization if Core evolves beyond private data-service role.
- Add workspace/root-note isolation UI for shared Core instances.
- Add audit logs for every AI/Core write.
- Add admin UI for token rotation and integration disconnect history.

## Appendix: UI and UX Forensic Audit

Audit date: 2026-05-06

Audit scope:

- URLs audited:
  - `http://localhost:3000/`
  - `http://localhost:3000/lore`
  - `http://localhost:3000/lore/PalipvHQTbmK`
  - `http://localhost:3000/lore/PalipvHQTbmK/edit`
- Viewports covered:
  - `1920x1080`
  - `1440x900`
  - `1024x768`
  - `768x1024`
  - `375x812`
- Evidence artifacts:
  - screenshots and snapshots captured under `.playwright-cli/`
  - representative files: `dashboard-1920x1080.png`, `lore-1920x1080.png`, `detail-1920x1080.png`, `edit-1920x1080.png`, plus mobile full-page variants
- Accessibility tooling note:
  - Lighthouse and `axe` CLI did not complete reliably against the current dev session, so contrast findings below are manual spot-checks from computed values

### Top-line diagnosis

The interface is over-chromed and under-hierarchized. Shell, borders, pills, and accent color all compete at nearly the same visual intensity, so the lore content is not the first thing the eye lands on. The underlying token system is also fragmented, which makes the product feel assembled rather than art-directed.

### P0 - Critical issues

- Lore Browser accessible names leak storage internals.
  - Current: lore entry links expose raw strings like `relationNote:`, `themeSongUrl:`, and `label:...promoted,alias=...` in the interactive label, not just the visible title and note metadata.
  - Evidence: `.playwright-cli/current-depth7.yaml` lines 98-120 show the full raw accessible names.
  - Recommended: accessible name should be limited to title, type, and modified date; raw metadata belongs in hidden inspect/debug UI and secondary descriptions.
  - Why: screen readers and keyboard users are forced through implementation detail instead of content.

- Touch targets are below minimum production size.
  - Current:
    - `GM` toggle is `61x28`
    - `Player` toggle is `71x28`
    - `Edit` button is `69x32`
    - dashboard `New Lore Entry` CTA is `133x32`
  - Recommended: all tappable controls should be at least `44x44`, with icon-only buttons also landing on `44x44`.
  - Why: the controls are visually delicate and physically unreliable on touch devices.

- Landmark and render integrity are not production-safe.
  - Current: dashboard contains nested `main` landmarks.
  - Evidence: `.playwright-cli/home-depth8.yaml` lines 99-110.
  - Current: lore index and detail pages log duplicate React keys for relations.
  - Evidence:
    - `.playwright-cli/console-2026-05-06T09-50-20-944Z.log` lines 3-7
    - `.playwright-cli/console-2026-05-06T09-51-11-054Z.log` lines 3-8
  - Recommended: enforce one `main` landmark per view and generate relation row keys from stable ids, not repeated note labels.
  - Why: assistive tech gets noisy landmarks and the relation UI can duplicate or omit items.

### P1 - High-impact issues

- Type scale collapse.
  - Current font sizes in use: `10, 11, 12, 14, 16, 17.2, 18, 24, 48, 60px`.
  - Current line-heights in use: `13.33, 15, 16, 16.5, 18, 19.5, 20, 24, 24.75, 27.95, 32, 48, 60px`.
  - Recommended: lock the system to a small deliberate scale, for example `12 / 14 / 16 / 20 / 28 / 40 / 56`, with integer line-heights.
  - Why: too many adjacent sizes erase hierarchy because nothing feels intentionally larger or smaller.

- Typography changes personality between modes.
  - Current:
    - dashboard H1: `Cinzel 24/32`
    - detail H1: `Cinzel 60/60`, `900` weight, `2.4px` tracking
    - published prose: `Crimson Text 17.2/27.95`
    - editor body: `Inter 16/28` inside BlockNote
  - Recommended: `Cinzel` for display only, one prose face for both read and edit content, and one UI face for controls/metadata.
  - Why: edit mode currently feels like a different product from read mode.

- Surface hierarchy is muddled by too many near-black values.
  - Current sampled surface colors include `#000103`, `#010106`, `#0f0e27`, `#252541`, `#02020c`, plus multiple translucent overlays.
  - Current sampled border family includes `#2f2f4c`, `#2f2f4b66`, `#2f2f4b99`, `#30304c40`, `#2e2e4c4d`.
  - Recommended: reduce the system to three surface tiers and one primary border token.
  - Why: when every panel is a slightly different black, depth becomes guesswork.

- Chrome is heavier than content.
  - Current:
    - at `768px`, the left rail still consumes roughly one third of the viewport
    - command palette remains permanently mounted
    - detail side-rail modules visually compete with the actual article body
  - Recommended: collapse the sidebar by `<=1280px`, move command palette to a modal/off-canvas surface, and progressively disclose the detail rail.
  - Why: the interface spends too much visual capital on its frame.

- Lore cards are polluted by implementation detail.
  - Current: lore browser cards render raw labels and relations as visible card content, not clean excerpts.
  - Evidence: `lore-1920x1080.png` and `lore-375x812-full.png`.
  - Recommended: title, one clean excerpt, and at most two or three metadata chips.
  - Why: scanning fails when storage syntax is treated as user-facing content.

### P2 - Polish and consistency issues

- Radius system is visibly inconsistent.
  - Current non-zero radii sampled from the live UI: `7.6, 9.6, 13.6, 17.6, 21.6px`, plus pill radius rendered as `3.35544e+07px`.
  - Recommended: `8 / 12 / 16 / 9999`.
  - Why: small mathematical differences accumulate into a nearly-aligned, slightly-sloppy feel.

- Accent gold is overused.
  - Current: `#FFA600` is used for brand, page titles, stat values, CTA fills, icons, chips, and dividers.
  - Recommended: reserve gold for primary action and major display headings; move section labels and secondary icons to neutral text tokens.
  - Why: if everything is accented, nothing is accented.

- Action rows are not optically aligned.
  - Current: dashboard heading and CTA do not share a strong baseline; detail-mode `GM / Player / Edit` reads like a detached utility strip instead of part of the header composition.
  - Recommended: align title/action rows to a consistent 8px baseline rhythm.
  - Why: the layout reads assembled, not composed.

- Empty states read as rendering failures.
  - Current: `RAG Indexed` shows a lone dash in a full KPI card.
  - Recommended: explicit empty copy such as `Not indexed yet` plus a short explanatory line.
  - Why: the current state looks broken, not intentional.

- Interaction states are too subtle.
  - Current: hover and focus treatments stay in the same visual family, especially on smaller controls; state change relies mostly on slight tint and border changes.
  - Recommended: define distinct default, hover, focus, and active tokens; focus ring should be unmistakable and external.
  - Why: affordance depends on clear state separation.

### Systemic recommendations

- Type tokens:
  - `Cinzel` only for display headings
  - one prose face for read and edit surfaces
  - one UI face for chrome, forms, and controls
  - baseline sizes: `12 / 14 / 16 / 20 / 28 / 40 / 56`

- Spacing tokens:
  - standardize on `4 / 8 / 12 / 16 / 24 / 32 / 48`
  - remove ad hoc card padding and oversized shell gutters

- Radius tokens:
  - standardize on `8 / 12 / 16 / pill`

- Color roles:
  - define explicit tokens for page bg, surface 1, surface 2, border, text-primary, text-muted, primary accent, AI accent, success, warning
  - keep the gold-black pairing, but reduce accent usage sharply

- Layout policy:
  - collapse the sidebar by `1280px`
  - move command palette to overlay
  - convert detail side-rail modules into tabs or accordions on tablet/mobile
  - keep prose measure near `65-75ch`

- Accessibility policy:
  - one `main` landmark per page
  - no raw metadata in interactive names
  - minimum `44x44` touch targets
  - duplicate-key and semantic-landmark issues should be treated as release blockers

### Manual contrast spot-checks

Automated contrast audit:

- Tooling used:
  - Playwright Chromium `147.0.7727.15`
  - injected `axe-core 4.11.0`
  - rule run: `color-contrast`
- Runtime note:
  - audit ran against the live dev server on `http://localhost:3000`
  - production build remains blocked by the known `app/api/config/wipe/route.ts` import failure, so this is the best stable automated pass available without code changes

Automated results summary:

- Dark theme:
  - `/`: `0` contrast violations
  - `/lore`: `36` violations
  - `/lore/PalipvHQTbmK`: `8` violations
  - `/lore/PalipvHQTbmK/edit`: `1` violation

- Light theme:
  - `/`: `8` violations
  - `/lore`: `55` violations
  - `/lore/PalipvHQTbmK`: `39` violations
  - `/lore/PalipvHQTbmK/edit`: `7` violations

Most important automated failures:

- Dark theme lore browser metadata chips.
  - Current: the small metadata pills under lore cards use `10px` text at `3.16:1` contrast.
  - Example selector: `a[href$="MvtyY7crSwsI"] > .flex-wrap.mt-2.gap-1 > .text-muted-foreground/70.bg-secondary/50.rounded:nth-child(1)`
  - Current colors: foreground `#5f6071`, background `#0b0a1c`
  - Recommended: raise the foreground to at least `4.5:1` against the chip background or remove this microtext from the primary card surface entirely.
  - Why: these chips are both semantically noisy and below AA.

- Dark theme lore detail labels.
  - Current: wiki detail labels in the side rail use `11px` text at `3.52:1`.
  - Example selector: `.grid-cols-[96px_minmax(0,1fr)].border-border/25.last:border-0:nth-child(1) > .wiki-detail-label`
  - Current colors: foreground `#636473`, background `#02030b`
  - Recommended: use a stronger muted token that still lands above `4.5:1`, or increase size and weight if they remain utility labels.
  - Why: the current label tone is too faint for supporting metadata that users still need to read.

- Dark theme edit action button.
  - Current: one amber outline action on the edit page lands at `4.31:1`.
  - Example selector: `.border-amber-500/30`
  - Current colors: foreground `#b55b00`, background `#04050b`
  - Recommended: either darken the amber text further or use the filled primary treatment consistently.
  - Why: this is close to AA but still fails, which makes it an easy token-level fix.

- Light theme shell and headings.
  - Current:
    - app wordmark: `2.82:1`
    - top-rail product name: `3.04:1`
    - primary page H1 on dashboard/edit: around `2.98:1`
  - Example selectors:
    - `.font-bold.tracking-wider.uppercase`
    - `.tracking-[0.2em]`
    - `h1`
  - Current colors: gold/orange foregrounds around `#cc7800` on pale surfaces `#f0ebe0` to `#f8f3ea`
  - Recommended: either darken the gold substantially for light mode or stop using gold as the default text color on pale backgrounds.
  - Why: the entire light-mode identity collapses contrast at the brand level.

- Light theme primary actions.
  - Current: primary filled buttons fail at `3.16:1`.
  - Example selector: `.bg-primary`
  - Current colors: foreground `#fbf8f1`, background `#cc7800`
  - Recommended: switch button text to a much darker foreground in light mode, or deepen the primary fill.
  - Why: the most important actions in light mode do not meet AA.

- Light theme warning/status surfaces.
  - Current:
    - warning title text at `1.15:1`
    - warning secondary text at `1.30:1`
    - yellow outlined button at `1.17:1`
  - Example selectors:
    - `.flex-1.min-w-0 > .text-yellow-300`
    - `.text-yellow-400/80`
    - `.border-yellow-500/40`
  - Recommended: rebuild warning semantics in light mode around darker amber/brown text on pale-warning surfaces.
  - Why: these are catastrophic failures, not edge cases.

- Light theme relation chips.
  - Current: violet relation chips on the lore detail page fail at `1.04:1`.
  - Example selector: `.wiki-relation-chip.wiki-relation-chip--violet`
  - Current colors: foreground `#ede9fe`, background `#ede3f2`
  - Recommended: use a dark violet foreground on the pale chip or invert the chip to a dark fill.
  - Why: the chip text is effectively unreadable.

- Light theme rail kickers and utility labels.
  - Current:
    - wiki rail kickers: `3.20:1`
    - wiki detail labels: `3.82:1`
    - section title `RELATED ENTRIES` style: `2.67:1`
  - Example selectors:
    - `.wiki-rail-kicker`
    - `.wiki-detail-label`
    - `.wiki-section-title`
  - Recommended: promote all light-mode utility text one full contrast step darker.
  - Why: the system currently confuses “subtle” with “illegible.”

- `#FFA600` on `#000103`: `10.65:1`
- `#F3EEE6` on `#000103`: `18.08:1`
- `#CFCFCF` on `#000103`: `13.40:1`
- `#838595` on `#000103`: `5.72:1`
- `#010102` on `#FFA600`: `10.64:1`

Conclusion:

- primary contrast is generally strong
- dark theme debt is not mostly contrast
- light theme has broad contrast debt across shell, headings, chips, badges, and warning states
- the biggest systemic problems are still semantics, touch targets, hierarchy, token inconsistency, and a light theme whose accent strategy does not survive pale surfaces

### What is working and should be preserved

- The core gold-on-black direction has strong brand potential.
- The desktop lore detail prose card is readable once the surrounding chrome gets out of the way.
- Type/category chips in the lore browser provide useful at-a-glance classification once the card content is cleaned up.
