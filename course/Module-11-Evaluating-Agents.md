# Module 11: Evaluating Agents

**Time:** ~1.5 hours (≈45 min reading · ≈30 min hands-on · ≈15 min reflection)
**Builds on:** Module 03 (Tool Design), Module 09 (Harness Design)    **Feeds:** Module 12 (Eval Pitfalls)

## Learning Objectives

- Define the five eval components (task, trial, grader, transcript, outcome) and explain why each is distinct from the others.
- Select the right grader type (code-based, model-based, human) for a given task type, and explain the trade-offs.
- Distinguish pass@k from pass^k and predict which metric matters for a given product requirement.
- Apply the 8-step eval roadmap to build a minimal but useful eval suite from scratch.
- Explain why agent evals must grade outcomes rather than paths, and what breaks when they don't.

---

## 1. Concept Synthesis

### Why agent evaluation is harder than model evaluation

A language model evaluation is relatively straightforward: give the model an input, check its output against a rubric. An agent evaluation is fundamentally different — the agent operates across many steps, calling tools, modifying state, and accumulating context in ways that create cascading effects. The same capability that makes agents useful (autonomy, multi-turn reasoning, state manipulation) makes them difficult to evaluate.

The core tension: you want to verify whether the agent accomplished the goal, not whether it took the steps you expected. A software engineering agent that fixes a failing test by rewriting the test body rather than fixing the underlying bug "succeeds" on naive outcome checks but is useless in practice. An agent that finds a valid alternative solution path that your eval design didn't anticipate should pass, not fail. Designing evals that correctly capture this distinction is the central challenge.

*— Ch 16 (demystifying-evals-for-ai-agents)*

### The five components

Every agent eval is built from five elements. Understanding their precise definitions prevents a common class of eval bugs:

**Task:** A single test with defined inputs and success criteria. The task specification is the contract — if two domain experts can't independently reach the same verdict on whether the agent succeeded, the specification is ambiguous. Ambiguous tasks produce noisy graders.

**Trial:** One attempt at a task. Because agents are non-deterministic, a single trial is usually insufficient to characterize performance. Multiple trials over the same task reveal whether the agent succeeds consistently or occasionally.

**Grader:** Logic that scores agent performance on one dimension. A single task can have multiple graders: one checks whether the output is correct, another checks whether it's appropriately concise, a third checks whether it avoided calling a particular tool. Graders compose.

**Transcript:** The complete record of the agent's interaction — all tool calls, tool results, model turns, reasoning, and intermediate state changes. Transcripts are the primary debugging artifact. An eval score without transcript analysis is untrustworthy; the score might be right for the wrong reason.

**Outcome:** The final state of the environment after task completion. This is distinct from what the agent claims. A support agent that says "I've processed your refund" but didn't update the database has the wrong outcome regardless of its claims. Grading outcomes directly — database state, file system changes, API call records — beats grading agent assertions.

The article's framing: "An evaluation is a test for an AI system: give an AI an input, then apply grading logic to its output to measure success." The eval harness is the infrastructure that orchestrates tasks, records transcripts, and aggregates results.

*— Ch 16 (demystifying-evals-for-ai-agents)*

### Three grader types

**Code-based graders** use string matching, binary tests, static analysis, and direct outcome verification. They're fast, deterministic, and objective. The failure mode is brittleness: a correct agent output that doesn't match the expected format fails. The classic case is the building_evals.ipynb animal-legs example — the grader checks `output == golden_answer` exactly. If the agent outputs "2 legs" instead of "2", it fails despite being right. Code-based graders work best when the space of correct outputs is well-defined and narrow.

```python
def grade_completion(output, golden_answer):
    return output == golden_answer
```

**Model-based graders** use a second model (often Claude itself) with a rubric prompt to evaluate the agent's output. They handle the open-ended cases where code graders break: "Did the agent explain its reasoning clearly?" or "Did the customer support response use an appropriate tone?" The failure mode is non-determinism — model graders aren't perfectly consistent, which adds noise to your eval results. They require calibration against human judgments before you can trust their scores. The grader prompt pattern:

