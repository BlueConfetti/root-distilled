---
description: Audit and extend the Root help/hurt cheatsheet for a single faction. Runs in the main session so it can dispatch root-strategist, root-referee, root-pragmatist, and root-awareness-curator workers in parallel. Pipeline is generate -> rules-verify -> filter (pragmatist for H/H, awareness curator for Awareness) -> synthesize+dedup. Phase 1 (strategist) fans out per category (3 dispatches); phases 2 and 3 fan out per claim — one agent per claim — so each verdict stays focused on a single claim.
model: opus
allowed-tools:
  - Agent
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
argument-hint: <faction-slug> [--audit-only] [--additions-only] [--dry-run]
---

# /root-cheatsheet

You orchestrate revision of `docs/root-faction-help-hurt-cheatsheet.md`
for **a single faction**. You run in the main session (slash commands
are not subagents), which is the **only place where the Agent tool can
actually dispatch new subagents** — nested subagent dispatch is
forbidden by Claude Code's design.

You run a four-phase pipeline that audits existing entries and
proposes net-new candidates across three categories (Helping, Hurting,
Awareness). Sequential dependencies live inside a single claim's
lifecycle (`generate -> rules-verify -> pragmatic-vet -> author`);
everything else parallelizes — across categories and across claims
inside a phase.

## Critical constraints

- **One faction per invocation.** This command does not loop over
  factions; if the user wants to process several, they invoke
  `/root-cheatsheet` once per slug.
- **Multiple Agent calls in a single assistant message run concurrently.**
  This is the parallelization mechanism. Dispatch each phase's full
  fan-out in one message.
- **Your dispatch allowlist is fixed**: `root-strategist`,
  `root-referee`, `root-pragmatist`, `root-awareness-curator`. No
  other agents.
- **Fan-out shape**: Phase 1 (strategist) is one dispatch per category
  — 3 strategists per run. Phase 2 (referee) and Phase 3 (pragmatist
  and awareness curator) fan out **per claim** — one agent per claim
  — so claim count, not category count, sets the fan-out for those
  phases. A typical run with ~6 claims per category yields ~18
  referees in Phase 2, ~12 pragmatists in Phase 3a (Helping +
  Hurting), and ~6 awareness curators in Phase 3b (Awareness),
  plus a small Phase 3c if the pragmatist surfaced any
  `MISFRAMED`/`MOVE_TO_AWARENESS` migrations.
- **One claim, one agent.** Per the project memory note on fleet
  parallelism: never bundle claims through a single referee or
  pragmatist invocation. Each claim gets its own agent so the verdict
  stays focused on that single claim's framing, citations, and
  scenario.
- **No category caps.** Strategists propose freely. The referee and
  pragmatist gates do the filtering; whatever survives lands in the
  cheatsheet.
- **Audience is intermediate.** Awareness items survive even if
  veterans already know them, as long as they're non-obvious from
  the player board / plain rule text and change a tactical decision.
- **Backup before writing.** Copy the cheatsheet to
  `docs/_runs/cheatsheet.<run-id>.bak.md` before Phase 4 writes.

## Argument parsing

Parse `$ARGUMENTS`:

| Argument | Effect |
|---|---|
| `<faction-slug>` | **Required.** Process this one faction. |
| `--audit-only` | Verdict on existing entries; do not propose new candidates |
| `--additions-only` | Skip audit; only propose net-new candidates that don't duplicate existing |
| `--dry-run` | Print proposed section + verdict table; do not modify the file |

If `$ARGUMENTS` is empty, contains an unknown slug, or contains more
than one slug, stop and ask the user for clarification. Do not pick
a default faction.

## Faction slug whitelist

Use these slugs only (no aliases). Mapping to display name and
citation prefix:

