---
date: 2026-06-04
tags:
  - linux
  - security
  - docker
  - raspberry-pi
  - ufw
  - cloudflare
---

# Pi Fan Spinning Fast + Security Audit

**Date:** June 4, 2026  
**Device:** Raspberry Pi 5  
**Uptime at time of incident:** 45 days

---

## What happened?

The Pi's fan was going absolutely nuts — full speed, non-stop. Normally it ramps up and down, but this time it just wouldn't calm down.

SSHed in and found the culprit pretty fast.

---

## Root Cause — `npm` stuck at 108% CPU

```
USER   PID  %CPU  CMD
***  391038   108  npm
```

Some `npm` process (likely `npm run dev` / Next.js dev server) had been running since **Jun 3 at 23:00** and chewed through **2 hours 43 minutes** of solid CPU time. It was orphaned — launched via a `bash -s` shell parented directly to PID 1, meaning whoever started it had already disconnected and left it running unsupervised.

Because of that one process:

- CPU temp hit **64.8°C** (Pi 5 fan starts around 50°C)
- Fan hit **8,181 RPM** 🌪️
- Load average climbed to 1.37

The `next-server (v15.2.4)` on port 3000 was started at the same time — that survived and is still correctly serving the profile site. The `npm` process was just the compiler/watcher that got stuck.

### Fix

```bash
kill -9 391038   # the npm process itself
kill 391034      # the orphaned bash parent
```

Temp dropped from 64.8°C → 57.6°C within minutes. Fan calmed down.

!!! tip "Lesson learned"
    Never run `npm run dev` in a bare shell you're about to close. Use `screen`, `tmux`, or `pm2` so it has a proper home. An orphaned dev process with no CPU cap will just spin forever.

---

## Security Audit — What else was found?

While SSH was already open, did a full sweep.

### System state before fixes

| Check | Result |
|---|---|
| Firewall | ❌ None installed |
| Docker port binding | ❌ `0.0.0.0` (all interfaces) |
| AnyDesk | ❌ Running as root on port 7071 |
| Nginx | 🟡 Default config, serving nothing |
| SSH failed logins | ✅ Zero — clean logs |
| Auth method | ✅ Public key only |
| Docker containers | ✅ Healthy |
| Disk usage | ✅ 70G / 235G (32%) |
| Memory | ✅ 1.2Gi / 7.9Gi |

---

## Risk Register

### 🔴 Critical — AnyDesk running as root, port 7071 open

```
anydesk  777  root  TCP *:7071 (LISTEN)
```

AnyDesk (remote desktop) was running as `root`, listening on all interfaces. No rate limiting, no auth wall before the daemon itself.

**Impact:** If AnyDesk had a protocol-level bug (it has had CVEs including auth bypasses), an attacker gets a root shell — no login screen needed. Pi already had three remote access methods (Cloudflare Tunnel, SSH, RPI Connect), so AnyDesk was pure unnecessary attack surface.

**Fix:** `sudo systemctl disable --now anydesk` — port 7071 closed.

---

### 🔴 High — No firewall, SSH open with no rate limiting

UFW wasn't installed. SSH on port 22 was globally open with no connection limits. Bots scan for port 22 24/7 — unlimited brute-force attempts were possible.

Auth logs were clean when checked, but that just means nobody had found it yet.

**Fix:**

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.0.0/24   # full LAN access
sudo ufw limit 22/tcp                # max 6 attempts / 30 seconds
sudo ufw enable
```

---

### 🔴 High — Docker ports exposed on all interfaces, bypassing Cloudflare

```
0.0.0.0:8080->8080/tcp   axon_app
0.0.0.0:8090->8090/tcp   axon_prod_app
```

Both app containers were reachable directly from anywhere on the network — bypassing Cloudflare WAF, rate limiting, and DDoS protection entirely.

!!! warning "Docker bypasses UFW"
    Docker injects its own iptables rules and UFW won't block Docker-exposed ports even if you add a deny rule. The only reliable fix is binding to `127.0.0.1` in the compose file itself.

**Fix** — edited both `/home/deploy/axon/docker-compose.yml` and `/home/deploy/axon-prod/docker-compose.yml`:

```yaml
# Before
ports:
  - "${APP_PORT:-3000}:${APP_PORT:-3000}"

# After
ports:
  - "127.0.0.1:${APP_PORT:-3000}:${APP_PORT:-3000}"
