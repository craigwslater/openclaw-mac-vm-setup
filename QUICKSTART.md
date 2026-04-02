# OpenClaw Quick Start: Linux VM on macOS

> ⚡ One-page cheat sheet for experienced users. For detailed instructions with troubleshooting, see [the full guide](openclaw-mac-vm-complete-setup.md).

## How to Use This Guide

Choose the workflow that fits your style:

### Option 1: Self-Guided (Manual)
**Best for:** Experienced Linux users

1. Follow the TL;DR commands below
2. Copy-paste configs from this file
3. Check the Common Gotchas table if you hit an error
4. Run the Validation Checklist to confirm success

**Time:** 30-45 minutes | **Pros:** Works offline | **Cons:** Self-troubleshooting

---

### Option 2: AI-Assisted (Recommended)
**Best for:** Those who want interactive help

1. **Copy this entire guide** (or the full guide)
2. **Paste into ChatGPT, Claude, or another AI**
3. **Use this prompt:**

```
I want to set up OpenClaw on an Ubuntu VM on my Mac using the attached guide.

Walk me through this step-by-step. Before each step, tell me what we're doing and why.
After I run each command, ask me to share the output so you can verify it matches the expected results.

If something fails, reference the Edge Cases section and help me fix it before moving on.
```

**Time:** 45-60 minutes | **Pros:** Real-time troubleshooting | **Cons:** Requires AI access

---

### Option 3: Hybrid
**Best for:** Learning while doing

1. Start with Option 1 (Self-Guided)
2. When you hit an error, switch to Option 2 (ask AI for help)
3. Share the error message + which step you're on

---

## Prerequisites

- macOS with VMware Fusion, Parallels, UTM, or VirtualBox
- Ubuntu 22.04 LTS ISO (or your preferred Linux distro)
- Telegram account (or Discord, Signal, Slack, etc.)
- 30-60 minutes

**This guide uses our tested stack** (VMware Fusion + Ubuntu + Telegram), but OpenClaw works with other hypervisors, Linux distros, and messaging providers. See the [Complete Guide](openclaw-mac-vm-complete-setup.md) for alternative options.

## TL;DR Commands

```bash
# 1. Update system
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git nano htop

# 2. Install Node.js 22 (REQUIRED - not v18!)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
node --version  # MUST show v22.x.x

# 3. Install OpenClaw
sudo npm install -g openclaw
openclaw --version  # Should show version

# 4. Initialize OpenClaw
openclaw setup

# 5. Edit config
nano ~/.openclaw/config.yaml
```

## Minimal Working Config

**`~/.openclaw/config.yaml:`**

```yaml
channels:
  telegram:
    enabled: true
    botToken: "YOUR_BOT_TOKEN"
    dmPolicy: "pairing"
    allowFrom:
      - "YOUR_TELEGRAM_USER_ID"

gateway:
  bind: "127.0.0.1:8080"
```

**`~/.openclaw/exec-approvals.json:`**

```json
{
  "version": 1,
  "defaults": {
    "security": "full",
    "ask": "off",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "full",
      "ask": "off",
      "askFallback": "deny",
      "autoAllowSkills": true
    }
  }
}
```

## Start & Pair

```bash
# Start gateway
openclaw gateway start
openclaw gateway status  # Should show "running"

# Pair Telegram (send /start to bot first)
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>

# Test
openclaw exec -- ls -la ~/.openclaw/
```

## Common Gotchas

| Issue | Solution |
|-------|----------|
| **Node v18 installed** | Remove: `sudo apt remove nodejs npm && sudo apt autoremove`<br>Then reinstall via NodeSource |
| **"Exec approval required"** | Set `"security": "full"` in `exec-approvals.json` (both `defaults` AND `agents.main`) |
| **YAML parse error** | Use spaces, not tabs. Validate: `openclaw config validate` |
| **Config changes not working** | Must restart: `openclaw gateway restart` |
| **"allowlist miss" error** | Either add `"allowlist": []` or switch to `"security": "full"` |
| **Pairing expired** | Codes expire after 1 hour. Remove old: `openclaw pairing remove telegram <CODE>` |

## Validation Checklist

Run these to verify your setup:

```bash
# 1. Node.js v22+
node --version  # v22.x.x

# 2. OpenClaw installed
openclaw --version

# 3. Gateway running
openclaw gateway status

# 4. Exec works
openclaw exec -- ls -la ~/.openclaw/

# 5. Telegram works (if configured)
openclaw message send --channel telegram --to YOUR_ID --message "Test"
```

## File Locations

| File | Purpose |
|------|---------|
| `~/.openclaw/config.yaml` | Gateway, Telegram, channels |
| `~/.openclaw/exec-approvals.json` | **Security policy** (this controls exec!) |
| `~/.openclaw/workspace/` | Agent memory and files |

## Optional Add-ons

**Ollama (local AI):**
```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.2
# Add to config.yaml: models.default: ollama/llama3.2
```

**Tailscale (remote access):**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale ip -4
```

## Need Help?

**Using this guide:**
- **Self-Guided:** Follow TL;DR commands → Check Common Gotchas → Run Validation Checklist
- **AI-Assisted:** Copy this file → Paste into ChatGPT/Claude → Use the prompt in [How to Use This Guide](#how-to-use-this-guide)
- **Full guide with screenshots:** [openclaw-mac-vm-complete-setup.md](openclaw-mac-vm-complete-setup.md)

**Resources:**
- OpenClaw docs: https://docs.openclaw.ai
- Community: https://discord.com/invite/clawd

---

*Tested on: VMware Fusion + Ubuntu 22.04 + OpenClaw 2026.4.1*
