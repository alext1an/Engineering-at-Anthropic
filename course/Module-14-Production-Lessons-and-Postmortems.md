# Module 14: Production Lessons & Postmortems

**Time:** ~1.5 hours (≈60 min reading · ≈15 min hands-on · ≈15 min reflection)
**Builds on:** Module 11 (Evaluating Agents), Module 12 (Eval Pitfalls), Module 09 (Harness Design)

## Learning Objectives

- Extract generalizable failure patterns from two production postmortems — what category of defect each represents and what monitoring would have caught it earlier.
- Distinguish evaluation gaps (wrong eval design) from detection gaps (right eval, wrong feedback loop) in production quality failures.
- Apply the explore-plan-code-commit workflow and explain what problems each phase catches.
- Design a minimal production monitoring strategy that complements pre-deployment evals.

---

## 1. Concept Synthesis

### What production failure looks like for AI systems

Two postmortems from Anthropic's production systems — one from September 2025, one from April 2026 — document what actually goes wrong when AI agents operate at scale. These aren't hypothetical failure modes. They're specific bugs that affected users, took time to detect, and prompted structural changes to monitoring and development process.

The failure categories are worth studying because they represent failure modes that appear in any system where model inference depends on infrastructure configuration, where system prompt changes affect behavior, and where user-facing degradation is harder to measure than binary correctness.

*— Ch 09 (a-postmortem-of-three-recent-issues), Ch 24 (april-23-postmortem)*

---

### Postmortem 1: Three Infrastructure Bugs (September 2025)

Between August and early September 2025, three separate bugs degraded Claude's response quality. All three were eventually fixed, but detection took longer than it should have.

**Bug 1: Context Window Routing Error**

A load balancing change on August 29 caused some Sonnet 4 requests to be misrouted to servers configured for the 1M token context window. At peak impact: 16% of Sonnet 4 requests affected. About 30% of Claude Code users experienced at least one degraded response during this period.

The failure mode: a request intended for one server configuration hit a different one, where inference operated under different assumptions. The user experience was degraded output quality — more difficult to trace than a clear error.

**Bug 2: Output Corruption**

A misconfiguration on TPU servers caused token generation errors, producing unexpected characters in outputs — Thai or Chinese text appearing in English responses, syntax errors in code. Affected Opus 4.1, Opus 4, and Sonnet 4 across different timeframes.

The failure mode: hardware misconfiguration producing token-level errors. These are detectable (wrong characters in output are objectively wrong) but inconsistent — not every request fails, and the pattern varies by server.

**Bug 3: Approximate Top-K XLA:TPU Miscompilation**

The most complex failure. A compiler bug in XLA:TPU was triggered when deploying improved token selection code. The root cause: "operations that should have agreed on the highest probability token were running at different precision levels." Mixed precision arithmetic caused the best token to sometimes disappear entirely.

Critically: a December 2024 workaround had inadvertently masked this latent bug. When engineers later removed that workaround (reasonably, since it seemed no longer necessary), they exposed the deeper problem. The bug was latent in the system for months before it became visible.

**Why detection took too long — four structural causes:**

1. **Privacy protections** limited engineers' ability to examine problematic user interactions without explicit feedback. The data to diagnose the bug existed, but access required friction.

2. **Evaluation gaps:** Standard benchmarks didn't capture the degradation users experienced. The evals were measuring the right thing at the wrong granularity, or the wrong thing entirely.

3. **Inconsistent symptoms across platforms** created contradictory reports. When failure is intermittent and platform-dependent, the signal looks like noise.

4. **Over-reliance on noisy evaluations** made it hard to connect symptoms to specific recent changes.

**The structural changes:**
- More sensitive evaluations designed to differentiate working vs. broken implementations
- Continuous quality evaluations running on production systems, not only during deployment
- Faster debugging tooling for user-sourced feedback while protecting privacy
- Enhanced user feedback channels (`/bug` command in Claude Code)

The key insight: the eval suite caught capability regressions on benchmarks but not infrastructure-induced degradation on real workloads. These are different failure modes requiring different detection strategies.

*— Ch 09 (a-postmortem-of-three-recent-issues)*

---

### Postmortem 2: Three Claude Code Quality Issues (April 2026)

