# Module 06: Context as Finite Resource

**Time:** ~1.5 hours (≈45 min reading · ≈30 min hands-on · ≈15 min reflection)
**Builds on:** Module 01 (Workflows vs Agents), Module 04 (Advanced Tool Use)    **Feeds:** Module 07 (Compaction, Notes, Memory), Module 08 (Sub-Agents)

## Learning Objectives

- Explain why context window size and context quality are *inversely correlated* under real workloads — and what "context rot" means mechanically.
- Distinguish the three types of context growth (conversational accumulation, tool result flooding, cross-session state) and match each to its appropriate management strategy.
- Apply Contextual Retrieval to improve retrieval accuracy by 49-67% over standard RAG.
- Reason about which information deserves inclusion in a context budget and which should be loaded just-in-time.

---

## 1. Concept Synthesis

### Context as an attention budget

Both workflows and agents share a hard constraint: the model's context window is finite, and its ability to attend to information degrades as it fills. Anthropic's *Effective Context Engineering for AI Agents* (Sep 2025) opens with the framing that matters:

> Context engineering represents a shift from traditional prompt engineering toward managing the entire information landscape available to language models during inference.

Prompt engineering asks "what should I say to the model?" Context engineering asks "what should the model be able to see at this moment, given everything the conversation has accumulated?" The distinction becomes critical as agents operate across longer time horizons.

The architectural constraint behind context degradation: transformer models require n² pairwise attention relationships for n tokens. As sequences grow, the model's attention budget becomes thinner per token. Additionally, models have more training experience with shorter sequences, meaning their "context-wide dependencies" — the ability to connect information across large distances in the window — are weaker than their local reasoning.

This produces **context rot**: retrieval accuracy declines measurably as context window size increases, even when the relevant information is present. The problem is not that the information disappears — it's that attention quality degrades. The same information in a 10,000-token context is more accessible than the same information buried in a 100,000-token context.

*— Ch 10 (effective-context-engineering-for-ai-agents)*

### The core principle: smallest high-signal set

The entire discipline of context engineering reduces to one principle:

> **The smallest possible set of high-signal tokens that maximize the likelihood of some desired outcome.**

Every token in the context window has a cost: it consumes attention budget, contributes to context rot, and pushes other tokens farther from the model's focus. The goal is not "include everything relevant" but "include only what's necessary, at maximum signal density."

This principle has a corollary: information that *can* be loaded on demand should *not* be loaded upfront. The human analogy is accurate — we don't memorize entire encyclopedias before a research task; we maintain lightweight references and look things up when we need them. Agents should work the same way: keep pointers, retrieve content just-in-time.

*— Ch 10 (effective-context-engineering-for-ai-agents)*

### Three types of context growth

Long-running agents face three distinct context growth problems, and each requires a different fix. The context engineering notebook makes this taxonomy explicit:

**1. Conversational accumulation.** As interactions and reasoning steps accumulate, the dialogue grows. User messages, assistant turns, and reasoning chains pile up. This is a *whole-transcript* problem: every part of the conversation contributes.

Fix: **Compaction.** Summarize the accumulated conversation and reinitiate with the condensed summary. The summary preserves architectural decisions, unresolved questions, and key findings while discarding redundant content and old tool outputs no longer needed. This is lossy by design — but the right losses.

**2. Tool result flooding.** When agents call tools that return large responses (file contents, API responses, document chunks), those results become part of message history and count against the context budget on every subsequent turn. An agent reading eight 40,000-token research documents produces ~320,000 tokens of tool-result volume.

Fix: **Tool-result clearing.** Walk the message list and surgically replace old `tool_result` content blocks with short placeholders, while keeping the `tool_use` records intact. The model still knows it made the call; it just doesn't carry the full payload forward. If the agent needs the data again, it calls the tool again. This is a *sub-transcript* operation that targets only tool results — safer and lighter than compaction.

**3. Cross-session state.** Complex tasks span multiple sessions. Information needed in session 3 was generated in session 1 — but session 3 starts with a clean context window.

Fix: **Memory / structured note-taking.** The agent writes notes to external storage during execution, then reads them back at the start of subsequent sessions. Implement the storage backend yourself; control what's stored and for how long. Notes that matter: progress trackers, architectural decisions, key findings, unresolved questions.

