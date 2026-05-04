# Module 13: Multi-Agent Systems & Decoupled Architecture

**Time:** ~1.5 hours (≈45 min reading · ≈30 min hands-on · ≈15 min reflection)
**Builds on:** Module 08 (Sub-Agents), Module 09 (Harness Design)    **Feeds:** Module 14 (Production)

## Learning Objectives

- Identify the architectural preconditions that make multi-agent systems worth their 15× token cost, and predict when single-agent systems are better.
- Explain the three-component decoupled architecture (session, harness, sandbox) and why coupling them creates infrastructure brittleness.
- Describe the coordination mechanisms used in the C compiler project (lock files, GCC oracle, role specialization) and analyze what would break without each.
- Apply the stateless harness pattern: explain what `wake(sessionId)` enables that a stateful harness cannot.

---

## 1. Concept Synthesis

### When multi-agent is worth the cost

Module 08 established that sub-agents earn their 15× token cost when tasks are genuinely independent and each requires substantial exploration. Module 13 extends this to larger-scale multi-agent systems and the infrastructure required to run them reliably.

The 15× cost figure comes from Anthropic's multi-agent research system: running multiple agents in parallel with orchestration overhead costs roughly 15× what a single agent would cost for the same task. This is a real, load-bearing number for product decisions. A system that costs $0.10/query as a single agent costs ~$1.50/query as a multi-agent system. The benefit must be proportional.

The preconditions that justify the cost:
- **Genuine independence:** Sub-tasks don't share state that would require coordination. Reading four quarterly reports is independent; debugging a cross-file race condition is not.
- **Each sub-task requires real exploration:** If each sub-task is a single lookup, the overhead of spawning, prompting, and collecting from sub-agents exceeds the cost of running them in sequence.
- **Context isolation provides signal value:** The orchestrator needs uncontaminated views of each sub-task. If all sub-tasks benefit from shared context, isolation degrades quality.
- **Parallelism reduces latency meaningfully:** If the task is time-sensitive and sub-tasks can run simultaneously, the wall-clock time reduction justifies the cost.

When these preconditions are absent, multi-agent adds cost without benefit. The research system article makes this concrete for coding work: "Most coding tasks involve fewer truly parallelizable parts than research — even large refactors are mostly serial, and the sub-agent overhead produces no gain."

*— Ch 06 (built-multi-agent-research-system)*

### The Anthropic multi-agent research system: scaling rules

The research system article provides empirical guidance on how many agents to use:

| Task type | Agents | Tool calls each |
|---|---|---|
| Simple fact-finding | 1 | 3–10 |
| Direct comparisons | 2–4 | 10–15 |
| Complex research | 10+ | Many, divided |

The failure mode at the lower end: deploying 10 agents for a simple fact-finding task produces 10 sub-agents duplicating each other's searches. The system showed exactly this problem — "when the lead agent allowed simple instructions like 'research the semiconductor shortage,' sub-agents duplicated work and explored the same search trajectories."

The fix requires four elements in every sub-agent invocation: (1) a specific angle (what to look for), (2) a tool budget (how many searches), (3) an output format (what structure to return), and (4) source guidance (where to look, what to avoid). Without angle specificity, agents converge on the same questions and findings.

The compression discipline is equally critical: sub-agents return 1,000-2,000 token summaries to the orchestrator, not their full exploration traces. With 10 sub-agents each returning 20,000-token traces, the orchestrator faces the same context rot problem that sub-agents were meant to solve. The compression step is what makes the architecture work.

The system produced a 90.2% improvement in research quality — but only when agents were given distinct angles and required to compress their findings.

*— Ch 06 (built-multi-agent-research-system)*

### The C compiler case study: coordination at scale

Nicholas Carlini's C compiler project (Feb 2026) pushed multi-agent coordination to an extreme: 16 parallel Claude agents building a Rust-based C compiler from scratch, capable of compiling the Linux kernel. 100,000 lines of code, $20,000 in API costs, nearly 2,000 sessions.

**Architecture:** Multiple agents running in Docker containers, each mounting a shared git repository. Each agent runs a persistent loop:

```bash
while true; do
    COMMIT=$(git rev-parse --short=6 HEAD)
    LOGFILE="agent_logs/agent_${COMMIT}.log"
    claude --dangerously-skip-permissions \
           -p "$(cat AGENT_PROMPT.md)" \
           --model claude-opus-X-Y &> "$LOGFILE"
done
```

No stopping point, no waiting for human direction. The agents commit, push, merge, and continue.

**Coordination mechanisms:**

*Lock files in `current_tasks/`:* Each agent claims a task by writing a lock file before starting. This prevents two agents from implementing the same function simultaneously — a problem that would produce merge conflicts and wasted work. The lock file pattern is simple but effective: no central coordinator needed.

