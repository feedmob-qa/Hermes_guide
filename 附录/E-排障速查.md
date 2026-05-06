## 附录 E：排障速查

### E.1 排障总原则

Hermes 的问题通常落在六层：

1. 安装与路径
2. 模型与凭证
3. 工具与终端后端
4. 消息网关与平台权限
5. 记忆、技能、MCP、自动化
6. 成本、性能、上下文与并发

不要一上来就重装。先确认是哪一层坏了，再逐层排查。

### E.2 第一组通用诊断命令

```bash
hermes doctor
hermes status --deep
hermes config
hermes logs
hermes dump
hermes debug share --local
```

推荐顺序：

1. `hermes doctor`
2. `hermes status --deep`
3. 查看 `~/.hermes/logs/`
4. 需要对外求助时再用 `hermes dump` 或 `hermes debug share --local`

### E.3 安装与路径问题

#### E.3.1 `hermes: command not found`

| 症状 | 原因 | 处理 |
|------|------|------|
| 安装后终端找不到 `hermes` | shell 没重载，或 `~/.local/bin` 不在 PATH | `source ~/.bashrc`、`source ~/.zshrc`，再执行 `which hermes` |

检查命令：

```bash
which hermes
ls ~/.local/bin/hermes
echo $PATH
```

#### E.3.2 Python 版本过低

| 症状 | 原因 | 处理 |
|------|------|------|
| 安装时报 Python 版本不满足 | Hermes 需要较新的 Python | 执行 `python3 --version`，按平台升级 Python 后重新安装 |

#### E.3.3 `uv: command not found`

| 症状 | 原因 | 处理 |
|------|------|------|
| 手动安装时缺 `uv` | 没装 `uv` 或 PATH 未刷新 | 安装 `uv`，重开 shell，再执行安装脚本 |

### E.4 模型与认证问题

#### E.4.1 模型无法调用

| 症状 | 常见原因 | 处理 |
|------|----------|------|
| 发送消息后模型报错 | API Key 缺失、provider 配错、base_url 不对、模型名不存在 | 先执行 `hermes model` 检查配置，再用 `hermes config` 验证最终值 |

排查顺序：

1. 检查 `.env` 中对应供应商密钥是否存在
2. 检查 `config.yaml` 中 `model.provider`、`model.default`、`model.base_url`
3. 如果是自定义 OpenAI-compatible 端点，确认服务能从本机访问
4. 如果是 OAuth 供应商，重新执行 `hermes model` 或 `hermes auth`

#### E.4.2 `/model` 切不出新供应商

| 症状 | 原因 | 处理 |
|------|------|------|
| 会话里 `/model` 看不到新供应商 | `/model` 只能切换已配置供应商 | 退出会话后运行 `hermes model` 添加新供应商 |

#### E.4.3 本地模型超时

| 症状 | 原因 | 处理 |
|------|------|------|
| Ollama/vLLM/SGLang 长时间无响应 | 模型慢、上下文太大、超时过短 | 拉长 provider timeout，减小上下文，确认本地服务健康 |

建议检查：

- `context_length` 是否与本地服务真实窗口一致
- `providers.<id>.request_timeout_seconds` 是否太短
- 是否把高推理成本任务丢给了慢模型

### E.5 工具与终端问题

#### E.5.1 工具看起来“消失了”

| 症状 | 原因 | 处理 |
|------|------|------|
| Agent 不再调用某工具 | 当前平台 toolset 不包含该工具，或工具被手动禁用 | CLI 执行 `hermes tools`；会话里执行 `/tools list`、`/toolsets` |

快速检查：

```bash
hermes tools
```

```text
/tools list
/toolsets
```

#### E.5.2 命令总是要求审批

| 症状 | 原因 | 处理 |
|------|------|------|
| 执行 shell 命令总被卡住 | Hermes 识别到危险命令，需要人工批准 | 在 CLI 里手动批准；只在受控场景使用 `/yolo` 或 `--yolo` |

