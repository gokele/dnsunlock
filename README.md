# dnsunlock

<p align="center">
  <img src="https://raw.githubusercontent.com/gokele/dnsunlock/refs/heads/main/logo.png" width="120" />
</p>

<p align="center">
  单二进制 DNS 解锁服务器 · 零依赖 · 一键部署
</p>

<p align="center">
  <img src="https://img.shields.io/github/v/release/gokele/dnsunlock?style=flat-square" />
  <img src="https://img.shields.io/github/downloads/gokele/dnsunlock/total?style=flat-square" />
  <img src="https://img.shields.io/github/license/gokele/dnsunlock?style=flat-square" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Linux-supported-2bbc8a?logo=linux&logoColor=white&style=flat-square" />
  <img src="https://img.shields.io/badge/Windows-supported-0078D6?logo=windows&logoColor=white&style=flat-square" />
  <img src="https://img.shields.io/badge/macOS-supported-000000?logo=apple&logoColor=white&style=flat-square" />
  <img src="https://img.shields.io/badge/Architecture-amd64%20%7C%20arm64-blue?style=flat-square" />
</p>

---

## 项目简介

dnsunlock 是一个单二进制 DNS 解锁服务，通过将特定域名解析到本机，并结合 SNI 透明代理，将请求转发至真实目标，从而实现对地区限制服务的访问。

特点：

* 无需 dnsmasq、sniproxy、数据库或前端
* 一个二进制文件即可运行
* 解锁能力由服务器出口 IP 决定
* 支持自动更新与热重载

---

## 工作原理

```mermaid
flowchart LR
    A[客户端] -->|DNS 查询| B[dnsunlock]
    B -->|返回服务器 IP| C[透明代理]
    C --> D[目标服务]
```

---

## 目录

* [项目简介](#项目简介)
* [特性](#特性)
* [快速开始](#快速开始)
* [客户端配置](#客户端配置)
* [验证](#验证)
* [日常运维](#日常运维)
* [配置说明](#配置说明)
* [支持服务](#支持服务)
* [限制](#限制)
* [排错](#排错)
* [常见问题](#常见问题)
* [贡献](#贡献)

---

## 特性

* 单二进制部署
* 内置 DNS / HTTP / HTTPS 服务
* IP 白名单控制
* 自动更新（GitHub Releases）
* 域名订阅自动刷新
* 配置热重载
* QUIC 自动回退 TCP

---

## 快速开始

服务器要求：

* Linux
* root 权限
* 公网 IPv4

安装：

```bash
curl -L -o /tmp/d \
https://github.com/gokele/dnsunlock/releases/latest/download/dnsunlock_linux_amd64

chmod +x /tmp/d && sudo /tmp/d install --clients <your_client_ip>
```

安装过程将自动完成：

* 安装二进制到 /usr/local/bin
* 创建配置文件 /etc/dnsunlock/.env
* 创建系统服务并启动
* 处理端口占用问题

查看日志：

```bash
journalctl -u dnsunlock -f
```

---

## 客户端配置

### Linux（systemd-resolved）

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

### Linux（resolv.conf）

```bash
nameserver <server_ip>
```

### macOS

系统设置 → 网络 → DNS → 添加服务器 IP

### Windows

设置 → 网络 → DNS → 手动配置服务器 IP

### 路由器

在 DHCP DNS 设置中填写服务器 IP

---

## 验证

```bash
nslookup chatgpt.com <server_ip>
nslookup google.com <server_ip>
```

---

## 日常运维

白名单管理：

```bash
dnsunlock allow 1.2.3.4
dnsunlock deny  1.2.3.4
dnsunlock list
```

更新：

```bash
dnsunlock update
```

修改配置：

```bash
vim /etc/dnsunlock/.env
systemctl reload dnsunlock
```

查看状态：

```bash
systemctl status dnsunlock
```

---

## 配置说明

| 参数              | 默认值  | 说明      |
| --------------- | ---- | ------- |
| LISTEN_DNS      | :53  | DNS 监听  |
| LISTEN_HTTP     | :80  | HTTP    |
| LISTEN_HTTPS    | :443 | HTTPS   |
| PUBLIC_IP       | 自动   | 返回 IP   |
| FILTER_AAAA     | true | 屏蔽 IPv6 |
| REJECT_QUIC     | true | 禁用 QUIC |
| ALLOWED_CLIENTS | 必填   | 客户端白名单  |

---

## 支持服务

* AI：OpenAI、Claude、Gemini、Copilot、Perplexity
* 流媒体：Netflix、Disney+、HBO、Prime Video、Apple TV+
* 亚洲：Bilibili、腾讯视频、爱奇艺、AbemaTV、TVer
* 音乐：Spotify、Apple Music、TIDAL

---

## 限制

* QUIC 需要回退至 TCP
* DoH / DoT 会绕过 DNS，需要关闭
* IPv6 可能绕过解析
* TLS ECH 未来可能影响

---

## 排错

| 问题       | 原因     | 解决          |
| -------- | ------ | ----------- |
| DNS 超时   | 未加入白名单 | 添加 IP       |
| 可解析但无法访问 | DoH 启用 | 关闭浏览器安全 DNS |
| 连接慢      | QUIC   | 保持默认配置      |
| 服务启动失败   | 端口占用   | 检查 53 端口    |

---

## 常见问题

### 为什么可以解锁？

请求通过服务器出口 IP 发出，因此使用该 IP 的地区属性。

### 是否记录数据？

不记录用户访问内容，仅做转发。
