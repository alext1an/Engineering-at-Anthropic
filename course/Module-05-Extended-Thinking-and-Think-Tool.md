# Module 05: Extended Thinking & The Think Tool

**Time:** ~1.0 hours (≈30 min reading · ≈15 min hands-on · ≈15 min reflection)
**Builds on:** Module 03 (Tool Design Principles), Module 04 (Advanced Tool Use)    **Feeds:** Module 09 (Harness Design), Module 11 (Evaluating Agents)

## Learning Objectives

- Articulate the precise *timing* difference between extended thinking and the think tool — and why timing is the operationally relevant dimension.
- Predict which mode will help on a given task by identifying whether the reasoning challenge is *pre-response planning* or *mid-execution analysis*.
- Implement the think tool correctly, including the optimized prompt that accounts for most of its gains.
- Explain the thinking block preservation requirement and debug the 400 error that results from violating it.

---

## 1. Concept Synthesis

### Two different reasoning problems

"Extended thinking" and "the think tool" both add explicit reasoning steps to Claude's responses. They look similar from a distance — both produce visible reasoning traces — but they address *different* problems in the agent lifecycle.

The core distinction from Anthropic's *Claude's Extended Thinking and the "Think" Tool* (Jan 2025):

> **Extended thinking** is all about what Claude does *before* it starts generating a response.
>
> The think tool allows Claude to add a step to stop and think about whether it has all the information it needs to *move forward* — after receiving tool results.

This timing difference is architecturally significant. Extended thinking is pre-execution reasoning: the model plans, weighs options, and formulates its approach *before* taking any action. The think tool is mid-execution reasoning: the model has called a tool, received a result, and now needs to analyze what it just learned before deciding the next step.

An agent without either capability reasons atomically: each tool call is followed by the next tool call without any visible deliberation about what the intermediate result means. The think tool adds explicit deliberation *between* steps. Extended thinking adds deliberation *before* any steps.

*— Ch 04 (claude-think-tool)*

### When extended thinking is right vs. when the think tool is right

The article provides clean decision criteria:

**Use extended thinking for:**
- Coding, math, and physics without tool requirements
- Straightforward instruction-following tasks
- Simple tool use scenarios with non-sequential, independent calls
- One-shot generation where quality depends on upfront planning

**Use the think tool for:**
- Complex tool chains where each result informs the next call
- Policy-heavy environments with detailed rule sets to apply
- Sequential decisions where mistakes in early steps cascade
- Situations where context discovered mid-execution changes what to do next

The difference reduces to *when the reasoning challenge occurs*. If the challenge is "figure out the right approach to a hard problem," that's pre-response reasoning — extended thinking. If the challenge is "correctly interpret a tool result and decide what to do next given the rules I've been given," that's mid-execution reasoning — think tool.

> The reasoning Claude performs with the 'think' tool is less comprehensive than what can be obtained with extended thinking, and is focused on new information discovered during execution.

This is an honest limitation: the think tool isn't a general-purpose reasoning booster. It specifically helps with the "pause and analyze what I just learned" step.

*— Ch 04 (claude-think-tool)*

### The benchmark results

The think tool was evaluated on τ-Bench (T-Bench), which tests agents on realistic customer service tasks requiring multi-turn tool use and policy compliance. Two domains:

**Airline domain (complex, policy-heavy):**

| Configuration | pass^1 | pass^5 |
|---|---|---|
| Think tool + optimized prompt | 0.570 | 0.340 |
| Extended thinking | 0.412 | 0.160 |
| Think tool alone (no prompt) | 0.404 | 0.100 |
| Baseline | 0.370 | 0.100 |

Think + prompt over baseline: **54% relative gain**. Extended thinking alone: 11% gain. Think tool alone: 9% gain.

**Retail domain (simpler, less policy-heavy):**

| Configuration | pass^1 | pass^5 |
|---|---|---|
| Think tool alone | 0.812 | 0.626 |
| Extended thinking | 0.770 | 0.548 |
| Baseline | 0.783 | 0.583 |

