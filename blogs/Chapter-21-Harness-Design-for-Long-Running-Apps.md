# Harness Design for Long-Running Application Development

**Published:** March 24, 2026
**Author:** Prithvi Rajasekaran, Anthropic Labs
**Source:** https://www.anthropic.com/engineering/harness-design-long-running-apps

## Overview

This blog post explores how Anthropic pushed Claude's capabilities in frontend design and autonomous software engineering through innovative harness design—specifically using a multi-agent architecture inspired by Generative Adversarial Networks (GANs).

## Key Concepts

### The Generator-Evaluator Pattern

The core innovation involves separating an agent that produces work from an agent that evaluates it. As the post explains, "agents tend to respond by confidently praising the work—even when quality is obviously mediocre" when self-evaluating. By creating independent evaluator agents, the system can provide concrete feedback loops that drive improvement.

### Two Critical Problems Solved

1. **Context Window Management**: Rather than compacting context (summarizing earlier conversation in place), the harness uses "context resets—clearing the context window entirely and starting a fresh agent" with structured handoffs between sessions.

2. **Self-Evaluation Bias**: Separating generator from evaluator proved more effective than prompting models to critique their own work.

## Frontend Design Application

For frontend design tasks, the team developed four grading criteria:

- **Design Quality**: Coherence and distinct identity
- **Originality**: Evidence of custom decisions over templates
- **Craft**: Technical execution (typography, spacing, contrast)
- **Functionality**: Usability and task completion

The evaluator used Playwright to interact with live pages before scoring. After 5-15 iterations per generation, designs improved measurably, with some showing "creative leaps" that weren't visible in single-pass generations.

## Full-Stack Coding Architecture

For complete application development, the team built a three-agent system:

**Planner Agent**: Expands simple prompts into detailed product specifications with ambitious scope and AI feature integration.

**Generator Agent**: Implements features using React, Vite, FastAPI, and SQLite/PostgreSQL, working in sprints with self-evaluation before handoff.

**Evaluator Agent**: Uses Playwright to test running applications like users would, checking against negotiated "sprint contracts" that define success criteria before implementation.

### Results Comparison

A retro game maker prompt yielded stark differences:

| Harness Type | Duration | Cost | Outcome |
|---|---|---|---|
| Solo Agent | 20 min | $9 | Broken gameplay, rigid workflow |
| Full Harness | 6 hours | $200 | Polished interface, working features, AI integration |

The harness version featured working entity systems, functional editors, and integrated AI assistance for content generation—capabilities completely absent from the solo run.

## Evolution and Simplification

With Claude Opus 4.6's improvements in planning, long-task coherence, and code review capabilities, the team iteratively simplified the harness:

- Removed sprint decomposition while maintaining planner and evaluator
- Shifted evaluator from per-sprint to single-pass final review for most tasks
- Improved prompting for agent-building capabilities

A Digital Audio Workstation prompt demonstrated the updated approach:

| Phase | Duration | Cost |
|---|---|---|
| Planner | 4.7 min | $0.46 |
| Build Rounds (3) | 3 hrs 20 min | $113.85 |
| QA Rounds (3) | 25 min | $10.39 |
| **Total** | **3 hrs 50 min** | **$124.70** |

The evaluator still caught critical gaps like missing interactive features (clip dragging, instrument panels, effect visualizations) that the generator initially implemented only as stubs.

## Design Principles

The post emphasizes several core lessons:

1. **Stress-test assumptions**: Each harness component encodes an assumption about model limitations; these become stale as models improve.

2. **Iterative simplification**: Remove components one at a time to understand what's load-bearing.

3. **Task-dependent evaluation**: The evaluator's value depends on whether tasks sit "beyond what the current model does reliably solo."

4. **Dynamic harness design**: As model capabilities expand, the space for novel harness combinations shifts rather than shrinks.

## Practical Insights

- **Prompting matters**: Detailed criteria and language ("museum quality" designs) directly shaped output character before evaluator feedback occurred.

- **Iterative scoring isn't linear**: Later implementations tend to be better overall, but specific iterations sometimes outperformed final versions.

- **QA tuning is necessary**: Out-of-the-box Claude requires several rounds of prompt refinement to avoid approving substandard work.

- **Context anxiety is real**: Sonnet 4.5 exhibited "context anxiety" where models wrap up prematurely near perceived context limits; Opus 4.6 largely eliminated this behavior.

## Conclusion

The work demonstrates that effective harness design scales with model improvement rather than becoming obsolete. The generator-evaluator pattern, combined with structured task decomposition and careful prompt engineering, enables autonomous development of complex, multi-hour applications that exceed baseline model capabilities by orders of magnitude.
