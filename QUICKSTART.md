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
- **Minimum 4GB RAM, 40GB disk for VM**

**⚠️ NETWORK SECURITY DECISION:**

Choose **BEFORE** you start:

**Option A: NAT (Share with Mac) - RECOMMENDED** ⭐
- VM isolated on own subnet, cannot access connected devices
- Best for: Most users, good security without complexity
- Setup: Default VMware option (you probably already have this)

**Option B: USB WiFi → Guest Network (Maximum)**
- Uses separate USB WiFi adapter on Guest network
- Best for: Maximum isolation, sensitive connected devices
- Setup: [SECURITY_HARDENING.md](SECURITY_HARDENING.md#option-b-usb-wifi--guest-network-maximum-isolation)

**This guide uses our tested stack** (VMware Fusion + Ubuntu + Telegram), but OpenClaw works with other hypervisors, Linux distros, and messaging providers. See the [Complete Guide](openclaw-mac-vm-complete-setup.md) for alternative options.

**Resource Requirements:**

| Spec | Minimum | Recommended | Our Setup |
|------|---------|-------------|-----------|
| **VM RAM** | 2GB | 4GB | 4GB |
| **VM Disk** | 20GB | 40GB | 40GB |
| **Mac Free Space** | 60GB | 100GB | Varies |

**At Minimum (2GB/20GB):**
- ✅ OpenClaw works
- ⚠️ Swap used (slower performance)
- ❌ No Ollama local AI
- ❌ Disk fills quickly (~6 months)

**At Recommended (4GB/40GB):**
- ✅ Smooth performance
- ✅ Room for growth
- ✅ Can add Ollama later

## System Discovery (Do This First)

Before running commands, verify your specific environment:

### 1. Find Your VM's IP Address
```bash
ip addr show | grep "inet " | head -1 | awk '{print $2}'
```
✅ **Expected:** Something like `192.168.1.100/24`  
⚠️ **If blank:** Your VM network isn't configured. Check VM settings → Network → Bridged mode

### 2. Check for Existing Software
```bash
# Node.js (will conflict if wrong version)
node --version  # Should error OR show v22.x.x

# OpenClaw (should error if not installed)
openclaw --version 2>/dev/null || echo "Not installed"

# Check port 8080 (must be free)
sudo lsof -i :8080 2>/dev/null || echo "Port 8080 is free"
```

### 3. Check Disk Space (Critical!)
```bash
df -h / | awk 'NR==2 {print "Disk Usage: " $5 " full (" $3 " used of " $2 ")"}'
```
✅ **Safe:** Under 70% full  
⚠️ **Warning:** 70-90% full (clean up soon)  
❌ **Critical:** Over 90% full (STOP and free space now)

**If over 90% full, clean up:**
```bash
# Remove package cache
sudo apt clean

# Remove unused packages
sudo apt autoremove -y

# Clear old logs (keep 7 days)
sudo journalctl --vacuum-time=7d

# Check what's using space
du -sh /var/log ~/.openclaw /tmp 2>/dev/null | sort -h
```

### 4. Verify VM Network Mode
```bash
ip route | grep default
```
✅ **Bridged mode:** Shows your router's IP  
✅ **NAT mode:** Shows VMware/VirtualBox gateway (usually 192.168.x.x)  
⚠️ **Host-only:** Limited to VM-to-Mac only

**If you need to switch network modes:** Shut down VM → VM Settings → Network → Change from NAT to Bridged

### 5. Identify Your Telegram User ID
Before configuring, get your ID:
```bash
# Option A: Message @userinfobot, note the number
# Option B: Check from OpenClaw logs (after first Telegram message):
# openclaw logs --follow
# Send /start to your bot, look for "from.id" in logs
```

**Record these values:**
- VM IP: ________________
- Network Mode: ________________
- Telegram User ID: ________________

## TL;DR Commands

### ⚡ Quick Prerequisites Check

```bash
# Run these first - if any fail, see troubleshooting below
node --version  # Should be v22+ (if installed) or error
ip addr show    # Should show an IP address
```

If these pass, proceed. If not, see [System Discovery](#system-discovery-do-this-first) below.

---

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

⚠️ **SECURITY WARNING:** See [SECURITY_HARDENING.md](SECURITY_HARDENING.md) for secure token storage options. Below uses plain text for simplicity, but environment variables are recommended for production.

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