The retail domain pattern is revealing: the baseline is already 0.783, and think tool adds modest gain. The airline domain baseline is 0.370 — much harder — and think + prompt jumps to 0.570. **The gain from the think tool scales with task difficulty.**

The article's key observation: "Simply making the 'think' tool available might improve performance somewhat, but pairing it with optimized prompting yielded dramatically better results for difficult domains."

On SWE-bench: Claude 3.7 Sonnet achieved 0.623 (state-of-the-art) with the think tool. The isolated effect of the think tool improved performance by 1.6% on average (n=30 with tool, n=144 without), with high statistical significance: t(38.89) = 6.71, p < .001, effect size d = 1.47.

*— Ch 04 (claude-think-tool)*

### The think tool definition

The standard implementation is intentionally minimal:

```json
{
  "name": "think",
  "description": "Use the tool to think about something. It will not obtain new information or change the database, but just append the thought to the log. Use it when complex reasoning or some cache memory is needed.",
  "input_schema": {
    "type": "object",
    "properties": {
      "thought": {
        "type": "string",
        "description": "A thought to think about."
      }
    },
    "required": ["thought"]
  }
}
```

Two important properties of this design:
1. **No side effects.** The description explicitly says it "will not obtain new information or change the database." This is a scratchpad, not a tool with real actions. The model can call it freely without concern about costs or side effects.
2. **The implementation does nothing.** When the model calls the think tool, you return an empty acknowledgment. The value comes entirely from what the model writes in the `thought` field — which becomes part of its context for the next step.

*— Ch 04 (claude-think-tool)*

### The optimized prompt that accounts for most of the gain

The 54% improvement in the airline domain doesn't come primarily from having the think tool available — it comes from *telling the model how to use it*. The article provides this guidance framework as a system prompt addition:

```
Before taking any action after receiving tool results, use the think tool as a scratchpad to:
- List specific rules applying to the current request
- Check if all required information is collected
- Verify planned action complies with all policies
- Iterate over tool results for correctness
```

This tells the model *when* to think (after receiving tool results) and *what* to think about (rules, completeness, compliance, correctness). Without this prompt, the model sometimes thinks, sometimes doesn't, and the benefit is inconsistent. With this prompt, the model reliably reasons through the right questions before each action.

Two domain-specific prompt examples from the article (airline tasks):
- Flight cancellation: "Before cancelling, list: (1) which rules apply, (2) whether all required info is collected, (3) whether the cancellation complies with all applicable constraints."
- Ticket booking: "Before booking, verify: (1) available seats on the requested date, (2) passenger eligibility per membership tier, (3) no conflicting reservations in the current booking."

The meta-principle: the think tool is a structured interrupt — a forced pause before consequential actions. The optimized prompt defines what the model should do during that pause.

*— Ch 04 (claude-think-tool)*

### Extended thinking with tools: the mechanics

When using extended thinking with tool calls (as demonstrated in the notebook), several mechanics matter:

**Thinking happens before tool calls, not after.** The model produces a thinking block, then a text block, then a tool_use block — in that order. After the tool result is returned, the model's next turn typically *does not* produce another thinking block. The thinking was front-loaded: "I need to call weather(location='Paris'), so let me verify that's the right tool."

**Thinking blocks require cryptographic signatures.** Each thinking block contains a signature that validates the conversation context. If you strip thinking blocks from conversation history when passing tool results back, the API returns a 400 error:
```
"When thinking is enabled, a final assistant message must start with a thinking block 
(preceding the lastmost set of tool_use and tool_result blocks)."
```
The fix: always include thinking blocks in the conversation history you pass back. The model needs its own prior reasoning to continue coherently.

**Thinking budget sets a token ceiling.** The `budget_tokens` parameter (minimum 1,024) controls how much the model can spend on thinking. For tool use scenarios, start with 2,000 and adjust based on how much reasoning the task requires.

*— Ch 04 (claude-think-tool)*

### When neither is right

