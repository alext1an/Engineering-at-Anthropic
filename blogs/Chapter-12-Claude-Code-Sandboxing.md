# Beyond Permission Prompts: Making Claude Code More Secure and Autonomous

**Published:** October 20, 2025
**Authors:** David Dworken and Oliver Weller-Davies, with contributions from Meaghan Choi, Catherine Wu, Molly Vorwerck, Alex Isken, Kier Bradwell, and Kevin Garcia
**Source:** https://www.anthropic.com/engineering/claude-code-sandboxing

---

## Overview

Anthropic announced two new sandboxing features for Claude Code designed to enhance both security and developer autonomy. According to the announcement, sandboxing "safely reduces permission prompts by 84%" in internal usage.

## The Challenge

Claude Code grants the AI system access to codebases and files to write, test, and debug code. This capability introduces security risks, particularly regarding prompt injection attacks. The traditional permission-based model requires constant approval for modifications and commands, leading to "approval fatigue" where users may approve actions without proper scrutiny.

## Sandboxing Solution

The security approach implements two complementary boundaries:

**Filesystem Isolation**: Restricts Claude's access to specific directories, preventing modification of sensitive system files even if the AI is compromised.

**Network Isolation**: Limits connections to approved servers only, blocking data exfiltration and malware downloads. The implementation notes that "effective sandboxing requires both" isolation types for genuine protection.

## Technical Implementation

The sandboxing infrastructure leverages OS-level primitives including Linux bubblewrap and macOS seatbelt to enforce restrictions at the operating system level, covering direct interactions plus spawned subprocesses.

### Sandboxed Bash Tool

A new runtime (available as beta research preview and open source) allows developers to define accessible directories and network hosts. Claude executes commands within these defined limits while maintaining autonomy. Attempts to access restricted resources trigger user notifications.

### Claude Code on the Web

Cloud-based execution isolates each session in a secure sandbox where sensitive credentials never coexist with Claude Code. A custom proxy service manages git interactions, using scoped credentials and validating all git commands before authentication token attachment.

## Getting Started

Users can activate sandboxing by running `/sandbox` in Claude and consulting documentation, or visit claude.com/code to try cloud-based Claude Code.
