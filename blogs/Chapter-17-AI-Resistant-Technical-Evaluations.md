# Designing AI-Resistant Technical Evaluations

**Published:** January 21, 2026
**Author:** Tristan Hume, Lead, Performance Optimization Team at Anthropic
**Source:** https://www.anthropic.com/engineering/AI-resistant-technical-evaluations

---

## Overview

Anthropic's performance engineering team faced a critical challenge: as Claude models became increasingly capable, their technical hiring evaluation—a take-home test involving accelerator code optimization—stopped effectively distinguishing top human candidates from AI. This post documents three iterations of the assessment and lessons learned about maintaining evaluation integrity.

## The Original Take-Home Design

### Context and Motivation

In late 2023, Anthropic needed performance engineers for newly acquired TPU and GPU clusters. A Twitter post attracted over 1,000 candidates—too many for traditional interviews. Tristan Hume developed a 4-hour take-home test as an efficient alternative.

### Design Principles

The assessment prioritized:

- **Realistic conditions**: Candidates worked in their own environments without observation
- **Adequate time**: Unlike 50-minute interviews, the longer horizon reflects actual engineering work
- **High signal**: Problems offered multiple opportunities to demonstrate skills with wide scoring distribution
- **Engagement**: Candidates found the work genuinely interesting

The test explicitly permitted AI assistance, recognizing that performance engineers use these tools on the job.

### The Simulated Machine

Hume built a Python simulator resembling TPU architecture with:

- Manually managed scratchpad memory
- VLIW (multiple parallel execution units)
- SIMD (vector operations)
- Multicore capabilities

The task involved parallel tree traversal—deliberately non-ML-specific to avoid requiring deep learning background. Candidates progressed from serial implementation through exploiting various parallelism types.

## Performance Against Claude Models

### Initial Success

The original test successfully identified strong candidates. One early hire immediately began optimizing production kernels and found compiler bugs. Over 18 months, approximately 1,000 candidates completed it, with dozens now working at Anthropic on infrastructure and all models since Claude 3 Opus.

### Claude Opus 4 (May 2025)

Claude 3.5 Sonnet had already reached 50% of human candidate performance. Testing Claude Opus 4 revealed it outperformed most humans within the 4-hour limit. "Hume identified where the model began struggling and made that the new starting point" for version 2, which shortened the time limit to 2 hours and added architectural complexity.

### Claude Opus 4.5 (Pre-Release)

Version 2 lasted several months before Claude Opus 4.5 solved it within the 2-hour window, matching best human performance. Crucially, it identified memory bandwidth bottlenecks—where most humans stopped—then discovered clever workarounds exploiting problem structure.

## Redesign Attempts

### Option 1: Ban AI Assistance

Hume rejected this approach, viewing it as impractical and contrary to real-world performance engineering where AI tools remain integral.

### Option 2: Raise the Bar

Colleagues suggested requiring candidates to "substantially outperform what Claude Code achieves alone." This risked making human deliberation irrelevant—candidates might spend half the session understanding Claude's work rather than driving it.

### Attempt 1: Data Transposition Problem

Based on real Anthropic kernel optimization involving register transposition and bank conflict avoidance, this appeared promising until Claude Opus 4.5 discovered an unanticipated optimization: transposing the entire computation rather than the data. When Hume patched this, Claude Code with extended thinking still solved it, drawing on broader training data.

### Attempt 2: Zachtronics-Inspired Constraints

The successful redesign leveraged programming puzzle games using "tiny, heavily constrained instruction sets" that force unconventional thinking. This problem set proved sufficiently out-of-distribution to challenge Claude while remaining solvable for capable humans without AI assistance.

Critically, Hume provided "no visualization or debugging tools," requiring candidates to make strategic decisions about tooling investment—a genuine signal of professional judgment.

## Key Insights

**Training Data Advantage**: Claude excels at problems where substantial engineering literature exists (transposition, bank conflicts). Novelty matters.

**Problem Structure Matters**: Zachtronics-style puzzles succeeded because they require unconventional approaches that diverge from standard optimization patterns in training data.

**Realism vs. Robustness**: The original assessment's strength—resembling actual work—became its weakness. The replacement "simulates novel work" rather than realistic work.

## The Open Challenge

Anthropic released the original take-home with unlimited time, noting that humans retain advantages at "sufficiently long time horizons." Performance benchmarks (measured in clock cycles):

- Claude Opus 4 (extensive compute): 2164 cycles
- Claude Opus 4.5 (2-hour session): 1790 cycles
- Claude Opus 4.5 (improved harness): 1363 cycles
- Best human submission: exceeded all above

Candidates scoring below 1487 cycles are invited to email performance-recruiting@anthropic.com.

## Conclusion

As AI capabilities advance, hiring evaluations must evolve. The solution isn't eliminating AI assistance—that's unrealistic and misaligned with professional practice—but rather designing assessments that leverage human reasoning advantages in domains where current models struggle. The journey from accelerator optimization to constrained programming puzzles illustrates this principle.
