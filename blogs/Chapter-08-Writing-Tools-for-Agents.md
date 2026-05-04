# Writing Effective Tools for Agents — with Agents

**Published:** September 11, 2025
**Source:** https://www.anthropic.com/engineering/writing-tools-for-agents

## Overview

This Anthropic engineering blog post explores how to develop high-quality tools for AI agents using Claude itself as a collaborator. The authors argue that agent tools require fundamentally different design approaches than traditional software APIs.

## Key Sections

### What is a Tool?

The post distinguishes between deterministic systems (producing identical outputs for identical inputs) and non-deterministic agents. Tools create a new contract between these two types of systems. As the authors note, "Tools are software reflecting a contract between deterministic systems and non-deterministic agents."

### How to Write Tools

The recommended workflow involves three stages:

1. **Building Prototypes:** Start with quick implementations using Claude Code, leveraging LLM-friendly documentation (like `llms.txt` files). Test locally via MCP servers or Desktop extensions.

2. **Running Evaluations:** Generate realistic evaluation tasks grounded in actual workflows, then measure performance systematically. The post recommends collecting metrics beyond accuracy, including runtime, token consumption, and error rates.

3. **Collaborating with Agents:** Use Claude to analyze evaluation transcripts and automatically improve tool implementations across multiple tools simultaneously.

### Principles for Effective Tools

**Choosing the right tools:** More tools don't necessarily improve outcomes. Instead of wrapping every API endpoint, focus on "high-impact workflows" that match your evaluation tasks. Tools should handle multiple operations efficiently rather than exposing granular, low-level functions that waste agent context.

**Namespacing:** Group related tools using consistent prefixes (like `asana_projects_search`) to help agents select appropriate tools and reduce confusion.

**Meaningful context:** Return only high-signal information. Replace cryptic identifiers with "semantically meaningful and interpretable language," which significantly reduces hallucinations.

**Token efficiency:** Implement pagination, filtering, and truncation with sensible defaults. Include "helpful instructions" in error messages to steer agents toward more efficient strategies.

**Prompt engineering:** Tool descriptions heavily influence performance. Write descriptions as if explaining to a new team member, making implicit context explicit and avoiding ambiguous parameter names.

## Notable Examples

The post includes comparative performance graphs showing human-written versus Claude-optimized Slack and Asana tools, with the AI-optimized versions achieving measurably better accuracy on held-out test sets.

## Conclusion

The authors emphasize that effective agent tools require reorienting software development "from predictable, deterministic patterns to non-deterministic ones," using systematic evaluation-driven approaches to ensure tools evolve alongside increasingly capable AI agents.
