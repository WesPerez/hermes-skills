---
name: remote-desktop-setup
description: "Set up a remote desktop (Xfce4 + VNC + NoVNC) on a headless server with nginx reverse proxy and Basic Auth. Used for browser-automation workflows where a human must pre-authenticate (log in) to websites."
version: 1.0.0
author: Hermes Agent
tags: [devops, remote-desktop, vnc, novnc, xfce4, headless-server]
---

# Remote Desktop Setup (Xfce4 + VNC + NoVNC + nginx)

Set up a lightweight remote desktop on a headless Linux server so a human can
interact with a GUI (browser, IDE, etc.) while the AI agent reuses that session
via CDP or other automation tools.

## Typical Use Cases

- **Browser pre-authentication**: User logs into DeepSeek/ChatGPT/etc. via GUI,
  agent reuses cookies via CDP
- **Visual debugging**: Inspect browser behavior, layout issues, or screenshot
  results
- **Manual review**: User verifies agent's work via a shared desktop

## Full Architecture (with Browser CDP)

```
┌─────────────────────────────────────────────────┐
│  Server                                           │
│                                                   │
│  ┌──────────┐    ┌──────────┐    ┌────────────┐  │
│  │  Edge     │────│  nginx   │◄───│  Browser:  │  │
│  │  (GUI)   │    │  port 80 │    │  /desktop/ │  │
│  │  :1 VNC  │    └──────────┘    └────────────┘  │
│  └────┬─────┘                                     │
│       │ CDP :9222                                  │
│       ├──────────────────────┐                    │
│  ┌────┴─────┐         ┌─────┴─────┐               │
│  │ Hermes   │         │ websockify│               │
│  │ browser  │         │ NoVNC     │               │
│  │ tool     │         │ :15901    │               │
│  └──────────┘         └───────────┘               │
└─────────────────────────────────────────────────┘
```

## Architecture

```
User Browser ──► nginx :80 (/desktop/) ──► websockify :15901 ──► VNC :5901 ──► Xfce4
                                    │
                                    └── auth_basic (username/password)
```

## Quick Install

```bash
# 1. Install desktop environment
apt-get update && apt-get install -y xfce4 xfce4-goodies

# 2. Install VNC
apt-get install -y tigervnc-standalone-server tigervnc-common

# 3. Set VNC password
vncpasswd   # enter password, answer 'n' for view-only

# 4. Configure Xfce4 as VNC desktop
mkdir -p ~/.vnc
cat > ~/.vnc/xstartup << 'EOF'
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
exec startxfce4
EOF
chmod +x ~/.vnc/xstartup

# 5. Start VNC
vncserver :1 -geometry 1920x1080 -depth 24

# 6. Install NoVNC
apt-get install -y novnc python3-websockify

# 7. Start websockify (foreground test)
websockify --web /usr/share/novnc 15901 127.0.0.1:5901

# 8. Verify
curl -s http://127.0.0.1:15901/vnc.html | head -3
```

## nginx Configuration

Add this location to your nginx config (e.g. `/etc/nginx/sites-enabled/default`):

```nginx
location /desktop/ {
    proxy_pass http://127.0.0.1:15901/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    auth_basic "Remote Desktop";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

Create auth credentials:
```bash
apt-get install -y apache2-utils
htpasswd -c /etc/nginx/.htpasswd admin
nginx -t && systemctl reload nginx
```

## Edge Browser + CDP (Persistent Login Sessions)

Install Edge and set up a persistent browser profile with Chrome DevTools Protocol
so the user can pre-authenticate and the AI agent reuses those sessions.

```bash
# Install Edge
apt-get install -y microsoft-edge-dev

# Create persistent profile directory
mkdir -p /root/.edge-profile

# Start Edge with CDP on display :1 (where VNC runs)
DISPLAY=:1 /opt/microsoft/msedge-dev/microsoft-edge-dev \
  --user-data-dir=/root/.edge-profile \
  --remote-debugging-port=9222 \
  --no-first-run \
  --no-sandbox \
  --disable-features=LockProfile \
  --start-maximized \
  https://chat.deepseek.com/

