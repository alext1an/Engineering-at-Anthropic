# Module 09: Harness Design for Long-Running Agents

**Time:** ~1.5 hours (≈50 min reading · ≈25 min hands-on · ≈15 min reflection)
**Builds on:** Module 02 (Composition Patterns), Module 07 (Compaction, Notes, Memory), Module 08 (Sub-Agents)    **Feeds:** Module 13 (Multi-Agent), Module 14 (Production)

## Learning Objectives

- Design a harness that prevents both failure modes of long-running coding agents: overambition and premature completion.
- Explain the sprint contract mechanism and why it's more effective than post-hoc evaluation.
- Apply the "harness components encode assumptions" principle to decide when to add vs. remove harness complexity.
- Read the DAW implementation cost table and extract what it tells you about evaluator ROI.

---

## 1. Concept Synthesis

### The problem with engineer shifts and context windows

Both the Ch 15 and Ch 21 harness articles open with the same fundamental problem: agents working on multi-session tasks have no memory of previous work. The Ch 15 framing: "Imagine a software project staffed by engineers working in shifts, where each new engineer arrives with no memory of what happened on the previous shift."

This isn't a memory problem (Module 07 covers that) — it's a *task continuity* problem. Even with compaction or notes, a new session agent needs to answer: What state is the codebase in right now? What was working before this session? What am I supposed to build next? Without structured handoffs that answer these questions precisely, agents either:

1. **Overambition:** Try to complete everything at once, exhaust the context window mid-implementation, leave features half-built and undocumented.
2. **Premature completion:** Read partial progress and declare the project done.

Both failure modes were observed with Claude Opus 4.5 running with only high-level prompts and compaction. The harness is the solution to both.

*— Ch 15 (effective-harnesses-for-long-running-agents)*

### The two-part harness for coding agents

**Part 1: The Initializer Agent** runs exactly once at project start. Its job is to create the infrastructure that all subsequent agents depend on:

- **`init.sh`**: A script that starts the development server. Every session begins by running this, so agents never have to figure out how to start the environment.
- **`claude-progress.txt`**: A plain-text log of agent activities. Agents read this at session start to understand what happened previously.
- **`feature_list.json`**: A comprehensive JSON file breaking the user's requirements into granular, testable features. The claude.ai clone example contained over 200 features.
- **Initial git commit**: Establishes a clean baseline the agent can diff against and revert to.

**Part 2: The Coding Agent** runs for every subsequent session. Each session follows a strict startup protocol:
```
1. pwd                           # Confirm working directory
2. Read claude-progress.txt      # Recent activity
3. Read feature_list.json        # Find next incomplete feature
4. git log --oneline -20         # Recent code changes
5. Run init.sh + baseline test   # Confirm environment works
6. Select + implement next feature
7. Verify end-to-end (Puppeteer/browser automation)
8. Commit + update progress file
```

The startup protocol prevents both failure modes. The feature list prevents premature completion — there's an authoritative list of what's done and what isn't. The progress file and git log prevent overambition — the agent can see exactly where the prior session stopped.

Why JSON over Markdown for the feature list? The agents are given instructions like "it is unacceptable to remove or edit tests because this could lead to missing or buggy functionality." JSON format is a behavioral enforcement mechanism: models treat structured data more conservatively than prose, making them less likely to creatively edit or reorganize the content.

*— Ch 15 (effective-harnesses-for-long-running-agents)*

### The generator-evaluator pattern, at scale

Ch 21 describes a three-agent system for full-stack application development:

**Planner:** Expands a 1-4 sentence user prompt into a multi-feature product specification. The planner is deliberately *high-level* on technical implementation to avoid cascading errors from over-specified technical decisions. It weaves AI features into specs and sets ambitious but coherent scope.

**Generator:** Implements features in sprints. Between cycles, it self-evaluates — but the self-evaluation is treated as a sanity check, not a quality bar.

**Evaluator:** Runs the actual application using Playwright, interacting with it as a user would. It tests against specific success criteria negotiated *before implementation began*.

The evaluator's use of Playwright is important. Static code review — reading the code and inferring whether it works — misses the kinds of bugs that only appear at runtime:
- A rectangle fill tool that only placed tiles at start/end points rather than filling regions
- FastAPI route ordering causing frame reordering requests to be parsed as frame IDs (returning 422 errors)
- Audio recording implemented as a stub (button toggles without mic capture)

These bugs don't appear in code inspection. They appear when a user (or Playwright) tries to use the feature end-to-end.

