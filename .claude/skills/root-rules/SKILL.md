---
name: root-rules
description: Use when adjudicating any Root board game rules question, resolving rules conflicts, or citing rule:X.Y.Z. Wraps rules.yml and faq.yml with schema, navigation, and citation conventions.
user-invocable: false
allowed-tools:
  - "Read"
  - "Grep"
---

# Root Rules Lens

Reference recipe for navigating the Law of Root and FAQ. Citation conventions are
defined in the project `CLAUDE.md` — do not restate them; just use them.

## Files

- `rules/content/rules/root/en-US/p16/rules.yml` — full Law of Root, 1988 lines, Oct 2025
- `rules/content/rules/root/en-US/p16/faq.yml` — 250 lines, list of `{q, a, rules[]}`

## rules.yml Schema

YAML list of nested nodes. Each node:

| Field | Required | Notes |
|---|---|---|
| `name` | yes | Section title |
| `text` | optional | Rule body, markdown-aware (`**bold**`, `_(italic)_`, backtick-wrapped cites, `>` block) |
| `children` | optional | Array of child nodes (4+ levels deep observed) |
| `color` | optional | Hex color on top-level books (faction theming) |
| `icon` | optional | Faction icon id on top-level faction books |
| `pretext` | optional | Intro prose before children |
| `plainName` | optional | Plain-text alt of `name` (markdown-stripped) |
| `appendix` | optional | Letter designator (`C`, `G`, `H`) for appendix sections |

## Top-Level Book Map (rules.yml)

| Line | Book | Notes |
|---|---|---|
| 1 | Golden Rules | Includes Rules Conflicts → Precedence (the conflict-resolution authority) |
| 57 | Key Concepts | Shared mechanics |
| 102 | Victory | Win conditions |
| 136 | Key Actions | Move, Battle, Craft, etc. |
| 190 | Setup | |
| 240 | Marquise de Cat | rule 6 |
| 296 | Eyrie Dynasties | rule 7 |
| 387 | Woodland Alliance | rule 8 |
| 480 | Vagabond | rule 9 |
| 615 | Lizard Cult | rule 10 |
| 687 | Riverfolk Company | rule 11 |
| 790 | Underground Duchy | rule 12 |
| 883 | Corvid Conspiracy | rule 13 |
| 956 | Lord of the Hundreds | rule 14 |
| 1050 | Keepers in Iron | rule 15 |
| 1132 | Lilypad Diaspora | rule 16 |
| 1216 | Twilight Council | rule 17 |
| 1295 | Knaves of the Deepwood | rule 18 |
| 1379 | Advanced Setup | |
| 1454 | Components | appendix C |
| 1585 | Glossary | appendix G — leaf nodes referencing back into the Law |
| 1697 | Hirelings | appendix H |
| 1739 | Knave Captains | |
| 1827 | Landmarks | |
| 1838 | Maps | |
| 1902 | Vagabonds | Vagabond character roster |

## faq.yml Schema

Top-level YAML list. Each entry:

```yaml
- q: <question text>
  a: <answer; may include `rule:X.Y.Z` cites in backticks>
  rules:
    - 'X.Y.Z'      # always quoted strings, can include faction-prefixed (e.g., '16.4.3.7')
```

## Lookup Recipes

**Find by rule number** — the file is structured but rule numbers don't appear as
explicit fields. Best path: navigate by section name. For `rule:7.2.3` (Eyrie),
read from line 296 and walk the `children` tree until you hit the matching node.

**Search by keyword across the Law:**
```
grep -n "<keyword>" rules/content/rules/root/en-US/p16/rules.yml
```

**Find all FAQ entries that touch a rule:**
```
grep -n "<rule-number>" rules/content/rules/root/en-US/p16/faq.yml
```

**Find faction-specific cross-references** (e.g., Eyrie referenced from another faction's section):
```
grep -n "faction:eyrie" rules/content/rules/root/en-US/p16/rules.yml
```

## Conflict Resolution (Golden Rule of Precedence)

`rules.yml` lines 7-9 establish the canonical precedence:
> If a card conflicts with the Law, follow the card. If the Learning to Play
> guide conflicts with the Law, follow the Law. If you can follow both a general
> rule and a faction or hireling rule, follow both; if you cannot, follow the
> faction or hireling rule.

Cite `rule:1.1.X` when invoking precedence (verify exact subsection at the source).

## Output Contract for Callers

- Every rules claim cites `rule:X.Y.Z` or `faq:N` in backticks
- If the Law is silent on the question, **say so explicitly** — do not confabulate
- When card text and Law conflict, apply the Golden Rule of Precedence and name the winner
- Flag references to anything published after October 2025 (the data cutoff) rather than guessing
