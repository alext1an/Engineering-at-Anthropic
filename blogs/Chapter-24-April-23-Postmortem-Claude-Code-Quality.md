# An Update on Recent Claude Code Quality Reports

**Published:** April 23, 2026
**Source:** https://www.anthropic.com/engineering/april-23-postmortem

## Summary

Anthropic identified and resolved three separate issues affecting Claude Code quality over the past month. All problems were fixed by April 20, 2026 (v2.1.116), and the company reset usage limits for all subscribers as compensation.

## The Three Issues

**1. Reasoning Effort Default Change (March 4 – April 7)**

The team adjusted Claude Code's default reasoning effort from "high" to "medium" to address extremely long response times that made the interface appear frozen. However, user feedback revealed this was the wrong tradeoff. Users preferred higher intelligence with the option to manually select lower effort for simpler tasks. The change was reverted on April 7.

**2. Caching Bug Causing Memory Loss (March 26 – April 10)**

A prompt caching optimization intended to reduce latency for idle sessions contained a critical bug. Instead of clearing old thinking sections once, the code repeatedly cleared reasoning history on every turn. This caused Claude to appear "forgetful and repetitive," losing context about its own previous decisions. The issue compounded when Claude was executing tool calls, progressively stripping away reasoning blocks. This also triggered more cache misses, which explains reports of faster usage limit depletion.

**3. Verbosity-Reduction Prompt (April 16 – April 20)**

Anthropic added system prompt instructions to reduce output: "keep text between tool calls to ≤25 words. Keep final responses to ≤100 words unless the task requires more detail." Despite internal testing showing no regressions, broader evaluations revealed a 3% intelligence drop. The change was immediately reverted.

## Detection Challenges

The staggered rollout schedule meant these issues affected different user segments at different times, creating an appearance of broad, inconsistent degradation. Initial internal testing and evaluations didn't reproduce the problems, delaying identification.

## Preventive Measures

Going forward, Anthropic will:

- Ensure more internal staff use the public Claude Code build
- Improve Code Review tooling with multi-repository context support
- Implement broader per-model evaluations for all system prompt changes
- Add stricter oversight and audit tooling for prompt modifications
- Require soak periods and gradual rollouts for intelligence-affecting changes
- Create clearer guidance ensuring model-specific changes target only intended versions
