## 附录 C：配置模板

### C.1 最小可用配置

适合第一次跑通 Hermes CLI。

`config.yaml`

```yaml
model:
  default: "anthropic/claude-sonnet-4.6"
  provider: "openrouter"

terminal:
  backend: "local"
  cwd: "."
  timeout: 180

platform_toolsets:
  cli: [hermes-cli]

compression:
  enabled: true
  threshold: 0.50
  target_ratio: 0.20
  protect_last_n: 20

agent:
  max_turns: 60
  reasoning_effort: "medium"
```

`.env`

```bash
OPENROUTER_API_KEY=sk-or-v1-xxxx
```

初始化顺序：

```bash
hermes setup
hermes
hermes doctor
```

### C.2 OpenRouter + CLI 开发模板

适合本地开发、写代码、查文档、改文件。

```yaml
model:
  default: "anthropic/claude-sonnet-4.6"
  provider: "openrouter"
  base_url: "https://openrouter.ai/api/v1"

terminal:
  backend: "local"
  cwd: "."
  timeout: 240
  lifetime_seconds: 600

platform_toolsets:
  cli: [web, file, terminal, browser, skills, todo, memory, session_search, delegation, code_execution]

compression:
  enabled: true
  threshold: 0.55
  target_ratio: 0.20
  protect_last_n: 24

memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375

skills:
  creation_nudge_interval: 15

display:
  tool_progress: all
  streaming: true
  busy_input_mode: interrupt

agent:
  max_turns: 90
  reasoning_effort: "medium"
```

`.env`

```bash
OPENROUTER_API_KEY=sk-or-v1-xxxx
GITHUB_TOKEN=ghp_or_github_pat_xxxx
```

### C.3 Docker 沙箱模板

适合想保留终端能力但降低宿主机风险的场景。

```yaml
model:
  default: "anthropic/claude-sonnet-4.6"
  provider: "openrouter"

terminal:
  backend: "docker"
  cwd: "/workspace"
  timeout: 180
  lifetime_seconds: 600
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  docker_mount_cwd_to_workspace: true
  docker_run_as_host_user: true
  docker_forward_env:
    - "GITHUB_TOKEN"
    - "NPM_TOKEN"
  container_cpu: 2
  container_memory: 6144
  container_disk: 51200
  container_persistent: true

platform_toolsets:
  cli: [web, file, terminal, skills, todo, debugging]

display:
  tool_progress: all

agent:
  max_turns: 80
```

`.env`

```bash
OPENROUTER_API_KEY=sk-or-v1-xxxx
GITHUB_TOKEN=github_pat_xxxx
```

建议：

- `docker_mount_cwd_to_workspace` 只在明确需要对当前项目工作目录读写时开启。
- 默认不要把宿主机整个 home 目录挂进去。

### C.4 本地模型模板（Ollama / OpenAI-compatible）

适合在本地模型上跑 Hermes。

```yaml
model:
  default: "qwen3.5-coder:30b"
  provider: "custom"
  base_url: "http://localhost:11434/v1"
  context_length: 32768

terminal:
  backend: "local"
  cwd: "."
  timeout: 240

platform_toolsets:
  cli: [web, file, terminal, skills, todo]

providers:
  ollama-local:
    request_timeout_seconds: 300
    stale_timeout_seconds: 900

compression:
  enabled: true
  threshold: 0.45
  target_ratio: 0.20
  protect_last_n: 18
```

`.env`

```bash
OPENAI_API_KEY=ollama
OPENAI_BASE_URL=http://localhost:11434/v1
```

说明：

- `context_length` 要和本地模型真实 `num_ctx` 一致。
- 本地端点慢时，优先拉长超时，不要盲目把 `max_turns` 拉太高。

### C.5 Telegram 网关模板

适合把 Hermes 作为个人 Bot 运行在 Telegram。

```yaml
model:
  default: "anthropic/claude-sonnet-4.6"
  provider: "openrouter"

terminal:
  backend: "docker"
  cwd: "/workspace"
  timeout: 180
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  docker_mount_cwd_to_workspace: true
  container_persistent: true

platform_toolsets:
  cli: [hermes-cli]
  telegram: [hermes-telegram]

session_reset:
  mode: both
  idle_minutes: 1440
  at_hour: 4

group_sessions_per_user: true

streaming:
  enabled: false

display:
  interim_assistant_messages: true
  tool_progress: all

agent:
  max_turns: 60
  gateway_timeout: 1800
```

`.env`

```bash
OPENROUTER_API_KEY=sk-or-v1-xxxx
TELEGRAM_BOT_TOKEN=123456:ABCDEF
TELEGRAM_ALLOWED_USERS=123456789
TELEGRAM_HOME_CHANNEL=123456789
```

