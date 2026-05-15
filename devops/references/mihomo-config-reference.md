# Mihomo (MetaCubeX) VLESS+WS Configuration Reference

## Complete VLESS+WS Proxy Config

```yaml
proxies:
- name: secure-proxy
  type: vless
  server: mylittlesmallproxy.abrdns.com
  port: 8443
  uuid: 3a7c2e91-5608-4d2b-a89f-71c35d829c68
  alterId: 0
  network: ws              # must be "ws" not "websocket"
  tls: true
  servername: mylittlesmallproxy.abrdns.com
  ws-path: /proxy
  http3: false            # disable HTTP/3 (explicit)
  http2: false            # disable HTTP/2 (may not work in all versions)
  ws-headers:
    Host: mylittlesmallproxy.abrdns.com
```

## Key Fields

| Field | Required | Notes |
|-------|----------|-------|
| `type` | Yes | Must be `vless` |
| `server` | Yes | Domain or IP |
| `port` | Yes | 443 for XHTTP, 8443 for WS |
| `uuid` | Yes | Must match Xray server config |
| `alterId` | Yes | Set to 0 for VLESS |
| `network` | Yes | `ws` for WebSocket, `xhttp` for XHTTP |
| `tls` | Yes | `true` for TLS, `false` for plain |
| `servername` | For TLS | SNI value (usually same as server) |
| `ws-path` | For WS | Must match Xray's `path` |
| `http2` | No | Attempts to disable HTTP/2 (buggy in some Mihomo versions) |
| `http3` | No | Attempts to disable HTTP/3 |
| `ws-headers` | For WS | Host header must match servername |

## VLESS+XHTTP Config (Mihomo)

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

## Common Mihomo Log Errors

| Error Pattern | Cause |
|--------------|-------|
| `unexpected status: 404 Not Found` | Xray rejected WS path or HTTP/2 vs HTTP/1.1 mismatch |
| `TLS handshake error: EOF` | TLS layer failure |
| `i/o timeout` | Network connectivity or Xray not responding |
| `match Match using proxy[xxx]` | Connection routed to proxy (success) |
| `dial proxy ... connect error` | Failed to connect through proxy |

## Known Mihomo Issues

1. **`http2: false` not respected**: Mihomo v1.18.x may still send h2 ALPN even with `http2: false`. Use `http3: false` as well.

2. **XHTTP compatibility**: Mihomo's XHTTP implementation may not be fully compatible with Xray 26.x. Test WS first.

3. **Version**: Mihomo v1.18.8 (Sep 2024) is relatively old. Newer versions may have better protocol handling.
