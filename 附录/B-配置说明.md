## 附录 B：配置说明

### B.1 配置文件与目录结构

Hermes 的用户配置默认保存在 `~/.hermes/`。如果设置了 `HERMES_HOME`，则以该环境变量指定的目录为准。

```text
~/.hermes/
├── config.yaml       # 主配置：模型、终端、工具、记忆、显示、网关等
├── .env              # 密钥：API Key、Bot Token、云沙箱凭证等
├── auth.json         # OAuth 凭证
├── SOUL.md           # Agent 身份与长期行为约束
├── memories/         # MEMORY.md、USER.md 等持久记忆
├── skills/           # 用户创建或安装的技能
├── cron/             # 定时任务数据
├── sessions/         # 会话数据
├── logs/             # agent.log、gateway.log、errors.log 等日志
├── checkpoints/      # `/rollback` 使用的影子存储
└── kanban/           # 多项目看板数据
```

### B.2 配置优先级

Hermes 解析配置时按以下顺序覆盖：

1. CLI 参数：例如 `hermes chat --provider openrouter --model anthropic/claude-sonnet-4.6`
2. `~/.hermes/config.yaml`
3. `~/.hermes/.env`
4. 内置默认值

实践规则：

- API Key、Bot Token、密码、云服务令牌放进 `.env`。
- 模型选择、终端后端、工具集、上下文压缩、显示样式、记忆限额放进 `config.yaml`。
- 临时覆盖用 CLI 参数，不要为了单次实验改全局配置。

### B.3 配置管理命令

| 命令 | 说明 |
|------|------|
| `hermes setup` | 首次或重新运行交互式配置向导 |
| `hermes setup model` | 只配置模型供应商和模型 |
| `hermes setup terminal` | 只配置终端后端 |
| `hermes setup gateway` | 只配置消息平台 |
| `hermes setup tools` | 只配置工具可用性 |
| `hermes model` | 添加供应商、输入 API Key、执行 OAuth、选择默认模型 |
| `hermes config` | 查看当前配置 |
| `hermes config edit` | 用编辑器打开 `config.yaml` |
| `hermes config set KEY VALUE` | 设置配置项；密钥会自动写入 `.env` |
| `hermes config check` | 检查升级后缺失的配置项 |
| `hermes config migrate` | 交互式补齐新增配置 |
| `hermes doctor` | 诊断安装、配置、依赖和运行环境 |

### B.4 模型配置

最小模型配置：

```yaml
model:
  default: "anthropic/claude-opus-4.6"
  provider: "auto"
```

常见字段：

| 字段 | 说明 |
|------|------|
| `model.default` | 默认模型名称 |
| `model.provider` | 供应商，常见值：`auto`、`openrouter`、`nous`、`anthropic`、`gemini`、`zai`、`kimi-coding`、`minimax`、`deepseek`、`custom` |
| `model.base_url` | OpenAI-compatible 自定义端点 |
| `model.context_length` | 总上下文窗口。通常让 Hermes 自动检测，本地模型检测不准时再手动设置 |
| `model.max_tokens` | 单次回复输出上限，不等于上下文长度 |

供应商示例：

```yaml
model:
  default: "anthropic/claude-sonnet-4.6"
  provider: "openrouter"
  base_url: "https://openrouter.ai/api/v1"
```

```bash
OPENROUTER_API_KEY=sk-or-v1-...
```

本地模型示例：

```yaml
model:
  default: "qwen3-coder:30b"
  provider: "custom"
  base_url: "http://localhost:11434/v1"
  context_length: 32768
```

### B.5 Provider 超时与路由

Provider 可配置请求超时、非流式调用 stale timeout、模型级例外：

```yaml
providers:
  ollama-local:
    request_timeout_seconds: 300
    stale_timeout_seconds: 900
  anthropic:
    request_timeout_seconds: 60
    models:
      claude-opus-4.6:
        timeout_seconds: 600
```

OpenRouter 可配置 provider routing：

