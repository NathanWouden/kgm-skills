---
name: token-economics-formula
description: "Explains and applies the LLM token-cost formula Total(N turns) ≈ N×base + increment×N(N+1)/2 to analyze and reduce token spend, for both personal use (context engineering, loop engineering, second-brain/Obsidian/Claude Projects setups) and production systems (RAG, fine-tuning, self-hosted/rented GPU inference). Use this skill whenever the user asks about token costs, API pricing, why their AI bill is high, how to optimize prompts/context/loops to save money, RAG vs fine-tuning tradeoffs, whether to self-host or rent GPUs for an LLM, break-even analysis for fine-tuning or training, or wants a worked numeric example of any of these. Trigger even if the user doesn't say 'formula' explicitly — e.g. 'my Claude Code tokens are climbing fast', 'should I fine-tune or use RAG', 'is it worth running my own model on Azure', 'how do I reduce context cost', 'what's the ROI on training my own model'."
---

# Token Economics Formula

A framework for explaining and calculating LLM token spend, covering both personal-scale habits (context/loop engineering, Obsidian-style second brains, Claude Projects) and production-scale architecture (RAG, fine-tuning, self-hosted/rented GPU inference). This skill exists because token-cost questions are almost always really asking "where exactly is my money going, and which of several different fixes actually addresses that," and the formula below is the throughline that makes every fix legible as attacking one of three specific variables.

## When to reach for this skill

Use it for: rising token bills, "why does my agent loop cost so much," RAG vs. fine-tuning decisions, whether to self-host a model on a rented GPU, break-even analysis for any of the above, or requests for a worked example with real numbers. Don't use it for simple one-off "how many tokens is this string" questions — this skill is for the *economics*, not basic tokenization.

## The core formula

```
Total(N turns) ≈ N × base + increment × N(N+1)/2
```

This models the cost of N independent calls **or** N turns of one accumulating conversation/agent loop.

