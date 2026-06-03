# AllCodex Shared Documentation

Canonical docs for the [AllCodex ecosystem](../../AGENTS.md) — read by all three services and all AI agents.

These files are checked out as a git submodule (`ThunderRonin/allcodex-docs`) in:
- `allcodex-core/docs/shared/`
- `allknower/docs/shared/`
- `allcodex-portal/docs/shared/`

---

## Reference — Canonical Technical Docs

| File | Purpose |
|---|---|
| [reference/architecture.md](reference/architecture.md) | Full ecosystem architecture: services, data flows, internals, Mermaid diagram |
| [reference/canonical-lore-schema.md](reference/canonical-lore-schema.md) | 21 lore entity types — single source of truth for AllKnower and Portal |
| [reference/portal-api-reference.md](reference/portal-api-reference.md) | Complete Portal API route and method reference |

## Planning — Roadmaps & Feature Work

| File | Purpose |
|---|---|
| [planning/ROADMAP.md](planning/ROADMAP.md) | DM-first feature roadmap — Phases 0–4 ✅, Phase 5 ⚠️ partial |
| [planning/v1_final_hit_plan.md](planning/v1_final_hit_plan.md) | Final v1 release checklist and verification gates |
| [planning/implementation_plan_phases_0_3.md](planning/implementation_plan_phases_0_3.md) | Detailed implementation plan for Phases A–G (all complete) |
| [planning/allcodex_dm_first_scope.md](planning/allcodex_dm_first_scope.md) | Original product scope definition (still valid) |
| ~~planning/remaining-features-plan.md~~ | Moved to archive (all features shipped) |

## Analysis — Audits & Trackers

| File | Purpose |
|---|---|
| [analysis/gap_analysis_vs_worldanvil.md](analysis/gap_analysis_vs_worldanvil.md) | World Anvil parity: P0–P8 shipped, visual maps/per-user ACL/embedded graphs/onboarding remaining |
| [analysis/worldanvil_feature_matrix.md](analysis/worldanvil_feature_matrix.md) | Feature-by-feature audit against World Anvil pages |
| [analysis/context_compaction_plan.md](analysis/context_compaction_plan.md) | AllKnower AI/RAG improvement backlog (future work) |
| [analysis/hybrid-search-hardening.md](analysis/hybrid-search-hardening.md) | Shipped hybrid RAG tuning knobs, FTS health tracking, and verification notes |
| [analysis/progressive-brain-dump-streaming.md](analysis/progressive-brain-dump-streaming.md) | Shipped progressive brain-dump entity rendering and SSE keepalive behavior |
| [analysis/qa-report-2026-05-25.md](analysis/qa-report-2026-05-25.md) | QA validation for hybrid search and progressive brain-dump streaming |

## Release History

| File | Purpose |
|---|---|
| [v1_release_changelog.md](v1_release_changelog.md) | v1 release summary — architecture boundary, user-scoped credentials, auto-provisioning, code quality |

## Archive

| File | Purpose |
|---|---|
| [archive/allcodex_dm_first_roadmap.md](archive/allcodex_dm_first_roadmap.md) | 📦 Archived — superseded by planning/ROADMAP.md |
| [archive/remaining-features-plan.md](archive/remaining-features-plan.md) | 📦 Archived — all three tracked features shipped |

---

## Quick Links

- [AGENTS.md](../../AGENTS.md) — root agent guide (commands, conventions, key files, pitfalls)
- [allcodex-core/CLAUDE.md](../../allcodex-core/CLAUDE.md) — AllCodex Core deep dive
- [allknower/docs/ai_architecture_investigation.md](../../allknower/docs/ai_architecture_investigation.md) — RAG/AI improvement research (Refer to AllKnower's `.env.example` for current LLM model preferences)
- [allcodex-portal/ROADMAP.md](../../allcodex-portal/ROADMAP.md) — portal-specific feature backlog
