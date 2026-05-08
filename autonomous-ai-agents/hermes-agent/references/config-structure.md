# Hermes Config Structure Reference

## File Relationship

```
~/.hermes/.env          → 密钥 (API keys, tokens, credentials)
~/.hermes/config.yaml   → 行为配置 (models, agent, tools, platforms)
```

`.env` 中的变量名被 `config.yaml` 的 `api_key_env` 字段引用。

## .env 格式

```bash
# 每行一个 KEY=VALUE，无引号
DEEPSEEK_API_KEY=sk-xxxx
ANTHROPIC_AUTH_TOKEN=db06b689-xxxx
WEIXIN_ACCOUNT_ID=d4a433f32ffb@im.bot
WEIXIN_TOKEN=d4a433f32ffb@im.bot:0600xxxx
WEIXIN_ALLOW_ALL_USERS=true
GATEWAY_ALLOW_ALL_USERS=true
```

## config.yaml 关键段落

### 模型选择（顶层）

```yaml
model:
  default: glm-5.1              # 模型名 → 必须在 custom_providers[].models 中
  provider: volcengine-coding   # 供应商名 → 必须与 custom_providers[].name 对应
```

切换模型只改这两行。确保对应的 `.env` key 存在。

### 自定义供应商（文件末尾）

```yaml
custom_providers:
- name: deepseek                     # 供应商标识（model.provider 引用此名）
  api_key_env: DEEPSEEK_API_KEY      # → .env 中的变量名
  base_url: https://api.deepseek.com # API 端点
  models:                            # 可用模型列表
  - deepseek-v4-flash
  - deepseek-v4-pro

- name: volcengine-coding
  api_key_env: ANTHROPIC_AUTH_TOKEN
  base_url: https://ark.cn-beijing.volces.com/api/coding/v1
  models:
  - glm-5.1
```

**链条：** `model.provider` → `custom_providers[].name` → `api_key_env` → `.env`

### agent（行为控制）

```yaml
agent:
  max_turns: 90                # 工具调用最大轮数
  gateway_timeout: 1800        # 单次会话超时（秒）
  reasoning_effort: ''         # 推理深度：high/medium/low/''(空=不传)
  image_input_mode: auto       # 图片输入模式
  api_max_retries: 3           # API 失败重试次数
```

**注意：** 切换到不支持 reasoning_effort 的模型（如 GLM）时，必须设为空字符串 `''`，否则报错。

### auxiliary（辅助模型）

```yaml
auxiliary:
  vision:         {provider: auto, model: ''}  # 图片识别
  web_extract:    {provider: auto, model: ''}  # 网页提取
  compression:    {provider: auto, model: ''}  # 会话压缩
  session_search: {provider: auto, model: ''}  # 历史搜索
  curator:        {provider: auto, model: ''}  # 技能整理
  # ...共 9 个辅助任务
```

`provider: auto` = 跟主模型用同一个供应商。辅助任务共享主 provider 的 key。

### compression（会话压缩）

```yaml
compression:
  enabled: true
  threshold: 0.5               # 上下文超过 50% 时触发
  target_ratio: 0.2            # 压缩到 20%
  protect_last_n: 20           # 保留最近 20 轮
```

### platforms（平台连接）

```yaml
platforms:
  weixin:
    enabled: true

# 微信相关环境变量在 .env 和 config.yaml 末尾
WEIXIN_ALLOW_ALL_USERS: true
WEIXIN_DM_POLICY: open
WEIXIN_HOME_CHANNEL: o9cq80yh...@im.wechat
```

## 常见操作

### 添加新供应商

1. `.env` 加 key
2. `config.yaml` 的 `custom_providers` 加条目
3. `model.default` + `model.provider` 指过去
4. 重启

### 删除供应商

1. `custom_providers` 删条目
2. 如果是当前默认，先切到另一个
3. `.env` 的 key 可留可删
4. 重启

### 验证当前模型

```bash
grep "default:" ~/.hermes/config.yaml           # 配置值
tail ~/.hermes/logs/agent.log | grep auto-detect # 运行时值
```

两个都要检查——配置改了但没重启，运行时还是旧模型。
