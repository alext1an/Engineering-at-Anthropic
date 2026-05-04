# Module 03: Tool Design Principles

**Time:** ~1.5 hours (≈45 min reading · ≈30 min hands-on · ≈15 min reflection)
**Builds on:** Module 01 (Workflows vs Agents), Module 02 (Composition Patterns)    **Feeds:** Module 04 (Advanced Tool Use), Module 09 (Harness Design)

## Learning Objectives

- Explain why "tools are a contract" — and what breaks when the contract is poorly written.
- Apply the five core principles (tool selection, namespacing, meaningful context, token efficiency, description engineering) to critique and improve a real tool definition.
- Choose between `auto`, `tool`, and `any` tool_choice modes — and predict what breaks when you pick the wrong one.
- Argue why evaluation-driven iteration (run evals → analyze transcripts → refine) produces better tools than intuition-driven design.

---

## 1. Concept Synthesis

### Tools as contracts

Module 01 introduced the **augmented LLM** — the building block of both workflows and agents — as a model with tools, retrieval, and memory wired in. Module 03 zooms into the tool layer specifically, because this is where most agent failures actually live.

The core framing from Anthropic's *Writing Effective Tools for Agents* (Sep 2025):

> **Tools are software reflecting a contract between deterministic systems and non-deterministic agents.**

This "contract" framing is precise and important. Deterministic systems — the tools themselves, the APIs they wrap — produce identical outputs for identical inputs. Agents are non-deterministic: given the same tool outputs, they may respond differently depending on context, prior turns, and generation variability. The tool is the seam between these two worlds. When the seam is designed badly, neither side can compensate.

The practical implication: tool quality is not about the tool's internal implementation — it's about *what the agent sees*. An API endpoint with correct logic but a misleading description is a bad tool. A description that returns noise along with signal is a bad tool. The agent's reasoning can only be as good as the information the tool gives it.

*— Ch 08 (writing-tools-for-agents)*

### Build-evaluate-iterate: the only workflow that works

The article describes a three-phase workflow for developing effective tools. The order matters.

**Phase 1: Build a prototype.** Start fast — use Claude Code to draft a tool implementation from documentation. The goal is something runnable, not something correct. Test locally with an MCP server or Claude Desktop extension. Expect this to take an hour, not a week.

**Phase 2: Run evaluations.** This is the phase teams skip and then regret. Evaluation tasks should be *realistic* — grounded in actual workflows, requiring multiple tool calls. The article is specific: complex research tasks should require "potentially dozens" of tool calls. Shallow evaluations (single-turn, simple queries) don't reveal the compounding failure modes that show up in real use.

Collect more than accuracy. Record runtime, total tool calls, token consumption per call, and error patterns. These secondary metrics reveal *why* accuracy is low — is the agent calling the wrong tool? Retrying too many times? Getting confused by ambiguous returns?

**Phase 3: Use Claude to improve the tools.** This is the novel part: after collecting evaluation transcripts, feed them back to Claude with a prompt like "analyze these failures and suggest improvements to the tool definitions." The article shows Claude can suggest improvements *across multiple tools simultaneously* and produce measurably better accuracy on held-out test sets — better than what human engineers produced through intuition alone.

The cycle: run evals → get transcripts → Claude analyzes → refine tool definitions → repeat. This is the same evaluator-optimizer pattern from Module 02, applied to tool design itself.

*— Ch 08 (writing-tools-for-agents)*

### Five principles that determine tool quality

**1. Choose the right tools, not all the tools.**

> More tools don't always lead to better outcomes.

This is the first and most violated principle. Teams building tool sets instinctively wrap every available API endpoint. The result is a tool list 40 functions long that makes the agent spend tokens trying to figure out which of the 40 is relevant to the current task.

The better approach: identify *high-impact workflows* from evaluation tasks and build tools that serve those workflows directly. A single `schedule_event` tool that handles availability-checking and calendar booking is better than three separate list/check/create tools — less context overhead, clearer intent, fewer failure modes. Design for the workflow, not for the API surface.

**2. Namespace consistently.**

