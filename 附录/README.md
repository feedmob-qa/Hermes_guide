## 附录：Hermes Agent 速查手册

### 附录定位

本附录面向已经读完主体章节、需要快速查配置、命令和排障步骤的读者。主体章节负责解释原理与场景，本附录负责提供可复制、可对照、可落地的参考材料。

Hermes 在本书中指 **Nous Research 的 Hermes Agent**：一个可通过 CLI、TUI、消息网关、多工具系统、记忆、技能和自动化能力运行的 AI Agent。它不是 Meta 的 Hermes JavaScript 引擎；本附录所有配置、命令和术语均围绕 Hermes Agent 展开。

### 内容索引

| 附录 | 文件 | 用途 |
|------|------|------|
| 附录 A | [术语表](A-术语表.md) | 快速理解 Agent Loop、Toolset、Gateway、Skill、MCP 等核心概念 |
| 附录 B | [配置说明](B-配置说明.md) | 解释 `~/.hermes/` 目录、`config.yaml`、`.env`、模型、终端、工具、网关等配置项 |
| 附录 C | [配置模板](C-配置模板.md) | 提供最小可用、CLI 开发、消息网关、安全沙箱、本地模型、MCP 等模板 |
| 附录 D | [命令速查](D-命令速查.md) | 汇总终端命令、聊天 slash 命令、网关命令和自动化命令 |
| 附录 E | [排障速查](E-排障速查.md) | 按安装、模型、工具、终端、网关、记忆、自动化分层排障 |
| 附录 F | [迁移与资源](F-迁移与资源.md) | OpenClaw 迁移、官方资源、相关工具和学习路径 |
| 附录 G | [最佳实践](G-最佳实践.md) | 安全、成本、性能、团队部署、备份和可观测性建议 |

### 资料来源

本附录参考 Hermes Agent 官方仓库与文档：

- 官方文档：`https://hermes-agent.nousresearch.com/docs/`
- 官方源码：`https://github.com/NousResearch/hermes-agent`
- 官方配置示例：`cli-config.yaml.example`
- 命令参考：`website/docs/reference/cli-commands.md`
- Slash 命令参考：`website/docs/reference/slash-commands.md`
- Toolsets 参考：`website/docs/reference/toolsets-reference.md`
- 环境变量参考：`website/docs/reference/environment-variables.md`

### 使用方式

- 刚安装：先看附录 C 的最小可用配置，再用附录 D 的 `hermes setup`、`hermes model`、`hermes doctor` 验证。
- 要接入 Telegram、Discord、Slack 等平台：先看附录 B 的 Gateway 配置，再看附录 C 的消息网关模板。
- 要限制风险：重点看附录 B 的 `terminal`、`platform_toolsets`、`security` 和附录 G 的安全最佳实践。
- 出问题：先按附录 E 的六层诊断顺序排查，再用 `hermes dump` 或 `hermes debug share --local` 收集信息。
