# The "think" Tool: Enabling Claude to Stop and Think in Complex Tool Use Situations

**Published:** March 20, 2025
**Source:** https://www.anthropic.com/engineering/claude-think-tool

---

## Overview

Anthropic has introduced a "think" tool that improves Claude's performance on complex problem-solving tasks. This feature creates dedicated space for structured reasoning during tool use, distinct from the extended thinking capability.

## What is the "think" Tool?

The "think" tool allows Claude to pause and reflect during response generation, particularly when processing information from tool results. Unlike extended thinking (which occurs before response generation), this tool operates within the response stream itself.

According to the post: *"The 'think' tool is for Claude, once it starts generating a response, to add a step to stop and think"* about whether it has gathered sufficient information to proceed.

## Key Use Cases

The tool excels in three specific scenarios:

1. **Tool output analysis** - Processing previous tool call results before taking action
2. **Policy-heavy environments** - Following detailed guidelines with compliance verification
3. **Sequential decision-making** - Complex multi-step tasks where earlier decisions affect later ones

## Performance Results

Testing on τ-Bench showed substantial improvements:

- **Airline domain**: The optimized prompt version achieved 0.570 on pass¹ metrics, representing a *"54% relative improvement"* over baseline (0.370)
- **Retail domain**: Achieved 0.812 without additional prompting versus 0.783 baseline
- **SWE-Bench**: Contributed to a state-of-the-art 0.623 score, with isolated improvements of 1.6% average

## Implementation Guidance

**Best practices include:**
- Strategic prompting with domain-specific examples
- Placing complex guidance in system prompts rather than tool descriptions
- Starting with challenging use cases to test effectiveness

**Not recommended for:**
- Non-sequential tool calls
- Simple instruction-following without constraints

The tool requires minimal code implementation while offering meaningful performance gains in the right contexts.
