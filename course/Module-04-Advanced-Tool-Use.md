# Module 04: Advanced Tool Use

**Time:** ~1.5 hours (≈45 min reading · ≈30 min hands-on · ≈15 min reflection)
**Builds on:** Module 03 (Tool Design Principles)    **Feeds:** Module 06 (Context as Finite Resource), Module 08 (Sub-Agents)

## Learning Objectives

- Explain when Tool Search, Programmatic Tool Calling, and Tool Use Examples each address a *different* bottleneck — and why applying the wrong one leaves the real problem unsolved.
- Predict the token cost of a multi-tool workflow and identify where Programmatic Tool Calling cuts the bill.
- Implement a batch tool to force parallel execution in models that default to sequential tool calling.
- Argue why intermediate result flooding is architecturally different from "large context" problems.

---

## 1. Concept Synthesis

### The three failure modes of tool-heavy agents

Module 03 established tool quality principles for individual tools. This module addresses a different problem: what happens when an agent has *many* tools, or when tool results flood the context window. Anthropic's *Introducing Advanced Tool Use* (Nov 2025) identifies three distinct failure modes in large-scale tool use, each requiring a different fix.

**Failure mode 1: Context overload from tool definitions.** Every tool in an agent's toolkit consumes context window tokens, even before any conversation begins. A five-server MCP setup with 58 tools consumed approximately 55,000 tokens before the first user message — tokens spent on *definitions*, not on reasoning. As agents scale to hundreds or thousands of available tools (typical in enterprise MCP deployments), this overhead can crowd out the actual task context.

**Failure mode 2: Context pollution from intermediate results.** In traditional tool-call loops, every tool result flows through the model's context window, where it accumulates regardless of whether it's still relevant. Processing a complex workflow — fetch team members, then expenses for each, then check budget exceptions — means all 100+ raw expense line items with full metadata (receipt URLs, approval chains, merchant locations) pile up in context. The model must pay attention to all of it. In the notebook for this module, a traditional approach consumed **110,473 tokens** for a budget compliance check; the same task with Programmatic Tool Calling used **15,919 tokens** — an 85.6% reduction.

**Failure mode 3: Schema ambiguity in complex tool invocations.** JSON schema definitions specify *structure* but not *usage patterns*. When a tool has nested objects, optional parameters with non-obvious inclusion rules, or API conventions that aren't derivable from types alone, the model makes parameter errors that schema validation won't catch. The model might pass a flat structure where a nested one is required, or omit a parameter that's technically optional but practically necessary.

*— Ch 14 (advanced-tool-use)*

### Tool Search Tool: solve failure mode 1

The fix for definition overload is deferred loading — don't load all tool definitions upfront, let the model discover them on demand. The **Tool Search Tool** implements this by marking tools with `defer_loading: true`. The model starts with a minimal context: just the search tool itself and any always-needed tools. When it needs a specific capability, it calls the search tool, which expands the relevant definition into context.

The measured results:
- 85% token reduction while maintaining access to the full tool library
- Accuracy on Opus 4 improved from 49% to 74%

The accuracy improvement deserves unpacking. It's not just that less context is cheaper — it's that a model with more available context budget *for the task* reasons better about the task. The 55,000 tokens of tool definitions displaced 55,000 tokens of task-relevant reasoning. Deferred loading reclaims those tokens.

Implementation pattern: mark non-critical tools as deferred; keep always-needed tools (authentication, core actions) loaded. The search tool itself has a small, fixed footprint.

*— Ch 14 (advanced-tool-use)*

### Programmatic Tool Calling: solve failure mode 2

The fix for context pollution is architectural: instead of returning raw tool results to the model, let the model write *code* that calls tools programmatically inside a sandboxed execution environment. The code processes data within the sandbox, and only the final, filtered result reaches the model's context window.

The key insight is that computational work should happen in code, not in the model. Consider the budget compliance workflow:
- Traditional: fetch all expense records → all 100+ records/employee flow through context → model does arithmetic and filtering in its head → token cost: 110,473
- PTC: model writes Python code → code calls tools inside sandbox → code filters to approved expenses, sums by category, compares against budgets → only final result (3 names + amounts) reaches model's context → token cost: 15,919

The reduction is 85.6%. The task result is identical.

