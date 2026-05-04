# Demystifying Evals for AI Agents

**Published:** January 9, 2026
**Source:** https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents

## Introduction

Evaluations ("evals") are critical infrastructure for shipping AI agents with confidence. Without them, teams operate reactively—discovering issues only after users encounter them. As the article notes, "Good evaluations help teams ship AI agents more confidently."

The challenge is that agent capabilities that make them useful—autonomy, flexibility, and multi-turn reasoning—simultaneously complicate evaluation. Unlike single-turn language model assessments, agents operate across many steps, calling tools and modifying state in ways that create cascading effects.

## Core Evaluation Framework

**Key Definitions:**

- **Task**: A single test with defined inputs and success criteria
- **Trial**: One attempt at a task (multiple trials account for non-determinism)
- **Grader**: Logic that scores agent performance on specific dimensions
- **Transcript**: Complete record of all interactions, tool calls, and reasoning
- **Outcome**: Final environment state after task completion
- **Evaluation harness**: Infrastructure orchestrating tasks, recording steps, and aggregating results

The article emphasizes that "An evaluation is a test for an AI system: give an AI an input, then apply grading logic to its output to measure success."

## Grader Types and Trade-offs

**Code-based graders** offer speed, objectivity, and reproducibility but struggle with valid variations that don't match exact patterns.

**Model-based graders** provide flexibility and handle subjective tasks effectively, yet introduce non-determinism and require calibration against human judgment.

**Human graders** deliver gold-standard quality but at significant cost in time and resources.

Effective evaluations combine all three types strategically.

## Agent-Specific Evaluation Approaches

### Coding Agents

Deterministic testing naturally fits software evaluation. Benchmarks like SWE-bench Verified verify solutions by running test suites—agents succeed only by fixing failing tests without breaking existing ones. The article notes that "LLMs have progressed from 40% to >80% on this eval in just one year."

Beyond pass/fail outcomes, teams evaluate transcript quality through code analysis and rubric-based assessments of agent behaviors.

### Conversational Agents

These agents present unique challenges because interaction quality itself matters. Success becomes multidimensional: task completion, turn efficiency, and communication tone. The article highlights that "conversational agents present a distinct challenge: the quality of the interaction itself is part of what you're evaluating."

Some evaluations use second LLMs to simulate users in extended conversations, testing reliability through multi-turn scenarios.

### Research Agents

Research quality resists simple verification. What counts as "comprehensive" or "well-sourced" depends on context. Effective approaches layer multiple grader types: groundedness checks verify source support, coverage checks identify missing key facts, and source quality assessments confirm authoritative references.

### Computer Use Agents

These agents interact through screenshots and clicks rather than APIs. Evaluation requires sandboxed or real environments where agents operate actual applications. Success verification combines URL/state checks with backend validation—confirming an order was genuinely placed, not just that a confirmation page appeared.

## Managing Non-Determinism

Two metrics capture agent reliability:

**pass@k**: Probability of at least one success in k attempts. At 50% pass@1, the agent succeeds half the time on first try.

**pass^k**: Probability that all k trials succeed. For an agent with 75% per-trial success, pass^3 ≈ 42%—much stricter for customer-facing systems.

The choice depends on product requirements: "pass@k for tools where one success matters, pass^k for agents where consistency is essential."

## Building Evaluations: A Practical Roadmap

### Starting Early (Steps 0-3)

Begin with 20-50 tasks sourced from actual failures rather than waiting for hundreds. Convert manual checks and user-reported bugs into test cases. Write unambiguous specifications with reference solutions—two domain experts should independently reach identical verdicts.

Build balanced problem sets testing both positive and negative cases. Imbalanced evals create distorted optimization. For instance, search integration requires testing when to search *and* when not to, preventing overtriggering.

### Infrastructure and Design (Steps 4-5)

Construct evaluation harnesses with stable, isolated environments where each trial starts fresh. Shared state between runs introduces spurious failures or inflated performance.

Design graders thoughtfully. The article cautions against checking specific step sequences: "agents regularly find valid approaches that eval designers didn't anticipate. So as not to unnecessarily punish creativity, it's often better to grade what the agent produced, not the path it took."

Incorporate partial credit for multi-component tasks. A support agent correctly identifying problems but failing to process refunds demonstrates meaningful progress over immediate failure.

### Long-term Maintenance (Steps 6-8)

Regularly read transcripts to verify graders work correctly. Failures should reveal clear mistakes, not grader bugs or ambiguous specifications. High saturation rates (near 100% pass rates) indicate evals no longer drive improvement.

The article emphasizes: "We do not take eval scores at face value until someone digs into the details of the eval and reads some transcripts."

Maintain eval suites as living artifacts with clear ownership. Domain experts and product teams should contribute tasks—treating evaluations like unit tests that require routine maintenance.

## Integration with Other Assessment Methods

Automated evaluations represent just one layer in comprehensive agent assessment:

| Method | Strength | Limitation |
|--------|----------|-----------|
| **Automated evals** | Fast iteration without user impact | Requires upfront investment; maintenance burden |
| **Production monitoring** | Reveals real behavior at scale | Reactive; issues reach users first |
| **A/B testing** | Measures actual user outcomes | Slow; requires sufficient traffic |
| **User feedback** | Surfaces unexpected problems | Sparse, skewed toward severe issues |
| **Transcript review** | Builds intuition for failure modes | Time-intensive; doesn't scale |
| **Human studies** | Gold-standard judgments | Expensive; requires trained raters |

Effective teams employ multiple methods: automated evals for rapid iteration, production monitoring for ground truth, periodic human review for calibration.

## Why Evals Matter

Teams without rigorous evaluations struggle in reactive loops, fixing one failure while creating others. The article notes that "Good evaluations help teams ship AI agents more confidently" by making problems visible before production impact.

Evals compound in value: enabling faster model upgrades (weeks of testing reduces to days), establishing baselines for latency and cost, and creating high-bandwidth communication channels between product and research teams.

Perhaps most importantly, evaluations force teams to specify success concretely. As the article states, "Defining eval tasks is one of the best ways to stress-test whether the product requirements are concrete enough to start building."

## Practical Tools and Frameworks

Several frameworks accelerate eval implementation:

- **Harbor**: Containerized agent execution with standardized task formats
- **Braintrust**: Combines offline evaluation with production observability
- **LangSmith/Langfuse**: Tracing and evaluation within LangChain ecosystems
- **Arize Phoenix**: Open-source LLM tracing and debugging

The article advises: "It's often best to quickly pick a framework that fits your workflow, then invest your energy in the evals themselves."

## Key Takeaway

Evaluations transform development from reactive firefighting to proactive improvement. Starting early with realistic tasks, designing thoughtful graders, maintaining infrastructure carefully, and reading transcripts regularly creates the foundation for shipping confident, reliable AI agents.
