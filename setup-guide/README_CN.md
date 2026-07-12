# Claude Code WSL2 完整安装配置指南

[![GitHub stars](https://img.shields.io/github/stars/2281516753/hermes-setup-guide?style=social)](https://github.com/2281516753/hermes-setup-guide)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Platform: WSL2](https://img.shields.io/badge/Platform-WSL2-blue)](https://learn.microsoft.com/en-us/windows/wsl/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-26.04-E95420?logo=ubuntu)](https://ubuntu.com/)
[![Claude Code](https://img.shields.io/badge/Claude-Code-D97757)](https://claude.ai/code)
[![DeepSeek](https://img.shields.io/badge/Model-DeepSeek_V4-brightgreen)](https://deepseek.com/)

一份从零开始的详细指南，带你在 WSL2 上从空白环境到完全可用的 Claude Code，涵盖代理、邮箱、MCP、systemd 服务化等所有关键环节。

---

## 目录

- [前置条件](#前置条件)
- [1. 安装 Claude Code](#1-安装-claude-code)
- [2. 配置 API Key 和模型](#2-配置-api-key-和模型)
- [3. 配置代理 mihomo 白名单模式](#3-配置代理-mihomo-白名单模式)
- [4. 安装工具 ripgrep](#4-安装工具-ripgrep)
- [5. Claude Code 配置优化](#5-claude-code-配置优化)
- [6. 配置 QQ 邮箱](#6-配置-qq-邮箱)
- [7. 配置 MCP 服务器](#7-配置-mcp-服务器)
- [8. systemd 服务化](#8-systemd-服务化)
- [9. 踩坑参考](#9-踩坑参考)
- [验证清单](#验证清单)
- [环境信息表](#环境信息表)

---

## 前置条件

开始之前，请确保满足以下条件：

| 条件 | 要求 | 说明 |
|------|------|------|
| **Windows** | Windows 10 22H2+ 或 Windows 11 | 需要 WSL2 支持 |
| **WSL2** | 已安装并设为默认版本 | `wsl --set-default-version 2` |
| **Ubuntu** | Ubuntu 26.04 (Noble Numbat) | 在 Microsoft Store 安装 |
| **systemd** | 已启用 | `/etc/wsl.conf` 中设置 `systemd=true` |
| **Node.js** | >= 18.x | Claude Code 运行环境 |

### 启用 systemd

WSL2 默认可能未启用 systemd。编辑 `/etc/wsl.conf`：

```ini
[boot]
systemd=true
```

然后在 PowerShell 中重启 WSL：

```powershell
wsl --shutdown
wsl
```

验证：

```bash
systemctl is-system-running
# 输出应为 running 或 degraded
```

---

## 1. 安装 Claude Code

### 1.1 安装 Node.js（推荐使用 nvm）

```bash
# 安装 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# 重新加载 shell 配置
source ~/.bashrc

# 安装 Node.js LTS
nvm install --lts
nvm use --lts

# 验证
node --version   # 应 >= 20.x
npm --version
```

### 1.2 安装 Claude Code

```bash
# 全局安装
npm install -g @anthropic-ai/claude-code

# 验证
claude --version
```

> 🐛 **踩坑提示**：如果 npm 安装超时，需要先配置代理。见 [3. 配置代理](#3-配置代理-mihomo-白名单模式)。

### 1.3 如果 `claude` 命令找不到

```bash
# 确认 npm 全局安装路径
npm config get prefix

# 添加到 PATH
echo 'export PATH="$(npm config get prefix)/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

## 2. 配置 API Key 和模型

Claude Code 通过设置环境变量，可以对接 DeepSeek 兼容 API，无需 Anthropic 账号。

### 2.1 获取 DeepSeek API Key

1. 访问 [platform.deepseek.com](https://platform.deepseek.com/)
2. 注册 / 登录账号
3. 进入 **API Keys** 页面
4. 点击 **创建 API Key**，复制保存

### 2.2 配置环境变量

推荐配置在 `~/.claude/settings.json`（或项目的 `.claude/settings.json`）中：

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "sk-ba2caa600ddc4e77a8e1069a7d31ce0f",
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-pro",
    "ANTHROPIC_MODEL": "deepseek-v4-pro[1m]"
  },
  "model": "deepseek-v4-pro"
}
```

> 💡 **模型选择说明**：
> - `deepseek-v4-pro[1m]`：1M 上下文，适合大型项目
> - `deepseek-v4-pro`：标准上下文，日常使用
> - `[1m]` 后缀启用 1M token 上下文窗口

### 2.3 验证配置

```bash
claude "你好，请介绍一下你自己"

# 如果正常回复，说明 API 配置成功
# 也可以用 claude --version 确认版本
```

---

## 3. 配置代理 mihomo 白名单模式

由于中国大陆网络环境，访问 GitHub、npm registry 等需要代理。推荐使用 **mihomo（Clash Meta）** 并配置白名单模式。

### 3.1 安装 mihomo

```bash
# 下载最新版本
MCLASH_VER=$(curl -s https://api.github.com/repos/MetaCubeX/mihomo/releases/latest | grep tag_name | cut -d '"' -f 4)
wget "https://github.com/MetaCubeX/mihomo/releases/download/${MCLASH_VER}/mihomo-linux-amd64-${MCLASH_VER}.gz" -O mihomo.gz
gzip -d mihomo.gz
chmod +x mihomo
sudo mv mihomo /usr/local/bin/

# 验证
mihomo --version
```

### 3.2 配置文件

创建 `~/.config/mihomo/config.yaml`：

```yaml
# 基础配置
mixed-port: 7890
mode: rule
log-level: info

# DNS 配置（注意：用 redir-host 避免 fake-ip 劫持）
dns:
  enable: true
  listen: 0.0.0.0:5353
  enhanced-mode: redir-host
  nameserver:
    - 223.5.5.5
    - 119.29.29.29
  fallback:
    - 8.8.8.8
    - 1.1.1.1

# 你的代理服务器节点配置
proxies:
  - name: "你的节点"
    type: vmess
    server: your-server.com
    port: 443
    uuid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    alterId: 0
    cipher: auto

# 代理组
proxy-groups:
  - name: 🚀 代理
    type: select
    proxies:
      - 你的节点
      - DIRECT

# 规则：白名单模式 — 只有匹配的走代理
rules:
  # GitHub
  - DOMAIN-SUFFIX,github.com,🚀 代理
  - DOMAIN-SUFFIX,githubassets.com,🚀 代理
  - DOMAIN-SUFFIX,githubusercontent.com,🚀 代理

  # npm
  - DOMAIN-SUFFIX,npmjs.org,🚀 代理
  - DOMAIN-SUFFIX,npmjs.com,🚀 代理

  # Google
  - DOMAIN-SUFFIX,google.com,🚀 代理

  # OpenAI / Anthropic（如需直连官方 API）
  - DOMAIN-SUFFIX,openai.com,🚀 代理
  - DOMAIN-SUFFIX,anthropic.com,🚀 代理

  # 其余全部直连
  - MATCH,DIRECT
```

### 3.3 启动 mihomo

```bash
# 前台测试
mihomo -d ~/.config/mihomo

# 验证代理
curl -x http://127.0.0.1:7890 https://github.com
```

### 3.4 配置 Git 使用代理

```bash
# 仅对 GitHub 生效
git config --global http.https://github.com.proxy http://127.0.0.1:7890
```

---

## 4. 安装工具 ripgrep

Claude Code 依赖 `ripgrep`（`rg`）进行高效的代码搜索。

### 方法一：GitHub Releases 下载（推荐）

```bash
curl -fsSL -o /tmp/ripgrep.deb \
  "https://github.com/BurntSushi/ripgrep/releases/download/14.1.1/ripgrep_14.1.1-1_amd64.deb"
sudo dpkg -i /tmp/ripgrep.deb

# 验证
rg --version
```

### 方法二：通过 apt 安装

```bash
sudo apt update
sudo apt install ripgrep -y
```

> 🐛 **Ubuntu 26.04 特别说明**：apt 仓库中 `ripgrep` 包名可能变化。如果 `apt install ripgrep` 找不到，用方法一从 GitHub Releases 下载。

---

## 5. Claude Code 配置优化

### 5.1 目录结构说明

```
~/.claude/
├── settings.json          # 用户全局配置
├── settings.local.json    # 个人本地配置（权限、敏感信息）
├── .mcp.json              # MCP 服务器配置
├── plugins/               # 已安装插件
├── projects/              # 项目会话和记忆
│   └── C--Users-22815/
│       └── memory/        # 自动记忆文件
└── file-history/          # 文件编辑历史
```

项目目录：
```
your-project/
├── .claude/
│   ├── settings.json      # 项目级配置（可提交 Git）
│   └── settings.local.json # 项目本地配置（不提交 Git）
└── CLAUDE.md              # 项目指令文件
```

### 5.2 权限配置

创建/编辑 `~/.claude/settings.local.json`：

```json
{
  "permissions": {
    "allow": [
      "Bash(npm *)",
      "Bash(git *)",
      "Bash(python *)",
      "Bash(node *)",
      "Bash(curl *)",
      "Bash(wsl.exe *)",
      "WebFetch",
      "WebSearch",
      "Read(/mnt/c/Users/**)",
      "Edit(/mnt/c/Users/**)"
    ],
    "deny": [
      "Bash(rm -rf / *)",
      "Bash(sudo rm -rf / *)"
    ],
    "additionalDirectories": [
      "C:\\Users\\22815"
    ],
    "defaultMode": "default"
  },
  "enableAllProjectMcpServers": true
}
```

### 5.3 自动内存 (Auto-Memory)

```json
{
  "autoMemoryEnabled": true,
  "autoMemoryDirectory": "~/.claude/projects/C--Users-22815/memory/"
}
```

### 5.4 状态栏插件（DeepSeek 余额显示）

```json
{
  "statusLine": {
    "type": "command",
    "command": "node --experimental-strip-types ~/.claude/plugins/deepseek-balance-statusline/index.ts"
  }
}
```

### 5.5 Thinking 模式与 Effort Level

```json
{
  "effortLevel": "high",
  "alwaysThinkingEnabled": true
}
```

---

## 6. 配置 QQ 邮箱

### 6.1 获取 QQ 邮箱授权码

1. 登录 [mail.qq.com](https://mail.qq.com/)
2. **设置** → **账户**
3. 找到 **POP3/IMAP/SMTP/Exchange/CardDAV/CalDAV服务**
4. 开启 **IMAP/SMTP 服务**
5. 按提示发送短信验证，获取 **16 位授权码**

### 6.2 安装 himalaya

```bash
# 通过预编译二进制安装
wget https://github.com/pimalaya/himalaya/releases/latest/download/himalaya-linux-x86_64.tar.gz
tar xzf himalaya-linux-x86_64.tar.gz
sudo mv himalaya /usr/local/bin/
```

### 6.3 配置 himalaya

创建 `~/.config/himalaya/config.toml`：

```toml
[accounts.qq]
default = true
email = "your_qq_number@qq.com"

imap.host = "imap.qq.com"
imap.port = 993
imap.login = "your_qq_number@qq.com"
imap.passwd.cmd = "echo '你的授权码'"

smtp.host = "smtp.qq.com"
smtp.port = 465
smtp.login = "your_qq_number@qq.com"
smtp.passwd.cmd = "echo '你的授权码'"
```

### 6.4 HTTP CONNECT 隧道（绕过 ISP DPI）

如果 QQ 邮箱的 IMAP/SMTP 端口在 WSL2 中无法直连，使用邮件隧道：

```bash
# 安装隧道工具
# 从 hermes-tools 仓库获取
git clone https://github.com/2281516753/hermes-tools.git
cp hermes-tools/hermes-email-tunnel ~/.local/bin/
chmod +x ~/.local/bin/hermes-email-tunnel

# 启动隧道
hermes-email-tunnel 11993 imap.qq.com 993 &
hermes-email-tunnel 11587 smtp.qq.com 587 &
```

修改 himalaya 配置指向本地隧道：

```toml
imap.host = "127.0.0.1"
imap.port = 11993
smtp.host = "127.0.0.1"
smtp.port = 11587
```

---

## 7. 配置 MCP 服务器

MCP (Model Context Protocol) 让 Claude Code 能访问外部工具和数据源。

### 7.1 创建 MCP 配置

`~/.claude/.mcp.json`：

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
    },
    "desktop-commander": {
      "command": "npx",
      "args": [
        "-y",
        "@anthropic-ai/desktop-commander"
      ]
    }
  }
}
```

### 7.2 获取 GitHub Token

1. 访问 [github.com/settings/tokens](https://github.com/settings/tokens)
2. **Generate new token (classic)**
3. 勾选权限：`repo`、`read:org`、`workflow`
4. 生成并保存 Token

> ⚠️ 使用 **classic PAT**（`ghp_` 开头），fine-grained PAT 兼容性不如经典 PAT。

### 7.3 批准 MCP 服务器

```bash
# 列出已配置的 MCP 服务器
claude mcp list

# 批准所有项目 MCP 服务器（或在 settings.local.json 中设置 enableAllProjectMcpServers: true）
claude mcp approve github
claude mcp approve fetch
```

### 7.4 验证 MCP

启动 Claude Code 后，输入：

```
列出你可用 的 MCP 工具有哪些
```

如果回复中包含 GitHub、Fetch 等工具，说明配置成功。

---

## 8. systemd 服务化

### 8.1 邮件隧道服务

创建 `~/.config/systemd/user/email-tunnel.service`：

```ini
[Unit]
Description=Email IMAP/SMTP Tunnel
After=network-online.target

[Service]
Type=simple
ExecStart=/bin/bash -c '\
  python3 %h/.local/bin/hermes-email-tunnel 11993 imap.qq.com 993 & \
  python3 %h/.local/bin/hermes-email-tunnel 11587 smtp.qq.com 587 & \
  wait'
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

启用：

```bash
systemctl --user daemon-reload
systemctl --user enable --now email-tunnel

# WSL2 必须启用 linger
sudo loginctl enable-linger $USER

# 查看状态
systemctl --user status email-tunnel
```

### 8.2 mihomo 服务（可选）

```bash
# 创建 systemd 用户服务
cat > ~/.config/systemd/user/mihomo.service << 'EOF'
[Unit]
Description=mihomo proxy
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/mihomo -d %h/.config/mihomo
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now mihomo
```

### 8.3 确保 WSL2 关闭后服务不丢失

```bash
# 确认 linger 已启用
sudo loginctl enable-linger $USER

# 确认 wsl.conf 启用 systemd
cat /etc/wsl.conf | grep systemd
# 应为 systemd=true

# 列出所有用户服务
systemctl --user list-unit-files --state=enabled
```

---

## 9. 踩坑参考

在配置过程中可能遇到各种问题，建议参考：

- **[claude-code-pitfalls](https://github.com/2281516753/hermes-pitfalls)** — Claude Code on WSL2 踩坑指南，收录了社区反馈的常见问题和解决方案

### 常见踩坑速查

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| `claude` 命令找不到 | npm 全局路径未在 PATH | `npm config get prefix`，添加到 PATH |
| API 返回 401 | DeepSeek API Key 无效 | 重新生成 API Key |
| API 超时 | 代理未配置或 `deepseek.com` 走代理 | DeepSeek 必须直连 |
| 每次启动都要信任文件夹 | `additionalDirectories` 未配置 | 添加到 `settings.local.json` |
| 权限弹窗太多 | 未配置 allow 规则 | 添加 `permissions.allow` 规则 |
| MCP 工具未加载 | 服务器未批准 | `claude mcp list` + `claude mcp approve` |
| ripgrep 找不到 | 未安装或 PATH 问题 | GitHub Releases 手动安装 |
| `git push` 失败 | Git 不走代理 | `git config --global http.https://github.com.proxy` |
| WSL2 重启后服务丢失 | 未设 linger + systemd 自启 | `loginctl enable-linger` + `systemctl --user enable` |
| Playwright 安装失败 | Ubuntu 26.04 未适配 | 使用 Docker 或等待官方更新 |

---

## 验证清单

全部配置完成后，按以下步骤验证：

- [ ] `claude --version` 正常输出
- [ ] `claude "你好"` 能正常回复（DeepSeek API 正常）
- [ ] Claude Code 能使用 MCP 工具（如 GitHub、Fetch）
- [ ] `rg --version` 正常输出
- [ ] `himalaya envelope list` 能列出邮件
- [ ] `systemctl --user status email-tunnel` 显示 active
- [ ] `systemctl --user status mihomo` 显示 active
- [ ] WSL2 重启后所有服务自动恢复

---

## 环境信息表

| 组件 | 版本/信息 |
|------|-----------|
| **Windows** | Windows 11 |
| **WSL 版本** | WSL 2 |
| **Linux 发行版** | Ubuntu 26.04 |
| **Node.js** | 22.x LTS |
| **npm** | 10.x |
| **Claude Code** | latest |
| **mihomo** | v1.19.x |
| **ripgrep** | 14.1.1 |
| **himalaya** | latest |
| **Python** | 3.14.x |

---

## 参考链接

- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code)
- [DeepSeek API 文档](https://platform.deepseek.com/api-docs)
- [mihomo (Clash Meta)](https://github.com/MetaCubeX/mihomo)
- [himalaya 邮件客户端](https://github.com/pimalaya/himalaya)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [claude-code-pitfalls](https://github.com/2281516753/hermes-pitfalls) — 踩坑参考

---

## 相关项目

| 项目 | 说明 |
|------|------|
| [claude-code-pitfalls](https://github.com/2281516753/hermes-pitfalls) | Claude Code 踩坑指南 |
| [hermes-tools](https://github.com/2281516753/hermes-tools) | 邮件隧道工具 |
| [wsl-dev-setup](https://github.com/2281516753/wsl-dev-setup) | WSL2 开发环境一键部署 |
| [cloud-lab](https://github.com/2281516753/cloud-lab) | 云基础设施实验平台 |
| [net-auto](https://github.com/2281516753/net-auto) | 网络自动化工具集 |

---

## 作者

王炯 (Wang Jiong) — 网络工程专业，云计算方向求职中。

[GitHub](https://github.com/2281516753)

---

<p align="center">
  <b>Happy coding with Claude Code! 🚀</b>
</p>