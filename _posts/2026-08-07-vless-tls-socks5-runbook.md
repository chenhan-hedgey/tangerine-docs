---
layout: post
title: "VLESS + TLS + SOCKS5 链式代理：可复现部署与排障 Runbook"
date: 2026-08-07 15:00:00 +0800
description: "面向工程交接的脱敏部署、验证和排障手册"
tags: vless xray 3x-ui cloudflare clash socks5 运维
categories: 笔记
---

这是一份面向工程交接的脱敏 Runbook。它描述如何将 Cloudflare DNS、3X-UI/Xray、VLESS + TLS、Clash Verge Rev 和可选的 SOCKS5 链式代理组合起来，并通过可复现的命令定位问题。

文中的 <...> 都必须替换为自己的值。不要把真实域名、订阅令牌、面板路径、UUID、私钥或代理凭据提交到仓库。

## 1. 版本与边界

本流程在下列版本上验证过：

| 组件        |     版本 | 作用                                       |
| ----------- | -------: | ------------------------------------------ |
| 3X-UI       |      3.6 | 管理 Xray 入站、客户端与订阅               |
| Xray-core   |  26.7.28 | 服务端代理核心                             |
| Mihomo Meta |  1.19.29 | Clash Verge Rev 的代理核心                 |
| Cloudflare  | 托管 DNS | 提供域名解析；本流程的 Xray 入站使用仅 DNS |

这里的 main 是站点的**源码分支**。推送到 main 后，仓库现有的 GitHub Actions 会构建 Jekyll，并将静态产物发布到 gh-pages。因此不要直接修改 gh-pages 中生成的 HTML。

## 2. 目标架构

```text
Clash Verge Rev
  |
  +-- VLESS + TLS --> <VPS_IP>:8443
  |                    |
  |                    +-- 可选：SOCKS5 --> <RESIDENTIAL_SOCKS_HOST>:<PORT>
  |
  +-- HTTPS 订阅 ---> https://<PANEL_HOST>:<PANEL_PORT>/clash/<SUB_TOKEN>

Cloudflare DNS（仅 DNS）
  +-- <PANEL_HOST> A 记录 --> <VPS_IP>
```

如果 SOCKS5 节点使用 dialer-proxy 指向 VLESS 节点，SOCKS5 服务看到的来源地址是 VPS 的出口地址，而不是运行 Clash 的个人电脑。这一点对使用来源 IP 白名单的住宅代理尤其重要。

## 3. 部署前检查

准备以下参数：

| 参数           | 示例占位符                | 说明                     |
| -------------- | ------------------------- | ------------------------ |
| VPS 公网地址   | <VPS_IP>                  | 可 SSH 管理的 Linux 主机 |
| 面板域名       | <PANEL_HOST>              | 例如 panel.example.com   |
| 面板端口       | <PANEL_PORT>              | 例如 54321               |
| TLS 入站端口   | <TLS_PORT>                | 本文使用 8443            |
| TLS 客户端别名 | <TLS_CLIENT_NAME>         | 例如 US-TLS-primary      |
| SOCKS5 服务    | <SOCKS_HOST>:<SOCKS_PORT> | 由服务商提供             |

确认 DNS 解析指向 VPS：

```bash
dig +short <PANEL_HOST>
```

预期：返回 <VPS_IP>。用于 Xray/3X-UI 入站的记录必须设置为 Cloudflare **仅 DNS**（灰云）；若启用了代理，非 HTTP 的入站端口通常无法正常工作。

在云厂商安全组或防火墙中放行：

- TCP <PANEL_PORT>：仅管理来源；不要向公网开放。
- TCP <TLS_PORT>：供 VLESS + TLS 客户端使用。
- TCP 80：首次 HTTP-01 证书签发时需要；使用 DNS-01 时按 DNS API 权限配置。

## 4. 证书与 VLESS + TLS 入站

先通过 3X-UI 的证书管理/ACME 功能为 <PANEL_HOST> 申请证书。以下是常见的证书文件约定：

```text
/root/cert/<PANEL_HOST>/fullchain.pem
/root/cert/<PANEL_HOST>/privkey.pem
```

然后创建一个独立的 VLESS 入站，关键参数如下：