*GCC as oracle:* The hardest coordination problem emerged when all agents converged on compiling the Linux kernel — a single massive task that collapsed parallelism to zero. The solution: use GCC (a known-good compiler) to randomly distribute work. Each agent compiles a different subset of files using Claude's compiler and compares output against GCC. The oracle creates unbounded parallel work from a single goal.

*Role specialization:* Beyond core compiler development, agents focused on specific work: code deduplication, performance optimization, output efficiency, design critique and refactoring, documentation maintenance. Specialization prevented the agents from all optimizing the same functions.

**Results:** The compiler passes 99% of standard test suites, compiles QEMU, FFmpeg, SQLite, PostgreSQL, Redis, and can compile itself. The gap: no 16-bit x86 code generation, incomplete assembler and linker, suboptimal code generation efficiency.

**What broke:** "New features and bugfixes frequently broke existing functionality near the project's conclusion" — the compiler approaches Claude Opus 4.6's capability ceiling. At some point, the agent's local context doesn't include enough of the codebase to make changes that are globally consistent.

**The key insight:** Test suite quality is foundational. "Claude will work autonomously to solve whatever problem I give it — therefore the testing harness must be nearly flawless — poor tests cause agents to optimize toward wrong objectives." Multi-agent systems amplify both good and bad incentives in the evaluation function.

*— Ch 18 (building-c-compiler)*

### Decoupling session, harness, and sandbox

The Scaling Managed Agents article (Apr 2026) addresses a different dimension of multi-agent systems: the infrastructure architecture. The initial design combined session logs, harness logic, and execution environments into single containers. This coupling created three problems:

**Infrastructure brittleness:** Coupled systems use a "pet" model — each container is unique and irreplaceable. When a container fails, the session is lost. Debugging requires shell access to the container, which risks exposing user data.

**Assumption staleness:** Hardcoded assumptions about model behavior become incorrect as models improve. Context-anxiety workarounds designed for earlier Claude versions proved unnecessary with newer models — but the code remained, adding complexity without benefit.

**Connectivity constraints:** Customers wanting private network integration faced architectural constraints imposed by the coupling.

The redesign virtualizes infrastructure into three independent interfaces:

**Session:** An append-only event log that exists *outside* the harness. The harness can fail, restart, and recover by reading the session. The `getEvents()` interface allows flexible retrieval — positional slices, rewinding before specific moments, rereading previous context. This addresses the problem of long-horizon tasks that exceed context limits: rather than deciding irreversibly what to discard, the session serves as a queryable external store.

**Harness:** Now stateless. It treats sandboxes as tools via `execute(name, input) → string`. A failed harness restarts via `wake(sessionId)`, recovering from `getSession(id)`. "Wake" is the key operation: any harness instance can resume any session because session state lives outside the harness.

**Sandbox:** Containers are now interchangeable — "cattle" instead of "pets." If one fails, the harness treats it as a tool error and may provision a replacement. No container is uniquely necessary.

The stateless harness + external session combination unlocks architectural patterns not possible with coupled systems:
- Many stateless harnesses connecting to different execution environments only when needed
- Brains (harnesses) operating against resources in customer VPCs
- A single brain coordinating work across multiple sandboxes simultaneously
- Brains delegating to other brains

**Performance:** Decoupling eliminated forced container provisioning on each request. Time-to-first-token improved ~60% at p50 and >90% at p95 — inference now starts immediately, not after waiting for container setup.

**Security improvement:** Credentials are never exposed to generated code. Two patterns:
1. Bundle authentication with resources during initialization — the agent has access to resources but never sees the credentials
2. Use secure vaults with MCP proxy servers for external tools

*— Ch 23 (managed-agents)*

### The composability principle

The decoupled architecture follows the same principle operating systems established decades ago: virtualize underlying components into stable abstractions. POSIX's `read()` works across hardware from the 1970s to today because the interface is stable even as implementations changed.

"Opinionated about interfaces, unopinionated about implementations" allows the same framework to accommodate Claude Code, task-specific harnesses, and future approaches that don't exist yet. A harness written for today's Claude can be replaced by a harness optimized for next year's model without changing the session or sandbox layers.

This composability is what enables the "multiple brains, multiple hands" pattern: the brains (harnesses) and the hands (sandboxes) can be mixed and matched because they communicate through a stable interface rather than direct coupling.

The managed agents notebook demonstrates this: an orchestrating agent reads a bug report, finds the bug, writes a fix, opens a PR, survives CI, and addresses review feedback — all through a stateful session that persists across the full workflow. The harness is minimal; the session records everything.

*— Ch 23 (managed-agents)*

