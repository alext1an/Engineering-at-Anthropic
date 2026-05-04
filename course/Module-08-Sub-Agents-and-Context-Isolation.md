# Module 08: Sub-Agents & Context Isolation

**Time:** ~1.0 hours (≈30 min reading · ≈15 min hands-on · ≈15 min reflection)
**Builds on:** Module 02 (Composition Patterns), Module 06 (Context as Finite Resource)    **Feeds:** Module 09 (Harness Design), Module 13 (Multi-Agent)

## Learning Objectives

- Articulate why context isolation is the key architectural benefit of sub-agents — not parallelism, not cost, but the clean information boundary.
- Design a sub-agent invocation that correctly specifies objective, output format, tool budget, and context scope.
- Predict when single-agent vs. sub-agent architecture produces better results, given a specific task structure.
- Explain the model cost asymmetry pattern (expensive orchestrator + cheap sub-agents) and when it's justified.

---

## 1. Concept Synthesis

### Why sub-agents exist: context isolation, not just parallelism

Sub-agents are routinely described as a parallelization strategy: spawn multiple agents, run them concurrently, collect results. This framing is true but incomplete. The deeper benefit is **context isolation**: each sub-agent starts with a clean, focused context window containing only what's relevant to its specific task. The orchestrator's accumulated context — its reasoning history, prior tool results, its model of the overall problem — doesn't bleed into the sub-agent's window.

This matters because context contamination is a real problem. An agent that has already concluded "the answer is X" brings that prior to any further investigation. A fresh sub-agent, given only the sub-task, has no prior. It can contradict what the orchestrator thinks it already knows.

The Module 02 insight about generator-evaluator isolation applies here: what matters isn't model identity (you can use the same model class) but *context isolation*. An evaluator with a different context window is more reliable not because it's a different model, but because it hasn't been primed by the generator's reasoning. Sub-agents benefit from the same property.

From Anthropic's *How We Built Our Multi-Agent Research System* (Jun 2025):

> **Subagents facilitate compression by operating in parallel with their own context windows, exploring different aspects of the question simultaneously before condensing the most important tokens for the lead research agent.**

The "condensing" part is as important as the "parallel" part. Each sub-agent explores extensively within its isolated context, then returns a condensed summary (typically 1,000-2,000 tokens) to the orchestrator. The orchestrator sees a compressed, high-signal view of each sub-task — not the full exploration trace.

*— Ch 06 (built-multi-agent-research-system), Ch 10 (effective-context-engineering-for-ai-agents)*

### What context isolation prevents

Consider the alternative: a single agent trying to process four quarterly financial reports to answer "how did net sales change quarter to quarter?" The agent reads Q1 (with its full PDF content in context), forms an initial understanding, reads Q2, reads Q3, reads Q4. By the time it synthesizes, its context window contains:
- Its own prior reasoning about each quarter
- All the extracted tables and financial figures
- Its in-progress hypothesis formation
- Error corrections and clarifications from multiple turns

The context is rich but noisy. The agent must attend to all of it while synthesizing. Context rot (Module 06) degrades its ability to connect early insights with later ones.

With sub-agents:
- Sub-agent 1 reads Q4 in isolation, extracts only the relevant data, returns 200 tokens
- Sub-agent 2 reads Q3 in isolation, same process
- Sub-agents 3 and 4 for Q2 and Q1
- Orchestrator receives four clean 200-token summaries, synthesizes with minimal noise

The orchestrator's synthesis context contains exactly what it needs for the final answer, nothing more. This is context isolation as a deliberate design choice.

*— Ch 06 (built-multi-agent-research-system)*

### Model cost asymmetry: expensive orchestrator, cheap sub-agents

The sub-agents notebook demonstrates a practical cost pattern: Claude Opus (expensive, high-capability) as orchestrator, Claude Haiku (cheap, fast) as sub-agents.

The division of labor justifies the asymmetry:
- **Orchestrator (Opus):** Generates the sub-agent prompt from the user question; synthesizes results into final answer; writes visualization code; requires high-capability reasoning
- **Sub-agents (Haiku):** Extract specific structured information from a single document; task is constrained and mechanical; cheap models are sufficient