```yaml
provider_routing:
  sort: "throughput"       # price | throughput | latency
  require_parameters: true
  data_collection: "deny"
  # only: ["anthropic", "google"]
  # ignore: ["deepinfra"]
```

### B.6 终端后端配置

`terminal` 决定 Agent 执行 shell 命令的位置。

```yaml
terminal:
  backend: "local"
  cwd: "."
  timeout: 180
  lifetime_seconds: 300
```

| 后端 | 配置重点 | 风险与建议 |
|------|----------|------------|
| `local` | 无需额外服务 | 权限等同当前用户，适合个人低风险任务 |
| `docker` | `docker_image`、资源限制、挂载目录、转发环境变量 | 推荐用于隔离；默认不要随意挂载整个宿主目录 |
| `ssh` | `ssh_host`、`ssh_user`、`ssh_key` | 适合远程机器和专用沙箱账号 |
| `modal` | `MODAL_TOKEN_ID`、`MODAL_TOKEN_SECRET` | 云端临时执行 |
| `daytona` | `DAYTONA_API_KEY` | 云工作区持久化 |
| `vercel_sandbox` | `VERCEL_TOKEN`、`VERCEL_PROJECT_ID` | 云端微虚拟机 |
| `singularity` | `singularity_image` | HPC 和共享集群 |

Docker 示例：

```yaml
terminal:
  backend: "docker"
  cwd: "/workspace"
  timeout: 180
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  docker_mount_cwd_to_workspace: true
  docker_run_as_host_user: true
  docker_forward_env:
    - "GITHUB_TOKEN"
  container_cpu: 1
  container_memory: 5120
  container_disk: 51200
  container_persistent: true
```

### B.7 工具与 Toolsets 配置

Hermes 使用 `platform_toolsets` 按平台控制工具可用性。

```yaml
platform_toolsets:
  cli: [hermes-cli]
  telegram: [hermes-telegram]
  discord: [hermes-discord]
  slack: [hermes-slack]
```

常用核心 toolset：

| Toolset | 主要工具 | 用途 |
|---------|----------|------|
| `web` | `web_search`、`web_extract` | 搜索与网页内容提取 |
| `file` | `read_file`、`write_file`、`patch`、`search_files` | 文件读写和补丁 |
| `terminal` | `terminal`、`process` | 命令执行和进程管理 |
| `browser` | 导航、点击、输入、截图、视觉分析 | 浏览器自动化 |
| `vision` | `vision_analyze` | 图片理解 |
| `image_gen` | `image_generate` | 图片生成 |
| `skills` | `skills_list`、`skill_view`、`skill_manage` | 技能加载和管理 |
| `memory` | `memory` | 长期记忆管理 |
| `session_search` | `session_search` | 历史会话检索 |
| `cronjob` | `cronjob` | 定时任务管理 |
| `delegation` | `delegate_task` | 子 Agent 委托 |
| `safe` | 只读研究和媒体工具 | 不包含终端和文件写入 |
| `debugging` | `file` + `terminal` + `web` | 调试组合工具 |

临时覆盖：

```bash
hermes chat --toolsets web,file,terminal
hermes chat --toolsets safe
```

交互式管理：

```bash
hermes tools
```

### B.8 上下文压缩

长会话接近模型上下文限制时，Hermes 可自动压缩中间历史，保留开头、最近消息和摘要。

```yaml
compression:
  enabled: true
  threshold: 0.50
  target_ratio: 0.20
  protect_last_n: 20
```

| 字段 | 说明 |
|------|------|
| `enabled` | 是否自动压缩 |
| `threshold` | 到达上下文窗口多少比例时触发 |
| `target_ratio` | 压缩后保留最近尾部的比例 |
| `protect_last_n` | 永远保留的最近消息数量 |

手动压缩：

```text
/compress
/compress 只保留部署问题相关上下文
```

### B.9 记忆配置

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375
  nudge_interval: 10
  flush_min_turns: 6
