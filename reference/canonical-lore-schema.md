# Canonical Lore Schema — AllCodex Ecosystem

Single source of truth for all label, attribute, and relation conventions across AllCodex Core, AllKnower, and AllCodex Portal.

Last updated: 2026-04-02

---

## Lore Types

All lore types are expressed via the `#loreType` label on a note.

| Type | Template ID | Description |
|---|---|---|
| `character` | `_template_character` | NPCs, PCs, historical figures |
| `location` | `_template_location` | Cities, dungeons, regions, landmarks |
| `faction` | `_template_faction` | Organizations with goals and leadership |
| `creature` | `_template_creature` | Monsters, animals, entities with stat hints |
| `event` | `_template_event` | Historical or in-world events |
| `timeline` | `_template_timeline` | Ordered collection of events |
| `manuscript` | `_template_manuscript` | In-world written works |
| `statblock` | `_template_statblock` | D&D/TTRPG-style stat blocks |
| `item` | `_template_item` | Magic items, artifacts, equipment |
| `spell` | `_template_spell` | Spells and magical abilities |
| `building` | `_template_building` | Structures — inns, towers, temples |
| `language` | `_template_language` | Languages and writing systems |
| `organization` | `_template_organization` | Guilds, governments, clubs |
| `race` | `_template_race` | Playable and non-playable races/species |
| `myth` | `_template_myth` | Legends, myths, folklore |
| `cosmology` | `_template_cosmology` | Planes of existence, cosmic structures |
| `deity` | `_template_deity` | Gods, demigods, divine beings |
| `religion` | `_template_religion` | Faiths, cults, religious orders |
| `session` | `_template_session` | Play session records |
| `quest` | `_template_quest` | Quests, bounties, story hooks |
| `scene` | `_template_scene` | Individual scenes within a session |

---

## Canonical Labels

### Classification Labels

| Label | Values | Description |
|---|---|---|
| `#lore` | (valueless) | Marks any note as a lore entry — required for indexing |
| `#loreType` | see Lore Types above | The specific type of lore note |
| `#draft` | (valueless) | Note is unpublished — hidden from share output |
| `#gmOnly` | (valueless) | Note is GM-only — stripped from player share views |
| `#quest` | (valueless) | Marks a note as a quest/hook (on quest-type notes) |
| `#statblock` | (valueless) | Marks a note as a statblock (used for quick search) |
| `#session` | (valueless) | Marks a note as a session record |

### Sharing Labels

| Label | Values | Description |
|---|---|---|
| `#shareRoot` | (valueless) | Designates the subtree root for public sharing |
| `#shareAlias` | slug string | Custom URL slug, e.g. `blackstone-keep` |
| `#shareCredentials` | `user:password` | Basic Auth for password-protected shares |
| `#shareHiddenFromTree` | (valueless) | Hides from share navigation tree |
| `#shareIndex` | (valueless) | Marks as the index/landing page of a share |
| `#shareDisallowRobotIndexing` | (valueless) | Sets `noindex` on the share page |

### Provenance Labels

| Label | Values | Description |
|---|---|---|
| `#brainDumpId` | UUID | References the `brainDumpHistory.id` that created/updated this note |
| `#importSource` | string | Source identifier for imported notes (e.g., `azgaar`, `system-pack`) |

---

## Promoted Attribute Keys by Lore Type

### Character
`fullName`, `aliases`, `age`, `race`, `gender`, `affiliation`, `role`, `status`, `secrets`, `goals`

### Location
`locationType`, `region`, `population`, `ruler`, `secrets`, `geolocation`

### Faction
`factionType`, `foundingDate`, `leader`, `goals`, `secrets`, `status`

### Creature
`creatureType`, `habitat`, `diet`, `dangerLevel`, `abilities`

### Event
`inWorldDate`, `outcome`, `consequences`, `secrets`

### Timeline
`calendarSystem` — sort order driven by `#sorted` + `#sortBy=inWorldDate`

### Manuscript
`genre`, `manuscriptStatus`, `wordCount`

### Statblock
`crName`, `challengeRating`, `crLevel`, `creatureType`, `size`, `alignment`, `ac`, `hp`, `speed`, `str`, `dex`, `con`, `int`, `wis`, `cha`, `immunities`, `resistances`, `vulnerabilities`, `abilities`, `actions`, `legendaryActions`

### Item
`itemType`, `rarity`, `creator`, `magicProperties`, `history`

### Spell
`school`, `level`, `castingTime`, `range`, `components`, `duration`

### Building
`buildingType`, `owner`, `purpose`, `condition`, `secrets`

### Language
`languageFamily`, `speakers`, `script`, `samplePhrase`

### Organization
`orgType`, `purpose`, `foundingDate`, `leader`, `headquarters`, `status`, `secrets`

### Race
`racialType`, `homeland`, `physicalTraits`, `culture`, `languages`, `lifespan`, `abilities`

### Myth
`mythType`, `origin`, `tellers`, `truthBasis`, `significance`, `secrets`

### Cosmology
`domain`, `laws`, `source`, `planes`

### Deity
`domains`, `alignment`, `rank`, `symbol`, `worshippers`

### Religion
`deity`, `pantheon`, `tenets`, `clergy`, `holyDays`, `headquarters`

### Session
`sessionDate`, `players`, `sessionStatus`, `recap`, `hooks`, `gmNotes`

### Quest
`questStatus` (`active` | `completed` | `failed` | `deferred`), `questGiver`, `reward`, `location`, `hooks`, `consequences`

### Scene
`location`, `participants`, `outcome`, `gmNotes`

---

## Relation Types

These are AllCodex relation (`~`) attributes used by AllKnower's auto-relate pipeline.

| Relation | From → To | Description |
|---|---|---|
| `~template` | note → template | Links a note to its template |
| `~character` | note → character | References a character |
| `~location` | note → location | References a location |
| `~faction` | note → faction | References a faction |
| `~session` | scene/quest → session | Links scene or quest to its session |
| `~questSource` | quest → character/faction | Quest giver relation |
| `~relatedNote` | any → any | Generic cross-reference |

---

## AllKnower TEMPLATE_ID_MAP

Maps AllKnower `LoreEntityType` values to AllCodex Core template IDs:

```typescript
{
  character:    "_template_character",
  location:     "_template_location",
  faction:      "_template_faction",
  creature:     "_template_creature",
  event:        "_template_event",
  timeline:     "_template_timeline",
  manuscript:   "_template_manuscript",
  statblock:    "_template_statblock",
  item:         "_template_item",
  spell:        "_template_spell",
  building:     "_template_building",
  language:     "_template_language",
  organization: "_template_organization",
  race:         "_template_race",
  myth:         "_template_myth",
  cosmology:    "_template_cosmology",
  deity:        "_template_deity",
  religion:     "_template_religion",
  session:      "_template_session",
  quest:        "_template_quest",
  scene:        "_template_scene",
}
```

---

## Notes on HTML Classes

AllCodex Core's share renderer recognizes these CSS classes in note HTML:

- `.gm-only` — element stripped from player-facing share output (complement to `#gmOnly` label)
- `.lore-content` — applied by the Portal's lore detail view
- `.spoiler` — allcodex-specific class for spoiler blocks
