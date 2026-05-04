# Best Practices for Claude Code

**Published:** April 18, 2025
**Source:** https://www.anthropic.com/engineering/claude-code-best-practices (redirects to https://code.claude.com/docs/en/best-practices)

> Tips and patterns for getting the most out of Claude Code, from configuring your environment to scaling across parallel sessions.

Claude Code is an agentic coding environment. Unlike a chatbot that answers questions and waits, Claude Code can read your files, run commands, make changes, and autonomously work through problems while you watch, redirect, or step away entirely.

This changes how you work. Instead of writing code yourself and asking Claude to review it, you describe what you want and Claude figures out how to build it. Claude explores, plans, and implements.

But this autonomy still comes with a learning curve. Claude works within certain constraints you need to understand.

This guide covers patterns that have proven effective across Anthropic's internal teams and for engineers using Claude Code across various codebases, languages, and environments.

---

Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills.

Claude's context window holds your entire conversation, including every message, every file Claude reads, and every command output. However, this can fill up fast. A single debugging session or codebase exploration might generate and consume tens of thousands of tokens.

This matters since LLM performance degrades as context fills. When the context window is getting full, Claude may start "forgetting" earlier instructions or making more mistakes. The context window is the most important resource to manage.

---

## Give Claude a way to verify its work

> Include tests, screenshots, or expected outputs so Claude can check itself. This is the single highest-leverage thing you can do.

Claude performs dramatically better when it can verify its own work, like run tests, compare screenshots, and validate outputs.

Without clear success criteria, it might produce something that looks right but actually doesn't work. You become the only feedback loop, and every mistake requires your attention.

| Strategy | Before | After |
|---|---|---|
| **Provide verification criteria** | *"implement a function that validates email addresses"* | *"write a validateEmail function. example test cases: user@example.com is true, invalid is false, user@.com is false. run the tests after implementing"* |
| **Verify UI changes visually** | *"make the dashboard look better"* | *"[paste screenshot] implement this design. take a screenshot of the result and compare it to the original. list differences and fix them"* |
| **Address root causes, not symptoms** | *"the build is failing"* | *"the build fails with this error: [paste error]. fix it and verify the build succeeds. address the root cause, don't suppress the error"* |

UI changes can be verified using the Claude in Chrome extension. It opens new tabs in your browser, tests the UI, and iterates until the code works.

Your verification can also be a test suite, a linter, or a Bash command that checks output. Invest in making your verification rock-solid.

---

## Explore first, then plan, then code

> Separate research and planning from implementation to avoid solving the wrong problem.

Letting Claude jump straight to coding can produce code that solves the wrong problem. Use plan mode to separate exploration from execution.

The recommended workflow has four phases:

1. **Explore** — Enter plan mode. Claude reads files and answers questions without making changes.
2. **Plan** — Ask Claude to create a detailed implementation plan. Press `Ctrl+G` to open the plan in your text editor for direct editing before Claude proceeds.
3. **Implement** — Switch out of plan mode and let Claude code, verifying against its plan.
4. **Commit** — Ask Claude to commit with a descriptive message and create a PR.

Plan mode is useful, but also adds overhead. For tasks where the scope is clear and the fix is small (like fixing a typo, adding a log line, or renaming a variable) ask Claude to do it directly.

---

## Provide specific context in your prompts

> The more precise your instructions, the fewer corrections you'll need.

Claude can infer intent, but it can't read your mind. Reference specific files, mention constraints, and point to example patterns.

| Strategy | Before | After |
|---|---|---|
| **Scope the task.** | *"add tests for foo.py"* | *"write a test for foo.py covering the edge case where the user is logged out. avoid mocks."* |
| **Point to sources.** | *"why does ExecutionFactory have such a weird api?"* | *"look through ExecutionFactory's git history and summarize how its api came to be"* |
| **Reference existing patterns.** | *"add a calendar widget"* | *"look at how existing widgets are implemented... HotDogWidget.php is a good example. follow the pattern..."* |
| **Describe the symptom.** | *"fix the login bug"* | *"users report that login fails after session timeout. check the auth flow in src/auth/, especially token refresh. write a failing test that reproduces the issue, then fix it"* |

### Provide rich content

- **Reference files with `@`** instead of describing where code lives.
- **Paste images directly** via copy/paste or drag and drop.
- **Give URLs** for documentation and API references.
- **Pipe in data** by running `cat error.log | claude`.
- **Let Claude fetch what it needs** via Bash commands, MCP tools, or by reading files.

---

## Configure your environment

### Write an effective CLAUDE.md

> Run `/init` to generate a starter CLAUDE.md file based on your current project structure, then refine over time.

CLAUDE.md is a special file that Claude reads at the start of every conversation. Include Bash commands, code style, and workflow rules.

```markdown
# Code style
- Use ES modules (import/export) syntax, not CommonJS (require)
- Destructure imports when possible (eg. import { foo } from 'bar')

# Workflow
- Be sure to typecheck when you're done making a series of code changes
- Prefer running single tests, and not the whole test suite, for performance
```

Keep it concise. For each line, ask: *"Would removing this cause Claude to make mistakes?"* If not, cut it.

