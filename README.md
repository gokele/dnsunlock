# dnsunlock
<p align="center">
  <img src="https://raw.githubusercontent.com/gokele/dnsunlock/refs/heads/main/logo.png" width="120" />
</p>

<p align="center">
  单二进制 DNS 解锁服务器 · 零依赖 · 支持多平台
</p>

<p align="center">
  <img src="https://img.shields.io/github/v/release/gokele/dnsunlock?style=flat-square" />
  <img src="https://img.shields.io/github/downloads/gokele/dnsunlock/total?style=flat-square" />
  <img src="https://img.shields.io/github/license/gokele/dnsunlock?style=flat-square" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Linux-supported-2bbc8a?logo=linux&logoColor=white&style=flat-square" />
  <img src="https://img.shields.io/badge/macOS-supported-000000?logo=apple&logoColor=white&style=flat-square" />
  <img src="https://img.shields.io/badge/Windows-supported-0078D6?logo=windows&logoColor=white&style=flat-square" />
  <img src="https://img.shields.io/badge/amd64%20%7C%20arm64-supported-blue?style=flat-square" />
</p>
---

## 项目简介
单二进制 DNS 解锁服务器：在你自己的 VPS 上跑一份，把 Netflix / ChatGPT / Disney+ / YouTube 等地区受限服务的域名重定向到本机，再由内置 SNI 透明代理转发到真实目标。出口 IP 决定能解锁哪个区域。


支持 **Linux / macOS / Windows × amd64 / arm64** 共 6 种产物，按你的服务器架构自取。

---

## 快速开始

服务器要求：root（Linux/macOS）或管理员权限（Windows） + 公网 IPv4。

下载对应平台的二进制后,直接 sudo 运行就会进入交互式菜单 / 首次向导;熟手也可以一行命令 `install --clients` 直接装。

### Linux (amd64, 主流 Intel/AMD 服务器)

```bash
curl -LO https://github.com/gokele/dnsunlock/releases/latest/download/dnsunlock_linux_amd64 && chmod +x dnsunlock_linux_amd64 && sudo ./dnsunlock_linux_amd64
```

arm64 (Oracle Free / AWS Graviton / 树莓派) 把 `dnsunlock_linux_amd64` 全部换成 `dnsunlock_linux_arm64` 即可。

### macOS (Apple Silicon, M1/M2/M3/M4)

```bash
curl -LO https://github.com/gokele/dnsunlock/releases/latest/download/dnsunlock_macos_arm64 && chmod +x dnsunlock_macos_arm64 && sudo ./dnsunlock_macos_arm64
```

Intel Mac 把 `dnsunlock_macos_arm64` 换成 `dnsunlock_macos_amd64` 即可。

### Windows (amd64, 管理员 PowerShell)

```powershell
curl.exe -LO https://github.com/gokele/dnsunlock/releases/latest/download/dnsunlock_windows_amd64.exe ; .\dnsunlock_windows_amd64.exe
```

无参数运行进交互菜单,选 "1) 安装为系统服务" 跟随向导;熟手可以追加 `install --clients <your_client_ip>` 一行直装。

不论走哪条路,`install` 都会自动:

- 把二进制复制到该平台的标准位置
- 生成默认 `.env`(首次跑会询问 `PUBLIC_IP` / `OUTBOUND_PREFER` / `ALLOWED_CLIENTS` 等关键项)
- 注册系统服务(Linux 用 systemd / macOS 用 launchd / Windows 用 sc.exe)
- 处理端口 53 冲突(Linux 关 systemd-resolved stub,macOS / Windows 直接绑)
- 给二进制授予绑定低端口的权限(Linux setcap,macOS/Windows 服务以 root/SYSTEM 跑)
- 启动服务并设为开机自启

### `--clients` 是什么

客户端的**公网 IP 或 CIDR 段**。没在白名单里的客户端会被 53/80/443 直接拒绝,避免被陌生人白嫖带宽。

