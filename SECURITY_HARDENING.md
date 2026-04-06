# Security Hardening Guide

## Network Architecture Options

Choose your network setup based on your risk tolerance:

### Option A: NAT (Recommended Default)

**What it is:**
VM shares your Mac's network connection through NAT (Network Address Translation). VMware calls this "Share with my Mac."

**How it works:**
- VM gets its own isolated subnet (e.g., 192.168.37.x)
- Your Mac acts as a router for the VM
- VM can reach the internet but cannot directly access other devices on your home network

**Pros:**
- ✅ Good isolation from connected devices (smart speakers, TVs, etc.)
- ✅ Simple setup (default VMware option)
- ✅ No additional hardware needed
- ✅ File sharing still possible via VMware tools

**Cons:**
- ⚠️ File sharing requires VMware drag-and-drop or shared folders
- ⚠️ Slightly more complex than direct bridged mode

**Best for:**
- Most users (recommended default)
- Users with connected devices on their network
- Those who want security without complexity

**Setup Instructions:**
```bash
# In VMware: VM → Settings → Network Adapter
# Select: "Share with my Mac" (NAT mode)
# This is the DEFAULT - you're probably already using it

# Verify isolation:
ping 192.168.86.1  # Replace with your router IP - Should FAIL
ping 8.8.8.8       # Should SUCCEED (internet works)
```

---

### Option B: USB WiFi → Guest Network (Maximum Isolation)

**What it is:**
Use a USB WiFi adapter passed through to the VM, connecting directly to your router's Guest WiFi network.

**How it works:**
- Mac stays on your main network
- VM uses separate USB WiFi adapter on Guest network
- Complete physical network separation

**Pros:**
- ✅ Maximum isolation - VM cannot see your Mac or any connected devices
- ✅ True air-gap style separation (within same machine)
- ✅ Can still access internet
- ✅ Mac and VM on different networks simultaneously

**Cons:**
- ⚠️ Requires USB WiFi adapter purchase (~$15-25)
- ⚠️ Complex setup (drivers, passthrough)
- ⚠️ USB device must stay plugged in
- ⚠️ File transfer requires cloud or network tools

**Best for:**
- Security-conscious users with sensitive connected devices
- Users who need their Mac on main network but want VM isolated
- Those comfortable with hardware/driver configuration

**Setup Instructions:**

**Prerequisites:**
- USB WiFi adapter with Linux support (e.g., TP-Link Archer T3U, Netgear A6150)
- Router with Guest WiFi network enabled

**Step 1: Install Drivers in VM**
```bash
# Before passthrough, install drivers for your adapter
# For Realtek RTL8812BU chipsets (common):
cd /tmp
git clone https://github.com/RinCat/RTL88x2BU-Linux-Driver.git
cd RTL88x2BU-Linux-Driver
make
sudo make install
sudo modprobe 88x2bu

# Verify driver loaded
lsmod | grep 88x2bu
```

**Step 2: Configure VMware USB Passthrough**
```bash
# In VM:
sudo shutdown -h now
```

**On Mac:**
1. Plug in USB WiFi adapter
2. VMware: VM → Settings → USB & Bluetooth
3. Add USB Device → Select your WiFi adapter
4. Set to "Connect to VM"
5. Check "Connect at power on"
6. Start VM

**Step 3: Connect to Guest Network**
```bash
# In VM, verify adapter detected
lsusb | grep -i wifi

# List available networks
nmcli device wifi list

# Connect to Guest WiFi
nmcli device wifi connect "Your-Guest-SSID" password "your-password"

# Verify IP (should be Guest network range)
ip addr show | grep "inet " | grep -v "127.0.0.1"
```

**Step 4: Verify Isolation**
```bash
# Should SUCCEED
ping 8.8.8.8

# Should FAIL (cannot reach main network)
ping 192.168.86.1  # Your main router IP
```

---

## Decision Matrix

| Factor | NAT (Option A) | USB WiFi (Option B) |
|--------|---------------|---------------------|
| **Setup Complexity** | Low | High |
| **Cost** | Free | $15-25 |
| **Isolation Level** | Good | Maximum |
| **Mac Network Impact** | Shares connection | Independent |
| **File Sharing** | Easy (VMware) | Requires cloud/SCP |
| **Hardware Required** | None | USB WiFi adapter |

**Choose NAT if:** You want good security without complexity.
**Choose USB WiFi if:** You need your Mac on main network AND maximum VM isolation.

---

## Quick Decision (30 Seconds)

