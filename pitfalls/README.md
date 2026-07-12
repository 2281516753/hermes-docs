# Claude Code on WSL2 — Pitfalls & Solutions

[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude-Code-D97757)](https://claude.ai/code)
[![Platform: WSL2](https://img.shields.io/badge/Platform-WSL2-blue)](https://learn.microsoft.com/en-us/windows/wsl/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-26.04-E95420?logo=ubuntu)](https://ubuntu.com/)
[![DeepSeek](https://img.shields.io/badge/Model-DeepSeek_V4-brightgreen)](https://deepseek.com/)

> [📖 中文版 / Chinese Version](README_CN.md)

A living collection of problems and solutions encountered while installing, configuring, and using Claude Code on WSL2 (Ubuntu 26.04).

> Formerly a Hermes Agent pitfalls log. Hermes-specific content has been archived at the bottom.

> Continuously updated — new pitfalls added as encountered.

---

## Table of Contents

- [1. Installing Claude Code](#1-installing-claude-code)
- [2. API & Model Configuration](#2-api--model-configuration)
- [3. Claude Code Configuration System](#3-claude-code-configuration-system)
- [4. Networking & Proxy](#4-networking--proxy)
- [5. MCP Servers](#5-mcp-servers)
- [6. Email Integration](#6-email-integration)
- [7. Tool Installation](#7-tool-installation)
- [8. System Integration](#8-system-integration)
- [9. Quick Troubleshooting](#9-quick-troubleshooting)
- [Environment Info](#environment-info)
- [Archive: Hermes Agent Pitfalls](#archive-hermes-agent-pitfalls)

---

## 1. Installing Claude Code

### 1.1 npm global install blocked by GFW

**Symptom**: `npm install -g @anthropic-ai/claude-code` times out or SSL error.

**Cause**: npm registry is slow in mainland China; `@anthropic-ai` packages need proxy.

**Fix**:

```bash
# Option 1: Set npm proxy
npm config set proxy http://127.0.0.1:7890
npm config set https-proxy http://127.0.0.1:7890
npm install -g @anthropic-ai/claude-code

# Option 2: One-off with npx
HTTP_PROXY=http://127.0.0.1:7890 npx @anthropic-ai/claude-code --version
```

### 1.2 Node.js version requirement

Claude Code requires Node.js >= 18:

```bash
node --version   # Should be >= v18.0.0
```

Upgrade with nvm if needed:

```bash
nvm install --lts
nvm use --lts
```

### 1.3 `claude` command not found after install

**Symptom**: `claude: command not found` after npm install.

**Fix**:

```bash
# Check npm global prefix
npm config get prefix

# Add to PATH (~/.bashrc)
echo 'export PATH="$(npm config get prefix)/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 1.4 Cannot reach Anthropic API from China

**Symptom**: Claude Code fails to connect on startup.

**Cause**: `api.anthropic.com` is blocked in mainland China.

**Fix**: Use DeepSeek API compatibility layer (see next section), or configure HTTP proxy:

```bash
export HTTP_PROXY=http://127.0.0.1:7890
export HTTPS_PROXY=http://127.0.0.1:7890
```

---

## 2. API & Model Configuration

### 2.1 DeepSeek API as Anthropic replacement

Claude Code can connect to DeepSeek's Anthropic-compatible endpoint via environment variables:

```bash
export ANTHROPIC_AUTH_TOKEN="sk-your-deepseek-api-key"
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
export ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-v4-pro[1m]"
export ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-v4-pro[1m]"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek-v4-pro"
export ANTHROPIC_MODEL="deepseek-v4-pro[1m]"
```

> ⚠️ `ANTHROPIC_AUTH_TOKEN` should contain your DeepSeek API Key (`sk-` prefix), NOT an Anthropic token.

### 2.2 Getting a DeepSeek API Key

1. Visit [platform.deepseek.com](https://platform.deepseek.com/)
2. Register / login
3. API Keys → Create API Key
4. Copy and save the key

### 2.3 Checking balance

DeepSeek doesn't provide a standard Anthropic balance API. Check via the DeepSeek web dashboard.

Community statusline plugins can display balance in real-time in the Claude Code status bar.

### 2.4 DeepSeek API breaks when routed through proxy

**Symptom**: Correct config but API returns errors or times out.

**Cause**: DeepSeek is a domestic (China) service — routing through proxy breaks it.

**Fix**: In whitelist proxy mode, domestic traffic is direct by default. If using blacklist mode, ensure:

```yaml
- DOMAIN-SUFFIX,deepseek.com,DIRECT
- DOMAIN-SUFFIX,deepseek.io,DIRECT
```

---

## 3. Claude Code Configuration System

### 3.1 settings.json vs settings.local.json

| File | Purpose | Git-tracked? |
|------|---------|:--:|
| `.claude/settings.json` | Project-level shared config | ✅ |
| `.claude/settings.local.json` | Personal config (permissions, tokens) | ❌ |
| `~/.claude/settings.json` | User global config | - |

### 3.2 Permission system

By default Claude Code prompts for approval on every action. Optimize:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm *)",
      "Bash(git *)",
      "Bash(python *)",
      "WebFetch",
      "WebSearch"
    ],
    "deny": [
      "Bash(rm -rf *)"
    ],
    "additionalDirectories": [
      "C:\\Users\\22815"
    ]
  }
}
```

### 3.3 "Trust folder" prompt every startup

**Symptom**: Claude Code asks to trust the folder on every launch.

**Fix**: Add working directory to `additionalDirectories`:

```json
{
  "permissions": {
    "additionalDirectories": ["C:\\Users\\22815"]
  }
}
```

### 3.4 Auto-Memory

Claude Code supports cross-session memory via MEMORY.md:

```json
{
  "autoMemoryEnabled": true,
  "autoMemoryDirectory": "~/.claude/projects/C--Users-22815/memory/"
}
```

### 3.5 Permission modes

| Mode | Behavior | Best for |
|------|----------|----------|
| `default` | Prompt for each action | Daily dev |
| `acceptEdits` | Auto-accept file edits | Trusted refactors |
| `auto` | Auto most operations | Batch tasks |
| `plan` | Plan first, then execute | Complex changes |

### 3.6 Thinking mode & effort level

```bash
claude --effort high
```

Or in settings:
```json
{
  "effortLevel": "high",
  "alwaysThinkingEnabled": true
}
```

---

## 4. Networking & Proxy

### 4.1 mihomo fake-ip breaks QQ mail

**Symptom**: `imap.qq.com` / `smtp.qq.com` resolve to `198.18.0.x` (fake IP).

**Fix**: Switch to `redir-host` mode:

```yaml
dns:
  enable: true
  enhanced-mode: redir-host
```

### 4.2 ISP DPI blocks non-standard port TLS

**Symptom**: TCP connects but TLS handshake times out on ports 993/465.

**Fix**: HTTP CONNECT tunnel through proxy:

```bash
python3 hermes-email-tunnel 11993 imap.qq.com 993 &
python3 hermes-email-tunnel 11587 smtp.qq.com 587 &
```

Then point `/etc/hosts` entries to `127.0.0.1`.

### 4.3 Blacklist → Whitelist proxy mode

**Symptom**: All traffic routes through proxy unnecessarily.

**Fix**: Whitelist only blocked domains:

```yaml
rules:
  - DOMAIN-SUFFIX,github.com,🔰 Proxy
  - DOMAIN-SUFFIX,google.com,🔰 Proxy
  - MATCH,DIRECT
```

### 4.4 WSL2 DNS hijacked by Windows host

**Symptom**: `getent hosts` returns fake IPs.

**Fix**: Use `/etc/hosts` for critical domain static mappings instead of modifying `/etc/resolv.conf`.

### 4.5 `git push` to GitHub needs proxy

```bash
git config --global http.https://github.com.proxy http://127.0.0.1:7890
```

### 4.6 WSL2 network latency causes timeouts

WSL2 networking adds a virtualization layer. Increase timeouts in Claude Code settings where possible.

---

## 5. MCP Servers

### 5.1 MCP configuration

Claude Code uses `.mcp.json`:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_***"
      }
    },
    "fetch": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-fetch"]
    },
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp"]
    }
  }
}
```

### 5.2 MCP tools not appearing

**Fix**:
1. Ensure `enableAllProjectMcpServers: true` or server is in `enabledMcpjsonServers`
2. Restart Claude Code (MCP loads at startup)
3. Check with `claude mcp list`

```bash
claude mcp list
claude mcp add github
claude mcp approve github
```

### 5.3 Playwright Chromium not supported on Ubuntu 26.04

Ubuntu 26.04 is too new for Playwright. Use Docker or wait for official support. Web fetch/search still works via MCP fetch server.

---

## 6. Email Integration

### 6.1 QQ Mail SMTP auth code

1. Visit https://mail.qq.com
2. Settings → Account → POP3/IMAP/SMTP
3. Enable IMAP/SMTP, follow SMS verification
4. Get 16-character auth code

### 6.2 himalaya config

```toml
[accounts.qq]
default = true
email = "your_qq_number@qq.com"
imap.host = "imap.qq.com"
imap.port = 993
smtp.host = "smtp.qq.com"
smtp.port = 465
```

---

## 7. Tool Installation

### 7.1 ripgrep not in Ubuntu 26.04 apt

```bash
curl -fsSL -o /tmp/ripgrep.deb \
  "https://github.com/BurntSushi/ripgrep/releases/download/14.1.1/ripgrep_14.1.1-1_amd64.deb"
sudo dpkg -i /tmp/ripgrep.deb
```

### 7.2 astral.sh (uv) blocked by GFW

Use `npx` instead — Claude Code runs MCP servers via Node.js, no uv needed.

---

## 8. System Integration

### 8.1 Email tunnel as systemd service

```bash
systemctl --user enable --now email-tunnel
sudo loginctl enable-linger $USER   # Required for WSL2
```

### 8.2 WSL2 shutdown kills services

Enable linger + systemd to keep services alive after terminal closes.

### 8.3 Claude Code plugin management

```bash
claude plugins list
claude plugins search <keyword>
claude plugins install <plugin-name>
```

---

## 9. Quick Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| `claude` not found | npm bin not in PATH | Check `npm config get prefix`, add to PATH |
| API returns 401 | Invalid/expired token | Regenerate DeepSeek API Key |
| API timeout | Proxy misconfigured | Check `ANTHROPIC_BASE_URL` and proxy status |
| MCP tools missing | Server not approved | `claude mcp list`, `claude mcp approve` |
| Trust prompt every time | `additionalDirectories` not set | Add working dir to permissions |
| Too many permission prompts | No allow rules | Add `permissions.allow` in settings.local.json |
| ripgrep not found | Not installed | Install from GitHub Releases |
| `git push` fails | Git not using proxy | `git config --global http.https://github.com.proxy` |
| Services lost after WSL2 restart | No systemd enable | `systemctl --user enable` + `loginctl enable-linger` |

---

## Environment Info

| Component | Detail |
|-----------|--------|
| OS | WSL2 Ubuntu 26.04 x86_64 |
| Proxy | mihomo v1.19.25 (whitelist mode) |
| Claude Code | latest (npm global) |
| Model | DeepSeek V4 Pro (via ANTHROPIC_BASE_URL) |
| Node | v22.22.3 |
| Python | 3.14.4 |
| MCP | github, fetch, playwright, desktop-commander |

---

## Archive: Hermes Agent Pitfalls

> Hermes-specific content has been archived. See the Chinese README for the full Hermes archive, or check git history for the original README.

Key Hermes features no longer relevant:
- `hermes config migrate` / `hermes doctor --fix`
- `smart_model_routing` with cheap/expensive model switching
- Hermes CN Desktop app
- Telegram/QQ Bot Gateway
- Hermes cron system

The generic infrastructure content (proxy, DNS, email tunnels, tool installation) has been merged into the sections above.

---

## Author

Wang Jiong — Network Engineering student, cloud computing career path.

[GitHub](https://github.com/2281516753)

---

## Related Projects

- [wsl-dev-setup](https://github.com/2281516753/wsl-dev-setup) — WSL2 dev environment one-click setup
- [hermes-tools](https://github.com/2281516753/hermes-tools) — Email tunnel tool
- [cloud-lab](https://github.com/2281516753/cloud-lab) — Cloud infrastructure lab

---

MIT License