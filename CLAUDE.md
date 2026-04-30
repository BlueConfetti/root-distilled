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
- **Faction profiles** — 13 in-depth per-faction reference docs at `docs/factions/<slug>.md` (plus `README.md` index). Each profile follows a fixed 12-section spine: At a Glance, Win Condition, Setup, Faction Rules and Abilities, Turn Structure, Scoring Engine, Playbook, Threat Factions, Card Priorities, Common Pitfalls, Mechanic Clarifications, Reference Index. Profiles are derivative — when content conflicts with the Law, the Law wins. Spec: `docs/superpowers/specs/2026-04-28-faction-profiles-design.md`. Slug = citation key where available (`eyrie`, `vagabond`, etc.); display-name slug otherwise (`marquise`, `keepers-iron`).
- **Curated third-party strategy** — `docs/strategy/sources/<domain>--<title-slug>.md`. **One file per source** (blog, BGG thread, Reddit thread, etc.); a source covering multiple factions writes one file with one `## <Faction Display Name>` H2 per faction discussed plus an optional `## General` section. Frontmatter `factions[]` is the index. Two layers out from the Law: profiles are the project's derivative strategy view, and `docs/strategy/` is *what other people say*. Source roster + schema in `docs/strategy/README.md`. Accuracy/rating fields (`accuracy_rating`, `accuracy_notes`, `last_reviewed`, `reviewed_by`) are populated by a separate review pipeline, not by the curator. Written by `root-strategy-curator` or by hand. Treat as derivative; pass-through, do not promote a source's claim to a `rule:X.Y.Z` cite.

### Which printing to cite

Default to **p16** for any rules question — it is the current Law and the source the citation conventions below are calibrated against. Consult **p6** or **p4** only when:

- The user explicitly asks about an older Law revision or a rule that has since changed.
- You need to confirm whether a rule existed (or was numbered differently) in a prior printing.

Rule numbering is **not stable across printings** — `rule:6.2.3` in p4 may correspond to `rule:6.2.2` in p16 (and similar shifts elsewhere). When citing from a non-p16 printing, prefix the citation with the printing tag: `p6:rule:6.2.3` or `p4:rule:5.1.1`. Bare `rule:X.Y.Z` always means p16.

## Orchestration: subagents cannot dispatch other subagents