# Verify CDP works
curl -s http://127.0.0.1:9222/json/version
```

### Configure Hermes to Use CDP Browser

```bash
hermes config set browser.cdp_url "ws://127.0.0.1:9222"
# New session required: /reset
```

### Systemd Service for Edge

```ini
# /etc/systemd/system/edge-browser.service
[Unit]
Description=Persistent Edge Browser with CDP Debugging
After=network.target

[Service]
Type=simple
User=root
Environment=DISPLAY=:1
ExecStart=/opt/microsoft/msedge-dev/microsoft-edge-dev \
  --user-data-dir=/root/.edge-profile \
  --remote-debugging-port=9222 \
  --no-first-run \
  --no-sandbox \
  --disable-features=LockProfile \
  --start-maximized \
  https://chat.deepseek.com/
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

```bash
systemctl daemon-reload
systemctl enable --now edge-browser.service
```

### Usage Flow

1. User opens `http://<server>/desktop/` → sees Xfce4 desktop with Edge
2. Logs into target website (DeepSeek, ChatGPT, etc.) in Edge
3. Hermes connects via CDP → inherits all cookies and sessions

## Auto-Connect NoVNC

Create an index.html that skips the NoVNC connection dialog:

Create an index.html that skips the NoVNC connection dialog:

```bash
cat > /usr/share/novnc/index.html << 'HTMLEOF'
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>Remote Desktop</title></head>
<body>
<script>
  var params = new URLSearchParams({
    host: location.hostname,
    port: location.port || '80',
    path: location.pathname,
    password: 'your-vnc-password',
    autoconnect: '1',
    resize: 'scale'
  });
  window.location.href = 'vnc.html?' + params.toString();
</script>
</body>
</html>
HTMLEOF
```

## Systemd Services

### NoVNC

```ini
# /etc/systemd/system/novnc.service
[Unit]
Description=NoVNC WebSocket Proxy for VNC
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/websockify --web /usr/share/novnc 15901 127.0.0.1:5901
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl daemon-reload
systemctl enable --now novnc.service
```

## Pitfalls

- **VNC only listens on 127.0.0.1 by default** — this is correct when paired
  with NoVNC on the same machine. Do NOT expose VNC port 5901 directly.
- **Do NOT start Edge manually AND via systemd** — this creates duplicate
  browser instances competing for CDP port 9222. Either use `edge-browser.service`
  exclusively, or start manually, but not both. Always `systemctl stop edge-browser.service`
  first if debugging manually, then `systemctl start edge-browser.service` to resume.
  Use `pkill -9 -f msedge` to kill ALL Edge processes, then restart the service cleanly.
- **After pkill msedge, the current terminal may also die** if the grep pattern
  matches a parent process. Use specific PID targeting or `ps aux | grep msedge | grep -v grep`
  to check before killing.
- **sites-enabled is NOT a symlink** on some Ubuntu installs — always check
  whether `sites-available/default` and `sites-enabled/default` are the same
  file or separate copies. Edit the one nginx actually reads.
  Check with: `ls -la /etc/nginx/sites-enabled/` — if files are regular (not
  symlinks), edit `sites-enabled/default` directly.
- **sites-enabled is NOT a symlink** on some Ubuntu installs — always check
  whether `sites-available/default` and `sites-enabled/default` are the same
  file or separate copies. Edit the one nginx actually reads.
- **WebSocket connections need Upgrade headers** — the `proxy_set_header Upgrade`
  and `Connection` lines in the nginx location are mandatory for NoVNC.
- **Multiple X servers** can conflict. Choose a free display number (`:1`, `:2`).
  Check with `ls /tmp/.X*`.
- **Memory**: Xfce4 uses ~200-400MB RAM. Pair it with a swap file if the
  server is memory-constrained.
- **No input devices**: If the VNC desktop has no mouse cursor interaction,
  check that `startxfce4` ran without errors in `~/.vnc/localhost.localdomain:1.log`.
- **VNC + Edge die on reboot**: `vncserver :1` must be restarted after reboot
  (not systemd-managed by default). Create a `vncserver@.service` or start
  manually. Edge depends on `DISPLAY=:1` being available — if VNC is down,
  Edge won't render its GUI window (though CDP may still work).

## Related Skills

- `hermes-agent` — has `references/browser-cdp-setup.md` for pairing this
  remote desktop with Hermes browser CDP integration
- `hermes-agent` — dashboard deployment reference for nginx proxy patterns
