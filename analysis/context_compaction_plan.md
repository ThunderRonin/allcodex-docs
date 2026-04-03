# AllKnower Context Compaction — Full 3-Tier Implementation Plan (v2)

> Inspired by Claude Code's leaked compaction architecture (March 31, 2026 npm leak).
> Adapted for AllKnower's stateless LLM pipeline and RAG-based lore retrieval.

**Implementation status (April 3, 2026):**

| Tier | Description | Status |
|---|---|---|
| Tier 0 | Token counting infrastructure (`utils/tokens.ts`) | ✅ Shipped |
| Tier 1 | RAG budget enforcement (`prompt.ts`) | ✅ Shipped |
| Tier 1.5 | Chunk deduplication (`rag/chunk-dedup.ts`, 0.85 cosine threshold) | ✅ Shipped |
| Tier 2 | Chunk summarization (`rag/chunk-compactor.ts`) | ✅ Shipped |
| Tier 3 | Session compaction (`pipeline/session-compactor.ts`, Prisma tables) | 🔧 Prepared (code + DB tables exist, not wired to routes) |

---

## Background

Claude Code manages context pressure through a tiered cascade:

```
tool result budgeting → microcompact → context collapse → autocompact
```

AllKnower doesn't have a conversation session (yet), but it has the **exact same
problem in a different dimension**: as All Reach's lore base grows, RAG context
injected into `brain-dump` calls becomes an unbounded token sink. A mature grimoire
with 500+ detailed notes means `queryLore(rawText, 10)` can return 10,000+ tokens
of context with zero enforcement — silently bloating every call.

The three-tier compaction plan solves this now (Tiers 1–2) and future-proofs the
architecture for multi-turn sessions (Tier 3).

---

## Architecture Overview

```
Tier 1 — RAG Budget Enforcement      free, pure logic,    prompt.ts
Tier 2 — Chunk Summarization         cheap LLM call,      rag/chunk-compactor.ts
Tier 3 — Session State + AutoCompact full LLM + DB,       pipeline/session-compactor.ts
```

Tiers are **independent and additive**:
- Tier 1 ships today, no dependencies
- Tier 2 requires Tier 1's env vars to already exist
- Tier 3 is a standalone service, wired into routes when multi-turn lands

---

## Files Touched

| File | Status | Change |
|---|---|---|
| `src/env.ts` | modify | Add 7 new env vars |
| `src/pipeline/model-router.ts` | modify | Add `compact` + `session-compact` to TaskType |
| `src/pipeline/prompt.ts` | modify | Token budget enforcement + cache_control header; **becomes async** |
| `src/rag/chunk-compactor.ts` | **new** | Tier 2 chunk summarization service + LRU summary cache |
| `src/rag/chunk-dedup.ts` | **new** | Tier 1.5 semantic dedup for near-duplicate chunks |
| `src/pipeline/session-compactor.ts` | **new** | Tier 3 session state + autocompact service |
| `src/types/lore.ts` | modify | Add `LoreSessionState` schema |
| `prisma/schema.prisma` | modify | Add `LoreSession` + `LoreSessionMessage` models |
| `src/utils/tokens.ts` | **new** | Centralized token counting with tiktoken + heuristic fallback |

---

## Tier 0 — Token Counting Infrastructure ✅

> **Cost:** zero LLM calls
> **Effort:** ~20 min
> **Impact:** foundation — every downstream tier depends on accurate-ish counts

### Problem with `Math.ceil(text.length / 4)`

The `chars / 4` heuristic is fine for English prose, but lore entries have proper
nouns, invented words, hyphenated compound names, and inline JSON/YAML blocks. These
tokenize *worse* than prose — `"Übermenschreich"` is 4+ tokens, not the 3.75 the
heuristic predicts. Across a 10-chunk budget loop, this drift compounds to 15–25%
over-admission.

### `src/utils/tokens.ts` (new file)

```ts
import { encoding_for_model } from "tiktoken";

let encoder: ReturnType<typeof encoding_for_model> | null = null;

function getEncoder() {
    if (!encoder) {
        try {
            encoder = encoding_for_model("cl100k_base"); // Claude + GPT-4 tokenizer
        } catch {
            // tiktoken WASM init can fail in edge runtimes — fall back gracefully
            return null;
        }
    }
    return encoder;
}

/** Count tokens. Uses tiktoken if available, heuristic fallback otherwise. */
export function countTokens(text: string): number {
    const enc = getEncoder();
    if (enc) return enc.encode(text).length;
    // Fallback: slightly pessimistic to avoid over-admission
    return Math.ceil(text.length / 3.5);
}

/**
 * Inverse: approximate character count from token target.
 * Used for string slicing in rebuildContext (Tier 3).
 */
export function tokensToChars(tokens: number): number {
    return Math.floor(tokens * 3.5); // pessimistic = safe truncation
}
```

