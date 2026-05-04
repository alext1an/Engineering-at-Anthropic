# Module 10: Skills, Sandboxing, Permissions

**Time:** ~1.5 hours (≈45 min reading · ≈30 min hands-on · ≈15 min reflection)
**Builds on:** Module 03 (Tool Design), Module 07 (Memory)    **Feeds:** Module 14 (Production)

## Learning Objectives

- Design a SKILL.md that uses progressive disclosure correctly — name/description that triggers loading, then content that justifies the token cost.
- Explain what sandboxing covers that permission prompts don't, and why the 84% reduction in prompts is not simply security being weakened.
- Distinguish the three permission tiers in Auto Mode and predict which tier each type of operation falls into.
- Reason about the two-layer defense model (input probe + output classifier) and identify which attacks each layer addresses.

---

## 1. Concept Synthesis

### Three complementary boundaries

This module covers three mechanisms that, together, define what an agent can know (skills), do (sandboxing), and is allowed to do without human approval (permissions). They're architecturally distinct but designed to compose:

- **Skills:** Extend capability by giving agents procedural knowledge on demand
- **Sandboxing:** Restrict execution scope at the OS level regardless of what the agent wants to do
- **Auto Mode permissions:** Gate sensitive actions through behavioral classification without requiring manual approval for every operation

Together they implement the principle from the Building Effective Agents article: "appropriate guardrails" for autonomous systems. None of the three is sufficient alone.

### Skills: procedural knowledge with progressive disclosure

Agent Skills address a gap that tool definitions and system prompts can't fill: *procedural knowledge*. Claude might know that PDFs exist and that forms need to be filled, but it lacks the specific knowledge of how to programmatically extract form fields from a PDF, handle encoding edge cases, and write output correctly. A skill packages this knowledge.

The architecture is a directory with a `SKILL.md` file containing YAML frontmatter:

```yaml
---
name: pdf
description: Extracts form fields and content from PDF files. Use when working with PDF documents that need reading, form filling, or content extraction.
---

## When to use this skill
Use for any task involving PDF documents...

## Reading PDFs
[detailed instructions]

## Reference
- See `reference.md` for edge cases
- See `forms.md` for form field extraction
```

**Progressive disclosure in three levels:**

**Level 1 (always loaded):** Name and description are pre-loaded into the system prompt for every installed skill. The model uses these to decide relevance: small, cheap — just enough to know when to load more.

**Level 2 (loaded on relevance):** When the model determines the skill is relevant, it reads the full `SKILL.md`. This is the main instruction set.

**Level 3+ (loaded as needed):** Additional files bundled in the skill directory (`reference.md`, `forms.md`, scripts) load contextually. The model navigates them only when the specific content is needed.

The benefit: "the amount of context that can be bundled into a skill is effectively unbounded" because only the relevant portion reaches the model's context at any given time. This is the same just-in-time loading principle from Module 06, applied to capability packages rather than data.

The notebook shows the token comparison:
- Manual instructions in prompt: 5,000-10,000 tokens per request
- Skills (metadata only): minimal (just name/description)
- Skills (full load, when used): ~5,000 tokens when skill is triggered

For common skills that trigger on most tasks, this doesn't save much. For specialist skills that trigger rarely, it prevents constant token overhead.

**Code execution within skills.** Skills can include pre-written scripts that Claude runs rather than generates. The PDF skill includes a Python script that extracts form fields: "Claude can run this script without loading either the script or the PDF into context." The rationale is precise:

> Large language models excel at many tasks, but certain operations are better suited for traditional code execution. Sorting a list via token generation is far more expensive than simply running a sorting algorithm. Beyond efficiency concerns, many applications require the deterministic reliability that only code can provide.

This is the same principle as Programmatic Tool Calling from Module 04 — delegate computational work to code, not to model reasoning.

**Development discipline.** The article's guidance:
- Start with evaluation: identify where your agent struggles before writing skills
- Structure for scale: when SKILL.md gets large, split into referenced files
- Watch the name and description carefully: the model uses these for triggering, so vague descriptions cause under-triggering, over-specific descriptions cause missing valid use cases
- Ask Claude to help build the skill: "capture its successful approaches and common mistakes into reusable context and code within a skill"

**Security note:** Skills execute code and can connect to external networks. Install skills only from trusted sources; audit carefully before deploying, particularly: code dependencies, bundled resources, and instructions that direct Claude to external network sources.