```python
def build_grader_prompt(answer, rubric):
    user_content = f"""You will be provided an answer that an assistant gave 
    to a question, and a rubric that instructs you on what makes the answer 
    correct or incorrect.

    <answer>{answer}</answer>
    <rubric>{rubric}</rubric>

    An answer is correct if it entirely meets the rubric criteria, and is 
    otherwise incorrect. First, think through whether the answer meets the 
    rubric criteria..."""
```

**Human graders** are ground truth. Expert review, crowdsourcing, and annotation pipelines produce the most reliable judgments but at significant cost and latency. Use them to: calibrate your model-based graders, establish baselines on tasks you don't yet know how to automate, and handle high-stakes edge cases.

The practical rule: default to code-based, escalate to model-based where the output space is too large, reserve human graders for calibration and high-value spot-checks.

*— Ch 16 (demystifying-evals-for-ai-agents), building_evals.ipynb*

### Handling non-determinism: pass@k vs pass^k

Because agents are non-deterministic, a single trial per task produces a noisy performance estimate. The two metrics that address this have importantly different implications:

**pass@k:** Probability that the agent succeeds in at least one of k attempts. Optimized for tasks where one correct solution is sufficient — you're testing whether the agent *can* do something, not whether it does it reliably. For a research agent producing a draft, one good draft out of three may be fine.

**pass^k:** Probability that *all* k trials succeed. This is the stricter, reliability-oriented metric. If an agent succeeds on each trial with probability p, then pass^k = p^k. An agent with 75% per-trial success rate has pass^3 ≈ 42%. The gap between pass@k and pass^k reveals the agent's consistency profile: high pass@3 but low pass^3 means the agent occasionally nails it but can't be counted on.

The choice reflects the product requirement:
- **pass@k** for tools where one success matters (a code assistant that finds *a* correct solution)
- **pass^k** for agents where consistency is essential (a customer support agent that must handle every ticket correctly)

A common mistake: measuring only pass@1 (single trial per task) and treating it as a reliable performance estimate. At 50% pass@1, you can't distinguish an agent that succeeds 50% of the time from one that succeeds 80% of the time with high variance.

*— Ch 16 (demystifying-evals-for-ai-agents)*

### Grade outcomes, not paths

One of the most consequential design decisions in agent evaluation: whether to grade the path (the sequence of tool calls the agent took) or the outcome (the final state).

Path grading is tempting because it's easy to verify — did the agent call tool A before tool B? Did it generate the right intermediate output? But path grading fails systematically: agents regularly find valid alternative approaches that eval designers didn't anticipate. Grading paths penalizes creativity and produces false negatives on good solutions.

Outcome grading is harder — you need to check the actual state of the environment, not just what the agent said — but it's more robust. For a software engineering agent, outcome grading means running the test suite after the agent's changes, not checking whether the agent used a specific edit pattern. For a data pipeline agent, it means verifying the output data matches the specification, not that the agent used the expected transformation steps.

The article's guidance: "Agents regularly find valid approaches that eval designers didn't anticipate. So as not to unnecessarily punish creativity, it's often better to grade what the agent produced, not the path it took."

SWE-bench Verified implements this correctly: it grades by running the original test suite after the agent's code changes. The agent's solution passes if it makes previously-failing tests pass without breaking previously-passing ones — regardless of which files it edited or what sequence of commands it used.

*— Ch 16 (demystifying-evals-for-ai-agents), Ch 03 (swe-bench-sonnet)*

### SWE-bench as a case study in agent evaluation design

SWE-bench Verified is worth studying not just for its results but for its methodology. It evaluates agents on real GitHub issues from popular Python repositories — not synthetic problems. The task definition is concrete: resolve the issue such that the original test suite passes. The outcome grader is deterministic: run the tests.

The architecture around Claude 3.5 Sonnet that achieved 49% (surpassing the previous SOTA of 45%) was minimal:
- A Bash Tool for executing shell commands
- An Edit Tool with five commands: `view`, `create`, `str_replace`, `insert`, `undo_edit`
- Continued sampling until the model signals completion or hits the 200k token limit
- No complex orchestration, no retrieval pipeline, no specialized planning

```
| Model                    | SWE-bench Verified |
|--------------------------|-------------------|
| Claude 3.5 Sonnet (new)  | 49%               |
| Previous SOTA            | 45%               |
| Claude 3.5 Sonnet (old)  | 33%               |
| Claude 3 Opus            | 22%               |
```

