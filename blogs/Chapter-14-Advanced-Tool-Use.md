# Introducing Advanced Tool Use on the Claude Developer Platform

**Published:** November 24, 2025
**Author:** Bin Wu, with contributions from Adam Jones, Artur Renault, Henry Tay, Jake Noble, Noah Picard, Sam Jiang, and the Claude Developer Platform team.
**Source:** https://www.anthropic.com/engineering/advanced-tool-use

---

## Overview

Anthropic has released three beta features enabling Claude to discover, learn, and execute tools dynamically. These capabilities address fundamental challenges in building AI agents that work effectively across extensive tool libraries.

## The Three Features

### 1. Tool Search Tool

**Problem:** Loading all tool definitions upfront consumes enormous context. A five-server MCP setup with 58 tools consumed approximately 55,000 tokens before any conversation began.

**Solution:** The Tool Search Tool allows Claude to discover tools on-demand rather than loading all definitions initially. Only relevant tools get expanded into context when needed.

**Key benefits:**
- Reduces token usage by 85% while maintaining access to full tool libraries
- Improves accuracy significantly (Opus 4 improved from 49% to 74%)
- Preserves context for conversation history and reasoning

**Implementation approach:** Mark tools with `defer_loading: true` to make them discoverable on-demand, while keeping critical tools always loaded.

### 2. Programmatic Tool Calling

**Problem:** Traditional tool calling creates context pollution from intermediate results. Processing a complex workflow requires multiple inference passes, with each tool result accumulating in context regardless of relevance.

**Solution:** Claude orchestrates tools through code rather than sequential API calls. The code executes in a sandboxed environment, allowing Claude to process data without polluting its context window.

**Key benefits:**
- Reduces token consumption by approximately 37% on complex tasks
- Eliminates 19+ unnecessary inference passes in multi-step workflows
- Improves accuracy (knowledge retrieval improved from 25.6% to 28.5%)

**Use case example:** A budget compliance check that processes 2,000+ expense line items returns only the three people who exceeded their budgets to Claude's context.

### 3. Tool Use Examples

**Problem:** JSON schemas define structural validity but cannot express usage patterns, optional parameter inclusion rules, or API conventions.

**Solution:** Provide concrete example tool calls demonstrating correct usage patterns. This shows Claude format conventions, nested structure patterns, and parameter correlations.

**Key finding:** Tool use examples improved accuracy from 72% to 90% on complex parameter handling in internal testing.

## Strategic Implementation

These features work synergistically:

- **Start with your primary bottleneck** (context bloat, intermediate results, or parameter errors)
- **Layer additional features** as needed rather than implementing all three simultaneously
- **Combine strategically:** Tool Search ensures correct tool discovery, Programmatic Tool Calling ensures efficient execution, and Tool Use Examples ensure proper invocation

## Getting Started

The features are available in beta through the Claude Developer Platform. Enable them using:

```
betas=["advanced-tool-use-2025-11-20"]
```

Complete documentation and cookbooks are available in the platform's developer resources.

---

**Key Insight:** These features enable building agents that "work seamlessly across hundreds or thousands of tools" without traditional context window constraints, representing a fundamental shift toward intelligent tool orchestration rather than simple function calling.
