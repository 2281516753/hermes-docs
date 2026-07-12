# Claude Code on WSL2 — Complete Setup Guide

[![GitHub stars](https://img.shields.io/github/stars/2281516753/hermes-setup-guide?style=social)](https://github.com/2281516753/hermes-setup-guide)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Platform: WSL2](https://img.shields.io/badge/Platform-WSL2-blue)](https://learn.microsoft.com/en-us/windows/wsl/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-26.04-E95420?logo=ubuntu)](https://ubuntu.com/)
[![Claude Code](https://img.shields.io/badge/Claude-Code-D97757)](https://claude.ai/code)

> [📖 中文版 / Chinese Version](README_CN.md)

A step-by-step guide to set up Claude Code on WSL2 from scratch — covering proxy config, DeepSeek API, QQ Mail, MCP servers, and systemd services.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [1. Install Claude Code](#1-install-claude-code)
- [2. Configure API & Model](#2-configure-api--model)
- [3. Configure mihomo Proxy](#3-configure-mihomo-proxy)
- [4. Install ripgrep](#4-install-ripgrep)
- [5. Claude Code Optimization](#5-claude-code-optimization)
- [6. QQ Mail Setup](#6-qq-mail-setup)
- [7. MCP Servers](#7-mcp-servers)
- [8. systemd Services](#8-systemd-services)
- [9. Pitfalls Reference](#9-pitfalls-reference)
- [Verification Checklist](#verification-checklist)

---

## Prerequisites

| Requirement | Detail | Notes |
|-------------|--------|-------|
| **Windows** | 10 22H2+ or 11 | WSL2 support needed |
| **WSL2** | Installed & default | `wsl --set-default-version 2` |
| **Ubuntu** | 26.04 | From Microsoft Store |
| **systemd** | Enabled | `systemd=true` in `/etc/wsl.conf` |
| **Node.js** | >= 18.x | Claude Code runtime |

### Enable systemd

Edit `/etc/wsl.conf`:

```ini
[boot]
systemd=true
```

Restart WSL from PowerShell:

```powershell
wsl --shutdown
wsl
```

Verify:

```bash
systemctl is-system-running
```

---

## 1. Install Claude Code

### 1.1 Install Node.js (via nvm)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install --lts
nvm use --lts
node --version   # >= 20.x
npm --version
```

### 1.2 Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
claude --version
```

> 🐛 If npm times out, configure proxy first (see Section 3).

### 1.3 `claude` not found?

```bash
npm config get prefix
echo 'export PATH="$(npm config get prefix)/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

## 2. Configure API & Model

### 2.1 Get DeepSeek API Key

1. Visit [platform.deepseek.com](https://platform.deepseek.com/)
2. Register / login
3. API Keys → Create API Key
4. Copy and save

### 2.2 Configure Environment Variables

In `~/.claude/settings.json`:

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "sk-your-deepseek-api-key",
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-pro",
    "ANTHROPIC_MODEL": "deepseek-v4-pro[1m]"
  },
  "model": "deepseek-v4-pro"
}
```

> 💡 `[1m]` suffix enables 1M token context window.

### 2.3 Verify

```bash
claude "Hello, introduce yourself"
```

---

## 3. Configure mihomo Proxy

### 3.1 Install

```bash
MCLASH_VER=$(curl -s https://api.github.com/repos/MetaCubeX/mihomo/releases/latest | grep tag_name | cut -d '"' -f 4)
wget "https://github.com/MetaCubeX/mihomo/releases/download/${MCLASH_VER}/mihomo-linux-amd64-${MCLASH_VER}.gz" -O mihomo.gz
gzip -d mihomo.gz
chmod +x mihomo
sudo mv mihomo /usr/local/bin/
```

### 3.2 Whitelist Mode Config

Create `~/.config/mihomo/config.yaml`:

```yaml
mixed-port: 7890
mode: rule
log-level: info

dns:
  enable: true
  enhanced-mode: redir-host
  nameserver:
    - 223.5.5.5
    - 119.29.29.29

rules:
  - DOMAIN-SUFFIX,github.com,🚀 Proxy
  - DOMAIN-SUFFIX,google.com,🚀 Proxy
  - DOMAIN-SUFFIX,npmjs.org,🚀 Proxy
  - MATCH,DIRECT
```

### 3.3 Git Proxy

```bash
git config --global http.https://github.com.proxy http://127.0.0.1:7890
```

---

## 4. Install ripgrep

```bash
curl -fsSL -o /tmp/ripgrep.deb \
  "https://github.com/BurntSushi/ripgrep/releases/download/14.1.1/ripgrep_14.1.1-1_amd64.deb"
sudo dpkg -i /tmp/ripgrep.deb
rg --version
```

---

## 5. Claude Code Optimization

### 5.1 Permission Config

`~/.claude/settings.local.json`:

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
    "additionalDirectories": ["C:\\Users\\22815"],
    "defaultMode": "default"
  },
  "enableAllProjectMcpServers": true
}
```

### 5.2 Auto-Memory

```json
{
  "autoMemoryEnabled": true,
  "autoMemoryDirectory": "~/.claude/projects/C--Users-22815/memory/"
}
```

### 5.3 Status Line Plugin (DeepSeek Balance)

```json
{
  "statusLine": {
    "type": "command",
    "command": "node --experimental-strip-types ~/.claude/plugins/deepseek-balance-statusline/index.ts"
  }
}
```

---

## 6. QQ Mail Setup

### 6.1 Get QQ Mail Auth Code

1. Visit [mail.qq.com](https://mail.qq.com/)
2. Settings → Account → POP3/IMAP/SMTP
3. Enable IMAP/SMTP, send SMS verification
4. Get 16-character auth code

### 6.2 Install himalaya

```bash
wget https://github.com/pimalaya/himalaya/releases/latest/download/himalaya-linux-x86_64.tar.gz
tar xzf himalaya-linux-x86_64.tar.gz
sudo mv himalaya /usr/local/bin/
```

### 6.3 Configure

`~/.config/himalaya/config.toml`:

```toml
[accounts.qq]
default = true
email = "your_qq_number@qq.com"
imap.host = "imap.qq.com"
imap.port = 993
smtp.host = "smtp.qq.com"
smtp.port = 465
```

### 6.4 HTTP CONNECT Tunnel (bypass ISP DPI)

If IMAP/SMTP ports are blocked:

```bash
git clone https://github.com/2281516753/hermes-tools.git
cp hermes-tools/hermes-email-tunnel ~/.local/bin/
chmod +x ~/.local/bin/hermes-email-tunnel

hermes-email-tunnel 11993 imap.qq.com 993 &
hermes-email-tunnel 11587 smtp.qq.com 587 &
```

Update himalaya to use `127.0.0.1:<tunnel_port>`.

---

## 7. MCP Servers

### 7.1 Config

`~/.claude/.mcp.json`:

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
    }
  }
}
```

### 7.2 GitHub Token

1. [github.com/settings/tokens](https://github.com/settings/tokens)
2. Generate new token (classic) with `repo` scope

### 7.3 Approve Servers

```bash
claude mcp list
claude mcp approve github
claude mcp approve fetch
```

---

## 8. systemd Services

### 8.1 Email Tunnel

```bash
# Create service file at ~/.config/systemd/user/email-tunnel.service
systemctl --user daemon-reload
systemctl --user enable --now email-tunnel

# Required for WSL2
sudo loginctl enable-linger $USER
```

### 8.2 mihomo (Optional)

Create `~/.config/systemd/user/mihomo.service`:

```ini
[Unit]
Description=mihomo proxy
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/mihomo -d %h/.config/mihomo
Restart=on-failure

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now mihomo
```

---

## 9. Pitfalls Reference

See [claude-code-pitfalls](https://github.com/2281516753/hermes-pitfalls) for a comprehensive collection of common issues and solutions.

### Quick Reference

| Problem | Fix |
|---------|-----|
| `claude` not found | Add npm bin to PATH |
| API 401 | Regenerate DeepSeek API Key |
| API timeout | DeepSeek must be DIRECT (no proxy) |
| Trust prompt every startup | Add dir to `additionalDirectories` |
| MCP tools missing | `claude mcp approve <server>` |
| `git push` fails | `git config --global http.https://github.com.proxy` |
| WSL2 services lost | `loginctl enable-linger` + systemd enable |

### 故障排除详细指南

#### 1. 网络代理问题

```bash
# 检查代理是否运行
curl -x http://127.0.0.1:7890 https://api.deepseek.com/v1/models

# 检查 mihomo 进程
ps aux | grep mihomo

# 查看 mihomo 日志
journalctl -u mihomo -f

# 测试 DNS 解析
nslookup deepseek.com 127.0.0.1:5353
```

#### 2. API 连接问题

```bash
# 测试 DeepSeek API
curl https://api.deepseek.com/v1/models \
  -H "Authorization: Bearer sk-xxxxxxxx"

# 检查网络连通性
ping api.deepseek.com

# 检查端口是否开放
nc -zv api.deepseek.com 443
```

#### 3. MCP 服务器问题

```bash
# 测试 GitHub MCP
npx -y @modelcontextprotocol/server-github --help

# 检查 Node.js 版本
node --version  # 需要 >= 18.x

# 检查 npm 缓存
npm cache clean --force
```

#### 4. systemd 服务问题

```bash
# 查看服务状态
systemctl status hermes-gateway

# 查看服务日志
journalctl -u hermes-gateway -n 100

# 重启服务
sudo systemctl restart hermes-gateway

# 检查服务文件语法
systemd-analyze verify /etc/systemd/system/hermes-gateway.service
```

#### 5. WSL2 特有问题

```bash
# 检查 WSL 版本
wsl --version

# 重启 WSL
wsl --shutdown

# 检查 systemd 状态
systemctl is-system-running

# 检查网络接口
ip addr show
```

---

## Verification Checklist

- [ ] `claude --version` works
- [ ] `claude "Hello"` responds correctly
- [ ] MCP tools available (GitHub, Fetch)
- [ ] `rg --version` works
- [ ] `himalaya envelope list` works
- [ ] systemd services active
- [ ] Services survive WSL2 restart

---

## Related Projects

| Project | Description |
|---------|-------------|
| [claude-code-pitfalls](https://github.com/2281516753/hermes-pitfalls) | Claude Code pitfalls & solutions |
| [hermes-tools](https://github.com/2281516753/hermes-tools) | Email tunnel tool |
| [wsl-dev-setup](https://github.com/2281516753/wsl-dev-setup) | WSL2 dev env setup |
| [cloud-lab](https://github.com/2281516753/cloud-lab) | Cloud infrastructure lab |
| [net-auto](https://github.com/2281516753/net-auto) | Network automation tools |

---

## Author

Wang Jiong — Network Engineering student.

[GitHub](https://github.com/2281516753)

---

<p align="center">
  <b>Happy coding with Claude Code! 🚀</b>
</p>