Grouping related tools under common prefixes (e.g., `asana_projects_search`, `asana_projects_create`, `asana_users_search`) does two things: it helps the agent recognize which tools belong to which service, and it reduces the cognitive overhead of choosing between tools that might otherwise look like synonyms. For large tool sets, consistent namespacing is the difference between an agent that can navigate the space and one that random-walks through it.

**3. Return meaningful context, not raw data.**

> Return only high-signal information. Replace cryptic identifiers with semantically meaningful and interpretable language.

This principle has a concrete measurable effect: replacing UUID-based identifiers with human-readable names significantly reduces model hallucinations. When the agent sees `user_id: "b2f4a1c9-..."`, it has to interpret that value in context. When it sees `user: "Alex Tian (alex@example.com)"`, it can reason directly. The tool's job is to do that translation before returning, not to pass raw API responses up.

A related technique: build a `response_format` parameter that lets the agent request `"concise"` or `"detailed"` responses depending on what phase of the task it's in. This gives the agent control over its own information density.

**4. Design for token efficiency.**

Every byte the tool returns is a byte the agent must process. Implement pagination, filtering, and truncation with sensible defaults. Return five results by default, not 500. When you must truncate, include *an instructive message* alongside the truncation — something like "returned top 5 of 847 results; use `offset` and `limit` parameters to paginate." This steers the agent toward a better strategy rather than silently capping the data it sees.

Unhelpful truncation: return 5 items, agent doesn't know there are more, agent draws false conclusions.
Helpful truncation: return 5 items plus pagination guidance, agent decides whether to fetch more.

**5. Description engineering is prompt engineering.**

> Small refinements to tool descriptions can yield dramatic improvements.

Tool descriptions are the only place the model learns what each tool is for. Writing them badly — vague verb phrases, ambiguous parameter names, missing edge case guidance — is equivalent to writing a bad system prompt. Except the blast radius is larger: a bad system prompt affects one conversation; a bad tool description affects every invocation of that tool.

The heuristic the article offers: write descriptions as if explaining to a *new team member* who has no access to any other documentation. What does this tool do? When should you call it versus the other tools? What do each of the parameters mean? What will you get back?

From Module 01, a concrete case: the SWE-bench reference implementation improved dramatically by switching from relative to absolute filepaths in a file-editing tool description. The model kept constructing paths relative to the wrong base directory. One line of description changed ("always pass absolute paths, not relative") eliminated an entire class of error. The model wasn't broken; the contract was.

*— Ch 08 (writing-tools-for-agents)*

### Tool choice: who decides whether to call a tool

The three `tool_choice` modes are architectural decisions, not just API flags. Each one changes who holds the steering wheel.

**`auto` (default):** The model decides whether to call any provided tools. This is correct for most conversational agents — you want the model to use judgment about when tool use is warranted. The risk: without a carefully written prompt, the model may be over-eager (calling tools when it could answer from knowledge) or under-eager (answering from stale knowledge when a fresh lookup is warranted). The system prompt carries load here: explicit guidance ("only search the web for queries you can't confidently answer") prevents both failure modes.

**`tool` (forced specific tool):** The model *must* call the named tool. Use this when you need guaranteed structured output — for instance, forcing a sentiment analysis tool call ensures the output schema is always populated, regardless of what the user says. This is the "structured extraction" pattern: you define the tool schema to match your desired output format, force the model to call it, and you get JSON rather than free text. The tool never has to execute; it's a prompting trick.

**`any` (forced some tool):** The model must call one of the provided tools, but chooses which one. Use this when your system operates entirely through tool side effects — an SMS chatbot that can *only* communicate by calling `send_text_to_user`, for instance, should never return a raw text response. `tool_choice: any` enforces that contract.

The key insight: even when forcing tool use, the system prompt still matters. `tool_choice: tool` tells the model *that* it must call the tool; the prompt tells it *how* to call it well.

*— Ch 08 (writing-tools-for-agents)*

### What good tool design looks like in practice

The article shows before/after graphs comparing human-written tools against Claude-optimized versions on Slack and Asana integrations. The optimized versions achieve measurably better accuracy on held-out test sets. The improvements were not to the underlying API calls — the same endpoints were called. The improvements were to:

