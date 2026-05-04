# Scaling Managed Agents: Decoupling the Brain from the Hands

**Published:** April 8, 2026
**Authors:** Lance Martin, Gabe Cemaj, and Michael Cohen
**Source:** https://www.anthropic.com/engineering/managed-agents

---

## Overview

Anthropic's Managed Agents service addresses a fundamental challenge in building long-running AI systems: how to design infrastructure that remains effective as underlying models improve. The solution involves decoupling the "brain" (Claude and its harness) from the "hands" (execution environments and tools).

## The Problem with Coupled Systems

Initially, Managed Agents combined all components—session logs, harness logic, and sandboxes—into single containers. This approach created several issues:

**Infrastructure brittleness:** The system adopted a "pet" model where failing containers meant lost sessions. Debugging required shell access, but this risked exposing user data.

**Assumption staleness:** As models improved, hardcoded assumptions became obsolete. For example, context-anxiety workarounds designed for earlier Claude versions proved unnecessary with newer models.

**Connectivity constraints:** Customers requesting integration with private networks faced difficult choices about network peering or running infrastructure themselves.

## The Solution: Decoupling Architecture

### Three Core Components

The redesign virtualizes agent infrastructure into three independent interfaces:

1. **Session** - An append-only event log existing outside the harness
2. **Harness** - The loop orchestrating Claude's interactions
3. **Sandbox** - Execution environments for code and file operations

Each can fail or be replaced independently without affecting others.

### Key Implementation Changes

**Stateless harnesses:** The harness no longer lives in containers. Instead, it treats sandboxes as tools via `execute(name, input) → string`. Failed harnesses restart via `wake(sessionId)`, recovering context from `getSession(id)`.

**Container-as-cattle:** Sandboxes are now interchangeable. If one fails, the harness treats it as a tool error and may provision a replacement.

**Security improvements:** Credentials are never exposed to generated code. Two patterns protect against prompt injection:
- Bundling authentication with resources during initialization
- Using secure vaults with MCP proxy servers for external tools

### Context Beyond the Context Window

Long-horizon tasks exceed Claude's context limits. Rather than making irreversible decisions about which information to keep, the session serves as a queryable context object outside the window. The `getEvents()` interface allows flexible retrieval—selecting positional slices, rewinding before specific moments, or rereading previous context.

## Performance and Scalability Benefits

**Dramatic latency improvements:** Decoupling eliminated forced container provisioning. Time-to-first-token (TTFT) improved roughly 60% at p50 and over 90% at p95 because inference starts immediately without waiting for unnecessary container setup.

**Multiple brains, multiple hands:** Many stateless harnesses can connect to different execution environments only when needed. This allows:
- Brains operating against resources in customer VPCs
- Single brains coordinating work across multiple sandboxes
- Brains delegating to other brains

## Design Philosophy

The architecture follows principles established by operating systems decades ago: virtualizing underlying components into stable abstractions. Just as `read()` works across hardware from the 1970s to today, Managed Agents' interfaces accommodate future harnesses, models, and tools.

The system remains "opinionated about interfaces, unopinionated about implementations," allowing Claude Code, task-specific harnesses, and future approaches to coexist within the same framework.

---

**Acknowledgements:** Nodir Turakulov, Jeremy Fox, and the Agents API team contributed to this work.
