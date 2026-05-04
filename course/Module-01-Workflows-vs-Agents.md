# Module 01: Workflows vs Agents

**Time:** ~1.5 hours (≈45 min reading · ≈30 min hands-on · ≈15 min reflection)
**Builds on:** —    **Feeds:** Module 02 (Composition Patterns), Module 09 (Harness Design)

## Learning Objectives

- Argue, in concrete terms, when a *workflow* beats an *agent* and vice versa — not "what each is" but the cost/benefit calculus that decides between them.
- Identify the load-bearing assumption in any agent system: *can the model assess its own progress from environmental feedback?* If no, you don't have an agent — you have a fragile workflow that pretends to be one.
- Resist the framework-first reflex: predict, before reading any SDK, when "direct API calls" will outperform a framework abstraction.

---

## 1. Concept Synthesis

### The category that matters: workflows vs. agents

Erik Schluntz and Barry Zhang's *Building Effective Agents* (Dec 2024) opens with a definitional cut that the rest of this course depends on. Both are species of *agentic systems* — LLMs interacting with the world via tools — but they differ in who holds the steering wheel:

> **Workflows** are systems where LLMs and tools are orchestrated through predefined code paths.
>
> **Agents** are systems where LLMs dynamically direct their own processes and tool usage, maintaining control over how they accomplish tasks.

Read those again. The difference is not "how smart is the model" or "how many tools are wired up." It is *who decides the control flow*. In a workflow, you (the engineer) decide; in an agent, the model decides. Everything else — multi-turn loops, tool calling, parallelism — can appear in either.

The article's first practical claim is unsentimental: across "dozens of teams building LLM agents across industries," the patterns that worked weren't elaborate frameworks, they were "simple, composable patterns." Compositionality matters more than novelty.

*— Ch 02 (building-effective-agents)*

### When *not* to build an agent

The post is unusually direct about this:

> When building applications with LLMs, we recommend finding the simplest solution possible, and only increasing complexity when needed. This might mean not building agentic systems at all. Agentic systems often trade latency and cost for better task performance, and you should consider when this tradeoff makes sense.

Three escalation rungs sit underneath any agent decision:

1. **A single optimized LLM call**, possibly with retrieval and in-context examples. Many production systems are this and stop here.
2. **A workflow** — predictable pipeline of LLM calls — when the task decomposes cleanly and you want predictability and traceability.
3. **An agent** — when the task is open-ended, the number of steps is unknowable, and rigid pipelines would be either brittle (paths you didn't anticipate) or wastefully exhaustive (paths that don't apply).

The decision rule: *agents earn their keep only when flexibility and model-driven decision-making are essential at scale.* For "many" applications, the authors note, optimizing a single LLM call is enough.

The same logic applies to frameworks. The article lists Anthropic's Claude Agent SDK, AWS Strands Agents, Rivet, and Vellum, and then warns:

> These frameworks make it easy to get started by simplifying standard low-level tasks like calling LLMs, defining and parsing tools, and chaining calls together. However, they often create extra layers of abstraction that can obscure the underlying prompts and responses, making them harder to debug. They can also make it tempting to add complexity when a simpler setup would suffice.

Their recommendation: "start by using LLM APIs directly: many patterns can be implemented in a few lines of code." If you adopt a framework, "make sure you understand the underlying code." This will be a recurring theme of the course.

*— Ch 02 (building-effective-agents)*

### The augmented LLM is the building block

Both workflows and agents are built from a single primitive: the **augmented LLM** — an LLM that can call tools, retrieve information, and (sometimes) write to memory. Modern Claude models "actively use these capabilities — generating their own search queries, selecting appropriate tools, and determining what information to retain."

Two implementation principles for this primitive:

