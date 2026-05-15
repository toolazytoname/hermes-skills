---
name: debugging-clash-mihomo-xray-proxy
description: Systematic debugging of VLESS proxy connections between Mihomo/MetaCubeX clients and Xray servers. Covers WS, XHTTP, Shadowsocks+TLS, and REALITY transport layers.
tags:
  - proxy
  - mihomo
  - xray
  - vless
  - reality
  - websocket
  - network-debugging
---

# Debugging Clash/Mihomo + Xray VLESS Proxy Issues

## Context

This skill covers systematic debugging of VLESS proxy connections between Mihomo (MetaCubeX) clients and Xray servers. **REALITY is the recommended production config** — other transports (WS, XHTTP, SS+TLS) have fatal compatibility issues with Mihomo v1.18.8 + Xray 26.3.27.

## Privacy in Deploy Scripts

When creating reusable deployment scripts for sharing (GitHub, EvoMap, team):
- **Never hardcode**: UUIDs, privateKeys, publicKeys, domain names, IP addresses
- **Use runtime generation**: `UUID=$(cat /proc/sys/kernel/random/uuid)` auto-generates per-install
- **Use placeholders**: `YOUR_DOMAIN_OR_IP`, `YOUR_UUID_HERE` that users fill in
- **EvoMap capsule content limit**: 8000 chars max; reference GitHub repos for full scripts

## Diagnostic Philosophy: Network Layer Isolation First

> **⚠️ Container environment note:** If running inside a Docker container (e.g., 1Panel Hermes), standard tools (`ps`, `ss`, `netstat`, `journalctl`, `systemctl`) are unavailable. Use Python socket scanning to discover services. See `references/containerized-clash-discovery.md`.

### Step 0: Match User's Mode

The user has two distinct modes:
- **DEBUG-THEN-FIX** (default): Present the plan, wait for outputs, explain findings, then give fix commands. User executes fixes themselves ("怕你给我改坏了").
- **DIRECT**: User says "直接改" / "你确认完方案，就直接改完试试吧" → Proceed immediately with fix + inline backup. No confirmation round.

Example opener for DEBUG-THEN-FIX mode:
```
诊断计划：
1. openssl TLS 层测试
2. curl --http1.1 + WS header 测试
3. Mihomo/Xray 日志
4. 根因定位

开始执行 — 请把每一步输出贴回来：
```

### Step 1: Always Isolate TLS Layer First

```
Step 1: openssl s_client — does TLS itself work?
Step 2: curl --http1.1 + WS headers — does WS/XHTTP upgrade work?
Step 3: Mihomo logs — where does it break?
Step 4: Xray logs — does it receive the connection?
```

## Key Diagnostic Commands

### TLS Layer Test (OpenSSL)
```bash
openssl s_client -connect <host>:<port> -servername <domain> </dev/null 2>&1
# Look for: CONNECTED, Verification: OK, DONE
# If this fails → network/firewall issue
# If this succeeds → TLS layer OK, problem is in WS/HTTP/REALITY layer
```

### WebSocket Upgrade Test (Curl HTTP/1.1)
```bash
curl -vk --max-time 5 --http1.1 \
  -H "Host: <domain>" \
  -H "Upgrade: websocket" \
  -H "Connection: Upgrade" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  https://<domain>:<port>/<ws-path>
# Success: HTTP/1.1 101 Switching Protocols
# Failure: HTTP/2 400 (client sent h2 request to WS endpoint)
# Failure: HTTP/1.1 404 (path mismatch)
```

### Mihomo Log Reading
```bash
journalctl -u clash --no-pager -n 50
# Key errors: "unexpected status: 404 Not Found", "TLS handshake error: EOF", "i/o timeout"
```

### Xray Log Reading
```bash
journalctl -u xray --no-pager -n 50
# Key errors: "TLS handshake error: EOF", "tls: client offered only unsupported versions",
# "http: TLS handshake error from X: unexpected status: 404", "failed to validate path, request:/, config:/proxy"
# No logs → connection never reached Xray (network/firewall)
```

## REALITY — Recommended Production Config

> **This is the only stable production config for Mihomo v1.18.8 + Xray 26.3.27.**

