---
name: Update from code changes
type: source-code-agent
on:
  push:
    branches:
      - main
context:
  - repo: thunderronin/allcodex-aio
automerge: false
---
Review changes pushed to the `allcodex-aio` repository, specifically:
1. Lore type schemas in `allknower/src/types/lore.ts`.
2. Portal API route handlers or client libs under `allcodex-portal/lib/`.
3. Express server routes or database structures in `allcodex-core/`.

If any entity fields, API endpoints, environment variables, or setup instructions have been modified, locate the corresponding documentation files under:
- `reference/lore-schema.mdx`
- `reference/portal-api.mdx`
- `self-hosting/configuration.mdx`

Propose precise updates to keep the documentation fully aligned with the codebase. Write in a clear, concise developer style.