*— Ch 10 (effective-context-engineering-for-ai-agents)*

### What deserves space in the context budget

Not all context is equal. The article's guidance for each component:

**System prompts** should strike a balance between specificity and flexibility:
- Use XML tags or Markdown headers for clear organizational structure
- Simple, direct language at appropriate abstraction levels
- Minimal yet sufficient: include information needed for desired behavior, exclude everything else
- Diverse, canonical examples rather than exhaustive edge cases (3 good examples > 30 mediocre ones)

**Tools** should return token-efficient information:
- Resolve UUIDs to readable names (Module 03)
- Return only relevant fields, not full API responses
- Minimal viable toolset: fewer tools = less context overhead at every turn
- Self-contained, unambiguous descriptions that don't require the model to reference other tools

**Message history and few-shot examples** should be curated:
- Diverse, canonical examples that portray expected behavior
- Do not stuff exhaustive edge cases — this produces context pollution, not better performance

**Dynamic retrieval** should replace static loading wherever possible:
- Maintain lightweight references (file paths, query strings, database IDs)
- Load data at runtime through tools, only when the agent needs it
- "Just-in-time" retrieval enables progressive disclosure: the agent discovers what it needs as the task unfolds, rather than committing upfront to a potentially wrong information set

*— Ch 10 (effective-context-engineering-for-ai-agents)*

### Contextual Retrieval: when retrieval itself is the problem

The techniques above address context management *during* agent execution. But there's an earlier failure mode: retrieval quality. Standard Retrieval-Augmented Generation (RAG) fails in a specific way that the Contextual Retrieval article (Sep 2024) precisely diagnoses.

Standard RAG breaks a knowledge base into chunks (typically a few hundred tokens each), embeds them, and retrieves by semantic similarity. The problem: chunks lose their surrounding context when extracted. A chunk containing "The company's revenue grew by 3% over the previous quarter" is meaningless without knowing which company and which quarter. The retrieval system can't connect this chunk to a query about "ACME Corp Q2 2023 revenue growth" because the chunk contains no identifying information.

Contextual Retrieval adds a 50-100 token context prefix to each chunk before embedding:

**Original chunk:**
> "The company's revenue grew by 3% over the previous quarter."

**Contextualized chunk:**
> "This chunk is from an SEC filing on ACME Corp's performance in Q2 2023; the previous quarter's revenue was $314 million. The company's revenue grew by 3% over the previous quarter."

The context is generated by passing the whole document and the chunk to Claude with a simple prompt asking for a "short succinct context to situate this chunk within the overall document for the purposes of improving search retrieval."

The performance numbers:

| Approach | Top-20-chunk failure rate |
|---|---|
| Standard embeddings | 5.7% |
| Contextual Embeddings alone | 3.7% (−35%) |
| Contextual Embeddings + Contextual BM25 | 2.9% (−49%) |
| Reranked Contextual Embedding + Contextual BM25 | 1.9% (−67%) |

Cost with prompt caching: **$1.02 per million document tokens** for one-time contextualization. This is a preprocessing cost, not a per-query cost — you pay once and reuse.

**Why BM25 alongside embeddings?** Embedding models excel at semantic similarity but miss exact lexical matches. A query for "Error code TS-999" should find the chunk that contains "TS-999" verbatim, but an embedding model might retrieve chunks about generic error handling instead. BM25 (lexical matching, built on TF-IDF) finds those exact matches. The two techniques are complementary: embeddings for semantic meaning, BM25 for precise terms. Combined, they outperform either alone.

*— Ch 01 (contextual-retrieval)*

### Putting it together: context management as a decision tree

When designing a long-running agent, the context engineering questions to answer, in order:

1. **Retrieval layer:** are you using RAG? If yes, apply Contextual Retrieval (add 50-100 token prefix per chunk, combine embeddings + BM25).
2. **Tool design layer:** do tools return raw API responses? Apply token efficiency principles — filter, paginate, truncate (Module 03).
3. **Session layer:** does the task span multiple sessions? Implement a memory/note-taking mechanism.
4. **Execution layer:** are tool results accumulating? Configure tool-result clearing at an appropriate token threshold.
5. **Conversation layer:** is the whole conversation growing? Configure compaction at an appropriate token threshold.
6. **System prompt layer:** is the system prompt longer than necessary? Cut to minimum viable specification.

