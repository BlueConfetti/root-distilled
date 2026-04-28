---
name: root-cards
description: Use when looking up Root cards, faction abilities, hirelings, vagabonds, landmarks, clockwork bot traits, or anything from the 13 card YAMLs. Indexes the data by faction, expansion, and ID.
user-invocable: false
allowed-tools:
  - "Read"
  - "Grep"
---

# Root Cards Index

Reference recipe for navigating the 13 card YAMLs. Citation conventions are defined
in the project `CLAUDE.md` — do not restate them; just use them.

## Card Schema

Every card is a flat YAML object:

| Field | Required | Notes |
|---|---|---|
| `id` | yes | `ROOT-N` (N is a decimal integer 0..302) |
| `name` | yes | Display name |
| `image` | yes | Image identifier |
| `tags` | yes | List of strings (expansion + faction/category) |
| `text` | optional | Card body text. **Absent** on landmark cards (image-only) |
| `imageClass` | optional | Hirelings only; observed value: `rotated` |

## File Map (303 cards across 13 files)

| File | Cards | ID range | Primary content |
|---|---|---|---|
| `rootbasegame.yml` | 72 | ROOT-0..ROOT-163 | Eyrie leaders, Vagabond classes & quests, Marquise/Woodland abilities, base deck |
| `exiles-partisans.yml` | 46 | ROOT-57..ROOT-227 | Alternate deck (Ambush!, Dominance, Craftables) |
| `riverfolk.yml` | 11 | ROOT-22..ROOT-161 | Riverfolk Company faction cards |
| `riverfolk-hirelings.yml` | 8 | ROOT-167..ROOT-195 | Riverfolk-flavored hirelings |
| `underworld.yml` | 9 | ROOT-4..ROOT-12 | Underworld factions (Moles) |
| `underworld-hirelings.yml` | 8 | ROOT-164..ROOT-191 | Underworld-flavored hirelings |
| `marauders.yml` | 19 | ROOT-13..ROOT-277 | Marauder faction content (Rats, etc.) |
| `marauders-hirelings.yml` | 20 | ROOT-165..ROOT-199 | Marauder-flavored hirelings |
| `vagabonds.yml` | 3 | ROOT-28..ROOT-30 | Expansion vagabond classes |
| `clockwork1.yml` | 31 | ROOT-110..ROOT-284 | Clockwork bot trait cards (set 1) |
| `clockwork2.yml` | 47 | ROOT-228..ROOT-287 | Clockwork bot trait cards (set 2) |
| `landmarks.yml` | 14 | ROOT-138..ROOT-151 | Landmark setup + active cards (no `text`) |
| `homeland.yml` | 15 | ROOT-288..ROOT-302 | Homeland map vagabond content (Captains) |

**ID ranges overlap across files** — `ROOT-23` appears in multiple files. Always
include the deck name in citations when ambiguity is possible.

## Tag Taxonomy (27 unique values)

**Expansion / deck**: Base Game · Exiles & Partisans Deck · Riverfolk · Underworld ·
Marauder / Marauders · Vagabond · Homeland · Landmarks · Clockwork · ADSET ·
Advanced Setup · Standard Deck · Learn to Play · Mechanical Marquise 1.0

**Faction / class**: Eyrie · Moles · Rats · Captain (also: Bird / Bunny / Fox / Mouse — dual role: suit tag in base, faction-suit-variant tag in Exiles)

**Category**: Craftable · Quest · Hirelings

A single card has 2-3 tags. Use tags to filter (e.g., everything tagged `Eyrie` is
in `rootbasegame.yml`).

## Faction → File Routing

| Faction / category | Primary file | Secondary | Tags |
|---|---|---|---|
| Eyrie cards | `rootbasegame.yml` | `clockwork1.yml`, `marauders-hirelings.yml` | `[Base Game, Eyrie]` |
| Marquise cards | `rootbasegame.yml` | `marauders-hirelings.yml` | `[Base Game]` |
| Woodland Alliance | `rootbasegame.yml` | — | `[Base Game]` |
| Vagabond classes | `rootbasegame.yml`, `vagabonds.yml`, `homeland.yml` | `riverfolk.yml`, `underworld.yml` | `[Base Game, Vagabond]`, `[Vagabond]`, `[Homeland, Vagabond]` |
| Riverfolk Company | `riverfolk.yml` | `riverfolk-hirelings.yml` | `[Riverfolk]` |
| Underground Duchy | `underworld.yml` | `underworld-hirelings.yml` | `[Underworld, Moles]` |
| Marauder factions | `marauders.yml` | `marauders-hirelings.yml` | `[Marauders]` |
| Exiles & Partisans deck | `exiles-partisans.yml` | — | `[Exiles & Partisans Deck, ...]` |
| Clockwork bots | `clockwork1.yml`, `clockwork2.yml` | — | `[Clockwork]` |
| Landmarks | `landmarks.yml` | — | `[Landmarks]` |
| Homeland | `homeland.yml` | — | `[Homeland, ...]` |

## Card Collision Set (deck-required citations)

These names exist in BOTH `rootbasegame.yml` and `exiles-partisans.yml`. When citing
any of them, the deck must be named:

- **Ambush!** — 4 base (suit variants), 4 Exiles (ROOT-200..ROOT-203)
- **Dominance** — 4 base (suit variants), 4 Exiles (ROOT-211..ROOT-214)
- **Crossbow** — 2 base, 2 Exiles (ROOT-209, ROOT-210)
- **Anvil** — base + Exiles (ROOT-204)
- **Arms Trader** — base + Exiles (ROOT-205)
- **A Visit To Friends** — base + Exiles (ROOT-206)
- **Bake Sale** — base + Exiles (ROOT-207)
- **Birdy Bindle** — base + Exiles (ROOT-208)

13 ID instances, 8 unique names. No collisions outside the base ↔ Exiles & Partisans pair.

## Lookup Recipes

**Look up a card by ID** (when you know the deck):
```
grep -n "id: ROOT-23" cards/content/card-data/root/en-US/<file>.yml
```

**Find a card by name across all decks** (essential for collision detection):
```
grep -nE "name: '?<Name>" cards/content/card-data/root/en-US/*.yml
```
If the result has multiple files, surface deck disambiguation in the answer.

**List every card belonging to a faction or category**:
```
grep -B1 -A4 -E "tags:" cards/content/card-data/root/en-US/<file>.yml | grep -B5 "<TagName>"
```
Or just Read the file and filter by tag.

**List every Eyrie leader** (concrete recipe):
```
grep -B5 "Eyrie" cards/content/card-data/root/en-US/rootbasegame.yml | grep -E "name|text|id"
```

## Output Contract for Callers

- Quote card text verbatim when adjudicating its effect
- Cite `card:ROOT-N` with deck name whenever the ID falls in the collision set, or whenever the name alone is ambiguous
- For landmarks (no `text` field), describe what's known and note the absence; cross-reference Landmarks rules in `rules.yml`
- For hirelings, note the `imageClass: rotated` flag if relevant (it indicates the demoted/promoted side)
- Rules-correctness on card effects is the referee's job — when in doubt about a card-rules interaction, verify against `rules.yml` directly via Read/Grep