| ✅ Include | ❌ Exclude |
|---|---|
| Bash commands Claude can't guess | Anything Claude can figure out by reading code |
| Code style rules that differ from defaults | Standard language conventions Claude already knows |
| Testing instructions and preferred test runners | Detailed API documentation (link to docs instead) |
| Repository etiquette (branch naming, PR conventions) | Information that changes frequently |
| Architectural decisions specific to your project | Long explanations or tutorials |
| Developer environment quirks (required env vars) | File-by-file descriptions of the codebase |
| Common gotchas or non-obvious behaviors | Self-evident practices like "write clean code" |

CLAUDE.md placement options:
- **Home folder (`~/.claude/CLAUDE.md`)**: applies to all sessions
- **Project root (`./CLAUDE.md`)**: check into git to share with your team
- **Project root (`./CLAUDE.local.md`)**: personal project-specific notes; add to `.gitignore`
- **Parent directories**: useful for monorepos
- **Child directories**: pulled in on demand

### Configure permissions

By default, Claude Code requests permission for actions that might modify your system. Three ways to reduce interruptions:

- **Auto mode**: a separate classifier model reviews commands and blocks only what looks risky
- **Permission allowlists**: permit specific tools you know are safe
- **Sandboxing**: enable OS-level isolation that restricts filesystem and network access

### Use CLI tools

CLI tools are the most context-efficient way to interact with external services. If you use GitHub, install the `gh` CLI.

### Connect MCP servers

Run `claude mcp add` to connect external tools like Notion, Figma, or your database.

### Set up hooks

Use hooks for actions that must happen every time with zero exceptions. Unlike CLAUDE.md instructions which are advisory, hooks are deterministic.

### Create skills

Create `SKILL.md` files in `.claude/skills/` to give Claude domain knowledge and reusable workflows.

```markdown
---
name: api-conventions
description: REST API design conventions for our services
---
# API Conventions
- Use kebab-case for URL paths
- Use camelCase for JSON properties
- Always include pagination for list endpoints
- Version APIs in the URL path (/v1/, /v2/)
```

### Create custom subagents

Define specialized assistants in `.claude/agents/` that Claude can delegate to for isolated tasks.

### Install plugins

Run `/plugin` to browse the marketplace.

---

## Communicate effectively

### Ask codebase questions

Ask Claude questions you'd ask a senior engineer:
- How does logging work?
- How do I make a new API endpoint?
- What does `async move { ... }` do on line 134 of `foo.rs`?
- What edge cases does `CustomerOnboardingFlowImpl` handle?

### Let Claude interview you

For larger features, have Claude interview you first.

```text
I want to build [brief description]. Interview me in detail using the AskUserQuestion tool.

Ask about technical implementation, UI/UX, edge cases, concerns, and tradeoffs. Don't ask obvious questions, dig into the hard parts I might not have considered.

Keep interviewing until we've covered everything, then write a complete spec to SPEC.md.
```

---

## Manage your session

### Course-correct early and often

- **`Esc`**: stop Claude mid-action
- **`Esc + Esc` or `/rewind`**: open the rewind menu and restore previous state
- **`"Undo that"`**: have Claude revert its changes
- **`/clear`**: reset context between unrelated tasks

If you've corrected Claude more than twice on the same issue in one session, the context is cluttered with failed approaches.

### Manage context aggressively

- Use `/clear` frequently between tasks
- Use `/compact <instructions>` to compact with focus
- Use `Esc + Esc` or `/rewind` to summarize from a checkpoint
- Use `/btw` for quick questions that shouldn't enter context

### Use subagents for investigation

Subagents run in separate context windows and report back summaries.

### Rewind with checkpoints

Every action Claude makes creates a checkpoint. Double-tap `Escape` or run `/rewind` to open the rewind menu.

### Resume conversations

Run `claude --continue` to pick up the most recent session, or `claude --resume` to choose from a list.

---

## Automate and scale

### Run non-interactive mode

```bash
# One-off queries
claude -p "Explain what this project does"

# Structured output for scripts
claude -p "List all API endpoints" --output-format json

# Streaming for real-time processing
claude -p "Analyze this log file" --output-format stream-json
```

### Run multiple Claude sessions

- **Worktrees**: separate CLI sessions in isolated git checkouts
- **Desktop app**: manage multiple local sessions visually
- **Claude Code on the web**: cloud-managed sessions in isolated VMs
- **Agent teams**: automated coordination of multiple sessions

Writer/Reviewer pattern: have one Claude write, another review with fresh context.

### Fan out across files

```bash
for file in $(cat files.txt); do
  claude -p "Migrate $file from React to Vue. Return OK or FAIL." \
    --allowedTools "Edit,Bash(git commit *)"
done
```

### Run autonomously with auto mode

```bash
claude --permission-mode auto -p "fix all lint errors"
```

---

## Avoid common failure patterns

- **The kitchen sink session.** Fix: `/clear` between unrelated tasks.
- **Correcting over and over.** Fix: After two failed corrections, `/clear` and write a better initial prompt.
- **The over-specified CLAUDE.md.** Fix: Ruthlessly prune.
- **The trust-then-verify gap.** Fix: Always provide verification.
- **The infinite exploration.** Fix: Scope investigations narrowly or use subagents.

---

## Develop your intuition

The patterns in this guide aren't set in stone. Pay attention to what works. When Claude produces great output, notice what you did. When Claude struggles, ask why.
