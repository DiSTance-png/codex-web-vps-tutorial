# 在 VPS 上部署 Codex Web，并通过 WireGuard 移动访问

本文介绍如何在一台 VPS 上部署基于 `codex app-server` 的 Codex Web，再通过另一台 VPS 的 WireGuard 网络入口和 SSH 转发，让手机或 iPad 通过浏览器远程使用 Codex。

最终结构如下：

```text
手机 / iPad
    │ WireGuard
    ▼
入口 VPS：10.20.0.1:3001
    │ SSH / autossh 转发
    ▼
应用 VPS：127.0.0.1:3001
    │
    ▼
Codex Web → codex app-server → Codex CLI
```

本文不会把 Codex Web 直接暴露到公网，手机只通过 WireGuard 私网访问。

## 一、准备条件

应用 VPS 建议满足：

- Ubuntu 22.04 或更新版本
- 2 核 CPU
- 2 GB 以上内存
- 20 GB 以上可用磁盘
- Node.js 20.9 或更高版本
- 能够安装并运行 Codex CLI

入口 VPS 需要：

- 已安装 WireGuard
- 可以通过 SSH 连接应用 VPS
- 能够使用 SSH 密钥自动重连

本文使用以下占位符，请替换为自己的值：

```text
APP_VPS              应用 VPS 的地址或主机名
APP_USER             应用 VPS 上运行 Codex 的用户
ENTRY_VPS_WG_IP      入口 VPS 的 WireGuard 地址
ENTRY_VPS_PUBLIC_IP  入口 VPS 的公网地址
```

## 二、配置 WireGuard

如果已经有可用的 WireGuard 网络，可以跳过本节，只需要确认手机能够访问入口 VPS 的 WireGuard 地址。

### 1. 安装 WireGuard

在入口 VPS 上执行：

```bash
sudo apt update
sudo apt install -y wireguard
```

生成服务器密钥：

```bash
sudo sh -c 'umask 077; wg genkey > /etc/wireguard/server_private.key'
sudo sh -c 'wg pubkey < /etc/wireguard/server_private.key > /etc/wireguard/server_public.key'
```

服务器私钥只能保存在服务器上，不要发布或提交到 Git。

### 2. 创建服务器配置

```bash
sudo nano /etc/wireguard/wg0.conf
```

示例：

```ini
[Interface]
Address = 10.20.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

[Peer]
PublicKey = PHONE_PUBLIC_KEY
AllowedIPs = 10.20.0.2/32
```

其中：

- `SERVER_PRIVATE_KEY` 是入口 VPS 的私钥
- `PHONE_PUBLIC_KEY` 是手机端的公钥

保护配置文件：

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
```

### 3. 创建手机端配置

在安全设备上生成手机端密钥：

```bash
umask 077
wg genkey | tee phone_private.key | wg pubkey > phone_public.key
```

在手机 WireGuard App 中创建隧道：

```ini
[Interface]
PrivateKey = PHONE_PRIVATE_KEY
Address = 10.20.0.2/24

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = ENTRY_VPS_PUBLIC_IP:51820
AllowedIPs = 10.20.0.0/24
PersistentKeepalive = 25
```

`AllowedIPs = 10.20.0.0/24` 只把 WireGuard 内网流量交给 VPN，不会接管手机的全部网络流量。

### 4. 启动并检查 WireGuard

```bash
sudo systemctl enable --now wg-quick@wg0
sudo wg show
```

云厂商安全组需要放行：

```text
UDP 51820
```

如果入口 VPS 使用 UFW：

```bash
sudo ufw allow 51820/udp
```

手机连接后，可以测试：

```text
ping 10.20.0.1
```

本文只通过 SSH 转发访问应用 VPS，因此不需要为 Codex Web 设置公网端口，也不需要开启 VPS 的 IP 转发和 NAT。

## 三、安装 Codex CLI

在应用 VPS 上执行：

```bash
npm install -g @openai/codex
```

确认当前版本支持 app-server：

```bash
codex --version
codex app-server --help
```

完成登录：

```bash
codex login
```

如果使用 API Key：

```bash
printf '%s' "$OPENAI_API_KEY" | codex login --with-api-key
```

认证信息应保存在应用 VPS 上，不要写入项目仓库或前端代码。

## 四、安装 Codex Web

项目地址：

```text
https://github.com/wangru8080/codex-web
```

安装：

```bash
npm install -g @wangru8080/codex-web
```

检查：

```bash
codex-web --help
```

Codex Web 的工作方式是：

```text
浏览器 → Codex Web → codex app-server → Codex CLI
```

浏览器不会直接启动本地进程，服务端 Web bridge 会启动本机的 `codex app-server`。

## 五、前台测试

在应用 VPS 上启动：

```bash
codex-web serve --host 127.0.0.1 --port 3001
```

另开一个终端检查：

```bash
ss -lntp | grep 3001
curl http://127.0.0.1:3001
```

确认服务可以启动后，再配置 systemd 常驻运行。

## 六、配置 systemd

先查找可执行文件路径：

```bash
command -v codex-web
command -v codex
```

创建服务：

```bash
sudo nano /etc/systemd/system/codex-web.service
```

示例：

```ini
[Unit]
Description=Codex Web
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=APP_USER
WorkingDirectory=/home/APP_USER
Environment=NODE_ENV=production
Environment=PORT=3001
ExecStart=/usr/local/bin/codex-web serve --host 127.0.0.1 --port 3001
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

