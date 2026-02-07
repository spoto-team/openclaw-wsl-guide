# Windows 11 环境下 WSL 安装 OpenClaw 完整指南

本教程提供在 Windows 11 WSL 环境下安装和配置 OpenClaw 的完整流程，涵盖从环境初始化到国内 API 配置的全部步骤。

## 目录

- [一、WSL 安装与网络配置](#一wsl-安装与网络配置)
- [二、WSL 环境下安装 OpenClaw](#二wsl-环境下安装-openclaw)
- [三、国内 API 配置](#三国内-api-配置)
- [四、局域网访问配置](#四局域网访问配置)
- [五、常见问题](#五常见问题)

---

## 一、WSL 安装与网络配置

微软在 Windows 11 中极大地简化了安装流程，通过单一命令即可完成核心组件的启用。

### 1.1 WSL 版本说明

WSL（适用于 Linux 的 Windows 子系统）目前有两个主要版本：

- **WSL 1**：早期版本，采用系统调用转换层
- **WSL 2**：最新版本，运行完整的 Linux 内核，性能更佳

**强烈建议使用 WSL 2**，因为它提供了完整的内核支持和更好的性能。

### 1.2 一键安装 WSL

在具有管理员权限的 PowerShell 或 Windows 终端中，执行以下命令：

```powershell
wsl --install
```

此操作将自动执行以下逻辑：

1. **启用虚拟机平台（Virtual Machine Platform）** - WSL 2 的底层虚拟化技术
2. **启用适用于 Linux 的 Windows 子系统** - WSL 核心组件
3. **下载并安装最新的 Linux 内核** - WSL 2 需要的完整内核
4. **默认下载并安装 Ubuntu 发行版** - 主流的 Linux 发行版

安装完成后，必须**重启计算机**以使 Hyper-V 管理程序生效。

### 1.3 验证安装状态

重启后，通过以下命令验证状态：

```powershell
wsl --status
wsl --version
```

**预期输出示例**：

```
WSL 版本: 2.0.xxxx.0
内核版本: 5.15.xxxx
WSLg 版本: 1.0.xxxx
MSRDC 版本: 1.3.xxxx
Direct3D 版本: 1.611.xxxx
DXCore 版本: 10.0.xxxx
Windows 版本: 10.0.22631.xxxx
```

分析结果应确认当前 WSL 版本为 2.0 以上。

### 1.4 现有 WSL 用户更新

如果系统中已存在旧版 WSL，建议运行以下命令更新：

```powershell
wsl --update
```

此命令将下载并安装最新的 Linux 内核，确保镜像网络模式所需的内核组件已就绪。

### 1.5 Ubuntu 初始化配置

首次启动 Ubuntu 时，系统会提示创建 UNIX 用户名和密码：

```
Installing, this may take a few minutes...
Please create a default UNIX user account. The username and password must not match your Windows username.
New UNIX username: spoto
New password:
Retype password:
```

此账户拥有 `sudo` 权限，是后续执行安装操作的核心身份。

### 1.6 系统更新与软件源配置

配置国内镜像源（如清华大学 TUNA 镜像或阿里云镜像），然后执行系统更新：

```bash
# 更新软件包索引
sudo apt update

# 升级已安装的软件包
sudo apt upgrade -y

# 安装常用工具
sudo apt install -y curl wget git
```

---

## 二、WSL 网络架构详解与配置

理解 WSL 网络架构对于 OpenClaw 的局域网访问和 VPN 兼容性至关重要。

### 2.1 WSL 2 网络架构原理

WSL 2 采用轻量级虚拟机技术，运行完整的 Linux 内核。在网络层面，WSL 2 有两种主要模式：

#### NAT 模式（默认）

在默认的 NAT（网络地址转换）模式下，WSL 2 虚拟机拥有独立的虚拟网络接口：

```
┌─────────────────────────────────────────────────────────┐
│                      Windows 主机                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              WSL 2 虚拟机 (NAT)                  │   │
│  │                                                 │   │
│  │    eth0: 172.23.64.x (独立虚拟子网)              │   │
│  │    Gateway: 172.23.64.1                         │   │
│  │    DNS: 172.23.64.1                             │   │
│  └─────────────────────────────────────────────────┘   │
│              │                    │                     │
│              ▼                    ▼                     │
│    ┌─────────────────┐   ┌─────────────────┐           │
│    │ 虚拟交换机       │   │   Windows       │           │
│    │ (Hyper-V)       │   │   网络适配器     │           │
│    └─────────────────┘   └─────────────────┘           │
│              │                                        │
│              ▼                                        │
│    ┌─────────────────────────────────────────────────┐
│    │            物理网络 / Internet                    │
│    └─────────────────────────────────────────────────┘
```

**NAT 模式特点**：
- WSL 2 拥有独立的 IP 地址（172.x.x.x 范围）
- 外部网络无法直接访问 WSL 2
- 需要端口转发才能实现局域网访问
- VPN 兼容性较差，经常出现 DNS 解析问题

#### 镜像模式（推荐）

镜像模式通过直接将 Windows 宿主机的网络接口状态同步到 Linux 内核中，消除了虚拟子网带来的复杂性：

```
┌─────────────────────────────────────────────────────────┐
│                      Windows 主机                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              WSL 2 虚拟机 (镜像)                  │   │
│  │                                                 │   │
│  │    eth0: 与宿主机共享相同局域网 IP                │   │
│  │    直接暴露在 Windows 防火墙中                    │   │
│  └─────────────────────────────────────────────────┘   │
│              │                                        │
│              │  (网络接口直接同步)                      │
│              ▼                                        │
│    ┌─────────────────────────────────────────────────┐
│    │            物理网络 / Internet                    │
│    └─────────────────────────────────────────────────┘
```

**镜像模式特点**：
- 与宿主机共享相同的局域网 IP
- 原生 IPv6 支持
- VPN 兼容性显著提升
- 局域网内其他设备可直接发现
- 防火墙规则直接作用于 WSL 应用

### 2.2 镜像模式技术优势对比

| 维度 | NAT 模式 (默认) | 镜像模式 (推荐) |
|------|----------------|----------------|
| IP 地址一致性 | 独立虚拟 IP (172.x.x.x) | 与宿主机共享相同局域网 IP |
| IPv6 支持 | 极其有限 | 全面原生支持 |
| localhost 行为 | 跨平台回环复杂 | 原生双向回环支持 |
| VPN 兼容性 | 经常导致 DNS 丢包 | 显著提升兼容性 |
| 局域网可见性 | 需要手动端口转发 | 局域网内其他设备可直接发现 |
| 防火墙控制 | 独立规则 | 直接使用 Windows 防火墙 |
| 端口转发 | 需要手动配置 | 无需配置 |

### 2.3 配置.wslconfig 文件

镜像模式属于全局配置，需在 Windows 用户目录下创建或修改 `.wslconfig` 文件。

**步骤 1**：打开文件资源管理器，导航到以下路径：

```
C:\Users\<您的用户名>\
```

**步骤 2**：如果文件不存在，创建名为 `.wslconfig` 的新文件（注意前面的点号）。

**步骤 3**：使用记事本打开文件，添加以下配置：

```ini
[wsl2]
# 启用镜像网络模式 - 这是最重要的配置
networkingMode=mirrored
# 启用 DNS 隧道，防止 VPN 环境下域名解析失效
dnsTunneling=true
# 强制 WSL 使用 Windows 的 HTTP 代理设置
autoProxy=true
# 启用集成防火墙支持
firewall=true

[experimental]
# 自动回收闲置内存，优化性能
autoMemoryReclaim=gradual
# 支持主机回环地址访问
hostAddressLoopback=true
```

**步骤 4**：保存文件。

**步骤 5**：在 Windows 终端中执行以下命令以应用配置：

```powershell
wsl --shutdown
```

**步骤 6**：等待约 8 秒钟以确保虚拟机彻底关闭，然后重新启动 Ubuntu。

```bash
wsl -d Ubuntu-24.04
```

**验证配置**：

进入 WSL 后，执行以下命令验证网络模式：

```bash
# 查看网络接口
ip addr show

# 查看路由表
ip route show

# 测试与局域网的连通性
ping 192.168.1.1
```

如果配置成功，你应该能够看到 WSL 使用与 Windows 相同的局域网 IP 地址段。

### 2.4 防火墙与安全策略调整

在镜像模式下，WSL2 应用将直接暴露在 Windows 防火墙规则中。为了确保 OpenClaw 的网络端口能够被正确访问，需要配置 Windows 防火墙规则。

**步骤 1**：以管理员身份打开 PowerShell

在开始菜单中搜索 "PowerShell"，右键选择"以管理员身份运行"。

**步骤 2**：配置防火墙规则

在镜像模式下，WSL2 应用将直接暴露在 Windows 防火墙规则中。以下提供两种配置方法：

#### 方法一：标准防火墙命令（推荐用于 WSL2）

对于 WSL2 镜像网络模式，推荐使用标准的 Windows 防火墙命令：

```powershell
# 创建入站防火墙规则，允许 OpenClaw 服务端口
New-NetFirewallRule -DisplayName "OpenClaw-Service" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 18789

# 验证规则是否创建成功
Get-NetFirewallRule -DisplayName "OpenClaw-Service" | Format-Table
```

#### 方法二：Hyper-V 防火墙命令（需要动态获取 VMCreatorId）

如果需要使用 Hyper-V 防火墙管理 WSL2 流量，请先动态获取系统的 VMCreatorId：

```powershell
# 步骤 1：获取当前系统所有已注册的 VM 创建者
Get-NetFirewallHyperVVMCreator
```

**重要说明**：`{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}` 是示例 GUID，实际值因系统而异。VMCreatorId 是 Hyper-V 防火墙用于标识特定虚拟机创建者的 GUID，**每个系统的值都不同**。

```powershell
# 步骤 2：动态获取并配置（推荐）
# 获取所有 VMCreatorId 并配置
Get-NetFirewallHyperVVMCreator | ForEach-Object {
    Set-NetFirewallHyperVVMSetting -Name $_.Name -DefaultInboundAction Allow
}

# 步骤 3：创建细粒度规则
New-NetFirewallHyperVRule -DisplayName "OpenClaw-Service" -Direction Inbound -Action Allow -Protocol TCP -LocalPorts 18789
```

**步骤 3**：确认防火墙规则

验证规则已正确创建：

```powershell
# 查看所有 OpenClaw 相关规则
Get-NetFirewallRule | Where-Object {$_.DisplayName -like "*OpenClaw*"} | Format-Table
```

**步骤 4**：测试端口连通性

在 Windows 主机上测试端口是否开放：

```powershell
# 测试本地端口
Test-NetConnection -ComputerName localhost -Port 18789

# 或使用 telnet
telnet localhost 18789
```

### 2.5 NAT 模式下的端口转发（备选方案）

如果由于某些原因无法使用镜像模式，可以在 NAT 模式下配置端口转发：

**步骤 1**：在 PowerShell（管理员）中执行：

```powershell
# 将 WSL 2 的 18789 端口转发到 localhost
netsh interface portproxy add v4tov4 listenport=18789 listenaddress=0.0.0.0 connectport=18789 connectaddress=(wsl hostname -I)
```

**步骤 2**：获取 WSL 的 IP 地址：

```powershell
wsl hostname -I
```

**步骤 3**：或者使用以下脚本自动获取并设置：

```powershell
# 获取 WSL IP 并设置端口转发
$wslIp = wsl hostname -I
netsh interface portproxy delete v4tov4 listenport=18789 listenaddress=0.0.0.0
netsh interface portproxy add v4tov4 listenport=18789 listenaddress=0.0.0.0 connectport=18789 connectaddress=$wslIp
```

**注意**：每次重启 WSL 后，IP 地址可能会变化，需要重新配置端口转发。

---

## 三、WSL 环境下安装 OpenClaw

### 3.1 基础环境配置（免密设置）

为了在后续安装脚本运行中避免频繁输入密码，建议为当前用户开启 `sudo` 免密权限：

```bash
sudo visudo
```

在文件末尾添加以下内容（将 `spoto` 替换为你的实际用户名）：

```
spoto ALL=(ALL) NOPASSWD: ALL
```

保存并退出（Ctrl+O → Enter → Ctrl+X）。

### 3.2 安装基础工具

```bash
sudo apt install -y curl wget git
```

### 3.3 安装 Opencode 工具

Opencode 用于简化后续的复杂配置过程：

```bash
curl -fsSL https://opencode.ai/install | bash
```

验证安装：

```bash
which opencode
opencode --version
```

### 3.4 安装 OpenClaw 官方脚本

使用官方提供的一键安装脚本进行部署：

```bash
curl -fsSL https://molt.bot/install.sh | bash
```

### 3.5 运行初始化向导

安装完成后，运行初始化向导完成基本配置：

```bash
openclaw onboard --install-daemon
```

按照向导提示完成配置：
- 选择安全选项（理解风险）
- 配置工作区目录
- 设置 Gateway 认证方式
- 其他基本配置

### 3.6 验证安装

```bash
which openclaw
openclaw --version
openclaw gateway status
```

---

## 四、国内 API 配置

本章节帮助已安装 OpenClaw 的用户快速将 LLM API 替换为国内版本，支持 MiniMax 和智谱（GLM）两个主流国内 API。

### 4.1 前置条件检查

检查 OpenClaw 是否已安装：

```bash
which openclaw
ps aux | grep openclaw
openclaw --version
```

### 4.2 获取 API Key

根据你要使用的 API 服务，前往对应平台获取 API Key：

| 服务商 | 控制台地址 | API Key 获取位置 |
|--------|-----------|------------------|
| MiniMax | https://www.minimaxi.com | 控制台 → API Keys |
| 智谱 AI | https://bigmodel.cn | 控制台 → API Keys |

### 4.3 备份配置文件

```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup
```

### 4.4 MiniMax 国内版配置

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

### 4.5 智谱（GLM）配置

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

### 4.6 重启与验证

配置修改后，需要重启 Gateway 使配置生效：

```bash
openclaw gateway restart
```

验证模型列表：

```bash
openclaw models list
```

---

## 五、局域网访问配置

本章节帮助用户配置 OpenClaw Web UI，使其可以通过局域网访问。

### 5.1 获取本机局域网 IP

```bash
ip addr show | grep -E "inet " | awk '{print $2}' | cut -d'/' -f1 | grep -v "^127"
```

### 5.2 编辑配置文件

```bash
nano ~/.openclaw/openclaw.json
```

### 5.3 修改 Gateway 配置

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

### 5.4 重启 Gateway

```bash
openclaw gateway restart
openclaw gateway status
```

### 5.5 访问 Web UI

根据你的局域网 IP，访问地址为：`http://<你的局域网IP>:18789/`

---

## 六、常见问题

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

### Q5: 镜像模式不生效

**解决方案**:
1. 确认已执行 `wsl --shutdown` 并等待 8 秒后重新启动
2. 检查 `.wslconfig` 文件路径是否正确
3. 确认 WSL 版本为 2.0 以上
4. 更新 WSL 内核：`wsl --update`

### Q6: 防火墙规则创建失败

**问题表现**：运行 `New-NetFirewallHyperVRule` 或 `Set-NetFirewallHyperVVMSetting` 时报错。

**解决方案**:
1. 确认以管理员身份运行 PowerShell
2. 检查 Hyper-V 服务是否正在运行
3. **VMCreatorId 错误**：如果使用 Hyper-V 防火墙命令报错，可能是因为硬编码的 GUID 不存在于当前系统。请改用标准防火墙命令：
   ```powershell
   # 推荐：使用标准防火墙命令（适用于 WSL2）
   New-NetFirewallRule -DisplayName "OpenClaw-Service" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 18789
   ```
4. **动态获取 VMCreatorId**（仅当需要使用 Hyper-V 防火墙时）：
   ```powershell
   # 获取系统所有 VM 创建者
   Get-NetFirewallHyperVVMCreator

   # 使用动态获取的 GUID 配置
   Get-NetFirewallHyperVVMCreator | ForEach-Object {
       Set-NetFirewallHyperVVMSetting -Name $_.Name -DefaultInboundAction Allow
   }
   ```

---

## 相关资源

- [OpenClaw 官方文档](https://docs.clawd.bot)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [MiniMax 官网](https://www.minimaxi.com)
- [智谱 AI 官网](https://bigmodel.cn)
- [Opencode 官网](https://opencode.ai)

---

**注意**：请妥善保管您的 API Key 和 Token，定期检查系统安全。