建议：

- 个人本地开发可以临时用 `/yolo`
- 团队或消息平台入口不要长期无审批放开

#### E.5.3 Docker 后端无法工作

| 症状 | 原因 | 处理 |
|------|------|------|
| `terminal` 工具报 Docker 不可用 | Docker 未启动、镜像名错误、当前用户无权限 | 先确认 Docker 本机可用，再检查 `terminal.backend` 和 `docker_image` |

检查项：

```bash
docker version
docker images
```

配置重点：

- `terminal.backend: docker`
- `docker_image` 是否存在
- 是否错误挂载了无权限目录

#### E.5.4 SSH 后端连接失败

| 症状 | 原因 | 处理 |
|------|------|------|
| Agent 无法在远程机执行命令 | SSH 主机、账号、密钥或 host key 有问题 | 先手工 SSH 成功，再让 Hermes 接管 |

先手工验证：

```bash
ssh user@host
```

#### E.5.5 浏览器工具不可用

| 症状 | 原因 | 处理 |
|------|------|------|
| `/browser connect` 失败或工具不可见 | CDP 端点没起来，或当前平台没有 browser toolset | 确认 Chrome 调试端口可用，检查 toolset |

### E.6 Gateway 与平台问题

#### E.6.1 机器人不回消息

| 症状 | 常见原因 | 处理 |
|------|----------|------|
| Telegram/Discord/Slack 没反应 | 网关未运行、Bot Token 错、allowlist 未放行、平台权限不足 | 先看 `hermes gateway status` 和日志，再查平台权限 |

排查顺序：

1. `hermes gateway status`
2. `hermes logs`
3. 检查 `.env` 中平台 Token
4. 检查 allowlist / allowed chats 配置
5. 检查机器人是否已被邀请到群组或频道

#### E.6.2 WSL 里网关容易断

| 症状 | 原因 | 处理 |
|------|------|------|
| `hermes gateway start` 不稳定 | WSL 下后台服务管理不如原生 Linux 稳定 | 改用 `hermes gateway run`，必要时用 `tmux` 持续运行 |

#### E.6.3 Telegram/Discord 只能私聊不能群聊

| 症状 | 原因 | 处理 |
|------|------|------|
| 私聊正常，群聊无反应 | 群组/频道未被放行，或机器人没有发送权限 | 检查群组 allowlist、Bot 权限、是否设置了 `group_sessions_per_user` |

#### E.6.4 定时任务不投递

| 症状 | 原因 | 处理 |
|------|------|------|
| `cron` 创建成功但没有消息 | Gateway 没跑、deliver target 错、home channel 没设、平台凭证失效 | 先跑 `hermes cron list` 和 `hermes gateway status`，再核对 deliver 目标 |

### E.7 Cron 问题

#### E.7.1 任务根本没触发

| 检查点 | 说明 |
|--------|------|
| 任务是否存在且是 active | `hermes cron list` |
| 调度表达式是否正确 | 检查 cron 表达式、相对时间或自然语言 |
| Gateway 是否在运行 | 定时任务依赖 Gateway 的后台 ticker |
| 本机时区是否正确 | `date` 与 `hermes cron list` 中的 next run 是否一致 |

#### E.7.2 任务执行了但没发出来

| 检查点 | 说明 |
|--------|------|
| `deliver` 目标是否正确 | `origin`、`telegram`、`discord`、`slack` 等 |
| 平台 Token 是否有效 | `.env` 中重新确认 |
| Bot 是否有发言权限 | 群组、频道、Slack scope 等 |
| Prompt 是否要求 `[SILENT]` | 某些监控型 prompt 会故意抑制输出 |

#### E.7.3 多个任务同时卡住

| 原因 | 处理 |
|------|------|
| 调度器串行执行同一 tick 内的任务 | 错开任务时间，不要都压在整点 |
| 同一机器跑了多个 Gateway/cron 进程 | 保留一个实例，清理重复进程 |