Seven months later, three separate issues degraded Claude Code quality over the same month. All three were fixed by April 20 (v2.1.116). Anthropic reset usage limits for all subscribers as compensation.

**Issue 1: Reasoning Effort Default Change (March 4 – April 7)**

The team adjusted Claude Code's default reasoning effort from "high" to "medium" to address extremely long response times. Reasonable motivation: if the interface appears frozen, users abandon it.

The problem: user feedback revealed this was the wrong tradeoff. Users preferred higher intelligence with the option to manually select lower effort for simpler tasks. The change was reverted on April 7.

The failure mode is not a bug — it's a product decision that was wrong. The team measured one metric (response time) and optimized it, degrading another (reasoning quality) that was harder to measure.

**Issue 2: Caching Bug Causing Memory Loss (March 26 – April 10)**

A prompt caching optimization intended to reduce latency for idle sessions contained a critical bug. The intended behavior: clear old thinking sections once. The actual behavior: clear reasoning history on every turn.

Effect: Claude appeared "forgetful and repetitive," losing context about its own previous decisions. The issue compounded on tool calls, progressively stripping reasoning blocks. This also triggered more cache misses — explaining reports of faster usage limit depletion.

The failure mode: a caching optimization with a logic error that affected the model's access to its own reasoning history. The symptom (forgetfulness) is hard to distinguish from other causes. The compounding effect (more cache misses → faster limit depletion) created multiple user-visible symptoms from a single root cause.

**Issue 3: Verbosity-Reduction Prompt (April 16 – April 20)**

Anthropic added system prompt instructions to reduce output:
> "keep text between tool calls to ≤25 words. Keep final responses to ≤100 words unless the task requires more detail."

Internal testing showed no regressions. Broader evaluations revealed a **3% intelligence drop**. Immediately reverted.

The failure mode: a system prompt constraint that compressed outputs also compressed reasoning. The mechanism is straightforward — if the model has fewer words to work with between tool calls, it must compress or discard intermediate reasoning. Compressed reasoning → reduced quality.

The 3% figure is worth holding: a system prompt change caused a 3% intelligence drop. This is the magnitude of change you need your eval suite to detect. If your evals can't distinguish 97% from 100% performance, you're not monitoring system prompt changes.

**Structural changes:**
- Internal staff to use the public Claude Code build (dogfooding as detection)
- Broader per-model evaluations for all system prompt changes
- Stricter oversight for prompt modifications
- Soak periods and gradual rollouts for intelligence-affecting changes
- Clearer guidance ensuring model-specific changes target only intended versions

Two recurring themes across both postmortems:
1. **Evals must be sensitive enough to catch small regressions.** A 3% intelligence drop matters; a benchmark that only catches catastrophic failures doesn't.
2. **Continuous monitoring, not just pre-deployment checks.** Both postmortems involved problems that passed pre-deployment evaluation but degraded production performance.

*— Ch 24 (april-23-postmortem)*

---

### Production operational patterns: Claude Code Best Practices

The best practices article (April 2025) translates the architectural principles from earlier modules into operational patterns. The framing is direct: "Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills."

**Give the agent a way to verify its work.** This is the single highest-leverage change:

| Strategy | Weak prompt | Strong prompt |
|---|---|---|
| Verification criteria | "implement a validateEmail function" | "write a validateEmail function. example test cases: user@example.com → true, invalid → false, user@.com → false. run the tests." |
| UI verification | "make the dashboard look better" | "[paste screenshot] implement this design. take a screenshot of the result and compare it to the original. list differences and fix them." |
| Root cause | "the build is failing" | "the build fails with this error: [paste error]. fix it and verify the build succeeds. address the root cause, don't suppress the error." |

Without verification, the agent's only feedback loop is the user noticing failure. With verification, the agent closes its own loop. This reduces error rate and reduces the number of turns needed to reach a working solution.

**Explore-plan-code-commit workflow:**
1. **Explore** — Plan mode only. Read files, answer questions, make no changes.
2. **Plan** — Produce a detailed implementation plan. Edit it directly before proceeding (`Ctrl+G` opens it in your editor).
3. **Implement** — Code against the plan, verifying as you go.
4. **Commit** — Commit with a descriptive message and create a PR.