The article is explicit about this: "The think tool offers no improvements for non-sequential tool calls (single or parallel calls only), simple instruction following without extensive constraints, and cases where default model behavior is already sufficient."

The practical filter: if you can't point to a specific step in the agent's workflow where explicit mid-execution reasoning would change the decision, the think tool probably won't help. Similarly, extended thinking adds token cost — don't enable it for tasks where the challenge is recall, not reasoning.

Both features come "at the cost of increased prompt length and output tokens." The think tool in particular generates visible reasoning that adds to context and output costs. The benchmark results justify this for difficult sequential tool tasks; they don't justify it for simple queries.

*— Ch 04 (claude-think-tool)*

---

## 2. Key Questions (the depth check)

<details>
<summary><b>Q1.</b> A customer service agent processes returns using 8 policy rules. It keeps approving returns that should be rejected because it misapplies rule 4 after already looking up the order in a database. Which feature helps: extended thinking or think tool? Why?</summary>

**Think tool.** The problem occurs *after* a tool call (database lookup) when the model must apply policy rules to the data it just retrieved. Extended thinking helps with pre-execution planning — the model already has a plan, it's just failing at the "analyze what I just retrieved and apply policy rule 4" step. The think tool with a prompt like "before deciding on return eligibility, list all applicable policy rules and check whether the returned data satisfies each" gives the model a structured pause to do this correctly.
</details>

<details>
<summary><b>Q2.</b> The T-Bench results show think tool alone improves airline accuracy from 0.370 to 0.404, but think + prompt improves it to 0.570. What does this tell you about the mechanism?</summary>

The think tool without the prompt makes the capability available but doesn't reliably activate it. The model sometimes thinks, sometimes doesn't, and when it does, it may think about the wrong things. The optimized prompt specifies *when* to think (after every tool result) and *what* to analyze (rules, completeness, compliance, correctness). Most of the 54% gain comes from this specification, not from the tool itself. The tool creates the possibility; the prompt creates the behavior. This mirrors Module 03's lesson about tool descriptions: the interface (what the model sees) determines the behavior.
</details>

<details>
<summary><b>Q3.</b> Why does the extended thinking notebook show that the model doesn't produce a thinking block after the tool result turn?</summary>

Extended thinking happens *before* the model commits to a course of action. When the model gets a tool result, it already has its plan (from the pre-response thinking block). The tool result updates the data available, but doesn't require replanning — the model continues with the plan it already formed. If the task requires reasoning after each result, that's the think tool's job. Extended thinking front-loads reasoning; the think tool distributes it.
</details>

<details>
<summary><b>Q4.</b> What breaks if you omit thinking blocks from conversation history when passing tool results back with extended thinking enabled?</summary>

The API returns a 400 error. The exact message: "a final assistant message must start with a thinking block (preceding the lastmost set of tool_use and tool_result blocks)." Thinking blocks contain cryptographic signatures that tie them to the conversation context. The model's subsequent turn depends on its prior reasoning being present — it can't maintain coherence without it. Stripping thinking blocks to save tokens breaks the conversation contract.
</details>

<details>
<summary><b>Q5.</b> The think tool's description says it "will not obtain new information or change the database." Why is this explicit disclaimer important?</summary>

It tells the model that calling this tool has no side effects. Without this, the model might hesitate to call it (worried about unintended state changes) or call it sparingly (assuming it costs something). By making explicit that the tool is a free-action scratchpad, the description encourages the model to use it freely whenever reasoning would help, without second-guessing whether the call itself causes harm. This is a case where description engineering directly shapes behavioral frequency.
</details>

<details>
<summary><b>Q6.</b> The retail T-Bench domain shows the think tool slightly *outperforms* the baseline, and extended thinking slightly *underperforms* it. Why might extended thinking hurt on easier tasks?</summary>

On easy tasks, extended thinking adds tokens and potentially "overthinks" — generating complex reasoning for straightforward situations, which can introduce uncertainty or contradictions into decisions that the baseline handles correctly by default. The model may also become overconfident in its reasoning even when that reasoning is slightly off-track. The performance degradation is modest (0.783 baseline → 0.770 with extended thinking), but it illustrates that these features are not universally beneficial — cost goes up, performance can go down for tasks that don't require the added reasoning capacity.
</details>