将 `APP_USER` 和 `/usr/local/bin/codex-web` 替换为实际值。随后执行：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now codex-web
systemctl status codex-web
```

查看日志：

```bash
journalctl -u codex-web -n 100 --no-pager
```

## 七、配置 SSH / autossh 转发

入口 VPS 将 WireGuard 地址上的端口转发到应用 VPS 的本地端口：

```text
入口 VPS：10.20.0.1:3001
应用 VPS：127.0.0.1:3001
```

先手动测试：

```bash
ssh -N -T \
  -i /root/.ssh/codex-relay \
  -o BatchMode=yes \
  -o IdentitiesOnly=yes \
  -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -L ENTRY_VPS_WG_IP:3001:127.0.0.1:3001 \
  APP_USER@APP_VPS
```

如果需要自动重连，在入口 VPS 安装 autossh：

```bash
sudo apt update
sudo apt install -y autossh
```

创建服务：

```bash
sudo nano /etc/systemd/system/codex-web-relay.service
```

示例：

```ini
[Unit]
Description=Codex Web SSH Relay
Wants=network-online.target
After=network-online.target wg-quick@wg0.service
Requires=wg-quick@wg0.service

[Service]
Type=simple
User=root
Environment=AUTOSSH_GATETIME=0
ExecStart=/usr/bin/autossh \
  -M 0 \
  -N \
  -T \
  -i /root/.ssh/codex-relay \
  -o BatchMode=yes \
  -o IdentitiesOnly=yes \
  -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -L ENTRY_VPS_WG_IP:3001:127.0.0.1:3001 \
  APP_USER@APP_VPS
Restart=always
RestartSec=5
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

启动并检查：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now codex-web-relay
systemctl status codex-web-relay
ss -lntp | grep 3001
```

## 八、手机访问

手机开启 WireGuard 后，在浏览器打开：

```text
http://ENTRY_VPS_WG_IP:3001
```

完整访问链路：

```text
手机浏览器
 → WireGuard
 → 入口 VPS:3001
 → autossh
 → 应用 VPS:127.0.0.1:3001
 → Codex Web
```

## 九、第三方 API

Codex Web 负责界面和会话管理，模型请求仍由 Codex CLI / app-server 完成。

如果使用第三方 OpenAI-compatible API，需要确认中转服务支持：

- Responses API
- 流式输出
- Tool calling
- Reasoning 参数
- 较长上下文
- Codex 所需模型接口

只支持 Chat Completions 的中转服务，不一定能完整支持 Codex app-server。配置完成后，建议先测试：

```bash
codex exec "只回复 OK"
```

命令行测试成功后，再通过 Web 页面创建任务。

## 十、安全建议

- Codex Web 只监听 `127.0.0.1`。
- 只开放 WireGuard 的 UDP 端口。
- 不要把 Web 端口直接暴露到公网。
- SSH 使用密钥登录，并限制转发专用密钥的权限。
- 不要把 API Key、WireGuard 私钥和 SSH 私钥提交到 Git。
- Web 页面应启用登录保护。
- 生产项目与测试项目尽量分开。
- 不要直接使用项目示例中的多用户 root runtime 配置。

如果需要让 Codex 查看其他项目的日志，可以只授予指定日志目录的读取权限：

```bash
sudo setfacl -Rm u:APP_USER:rX /opt/other-project/logs
sudo setfacl -Rdm u:APP_USER:rX /opt/other-project/logs
```

## 十一、常见问题

### WireGuard 没有握手

检查：

```bash
sudo wg show
```

并确认云安全组允许 UDP `51820`。

### Web 页面无法打开

检查：

```bash
systemctl status codex-web
systemctl status codex-web-relay
ss -lntp | grep 3001
```

### 页面能打开但无法创建会话

检查：

```bash
codex login status
codex app-server --help
journalctl -u codex-web -f
```

### SSH 转发失败

检查：

- 应用 VPS 的 `3001` 是否监听
- SSH 用户和私钥是否正确
- SSH 服务端是否允许 TCP forwarding
- WireGuard 地址是否填写正确
- `autossh` 服务日志是否出现连接错误

## 总结

整个方案由四个部分组成：

```text
WireGuard：提供手机到入口 VPS 的私网连接
SSH / autossh：把入口 VPS 的端口转发到应用 VPS
Codex Web：提供浏览器工作台
Codex app-server：负责实际的 Agent 会话和工具执行
```

这样就可以在手机或 iPad 上，通过浏览器远程恢复会话、查看代码修改、处理工具审批，并继续运行 VPS 上的 Codex 任务。
