# Windows 11 环境下 WSL 安装 OpenClaw 完整指南

本教程提供在 Windows 11 WSL 环境下安装和配置 OpenClaw 的完整流程，涵盖从环境初始化到国内 API 配置的全部步骤。

## 目录

- [一、WSL 一键安装与配置](#一wsl-一键安装与配置)
- [二、WSL 环境下安装 OpenClaw](#二wsl-环境下安装-openclaw)
- [三、国内 API 配置](#三国内-api-配置)
- [四、局域网访问配置](#四局域网访问配置)
- [五、常见问题](#五常见问题)

---

## 一、WSL 一键安装与配置

微软在 Windows 11 中极大地简化了安装流程，通过单一命令即可完成核心组件的启用。

### 1.1 一键安装 WSL

在具有管理员权限的 PowerShell 或 Windows 终端中，执行以下命令：

```powershell
wsl --install
```

此操作将自动执行以下逻辑：启用虚拟机平台（Virtual Machine Platform）和适用于 Linux 的 Windows 子系统功能；下载并安装最新的 Linux 内核；默认下载并安装 Ubuntu 发行版。如果系统中已存在旧版 WSL，建议运行 `wsl --update` 以确保获取镜像网络模式所需的内核组件。

### 1.2 验证安装状态

安装完成后，必须重启计算机以使 Hyper-V 管理程序生效。重启后，可以通过以下命令验证状态：

```powershell
wsl --status
wsl --version
```

分析结果应确认当前 WSL 版本已切换至 2.0 以上，且默认版本设置为 2。

### 1.3 Ubuntu 初始化配置

首次运行 Ubuntu 时，系统会提示创建 UNIX 用户名和密码。此账户拥有 `sudo` 权限，是后续执行安装操作的核心身份。

配置国内镜像源后，执行系统更新：

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 二、WSL 环境下安装 OpenClaw

### 2.1 基础环境配置（免密设置）

为了在后续安装脚本运行中避免频繁输入密码，建议为当前用户开启 `sudo` 免密权限：

```bash
sudo visudo
```

在文件末尾添加以下内容（将 `spoto` 替换为你的实际用户名）：

```
spoto ALL=(ALL) NOPASSWD: ALL
```

### 2.2 安装基础工具

```bash
sudo apt install -y curl
```

### 2.3 安装 Opencode 工具

Opencode 用于简化后续的复杂配置过程：

```bash
curl -fsSL https://opencode.ai/install | bash
```

验证安装：

```bash
which opencode
opencode --version
```

### 2.4 安装 OpenClaw 官方脚本

使用官方提供的一键安装脚本进行部署：

```bash
curl -fsSL https://molt.bot/install.sh | bash
```

### 2.5 运行初始化向导

安装完成后，运行初始化向导完成基本配置：

```bash
openclaw onboard --install-daemon
```

按照向导提示完成配置：
- 选择安全选项（理解风险）
- 配置工作区目录
- 设置 Gateway 认证方式
- 其他基本配置

### 2.6 验证安装

```bash
which openclaw
openclaw --version
openclaw gateway status
```

---

## 三、国内 API 配置

本章节帮助已安装 OpenClaw 的用户快速将 LLM API 替换为国内版本，支持 MiniMax 和智谱（GLM）两个主流国内 API。

### 3.1 前置条件检查

检查 OpenClaw 是否已安装：

```bash
which openclaw
ps aux | grep openclaw
openclaw --version
```

### 3.2 获取 API Key

根据你要使用的 API 服务，前往对应平台获取 API Key：

| 服务商 | 控制台地址 | API Key 获取位置 |
|--------|-----------|------------------|
| MiniMax | https://www.minimaxi.com | 控制台 → API Keys |
| 智谱 AI | https://bigmodel.cn | 控制台 → API Keys |

### 3.3 备份配置文件

```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup
```

### 3.4 MiniMax 国内版配置

MiniMax 国内版 API 配置适用于中国区域用户，提供 M2.1 等高质量模型。

**API 端点**: `https://api.minimaxi.com/anthropic`

**完整配置示例**:

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "minimax-cn": {
        "baseUrl": "https://api.minimaxi.com/anthropic",
        "apiKey": "YOUR_MINIMAX_API_KEY_HERE",
        "api": "anthropic-messages",
        "models": [
          {
            "id": "MiniMax-M2.1",
            "name": "MiniMax M2.1 (China)",
            "reasoning": true,
            "input": ["text"],
            "contextWindow": 204800,
            "maxTokens": 131072
          }
        ]
      }
    }
  }
}
```

### 3.5 智谱（GLM）配置

**API 端点**: `https://open.bigmodel.cn/api/coding/paas/v4`

**完整配置示例**:

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "zhipu-cn": {
        "baseUrl": "https://open.bigmodel.cn/api/coding/paas/v4",
        "apiKey": "YOUR_ZHIPU_API_KEY_HERE",
        "api": "anthropic-messages",
        "models": [
          {
            "id": "glm-4.7",
            "name": "GLM-4.7 (China)",
            "reasoning": true,
            "input": ["text"],
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

### 3.6 重启与验证

配置修改后，需要重启 Gateway 使配置生效：

```bash
openclaw gateway restart
```

验证模型列表：

```bash
openclaw models list
```

---

## 四、局域网访问配置

本章节帮助用户配置 OpenClaw Web UI，使其可以通过局域网访问。

### 4.1 获取本机局域网 IP

```bash
ip addr show | grep -E "inet " | awk '{print $2}' | cut -d'/' -f1 | grep -v "^127"
```

### 4.2 编辑配置文件

```bash
nano ~/.openclaw/openclaw.json
```

### 4.3 修改 Gateway 配置

将 `bind` 值从 `loopback` 改为 `lan`，并添加 `controlUi` 配置：

```json
{
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "lan",
    "auth": {
      "mode": "token",
      "token": "your_token_here"
    },
    "controlUi": {
      "allowInsecureAuth": true
    }
  }
}
```

### 4.4 重启 Gateway

```bash
openclaw gateway restart
openclaw gateway status
```

### 4.5 访问 Web UI

根据你的局域网 IP，访问地址为：`http://<你的局域网IP>:18789/`

---

## 五、网络通信优化：镜像模式配置

传统的 WSL2 采用 NAT（网络地址转换）架构。对于需要与局域网设备深度互通的场景，切换至"镜像网络模式"是最佳选择。

### 5.1 镜像模式技术优势

| 维度 | NAT 模式 (默认) | 镜像模式 (推荐) |
|------|----------------|----------------|
| IP 地址一致性 | 独立虚拟 IP (172.x.x.x) | 与宿主机共享相同局域网 IP |
| IPv6 支持 | 极其有限 | 全面原生支持 |
| localhost 行为 | 跨平台回环复杂 | 原生双向回环支持 |
| VPN 兼容性 | 经常导致 DNS 丢包 | 显著提升兼容性 |
| 局域网可见性 | 需要手动端口转发 | 局域网内其他设备可直接发现 |

### 5.2 配置.wslconfig

使用记事本打开 `C:\Users\<您的用户名>\.wslconfig`，添加以下内容：

```ini
[wsl2]
networkingMode=mirrored
dnsTunneling=true
autoProxy=true
firewall=true

[experimental]
autoMemoryReclaim=gradual
hostAddressLoopback=true
```

配置保存后，在 Windows 终端中执行 `wsl --shutdown`，等待约 8 秒钟后重新启动 Ubuntu。

### 5.3 防火墙配置

在镜像模式下，WSL2 应用将直接暴露在 Windows 防火墙规则中。执行以下 PowerShell 指令以允许必要的入站连接：

```powershell
# 允许 Hyper-V VM 接收特定的入站流量
Set-NetFirewallHyperVVMSetting -Name '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -DefaultInboundAction Allow

# 针对特定端口创建细颗粒度规则
New-NetFirewallHyperVRule -DisplayName "OpenClaw-Service" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 18789
```

---

## 常见问题

### Q1: Gateway 启动失败

**解决方案**:
```bash
cat ~/.openclaw/openclaw.json | jq
tail -f ~/.openclaw/logs/gateway.log
cp ~/.openclaw/openclaw.json.backup ~/.openclaw/openclaw.json
openclaw gateway restart
```

### Q2: 局域网无法访问

**解决方案**:
```bash
sudo ufw status
sudo ufw allow 18789/tcp
openclaw gateway status
grep '"bind"' ~/.openclaw/openclaw.json
```

### Q3: 模型未显示在列表中

**解决方案**:
1. 检查 JSON 语法是否正确
2. 确认 provider 名称与模型引用一致
3. 确认 API key 有效且有访问权限
4. 重启 Gateway：`openclaw gateway restart`

### Q4: 认证失败 (HTTP 401)

**解决方案**:
1. 验证 API key 格式是否正确
2. 确认 API key 未过期
3. 检查 API key 是否有访问所用 API 的权限

---

## 相关资源

- [OpenClaw 官方文档](https://docs.clawd.bot)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [MiniMax 官网](https://www.minimaxi.com)
- [智谱 AI 官网](https://bigmodel.cn)
- [Opencode 官网](https://opencode.ai)

---

**注意**：请妥善保管您的 API Key 和 Token，定期检查系统安全。