---

## 2. Key Questions (the depth check)

<details>
<summary><b>Q1.</b> The C compiler project used lock files in `current_tasks/` for coordination. What would happen without this mechanism, and is it sufficient for all coordination problems in the project?</summary>

Without lock files, multiple agents would claim the same task simultaneously. Two agents implementing `strtol()` produce a merge conflict; resolution requires human judgment or deterministic conflict resolution rules the agents don't have. The resulting compiler would have either merged code with subtle inconsistencies or lost one implementation entirely. However, lock files are not sufficient for all coordination: they prevent duplicate task selection but don't solve the "kernel convergence" problem (all agents wanting to work on the same large task). That required the GCC oracle approach — a fundamentally different mechanism that creates parallel work from a single goal rather than preventing parallelism collapse.
</details>

<details>
<summary><b>Q2.</b> A managed agent system uses a stateful harness (harness holds session state). It crashes mid-task. What's the recovery path, and what state is potentially lost?</summary>

With a stateful harness, the session state — the full conversation history, tool call records, intermediate results — lives in the harness process. When the harness crashes, that state is lost. Recovery requires either: (a) restarting the entire task from scratch, (b) restoring from periodic checkpoints if implemented, or (c) relying on any persisted artifacts (files written to disk, database writes made before crash). State between checkpoints or in memory is gone. With a decoupled architecture (session external, harness stateless), the harness crash loses no session state — it's all in the external event log. `wake(sessionId)` reconstructs the harness state from the session, and the harness can resume from where the session left off.
</details>

<details>
<summary><b>Q3.</b> The research system article says multi-agent adds 15× token cost. The C compiler project spent $20,000. Is cost a meaningful objection to multi-agent systems, and how should you think about it?</summary>

Cost is a product constraint, not an architectural objection. $20,000 to build a C compiler from scratch is substantially less than hiring engineers to do the same (even an incomplete compiler would cost hundreds of thousands in human time). The relevant comparison is always: cost of multi-agent system vs. cost of the alternative (human labor, single-agent + retries, not doing the task at all). The 15× overhead figure is a multiplier on single-agent cost — if the single-agent approach couldn't do the task at all, the 15× doesn't apply. The principled question: does the multi-agent approach produce results that are worth the cost difference from single-agent? For research tasks with 90.2% quality improvement, yes. For tasks where the quality gain is marginal, no.
</details>

<details>
<summary><b>Q4.</b> The C compiler project noted that "new features and bugfixes frequently broke existing functionality near the project's conclusion." What's the root cause, and is this addressable with more agents?</summary>

