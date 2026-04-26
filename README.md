# dnsunlock

> 仓库: <https://github.com/gokele/dnsunlock>

单二进制 DNS 解锁服务器：在你自己的 VPS 上跑一份，把 Netflix / ChatGPT / Disney+ / YouTube 等地区受限服务的域名重定向到本机，再由内置 SNI 透明代理转发到真实目标。出口 IP 决定能解锁哪个区域，代码本身和地区无关。

无外部依赖：不装 dnsmasq、不装 sniproxy、不装数据库、不需要前端。一个二进制 + 一份 `.env` 解决一切。

---

## 快速开始（一条命令完成持续运行）

服务器要求：Linux + root 权限 + 公网 IPv4。

```bash
# 1. 从 GitHub Releases 下载对应架构的二进制
curl -L -o /tmp/d \
  https://github.com/gokele/dnsunlock/releases/latest/download/dnsunlock_linux_amd64
# ARM64 (Oracle Free / AWS Graviton / 树莓派) 用这个:
# curl -L -o /tmp/d https://github.com/gokele/dnsunlock/releases/latest/download/dnsunlock_linux_arm64

# 2. 一条命令完成: chmod + install (会自动启动 systemd 服务)
chmod +x /tmp/d && sudo /tmp/d install --clients <your_client_ip>
```

`install` 内部自动做了：
- 二进制原子复制到 `/usr/local/bin/dnsunlock`
- 生成 `/etc/dnsunlock/.env`（默认值合理，按需要后续修改）
- 创建 `dnsunlock` 系统用户
- `setcap CAP_NET_BIND_SERVICE` 让非 root 也能绑 53/80/443
- 自动关闭 `systemd-resolved` 的 53 端口占用（如果在用）
- 写 systemd unit 并 `enable + start`

执行完毕后运行 `journalctl -u dnsunlock -f` 看日志确认正常。

### `--clients` 是什么

客户端的**公网 IP 或 CIDR 段**。没在白名单里的客户端会被 53/80/443 直接拒绝，避免被陌生人白嫖带宽。

```bash
--clients 1.2.3.4
--clients 1.2.3.4,5.6.7.8,10.0.0.0/24
```

不知道自己公网 IP 时：在客户端机器上 `curl ifconfig.me`。

---

## 客户端配置（让 Linux / Windows / 路由器把 DNS 指向这台服务器）

### Linux（systemd-resolved，Debian 12+ / Ubuntu 默认）

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/dnsunlock.conf <<EOF
[Resolve]
DNS=<server_ip>
FallbackDNS=
Domains=~.
DNSStubListener=yes
EOF
sudo systemctl restart systemd-resolved
```

### Linux（直接改 resolv.conf，纯服务器场景）

```bash
sudo tee /etc/resolv.conf <<EOF
nameserver <server_ip>
nameserver 1.1.1.1
EOF
sudo chattr +i /etc/resolv.conf       # 防止 DHCP 覆盖
```

### macOS

系统设置 → 网络 → 当前连接 → 详细信息 → DNS → 把 DNS 服务器改成你的服务器 IP。

### Windows

设置 → 网络 → 以太网/WiFi → 编辑 DNS 服务器分配 → 手动 → 填入服务器 IP。

### 路由器

家用路由器后台一般有"DNS 服务器"或"DHCP DNS"设置，填服务器 IP，整个家庭网络的设备都自动用。

---

## 验证

在客户端跑：

```bash
# 解锁域名应返回服务器 IP
nslookup chatgpt.com <server_ip>

# 普通域名应返回真实 IP
nslookup google.com <server_ip>

