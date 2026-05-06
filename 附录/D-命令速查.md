## 附录 D：命令速查

### D.1 命令分层

Hermes 的命令分成两类：

1. **终端命令**：在 shell 中执行，例如 `hermes model`、`hermes gateway run`
2. **对话 slash 命令**：在 CLI/TUI 或消息平台聊天中输入，例如 `/model`、`/compress`

两者不要混淆：

- 添加新供应商、配置 API Key、跑 OAuth，用终端命令 `hermes model`
- 在当前会话里切换已存在模型，用 slash 命令 `/model`

### D.2 全局入口与常用调用

```bash
hermes [global-options] <command> [subcommand/options]
```

常用全局参数：

| 参数 | 说明 |
|------|------|
| `--profile <name>` | 选择 Profile |
| `--resume <session>` | 恢复指定会话 |
| `--continue [name]` | 恢复最近会话 |
| `--worktree` | 在 Git worktree 中工作 |
| `--yolo` | 跳过危险命令审批 |
| `--ignore-user-config` | 忽略 `~/.hermes/config.yaml` |
| `--ignore-rules` | 忽略 `AGENTS.md`、`SOUL.md`、记忆与预加载技能 |
| `--tui` | 启动 TUI |

### D.3 顶层终端命令

| 命令 | 用途 |
|------|------|
| `hermes` / `hermes chat` | 启动交互式对话或单次对话 |
| `hermes model` | 添加供应商、配置 API Key、执行 OAuth、选择默认模型 |
| `hermes fallback` | 管理主模型失败后的回退链 |
| `hermes setup` | 运行首次或重新配置向导 |
| `hermes gateway` | 运行和管理消息网关 |
| `hermes auth` | 管理凭证池 |
| `hermes status` | 查看 Agent、认证和平台状态 |
| `hermes cron` | 管理定时任务 |
| `hermes tools` | 交互式管理各平台工具 |
| `hermes skills` | 浏览、安装、发布和审计技能 |
| `hermes memory` | 管理外部记忆提供方 |
| `hermes mcp` | 管理 MCP 配置与 MCP Server |
| `hermes plugins` | 管理插件 |
| `hermes config` | 查看、编辑、设置、迁移配置 |
| `hermes sessions` | 浏览、导出、删除会话 |
| `hermes logs` | 查看日志 |
| `hermes doctor` | 诊断问题 |
| `hermes dump` | 输出适合分享的环境摘要 |
| `hermes debug share` | 打包系统信息和日志，生成调试报告 |
| `hermes backup` | 备份 Hermes 用户数据 |
| `hermes checkpoints` | 管理 `/rollback` 使用的 checkpoint 存储 |
| `hermes kanban` | 多项目协作看板 |
| `hermes profile` | 管理 Profile |
| `hermes claw migrate` | 从 OpenClaw 迁移 |
| `hermes update` | 更新 Hermes |
| `hermes uninstall` | 卸载 |

### D.4 `hermes chat` 常用参数

```bash
hermes chat [options]
```

| 参数 | 说明 | 示例 |
|------|------|------|
| `-q`, `--query` | 单次对话 | `hermes chat -q "总结这个目录"` |
| `-m`, `--model` | 本次运行覆盖模型 | `hermes chat -m anthropic/claude-sonnet-4.6` |
| `--provider` | 本次运行覆盖供应商 | `hermes chat --provider openrouter` |
| `-t`, `--toolsets` | 指定本次可用工具集 | `hermes chat --toolsets web,file,terminal` |
| `-s`, `--skills` | 预加载技能 | `hermes chat --skills research/llm-wiki` |
| `--image <path>` | 给单次请求附带本地图片 | `hermes chat --image ./error.png -q "分析报错"` |
| `--quiet` | 抑制 banner 和过程输出 | `hermes chat --quiet -q "只返回 JSON"` |
| `--worktree` | 在独立 worktree 中工作 | `hermes chat --worktree -q "审查仓库"` |
| `--checkpoints` | 对破坏性文件改动启用 checkpoint | `hermes chat --checkpoints` |
| `--max-turns <N>` | 限制单轮工具迭代次数 | `hermes chat --max-turns 40` |

### D.5 纯净单次输出：`hermes -z`

`hermes -z` 适合脚本、CI、父进程调用，只返回最终答案文本。

```bash
hermes -z "法国首都是什么？"
```

常用场景：

- Shell 脚本捕获返回值
- CI 中生成摘要
- 外部程序调用，不想看到 banner、spinner、tool previews

### D.6 设置与配置命令

| 命令 | 说明 | 示例 |
|------|------|------|
| `hermes setup` | 完整配置向导 | `hermes setup` |
| `hermes setup model` | 只配置模型 | `hermes setup model` |
| `hermes setup gateway` | 只配置消息平台 | `hermes setup gateway` |
| `hermes setup terminal` | 只配置终端后端 | `hermes setup terminal` |
| `hermes config` | 查看配置 | `hermes config` |
| `hermes config edit` | 编辑配置文件 | `hermes config edit` |
| `hermes config set KEY VALUE` | 设置键值 | `hermes config set terminal.backend docker` |
| `hermes config check` | 检查缺项 | `hermes config check` |
| `hermes config migrate` | 迁移补全配置 | `hermes config migrate` |

