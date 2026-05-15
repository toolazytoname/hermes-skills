# Session: VLESS+XHTTP path missing on server side (2025-05-15)

## Topology
- **国内 (Clash/Mihomo client):** 115.191.29.26 — Mihomo v1.18.8 at `/etc/clash/clash`
- **海外 (Xray server):** 67.216.196.122 — Xray 26.3.27 at `/usr/local/bin/xray`
- **Domain:** `mylittlesmallproxy.abrdns.com`
- **Protocol:** VLESS + XHTTP + TLS, port 443

## Symptom
`curl https://www.google.com` via Clash proxy times out/fails. Direct IP connection worked before migrating to TLS.

## Root Cause
Xray server-side `xhttpSettings` was missing `"path"`. Mihomo client had `xhttp-uri: /proxy` but Xray had no listener on `/proxy`.

## Working Configs

### Xray server `/usr/local/etc/xray/config.json` (FIXED — had no `path`)
```json
{
  "log": { "loglevel": "warning" },
  "inbounds": [{
    "port": 443,
    "protocol": "vless",
    "settings": {
      "clients": [{ "id": "3a7c2e91-5608-4d2b-a89f-71c35d829c68", "alterId": 0 }],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "xhttp",
      "security": "tls",
      "tlsSettings": {
        "alpn": ["h2"],
        "certificates": [{
          "certificateFile": "/etc/letsencrypt/live/mylittlesmallproxy.abrdns.com/fullchain.pem",
          "keyFile": "/etc/letsencrypt/live/mylittlesmallproxy.abrdns.com/privkey.pem"
        }]
      },
      "xhttpSettings": {
        "mode": "stream-one",
        "path": "/proxy"
      }
    }
  }],
  "outbounds": [{ "protocol": "freedom" }]
}
```

### Mihomo client `/etc/clash/config.yaml`
```yaml
mixed-port: 7890
allow-lan: false
mode: rule
log-level: info
external-controller: 127.0.0.1:9090
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
proxy-groups:
- name: proxy
  type: select
  proxies:
  - secure-proxy
  - DIRECT
rules:
- DOMAIN-SUFFIX,github.com,proxy
- GEOIP,CN,DIRECT
- MATCH,proxy
```

## Fix Command
```bash
# On overseas server (67.216.196.122)
sudo sed -i 's/"xhttpSettings": {/"xhttpSettings": {\n          "path": "\/proxy",/' /usr/local/etc/xray/config.json
sudo systemctl restart xray
```

## Diagnostic Steps Used
1. `ssh` with ed25519 key to both servers (hermes-ro user)
2. `ps aux | grep -iE 'clash|xray'` confirmed processes running
3. `cat /etc/clash/config.yaml` + `cat /usr/local/etc/xray/config.json` revealed path mismatch
4. Confirmed via OpenSSL TLS test: TLS OK (CONNECTED), but HTTP layer broken

## Key Insight
When using XHTTP transport, `xhttpSettings.path` on the server is independent of and must match `xhttp-uri` on the client. WS has the same pattern with `wsSettings.path` vs `ws-path`. The TLS layer can succeed while the HTTP/XHTTP layer 404s if path is missing.
