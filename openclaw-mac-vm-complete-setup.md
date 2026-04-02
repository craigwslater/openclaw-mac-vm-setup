# Complete OpenClaw Setup Guide: Linux VM on macOS

A comprehensive, step-by-step guide for running OpenClaw securely in a Linux virtual machine on macOS.

**Prerequisites:**
- macOS (Intel or Apple Silicon)
- VMware Fusion, Parallels, UTM, or VirtualBox
- Ubuntu 22.04 LTS ISO (or your preferred Linux distro)
- 30-60 minutes of setup time

**Why a VM?**
Security and isolation. OpenClaw can execute commands on your system. Running it in a VM keeps your main machine pristine — even if something goes wrong, your Mac and personal data remain untouched.

---

## Pre-Flight Checklist

Before starting, verify you have:
- [ ] Ubuntu 22.04 LTS ISO downloaded
- [ ] VMware Fusion, Parallels, UTM, or VirtualBox installed
- [ ] At least 40GB free disk space
- [ ] Telegram account (for bot integration)
- [ ] 30-60 minutes uninterrupted time

---

## Step 1: Create Your Linux VM

**⏱️ Time Estimate: 15-20 minutes**

### VMware Fusion (Recommended)

1. Download VMware Fusion Player (free for personal use) or Pro
2. Click **File → New**
3. Select **Install from disk or image**
4. Choose your Ubuntu 22.04 LTS ISO
5. Configure VM settings:

| Setting | Recommendation |
|---------|---------------|
| **RAM** | 4GB minimum, 8GB preferred |
| **CPU** | 2 cores minimum |
| **Disk** | 40GB+ (thin provisioned) |
| **Network** | **Bridged** (recommended) or NAT |

