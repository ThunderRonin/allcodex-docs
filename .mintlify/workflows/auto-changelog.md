---
name: Draft changelog
type: changelog
on:
  push:
    branches:
      - main
context:
  - repo: thunderronin/allcodex-aio
automerge: false
---
Review all commits and pull requests merged into the `allcodex-aio` repository. 

Draft a new changelog post detailing the new features, bug fixes, or performance improvements. 
Format it nicely to fit the grimoire TTRPG style of AllCodex, and append it to `changelog/overview.mdx`.