```

Recreated containers with `docker compose up -d`. Brief ~10s downtime. Cloudflare Tunnel still works — `cloudflared` connects from `localhost`, which `127.0.0.1` allows.

After:

```
axon_app        127.0.0.1:8080->8080/tcp   Up (healthy)
axon_prod_app   127.0.0.1:8090->8090/tcp   Up (healthy)
```

---

### 🟠 Medium — No auth on Cloudflare Tunnel apps

Anyone who knows the tunnel URL can reach the apps with zero login. The tunnel encrypts transport and hides the Pi's IP, but doesn't add authentication.

Uptime Kuma's dashboard was fully public — exposes internal hostnames, service names, and health status.

**Fix (to do in Cloudflare dashboard):**

1. Go to [one.dash.cloudflare.com](https://one.dash.cloudflare.com)
2. **Access → Applications → Add → Self-hosted**
3. Add each tunnel hostname
4. Policy: Allow, Identity → Emails → your email
5. Login method: **One-time PIN** (free, no OAuth app needed)

---

### 🟠 Medium — Cloudflare Tunnel token visible in `ps aux`

```bash
/usr/bin/cloudflared tunnel run --token eyJhIjoiNzkwMDA...
```

Token passed as a CLI argument shows up to any local user via `ps aux`. With the token, someone could connect their own cloudflared instance to your tunnel.

**Proper fix (not done yet):** Move token to `/etc/cloudflared/config.yml` with `600` permissions so it's not visible in the process list.

---

### 🟡 Low — Unnecessary services running

| Service | What it does | Risk |
|---|---|---|
| `nginx` (default) | Served a 404 on port 80, no proxying | Unnecessary open port |
| `cups` + `cups-browsed` | Print server | Has had RCE CVEs (CVE-2024-47176 chain) |
| `ModemManager` | USB modem manager | DBus-accessible, not needed |
| `triggerhappy` | Hardware hotkey daemon | Runs as root |
| `colord` | Color profile manager | DBus-accessible, not needed |

**Fix:**

```bash
sudo systemctl disable --now nginx cups cups-browsed ModemManager triggerhappy triggerhappy.socket colord
```

---

### 🟡 Low — Runaway process, no supervision

108% CPU for 2h43m unsupervised. Not a security issue but causes thermal stress and degrades production app performance. Pi 5 throttles at ~80°C; we were at 64.8°C — not dangerous, but continuous.

---

## Summary of all fixes applied

| Issue | Severity | Fixed |
|---|---|---|
| AnyDesk as root on port 7071 | 🔴 Critical | ✅ |
| No firewall / SSH rate limiting | 🔴 High | ✅ |
| Docker ports on all interfaces | 🔴 High | ✅ |
| No auth on Cloudflare Tunnel apps | 🟠 Medium | ⏳ Dashboard |
| Tunnel token in `ps aux` | 🟠 Medium | ❌ |
| Unnecessary services | 🟡 Low | ✅ |
| Runaway npm process | 🟡 Low | ✅ |

---

## Final open ports

| Port | Service | Accessible from |
|---|---|---|
| `127.0.0.1:8080` | axon app (Docker) | localhost only |
| `127.0.0.1:8090` | axon-prod (Docker) | localhost only |
| `127.0.0.1:20241` | cloudflared metrics | localhost only |
| `0.0.0.0:22` | SSH | anywhere, rate-limited |
| `*:3000` | Next.js profile site | LAN + Cloudflare Tunnel |
| `*:3001` | Uptime Kuma | LAN + Cloudflare Tunnel |

---

## Cloudflare free tier — remaining steps

=== "1. Zero Trust Access"
    Protect all tunnel apps with an auth wall. Free for up to 50 users.

    1. [one.dash.cloudflare.com](https://one.dash.cloudflare.com) → **Access → Applications → Add → Self-hosted**
    2. Set each tunnel hostname
    3. Policy: Allow by email, login method: One-time PIN

=== "2. SSH via browser"
    On the Pi:
    ```bash
    cloudflared access ssh-gen --hostname ssh.***.org
    ```
    Dashboard: **Access → Applications → Add → SSH**

    SSH config on any machine:
    ```
    Host raspi-remote
      HostName ssh.***.org
      User ***
      ProxyCommand cloudflared access ssh --hostname %h
    ```

=== "3. Bot Fight Mode"
    **Security → Bots → Bot Fight Mode → ON**

    Blocks known scrapers and malicious bots for free.

=== "4. Harden SSL"
    **SSL/TLS → Edge Certificates:**

    - Always Use HTTPS → ON
    - HSTS → Enable (`max-age=31536000`, include subdomains)
    - Minimum TLS Version → TLS 1.2

=== "5. Security Level"
    **Security → Settings → Security Level → Medium**

    Challenges Tor exits and known attacker IPs with a CAPTCHA.
