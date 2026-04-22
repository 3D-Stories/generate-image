---
name: setup
description: >-
  Set up the Replicate MCP server for image generation. Detects and installs
  prerequisites (Node.js, npm, npx), configures the API token, and registers
  the MCP server with Claude Code. Use when user says "setup replicate",
  "configure image generation", when generate-image skill reports missing MCP,
  or when the user gets errors trying to generate images. Also use proactively
  if you detect Replicate MCP tools are missing when the user asks to generate
  an image.
---

# Replicate MCP Setup

Automatically detect, install, and configure everything needed for image generation. Run each step in order — skip steps that are already satisfied.

## Step 1: Check if Already Working

Check whether you have access to any tools starting with `mcp__replicate__`.

**If Replicate MCP tools are available:**
1. Run `mcp__replicate__get_account` to verify the token works.
2. If it succeeds: report "Replicate MCP is ready. Generate images by asking Claude to create any visual asset." and STOP.
3. If it fails with an auth error: the token is bad — jump to Step 4 (Token Setup).

**If no Replicate MCP tools are found:** continue to Step 2.

## Step 2: Check Node.js and npm

Run these checks via Bash:

```bash
echo "=== node ===" && which node && node --version || echo "MISSING"
echo "=== npm ===" && which npm && npm --version || echo "MISSING"
echo "=== npx ===" && which npx && npx --version || echo "MISSING"
```

**If all three are present and node >= v18:** skip to Step 3.

**If Node.js is missing or too old (< v18):** detect the platform and offer to install.

### Detect Platform

```bash
echo "=== platform ===" && uname -s && uname -m
echo "=== package manager ===" && which brew 2>/dev/null && echo "homebrew" || which apt 2>/dev/null && echo "apt" || which dnf 2>/dev/null && echo "dnf" || which pacman 2>/dev/null && echo "pacman" || echo "unknown"
```

### Install Node.js

Tell the user what you're about to do and ask for confirmation:

"Node.js is required for the Replicate MCP server. I'd like to install it via [detected method]. OK?"

Then use the appropriate method:

**macOS (Homebrew):**
```bash
brew install node
```

