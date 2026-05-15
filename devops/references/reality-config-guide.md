# VLESS + REALITY 完整部署指南

## 架构

```
国内机器 (Mihomo 客户端)
    │  出站: VLESS+REALITY TLS
    │  client-fingerprint: chrome
    ▼
域名 mylittlesmallproxy.abrdns.com:8443
    │  DNS 解析到 海外 IP
    ▼
海外机器 (Xray 服务端)
    │  入站: VLESS+REALITY
    │  用真实网站的证书做伪装 (www.microsoft.com)
    ▼
海外出口 (freedom) → 访问 Google
```

## REALITY 工作原理

REALITY 伪装成访问微软官网：
- TLS ClientHello 里 SNI = www.microsoft.com
- 证书链完全真实（Xray 动态从目标获取）
- HTTP 层夹带真实目标 (google.com)
- 防火墙看到：访问 microsoft.com，证书对 → 放行

## 密钥生成（海外服务器执行）

```bash
/usr/local/bin/xray x25519
# 输出：
# PrivateKey: <私钥>
# Password (PublicKey): <公钥>  ← 这是 Mihomo 的 public-key
```

## 海外 Xray 服务端部署脚本 (overseas_deploy.sh)

```bash
#!/bin/bash
#===============================================
# VLESS + REALITY 部署脚本 - 海外服务器 (Xray 服务端)
#===============================================

set -e

#-------- 配置区 (按需修改) --------
DOMAIN_OR_IP="YOUR_DOMAIN_OR_IP"            # 你的域名或IP
LISTEN_PORT="8443"                           # Xray 监听端口
UUID=$(cat /proc/sys/kernel/random/uuid)     # 自动生成UUID
DEST_SERVER="www.microsoft.com"              # 伪装目标域名
DEST_PORT="443"                              # 伪装目标端口
#------------------------------------

install_xray() {
    bash -c "$(curl -L https://github.com/XTLS/Xray-core/releases/latest/download/install.sh)" @ install
}

generate_keys() {
    KEY_OUTPUT=$(/usr/local/bin/xray x25519)
    PRIVATE_KEY=$(echo "$KEY_OUTPUT" | grep "PrivateKey:" | awk '{print $2}')
    PUBLIC_KEY=$(echo "$KEY_OUTPUT" | grep "Password (PublicKey):" | awk '{print $3}')
    echo "私钥: $PRIVATE_KEY"
    echo "公钥: $PUBLIC_KEY"
}

create_config() {
    cat > /usr/local/etc/xray/config.json <<EOF
{
  "log": {"loglevel": "warning"},
  "inbounds": [{
    "listen": "0.0.0.0",
    "port": ${LISTEN_PORT},
    "protocol": "vless",
    "settings": {"clients": [{"id": "${UUID}", "flow": ""}], "decryption": "none"},
    "streamSettings": {
      "network": "tcp",
      "security": "reality",
      "realitySettings": {
        "show": false,
        "dest": "${DEST_SERVER}:${DEST_PORT}",
        "serverNames": ["${DEST_SERVER}"],
        "privateKey": "${PRIVATE_KEY}",
        "shortIds": [""]
      }
    }
  }],
  "outbounds": [{
    "protocol": "freedom",
    "tag": "direct",
    "settings": {"domainStrategy": "UseIPv4"}
  }]
}
EOF
}

start_xray() {
    systemctl enable --now xray
}

main() {
    install_xray
    generate_keys
    create_config
    start_xray
    echo "========== 客户端配置信息 =========="
    echo "服务器: ${DOMAIN_OR_IP}"
    echo "端口: ${LISTEN_PORT}"
    echo "UUID: ${UUID}"
    echo "公钥 (public-key): ${PUBLIC_KEY}"
    echo "SNI: ${DEST_SERVER}"
}
main "$@"
```

## 国内 Mihomo 客户端部署脚本 (domestic_deploy.sh)

