# Hermes Agent Skills

Hermes Agent 通用技能库，存放调试经验、脚本、部署方案。

## 目录结构

```
hermes-skills/
├── devops/
│   └── debugging-clash-mihomo-xray-proxy/   # Mihomo + Xray VLESS 代理调试指南
│       ├── SKILL.md                          # 核心技能文档
│       └── references/                       # 参考资料
│           ├── reality-config-guide.md
│           ├── vless-ws-config-examples.md
│           ├── mihomo-config-reference.md
│           ├── session-2025-05-15-xudp-mihomo.md
│           ├── session-2025-05-15-xhttp-path-missing.md
│           └── ssh-troubleshoot-readonly.md
│
├── network-proxy-primer.md                   # 网络代理科普文章
└── README.md
```

## 内容说明

### debugging-clash-mihomo-xray-proxy

Mihomo (MetaCubeX/Clash) + Xray VLESS 代理的完整调试指南，涵盖：

- **REALITY**（推荐生产配置）：TLS 伪装成微软官网，流量无法被墙识别
- **VLESS + WebSocket**：HTTP/1.1 ALPN 配置，path 匹配
- **VLESS + XHTTP**：stream-one 模式，path 路由
- **调式方法**：从 TLS 层到应用层，逐层排查
- **已知坑**：XUDP、XTLS 移除、HTTP/2 vs HTTP/1.1、path 不匹配

详见 [devops/debugging-clash-mihomo-xray-proxy/SKILL.md](devops/debugging-clash-mihomo-xray-proxy/SKILL.md)

### network-proxy-primer.md

《网络代理是如何工作的：从原理到调试》，面向读者的科普文章，解释：

- 网络分层模型与逐层调式思路
- TLS 握手、SNI、证书链
- VLESS / WebSocket / REALITY 协议原理
- client-fingerprint 浏览器指纹模拟
- 完整 REALITY 代理链架构

## 部署参考

### 海外服务器（Xray 服务端）

```bash
# 生成密钥对
/usr/local/bin/xray x25519

# 配置 VLESS + REALITY
# privateKey  → 填入 xray config
# Password(PublicKey) → 填入 Mihomo public-key
```

### 国内机器（Mihomo 客户端）

```yaml
proxies:
  - name: "reality-proxy"
    type: vless
    server: <your-domain>
    port: 8443
    uuid: <uuid>
    network: tcp
    tls: true
    client-fingerprint: chrome      # 必须
    servername: www.microsoft.com  # 必须与 serverNames[0] 一致
    reality-opts:
      public-key: <Password from xray x25519>
      short-id: ""
    udp: false
```

## 关键配置对照

| 功能 | Mihomo | Xray |
|------|--------|------|
| REALITY 公钥 | `reality-opts.public-key` | `x25519 Password(PublicKey)` |
| 伪装域名 | `servername` | `serverNames` |
| 浏览器指纹 | `client-fingerprint` | — |

## 参考资料

- [Xray 官方文档](https://xtls.github.io/)
- [Mihomo GitHub](https://github.com/MetaCubeX/mihomo)
- [REALITY 协议](https://github.com/XTLS/REALITY)
- [TLS 1.3 RFC 8446](https://www.rfc-editor.org/rfc/rfc8446)
