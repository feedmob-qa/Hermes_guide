## 附录 A：术语表

### A.1 Agent 与运行时核心

| 术语 | 英文 | 说明 | 常见位置 |
|------|------|------|----------|
| Hermes Agent | Hermes Agent | Nous Research 开源的自进化 AI Agent，提供 CLI/TUI、消息网关、工具、记忆、技能、自动化等能力 | `hermes` 命令、官方仓库 |
| Agent Loop | Agent Loop | Agent 的核心循环：接收消息、组织上下文、调用模型、执行工具、回写结果、继续下一轮 | `run_agent.py` |
| AIAgent | AIAgent | Hermes 的核心 Agent 类，可被 CLI、Gateway、ACP、API Server 等入口复用 | `run_agent.py` |
| Session | Session | 一段连续对话，包含消息历史、工具调用轨迹、模型状态和可恢复 ID | `~/.hermes/sessions/`、`logs/` |
| Profile | Profile | 多套隔离的 Hermes 实例配置。不同 Profile 可拥有独立的 `HERMES_HOME`、模型、记忆、网关和技能 | `hermes profile` |
| Hermes Home | HERMES_HOME | Hermes 的用户数据目录，默认 `~/.hermes/` | 环境变量 `HERMES_HOME` |
| SOUL.md | SOUL.md | Agent 的主身份和长期行为约束，位于系统提示词的高优先级位置 | `~/.hermes/SOUL.md` |
| USER.md | USER.md | 用户画像与偏好记忆，帮助 Agent 跨会话理解用户习惯 | `~/.hermes/memories/USER.md` |
| MEMORY.md | MEMORY.md | Agent 对环境、项目、工作流等事实的长期记忆 | `~/.hermes/memories/MEMORY.md` |

### A.2 配置与凭证

| 术语 | 说明 |
|------|------|
| `config.yaml` | Hermes 的主配置文件，保存模型、终端、工具、压缩、记忆、显示、网关等非密钥设置 |
| `.env` | Hermes 的密钥文件，保存 API Key、Bot Token、SSH/沙箱凭证等敏感变量 |
| 配置优先级 | CLI 参数最高，其次是 `config.yaml`，再是 `.env`，最后是内置默认值 |
| Provider | 模型供应商，例如 OpenRouter、Nous Portal、Anthropic、Gemini、DeepSeek、Kimi、MiniMax、本地 OpenAI-compatible 端点 |
| Model Alias | 模型别名，让 `/model fav` 这类短命令映射到具体 provider/model/base_url |
| Credential Pool | 同一供应商下多组凭证的池化管理，用于轮换、冷却和故障转移 |
| OAuth Credential | 通过浏览器或设备授权获得的凭证，例如 Nous、Anthropic、Copilot、Qwen OAuth |

### A.3 工具系统

| 术语 | 英文 | 说明 |
|------|------|------|
| Tool | Tool | Agent 可调用的能力单元，例如 `web_search`、`read_file`、`terminal`、`memory` |
| Tool Schema | Tool Schema | 描述工具名称、参数、类型、必填项的结构化 schema，供模型生成工具调用 |
| Tool Handler | Tool Handler | 工具的实际执行函数，负责执行命令、读写文件、调用 API 或返回结果 |
| Tool Registry | Tool Registry | 工具注册中心，汇总内置工具、MCP 工具、插件工具并暴露给 Agent |
| Toolset | Toolset | 工具集合，例如 `web`、`file`、`terminal`、`browser`、`skills` |
| Composite Toolset | Composite Toolset | 组合工具集，例如 `debugging` = `file` + `terminal` + `web`，`safe` = 只读研究类工具 |
| Platform Toolset | Platform Toolset | 面向部署平台的完整工具配置，例如 `hermes-cli`、`hermes-telegram`、`hermes-discord` |
| Dynamic Toolset | Dynamic Toolset | 运行时生成的工具集，例如 `mcp-github` 或插件注册的工具集 |
| MCP | Model Context Protocol | 外部工具协议。Hermes 可连接 MCP Server，将其工具作为动态 toolset 注入 |
| Tool Loop Guardrail | Tool Loop Guardrail | 针对重复失败、无进展工具循环的提示和硬停止机制 |

