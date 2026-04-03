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
- [ ] Ubuntu 22.04 LTS ISO downloaded (correct architecture for your Mac)
- [ ] VMware Fusion, Parallels, UTM, or VirtualBox installed
- [ ] At least 40GB free disk space on Mac
- [ ] 4GB+ RAM available for VM
- [ ] Telegram account (for bot integration)
- [ ] 30-60 minutes uninterrupted time

**Download the Correct Ubuntu ISO:**

| Your Mac | Download This ISO | File Name Pattern |
|----------|-----------------|-------------------|
| Intel Mac (2019 or older) | Ubuntu 22.04 LTS AMD64 | `ubuntu-22.04.x-desktop-amd64.iso` |
| Apple Silicon (M1/M2/M3) | Ubuntu 22.04 LTS ARM64 | `ubuntu-22.04.x-desktop-arm64.iso` |

⚠️ **Using the wrong ISO:** ARM64 on Intel = won't boot. AMD64 on Apple Silicon = extremely slow emulation.

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

### ⚠️ CRITICAL: Take a Snapshot NOW

**Before installing ANYTHING else, snapshot your VM:**

**VMware:** VM → Snapshot → Take Snapshot → Name: "Clean Ubuntu Install"

**Why:** If OpenClaw setup goes wrong, you can revert to this clean state in 30 seconds instead of reinstalling Ubuntu.

**When to snapshot:**
- ✅ After Ubuntu install (NOW)
- ✅ After OpenClaw working
- ❌ Skip = hours of rework if something breaks

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

### Discover Your System (5 minutes)

**Why:** Every VM is slightly different. These commands reveal your specific configuration.

#### 1. What's Your VM's IP?
```bash
ip addr show | grep "inet " | grep -v "127.0.0.1" | head -1
```

✅ **Expected Output:**
```
inet 192.168.1.100/24 brd 192.168.1.255 scope global eth0
```
**Your IP is:** `192.168.1.100` (write this down)

⚠️ **If you see nothing:** Your VM is not getting an IP. Solutions:
- Bridged mode: Check your Mac's WiFi is connected
- NAT mode: This is expected, use port forwarding instead

#### 2. Check for Existing Node.js (Conflict Detection)
```bash
which node && node --version
```

**Three possibilities:**
1. **Command not found** ✅ Good, proceed with Node.js 22 install
2. **v22.x.x** ✅ Perfect, skip Node.js install
3. **v18.x.x** ⚠️ **CONFLICT!** See Edge Case 1 (remove old version first)

#### 3. Verify Your Hypervisor's Network
```bash
# Check default gateway
ip route | grep default
```

**What this tells you:**
- Gateway is your Mac's router IP (e.g., 192.168.1.1) → **Bridged mode** ✅
- Gateway is 192.168.x.1 or 10.0.x.1 → **NAT mode** (may need port forwarding)
- No gateway → **Host-only or disconnected** ❌

#### 4. Check Disk Space (Critical!)
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

#### 5. Check for Port Conflicts
```bash
sudo lsof -i :8080
```

**If you see output (process listed):**
Something is using port 8080. Common culprits:
- Docker: `docker ps` to check
- Another OpenClaw instance: `openclaw gateway stop`
- Other dev tools

**Fix:** Either stop the conflicting service, or change OpenClaw port in `config.yaml`:
```yaml
gateway:
  bind: "127.0.0.1:8081"  # Use different port
```

#### 6. Identify Yourself (Telegram User ID)
**Before pairing**, you need your numeric Telegram ID:

**Method A - Bot (Easiest):**
1. Message @userinfobot on Telegram
2. It replies: "Id: 8748931722"
3. **Your ID:** `8748931722`

**Method B - Logs (After pairing attempt):**
```bash
openclaw logs --follow
# Send /start to your bot
# Look for: "from.id": 8748931722
```

**📝 Record These Values:**
```
VM IP Address: ____________________
Network Mode: _____________________ (Bridged/NAT/Host-only)
Node.js Status: ___________________ (None/v18/v22/Other)
Disk Usage: _______________________ (e.g., 45%)
Port 8080: ________________________ (Free/In Use)
Telegram User ID: _________________
```

**🔄 File Transfer Setup:**

To copy files between Mac and VM during setup:
- **VMware:** Enable Shared Folders: VM Settings → Options → Shared Folders → Always Enabled
- **Purpose:** Transfer guide files, configs, scripts
- **Security:** Disable after setup if you want maximum isolation

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

⚠️ **Before editing any config files, back them up:**

```bash
cp ~/.openclaw/config.yaml ~/.openclaw/config.yaml.bak
cp ~/.openclaw/exec-approvals.json ~/.openclaw/exec-approvals.json.bak
```

If something breaks, restore with: `cp ~/.openclaw/config.yaml.bak ~/.openclaw/config.yaml`

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

