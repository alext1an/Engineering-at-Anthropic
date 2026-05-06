Claude code 101
=================
prompt -> (agentic loop) ->gather context ->Task action -> Verify results -> Done
Tools backbone
Permission
Plan mode for complex changes and doing a safe code review

The Workflow
The explore -> Plan -> Code -> Commit

Context Management
/compact
use subagent to save context window
use skills or shut down unnecessary mcps to save context window
detailed prompt always save context window in the long run, cuz it can help the agent to understand the task better and avoid unnecessary self-exploration

The CLAUDE.md File
use /init to create one
let cc put what u want to remember in this file
use @ to add reference docs
better not use it when starting off a project

Subagent
keep context window clean

Skill

MCP
MCP connects Claude Code to your external tools and data sources. 

Hooks
Hooks give you deterministic control over Claude Code's behavior.