### Why REALITY
- Does NOT use XUDP → no `ws-path` bypass problem (Issue 6)
- Does NOT use legacy XTLS → works with Xray 26.3.27 (Issue 7)
- Uses real TLS certificate (Microsoft/GitHub) → maximally stealthy
- Single required client-side addition: `client-fingerprint: chrome`

### Step A: Generate REALITY Keypair on Xray Server

```bash
# Run on Xray server — works even if Xray won't start
/usr/local/bin/xray x25519

# Output:
# PrivateKey: AAGqT8xhHjlyx46neXolvA-Sbleez7RCSUZkhXtO8WU
# Password (PublicKey): TFrJDSjUuR2KK829QlvJzuiKvTEi_BnAM-845awI7Uo  ← goes in Mihomo as public-key
# Hash32: <base64>
```

**Critical**: The field called `Password (PublicKey)` is what goes into Mihomo's `reality-opts.public-key`. There is NO separate "public key" field — that is the password.

### Step B: Xray Server Config (VLESS + REALITY, Xray 26.x)

```json
{
  "log": {"loglevel": "warning"},
  "inbounds": [{
    "listen": "0.0.0.0",
    "port": 8443,
    "protocol": "vless",
    "settings": {
      "clients": [{"id": "<uuid>", "flow": ""}],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "tcp",
      "security": "reality",
      "realitySettings": {
        "show": false,
        "dest": "www.microsoft.com:443",
        "serverNames": ["www.microsoft.com"],
        "privateKey": "<PrivateKey from xray x25519>",
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
```

**Common Xray 26.x REALITY config errors** (all verified in this session):
- `shortId` (singular string) → **must be `shortIds` (plural array)**: `["68"]` not `"68"`
- Missing `decryption: "none"` → VLESS in 26.x requires this explicitly
- `privateKey` not a valid X25519 key → generate with `xray x25519`
- Port 8443 shows GFW warning → **warning only**, Xray still works
- Connection silently rejected, no logs → check `shortIds` is array, check `client-fingerprint`

### Step C: Mihomo Client Config (VLESS + REALITY)

```yaml
proxies:
  - name: "reality-proxy"
    type: vless
    server: mylittlesmallproxy.abrdns.com
    port: 8443
    uuid: <uuid — same as server>
    network: tcp
    tls: true
    client-fingerprint: chrome        # REQUIRED — without this Xray rejects TLS ClientHello
    servername: www.microsoft.com     # must match serverNames[0] in Xray config
    reality-opts:
      public-key: <Password from xray x25519 — NOT the private key>
      short-id: ""                   # match shortIds[0] in Xray config
    udp: false
```

**Test**:
```bash
curl -s --proxy http://127.0.0.1:7890 https://www.google.com -o /dev/null -w '%{http_code}'
# Expected: 200
```

## Known Issues

### Issue 6: Mihomo VLESS Force-Enables XUDP (ws-path ignored)
- **Symptom**: Xray logs `failed to validate path, request:/, config:/proxy` — client sends path `/` instead of configured `/proxy`
- **Root cause**: Mihomo v1.18.8 **force-enables XUDP** for VLESS regardless of config. `ws-path` is silently ignored.
- **Effect**: VLESS+WS is **functionally broken** with Mihomo v1.18.8
- **Workaround**: Use **REALITY** (recommended) or plain Shadowsocks (no XUDP involvement)

### Issue 8: XHTTP Path Mismatch (Verified 2025-05-15)
- **Symptom**: TLS handshake OK, but application layer returns 404 or times out. OpenSSL test succeeds.
- **Root cause**: Client (`xhttp-uri: /proxy`) and server (`xhttpSettings.path`) do not match. Server has no listener on the client's requested path.
- **Key diagnostic**: OpenSSL TLS succeeds → problem is NOT network/firewall, it's in the HTTP/XHTTP layer
- **Fix**: Add `"path": "/proxy"` to server-side `xhttpSettings`
- **Reference**: `references/session-2025-05-15-xhttp-path-missing.md`

### Issue 7: Xray 26.3.27 Removes Legacy XTLS
- **Symptom**: Xray fails to start with "Legacy XTLS has been removed"
- **Root cause**: Xray 26.x removed `xtls-rprx-vision` + `tls` combination
- **Fix**: Migrate to **REALITY**

