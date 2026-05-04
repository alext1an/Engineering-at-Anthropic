# Agent Engineering in Depth: A 20-Hour Course

## Course Intent

This course goes deep on the engineering decisions behind production AI agent systems. It is built from Anthropic's engineering blog posts and runnable notebooks, weighted heavily toward concepts — the *why* behind architectural choices — with hands-on coding to make the concepts concrete.

The target reader already understands agent patterns at a basic level: you know what an orchestrator is, you've called an LLM API, you understand tool use in principle. What this course adds is the depth that separates knowing a pattern from knowing when to apply it, what breaks it, and how to measure whether it's working. You should finish with opinions — not just familiarity.

---

## What This Course Is NOT

- **Not an SDK tutorial.** The Anthropic Agent SDK and Managed Agents API appear in two modules (09, 13) but only as architectural illustrations. The course is SDK-agnostic.
- **Not for LLM beginners.** This course skips foundational LLM concepts (transformers, tokens, sampling) entirely. It starts at "you understand agent loops."
- **Not framework-specific.** The patterns here apply whether you're using Claude, GPT, Gemini, or a fine-tuned model.
- **Not exhaustive.** Fourteen modules can't cover everything. This course emphasizes the concepts where depth matters most and where Anthropic's engineering experience provides specific, concrete insight.

---

## How to Use This Course

**Suggested pace:** One module per session, 1-1.5 hours each. Don't compress. The reflection questions at the end of each module require sitting with the ideas.

**When to skip ahead:** If you have deep experience with a topic, the Key Questions section (§2 in each module) is a fast check. If you can answer all six questions correctly without reading, skip the module and move on. If you miss two or more, read it.

**When to go slower:** The hands-on sections have a "one modification" task that makes the key concept concrete. If you're skipping the hands-on, you're getting a reading course, not a 20-hour course. The modification tasks are where the concepts stop being abstract.

**Prerequisite order matters.** The "Builds on / Feeds" headers in each module are real dependencies, not suggestions. Module 08 (Sub-Agents) requires Module 06 (Context) — the isolation concept only makes sense after you understand context rot. Don't skip ahead without reading dependencies.

**Part structure:**
- **Part 1: Patterns** (Modules 01-02, ~3h) — Foundational agent architecture
- **Part 2: Tools** (Modules 03-05, ~4h) — Tool design, advanced use, extended thinking
- **Part 3: Context** (Modules 06-08, ~4h) — Context engineering, compaction, sub-agents
- **Part 4: Harness & Skills** (Modules 09-10, ~3h) — Long-running agent infrastructure
- **Part 5: Evaluation** (Modules 11-12, ~3h) — How to measure whether agents work
- **Part 6: Production** (Modules 13-14, ~3h) — Multi-agent systems and production reality

---

## Module Table

| # | Title | Time | What you'll be able to do after |
|---|---|---|---|
| 01 | [Workflows vs Agents](Module-01-Workflows-vs-Agents.md) | 1.5h | Argue when to use a workflow vs. an agent, and predict where each breaks |
| 02 | [Composition Patterns](Module-02-Composition-Patterns.md) | 1.5h | Design parallelization, orchestrator-workers, and evaluator-optimizer systems; predict their failure modes |
| 03 | [Tool Design Principles](Module-03-Tool-Design-Principles.md) | 1.5h | Evaluate a tool definition against five principles and predict where it will fail in the build-evaluate-iterate loop |
| 04 | [Advanced Tool Use](Module-04-Advanced-Tool-Use.md) | 1.5h | Apply Tool Search, Programmatic Tool Calling, and Tool Use Examples to reduce token cost and increase accuracy |
| 05 | [Extended Thinking & The Think Tool](Module-05-Extended-Thinking-and-Think-Tool.md) | 1.0h | Choose between extended thinking and the think tool for a given task type, and correctly preserve thinking blocks |
| 06 | [Context as Finite Resource](Module-06-Context-as-Finite-Resource.md) | 1.5h | Match each of the three context growth types to its correct fix, and explain why Contextual Retrieval beats standard RAG |
| 07 | [Compaction, Notes, Memory](Module-07-Compaction-Notes-Memory.md) | 1.5h | Apply the three context management primitives (compact, clear, memory) correctly, and design a memory format for an agent |
| 08 | [Sub-Agents & Context Isolation](Module-08-Sub-Agents-and-Context-Isolation.md) | 1.0h | Predict when sub-agents hurt vs. help, and specify sub-agent invocations with all four required elements |
| 09 | [Harness Design](Module-09-Harness-Design.md) | 1.5h | Design a sprint contract, write the 7-step startup protocol, and explain why feature lists beat Markdown notes |
| 10 | [Skills, Sandboxing, Permissions](Module-10-Skills-Sandboxing-Permissions.md) | 1.5h | Write a SKILL.md with correct progressive disclosure, and explain how sandboxing + input probe + output classifier compose |
| 11 | [Evaluating Agents](Module-11-Evaluating-Agents.md) | 1.5h | Design a minimal eval suite with correct grader types, and choose between pass@k and pass^k for a given deployment |
| 12 | [Eval Pitfalls](Module-12-Eval-Pitfalls.md) | 1.5h | Identify infrastructure noise, eval awareness, and eval resistance in a given eval design, and apply the 3× threshold rule |
| 13 | [Multi-Agent & Decoupled Architecture](Module-13-Multi-Agent-and-Decoupled-Architecture.md) | 1.5h | Apply the session/harness/sandbox decoupling pattern and explain what `wake(sessionId)` enables |
| 14 | [Production Lessons & Postmortems](Module-14-Production-Lessons-and-Postmortems.md) | 1.5h | Classify production AI failures by category, design a four-level monitoring strategy, and write a postmortem template |