**Why `cl100k_base`?** It's the tokenizer for both Claude and GPT-4 families. Not
perfect for every model OpenRouter might route to, but within 5% for all modern LLMs.
Good enough for budgeting, which is all we need.

**Install:** `bun add tiktoken`

> **Fallback note:** The heuristic divisor is changed from `4` to `3.5` (pessimistic)
> so that when tiktoken is unavailable, the budget loop *under-admits* rather than
> *over-admits*. Over-admission silently blows the context window. Under-admission
> drops a low-relevance chunk. The failure mode matters.

---

## Tier 1 — RAG Budget Enforcement ✅

> **Cost:** zero LLM calls, pure logic
> **Effort:** ~30 min
> **Impact:** immediate — prevents prompt bloat as lore base scales

### Core Idea

RAG chunks from `queryLore` are already sorted by relevance score (hybrid reranker
output). Dropping the tail is safe — you always keep the best ones. Just enforce a
hard token budget before building `contextBlock`.

### `env.ts` additions

```ts
// Tier 1
RAG_CONTEXT_MAX_TOKENS: z.coerce.number().default(6000),

// Tier 2 (add now so it's ready)
RAG_CHUNK_SUMMARY_THRESHOLD_TOKENS: z.coerce.number().default(600),
COMPACT_MODEL: z.string().default("anthropic/claude-haiku-4-5-20251001"),
COMPACT_FALLBACK_1: z.string().default("openai/gpt-4.1-nano"),
COMPACT_FALLBACK_2: z.string().default(""),
RAG_CHUNK_DEDUP_SIMILARITY_THRESHOLD: z.coerce.number().default(0.85),

// Tier 3
SESSION_TOKEN_THRESHOLD: z.coerce.number().default(80000),
```

### `prompt.ts` changes

**0. Import the centralized token counter:**

```ts
import { countTokens } from "../utils/tokens";
```

Kill any local `countTokens` heuristic. Single source of truth.

**1. Explicit prompt cache control** on the system message in `callLLM`:

The `BRAIN_DUMP_SYSTEM` prompt is ~4,000 tokens. OpenRouter passes `cache_control`
through to Anthropic. At a 90% cache discount on repeated calls, this is real money.

```ts
// In the messages array inside callLLM:
{ role: "system", content: system, cache_control: { type: "ephemeral" } }
```

Check OpenRouter SDK docs for exact field name — may require casting to `any` if
the SDK types don't expose it yet.

> **Cache topology note:** `cache_control: "ephemeral"` only works on Anthropic models
> via OpenRouter. For OpenAI-routed models, the cache behavior is automatic (prompt
> prefix caching). No harm in sending the header to non-Anthropic models — it's
> silently ignored. But don't *rely* on cache hits for cost modeling unless you know
> the routing destination.

**2. Budget enforcement loop** in `buildBrainDumpPrompt`:

Replace the existing `ragContext.map(...)` contextBlock assembly with:

```ts
const MAX_RAG_TOKENS = env.RAG_CONTEXT_MAX_TOKENS;
let budgetUsed = 0;
const admittedChunks: RagChunk[] = [];

for (const chunk of ragContext) {
    // ragContext is already sorted by relevance score desc (hybrid reranker output)
    const chunkTokens = countTokens(chunk.content);

    if (budgetUsed + chunkTokens > MAX_RAG_TOKENS) {
        // Don't break immediately — check if a smaller chunk later still fits.
        // The relevance sort makes this rare, but a 3,000-token chunk followed
        // by a 200-token highly relevant chunk shouldn't kill the small one.
        if (chunkTokens > MAX_RAG_TOKENS - budgetUsed) {
            rootLogger.info("RAG chunk skipped — exceeds remaining budget", {
                noteTitle: chunk.noteTitle,
                chunkTokens,
                budgetUsed,
                MAX_RAG_TOKENS,
            });
            continue; // try the next one — it might fit
        }
    }

    admittedChunks.push(chunk);
    budgetUsed += chunkTokens;

    if (budgetUsed >= MAX_RAG_TOKENS) break; // hard stop once full
}

rootLogger.info("RAG budget summary", {
    totalChunks: ragContext.length,
    admittedChunks: admittedChunks.length,
    droppedChunks: ragContext.length - admittedChunks.length,
    budgetUsed,
    MAX_RAG_TOKENS,
});

// Use admittedChunks instead of ragContext for contextBlock
const contextBlock =
    admittedChunks.length > 0
        ? admittedChunks.map((c) => `### ${c.noteTitle}\n${c.content}`).join("\n\n")
        : "No existing lore found — this appears to be new content.";
