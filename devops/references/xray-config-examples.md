# Xray Configuration Examples

## VLESS + WebSocket (TLS)

Port 8443, path `/proxy`, minimal working config:

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 8443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "UUID-HERE",
            "alterId": 0
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "tlsSettings": {
          "certificates": [
            {
              "certificateFile": "/etc/letsencrypt/live/example.com/fullchain.pem",
              "keyFile": "/etc/letsencrypt/live/example.com/privkey.pem"
            }
          ]
        },
        "wsSettings": {
          "path": "/proxy"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom"
    }
  ]
}
```

## VLESS + XHTTP (TLS)

> **⚠️ Critical:** `xhttpSettings` MUST include `"path"` matching the client's `xhttp-uri`. Without it, Xray accepts TLS (CONNECTED, cert OK) but 404s at the HTTP layer — the #1 failure mode when migrating direct-IP→TLS or WS→XHTTP. The server-side `path` and client-side `xhttp-uri` are independent and must both be set.

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "UUID-HERE",
            "alterId": 0
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "xhttp",
        "security": "tls",
        "tlsSettings": {
          "alpn": ["h2"],
          "certificates": [
            {
              "certificateFile": "/etc/letsencrypt/live/example.com/fullchain.pem",
              "keyFile": "/etc/letsencrypt/live/example.com/privkey.pem"
            }
          ]
        },
        "xhttpSettings": {
          "mode": "stream-one",
          "path": "/proxy"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom"
    }
  ]
}
```

### Mihomo VLESS+XHTTP Client Config

```yaml
proxies:
- name: secure-proxy
  type: vless
  server: mylittlesmallproxy.abrdns.com
  port: 443
  uuid: 3a7c2e91-5608-4d2b-a89f-71c35d829c68
  alterId: 0
  network: xhttp
  tls: true
  servername: mylittlesmallproxy.abrdns.com
  xhttp-mode: stream-one
  xhttp-uri: /proxy
  http3: false
```

## Key Xray Log Patterns

| Pattern | Meaning |
|---------|---------|
| `TLS handshake error from X: EOF` | TLS connection failed/closed |
| `tls: client offered only unsupported versions` | ALPN mismatch |
| `unexpected status: 404` | WS path mismatch or HTTP/2 sent to WS endpoint |
| `accepted tcp` | Connection successfully proxied |
| `Failed to start: ... permission denied` | Certificate file not readable |
