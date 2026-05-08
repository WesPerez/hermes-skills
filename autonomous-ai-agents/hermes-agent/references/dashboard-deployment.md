# Hermes Dashboard 部署指南

## 快速启动

```bash
# 1. 装依赖
pip install fastapi uvicorn

# 2. 构建前端（Vite + React）
cd /path/to/hermes-agent-repo/web
npm install
npm run build    # 输出到 ../hermes_cli/web_dist/

# 3. 启动（仅本地）
hermes dashboard --host 127.0.0.1 --port 9119 --no-open
```

## 通过 nginx 反向代理对外暴露

### 核心难点：Host header 验证

FastAPI 有 DNS rebinding 防护：`_is_accepted_host()` 检查 Host header 是否匹配绑定的 hostname。

- 绑定 `127.0.0.1` → 只接受 `localhost` / `127.0.0.1` / `::1`
- 绑定 `0.0.0.0`（需 `--insecure`）→ 接受任意 Host（不安全，暴露 API key）

**解决方案**：nginx 发送 `proxy_set_header Host localhost;` 到后端。

### proxy_pass 尾部斜杠陷阱

| proxy_pass 写法 | 效果 | 示例 |
|---|---|---|
| `proxy_pass http://backend/;` （有斜杠） | **截掉**匹配的 location 前缀 | `/hermes/foo` → `/foo` |
| `proxy_pass http://backend;` （无斜杠） | **保留**完整路径 | `/assets/foo.js` → `/assets/foo.js` |

Dashboard 的 SPA 资源路径是绝对路径（`/assets/...`, `/fonts/...`, `/api/...`），所以：
- `/hermes/` → 用有斜杠（截掉前缀）
- `/assets/`, `/fonts/`, `/api/` → 用无斜杠（保留路径）

### nginx 完整配置示例

```nginx
# 需要 auth 保护的页面入口
location /hermes/ {
    proxy_pass http://127.0.0.1:9119/;
    proxy_set_header Host localhost;
    auth_basic "Hermes Dashboard";
    auth_basic_user_file /etc/nginx/.htpasswd;
}

# 静态资源（无 auth，浏览器直接请求）
location /assets/ { proxy_pass http://127.0.0.1:9119; proxy_set_header Host localhost; }
location /fonts/ { proxy_pass http://127.0.0.1:9119; proxy_set_header Host localhost; }
location /fonts-terminal/ { proxy_pass http://127.0.0.1:9119; proxy_set_header Host localhost; }
location /ds-assets/ { proxy_pass http://127.0.0.1:9119; proxy_set_header Host localhost; }
location = /favicon.ico { proxy_pass http://127.0.0.1:9119/favicon.ico; proxy_set_header Host localhost; }

# API + WebSocket（无 auth，由 Dashboard 内部校验 session token）
location /api/ {
    proxy_pass http://127.0.0.1:9119;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host localhost;
}
location /ws {
    proxy_pass http://127.0.0.1:9119;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host localhost;
}
```

## 安全建议

1. **不要用 `--insecure` + `--host 0.0.0.0` 直接暴露到公网** — Dashboard 能查看/编辑 API key
2. 用 nginx 反代 + HTTP Basic Auth 或 IP 白名单
3. Dashboard 本身只监听 127.0.0.1，不让外界直连端口 9119
4. 如果绑定了 `0.0.0.0`，FastAPI 的 Host header 防护自动关闭（`_is_accepted_host` 返回 True 任意 host）

## 故障排查

- **"Invalid Host header"** → nginx 发 `proxy_set_header Host localhost;`
- **JS/CSS 返回 HTML** → proxy_pass 尾部斜杠问题，检查是否需保留路径
- **页面空白** → 浏览器控制台看哪个资源 404，补对应的 nginx location
- **API 401** → SPA 内嵌的 session token 只在首次加载时注入，刷新页面即可