*— Ch 21 (harness-design-long-running-apps)*

### Sprint contracts: negotiate "done" before you build

The mechanism that makes the generator-evaluator loop work is the **sprint contract**: before each sprint, the generator and evaluator negotiate explicit success criteria. The generator proposes what it will build and how success will be verified; the evaluator reviews the proposal; they iterate until they agree.

Why does this matter? Without upfront contracts:
- The evaluator can rationalize any output as acceptable (no pre-agreed bar)
- The generator can declare features done without knowing what "done" means
- Disagreements about completeness arise after implementation, when they're expensive to resolve

With contracts:
- Success criteria are defined before any implementation begins
- The evaluator's job is checking against agreed criteria, not forming subjective judgments
- The generator knows exactly what to build to satisfy the contract

Sprint 3 of the game maker included 27 specific criteria covering the level editor alone. This forced precision at the task specification layer — if you can't write 27 testable criteria for a feature, the feature isn't specified precisely enough to build.

The contracts are communicated through files: the generator writes a proposal file; the evaluator reads and responds; they continue via files until agreement. This file-based communication is itself a form of structured handoff (Module 07 pattern).

*— Ch 21 (harness-design-long-running-apps)*

### The performance evidence

The retro game maker comparison is the starkest data point in the course:

| Setup | Time | Cost | Outcome |
|---|---|---|---|
| Solo agent (no harness) | 20 min | $9 | "Non-functional application with broken entity controls, rigid workflow, and wasted UI space." |
| Generator + evaluator harness | 6 hr | $200 | 16-feature spec, 10 sprints, working game editor with sprite animation, sound, music, AI-assisted generation, and shareable exports. |

The harness costs **22× more** and takes **18× longer**. It also produces something that works vs. something that doesn't. This is the cost/benefit calculation that justifies harness complexity: not that the harness is efficient, but that it produces results the solo agent cannot.

The Digital Audio Workstation (DAW) implementation with the updated Opus 4.6 harness (sprint decomposition removed) shows the evolved picture:

| Phase | Duration | Cost |
|---|---|---|
| Planner | 4.7 min | $0.46 |
| Build Round 1 | 2 hr 7 min | $71.08 |
| QA Round 1 | 8.8 min | $3.24 |
| Build Round 2 | 1 hr 2 min | $36.89 |
| QA Round 2 | 6.8 min | $3.09 |
| Build Round 3 | 10.9 min | $5.88 |
| QA Round 3 | 9.6 min | $4.06 |
| **Total** | **3 hr 50 min** | **$124.70** |

Note the QA/Build ratio: each QA round costs $3-4 and catches genuine gaps that the build rounds missed. QA Round 1 identified that "clips can't be dragged on the timeline, there are no instrument UI panels, and no visual effect editors." These are not minor issues — they're core features. The evaluator's ROI is clear: $3.24 to find bugs that would otherwise ship.

*— Ch 21 (harness-design-long-running-apps)*

### Self-evaluation bias: why the evaluator must be external

> Agents tend to respond by confidently praising the work — even when, to a human observer, the quality is obviously mediocre.

This bias is architectural, not incidental. When a generator evaluates its own output, it evaluates within the context that produced that output — primed by all the reasoning that led to the current state. The context doesn't allow the model to contradict itself without also invalidating its own prior reasoning.

An external evaluator, with an isolated context window, hasn't been primed by the generator's work. It can find what the generator missed precisely because it doesn't share the generator's mental state.

The harness article notes: "separating generation from evaluation proved tractable — tuning an external evaluator to be skeptical works better than making generators self-critical." Skepticism is a prompt parameter you can tune. Self-criticism is a structural property that fights against the model's training.

This is why the generator's self-evaluation in the harness is treated as a sanity check, not a quality bar. It catches obvious errors (syntax, missing files) but not the runtime bugs and UX gaps that the Playwright evaluator finds.

*— Ch 21 (harness-design-long-running-apps)*

### Harness components are temporary assumptions

The most important principle in both harness articles:

> Every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing.

When Opus 4.5 → 4.6, Rajasekaran removed sprint decomposition. Opus 4.6 had improved planning, longer task sustainability, better code review capabilities, and enhanced long-context retrieval. The sprint structure existed because 4.5 needed it; 4.6 could maintain coherence for two-hour builds without it.

Removing it was the right call — but it required testing, not assumption. The harness article's process: remove components one at a time, run the same task, compare results. If removal produces no regression, the component was overhead. If it produces regression, the component is still load-bearing.

