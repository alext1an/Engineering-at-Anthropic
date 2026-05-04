# Raising the bar on SWE-bench Verified with Claude 3.5 Sonnet

**Published:** January 6, 2025
**Source:** https://www.anthropic.com/engineering/swe-bench-sonnet

---

## Overview

Anthropic's upgraded Claude 3.5 Sonnet achieved 49% on SWE-bench Verified, surpassing the previous state-of-the-art score of 45%. This blog post details the agent architecture and scaffolding built around the model to optimize performance on real-world software engineering tasks.

## What is SWE-bench?

SWE-bench is an evaluation benchmark measuring an AI model's ability to complete genuine software engineering tasks. It specifically assesses how models can resolve GitHub issues from popular open-source Python repositories by:

- Providing a prepared Python environment
- Supplying a repository checkout from before the issue was resolved
- Requiring the model to understand, modify, and test code
- Grading solutions against actual unit tests from the original pull request

## Agent Architecture Philosophy

Rather than constraining the model to rigid workflows, Anthropic's approach emphasizes flexibility. The agent includes:

- A minimal prompt outlining suggested steps
- A Bash Tool for executing commands
- An Edit Tool for viewing and modifying files
- Continued sampling until the model decides it's finished or reaches the 200k context limit

The design philosophy prioritizes giving "as much control as possible to the language model itself."

## Tool Design

### Bash Tool

The Bash Tool enables command execution with important contextual information embedded in the tool description, including details about escaping, internet access limitations, and background process handling.

### Edit Tool

The Edit Tool supports viewing, creating, and editing files with five commands: `view`, `create`, `str_replace`, `insert`, and `undo_edit`. Notably, the tool uses string replacement requiring exact matches to prevent errors. The implementation requires absolute paths to avoid relative path issues.

## Performance Results

| Model | SWE-bench Verified Score |
|-------|-------------------------|
| Claude 3.5 Sonnet (new) | 49% |
| Previous SOTA | 45% |
| Claude 3.5 Sonnet (old) | 33% |
| Claude 3 Opus | 22% |

## Key Improvements

The upgraded Claude 3.5 Sonnet demonstrates enhanced self-correction capabilities. Unlike older models that repeated mistakes, this version attempts multiple solution approaches and exhibits stronger reasoning for coding and mathematical tasks.

## Documented Challenges

1. **Duration and Cost:** Successful runs often required hundreds of turns exceeding 100k tokens, making evaluation expensive despite the model's persistence.

2. **Grading Complexity:** Environment setup issues and installation patch conflicts occasionally affected accuracy assessments, requiring careful systems troubleshooting.

3. **Hidden Tests:** Without visibility into test cases, models sometimes incorrectly assessed their success, particularly when applying surface-level rather than structural fixes.

4. **Multimodal Limitations:** The agent lacked capability to display files or referenced URLs visually, complicating debugging—especially for visualization-dependent tasks like Matplotlib problems.

## Conclusion

The new Claude 3.5 Sonnet achieved state-of-the-art SWE-bench performance using straightforward prompting with two general-purpose tools, suggesting significant room for developer optimization beyond this baseline implementation.