常用命令：

```bash
hermes gateway setup
hermes gateway run
```

### C.6 Discord / Slack 团队协作模板

适合把 Hermes 投放到团队频道。

```yaml
model:
  default: "anthropic/claude-sonnet-4.6"
  provider: "openrouter"

terminal:
  backend: "docker"
  cwd: "/workspace"
  timeout: 180
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  docker_mount_cwd_to_workspace: false
  container_persistent: true

platform_toolsets:
  cli: [hermes-cli]
  discord: [safe, web, vision, skills, todo, memory, session_search, messaging]
  slack: [safe, web, vision, skills, todo, memory, session_search, messaging]

session_reset:
  mode: both
  idle_minutes: 720
  at_hour: 4

group_sessions_per_user: true

memory:
  memory_enabled: true
  user_profile_enabled: true

display:
  interim_assistant_messages: true
  tool_progress: new

agent:
  max_turns: 50
```

`.env`

```bash
OPENROUTER_API_KEY=sk-or-v1-xxxx
DISCORD_BOT_TOKEN=xxxx
SLACK_BOT_TOKEN=xoxb-xxxx
SLACK_SIGNING_SECRET=xxxx
SLACK_APP_TOKEN=xapp-xxxx
```

说明：

- 团队频道优先使用 `safe` 风格工具组合，避免默认开放终端和文件写入。
- 需要更高权限时，再按平台单独加 `terminal` 或 `file`。

### C.7 MCP 集成模板

适合把 GitHub、Notion、filesystem 等外部能力接入 Hermes。

```yaml
model:
  default: "anthropic/claude-sonnet-4.6"
  provider: "openrouter"

platform_toolsets:
  cli: [hermes-cli, mcp-github, mcp-filesystem]

mcp_servers:
  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: ${GITHUB_TOKEN}

  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/Users/hao/Desktop/feedmob"]

  notion:
    url: "https://mcp.notion.com/mcp"
```

`.env`

```bash
OPENROUTER_API_KEY=sk-or-v1-xxxx
GITHUB_TOKEN=github_pat_xxxx
```

修改后在会话内执行：

```text
/reload-mcp
```

### C.8 只读研究模板

适合信息检索、阅读、总结，不允许终端和写文件。

```yaml
model:
  default: "anthropic/claude-sonnet-4.6"
  provider: "openrouter"

platform_toolsets:
  cli: [safe, browser, skills, session_search]
  telegram: [safe, browser, skills]
  discord: [safe, browser, skills]

display:
  tool_progress: all

agent:
  max_turns: 50
```

说明：

- `safe` 默认不包含终端与写文件能力。
- 想进一步收紧时，可把 `browser` 去掉，仅保留 `web`、`vision`、`skills`。

### C.9 Cron 自动化模板

适合日报、提醒、自动汇总、定时审查。

```yaml
model:
  default: "anthropic/claude-sonnet-4.6"
  provider: "openrouter"

terminal:
  backend: "docker"
  cwd: "/workspace"
  timeout: 180
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  container_persistent: true

platform_toolsets:
  cli: [hermes-cli]
  telegram: [hermes-telegram]

display:
  background_process_notifications: result

agent:
  max_turns: 50
  gateway_timeout: 1800
```

`.env`

```bash
OPENROUTER_API_KEY=sk-or-v1-xxxx
TELEGRAM_BOT_TOKEN=123456:ABCDEF
TELEGRAM_HOME_CHANNEL=123456789
```

常用命令：

```bash
hermes cron list
hermes cron create --name daily-brief --schedule "0 9 * * 1-5" --prompt "总结今天最重要的待办"
hermes cron status
```

### C.10 多 Profile 模板

适合把个人助手、团队机器人、研究环境分开。

```bash
hermes profile create personal
hermes profile create work
hermes profile use personal
```

`personal` 的 `config.yaml`：

```yaml
model:
  default: "anthropic/claude-sonnet-4.6"
  provider: "openrouter"

platform_toolsets:
  cli: [hermes-cli]
  telegram: [hermes-telegram]
```

`work` 的 `config.yaml`：

```yaml
model:
  default: "openai/gpt-5.5"
  provider: "openrouter"

platform_toolsets:
  cli: [web, file, terminal, skills, memory, session_search]
  slack: [safe, web, skills, messaging]
```

建议：

- 把个人 Bot、团队 Bot、开发沙箱拆成不同 Profile。
- 共享基础技能时用 `skills.external_dirs`，不要手工复制整个 `~/.hermes/skills/`。