The root cause is bounded context: each agent can see only a portion of a 100,000-line codebase at a time. Changes that are locally correct (given the subset of the codebase the agent sees) can be globally inconsistent (breaking invariants that depend on code outside the agent's context). This is a context limitation, not a parallelism limitation. Adding more agents doesn't fix it — it adds more agents operating with the same bounded context, producing more locally-correct-but-globally-inconsistent changes. The fix requires either: better context management (agents that maintain global state through structured summaries or a shared state layer), or task decomposition that enforces global invariants as a precondition (an agent that checks global consistency before committing). Neither was in the original architecture.
</details>

<details>
<summary><b>Q5.</b> The managed agents architecture improved TTFT by 60% at p50 and >90% at p95. Why is the p95 improvement larger than p50?</summary>

At p50 (median request), the performance was already reasonable — the container provisioning overhead was low because containers were often warm or reused. At p95 (tail latency), the bottleneck was cold-start container provisioning: a request that needed a fresh container had to wait for full container setup before inference started. Decoupling eliminates this: inference starts immediately using any available harness, and sandboxes are provisioned as needed (or reused). The p95 improvement is larger because p95 was dominated by the cold-start case, which decoupling eliminates entirely. This is a common pattern in distributed systems: architectural changes that eliminate whole classes of slow operations have a disproportionate effect on tail latency.
</details>

<details>
<summary><b>Q6.</b> The research system's sub-agent prompts require four elements: angle, tool budget, output format, and source guidance. The compiler project's agent prompt is minimal (AGENT_PROMPT.md with simple instructions). Why does one system need detailed invocation specifications and the other doesn't?</summary>

The research system has inherently ambiguous task space: without angle specification, multiple agents explore the same questions. The task — research a topic — has many valid approaches that need to be partitioned across agents to avoid duplication. Explicit angle specification is the coordination mechanism that assigns each agent a distinct slice of the problem space. The compiler project has a different coordination mechanism: lock files assign specific tasks, and the GCC oracle creates naturally distinct subtasks (compile this subset of files). The task space is already partitioned by the coordination layer, so the agent prompt doesn't need to specify angles. The general principle: where implicit coordination fails, explicit invocation specification must compensate.
</details>

---

## 3. Hands-On

**Notebook:**
- [`claude-cookbooks/managed_agents/CMA_orchestrate_issue_to_pr.ipynb`](../claude-cookbooks/managed_agents/CMA_orchestrate_issue_to_pr.ipynb)

**Read for architecture, not as an SDK tutorial.**

This notebook demonstrates an agent completing a realistic end-to-end software maintenance workflow: read a bug report, find the bug, write a fix, open a PR, survive CI, and respond to review feedback. The implementation uses Anthropic's Managed Agents API, but the architectural patterns are the lesson — not the specific API calls.

Pay attention to:
- **Cell 4 (Agent + environment + session):** The agent, environment (sandbox with pytest), and session are created as three separate objects. Notice how they're linked but independently managed.
- **Cell 5 (Agent creation):** The system prompt is minimal — the agent is not given a rigid workflow, just a role ("maintainer bot") and available tools. The agent determines the workflow itself.
- **Cell 6 (Run the full chain):** A single instruction kicks off the full loop. The session persists through CI failures and review feedback — the agent recovers from intermediate failures without a restart.
- **Cell 8 (Multi-turn verification):** Sessions are stateful across turns. A follow-up message can verify the final state because the session remembers the entire prior interaction.
- **Sidebar (Cell 10):** Real vs. mock repository setup. Note how swapping from mock to real requires only changing the mount type — the architecture is unchanged.

**Design exercise (≈20 min):**

Draw the session/harness/sandbox interaction diagram for the `bn-fit-modify` task from the infrastructure noise article (the task where agents diverge on solution strategy under different resource limits). Assume a decoupled architecture:
- What does the session record when the agent installs a large dependency stack?
- What does the session record when the agent instead writes a standard-library solution?
- If the harness crashes between tool calls and restarts via `wake(sessionId)`, which state is preserved and which is lost?

This isn't a coding exercise — sketch it in your notes. The goal is to make the decoupled model concrete.

**What to record in your notes:**
- The three components (session/harness/sandbox) and what each one owns.
- The `wake(sessionId)` operation: what it requires, what it restores, what it can't restore.
- One failure mode in the C compiler architecture that the managed agents decoupled design would fix.

---

## 4. Reflection

1. **The C compiler project spent $20,000 and produced a compiler that passes 99% of standard test suites but has significant gaps.** At $20k, the project is already cheaper than hiring engineers. But the 99% figure glosses over the missing pieces (no 16-bit x86, incomplete assembler). In production software, the gap between 99% and 100% is often where the hard problems live. How do you evaluate the success of multi-agent systems on tasks where completeness matters? What does "done" mean when you can't get to 100%?

2. **The decoupled architecture makes harnesses "opinionated about interfaces, unopinionated about implementations."** This is a clean principle, but interface design is hard — once you publish an interface, everything that depends on it is coupled to your design decisions. What properties would you want in the `execute(name, input) → string` harness-to-sandbox interface to ensure it accommodates use cases that don't exist yet? What's the minimal interface that doesn't foreclose future options?

3. **The research system and the compiler project use fundamentally different coordination mechanisms** (explicit angle specification vs. lock files + oracle). Both work for their specific domains. Is there a general coordination principle that explains why each mechanism fits its domain? What would a coordination mechanism for a third domain (e.g., multi-agent code review of a large PR) look like?

---

## 5. Key Takeaways

- **Multi-agent earns its 15× cost only when tasks are genuinely independent, parallel, and benefit from context isolation.** Research tasks with distinct angles are strong fits; serial coding tasks are not. Use the agent count table (1/2-4/10+) as a starting heuristic, but verify independence before scaling.
- **Decouple session, harness, and sandbox.** Coupling all three into one container creates a "pet" — unique, irreplaceable, and brittle. Decoupling makes each component independently replaceable, enables `wake(sessionId)` recovery, and eliminates cold-start latency at the p95.
- **Coordination mechanisms determine parallelism quality.** Lock files prevent duplicate work; oracle-driven distribution creates parallelism from a single goal; explicit angle specification prevents convergence on the same research threads. The coordination mechanism must be matched to the task structure.
- **Test suite quality is the bottleneck in autonomous multi-agent systems.** Poor graders → agents optimize toward the wrong objective. In the C compiler project, high-quality test suites (GCC torture tests, real project compilation) drove coherent progress. The agent's goal is exactly what the test measures.
- **The "opinionated about interfaces, unopinionated about implementations" principle enables evolution.** Infrastructure that virtualizes components behind stable interfaces can swap implementations as models and tools improve — the same property that makes POSIX relevant 50 years later.