The orchestrator needs to understand the *intent* of the user's question and craft sub-agent prompts that capture what to look for. This is genuine reasoning work. The sub-agents need to follow a specific prompt against a specific document. That's pattern-matching work — cheaper models handle it well.

This cost structure is worth thinking through for any sub-agent architecture: what's the reasoning complexity at each level, and does the model tier match?

*— Sub-agents notebook (using_sub_agents.ipynb)*

### Designing sub-agent invocations

The research system article identifies four elements each sub-agent needs to perform well:

1. **Objective:** What specifically to find or produce. Vague objectives produce vague outputs.
2. **Output format:** How to structure the result — XML tags, JSON, numbered list. Without a format specification, sub-agent outputs require additional parsing work on the orchestrator side.
3. **Tool guidance:** Which tools to use, in what order, when to stop. Without this, sub-agents either over-invest ("searched 50 sources for a simple fact") or under-invest ("gave up after one search that returned nothing").
4. **Source guidance:** Where to look, what not to look at, quality criteria for sources.

The article gives a concrete failure mode: when the lead agent allowed simple instructions like "research the semiconductor shortage," sub-agents duplicated work and explored the same search trajectories. One explored the 2021 automotive chip crisis; two others duplicated investigation of 2025 supply chains.

Detailed task descriptions — specifying the exact angle, the tool budget (3-10 calls for simple facts, 10-15 for comparisons), and the output format — prevented duplication and left gaps.

The notebook's orchestrator-generated prompt for Haiku sub-agents is a clean example:
```
Extract the following information from the Apple earnings report PDF for the quarter:
1. Apple's net sales for the quarter
2. Quarter-over-quarter change in net sales
3. Key product categories, services, or regions that contributed significantly to the change
4. Any explanations provided for the changes in net sales

Organize the extracted information in a clear, concise format focusing on the key data points.
```

This is specific about *what to extract* (four named items), *what output format to use* (clear, concise), and implicitly bounds scope to *one quarter's document*. Each sub-agent knows its task.

*— Ch 06 (built-multi-agent-research-system)*

### When sub-agents help vs. hurt

The 15× token cost of multi-agent systems (Module 02) has concrete implications for sub-agent design. The research system article's own guidance:

| Task type | Sub-agents needed | Tool calls each |
|---|---|---|
| Simple fact-finding | 1 | 3-10 |
| Direct comparisons | 2-4 | 10-15 |
| Complex research | 10+ | Many, divided |

Sub-agents earn their cost when:
- **Tasks are genuinely independent.** Reading Q1, Q2, Q3, Q4 of Apple financials is independent — each quarter doesn't affect what the other finds. A multi-file refactor is NOT independent — changes to `auth.py` affect what changes are needed in `session.py`.
- **Each sub-task requires substantial exploration.** If the sub-task is a single lookup, the sub-agent overhead (spawning, prompt, context) isn't worth it.
- **Context isolation provides signal value.** If each sub-agent will bring different information that would contaminate each other's exploration, isolation helps.

Sub-agents hurt when:
- **Tasks are sequential.** Each step depends on the prior result. A single agent with sequential tool calls is faster and cheaper.
- **Context sharing would help.** If all sub-tasks need the same background context, isolating them means duplicating that context across every sub-agent.
- **The synthesis step requires full information.** If the orchestrator needs to compare sub-tasks that require shared context to compare, isolation fragments that context unhelpfully.

The research system article's coding note is pointed: "Most coding tasks involve fewer truly parallelizable parts than research" — even large refactors are mostly serial, and the sub-agent overhead produces no gain.

*— Ch 06 (built-multi-agent-research-system)*

### Context isolation applied to the evaluator-optimizer pattern

The evaluator-optimizer pattern from Module 02 is a special case of context isolation with a specific application: independent evaluation. The evaluator gets only the generator's *output*, not its *reasoning trace*. This is the isolation property that makes the evaluator reliable.