# 实际访问验证
curl -v --resolve chatgpt.com:443:<server_ip> https://chatgpt.com/cdn-cgi/trace
```

测试失败时优先看：
- 浏览器是否开了 DoH / 安全 DNS（关掉，否则它绕过系统 DNS）
- 客户端是否有 IPv6（关掉 v6 或确保 DNS 配置覆盖 v6）
- 公网 IP 是否在 `ALLOWED_CLIENTS` 里（动态 IP 经常变，需要重新加）

---

## 日常运维（一条命令搞定）

### 增删客户端白名单

```bash
sudo dnsunlock allow 1.2.3.4               # 加单个 IP
sudo dnsunlock allow 1.2.3.4 10.0.0.0/24   # 一次加多个
sudo dnsunlock deny  1.2.3.4               # 删除
sudo dnsunlock list                         # 看当前列表
```

`allow` / `deny` 自动 `systemctl reload`，不掉线、不断连接。

**远程加自己 IP 的快捷写法**（在客户端上执行）：
```bash
ssh root@<server> "dnsunlock allow $(curl -s ifconfig.me)"
```

### 升级二进制（从 GitHub 自动拉最新）

```bash
sudo dnsunlock update              # 查最新 release, 有更新则自动下载并重启
sudo dnsunlock update --check      # 只查不更新
sudo dnsunlock update --force      # 强制重装一次 (即使版本号相同)
sudo dnsunlock update --file /tmp/d # 离线场景: 用本地二进制升级
```

`update` 内部：查询 `https://api.github.com/repos/gokele/dnsunlock/releases/latest` → 比较 semver → 下载对应架构资产到临时文件 → 原子替换 `/usr/local/bin/dnsunlock`（运行中的旧进程不影响）→ `setcap` → `systemctl restart`。

### 查看版本

```bash
dnsunlock version
# dnsunlock v0.1.0
# source:    https://github.com/gokele/dnsunlock
# platform:  linux/amd64
# go:        go1.26.2
```

### 修改 `.env`

```bash
sudo vim /etc/dnsunlock/.env
sudo systemctl reload dnsunlock     # 触发热重载, 不重启进程
```

### 查看日志 / 状态

```bash
systemctl status dnsunlock                  # 状态
journalctl -u dnsunlock -f                  # 实时日志
journalctl -u dnsunlock --since 1h          # 最近 1 小时
```

每分钟会输出一条 stats 摘要，包含 DNS 查询数、命中数、代理字节数、当前白名单大小等。

### 卸载

```bash
sudo dnsunlock uninstall
# 二进制 / .env / 用户保留. 完全清理:
sudo rm /usr/local/bin/dnsunlock
sudo rm -rf /etc/dnsunlock
sudo userdel dnsunlock
```

---

## 配置文件 `.env`

`/etc/dnsunlock/.env`，所有字段都有合理默认，初次部署一般只需要调整 `ALLOWED_CLIENTS`。

| 字段 | 默认 | 说明 |
|---|---|---|
| `LISTEN_DNS` | `:53` | DNS 监听地址（双栈） |
| `LISTEN_HTTP` | `:80` | HTTP 透明代理 |
| `LISTEN_HTTPS` | `:443` | TLS SNI 透明代理 |
| `PUBLIC_IP` | 自动探测 | 解锁域名 A 记录返回的 IP |
| `UPSTREAM_DNS` | 自动选择 | 留空 = 启动时探测 v6 出口决定 v4-only / 双栈默认列表，否则按你写的来 |
| `FILTER_AAAA` | `true` | 命中域名屏蔽 AAAA，防客户端走 v6 绕过 |
| `REJECT_QUIC` | `true` | 监听 UDP/443，对 QUIC 包回 Version Negotiation 强制客户端走 TCP |
| `ALLOWED_CLIENTS` |（必填） | 客户端 IP / CIDR 白名单，逗号分隔 |
| `SUBSCRIPTION_INTERVAL` | `24h` | 订阅源刷新间隔 |
| `SUBSCRIPTION_TIMEOUT` | `30s` | 单次拉取超时 |
| `LOG_LEVEL` | `info` | `debug` / `info` / `warn` / `error` |
| `LOG_QUERY` | `false` | 是否逐条记录 DNS 查询（很啰嗦，调试用） |

修改后 `systemctl reload dnsunlock` 立即生效（除了监听端口和 `UPSTREAM_DNS`，这两个改完要 `restart`）。

---

## 内置可解锁服务

代码里已经收录 50+ 个常用服务的域名清单，**全部默认启用**，不需要在 `.env` 里挑选。能否真正解锁取决于你服务器的出口 IP 区域：

