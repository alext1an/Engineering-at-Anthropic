# Module 02: Composition Patterns

**Time:** ~1.5 hours (≈45 min reading · ≈30 min hands-on · ≈15 min reflection)
**Builds on:** Module 01 (Workflows vs Agents)    **Feeds:** Module 06 (Context), Module 09 (Harnesses), Module 13 (Multi-Agent)

## Learning Objectives

- Distinguish *parallelization* (predetermined subtasks), *orchestrator-workers* (subtasks decided at runtime), and *evaluator-optimizer* (iterative refinement of one artifact) — and explain why mixing them up costs money.
- Argue concretely when separating *generator* from *evaluator* beats prompting a model to self-critique. (Spoiler: almost always, for non-trivial work.)
- Recognize when "multi-agent" is the right tool, and when it is just "expensive single-agent" with a 15× token bill attached.

---

## 1. Concept Synthesis

### Three patterns that look similar and are not

Module 01 catalogued five patterns. Three of them keep getting confused for each other in practice, because they all involve "more than one LLM call." This module's first job is to make the cut between them sharp.

**Parallelization** runs subtasks *concurrently* with predetermined structure. The decomposition is yours. Two flavors:
- *Sectioning* — split one task into independent parts (e.g., one model screens the user query for safety while another generates the response).
- *Voting* — run the *same* task many times, then aggregate (e.g., three different review prompts on the same code; flag if any votes "vulnerable").

**Orchestrator-workers** also runs LLMs in parallel, but the decomposition is *runtime*. A central LLM looks at the input, decides what subtasks exist, dispatches them to workers, and synthesizes. The article's framing is sharp:

> The key difference from parallelization is its flexibility — subtasks aren't pre-defined, but determined by the orchestrator based on the specific input.

The canonical example is a coding change that touches an unknown set of files. Parallelization can't help — you don't know what to parallelize. The orchestrator reads the request, decides "I need to edit `auth.py`, `session.py`, and `tests/test_auth.py`," and farms each out.

**Evaluator-optimizer** is sequential and iterative on a *single artifact*. One LLM generates, another evaluates against criteria, the first revises, repeat. This works when:

> First, LLM responses can be demonstrably improved when a human articulates their feedback; second, that the LLM can provide such feedback.

Two clean test cases: literary translation (where critic-loop catches nuance the generator missed in one shot), and complex search (where the evaluator decides whether more rounds are needed).

The mental model: *parallelization is for known-shape work, orchestrator-workers is for unknown-shape work, evaluator-optimizer is for known-target work.*

*— Ch 02 (building-effective-agents)*

### The generator-evaluator pattern, deep

The evaluator-optimizer pattern looks small in the original article — a paragraph, a diagram, two examples. Eighteen months later, Prithvi Rajasekaran's *Harness Design for Long-Running Application Development* (Mar 2026) shows it doing the heaviest lifting in the most ambitious agent system Anthropic Labs has shipped publicly. The same pattern, scaled.

The core observation: **agents asked to evaluate their own work are unreliable critics.**

> Agents tend to respond by confidently praising the work — even when, to a human observer, the quality is obviously mediocre.

Self-evaluation is biased toward the work the model has already produced. An *independent* evaluator agent, with its own context window and its own prompt, doesn't share that bias. The harness Rajasekaran describes for full-stack app development uses three roles:

- **Planner** — expands a 1–4 sentence user prompt into a multi-feature product spec. Deliberately *high-level*, to avoid cascading errors from over-specified technical details.
- **Generator** — implements features. Self-evaluates between cycles, but the self-eval is treated as a sanity check, not a quality bar.
- **Evaluator** — uses Playwright to actually drive the running application like a user would. Files concrete bugs against *sprint contracts* — explicit success criteria the evaluator and generator agreed on *before* the generator started.

The "sprint contract" detail is the part that separates this from naive critique loops. The evaluator's job is not "tell me if this is good" — it's "does this meet the criteria you and the generator already agreed on?" Definition of done is negotiated upfront and is itself an artifact in the loop.

