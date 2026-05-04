# Building Effective Agents

**Published:** December 19, 2024
**Authors:** Erik S. and Barry Zhang
**Source:** https://www.anthropic.com/engineering/building-effective-agents

---

## Overview

Anthropic's research team, having collaborated with numerous organizations implementing LLM agents, found that "the most successful implementations weren't using complex frameworks or specialized libraries. Instead, they were building with simple, composable patterns."

## Core Definitions

The post distinguishes between two fundamental approaches:

- **Workflows**: Systems where language models and tools follow predetermined code paths
- **Agents**: Systems where models autonomously direct their processes and tool usage

## Decision Framework: When to Build Agents

Rather than defaulting to complex agent systems, developers should pursue the simplest viable solution. Agentic systems involve tradeoffs—increased latency and costs for improved task performance.

Workflows suit well-defined tasks requiring predictability. Agents excel when flexibility and model-driven decision-making are essential at scale. Many applications succeed through optimized single LLM calls enhanced with retrieval and contextual examples.

## Framework Selection Guidance

Available frameworks include the Claude Agent SDK, AWS Strands Agents, Rivet, and Vellum. While these tools simplify implementation, they introduce abstraction layers complicating debugging. Starting with direct API calls allows developers to implement many patterns in minimal code before adopting frameworks.

## Building Blocks and Patterns

### The Augmented LLM

The foundational element combines language models with retrieval capabilities, tools, and memory systems. Modern models actively generate search queries, select appropriate tools, and determine what information to preserve.

### Workflow: Prompt Chaining

This pattern decomposes tasks into sequential steps, with each call processing the previous output. Programmatic checkpoints verify progress.

**Ideal for:** Tasks decomposable into fixed subtasks, prioritizing accuracy over speed

**Applications:**
- Generating marketing copy, then translating it
- Creating document outlines before full composition

### Workflow: Routing

Classification directs inputs to specialized handlers, enabling separation of concerns and optimized prompts for distinct categories.

**Ideal for:** Complex tasks with distinct categories benefiting from separate handling

**Applications:**
- Directing different customer service query types to appropriate processes
- Routing simple queries to efficient models (Haiku) and complex ones to capable models (Sonnet)

### Workflow: Parallelization

Tasks run simultaneously with aggregated outputs. Two variations exist:

- **Sectioning**: Breaking tasks into independent parallel subtasks
- **Voting**: Running tasks multiple times for diverse outputs

**Ideal for:** Tasks benefiting from parallel processing or multiple perspectives

**Applications:**
- Content moderation with parallel screening and response generation
- Code review by multiple prompt variations
- Performance evaluation assessing different aspects

### Workflow: Orchestrator-Workers

A central model breaks down tasks, delegates to specialized workers, and synthesizes results—differing from parallelization through dynamic rather than predefined subtasks.

**Ideal for:** Complex tasks with unpredictable subtasks

**Applications:**
- Multi-file code modifications
- Search tasks analyzing multiple information sources

### Workflow: Evaluator-Optimizer

One model generates responses while another evaluates and provides feedback iteratively.

**Ideal for:** Tasks with clear evaluation criteria where iterative refinement demonstrably improves outputs

**Applications:**
- Literary translation capturing nuance
- Complex research requiring multiple search rounds

### Agents

Autonomous systems operate independently on open-ended problems, using environmental feedback (tool results, execution outputs) to assess progress. These emerge as models improve in reasoning, tool usage, and error recovery.

**Ideal for:** Open-ended problems with unpredictable step counts and unavoidable hardcoded paths

**Key characteristics:**
- Begin with human direction or discussion
- Operate independently after task clarification
- Pause at checkpoints or blockers for human input
- Include stopping conditions maintaining control

**Applications:**
- Software engineering tasks (GitHub issue resolution)
- Computer use automation

## Implementation Principles

Three core principles guide effective agent development:

1. **Simplicity** in design
2. **Transparency** explicitly showing planning steps
3. **Tool Documentation and Testing** through careful agent-computer interface (ACI) design

## Tool Design Best Practices

Tool definitions deserve equivalent prompt engineering attention as overall prompts. Key recommendations:

- Grant models sufficient tokens for reasoning before committing to outputs
- Keep formats resembling naturally occurring internet text
- Eliminate formatting overhead (line counting, string escaping)
- Include example usage, edge cases, and clear boundaries
- Optimize parameter names and descriptions for clarity
- Extensively test model tool usage in development environments
- Apply mistake-proofing principles (poka-yoke) to arguments

The authors note investing substantial effort in agent-computer interfaces mirrors human-computer interface design philosophy.

## Practical Applications

### Customer Support

Agents effectively combine conversational interfaces with tool integration:
- Natural conversation flows with external data access
- Programmatic actions (refunds, ticket updates)
- Clear success measurement

### Coding Agents

Software development shows remarkable agent potential:
- Automated test verification
- Iterative solution improvement
- Well-defined problem space
- Objective output quality measurement

---

## Conclusion

Success requires building appropriate systems for specific needs rather than maximizing sophistication. Start with simple optimized prompts, evaluate comprehensively, and add complexity only when demonstrably improving outcomes. Frameworks assist rapid prototyping but shouldn't obscure underlying mechanics—developers should understand their complete implementation before production deployment.
