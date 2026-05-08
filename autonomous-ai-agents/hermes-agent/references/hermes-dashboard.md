# Hermes Agent Web Dashboard

Hermes Agent has a **FastAPI + React web UI** dashboard for managing config, API keys, and sessions.

## Quick Start

```bash
# 1. Install deps (if not already)
~/.hermes-venv/bin/pip3 install fastapi uvicorn

# 2. Build frontend
cd ~/hermes-agent-repo/web
npm install && npm run build   # outputs to ../hermes_cli/web_dist/

# 3. Start dashboard
hermes dashboard --host 127.0.0.1 --port 9119 --no-open
```

## CLI Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--port` | 9119 | Listen port |
| `--host` | 127.0.0.1 | Bind address |
| `--insecure` | off | Allow non-localhost binding (exposes API keys on network) |
| `--no-open` | — | Skip auto-open browser |
| `--tui` | off | Enable embedded Chat tab (PTY/WebSocket) |
| `--stop` | — | Kill all running dashboard processes |
| `--status` | — | Show running dashboard PIDs |

## Security

The dashboard exposes **all API keys and configuration** via the web UI.

### Safe: nginx reverse proxy (recommended)

Dashboard binds `127.0.0.1:9119`, nginx handles external access with auth.

```nginx
location /hermes/ {
    proxy_pass http://127.0.0.1:9119/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host localhost;   # ⚠️ MUST be localhost, see pitfall below
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    auth_basic "Hermes Dashboard";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

Setup basic auth:
```bash
# Install tools if needed
apt-get install -y apache2-utils

# Create user/password
htpasswd -cb /etc/nginx/.htpasswd admin <password>

# Test + reload nginx
nginx -t && nginx -s reload
```

**Pitfall: Host header must be `localhost`** — The dashboard's FastAPI has DNS rebinding protection (`_is_accepted_host` in `web_server.py`). When bound to `127.0.0.1`, it **only accepts** `localhost`, `127.0.0.1`, or `::1` as the Host header. Setting `proxy_set_header Host $host;` (the public IP) will produce `400 Invalid Host header`. Always override to `localhost` in the nginx config.

**Pitfall: WebSocket support** — For the embedded chat tab (--tui), the proxy must forward `Upgrade` and `Connection` headers. The config above includes them.

### Risky: direct public exposure
```bash
hermes dashboard --host 0.0.0.0 --insecure --no-open
```
Dashboard accessible at `http://<public-ip>:9119` with **no auth**.
Only do this behind a firewall or VPN.

## Known Issues

- **`hermes dashboard` fails with import error**: Install `fastapi` + `uvicorn` in the Hermes venv (`~/.hermes-venv/bin/pip3 install fastapi uvicorn`). System-wide pip won't work due to PEP 668.
- **No `web_dist/` directory**: Build frontend with `cd web && npm install && npm run build`. The dashboard auto-builds if `HERMES_WEB_DIST` env is not set, but the build may fail silently if npm deps are missing. Pre-build manually for reliability.
- **Port in use**: Change with `--port <custom>`.