- Description wording (clearer, less ambiguous)
- Return value formatting (UUIDs → readable names)
- Response size defaults (less noise per call)
- Namespacing conventions (consistent prefixes)

None of these required changes to the business logic. They were all changes to *what the agent sees*. This is the practical takeaway: most tool quality problems are presentation problems, not implementation problems. The same tool, described better, returns more useful output, and the agent performs measurably better without any changes to the model.

*— Ch 08 (writing-tools-for-agents)*

---

## 2. Key Questions (the depth check)

<details>
<summary><b>Q1.</b> The article says "more tools don't always lead to better outcomes." Why specifically does a larger tool set hurt agent performance, beyond just "more to choose from"?</summary>

Every tool in the list consumes tokens in the system prompt. An agent with 40 tools sees a much longer tool-definition section than one with 5 tools, leaving less context budget for the task itself. More practically, a large tool set creates ambiguity at selection time: when tools overlap or have similar descriptions, the model makes more selection errors, and the errors compound across multi-turn tasks. Fewer, higher-quality tools that target real workflows consistently outperform large tool catalogs.
</details>

<details>
<summary><b>Q2.</b> Why is "tool description as implicit knowledge documentation" a more precise framing than "tool description as documentation"?</summary>

Regular documentation is read once and cached in memory; a developer can recall it later. Tool descriptions are re-read by the model *on every turn* where tools are available — the model has no persistent memory of having read the description before. More importantly, the description must make explicit things a human engineer would consider "obvious" background knowledge: what service the tool talks to, what format input should be in, what the output represents, edge cases, when *not* to call it. Anything implicit in the human mental model of the tool must be made explicit in the description, or the model can't use it.
</details>

<details>
<summary><b>Q3.</b> A team builds an agent with a file-search tool. The tool returns full file content for every match. What specific failure mode does this create, and how should it be fixed?</summary>