6. Complete Ubuntu installation
   - Choose minimal installation (no need for GUI if you're comfortable with CLI)
   - Create your user account
   - Note your VM's IP address: `ip addr show`

✅ **Expected Result:** VM boots to Ubuntu login prompt; you can log in successfully

### UTM (Apple Silicon Macs)

1. Download UTM from https://mac.getutm.app
2. Create a new VM with Ubuntu Server ARM64
3. Similar resource allocation as above

✅ **Expected Result:** VM boots successfully; can log in via terminal

---

## Step 2: Initial System Setup

**⏱️ Time Estimate: 10 minutes**

Once your VM is running, update the system:

```bash
# Update package lists and upgrade
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y curl git nano htop

# Install Node.js 22 (required for OpenClaw)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Verify installation
node --version  # Should show v22.x.x
npm --version
```

✅ **Expected Result:**
```
v22.22.2
10.x.x
```

⚠️ **If you see v18.x.x:** You used the wrong install method. See Edge Case 1 below.

---

## Step 3: Install OpenClaw

**⏱️ Time Estimate: 5 minutes**

```bash
# Install OpenClaw globally via npm
sudo npm install -g openclaw

# Verify installation
openclaw --version
openclaw help
```

✅ **Expected Result:**
```
OpenClaw 2026.4.1 (da64a97)

Usage: openclaw <command> [options]

Commands:
  ...
```

You should see OpenClaw version information and available commands.

---

## Step 4: Initial OpenClaw Configuration

**⏱️ Time Estimate: 15 minutes**

### Run the Setup Wizard

```bash
# Initialize OpenClaw
openclaw setup
```

This creates:
- `~/.openclaw/` — configuration directory
- `~/.openclaw/workspace/` — your agent workspace
- `~/.openclaw/config.yaml` — main configuration file

✅ **Expected Result:** No errors; directories created successfully

### Edit Your Config File

```bash
nano ~/.openclaw/config.yaml
```

Add your Telegram bot configuration:

```yaml
channels:
  telegram:
    enabled: true
    botToken: "YOUR_BOT_TOKEN_HERE"
    dmPolicy: "pairing"
    allowFrom:
      - "YOUR_TELEGRAM_USER_ID"

gateway:
  bind: "127.0.0.1:8080"
```

**Getting your Telegram bot token:**
1. Message @BotFather on Telegram
2. Send `/newbot`
3. Follow prompts, copy the token

**Finding your Telegram user ID:**
1. Message @userinfobot or check your Telegram profile
2. It's a numeric ID like `8748931722`

---

## Step 5: Start OpenClaw

**⏱️ Time Estimate: 5 minutes**

```bash
# Start the OpenClaw gateway
openclaw gateway start

# Check status
openclaw gateway status

# View logs (in another terminal)
openclaw logs --follow
```

✅ **Expected Result:**
```
openclaw gateway status
Gateway: running
PID: 12345
```

---

## Step 6: Pair Your Telegram Account

**⏱️ Time Estimate: 5 minutes**

Send `/start` to your bot on Telegram. Then in your VM:

```bash
# List pending pairing requests
openclaw pairing list telegram

# Approve the pairing
openclaw pairing approve telegram <CODE>
```

✅ **Expected Result:** Bot responds to your messages on Telegram

Now you can message your bot and it will respond!

---

## Step 7: Configure Exec Approvals (Security)

**⏱️ Time Estimate: 10 minutes**

This is the step we spent the most time on. Here's the working configuration.

### Edit the Exec Approvals File

```bash
nano ~/.openclaw/exec-approvals.json
```

Replace with:

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

**What this does:**
- `security: "full"` — Allows all shell commands
- `ask: "off"` — No approval prompts for commands
- `autoAllowSkills: true` — Automatically allow skill binaries

**Security note:** This configuration removes all approval friction for your user account. It's fine for personal use in an isolated VM, but you'd want stricter settings (like `"security": "allowlist"` with explicit command patterns) for shared environments.

### Alternative: Keep Approval Buttons

If you want approval prompts with Telegram buttons, use this config instead:

```yaml
# In ~/.openclaw/config.yaml
channels:
  telegram:
    enabled: true
    botToken: "YOUR_TOKEN"
    dmPolicy: "pairing"
    allowFrom:
      - "YOUR_USER_ID"
    execApprovals:
      enabled: true
      target: "dm"
    capabilities:
      inlineButtons: "dm"

gateway:
  bind: "127.0.0.1:8080"

approvals:
  exec:
    enabled: true
    mode: "session"
    targets:
      - channel: "telegram"
        to: "YOUR_USER_ID"
```

Then set `~/.openclaw/exec-approvals.json` to:

```json
{
  "version": 1,
  "defaults": {
    "security": "allowlist",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": []
    }
  }
}
```

**Note:** We found this button-based approach didn't work reliably in our setup. The auto-approve method above is more reliable.

---

## Step 8: Restart and Test

**⏱️ Time Estimate: 5 minutes**

After configuration changes:

```bash
openclaw gateway restart
```

Then test from Telegram:

```
Can you list the files in my workspace directory?
```

The agent should execute `ls -la ~/.openclaw/workspace/` and show you the results.

---

## Step 9: (Optional) Install Ollama for Local AI

**⏱️ Time Estimate: 10 minutes**

If you want to run AI models locally instead of using APIs:

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull a model
ollama pull llama3.2

# Update OpenClaw config
nano ~/.openclaw/config.yaml
```

Add:

```yaml
models:
  default: ollama/llama3.2
  ollama:
    baseUrl: http://localhost:11434
```

Restart:

```bash
openclaw gateway restart
```

✅ **Expected Result:** `ollama list` shows downloaded models; OpenClaw can generate responses using local models

---

## Step 10: (Optional) Remote Access via Tailscale

**⏱️ Time Estimate: 10 minutes**

For accessing OpenClaw from outside your local network:

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Authenticate
sudo tailscale up

# Get your Tailscale IP
tailscale ip -4
```

Update OpenClaw config:

```yaml
gateway:
  bind: "0.0.0.0:8080"
  remote:
    url: "https://your-vm.tailXXXX.ts.net:8080"
```

✅ **Expected Result:** `tailscale status` shows connected; you can access OpenClaw from Tailscale IP

---

## Edge Cases & Real-World Gotchas

This section documents actual issues we encountered during setup and their solutions. If you're following this guide and hit a snag, check here first.

### Edge Case 1: Wrong Node.js Version (v18.19.1 vs v22+)

**The Problem:**
Ubuntu's default repositories ship with Node.js v18.19.1, but OpenClaw requires Node.js 22+. If you install with `sudo apt install nodejs`, you'll get an incompatible version.

**Error you'll see:**
```
npm ERR! engine Unsupported engine
npm ERR! engine Not compatible with your version of nodejs
```

**The Solution:**
Always use NodeSource setup script (as shown in Step 2):
```bash
# This installs the CORRECT version
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Verify: MUST show v22.x.x
node --version
```

**If you already installed the wrong version:**
```bash
# Remove old Node.js
sudo apt remove nodejs npm

# Clean up apt cache
sudo apt autoremove

# Then reinstall using NodeSource method above
```

---

### Edge Case 2: YAML Formatting Errors

**The Problem:**
`~/.openclaw/config.yaml` uses YAML, which is whitespace-sensitive. A missing space or incorrect indentation breaks the entire config.

**Error you'll see:**
```
Error: Failed to parse config.yaml
```

**The Solution:**
Use spaces (not tabs), and ensure consistent indentation. Here's a valid example:

```yaml
# CORRECT - spaces, not tabs
channels:
  telegram:
    enabled: true
    botToken: "your-token"
    allowFrom:
      - "123456789"

# WRONG - tabs or inconsistent indentation
channels:
  telegram:
  enabled: true  # ← This is wrong (should be 4 spaces deeper)
```

**Quick validation:**
```bash
# If this shows errors, your YAML is malformed
openclaw config validate
```

---

### Edge Case 3: The Hidden exec-approvals.json File

**The Problem:**
Most people expect exec approvals to be configured in `~/.openclaw/config.yaml`. But the actual security policy lives in a **separate file**: `~/.openclaw/exec-approvals.json`. This caused hours of confusion during our setup.

**What happens:**
You set approvals in `config.yaml`, but commands still get rejected because `exec-approvals.json` has different settings that take precedence.

**The Solution:**
Always edit **both** files:
1. `~/.openclaw/config.yaml` — for Telegram, gateway, and general settings
2. `~/.openclaw/exec-approvals.json` — for security policy and command permissions

**File hierarchy:**
```
~/.openclaw/
├── config.yaml           ← Gateway, channels, Telegram
├── exec-approvals.json   ← Security policy ← This is what actually controls exec!
└── workspace/           ← Your agent's memory and files
```

---

### Edge Case 4: "Exec approval is required but chat exec approvals are not enabled"

**The Problem:**
You configured Telegram approvals but still get this error. This happened to us — the Telegram approval buttons never actually worked despite correct config.

**Root causes we discovered:**
1. The `exec-approvals.json` defaults override Telegram-specific settings
2. Missing `allowFrom` at the right config level
3. Gateway needs restart after config changes

**What we tried (didn't work):**
- Adding `execApprovals` to `config.yaml`
- Adding `approvers` list
- Setting `capabilities.inlineButtons: "dm"`
- Various `target: "dm"` / `target: "channel"` combinations

**What actually worked:**
Setting `"security": "full"` and `"ask": "off"` in `~/.openclaw/exec-approvals.json` for auto-approval.

**Our recommendation:**
Skip the Telegram approval buttons for now. Use the auto-approve configuration shown in Step 7. It's simpler and works reliably.

---

### Edge Case 5: Config Changes Don't Take Effect

**The Problem:**
You edited `config.yaml` or `exec-approvals.json` but the old behavior persists.

**The Solution:**
**Always restart the gateway after config changes:**
```bash
openclaw gateway restart
```

**Important distinction:**
- `~/.openclaw/config.yaml` → Requires **restart**
- `~/.openclaw/exec-approvals.json` → Read at **exec time** (usually no restart needed)

**If even restart doesn't help:**
Check which config file is actually being read:
```bash
# Check if config is valid
openclaw config validate

# View effective configuration
openclaw config show
```

---

### Edge Case 6: "allowlist miss" Error

**The Problem:**
You see this error when trying to run commands:
```
exec denied: allowlist miss
```

**Root cause:**
Your `exec-approvals.json` has `"security": "allowlist"` but no allowed commands in the `allowlist` array.

**Two solutions:**

**Option A: Switch to full security (recommended for personal use)**
```json
{
  "defaults": {
    "security": "full",
    "ask": "off"
  }
}
```

**Option B: Add specific commands to allowlist (more secure)**
```json
{
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "allowlist": [
        {
          "pattern": "/usr/bin/ls",
          "id": "some-uuid"
        },
        {
          "pattern": "/usr/bin/cat",
          "id": "another-uuid"
        }
      ]
    }
  }
}
```

---

### Edge Case 7: Pairing Code Expires

**The Problem:**
You see a pairing code in logs but by the time you try to approve it, you get an error.

**The Solution:**
Pairing codes expire after **1 hour**. If it expires:

```bash
# Remove expired pairing request
openclaw pairing remove telegram <OLD_CODE>

# Start fresh - message your bot again to generate new code
```

Then quickly approve:
```bash
openclaw pairing approve telegram <NEW_CODE>
```

---

### Edge Case 8: "Socket not found" or Gateway Connection Issues

**The Problem:**
Commands fail with connection errors or "cannot connect to gateway".

**Check these in order:**

1. **Is OpenClaw running?**
   ```bash
   openclaw gateway status
   # Should show: running
   ```

2. **Is something else using port 8080?**
   ```bash
   sudo lsof -i :8080
   # If something else is using it, change port in config.yaml
   ```

3. **Are permissions correct?**
   ```bash
   ls -la ~/.openclaw/
   # Should be owned by your user, not root
   ```

4. **Try full restart:**
   ```bash
   openclaw gateway stop
   sleep 2
   openclaw gateway start
   ```

---

### Edge Case 9: VM Network Issues

**The Problem:**
OpenClaw works inside the VM but you can't access it from your Mac host.

**Common causes & fixes:**

**NAT mode (VMware default):**
- VM shares host IP through NAT
- You need port forwarding: VMware → VM Settings → Network → Advanced → Port Forwarding
- Map Host Port 8080 → Guest Port 8080

**Bridged mode (recommended):**
- VM gets its own IP on your LAN
- Find VM IP: `ip addr show | grep inet`
- Access from Mac: `http://VM_IP:8080`
- If no IP shown, check: `sudo dhclient -v`

**Firewall blocking:**
```bash
# Check if firewall is active
sudo ufw status

# If active, allow port 8080
sudo ufw allow 8080/tcp
```

---

### Testing This Guide

Want to verify this guide works? Here's a quick validation checklist:

```bash
# 1. Check Node.js version
node --version  # Should be v22.x.x

# 2. Verify OpenClaw installed
openclaw --version  # Should show version

# 3. Check gateway is running
openclaw gateway status

# 4. Test exec without prompts
openclaw exec -- ls -la ~/.openclaw/

# 5. Send Telegram message (if configured)
openclaw message send --channel telegram --to YOUR_ID --message "Test"
```

If all 5 pass, your setup is complete and working!

---

## Troubleshooting

### "Exec approval is required but chat exec approvals are not enabled"

This means the approval system is active but can't find a delivery method. Solutions:

1. **Use the auto-approve config** (recommended, see Step 7)
2. Or approve via Web UI at `http://VM_IP:3000`

### Commands still asking for approval after config changes

- Check `~/.openclaw/exec-approvals.json` has `"security": "full"` in both `defaults` AND `agents.main`
- Restart OpenClaw: `openclaw gateway restart`

### Can't connect from Mac host to VM

- Verify network mode (Bridged gives VM its own IP)
- Check firewall: `sudo ufw status`
- Test with `curl http://VM_IP:8080/health` from Mac terminal

### Telegram bot not responding

- Verify bot token is correct in config
- Check OpenClaw logs: `openclaw logs --follow`
- Ensure you've paired your account (Step 6)

---

## Your Workspace

OpenClaw creates files in `~/.openclaw/workspace/`:

| File | Purpose |
|------|---------|
| `SOUL.md` | Your bot's personality and behavior |
| `USER.md` | Information about you |
| `MEMORY.md` | Long-term memory (curated) |
| `memory/YYYY-MM-DD.md` | Daily activity logs |
| `AGENTS.md` | Agent configuration |
| `HEARTBEAT.md` | Periodic task checklist |

Customize these to make your agent truly yours!

---

## Alternative Options (Mix and Match)

This guide uses **our tested stack**, but OpenClaw is flexible. Here's what you can swap:

### Hypervisor Options

| Option | Platform | Notes |
|--------|----------|-------|
| **VMware Fusion** ⭐ | Intel & Apple Silicon | What we use. Player is free for personal use. |
| Parallels Desktop | Apple Silicon | Fast, but paid subscription |
| UTM | Apple Silicon | Free, native ARM support |
| VirtualBox | Intel | Free, widely used |

**Our recommendation:** VMware Fusion (reliable, well-documented, free tier available)

### Guest OS Options

| Option | Notes |
|--------|-------|
| **Ubuntu 22.04 LTS** ⭐ | What we use. Best documentation, widest support. |
| Ubuntu 24.04 LTS | Newer, may have minor differences |
| Debian 12 | Similar to Ubuntu, slightly more minimal |
| Fedora | Newer packages, different package manager (dnf vs apt) |

**Our recommendation:** Ubuntu 22.04 LTS (most tested with OpenClaw)

### Messaging Provider Options

| Option | Setup Complexity | Best For |
|--------|------------------|----------|
| **Telegram** ⭐ | Easy | Mobile-first, quick pairing |
| Discord | Medium | Community servers, rich formatting |
| Signal | Hard | Maximum privacy (requires phone number) |
| Slack | Medium | Workplace integration |
| WhatsApp | Hard | Phone-centric users |
| Web UI | None | Browser-only (no persistent chat) |

**Our recommendation:** Telegram (fastest to get running, great mobile experience)

### AI Model Options

| Option | Setup | Speed | Privacy |
|--------|-------|-------|---------|
| **Cloud APIs** (OpenAI, Anthropic, etc.) | Just add API key | Fast | Data leaves VM |
| **Ollama (local)** ⭐ | Install + download models | Slower (depends on VM specs) | 100% private |
| **Self-hosted** | Advanced setup | Varies | Varies |

**Our recommendation:** Start with cloud for speed, migrate to Ollama for privacy

### Network Mode Options

| Mode | Use Case | Access from Mac |
|------|----------|-----------------|
| **Bridged** ⭐ | VM has its own IP | Direct via VM IP |
| NAT | VM shares host IP | Requires port forwarding |
| Host-only | Isolated network only | Not accessible |

**Our recommendation:** Bridged (simplest for cross-device access)

---

## Security Best Practices

1. **Keep the VM isolated** — Don't mount your Mac home directory
2. **Regular snapshots** — Before major changes, snapshot the VM
3. **Monitor exec-approvals.json** — Review what's been auto-approved
4. **Use allowlists in production** — If you open this to others, switch from `"security": "full"` to `"allowlist"` with explicit command patterns
5. **Backup your workspace** — The `~/.openclaw/workspace/` contains your agent's memory

---

## Next Steps

- Explore built-in skills: `openclaw skill list`
- Install new skills: `openclaw skill install weather`
- Create custom skills in your workspace
- Set up cron jobs for scheduled tasks
- Read the docs: https://docs.openclaw.ai
- Join the community: https://discord.com/invite/clawd

---

**Questions or issues?** This guide reflects a real working setup tested on VMware Fusion + Ubuntu 22.04 + OpenClaw 2026.4.1.

*Happy automating!*
