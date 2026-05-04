# How We Built Our Multi-Agent Research System

**Published:** June 13, 2025
**Authors:** Jeremy Hadfield, Barry Zhang, Kenneth Lien, Florian Scholz, Jeremy Fox, and Daniel Ford
**Source:** https://www.anthropic.com/engineering/multi-agent-research-system

---

## Overview

Anthropic's Research feature employs multiple Claude agents working collaboratively to tackle complex topics. The system uses an orchestrator-worker architecture where a lead agent coordinates specialized subagents operating in parallel, enabling dynamic exploration of open-ended problems.

## Benefits of Multi-Agent Systems

### Why Multiple Agents?

Research tasks are inherently unpredictable. A single agent cannot pre-determine all necessary steps, as investigations unfold dynamically based on discoveries. Multi-agent systems excel because they:

- **Enable parallel exploration**: Subagents investigate different aspects simultaneously, with each maintaining separate context windows for independent reasoning
- **Facilitate compression**: Multiple agents distill insights from vast information sources before synthesizing findings
- **Scale performance significantly**: According to Anthropic's evaluations, their multi-agent system with Claude Opus 4 as lead and Claude Sonnet 4 subagents "outperformed single-agent Claude Opus 4 by 90.2%"

### Performance Metrics

Analysis of the BrowseComp evaluation revealed that three factors explain 95% of performance variance:
- Token usage alone accounts for 80% of variance
- Number of tool calls and model selection comprise the remaining factors

This validates the architecture's distributed approach, where agents with separate context windows enable parallel reasoning capacity.

### Trade-offs

The primary downside is token consumption. Multi-agent systems typically use approximately 15× more tokens than chat interactions. Economic viability requires tasks with sufficient value to justify increased costs.

## Architecture Overview

### System Design

The orchestrator-worker pattern functions as follows:

1. **User Query Reception**: The lead agent analyzes incoming requests and develops a research strategy
2. **Subagent Spawning**: Specialized subagents are created to explore different aspects in parallel
3. **Iterative Searching**: Each subagent independently executes searches, evaluates results using interleaved thinking, and identifies information gaps
4. **Result Synthesis**: The lead agent compiles findings and determines whether additional research is necessary
5. **Citation Processing**: A final CitationAgent identifies specific source locations for all claims

This differs fundamentally from static Retrieval Augmented Generation (RAG), which fetches pre-determined chunks. The research system "uses multi-step search that dynamically finds relevant information, adapts to new findings, and analyzes results to formulate high-quality answers."

### Key Components

**Memory Integration**: The lead agent saves its research plan to persistent memory to prevent context loss if the 200,000-token limit is exceeded.

**Extended Thinking**: Both lead and subagents use extended thinking modes—the lead agent for planning and subagents for post-tool result evaluation via interleaved thinking.

**Parallel Tool Calling**: Subagents execute multiple tools simultaneously, reducing research time by up to 90% for complex queries.

## Prompt Engineering Principles

Anthropic identified eight core prompting strategies:

### 1. Develop Agent Understanding
Effective iteration requires understanding agent behavior. Using the Console tool to simulate exact system conditions reveals failure modes: agents continuing past sufficient results, crafting overly verbose queries, or selecting incorrect tools.

### 2. Teach Delegation Skills
Lead agents require detailed instructions for decomposing tasks. Subagents need:
- Clear objectives
- Specified output formats
- Tool and source guidance
- Defined task boundaries

Without detailed descriptions, agents duplicate work or leave information gaps.

### 3. Scale Effort Appropriately
Embedding explicit scaling rules prevents both underinvestment and resource waste:
- Simple fact-finding: 1 agent, 3-10 tool calls
- Direct comparisons: 2-4 subagents, 10-15 calls each
- Complex research: 10+ subagents with divided responsibilities

### 4. Prioritize Tool Design
Quality tool descriptions are essential. "Bad tool descriptions can send agents completely wrong paths," so each tool requires distinct purpose and clear documentation. Anthropic created a tool-testing agent that iteratively used flawed tools and rewrote descriptions, achieving a "40% decrease in task completion time for future agents."

### 5. Enable Self-Improvement
Claude 4 models demonstrate capability as prompt engineers. When presented with failure modes, they diagnose issues and suggest improvements, enhancing tool ergonomics systematically.