The same principle extends to any situation where you want uncontaminated judgment:
- Code reviewer that shouldn't see the author's explanation of why they chose the approach
- Fact-checker that shouldn't see the draft before seeing the sources
- Security auditor that shouldn't see the "this is safe" self-assessment before auditing

In each case, the value comes from the clean context, not from using a different model.

*— Ch 10 (effective-context-engineering-for-ai-agents)*

---

## 2. Key Questions (the depth check)

<details>
<summary><b>Q1.</b> Two teams are designing multi-agent systems. Team A uses Haiku for all agents (orchestrator and sub-agents). Team B uses Opus for orchestration and Haiku for sub-agents, at ~3× the cost. Under what conditions is Team B's approach worth the cost, and under what conditions is Team A's approach fine?</summary>

Team B's Opus orchestrator earns its cost when synthesis requires reasoning across complex, multi-source information — formulating the right sub-agent prompts, resolving contradictions between sub-agent findings, generating code or analysis that integrates all outputs. Team A's all-Haiku approach is fine when: sub-tasks are so well-specified that orchestration is just routing (not reasoning), the final synthesis is templated, or the whole system is primarily doing extraction and formatting. The decision criterion: what's the reasoning complexity of the orchestrator's synthesis step? If it's high, the Opus upgrade pays for itself; if it's low, you're paying for capability you don't use.
</details>

<details>
<summary><b>Q2.</b> The sub-agents notebook uses ThreadPoolExecutor to run Haiku sub-agents concurrently. What would break if you ran them sequentially instead, and what wouldn't change?</summary>

**What breaks:** Wall-clock time. Running 4 sub-agents sequentially takes ~4× longer than in parallel. **What doesn't change:** Result quality. Each sub-agent is isolated — it doesn't see what other sub-agents found. Sequential vs. parallel execution produces identical outputs from each sub-agent because their context windows are independent. The only benefit of parallelism here is latency reduction, not accuracy improvement. (In contrast, for tasks where sub-agents influence each other, ordering would matter.)
</details>

<details>
<summary><b>Q3.</b> The article says the evaluator-optimizer pattern's value comes from "context isolation," not from "using a different model." What does this mean concretely?</summary>

You can run the evaluator and the generator using the *same* model class (both claude-sonnet-4-6, for example) and still get reliable independent evaluation — because the evaluator sees only the generator's output, not the generator's reasoning. The isolation is positional: the evaluator's context window starts fresh. In contrast, if you use the same model instance within the same context window (asking the model to "critique your own answer"), the model has been primed by its own reasoning and will systematically avoid contradicting its prior conclusions. It's the context boundary that matters, not the model identity.
</details>

<details>
<summary><b>Q4.</b> The research system's sub-agents each return 1,000-2,000 token summaries to the orchestrator. What would happen if they returned their full exploration traces instead?</summary>

The orchestrator's context would grow proportionally: with 10 sub-agents each returning 20,000 tokens, the orchestrator's synthesis context would contain 200,000 tokens of sub-agent trace before any synthesis work begins. The orchestrator would face the same context rot problem that sub-agents were designed to prevent. The 1,000-2,000 token summary limit is the mechanism that makes the pattern work — sub-agents compress information within their isolated contexts, then pass only essential findings to the coordinator. The compression step is not optional.
</details>

<details>
<summary><b>Q5.</b> Why does the research system's orchestrator generate a specific prompt for Haiku sub-agents (using Opus to write the sub-agent prompt) rather than using the same high-level user question directly?</summary>

The user's question ("how did Apple's net sales change quarter to quarter?") is framed for a human reader who has all quarters' context. A sub-agent given only Q4's PDF can't answer that question — it doesn't have Q1-Q3. The orchestrator's job is to translate the high-level question into a sub-task that a context-isolated agent can actually execute: "extract these four specific data points from this one document." Without this translation, sub-agents get questions they can't answer, or attempt to answer by scope-expanding ("let me find the other quarters too") which defeats isolation.
</details>

<details>
<summary><b>Q6.</b> The module identifies tasks where "context sharing would help" as a case where sub-agents hurt. Give a concrete example of such a task.</summary>

