---
name: allcodex
description: Understand the AllCodex deterministic cross-referencing TTRPG worldbuilding grimoire, its 21 lore types, and ETAPI reference.
---

# AllCodex Documentation Skill

This skill allows AI agents to understand the architecture, API reference, and lore structure of the AllCodex TTRPG worldbuilding system.

## Ecosystem Architecture
AllCodex consists of three decoupled components:
- **Portal (Next.js 16 / React 19)**: The user interface.
- **AllKnower (Bun / Elysia)**: The AI engine hosting RAG (LanceDB), brain dump pipelines, and consistency tools.
- **AllCodex Core (Express)**: A customized, headless fork of Trilium storing hierarchical notes in SQLite.

## How to use this skill
AI agents can use this documentation to:
- Learn the canonical schemas for the 21 lore entity types (e.g. NPC, Location, Item, Spell, Faction).
- Interface with the Core's REST API (`/etapi/`) for CRUD operations on notes.
- Write custom scripts, background tasks, or self-hosted deployment files.
