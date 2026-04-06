# generate-image

A Claude Code plugin for generating images using AI models on Replicate. Supports Nano Banana 2 (high quality) and FLUX Schnell (fast/cheap), with automatic background removal for transparent PNGs.

## Prerequisites

- [Claude Code](https://claude.ai/code) installed
- A [Replicate](https://replicate.com) account with an API token

## Installation

```bash
claude plugin install 3D-Stories/generate-image
```

Then set up the Replicate MCP server:

```
/generate-image:setup
```

The setup skill will guide you through configuring the Replicate MCP server and verifying your API token.

## Usage

Just ask Claude to generate images naturally:

- "Generate an icon of a golden trophy for my app"
- "Create a hero image for the landing page — a mountain landscape at sunset"
- "Remove the background from logo.png"
- "Generate a set of 5 category icons matching my style guide"

The skill handles model selection, prompt engineering, background removal, and file management automatically.

## Models

| Model | Best For | Cost | Speed |
|-------|----------|------|-------|
| Nano Banana 2 | Icons, illustrations, complex scenes, high quality | ~$0.04/image | ~20s |
| FLUX Schnell | Quick drafts, bulk generation | ~$0.003/image | ~0.5s |
| Background Remover | Transparent PNGs from any image | ~$0.01/image | ~5s |

NB2 is the default for production assets. FLUX is used for quick iteration when cost matters more than quality.

## Upgrading from Standalone Skill

If you previously used the standalone `generate-image` skill (installed at `~/.claude/skills/generate-image/`), you **must** remove it after installing this plugin to avoid duplicate triggering:

```bash
rm -r ~/.claude/skills/generate-image/
```

Both the plugin and standalone skill have similar trigger descriptions, so Claude may invoke either or both unpredictably if both are present.

## License

MIT
