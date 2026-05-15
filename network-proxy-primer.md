# 网络代理是如何工作的：从原理到调式

> 本文以 Mihomo (Clash) + Xray 搭建代理链的实战经验为背景，系统介绍网络代理、流量加密、协议伪装的核心概念。

---

## 一、为什么需要代理？

### 1.1 裸奔的 HTTP 请求

当你直接访问 `https://www.google.com` 时，你的请求大概是这个样子：

```
客户端 ──────────────────────────────► Google 服务器
     源IP: 115.191.29.26 (中国)
     目标IP: 142.250.x.x
     SNI: www.google.com
     TLS 证书: google.com
```

一路上每一跳（路由器、ISP、墙）都能看到：你在访问 Google，你的 IP 是什么，你的 ISP 知道你在干什么。GFW（长城防火墙）可以：
- 识别特定域名（SNI 检测）
- 阻断连接（TCP RST / 流量丢弃）
- 记录你的访问行为

### 1.2 代理的本质

代理就是**让你先连到一个可信的中转服务器，再由它替你访问目标**：

```
客户端 ──────── 加密隧道 ────────► 中转服务器（海外） ──────► Google
     源IP: 海外IP                              目标IP: google.com
     SNI: 海外服务器域名                        看起来是正常 HTTPS
```

- **GFW 看到**：你在连接一个海外服务器（假设无害）
- **Google 看到**：请求来自海外 IP，不是中国

关键在于：**让墙认为这个连接是无害的**。

---

## 二、协议的分层结构

理解代理的第一步是理解网络分层模型：

```
┌─────────────────────────────────────────┐
│  7. 应用层   (HTTP, WebSocket, DNS)       │  ← 应用数据
├─────────────────────────────────────────┤
│  6. 表示层   (TLS 加密, JSON, Protobuf)  │  ← 加密/编码
├─────────────────────────────────────────┤
│  5. 会话层   (连接管理)                   │
├─────────────────────────────────────────┤
│  4. 传输层   (TCP, UDP, TLS 握手)         │  ← 端口、流量控制
├─────────────────────────────────────────┤
│  3. 网络层   (IP 路由)                    │  ← IP 地址
├─────────────────────────────────────────┤
│  2. 数据链路层 (Ethernet, ARP)            │
├─────────────────────────────────────────┤
│  1. 物理层   (光纤、电缆)                  │
└─────────────────────────────────────────┘
```

调试的核心思路：**从下往上，逐层排查**。

### 为什么 TLS 通了，但代理仍然失败？

这是一个经典问题，也是这个 skill 反复遇到的情况：

```
Layer 1-4 (IP + TCP + TLS): ✅ 连接建立成功
Layer 5-7 (HTTP/WebSocket): ❌ 应用层返回 404
```

就像打电话：
- 电话线通了（TLS 握手成功）
- 但对方说的是"您拨打的号码不存在"（HTTP 404）

---

## 三、TLS 层——为什么那么重要？

### 3.1 什么是 TLS？

TLS（Transport Layer Security）是 HTTPS 的安全基础，它提供：
- **加密**：第三方无法窥探内容
- **身份认证**：证明你连接的是真实的服务器（而非中间人）
- **完整性校验**：数据不被篡改

TLS 握手过程（简化版）：

```
客户端                              服务端
  │                                   │
  │──── ClientHello (支持的TLS版本,    │
  │      加密套件, SNI)              │──► SNI = www.microsoft.com
  │◄─── ServerHello (选定加密套件)     │
  │◄─── 证书链 (microsoft.com)        │
  │◄─── ServerKeyExchange             │
  │──── ClientKeyExchange ───────────►│
  │    (用公钥加密会话密钥)             │
  │==== 加密通道建立 ======            │
  │    (之后的 HTTP 请求都加密)         │
```

### 3.2 SNI——墙识别的关键

**SNI（Server Name Indication）** 是 TLS ClientHello 中明文传输的字段：

```
明文可见：
  SNI: www.google.com   ← 墙直接知道你要访问 Google
```

这就是为什么裸连 Google 会被发现——TLS 握手第一步 SNI 就暴露了目标。

### 3.3 证书链——身份认证

真实的 HTTPS 网站有证书链验证：

```
根证书 (CA) ──► 中间证书 ──► 服务器证书 (www.google.com)
     ↑              ↑               ↑
  内置于系统      签名              签名
```

浏览器会验证：证书是不是由可信 CA 签发的、域名是否匹配、是否过期。

---

## 四、代理协议：VLESS、VMess、Shadowsocks

### 4.1 什么是 VLESS？

VLESS 是一个**轻量级代理协议**，由 Xray 开发。它的特点：

- **无状态**：不像 VMess 需要维护连接状态
- **支持多种传输层**：TCP、WebSocket、gRPC、HTTP/2、XHTTP
- **可搭配 TLS**：让流量看起来像普通 HTTPS
- **REALITY 模式**：更强的伪装能力

