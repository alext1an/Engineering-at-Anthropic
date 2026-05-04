# Code Execution with MCP: Building More Efficient Agents

**Published:** November 4, 2025
**Authors:** Adam Jones and Conor Kelly
**Source:** https://www.anthropic.com/engineering/code-execution-with-mcp

---

## Overview

The Model Context Protocol (MCP) enables AI agents to connect with external tools and data through a universal standard. However, as agents scale to hundreds or thousands of tools, efficiency challenges emerge. This article explores how code execution can help agents interact with MCP servers more effectively while consuming fewer tokens.

## The Problem: Token Inefficiency at Scale

Two primary challenges arise when agents connect to many MCP servers:

### 1. Tool Definition Overload

Loading all tool definitions upfront directly into the model's context window creates significant overhead. When agents access thousands of tools, they must process hundreds of thousands of tokens before processing user requests. This increases both latency and costs.

### 2. Intermediate Result Duplication

When models directly call tools, results flow through the model's context multiple times. For example, retrieving a document and then using it in another tool call means the full content passes through twice. Large documents—such as meeting transcripts spanning 50,000 tokens—create substantial inefficiencies and may exceed context limits.

## Solution: Code Execution with MCP

Rather than exposing tools through direct tool-calling syntax, the approach presents MCP servers as code APIs. Agents write code to interact with these services, addressing both challenges:

**Implementation Structure:**

Tools are organized as a file tree where each tool maps to a specific file. Agents discover tools by exploring the filesystem and loading only definitions needed for current tasks. The Google Drive to Salesforce example demonstrates the efficiency gain: reducing token usage from 150,000 to 2,000 tokens represents a 98.7% reduction.

**Code-Based Interaction:**

Instead of chaining tool calls, agents write traditional programming logic:

- Filter large datasets before returning results to the model
- Use loops and conditionals without repeated tool calls
- Execute complex operations in a single step

## Key Benefits

### Progressive Tool Discovery

Models can navigate filesystems effectively, reading tool definitions on-demand. A `search_tools` function allows agents to find relevant tools without loading everything upfront.

### Context-Efficient Results

Agents can filter, aggregate, and transform data within the execution environment. Processing a 10,000-row spreadsheet by filtering for specific statuses means the model sees only relevant rows, not the entire dataset.

### Enhanced Control Flow

Familiar programming constructs like loops and conditionals execute more efficiently than chaining multiple tool calls through the agent loop, reducing latency and token consumption.

### Privacy Preservation

Intermediate results remain in the execution environment by default. Sensitive data stays out of the model's context unless explicitly shared. The MCP client can tokenize personally identifiable information automatically, ensuring real data flows between services while the model never sees sensitive details.

### State Management and Skill Building

Agents can persist intermediate results and reusable code functions. Over time, agents develop higher-level capabilities that evolve their operational scaffolding.

## Implementation Considerations

Code execution introduces operational complexity requiring:

- Secure sandboxing environments
- Resource limits and monitoring
- Infrastructure overhead

These requirements should be weighed against benefits including reduced token costs, lower latency, and improved tool composition.

## Conclusion

As MCP adoption expands, code execution offers established software engineering patterns to address agent scalability. By enabling agents to write code rather than making direct tool calls, organizations can build more efficient systems handling larger tool ecosystems while reducing computational costs and improving response times.

---

*The article acknowledges feedback from Jeremy Fox, Jerome Swannack, Stuart Ritchie, Molly Vorwerck, Matt Samuels, and Maggie Vo.*