| 字段              | 值                       |
| ----------------- | ------------------------ |
| 协议              | vless                    |
| 传输              | tcp                      |
| 安全              | tls                      |
| 监听端口          | <TLS_PORT>               |
| SNI / Server Name | <PANEL_HOST>             |
| 证书              | fullchain.pem            |
| 私钥              | privkey.pem              |
| 客户端 UUID       | 新生成，不与其他入站复用 |

创建后，在 VPS 上验证服务进程、端口和证书：

```bash
systemctl is-active x-ui
ss -ltn | grep ":<TLS_PORT>"
openssl s_client -connect 127.0.0.1:<TLS_PORT> -servername <PANEL_HOST> -brief </dev/null
```

预期分别为：

1. active；
2. Xray 在 <TLS_PORT> 监听；
3. 输出中出现 Peer certificate，且证书主体是 <PANEL_HOST>。

## 5. 生成 Clash YAML 订阅

3X-UI 的普通订阅端点可能返回 Base64 编码的 URI 列表，适合某些客户端，但不能直接作为 Clash 配置导入。Clash Verge Rev 需要的是 YAML。

在 3X-UI 中启用 Clash 订阅后，使用类似下面的 URL：

```text
https://<PANEL_HOST>:<PANEL_PORT>/clash/<SUB_TOKEN>
```

在 VPS 或任意可信终端检查内容：

```bash
curl -fsSL "https://<PANEL_HOST>:<PANEL_PORT>/clash/<SUB_TOKEN>" | sed -n '1,80p'
```

预期内容以 proxies: 开始，并包含 TLS 节点，例如：

```yaml
proxies:
  - name: <TLS_CLIENT_NAME>
    type: vless
    server: <PANEL_HOST>
    port: <TLS_PORT>
    network: tcp
    tls: true
    servername: <PANEL_HOST>
```

将此 URL 作为订阅地址导入 Clash Verge Rev。

## 6. 多订阅与配置文件脚本

Clash Verge Rev 的“全局扩展脚本”会作用于每一份订阅。若脚本引用了只存在于订阅 A 的节点，而订阅 B 没有该节点，验证将出现类似错误：

```text
proxy [<NODE_NAME>] dialer-proxy [<MISSING_NODE>] not found
```

正确做法是：

1. 将仅属于订阅 A 的脚本放入订阅 A 的**配置文件脚本**。
2. 将订阅 B 的脚本放入订阅 B 的**配置文件脚本**。
3. 全局脚本保持空操作，或仅放置不依赖任一订阅节点的通用规则。

下面模板将 SOCKS5 节点注入最新订阅的 PROXY 组，并让它经 TLS 节点出站：

```js
function main(config, profileName) {
  const node = {
    name: "residential-socks",
    type: "socks5",
    server: "<RESIDENTIAL_SOCKS_HOST>",
    port: <RESIDENTIAL_SOCKS_PORT>,
    username: "<RESIDENTIAL_SOCKS_USERNAME>",
    password: "<RESIDENTIAL_SOCKS_PASSWORD>",
    udp: false,
    "dialer-proxy": "<TLS_CLIENT_NAME>"
  };

  config.proxies = config.proxies || [];
  if (!config.proxies.some((proxy) => proxy.name === node.name)) {
    config.proxies.unshift(node);
  }

  for (const group of config["proxy-groups"] || []) {
    if (group.name === "PROXY") {
      group.proxies = group.proxies || [];
      if (!group.proxies.includes(node.name)) {
        group.proxies.unshift(node.name);
      }
    }
  }

  return config;
}
```

组名不是固定的。先打开新订阅的 YAML，确认 proxy-groups 下的真实名称；若它叫 SELECT，则将脚本中的 PROXY 改为 SELECT。不要猜测或跨订阅引用节点名称。

## 7. SOCKS5 的三层验证

从 VPS 验证最有意义，因为它模拟了 dialer-proxy 的真实来源。以下命令会提示输入凭据，避免密码留在 shell 历史中：

