# generate-image

A Claude Code plugin for generating images using AI models on [Replicate](https://replicate.com). Supports Nano Banana 2 (high quality) and FLUX Schnell (fast/cheap), with automatic background removal for transparent PNGs.

## How It Works

The plugin provides two skills that instruct Claude how to generate images via the [Replicate MCP server](https://github.com/replicate/replicate-mcp). When you ask Claude to create an image, the generate-image skill selects the right model, constructs a prompt, calls Replicate's API through MCP tools, and downloads the result. For transparent images (icons, logos, illustrations), it automatically runs a two-step pipeline: generate on white background, then remove the background.

```
You → "make an icon of a gold star"
  → Claude (generate-image skill)
    → Replicate MCP (Nano Banana 2)
      → Image generated
    → Replicate MCP (recraft bg removal)
      → Transparent PNG
  → Saved to your project
```

## Prerequisites

- [Claude Code](https://claude.ai/code) installed
- **Node.js v18+** with npm and npx — required by the Replicate MCP server
  - macOS: `brew install node`
  - Ubuntu/Debian: install via [NodeSource](https://github.com/nodesource/distributions) (the default `apt` package is outdated)
  - Or use [nvm](https://github.com/nvm-sh/nvm) on any platform
- A [Replicate](https://replicate.com) account with an API token (pay-as-you-go, ~$0.003–$0.04 per image)

## Installation

```bash
claude plugin install generate-image@generate-image
```

Then run the setup skill to configure the Replicate MCP server:

```
/generate-image:setup
```

The setup skill will:
1. Check your Node.js/npm/npx installation (and help install if missing)
2. Register the Replicate MCP server with Claude Code (using the absolute path to npx — prevents PATH issues)
3. Walk you through creating and configuring your API token
4. Verify everything works

## Usage

Ask Claude to generate images naturally:

- "Generate an icon of a golden trophy for my app"
- "Create a hero image for the landing page — a mountain landscape at sunset"
- "Remove the background from logo.png"
- "Generate a set of 5 category icons matching my style guide"
- "Make a favicon for my website"

The skill handles model selection, prompt engineering, background removal, and file management automatically.

## Skills

| Skill | Description |
|-------|-------------|
| `/generate-image:generate-image` | Generate images — model selection, prompt construction, bg removal, file saving |
| `/generate-image:setup` | Detect and install prerequisites, register MCP, configure API token |

## Models

| Model | Best For | Cost | Speed |
|-------|----------|------|-------|
| **Nano Banana 2** | Icons, illustrations, complex scenes, text in images | ~$0.04/image | ~20s |
| **FLUX Schnell** | Quick drafts, bulk generation | ~$0.003/image | ~0.5s |
| **Background Remover** (recraft) | Transparent PNGs from any image | ~$0.01/image | ~5s |

Nano Banana 2 is the default for production assets. FLUX Schnell is used when cost or speed matters more than quality.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Replicate MCP not detected" | Run `/generate-image:setup` — it detects and fixes the issue |
| "npx: command not found" in MCP but works in terminal | MCP subprocess PATH issue — re-register with absolute path: `claude mcp add -s user replicate -- $(which npx) -y replicate-mcp@latest` |
| "Failed to connect" in `claude mcp list` | Token missing or expired — check `echo $REPLICATE_API_TOKEN`, regenerate at replicate.com/account/api-tokens |
| MCP tools missing after setup | Restart Claude Code — MCP servers load at session start |
| "401 Unauthorized" from Replicate | Bad API token — generate a new one at replicate.com/account/api-tokens |
| Dark halos after bg removal | Source image was on a dark background — regenerate on white |
| nvm-installed Node not found by MCP | nvm only loads in interactive shells — use full path in registration: `~/.nvm/versions/node/v22.x.x/bin/npx` |

For detailed troubleshooting, run `/generate-image:setup` — it diagnoses and fixes most issues automatically.

## Upgrading from Standalone Skill

If you previously used the standalone `generate-image` skill (installed at `~/.claude/skills/generate-image/`), remove it after installing this plugin to avoid duplicate triggering:

```bash
rm -r ~/.claude/skills/generate-image/
```

## License

MIT