```

| 字段 | 说明 |
|------|------|
| `memory_enabled` | 启用 Agent 长期记忆 |
| `user_profile_enabled` | 启用用户画像 |
| `memory_char_limit` | `MEMORY.md` 注入预算 |
| `user_char_limit` | `USER.md` 注入预算 |
| `nudge_interval` | 每多少轮提醒 Agent 考虑保存记忆 |
| `flush_min_turns` | 会话重置或退出前，达到多少用户轮次才触发记忆保存 |

### B.10 Skills 配置

```yaml
skills:
  creation_nudge_interval: 15
  external_dirs:
    - ~/.agents/skills
    - /home/shared/team-skills
```

| 字段 | 说明 |
|------|------|
| `creation_nudge_interval` | 复杂任务进行若干轮后提醒 Agent 创建或更新技能 |
| `external_dirs` | 额外技能目录，只读加载；新技能仍写入 `~/.hermes/skills/` |

常用命令：

```bash
hermes skills
/skills
/reload-skills
```

### B.11 Gateway 配置

Gateway 负责把 Telegram、Discord、Slack、WhatsApp、Signal、Email、Home Assistant 等平台接入同一个 Agent。

常见环境变量：

```bash
TELEGRAM_BOT_TOKEN=123456:ABC...
TELEGRAM_ALLOWED_USERS=123456789
DISCORD_BOT_TOKEN=...
SLACK_BOT_TOKEN=xoxb-...
SLACK_SIGNING_SECRET=...
```

常见命令：

```bash
hermes gateway setup
hermes gateway run
hermes gateway start
hermes gateway status
hermes gateway restart
```

WSL、Docker、Termux 场景建议使用前台命令：

```bash
hermes gateway run
```

### B.12 Messaging 会话策略

```yaml
session_reset:
  mode: both
  idle_minutes: 1440
  at_hour: 4

group_sessions_per_user: true
```

| 字段 | 说明 |
|------|------|
| `session_reset.mode` | `both`、`idle`、`daily`、`none` |
| `idle_minutes` | 多久不活跃后自动重置 |
| `at_hour` | 每日重置小时 |
| `group_sessions_per_user` | 群组中是否按用户隔离会话，默认建议为 `true` |

### B.13 MCP 配置

```yaml
mcp_servers:
  time:
    command: uvx
    args: ["mcp-server-time"]
  filesystem:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user"]
  notion:
    url: "https://mcp.notion.com/mcp"
```

配置后会生成动态 toolset，例如 `mcp-time`、`mcp-filesystem`。修改后可在会话里执行：

```text
/reload-mcp
```

### B.14 显示与交互

```yaml
display:
  compact: false
  tool_progress: all
  interim_assistant_messages: true
  busy_input_mode: interrupt
  background_process_notifications: all
  bell_on_complete: false
  show_reasoning: false
  streaming: true
  skin: default
```

| 字段 | 说明 |
|------|------|
| `tool_progress` | 工具进度显示：`off`、`new`、`all`、`verbose` |
| `busy_input_mode` | Agent 忙碌时新输入行为：`interrupt`、`queue`、`steer` |
| `streaming` | CLI 是否流式输出 |
| `skin` | TUI/CLI 皮肤主题 |

### B.15 Hooks 配置

Hooks 可在工具调用、模型调用、会话开始/结束等事件前后执行脚本。

```yaml
hooks:
  pre_tool_call:
    - matcher: "terminal"
      command: "~/.hermes/agent-hooks/block-dangerous.sh"
      timeout: 10
  post_tool_call:
    - matcher: "write_file|patch"
      command: "~/.hermes/agent-hooks/auto-format.sh"

hooks_auto_accept: false
```

建议：

- 团队和生产环境使用 hooks 前先建立脚本审计流程。
- 非交互式 gateway/cron 场景不要默认自动接受未知 hooks。
- Hooks 输出可影响后续 LLM 上下文，应避免泄露密钥。
