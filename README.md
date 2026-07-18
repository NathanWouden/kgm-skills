# Token spend skills

Two Claude "skills" — plain markdown instruction files, no code or install required. Either drop them into a Claude Code `skills/` folder to use as slash commands, or just paste the file's content into any Claude conversation (Claude.ai, Cowork, Claude Code) and ask your question — the instructions work the same way either way.

## token-economics-formula.md

**What it is:** A reference framework for explaining and calculating LLM token spend — one formula (`Total(N turns) ≈ N×base + increment×N(N+1)/2`) plus a map of which optimization technique (RAG, prompt caching, compaction, subagents, fine-tuning, self-hosted GPUs, etc.) fixes which part of that formula.

**How to use it:**
- **Claude Code:** save it in your project's `skills/` folder — Claude will pull it in automatically whenever you ask about token costs, API pricing, RAG vs. fine-tuning, or "why is my AI bill high."
- **Any other Claude interface (Claude.ai, Cowork, etc.):** paste the file's contents at the start of a conversation, then ask your question — e.g. "using this framework, help me figure out if I should fine-tune or use RAG for X." Claude will apply the formula and technique map to your specific numbers.
- No file access or tools needed — it's pure reasoning/explanation, so it works even in a plain chat with no agentic capability.

## token-spend-audit.md

**What it is:** A step-by-step method to audit a project's *actual, measured* Claude Code token usage — not an estimate. It reads real session transcripts, fits the token-economics formula to what really happened, prices it in dollars, and produces a concrete cost-reduction recommendation.

**How to use it:**
- **Requires Claude Code specifically** (or another agent with file-reading and bash access) — it needs to read local session log files (`~/.claude/projects/.../*.jsonl`), so it won't work in a plain chat interface with no file access.
- In Claude Code: save it in your `skills/` folder, then run `/token-spend-audit <project name or path>` (e.g. `/token-spend-audit bs-crm`).
- It depends on `token-economics-formula.md` for the underlying formula and technique map, so keep both files together.
- Output: a written case-study report with real token/turn counts, actual vs. no-caching cost, and a specific recommended fix.