| Slug | Display name | Citation prefix |
|---|---|---|
| `marquise` | Marquise de Cat | `rule:6.X` |
| `eyrie` | Eyrie Dynasties | `faction:eyrie$7.X` |
| `woodland` | Woodland Alliance | `faction:woodland$8.X` |
| `vagabond` | Vagabond | `faction:vagabond$9.X` |
| `cult` | Lizard Cult | `faction:cult$10.X` |
| `riverfolk` | Riverfolk Company | `faction:riverfolk$11.X` |
| `duchy` | Underground Duchy | `faction:duchy$12.X` |
| `corvid` | Corvid Conspiracy | `faction:corvid$13.X` |
| `warlord` | Lord of the Hundreds | `faction:warlord$14.X` |
| `keepers-iron` | Keepers in Iron | `rule:15.X` |
| `diaspora` | Lilypad Diaspora | `faction:diaspora$16.X` |
| `council` | Twilight Council | `faction:council$17.X` |
| `knaves` | Knaves of the Deepwood | `faction:knaves$18.X` |

## Workflow

### 1. Initialize

- Resolve the single faction slug from `$ARGUMENTS`. If missing,
  unknown, or multiple slugs, stop and ask.
- Generate a run_id:
  ```
  Bash: echo "cheatsheet-<slug>-$(date -u +%Y-%m-%dT%H-%M-%SZ)"
  ```
- Ensure runs dir exists:
  ```
  Bash: mkdir -p docs/_runs
  ```
- **Back up the cheatsheet** before Phase 4 writes:
  ```
  Bash: cp docs/root-faction-help-hurt-cheatsheet.md docs/_runs/cheatsheet.<run-id>.bak.md
  ```
  Skip the backup only when `--dry-run` is set.
- Read the cheatsheet once into memory. Extract the faction's section
  (between `### <Display name>` and the next `---` divider) so you
  can pass existing entries to the strategist.

### 2. Run phases 1-4

Phases are sequential; dispatches inside a phase are parallel.

#### Phase 1 — Generate (parallel ×3 categories)

Dispatch one `root-strategist` per category — Helping, Hurting,
Awareness — for a total of 3 strategists in a single assistant
message.