```

> **Change from v1:** replaced `break` with `continue`. The original greedy-fill
> approach ("break on first overflow") is correct for strictly monotonically-sized
> chunks. But in practice, chunk sizes vary wildly. A 2,500-token location entry
> followed by a 150-token character stub means the stub gets killed even though it
> fits. `continue` turns this into a best-fit scan — still O(n), negligible cost.

**Return `admittedChunks` alongside the prompt strings** so Tier 2 can receive them
without re-running the budget loop.

Update `buildBrainDumpPrompt` return type:
```ts
}: { system: string; context: string; user: string; admittedChunks: RagChunk[] }
```

---

## Tier 1.5 — Semantic Deduplication

> **Cost:** zero LLM calls, vector similarity only
> **Effort:** ~1 hour
> **Impact:** prevents near-duplicate chunks from wasting budget

### Problem

Chunking strategies (especially overlapping windows) produce near-duplicate content.
Two chunks from the same long note with 70% overlap both get admitted, wasting budget
on redundant context. The reranker doesn't catch this because both are independently
relevant.

### `src/rag/chunk-dedup.ts` (new file)

```ts
import { countTokens } from "../utils/tokens";

/**
 * Deduplicate RAG chunks by content similarity.
 * Uses Jaccard similarity on token trigrams — fast, no LLM, no embeddings.
 *
 * Must run BEFORE Tier 1 budget loop (operates on the full candidate set).
 */
export function deduplicateChunks(
    chunks: RagChunk[],
    similarityThreshold: number = env.RAG_CHUNK_DEDUP_SIMILARITY_THRESHOLD,
): RagChunk[] {
    if (chunks.length <= 1) return chunks;

    const seen: RagChunk[] = [];

    for (const chunk of chunks) {
        const isDuplicate = seen.some(
            (s) => jaccardSimilarity(trigrams(s.content), trigrams(chunk.content)) >= similarityThreshold
        );
        if (!isDuplicate) {
            seen.push(chunk);
        } else {
            rootLogger.debug("RAG chunk deduped", {
                noteTitle: chunk.noteTitle,
                threshold: similarityThreshold,
            });
        }
    }

    return seen;
}

function trigrams(text: string): Set<string> {
    const words = text.toLowerCase().split(/\s+/);
    const result = new Set<string>();
    for (let i = 0; i < words.length - 2; i++) {
        result.add(`${words[i]} ${words[i + 1]} ${words[i + 2]}`);
    }
    return result;
}

function jaccardSimilarity(a: Set<string>, b: Set<string>): number {
    let intersection = 0;
    for (const item of a) {
        if (b.has(item)) intersection++;
    }
    const union = a.size + b.size - intersection;
    return union === 0 ? 0 : intersection / union;
}
```

### Integration in `prompt.ts`

Before the Tier 1 budget loop:

```ts
// Tier 1.5: deduplicate near-identical chunks
const dedupedContext = deduplicateChunks(ragContext);
// Then run Tier 1 budget loop on dedupedContext instead of ragContext
```

---

## Tier 2 — Chunk Summarization

> **Cost:** one cheap LLM call per oversized chunk
> **Effort:** ~2–3 hours
> **Impact:** medium-term — prevents individual large notes from dominating the budget

### Core Idea

After Tier 1, some admitted chunks might still be individually huge — a 2,500-token
detailed location entry eats 40% of the budget for one entity. Summarize anything
over the threshold to its semantic essence before injecting.

The summary model should be the cheapest/fastest available (Haiku tier). The static
system prompt means it cache-hits on every call.

### Summary Cache (critical for cost)

**Without caching, the same 2,500-token chunk gets re-summarized on every single
brain-dump call.** If the underlying note hasn't changed, this is pure waste.

```ts
// In chunk-compactor.ts
import { createHash } from "crypto";

/** LRU cache: content hash → summarized content */
const summaryCache = new Map<string, { summary: string; cachedAt: number }>();
const CACHE_MAX_SIZE = 200;
const CACHE_TTL_MS = 1000 * 60 * 60; // 1 hour — notes don't change mid-session