### 6. Search Strategy: Wide to Narrow
Agents should mirror expert human research by exploring broadly before focusing. Initial searches should use short, broad queries before progressively narrowing scope.

### 7. Guide Thinking Processes
Extended thinking serves as a controllable scratchpad where agents:
- Plan approaches
- Assess tool-task fit
- Determine query complexity
- Define subagent roles

Testing showed extended thinking improved instruction-following, reasoning, and efficiency.

### 8. Leverage Parallel Tool Calling
Sequential execution proved painfully slow. Parallelization strategies include:
- Spawning 3-5 subagents simultaneously rather than serially
- Enabling subagents to call 3+ tools in parallel

## Effective Agent Evaluation

### Start Small, Iterate Fast
Early development shows dramatic improvements from minor changes. Anthropic began with approximately 20 test queries representing real usage patterns, allowing observable impact assessment without requiring hundreds of test cases initially.

### LLM-as-Judge Evaluation
Research outputs resist programmatic evaluation due to free-form text and multiple valid answers. LLM judges evaluated outputs against rubrics assessing:
- Factual accuracy
- Citation accuracy
- Completeness
- Source quality
- Tool efficiency

A single LLM call outputting 0.0-1.0 scores proved most consistent with human judgment.

### Human Testing Remains Essential
Automated evaluations miss edge cases that human testers identify. Anthropic's manual testing revealed agents consistently chose SEO-optimized content farms over authoritative sources like academic PDFs, prompting addition of source quality heuristics.

## Production Reliability Challenges

### Stateful Error Compounding
Agents maintain state across extended operations spanning many tool calls. Minor failures cascade unpredictably. Rather than restarting from scratch (expensive and frustrating), Anthropic implemented:
- Durable code execution
- Error recovery systems
- Graceful degradation mechanisms
- Model-driven adaptation when tools fail

### Debugging Complexity
Non-deterministic agent behavior prevents simple reproduction of failures. Anthropic deployed full production tracing to diagnose decision patterns and interaction structures without monitoring conversation contents.

### Deployment Coordination
Highly stateful systems require careful update strategies. Rather than simultaneous version upgrades, Anthropic employs rainbow deployments, gradually shifting traffic while maintaining both old and new versions simultaneously.

### Execution Bottlenecks
Currently, lead agents execute subagents synchronously, waiting for completion before proceeding. This simplifies coordination but creates information flow bottlenecks. Asynchronous execution would enable concurrent agent operation but introduces challenges in result coordination, state consistency, and error propagation.

## Use Cases and Applications

Anthropic's Clio embedding plot reveals top usage categories:
- Developing software systems across specialized domains (10%)
- Developing professional and technical content (8%)
- Business growth and revenue strategy development (8%)
- Academic research and educational material assistance (7%)
- Information research and verification (5%)

Users reported discovering business opportunities, navigating complex healthcare options, resolving technical challenges, and saving days of work through research connections they wouldn't have found independently.

## Additional Production Patterns

### End-State vs. Turn-by-Turn Evaluation
For agents modifying persistent state across conversations, focus on end-state evaluation rather than validating every intermediate step. Agents may find alternative paths to correct outcomes; evaluation should confirm achieved goals rather than prescribed processes.

### Long-Horizon Conversation Management
Production agents engage in conversations spanning hundreds of turns. Anthropic implemented:
- Phase completion summaries
- Essential information storage in external memory
- Fresh subagent spawning with clean contexts when approaching limits
- Stored context retrieval to prevent work loss

### Filesystem Output for Subagents
Rather than funneling all subagent outputs through coordinators, specialized agents can persist work independently. This pattern prevents information loss during multi-stage processing and reduces token overhead from copying large outputs through conversation history.

## Conclusion

Building production-grade multi-agent systems requires substantially more engineering than prototypes. The compound nature of errors means minor issues can derail agents entirely. Success demands:
- Careful prompt and tool design
- Solid heuristics
- Observability and tight feedback loops
- Robust operational practices
- Cross-functional collaboration between research, product, and engineering teams

Despite these challenges, multi-agent research systems demonstrate significant value for open-ended problems, enabling users to accomplish research objectives that would require substantially more time working independently.
