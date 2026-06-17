# Progressive Brain Dump Streaming

Status: shipped on 2026-05-25 across `allcodex-portal` and `allknower`.

This note records the shipped progressive entity rendering and SSE reliability work for the brain-dump flow.

## Current Flow

The Portal calls AllKnower's `/brain-dump/stream` endpoint and renders entity cards while the parser response is still streaming. The UI no longer waits for the final write phase before showing extracted entities.

```text
SSE token event
  -> append token to Portal stream buffer
  -> parse completed and partial entities
  -> render completed cards and the in-flight card

SSE status: write
  -> keep completed cards visible while AllKnower writes notes

SSE done
  -> normalize result, invalidate queries, and show final completion state
```

## Shipped Changes

- Extracted the streaming entity parser to `allcodex-portal/lib/parse-streaming-entities.ts`.
- Added parser unit coverage in `allcodex-portal/lib/parse-streaming-entities.test.ts`.
- Re-exported parser compatibility helpers from `allcodex-portal/lib/sanitize.ts`.
- Moved streaming ingestion state into `allcodex-portal/lib/stores/brain-dump-store.ts`.
- Added token event count, elapsed timer, abort handling, stream result normalization, toasts, and query invalidation from the store.
- Updated the Brain Dump page to show completed and partial entity cards while streaming and through the write/complete phases.
- Fixed the Portal SSE parser to flush any final buffered line on stream close, so a terminal `done` event is not dropped.
- Added AllKnower SSE keepalive comments every 10 seconds to prevent idle stream closure during long RAG, parser, and write gaps.

## Reliability Fix

Live integration initially failed after the backend created history and notes because the browser never received the final `done` event. Two conditions caused the failure:

- AllKnower had long quiet periods during RAG, parser, and database writes.
- The Portal client parser could drop a final buffered event if the stream closed immediately after the terminal line.

The fix is two-sided:

- AllKnower sends SSE comment keepalives (`: keepalive`) every 10 seconds.
- Portal flushes its remaining SSE buffer when the reader finishes.

## Verification

The shipped change was validated with:

```bash
cd /Users/allmaker/projects/allcodex-aio/allcodex-portal
bun run check
npx playwright test --project=integration tests/integration/brain-dump-live.spec.ts
npx playwright test --project=integration
```

Result: green. The direct AllKnower SSE probe also exited `0`, emitted one `event: done`, and emitted keepalive comments during long quiet phases.