- **AI**：OpenAI / Claude / Gemini / Copilot / Perplexity / Grok / HuggingFace / Suno / Midjourney / Character.AI
- **Google 系**：Google Search / YouTube / YouTube Music
- **全球流媒体**：Netflix / Disney+ / HBO Max / Prime Video / Apple TV+
- **美国**：Paramount+ / Peacock / Hulu
- **亚洲**：Viu / iQIYI / WeTV / Bilibili / Crunchyroll / meWATCH (SG) / TVB / myTV SUPER / Bahamut / KKTV / TencentVideo
- **音乐**：Spotify / Apple Music / TIDAL / Deezer
- **社交 / 工具**：TikTok / Twitch / Discord / Reddit / Notion / Civitai
- **日本**：LINE / AbemaTV / TVer / DAZN / U-NEXT / Niconico / Radiko / Pixiv / DMM
- **英国**：BBC iPlayer

启动时每个服务从 GitHub 自动拉取最新域名清单（未来 24h 自动刷新）；拉不到就用代码内置的兜底域名。

---

## 已知限制

1. **HTTP/3 (QUIC)**：默认 `REJECT_QUIC=true`，服务端会监听 UDP/443 并对 QUIC 初始包回 supported_versions 为空的 Version Negotiation，客户端按 RFC 9000 §6 立即放弃 QUIC 改走 TCP（毫秒级回退，无超时）。如果你不想让我们碰 UDP/443（端口被别的 QUIC 服务占用等），可以 `REJECT_QUIC=false` 关掉，但客户端可能要等 1-3 秒 QUIC 超时才回退。

2. **DoH / DoT 旁路**：Firefox 默认开 DoH (Cloudflare)、iOS 私人中继、Windows 11 安全 DNS 都会绕过系统 DNS，解锁失效。**必须**关掉浏览器和系统的"安全 DNS"。

3. **TLS ECH**：未来 ECH 普及后，SNI 加密了我们看不到目标域名，L4 透明代理这条路就走不通了。目前 ECH 部署率很低，能稳定用 1-2 年。

4. **客户端 IPv6 绕过**：客户端如果有 IPv6 公网，AAAA 查询路径上必须经过我们的 DNS 才会被 `FILTER_AAAA` 过滤。如果客户端有别的 IPv6 DNS（路由器 RA 推下来的），AAAA 会绕过我们。客户端配置时要把 v6 的 DNS 也指向我们（或干脆禁用 v6）。

---

## 常用排错

| 现象 | 可能原因 | 处理 |
|---|---|---|
| `nslookup` 返回 `;; connection timed out` | 客户端 IP 不在白名单 | `dnsunlock allow <ip>` |
| `nslookup` 通了但应用打不开 | 浏览器开了 DoH | 关掉浏览器的"安全 DNS" |
| 视频缓冲很慢首次连接卡 | QUIC 在尝试 UDP/443 | 默认 `REJECT_QUIC=true` 已自动处理；若关掉了重新打开 |
| `systemctl status dnsunlock` 是 `failed` | 53 被 systemd-resolved 占了 | install 已自动处理；如果绕开了，手工 `DNSStubListener=no` |
| `dnsunlock update` 报 `update 未启用` | 这份二进制编译时没注入 repo | 用 release 页面的官方二进制重装一次 |
| `dnsunlock update` 报 GitHub 403 | API 速率限制 (60/小时未登录) | 换个出口 IP 或稍后再试，已下载完不影响运行 |

---

## 完整子命令

```
dnsunlock                       前台运行（开发调试）
dnsunlock --env <path>          前台运行，自定义 .env 路径

dnsunlock install --clients X   首次安装并启动（root 必需）
dnsunlock update                从 GitHub Releases 拉最新版（root 必需）
  --check                       只检查不下载
  --force                       即使版本相同也重装
  --file <path>                 用本地文件升级（离线场景）
dnsunlock uninstall             停止并移除 systemd 服务

dnsunlock allow <IP/CIDR>...    加白名单 + 热重载
dnsunlock deny  <IP/CIDR>...    去白名单 + 热重载
dnsunlock list                  打印当前白名单

dnsunlock version               打印版本元信息
dnsunlock help                  帮助
```