function contentHash(content: string): string {
    return createHash("sha256").update(content).digest("hex").slice(0, 16);
}

function getCachedSummary(content: string): string | null {
    const hash = contentHash(content);
    const entry = summaryCache.get(hash);
    if (!entry) return null;
    if (Date.now() - entry.cachedAt > CACHE_TTL_MS) {
        summaryCache.delete(hash);
        return null;
    }
    return entry.summary;
}

function setCachedSummary(content: string, summary: string): void {
    const hash = contentHash(content);
    // LRU eviction: if full, delete oldest entry
    if (summaryCache.size >= CACHE_MAX_SIZE) {
        const oldest = summaryCache.keys().next().value!;
        summaryCache.delete(oldest);
    }
    summaryCache.set(hash, { summary, cachedAt: Date.now() });
}
```

> **Why in-memory LRU and not Redis/LanceDB?** AllKnower is a single-process Elysia
> server. No horizontal scaling means no cache coherence problem. In-memory is the
> right call until you scale out. When you do, migrate this to a `chunk_summaries`
> table in Postgres keyed on content hash.

### `model-router.ts` changes

Add to `TaskType` union:
```ts
export type TaskType =
    | "brain-dump" | "consistency" | "suggest"
    | "gap-detect" | "autocomplete" | "rerank"
    | "compact"           // Tier 2: chunk summarization
    | "session-compact";  // Tier 3: session autocompact
```

Add to `chains` record in `getModelChain`:
```ts
"compact": {
    primary: env.COMPACT_MODEL,
    fallback1: env.COMPACT_FALLBACK_1,
    fallback2: env.COMPACT_FALLBACK_2,
},
"session-compact": {
    // Reuses compact model — same capability, different prompt
    primary: env.COMPACT_MODEL,
    fallback1: env.COMPACT_FALLBACK_1,
    fallback2: env.COMPACT_FALLBACK_2,
},
```

### `src/rag/chunk-compactor.ts` (new file)

**Signature:**
```ts
export async function compactChunk(chunk: RagChunk): Promise<RagChunk>
export async function compactChunks(chunks: RagChunk[], concurrency?: number): Promise<RagChunk[]>
```

**Fast path:** if `countTokens(chunk.content) <= env.RAG_CHUNK_SUMMARY_THRESHOLD_TOKENS`,
return the chunk unchanged — no LLM call.

**Cache path:** if `getCachedSummary(chunk.content)` returns a hit, return immediately.

**Slow path:** call `callWithFallback("compact", messages, { maxTokens: 512, temperature: 0.1 })`.

Static system prompt (cache-friendly):
```
You are a lore archivist for a fantasy worldbuilding grimoire.
Summarize the following lore entry into 3-4 sentences preserving:
- entity name and type (character, location, faction, event, etc.)
- key relationships to other named entities
- most important mechanical/narrative facts
- any unresolved contradictions or open questions
Omit backstory, flavor text, and secondary details.
Output plain prose only — no markdown, no headers.
```

> **Change from v1:** Added "key relationships to other named entities" and "unresolved
> contradictions" to the preservation list. Relationship edges are the single most
> important thing in a worldbuilding knowledge graph — lose them in compaction and the
> consistency checker goes blind. Contradictions are explicitly kept because they're
> the *reason* the lore base gets queried in the first place.

On success:
- Cache the summary via `setCachedSummary`
- Return new `RagChunk` with `.content` replaced by summary + `[summarized]`
  suffix on `noteTitle` for traceability in logs.

On failure: **always fall back to original chunk**. Never let compaction failure break
the brain-dump pipeline. Log the failure with `rootLogger.warn`.

**`compactChunks`:** runs chunks through a **concurrency-limited** executor with
per-chunk error isolation:

```ts
import pLimit from "p-limit";