**Are you comfortable with hardware/driver setup?**
- **No** → Use NAT (default). Skip to [Token Security](#telegram-token-security).
- **Yes, and I need maximum isolation** → Use USB WiFi. See setup instructions above.

---

## Telegram Token Security

### The Risk

Your Telegram bot token is stored in `config.yaml` in **plain text**. If:
- VM is compromised → attacker steals token
- Token exposed in screenshot → anyone can use your bot
- Accidentally committed to GitHub → public token

**Consequences:**
- Attacker can spam from your bot
- Can read your bot's message history
- Can impersonate your agent
- Hard to revoke without @BotFather

### Secure Token Storage (Choose One)

**Option 1: Environment Variable (Recommended)**

```bash
# 1. Add to ~/.bashrc or ~/.profile:
export TELEGRAM_BOT_TOKEN="your-token-here"

# 2. Reload shell:
source ~/.bashrc

# 3. Update config.yaml:
channels:
  telegram:
    enabled: true
    botToken: "${TELEGRAM_BOT_TOKEN}"  # Uses env var
    dmPolicy: "pairing"
```

**Pros:**
- ✅ Token not in config file
- ✅ Can restrict file permissions on .bashrc
- ✅ Can use secrets management tools later

**Cons:**
- ⚠️ Must reload shell after reboot
- ⚠️ Slightly more complex setup

---

**Option 2: Separate Secrets File**

```bash
# 1. Create secrets file:
echo 'TELEGRAM_TOKEN="your-token"' > ~/.openclaw/secrets
chmod 600 ~/.openclaw/secrets  # Only owner can read

# 2. Source it in your shell config:
echo 'source ~/.openclaw/secrets' >> ~/.bashrc
```

**Pros:**
- ✅ Dedicated secrets file with restricted permissions
- ✅ Can exclude from backups/version control

**Cons:**
- ⚠️ More complex to implement
- ⚠️ Another file to manage

---

**Option 3: Accept Risk (With Protections)** ⚠️

If using plain text token:

```yaml
channels:
  telegram:
    enabled: true
    botToken: "123456789:ABCdefGHIjklMNOpqrsTUVwxyz"  # ⚠️ PLAIN TEXT
```

**Required Protections:**
```bash
# 1. Restrict config file permissions:
chmod 600 ~/.openclaw/config.yaml

# 2. Never commit to Git:
echo "config.yaml" >> ~/.gitignore
echo "exec-approvals.json" >> ~/.gitignore

# 3. Never share screenshots with token visible
# 4. Set calendar reminder to rotate token monthly
```

---

## Git Configuration Security

### The Risk

Accidentally committing sensitive files to GitHub exposes:
- Telegram bot tokens (public, anyone can use)
- API keys (can be abused, cost you money)
- Configuration paths (reveals system structure)

### .gitignore Protection

**Create .gitignore:**

```bash
cat > .gitignore << 'EOF'
# OpenClaw sensitive configs (contain tokens, API keys)
config.yaml
exec-approvals.json
openclaw.json
*.token
*.key

# Environment files (may contain secrets)
.env
.env.local
.env.*.local

# Secrets and credentials
secrets/
credentials/
*.secret

# Logs (can contain sensitive info)
logs/
*.log
log/
EOF
```

**Verify it works:**

```bash
# Check what Git would commit
git status

# config.yaml should NOT appear in "Changes to be committed"
# Should appear as "untracked" or not at all
```

**Why each pattern matters:**

| Pattern | Why Protected |
|---------|---------------|
| `config.yaml` | Contains Telegram bot token in plain text |
| `exec-approvals.json` | Reveals allowed commands, paths |
| `*.token` | Any API tokens you save as files |
| `.env*` | Environment files often have passwords |
| `*.log` | May contain sensitive command output |

**If you already committed sensitive files:**

```bash
# 1. Stop! Don't just delete and commit again
# 2. Use BFG Repo-Cleaner or git-filter-branch to remove from history
# 3. Rotate all exposed tokens immediately
# 4. Force push (if personal repo) or create new repo
```

---

## Complete Secure Setup Checklist

### Network Security
- [ ] Choose NAT (default) or USB WiFi (maximum)
- [ ] Verify VM cannot reach main network (isolation test)
- [ ] Document which network mode you're using

### Token Security
- [ ] Store token in environment variable OR restrict file permissions
- [ ] Add config files to .gitignore
- [ ] Never share screenshots showing token
- [ ] Set calendar reminder to rotate token monthly

### VM Security
- [ ] Disable unnecessary shared folders
- [ ] Enable automatic security updates
- [ ] Set up weekly log rotation
- [ ] Configure disk space alerts (>80%)

### Monitoring
- [ ] Review `~/.openclaw/exec-approvals.json` monthly
- [ ] Check for unusual processes: `ps aux | head -20`
- [ ] Monitor network connections: `ss -tulnp`
- [ ] Verify no unexpected outbound traffic

---

## Quick Security Audit

Run this monthly:

```bash
#!/bin/bash
echo "=== OpenClaw Security Audit ==="
echo "Date: $(date)"
echo ""

echo "1. Config file permissions:"
ls -la ~/.openclaw/config.yaml | awk '{print $1 " " $9}'

echo ""
echo "2. Disk usage:"
df -h / | awk 'NR==2 {print "Disk: " $5 " full"}'

echo ""
echo "3. OpenClaw service status:"
openclaw gateway status 2>/dev/null | grep -E "Listening|Runtime" | head -2

echo ""
echo "4. Active network connections:"
ss -tulnp 2>/dev/null | grep ":8080" | head -2

echo ""
echo "5. Recent memory usage:"
free -h | grep "Mem:"

echo ""
echo "=== End Audit ==="
```

Save as `~/security-audit.sh` and run monthly.