```bash
--clients 1.2.3.4
--clients 1.2.3.4,5.6.7.8,10.0.0.0/24
```

不知道自己公网 IP 时:在客户端机器上 `curl ifconfig.me`。交互菜单 / 向导也会询问这一项,可以等到那里再填。

---

## 交互式菜单

不带参数运行（且 stdin 是 TTY）会进入主菜单，第 1 项按当前服务状态自动切换：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  dnsunlock v0.x.x  (linux/amd64)
  状态: ✓ 运行中
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  1) 停止服务                           — stop
  2) 从 GitHub 升级到最新版本           — update
  3) 卸载系统服务                       — uninstall
  4) 添加客户端 IP/CIDR 到白名单        — allow
  5) 从白名单移除客户端                 — deny
  6) 查看当前白名单                     — list
  7) 自检 (.env / 网络 / 解锁目录)      — test
  8) 版本信息                           — version
  h) 完整帮助
  q) 退出
```

- **运行中** → 第 1 项是"停止服务"，已隐藏 install
- **已装未跑** → 第 1 项是"启动服务"，已隐藏 install
- **未安装** → 第 1 项是"未安装 (按 1 开始安装)"，按 1 直接进 install 流程

输入序号或对应英文别名（`stop`/`start`/`install`/`update`/...）都可以。systemd / launchd / sc 拉起的 daemon 走 `serve` 路径不会进菜单。

---

## 默认安装路径

| | Linux | macOS | Windows |
|---|---|---|---|
| 二进制 | `/usr/local/bin/dnsunlock` | `/usr/local/bin/dnsunlock` | `%ProgramData%\dnsunlock\dnsunlock.exe` |
| `.env` | `/etc/dnsunlock/.env` | `/usr/local/etc/dnsunlock/.env` | `%ProgramData%\dnsunlock\.env` |
| 服务定义 | `/etc/systemd/system/dnsunlock.service` | `/Library/LaunchDaemons/cc.habu.dnsunlock.plist` | Windows Service `dnsunlock` |
| 日志 | `journalctl -u dnsunlock` | `/usr/local/var/log/dnsunlock.log` | Windows Event Log（来源 `dnsunlock`） |

`install` 时可加 `--env <path>` 自定义 `.env` 位置。

---

## 客户端配置（让设备 DNS 指向这台服务器）

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

最快的方式是在服务器上直接跑自检：

```bash
sudo dnsunlock test
```

会逐项检查 `.env` / 监听端口 / 上游 DNS / 公网 IP / 区域 / 解锁目录命中情况，并给出修复建议。

也可以在客户端手动验证：

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

## 日常运维

### 增删客户端白名单（一条命令搞定）

```bash
sudo dnsunlock allow 1.2.3.4               # 加单个 IP
sudo dnsunlock allow 1.2.3.4 10.0.0.0/24   # 一次加多个 / 加 CIDR
sudo dnsunlock deny  1.2.3.4               # 删除
sudo dnsunlock list                         # 看当前列表
```

`allow` / `deny` 自动触发热重载 —— Linux/macOS 走 SIGHUP 不掉线，Windows 重启服务（秒级中断）。

**远程加自己 IP 的快捷写法**（在客户端上执行）：
```bash
ssh root@<server> "dnsunlock allow $(curl -s ifconfig.me)"
```

### 升级二进制（自动从 GitHub 拉最新）

```bash
sudo dnsunlock update              # 查最新 release, 有更新自动下载并重启
sudo dnsunlock update --check      # 只查不更新
sudo dnsunlock update --force      # 强制重装一次 (即使版本号相同)
sudo dnsunlock update --file /tmp/d # 离线场景: 用本地二进制升级
```

`update` 内部：查询最新 release → 比较 semver → 下载对应平台资产到临时文件 → 原子覆盖默认安装路径 → 重启服务。

### 查看版本

```bash
dnsunlock version
# dnsunlock v0.1.0
# platform:  linux/amd64
# go:        go1.26.2
```

### 修改 `.env`

打开对应路径（见上面默认路径表）编辑，然后：

| OS | 热重载 |
|---|---|
| Linux | `sudo systemctl reload dnsunlock` |
| macOS | `sudo launchctl kill HUP system/cc.habu.dnsunlock` |
| Windows | `sc stop dnsunlock; sc start dnsunlock`（重启，秒级中断） |

### 查看日志 / 状态

| OS | 状态 | 实时日志 |
|---|---|---|
| Linux | `systemctl status dnsunlock` | `journalctl -u dnsunlock -f` |
| macOS | `sudo launchctl print system/cc.habu.dnsunlock` | `tail -f /usr/local/var/log/dnsunlock.log` |
| Windows | `sc query dnsunlock` | `Get-EventLog -LogName Application -Source dnsunlock -Newest 50` |

每分钟会输出一条 stats 摘要，包含 DNS 查询数、命中数、代理字节数、当前白名单大小等。

### 手动重启服务（替换二进制后用）

| OS | 命令 |
|---|---|
| Linux | `sudo systemctl restart dnsunlock` |
| macOS | `sudo launchctl kickstart -k system/cc.habu.dnsunlock` |
| Windows | `sc stop dnsunlock; sc start dnsunlock` |

### 卸载

```bash
sudo dnsunlock uninstall
```

只移除服务定义，二进制和 `.env` 保留。完全清理：

```bash
# Linux
sudo rm /usr/local/bin/dnsunlock
sudo rm -rf /etc/dnsunlock
sudo userdel dnsunlock 2>/dev/null