The numbers from the retro-game-maker case study make the cost of skipping this pattern visible:

| Setup | Time | Cost | Outcome |
|---|---|---|---|
| Solo agent (no harness) | 20 min | $9 | "Non-functional application with broken entity controls, rigid workflow, and wasted UI space." |
| Generator + evaluator harness | 6 hr | $200 | 16-feature spec, 10 sprints, working game editor with sprite animation, sound, music, AI-assisted generation, and shareable exports. |

The harness costs ~22× more. It also produces a result that exists, vs. a result that doesn't. This is the kind of comparison that makes the "concept > framework" stance from Module 01 concrete: the gain is not from a better framework, it's from a better *composition* of model calls.

Key principle Rajasekaran calls out:

> The evaluator's role scales with difficulty. For tasks beyond baseline model capability, external grading adds substantial value. For tasks within capability, it becomes overhead.

When Opus 4.5 → 4.6 happened, Rajasekaran *removed* the sprint decomposition and moved to a single-pass evaluator on most tasks, because the generator could plan and execute coherently for longer without intermediate gating. **The harness is not a sacred artifact. Stress-test it on every model release.**

*— Ch 21 (harness-design-long-running-apps)*

### Orchestrator-workers in production: the multi-agent research system

If evaluator-optimizer is "make one artifact better through critique," orchestrator-workers is "do many things in parallel that you didn't know you'd need to do." Anthropic's *How We Built Our Multi-Agent Research System* (Jun 2025) is the longest case study Anthropic has published on the pattern.

The lead agent is the orchestrator. On receiving a research query, it:

1. Develops a research plan (and *saves it to memory* — Module 07 will return to this).
2. Spawns subagents in parallel, each with a focused objective, output format, tool budget, and source guidance.
3. Synthesizes their results.
4. May spawn additional rounds based on what came back.

The performance claim:

> Internal evaluations show that a multi-agent system with Claude Opus 4 as the lead agent and Claude Sonnet 4 subagents outperformed single-agent Claude Opus 4 by 90.2% on our internal research eval.

But the same article is unsparing about the cost:

> Multi-agent systems use about 15× more tokens than chats. Token usage by itself explains 80% of the variance in our BrowseComp evaluation.

The implication for cost-conscious teams is severe: **the pattern only pays for itself when the task is valuable enough to justify the token bill.** Most coding tasks, the article notes, don't qualify — they have fewer parallelizable parts and benefit more from a single coherent agent.

Three scaling rules the team learned the hard way (these matter more than they look):