```bash
#!/bin/bash
#===============================================
# VLESS + REALITY 部署脚本 - 国内服务器 (Mihomo 客户端)
#===============================================

set -e

#-------- 配置区 (按需修改) --------
SERVER_ADDR="YOUR_OVERSEAS_DOMAIN_OR_IP"     # 海外服务器地址
SERVER_PORT="8443"                            # 海外服务器端口
UUID="YOUR_UUID_HERE"                        # 海外生成后填入
PUBLIC_KEY="YOUR_PUBLIC_KEY_HERE"            # xray x25519 输出的 Password
SNI="www.microsoft.com"
SHORT_ID=""
CLASH_PORT="7890"
#------------------------------------

install_mihomo() {
    ARCH=$(uname -m)
    case "$ARCH" in
        x86_64)  FILE="mihomo-linux-amd64-v1.18.8.gz" ;;
        aarch64) FILE="mihomo-linux-arm64-v1.18.8.gz" ;;
    esac
    curl -L "https://github.com/MetaCubeX/mihomo/releases/download/v1.18.8/${FILE}" -o /tmp/mihomo.gz
    gunzip -f /tmp/mihomo.gz && mv /tmp/mihomo /usr/local/bin/mihomo
    chmod +x /usr/local/bin/mihomo
    [[ ! -f /usr/local/bin/clash ]] && ln -s /usr/local/bin/mihomo /usr/local/bin/clash
}

create_config() {
    cat > /etc/mihomo/config.yaml <<EOF
port: ${CLASH_PORT}
socks-port: ${CLASH_PORT}
mixed-port: ${CLASH_PORT}
allow-lan: true
mode: rule
log-level: info

proxies:
  - name: "reality-proxy"
    type: vless
    server: ${SERVER_ADDR}
    port: ${SERVER_PORT}
    uuid: ${UUID}
    network: tcp
    tls: true
    client-fingerprint: chrome
    servername: ${SNI}
    reality-opts:
      public-key: ${PUBLIC_KEY}
      short-id: "${SHORT_ID}"
    udp: false

proxy-groups:
  - name: "proxy"
    type: select
    proxies:
      - reality-proxy

rules:
  - GEOIP,CN,DIRECT
  - MATCH,proxy
EOF
}

start_mihomo() {
    systemctl enable --now mihomo
}

main() {
    install_mihomo
    create_config
    start_mihomo
}
main "$@"
```

## 关键参数对照

| 参数 | 位置 | 说明 |
|------|------|------|
| `uuid` | 双方一致 | 身份标识，任意 UUID |
| `privateKey` | Xray | Xray `x25519` 生成的私钥 |
| `public-key` | Mihomo | Xray `x25519` 输出的 **Password (PublicKey)** |
| `serverNames` | Xray | 伪装网站域名列表 |
| `servername` | Mihomo | 必须匹配 serverNames 中的一个 |
| `dest` | Xray | 伪装目标，Xray 动态获取其证书 |
| `shortId` | Mihomo | Xray `shortIds` 数组的第一个元素 |
| `client-fingerprint` | Mihomo | **必须有**，chrome/ safari/ firefox |

## 生成 UUID

```bash
cat /proc/sys/kernel/random/uuid
```

## 验证

```bash
# 国内机器测试
curl -s --proxy http://127.0.0.1:7890 https://www.google.com -o /dev/null -w '%{http_code}'
# 输出 200 即成功
```

## 常见坑

1. **`shortId` vs `shortIds`** — Xray 26 要求 `shortIds` 数组格式
2. **`decryption: "none"`** — VLESS 需显式声明不加密
3. **`client-fingerprint`** — Mihomo 默认 TLS 指纹不被 Xray REALITY 认可
4. **GeoIP 数据库** — Mihomo 需要 Country.mmdb，Xray 需要 geoip.dat；无网环境从海外服务器中转