Each layer is independent. A well-designed agent applies the right fix at the right layer, rather than trying to solve all growth problems with a single technique.

*— Ch 10 (effective-context-engineering-for-ai-agents)*

---

## 2. Key Questions (the depth check)

<details>
<summary><b>Q1.</b> An agent is doing complex research across many documents. By turn 20, its performance has degraded noticeably even though all relevant information is still in context. What's the mechanism, and which type of context growth is most likely causing this?</summary>

**Context rot from tool result flooding.** If the agent has read multiple large documents, each read produces a large `tool_result` block that stays in context for all subsequent turns. By turn 20, the context window may contain hundreds of thousands of tokens of document content, most of which is no longer relevant to the current step. The n² attention mechanism means the model's ability to focus on recent, relevant content degrades as older, bulkier content occupies attention bandwidth. Tool-result clearing — replacing old tool_result blocks with placeholders — is the targeted fix. Compaction is an alternative but is a heavier-weight operation that also discards useful conversational context.
</details>

<details>
<summary><b>Q2.</b> Why does Contextual Retrieval use two sub-techniques (Contextual Embeddings + Contextual BM25) rather than just improving the embedding model?</summary>

Embedding models and BM25 capture different types of relevance. Embeddings capture semantic similarity — chunks whose meaning is related to the query. BM25 captures lexical overlap — chunks that contain the exact words from the query. These complement each other: a query for a specific error code or product name requires lexical matching (BM25), while a conceptual query requires semantic matching (embeddings). Improving only the embedding model leaves the lexical matching gap unaddressed. The combination reduces top-20-chunk failure rates from 5.7% (embeddings) to 2.9% (both) — the improvement is additive precisely because the two techniques fail on different query types.
</details>

<details>
<summary><b>Q3.</b> Tool-result clearing keeps `tool_use` records but deletes `tool_result` content. Why is this asymmetry the right design?</summary>

The `tool_use` record tells the model *what it tried* — which function it called with which parameters. This is cheap (small) and valuable (the model needs to know it already explored a path to avoid repeating it). The `tool_result` content is the data returned — often large and frequently re-fetchable if the agent needs it again. The asymmetry preserves the agent's memory of its own actions while discarding the payload. If the agent needs the data, it calls the tool again; the tool is cheaper than permanently bloating the context window.
</details>

<details>
<summary><b>Q4.</b> When should you use compaction vs. tool-result clearing? What's the decision criterion?</summary>

Tool-result clearing is a *surgical* operation that targets only tool result payloads — appropriate when tool results are the primary source of context growth but the conversational context itself is valuable and shouldn't be discarded. Compaction is a *whole-transcript* operation — appropriate when the accumulated dialogue itself is the problem, including tool calls, reasoning, user messages, and everything else. Use tool clearing when: tool payloads are large, they're re-fetchable, and the conversation history is important to preserve. Use compaction when: the entire conversation is growing unmanageably and a high-fidelity summary can capture what matters.
</details>

<details>
<summary><b>Q5.</b> The principle "smallest possible set of high-signal tokens" sounds simple. What makes it hard to apply in practice?</summary>

Two problems. First, *relevance is task-dependent*: what counts as high-signal depends on what the agent is about to do next, which you often don't know when you're deciding what to include. A file that was irrelevant at turn 5 might be crucial at turn 20. Second, *signal density is hard to measure*: you can count tokens, but you can't easily measure how much value each token contributes to model performance. In practice, the principle translates into conservative heuristics: only load files when explicitly needed, clear tool results after they've been processed, summarize rather than preserve verbatim. You're approximating optimal, not computing it.
</details>

<details>
<summary><b>Q6.</b> Contextual Retrieval costs $1.02 per million document tokens for preprocessing. When is this not worth the cost?</summary>