```bash
python3 - <<'PY'
import getpass
import socket
import struct

HOST, PORT = "<RESIDENTIAL_SOCKS_HOST>", <RESIDENTIAL_SOCKS_PORT>

def receive_exact(sock, size):
    data = b""
    while len(data) < size:
        part = sock.recv(size - len(data))
        if not part:
            raise RuntimeError("connection closed by peer")
        data += part
    return data

username = input("SOCKS username: ").encode()
password = getpass.getpass("SOCKS password: ").encode()

sock = socket.create_connection((HOST, PORT), timeout=15)
sock.settimeout(20)
print("1. TCP: OK")

sock.sendall(bytes([5, 1, 2]))
if receive_exact(sock, 2) != bytes([5, 2]):
    raise RuntimeError("unexpected SOCKS5 authentication method")
print("2. SOCKS5 method: username/password")

sock.sendall(bytes([1, len(username)]) + username + bytes([len(password)]) + password)
if receive_exact(sock, 2) != bytes([1, 0]):
    raise RuntimeError("SOCKS5 authentication failed")
print("3. SOCKS5 authentication: OK")

domain = b"api.ipify.org"
sock.sendall(bytes([5, 1, 0, 3, len(domain)]) + domain + struct.pack("!H", 80))
reply = receive_exact(sock, 4)
if reply[0] != 5 or reply[1] != 0:
    raise RuntimeError(f"SOCKS5 CONNECT failed: {reply.hex()}")

length = {1: 4, 4: 16}.get(reply[3])
if length is None and reply[3] == 3:
    length = receive_exact(sock, 1)[0]
if length is None:
    raise RuntimeError("unknown SOCKS5 reply address type")
receive_exact(sock, length + 2)

sock.sendall(b"GET / HTTP/1.1\r\nHost: api.ipify.org\r\nConnection: close\r\n\r\n")
print("4. Outbound request: OK")
print(sock.recv(1024).decode("utf-8", "replace"))
sock.close()
PY
```

| 失败阶段        | 含义                       | 下一步                                           |
| --------------- | -------------------------- | ------------------------------------------------ |
| TCP 超时        | VPS 到代理入口不通         | 检查服务商入口是否轮换、来源 IP 白名单和网络策略 |
| TCP 拒绝        | 目标端口关闭               | 向服务商确认最新地址和端口                       |
| SOCKS5 握手异常 | 地址/端口不是预期的 SOCKS5 | 核对协议、端口和服务商文档                       |
| 认证失败        | 凭据、套餐或授权失败       | 核对套餐状态、密码和 IP 授权方式                 |
| CONNECT 失败    | 已认证但不能出网           | 检查配额、目标限制和服务商线路                   |

## 8. 常见故障

### 8.1 expected a YAML mapping

导入了普通 Base64 订阅，而不是 Clash YAML 订阅。改用 /clash/<SUB_TOKEN> 类型的订阅端点，并确认响应以 proxies: 开始。

### 8.2 dialer-proxy not found

配置文件脚本中引用的节点不存在于当前订阅。检查组名和节点名，并将脚本从全局范围移到对应订阅的配置文件范围。

### 8.3 REALITY authentication failed

优先比对服务器和客户端的 UUID、SNI、Short ID、Public Key、传输类型与流控参数。若仍无法稳定互通，部署一个独立的 VLESS + TLS 入站作为兼容路径；TLS 使用标准证书，排障面更小。

### 8.4 住宅代理以前可用、现在 TCP 超时

通常不是 Clash 脚本问题。住宅代理服务商可能轮换入口地址，或根据来源 IP、套餐状态和网络策略限制访问。先在 VPS 上运行三层验证，再向服务商确认当前入口、端口、认证方式和是否需要将 <VPS_IP> 加入白名单。

## 9. 交接清单

- [ ] 记录组件版本与升级日期。
- [ ] 记录所有端口的用途和云防火墙规则。
- [ ] 将证书路径、续期方式和到期提醒写入内部密码库或运维系统。
- [ ] 保存订阅 URL 的存放位置，不把令牌写进文档。
- [ ] 分别记录每个订阅的配置文件脚本用途和依赖节点。
- [ ] 对 SOCKS5 服务记录供应商、当前入口、套餐到期时间和授权模型。
- [ ] 在入口变更、升级 Xray 或修改传输协议后，重跑第 4、5、7 节的验证。

这份流程将“服务是否存活”“订阅格式是否正确”“客户端配置是否引用了有效节点”“住宅代理是否允许 VPS 来源”拆分验证。只有先定位到失败层，才能避免把网络、订阅和凭据问题混为一谈。
