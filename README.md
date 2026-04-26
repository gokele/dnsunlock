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

dnsunlock 是一个单二进制 DNS 解锁服务器：

在你自己的 VPS 上运行，将 Netflix / ChatGPT / Disney+ / YouTube 等地区受限服务的域名解析到本机，并通过内置 SNI 透明代理转发至真实目标。

解锁能力由服务器出口 IP 决定。

---

## 工作原理

```mermaid
flowchart LR
    A[客户端] -->|DNS 查询| B[dnsunlock]
    B -->|返回服务器 IP| C[SNI 透明代理]
    C --> D[真实服务]
```

---

## 目录

* [快速开始](#快速开始)
* [默认安装路径](#默认安装路径)
* [客户端配置](#客户端配置)
* [验证](#验证)
* [日常运维](#日常运维)
* [配置文件](#配置文件-env)
* [支持服务](#内置可解锁服务)
* [限制](#已知限制)
* [排错](#常用排错)
* [命令](#完整子命令)

---

## 快速开始

服务器要求：

* root（Linux/macOS）或管理员权限（Windows）
* 公网 IPv4

---

### Linux

```bash
curl -L -o /tmp/d https://github.com/gokele/dnsunlock/releases/latest/download/dnsunlock_linux_amd64
chmod +x /tmp/d && sudo /tmp/d install --clients <your_client_ip>
```

---

### macOS

```bash
curl -L -o /tmp/d https://github.com/gokele/dnsunlock/releases/latest/download/dnsunlock_macos_arm64
chmod +x /tmp/d && sudo /tmp/d install --clients <your_client_ip>
```

---

### Windows

```powershell
curl.exe -L -o $env:TEMP\dnsunlock.exe https://github.com/gokele/dnsunlock/releases/latest/download/dnsunlock_windows_amd64.exe
& "$env:TEMP\dnsunlock.exe" install --clients <your_client_ip>
```

---

### `--clients` 说明

```bash
--clients 1.2.3.4
--clients 1.2.3.4,10.0.0.0/24
```

获取公网 IP：

```bash
curl ifconfig.me
```

---

## 默认安装路径

|     | Linux                      | macOS                           | Windows                                 |
| --- | -------------------------- | ------------------------------- | --------------------------------------- |
| 二进制 | `/usr/local/bin/dnsunlock` | `/usr/local/bin/dnsunlock`      | `%ProgramData%\dnsunlock\dnsunlock.exe` |
| 配置  | `/etc/dnsunlock/.env`      | `/usr/local/etc/dnsunlock/.env` | `%ProgramData%\dnsunlock\.env`          |
| 服务  | systemd                    | launchd                         | Windows Service                         |
| 日志  | journalctl                 | 文件日志                            | Event Log                               |

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

```bash
nslookup chatgpt.com <server_ip>
nslookup google.com <server_ip>
```

---

## 日常运维

```bash
dnsunlock allow 1.2.3.4
dnsunlock deny 1.2.3.4
dnsunlock update
dnsunlock list
```

---

## 配置文件 `.env`

| 字段              | 默认   | 说明    |
| --------------- | ---- | ----- |
| LISTEN_DNS      | :53  | DNS   |
| LISTEN_HTTP     | :80  | HTTP  |
| LISTEN_HTTPS    | :443 | HTTPS |
| ALLOWED_CLIENTS | 必填   | 白名单   |

---

## 内置可解锁服务

* AI：OpenAI / Claude / Gemini / Copilot
* 流媒体：Netflix / Disney+ / HBO
* 亚洲：Bilibili / 腾讯 / 爱奇艺
* 音乐：Spotify / Apple Music

---

## 已知限制

* QUIC 需回退 TCP
* DoH / DoT 会绕过 DNS
* IPv6 可能绕过
* TLS ECH 未来影响

---

## 常用排错

| 问题     | 原因   | 解决    |
| ------ | ---- | ----- |
| DNS 超时 | 未白名单 | allow |
| 无法访问   | DoH  | 关闭    |
| 卡顿     | QUIC | 默认开启  |

---

## 完整子命令

```
dnsunlock install --clients X
dnsunlock update
dnsunlock uninstall
dnsunlock allow / deny
dnsunlock list
dnsunlock version
dnsunlock help
```

---