export async function compactChunks(
    chunks: RagChunk[],
    concurrency: number = 3,
): Promise<RagChunk[]> {
    const limit = pLimit(concurrency);
    return Promise.all(
        chunks.map((chunk) =>
            limit(() =>
                compactChunk(chunk).catch((err) => {
                    rootLogger.warn("Chunk compaction failed, using original", {
                        noteTitle: chunk.noteTitle,
                        error: err.message,
                    });
                    return chunk; // graceful fallback
                })
            )
        )
    );
}
```

> **Change from v1:** Added `p-limit` concurrency control. `Promise.all` with no
> throttling on 8 oversized chunks = 8 simultaneous LLM calls. OpenRouter rate limits
> (or your billing) will not appreciate this. Default concurrency of 3 is conservative
> — tune based on your OpenRouter tier.

**Install:** `bun add p-limit`

### `prompt.ts` integration

`buildBrainDumpPrompt` **becomes async**. This is a breaking change — update all
callers.

```ts
// Signature change:
export async function buildBrainDumpPrompt(...): Promise<{
    system: string; context: string; user: string; admittedChunks: RagChunk[];
}>
```

In `buildBrainDumpPrompt`, after the Tier 1 budget loop, before building `contextBlock`:

```ts
// Tier 2: summarize oversized chunks (parallel, failure-isolated, concurrency-limited)
const compactedChunks = await compactChunks(admittedChunks);

// Recalculate budget after compaction — compacted chunks free up space,
// potentially allowing previously-dropped chunks to be re-admitted
const postCompactBudget = compactedChunks.reduce(
    (sum, c) => sum + countTokens(c.content), 0
);
const freed = budgetUsed - postCompactBudget;
if (freed > 0) {
    rootLogger.info("Tier 2 freed tokens", { freed, postCompactBudget });
}
```

Then build `contextBlock` from `compactedChunks` instead of `admittedChunks`.

> **Open question for future iteration:** Should the freed budget trigger a second
> admission pass on the dropped chunks? Probably yes, but adds complexity. Flag for
> v3 — the ROI is marginal until the lore base exceeds ~300 notes.

---

## Tier 3 — Session State + AutoCompact

> **Cost:** full LLM call + DB writes per compaction event
> **Effort:** 1–2 days
> **Trigger:** implement alongside multi-turn route wiring (item 6.3)

### Core Idea

When multi-turn lands, session history accumulates. Without compaction, long sessions
go incoherent — the LLM loses the plot of what's being built. The session compactor
produces a structured 7-section summary, then rebuilds context methodically
post-compaction so the session feels seamless to both user and model.

### Constants

```ts
// session-compactor.ts
const SESSION_TOKEN_THRESHOLD           = env.SESSION_TOKEN_THRESHOLD; // 80,000
const COMPACT_RESERVE_TOKENS            = 13_000; // reserved buffer post-compaction
const POST_COMPACT_BUDGET               = 50_000; // working token budget after compact
const MAX_COMPACT_RETRIES               = 3;      // circuit breaker — stop after 3 failures
const MAX_RECENT_NOTES_REINJECT         = 5;
const MAX_TOKENS_PER_REINJECTED_NOTE    = 5_000;
```

> **Clarification on `POST_COMPACT_BUDGET`:** After compaction, `tokensAccumulated`
> resets to this value (not zero). This is intentional — the compacted state summary
> itself consumes ~50k tokens worth of semantic content. Setting to 0 would mean the
> next compaction triggers at a "real" 80k, which would be too late. Setting to 50k
> means compaction re-triggers after ~30k of *new* session tokens, which is the
> actual headroom you have before coherence degrades.

### `prisma/schema.prisma` additions

```prisma
model LoreSession {
  id                  String               @id @default(cuid())
  title               String?
  state               Json                 // LoreSessionState
  tokensAccumulated   Int                  @default(0)
  compactionCount     Int                  @default(0)
  compactionFailed    Int                  @default(0)  // circuit breaker counter
  lockedAt            DateTime?            // optimistic lock for concurrent compaction
  createdAt           DateTime             @default(now())
  updatedAt           DateTime             @updatedAt
  messages            LoreSessionMessage[]

  @@map("lore_sessions")
}