The documented challenges are instructive for eval designers:
- **Duration and cost:** Successful runs often exceeded 100k tokens. Evaluation at scale is expensive.
- **Hidden tests:** Without test visibility, the agent sometimes made surface-level fixes that didn't address the structural issue.
- **Grading complexity:** Environment setup issues and installation patch conflicts occasionally produced false failures — the agent was right, but the eval harness misconfigured the environment.

The last point matters: eval infrastructure bugs produce false results indistinguishable from agent failures. This is why transcript review is mandatory, not optional.

*— Ch 03 (swe-bench-sonnet)*

### Eval types by agent class

Different agent types require different evaluation strategies:

**Coding agents:** Deterministic test suites are the gold standard. Pass/fail is cheap, unambiguous, and outcome-oriented. Layer transcript analysis (code quality, tool efficiency) on top of binary pass/fail for richer signal.

**Conversational agents:** Task completion alone is insufficient — the quality of the interaction is part of what you're evaluating. Multi-dimensional graders: Did the task complete? In how many turns? Was the tone appropriate? Use a second LLM to simulate the user in multi-turn scenarios. Turn limits (resolved in <10 turns?) are useful code-based graders.

**Research agents:** What counts as "comprehensive" is inherently subjective. Layer: groundedness checks (are claims supported by the cited sources?), coverage checks (did the agent include key facts X, Y, Z?), source quality assessment. These all benefit from model-based graders calibrated against expert human judgment.

**Computer use agents:** Require sandboxed environments where state can be verified. Grade backend state changes, not GUI observations — a confirmation page screenshot doesn't prove an order was placed; the order database entry does.

*— Ch 16 (demystifying-evals-for-ai-agents)*

### The 8-step eval roadmap

The article provides a concrete build sequence that matches how evaluation problems actually compound:

**Build the initial dataset (Steps 0-3):**
- Start with 20-50 tasks sourced from actual failures — not synthetic problems invented upfront
- Convert existing manual checks and user-reported bugs into test cases
- Write unambiguous specifications with reference solutions (two domain experts → identical verdict)
- Build balanced positive + negative cases: an eval that only tests when to trigger behavior misses the when-not-to cases

**Design infrastructure (Steps 4-5):**
- Build isolated trial environments — each trial starts from a clean state; shared state between runs inflates or deflates scores unpredictably
- Prefer deterministic graders where possible; escalate to model-based only when necessary
- Design for partial credit on multi-component tasks: an agent that correctly identifies the problem but fails to execute the fix is meaningfully different from one that doesn't understand the problem

**Maintain long-term (Steps 6-8):**
- Read transcripts regularly to verify grader correctness
- Watch for saturation: near-100% pass rates mean the eval is no longer providing improvement signal
- Assign clear ownership; domain experts should contribute tasks; treat eval suites like production code

The most important habit: "We do not take eval scores at face value until someone digs into the details of the eval and reads some transcripts." A 90% pass rate achieved by a buggy grader that flags correct answers as wrong is worthless.

*— Ch 16 (demystifying-evals-for-ai-agents)*

### Evals in the broader assessment strategy

Automated evals are one layer, not the whole picture:

| Method | Strength | Limitation |
|---|---|---|
| Automated evals | Fast iteration, no user impact | Upfront investment; maintenance burden |
| Production monitoring | Real behavior at scale | Reactive — issues reach users first |
| A/B testing | Measures actual user outcomes | Slow; requires sufficient traffic |
| User feedback | Surfaces unexpected problems | Sparse, skewed to severe issues |
| Transcript review | Builds failure mode intuition | Time-intensive; doesn't scale |
| Human studies | Gold-standard judgments | Expensive; trained raters needed |

The right investment distribution shifts over time: early on, transcript review builds intuition you can't get elsewhere; as the product matures, automated evals take over routine regression testing and production monitoring catches the rest.

Evals also force a useful discipline: "Defining eval tasks is one of the best ways to stress-test whether the product requirements are concrete enough to start building." If you can't write an unambiguous eval task, the product requirement isn't concrete yet.

*— Ch 16 (demystifying-evals-for-ai-agents)*

---

## 2. Key Questions (the depth check)