When the query distribution is simple enough that standard embeddings suffice. If users ask broad semantic questions ("summarize the revenue trends") rather than specific queries ("what was ACME's Q2 2023 revenue growth"), the context prefix adds cost without meaningful accuracy gain — the chunks are already retrievable by semantics. Also not worth it for small corpora where failure rates are already low, or for one-time queries where the preprocessing cost exceeds the value of improved retrieval. The 49-67% failure reduction matters most when: (a) queries are specific and varied, (b) chunks lose critical context when extracted, and (c) the corpus will be queried many times (amortizing preprocessing cost).
</details>

---

## 3. Hands-On

**Notebook:**
- [`claude-cookbooks/tool_use/context_engineering/context_engineering_tools.ipynb`](../claude-cookbooks/tool_use/context_engineering/context_engineering_tools.ipynb)

**Run as-is.**

The notebook demonstrates all three context management primitives on a single biology research agent that reads through a corpus of review documents and synthesizes findings. Pay attention to:

- **Baseline run:** watch the token trajectory graph. Note how tool results from reading documents accumulate dramatically — 320,000+ tokens from 8 documents of ~40K tokens each.
- **Compaction demo:** observe what the compaction summary preserves vs. discards. The summary keeps organism names, key stats (lifespan, tractability), and open questions; it discards full document content and intermediate reasoning.
- **Tool-result clearing:** contrast with compaction — the clearing operation targets only `tool_result` blocks, leaving the agent's reasoning and user messages intact.
- **Memory:** see how the agent writes structured notes (organism summaries) to external storage and reads them back to resume without the full context.

**One modification (≈15 min): inspect the compaction summary quality.**

After the compaction demo runs, read the generated summary carefully. Ask yourself: what would an agent need to continue the task from the summary alone? What critical information was preserved? What was lost? Write down three things the summary captured well and one thing it missed that a future agent would need.

Then re-run with a custom `instructions` parameter that prioritizes specific information:
```python
def demo_compact(client, messages, model):
    ...
    # Add: instructions="Prioritize: (1) organism names, (2) lifespan values, 
    #   (3) genetic tractability ratings. Discard: full document quotes, 
    #   intermediate reasoning steps."
```

Compare the two summaries.

**What to record in your notes:**
- The peak token count in the baseline run vs. after compaction.
- Three things the default compaction summary preserves vs. one thing it loses.
- Which primitive (compaction, clearing, or memory) would you reach for first in an agent you're building, and why.

---

## 4. Reflection

1. **The "just-in-time" retrieval principle sounds right, but it has a failure mode.** If the agent must decide what to load on demand, it might miss that it *needs* a particular piece of information until it's already made a wrong decision without it. Where does eager loading (loading relevant context upfront) produce better outcomes than just-in-time retrieval, and how do you identify those cases in advance?

2. **Contextual Retrieval adds a 50-100 token prefix per chunk.** For a large knowledge base, this is substantial storage overhead. The article claims the accuracy improvement justifies it. Design an experiment to test this claim on your own domain: what queries would reveal whether the prefix is helping or not? What's the cheapest way to run that experiment?

3. **The three context growth types (conversational, tool results, cross-session) are treated independently.** But in practice, they interact: a compacted conversation loses context that a note-taking system might have preserved. What's the right integration strategy — should memory notes be generated during compaction, or separately, or both?

---

## 5. Key Takeaways

- **Context rot is a real, measurable phenomenon.** As context grows, retrieval accuracy degrades — not because information disappears, but because n² attention becomes thinner. The fix is not a bigger context window; it's more disciplined context management.
- **Three growth types, three fixes.** Conversational accumulation → compaction. Tool result flooding → tool-result clearing. Cross-session state → structured note-taking. Each fix is targeted; don't apply a heavy operation (compaction) to solve a targeted problem (tool result flooding).
- **Smallest high-signal set is the governing principle.** Every token has a cost. Tokens that can be loaded just-in-time should not be loaded upfront. Tools should return filtered, relevant data — not raw API responses.
- **Contextual Retrieval reduces RAG failures by 49-67%.** Adding a 50-100 token context prefix to each chunk (generated by Claude, cached for reuse) addresses the fundamental RAG failure mode: chunks that lose meaning when extracted from documents.
- **BM25 + embeddings beats either alone.** Semantic similarity (embeddings) and lexical matching (BM25) fail on different query types. Combining them is additive, not redundant.