### Edge Case 10: VM IP Changes Every Reboot (Dynamic IP)

**The Problem:**
Bridged mode gives your VM an IP from your router's DHCP pool. This IP can change when:
- VM reboots
- Mac sleeps/wakes
- Router restarts

**Symptoms:**
- OpenClaw worked yesterday, can't connect today
- `ip addr show` shows different IP
- Telegram bot seems offline

**Solutions:**

**Option A: Static IP (Recommended)**
Configure your router to reserve an IP for your VM's MAC address:
1. `ip link show` → Note MAC address (ether value)
2. Router admin panel → DHCP Reservations
3. Assign fixed IP to your VM's MAC

**Option B: Use NAT + Port Forwarding (More stable)**
1. Switch VM to NAT mode
2. Set up port forwarding: Host 8080 → Guest 8080
3. Always access via `http://localhost:8080` from Mac

**Option C: mDNS/Avahi (Discovery-based)**
Install mDNS on VM for hostname.local access:
```bash
sudo apt install avahi-daemon
# Access via http://your-vm-name.local:8080
```

---

### Edge Case 11: VPN Blocking VM Network

**The Problem:**
Your Mac VPN (especially corporate VPNs) can block VM bridged networking.

**Symptoms:**
- VM has no internet
- `ip addr show` shows no IP
- OpenClaw can't reach Telegram APIs

**Solutions:**

**Option A: Disconnect VPN (Simplest)**
Pause VPN during setup, reconnect after.

**Option B: Use NAT Mode Instead**
1. Switch VM to NAT
2. Port forward 8080
3. VM uses Mac's VPN connection

**Option C: Split Tunnel (Advanced)**
Configure VPN to exclude VM network range.

---

### Edge Case 12: Mac Firewall Blocking VM

**The Problem:**
macOS firewall or third-party tools (Little Snitch, LuLu) block bridged networking.

**Symptoms:**
- VM gets IP but can't reach internet
- Can't ping VM from Mac
- OpenClaw partially works

**Solutions:**

**Check macOS Firewall:**
System Settings → Network → Firewall → Disable temporarily

**Check Third-Party Tools:**
- Little Snitch: Allow VMware/Parallels network access
- LuLu: Check for blocked connections
- Other security software: Whitelist VM processes

**Test:**
```bash
# From Mac terminal
ping <VM_IP_ADDRESS>
# Should get responses
```

---

### Edge Case 13: Time Drift Causing Auth Failures

**The Problem:**
VM time different from real time causes authentication failures.

**Symptoms:**
- "Pairing expired" errors immediately
- Telegram auth failures
- SSL certificate errors

**Fix:**
```bash
# Set timezone (replace with your timezone)
sudo timedatectl set-timezone America/Los_Angeles

# Install and enable NTP
sudo apt install -y ntp
sudo systemctl enable ntp
sudo systemctl start ntp

# Verify time sync
timedatectl status
# Should show: System clock synchronized: yes
```

---

### Edge Case 14: Exposed Telegram Token (Security Breach)

**The Problem:**
You accidentally committed `config.yaml` to public GitHub with bot token visible.

**Immediate Actions:**
1. **Revoke the token NOW:**
   - Message @BotFather
   - Send: `/revoke`
   - Select your bot
   - Token is now dead

2. **Generate new token:**
   - @BotFather → `/newbot` or `/token`
   - Update `config.yaml` with new token

3. **If committed to GitHub:**
   - Delete the repository or make it private
   - Even after deleting, token may be in Git history
   - Revoking is the only true fix

**Prevention:**
```bash
# Add to .gitignore (before any git init)
echo "config.yaml" >> .gitignore
echo "exec-approvals.json" >> .gitignore
echo "*.token" >> .gitignore
```

**Never commit:**
- Bot tokens
- API keys
- Passwords
- `exec-approvals.json` (contains sensitive paths)

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

### Device-Specific Considerations

**Apple Silicon Macs (M1/M2/M3):**
- Use UTM or Parallels (VMware Fusion works but is slower via Rosetta)
- Choose ARM64 Ubuntu image (not AMD64)
- Expect better battery life than Intel VMs

**Intel Macs:**
- All hypervisors work well
- AMD64 Ubuntu image is fine
- VMware Fusion performs best

**8GB RAM Macs:**
- Reduce VM RAM to 2-3GB (still works, slower)
- Close other apps before starting VM
- Consider swap space: `sudo fallocate -l 2G /swapfile`

**Older macOS versions (pre-Ventura):**
- VMware Fusion 12 works on Monterey
- Some features may differ slightly
- Tested primarily on Ventura/Sonoma

**Corporate Macs (MDM managed):**
- Check if virtualization is allowed by IT
- Some MDM policies block VM creation
- May need admin password for hypervisor install

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