### A.4 消息网关

| 术语 | 说明 |
|------|------|
| Gateway | Hermes 的统一消息网关，用一个后台进程连接 Telegram、Discord、Slack、WhatsApp、Signal、Email 等平台 |
| Platform Adapter | 平台适配器，将平台消息格式转换为 Hermes 内部统一消息格式 |
| Home Channel | 默认投递目标，常用于 cron、后台任务、跨平台消息回传 |
| Pairing | 私聊或设备接入时的批准机制，避免陌生用户直接控制 Agent |
| Allowlist | 允许访问 Agent 的用户 ID、群组 ID 或频道 ID 列表 |
| Runtime Footer | 网关最终回复附带的运行元信息，例如模型、工具调用次数、耗时 |
| Streaming | 在消息平台中边生成边编辑消息的流式体验 |
| Background Session | 后台会话，允许当前聊天继续使用，另一个任务独立运行并在完成后回传 |

### A.5 记忆、技能与知识库

| 术语 | 说明 |
|------|------|
| Memory | 跨会话稳定事实，适合保存用户偏好、项目约定、环境信息 |
| Session Search | 历史会话检索，通常结合 FTS5 全文索引和 LLM 摘要召回 |
| Skill | 可复用程序化知识，通常由 `SKILL.md` 描述触发条件、步骤、脚本和注意事项 |
| Skills Hub | 技能浏览、安装、审计和发布入口 |
| Curator | 后台技能维护机制，用于检查、固定、归档技能 |
| LMWiki | LLM 原生知识库方法，把原始材料、整理后的 wiki 和 schema 结合起来 |
| Memory Provider | 外部记忆插件提供方，例如 Honcho、Supermemory |

### A.6 自动化与协作

| 术语 | 说明 |
|------|------|
| Cron | Hermes 的定时任务系统，可用自然语言或 cron 表达式创建任务并投递到平台 |
| Delegation | 委托任务，让父 Agent 创建子 Agent 并行完成工作 |
| Subagent | 子 Agent，拥有独立上下文和受限工具集，可用于并行研究、代码审查、批量处理 |
| Kanban | 多 Profile、多项目协作看板，可创建任务、分配、阻塞、派发和追踪 |
| Checkpoint | 文件系统快照机制，配合 `/rollback` 撤销危险修改 |
| Hook | Shell 脚本钩子，在工具调用、LLM 调用、会话开始/结束等事件前后执行 |

### A.7 终端与执行环境

| 后端 | 说明 | 适用场景 |
|------|------|----------|
| `local` | 在当前机器直接执行命令 | 个人开发、低风险任务 |
| `docker` | 在长生命周期 Docker 容器中执行命令 | 隔离、CI、可复现环境 |
| `ssh` | 通过 SSH 在远程服务器执行命令 | 远程开发、强算力服务器 |
| `modal` | Modal 云端沙箱 | 临时云计算、批量任务 |
| `daytona` | Daytona 云工作区 | 云端持久开发环境 |
| `vercel_sandbox` | Vercel Sandbox 微虚拟机 | 云端轻量执行 |
| `singularity` | Singularity/Apptainer 容器 | HPC、共享计算集群 |

### A.8 常见误解

| 误解 | 正确理解 |
|------|----------|
| Hermes 是 JavaScript 引擎 | 本书的 Hermes 是 Nous Research 的 Hermes Agent，不是 Meta Hermes JS 引擎 |
| `/model` 可以添加新供应商 | `/model` 只能在已配置供应商之间切换；添加供应商用终端命令 `hermes model` |
| `.env` 和 `config.yaml` 可以随意混用 | 密钥放 `.env`，非密钥行为配置放 `config.yaml` |
| 开启 `terminal` 工具没有风险 | `terminal` 可执行真实命令，生产环境建议启用审批或使用 Docker/SSH/云沙箱 |
| Toolset 越多越好 | 工具越多，误调用和成本越高；应按平台和场景最小授权 |