### Issue 9: Shadowsocks + TLS Layer Order Conflict
- **Symptom**: Xray logs `tls: first record does not look like a TLS handshake`
- **Root cause**: Xray expects TLS-first, Mihomo sends SS data-first. Order reversed.
- **Fix**: Use **REALITY** or plain Shadowsocks without TLS on Xray side

### Issue 10: GFW Blocks Google from Chinese IPs
- **Symptom**: `curl https://www.google.com` times out; proxy also fails
- **Root cause**: Server in China, GFW blocks outbound to Google
- **Fix**: Ensure Clash has a properly configured **upstream 境外 proxy** outside China

## Containerized Environment Diagnostics

When Clash runs on the **host** but you are diagnosing from **inside a Docker container** (common with 1Panel, Coolify, etc.):

```python
import socket
# Docker gateway is typically 172.18.0.1
gateway = '172.18.0.1'
ports = [7890, 7891, 7892, 9090, 9091, 8080, 8118, 1080, 10808]
for p in ports:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(0.3)
    r = s.connect_ex((gateway, p))
    s.close()
    if r == 0:
        print(f"OPEN: {gateway}:{p}")
```

## SSH Access Pre-Verification (Critical — Always Check First)

SSH access to the hermes-ro service account can **fail silently mid-session** if the `authorized_keys` setup command did not complete on all servers. On 2025-05-15, both servers became unreachable after initial successful connections, halting the entire diagnostic session.

**ALWAYS verify SSH to BOTH servers before starting any diagnostics:**

```bash
# Verify domestic server
ssh -i /tmp/hermes_domestic -o StrictHostKeyChecking=no -o ConnectTimeout=5 hermes-ro@<DOMESTIC_IP> "echo ok"

# Verify overseas server
ssh -i /tmp/hermes_overseas -o StrictHostKeyChecking=no -o ConnectTimeout=5 hermes-ro@<OVERSEAS_IP> "echo ok"
```

If either fails → do NOT proceed. Ask user to re-run setup on the failing server.

**Why this happens**: The `tee` heredoc for `authorized_keys` can partially succeed on one server but fail silently on another, especially over unreliable connections. SSH then fails on the server where the key was not installed.

**Post-setup verification the user should run:**
```bash
# On each server — verify key is present AND permissions are correct:
sudo ls -la /home/hermes-ro/.ssh/
sudo cat /home/hermes-ro/.ssh/authorized_keys | grep hermes
sudo chmod 600 /home/hermes-ro/.ssh/authorized_keys
sudo chmod 700 /home/hermes-ro/.ssh
```

## User Communication Preference

When sending documentation to this user via Feishu, always prefer **file-based delivery**:

- Write content to `/opt/data/home/docs/<filename>.md`
- Send via Feishu with `MEDIA:/opt/data/home/docs/<filename>.md`
- User stated inline messages are hard to read ("一大段，我不方便阅读")
- Apply to: SSH guides, network primers, skill explanations, any content >5 paragraphs

## Educational Reference

The skill's `references/proxy-primer.md` contains a **popular science article** covering the full domain knowledge required to understand and debug proxy setups:

- 为什么需要代理 / GFW 检测原理
- 协议分层模型（OSI七层）+ 逐层排查思路
- TLS 握手、SNI、证书链认证
- VLESS / VMess / Shadowsocks 协议对比
- WebSocket 伪装原理（为什么 WS 让流量看起来像 HTTP）
- REALITY 核心原理（SNI 伪装成微软 + HTTP 层夹带真实目标）
- client-fingerprint 模拟浏览器 TLS 指纹
- HTTP/2 vs HTTP/1.1 ALPN 的坑
- 完整 REALITY 代理链架构图
- Mihomo ↔ Xray 字段对照表

Use this as onboarding/grounding material before deep-diving into configs, or when the user wants to understand the "why" behind each layer.

## Diagnostic Checklist
- [ ] OpenSSL TLS test to proxy host:port succeeds
- [ ] Xray logs show connection received
- [ ] Mihomo logs show `match Match using proxy`
- [ ] SSH access verified to BOTH servers before starting
- [ ] **REALITY**: `client-fingerprint: chrome` is set in Mihomo
- [ ] **REALITY**: `public-key` is the Password (PublicKey) from xray x25519 output
- [ ] **REALITY**: `shortIds` is an array, not a string
- [ ] Container env? Use Python socket scan to find proxy ports
