# Desktop Extensions: One-click MCP server installation for Claude Desktop

**Published:** Jun 26, 2025
**Source:** https://www.anthropic.com/engineering/desktop-extensions

---

## Overview

Anthropic has introduced Desktop Extensions (`.mcpb` files), a new packaging format that dramatically simplifies how users install Model Context Protocol (MCP) servers in Claude Desktop. Instead of requiring terminal commands and manual configuration, users can now install servers with a single click.

## The Problem Being Solved

Previously, MCP server installation required:
- Developer tools like Node.js or Python
- Manual editing of JSON configuration files
- Resolving dependency conflicts
- Searching GitHub to discover servers
- Manual reinstallation for updates

These barriers made powerful local MCP servers inaccessible to non-technical users.

## How Desktop Extensions Work

A Desktop Extension bundles an entire MCP server—including all dependencies—into a single installable package. The installation process is simplified to:

1. Download a `.mcpb` file
2. Double-click to open with Claude Desktop
3. Click "Install"

### Architecture

Desktop Extensions are ZIP archives containing:
- **manifest.json** (required): Extension metadata and configuration
- **server/**: MCP server implementation files
- **dependencies/**: Bundled packages and libraries
- **icon.png** (optional): Visual representation

Claude Desktop handles complexity by:
- Shipping Node.js built-in, eliminating external runtime requirements
- Providing automatic updates
- Storing sensitive data securely in the OS keychain

## The Manifest File

The only required file is `manifest.json`. A minimal example includes:

```json
{
  "mcpb_version": "0.1",
  "name": "my-extension",
  "version": "1.0.0",
  "description": "Extension description",
  "author": { "name": "Author Name" },
  "server": {
    "type": "node",
    "entry_point": "server/index.js",
    "mcp_config": {
      "command": "node",
      "args": ["${__dirname}/server/index.js"]
    }
  }
}
```

Key features include:
- **Template literals** like `${__dirname}` for installation paths
- **User configuration** with secure storage for sensitive values
- **Cross-platform support** for Windows, macOS, and Linux
- **Feature declaration** for tools and prompts

## Building Your First Extension

The process involves four steps:

**Step 1: Create the manifest**
```
npx @anthropic-ai/mcpb init
```

**Step 2: Handle user configuration** by declaring required inputs in the manifest's `user_config` section

**Step 3: Package the extension**
```
npx @anthropic-ai/mcpb pack
```

**Step 4: Test locally** by dragging the `.mcpb` file into Claude Desktop's Settings

## Advanced Features

- **Cross-platform configuration**: Platform-specific command overrides and environment variables
- **Dynamic template variables**: Runtime substitution of paths and user-provided values
- **Feature declarations**: Help users understand capabilities upfront

## Extension Directory

Anthropic is launching a curated directory of extensions built into Claude Desktop, enabling users to browse, search, and install extensions with one click.

## Open Ecosystem Commitment

The entire Desktop Extension specification and toolchain are being open-sourced, including:
- Complete MCPB specification
- Packaging and validation tools
- Reference implementation code
- TypeScript types and schemas

This allows "any AI desktop application" to support the format, not just Claude.

## Security and Enterprise Support

**For users:**
- Sensitive data stored in OS keychain
- Automatic updates
- Audit trail of installed extensions

**For enterprises:**
- Group Policy (Windows) and MDM (macOS) support
- Pre-installation of approved extensions
- Extension blocklists
- Private extension directories

## Getting Started

Developers can begin immediately:

```
npm install -g @anthropic-ai/mcpb
mcpb init
mcpb pack
```

Extensions are then ready for submission through the official form or local testing.

## Conclusion

Desktop Extensions represent a paradigm shift in accessibility, removing installation friction and making "powerful MCP servers accessible to everyone." The article notes that internally, Anthropic has experimented with experimental servers, including one connecting Claude to a GameBoy emulator running Super Mario Land.
