# Root Domain-Expert Tools

This project hosts structured data for the board game **Root** (Leder Games) and a
set of Claude Code components that turn this directory into a master-level domain
expert covering rules adjudication, card/faction lookup, strategy analysis, post-mortem
coaching, content drafting, and pattern-finding synthesis.

## Data Locations

- **Cards** (303 cards across base game and all expansions, schema `{id, name, image, tags[], text?, imageClass?}`):
  `cards/content/card-data/root/en-US/*.yml` — 13 files
- **Law of Root** — three printings live side-by-side under `rules/content/rules/root/en-US/`:
  - `p16/rules.yml` — printing 16, **Oct 8, 2025** (current canonical Law). Covers all 13 factions plus Hirelings, Landmarks, Maps, Knave Captains, Components, Glossary.
  - `p6/rules.yml` — printing 6, earlier revision. Covers 10 factions (no Lilypad Diaspora, Twilight Council, or Knaves of the Deepwood). Includes Hirelings, Landmarks, Variant Maps.
  - `p4/rules.yml` — printing 4, oldest revision. Covers 8 factions (no Lord of the Hundreds, Keepers in Iron, or any p16-only faction). No Hirelings/Landmarks sections.
  - All three use the same nested `name`/`text`/`children` tree schema.
- **FAQ** (entries shaped `{q, a, rules[]}`):
  - `p16/faq.yml` — current canonical FAQ.
  - `p6/faq.yml` — older FAQ (nearly identical to p16; differs in only a few rule references).
  - `p4/` has no FAQ file.
- **App config** (display dates, locale strings):
  `rules/content/rules/root/en-US/appconfig.yml`
- **PDF reference** (cross-check only, not consumed by tools):
  `Root_Base_Law_Oct_2025.pdf`

### Which printing to cite

Default to **p16** for any rules question — it is the current Law and the source the citation conventions below are calibrated against. Consult **p6** or **p4** only when:

- The user explicitly asks about an older Law revision or a rule that has since changed.
- You need to confirm whether a rule existed (or was numbered differently) in a prior printing.

Rule numbering is **not stable across printings** — `rule:6.2.3` in p4 may correspond to `rule:6.2.2` in p16 (and similar shifts elsewhere). When citing from a non-p16 printing, prefix the citation with the printing tag: `p6:rule:6.2.3` or `p4:rule:5.1.1`. Bare `rule:X.Y.Z` always means p16.

## Agent Roster (when to invoke what)

| Agent | Use when… |
|---|---|
| `root-referee` | Any rules question, conflict resolution, cited rule lookup |
| `root-strategist` | Faction win conditions, matchup analysis, board-state questions |
| `root-coach` | Post-mortem walking, decision-point analysis (Socratic) |
| `root-author` | Drafting faction primers, strategy articles, teaching material |
| `root-analyst` | Complex pattern-finding or original strategy synthesis (orchestrator: dispatches the four above in parallel) |

`/root-analyze <question>` is a thin convenience command that dispatches to `root-analyst`.

## Skills

- `root-rules` — lens onto `rules.yml` and `faq.yml`. Auto-preloaded into all layer-2 agents.
- `root-cards` — unified index across the 13 card YAMLs. Auto-preloaded into all layer-2 agents.

## Citation Conventions

All citations go inside backticks. Use these prefixes:

- `rule:X.Y.Z` — full rule reference (e.g., `rule:6.1.2`)
- `rule:X.` — section-level reference (e.g., `rule:22.`)
- `faction:NAME$X.Y.Z` — faction-specific rule (e.g., `faction:eyrie$7.2.3`)
- `item:NAME` — Glossary item (e.g., `item:sword`, `item:crossbow`)
- `hireling:TAG` — hireling subsection tag (e.g., `hireling:whenhired`)
- `card:ROOT-N (Deck Name)` — card by ID; **deck always required** for IDs in the collision set; ID-only OK otherwise
- `faq:<short-stub>` — FAQ entry

Two backtick groups adjacent with no space between them are two separate citations
(this pattern appears in the Law itself).

## Faction Display-Name → Citation-Key Map

| Display name | Citation key | Internal section |
|---|---|---|
| Marquise de Cat | (no `faction:` key — cite via `rule:6.X`) | rule 6 |
| Eyrie Dynasties | `eyrie` | rule 7 |
| Woodland Alliance | `woodland` | rule 8 |
| Vagabond | `vagabond` | rule 9 |
| Lizard Cult | `cult` | rule 10 |
| Riverfolk Company | `riverfolk` | rule 11 |
| Underground Duchy | `duchy` | rule 12 |
| Corvid Conspiracy | `corvid` | rule 13 |
| Lord of the Hundreds | `warlord` | rule 14 |
| Keepers in Iron | (no `faction:` key — cite via `rule:15.X`) | rule 15 |
| Lilypad Diaspora | `diaspora` | rule 16 |
| Twilight Council | `council` | rule 17 |
| Knaves of the Deepwood | `knaves` | rule 18 |

## Card Collision Set (deck always required)

These names exist in BOTH `rootbasegame.yml` and `exiles-partisans.yml` with different IDs.
When citing any of them, name the deck:

Ambush!, Dominance, Crossbow, Anvil, Arms Trader, A Visit To Friends, Bake Sale, Birdy Bindle.

(The 4-suit variants of Ambush! and Dominance bring the total ID-instance count to 13.)
Card IDs (`ROOT-N`) are not globally unique across files — same ID may appear in
multiple decks. When in doubt, include the deck name.

## Data Cutoff

The canonical p16 source data tracks the **October 8, 2025** Law of Root. p6 and
p4 capture earlier printings of the Law and are kept for historical lookup and
revision-comparison. Agents flag references to anything published *after* p16
rather than guessing.

## Implementation Notes

- All Claude Code components are project-scoped under `.claude/`.
- Layer-2 agents (`root-referee`, `-strategist`, `-coach`, `-author`) are
  self-sufficient — they Read/Grep the YAML files directly. Only `root-analyst`
  has the `Agent` tool and dispatches to other agents.
- Spec: `docs/superpowers/specs/2026-04-27-root-domain-tools-design.md`
- Implementation plan: `~/.claude/plans/plan-it-robust-lightning.md`