**Ubuntu/Debian (apt) — use NodeSource for a current version:**
```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
```
If `sudo` is unavailable or the user declines root access, suggest **nvm** instead:
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
export NVM_DIR="$HOME/.nvm" && [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
nvm install 22
```

**Fedora/RHEL (dnf):**
```bash
sudo dnf install -y nodejs
```

**Arch (pacman):**
```bash
sudo pacman -S nodejs npm
```

**Windows (WSL):** Use the Ubuntu/Debian instructions above inside WSL.

**Unknown platform or no package manager:** suggest installing from https://nodejs.org/en/download or using nvm (the curl command above works on any Linux/macOS).

After install, **verify** by re-running the checks from the top of this step. If node/npm/npx are still missing, STOP and report the error — don't proceed with a broken toolchain.

### npm/npx edge cases

- **npx installed but "command not found" in MCP context** — this is the most common gotcha. Claude Code spawns MCP servers as child processes that may not inherit the user's full PATH (especially with nvm or non-standard installs). The fix is to use the **absolute path** to npx when registering:
  ```bash
  # Find where npx lives
  which npx
  # Use the full path in the MCP registration (Step 3)
  claude mcp add -s user replicate -- /usr/local/bin/npx -y replicate-mcp@latest
  ```
  If the user installed Node via nvm, the path will be something like `~/.nvm/versions/node/v22.x.x/bin/npx` — use that instead.
- **npx missing but npm present** — older npm version. Fix: `npm install -g npx`
- **npm permission errors (EACCES)** — npm cache permissions. Fix:
  ```bash
  mkdir -p ~/.npm && npm config set cache ~/.npm
  ```

## Step 3: Register Replicate MCP with Claude Code

Check if the MCP server is already registered:
```bash
claude mcp list 2>&1 | grep -i replicate
```

First, resolve the absolute path to npx (avoids PATH issues in MCP subprocess environments):
```bash
NPX_PATH=$(which npx)
echo "npx is at: $NPX_PATH"
```

**If already registered but showing "Failed to connect":** likely a token issue or stale registration. Remove and re-add:
```bash
claude mcp remove replicate
claude mcp add -s user replicate -- "$NPX_PATH" -y replicate-mcp@latest
```

**If not registered:**
```bash
claude mcp add -s user replicate -- "$NPX_PATH" -y replicate-mcp@latest
```

Using the absolute path (e.g., `/usr/local/bin/npx`) instead of bare `npx` prevents "command not found" errors in the MCP subprocess — the most commonly reported setup failure. The `-s user` flag registers it globally (not per-project) so image generation works in any project.

After registration, verify:
```bash
claude mcp list 2>&1 | grep -i replicate
```

**Common failures at this step:**

| Symptom | Fix |
|---------|-----|
| "npx: command not found" | Step 2 didn't complete — go back and fix Node.js |
| "EACCES permission denied" | `mkdir -p ~/.npm && npm config set cache ~/.npm` then retry |
| Network/proxy errors downloading replicate-mcp | Check `HTTP_PROXY`/`HTTPS_PROXY` env vars, or `npm config set registry https://registry.npmjs.org/` |
| "Error: Cannot find module" | Stale npx cache: `rm -rf ~/.npm/_npx` then retry |

## Step 4: API Token Setup

Check if the token is already set:
```bash
echo "${REPLICATE_API_TOKEN:+SET}" || echo "NOT_SET"
```

**If already set:** skip to Step 5.

**If not set:** the user needs to create one on Replicate's website. Tell them:

"I need a Replicate API token to connect to the image generation service.

1. Go to **replicate.com** and sign up (or log in)
2. Go to **Account Settings > API Tokens** (replicate.com/account/api-tokens)
3. Click **Create token**, give it a name, and copy it
4. Paste it here — I'll configure it for you"

**When the user provides the token:**

1. Validate format — should start with `r8_` and be 40+ characters. If it looks wrong, ask them to double-check.
2. Persist to shell profile (idempotent):

```bash
# Detect the user's actual shell profile
if [[ "$SHELL" == */zsh ]]; then
    PROFILE="$HOME/.zshrc"
else
    PROFILE="$HOME/.bashrc"
fi

if grep -q 'REPLICATE_API_TOKEN' "$PROFILE" 2>/dev/null; then
    sed -i.bak "s|^export REPLICATE_API_TOKEN=.*|export REPLICATE_API_TOKEN=\"TOKEN_HERE\"|" "$PROFILE"
    echo "Updated existing token in $PROFILE"
else
    echo "" >> "$PROFILE"
    echo "# Replicate API token for image generation" >> "$PROFILE"
    echo "export REPLICATE_API_TOKEN=\"TOKEN_HERE\"" >> "$PROFILE"
    echo "Added token to $PROFILE"
fi
```

(Replace TOKEN_HERE with the actual token value.)

3. Export for the current shell session too:
```bash
export REPLICATE_API_TOKEN="<token>"
```

**Security reminder:** The token lives in the shell profile (not in any repo). Never echo or log the full token value.

## Step 5: Verify and Report

Report what was configured:

```
Setup complete!
  Node.js:       [version]
  npm:           [version]  
  Replicate MCP: registered (user scope)
  API token:     configured in [profile path]
```

Then tell the user:

"**Restart Claude Code** (or start a new session) for the MCP tools to appear. Then try: 'generate an icon of a gold star' to test."

If the user can't restart right now, explain that the MCP registration takes effect on the next session start — this is how Claude Code's MCP system works, not something we can change.

## Troubleshooting

If the user comes back with issues after setup:

| Symptom | Cause | Fix |
|---------|-------|-----|
| "npx: command not found" in MCP but `npx` works in terminal | MCP subprocess doesn't inherit full PATH | Re-register with absolute path: `claude mcp add -s user replicate -- $(which npx) -y replicate-mcp@latest` |
| "replicate: ... ✗ Failed to connect" | Token not set or expired | Check `echo $REPLICATE_API_TOKEN`, regenerate at replicate.com if needed |
| MCP tools missing after restart | Registration didn't persist | Re-run with `-s user` flag and absolute npx path |
| "401 Unauthorized" from Replicate | Bad or expired token | Generate a new token at replicate.com/account/api-tokens |
| npx downloads packages every time | Normal for `npx -y` | First run is slow (~10s), subsequent runs use npm cache |
| generate-image skill says "Replicate MCP not detected" | MCP registered but session not restarted | Restart Claude Code — MCP servers load at session start |
| nvm-installed Node not found by MCP | nvm only loads in interactive shells | Use full path: `~/.nvm/versions/node/v22.x.x/bin/npx` in the registration command |