- **Simple fact-finding:** 1 agent, 3–10 tool calls. (Don't spin up subagents for "what's the capital of France?")
- **Direct comparisons:** 2–4 subagents, 10–15 calls each.
- **Complex research:** 10+ subagents with divided responsibilities.

Embedding these *in the orchestrator's prompt* prevented both underinvestment ("it gave up too early") and resource waste ("it spent $40 to look something up"). The article calls this "scale effort appropriately."

*— Ch 06 (multi-agent-research-system)*

### Two failure modes that show up only at this layer

When you compose patterns, two specific failure modes appear that don't happen with single-LLM workflows. Both are worth knowing now, before they bite you.

**Failure mode 1: Stateful error compounding.** The multi-agent research article puts it bluntly: agents "are stateful and errors compound." A subagent that misunderstands its task at turn 3 will keep building on that misunderstanding through turn 30. The fix isn't smarter models — it's *durable execution* (resumable from checkpoints, not from scratch on failure) and graceful degradation (when a tool fails, the agent adapts rather than aborts).

This is also why the multi-agent team adopted *rainbow deployments* — gradually shifting traffic between versions, since highly stateful systems can't tolerate a hard cutover. We'll revisit this in Module 14.

**Failure mode 2: Synchronous bottlenecks at the coordinator.** In the current research system, the lead agent waits synchronously for all subagents before continuing. This simplifies coordination at the cost of throughput. The article flags asynchronous execution as the obvious next step, with the obvious caveat:

> Asynchronous execution introduces challenges in result coordination, state consistency, and error propagation throughout the system.

Translation: "we know the limitation, the fix is hard." If you're designing your own orchestrator, decide upfront whether you can tolerate sync-blocking or whether you need the complexity of async coordination.

*— Ch 06 (multi-agent-research-system)*

### When orchestrator-workers ≠ multi-agent

A subtle point worth pinning down: *not every multi-LLM system is "multi-agent."* The terminology gets sloppy.

- **Multi-LLM workflow:** several LLM calls composed by code (chaining, routing, parallelization). The composition is yours.
- **Orchestrator-workers:** an LLM dynamically composes other LLMs. The composition is the orchestrator's, but the workers usually don't loop — they run once and return.
- **Multi-agent:** at least one of the participants has its own agent loop (plan → act → observe → repeat) inside the larger system. The research system is multi-agent because subagents themselves call tools in loops.

You will see the term "multi-agent" used for all three. Be precise in your own work. The token bill, debug surface, and right-pattern decisions depend on which one you actually have.

*— Ch 02 + Ch 06 + Ch 21*

---

## 2. Key Questions (the depth check)

<details>
<summary><b>Q1.</b> A team has a code-review pipeline: take a PR, run it through three reviewers (security, performance, style), aggregate findings. Which pattern, and why is it not the others?</summary>

**Parallelization (sectioning).** The decomposition is fixed and known *before* you see the PR. It's not orchestrator-workers because no LLM is deciding what reviewers to spawn. It's not evaluator-optimizer because no one is iterating on a single artifact. The token cost is bounded and predictable. Use parallelization when the structure is fixed; pay the orchestrator-workers tax only when the structure is unknown.
</details>

<details>
<summary><b>Q2.</b> Why does the harness-design article insist that *self-evaluation* is biased toward praise, but a *separate evaluator agent* is not?</summary>

A separate evaluator has a different context window and a different prompt; it doesn't see the generator's reasoning, only the generator's output. It can't be "primed" by the work it's evaluating. This isn't about model identity (you can use the same model class); it's about *context isolation*. The evaluator hasn't already convinced itself the work is good. (This argument extends in Module 08 to sub-agents in general — context isolation is a recurring win.)
</details>

<details>
<summary><b>Q3.</b> The multi-agent research article reports 90.2% improvement over single-agent — and a 15× token cost. What does this tell you about when *not* to use multi-agent?</summary>

When the task value per query is less than ~15× the single-agent cost. For most coding tasks (one repository, one bug, mostly serial reasoning), the gain doesn't justify the bill. The pattern fits open-ended research, where parallel breadth dramatically reduces wall-clock time *and* the per-query value is high enough that 15× tokens is fine. The article's own filter: "valuable tasks that involve heavy parallelization, information that exceeds single context windows, and interfacing with numerous complex tools."
</details>

<details>
<summary><b>Q4.</b> Sprint contracts in the harness-design article: why is it important that the evaluator and generator agree on success criteria *before* the generator starts?</summary>

Two reasons. First, it prevents goalpost-moving — without an upfront contract, the evaluator can rationalize any output as "good" or "bad" depending on what it sees. Second, it forces the *task* to be specified at the level the evaluator can check. If you can't write the contract, the task is too vague to delegate. (This mirrors a pattern from eval design we'll meet in Module 11: writing the eval task forces the product spec to become concrete.)
</details>

<details>
<summary><b>Q5.</b> Rajasekaran removed sprint decomposition when moving from Opus 4.5 → 4.6. What's the general principle, and why is it scary if you take it seriously?</summary>

**Every harness component is an assumption about model limitations, and those assumptions go stale as models improve.** Scary because it means there is no "done" state for harness design — you must re-examine your harness on every model release. A harness component that helped at one model generation can become net-negative overhead at the next. Treat harnesses as load-bearing only as long as they are *currently* load-bearing.
</details>

<details>
<summary><b>Q6.</b> What's the difference between "stateful error compounding" and "ordinary cascading bugs in software," and why does the difference matter for design?</summary>

