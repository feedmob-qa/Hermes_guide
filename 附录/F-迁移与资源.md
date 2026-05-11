## 附录 F：迁移与资源

### F.1 从 OpenClaw 迁移到 Hermes

Hermes 内置了 `hermes claw migrate`，用于把 OpenClaw 的用户数据和配置迁移到 Hermes。

最常用命令：

```bash
hermes claw migrate
hermes claw migrate --dry-run
hermes claw migrate --preset user-data
hermes claw migrate --overwrite
```

#### 常见参数

| 参数 | 说明 |
|------|------|
| `--dry-run` | 只预览，不写入 |
| `--preset full` | 尽量迁移全部兼容设置 |
| `--preset user-data` | 只迁移用户数据，不带密钥 |
| `--migrate-secrets` | 把可识别的 API Key 一并迁移 |
| `--overwrite` | 覆盖 Hermes 中已有冲突项 |
| `--source <path>` | 指定自定义 OpenClaw 目录 |
| `--skill-conflict <mode>` | 技能冲突处理：`skip`、`overwrite`、`rename` |
| `--no-backup` | 跳过迁移前自动备份 |

#### 迁移前建议

1. 先执行 `hermes claw migrate --dry-run`
2. 给当前 Hermes Home 做一次备份
3. 明确哪些配置要迁移，哪些要手工重做
4. 尤其关注消息平台、Bot Token、MCP、技能目录和工作区指令文件

#### 常见会被迁移的内容

| 类别 | 说明 |
|------|------|
| `SOUL.md` | Agent 身份文件 |
| `MEMORY.md` / `USER.md` | 记忆与用户画像 |
| Skills | 用户自建技能及部分技能目录 |
| Provider 配置 | 默认模型、供应商、部分兼容端点 |
| 平台配置 | Telegram、Discord、Slack、WhatsApp、Signal 等 token 与 allowlist |
| Agent 行为 | reasoning、compression、部分 session reset 策略 |
| MCP 服务器 | 已声明的 MCP 配置 |
| TTS / 浏览器 / 工具配置 | 兼容字段可直接映射的部分 |

#### 迁移后必须手工核对的内容

| 项目 | 原因 |
|------|------|
| WhatsApp 配对 | 这类平台常需重新扫码或重新配对 |
| 自定义 SecretRef | 某些 `file` / `exec` 形式密钥 Hermes 无法自动解析 |
| 多 Agent / 多工作区结构 | OpenClaw 与 Hermes 的结构不完全一致 |
| Hooks / Webhooks / 插件 | 通常需要手工审阅 |
| Channel bindings / identity 侧文件 | 很多会被归档而不是直接启用 |

### F.2 迁移后的检查清单

```bash
hermes config check
hermes doctor
hermes model
hermes tools
hermes gateway status
hermes skills
```

重点确认：

- 默认模型是否正确
- `.env` 中密钥是否齐全
- 平台 token 是否仍可用
- `platform_toolsets` 是否符合新架构
- `MEMORY.md`、`USER.md` 是否导入成功
- 旧技能是否能被正确加载

### F.3 官方资源

| 资源 | 链接 | 用途 |
|------|------|------|
| 官方文档 | `https://hermes-agent.nousresearch.com/docs/` | 主文档入口 |
| GitHub 仓库 | `https://github.com/NousResearch/hermes-agent` | 源码、issue、release |
| 中文 README | `README.zh-CN.md` | 中文概览与快速入门 |
| CLI 命令参考 | `website/docs/reference/cli-commands.md` | 终端命令权威列表 |
| Slash 命令参考 | `website/docs/reference/slash-commands.md` | 聊天命令权威列表 |
| Toolsets 参考 | `website/docs/reference/toolsets-reference.md` | 工具集解释 |
| 环境变量参考 | `website/docs/reference/environment-variables.md` | 所有 `.env` 键 |
| 配置说明 | `website/docs/user-guide/configuration.md` | `config.yaml` 详解 |
| FAQ | `website/docs/reference/faq.md` | 常见问题与修复思路 |

### F.4 推荐阅读顺序

#### 对个人用户

1. 快速安装
2. `hermes setup`
3. `hermes model`
4. `hermes`
5. 配置说明与命令速查
6. 需要时再看 Gateway、Memory、Cron

#### 对开发者

1. 配置说明
2. Toolsets 参考
3. CLI / Slash 命令参考
4. Tools runtime 与 developer guide
5. MCP、Skills、Hooks

#### 对团队管理员

1. Gateway 与平台接入
2. Toolsets 最小授权
3. 备份与 Profile
4. Cron、Kanban、可观测性与权限控制

### F.5 常用外部工具与生态

| 工具 / 服务 | 用途 |
|-------------|------|
| OpenRouter | 聚合多模型供应商 |
| Nous Portal | 官方模型入口 |
| Ollama | 本地模型运行 |
| vLLM / SGLang / llama.cpp | OpenAI-compatible 本地或自托管推理端点 |
| Browserbase / 本地 CDP Chrome | 浏览器自动化 |
| Firecrawl / Tavily / Exa | 搜索与网页提取 |
| Honcho / Supermemory | 记忆扩展 |
| MCP Servers | GitHub、Notion、Filesystem 等外部能力接入 |
| Daytona / Modal / Vercel Sandbox | 云端终端后端 |

### F.6 常用环境变量速览

这里只列最常见的一组，完整列表见附录 B 与官方环境变量参考。

#### 模型与供应商

```bash
OPENROUTER_API_KEY=
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
OPENAI_BASE_URL=
GLM_API_KEY=
KIMI_API_KEY=
MINIMAX_API_KEY=
DEEPSEEK_API_KEY=
GOOGLE_API_KEY=
NVIDIA_API_KEY=
```

#### 网关与平台

```bash
TELEGRAM_BOT_TOKEN=
DISCORD_BOT_TOKEN=
SLACK_BOT_TOKEN=
SLACK_SIGNING_SECRET=
SLACK_APP_TOKEN=
```

#### 工具与扩展

```bash
FIRECRAWL_API_KEY=
TAVILY_API_KEY=
EXA_API_KEY=
FAL_KEY=
ELEVENLABS_API_KEY=
GITHUB_TOKEN=
DAYTONA_API_KEY=
VERCEL_TOKEN=
```

#### 运行时

```bash
HERMES_HOME=
HERMES_INFERENCE_PROVIDER=
HERMES_TIMEZONE=
HERMES_YOLO_MODE=
```

### F.7 什么时候查源码，什么时候查文档

优先查官方文档的情况：

- 命令怎么用
- 某个配置项怎么写
- 某平台支持哪些能力
- 某个环境变量叫什么

优先查源码的情况：

- 文档未覆盖的边界行为
- 某命令在 CLI 和 Gateway 下的差异
- 某 toolset 到底包含哪些具体工具
- 某功能当前是否已经实装而不是只在路线图中出现

### F.8 本书附录与官方文档的关系

本书附录的目标不是取代官方文档，而是提供一组高频可用的中文速查资料：

- **官方文档** 更全、更及时
- **本书附录** 更偏向可复制模板、对照表和落地流程

遇到两者不一致时，以官方仓库和官方文档为准，再回头修订本书内容。