The practical implication: **re-evaluate your harness on every significant model release.** A harness component that's helping at one model generation can become net-negative overhead at the next. The harness is not a permanent architecture; it's a set of scaffolding that should be removed as the building becomes self-supporting.

The converse is also true: "as models continue to improve, the better the models get, the more space there is to develop harnesses that can achieve complex tasks beyond what the model can do at baseline." Improved base capability expands what new harness designs can reach.

*— Ch 21 (harness-design-long-running-apps)*

### Context resets vs. compaction

The Ch 21 article makes a specific claim that contradicts a naive reading of Module 07:

> Context resets — clearing the context window entirely and starting a fresh agent with structured handoffs — proved more effective than compaction for Claude Sonnet 4.5.

This is not a contradiction of Module 07; it's a domain-specific finding. Compaction works well when the conversation's *narrative content* matters and should carry forward. For coding agents doing long implementations, the conversation accumulates tool results, intermediate debugging attempts, and wrong paths — content that a good compaction summary might inadvertently preserve.

A full context reset, with carefully structured handoffs (the progress file, feature list, git state), gives the next agent only what it needs: current task, current state, blockers. It doesn't carry forward the noise of the prior session's debugging attempts.

This is also where the Ch 21 "context anxiety" observation connects: Sonnet 4.5 would prematurely wrap work as it approached perceived context limits. Context resets eliminate this — the agent doesn't know how long the prior session was; it just sees the current clean context.

*— Ch 15 (effective-harnesses-for-long-running-agents), Ch 21 (harness-design-long-running-apps)*

---

## 2. Key Questions (the depth check)

<details>
<summary><b>Q1.</b> The harness produces a 22× cost multiplier on the retro game maker. What's the break-even calculation for using the harness vs. solo agent?</summary>

Break-even = (value of working application) vs. (22× cost of solo + value of time). If the working application has $200+ value (or if the solo agent at $9 consistently produces nothing usable), the harness is justified. The decision isn't really about cost — it's about whether the task is in the regime where solo agents reliably produce working results. For complex multi-feature apps, solo agents consistently fail to produce functional work; the harness's 22× cost is compared against 0 value output, not 1× value output.
</details>

<details>
<summary><b>Q2.</b> Sprint contracts require the evaluator and generator to agree on success criteria before implementation. What happens if a feature is too vague to specify testable criteria for?</summary>

The feature is too vague to delegate. This is a feature of the constraint, not a bug: if you can't write testable criteria, the feature specification is incomplete. The sprint contract forces task specification to become concrete enough to verify. A feature like "improve the UI" can't have a sprint contract; "increase the contrast ratio of body text to at least 4.5:1 and add hover states to all interactive elements" can. The contract mechanism catches specification vagueness before it becomes implementation waste.
</details>

<details>
<summary><b>Q3.</b> The article removed sprint decomposition for Opus 4.6 but kept the evaluator. What does this tell you about which harness components are model-capability-dependent vs. which are task-dependent?</summary>

Sprint decomposition was **model-capability-dependent**: it existed because the model couldn't maintain coherence over multi-hour implementations without structured checkpoints. Opus 4.6's improved long-context capability made it unnecessary. The evaluator is **task-dependent**: no matter how capable the model becomes, generating code and verifying that code works end-to-end are different tasks, and self-evaluation bias means an external verifier adds value even for highly capable models. This is the distinction the course needs: some harness components compensate for model limitations (temporary), others compensate for structural task properties (persistent).
</details>

<details>
<summary><b>Q4.</b> The DAW QA Round 1 costs $3.24 and identifies missing interactive depth. What would it cost *not* to have the QA round, and how do you calculate that?</summary>

Without QA Round 1, Build Round 2 would be guessing at what's missing. The generator would have to either (a) run the app itself and self-evaluate (biased, less reliable than Playwright-driven testing) or (b) just add more features without knowing which were incomplete. The cost is: (probability of shipping incomplete features) × (cost of fixing post-ship) + (probability of adding wrong features in Round 2) × (cost of re-implementing). The $3.24 QA buys precise knowledge of exactly what needs fixing — eliminating both wastage risks. Even if QA only catches one significant gap per round, it's almost certainly positive ROI.
</details>

<details>
<summary><b>Q5.</b> Why does context reset work better than compaction for long coding sessions, even though compaction seems more information-preserving?</summary>