<details>
<summary><b>Q1.</b> The five eval components are task, trial, grader, transcript, and outcome. Why is "outcome" defined as the final environmental state rather than the agent's final output?</summary>

Agent outputs and environmental outcomes can diverge: the agent can claim success ("I've processed your refund") while failing to produce it (the database was never updated). Grading the agent's output grades the agent's self-assessment, which is corrupted whenever the agent is wrong about its own success. Environmental outcome grading is harder to implement — you need to check the actual state of whatever the agent was operating on — but it's the only ground truth. This is why SWE-bench runs the test suite rather than asking the agent whether it thinks it succeeded.
</details>

<details>
<summary><b>Q2.</b> An agent has a per-trial success rate of 60%. What is its pass@3 and pass^3? What do these numbers tell you about deploying it?</summary>

pass@3 = 1 − (1−0.6)³ = 1 − 0.064 = **93.6%** — extremely likely to succeed in at least one of three attempts. pass^3 = 0.6³ = **21.6%** — less than one in four runs will be completely clean. If you're deploying an agent where users retry freely (search, drafting), pass@3 tells the relevant story. If you're deploying an agent operating on consequential state without human review (automated pipeline, production writes), pass^3 is the relevant metric — and 21.6% is probably not acceptable. The gap between 93.6% and 21.6% reveals that this agent is capable but inconsistent.
</details>

<details>
<summary><b>Q3.</b> A team builds an eval that checks whether the agent called tools in a specific sequence. The agent finds a valid alternative approach and fails the eval. What's wrong with the eval design, and how do you fix it?</summary>

The grader is checking paths rather than outcomes. The valid alternative approach represents the agent working correctly — the eval is producing a false negative. Fix: rewrite the grader to check what the agent produced (the final state) rather than how it got there. For a coding agent, run the tests. For a data pipeline, compare output data to specification. If you can't express the success criterion without referencing a specific path, that's a signal the task specification itself is under-specified — what you actually want to grade is unclear.
</details>

<details>
<summary><b>Q4.</b> The building_evals notebook's code grader fails if the agent outputs "2 legs" instead of "2". What does this reveal about the trade-off between code-based and model-based graders?</summary>

Code-based graders are correct only when the space of valid outputs is well-defined and narrow enough to specify exactly. The moment valid outputs have surface variation (format, phrasing, word choice), code graders produce false negatives on correct outputs. The fix is either to constrain the output format through the prompt (instruct the model to return only a number) or to use a model-based grader that understands semantic equivalence. The building_evals approach constrains via prompt ("Return just the number of legs as an integer and nothing else") — which is the right call for this task. But for any task where output format constraints would be artificial, model-based graders are appropriate.
</details>

<details>
<summary><b>Q5.</b> SWE-bench documented that "environment setup issues and installation patch conflicts occasionally affected accuracy assessments." Why is this a problem specifically for the eval team, not just for the agent team?</summary>

If the eval infrastructure produces false failures (the agent solution was correct, but the environment failed to run it), the eval score is corrupted. The corruption is systematically invisible — from the score alone, you can't tell whether the 51% failure rate means 51% of solutions were wrong or whether some percentage were correct solutions on broken environments. This erodes trust in the eval as a measuring instrument. It's why transcript review is non-negotiable: reading transcripts surfaces cases where the agent said "tests pass" but the harness logged an environment error. Eval infrastructure bugs are as harmful as agent bugs, but they're often attributed to the agent by default.
</details>

<details>
<summary><b>Q6.</b> The roadmap says to start with 20-50 tasks sourced from actual failures rather than synthetic problems. Why does the source of tasks matter, and what do synthetic problems miss?</summary>

Actual failures come from the real distribution of inputs the agent will encounter in production. Synthetic problems tend to reflect what the eval designer thinks will be hard, which is usually not what actually trips the agent up. The 20-50 task minimum from failures is an empirical anchor: you know the agent failed these, which means they're within the tractable task space (hard enough to be challenging, not impossible), and fixing them produces measurable improvement. Synthetic problems often lead to over-specified evals that don't generalize — the agent "learns" to pass them without improving on real failures. As the article puts it: start with the failures you already know about.
</details>

---

## 3. Hands-On

