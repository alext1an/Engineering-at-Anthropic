# Effective Harnesses for Long-Running Agents

**Published:** November 26, 2025
**Author:** Justin Young
**Source:** https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents

---

## Overview

Anthropic researchers addressed a fundamental challenge in autonomous agent development: enabling AI systems to maintain productivity across multiple context windows spanning hours or days. The core insight involved implementing a two-part system inspired by effective human engineering practices.

## The Core Problem

Long-running agents face significant obstacles because they operate in discrete sessions without memory of previous work. As the article notes, "each new session begins with no memory of what came before." Claude's failures in extended tasks manifested in two patterns:

1. **Overambition**: Agents attempted to complete entire projects at once, often leaving features partially implemented
2. **Premature completion**: Later sessions mistook partial progress for finished work

## The Two-Part Solution

### Initializer Agent
The first session uses specialized prompting to establish:
- An `init.sh` script for running the development environment
- A `claude-progress.txt` file documenting work history
- An initial git commit establishing the project foundation

### Coding Agent
Subsequent sessions employ prompts emphasizing:
- Single-feature incremental progress
- Clean, production-ready code states
- Clear git commits and progress documentation

## Environment Management Components

**Feature Lists**: The system generates comprehensive JSON files breaking down requirements. As emphasized in the guidelines, "It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality."

**Testing Protocols**: Agents improved performance when equipped with browser automation tools (Puppeteer MCP), enabling end-to-end human-like testing rather than unit tests alone.

**State Recovery**: Git integration allowed agents to identify and revert problematic changes while maintaining working code bases.

## Session Structure

Effective sessions follow this pattern:
1. Verify current working directory
2. Review progress files and git history
3. Run initial development server tests
4. Select next incomplete feature
5. Implement incrementally
6. Verify through end-to-end testing
7. Commit changes with documentation

## Remaining Challenges

The research identifies open questions regarding optimal agent architecture—whether specialized agents (testing, QA, code cleanup) might outperform single general-purpose agents. Future applications beyond web development remain unexplored, though financial modeling and scientific research present promising domains.

## Key Contributors

The work represents collaboration across Anthropic teams, with special recognition to David Hershey, Prithvi Rajasakeran, Jeremy Hadfield, and others who enabled safe long-horizon autonomous software engineering capabilities.
