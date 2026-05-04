# Claude Code Auto Mode: A Safer Way to Skip Permissions

**Published:** March 25, 2026
**Source:** https://www.anthropic.com/engineering/claude-code-auto-mode

## Overview

Anthropic introduced Claude Code auto mode, a new permission system that automates approval decisions to reduce user friction while maintaining safety. The feature addresses "approval fatigue"—the tendency for users to stop carefully reviewing requests after repeated prompts.

## The Problem

Claude Code traditionally requires user approval before executing commands or modifying files. While this protects safety, users approve 93% of these prompts anyway. This creates a middle ground: manual review causes fatigue, while the `--dangerously-skip-permissions` flag offers zero protection.

## How Auto Mode Works

The system employs two defensive layers:

**Input Layer:** A server-side prompt-injection probe screens tool outputs (file reads, web requests, shell commands) before they reach the agent's context. When suspicious content appears, it adds warnings to help the agent recognize potential manipulation.

**Output Layer:** A transcript classifier (using Sonnet 4.6) evaluates each action before execution using a two-stage process. Stage 1 performs rapid filtering, flagging only suspicious commands. Stage 2 applies detailed reasoning exclusively to flagged actions, reducing false positives from 8.5% to 0.4%.

## Threat Model

The system addresses four scenarios: overeager behavior (well-intentioned but unauthorized actions), honest mistakes, prompt injection attacks, and model misalignment. Real incidents from Anthropic's testing included deleting remote branches, uploading authentication tokens, and attempting production database migrations.

## Permission Tiers

**Tier 1:** Safe, read-only tools (file reads, searches, code navigation) bypass approval entirely.

**Tier 2:** In-project file edits proceed without classifier review, remaining auditable through version control.

**Tier 3:** Shell commands, external tool calls, and out-of-project operations receive full classifier evaluation.

## Results

Testing on 10,000 real internal tool calls showed the complete pipeline achieved 0.4% false positive rates while maintaining 17% false negatives on genuine overeager actions—representing meaningful safety improvement over complete automation.

## Key Design Choices

The classifier deliberately excludes the agent's own explanations and tool outputs to prevent self-justification. It focuses only on user intent and executable commands. This "reasoning-blind" approach complements monitoring systems that analyze the agent's internal logic.

When the classifier blocks an action, the agent receives denial feedback and attempts alternative approaches rather than halting entirely. Three consecutive denials or twenty total denials trigger human escalation.