The plan phase exists to separate problem identification from implementation. Letting an agent jump directly to coding often produces code that solves the wrong problem — the agent inferred intent from incomplete context and committed to a solution before the problem was clear.

**CLAUDE.md design:** The guidance is ruthless about scope:

| Include | Exclude |
|---|---|
| Bash commands Claude can't guess | Anything Claude can infer from the code |
| Code style that differs from defaults | Standard language conventions |
| Testing instructions and preferred runners | Detailed API documentation |
| Repository etiquette | Long explanations or tutorials |
| Project-specific architectural decisions | File-by-file codebase descriptions |

The test: "Would removing this line cause Claude to make mistakes?" If not, cut it. Over-specified CLAUDE.md files are a common failure pattern — they pollute context with information the agent either already has or doesn't need.

**Common failure patterns to avoid:**
- **The kitchen sink session:** Accumulating unrelated tasks in one session. Fix: `/clear` between tasks.
- **Correcting over and over:** After two failed corrections, the context contains the history of failed approaches. Fix: `/clear` and write a better initial prompt.
- **The over-specified CLAUDE.md:** Bloats every session context. Fix: ruthlessly prune.
- **The trust-then-verify gap:** Letting the agent run without specifying how it should verify its work. Fix: always provide verification criteria.
- **Infinite exploration:** Scoping investigation tasks too broadly. Fix: use subagents for investigation, scope narrowly.

*— Ch 05 (claude-code-best-practices)*

---

### Distribution and tooling: Desktop Extensions

The Desktop Extensions article (June 2025) documents a pattern that recurs across production AI tooling: the barrier between capability and adoption is often distribution, not the capability itself.

MCP servers had significant capability but required terminal commands, manual JSON editing, dependency resolution, and GitHub discovery. Non-technical users couldn't use them. Desktop Extensions solved this with a packaging format (`.mcpb` files) that bundles everything — server code, all dependencies, manifest — into a single installable file.

The architectural decisions reveal production deployment thinking:
- **Node.js bundled into Claude Desktop:** Eliminates the "user needs to install Node first" failure mode. External runtime requirements are a reliability hazard.
- **OS keychain for credentials:** Sensitive configuration values never stored in plain text. This is the same principle as the managed agents credential isolation pattern.
- **Automatic updates:** Removes the need for users to manually track and apply updates, which is a reliability hazard in production systems.
- **Enterprise controls (Group Policy, MDM, allowlists/blocklists):** Production deployments need centralized management. An extension system without enterprise controls won't be adopted in enterprise environments.

The pattern: capability is necessary but not sufficient. A useful capability that's hard to install, maintain, and govern won't be used reliably.

*— Ch 07 (desktop-extensions)*

---

### The production monitoring hierarchy

Synthesizing across all sources, production monitoring for AI agents operates at four levels:

**Level 1: Pre-deployment evals.** Run before each release. Must be sensitive enough to catch 3% intelligence regressions. Catch bugs before users see them but don't catch novel failure modes.

**Level 2: Continuous quality monitoring on production traffic.** The September 2025 postmortem's main lesson: standard benchmarks don't capture infrastructure-induced degradation. Continuous production monitoring catches regressions that pre-deployment evals miss because they test a different distribution.

**Level 3: User feedback channels.** `/bug` command, feedback mechanisms, explicit reporting. Surfaces issues that monitoring misses — particularly subtle quality degradation that's hard to quantify but obvious to users.

**Level 4: Internal dogfooding.** Using the production system internally before external release. The April 2026 postmortem's specific recommendation. Catches experience-level issues (UX degradation, frustration patterns) that metrics miss.