- **N** — turn count / call count. How many times a call happens: turns in one conversation, iterations in an agent loop, or independent calls per day if there's no shared growing history.
- **base** — the flat cost paid by every single call regardless of how far into the session you are: system prompt, instructions, any static knowledge injected (e.g. a full product catalog pasted into every call).
- **increment** — how much *new* content one turn adds to the running history, which then gets dragged forward into every subsequent turn if nothing prunes it.
- **N(N+1)/2** — shorthand for "1+2+3+...+N" (the sum of all turns' worth of accumulated history). This is why an unmanaged multi-turn conversation grows roughly with the *square* of its length, not linearly. Doubling N roughly quadruples this term.

**If calls are independent** (no growing history — e.g. a production API serving stateless requests), increment effectively doesn't apply, and the formula collapses to `Total = N × base`. This is the production-system case. The full quadratic term only matters within one accumulating session — chat, debugging thread, or agentic loop.

## Standard response pattern (what works)

When walking someone through this, the structure that lands well:

1. State the formula and define each variable in one plain sentence.
2. Pick ONE concrete anchor scenario and reuse it across every comparison (e.g. "100 product-catalog queries/day" or "a 6-turn debugging session") so before/after numbers are directly comparable.
3. Do the turn-by-turn manual sum once, then show it matches the formula shortcut — this builds trust that the formula isn't a black box.
4. Convert tokens → dollars using **current, verified** pricing (web-search it; don't rely on memory — prices change). State the assumption (which model, input vs. output split) explicitly.
5. Present a "before" state and one or more "after" states, and always give the reduction multiple (e.g. "~195x") AND the absolute dollar figure — the multiple convinces technically, the dollar figure convinces a non-technical stakeholder.
6. Name explicitly which term of the formula each fix attacks. This is the single most useful habit in this whole skill — it turns a grab-bag of "optimization tips" into a structured decision tree.
7. Always include the honest caveat / failure mode for whatever technique you just praised. Nothing in this space is a free lunch.

## Technique → formula-term map

**Shrinks `base`:**
- RAG (retrieve top-k relevant items instead of dumping the full knowledge base)
- Prompt caching (repeated static prefix billed at a fraction of input price on cache hits)
- Prompt/few-shot compression (fewer, tighter instructions and examples)
- Data segmentation (classify into a category first, search only within it — improves retrieval precision; doesn't shrink the final retrieved-token count much, but cuts the search space)
- Per-item compression (structured/compact representation instead of prose, e.g. `SKU421|Olive Oil|€4.20` instead of a full sentence)
- Claude Projects / persistent knowledge bases (retrieval-based context attached once per workspace instead of restated every chat)
- Obsidian-style "second brain" / Karpathy LLM-wiki pattern (human- or agent-curated linked notes you retrieve from instead of re-deriving everything each session) — same job as RAG, just hand-built; the honest counterpoint is that the *retrieval* (an LLM following links/grep) burns tokens, where a vector database's search does not
- Tool Search / deferred tool loading (load only the tool definitions actually needed instead of every connected MCP server's full schema upfront)

**Caps `increment` (the quadratic term):**
- Context engineering (curate what enters context; prune, don't resend everything)
- Loop engineering (bound how a multi-step loop unfolds: cap iterations, define a stopping rule, batch independent tool calls in parallel)
- Summarization instead of full transcript resend
- Sliding window / state objects instead of full dialogue history
- Subagents / Task-Explore (push noisy exploratory search into a separate context that returns only a short summary to the main thread)
- Compaction (periodically rewrite history into a condensed summary)
- Obsidian's "close the session with a note, start the next one fresh" habit — the personal-scale, manual version of compaction

**Collapses `N` itself:**
- Fine-tuning reasoning/judgment into the model's weights so a multi-step scaffolded process becomes a single pass
- Single-pass prompting (specify clearly enough upfront that the task doesn't need a clarify-retry cycle)

**Important: the model does NOT do this for you by default.** A model is trained to complete the task correctly, not to minimize token spend while doing it. Nothing in standard training tells a model "you've had 14 unpruned turns, summarize now." Bounding the loop is the user/harness's job unless explicitly engineered in (see "Efficient reasoning" below for the one place this gets trained in directly).

## Worked example template

Reuse this shape for any new scenario:

```
Turn k cost = base + k × increment

Total(N) = Σ(k=1 to N) [base + k×increment]
         ≈ N×base + increment × N(N+1)/2
```

Manual check: list turn 1..N costs, sum them, confirm it matches the formula. Then convert to $: `(input_tokens × input_price + output_tokens × output_price) / 1,000,000`. **Always web-search current model pricing before quoting it** — it changes, and a stale number undermines the whole exercise. As of this writing the input:output ratio for Claude models runs about 1:5 (e.g. Sonnet-tier: $3 input / $15 output per MTok) — note this explicitly when output share of a call grows large (it does once `base` has been optimized down, since output stops being a rounding error).

## RAG, mechanically (for explaining "what does RAG reduce")

RAG decouples per-call token cost from total knowledge-base size. Without it: tokens needed ∝ data_quantity × tokens_per_item, scaling with everything you know. With it: tokens needed ≈ a fixed k × tokens_per_item, regardless of catalog size, because the expensive search step (comparing a query against everything) happens in a vector database — ordinary, non-token-priced compute (cosine similarity over embeddings, sub-linear with proper indexing) — and only the top-k *result* of that search reaches the LLM's context window. Segmentation and per-item compression refine the retrieval step but don't replace it; RAG is what makes catalog size a non-factor in the first place.

Production platforms doing this: Azure AI Search + Microsoft Foundry (two separately-named products you connect) and Amazon Bedrock Knowledge Bases (bundled into one managed capability, vector store configurable — OpenSearch Serverless default, also Pinecone/MongoDB Atlas/Aurora/S3 Vectors). Claude integrates natively with Bedrock's RetrieveAndGenerate API via model ARN; Azure Foundry defaults toward Azure OpenAI models, so calling Claude there usually means using AI Search purely as the retrieval layer and calling Claude directly for generation.

## Self-hosted / rented GPU inference economics

This is the part where intuition usually breaks, so spell it out explicitly: **using your own fine-tuned model does NOT mean the dollar-per-token cost of the formula just gets smaller.** It means the billing unit changes entirely, from "$ per token" to "$ per GPU-hour" — and that change can make things *worse* at low volume even though raw token count per call dropped.

```
effective $/MTok (self-hosted) = (GPU $/hour × hours running) ÷ (tokens processed in that period) × 1,000,000
```

This is inversely proportional to volume — the opposite of metered API pricing, which is flat regardless of volume. A GPU billed by the hour costs the same whether it processes 1 token or 10 million that hour, so effective per-token cost is brutal at low utilization and becomes competitive (or cheaper) only once utilization is high. Always web-search current GPU rental pricing (Azure/AWS/GCP/neoclouds like RunPod/Lambda/Thunder Compute — rates move quickly and vary a lot by GPU generation and provider). Compute the break-even call volume where self-hosted effective $/MTok crosses below the metered API rate you'd otherwise pay — that's the real decision threshold, separate from (and in addition to) the training break-even below.

## Training cost (a separate multiplier, not a flat fee)

Training/fine-tuning cost isn't one clean number — it's the same formula's logic applied with a very different N:

```
N_train = (labeled training examples) × (rollouts per example, if using RL) × (epochs)
```

For supervised fine-tuning, rollouts-per-example is effectively 1. For RL-based efficient-reasoning training (length-penalized reward alongside an accuracy reward — real technique, active research area, e.g. budget-based rewards, dynamic length rewards, token-significance-aware penalties), rollouts-per-example is typically 4–64, which is why RL training cost multiplies up fast even from a modest labeled set. Reward-hacking risk: if the length penalty outweighs the accuracy reward too heavily, the model finds the degenerate shortcut of being short and wrong instead of being right — weight accuracy heavily, length lightly, and watch eval accuracy as a canary.

**Break-even** = one-time training/fine-tuning cost ÷ (monthly $ saved per call at the target volume, from the base-formula comparison above). Compute this at more than one plausible volume — the same training investment can be a multi-year payback at low volume and a three-week payback at high volume. Volume, not task difficulty, is what flips the sign.

## Decision checklist (synthesizes the whole skill)

A task is a good candidate for fine-tuning/self-hosting when, and only when, all three hold: the input/output shape is stable (not still actively changing), the call volume is high enough that the per-call token tax compounds past the training cost within a reasonable payback window, and (if self-hosting) volume is high enough to keep a rented GPU's utilization — not just its existence — economically justified. If the task is rare but hard: stay in context/loop-engineering territory regardless of difficulty. If the knowledge is large but volatile (prices, inventory, anything that changes often): RAG, not fine-tuning — fine-tuning would mean retraining every time the data changes; RAG just means re-embedding the changed records.

## Honest caveats to always include

- Output tokens typically cost ~5x input tokens — once `base` is optimized down, output becomes a much larger and more important share of remaining cost; don't optimize input and ignore output length.
- Self-hosting inference is not automatically cheaper just because token count per call dropped — check utilization-adjusted effective $/MTok, not raw token count.
- "Hand-built RAG" (Obsidian/LLM-wiki patterns) still costs real tokens for the retrieval step itself (an LLM reading an index, following links, grepping), unlike a vector database's search which is free relative to LLM billing. It saves the *re-explaining* tax but isn't a free escape from the formula.
- A model's own iterative behavior (agent loops, vibe-coding back-and-forth) is not "loop engineering" — it's the mechanism loop engineering manages. Don't conflate the two when explaining this to someone.
- Always verify current pricing (model $/MTok, GPU $/hour) via web search before presenting dollar figures — this entire framework is dollar-sensitive and prices move.