Multi-file debugging: "why does this function return wrong results?" The bug might exist in `utils.py` but manifest because of an import pattern in `main.py` that depends on an ordering assumption in `config.py`. If you isolate each file to a separate sub-agent, each agent sees only its file's logic and can't see the cross-file dependency that causes the bug. You need a single agent that can see all three files' contexts simultaneously to trace the call chain. Sub-agents are actively harmful here — they fragment the context you need to connect.
</details>

---

## 3. Hands-On

**Notebook:**
- [`claude-cookbooks/multimodal/using_sub_agents.ipynb`](../claude-cookbooks/multimodal/using_sub_agents.ipynb)

**Run as-is.**

Pay attention to:
- **Step 4: Orchestrator generates sub-agent prompt.** Opus takes the user's question and produces a specific extraction prompt for Haiku. Note what the orchestrator adds that the user's question didn't specify (output format, which specific data points to find).
- **Step 5: Sub-agents execute in parallel.** `ThreadPoolExecutor` runs all four PDF extractions concurrently. Each sub-agent sees one quarter's document only.
- **Step 6: Orchestrator synthesizes.** Opus receives the four tagged summaries and produces the final answer + visualization code. Note the size of what Opus receives (small) vs. what the sub-agents processed (full PDFs).

**One modification (≈15 min): break the context isolation.**

Instead of using sub-agents, modify the notebook to run a single agent that processes all four PDFs in sequence. Pass all extracted info in one long multi-turn conversation, asking the model to compare quarters as it reads each one.

Observe:
- Does the single agent's answer change as it reads more quarters (does earlier reasoning contaminate later conclusions)?
- Is the final synthesis better or worse than Opus's synthesis of the isolated sub-agent summaries?
- How does the token count compare?

This modification makes isolation vs. contamination concrete.

**What to record in your notes:**
- The size (tokens) of what each Haiku sub-agent received vs. what it returned.
- The size of what Opus received for synthesis vs. what it would have received if sub-agents returned full traces.
- Whether single-agent sequential processing produced a different conclusion than isolated sub-agents, and if so, how.

---

## 4. Reflection

1. **The cost asymmetry (Opus orchestrator + Haiku sub-agents) assumes the orchestrator's synthesis requires high capability.** Is this always true? Could you design the sub-agent protocol so that each sub-agent's summary is so well-formatted that synthesis is essentially concatenation, making a Haiku orchestrator sufficient? What would the sub-agent output format need to look like?

2. **Context isolation is presented as a benefit of sub-agents.** But what about coordination? In the research system, sub-agents can't see each other's findings and so might explore the same territory. The article showed this as a failure mode (duplicated work on semiconductor research). How would you design sub-agent protocols to prevent overlap without sacrificing isolation?

3. **The evaluator-optimizer pattern uses context isolation for unbiased judgment.** The sub-agent pattern uses it for focused exploration. Are there other uses of context isolation — situations where starting an agent with a clean window produces a qualitatively better outcome not because it's less biased, not because it reduces context rot, but for a third reason?

---

## 5. Key Takeaways

- **Context isolation is the primary benefit, not just parallelism.** Sub-agents explore their tasks in clean, focused windows — they can't be primed by the orchestrator's prior conclusions. Parallel execution is a secondary benefit (latency).
- **Design sub-agent invocations with four elements:** objective, output format, tool budget, and source guidance. Missing any of these leads to duplication, scope creep, or underinvestment.
- **Sub-agents return condensed summaries, not full traces.** 1,000-2,000 tokens to the orchestrator, regardless of how much the sub-agent explored internally. The compression step is not optional — it's the mechanism that keeps the orchestrator's synthesis context clean.
- **Expensive orchestrator + cheap sub-agents** is a rational cost structure when synthesis requires genuine reasoning and extraction is mechanical. Audit your own architecture's reasoning complexity at each level before committing to a model tier.
- **Sub-agents hurt on serial, dependency-heavy, or context-sharing tasks.** Coding tasks, sequential debugging, and cross-file analysis are poor fits. Research tasks with genuinely independent threads of inquiry are strong fits.