**Total: 20.0 hours**

---

## Time Budget Breakdown

| Part | Modules | Hours | Theme |
|---|---|---|---|
| Patterns | 01–02 | 3h | When agents make sense, how they compose |
| Tools | 03–05 | 4h | What agents can do and how to make it reliable |
| Context | 06–08 | 4h | How to keep agents coherent over time |
| Harness & Skills | 09–10 | 3h | Infrastructure for long-running systems |
| Evaluation | 11–12 | 3h | How to measure and trust agent performance |
| Production | 13–14 | 3h | What happens when agents run at scale |

---

## Source Material

Each module synthesizes 1-3 Anthropic engineering blog posts, read fresh for each module. Blog sources are attributed in each module's Concept Synthesis section as `*— Ch NN (slug)*`. The full source list:

| Blog | Slug | Used in |
|---|---|---|
| Ch 01: Contextual Retrieval | contextual-retrieval | M06 |
| Ch 02: Building Effective Agents | building-effective-agents | M01, M02 |
| Ch 03: SWE-bench Verified | swe-bench-sonnet | M11 |
| Ch 04: The Think Tool | think-tool | M05 |
| Ch 05: Claude Code Best Practices | claude-code-best-practices | M14 |
| Ch 06: Multi-Agent Research System | built-multi-agent-research-system | M02, M08, M13 |
| Ch 07: Desktop Extensions | desktop-extensions | M14 |
| Ch 08: Writing Tools for Agents | writing-tools-for-agents | M03 |
| Ch 09: Postmortem of Three Issues | a-postmortem-of-three-recent-issues | M14 |
| Ch 10: Effective Context Engineering | effective-context-engineering-for-ai-agents | M06, M07, M08 |
| Ch 11: Agent Skills | equipping-agents-for-the-real-world-with-agent-skills | M10 |
| Ch 12: Claude Code Sandboxing | claude-code-sandboxing | M10 |
| Ch 13: Code Execution with MCP | code-execution-with-mcp | M04 |
| Ch 14: Advanced Tool Use | advanced-tool-use | M04 |
| Ch 15: Effective Harnesses | effective-harnesses-for-ai-coding-agents | M07, M09 |
| Ch 16: Demystifying Evals | demystifying-evals-for-ai-agents | M11 |
| Ch 17: AI-Resistant Evals | AI-resistant-technical-evaluations | M12 |
| Ch 18: C Compiler with Parallel Claudes | building-c-compiler | M13 |
| Ch 19: Infrastructure Noise | infrastructure-noise | M12 |
| Ch 20: Eval Awareness | eval-awareness-browsecomp | M12 |
| Ch 21: Harness Design for Long-Running Apps | harness-design | M09 |
| Ch 22: Claude Code Auto Mode | claude-code-auto-mode | M10 |
| Ch 23: Scaling Managed Agents | managed-agents | M13 |
| Ch 24: April 2026 Postmortem | april-23-postmortem | M14 |

Notebooks are drawn from [`claude-cookbooks/`](../claude-cookbooks/) and are listed in each module's Hands-On section with their full relative path.