model LoreSessionMessage {
  id         String      @id @default(cuid())
  sessionId  String
  role       String      // "user" | "assistant" | "system"
  content    String
  tokenCount Int         @default(0)
  metadata   Json?       // { model, latencyMs, ragChunksUsed, compacted: bool }
  createdAt  DateTime    @default(now())
  session    LoreSession @relation(fields: [sessionId], references: [id], onDelete: Cascade)

  @@index([sessionId, createdAt])
  @@map("lore_session_messages")
}
```

> **Change from v1:** Added `lockedAt` field for optimistic locking. See race
> condition notes below.

Run `bun db:generate && bun db:migrate` after adding.

### Session cleanup

Sessions accumulate. Add a cleanup job or manual endpoint:

```ts
// Prune sessions older than 30 days with no activity
async function pruneStale SessionIds(): Promise<number> {
    const cutoff = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
    const { count } = await prisma.loreSession.deleteMany({
        where: { updatedAt: { lt: cutoff } },
    });
    rootLogger.info("Pruned stale sessions", { count });
    return count;
}
```

Wire this into a cron (if you have one) or expose as an admin endpoint.

### `src/types/lore.ts` additions

The 7-section session state — AllKnower's equivalent of Claude Code's 9-section summary:

```ts
export const LoreSessionStateSchema = z.object({
    // 1. What the user is trying to accomplish this session
    intent: z.string(),
    // 2. Which entity types came up
    loreTypesInPlay: z.array(LoreEntityTypeSchema),
    // 3. AllCodex note IDs created/updated this session
    noteIdsModified: z.array(z.string()),
    // 4. Things that failed and why
    skippedEntities: z.array(z.object({
        title: z.string(),
        reason: z.string(),
        errorCategory: z.string().optional(),
    })),
    // 5. Compressed summary of all user raw inputs this session
    rawInputsSummary: z.string(),
    // 6. Lore gaps or inconsistencies flagged but not resolved
    unresolvedGaps: z.array(z.string()),
    // 7. The entity currently being actively worked on
    currentFocus: z.string().optional(),
    // Metadata
    lastCompactedAt: z.string().optional(),
    totalTokensConsumed: z.number().default(0),
    // v2: version field for future schema migrations
    schemaVersion: z.literal(1).default(1),
});
export type LoreSessionState = z.infer<typeof LoreSessionStateSchema>;
```

> **Change from v1:** Added `schemaVersion`. When you inevitably add a section 8 or
> restructure the state, you'll need to migrate existing session states. Without a
> version discriminator, you're flying blind.

### `src/pipeline/session-compactor.ts` (new file)

**`shouldCompact(session: LoreSession): boolean`**

```ts
return session.tokensAccumulated >= SESSION_TOKEN_THRESHOLD
    && session.compactionFailed < MAX_COMPACT_RETRIES
    && session.lockedAt === null; // don't compact if another call is already compacting
```

The circuit breaker is critical — if compaction keeps failing (model error, schema
parse failure, etc.), `compactionFailed` increments and after 3 strikes `shouldCompact`
returns false permanently. The session degrades gracefully rather than retrying
into an infinite loop.

**Race condition protection:**

Two simultaneous requests hitting the same session can both trigger compaction,
producing duplicate or conflicting state. Use optimistic locking:

```ts
async function acquireCompactionLock(sessionId: string): Promise<boolean> {
    const staleThreshold = new Date(Date.now() - 5 * 60 * 1000); // 5min stale lock
    const result = await prisma.loreSession.updateMany({
        where: {
            id: sessionId,
            OR: [
                { lockedAt: null },
                { lockedAt: { lt: staleThreshold } }, // recover stale locks
            ],
        },
        data: { lockedAt: new Date() },
    });
    return result.count > 0;
}

async function releaseCompactionLock(sessionId: string): Promise<void> {
    await prisma.loreSession.update({
        where: { id: sessionId },
        data: { lockedAt: null },
    });
}
```

**`compactSession(session: LoreSession): Promise<LoreSessionState>`**

```ts
export async function compactSession(session: LoreSession): Promise<LoreSessionState> {
    const locked = await acquireCompactionLock(session.id);
    if (!locked) {
        rootLogger.info("Compaction skipped — lock held by another request", {
            sessionId: session.id,
        });
        throw new CompactionLockError(session.id);
    }

    try {
        // ... LLM call + validation (see below)
    } finally {
        await releaseCompactionLock(session.id);
    }
}
```

System prompt for session compaction (static, cache-friendly):
```
You are a lore session archivist for the fantasy world All Reach.
A worldbuilding session has accumulated too much context and must be compressed.
Produce a structured state object preserving ALL decisions made, ALL entities
touched, and ALL unresolved gaps. This summary REPLACES the session history —
anything omitted here cannot be recovered.

CRITICAL: Preserve relationship edges between entities. If entity A was connected
to entity B during this session, that connection MUST appear in the summary.

Return JSON matching this exact schema:
{ intent, loreTypesInPlay, noteIdsModified, skippedEntities,
  rawInputsSummary, unresolvedGaps, currentFocus }