Anthropic's *Code Execution with MCP* (Nov 2025) extends this pattern to MCP server access, describing tools organized as a file tree where agents discover and load definitions on demand. The article's most striking example: a Google Drive to Salesforce workflow reduced from **150,000 tokens to 2,000 tokens** (98.7% reduction) by having the agent write code to filter data before it ever touches the model's context.

A secondary benefit: sequential dependency chains collapse. In the traditional approach, checking custom budgets requires knowing who exceeded standard limits, which requires fetching all expenses first — a chain of dependent round-trips, each adding latency. With PTC, the model writes a loop that handles the entire chain in one code block. The model makes fewer API calls and processes the workflow more coherently.

Additional benefits from code execution (Ch 13):
- **Privacy preservation:** intermediate results stay in the sandbox. Sensitive data (PII, credentials) flows between services without entering the model's context.
- **State persistence:** agents can build reusable code functions that evolve over time, accumulating higher-level capabilities.
- **Control flow efficiency:** loops and conditionals in code execute faster than chaining equivalent logic through multiple model turns.

To enable PTC on a tool, add `allowed_callers: ["code_execution_20250825"]` to the tool definition and include a code execution tool in the tools list. Tools without `allowed_callers` default to model-only invocation; tools that need both modes can specify `["direct", "code_execution_20250825"]`.

*— Ch 14 (advanced-tool-use), Ch 13 (code-execution-with-mcp)*

### Tool Use Examples: solve failure mode 3

The fix for schema ambiguity is concrete demonstration: add `input_examples` to tool definitions showing correct usage. Schema says "this parameter takes an object with property X" — examples show *how that object looks in practice*.

The measured result: tool use examples improved accuracy from **72% to 90%** on complex parameter handling in internal testing. That's an 18-point accuracy gain from adding examples to tool definitions.

What examples communicate that schema can't:
- *Nested structure patterns:* how to construct a multi-level object
- *Optional parameter conventions:* which optional fields usually appear together
- *Value formatting:* what format dates, IDs, or codes typically take
- *API idioms:* patterns the API expects even if not technically required

The approach works synergistically with the tool description engineering from Module 03. Description explains *what* the tool does; examples show *how to call it correctly*. The two together leave almost no room for invocation errors.

*— Ch 14 (advanced-tool-use)*

### Parallel tool calling: the default problem and the batch tool fix

The Multi-Agent Research article (Ch 06) identified parallel tool calling as a major performance lever: enabling subagents to call 3+ tools in parallel cut research time by up to 90% for complex queries.

In practice, some Claude model versions default to sequential tool calling — making one tool call, waiting for the result, then making the next — even when the calls are independent and could run simultaneously. The `disable_parallel_tool_use` flag controls this globally, but there's a more targeted pattern when you need to *encourage* parallelism without forcing it: the **batch tool**.

A batch tool is a meta-tool that wraps multiple tool invocations in a single call:

```python
batch_tool = {
    "name": "batch_tool",
    "description": "Invoke multiple other tool calls simultaneously",
    "input_schema": {
        "type": "object",
        "properties": {
            "invocations": {
                "type": "array",
                "description": "The tool calls to invoke",
                "items": {
                    "properties": {
                        "name": {"type": "string"},
                        "arguments": {"type": "string"}
                    },
                    "required": ["name", "arguments"]
                }
            }
        },
        "required": ["invocations"]
    }
}
```

When this tool is present in the tool list, the model uses it to batch independent calls — fetching weather *and* time in a single turn instead of two sequential turns. The orchestration pattern mirrors the research system's architecture: the lead agent fans out work to multiple tools simultaneously rather than waiting for each to complete.

The performance math: if each tool call takes 1 second, 5 independent sequential calls take 5 seconds. 5 parallel calls via batch tool take ~1 second. For agents running dozens of independent tool calls per research session, this transforms wall-clock time.

*— Ch 06 (built-multi-agent-research-system)*

### When to use each feature

These three features address different problems; applying the wrong one to a given bottleneck won't help:

| Problem | Feature | Signal to use it |
|---|---|---|
| Tool definitions crowding out task context | Tool Search Tool | Token count before first message is high; accuracy scales with context budget |
| Raw tool results flooding context mid-task | Programmatic Tool Calling | Multiple round-trips on large datasets; intermediate results are large and mostly irrelevant to final answer |
| Invocation errors on complex schemas | Tool Use Examples | Model makes parameter formatting mistakes despite correct descriptions |
| Slow execution from sequential tool calls | Batch tool / parallel calling | Independent tool calls made in sequence; wall-clock time is a concern |

The features compose: Tool Search reduces upfront overhead, PTC reduces mid-task pollution, examples reduce invocation errors, and batch tool reduces sequential latency. The article recommends starting with whichever bottleneck is largest — don't add all three simultaneously, because the interactions are harder to debug.

*— Ch 14 (advanced-tool-use)*

---

## 2. Key Questions (the depth check)

<details>
<summary><b>Q1.</b> A team's agent uses 80 tools across 5 MCP servers. They're complaining about slow responses and high costs before any user message is processed. Which feature addresses this, and what specifically is it fixing?</summary>

**Tool Search Tool.** The problem is definition loading: 80 tool definitions consume tens of thousands of tokens before the first user message. Tool Search defers definition loading — the model starts with only the search tool and critical always-on tools, then fetches definitions on demand. The fix is in the *setup* phase, not in the task execution phase. Neither PTC (which fixes mid-task context pollution) nor Tool Use Examples (which fixes invocation errors) addresses pre-task token overhead.
</details>

<details>
<summary><b>Q2.</b> Explain why Programmatic Tool Calling produces an 85.6% token reduction in the expense analysis example. What exactly is being kept out of the model's context?</summary>

Traditional tool calling sends every tool result through the model's context: all 100+ expense line items per employee, with full metadata (receipt URLs, approval chains, merchant names, currencies, statuses). The model sees all of it. PTC instead has the model write Python code that runs in a sandbox. The code calls the same tools, but filters the data — only approved expenses, only relevant fields, only employees who exceeded budgets — before returning results to the model. The 85.6% reduction reflects the difference between raw API output volume and the small, processed summary the model actually needs.
</details>

<details>
<summary><b>Q3.</b> Why does Tool Use Examples improve accuracy on complex parameters, when the schema already specifies the correct structure?</summary>

JSON schema specifies *validity* — what values are structurally allowed. It cannot express *usage patterns* — which combinations of optional fields typically appear together, what value formats an API conventionally expects, how nested objects are typically constructed. A schema that says `"type": "object"` with a nested property doesn't show the model the typical nesting depth or field ordering. Examples add the "usually looks like this" knowledge that schema alone can't carry. The 72% → 90% improvement comes from eliminating the gap between "structurally valid" and "practically correct."
</details>

<details>
<summary><b>Q4.</b> The batch tool doesn't actually execute tools in parallel — it's still one sequential API call followed by one parallel execution on your side. Where does the real parallelism happen, and what's the model's role?</summary>

The real parallelism happens in *your* orchestration code: when the model returns a `batch_tool` call containing multiple invocations, you execute those invocations concurrently (using `ThreadPoolExecutor` or `asyncio`). The model's role is to decide which tools to batch in a single call rather than issuing them sequentially. Without the batch tool, the model makes one tool call, waits for the result, then makes the next. With the batch tool, it expresses all independent calls in one response, and you execute them in parallel. The model determines *what* to parallelize; your code determines *how*.
</details>

<details>
<summary><b>Q5.</b> PTC introduces a code execution sandbox. What new risks does this create that didn't exist with standard tool calling?</summary>

Code execution runs arbitrary code in a sandbox — the model writes the code, but the sandbox executes it with real side effects (reading data, calling APIs, writing files if permitted). Risks: (1) the model might write inefficient code that exhausts the sandbox's CPU/memory limits; (2) loops in generated code can make far more tool calls than intended, since there's no per-turn tool call limit; (3) if the sandbox has write access, the model can accumulate side effects in ways the operator didn't anticipate. Mitigations: resource limits, read-only sandboxes where possible, monitoring tool call counts from code execution, and explicit `allowed_callers` restrictions on sensitive tools.
</details>

<details>
<summary><b>Q6.</b> The multi-agent research system cuts research time by 90% using parallel tool calling. Why doesn't the same gain apply to most coding tasks?</summary>

