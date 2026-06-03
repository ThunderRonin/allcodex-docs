# AllCodex v1 Final Hit Plan

Date: 2026-05-06
Status: execution-ready

This is the final v1 completion checklist. It assumes current work continues on
the active `dev` branches in the root repo and submodules.

## Objective

Ship v1 only after the real production boundary is true:

```text
Browser -> Portal -> AllKnower -> Core
        -> Portal -> Core
```

Rules:

- Browser talks only to Portal.
- Portal is the only browser-facing BFF.
- Core and AllKnower are private backend services.
- Backend tokens never reach browser JavaScript.
- Core ETAPI tokens are per-user integration credentials.
- AllKnower owns encrypted per-user Core integration storage.
- All user-facing Core operations resolve credentials from the signed-in user.

## Safety Rules

- Do not kill occupied ports. If a service port is occupied or stale, start the
  needed service on another port and point dependents at it.
- Do not run DB reset, table drop, destructive migration repair, forced git
  reset, force push, or commit amend.
- Do not run wipe routes unless `NODE_ENV !== "production"` and
  `ALLOW_DEV_WIPE === "true"`.
- Do not log, print, or copy plaintext Core ETAPI tokens.
- Stage dirty and untracked files only after release gates pass. Do not commit
  unless explicitly requested.

## Phase 1: State Snapshot

Record current state before changes:

```bash
git status --short
git -C allknower status --short
git -C allcodex-portal status --short
git -C allcodex-core status --short
git -C docs/shared status --short
```

Expected current risk areas:

- Root has dirty/untracked files.
- `allknower`, `allcodex-portal`, and `allcodex-core` all have v1 changes.
- Docs are dirty/untracked.
- Local services may already occupy `3000`, `3001`, `8080`, or `5433`.

## Phase 2: Migration Safety Gate

AllKnower has v1 credential storage migration work. Before applying anything:

```bash
cd allknower
bunx prisma migrate status
bunx prisma generate
```

Rules:

- If Prisma reports pending additive migrations only, apply with the project
  migration command.
- If Prisma asks for reset, drift repair, or destructive action, stop.
- If DB looks production-like, take a dump before applying migration.

Acceptance:

- `UserIntegration` table exists through migration, not raw SQL.
- `INTEGRATION_CREDENTIALS_KEY` is required outside explicit test/dev paths.
- Stored Core token is encrypted at rest.

## Phase 3: Finish User-Scoped Core Credentials

Complete the credential sweep in AllKnower:

- Extend explicit `EtapiCredentials` support to every Core helper in
  `src/etapi/client.ts`.
- Thread user-resolved Core credentials through brain dump, brain dump commit,
  relationship apply, consistency checker, gap detector, RAG index/query paths,
  import/setup, and AZGaar flows.
- Authenticated user flows must not fall back to global `AppConfig`
  `allcodexUrl` or `allcodexToken`.
- Keep global Core config only for clearly marked legacy/dev/admin paths.
- Core connect must validate URL/token against Core before storing encrypted
  credentials.

Acceptance:

- User without Core integration gets clear reconnect error.
- User B cannot use User A's Core credentials.
- Two users connected to different Core URLs resolve different credentials.
- No authenticated Core write path can execute without `session.user.id`.

## Phase 4: Portal Auth and Integration Boundary

Verify Portal remains the only browser-facing integration surface:

- Browser auth routes call AllKnower server-side.
- Portal stores AllKnower session in HTTP-only cookie.
- Portal never stores `allcodex_token` in browser cookies.
- Core password is used only server-side to create ETAPI token.
- Portal sends generated Core token to AllKnower for encrypted storage.
- Browser-visible JSON responses never include AllKnower bearer token or Core
  ETAPI token.

Acceptance:

- `/api/auth/session` returns safe session state only.
- `/api/integrations/allcodex/status` returns safe connection metadata only.
- Settings UI shows signed-in state and Core connected/disconnected state.
- Disconnect removes only the current user's integration.

## Phase 5: Relationship Apply Fix

Release blocker: AI relationship apply must become durable and visible.

Implementation requirements:

- AllKnower maps canonical relationship types to stable Core relation names.
- Unknown relationship types are rejected, not collapsed to `relOther`.
- Duplicate equivalent relations return `skipped`, not duplicate writes.
- Partial failures return `failed` entries and show UI failure.
- Portal invalidates relationship and note queries after apply.
- Applied suggestions disappear from unapplied suggestions after refresh.
- Existing relationships render solid; unapplied AI suggestions render dashed.
- Existing labels display canonical names, not raw `relOther`.

Acceptance:

- Applying `serves` creates `relServes`.
- Applying `member_of` creates `relMemberOf`.
- Applying `leader_of` creates `relLeaderOf`.
- Refresh keeps relation visible as existing.
- Previously applied suggestion no longer shows as actionable.

## Phase 6: Live Runtime Strategy

Do not kill occupied services. Detect ports:

```bash
lsof -nP -iTCP:3000 -sTCP:LISTEN
lsof -nP -iTCP:3001 -sTCP:LISTEN
lsof -nP -iTCP:8080 -sTCP:LISTEN
lsof -nP -iTCP:5433 -sTCP:LISTEN
```

If occupied or stale:

- Portal fallback: `3100`.
- AllKnower fallback: `3101`.
- Core fallback: use existing healthy Core if possible; otherwise start on a
  supported alternate port if Core config allows.
- Wire Portal env to the selected AllKnower/Core URLs.

Health is not enough. Verify behavior through Portal.

## Phase 7: Release Verification

Run static checks:

```bash
cd allknower && bun run check
cd allcodex-portal && bun run check
cd allcodex-portal && bun run build
cd allcodex-core && pnpm typecheck
```

Run Core tests if time allows or Core changes expand:

```bash
cd allcodex-core && pnpm test:sequential
```

Run live smoke through Portal:

- Register/login user.
- Confirm only HTTP-only Portal/AllKnower session cookie exists.
- Connect Core from Portal.
- Confirm no Core ETAPI token in browser JSON, localStorage, or readable cookies.
- Open dashboard, lore list, lore detail, edit page, and settings.
- Run relationship suggestion/apply/refresh flow.
- Run gap detector and consistency checker with a connected Core integration.
- Verify no production wipe route is callable.
- Run axe/contrast checks on desktop and mobile for `/`, `/lore`, lore detail,
  edit, and settings.

## Phase 8: Stage Handoff

Only after gates pass:

```bash
git -C allknower add -A
git -C allcodex-portal add -A
git -C allcodex-core add -A
git -C docs/shared add -A
git add -A
```

Then record:

```bash
git status --short
git -C allknower status --short
git -C allcodex-portal status --short
git -C allcodex-core status --short
git -C docs/shared status --short
```

Do not commit. User will decide commit/tag flow.

## Stop Conditions

Stop and report exact blocker if:

- Migration requires reset, destructive repair, or data drop.
- Core credential needed for live connect is unavailable.
- Stale occupied service cannot be bypassed with alternate port.
- Same issue fails after three diagnose/fix/check cycles.
- Any check fails and root cause is not isolated.

## v1 Tag Criteria

Do not tag v1 until all are true:

- All static checks pass.
- Live Portal user flow passes.
- Per-user Core credentials are enforced.
- Relationship apply persists and refreshes correctly.
- Dev-only wipe routes are gated.
- No backend token leaks to browser.
- Dirty/untracked release work is staged, not committed.
- Residual risks are documented with exact evidence.