Pass each strategist the existing entries for its category (verbatim,
with citations) and ask for both an audit pass and net-new candidates.
Use the prompt template in [Phase 1 prompt](#phase-1-prompt-strategist).

If `--audit-only`, the strategist only audits and skips additions.
If `--additions-only`, the strategist only proposes new candidates and
treats existing entries as "do not duplicate."

Each strategist returns a structured list of items. Parse all results
into a working set of claims.

#### Phase 2 — Rules verify (parallel ×N claims)

**Dispatch one `root-referee` per claim.** Collect every claim from
Phase 1 across all three categories — including both audited-and-revised
entries and net-new candidates — assign each a `claim_id` (e.g.
`vagabond.helping.3`), and fan out one referee agent per claim in a
single assistant message.

Each referee receives just its one claim and returns the verdict for
that claim: `RULES_OK`, `RULES_WRONG`, or `RULES_AMBIGUOUS`, with
verified citations and (for non-OK) a correction.

Drop `RULES_WRONG` claims silently. Treat `RULES_AMBIGUOUS` as `FLAG`
and surface them in the final summary for the user to rule on.

Use the [Phase 2 prompt](#phase-2-prompt-referee).

#### Phase 3 — Filter gates

Phase 3 has three sub-phases. Sub-phases 3a and 3b run in parallel
(independent claim sets). Sub-phase 3c runs sequentially after 3a
because it operates on 3a's migration output.

##### Phase 3a — Pragmatic gate (parallel ×N claims, H/H only)

For Helping and Hurting claims only. **Dispatch one `root-pragmatist`
per claim** that survived Phase 2 with verdict `RULES_OK`. Each
pragmatist receives just its one claim and walks the three questions
(Q1 Scenario, Q2 Counterfactual, Q3 Actionability) before returning
a single verdict:

- `KEEP` — survives all three questions
- `MISFRAMED` — fails Q1 (framing assumes table agency that doesn't
  exist); rescue path is Awareness with the agent's `Reword:` line
- `DROP` — fails Q2 (directional sign inverted); discard
- `MOVE_TO_AWARENESS` — fails Q3 (no concrete move); rescue path is
  Awareness with the agent's `Reword:` line
- `FLAG` — borderline; surface in run summary for user adjudication

`MISFRAMED` and `MOVE_TO_AWARENESS` claims do NOT yet land in
Awareness — their `Reword:` lines become candidate Awareness claims
that must pass Phase 3c before being kept.

Use the [Phase 3a prompt](#phase-3a-prompt-pragmatist).

##### Phase 3b — Awareness curator (parallel ×N claims, Awareness only)

For Awareness claims only. **Dispatch one `root-awareness-curator`
per claim** that survived Phase 2 with verdict `RULES_OK`. Each
curator receives just its one claim and walks two questions (Q1
Surprise, Q2 Decision Impact) before returning a single verdict:

- `KEEP` — passes both questions
- `OBVIOUS` — fails Q1 (fact is on the player board, faction
  overview, or in a single rule's plain text); discard
- `INERT` — fails Q2 (true but doesn't shift any tactical decision);
  discard
- `FLAG` — borderline; surface in run summary for user adjudication

Awareness has no rescue path — items are kept-or-dropped at this
gate. The curator does NOT propose alternative wordings.

**Phases 3a and 3b dispatch in the same assistant message** — they
operate on disjoint claim sets and have no dependency between them.

Use the [Phase 3b prompt](#phase-3b-prompt-awareness-curator).

##### Phase 3c — Awareness curator on pragmatist migrations (sequential after 3a)

Once Phase 3a verdicts are in, collect every `MISFRAMED` and
`MOVE_TO_AWARENESS` claim. Each one's `Reword:` line is a new
candidate Awareness claim — assign it a fresh `claim_id` (e.g.
`marquise.helping.3` → `marquise.awareness.from-helping-3`) and
dispatch one `root-awareness-curator` per migrated claim in a single
assistant message.

Same verdict surface as 3b: `KEEP` / `OBVIOUS` / `INERT` / `FLAG`.

Use the [Phase 3b prompt](#phase-3b-prompt-awareness-curator) (same
template; the only difference is the source of the claim).

#### Phase 4 — Synthesize and write (sequential, in main session)

1. Assemble surviving claims:
   - Helping section: `KEEP` claims from Phase 3a (Helping).
   - Hurting section: `KEEP` claims from Phase 3a (Hurting).
   - Awareness section: `KEEP` claims from Phase 3b (original
     Awareness candidates) + `KEEP` claims from Phase 3c (migrated
     bullets, rendered using the pragmatist's `Reword:` line as
     the bullet text).

2. **Awareness dedup pass.** Run sequentially in the main session
   on the assembled Awareness `KEEP` set. The goal is to fold
   true paraphrases without dropping distinct interactions that
   happen to share a rule cite.

   Procedure:
   - Group bullets by their primary rule cite (the first
     `rule:X.Y.Z` or `faction:KEY$X.Y.Z` in the citation list).
   - Within each group, identify true duplicates — bullets that:
     - Restate the same mechanic with different wording, OR
     - Are sub-aspects of the **same single interaction** (e.g.,
       three bullets all describing what Field Hospitals does at
       a high level).
   - For each duplicate cluster, pick the strongest representative
     — the bullet with the most decision-shifting payload, the
     most counterintuitive angle, or the cleanest wording. Drop
     the rest.
   - **Distinct interactions that share a rule cite are NOT
     duplicates.** "Field Hospitals + ambush auto-end timing"
     and "Field Hospitals + Riverfolk Mercenaries exception"
     both cite `rule:6.2.3` but describe materially different
     decisions — keep both.
   - Note dropped duplicates in the run summary as `DEDUPED`,
     naming the surviving bullet they clustered to. Do not silently
     drop them.
3. Format each claim as a bulleted entry with a **Why:** line and
   citations, matching the existing cheatsheet style. Citations
   come from the Phase 2 referee's `verified_citations` list and
   from any profile/strategy markdown links the pragmatist's or
   curator's rationale relied on. Mix backtick rule cites and
   relative markdown links freely on one line, comma-separated,
   all inside one parenthetical group:
   ```
   - <one-line claim>
     - **Why**: <rationale>. (`rule:X.Y.Z`, `faction:KEY$X.Y.Z`, [<Faction> profile § <Section>](factions/<slug>.md#<anchor>), [<Source> § <Faction>](strategy/sources/<basename>.md#<anchor>))
   ```
   List **all** profile and strategy citations that genuinely apply
   — do not pick one. If profile and strategy are silent, render
   rule-only citations and that is fine.
4. Write the assembled section back into the cheatsheet, replacing
   the old `### <Display name>` block (between that header and the
   next `---` divider — keep the divider). Use `Edit` with
   sufficient surrounding context to make the replacement unique.
5. If `--dry-run` is set, do not write — print the proposed section
   to the conversation instead.

### 3. Surface the run summary

After Phase 4 completes, print:

- A **verdict table** showing the disposition of every claim with
  the gate that produced the verdict (referee, pragmatist, or
  awareness curator) and a one-line reason. Verdicts:
  - **Existing entries**: `KEEP_AS_IS` / `REVISED` / `DROPPED`.
  - **New H/H candidates**: `KEPT` / `MIGRATED_TO_AWARENESS`
    (kept by 3c) / `MIGRATED_BUT_DROPPED` (3a sent it to 3c, 3c
    dropped it as `OBVIOUS`/`INERT`) / `DROPPED` / `FLAGGED`.
  - **New Awareness candidates**: `KEPT` / `OBVIOUS` / `INERT` /
    `DEDUPED` (clustered into another bullet — name it) / `FLAGGED`.
- The **backup path** (`docs/_runs/cheatsheet.<run-id>.bak.md`) so the
  user can revert if needed.
- A **flagged-claims list** for any `RULES_AMBIGUOUS` or pragmatist
  `FLAG` items that need user adjudication.

If `--dry-run`, additionally print the full proposed section text.

---

## Phase 1 prompt (strategist)

Send each `root-strategist` a focused prompt of this shape:

```
You are auditing and extending the {DISPLAY_NAME} {CATEGORY} entries
in the Root help/hurt cheatsheet (audience: intermediate players).

Existing {CATEGORY} entries for {DISPLAY_NAME}:

{VERBATIM_EXISTING_ENTRIES_OR_"(none)"}

CATEGORY DEFINITIONS
- Helping: passive table behaviors that feed this faction's engine
  (the inverse — not doing the behavior — should provably hurt them)
- Hurting: actionable counterplay opponents can take to slow them
  (must pass actionability + counterfactual)
- Awareness: non-obvious neutral facts that change tactical decisions
  (no action required; survives if a veteran knows it, as long as it's
  non-obvious to an intermediate player from the player board / rule
  text and changes a real decision)

RESEARCH ORDER (mandatory before generating claims)

1. Read `docs/factions/{SLUG}.md` — the project's faction profile. Note
   which sections speak to {CATEGORY}-relevant levers (typically
   Scoring Engine, Playbook, Threat Factions, Common Pitfalls,
   Mechanic Clarifications).
2. Read every file under `docs/strategy/sources/` whose frontmatter
   `factions:` list contains `{SLUG}`, or whose body has a
   `## {DISPLAY_NAME}` H2 (or close paraphrase). For each such file,
   also read the `# Review` section if present — claims the review
   flagged as inaccurate must NOT be propagated.
3. Read the relevant rule sections in `rules/.../p16/rules.yml`. The
   Law is authoritative; profiles and strategy are derivative. On
   conflict, the Law wins.

You then generate claims grounded in this combined view, not in
rules alone.

YOUR TASK
{If --audit-only}: Audit each existing entry. Do not propose new
  candidates.
{If --additions-only}: Propose net-new candidate entries. Do not
  audit existing; treat them only as "do not duplicate."
{Otherwise}: Both — audit each existing entry, then propose net-new
  candidates that don't duplicate existing claims.

For AUDIT items, return one of:
  - KEEP_AS_IS: entry is accurate and well-stated
  - REVISE: entry has issues; provide the proposed revision text
  - DROP: entry should be removed; provide reason

For ADDITION items, propose as many candidate entries as you can
defend. No cap. Sharp, intermediate-level insights only — do not pad.

CITATION REQUIREMENTS

Every claim's `proposed_citations` MUST include:
- At least one `rule:X.Y.Z` or `faction:KEY$X.Y.Z` rules cite
  (backtick form), unless the claim is purely strategic with no
  ruleset anchor — in which case explain why in the rationale.
- A profile cite **for every profile section that speaks to this
  claim** — relative markdown link form, not a backtick string.
  Format: `[{DISPLAY_NAME} profile § <Section>](factions/{SLUG}.md#<section-anchor>)`.
  If the profile is genuinely silent on this lever, say so in the
  rationale rather than padding.
- A strategy cite **for every third-party source whose review-vetted
  content corroborates this claim** — relative markdown link form.
  Format: `[<Source> § {DISPLAY_NAME}](strategy/sources/<basename>.md#<faction-anchor>)`
  where `<Source>` is the file's frontmatter `source:` field
  truncated at the first ` — ` separator (fall back to basename if
  no `source:`). For files with no faction subsection, drop the
  `§ {DISPLAY_NAME}` and the anchor fragment. List as many as
  genuinely apply — do not pick one.
- For multi-citation lists, return the citations as a JSON array of
  STRINGS. Backtick cites and markdown links are both string
  literals; the orchestrator parses them.

OUTPUT FORMAT
Return a numbered list. For each item:

  [N] type=<AUDIT|ADDITION>
      audit_verdict=<KEEP_AS_IS|REVISE|DROP>   (AUDIT only)
      claim: <one-line claim text>
      rationale: <why it works / why it's true; reference profile
                  and strategy material where load-bearing>
      proposed_citations: [
        "`rule:X.Y.Z`",
        "`faction:NAME$X.Y.Z`",
        "[<Faction> profile § <Section>](factions/<slug>.md#<anchor>)",
        "[<Source> § <Faction>](strategy/sources/<basename>.md#<anchor>)",
        ...
      ]
      audit_note: <reason if REVISE/DROP>      (AUDIT only)

CITATION CONVENTIONS
{Paste the full Citation Conventions section from project CLAUDE.md,
including the faction citation prefix for {DISPLAY_NAME} and both the
backtick-form and markdown-link-form rules.}
```

## Phase 2 prompt (referee)

Send each `root-referee` exactly ONE claim. The fan-out is per-claim,
not per-category.

```
Verify the rules-correctness of this single {DISPLAY_NAME} {CATEGORY}
cheatsheet claim against the Law of Root (printing 16 by default).

CLAIM_ID: {CLAIM_ID}      (e.g. vagabond.helping.3)

CLAIM:
  text: {CLAIM_TEXT}
  rationale: {STRATEGIST_RATIONALE}
  proposed_citations: [{CITATION_LIST}]

The proposed_citations list mixes two citation forms:
- Backtick string cites (`rule:X.Y.Z`, `faction:KEY$X.Y.Z`,
  `card:ROOT-N`, `faq:stub`) — these you VERIFY against the Law and
  data files.
- Relative markdown links to `factions/<slug>.md#...` and
  `strategy/sources/<basename>.md#...` — these are derivative and
  pass-through. Do NOT promote them to rules-tier authority. Lightly
  validate that the file exists and the anchor section/H2 actually
  appears in the linked file (read the file and grep for the
  heading). If the file or anchor is missing, treat the link as
  malformed and drop it from verified_citations with a one-line note
  in correction.

Return exactly:

  claim_id: {CLAIM_ID}
  verdict=<RULES_OK|RULES_WRONG|RULES_AMBIGUOUS>
  verified_citations: [...]   (final correct citations — preserve
                                profile and strategy markdown links
                                that pass the existence check; revise
                                or replace any incorrect rule cites)
  correction: <only if WRONG/AMBIGUOUS — what's actually true; also
               note any dropped malformed markdown links>

Use rule:X.Y.Z for general rules and faction:NAME$X.Y.Z for
faction-specific rules. If the claim conflates two separate rules,
mark RULES_AMBIGUOUS and explain.

The rules verdict (RULES_OK / RULES_WRONG / RULES_AMBIGUOUS) judges
ONLY the rules-tier cites and the claim's rule-correctness. A claim
with a missing or malformed profile/strategy link still gets RULES_OK
if the rules cites are correct — the malformed link is just dropped.
```

## Phase 3a prompt (pragmatist)

Send each `root-pragmatist` exactly ONE claim. The fan-out is
per-claim, not per-category. The pragmatist agent walks its
three-question worksheet (Q1 Scenario → Q2 Counterfactual →
Q3 Actionability) internally before emitting the verdict; do not
ask it to dump the worksheet.

```
Apply the scenario + counterfactual + actionability gate to this
single {DISPLAY_NAME} {CATEGORY} claim (already rules-verified).

CLAIM_ID: {CLAIM_ID}      (e.g. vagabond.helping.3)

CLAIM:
  text: {CLAIM_TEXT}
  rationale: {STRATEGIST_RATIONALE}
  verified_citations: [{VERIFIED_CITATION_LIST}]

CONTEXT TO CONSULT (in parallel with the rules):
- `docs/factions/{SLUG}.md` — the project's faction profile.
  Especially: Scoring Engine, Playbook, Threat Factions, Common
  Pitfalls, Mechanic Clarifications.
- `docs/strategy/sources/*.md` whose frontmatter `factions:` contains
  `{SLUG}` or whose body has a `## {DISPLAY_NAME}` H2. Read each
  source's `# Review` section first — DO NOT lean on any claim the
  review flagged as inaccurate. The numeric `accuracy_rating` is not
  a filter; the review notes are.

The Law is authoritative; profile and strategy are derivative. On
conflict, the Law wins. Otherwise, treat all three as parallel
inputs to your judgment.

QUESTIONS (your three-question worksheet, in order):

  Q1 Scenario — when does X occur? Trigger / agency / constraints /
     table influence. If X is faction-driven and voluntary and the
     bullet frames it as "letting" something happen, fail Q1.
  Q2 Counterfactual — if X does not occur, what does the faction
     do instead? Consult the profile's Scoring Engine / Playbook to
     ground the alternative; check the rules to verify the
     alternative's mechanics. Fail if directional sign is wrong.
  Q3 Actionability — given the scenario, what concrete move
     responds? Cross-check the strategy corpus (review-vetted only)
     for whether intermediate-level community sources back the move.
     Observation ("be aware that…") fails actionability.

CITATION DISCIPLINE FOR YOUR RATIONALE:
- Cite a `rule:X.Y.Z` for any rule the judgment hangs on.
- Cite a profile section (relative markdown link) when the
  counterfactual or actionability judgment hangs on the profile —
  e.g. `[{DISPLAY_NAME} profile § Scoring Engine](factions/{SLUG}.md#scoring-engine)`.
- Cite a strategy source (relative markdown link) when a
  review-vetted third-party source corroborates or contradicts the
  judgment — e.g. `[<Source> § {DISPLAY_NAME}](strategy/sources/<basename>.md#<faction-anchor>)`.
- If the profile and strategy are silent, say so — "profile silent on
  this; rules-only judgment" — rather than padding.

Return exactly:

  claim_id: {CLAIM_ID}
  verdict=<KEEP|MISFRAMED|DROP|MOVE_TO_AWARENESS|FLAG>
  rationale: <one paragraph leading with the question that killed
              the claim (Q1/Q2/Q3) and the load-bearing rule and
              profile/strategy section where they apply>
  reword: <one-line Awareness rewording — required for MISFRAMED
           and MOVE_TO_AWARENESS, omit otherwise>
  salvage: <one-line note on what would resolve the borderline —
            required for FLAG, omit otherwise>
```

## Phase 3b prompt (awareness curator)

Send each `root-awareness-curator` exactly ONE claim. The fan-out is
per-claim. The curator walks its two-question worksheet (Q1 Surprise
→ Q2 Decision Impact) internally before emitting the verdict; do not
ask it to dump the worksheet. Used for both Phase 3b (original
Awareness candidates) and Phase 3c (pragmatist migrations) — same
prompt template, the only difference is the source of the claim.

```
Apply the surprise + decision-impact gate to this single
{DISPLAY_NAME} Awareness candidate (already rules-verified).

CLAIM_ID: {CLAIM_ID}      (e.g. marquise.awareness.4 or
                                marquise.awareness.from-helping-3)

CLAIM:
  text: {CLAIM_TEXT}
  rationale: {STRATEGIST_RATIONALE_OR_PRAGMATIST_REWORD}
  verified_citations: [{VERIFIED_CITATION_LIST}]

CONTEXT TO CONSULT (in parallel with the rules):
- `docs/factions/{SLUG}.md` — the project's faction profile.
  Especially: Mechanic Clarifications, Common Pitfalls. If the
  profile flags this exact fact as non-obvious, that's strong
  evidence for Q1.
- `docs/strategy/sources/*.md` whose frontmatter `factions:`
  contains `{SLUG}` or whose body has a `## {DISPLAY_NAME}` H2.
  Read each source's `# Review` section first — DO NOT lean on
  any claim the review flagged as inaccurate. If multiple
  review-vetted sources mention the fact as decision-relevant,
  that's evidence for Q2.

The Law is authoritative; profile and strategy are derivative. On
conflict, the Law wins.

QUESTIONS (your two-question worksheet, in order):

  Q1 Surprise — would an intermediate player learn something new
     from this? Stated on the player board, faction overview, or
     in a single rule's plain text → fail Q1 (`OBVIOUS`). Derived
     from rule combinations, glossary cross-refs, exception cases,
     or counterintuitive edges → pass.
  Q2 Decision Impact — does knowing this change a tactical decision
     (card spend, combat target, placement, march path, craft
     choice, tempo trade)? "Useful trivia" or board-position
     observations fixed at setup → fail Q2 (`INERT`). Items that
     change a move during play → pass.

CITATION DISCIPLINE FOR YOUR RATIONALE:
- For `OBVIOUS`: cite the rule reference or board location where
  the fact is plainly visible.
- For `INERT`: name the decision-context the bullet claims to
  shift; explain why no realistic decision pivots on it.
- For `KEEP`: name the decision the bullet shifts; cite the rules,
  profile section, or strategy section where the impact is
  load-bearing.
- If profile and strategy are silent on the bullet, say so —
  rules-only judgment is fine when the lever is genuinely
  unaddressed by the derivative layers.

Return exactly:

  claim_id: {CLAIM_ID}
  verdict=<KEEP|OBVIOUS|INERT|FLAG>
  rationale: <one paragraph leading with the question that killed
              the claim (Q1/Q2) and the load-bearing reference>
  salvage: <one-line note on what would resolve the borderline —
            required for FLAG, omit otherwise>

Do NOT propose alternative wordings. Awareness is kept-or-dropped
at this gate. If the wording is the issue (vs the substance), use
`FLAG`.
```

---

## Style notes for synthesized entries

- Match the existing cheatsheet voice: terse, declarative, one
  bulleted claim per entry with a nested `**Why**:` line.
- Citations live in a single comma-separated parenthetical at the
  end of the rationale. Two citation forms mix freely:
  - **Backtick rule cites** for YAML-indexed sources: `rule:X.Y.Z`,
    `faction:KEY$X.Y.Z`, `card:ROOT-N (Deck)`, `faq:stub`.
  - **Relative markdown links** for filesystem-navigable derivative
    sources: `[<Faction> profile § <Section>](factions/<slug>.md#<anchor>)`
    and `[<Source> § <Faction>](strategy/sources/<basename>.md#<anchor>)`.
- Two adjacent backtick groups with no space = two separate
  citations (this pattern appears in the Law itself).
- List **all** profile and strategy citations that genuinely apply
  — do not pick one when several speak to the same claim. Rule-only
  bullets are valid when profile and strategy are genuinely silent.
- Use `card:ROOT-N (Deck Name)` for any card cite in the collision
  set (Ambush!, Dominance, Crossbow, Anvil, Arms Trader, A Visit
  To Friends, Bake Sale, Birdy Bindle).
- Relative paths are computed from the cheatsheet location
  (`docs/root-faction-help-hurt-cheatsheet.md`); `factions/...` and
  `strategy/sources/...` resolve correctly from there.

---

User arguments: $ARGUMENTS

If `$ARGUMENTS` does not contain exactly one valid faction slug
(plus optional flags), stop and ask the user for the slug — do not
default to processing all factions.
