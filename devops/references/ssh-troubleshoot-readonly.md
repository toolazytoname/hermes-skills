# SSH Remote Troubleshooting Workflow

## Overview

This documents the standard workflow for using Hermes to SSH into remote Linux servers for read-only diagnostics, specifically for proxy/network troubleshooting.

## Prerequisites

- Hermes generates SSH keypairs on its server (`/tmp/hermes_<name>`)
- User receives public keys to install on target servers
- User creates a read-only service account (e.g., `hermes-ro`)
- No root password ever transmitted; key-based auth only

## Workflow

### 1. Generate Keypairs on Hermes Side

```bash
ssh-keygen -t ed25519 -C "hermes-<purpose>" -f /tmp/hermes_<name> -N ""
```

### 2. User Installs Public Keys on Target Servers

```bash
# Create user
sudo useradd -m -s /bin/bash hermes-ro

# Create SSH dir
sudo mkdir -p /home/hermes-ro/.ssh
sudo chmod 700 /home/hermes-ro/.ssh

# Add public key (国内用 domestic 公钥)
echo "ssh-ed25519 AAAAC3... hermes-domestic" | sudo tee /home/hermes-ro/.ssh/authorized_keys

# Set permissions
sudo chmod 600 /home/hermes-ro/.ssh/authorized_keys
sudo chown -R hermes-ro:hermes-ro /home/hermes-ro/.ssh

# Optional: lock read-only user files
sudo chattr +i /home/hermes-ro/.bashrc
sudo chattr +i /home/hermes-ro/.profile
```

### 3. Hermes Connects (Read-Only Diagnostics)

```bash
ssh -i /tmp/hermes_<name> -o StrictHostKeyChecking=no hermes-ro@<IP> "<diagnostic commands>"
```

### 4. Revoke Access

```bash
# Delete the public key line from authorized_keys
sudo sed -i '/hermes-<name>/d' /home/hermes-ro/.ssh/authorized_keys
```

## Diagnostic Sequence for Proxy Issues

```
1. systemctl status — verify service is running
2. Check config files — diff client vs server field names
3. openssl s_client — isolate TLS layer
4. curl with WS headers — test upgrade
5. journalctl — read service logs
6. ss -tlnp / netstat — check port listening
7. openssl x509 -in cert.pem -no_out -dates — check cert validity
```

## SSH Access — tee Partial Failure (2025-05-15)

Both servers became unreachable mid-session after initially successful connections. Root cause: `authorized_keys` setup likely partially succeeded on one server but failed silently on the other.

**Signs:**
- First SSH attempt succeeds, subsequent attempts fail with `Permission denied (publickey,password)`
- `ssh -v` shows `Authentications that can continue: publickey,password` — key offered but rejected
- No sudo prompt (connection fails before user interaction)

**Prevention**: After running `tee`, always verify:
```bash
sudo cat /home/hermes-ro/.ssh/authorized_keys | grep hermes
```
If nothing prints, the key was not installed.

**Recovery**: Re-run full setup on the failing server.

**Key insight**: `tee` with `sudo` over SSH can silently fail if the session drops mid-command, if sudo credentials expire, or if there's a TTY issue. Always verify file content after creation, not just exit code.

---

## Common Pitfalls (XHTTP/VLESS)

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| TLS OK, app fails 404 | Server `xhttpSettings` missing `path` | Add `"path": "/proxy"` on server |
| TLS OK, EOF/close | ALPN mismatch | Set `alpn: ["h2"]` server-side |
| 400 Bad Request | Mihomo sends h2 to WS endpoint | Use `http2: false` client-side |
| Certificate unreadable | Xray runs as `nobody`, certs not readable | `chmod -R 755 /etc/letsencrypt/live/` |