Returning full file content for every match floods the context window with low-signal data. If the agent is looking for one function in a codebase of 200 files, receiving full content for the top 10 matches consumes enormous context budget, potentially pushing earlier conversational context out of the window (the "context rot" problem we'll cover in Module 06). The fix: return file names and relevant snippets (the matching lines ± N lines of context) by default, with a separate `get_file_content(path)` tool for when the agent has narrowed down to the right file.
</details>

<details>
<summary><b>Q4.</b> When should you use `tool_choice: tool` (forced specific tool) vs. better prompt engineering?</summary>

Force a specific tool when you need a *guaranteed output schema* — structured JSON extraction is the canonical case. No amount of prompt engineering reliably produces structured output; the model may still decide to respond in prose. The `tool` mode makes the schema part of the contract, not a suggestion. Use prompt engineering (within `auto` mode) when you want the model to choose intelligently based on the input — forcing a specific tool removes that flexibility. The rule: if the *format* must be deterministic, force the tool. If the *decision* should be smart, stay in `auto`.
</details>

<details>
<summary><b>Q5.</b> The build-evaluate-iterate workflow uses Claude to analyze transcripts and improve tools. Why is this more effective than manual analysis?</summary>

Two reasons. First, Claude analyzes *all* failure transcripts simultaneously, not a sample. It can identify patterns across dozens of failures that a human analyst would miss or dismiss as noise. Second, Claude produces concrete suggested edits to tool definitions, not just diagnoses — the output is actionable rather than observational. Manual analysis tends to catch obvious failures; Claude-driven analysis tends to catch systemic failures in tool description wording or return value formatting that affect many tasks in the same way.
</details>

<details>
<summary><b>Q6.</b> Why does replacing UUID identifiers with readable names reduce model hallucinations?</summary>

When the model sees `user_id: "b2f4a1c9-3f2a..."`, it has an opaque string it must carry forward through reasoning steps. If the model needs to reference this user later, it has to either repeat the UUID verbatim (error-prone in long conversations) or synthesize a placeholder ("the user we just looked up"). Either path is fragile. When the model sees `user: "Alex Tian (alex@example.com)"`, it has a semantically grounded entity it can reference naturally. The model is less likely to confuse users, mix up IDs, or generate plausible-but-wrong identifiers because it can reason about names rather than opaque strings.
</details>

---

## 3. Hands-On

**Notebooks (run both, ~30 min total):**
- [`claude-cookbooks/tool_use/calculator_tool.ipynb`](../claude-cookbooks/tool_use/calculator_tool.ipynb)
- [`claude-cookbooks/tool_use/tool_choice.ipynb`](../claude-cookbooks/tool_use/tool_choice.ipynb)

**Run as-is.**

The calculator notebook demonstrates the minimal structure of a tool definition: `name`, `description`, and `input_schema`. Pay attention to:
- The input_schema format (JSON Schema — type + properties + required).
- The multi-turn request loop: initial call → detect `tool_use` stop reason → execute tool → send result back → get final response. This loop is the mechanical skeleton of all tool-using agents.
- What Claude puts in the `<thinking>` block before calling the tool. Notice it explicitly reasons about which parameter to pass — the description is doing work.

The tool_choice notebook walks through all three modes. Pay attention to:
- `auto`: the system prompt does the heavy lifting for calibrating when to call vs. not call. Without it, the model is over-eager.
- `tool` (forced): the sentiment analysis example shows how to use forced tool calling for guaranteed structured output — the tool never executes, but the model produces the schema.
- `any`: the SMS chatbot shows what happens when the system's only communication channel is tool side effects.

**One modification (≈15 min): break the tool description.**

In `calculator_tool.ipynb`, change the tool description for the `expression` parameter from:
```
"The mathematical expression to evaluate (e.g., '2 + 3 * 4')."
```
to something vague:
```
"Input for calculation."
```

Run the same test queries. Compare what appears in the `<thinking>` block — you should see the model less certain about how to format the expression, and potentially see more formatting errors (commas in numbers not stripped, expression not simplified before passing). This is the description-engineering principle made tangible: the example in the original description (`e.g., '2 + 3 * 4'`) is doing real work.

**What to record in your notes:**
- The three-step call structure for tool use (what `stop_reason` values indicate each step).
- One concrete difference you observed in the `<thinking>` blocks between good description vs. vague description.
- The `tool_choice` mode you'd use for: (a) a chatbot that must always respond via SMS, (b) an agent that should use web search only when needed, (c) a pipeline that needs structured JSON output every time.

---

## 4. Reflection

1. **Applying the contract framing to a real tool you use.** Think of an agent or chatbot you've built or used that has tool access. Pick one tool and audit it against the five principles: Is the description written for a "new team member"? Does it return meaningful context or raw API data? Is the token footprint justified? What would you change first, and why?

2. **The build-evaluate-iterate cycle requires evaluation tasks.** The article says evaluation tasks should be "realistic workflows requiring multiple tool calls." In practice, generating realistic eval tasks is often harder than building the tools themselves. What makes a good eval task for your domain? What's the minimum number of tool calls that reveals real failure modes vs. trivially simple calls that pass without testing anything?

3. **Push back on the "fewer, better tools" principle.** There are domains where exposing many tools is arguably correct — a large enterprise platform may have 50 genuinely distinct operations that can't be consolidated. Is the "fewer tools" principle always right, or is it a rule-of-thumb that breaks at scale? What's the underlying variable it's a proxy for, and can that variable be managed with a large tool set?

---

## 5. Key Takeaways

- **Tools are contracts.** The tool's job is not just to execute correctly but to give the agent *interpretable* information. A correct tool with a bad description is a bad tool.
- **Description engineering is prompt engineering.** Small wording changes in descriptions produce measurable accuracy changes. Treat description quality as a first-class engineering concern, not documentation.
- **Evaluate before you optimize.** Run realistic multi-step evals, collect transcripts, then use Claude to analyze failures. Intuition-driven tool design consistently underperforms evaluation-driven iteration.
- **Return signal, not noise.** UUID → readable name, full file → relevant snippet, 500 results → 5 + pagination instructions. Every token the tool returns is a token the agent must spend attention on.
- **`tool_choice` mode is an architectural decision.** `auto` for intelligent decision-making, `tool` for guaranteed structured output, `any` for tool-only communication channels. Match the mode to the contract you need.
