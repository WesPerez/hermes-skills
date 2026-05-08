# Persistent Browser CDP Setup for Hermes

Full walkthrough for setting up a persistent browser with remote desktop access,
where a user can pre-authenticate (log into websites) and Hermes reuses those sessions
via CDP (Chrome DevTools Protocol).

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Server 43.159.168.34                           │
│                                                  │
│  ┌──────────┐    ┌──────────┐    ┌────────────┐ │
│  │  Edge     │────│  nginx   │◄───│  Browser:  │ │
│  │  (GUI)    │    │  port 80 │    │  /desktop/ │ │
│  │  :1 VNC   │    └──────────┘    └────────────┘ │
│  └────┬─────┘                                    │
│       │ CDP :9222                                 │
│       ├─────────────────────┐                     │
│  ┌────┴─────┐         ┌────┴─────┐                │
│  │ Hermes   │         │ websockify│               │
│  │ browser  │         │ NoVNC     │               │
│  │ tool     │         │ :15901    │               │
│  └──────────┘         └──────────┘                │
└─────────────────────────────────────────────────┘
```

## Prerequisites

- Ubuntu/Debian server (this guide uses Ubuntu 24.04)
- Root access
- nginx installed

## Step 1: Install Xfce4 Desktop

```bash
apt-get update
apt-get install -y xfce4 xfce4-goodies
```

## Step 2: Install & Configure VNC

```bash
apt-get install -y tigervnc-standalone-server tigervnc-common

# Set VNC password
vncpasswd
# Enter password (e.g. hermes123), answer 'n' for view-only

# Create xstartup for Xfce4
mkdir -p ~/.vnc
cat > ~/.vnc/xstartup << 'EOF'
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
exec startxfce4
EOF
chmod +x ~/.vnc/xstartup

# Start VNC server on display :1
vncserver :1 -geometry 1920x1080 -depth 24
```

## Step 3: Install NoVNC

```bash
apt-get install -y novnc python3-websockify

# Start websockify (bridges WebSocket → VNC)
websockify --web /usr/share/novnc 15901 127.0.0.1:5901
```

Verify: `curl -s http://127.0.0.1:15901/vnc.html | head -5` should return HTML.

## Step 4: Install Microsoft Edge

```bash
# Edge Dev (more stable for --remote-debugging-port)
apt-get install -y microsoft-edge-dev

# Or Edge Stable
apt-get install -y microsoft-edge-stable
```

## Step 5: Start Edge with CDP

```bash
mkdir -p /root/.edge-profile

/opt/microsoft/msedge-dev/microsoft-edge-dev \
  --user-data-dir=/root/.edge-profile \
  --remote-debugging-port=9222 \
  --no-first-run \
  --no-sandbox \
  --disable-features=LockProfile \
  --start-maximized \
  https://chat.deepseek.com/
```

Verify CDP:
```bash
curl -s http://127.0.0.1:9222/json/version
# → should return JSON with webSocketDebuggerUrl
```

## Step 6: Configure nginx Proxy

Add to `/etc/nginx/sites-enabled/default`:

```nginx
# NoVNC Remote Desktop
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

The WebSocket upgrade headers (`Upgrade` and `Connection`) are essential for
NoVNC to work through the proxy.

Create `.htpasswd`:
```bash
apt-get install -y apache2-utils
htpasswd -c /etc/nginx/.htpasswd admin
# Enter password
```

Test and reload:
```bash
nginx -t && systemctl reload nginx
```

## Step 7: Configure Hermes

```bash
hermes config set browser.cdp_url "ws://127.0.0.1:9222"
```

Restart Hermes or start a new session for the change to take effect.

## Step 8: Systemd Services (Auto-start at Boot)

### Edge Browser

Create `/etc/systemd/system/edge-browser.service`:
```ini
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

### NoVNC

Create `/etc/systemd/system/novnc.service`:
```ini
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

Enable and start:
```bash
systemctl daemon-reload
systemctl enable edge-browser.service novnc.service
systemctl start edge-browser.service novnc.service
```

Note: VNC server (`vncserver :1`) must still be started manually after reboot,
or added as a third service.

## Usage Flow

1. User opens `http://<server>/desktop/` in their browser
2. Enters Basic Auth credentials
3. Sees Xfce4 desktop with Edge already open
4. Logs into target website (DeepSeek, ChatGPT, etc.) in Edge
5. Hermes can now use the browser tool — it connects via CDP to Edge,
   inheriting all cookies and login sessions

## NoVNC Auto-Connect

Create `/usr/share/novnc/index.html` with auto-connect parameters:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Remote Desktop</title>
</head>
<body>
  <script>
    window.location.href = "vnc.html?" +
      "host=<server-ip>&" +
      "port=80&" +
      "path=/desktop/&" +
      "password=<vnc-password>&" +
      "autoconnect=1&" +
      "resize=scale";
  </script>
</body>
</html>
```

## Pitfalls

- **sites-enabled vs sites-available**: Ubuntu's nginx may use a regular file
  (not symlink) at `sites-enabled/default`. Always edit the correct file.
- **WebSocket 404**: Ensure nginx proxy has `proxy_set_header Upgrade $http_upgrade`
  and `proxy_set_header Connection "upgrade"` in the WebSocket location.
- **Edge won't start (port taken)**: Kill old Edge instance first:
  `pkill -f microsoft-edge`
- **Playwright path resolution**: Hermes browser tool may auto-launch its own
  Chromium if `cdp_url` is not set or unreachable. Verify with:
  `curl -s http://127.0.0.1:9222/json`
- **Profile lock**: Edge/Chrome locks its user-data-dir. Multiple instances
  can't share the same profile. Use `--disable-features=LockProfile` or
  separate directories.
- **Display variable**: Edge needs `DISPLAY=:1` (or whatever display the VNC
  server is on) to show GUI windows.
- **CORS for WebSocket**: NoVNC vnc.html loaded from a different origin than
  the WebSocket endpoint may fail. Using nginx proxy for both the HTML and the
  WebSocket avoids this.