### E.8 技能、记忆、MCP 问题

#### E.8.1 技能安装了但不会触发

| 症状 | 原因 | 处理 |
|------|------|------|
| 技能在目录里，但对话里没反应 | 技能未重载、命名不匹配、触发条件不适用 | 执行 `/skills`、`/reload-skills`，确认技能名与说明 |

#### E.8.2 记忆没有生效

| 症状 | 常见原因 | 处理 |
|------|----------|------|
| Agent 忘了之前的约定 | `memory_enabled` 或 `user_profile_enabled` 关闭；预算太小；会话未达到 flush 条件 | 检查 `memory` 配置并确认内容真的写入 `MEMORY.md` / `USER.md` |

检查项：

- `memory.memory_enabled: true`
- `memory.user_profile_enabled: true`
- `~/.hermes/memories/` 中是否有对应内容

#### E.8.3 MCP 工具不出现

| 症状 | 原因 | 处理 |
|------|------|------|
| 配了 MCP 服务器但工具列表没有 | server 启动失败、配置错误、会话未 reload | 检查 `mcp_servers` 配置，然后执行 `/reload-mcp` |

进一步检查：

- 命令型 MCP：可执行文件是否存在
- HTTP MCP：URL 是否可达，认证头是否正确
- 动态 toolset 是否被包含在当前平台可用工具里

### E.9 上下文、性能与成本问题

#### E.9.1 上下文越来越大，响应变慢

| 症状 | 原因 | 处理 |
|------|------|------|
| 回复越来越慢、费用越来越高 | 会话过长，压缩阈值太晚，最近消息保护过多 | 使用 `/compress`，并调整 `compression.threshold` 与 `protect_last_n` |

#### E.9.2 背景任务或子 Agent 抢资源

| 症状 | 原因 | 处理 |
|------|------|------|
| 本轮对话明显变慢 | 同时跑了多个 background session、cron job 或 subagent | 用 `/agents` 查看活跃任务，必要时 `/stop`，并降低并发 |

#### E.9.3 成本突然升高

| 原因 | 处理 |
|------|------|
| 使用了高价模型 | 在 `hermes model` 或 `/model` 里切换到更便宜模型 |
| 会话太长没压缩 | 启用或调低压缩阈值 |
| 工具集过宽引发多轮尝试 | 收紧 `platform_toolsets` |
| cron 或多子 Agent 并行过多 | 降低 `agent.max_turns`、delegation 并发、任务频率 |

### E.10 备份与恢复问题

#### E.10.1 升级前怕坏

```bash
hermes backup
hermes backup --quick
```

#### E.10.2 新机器迁移

```bash
hermes backup
hermes import ~/hermes-backup-xxxx.zip
```

如果只迁移某个 Profile，优先使用对应的 profile 导出/导入流程，而不是整机覆盖。

### E.11 快速决策树

```text
没法启动？
  └─ 先查 PATH / Python / uv / 安装日志

能启动但不能回答？
  └─ 查 provider / API Key / model / base_url

能回答但不会用工具？
  └─ 查 platform_toolsets / /tools list / hermes tools

Telegram/Discord 不回消息？
  └─ 查 gateway status / Bot Token / allowlist / 平台权限

定时任务不触发？
  └─ 查 hermes cron list / gateway 是否运行 / 时区

记忆或技能失效？
  └─ 查 memory 配置 / 技能目录 / reload-mcp / reload-skills

越来越慢越来越贵？
  └─ 查 compression / model / background tasks / subagents
```

### E.12 求助前最少信息包

遇到复杂问题时，至少准备以下信息：

1. 使用入口：CLI、Telegram、Discord、Slack、cron、kanban 还是 MCP
2. 当前 provider 和 model
3. 当前 terminal backend
4. 最近执行的命令或 slash 命令
5. `hermes doctor` 输出结论
6. `hermes dump` 或 `hermes debug share --local` 摘要
7. 相关日志片段

这比单独一句“坏了”有效得多。
