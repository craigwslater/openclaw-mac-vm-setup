# 🔐 OpenClaw Mac VM Setup
> Security-first OpenClaw deployment in an isolated Linux VM on macOS

[![OpenClaw](https://img.shields.io/badge/OpenClaw-2026.4.1-blue)](https://docs.openclaw.ai)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04-orange)](https://ubuntu.com)
[![VMware](https://img.shields.io/badge/VMware-Fusion-brightgreen)]()

## 🎯 The Goal

**Give OpenClaw full permissions to be useful, while keeping it fully isolated from your sensitive data.**

OpenClaw is an AI agent framework that can execute commands on your system. Instead of wrestling with restrictive permissions or — worse — giving it direct access to your Mac, this setup uses **hardware-level virtualization** to create a secure boundary.

The VM gets complete freedom. Your Mac stays pristine.

```
┌─────────────────┐     ┌─────────────────┐
│   YOUR MAC      │     │   UBUNTU VM     │
│   (pristine)    │ ←→  │   (OpenClaw)    │
│                 │ VM  │                 │
│ • Documents     │wall │ • Full shell    │
│ • SSH keys      │     │ • Network       │
│ • Passwords     │     │ • File system   │
│ • Browser       │     │ • Packages      │
└─────────────────┘     └─────────────────┘
```

## 🛡️ Why This Matters

| Approach | Security | Friction | Recommendation |
|----------|----------|------------|----------------|
| OpenClaw on Mac (full) | ❌ Low | ✅ None | **Don't do this** |
| OpenClaw on Mac (restricted) | ⚠️ Medium | ❌ High | Constant approval battles |
| **OpenClaw in VM (this guide)** | ✅ **High** | ✅ **None** | **Best of both** |

**The VM is a blast containment chamber.** If something goes wrong, you lose a 40GB VM, not your digital life.

## 📚 Guides

| Guide | Best For | Time | Link |
|-------|----------|------|------|
| **QUICKSTART.md** | Experienced users who want commands now | 30-45 min | [View](QUICKSTART.md) |
| **Complete Setup** | First-timers who want hand-holding | 45-60 min | [View](openclaw-mac-vm-complete-setup.md) |

### Which guide should I use?

**QUICKSTART if:**
- You're comfortable with Linux
- You want copy-paste commands
- You've used OpenClaw before

**Complete Setup if:**
- This is your first time
- You want screenshots and expected outputs
- You want to understand *why* each step matters

## 🏗️ The Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Hypervisor** | VMware Fusion (or UTM for Apple Silicon) | Hardware isolation |
| **Guest OS** | Ubuntu 22.04 LTS | Clean, minimal base |
| **Runtime** | Node.js 22 | Required for OpenClaw |
| **AI Agent** | OpenClaw 2026.4.1 | Command execution & automation |
| **Interface** | Telegram Bot | Mobile-friendly control |
| **Security** | `exec-approvals.json` | Auto-approve for zero friction |

**Tested on:** VMware Fusion + Ubuntu 22.04 + OpenClaw 2026.4.1

## ⚡ TL;DR (The 30-Second Version)

```bash
# 1. Install Node.js 22 (NOT v18 from apt!)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# 2. Install OpenClaw
sudo npm install -g openclaw

# 3. Initialize and configure
openclaw setup
# Edit ~/.openclaw/config.yaml with your Telegram token
# Edit ~/.openclaw/exec-approvals.json with security settings

# 4. Start and pair
openclaw gateway start
openclaw pairing approve telegram <CODE>

# 5. Test
openclaw exec -- ls -la ~/.openclaw/
```

**See [QUICKSTART.md](QUICKSTART.md) for complete commands, configs, and troubleshooting.**

## 🔑 Key Configuration

The magic happens in two files:

### `~/.openclaw/config.yaml` — Gateway & Channels
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

### `~/.openclaw/exec-approvals.json` — Security Policy ⭐
```json
{
  "version": 1,
  "agents": {
    "main": {
      "security": "full",
      "ask": "off",
      "autoAllowSkills": true
    }
  }
}
```

**`"security": "full"` + `"ask": "off"`** = Zero friction, but contained within the VM.

## 🚨 Common Issues (We Hit These So You Don't Have To)

| Problem | Solution |
|---------|----------|
| **Node.js v18 installed** | Remove it: `sudo apt remove nodejs npm && sudo apt autoremove`, then reinstall via NodeSource |
| **"Exec approval required"** | Set `"security": "full"` in **both** `defaults` AND `agents.main` in `exec-approvals.json` |
| **Config changes not working** | Restart gateway: `openclaw gateway restart` |
| **Telegram buttons not working** | Use auto-approve config above (we found buttons unreliable) |

**Full troubleshooting:** See the Edge Cases section in [complete setup guide](openclaw-mac-vm-complete-setup.md).

## ✅ Validation Checklist

Run these to verify your setup:

```bash
# 1. Node.js v22+
node --version  # Should show v22.x.x

# 2. OpenClaw installed
openclaw --version  # Should show version

# 3. Gateway running
openclaw gateway status  # Should show "running"

# 4. Exec works (no approval prompt!)
openclaw exec -- ls -la ~/.openclaw/

# 5. Telegram works
openclaw message send --channel telegram --to YOUR_ID --message "Test"
```

## 🎓 Workflow Options

### Option 1: Self-Guided (Manual)
Follow the guides step-by-step. Check Common Gotchas when stuck.

### Option 2: AI-Assisted (Recommended)
Copy the complete guide into ChatGPT/Claude and use this prompt:

```
I want to set up OpenClaw on an Ubuntu VM on my Mac using the attached guide.

Walk me through this step-by-step. Before each step, tell me what we're doing and why.
After I run each command, ask me to share the output so you can verify it matches the expected results.

If something fails, reference the Edge Cases section and help me fix it before moving on.
```

### Option 3: Hybrid
Start self-guided, switch to AI when you hit an error.

## 🔒 Security Deep Dive

### What the VM protects:

| Your Mac (Protected) | VM (Isolated) |
|----------------------|---------------|
| Documents & files | OpenClaw workspace only |
| SSH keys (~/.ssh) | Separate key management |
| Browser data (cookies, passwords) | Clean browser if any |
| System settings | Cannot modify |
| Other user accounts | No access |

### What OpenClaw CAN do in the VM:
- Execute any shell command
- Read/write any VM file
- Install/remove packages
- Access network (if allowed)
- Run services

### What OpenClaw CANNOT do:
- Access your Mac files
- Read your Mac's SSH keys
- See browser data
- Escape the VM (without a vulnerability)

### Recovery:
If the VM gets compromised: **Delete VM → Restore from snapshot → 5 minutes later you're back**

## 🚀 Optional Add-ons

**Local AI with Ollama:**
```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.2
# Add to config.yaml: models.default: ollama/llama3.2
```

**Remote access with Tailscale:**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

## 🤝 Contributing

Found an edge case we missed? The guides document 9 real issues we encountered.

**Ways to contribute:**
- Open an issue with your error message
- Submit a PR with additional edge cases
- Share your own security improvements

## 📄 License

MIT License — Use freely, attribution appreciated.

## 🙏 Credits

- [OpenClaw](https://docs.openclaw.ai) — The AI agent framework
- [VMware Fusion](https://www.vmware.com/products/fusion.html) — Virtualization
- [Ubuntu](https://ubuntu.com) — Guest OS

---

**Questions?** Open an issue or reach out on [OpenClaw Discord](https://discord.com/invite/clawd).

**Tested:** VMware Fusion + Ubuntu 22.04 + OpenClaw 2026.4.1  
**Last Updated:** April 2026