```

> **Change from v1:** Added explicit relationship preservation instruction. The model
> will naturally summarize *entities* but silently drop *edges* between them unless
> told not to. This is the #1 failure mode in lore compaction.

User message: full message history serialized + current `session.state`.

Validate output with `LoreSessionStateSchema.parse(...)`. On parse failure:
- Increment `session.compactionFailed` in DB
- Throw — let caller handle
- Log with `task: "session-compact"` in `LLMCallLog`

On success:
- Reset `session.tokensAccumulated` to `POST_COMPACT_BUDGET`
- Increment `session.compactionCount`
- Reset `session.compactionFailed` to 0
- Persist new state to DB

> **Token reconciliation:** After every N messages (say, 10), verify
> `tokensAccumulated` against the actual sum of `tokenCount` in
> `LoreSessionMessage`. If drift exceeds 10%, log a warning and correct. Running
> counters accumulate rounding errors from the token heuristic.

**`rebuildContext(state: LoreSessionState, recentNotes: RagChunk[]): string`**

Post-compaction context rebuild. This is the **continuation message** pattern —
the most critical detail from the Claude Code architecture. Without it, the LLM
opens the next response with a recap the user doesn't want.

```ts
import { tokensToChars } from "../utils/tokens";

export function rebuildContext(
    state: LoreSessionState,
    recentNotes: RagChunk[],
    compactionCount: number,
): string {
    const noteSection = recentNotes
        .slice(0, MAX_RECENT_NOTES_REINJECT)
        .map((n) => {
            const maxChars = tokensToChars(MAX_TOKENS_PER_REINJECTED_NOTE);
            return `### ${n.noteTitle}\n${n.content.slice(0, maxChars)}`;
        })
        .join("\n\n");

    return `[SESSION COMPACTED — ${new Date().toISOString()}]
Compaction #${compactionCount} | Tokens reset to ${POST_COMPACT_BUDGET}

## Session State
${JSON.stringify(state, null, 2)}

## Recently Modified Lore (re-injected for continuity)
${noteSection}

---
You are actively building lore for the fantasy world All Reach.
Do not acknowledge this summary. Do not recap what has been done.
Continue the session exactly where it left off.`;
}
```

> **Change from v1:** Uses `tokensToChars()` instead of raw `* 4` multiplication.
> Consistent with the centralized token counting infrastructure.

The last three lines are non-negotiable. Every word matters.

---

## Observability

> **Effort:** ~30 min, sprinkle across all tiers
> **Impact:** you will thank yourself within one week of production use

### Structured metrics to emit

Add these to your existing `LLMCallLog` or a dedicated compaction log:

```ts
interface CompactionMetrics {
    tier: 1 | 1.5 | 2 | 3;
    // Tier 1
    chunksOffered?: number;
    chunksAdmitted?: number;
    chunksDropped?: number;
    budgetUtilization?: number; // 0-1, how full the budget got
    // Tier 1.5
    chunksDeduped?: number;
    // Tier 2
    chunksSummarized?: number;
    chunksCacheHit?: number;
    tokensFreed?: number;
    compactModelUsed?: string;
    compactLatencyMs?: number;
    // Tier 3
    sessionId?: string;
    preCompactTokens?: number;
    postCompactTokens?: number;
    compactionNumber?: number;
    lockContention?: boolean;
}
```

Even if you don't have a metrics backend yet, logging these as structured JSON means
you can `grep` and `jq` your way to insights. When you inevitably hook up Prometheus
or similar, the shape is already there.

---

## Testing Strategy

### Tier 1 — Deterministic, unit-testable

```ts
// test/rag-budget.test.ts
describe("RAG budget enforcement", () => {
    it("admits chunks up to budget and skips oversized", () => {
        const chunks = [
            mockChunk("Location A", 2000),  // admitted
            mockChunk("Event B", 5000),     // skipped (exceeds remaining)
            mockChunk("Character C", 500),  // admitted (fits in gap)
        ];
        const result = admitChunks(chunks, 3000);
        expect(result.map(c => c.noteTitle)).toEqual(["Location A", "Character C"]);
    });

    it("logs dropped chunks with correct metadata", () => { /* ... */ });
});
```

### Tier 2 — Integration test with mock LLM

```ts
describe("Chunk compaction", () => {
    it("preserves entity relationships in summary", async () => {
        const chunk = mockChunk("Zaheer", 2000, "Zaheer is the Archon of Übermenschreich. He has a blood pact with Constantin...");
        const result = await compactChunk(chunk);
        // The summary MUST mention both Zaheer and Constantin
        expect(result.content).toContain("Zaheer");
        expect(result.content).toContain("Constantin");
    });

    it("falls back to original on LLM failure", async () => { /* ... */ });
    it("returns cached summary on second call", async () => { /* ... */ });
});
```

### Tier 3 — Snapshot testing

Capture real session histories, run compaction, and snapshot the output. Review
snapshots manually for lore fidelity. This isn't automatable — you need human
eyes to verify that "the Allmaker's intent was preserved."

---

## Implementation Order

```
Day 1     Tier 0 — tokens.ts utility + tiktoken install (~20 min)
          Tier 1 — env.ts + prompt.ts changes (~30 min)
          Ship immediately, observe RAG drop logs