### D.7 模型与凭证命令

| 命令 | 说明 |
|------|------|
| `hermes model` | 交互式添加供应商、配置 API Key、执行 OAuth、选择默认模型 |
| `hermes fallback` | 管理 fallback provider chain |
| `hermes auth` | 交互式管理凭证池 |
| `hermes auth list` | 查看凭证池 |
| `hermes auth add <provider>` | 添加凭证 |
| `hermes auth remove <provider> <index>` | 删除指定凭证 |
| `hermes auth reset <provider>` | 清空某供应商凭证的 cooldown |

示例：

```bash
hermes model
hermes auth add openrouter --api-key sk-or-v1-xxxx
hermes auth list
```

### D.8 Gateway 命令

```bash
hermes gateway <subcommand>
```

| 子命令 | 说明 |
|--------|------|
| `run` | 前台运行网关 |
| `start` | 启动后台服务 |
| `stop` | 停止网关 |
| `restart` | 重启网关 |
| `status` | 查看状态 |
| `install` | 安装为 systemd / launchd 服务 |
| `uninstall` | 卸载服务 |
| `setup` | 交互式配置平台 |

建议：

- WSL、Docker、Termux 优先使用 `hermes gateway run`
- Linux/macOS 长驻服务再考虑 `start` / `install`

### D.9 Cron 命令

```bash
hermes cron <list|create|edit|pause|resume|run|remove|status|tick>
```

| 子命令 | 用途 |
|--------|------|
| `list` | 查看任务 |
| `create` / `add` | 创建任务 |
| `edit` | 修改任务 |
| `pause` | 暂停任务 |
| `resume` | 恢复任务 |
| `run` | 下一个 tick 触发一次 |
| `remove` | 删除任务 |
| `status` | 查看调度器状态 |
| `tick` | 执行一轮 due jobs 后退出 |

示例：

```bash
hermes cron create --name daily-brief --schedule "0 9 * * 1-5" --prompt "总结今天最重要的待办"
hermes cron list
hermes cron pause daily-brief
```

### D.10 Kanban 命令

```bash
hermes kanban [--board <slug>] <action> [options]
```

常用动作：

| 动作 | 用途 |
|------|------|
| `boards list` | 列出看板 |
| `boards create <slug>` | 新建看板 |
| `boards switch <slug>` | 切换当前看板 |
| `create "<title>"` | 新建任务 |
| `list` | 查看任务列表 |
| `show <id>` | 查看任务详情 |
| `assign <id> <profile>` | 分配任务 |
| `comment <id> "<text>"` | 写评论 |
| `block <id> "<reason>"` | 标记阻塞 |
| `unblock <id>` | 解除阻塞 |
| `complete <id>` | 完成任务 |
| `dispatch` | 跑一轮派发器 |

### D.11 备份、调试与诊断

| 命令 | 用途 |
|------|------|
| `hermes doctor` | 诊断依赖、配置、权限、平台接入 |
| `hermes doctor --fix` | 尝试自动修复 |
| `hermes dump` | 输出适合贴到 issue/群里的纯文本环境摘要 |
| `hermes debug share --local` | 本地打印调试报告 |
| `hermes debug share` | 上传日志和环境摘要，得到分享链接 |
| `hermes backup` | 完整备份 |
| `hermes backup --quick` | 快速备份关键状态 |
| `hermes checkpoints` | 查看 checkpoint 占用和项目数 |
| `hermes checkpoints prune` | 清理过旧 checkpoint |

### D.12 CLI / TUI Slash 命令

这些命令在交互式 CLI/TUI 中使用。

#### 会话类

| 命令 | 说明 |
|------|------|
| `/new` / `/reset` | 新建会话 |
| `/clear` | 清屏并开始新会话 |
| `/history` | 查看历史 |
| `/save` | 保存当前对话 |
| `/retry` | 重试上一条消息 |
| `/undo` | 删除上一轮问答 |
| `/title` | 设置会话标题 |
| `/compress [focus]` | 手动压缩上下文 |
| `/rollback [n]` | 查看或恢复 checkpoint |
| `/snapshot ...` | 创建或恢复 Hermes 状态快照 |
| `/stop` | 停止运行中的后台进程 |
| `/queue <prompt>` | 排队下一个 prompt，不打断当前运行 |
| `/steer <prompt>` | 在下一次工具调用后插入新引导 |
| `/goal <text>` | 设定跨多轮自动推进的目标 |
| `/resume [name]` | 恢复命名会话 |
| `/background <prompt>` | 在后台跑独立任务 |
| `/branch [name]` | 分叉会话 |

#### 配置类