---

## 3. Hands-On

**Notebook:**
- [`claude-cookbooks/extended_thinking/extended_thinking_with_tool_use.ipynb`](../claude-cookbooks/extended_thinking/extended_thinking_with_tool_use.ipynb)

**Run as-is.**

The notebook has three examples. Pay attention to:
- **Single tool call:** Notice the thinking block appears *before* the tool_use block, not after the tool result. The model thinks about which tool to call, calls it, and then the final response turn has no thinking block.
- **Multiple tool calls:** Watch the iteration pattern — thinking appears at the start, then each subsequent tool call turn produces no new thinking block. The pre-execution reasoning has to carry the model through all subsequent steps.
- **Thinking block preservation:** The demo intentionally breaks the API call by omitting thinking blocks, then shows the correct approach. Read the error message — it's exactly the rule described above.

**One modification (≈15 min): add the think tool and compare.**

After running the notebook as-is, add the think tool definition to one of the multi-tool examples:
```python
think_tool = {
    "name": "think",
    "description": "Use the tool to think about something. It will not obtain new information or change the database, but just append the thought to the log. Use it when complex reasoning or some cache memory is needed.",
    "input_schema": {
        "type": "object",
        "properties": {
            "thought": {"type": "string", "description": "A thought to think about."}
        },
        "required": ["thought"]
    }
}
```

Add it to the tools list alongside weather/news, and change the query to something that requires policy-like reasoning: "I need to plan a trip to London, but only go if the weather is below 70°F and there's technology news worth reading. What should I do?"

Observe whether the model uses the think tool, and if so, when — before or after tool calls?

**What to record in your notes:**
- The exact API error from the thinking block preservation demo (the error message is the constraint made explicit).
- Whether the model in your modification used the think tool, and at which point in the conversation.
- Your hypothesis for why the retail domain showed extended thinking slightly underperforming baseline.

---

## 4. Reflection

1. **The optimized prompt accounts for most of the think tool's gain.** This means the feature is primarily valuable when you know in advance what the model should reason about at each step. What happens when the task is genuinely open-ended — when you *can't* specify what to think about because the right reasoning depends on the data returned? Does the think tool still help, or does it require predetermined structure to be effective?

2. **Thinking blocks add tokens and latency.** In the T-Bench airline domain, the 54% gain from think + prompt probably justifies this cost for a production customer service system. But the SWE-bench gain was only 1.6% (though statistically significant). How do you decide whether the token/latency cost is worth it for your specific task? What metric would you compute?

3. **Extended thinking is front-loaded; the think tool distributes reasoning.** This raises a design question for long agent chains: if you have a 20-step tool chain where the reasoning challenge is spread throughout, does extended thinking (which only fires before step 1) provide enough coverage? Or does every step require mid-execution reasoning via the think tool?

---

## 5. Key Takeaways

- **Timing is the key variable.** Extended thinking = reasoning before execution. Think tool = reasoning during execution, after receiving tool results. Match the feature to where the reasoning challenge occurs.
- **The optimized prompt captures most of the gain.** Making the think tool available adds ~9% on hard tasks; adding a prompt that specifies when and what to think adds ~54%. The tool creates the capability; the prompt creates reliable behavior.
- **Think tool scales with task difficulty.** For easy tasks it adds noise; for hard, policy-heavy, multi-step tool chains it substantially improves accuracy and consistency.
- **Thinking block signatures must be preserved.** Omitting thinking blocks from conversation history when extended thinking is enabled causes a 400 error. The model's coherence across multi-turn tool use depends on its prior reasoning remaining intact.
- **Neither is universally beneficial.** Both add token cost. Apply extended thinking for hard reasoning tasks without mid-execution branching; apply the think tool for sequential tool chains where each result requires careful analysis before the next action.
