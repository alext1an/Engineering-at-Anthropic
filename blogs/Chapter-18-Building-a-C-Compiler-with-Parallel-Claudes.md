# Building a C Compiler with a Team of Parallel Claudes

**Published:** February 5, 2026
**Author:** Nicholas Carlini, Safeguards Team Researcher
**Source:** https://www.anthropic.com/engineering/building-c-compiler

## Overview

Anthropic researcher Nicholas Carlini pioneered a novel supervision approach called "agent teams," where multiple Claude instances collaborate on shared codebases without continuous human oversight. To demonstrate this capability, Carlini deployed 16 agents to construct a Rust-based C compiler from scratch—one capable of compiling the Linux kernel across multiple architectures.

The resulting compiler—spanning 100,000 lines of code and consuming $20,000 in API costs across nearly 2,000 sessions—successfully builds Linux 6.9 on x86, ARM, and RISC-V systems. More importantly, the project reveals critical insights about designing autonomous agent systems.

## Technical Architecture

### The Core Loop

Carlini implemented a persistent task loop that keeps Claude continuously engaged:

```bash
#!/bin/bash

while true; do
    COMMIT=$(git rev-parse --short=6 HEAD)
    LOGFILE="agent_logs/agent_${COMMIT}.log"

    claude --dangerously-skip-permissions \
           -p "$(cat AGENT_PROMPT.md)" \
           --model claude-opus-X-Y &> "$LOGFILE"
done
```

This approach eliminates the typical stopping point where models await human direction.

### Parallel Execution Strategy

Multiple agents work simultaneously through Docker containers mounting a shared git repository. Each agent:
- Clones the upstream repository locally
- Claims specific tasks via lock files in `current_tasks/`
- Merges changes from parallel colleagues
- Pushes updates back upstream

This design prevents duplicate work while enabling specialization across agents.

## Key Design Principles

### High-Quality Test Suites

Carlini emphasizes that "Claude will work autonomously to solve whatever problem I give it." Therefore, the testing harness must be nearly flawless—poor tests cause agents to optimize toward wrong objectives. The project incorporated compiler benchmarks, open-source package verifiers, and continuously-integrated regression detection.

### Context-Aware Feedback

Designing for Claude meant reconsidering standard testing practices. The harness maintains:
- Extensive README files with current status
- Brief console output (avoiding context pollution)
- Detailed logfiles for offline analysis
- Progress tracking with deterministic sampling

### Breaking Monolithic Tasks

When all agents converged on compiling the Linux kernel—a single massive task—parallelism collapsed. The solution leveraged GCC as a known-good oracle, randomly distributing compilation across agents by comparing GCC output against Claude's compiler output on different file subsets.

### Role Specialization

Beyond core compiler development, agents focused on:
- Code deduplication
- Performance optimization
- Output efficiency
- Design critique and refactoring
- Documentation maintenance

## Capabilities and Limitations

### Achievements

The Opus 4.6 compiler successfully:
- Passes 99% of standard compiler test suites (including GCC torture tests)
- Compiles major projects: QEMU, FFmpeg, SQLite, PostgreSQL, Redis
- Demonstrates self-compilation capability
- Runs complex applications (Doom included)

### Shortcomings

Notable gaps include:
- Absence of 16-bit x86 code generation (relying on GCC fallback)
- Incomplete assembler and linker implementation
- Suboptimal code generation efficiency
- Inconsistent preservation of existing functionality during new feature additions

Carlini notes that "new features and bugfixes frequently broke existing functionality" near the project's conclusion, indicating the compiler approaches Opus 4.6's capability ceiling.

## Cost-Benefit Analysis

At $20,000 for autonomous development, the expense remains substantially lower than human programmer costs for equivalent work—even accounting for incomplete functionality.

## Future Implications

The author expresses cautious optimism mixed with genuine concern. Autonomous agent teams enable ambitious goals previously requiring extensive human involvement. However, deploying unverified code introduces risks reminiscent of pre-modern software quality standards.

Carlini concludes that "we're entering a new world which will require new strategies to navigate safely," acknowledging both the tremendous potential and legitimate hazards of autonomous software development at scale.