| 命令 | 说明 |
|------|------|
| `/config` | 查看当前配置 |
| `/model [model]` | 切换当前会话模型 |
| `/personality` | 设置人格 |
| `/verbose` | 循环切换工具进度显示 |
| `/fast [normal|fast|status]` | 切换快速模式 |
| `/reasoning [level|show|hide]` | 设置 reasoning effort 或显示状态 |
| `/skin` | 切换皮肤 |
| `/statusbar` | 切换状态栏 |
| `/voice [on|off|tts|status]` | 语音模式控制 |
| `/yolo` | 跳过危险命令审批 |
| `/footer [on|off|status]` | 切换最终回复 footer |
| `/busy [queue|steer|interrupt|status]` | Agent 忙时新输入行为 |
| `/indicator [...]` | 忙碌指示器样式 |

#### 工具与技能类

| 命令 | 说明 |
|------|------|
| `/tools [list|disable|enable] [name...]` | 管理当前会话工具 |
| `/toolsets` | 查看可用工具集 |
| `/browser [connect|disconnect|status]` | 管理本地 Chrome CDP 连接 |
| `/skills` | 搜索、安装、检查技能 |
| `/cron` | 管理定时任务 |
| `/curator` | 技能后台维护 |
| `/kanban` | 协作看板 |
| `/reload-mcp` | 重载 MCP 服务器 |
| `/reload` | 重载 `.env` |
| `/plugins` | 查看插件状态 |

#### 信息类

| 命令 | 说明 |
|------|------|
| `/help` | 帮助 |
| `/usage` | Token、成本、会话时长和额度信息 |
| `/insights` | 最近一段时间使用分析 |
| `/platforms` | 查看网关平台状态 |
| `/paste` | 从剪贴板附加图片 |
| `/copy [n]` | 复制最后一次回答 |
| `/image <path>` | 给下一条消息附加本地图片 |
| `/debug` | 上传调试报告 |
| `/profile` | 查看当前 Profile |
| `/gquota` | 查看 Gemini Code Assist 配额 |
| `/quit` / `/exit` | 退出 CLI |

### D.13 消息平台 Slash 命令

以下命令可在 Telegram、Discord、Slack、WhatsApp、Signal、Email、Home Assistant、Teams 等网关聊天中使用。

| 命令 | 说明 |
|------|------|
| `/new` | 新对话 |
| `/reset` | 重置当前上下文 |
| `/status` | 查看会话信息 |
| `/stop` | 停止当前运行并清理后台进程 |
| `/model [provider:model]` | 切换已配置模型 |
| `/personality [name]` | 设置人格 |
| `/fast [normal|fast|status]` | 快速模式 |
| `/retry` | 重试 |
| `/undo` | 撤销上一轮 |
| `/sethome` | 设置当前聊天为 home channel |
| `/compress [focus]` | 压缩上下文 |
| `/title [name]` | 设置会话标题 |
| `/resume [name]` | 恢复会话 |
| `/usage` | 使用统计 |
| `/insights [days]` | 分析统计 |
| `/reasoning [...]` | 调整推理力度 |
| `/voice [...]` | 语音控制 |
| `/rollback [n]` | 文件系统回滚 |
| `/background <prompt>` | 后台任务 |
| `/queue <prompt>` | 队列任务 |
| `/steer <prompt>` | 中途引导 |
| `/goal ...` | 持续目标 |
| `/footer [on|off|status]` | Footer 控制 |
| `/curator ...` | 技能维护 |
| `/kanban ...` | 看板 |
| `/reload-mcp` | 重载 MCP |
| `/yolo` | 跳过危险命令审批 |
| `/commands [page]` | 分页查看命令和技能 |
| `/approve [session|always]` | 批准危险命令 |
| `/deny` | 拒绝危险命令 |
| `/update` | 更新 Hermes |
| `/restart` | 平滑重启网关 |
| `/debug` | 生成调试报告 |
| `/help` | 帮助 |

### D.14 常用操作对照

| 目标 | 终端命令 | 对话命令 |
|------|----------|----------|
| 添加新供应商 | `hermes model` | 不适用 |
| 当前会话切模型 | 不建议 | `/model` |
| 查看工具集 | `hermes tools` | `/toolsets` |
| 禁用某工具 | `hermes tools` | `/tools disable <name>` |
| 配网关 | `hermes gateway setup` | 不适用 |
| 启动网关 | `hermes gateway run` | 不适用 |
| 压缩上下文 | 不适用 | `/compress` |
| 查看日志 | `hermes logs` | 不适用 |
| 生成诊断摘要 | `hermes dump` | `/debug` 更适合分享 |
| 添加定时任务 | `hermes cron create ...` | `/cron create ...` |

### D.15 快速起步命令组

首次安装后最常见的一组命令：

```bash
hermes setup
hermes model
hermes tools
hermes
hermes doctor
```

消息网关最常见的一组命令：

```bash
hermes gateway setup
hermes gateway run
```

排障最常见的一组命令：

```bash
hermes doctor
hermes status --deep
hermes logs
hermes dump
hermes debug share --local
```