Compaction preserves narrative coherence — useful for conversational agents where the dialogue's arc matters. Coding agents accumulate a different mix: debugging attempts that didn't work, wrong paths explored and discarded, tool results from files that are no longer relevant. A compaction summary might "helpfully" preserve these as context, contaminating the next session with knowledge of dead ends. A full reset with structured handoffs (progress file, feature list, git state) gives the next agent *only what it needs*: current task, current state, where to start. The information is more targeted; the context is cleaner.
</details>

<details>
<summary><b>Q6.</b> The harness article says "the better the models get, the more space there is to develop harnesses that can achieve complex tasks beyond what the model can do at baseline." Why would improved models *expand* harness possibilities rather than make harnesses obsolete?</summary>

Harnesses extend model capability — they don't compensate for it. A model that can plan and execute two-hour builds coherently (Opus 4.6) can participate in harnesses that require sustained, complex reasoning within each session. Opus 4.5, which needed sprint decomposition to stay coherent, couldn't support the same harnesses. As models improve, they can handle more complex harness components (sophisticated sprint contracts, multi-evaluator systems, self-directed task decomposition) that earlier models would have failed at. The harness pushes what's achievable; the model's baseline capability sets the floor.
</details>

---

## 3. Hands-On

**Notebook (read as architecture, not SDK tutorial):**
- [`claude-cookbooks/claude_agent_sdk/00_The_one_liner_research_agent.ipynb`](../claude-cookbooks/claude_agent_sdk/00_The_one_liner_research_agent.ipynb)

**Read this notebook for architecture, not for SDK usage.** Pay attention to:
- The harness-level decisions: what goes in the system prompt, what tools are wired up, what the stopping condition is
- How the `query()` function abstracts the agent loop — but note that the underlying pattern is still: plan → act → observe → reflect → repeat
- What the agent does when it can't find a source vs. when it finds contradictory information

**Harness design exercise (≈25 min):**

Instead of running the SDK notebook, design a harness on paper for this task: *"Build an agent that reviews Python code files in a given directory and produces a security audit report."*

For each component below, write 1-3 sentences on what it should do and why:

1. **Initializer:** What does it set up? What files does it create?
2. **Progress tracking:** What does the progress file track? Format (JSON or text)?
3. **Session startup:** What does the agent read first? What baseline does it verify?
4. **Evaluator:** What would Playwright (or an equivalent verifier) test? Or is an external evaluator needed at all for this task?
5. **Sprint contract:** What would a testable success criterion for "security audit of `auth.py`" look like?
6. **Context management:** Context reset or compaction? Why?

Then answer: which components are model-capability-dependent (removable when models improve) and which are task-dependent (always needed for this type of task)?

**What to record in your notes:**
- Your answers to the 6 harness design questions above.
- Which components you'd remove first on the next model release and why.
- One component you'd keep regardless of model capability.

---

## 4. Reflection

1. **Sprint contracts impose specification discipline.** If you can't write testable success criteria for a feature, the feature can't be sprint-contracted. This is a feature of the constraint — but it also means complex, emergent features (where success is "I'll know it when I see it") can't be handed to the harness. Where does the sprint contract mechanism break down, and what do you use instead?

2. **The evaluator uses Playwright to act as a user would.** This works for web applications with browser interfaces. What's the equivalent mechanism for: (a) a CLI tool, (b) a backend API, (c) an agent that writes to a database? What makes each case harder or easier than the web app case?

3. **The article removes sprint decomposition on Opus 4.6 and keeps the evaluator.** You now have a new model release (hypothetical). How do you decide whether the evaluator is still load-bearing? Design a minimal experiment to test this: what would you compare, what metric would you use, and what result would cause you to remove the evaluator?

---

## 5. Key Takeaways

- **Two failure modes, two fixes.** Overambition → feature list with single-feature-at-a-time discipline and explicit progress tracking. Premature completion → authoritative feature list with pass/fail status the agent can't guess around.
- **Sprint contracts force specification precision.** Negotiate "done" before "do." If you can't write testable criteria, the task is too vague to delegate. The contract mechanism catches this before implementation waste occurs.
- **External evaluators with Playwright catch what self-evaluation cannot.** Runtime bugs, UI gaps, and interaction failures are invisible to static code review. A Playwright-driven evaluator catches them for $3-4 per QA round.
- **Harness components are temporary.** Each component encodes an assumption about model limitations. Test assumptions on every model release. Remove components that no longer earn their overhead.
- **Context resets beat compaction for long coding sessions.** Structured handoffs (progress file + feature list + git state) give the next agent exactly what it needs, without the noise of the prior session's debugging traces that compaction might inadvertently preserve.
