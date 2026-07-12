# Claude Code on WSL2 踩坑指南

[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude-Code-D97757)](https://claude.ai/code)
[![Platform: WSL2](https://img.shields.io/badge/Platform-WSL2-blue)](https://learn.microsoft.com/en-us/windows/wsl/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-26.04-E95420?logo=ubuntu)](https://ubuntu.com/)
[![DeepSeek](https://img.shields.io/badge/Model-DeepSeek_V4-brightgreen)](https://deepseek.com/)

WSL2 (Ubuntu 26.04) 环境下安装、配置和使用 Claude Code 时遇到的各类问题及解决方案。

> 原为 Hermes Agent 踩坑记录，现已全面转向 Claude Code。Hermes 相关内容见 [历史归档](#历史归档)。

> 持续更新中，每次踩坑都会追加。

---

## 目录

- [1. 安装 Claude Code](#1-安装-claude-code)
- [2. API 与模型配置](#2-api-与模型配置)
- [3. Claude Code 配置体系](#3-claude-code-配置体系)
- [4. 网络与代理](#4-网络与代理)
- [5. MCP 服务器](#5-mcp-服务器)
- [6. 邮件集成](#6-邮件集成)
- [7. 工具安装](#7-工具安装)
- [8. 系统集成](#8-系统集成)
- [9. 常见错误速查](#9-常见错误速查)
- [环境信息](#环境信息)
- [历史归档：Hermes Agent 踩坑](#历史归档)

---

## 1. 安装 Claude Code

### 1.1 npm 全局安装被墙

**现象**：`npm install -g @anthropic-ai/claude-code` 超时或 SSL 错误。

**原因**：npm registry 在中国大陆直连速度慢，`@anthropic-ai` 包需要走代理。

**解决**：

```bash
# 方法一：设置 npm 代理后安装
npm config set proxy http://127.0.0.1:7890
npm config set https-proxy http://127.0.0.1:7890
npm install -g @anthropic-ai/claude-code

# 方法二：使用 npx（临时代理）
HTTP_PROXY=http://127.0.0.1:7890 npx @anthropic-ai/claude-code --version
```

### 1.2 Node.js 版本要求

Claude Code 要求 Node.js >= 18。确认版本：

```bash
node --version   # 应 >= v18.0.0
```

如果版本过低，用 nvm 升级：

```bash
nvm install --lts
nvm use --lts
```

### 1.3 安装后 claude 命令找不到

**现象**：`claude` 命令报 `command not found`。

**原因**：npm 全局 bin 目录不在 PATH 中。

**解决**：

```bash
# 确认 npm 全局路径
npm config get prefix

# 确认 claude 安装位置
ls $(npm config get prefix)/bin/claude

# 添加到 PATH（添加到 ~/.bashrc）
echo 'export PATH="$(npm config get prefix)/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 1.4 国内网络无法直接使用 Anthropic API

**现象**：`claude` 启动后无法连接，报网络错误。

**原因**：Anthropic API (`api.anthropic.com`) 在中国大陆被墙。

**解决**：使用 DeepSeek API 兼容接口（见下一章节），或配置 HTTP 代理：

```bash
export HTTP_PROXY=http://127.0.0.1:7890
export HTTPS_PROXY=http://127.0.0.1:7890
```

---

## 2. API 与模型配置

### 2.1 DeepSeek API 替代 Anthropic

Claude Code 通过设置环境变量，可以对接 DeepSeek 兼容 API：

```bash
# ~/.claude/settings.json 或环境变量
export ANTHROPIC_AUTH_TOKEN="sk-ba2caa600ddc4e77a8e1069a7d31ce0f"
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
export ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-v4-pro[1m]"
export ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-v4-pro[1m]"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek-v4-pro"
export ANTHROPIC_MODEL="deepseek-v4-pro[1m]"
```

> ⚠️ 注意：`ANTHROPIC_AUTH_TOKEN` 是 DeepSeek API Key（`sk-` 开头），不是 Anthropic 的 token。

### 2.2 DeepSeek API Key 获取

1. 访问 [platform.deepseek.com](https://platform.deepseek.com/)
2. 注册 / 登录
3. API Keys → 创建 API Key
4. 复制 key，填入配置文件

### 2.3 余额查询

DeepSeek 不提供标准 Anthropic 余额 API。需要通过 DeepSeek 网页端查看余额。

可以使用社区 statusline 插件在 Claude Code 状态栏实时显示余额（详见 [2.5](#25-状态栏插件)）。

### 2.4 DeepSeek API 走代理导致不可用

**现象**：配置正确但返回错误或超时。

**原因**：DeepSeek 是国内服务，经过代理反而不可用。

**解决**：在 mihomo 白名单模式下默认直连即可。如果在黑名单模式，确保：

```yaml
# mihomo config
- DOMAIN-SUFFIX,deepseek.com,DIRECT
- DOMAIN-SUFFIX,deepseek.io,DIRECT
```

### 2.5 状态栏插件

社区插件 `deepseek-balance-statusline` 在 Claude Code 状态栏显示 DeepSeek 余额：

```bash
# 安装（已内置在 ~/.claude/plugins/ 下）
# 配置 ~/.claude/settings.json
{
  "statusLine": {
    "type": "command",
    "command": "node --experimental-strip-types ~/.claude/plugins/deepseek-balance-statusline/index.ts"
  }
}
```

---

## 3. Claude Code 配置体系

### 3.1 settings.json vs settings.local.json

Claude Code 有两套配置文件：

| 文件 | 用途 | 是否提交 Git |
|------|------|:--:|
| `.claude/settings.json` | 项目级共享配置 | ✅ |
| `.claude/settings.local.json` | 个人本地配置（权限、Token） | ❌ |
| `~/.claude/settings.json` | 用户全局配置 | - |

**关键区别**：
- `settings.json` 可以提交到 Git 供团队共享
- `settings.local.json` 存放敏感信息（API Key、权限规则），不入版本控制
- 用户目录下的 `~/.claude/settings.json` 对所有项目生效

### 3.2 权限系统配置

Claude Code 默认每次操作需要手动批准。优化权限配置：

```json
// ~/.claude/settings.local.json
{
  "permissions": {
    "allow": [
      "Bash(npm *)",
      "Bash(git *)",
      "Bash(python *)",
      "Bash(node *)",
      "Bash(curl *)",
      "WebFetch",
      "WebSearch",
      "Read(/mnt/c/Users/**)",
      "Edit(/mnt/c/Users/**)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(sudo rm *)"
    ],
    "additionalDirectories": [
      "C:\\Users\\22815"
    ],
    "defaultMode": "default"
  }
}
```

### 3.3 每次启动都要信任文件夹

**现象**：每次 `claude` 启动都弹出"是否信任此文件夹"提示。

**原因**：Claude Code 默认对每个新工作目录需要信任确认，但如果每次都出现，说明信任状态没有持久化，或工作目录不在 `additionalDirectories` 范围内。

**解决**：在 `settings.local.json` 中添加：

```json
{
  "permissions": {
    "additionalDirectories": [
      "C:\\Users\\22815"
    ]
  }
}
```

### 3.4 自动内存 (Auto-Memory)

Claude Code 支持跨会话记忆（MEMORY.md），配置：

```json
{
  "autoMemoryEnabled": true,
  "autoMemoryDirectory": "~/.claude/projects/C--Users-22815/memory/"
}
```

**注意**：
- MEMORY.md 上限约 2200 字符，超出影响 token 消耗
- 建议定期清理过时或不准确的记忆
- 记忆按项目隔离，不同目录有独立的 MEMORY.md

### 3.5 会话权限模式选择

| 模式 | 说明 | 适合场景 |
|------|------|----------|
| `default` | 每次操作确认 | 日常开发 |
| `acceptEdits` | 自动接受文件编辑 | 信任的代码修改 |
| `auto` | 大部分操作自动 | 信任的批量任务 |
| `plan` | 先规划再执行 | 复杂改动 |
| `bypassPermissions` | 跳过所有权限检查 | 完全隔离的环境 |

### 3.6 Thinking 模式与 Effort Level

Claude Code 支持控制模型思考深度：

```bash
# 命令行启动时设置
claude --effort high

# 或在 settings.json 中
{
  "effortLevel": "high",
  "alwaysThinkingEnabled": true
}
```

DeepSeek V4 兼容 thinking 模式，但注意：
- 开启 thinking 后 token 消耗增加约 20-30%
- 复杂任务收益明显，简单问答开启反而浪费

---

## 4. 网络与代理

### 4.1 mihomo fake-ip DNS 劫持导致 QQ 邮件不可用

**现象**：`imap.qq.com` / `smtp.qq.com` 被解析到 `198.18.0.x`（假 IP），邮件客户端无法连接。

**原因**：mihomo 默认 `enhanced-mode: fake-ip`，将所有 DNS 查询拦截并返回假 IP。WSL2 的 DNS 代理（`10.255.255.254`）将这些假 IP 转发给 Linux 子系统。

**解决**：

```yaml
# ~/.config/mihomo/config-minimal.yaml
dns:
  enable: true
  enhanced-mode: redir-host   # 关闭 fake-ip
```

同时将 QQ 邮件域名加入 `/etc/hosts`，指向真实 IP。

### 4.2 ISP 深度包检测拦截 QQ 邮件 TLS 握手

**现象**：TCP 连接到 QQ 邮件服务器成功（`157.148.54.34:587`），收到 SMTP banner，但 `STARTTLS` 或直接 SSL（465）均超时。

**原因**：ISP 对非 443 端口的 TLS 流量进行 DPI 拦截。

**解决**：创建 HTTP CONNECT 隧道，通过 mihomo 代理转发 TLS 流量：

```bash
# 本地监听 → HTTP CONNECT 代理 → 目标服务器
python3 hermes-email-tunnel 11993 imap.qq.com 993 &
python3 hermes-email-tunnel 11587 smtp.qq.com 587 &
```

然后将 `/etc/hosts` 中 `imap.qq.com` / `smtp.qq.com` 指向 `127.0.0.1`。

### 4.3 mihomo 黑名单模式 → 白名单模式

**现象**：默认 `MATCH,🔰 选择节点` 让所有未知流量走代理，国内网站也经代理绕一圈。

**解决**：改为白名单模式：

```yaml
rules:
  # 只列被墙域名走代理
  - DOMAIN-SUFFIX,google.com,🔰 选择节点
  - DOMAIN-SUFFIX,github.com,🔰 选择节点
  - DOMAIN-SUFFIX,githubassets.com,🔰 选择节点
  - DOMAIN-SUFFIX,githubusercontent.com,🔰 选择节点
  - DOMAIN-SUFFIX,npmjs.org,🔰 选择节点
  - DOMAIN-SUFFIX,npmjs.com,🔰 选择节点
  - DOMAIN-SUFFIX,anthropic.com,🔰 选择节点
  - DOMAIN-SUFFIX,openai.com,🔰 选择节点
  # 其余全部直连
  - MATCH,DIRECT
```

### 4.4 WSL2 DNS 被 Windows 宿主机代理劫持

**现象**：`getent hosts` 返回假 IP，`dig` / `nslookup` 也无法正常解析。

**原因**：WSL2 的 DNS 代理（`10.255.255.254`）将请求转发到 Windows 宿主，Windows 上的代理软件劫持了 DNS。

**解决**：避免修改 `/etc/resolv.conf`（会导致 WSL 网络彻底断裂）。改用 `/etc/hosts` 做关键域名的静态映射：

```bash
sudo bash -c 'echo "157.148.54.34 imap.qq.com smtp.qq.com" >> /etc/hosts'
```

### 4.5 `git push` 到 GitHub 需要走代理

**现象**：`git push` 报 `TLS connection non-properly terminated` 或 `Connection timed out`，但 `gh` 命令正常。

**原因**：GitHub 在中国大陆被墙，`git` 不走代理而 `gh` 内置代理支持。

**解决**：

```bash
# 为 git 设置代理（仅对 GitHub 生效）
git config --global http.https://github.com.proxy http://127.0.0.1:7890
```

### 4.6 WSL2 网络延迟导致超时

**现象**：Claude Code 执行工具调用时频繁超时。

**原因**：WSL2 的网络栈经过虚拟化层 + Windows 宿主转发，天然比原生 Linux 多一层延迟。叠加代理后延迟进一步放大。

**解决**：

```bash
# 调大 shell 命令超时（settings.json）
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "export TIMEOUT_LONG=300",
        "timeout": 300
      }]
    }]
  }
}
```

---

## 5. MCP 服务器

### 5.1 MCP 配置方式

Claude Code 使用 `.mcp.json` 文件配置 MCP 服务器（项目级或用户级）：

```json
// ~/.claude/.mcp.json
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
      "args": ["-y", "@anthropic-ai/desktop-commander"]
    }
  }
}
```

### 5.2 GitHub MCP 配置

#### 获取 GitHub Token

1. 访问 [github.com/settings/tokens](https://github.com/settings/tokens)
2. **Generate new token (classic)**
3. 勾选权限：`repo`、`read:org`、`workflow`
4. 生成并保存 Token

> ⚠️ 使用 **classic PAT**（`ghp_` 开头），fine-grained PAT 兼容性不如经典 PAT。

#### 配置

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_***"
      }
    }
  }
}
```

### 5.3 MCP 工具不生效

**现象**：`.mcp.json` 配置正确但 MCP 工具在 Claude Code 中不出现。

**解决**：
1. 确认 `enableAllProjectMcpServers: true` 或服务器在 `enabledMcpjsonServers` 列表中
2. 重启 Claude Code 会话（MCP 在启动时加载）
3. 检查 `claude mcp list` 输出确认各服务器状态

```bash
# 列出已加载的 MCP 服务器
claude mcp list

# 添加并批准 MCP 服务器
claude mcp add github
claude mcp approve github
```

### 5.4 Playwright Chromium 不支持 Ubuntu 26.04

**现象**：`npx playwright install chromium` 报 `Playwright does not support chromium on ubuntu26.04-x64`。

**原因**：Ubuntu 26.04 太新，Playwright 未适配。

**解决**：
- 等待 Playwright 官方支持
- 或使用 Docker 运行 Chromium
- Web fetch/search 功能通过 MCP fetch 服务器仍可用

### 5.5 Desktop Commander 文件访问权限

**现象**：Desktop Commander MCP 无法访问某些目录。

**原因**：MCP 服务器的 allowedDirectories 限制了文件系统访问范围。

**解决**：在 Desktop Commander 配置中指定允许的目录：

```json
{
  "mcpServers": {
    "desktop-commander": {
      "command": "npx",
      "args": [
        "-y",
        "@anthropic-ai/desktop-commander",
        "--allowedDirectories",
        "/home/user",
        "/mnt/c/Users"
      ]
    }
  }
}
```

---

## 6. 邮件集成

### 6.1 QQ 邮箱 SMTP 授权码获取

1. 登录 https://mail.qq.com
2. 设置 → 账户 → POP3/IMAP/SMTP 服务
3. 开启 IMAP/SMTP，按提示发送短信验证
4. 获得 16 位授权码

### 6.2 himalaya 邮件客户端配置

```toml
# ~/.config/himalaya/config.toml
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

### 6.3 himalaya 保存到"已发送"失败

**现象**：`himalaya template send` 返回 `cannot find UID of appended IMAP message`，但邮件实际已发出。

**原因**：QQ 邮箱的 Sent 文件夹 IMAP 名称与 himalaya 默认值不匹配。

**解决**：邮件会正常投递，该错误仅影响"已发送"副本。可接受为已知缺陷。

### 6.4 himalaya TLS 证书验证失败

**现象**：通过 `127.0.0.1` 连接隧道时，证书验证报 `certificate not valid for name "127.0.0.1"`。

**解决**：在 `/etc/hosts` 中将 `imap.qq.com` / `smtp.qq.com` 指向 `127.0.0.1`，himalaya 配置中使用域名而非 IP。

---

## 7. 工具安装

### 7.1 ripgrep 不在 Ubuntu 26.04 apt 源中

**现象**：`sudo apt install ripgrep` → `Unable to locate package ripgrep`。

**解决**：直接从 GitHub Releases 下载 `.deb` 安装：

```bash
curl -fsSL -o /tmp/ripgrep.deb \
  "https://github.com/BurntSushi/ripgrep/releases/download/14.1.1/ripgrep_14.1.1-1_amd64.deb"
sudo dpkg -i /tmp/ripgrep.deb
```

### 7.2 pip 未安装在系统 Python 中

**现象**：`pip: command not found`，`python3 -m pip: No module named pip`。

**解决**：

```bash
# 安装 pip
sudo apt install python3-pip -y

# 或使用 venv
python3 -m venv venv
source venv/bin/activate
pip install <package>
```

### 7.3 uv / astral.sh 被墙

**现象**：`curl ... astral.sh/uv/install.sh` → SSL 错误。

**原因**：astral.sh 被 GFW 封锁。

**解决**：使用 `npx` 替代（Claude Code 自带 Node.js 运行 MCP 服务器，无需 uv），或通过代理下载。

---

## 8. 系统集成

### 8.1 邮件隧道 systemd 服务化

隧道进程需要持久运行，避免手动启停：

```ini
# ~/.config/systemd/user/email-tunnel.service
[Unit]
Description=Email IMAP/SMTP Tunnel
After=network-online.target mihomo.service

[Service]
Type=simple
ExecStart=/bin/bash -c '\
  python3 ~/.local/bin/hermes-email-tunnel 11993 imap.qq.com 993 & \
  python3 ~/.local/bin/hermes-email-tunnel 11587 smtp.qq.com 587 & \
  wait'
Restart=always

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now email-tunnel
# WSL2 需要 linger
sudo loginctl enable-linger $USER
```

### 8.2 WSL2 关闭后服务停止

**现象**：关闭 WSL2 终端窗口后，所有后台进程暂停。

**原因**：WSL2 默认在最后一个终端关闭后自动休眠。

**解决**：

```bash
# 1. 启用 linger
sudo loginctl enable-linger $USER

# 2. wsl.conf 确保 systemd 可用
sudo bash -c 'echo -e "[boot]\nsystemd=true" > /etc/wsl.conf'

# 3. 服务设为 systemd 管理
systemctl --user enable --now email-tunnel
```

### 8.3 Claude Code 插件管理

```bash
# 列出已安装插件
claude plugins list

# 搜索插件
claude plugins search <keyword>

# 安装插件
claude plugins install <plugin-name>
```

---

## 9. 常见错误速查

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| `claude` 命令找不到 | npm 全局 bin 不在 PATH 中 | `npm config get prefix`，确认 PATH |
| API 返回 401 | Token 无效或过期 | 重新生成 DeepSeek API Key |
| API 超时 | 代理未配置或配置错误 | 检查 `ANTHROPIC_BASE_URL`，确认代理状态 |
| MCP 工具未加载 | 未批准 MCP 服务器 | `claude mcp list` 检查，`claude mcp approve` 批准 |
| 每次启动都要信任文件夹 | `additionalDirectories` 未配置 | 添加工作目录到 `permissions.additionalDirectories` |
| 权限弹窗太多 | 未配置 allow 规则 | 在 `settings.local.json` 添加 `permissions.allow` |
| ripgrep 未找到 | apt 包名变化或未安装 | GitHub Releases 手动安装 |
| `git push` 失败 | Git 不走代理 | `git config --global http.https://github.com.proxy` |
| WSL2 重启后服务丢失 | 未设 systemd 自启 | `systemctl --user enable` + `loginctl enable-linger` |
| Playwright 安装失败 | Ubuntu 26.04 未适配 | 使用 Docker 或等待官方更新 |

---

## 环境信息

| 项目 | 详情 |
|------|------|
| OS | WSL2 Ubuntu 26.04 x86_64 |
| Proxy | mihomo v1.19.25 (白名单模式) |
| Claude Code | latest (npm global) |
| 模型 | DeepSeek V4 Pro (via ANTHROPIC_BASE_URL) |
| Node | v22.22.3 |
| Python | 3.14.4 |
| MCP | github, fetch, playwright, desktop-commander |

---

## 参考链接

- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code)
- [Claude Code GitHub](https://github.com/anthropics/claude-code)
- [DeepSeek API 文档](https://platform.deepseek.com/api-docs)
- [mihomo (Clash Meta)](https://github.com/MetaCubeX/mihomo)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [hermes-tools](https://github.com/2281516753/hermes-tools) — 邮件隧道工具
- [wsl-dev-setup](https://github.com/2281516753/wsl-dev-setup) — WSL2 开发环境一键部署
- [cloud-lab](https://github.com/2281516753/cloud-lab) — 云基础设施实验平台

---

## 历史归档：Hermes Agent 踩坑

> 以下为原 Hermes Agent 使用期间记录的踩坑内容。部分通用内容（网络/代理/邮件）已合并到上文。

### Hermes 配置版本迁移

**现象**：`hermes doctor` 提示 `Config version outdated (v23 → v24)`。

**解决**：
```bash
hermes doctor --fix
# 或
hermes config migrate
```

### Hermes config.yaml 模型路由

```yaml
smart_model_routing:
  enabled: true
  cheap_model:
    model: deepseek-v4-flash
    provider: deepseek
  max_simple_chars: 160
```

### Hermes Desktop 桌面板

- 安装位置：`C:\Users\<用户名>\AppData\Local\Hermes Agent CN Desktop\`
- 用户数据：`C:\Users\<用户名>\AppData\Roaming\cn.org.hermesagent.desktop\runtime\hermes-home\`

### Hermes Bot / Gateway

配置 Telegram / QQ Bot 时，需要：
1. 给 gateway systemd 注入代理环境变量
2. WSL2 启用 linger 保证后台运行
3. QQ Bot 仅支持 WebSocket 模式（非普通 QQ 登录）

详细内容见 Git 历史中的原始 README。

---

## 作者

王炯 (Wang Jiong) — 网络工程专业，云计算方向求职中。

[GitHub](https://github.com/2281516753)

---

MIT License