**Notebooks:**
- [`claude-cookbooks/misc/building_evals.ipynb`](../claude-cookbooks/misc/building_evals.ipynb)
- [`claude-cookbooks/tool_evaluation/tool_evaluation.ipynb`](../claude-cookbooks/tool_evaluation/tool_evaluation.ipynb)

**Run building_evals.ipynb as-is.**

Pay attention to:
- **Section: Code-based Grading.** The animal-legs eval uses exact string matching. Notice what the prompt does to make this grader reliable: it constrains the output format ("Return just the number of legs as an integer and nothing else"). The grader is only correct because the prompt enforces the output format.
- **Section: Model-based Grading.** The grader prompt wraps `<answer>` and `<rubric>` in XML tags and asks the model to reason about correctness before returning a verdict. Read the grader prompt carefully — note that it instructs the model to think first, then return a verdict.
- **Section: Human grading.** The eval specifies "golden answers" that are instructions to the human grader, not expected outputs. This is the pattern for open-ended tasks.

**One modification (≈15 min): add partial credit.**

The code-based animal-legs grader is binary: right or wrong. Modify it to add a partial credit case: an answer that states the correct number of legs but includes extra words (e.g., "The animal has 2 legs") gets 0.5 credit instead of 0. The goal is to distinguish "completely wrong" from "right answer, wrong format":

```python
def grade_completion_partial(output, golden_answer):
    if output == golden_answer:
        return 1.0
    elif golden_answer in output:  # correct number present, extra words
        return 0.5
    else:
        return 0.0
```

Re-run the eval with this grader. Does any case now score 0.5? Does this change your assessment of the agent's performance?

**Run tool_evaluation.ipynb as-is.**

This notebook demonstrates evaluating a specific tool (calculator) using an XML evaluation file with multiple tasks. Observe:
- The `EVALUATION_PROMPT` instructs the agent to provide `<summary>`, `<feedback>`, and `<response>` tags — structured output that makes grading parseable.
- The report template aggregates accuracy, average duration, and average tool calls — metrics beyond just correctness.
- Each task has an expected response; grading is code-based (exact match on extracted `<response>` content).

**What to record in your notes:**
- The pass rate for each grader type in building_evals. Which grader was most reliable?
- Your partial credit modification: what percentage of outputs scored 0.5?
- One design decision in tool_evaluation.ipynb's EVALUATION_PROMPT that you'd adopt for your own evals, and why.

---

## 4. Reflection

1. **The article says to start with 20-50 tasks from actual failures.** But early in a project, there are no real users and thus no real failures. How do you bootstrap an eval when you haven't shipped yet? What's the closest proxy to real failures available at design time, and what are its limitations?

2. **pass^k for consistency-critical deployments sounds right in theory, but it implies you need k trials per task, which multiplies eval cost by k.** At k=5 trials over 100 tasks, you're running 500 agent invocations per eval run. At what point does eval cost become a bottleneck on iteration speed? What's the minimum k that gives you enough consistency signal to be useful, and how would you decide?

3. **The eval roadmap and the harness design from Module 09 both depend on clean environment isolation.** But clean environments don't always exist — legacy systems, shared databases, third-party APIs. Pick a realistic scenario where environment isolation is difficult, and design the minimum viable approximation that lets you run evals without full isolation.

---

## 5. Key Takeaways

- **Five components, precise definitions.** Task (test with criteria), trial (one attempt), grader (scoring logic), transcript (full interaction record), outcome (environmental state). Outcome ≠ agent's claims — grade what actually happened, not what the agent said happened.
- **Grade outcomes, not paths.** Agents find valid alternative approaches; path graders produce false negatives on correct solutions. SWE-bench's test suite grading is the model: did the code work, not did the agent use the expected approach.
- **pass@k vs pass^k is a product question.** pass@k measures whether the agent can succeed; pass^k measures whether it reliably does. Use pass^k for consistency-critical deployments. The gap between them reveals variance.
- **Three grader types serve different purposes.** Code-based for narrow output spaces (fast, deterministic, brittle). Model-based for open-ended outputs (flexible, calibration-required). Human for ground truth and calibration. Effective evals combine all three.
- **Transcripts are mandatory.** Eval scores without transcript review are untrustworthy. Grader bugs, environment failures, and false positives are only visible by reading what actually happened.