In ordinary software, a bug at step 3 produces wrong output at step 3, and step 4 either crashes (loud, locatable) or proceeds with the wrong input (visible in tests). In agents, a step-3 misunderstanding silently colors every subsequent decision the model makes — there's no exception, no test, just *gradual* drift. The fix is structural: durable execution + checkpointable state, so that you can resume from a known-good moment rather than restart from scratch. Module 09 (Harnesses) and Module 14 (Production) both lean on this.
</details>

---

## 3. Hands-On

**Notebooks (run both, ~30 min total):**
- [`claude-cookbooks/patterns/agents/orchestrator_workers.ipynb`](../claude-cookbooks/patterns/agents/orchestrator_workers.ipynb)
- [`claude-cookbooks/patterns/agents/evaluator_optimizer.ipynb`](../claude-cookbooks/patterns/agents/evaluator_optimizer.ipynb)

**Run as-is.**

The orchestrator-workers notebook walks through *marketing variation generation* — the orchestrator decides how many variations are needed and what angles each should take, then dispatches workers. Pay attention to:
- The orchestrator prompt (it's surprisingly short).
- How the worker outputs are aggregated (just concatenated; no clever merging).
- The fact that, with this pattern, *the orchestrator can decide at runtime to spawn five workers or fifty*.

The evaluator-optimizer notebook implements an iterative coding loop: generator writes code, evaluator critiques, generator revises, until the evaluator approves or budget runs out. Pay attention to:
- The evaluator's prompt vs. the generator's prompt (they should look genuinely different — different role, different criteria).
- The exit condition. What stops the loop?
- What happens on the *third* iteration vs. the first.

**One small modification (≈15 min): collapse the evaluator into the generator.**

Modify `evaluator_optimizer.ipynb` so the generator prompt now ends with: "After you produce the code, critique it and revise it. Do this twice." Run it on the same input. Compare the output quality to the original two-agent loop on the same task.

You will likely find that the self-critique version is more confident and worse — the model praises its own work, fixes superficial issues, and misses the kinds of structural problems the independent evaluator catches. This is the "self-evaluation bias" claim from the harness-design article, made tangible. Save both transcripts.

**What to record in your notes.**

- One sentence: in the orchestrator-workers notebook, who decides how many workers to spawn?
- One sentence: in evaluator-optimizer, what is the exit condition?
- The single most surprising difference between "evaluator agent" and "self-critique" outputs you saw in your modification.

---

## 4. Reflection

1. **Map a task you actually have to one of these patterns.** Pick a task — work, side project, doesn't matter. Decide which pattern fits. Write down *why the other two are worse fits*. If you can't write that part down, you don't yet understand the pattern; go back to the article.

2. **Push back on the harness-design philosophy.** Rajasekaran's stance is "remove harness components on every model release; only keep what is currently load-bearing." A skeptic could argue this overfits to the latest model and creates churn. What's the right balance — and how would you know your harness has gone stale (vs. is still helpful)? Concretely: what metric tells you?

3. **The 15× token cost.** If you took the multi-agent research system and dropped it into a coding-agent setting, where would the gains come from (if any), and where would you be just lighting money on fire? Be specific about which subtasks would benefit from parallelism and which wouldn't.

---

## 5. Key Takeaways

- **Three different patterns, three different problems.** Parallelization for known-shape work, orchestrator-workers for unknown-shape work, evaluator-optimizer for known-target work. Mixing them up either wastes tokens or fails to capture the pattern's gain.
- **Separate generator from evaluator.** Self-critique is biased; independent evaluators catch what the generator can't see. The same model class works fine — what matters is context isolation.
- **Sprint contracts: agree on "done" before "do."** Upfront success criteria prevent goalpost-shift in evaluator-optimizer loops, and force the task to be concrete enough to delegate.
- **Multi-agent costs ~15× more tokens.** Use it when (a) the task value per query justifies it, *and* (b) the work is genuinely parallel. Most coding work is neither.
- **Harness components encode assumptions about model limits, and those assumptions decay.** Every model release is an excuse to delete harness code. The harness is overhead; the model capability is the bet.