VLESS 的身份认证只靠 **UUID**，不需要 alterId（已废弃）。

### 4.2 VLESS + TLS + WebSocket 结构

```
┌─────────────────────────────────────────────────────────────┐
│                     你的电脑 (Mihomo 客户端)                  │
│                                                             │
│  浏览器/curl                                                 │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────────────────────────────┐           │
│  │  Mihomo (Clash)                              │           │
│  │   协议层: VLESS                               │           │
│  │   传输层: WebSocket                           │           │
│  │   TLS层:  ✓  (证书: mylittlesmallproxy.com)   │           │
│  └─────────────────────────────────────────────┘           │
│      │                                                       │
│      │  TCP 443, SNI: mylittlesmallproxy.com                │
└──────│─────────────────────────────────────────────────────│
       │                    加密隧道
       ▼
┌─────────────────────────────────────────────────────────────┐
│                 海外服务器 (Xray 服务端)                       │
│                                                             │
│  ┌─────────────────────────────────────────────┐           │
│  │  Xray                                      │           │
│  │    TLS 解密                                 │           │
│  │    WebSocket 解封装                         │           │
│  │    VLESS 协议解析 → 拿到真实目标 google.com  │           │
│  └─────────────────────────────────────────────┘           │
│      │                                                       │
│      │  自由出口 (freedom) → 访问 google.com                │
│      ▼                                                       │
│  Google 服务器                                               │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 什么是 WebSocket？

WebSocket 是一种**在 HTTP 协议基础上建立双向通信**的技术。

普通 HTTP（请求-响应）：
```
客户端 ──── GET / ────► 服务器
          ◄─── 200 OK ──
```

WebSocket（长连接，双向）：
```
客户端 ──── GET /proxy (Upgrade: websocket) ────► 服务器
          ◄─── 101 Switching Protocols ──
          
之后：客户端和服务器可以随时互相发消息
      ────── 数据帧 ──────►
      ◄───── 数据帧 ───────
```

**为什么代理要用 WebSocket？**
- WebSocket 的流量看起来像普通 HTTP 流量
- 可以通过标准 443 端口（TLS）传输
- 墙的深度包检测（DPI）看到的是 HTTP，不是代理协议

---

## 五、REALITY——当前最优方案

### 5.1 REALITY 的核心思想

REALITY 是 Xray 开发的一种**TLS 流量伪装技术**，它的设计目标：

> **让代理流量和访问真实网站的流量完全无法区分**

传统 TLS 代理的问题：
```
代理流量：  SNI = myserver.com,   证书 = myserver.com
真实网站：  SNI = microsoft.com,  证书 = microsoft.com
```
墙一看就知道哪个是代理。

REALITY 的解决方案：
```
代理流量：  SNI = microsoft.com,  证书 = microsoft.com (真实)
           HTTP层夹带：google.com
```

GFW 看到的是：你正在访问微软官网（完全正常）。但 Xray 在 TLS 解密后，读取 HTTP 层中的真实目标，进行代理。

### 5.2 REALITY 的握手过程

```
Step 1:  客户端 ClientHello
         SNI = www.microsoft.com  ← 看起来像在访问微软
         (tls.PublicKey 加密的 Xray 协议头)
         
Step 2:  Xray 收到 ClientHello
         用私钥解密协议头 → 拿到 UUID、真实目标
         
Step 3:  Xray 动态从 www.microsoft.com 获取真实证书
         把这个证书"镜像"给客户端
         
Step 4:  客户端以为在和微软 TLS 握手，实际上和 Xray 在握手
         (证书链完全一致，浏览器无法区分)

Step 5:  加密通道建立后
         HTTP 层: GET /  (伪装)
         Xray 看到真实目标: google.com → 代理访问
```

### 5.3 client-fingerprint——模拟浏览器 TLS 指纹

每个浏览器的 TLS ClientHello 内容不同：
- 支持的加密套件列表
- TLS 版本
- 扩展列表顺序
- ALPN（应用层协议协商）

墙会检测 ClientHello 指纹，识别出是代理工具还是真实浏览器。

`client-fingerprint: chrome` 让 Mihomo **模拟 Chrome 浏览器的 TLS 指纹**：
```
Mihomo ClientHello:
  TLS 1.3
  Cipher suites: [TLS_AES_128_GCM_SHA256, ...]
  Extensions: [server_name, application_layer_protocol_negotiation, ...]
  ← 这和 Chrome 浏览器发的一模一样
```

---

## 六、调式方法——逐层排查

### 6.1 排查顺序

```
① TLS 层 (OpenSSL 测试)      ← 最底层，先确认
② HTTP/应用层 (Curl 测试)    ← TLS 通了再看这个
③ 客户端日志 (Mihomo)         ← 应用是否正确发包
④ 服务端日志 (Xray)           ← 服务端是否收到
```

### 6.2 OpenSSL 测试 TLS

```bash
openssl s_client -connect mylittlesmallproxy.abrdns.com:8443 \
  -servername www.microsoft.com
