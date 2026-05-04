# Effective Context Engineering for AI Agents

**Published:** September 29, 2025
**Authors:** Anthropic's Applied AI team: Prithvi Rajasekaran, Ethan Dixon, Carly Ryan, and Jeremy Hadfield, with contributions from Rafi Ayub, Hannah Moran, Cal Rueb, and Connor Jennings.
**Source:** https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

---

## Overview

Context engineering represents a shift from traditional prompt engineering toward managing the entire information landscape available to language models during inference. As AI agents become more sophisticated and operate across longer time horizons, thoughtfully curating which tokens enter the model's limited attention budget becomes crucial for reliable performance.

## Context Engineering vs. Prompt Engineering

While prompt engineering focuses on crafting effective instructions, context engineering encompasses the broader strategy of maintaining an optimal token configuration throughout an agent's operation. This includes system instructions, tools, external data, message history, and other information that influences model behavior.

The distinction matters because agent systems operate iteratively. As interactions accumulate, engineers must continuously decide what information deserves inclusion in subsequent inference steps—treating context as "a finite resource with diminishing marginal returns."

## The Attention Budget Challenge

Research demonstrates that models experience "context rot," where retrieval accuracy declines as context window size increases. This stems from architectural constraints: transformer models require n² pairwise attention relationships for n tokens, creating inherent tension between sequence length and focus precision.

Models also develop attention patterns based on training data distributions favoring shorter sequences, resulting in "less experience with...context-wide dependencies." While position encoding techniques enable longer sequences, they introduce degradation in understanding token positions.

## Anatomy of Effective Context

### System Prompts

Optimal system prompts strike a balance between specificity and flexibility—avoiding both "hardcoded complex, brittle logic" and overly vague guidance that assumes unshared context. Effective prompts use:

- Clear organizational structure (XML tags, Markdown headers)
- Simple, direct language at appropriate abstraction levels
- Minimal yet sufficient information for desired behavior
- Diverse, canonical examples rather than exhaustive edge cases

### Tools

Well-designed tools promote efficiency by returning token-efficient information and encouraging smart agent behavior. Tools should exhibit:

- Self-contained, unambiguous functionality
- Minimal overlap with other available tools
- Clear input parameters without ambiguity
- Minimal viable toolsets to prevent agent confusion

### Message History and Examples

Few-shot prompting remains valuable, but teams should curate "diverse, canonical examples that effectively portray the expected behavior" rather than stuffing exhaustive edge cases into prompts.

## Dynamic Context Retrieval

Modern agents increasingly use "just-in-time" strategies that maintain lightweight references (file paths, queries, links) and load data dynamically at runtime through tools. This mirrors human cognition—rather than memorizing everything, we use external organization systems to retrieve information on demand.

Benefits include:

- Storage efficiency without loading complete datasets
- Metadata signals that guide behavior (folder hierarchies, naming conventions, timestamps)
- Progressive disclosure enabling incremental discovery through exploration
- Maintained focus through selective working memory

Trade-offs involve slower runtime exploration versus pre-computed retrieval speed. Hybrid strategies often prove most effective—retrieving some data upfront while enabling autonomous exploration when beneficial.

## Long-Horizon Task Techniques

### Compaction

Agents approaching context limits can summarize conversations, compress critical details, and reinitiate new windows with condensed summaries. The art involves selecting what to preserve while discarding redundancy. Claude Code implements this by having the model preserve "architectural decisions, unresolved bugs, and implementation details while discarding redundant tool outputs."

Start by maximizing recall, then iterate toward precision by eliminating superfluous content like tool results no longer needed.

### Structured Note-Taking

Agents regularly write notes stored outside the context window, retrieving them later for persistent memory with minimal overhead. Examples include:

- To-do lists tracking progress
- Strategic notes enabling learning across sessions
- Knowledge bases built over extended interactions

Anthropic released a memory tool allowing agents to "build up knowledge bases over time, maintain project state across sessions, and reference previous work without keeping everything in context."

### Sub-Agent Architectures

Specialized sub-agents handle focused tasks with clean context windows while a coordinator maintains high-level plans. Each sub-agent can explore extensively but returns only condensed summaries (typically 1,000-2,000 tokens). This approach:

- Achieves clean separation of concerns
- Isolates detailed search contexts
- Allows the lead agent to synthesize results
- Proves particularly effective for complex research tasks

Task characteristics determine optimal approaches: compaction maintains conversational flow; note-taking excels for iterative development; multi-agent systems handle parallel exploration effectively.

## Core Principle

Effective context engineering seeks "the smallest possible set of high-signal tokens that maximize the likelihood of some desired outcome." This principle applies whether implementing compaction, designing token-efficient tools, or enabling autonomous exploration.

## Conclusion

Context engineering will remain central to building reliable, effective agents even as model capabilities advance. Smarter models require less prescriptive engineering and more autonomy, but treating context as precious and finite resource continues driving superior performance.
