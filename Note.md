Claude code 101
=================
prompt -> (agentic loop) ->gather context ->Task action -> Verify results -> Done
Tools backbone
Permission
Plan mode for complex changes and doing a safe code review

The Workflow
The explore -> Plan -> Code -> Commit

Context Management
/compact
use subagent to save context window
use skills or shut down unnecessary mcps to save context window
detailed prompt always save context window in the long run, cuz it can help the agent to understand the task better and avoid unnecessary self-exploration

The CLAUDE.md File
use /init to create one
let cc put what u want to remember in this file
use @ to add reference docs
better not use it when starting off a project

Subagent
keep context window clean

Skill

MCP
MCP connects Claude Code to your external tools and data sources. 

Hooks
Hooks give you deterministic control over Claude Code's behavior.


Moudle 2
=========
agents asked to evaluate their own work are unreliable critics

The evaluator's role scales with difficulty. For tasks beyond baseline model capability, external grading adds substantial value. For tasks within capability, it becomes overhead.

The harness is not a sacred artifact. Stress-test it on every model release. It might be unnecessary for newer models. Every harness component is an assumption about model limitations, and those assumptions go stale as models improve. Scary because it means there is no "done" state for harness design — you must re-examine your harness on every model release. A harness component that helped at one model generation can become net-negative overhead at the next. Treat harnesses as load-bearing only as long as they are currently load-bearing.


Three scaling rules the team learned the hard way (these matter more than they look):

Simple fact-finding: 1 agent, 3–10 tool calls. (Don't spin up subagents for "what's the capital of France?")
Direct comparisons: 2–4 subagents, 10–15 calls each.
Complex research: 10+ subagents with divided responsibilities.

Two failure modes that show up at multi-agent scale:

Failure mode 1: Stateful error compounding. The multi-agent research article puts it bluntly: agents "are stateful and errors compound." A subagent that misunderstands its task at turn 3 will keep building on that misunderstanding through turn 30. The fix isn't smarter models — it's durable execution (resumable from checkpoints, not from scratch on failure) and graceful degradation (when a tool fails, the agent adapts rather than aborts).

In ordinary software, a bug at step 3 produces wrong output at step 3, and step 4 either crashes (loud, locatable) or proceeds with the wrong input (visible in tests). In agents, a step-3 misunderstanding silently colors every subsequent decision the model makes — there's no exception, no test, just gradual drift. The fix is structural: durable execution + checkpointable state, so that you can resume from a known-good moment rather than restart from scratch.

Failure mode 2: Synchronous bottlenecks at the coordinator. In the current research system, the lead agent waits synchronously for all subagents before continuing. This simplifies coordination at the cost of throughput. 

not every multi-LLM system is "multi-agent." The terminology gets sloppy.

Multi-agent: at least one of the participants has its own agent loop (plan → act → observe → repeat) inside the larger system. The research system is multi-agent because subagents themselves call tools in loops.

A separate evaluator has a different context window and a different prompt; it doesn't see the generator's reasoning, only the generator's output. It can't be "primed" by the work it's evaluating. This isn't about model identity (you can use the same model class); it's about context isolation. The evaluator hasn't already convinced itself the work is good.

Orchestrator-Worker Failure modes to consider:
- Orchestrator might not break down tasks optimally (prompt engineering is critical)
- Workers may return empty or malformed responses (we handle this with validation)
- XML parsing can fail if models don't follow format exactly (consider using JSON as an alternative)

Build-evaluate-iterate workflowfor developing effective tools
The cycle: run evals → get transcripts → Claude analyzes → refine tool definitions → repeat.

Fortunately, in our experience, the tools that are most “ergonomic” for agents also end up being surprisingly intuitive to grasp as humans.

Even your tool response structure—for example XML, JSON, or Markdown—can have an impact on evaluation performance: there is no one-size-fits-all solution. This is because LLMs are trained on next-token prediction and tend to perform better with formats that match their training data. The optimal response structure will vary widely by task and agent. We encourage you to select the best response structure based on your own evaluation.

Advanced tool design: 

Three distinct bottlenecks need three distinct fixes. Definition overload → Tool Search. Context pollution → Programmatic Tool Calling. Schema ambiguity → Tool Use Examples. Misdiagnosing the bottleneck means applying the wrong fix.
PTC keeps data in code, not in context. The model writes code that runs in a sandbox; only the final filtered result reaches the model's context. This is the mechanism behind 85%+ token reductions on data-heavy workflows.
Parallel execution requires deliberate design. Sequential tool calls are the default. The batch tool pattern and explicit parallel invocation make independent calls concurrent — and the research system's 90% time reduction shows this matters at scale.
Examples communicate what schema cannot. JSON schema specifies structure; examples communicate convention, idiom, and typical usage. The 72% → 90% accuracy improvement from examples is structurally similar to the tool description engineering gains from Module 03.
Layer features starting from your biggest bottleneck. Adding all three advanced features simultaneously makes failures harder to diagnose. Identify the dominant bottleneck, fix it, measure, then layer.