# macOS
sudo rm /usr/local/bin/dnsunlock
sudo rm -rf /usr/local/etc/dnsunlock
sudo rm /usr/local/var/log/dnsunlock.log
```

```powershell
# Windows
Remove-Item -Recurse "$env:ProgramData\dnsunlock"
```

---

## 配置文件 `.env`

所有字段都有合理默认，初次部署一般只需要 `ALLOWED_CLIENTS`。

| 字段 | 默认 | 说明 |
|---|---|---|
| `LISTEN_DNS` | `:53` | DNS 监听地址（双栈） |
| `LISTEN_HTTP` | `:80` | HTTP 透明代理 |
| `LISTEN_HTTPS` | `:443` | TLS SNI 透明代理 |
| `PUBLIC_IP` | 自动探测 | 解锁域名 A 记录返回的 IP |
| `UPSTREAM_DNS` | 自动选择 | 留空 = 启动时探测 v6 出口决定 v4-only / 双栈默认列表 |
| `FILTER_AAAA` | `true` | 命中域名屏蔽 AAAA，防客户端走 v6 绕过 |
| `REJECT_QUIC` | `true` | 监听 UDP/443，对 QUIC 包回 Version Negotiation 强制客户端走 TCP |
| `OUTBOUND_PREFER` | `auto` | SNI proxy 出站偏好：`auto` (Happy Eyeballs 双栈并发) / `v4` / `v6` / `v4-then-v6` / `v6-then-v4` |
| `SERVER_REGION` | `AUTO` | 服务器出口地区 (ISO 3166-1 alpha-2)，决定按地区过滤启用哪些服务；显式填 `JP` / `SG` / `US` / `GB` / `HK` / `TW` / `CN` 等可覆盖 |
| `ALLOWED_CLIENTS` |（必填） | 客户端 IP / CIDR 白名单，逗号分隔 |
| `SUBSCRIPTION_INTERVAL` | `24h` | 订阅源刷新间隔 |
| `SUBSCRIPTION_TIMEOUT` | `30s` | 单次拉取超时 |
| `UPDATE_CHECK_INTERVAL` | `168h` | 周期性查 GitHub 是否有新版（仅日志提示，不自动安装），实际间隔随机 ±50%；`0` = 关闭 |
| `LOG_LEVEL` | `info` | `debug` / `info` / `warn` / `error` |
| `LOG_QUERY` | `false` | 是否逐条记录 DNS 查询（很啰嗦，调试用） |

修改后调用上面的"修改 `.env`"小节里对应平台的命令立即生效。监听端口、`UPSTREAM_DNS`、`REJECT_QUIC` 修改要 restart 而不是 reload。

---

## 内置可解锁服务

收录 50+ 个常用服务的域名清单，**全部默认启用**，不需要在 `.env` 里挑选。能否真正解锁取决于你服务器的出口 IP 区域：

- **AI**：OpenAI / Claude / Gemini / Copilot / Perplexity / Grok / HuggingFace / Suno / Midjourney / Character.AI
- **Google 系**：Google Search / YouTube / YouTube Music
- **全球流媒体**：Netflix / Disney+ / HBO Max / Prime Video / Apple TV+
- **美国**：Paramount+ / Peacock / Hulu
- **亚洲**：Viu / iQIYI / WeTV / Bilibili / Crunchyroll / meWATCH (SG) / TVB / myTV SUPER / Bahamut / KKTV / TencentVideo
- **音乐**：Spotify / Apple Music / TIDAL / Deezer
- **社交 / 工具**：TikTok / Twitch / Discord / Reddit / Notion / Civitai
- **日本**：LINE / AbemaTV / TVer / DAZN / U-NEXT / Niconico / Radiko / Pixiv / DMM
- **英国**：BBC iPlayer

启动时每个服务会自动从远端拉取最新域名清单并 24 小时刷新一次；拉不到时使用内置兜底域名。

---

## 已知限制

1. **HTTP/3 (QUIC)**：默认 `REJECT_QUIC=true`，服务端监听 UDP/443 并对 QUIC 初始包回 supported_versions 为空的 Version Negotiation，客户端按 RFC 9000 §6 立即放弃 QUIC 改走 TCP（毫秒级回退）。如果不想让我们碰 UDP/443，可以 `REJECT_QUIC=false` 关掉，但客户端可能要等 1-3 秒 QUIC 超时才回退。

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
| 服务状态 `failed`（Linux） | 53 被 systemd-resolved 占了 | `install` 已自动处理；如果绕开了，手工 `DNSStubListener=no` |
| `dnsunlock update` 报 GitHub 403 | API 速率限制 (60/小时) | 换个出口 IP 或稍后再试，已下载完不影响运行 |
| 端口 53/80/443 被占（macOS / Windows） | 别的服务在用 | 停掉占用方，或在 `.env` 改 `LISTEN_*` 端口 |

---

## 完整子命令

```
dnsunlock                       无参数: TTY 进交互式菜单, 非 TTY (systemd/launchd/sc) 走 serve
dnsunlock --env <path>          前台运行，自定义 .env 路径
dnsunlock serve [--env <path>]  显式 serve, 等价上面
dnsunlock menu                  强制进交互式菜单

dnsunlock install --clients X   首次安装并启动 (root/管理员)
  --env <path>                  .env 路径（默认按 OS）
  --public-ip <ip>              手动指定 PUBLIC_IP
  --skip-resolved-fix           Linux: 不动 systemd-resolved
  --keep-location               不复制二进制，直接用当前路径

dnsunlock update                从 GitHub 自动拉最新版升级
  --check                       只检查不下载
  --force                       即使版本相同也重装
  --file <path>                 用本地文件升级（离线场景）

dnsunlock uninstall             停止并卸载系统服务

dnsunlock allow <IP/CIDR>...    加白名单 + 热重载
dnsunlock deny  <IP/CIDR>...    去白名单 + 热重载
dnsunlock list                  打印当前白名单

dnsunlock test                  端到端自检 (.env / 监听端口 / 上游 / 公网 IP / 区域 / 解锁示例)

dnsunlock version               打印版本元信息
dnsunlock help                  帮助
```
