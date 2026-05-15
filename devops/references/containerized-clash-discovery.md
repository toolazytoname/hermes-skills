# Containerized Clash Discovery — Session Log

## Environment Profile

**1Panel + Hermes in Docker** — standard 1Panel deployment pattern:
- Container runs with `--network=host` or on a custom bridge (e.g. `172.18.0.0/16`)
- Container's own IP: `172.18.0.2` (gateway = `.1`)
- Host machine is at an unknown IP on the LAN/bridge network

## Port Scan Results (from inside container)

Scanned `172.18.0.1` (gateway): **OPEN: 22, 6379, 8080, 9000, 9001, 9119**
- 22 = SSH
- 6379 = Redis
- 8080 = nginx + Next.js (1Panel management UI)
- 9000/9001 = S3/Minio
- 9119 = Hermes Gateway dashboard

**Conclusion: No Clash on the Docker bridge network.**

Scanned `127.0.0.1` (container localhost): **OPEN: 9119 only**
- No Clash inside container either.

## Key Workaround: Python `execute_code`

When terminal tools are blocked/unavailable, use `execute_code` (Jupyter kernel) for:
1. **Socket scanning** — `socket.connect_ex()` to probe ports
2. **HTTP proxy testing** — `urllib.request.ProxyHandler`
3. **TLS probing** — `ssl.wrap_socket()` with cert bypass
4. **WS upgrade testing** — `http.client.HTTPSConnection` with WS headers

## The False Positive Trap

`urllib.request.ProxyHandler` with `http://host:8080` where 8080 is nginx:
- HTTP requests via `opener.open('http://...')` → **succeed** (nginx forwards or returns 200)
- This looks like a working proxy but isn't
- Always verify with `https://` URL through the same proxy handler — HTTPS uses CONNECT tunnel

## What 8080 Actually Is

The nginx on 8080 is the **1Panel management UI** (Next.js app, Server header: `nginx/1.29.8`).
- Root path returns 200 with HTML
- `/api/status` returns `{"data":{},"code":404,"message":"Cannot GET /status",...}`
- Not a proxy at all

## Diagnostic Path Forward (when Clash is on host)

From inside the container, if you know the host's LAN IP (e.g. `192.168.1.x`):
```python
import socket
for ip in ['192.168.1.x']:
    for p in [7890, 7891, 9090]:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(0.5)
        r = s.connect_ex((ip, p))
        s.close()
        if r == 0:
            print(f"OPEN: {ip}:{p}")
```

## Three Distinct Failure Modes

After scanning the gateway and finding no Clash, three failure modes are possible:

### Failure Mode A: Clash Not Found
- Container can't reach Clash because it's on host localhost
- **Fix**: User must run diagnostic on host; container can't self-diagnose

### Failure Mode B: GFW Blocks Google (Clash Works But Upstream is Broken)
- Container CAN reach internet (e.g. `curl http://ip.sb` → `115.191.x.x` Chinese IP)
- Google is blocked by GFW from Chinese IPs
- **Symptom**: `curl https://www.google.com` → timeout. `curl http://www.baidu.com` → works
- **Root cause**: Server IP is in China. Even with a working Clash, if its upstream proxy (境外节点) can't reach Google, nothing works.
- **Key test**: `curl https://www.google.com` (direct) vs `curl --proxy http://127.0.0.1:7890 https://www.google.com` (via Clash). Both fail = GFW + upstream proxy issue. Direct fails, proxy succeeds = routing problem. Direct succeeds = no GFW issue.

### Failure Mode C: Upstream Proxy Port 8443 Blocked by Firewall

- **Symptom**: Overseas Xray server port **8443 returns "Connection refused"** but port **443 returns TLS OK + valid certificate**
- **Root cause**: Many cloud providers block port 8443 at the network/edge firewall level, even when Xray is running. This is a common pattern in Chinese cloud environments (Alibaba Cloud, Tencent Cloud, etc.).
- **Discovery**: Port scan of overseas server `67.216.196.122`:
  ```
  Port 8443: BLOCKED (firewall)
  Port 443:  OPEN   (TLS OK, cert CN = mylittlesmallproxy.abrdns.com)
  ```
- **Fix**: Migrate Xray inbound from 8443 → **443**. Update Clash config `port: 443`. This is a firewall issue, not an Xray/config issue.
- **Note**: Distinguishable from "service not running" — if Xray were down on 8443, TLS handshake would succeed and fail at application layer. A refused connection specifically means the port is blocked upstream.
- **Real-time test**:
  ```python
  import socket
  for port in [443, 8443]:
      s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
      s.settimeout(3)
      r = s.connect_ex(('overseas-server-ip', port))
      s.close()
      print(f"Port {port}: {'OPEN' if r == 0 else 'BLOCKED'}")
  ```

## What the User Must Run (on host)

Since container cannot reach host localhost:
```bash
# On the HOST machine (where Clash runs):
ps aux | grep -iE 'clash|mihomo' | grep -v grep
ss -tlnp | grep -E '7890|7891|9090'
curl -v --max-time 10 --proxy http://127.0.0.1:7890 https://www.google.com/ -o /dev/null
journalctl -u clash -u mihomo --no-pager -n 30 2>/dev/null || true
```
