# A Postmortem of Three Recent Issues

**Published:** September 17, 2025
**Author:** Sam McAllister
**Source:** https://www.anthropic.com/engineering/a-postmortem-of-three-recent-issues

## Summary

Between August and early September 2025, three infrastructure bugs intermittently degraded Claude's response quality. Anthropic has now resolved all three issues and published a detailed postmortem explaining what happened, why detection took time, and what changes they're implementing.

## The Three Bugs

**1. Context Window Routing Error (August 5)**

Some Sonnet 4 requests were misrouted to servers configured for the 1M token context window. A load balancing change on August 29 worsened the problem, affecting up to 16% of Sonnet 4 requests at peak impact. About 30% of Claude Code users experienced at least one degraded response during this period.

**2. Output Corruption (August 25)**

A misconfiguration on TPU servers caused token generation errors, occasionally producing unexpected characters like Thai or Chinese text in English responses, or syntax errors in code. This affected Opus 4.1, Opus 4, and Sonnet 4 across different timeframes.

**3. Approximate Top-K XLA:TPU Miscompilation (August 25)**

A compiler bug was triggered when deploying improved token selection code. The issue stemmed from mixed precision arithmetic where operations disagreed on which token had highest probability, sometimes causing the best token to disappear entirely.

## Root Cause Analysis

The most complex issue involved a latent bug in the XLA:TPU compiler. Anthropic's December 2024 workaround had inadvertently masked the real problem—when engineers later removed that workaround, they exposed a deeper bug in the approximate top-k operation.

The fundamental issue: "operations that should have agreed on the highest probability token were running at different precision levels."

## Detection Challenges

Why did it take so long to identify these problems? Several factors:

- **Privacy protections** limited engineers' ability to examine problematic user interactions without explicit feedback
- **Evaluation gaps** meant their standard benchmarks didn't capture the degradation users experienced
- **Inconsistent symptoms** across platforms created confusing, contradictory reports
- **Over-reliance on noisy evaluations** made connecting issues to recent changes difficult

## Changes Going Forward

Anthropic is implementing four major improvements:

1. **More sensitive evaluations** designed to reliably differentiate between working and broken implementations
2. **Continuous quality evaluations** running on production systems, not just during deployment
3. **Faster debugging tooling** that better processes user-sourced feedback while protecting privacy
4. **Enhanced user feedback channels** through the `/bug` command in Claude Code and feedback mechanisms in Claude apps

The company emphasizes that user reports remain crucial for identifying issues their internal testing misses.
