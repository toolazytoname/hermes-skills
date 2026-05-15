# VLESS+WS Config Reference (Known-Good)

Working combination confirmed May 2026:
- Xray 26.3.27 on server
- Mihomo/Clash 1.18.8 on client
- TLS + WebSocket + http/1.1 ALPN

## Server: Xray VLESS+WS (working)

Port: `8443` (not 443 — avoid port 443 to not conflict with other services)

```json
{
  "log": {"loglevel": "warning"},
  "inbounds": [{
    "port": 8443,
    "protocol": "vless",
    "settings": {
      "clients": [{"id": "<UUID>", "alterId": 0}],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "ws",
      "security": "tls",
      "tlsSettings": {
        "alpn": ["http/1.1"],
        "certificates": [{
          "certificateFile": "/etc/letsencrypt/live/<domain>/fullchain.pem",
          "keyFile": "/etc/letsencrypt/live/<domain>/privkey.pem"
        }]
      },
      "wsSettings": {
        "path": "/proxy"
      }
    }
  }],
  "outbounds": [{
    "protocol": "freedom",
    "settings": {"domainStrategy": "UseIPv4"}
  }]
}
```

Key: `alpn: ["http/1.1"]` — NOT h2. Xray will warn WS is deprecated but it works.

## Client: Mihomo VLESS+WS (working)

```yaml
proxies:
- name: secure-proxy
  type: vless
  server: <domain>
  port: 8443
  uuid: <UUID>
  alterId: 0
  network: ws
  tls: true
  servername: <domain>
  ws-path: /proxy
  http3: false
  http2: false
  ws-headers:
    Host: <domain>
```

## What does NOT work

- XHTTP + stream-one from Mihomo 1.18.8 → SSL_ERROR_SYSCALL at tunnel layer, Xray never sees the connection
- h2 ALPN for WebSocket (use http/1.1)
- Server on port 443 with XHTTP (port conflict risk)

## IPv6 DNS issue on overseas servers

When `getent hosts www.google.com` returns only IPv6 addresses but Xray is bound to IPv4:
- Add `"domainStrategy": "UseIPv4"` to the freedom outbound settings
- Or ensure `/etc/gai.conf` has `precedence ::ffff:0:0/96  100` for IPv4 preference
