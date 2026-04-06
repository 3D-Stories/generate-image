---
name: setup
description: Set up the Replicate MCP server for image generation. Use when user says "setup replicate", "configure image generation", or when generate-image skill reports missing MCP.
---

# Replicate MCP Setup

Check whether you have access to any tools starting with `mcp__replicate__` (check your available tool list).

## If Replicate MCP tools are available

1. Confirm: "Replicate MCP is configured."
2. Run a quick connectivity test by calling `mcp__replicate__get_account` to verify the API token works.
3. If the test succeeds, report: "Replicate MCP is ready. You can now generate images by asking Claude to create any visual asset."
4. If the test fails, report the error and suggest the user check their `REPLICATE_API_TOKEN`.

## If no Replicate MCP tools are found

Tell the user:

**The Replicate MCP server is not installed.** Follow these steps:

### Step 1: Get a Replicate API Token

1. Sign up or log in at [replicate.com](https://replicate.com)
2. Go to **Account > API Tokens**
3. Create a new token and copy it

### Step 2: Set the API Token

Add to your shell profile (`~/.bashrc`, `~/.zshrc`, or equivalent):

```bash
export REPLICATE_API_TOKEN="r8_your_token_here"
```

Then reload your shell or run `source ~/.bashrc`.

**Security:** Store your token in an environment variable or shell profile. Never commit it to version control.

### Step 3: Add the Replicate MCP Server

```bash
claude mcp add replicate -- npx -y replicate-mcp@latest
```

### Step 4: Verify

Restart Claude Code (or start a new session), then run `/generate-image:setup` again to confirm everything works.