*— Ch 11 (equipping-agents-for-the-real-world-with-agent-skills)*

### Sandboxing: OS-level execution boundaries

Anthropic's *Claude Code Sandboxing* (Oct 2025) introduces sandboxing that reduced permission prompts by **84%** while enhancing security. The reduction comes from moving from prompt-based permission to OS-level constraint: the agent simply *cannot* access or modify things outside its sandbox, so there's no need to ask for permission to do things it can't do anyway.

**Two isolation mechanisms:**

**Filesystem isolation:** Claude can only access or modify specific directories — typically the current working directory. Files outside this boundary are inaccessible, not just permission-gated. This prevents:
- A prompt-injected Claude from modifying system files
- An agent that wanders outside its project scope due to misunderstanding
- Code generated by the agent from accessing credentials or configuration files in parent directories

**Network isolation:** Claude can only connect to approved domains. Outbound connections flow through a Unix domain socket connected to a proxy server running *outside the sandbox*, which enforces domain allowlisting. This prevents:
- A prompt-injected Claude from exfiltrating sensitive information
- Claude Code downloading malware or unauthorized packages
- Generated code making unauthorized API calls

**What the sandbox covers:** Both mechanisms apply not just to Claude Code's direct interactions, but to "any scripts, programs, or subprocesses that are spawned by the command." When Claude generates and runs a script, the script runs inside the same sandbox. The model can't escape by delegating to a subprocess.

**Implementation:** Linux bubblewrap (Linux) and macOS seatbelt (macOS) — OS-level kernel primitives, not application-layer filtering. The sandbox is enforced by the kernel, not by code that the model could potentially circumvent.

**Web implementation:** Claude Code on the web adds credential isolation through a proxy service that handles all git interactions, so git credentials never appear in the agent's accessible context.

The 84% permission prompt reduction is worth unpacking. Users were approving ~93% of all permission prompts anyway (from the Auto Mode article). The sandbox eliminates the entire class of prompts for actions that are now simply impossible (out-of-scope file access, unapproved network access). What remains is the genuinely consequential set of decisions.

*— Ch 12 (claude-code-sandboxing)*

### Auto Mode: behavioral classification for consequential actions

Claude Code's traditional permission system prompts users before every operation. Auto Mode (*Claude Code Auto Mode*, Mar 2026) automates most approval decisions to address "approval fatigue" — users were approving 93% of prompts, often without reading them carefully.

The solution has two complementary layers:

**Layer 1 — Input probe (server-side):** Before tool outputs (file reads, web requests, shell command results) reach the agent's context, a prompt-injection probe screens them. When suspicious content is detected, a warning is added to help the agent recognize potential manipulation. This addresses the threat at the point of entry: malicious instructions in read files or web content can't silently reach the agent's reasoning.

**Layer 2 — Output classifier (using Claude Sonnet 4.6):** Before each action is executed, a transcript classifier evaluates it. Two-stage process:
- Stage 1: Rapid filter — flags only suspicious commands
- Stage 2: Detailed reasoning applied *only to flagged commands* — reduces false positives from 8.5% to 0.4%

The classifier is "reasoning-blind": it deliberately excludes the agent's own explanations and tool outputs. It evaluates *what the user intended* vs. *what the agent is about to do*, not what the agent says about why. This prevents self-justification — an agent can't talk its way past the classifier by explaining its reasoning.

**Three permission tiers:**

**Tier 1 (always approved):** Safe, read-only operations — file reads, searches, code navigation. No classifier evaluation needed.

**Tier 2 (no classifier review):** In-project file edits. These are auditable through version control and reversible. The operator (user) implicitly approved these when setting up the project.

**Tier 3 (full classifier evaluation):** Shell commands, external tool calls, and out-of-project operations. These have higher irreversibility risk and wider blast radius.

**Test results:** On 10,000 real internal tool calls:
- False positive rate (blocked legitimate actions): 0.4%
- False negative rate (allowed genuine overeager actions): 17%

The 17% false negative rate is the known limitation — the system catches 83% of genuinely problematic actions, not all of them. This is explicitly chosen over higher false positive rates, which would undermine usability.

**Escalation logic:** Three consecutive denials or twenty total denials trigger human escalation. When the classifier blocks an action, the agent receives denial feedback and attempts alternatives — it doesn't simply halt.

