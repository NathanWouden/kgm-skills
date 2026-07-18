---
description: Audit a project's real Claude Code token spend by reading its actual session transcripts, fitting the token-economics formula to what really happened, and filing the findings to the wiki. Usage: /token-spend-audit [project name or path] e.g. /token-spend-audit bs-crm
---

You are auditing real, measured token spend for a project — not estimating. Companion to the `token-economics-formula` skill; use its formula and technique map throughout. Target: $ARGUMENTS

## Step 1 — Find the real transcripts

Claude Code logs every turn's token usage locally under `~/.claude/projects/<hashed-path>/*.jsonl`. The hashed folder name is the project's working-directory path with slashes replaced by dashes — check for the current path AND any prior root name the workspace may have had (e.g. this workspace was previously called `agentic-workflows` before being renamed; old sessions live under the old hash).

`grep -l` across candidate `.jsonl` files for the project's distinctive filenames or keywords to confirm which sessions actually touched this project — a project's own transcript folder can contain unrelated sessions too.

## Step 2 — Parse real usage per turn

For each relevant `.jsonl`, each line is one event; assistant turns have `message.usage` with `input_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, `output_tokens`. Use a small Python script (not manual reading — these files are large) to extract, per assistant turn:
- context size = `input_tokens + cache_creation_input_tokens + cache_read_input_tokens`
- output tokens
- model name, timestamps

## Step 3 — Detect legs (compaction resets)

Real sessions don't grow forever — Claude Code force-compacts when context nears its ceiling. Find turns where context size drops more than 50% versus the previous turn; these mark leg boundaries. A "leg" is one uninterrupted accumulating stretch — this is the real-world unit the formula applies to, not the whole session.

## Step 4 — Fit base and increment per leg

For each leg of N turns with context sizes `c_1...c_N`:
- `base = c_1` (first turn's context — the flat cost paid before any history accumulates)
- `increment ≈ (c_N - c_1) / (N-1)` (average tokens of unpruned history added per turn)

## Step 5 — Validate against the formula

Compute `N×base + increment×N(N+1)/2` and compare to the actual summed context tokens for that leg. Report the % difference — real growth is noisy (turns read different-sized files), a match within ~10% confirms the fit is meaningful.

## Step 6 — Price it

Convert to dollars using current pricing for the model(s) actually used (web-search current rates — don't assume). Report:
- Actual cost as it happened (with prompt caching)
- Hypothetical cost with no caching (cache-read tokens billed at the full input rate instead) — this shows how much caching alone is masking the raw token volume

## Step 7 — Diagnose and model a fix

Per `token-economics-formula`'s Step 3 diagnosis: is `base` or `increment` dominant? For coding sessions it is almost always `increment` (unpruned accumulating history). Model 2-3 alternative compaction cadences using the SAME fitted base/increment (e.g., "what if this leg had been split into 4 shorter legs instead of 1") and report the token/dollar reduction multiple for each. Recommend the matching fix from the technique map (loop engineering, compaction, subagents, RAG-style compiled reference).

## Step 8 — File the findings

Write a case-study page to `llm-wiki/knowledge-base-operations/[project]-token-audit-[YYYY-MM-DD].md` with: session dates, total turns, total tokens, leg-by-leg base/increment, actual vs. no-caching cost, the compaction-cadence comparison, and the specific recommendation for this project. Add it to `llm-wiki/knowledge-base-operations/_index.md`.

Append to `llm-wiki/log.md`:
```
## [TODAY'S DATE] token-audit | [project] — N turns, X tokens, $Y actual ($Z uncached)
Dominant term: [base|increment]. Recommendation: [fix].
Filed into: wiki/knowledge-base-operations/[project]-token-audit-[date].md
```
