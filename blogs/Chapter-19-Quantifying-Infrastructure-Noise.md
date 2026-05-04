# Quantifying Infrastructure Noise in Agentic Coding Evals

**Published:** February 5, 2026
**Source:** https://www.anthropic.com/engineering/infrastructure-noise

---

## Overview

Anthropic researchers discovered that infrastructure configuration significantly impacts agentic coding benchmarks like SWE-bench and Terminal-Bench. Rather than purely measuring model capability, these evaluations are influenced by hardware resources, time limits, and system conditions.

## Key Findings

The research quantified infrastructure's effect by testing Claude across six resource configurations on Terminal-Bench 2.0:

- **Strict enforcement (1x specs):** 5.8% infrastructure error rate
- **3x headroom:** 2.1% error rate (statistically significant improvement, p < 0.001)
- **Uncapped resources:** 0.5% error rate, +6 percentage point success gain overall (p < 0.01)

The team notes: "Infrastructure configuration can swing agentic coding benchmarks by several percentage points—sometimes more than the leaderboard gap between top models."

## Critical Distinction

Up to roughly 3x the specified resources, improvements stem from fixing infrastructure reliability issues. Beyond that threshold, additional resources actively enable agents to solve previously unsolvable problems—fundamentally changing what gets measured.

The research illustrates this with the `bn-fit-modify` task: some models install large dependency stacks (succeeding under generous limits), while others implement solutions using only standard libraries (succeeding under tight constraints).

## Recommendations for Evaluators

1. **Specify both parameters:** Container runtimes use guaranteed allocation and hard kill thresholds separately—specify both rather than a single pinned value
2. **Calibrate the buffer:** Set the ceiling around 3x the per-task specs to eliminate spurious failures while maintaining resource pressure
3. **Document methodology:** Report enforcement approaches and resource configurations alongside results

## Implications

Leaderboard differences below 3 percentage points warrant skepticism without documented infrastructure matching. The authors conclude: "A few-point lead might signal a real capability gap—or it might just be a bigger VM."
