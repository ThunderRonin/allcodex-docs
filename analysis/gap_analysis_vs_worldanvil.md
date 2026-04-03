# AllCodex vs World Anvil — Gap Analysis

> What essentials are we missing, how far are we from them, and what can be skipped.

---

## How to Read This Document

Each World Anvil feature is categorized as:
- 🔴 **Essential Gap** — Core worldbuilding feature we lack. Needed for parity.
- 🟡 **Nice-to-Have** — Useful but deferrable. Our AI capabilities compensate.
- ⚪ **Skip** — Not aligned with our architecture or philosophy.

Effort is rated: **S** (days), **M** (1–2 weeks), **L** (3+ weeks).

---

## 🔴 Essential Gaps

### 1. Rich Text Editor (WYSIWYG Authoring Experience) ✅ Shipped

| Aspect | World Anvil | AllCodex |
|---|---|---|
| Inline formatting | Highlight-to-format toolbar (bold, italic, color, dice, anchors) | BlockNote `LoreEditor` with formatting toolbar (bold, italic, underline, strike, color, highlight) |
| Slash commands | `/` menu for headers, tables, line breaks, spoilers, quotes | `/` command palette (headings, blockquote, table, image, divider) |
| Markdown triggers | `---`, `#`, etc. auto-convert | Supported via BlockNote's input rules |
| Drag-and-drop images | Drag file onto editor → embedded | Image upload via BlockNote file block |
| `/image` command | Inline image embed from hosted assets | Available via slash command |

**Status: ✅ Shipped.** `LoreEditor.tsx` (BlockNote) is the default editor across the create and edit flows.

**Previously:** `L` (3+ weeks). Required selecting and integrating a block editor, wiring it to ETAPI, and building an image upload pipeline.

---

### 2. Interlinking / `@`-mention System ✅ Shipped

| Aspect | World Anvil | AllCodex |
|---|---|---|
| `@` mentions | Type `@` + 3 letters → article picker → inline link | `mention-extension.tsx` — type `@` → fuzzy search via ETAPI → inserts inline link |
| Autolinker | `/autolinker` scans text for matching article titles → auto-links | `AutolinkerDialog.tsx` — scans for note title matches and injects links |
| Tooltip previews | Hover over a linked article → tooltip with summary | Hover tooltip with note title and type badge |

**Status: ✅ Shipped.** `mention-extension.tsx` handles `@` autocomplete; `AutolinkerDialog` scans for unlinked title matches.

---

### 3. Template Selection UX (Guided Creation Flow) ✅ Shipped

| Aspect | World Anvil | AllCodex |
|---|---|---|
| Template browser | Visual menu with descriptions per template on hover | `TemplatePicker` modal with icon grid, description cards, and hover previews |
| Template-specific prompts | Each template shows inspirational prompts ("5 Ws") | Promoted attributes rendered as a dedicated form section via `PromotedFields.tsx` |
| Template count | 25+ specialized templates | 21 templates (all lore types) + General Lore — 22 total options in the Portal picker |

**Status: ✅ Shipped.** `TemplatePicker.tsx` is the creation flow entry point. `PromotedFields.tsx` renders template-specific form fields. All 21 AllKnower entity types now have AllCodex Core templates and Portal picker entries. Only `timeline` in Portal picker is pending.

---

### 4. Article Categories / Hierarchical Organization ✅ Shipped

| Aspect | World Anvil | AllCodex |
|---|---|---|
| Categories | Folder-like hierarchy; drag-and-drop reorder; nested sub-categories | `LoreTree.tsx` — collapsible hierarchy in the lore sidebar, built from AllCodex branches |
| Drag-and-drop | Drag articles into categories, drag categories into each other | Drag-and-drop branch reparenting via ETAPI |
| Tree navigation | Filterable sidebar tree | `LoreTree` with `selectedId` filtering in the lore grid |

**Status: ✅ Shipped.** `LoreTree.tsx` renders the full AllCodex branch tree in the lore sidebar with filtering support.

---

### 5. Draft vs. Published Lifecycle ✅ Shipped

| Aspect | World Anvil | AllCodex |
|---|---|---|
| Draft toggle | Orange circle → Publish slider (Draft/Published) | `isDraft` toggle in the edit toolbar — adds/removes the `#draft` label via ETAPI |
| Visual indicator | Clear UI badge showing draft state | Orange "Draft" badge on cards in the lore grid |

**Status: ✅ Shipped.** `#draft` label convention is enforced. The share renderer hides `#draft` notes. The edit page exposes a toggle button that adds/removes the attribute via ETAPI.

---

## 🟡 Nice-to-Have (Deferrable)

### 6. Interactive Maps

| Aspect | World Anvil | AllCodex |
|---|---|---|
| Interactive maps | Upload image → pin locations → link to articles | Azgaar FMG import is **shipped** (data pipeline + Portal UI), but no visual map viewer yet |