**The threat model addresses four scenarios:**
1. Overeager behavior: well-intentioned but unauthorized actions
2. Honest mistakes: agent misunderstands scope
3. Prompt injection attacks: malicious external content redirects the agent
4. Model misalignment: model pursues goals other than user intent

Real incidents from testing included: deleting remote branches, uploading authentication tokens, attempting production database migrations. These are the kinds of consequential, hard-to-reverse actions the classifier targets.

*— Ch 22 (claude-code-auto-mode)*

### How the three mechanisms compose

The three mechanisms operate at different layers and target different failure modes:

| Mechanism | Layer | Targets |
|---|---|---|
| Sandboxing | OS / execution | What the agent *can* do, regardless of intent |
| Input probe | Pre-context | External content injecting instructions |
| Output classifier | Pre-action | Agent behavior that exceeds authorized scope |

An agent that has been prompt-injected via a malicious file:
- Sandboxing prevents the injected instructions from accessing out-of-scope files
- Input probe flags the malicious content before it reaches the agent's reasoning
- Output classifier blocks any actions the injected instructions attempt to trigger

All three layers must work together because each addresses a different attack surface. A sandboxed agent without input probing can still be manipulated through in-scope content. An agent with input probing but no sandboxing can escape file access scope. An agent with both but no output classifier can still execute overeager actions that aren't injections.

*— Ch 11, 12, 22 combined*

---

## 2. Key Questions (the depth check)

<details>
<summary><b>Q1.</b> A skill's name and description are "always loaded" into the system prompt. What happens to token budget when an agent has 20 skills installed?</summary>

Each skill's name and description adds to the system prompt. For 20 skills with descriptions averaging 30 tokens each, that's 600 tokens of constant overhead per turn — before any skill content is loaded. This is modest for most contexts but meaningful for agents operating near context limits or with many tools. The design assumes the name/description is much cheaper than the full skill content. If skill descriptions grow long (trying to over-specify trigger conditions), the always-loaded overhead becomes a problem. The solution: keep descriptions crisp (what the skill does and when to use it), not comprehensive.
</details>

<details>
<summary><b>Q2.</b> The sandbox covers subprocesses spawned by the agent. Why is this coverage critical, and what would break without it?</summary>

Without subprocess coverage, the sandbox would be bypassable: Claude generates a script, runs it, and the script (running outside the sandbox) accesses files or networks the sandbox prohibits. Every code execution capability would be an escape hatch. By covering the full process tree, the sandbox ensures that code Claude generates is subject to the same restrictions as Claude's direct actions. This is the difference between a sandbox that's a security guarantee vs. a sandbox that's security theater with a well-known bypass.
</details>

<details>
<summary><b>Q3.</b> Auto Mode's output classifier is "reasoning-blind" — it excludes the agent's explanations. Why does this specifically matter for security?</summary>

If the classifier could be influenced by the agent's reasoning ("I'm doing this because the user said..."), an attacker who controls the agent's reasoning could potentially justify any action. A prompt-injected agent could generate plausible-sounding justifications for its injected goals. By ignoring the agent's reasoning and focusing only on user intent + executable commands, the classifier evaluates the *action itself* against the *authorized scope*, not against the agent's potentially manipulated explanation. This makes the classifier resistant to reasoning-based circumvention attempts.
</details>

<details>
<summary><b>Q4.</b> The auto mode test reports 17% false negatives on "genuine overeager actions." What does this mean in practice, and why might 17% be acceptable?</summary>

17% of genuinely problematic overeager actions get approved. In absolute terms: on 10,000 tool calls with a realistic rate of overeager actions, 17% of those specific bad actions slip through. This is acceptable because: (a) the alternative (0% false negatives) requires accepting many more false positives, which breaks usability; (b) the 0.4% false positive rate means legitimate work is almost never blocked; (c) the classifier operates alongside sandboxing, which limits the blast radius of any overeager action that does slip through. Defense-in-depth means no single layer needs to be perfect.
</details>

<details>
<summary><b>Q5.</b> Skills include code that Claude runs as tools. How does this differ architecturally from the Model Context Protocol (MCP) tool pattern from Module 03?</summary>

MCP tools are *external* — the tool definition is registered by a server, and Claude calls through the MCP protocol. The tool executes on the server. Skills are *local and bundled* — the code is bundled in the skill directory and runs in Claude's code execution environment. The difference: MCP tools can connect to external services, have their own authorization, and run infrastructure you manage. Skill code runs locally with the same access constraints as Claude itself. MCP is appropriate for accessing external systems (databases, APIs, services); skills are appropriate for local procedural work (parsing files, formatting output, running deterministic algorithms).
</details>

