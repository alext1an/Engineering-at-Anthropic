# Equipping Agents for the Real World with Agent Skills

**Published:** October 16, 2025
**Source:** https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills

## Overview

Agent Skills represent a new framework for enhancing Claude's capabilities through organized directories containing instructions, scripts, and resources. As described in the announcement, "Claude is powerful, but real work requires procedural knowledge and organizational context."

## What Are Agent Skills?

Skills function as specialized capability packages that agents can dynamically discover and load. The core structure requires a `SKILL.md` file containing YAML frontmatter with metadata (name and description), followed by the actual guidance content.

## Key Design Principle: Progressive Disclosure

The architecture employs a multi-level disclosure strategy:

- **Level 1:** Skill name and description preload into the system prompt
- **Level 2:** Full `SKILL.md` content loads when Claude determines relevance
- **Level 3+:** Additional bundled files load contextually as needed

This approach resembles "a well-organized manual that starts with a table of contents, then specific chapters, and finally a detailed appendix."

## Practical Implementation

Skills can include executable code that Claude runs as tools. The PDF skill example demonstrates this: Claude executes Python scripts to extract form fields without loading large files into context, maintaining "consistency and repeatability" through code's deterministic nature.

## Development Guidelines

Best practices include:
- Starting with evaluation to identify capability gaps
- Structuring content for scalability by separating files
- Considering Claude's decision-making perspective
- Iterating based on real usage patterns

## Security Considerations

Users should "install skills only from trusted sources" and thoroughly audit unfamiliar skills before deployment, particularly examining code dependencies and external network connections.

## Availability and Future Direction

Agent Skills are currently supported across Claude.ai, Claude Code, the Claude Agent SDK, and the Claude Developer Platform, with ongoing feature development planned.