```

结果解读：
- `CONNECTED` → TCP 层面通了
- `Verification: OK` → 证书链有效
- `DONE` → TLS 握手完成，连接可用
- `Connection refused` → 网络/防火墙问题

### 6.3 Curl 测试 WebSocket 升级

```bash
curl -vk --max-time 5 --http1.1 \
  -H "Upgrade: websocket" \
  -H "Connection: Upgrade" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  https://mylittlesmallproxy.abrdns.com:8443/proxy
```

- `HTTP/1.1 101 Switching Protocols` → WebSocket 升级成功
- `HTTP/1.1 404` → 服务端没有这个路径的配置
- `HTTP/2 400` → 客户端用了 HTTP/2，但服务端 WebSocket 需要 HTTP/1.1

### 6.4 HTTP/2 vs HTTP/1.1 的坑

ALPN（应用层协议协商）决定了 TLS 握手时协商用哪个协议：

```
ClientHello:
  ALPN: ["h2", "http/1.1"]   ← 客户端支持 HTTP/2 和 1.1
  
ServerHello:
  ALPN: h2                    ← 协商成 HTTP/2 了
  
问题: WebSocket 只兼容 HTTP/1.1 !
```

所以在 Xray 的 WebSocket 配置中必须指定：
```json
"tlsSettings": {
  "alpn": ["http/1.1"]   // 强制用 HTTP/1.1，否则 WS 升级失败
}
```

---

## 七、常见坑与解决方案

| 问题现象 | 根因 | 解决 |
|---------|------|------|
| TLS 通，但应用层 404 | path 不匹配（`/proxy` vs `/`） | 服务端添加对应 path |
| `TLS handshake error: EOF` | TLS 层失败（证书、指纹） | 检查 client-fingerprint |
| Xray: "Legacy XTLS removed" | Xray 26.x 移除了 XTLS | 换用 REALITY |
| `tls: first record does not look like TLS` | SS+TLS 层顺序冲突 | 用 REALITY 或明文 SS |
| GFW 封锁代理 IP | 流量特征太明显 | 换 IP + 换端口 + 换协议 |
| 连接超时 | 防火墙/网络不通 | 逐跳排查 from 本地 to 目标 |

---

## 八、完整的 REALITY 代理链

```
┌──────────────────────────────────────────────────────────────┐
│ 国内机器                                                      │
│ 浏览器 → Mihomo → TLS (SNI=microsoft.com) → ...              │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼  公网
┌──────────────────────────────────────────────────────────────┐
│ 海外服务器 (Xray)                                             │
│ 接收 TLS → 证书伪装成 microsoft.com → 解密 VLESS 协议头        │
│ → 读取真实目标 (google.com) → freedom 出口访问                │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼
                    Google / 任意网站
```

**配置要点（服务端 Xray）**：
- `protocol: vless`
- `security: reality`
- `dest: www.microsoft.com:443`（伪装目标）
- `privateKey`: Xray x25519 生成

**配置要点（客户端 Mihomo）**：
- `type: vless`
- `network: tcp`
- `tls: true`
- `client-fingerprint: chrome`（必须）
- `servername: www.microsoft.com`（与服务端一致）
- `reality-opts.public-key`: Xray x25519 输出的 Password

---

## 九、附录：Mihomo/Xray 关键配置对照

| 功能 | Mihomo 字段 | Xray 字段 |
|------|------------|----------|
| 协议类型 | `type: vless` | `protocol: vless` |
| 传输方式 | `network: ws` | `network: ws` |
| TLS | `tls: true` | `security: tls` |
| WebSocket 路径 | `ws-path: /proxy` | `wsSettings.path: /proxy` |
| XHTTP 路径 | `xhttp-uri: /proxy` | `xhttpSettings.path: /proxy` |
| 伪装域名 | `servername: xxx` | `serverNames: [xxx]` |
| REALITY 公钥 | `reality-opts.public-key` | `x25519 Password(PublicKey)` |
| 浏览器指纹 | `client-fingerprint: chrome` | （服务端无此字段）|

---

## 十、参考阅读

- [Xray 官方文档](https://xtls.github.io/)
- [Mihomo (MetaCubeX) GitHub](https://github.com/MetaCubeX/mihomo)
- [REALITY 协议说明](https://github.com/XTLS/REALITY)
- [TLS 1.3 RFC 8446](https://www.rfc-editor.org/rfc/rfc8446)
- [WebSocket RFC 6455](https://www.rfc-editor.org/rfc/rfc/rfc6455)

---

*本文档基于真实调试经验编写，涵盖 VLESS+WebSocket、VLESS+XHTTP、VLESS+REALITY、Shadowsocks+TLS 等多种协议组合的部署与调式方法。*