<details>
<summary><b>Q6.</b> An agent reads a web page and extracts a summary for later use. The web page contains malicious instructions embedded in invisible text. Which of the three mechanisms handles this threat, and which do not?</summary>

**Input probe (Layer 1)** is designed to catch this: it screens tool outputs (including web request results) before they reach the agent's context, and adds warnings about suspicious content. **Sandboxing** does not directly help — the content is accessed within the approved network scope (normal web browsing), so the sandbox doesn't block it. **Output classifier** is a fallback — if the injected instructions cause the agent to attempt a consequential action, the classifier may block that action. But this is reactive, not preventive. The input probe is the primary defense for this attack vector.
</details>

---

## 3. Hands-On

**Notebook:**
- [`claude-cookbooks/skills/notebooks/01_skills_introduction.ipynb`](../claude-cookbooks/skills/notebooks/01_skills_introduction.ipynb)

**Run as-is.**

Focus on:
- **Section 2 (Understanding Skills):** The conceptual overview of progressive disclosure levels. Note the token comparison table: manual instructions (5,000-10,000 tokens/request) vs. skills metadata (minimal).
- **Section 3 (Discovering Available Skills):** The `client.beta.skills.list()` call shows available skill IDs and their descriptions. Read a few descriptions — notice how they're written (specific trigger conditions, not vague capability labels).
- **Section 4 (Quick Start: Excel):** Watch a skill being triggered in practice. Note that the model reads the skill before executing.

**One modification (≈15 min): write a minimal custom skill.**

Create a file `my_skill/SKILL.md` with:
```yaml
---
name: word_counter
description: Count words, sentences, and paragraphs in text. Use when asked to analyze text statistics or length.
---

## Word Counter Skill

Count the following in any provided text:
1. Total words (split by whitespace)
2. Total sentences (split by periods, question marks, exclamation points)
3. Total paragraphs (split by double newlines)

Return results as:
Words: [count]
Sentences: [count]  
Paragraphs: [count]
```

Then ask Claude (via the API with code execution enabled): "Analyze this text: [any paragraph you choose]"

Without the skill: the model produces a word count through reasoning (potentially inconsistent). With the skill loaded: the model follows the specific format and counting rules. Compare the outputs.

**What to record in your notes:**
- The token footprint at each progressive disclosure level for a skill you observed.
- One thing about how the skill description triggered loading that surprised you.
- Your assessment: for the word counter, does the skill improve consistency over baseline, and why?

---

## 4. Reflection

1. **Progressive disclosure assumes the model will correctly use the name/description to decide relevance.** This assumption can fail in both directions: the model loads skills unnecessarily (poor description), or fails to load them when needed (description doesn't match user phrasing). How would you test whether a skill's description correctly triggers in appropriate vs. inappropriate situations? What's the eval design?

2. **Sandboxing reduces permission prompts by 84% but maintains security.** The logic is: sandbox eliminates the need to ask about actions that are now impossible. But this means users see fewer prompts — could this actually make the remaining 16% of prompts *more* dangerous because users pay less attention after so many approvals? Or does it make them more valuable because each one is actually consequential?

3. **The Auto Mode classifier deliberately has 17% false negatives.** This was a design choice. What would the failure mode look like if you pushed for 0% false negatives — and at what false positive rate would you give up on auto mode entirely and revert to manual approval?

---

## 5. Key Takeaways

- **Progressive disclosure manages the tension between capability breadth and context budget.** Name/description (always loaded) costs near nothing. Full skill content (loaded on relevance) costs tokens only when useful. Bundled files (loaded as needed) are effectively unbounded.
- **Sandboxing covers the full process tree.** OS-level primitives (bubblewrap/seatbelt) constrain what the agent's generated code can do, not just what the model's API calls can do. The 84% reduction in permission prompts comes from eliminating prompts for actions that are now impossible.
- **Auto Mode's two layers address different attack surfaces.** Input probe catches external content injection before it reaches reasoning. Output classifier catches consequential actions before they execute — using only user intent vs. action, not the agent's potentially manipulated reasoning.
- **Three tiers match action risk to review overhead.** Read-only operations need no approval; in-project edits are auditable by git; shell commands and external operations get classifier evaluation.
- **Skills code execution is local; MCP tools are external.** The distinction determines trust model, access scope, and appropriate use cases.