Each level catches failures that the others don't. The September 2025 postmortem involved failures at Level 1 (evals didn't catch it) and Level 2 (no continuous monitoring). The April 2026 postmortem involved failures at Level 1 (internal evals didn't reproduce the production issue) and Level 4 (insufficient dogfooding).

---

## 2. Key Questions (the depth check)

<details>
<summary><b>Q1.</b> The XLA:TPU bug was latent — masked by a December 2024 workaround. When the workaround was removed, the latent bug surfaced. What does this imply about the relationship between workarounds and production reliability?</summary>

Workarounds that mask underlying bugs without fixing them create technical debt of a specific kind: hidden landmines that can detonate when the workaround is removed. The December 2024 workaround was removed for a good reason — it appeared unnecessary — but the appearance of unnecessary-ness was itself caused by the workaround's masking effect. This creates an epistemically difficult situation: engineers reasonably conclude the workaround is unnecessary, remove it, and expose the bug they didn't know existed. The implication for production reliability: document the reason for every workaround, not just what it does. "Workaround for unknown issue — don't remove without investigation" is more reliable than "removed old workaround" in a commit message. The root cause should be understood before the workaround is removed.
</details>

<details>
<summary><b>Q2.</b> The verbosity-reduction prompt caused a 3% intelligence drop. What mechanism explains this? And why did internal testing show no regressions while broader evaluations caught it?</summary>

Mechanism: constraining word count between tool calls forces the model to compress intermediate reasoning. Compressed reasoning → reduced quality. The model has less context for its own reasoning steps, which produces subtly lower-quality decisions across many tasks. The internal testing gap: internal tests likely covered specific task types with well-defined success criteria. The 3% drop is distributed across many task types, each showing small degradation — not a catastrophic failure on any specific benchmark. Broader evaluations covering more diverse task types aggregate the distributed degradation into a visible overall regression. This is why diversity of eval tasks matters: targeted evals catch known failure modes, but distributed regressions require broad coverage to detect.
</details>

<details>
<summary><b>Q3.</b> The caching bug caused progressive reasoning loss on tool calls. Why does this failure mode compound rather than being consistent across turns?</summary>

Each turn removes reasoning blocks. On turn 1, the agent loses the reasoning from the previous turn. On turn 2, it has no reasoning from turns 0 or 1. On turn 3, it has no reasoning history at all. The agent operates increasingly without prior context. This compounds because each degraded turn produces a lower-quality response, which the next turn must work from — so the input quality also degrades over time. The failure is invisible early (turn 1 looks nearly normal) and catastrophic late (turn 10 has no accumulated reasoning). This is a specific failure mode of any system that processes and accumulates state — progressive corruption becomes visible only after several turns. Detection requires multi-turn eval tasks, not single-turn checks.
</details>

<details>
<summary><b>Q4.</b> The "explore first, then plan, then code" workflow adds overhead. When should you skip the plan phase?</summary>

Skip planning when: (a) the scope is clear and bounded — fixing a typo, renaming a variable, adding a log line; (b) the task is purely mechanical — "run all tests and report failures"; (c) the implementation is a direct mapping from a well-specified requirement with no ambiguity. The plan phase exists to surface misunderstanding before implementation begins — it catches "I thought you meant X but you meant Y" errors. If there's no meaningful ambiguity to surface, the plan phase adds overhead without benefit. The diagnostic: if the plan would just be a restatement of the prompt, skip it. If the plan would add specificity (what files to change, what approach to take, what edge cases to handle), it's earning its cost.
</details>

<details>
<summary><b>Q5.</b> The Claude Code Best Practices article says: "If you've corrected Claude more than twice on the same issue, the context is cluttered with failed approaches." Why does correction history hurt rather than help?</summary>

Failed approaches occupy context tokens and prime the model's attention toward the failure pattern. Instead of "here's the problem, here's the solution space," the context says "here's the problem, here's a wrong approach, here's another wrong approach, here's why both were wrong, here's the problem again." The model must now avoid two specific wrong approaches while solving the problem — which anchors its reasoning to the failure history rather than the solution space. A clean context with a better-specified prompt produces better results than an accumulated context with failed correction history. The principle: context quality matters as much as context volume. Accumulated failures are low-signal tokens that crowd out high-signal tokens.
</details>

<details>
<summary><b>Q6.</b> The Desktop Extensions spec addresses both individual user convenience and enterprise governance requirements. Why do both matter for production reliability?</summary>

Individual convenience (one-click install, automatic updates) determines adoption rate — a capability that's hard to install is installed inconsistently or not at all. Inconsistent installation creates a fleet reliability problem: different users on different versions with different configurations. Automatic updates ensure the fleet stays coherent. Enterprise governance (MDM, Group Policy, blocklists, private directories) is necessary for the corporate deployment scenario where IT manages the fleet centrally — without it, enterprises can't approve and control which extensions employees use, which is a security and compliance requirement. Production reliability for a tool used at scale requires both: high individual adoption (from ease of use) and centralized control (from enterprise governance).
</details>

---

## 3. Hands-On

This module is case-study focused. The hands-on exercise is postmortem design.

**Write your own postmortem template (≈25 min).**

Design a postmortem template that would help a team investigate a future AI agent quality regression. Your template should capture:

1. **Symptoms:** What users observed, when, and on what platforms/models. Include quantitative measures where possible.

2. **Detection timeline:** When the issue started, when it was first reported, when it was identified, when it was fixed. Include the gap between start and detection.

3. **Root cause classification:**
   - Infrastructure bug (hardware misconfiguration, routing error, compiler bug)
   - Model behavior change (prompt modification, training difference)
   - Eval gap (working as intended, but intention was wrong)
   - Product decision tradeoff (optimizing metric A degraded metric B)

4. **Detection gap analysis:**
   - What monitoring would have caught this earlier?
   - What eval would have caught this in pre-deployment testing?
   - What user feedback mechanism would have surfaced this faster?

5. **Fix and verification:**
   - What was changed to fix it?
   - How was the fix verified as sufficient?
   - What residual risk remains?

6. **Structural changes:**
   - What processes changed to prevent recurrence?
   - What monitoring was added?

Then apply your template retroactively to one of the six bugs from the two postmortems. Fill in every field as completely as the blog content allows. Where information is missing from the blog, note what you'd want to know.

**What to record in your notes:**
- Your completed postmortem template for one bug.
- The most important structural change each postmortem identified, and why it addresses the root cause rather than the symptom.
- One monitoring gap from the September 2025 postmortem that you would have added based on everything you learned in this course.

---

## 4. Reflection

1. **Both postmortems note that "staggered rollouts" created confusing signals** — different users experiencing different versions created an appearance of broad inconsistent degradation. But staggered rollouts are also a best practice for catching problems before full deployment. How do you design a rollout strategy that catches problems early without creating detection noise? What's the rollout schedule that balances speed of detection against signal clarity?

2. **The April 2026 postmortem explicitly recommends "ensure more internal staff use the public Claude Code build" as a preventive measure.** This is dogfooding — using your own product in production. But internal users have different usage patterns than external users (more sophisticated, less representative). How do you get the signal value of dogfooding without the selection bias of highly-technical internal users? What's the minimum viable dogfooding program that would have caught the reasoning effort default change before external rollout?

3. **The best practices article's CLAUDE.md guidance says: remove any line whose absence wouldn't cause mistakes.** This is a context budget discipline. But agents working on new domains may not know what they're missing. How do you discover what to put in CLAUDE.md in the first place? What's the process for identifying the gaps that a CLAUDE.md entry would fill — and how is that different from the process for identifying eval gaps?

---

## 5. Key Takeaways

- **Production failures in AI systems fall into distinct categories.** Infrastructure bugs (misconfiguration, compiler bugs), model behavior changes (prompt modifications), eval gaps (wrong things measured), and product decision tradeoffs each require different detection and prevention strategies.
- **Pre-deployment evals and continuous production monitoring are complementary, not substitutes.** Both postmortems involved failures that passed pre-deployment testing. Continuous monitoring on production traffic catches infrastructure-induced and distribution-shift failures that benchmarks miss.
- **3% intelligence drops matter and must be detectable.** A system prompt change can cause a 3% regression. Your eval suite needs to be sensitive enough to catch this. If it only catches catastrophic failures, you're flying blind on smaller regressions.
- **Give the agent verification, not just instructions.** The single highest-leverage operational change: specify how the agent should verify its own work. This closes the feedback loop without requiring user intervention.
- **Context discipline compounds.** Clean context + specific prompts + working verification criteria + ruthlessly pruned CLAUDE.md produces better results than any single optimization. The failure modes (kitchen sink sessions, over-correcting, over-specified CLAUDE.md) all accumulate noise in context that crowds out signal.