Research tasks are naturally breadth-first: the right answer emerges from exploring many independent sources simultaneously. Parallel tool calling lets subagents explore those sources concurrently. Coding tasks are typically depth-first and sequential: write a function, test it, fix the error, re-test. The next step depends on the result of the current step. There's less parallelizable work because the subtasks are not independent — fixing `auth.py` may require first understanding how `session.py` imports it. The research gain comes from genuine independence of subtasks; coding rarely has that structure.
</details>

---

## 3. Hands-On

**Notebooks (run both, ~30 min total):**
- [`claude-cookbooks/tool_use/programmatic_tool_calling_ptc.ipynb`](../claude-cookbooks/tool_use/programmatic_tool_calling_ptc.ipynb)
- [`claude-cookbooks/tool_use/parallel_tools.ipynb`](../claude-cookbooks/tool_use/parallel_tools.ipynb)

**Run as-is.**

The PTC notebook runs the same budget compliance task with traditional tool calling and with PTC, then prints a side-by-side comparison. Pay attention to:
- The token count difference: 110,473 (traditional) vs 15,919 (PTC) — an 85.6% reduction.
- The `allowed_callers` field on tools — this is what opts a tool into code execution invocation.
- The `caller` field in tool_use blocks — `"code_execution_20250825"` vs `"direct"` tells you whether this tool call came from the model or from the sandbox.
- What information the final response contains vs. what raw data the traditional run accumulates.

The parallel tools notebook shows the batch tool pattern. Pay attention to:
- Without the batch tool: Claude calls `get_weather`, waits, then calls `get_time` — two turns for two independent calls.
- With the batch tool: Claude issues both in one `batch_tool` call — one turn, both results.
- How `process_tool_with_maybe_batch` fans out the batch invocations on your side.

**One modification (≈15 min): count the tool calls.**

In the PTC notebook, add print statements to count how many times each tool is called in the traditional run vs. the PTC run. Which tool gets called most often, and why? You should see that `get_expenses` is called once per employee in both runs — but in the traditional run, all that data passes through the model's context, while in the PTC run, it's filtered in the sandbox. The call count isn't the difference; the *context pollution* is.

**What to record in your notes:**
- Exact token counts for traditional vs. PTC on the expense workflow.
- What the model actually receives as context in the PTC run (the final aggregated result, not all raw expenses).
- The `allowed_callers` values and what each enables.

---

## 4. Reflection

1. **When does PTC not help?** If a tool returns a small, always-relevant result (a single database lookup returning 10 rows of clean data), PTC adds overhead without reducing context. What's the threshold — in terms of result size, filtering ratio, or number of calls — where PTC becomes worthwhile vs. unnecessary complexity?

2. **The 98.7% token reduction (Google Drive → Salesforce example) is striking.** But the article notes it "introduces operational complexity requiring secure sandboxing environments, resource limits and monitoring, infrastructure overhead." Is this tradeoff worth it at your scale? What's the break-even point in terms of token cost savings vs. infrastructure cost?

3. **Tool Use Examples vs. better tool descriptions.** Both improve invocation accuracy. The article implies examples handle cases descriptions can't. Design a test: take one of your tools, write the best description you can, measure accuracy, then add examples, measure again. What types of errors disappear with descriptions but persist, and what types do examples uniquely fix?

---

## 5. Key Takeaways

- **Three distinct bottlenecks need three distinct fixes.** Definition overload → Tool Search. Context pollution → Programmatic Tool Calling. Schema ambiguity → Tool Use Examples. Misdiagnosing the bottleneck means applying the wrong fix.
- **PTC keeps data in code, not in context.** The model writes code that runs in a sandbox; only the final filtered result reaches the model's context. This is the mechanism behind 85%+ token reductions on data-heavy workflows.
- **Parallel execution requires deliberate design.** Sequential tool calls are the default. The batch tool pattern and explicit parallel invocation make independent calls concurrent — and the research system's 90% time reduction shows this matters at scale.
- **Examples communicate what schema cannot.** JSON schema specifies structure; examples communicate convention, idiom, and typical usage. The 72% → 90% accuracy improvement from examples is structurally similar to the tool description engineering gains from Module 03.
- **Layer features starting from your biggest bottleneck.** Adding all three advanced features simultaneously makes failures harder to diagnose. Identify the dominant bottleneck, fix it, measure, then layer.