**Claude Code forbids nested subagent dispatch** ([official docs](https://code.claude.com/docs/en/sub-agents)). A subagent that calls the `Agent` tool gets nothing — its dispatches silently fail and it does the work itself, breaking any perceived parallelism.

The supported pattern for parallel orchestration in this project: **put orchestration logic in slash command bodies**, not in subagents. Slash commands run in the **main session**, where the `Agent` tool actually works for parallel dispatch. **Multiple `Agent` calls in a single assistant message run concurrently** — that is the parallelization mechanism.

When designing a new pipeline:
- **Workers go in `.claude/agents/`** — each agent is a leaf that does one focused job and reports results.
- **Orchestrators go in `.claude/commands/`** — fat slash commands whose body contains the dispatch logic, manifest management, and synthesis. They run in the main session and can dispatch workers in parallel.
- A subagent CAN have `Agent(...)` in its `tools:` list, but those dispatches will silently fail at runtime. Do not write subagents that orchestrate.

**Existing fat slash commands in this project:**
- `/root-analyze` — synthesis orchestrator (decomposes questions, fans out the five layer-2 agents in parallel)
- `/root-strategy-pipeline` — curation pipeline orchestrator (parallel discovery + parallel extraction in waves of 3, manifest-tracked, resumable)
- `/root-3p-review` — third-party review pipeline orchestrator (parallel review of curated docs in waves of 3, fills accuracy frontmatter + appends `# Review` sections, manifest-tracked)
- `/root-cheatsheet` — help/hurt cheatsheet revision orchestrator for a **single faction per invocation**. Four-phase pipeline `generate -> rules-verify -> pragmatic-vet -> synthesize`. Phase 1 fans out 3 strategists (one per category); phases 2 and 3 fan out **per claim** — one referee and one pragmatist per claim — so each verdict stays focused on a single claim. Backs up the cheatsheet before Phase 4 writes.

**Existing thin slash commands (single-target dispatch — fine to write as a thin wrapper):**
- `/root-strategy-curate` — single curation target hand-off to `root-strategy-curator`

## Agent Roster (workers — dispatched FROM slash commands, do not orchestrate)

| Agent | Use when… |
|---|---|
| `root-referee` | Any rules question, conflict resolution, cited rule lookup |
| `root-strategist` | Faction win conditions, matchup analysis, board-state questions |
| `root-coach` | Post-mortem walking, decision-point analysis (Socratic) |
| `root-author` | Drafting faction primers, strategy articles, teaching material |
| `root-pragmatist` | Validating strategy claims for actionability + counterfactual correctness; categorizes claims into KEEP / MISFRAMED / DROP / MOVE_TO_AWARENESS / FLAG |
| `root-awareness-curator` | Filtering Awareness claims for surprise (non-obvious to an intermediate player) + decision impact (changes a tactical decision); categorizes claims into KEEP / OBVIOUS / INERT / FLAG. No rescue path — kept-or-dropped at the gate. |
| `root-strategy-curator` | Fetches and distills one third-party Root strategy source per invocation. Used by `/root-strategy-curate` (single) and `/root-strategy-pipeline` (parallel waves). Content pipeline, not strategist. |
| `root-3p-reviewer` | Reviews one curated third-party file claim-by-claim against the Law and faction profiles. Fills frontmatter accuracy fields and appends/replaces a `# Review` section. Used by `/root-3p-review` (parallel waves). Owns only accuracy frontmatter + Review section; never modifies curator-owned content. |

## Skills

**Domain lenses** (auto-preloaded into the analyst pipeline):

- `root-rules` — lens onto `rules.yml` and `faq.yml`. Auto-preloaded into all layer-2 agents.
- `root-cards` — unified index across the 13 card YAMLs. Auto-preloaded into all layer-2 agents.
- `root-factions` — lens onto the 13 faction profile docs at `docs/factions/` (project-derivative strategy view, layer 2). Auto-preloaded into `root-strategist`, `root-coach`, `root-author`, `root-pragmatist`. Not preloaded into `root-referee` (rules adjudication uses primary sources only).
- `root-strategy` — lens onto curated third-party content at `docs/strategy/sources/` (layer 3 — *what other people say*, rated 0–10 by `/root-3p-review`). Auto-preloaded into `root-strategist`, `root-coach`, `root-author`, `root-pragmatist`. Not preloaded into `root-referee` (same exclusion as `root-factions`). Use **after** consulting `root-factions`; the corpus is supplementary, not authoritative.

**Strategy curation pipeline** (preloaded into `root-strategy-curator`; the four non-internal ones are also user-invocable as standalone slash commands):

- `strategy-source-roster` — lens onto `docs/strategy/README.md` (curated third-party source list, fetch-method dispatch, quality tiers).
- `strategy-fetch-blog` — generic WebFetch-based fetcher for blogs and articles.
- `strategy-fetch-reddit` — Bash + Python fetcher hitting Reddit's public JSON endpoint (Reddit blocks both WebFetch *and* Chrome MCP, but the JSON endpoint is shell-reachable). Bundles `scripts/extract-thread.py`.
- `strategy-fetch-bgg` — Bash + Python fetcher hitting BGG's private SPA-backing JSON endpoint (`api.geekdo.com/api/article`) with browser-style Origin/Referer headers. Paginates every page; resolves author user IDs to display names. Bundles `scripts/extract-thread.py`. No Chrome required.
- `strategy-discover` — WebSearch-based source discovery; returns a ranked candidate list for user approval before fetching.
- `strategy-distill` — internal output schema for distilled files written under `docs/strategy/sources/`. Not user-invocable.

## Citation Conventions

Two citation forms in this project, asymmetric by storage:

### Backtick string cites — for YAML-indexed sources

These index data inside `rules/`, `cards/`, etc. — no clickable target. All
go inside backticks:

- `rule:X.Y.Z` — full rule reference (e.g., `rule:6.1.2`)
- `rule:X.` — section-level reference (e.g., `rule:22.`)
- `faction:NAME$X.Y.Z` — faction-specific rule (e.g., `faction:eyrie$7.2.3`)
- `item:NAME` — Glossary item (e.g., `item:sword`, `item:crossbow`)
- `hireling:TAG` — hireling subsection tag (e.g., `hireling:whenhired`)
- `card:ROOT-N (Deck Name)` — card by ID; **deck always required** for IDs in the collision set; ID-only OK otherwise
- `faq:<short-stub>` — FAQ entry

Two backtick groups adjacent with no space between them are two separate citations
(this pattern appears in the Law itself).

### Relative markdown links — for filesystem-navigable derivative sources

These point at markdown files under `docs/factions/` and
`docs/strategy/sources/` and render as clickable links from
`docs/root-faction-help-hurt-cheatsheet.md` and other docs in the same
tree. Use standard `[text](relative/path.md#anchor)` form. Paths are
**relative to the citing document**, so a citation written into
`docs/root-faction-help-hurt-cheatsheet.md` uses `factions/eyrie.md`,
not `docs/factions/eyrie.md`.

- **Profile**: `[<Faction display name> profile § <Section>](factions/<slug>.md#<section-anchor>)`
  — section-anchor is the GitHub-style kebab-cased H2/H3 from the profile.
  - Example: `[Eyrie profile § Scoring Engine](factions/eyrie.md#scoring-engine)`
  - Example (rules-section deep cite): `[Eyrie profile § Decree Resolution](factions/eyrie.md#decree-resolution)`
- **Strategy**: `[<Source> § <Faction>](strategy/sources/<basename>.md#<faction-anchor>)`
  — `<basename>` is the filename without `.md`. `<Source>` is the
  frontmatter `source:` field truncated at the first ` — ` separator
  (e.g., "BoardGameGeek — Root Strategy Guide thread" → "BoardGameGeek");
  fall back to `<basename>` if no `source:` is present. For sources
  without a faction subsection, drop the `§ <Faction>` and the anchor
  fragment.
  - Example: `[BoardGameGeek § Eyrie](strategy/sources/boardgamegeek--root-strategy-guide.md#eyrie)`
  - Example (no faction subsection): `[MakeCraftGame § Faction Scoring Models](strategy/sources/makecraftgame--faction-scoring-models.md)`

### Layering and authority

- `rule:` and `faction:` are **authoritative** — the Law of Root.
- `profile:` (markdown link to `factions/`) is the project's
  **derivative** strategy view. Used in parallel with rules during
  analysis; does **not** override the Law on conflicts.
- Strategy markdown links are **third-party corroboration**. Used in
  parallel with rules and profiles; same precedence rule — Law wins on
  conflicts. Read the source's `# Review` section before propagating a
  strategy claim — claims the review flagged as inaccurate must not be
  cited as if accurate. (Numeric `accuracy_rating` is not a filter; the
  review notes are.)
- A bullet that genuinely has no profile or strategy backing stays
  rule-only. Do not pad citations with irrelevant section links just
  to hit a quota — the rationale should say "profile silent on this"
  if relevant.

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