1. *Tailor capabilities to your specific use case.* You don't need every tool wired up; minimum viable tool set wins.
2. *Ensure a clean, well-documented interface.* Tools must do what their docs say. (We'll spend Module 03 entirely on this.)

The article points to Anthropic's MCP (Model Context Protocol) for integrating with tools via "a simple client implementation." That's the protocol layer the rest of the field is converging on; we'll meet it again in Module 04 and Module 13.

*— Ch 02 (building-effective-agents)*

### Five workflow patterns (the concept-only summary; depth in Module 02)

The article catalogs five patterns. Module 02 goes deep on the most interesting three; here we just need names and one-line intents to anchor the taxonomy.

| Pattern | One-line intent | Best when |
|---|---|---|
| **Prompt chaining** | Sequence of LLM calls, each operating on the prior output, with optional gates between. | Task decomposes into clean fixed steps. |
| **Routing** | Classify input, dispatch to a specialized handler. | Distinct categories benefit from separate prompts/models (often a cheap model for easy cases, a strong one for hard). |
| **Parallelization** | Run subtasks concurrently, then aggregate. Two flavors: *sectioning* (independent parts of one task) and *voting* (multiple attempts at the same task). | The task either has independent parts, or benefits from multiple perspectives or guardrails. |
| **Orchestrator-workers** | Central LLM dynamically decomposes the task, delegates to worker LLMs, synthesizes results. | The subtasks aren't predictable in advance (e.g., complex code edits across unknown files). |
| **Evaluator-optimizer** | One LLM produces, another critiques; iterate. | You can articulate clear evaluation criteria *and* iterative refinement measurably improves the output. |

*— Ch 02 (building-effective-agents)*

### What an agent actually is, mechanically

After workflows, the article defines an agent in operational terms — what it *does*, not what it *is*:

> Agents can handle sophisticated tasks, but their implementation is often straightforward. They are typically just LLMs using tools based on environmental feedback in a loop. It is therefore crucial to design toolsets and their documentation clearly and thoughtfully.

A canonical agent loop, reduced:

```
plan → act (call a tool) → observe (read tool result) → reflect → repeat
                                      ↑                              │
                                      └──────────────────────────────┘
                                          until "done" or escalation
```

The two non-obvious requirements:

1. **Ground truth from the environment.** The agent must be able to verify its progress against something real — test results, screenshots, search results, compiler output. Without this, every loop iteration just compounds the model's prior errors. *This is the load-bearing assumption.*
2. **Stopping conditions and human checkpoints.** "Stopping conditions (such as a maximum number of iterations) to maintain control." Agents that don't stop are agents that burn money producing slop.

Two real-world examples the article highlights:

- **Coding agents** — solving GitHub issues, where "results can be objectively measured through automated tests." Anthropic's SWE-bench reference implementation (Module 03 will inspect this) hits state-of-the-art with two tools and a minimal scaffold.
- **Computer use** — Claude operating a desktop via screenshots and keyboard/mouse actions. Same loop; a much messier observation channel.

Agents shine on "open-ended problems where it's difficult or impossible to predict the required number of steps" — exactly the opposite of where workflows shine. The cost is autonomy: "agents' autonomy makes them ideal for scaling tasks in trusted environments. The autonomous nature of agents means higher costs, and the potential for compounding errors. We recommend extensive testing in sandboxed environments, along with the appropriate guardrails."

*— Ch 02 (building-effective-agents)*

### Three principles that survive every iteration

The article distills three principles its authors keep returning to:

1. **Simplicity.** Maintain agent design simplicity. Treat every additional layer as a tax on debugging.
2. **Transparency.** Show the planning steps explicitly. (You will thank yourself when something goes wrong.)
3. **Agent-Computer Interface (ACI) craftsmanship.** Document tools as carefully as you document prompts. Test edge cases. The tools *are* the interface; the model is only as good as the surface area you give it.

The framework guidance follows from these: "If a framework makes it harder to see what the LLM is actually doing, it's working against you."

*— Ch 02 (building-effective-agents)*

---

## 2. Key Questions (the depth check)

<details>
<summary><b>Q1.</b> A team has a customer-support task that "agents handle dozens of distinct ticket types." Should this be a workflow or an agent?</summary>

**Workflow — specifically *routing*.** Distinct categories with predictable handling per category is the canonical fit for routing. Agents earn their keep when *the steps themselves* are unknowable, not when *which lane* is unknowable. (Routing + a per-lane prompt is also dramatically cheaper to debug.)
</details>

<details>
<summary><b>Q2.</b> What is the single property that distinguishes "an agent loop" from "a chained workflow that runs many times"?</summary>

**The model decides whether to continue.** In a workflow the loop condition is hardcoded ("repeat 3 times" or "until validator passes"). In an agent the model itself, on each turn, evaluates environmental feedback and decides what to do next, including whether to stop. If the looping condition is yours, it's a workflow.
</details>

<details>
<summary><b>Q3.</b> The article warns that frameworks "obscure the underlying prompts and responses." Why is this specifically dangerous for agent systems vs. ordinary applications?</summary>

Because agent failures are usually *prompt* failures, *tool description* failures, or *tool output* failures — exactly the layer the framework hides. A regular bug shows up in a stack trace; an agent bug shows up as the model "acting weird," and the only way to debug it is to read what the model actually saw. If you can't see the system prompt, the tool definitions, and the tool results that came back, you can't fix it.
</details>

<details>
<summary><b>Q4.</b> Give a concrete example where evaluator-optimizer is a worse choice than orchestrator-workers, and explain why.</summary>

A multi-file refactor across 30 files. Evaluator-optimizer optimizes a single artifact iteratively; refactor work is *parallel and decomposable*, not iterative — the right shape is "decompose into per-file tasks, dispatch, synthesize," which is orchestrator-workers. Use evaluator-optimizer when the *same artifact* needs to get better; use orchestrator-workers when *different artifacts* need to get done.
</details>

<details>
<summary><b>Q5.</b> Why is "stopping conditions" listed as a *core* design element rather than a defensive nice-to-have?</summary>

Because without them an agent is a money-burning machine. Without stopping conditions, an agent that's stuck in a confused state — say, repeatedly trying the same failing fix — has no exit. Every iteration costs tokens. The article phrases it as "maintain control"; in practice it's also "maintain a budget."
</details>

<details>
<summary><b>Q6.</b> The article claims the highest-leverage place to spend prompt-engineering effort, beyond the system prompt, is on tool descriptions. Why?</summary>

Because tool descriptions are the only place the model learns *what your tools are for*. A wrong system prompt costs you one wrong answer; a wrong tool description costs you wrong answers across every future call to that tool, and worse, the model picks the wrong tool entirely. Module 03 makes this concrete with the SWE-bench example: switching from relative to absolute filepaths (a *one-line tool-description change*) eliminated a whole class of agent error.
</details>

---

## 3. Hands-On

**Notebook:** [`claude-cookbooks/patterns/agents/basic_workflows.ipynb`](../claude-cookbooks/patterns/agents/basic_workflows.ipynb)

This is the official reference implementation for the workflow patterns described above. It contains tiny, dependency-light implementations of *prompt chaining*, *parallelization*, and *routing* using direct API calls — no framework. Reading the source is the point.

**Run as-is (≈10 min).**

1. Walk through the `chain()` function (cell 2). It is six lines of Python. This is the entire prompt-chaining "framework."
2. Run Example 1 (data extraction → formatted table). Note the gate between steps: each prompt operates on the *previous step's output*, not on the original input.
3. Run Example 2 (parallel stakeholder analysis). Notice that parallelization here is `ThreadPoolExecutor`, not anything Claude-specific. The model is unaware it's being called concurrently.
4. Run Example 3 (routing). Notice that the router is itself an LLM call that returns a label, and the label is a Python dictionary key. The whole "routing system" is a one-line dispatch.

**One small modification (≈15 min).**

Add a fourth example to the notebook that demonstrates the *failure mode* of using the wrong pattern. Take a task that is genuinely *agent-shaped* (e.g., "given this support ticket, decide whether to refund, ask a clarifying question, or escalate, and when you've decided, take the action by calling one of three tools") and try to express it as a chain. You should find that the chain has to encode every branch as a hardcoded prompt step, and the moment the user says something off-script, the chain has nowhere to go. This is *exactly* the situation where you need an agent — model-decided control flow.

Don't make it work. The point is to feel the constraint.

**What to record in your notes.**

- The line count of `chain()` and `route()` versus what you'd expect a "workflow framework" to need.
- The exact failure mode when you tried to chain an agent-shaped task — write down the symptom in one sentence.
- Your guess for *why* the article keeps insisting on direct API calls. (Compare to your guess after Module 02.)

---

## 4. Reflection

1. **Push back on the framework warning.** The article says frameworks "obscure the underlying prompts and responses." Is this *always* bad? Imagine a team of 30 engineers shipping agent code; some level of abstraction may be necessary just to keep them on the same page. What's the principled criterion for when a framework is helping vs. hurting? (Hint: think about what the framework lets you observe.)

2. **Apply to your own system.** Pick something you're working on or have worked on recently. Where is it on the workflow ↔ agent spectrum? What single change would push it one rung simpler? What single change would push it one rung more agentic? Which direction is more honest about your actual problem?

3. **The "ground truth from environment" requirement is doing a lot of work.** What happens when the environment doesn't give clean ground truth — say, a writing task where there's no test suite? Are agents simply the wrong tool for those tasks, or is there a way to *manufacture* environmental feedback? (Module 02's evaluator-optimizer pattern and Module 09's harness design are the two main answers; preview them now.)

---

## 5. Key Takeaways

- **The cut is who decides control flow.** Workflow = code decides; agent = model decides. Everything else is implementation detail.
- **Start at the simplest rung.** Single LLM call → workflow → agent. Each step up trades latency and cost for flexibility. Make sure you're paying the price for a real benefit.
- **Frameworks are a debugging tax.** Use them after you understand the layer beneath. Direct API calls expose the prompts and responses that are the *actual* thing failing when things fail.
- **Agents need ground truth.** The agent loop is plan → act → observe → reflect. Without honest observations from the environment (tests, search, compiler, screenshots), the loop just compounds error.
- **Tool craftsmanship is non-negotiable.** The agent-computer interface deserves the same prompt-engineering effort as the system prompt. Module 03 will spend 1.5 hours on exactly this.
