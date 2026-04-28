---
description: Decompose a Root question or scenario, dispatch domain agents in parallel, and synthesize an analysis or original strategy with cited provenance.
model: haiku
allowed-tools:
  - Agent
argument-hint: <question or scenario>
---

# /root-analyze

Thin wrapper that hands a Root question off to the `root-analyst` orchestrator.

## Workflow

1. Take the user's input as the analysis question. If empty, ask the user once
   for the question or scenario, then proceed.
2. Dispatch via the Agent tool:
   - `subagent_type: root-analyst`
   - `description`: short summary of the question (e.g. "Eyrie vs. Marquise turtle")
   - `prompt`: the full user question, verbatim, plus any context the user
     supplied
3. Relay the analyst's response to the user without modification. Do not
   second-guess, summarize, or add commentary — `root-analyst` already produces
   the synthesis with cited provenance.

## Notes

- The analyst dispatches `root-referee`, `root-strategist`, `root-coach`, and
  `root-author` as needed. Expect parallel subagent runs and a multi-paragraph
  synthesis.
- For simple, single-domain questions, the user is better served by invoking the
  layer-2 agent directly (e.g., `root-referee` for a rules question). This
  command is for complex, pattern-finding, or synthesis-heavy asks.