Day 2     Tier 1.5 — chunk-dedup.ts (~1 hour)
          Observe dedup rates in logs

Week 1    Tier 2 — chunk-compactor.ts + model-router task types + summary cache (~2–3 hours)
          Run brain dumps, compare token counts before/after in LLMCallLog

When      Tier 3 — prisma migration + session-compactor.ts + optimistic locking (~1–2 days)
ready     Wire into routes when multi-turn API surface is designed
```

Tier 3 deliberately has no route wiring in this plan — that's a separate concern.
`session-compactor.ts` is a pure service module callable from whatever route
structure multi-turn ends up using.

---

## Key Design Principles (from Claude Code's architecture)

1. **Cache hits are everything.** Every architectural decision should maximize
   prompt cache utilization. Static content first, dynamic content last.

2. **Never let compaction break the pipeline.** Tier 1 drops silently. Tier 1.5
   dedupes silently. Tier 2 falls back to original chunk. Tier 3 has a circuit
   breaker + optimistic lock. Compaction is a quality-of-life feature, not a
   critical path dependency.

3. **The continuation message is non-negotiable.** Post-compaction, the model
   MUST be told to continue without acknowledging the compaction. Without this,
   every compaction event produces a disruptive UX moment.

4. **Surgical over nuclear.** Prefer dropping/summarizing individual items over
   rewriting the full message array. Cache invalidation costs 1.25x write rate
   vs 90% discount on cache hits — the economics dictate the architecture.

5. **Structured summaries, not free-form.** Claude Code's 9-section summary is
   machine-parseable. AllKnower's 7-section `LoreSessionState` is Zod-validated.
   Free-form summaries are untrustworthy at scale.

6. **Relationships are first-class citizens.** Entity names are cheap to preserve.
   Edges between entities are not — they require explicit instruction to the
   compaction model, explicit testing, and explicit monitoring. A compacted lore
   state that loses relationship edges is worse than no compaction at all.

7. **Pessimistic token counting.** When the heuristic is wrong, be wrong in the
   direction that *under-admits* rather than *over-admits*. A dropped low-relevance
   chunk is invisible. A blown context window is a hard failure.

---

## Changelog (v1 → v2)

| Area | Change | Rationale |
|---|---|---|
| Token counting | Centralized `tokens.ts` with tiktoken + pessimistic fallback | `chars/4` drifts 15–25% on fantasy proper nouns |
| Tier 1 budget loop | `break` → `continue` (best-fit scan) | Small relevant chunks were killed behind large ones |
| Tier 1.5 (new) | Semantic dedup via Jaccard trigrams | Overlapping chunks waste budget on redundant context |
| Tier 2 prompt | Added relationship + contradiction preservation | Edges between entities are the #1 compaction loss |
| Tier 2 cache | In-memory LRU summary cache | Without it, same chunk re-summarized every call |
| Tier 2 concurrency | `p-limit` throttle (default 3) | Unbounded `Promise.all` hammers OpenRouter rate limits |
| Tier 2 integration | `buildBrainDumpPrompt` becomes async | Necessary for `await compactChunks()` |
| Tier 3 locking | `lockedAt` + optimistic lock | Race condition: two requests compacting same session |
| Tier 3 cleanup | Session pruning (30-day TTL) | Unbounded session accumulation in DB |
| Tier 3 schema | Added `schemaVersion` to `LoreSessionState` | Future-proofs state migration |
| Tier 3 system prompt | Explicit relationship preservation instruction | Models drop edges unless told not to |
| Tier 3 rebuild | Uses `tokensToChars()` instead of `* 4` | Consistent with centralized token infrastructure |
| Observability | `CompactionMetrics` interface | Structured logging for all tiers |
| Testing | Strategy per tier | Tier 1 unit, Tier 2 integration, Tier 3 snapshot |
| Cache topology | Note on OpenRouter passthrough behavior | `cache_control` only works Anthropic-side |