**Why deferrable:** Maps are visually impressive but complex to build (Leaflet/MapLibre integration, marker management, zoom levels). The Azgaar import pipeline and Portal UI are shipped (`POST /import/azgaar` with preview mode). A visual map viewer is a `L` effort that can come later.

**Effort if pursued:** `L` (3+ weeks)

---

### 7. Timelines (Visual Chronological View) ✅ Shipped

| Aspect | World Anvil | AllCodex |
|---|---|---|
| Visual timeline | Interactive chronological view with event cards | `/timeline` page with chronological event list, in-world date display, and sort controls |

**Status: ✅ Shipped.** The `/timeline` page renders events chronologically with in-world dates. Timeline template (`_template_timeline`) plus Event template (`_template_event`) provide the data structure.

---

### 8. Public/Private Toggle (Granular Sharing) ✅ Shipped

| Aspect | World Anvil | AllCodex |
|---|---|---|
| Per-article padlock | Public/Private toggle per article + per-user access | `ShareSettings.tsx` — per-note share toggle, password protection, GM vs. player preview via `PreviewToggle.tsx`. `/shared` browser lists all shared content. |

**Status: ✅ Shipped.** `ShareSettings` manages share visibility per note. `PreviewToggle` lets the GM switch between GM and player views. The `/shared` page browses all shared content with toggle controls. Per-user access control (individual player permissions) remains a future enhancement.

---

### 9. Complex Content Embedding (Family Trees, Diplomacy Webs)

| Aspect | World Anvil | AllCodex |
|---|---|---|
| Embeddable widgets | Family trees, diplomacy webs, interactive maps inline in articles | Note relations exist but no visual graph rendering in articles |

**Why deferrable:** AllKnower's relationship suggester generates the *data*. Visualizing it as an inline embed (force-directed graph or tree) is a frontend enhancement, not a structural gap.

**Effort if pursued:** `M`

---

### 10. Onboarding / Guided Tour

| Aspect | World Anvil | AllCodex |
|---|---|---|
| Onboarding wizard | Bottom widget + 10-lesson course + Writer/GM workflows | None |

**Why deferrable:** Onboarding is a growth feature. Until the editor and core UX are solid, onboarding is premature. A simple "Getting Started" page or tooltip tour (Shepherd.js) is a `S` effort when the time comes.

**Effort if pursued:** `S–M`

---

## ⚪ Skip / Not Applicable

| World Anvil Feature | Why Skip |
|---|---|
| **RPG Campaigns** (session tools, encounter builders) | Partially shipped. Session workspace, quest tracker, and scene templates exist. Full encounter builder remains out of scope. |
| **Dice Roll Buttons** (inline `[dice:2d6]`) | Niche RPG feature. Not aligned with author/worldbuilder focus. |
| **Custom Text Colors** (per-word coloring in editor) | Low value-add with effort. Standard formatting covers needs. |
| **"5 Ws" Heuristic** (integrated prompting philosophy) | Our Brain Dump AI does this better — it extracts structure from raw text instead of asking the user to fill in fields. |
| **Documentation / "Learn" button** | Standard docs site — not a product feature to replicate. Will be needed eventually but is not a gap. |

---

## Summary: Priority Roadmap

| Priority | Feature | Effort | Status | Depends On |
|---|---|---|---|---|
| **P0** | Rich Text Editor (Tiptap — `LoreEditor`) | `L` | ✅ Shipped | — |
| **P1** | `@`-mention interlinking + autolinker | `M` | ✅ Shipped | Rich Editor |
| **P2** | Template Selection UX + missing templates | `S–M` | ✅ Shipped (15 templates) | — |
| **P3** | Category tree / hierarchical nav in Portal | `M` | ✅ Shipped | — |
| **P4** | Draft/Published lifecycle toggle | `S` | ✅ Shipped | — |
| **P5** | Visual Timeline | `M` | ✅ Shipped | — |
| **P6** | Public/Private sharing (note-level) | `M` | ✅ Shipped | — |
| **P7** | Statblock Library + System Pack Import | `M` | ✅ Shipped | — |
| **P8** | Session Workspace + Quests | `M` | ✅ Shipped | — |
| — | Interactive Maps (visual layer) | `L` | 🔲 Pending | Azgaar Import |
| — | Azgaar Import (data layer) | `M` | 🔲 Pending | — |
| — | Per-user access control | `L` | 🔲 Pending | Sharing |
| — | Embedded graphs (family tree, diplomacy) | `M` | 🔲 Pending | Relationship data |
| — | Onboarding wizard | `S–M` | 🔲 Pending | Stable UX |

> [!NOTE]
> **P0–P8 are all shipped.** The essential authoring, session runtime, and player sharing gaps are closed. Remaining items are maps, per-user access, embedded graphs, and onboarding.

---

## Where AllCodex Is Ahead (Not Asked, But Context)

For perspective: World Anvil has *zero* AI features. AllCodex's Brain Dump, RAG search, consistency checking, gap detection, and relationship suggestion are capabilities World Anvil doesn't offer at any tier. Our gap is in **authoring UX**, not in intelligence.
