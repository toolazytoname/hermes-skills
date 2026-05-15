# Session 2025-05-15: Mihomo XUDP + Xray 26.3.27 + SS+TLS

## Summary

Goal: Get domestic Mihomo (115.191.29.26) → overseas Xray (67.216.196.122, mylittlesmallproxy.abrdns.com) → Google working after migrating from direct IP to TLS.

## Protocol Evolution (all failed)

| Attempt | Client Config | Server Config | Result |
|---------|-------------|---------------|--------|
| VLESS+XHTTP | network:xhttp, tls:true | xhttp+TLS | SSL_ERROR_SYSCALL |
| VLESS+WS | network:ws, ws-path:/proxy | ws+TLS | Xray 404 (path / vs /proxy) |
| VLESS+WS+HTTP/1.1 | http2:false, alpn h2 off | ws+TLS | Same 404 |
| XTLS Vision | flow:xtls-rprx-vision, tls:true | xtls+TLS | Xray exit 23 (Legacy XTLS removed) |
| SS+TLS | ss+TLS | ss+TLS | Xray: "tls: first record does not look like TLS" |
| SS plain | ss (no TLS) | ss (no TLS) | Not tested — likely would work |

## Root Causes Found

1. **XHTTP path missing** (fixed early): Xray xhttpSettings had no `path` field
2. **DNS IPv6** (fixed): Xray freedom returned IPv6, OS dropped packets
3. **Mihomo XUDP bypasses ws-path** (UNRESOLVED): v1.18.8 always xudp=true for VLESS; ws-path ignored; Xray receives `path:/`
4. **Xray 26.3.27 removes Legacy XTLS** (UNRESOLVED for XTLS path): `xtls-rprx-vision` + `tls` fails exit 23
5. **SS+TLS layer order** (UNRESOLVED): Mihomo sends raw SS bytes, Xray expects TLS-first

## Final Working Solution Path

**REALITY** — VLESS+REALITY has no XUDP involvement, path verification works, Xray 26.3.27 supports it.

Xray REALITY inbound (server):
```json
{
  "inbounds": [{
    "port": 8443,
    "protocol": "vless",
    "settings": {"clients": [{"id": "<uuid>", "alterId": 0}], "decryption": "none"},
    "streamSettings": {
      "network": "tcp",
      "security": "reality",
      "realitySettings": {
        "show": false,
        "dest": "www.microsoft.com:443",
        "xver": 0,
        "serverNames": ["www.microsoft.com"],
        "privateKey": "<privateKey>",
        "shortIds": [""]
      }
    }
  }]
}
```

Mihomo REALITY outbound/client (domestic):
```yaml
proxies:
- name: secure-proxy
  type: vless
  server: mylittlesmallproxy.abrdns.com
  port: 8443
  uuid: <uuid>
  alterId: 0
  network: tcp
  tls: true
  servername: www.microsoft.com
  reality-opts:
    publickey: <publicKey>
    shortid: ""
```

## Key Diagnostic Commands

```bash
# Check Mihomo XUDP status (critical!)
curl -s http://127.0.0.1:9090/configs?action=proxies | python3 -m json.tool

# Check Xray version
ssh overseas "xray version"

# TLS test
openssl s_client -connect mylittlesmallproxy.abrdns.com:8443 -servername www.microsoft.com

# WS path test
curl -vk --max-time 5 --http1.1 \
  -H "Upgrade: websocket" -H "Connection: Upgrade" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  https://mylittlesmallproxy.abrdns.com:8443/proxy

# Server logs
ssh overseas "journalctl -u xray --no-pager -n 30"
```

## Environment

- Domestic: 115.191.29.26, Mihomo v1.18.8, Clash config at /etc/clash/config.yaml
- Overseas: 67.216.196.122, Xray 26.3.27, config at /usr/local/etc/xray/config.json
- Domain: mylittlesmallproxy.abrdns.com → 67.216.196.122
- SSH: hermes-ro user with NOPASSWD sudo (key deployed)
