---
description: Show usage for every invocable Root project command, agent, and skill.
model: sonnet
---

# /root-help

Print the help block below verbatim. Do not paraphrase, do not add commentary, do not invoke any tools.

---

# Root Domain Toolkit

## Worker agents (invoke via natural language)

| Agent | Use for |
|---|---|
| `root-referee` | Rules adjudication, cited rule lookup, conflict resolution |
| `root-strategist` | Faction win conditions, matchups, board-state, scoring tempo |
| `root-coach` | Post-mortem decision-point analysis (Socratic) |
| `root-author` | Drafting faction primers, strategy articles, teaching material |
| `root-pragmatist` | Validating H/H strategy claims — KEEP / MISFRAMED / DROP / MOVE_TO_AWARENESS / FLAG |
| `root-awareness-curator` | Filtering Awareness claims for surprise + decision impact — KEEP / OBVIOUS / INERT / FLAG |
| `root-strategy-curator` | Fetch + distill one third-party source (typically used by `/root-strategy-curate`) |
| `root-3p-reviewer` | Review one curated file (typically used by `/root-3p-review`) |

## Skills

| Skill | Use for |
|---|---|
| `root-rules` | `rules.yml` + `faq.yml` lookup recipes (layer 1: canonical Law) |
| `root-cards` | Card YAMLs across base + every expansion |
| `root-factions` | Faction profile lookup (`docs/factions/`) — layer 2: project-derivative strategy view |
| `root-strategy` | Curated third-party strategy lookup (`docs/strategy/sources/`) — layer 3: *what other people say*, rated 0–10 |
| `strategy-source-roster` | Curated third-party source roster |
| `strategy-fetch-blog <url>` | Fetch a blog post via WebFetch |
| `strategy-fetch-reddit <url>` | Fetch a Reddit thread via public JSON |
| `strategy-fetch-bgg <url>` | Fetch a BGG thread (paginated, all articles) |
| `strategy-discover <topic>` | Find new strategy sources via WebSearch |

## Slash commands

| Command | Use for |
|---|---|
| `/root-analyze <question>` | Multi-domain Root question — fans out 5 layer-2 agents in parallel and synthesizes a cited answer |
| `/root-strategy-curate <target>` | Curate one third-party source (URL, source name, faction, or topic) |
| `/root-strategy-pipeline [scope] [flags]` | End-to-end strategy curation — parallel discovery + extraction in waves |
| `/root-3p-review [scope] [flags]` | Review curated third-party content against the Law of Root |
| `/root-cheatsheet <faction-slug> [flags]` | Audit + extend the help/hurt cheatsheet for a single faction — strategist per category, then per-claim referee + pragmatist |
| `/root-help` | This help |

### `/root-strategy-pipeline` arguments

| Argument | Effect |
|---|---|
| (no args), `roster` | Enumerate the curated roster only — no WebSearch |
| `all` | Every faction, every roster source |
| `<faction-slug>` | Scope to one faction (e.g. `eyrie`, `keepers-iron`) |
| `topic: <text>` | Topic-scoped (e.g. `topic: tournament meta`) |
| `--discover` | Add WebSearch slices to find sources beyond the roster |
| `--force-refresh` | Re-curate sources already in `docs/strategy/sources/` |
| `--resume <run-id>` | Re-enter Phase 2 of a prior manifest, retry pending/failed only |

### `/root-3p-review` arguments

| Argument | Effect |
|---|---|
| (no args), `all` | Review every curated source, refreshing existing reviews |
| `<faction-slug>` | Filter to docs whose `factions[]` contains this slug |
| `<file-path>` | Review only that one file (path with `/` or `.md`) |
| `--last <n>` | Filter to docs curated in the last N days |
| `--only-new` | Filter to docs that have never been reviewed |

### `/root-cheatsheet` arguments

| Argument | Effect |
|---|---|
| `<faction-slug>` | **Required.** Process this one faction (slug only — no aliases like `cat`). Invoke once per faction. |
| `--audit-only` | Verdict on existing entries; no new candidates |
| `--additions-only` | Skip audit; only propose net-new candidates |
| `--dry-run` | Print proposed section + verdict table; do not modify the file |
