# 单台 VPS 部署 Codex Web，并通过 WireGuard 远程访问

本文介绍在一台 Linux VPS 上部署 [Codex Web](https://github.com/wangru8080/codex-web)，让手机或 iPad 通过 WireGuard 和浏览器远程使用 Codex。

```text
手机 / iPad
    │ WireGuard
    ▼
VPS：10.20.0.1:3001
    │
    ▼
Codex Web → codex app-server → Codex CLI
```

Codex Web 只监听 WireGuard 地址，不直接暴露到公网。访问需要通过 WireGuard 和 Codex Web 登录；模型服务的认证由 VPS 上的 Codex CLI 负责。

## 一、准备条件

- Ubuntu 22.04 或更新版本
- 2 核 CPU、2 GB 以上内存、20 GB 以上可用磁盘
- Node.js 20.9 或更高版本
- 一个公网 IPv4 地址

本文示例使用：

```text
WireGuard 网段：10.20.0.0/24
VPS WireGuard 地址：10.20.0.1
手机 WireGuard 地址：10.20.0.2
WireGuard 端口：51820/UDP
Codex Web 端口：3001/TCP
```

请把示例密钥和公网 IP 替换为自己的值。私钥、密码和 API Key 不要提交到 Git。

## 二、安装基础软件

```bash
sudo apt update
sudo apt install -y wireguard git acl openssl
node --version
npm --version
```

Node.js 需要满足 `>= 20.9.0`。如果版本过低，先使用 nvm、NodeSource 或发行版提供的方式升级。

## 三、配置 WireGuard

### 1. 生成 VPS 密钥

```bash
sudo sh -c 'umask 077; wg genkey > /etc/wireguard/server_private.key'
sudo sh -c 'wg pubkey < /etc/wireguard/server_private.key > /etc/wireguard/server_public.key'
```

服务器私钥只能留在 VPS 上。

### 2. 创建 VPS 配置

```bash
sudo nano /etc/wireguard/wg0.conf
```

写入：

```ini
[Interface]
Address = 10.20.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

[Peer]
PublicKey = PHONE_PUBLIC_KEY
AllowedIPs = 10.20.0.2/32
```

替换 `SERVER_PRIVATE_KEY` 和 `PHONE_PUBLIC_KEY`，然后保护文件：

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
```

### 3. 创建手机端配置

在手机 WireGuard App 中生成密钥，或在安全设备上生成：

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
Endpoint = VPS_PUBLIC_IP:51820
AllowedIPs = 10.20.0.0/24
PersistentKeepalive = 25
```

手机私钥只保存在手机；服务器私钥不能公开。

### 4. 启动并检查

```bash
sudo systemctl enable --now wg-quick@wg0
sudo wg show
```

云安全组需要放行 `UDP 51820`。如果使用 UFW：

```bash
sudo ufw allow 51820/udp
```

手机连接 WireGuard 后测试 `ping 10.20.0.1`。看到 `latest handshake` 和流量统计，说明连接正常。

## 四、安装 Codex CLI

```bash
npm install -g @openai/codex
codex --version
codex app-server --help
```

必须确认当前版本提供 `app-server` 子命令。

### Codex 模型认证

这是模型服务的认证，与 WireGuard 和 Web 登录相互独立。

使用 OpenAI 账户：

```bash
codex login
```

使用 API Key：

```bash
printf '%s' "$OPENAI_API_KEY" | codex login --with-api-key
```

如果使用第三方 OpenAI-compatible API，应按当前 Codex CLI 版本配置 provider、API 地址、模型和密钥。API Key 应保存在权限为 `600` 的环境文件或 Codex 配置目录中。

先验证命令行：

```bash
codex exec "只回复 OK"
```

## 五、安装 Codex Web

```bash
npm install -g @wangru8080/codex-web
codex-web --help
```

调用链为：

```text
浏览器 → Codex Web → codex app-server → Codex CLI → 模型 API
```

## 六、配置 Web 登录

建议启用独立的 Codex Web 登录。认证层次如下：

| 层级 | 作用 |
|---|---|
| WireGuard | 限制哪些设备能到达 Web 端口 |
| Codex Web | 防止已进入 VPN 的设备直接使用工作台 |
| Codex CLI | 认证 OpenAI 或第三方模型服务 |

单用户模式可使用以下环境变量（具体名称以所安装版本的项目 README 为准）：

```text
CODEX_WEB_LOGIN_EMAIL
CODEX_WEB_LOGIN_PASSWORD
CODEX_WEB_SESSION_SECRET
```

创建只有服务用户可读的环境文件：

```bash
sudo install -m 600 /dev/null /etc/codex-web.env
sudo nano /etc/codex-web.env
```

示例：

```dotenv
CODEX_WEB_LOGIN_EMAIL=you@example.com
CODEX_WEB_LOGIN_PASSWORD=CHANGE_THIS_TO_A_LONG_RANDOM_PASSWORD
CODEX_WEB_SESSION_SECRET=CHANGE_THIS_TO_A_RANDOM_SECRET
```

生成随机 Session secret：

```bash
openssl rand -hex 32
```

不要把真实密码和 secret 写入仓库。WireGuard 私钥不能代替 Web 密码，Web 密码也不能代替模型 API 认证。

## 七、前台启动测试

让服务只监听 VPS 的 WireGuard 地址：

```bash
set -a
. /etc/codex-web.env
set +a
codex-web serve --host 10.20.0.1 --port 3001
```

另开终端检查：

```bash
ss -lntp | grep 3001
curl http://10.20.0.1:3001
```

手机连接 WireGuard 后打开：

```text
http://10.20.0.1:3001
```

确认可以登录、创建会话、选择工作目录、看到 Codex 输出并处理审批后，再配置 systemd。

## 八、配置 systemd 常驻运行

查找路径：

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
After=network-online.target wg-quick@wg0.service
Requires=wg-quick@wg0.service

[Service]
Type=simple
User=APP_USER
WorkingDirectory=/home/APP_USER
EnvironmentFile=/etc/codex-web.env
Environment=NODE_ENV=production
Environment=PORT=3001
ExecStart=/usr/local/bin/codex-web serve --host 10.20.0.1 --port 3001
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

将 `APP_USER` 和 `/usr/local/bin/codex-web` 替换为实际值。确认环境文件权限：

```bash
sudo chown APP_USER:APP_USER /etc/codex-web.env
sudo chmod 600 /etc/codex-web.env
```

启动并查看日志：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now codex-web
systemctl status codex-web
journalctl -u codex-web -n 100 --no-pager
```

## 九、访问流程

```text
1. 手机启用 WireGuard，使用手机私钥完成 VPN 身份认证。
2. 浏览器打开 http://10.20.0.1:3001，输入 Codex Web 邮箱和密码。
3. Codex Web 启动 app-server，Codex CLI 使用已配置的账户或 API Key 调用模型。
```

## 十、读取其他项目日志

如果需要让 Codex 查看其他项目的日志，可以只授予指定目录读取权限：

```bash
sudo setfacl -Rm u:APP_USER:rX /opt/other-project/logs
sudo setfacl -Rdm u:APP_USER:rX /opt/other-project/logs
```

不要为了查看日志而把整个 VPS 的 root 权限暴露给 Web Agent。

## 十一、安全建议

- Codex Web 只绑定 `10.20.0.1`，不绑定公网地址。
- 公网只开放 WireGuard 的 UDP 端口。
- 保留 Web 登录保护，使用长随机密码和 Session secret。
- API Key、WireGuard 私钥、SSH 私钥和 Web 密码不要提交到 Git。
- Web 服务尽量使用普通 Linux 用户运行；使用 root 意味着 Agent 具备整台 VPS 的读写能力。
- 生产项目与测试项目尽量分开。

## 十二、常见问题

### WireGuard 没有握手

```bash
sudo wg show
```

检查云安全组是否放行 UDP `51820`，并确认两端公钥没有填反。

### Web 页面无法打开

```bash
systemctl status wg-quick@wg0
systemctl status codex-web
ss -lntp | grep 3001
```

确认服务监听 `10.20.0.1:3001`，手机端 `AllowedIPs` 包含 `10.20.0.0/24`。

### 页面能打开但无法创建会话

```bash
codex login status
codex app-server --help
journalctl -u codex-web -f
```

### Codex 没有模型输出

先执行：

```bash
codex exec "只回复 OK"
```

如果 CLI 本身无法调用模型，应先解决 Codex 登录、模型、API 地址或第三方中转兼容性问题。

## 总结

单主机方案只需要：

```text
WireGuard：让手机安全进入 VPS 私网
Codex Web：提供浏览器工作台和 Web 登录
codex app-server：管理会话、工具和审批
Codex CLI：执行 Agent 任务并调用模型
```

手机或 iPad 连接 WireGuard 后，打开 `http://10.20.0.1:3001`，完成 Web 登录即可远程使用 VPS 上的 Codex。
