## 第七章 Agent Loop 内核剖析

> 本章定位：第三部分·精通 | 预估篇幅：~4000行 | 前置阅读：第1-3章

**内容导读**

```
○ 7.1 请求流转
  - ○ 7.1.1 消息完整生命周期追踪
  - ○ 7.1.2 四层架构：入口→控制→执行→能力
  - ○ 7.1.3 端到端消息链路图
  - ○ 7.1.4 分层排障策略

○ 7.2 嵌入式集成
  - ○ 7.2.1 Hermes 如何嵌入 LLM 运行时
  - ○ 7.2.2 工具注入机制
  - ○ 7.2.3 事件订阅

○ 7.3 事件驱动模型
  - ○ 7.3.1 事件总线设计
  - ○ 7.3.2 Tick 机制
  - ○ 7.3.3 状态恢复

○ 7.4 入口与排队
  - ○ 7.4.1 协议归一化
  - ○ 7.4.2 幂等控制（防重放）
  - ○ 7.4.3 分道排队（Session/Global/Background）
  - ○ 7.4.4 反压与超时

○ 7.5 提示词装配
  - ○ 7.5.1 动态 System Prompt
  - ○ 7.5.2 上下文预算与裁剪
  - ○ 7.5.3 注入防御

○ 7.6 工具执行
  - ○ 7.6.1 工具签名与调度
  - ○ 7.6.2 结果回注
  - ○ 7.6.3 大尺寸结果裁剪

○ 7.7 流式输出
  - ○ 7.7.1 Block Chunker
  - ○ 7.7.2 重试策略
  - ○ 7.7.3 提前终止

○ 7.8 本章小结
```

---

### 7.1 请求流转

#### 7.1.1 消息完整生命周期追踪

每条用户消息进入 Hermes 后，经历一条从入口到输出的完整流水线。理解这条流水线是调优和排障的基础。

**消息生命周期时序**

```
用户发送消息
    │
    ▼
┌─────────────┐     ┌──────────────────┐
│  入口层      │────▶│  控制层           │
│  Platform    │     │  AIAgent         │
│  Adapter     │     │  run_conversation│
└─────────────┘     └───────┬──────────┘
                            │
                            ▼
                    ┌──────────────────┐
                    │  执行层           │
                    │  Agent Loop      │
                    │  (感知→思考→行动) │
                    └───────┬──────────┘
                            │
                            ▼
                    ┌──────────────────┐
                    │  能力层           │
                    │  Tool → Provider │
                    │  → LLM API       │
                    └──────────────────┘
```

每条消息的详细生命周期步骤如下：

| 阶段 | 组件 | 关键函数 | 耗时占比 |
|------|------|----------|---------|
| 1. 协议解析 | Platform Adapter | `parse_incoming()` | <1% |
| 2. 队列调度 | Gateway Queue | `enqueue()` | <1% |
| 3. 会话查找 | Session Manager | `get_or_create_session()` | 1-2% |
| 4. 提示词装配 | AIAgent | `_build_system_prompt()` | 3-5% |
| 5. LLM 调用 | AIAgent | `_interruptible_api_call()` | 60-80% |
| 6. 工具调用 | AIAgent | `_process_tool_calls()` | 15-25% |
| 7. 流式输出 | Block Chunker | `chunk_and_send()` | 5-10% |

#### 7.1.2 四层架构：入口→控制→执行→能力

Hermes 的四层架构设计使其能够支持 15+ 消息平台、20+ LLM 供应商、80+ 内置工具，而核心逻辑保持单一。

**第1层：入口层（Gateway）**

源码路径：`gateway/` 目录。每个平台对应一个适配器文件。

```
gateway/
├── run.py                # 网关主循环
├── message_handler.py    # 消息统一处理
├── approval_queue.py     # 审批队列管理
└── platforms/
    ├── telegram_adapter.py
    ├── discord_adapter.py
    ├── slack_adapter.py
    ├── feishu_adapter.py
    ├── dingtalk_adapter.py
    └── ...
```

每个适配器实现两个核心接口：

```python
# gateway/base_adapter.py (抽象)
class BaseAdapter:
    def parse_incoming(self, raw: dict) -> UnifiedMessage: ...
    def send_message(self, message: UnifiedMessage) -> None: ...
```

`UnifiedMessage` 是平台无关的消息中间表示，包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `text` | str | 消息文本 |
| `user_id` | str | 发送者 ID |
| `chat_id` | str | 会话 ID |
| `platform` | str | 来源平台 |
| `attachments` | list | 附件（图片、文件等） |
| `raw` | dict | 原始平台数据 |

**第2层：控制层（AIAgent）**

源码路径：`run_agent.py` 中的 `class AIAgent`。这是整个系统的核心。

```python
class AIAgent:
    def __init__(self, base_url, model, provider, ...):
        # 初始化模型客户端、工具集、记忆系统
        self.iteration_budget = IterationBudget(max_iterations=90)
        self._cached_system_prompt = None
        self._memory_store = ...
        self._todo_store = ...
        self._tool_guardrails = ToolCallGuardrailController(...)
```

**第3层：执行层（Agent Loop）**

即 `run_conversation()` 方法中的主循环。

**第4层：能力层（Tool + Provider）**

工具注册在 `tools/registry.py`，供应商路由在 `run_agent.py` 的 `_call_llm()` 中。

#### 7.1.3 端到端消息链路图

以下是用例为"在 Telegram 上发送 `/skills` 查看技能列表"的完整链路：

```
Telegram用户 → [Webhook] → TelegramAdapter.parse_incoming()
    → UnifiedMessage{text: "/skills", platform: "telegram"}
    → GatewayQueue.enqueue()
    → SessionManager.get_or_create_session(chat_id)
    → AIAgent.run_conversation("/skills")
        → _build_system_prompt()  # 缓存命中则跳过
        → messages.append(user_msg)
        → Loop start:
            1. _interruptible_api_call(messages) → LLM 返回 tool_call
            2. _process_tool_calls() → dispatch skills_list()
            3. Append tool result → back to LLM
            4. LLM 返回 text response
        → Loop end
    → send_message("当前已安装技能：...")
    → TelegramAdapter.send_message(unified_response)
    → Telegram API → 用户看到回复
```

#### 7.1.4 分层排障策略

当消息链路出现异常时，按层逐步排查：

| 层 | 症状 | 诊断命令 | 常见原因 |
|----|------|----------|---------|
| 1-入口 | 平台无响应 | `hermes platform telegram --test` | Webhook 配置错误 |
| 2-控制 | 网关崩溃 | `hermes gateway status` | 凭证过期 |
| 3-执行 | 循环卡死 | `hermes logs --filter "iteration"` | 工具调用死循环 |
| 4-能力 | 模型报错 | `hermes doctor` | API Key 无效 |

---

### 7.2 嵌入式集成

#### 7.2.1 Hermes 如何嵌入 LLM 运行时

Hermes 不是 LLM 本身——它是一个围绕 LLM 构建的智能体运行时。核心思想是：**把 LLM 当作一个"推理引擎"来调用，外围封装工具执行、记忆管理、上下文控制的完整循环**。

架构上，Hermes 通过 OpenAI 兼容的 API 格式与 LLM 通信：

```
LLM Provider (OpenAI/Anthropic/本地)
    ▲ OpenAI-format messages + tools
    │
    ▼ Tool call JSON / Text response
Hermes AIAgent
    ▲ Tool call → dispatch to handler → result string
    │
    ▼ Terminal | File | Web | Vision | ...
```

代码层面，`run_agent.py` 的 `_call_llm()` 方法构建请求体：

```python
def _call_llm(self, messages, tools=None, stream=True):
    kwargs = {
        "model": self.model,
        "messages": messages,
        "tools": tools or self._tool_definitions,
        "stream": stream,
    }
    # 调用 OpenAI SDK
    response = self.client.chat.completions.create(**kwargs)
    return response
```

#### 7.2.2 工具注入机制

工具注入发生在两个层面：

**1. Schema 注入**：在每次 LLM 调用时，将工具定义以 OpenAI Function Calling 格式传入。

```python
# model_tools.py
def get_tool_definitions(tool_names: list[str]) -> list[dict]:
    """返回 OpenAI-format 工具 schema 列表"""
    definitions = []
    for name in tool_names:
        entry = registry.get_entry(name)
        if entry and _check_fn_cached(entry.check_fn):
            schema = {"type": "function", "function": entry.schema}
            # 动态 schema 覆盖（如 delegate_task 的并发限制）
            if entry.dynamic_schema_overrides:
                schema["function"].update(entry.dynamic_schema_overrides())
            definitions.append(schema)
    return definitions
```

**2. Handler 分发**：LLM 返回 `tool_calls` 后，通过 `handle_function_call()` 分派。

```python
# model_tools.py
def handle_function_call(tool_name, arguments, task_id=None):
    entry = registry.get_entry(tool_name)
    if not entry:
        return json.dumps({"error": f"Unknown tool: {tool_name}"})
    # 可用性检查
    if not _check_fn_cached(entry.check_fn):
        return json.dumps({"error": f"Tool {tool_name} not available"})
    # 执行
    return entry.handler(**parsed_args, task_id=task_id)
```

#### 7.2.3 事件订阅

Hermes 支持插件系统，通过事件钩子（Event Hook）机制扩展功能。当前实现的事件点：

| 事件 | 触发时机 | 源码位置 |
|------|----------|----------|
| `on_session_start` | 全新会话创建时 | `run_agent.py:11750` |
| `on_tool_complete` | 工具执行完成后 | `run_agent.py:_process_tool_calls` |
| `on_message_send` | 消息投递前 | `gateway/message_handler.py` |

插件注册方式：

```python
# 在 plugin.py 中
from hermes_cli.plugins import register_hook

@register_hook("on_session_start")
def my_hook(session_id, model, platform):
    print(f"Session started: {session_id}")
```

---

### 7.3 事件驱动模型

#### 7.3.1 事件总线设计

Hermes 的核心不依赖外部事件总线——控制流是同步的 `run_conversation()` 循环。但在网关层，每个平台适配器通过独立线程或异步事件循环监听消息。

**网关事件模型**：

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Telegram     │    │ Discord      │    │ Slack        │
│ Polling      │    │ WebSocket    │    │ Events API   │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           ▼
                  ┌──────────────────┐
                  │  Gateway Queue   │
                  │  (线程安全队列)   │
                  └──────────────────┘
                           │
                           ▼
                  ┌──────────────────┐
                  │  Message Handler │
                  │  → AIAgent       │
                  └──────────────────┘
```

#### 7.3.2 Tick 机制

Hermes 中的"Tick"指 `run_conversation()` 内部的一次迭代（iteration）。每个 tick 包含三个步骤：

```
Tick N:
  1. 组装消息 → 调用 LLM（_interruptible_api_call）
  2. 解析响应
     ├── 如果包含 tool_calls → 执行工具 → 结果回注 → Tick N+1
     ├── 如果包含 text → 输出 → 结束 loop
     └── 如果出错 → 分类恢复 → 重试 Tick
```

控制 tick 数目的关键参数：

```yaml
# config.yaml
agent:
  max_turns: 90                  # 最大迭代次数（默认）
delegation:
  max_iterations: 50             # 子 agent 最大迭代次数
```

`IterationBudget` 类确保 tick 计数线程安全：

```python
class IterationBudget:
    def __init__(self, max_total: int):
        self.max_total = max_total
        self._used = 0
        self._lock = threading.Lock()

    def consume(self) -> bool:
        with self._lock:
            if self._used >= self.max_total:
                return False
            self._used += 1
            return True
```

#### 7.3.3 状态恢复

在以下情况下，Hermes 需要恢复会话状态：

**1. 网关重启后**：`AIAgent` 从 `session_db` 中加载缓存过的 `system_prompt`，避免重复构建导致 Anthropic 前缀缓存失效。

```python
stored_prompt = self._session_db.get_session(self.session_id)
if stored_prompt:
    self._cached_system_prompt = stored_prompt["system_prompt"]
```

**2. 用户 /continue 继续会话**：传入 `conversation_history`，AIAgent 从中重建 `_user_turn_count` 和 `_turns_since_memory`，使 nudge 逻辑在会话恢复后继续正确工作。

**3. 后台任务 /background**：通过独立线程运行，任务完成时自动通知。

---

### 7.4 入口与排队

#### 7.4.1 协议归一化

每个平台适配器将平台特有的消息格式转换成 `UnifiedMessage`。这是整个系统的"通用语言"。

Telegram 的 `Message` → UnifiedMessage 示例：

```python
# gateway/platforms/telegram_adapter.py
class TelegramAdapter(BaseAdapter):
    def parse_incoming(self, update: dict) -> UnifiedMessage:
        msg = update.get("message", {})
        return UnifiedMessage(
            text=msg.get("text", ""),
            user_id=str(msg["from"]["id"]),
            chat_id=str(msg["chat"]["id"]),
            platform="telegram",
            attachments=[
                Attachment(type="photo", file_id=photo["file_id"])
                for photo in msg.get("photo", [])
            ],
            raw=update,
        )
```

#### 7.4.2 幂等控制（防重放）

消息平台（特别是 Webhook 模式）可能重复投递同一条消息。Hermes 通过以下机制防止重复处理：

| 机制 | 实现 | 适用场景 |
|------|------|----------|
| 消息 ID 去重 | `update_id` 缓存 | Telegram Webhook |
| 幂等 Key | 计算消息哈希 | Discord |
| 时间窗口 | 5秒内相同内容丢弃 | 所有平台 |

#### 7.4.3 分道排队（Session/Global/Background）

Hermes 支持三类队列，优先级依次递减：

| 队列 | 用途 | 优先级 | 并发限制 |
|------|------|--------|----------|
| **Session** | 当前会话的实时消息 | 最高 | 1（串行） |
| **Global** | 其他会话的消息 | 中 | 取决于配置 |
| **Background** | `/background` 任务 | 低 | 可配置 |

网关层的线程池实现：

```python
# gateway/run.py (简化)
MAX_TOOL_WORKERS = 8  # 最大并行工具执行线程

def process_queue():
    while True:
        message = queue.dequeue()
        if message.priority == "session":
            # 直接处理
            handle_message(message)
        else:
            # 放入线程池
            ThreadPoolExecutor(max_workers=MAX_TOOL_WORKERS).submit(handle_message, message)
```

#### 7.4.4 反压与超时

当系统负载超过容量时，反压机制防止雪崩：

| 反压策略 | 实现 | 触发条件 |
|----------|------|----------|
| 背压（Backpressure） | 队列满时拒绝新请求 | 队列容量 > 100 |
| 超时（Timeout） | API 调用可配超时 | `timeout: 180` (默认) |
| 限流（Rate Limit） | 每分钟请求数限制 | `requests_per_minute: 60` |

---

### 7.5 提示词装配

#### 7.5.1 动态 System Prompt

`_build_system_prompt()` 方法按顺序组装多个层次。这是理解 Hermes 行为的关键。

**System Prompt 装配图**：

```
Layer 1: Agent Identity
  ├── SOUL.md（如果存在）
  └── DEFAULT_AGENT_IDENTITY（回退）

Layer 2: Hermes 使用指导
  └── HERMES_AGENT_HELP_GUIDANCE

Layer 3: 工具行为指导
  ├── MEMORY_GUIDANCE（如果 memory 工具已加载）
  ├── SESSION_SEARCH_GUIDANCE（如果 session_search 已加载）
  ├── SKILLS_GUIDANCE（如果 skill_manage 已加载）
  └── KANBAN_GUIDANCE（如果 kanban_show 已加载）

Layer 4: 工具执行强制
  ├── TOOL_USE_ENFORCEMENT_GUIDANCE
  ├── GOOGLE_MODEL_OPERATIONAL_GUIDANCE（Gemini 模型）
  └── OPENAI_MODEL_EXECUTION_GUIDANCE（GPT/Codex 模型）

Layer 5: 用户提供的 system_message

Layer 6: 持久记忆
  ├── MEMORY.md（用户偏好/环境事实）
  └── USER.md（用户画像）

Layer 7: 技能系统
  └── build_skills_system_prompt()

Layer 8: 上下文文件
  ├── AGENTS.md / .cursorrules（项目约束）
  └── HERMES.md（Hermes 特定配置）

Layer 9: 时间戳 & 环境信息

Layer 10: 平台特定提示
```

关键代码实现：

```python
def _build_system_prompt(self, system_message=None):
    prompt_parts = []

    # Layer 1: Identity
    if self.load_soul_identity or not self.skip_context_files:
        soul = load_soul_md()
        if soul:
            prompt_parts.append(soul)
    if not prompt_parts:
        prompt_parts.append(DEFAULT_AGENT_IDENTITY)

    # Layer 2: Help guidance
    prompt_parts.append(HERMES_AGENT_HELP_GUIDANCE)

    # Layer 3: Tool behavioral guidance (conditional)
    tool_guidance = []
    if "memory" in self.valid_tool_names:
        tool_guidance.append(MEMORY_GUIDANCE)
    if "session_search" in self.valid_tool_names:
        tool_guidance.append(SESSION_SEARCH_GUIDANCE)
    # ... more conditional blocks
    if tool_guidance:
        prompt_parts.append(" ".join(tool_guidance))

    # ... Layer 4-10 ...
```

**缓存机制**：`_cached_system_prompt` 在整个会话中只构建一次。只有在上下文压缩事件后才会失效重建。这确保了 Anthropic 风格的前缀缓存（Prefix Caching）始终命中。

#### 7.5.2 上下文预算与裁剪

上下文预算管理是 Agent 长时间运行的关键。Hermes 使用 `ContextCompressor` 类（`agent/context_compressor.py`）自动管理上下文。

**压缩触发条件**：

| 条件 | 阈值 | 行为 |
|------|------|------|
| 接近上下文限制 | 达到 `max_context_tokens` 的 70% | 自动提示压缩 |
| 上下文即将溢出 | 超过 `context_compression_threshold` | 自动执行压缩 |
| 手动触发 | `/compress` 命令 | 用户主动压缩 |

**压缩算法**：

```
输入: 消息列表 [msg1, msg2, ..., msgN]
输出: 压缩后的消息列表 [summary, msgN-2, msgN-1, msgN]

流程:
1. 保护策略：保留开头消息（系统消息）和末尾消息（最近对话）
2. 中间内容 → 发送到辅助 LLM 生成摘要
3. 摘要格式：
   [CONTEXT COMPACTION — REFERENCE ONLY]
   ## 已完成的工作
   ## 活跃的任务
   ## 待解决的问题
   ## 重要决策
4. 将摘要作为新消息插入
```

```python
# agent/context_compressor.py
class ContextCompressor:
    def compress(self, messages, max_tokens):
        # 1. 确定要压缩的范围
        head, middle, tail = self._split_messages(messages)
        # 2. 如果中间内容比摘要预算大很多
        if self._exceeds_budget(middle, max_tokens):
            # 3. 用辅助 LLM 生成摘要
            summary = self._summarize(middle)
            # 4. 返回 [head, summary, tail]
            return head + [summary] + tail
        return messages  # 不需要压缩
```

**Token 预算计算**：

```python
_MIN_SUMMARY_TOKENS = 1000
_SUMMARY_RATIO = 0.20  # 压缩内容 20% 用作摘要
_SUMMARY_TOKENS_CEILING = 8000

summary_budget = min(
    max(_MIN_SUMMARY_TOKENS, int(compressed_chars * _SUMMARY_RATIO)),
    _SUMMARY_TOKENS_CEILING
)
```

#### 7.5.3 注入防御

Hermes 在读取项目上下文文件（AGENTS.md、.cursorrules、HERMES.md）时会检测 Prompt Injection。

```python
# agent/prompt_builder.py
_CONTEXT_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    (r'system\s+prompt\s+override', "sys_prompt_override"),
    (r'disregard\s+(your|all|any)\s+(instructions|rules|guidelines)', "disregard_rules"),
    # ...
]
```

检测到威胁模式时，上下文文件被阻止加载并记录日志：

```python
if findings:
    logger.warning("Context file %s blocked: %s", filename, ", ".join(findings))
    return f"[BLOCKED: {filename} contained potential prompt injection]"
```

此外，不可见 Unicode 字符（零宽空格、双向文本覆盖等）也会被检测和拦截。

---

### 7.6 工具执行

#### 7.6.1 工具签名与调度

**自注册机制**：每个工具文件在 `tools/*.py` 中通过 `registry.register()` 声明自身：

```python
# tools/web_search.py
def check_web_search() -> bool:
    return True  # 始终可用

registry.register(
    name="web_search",
    toolset="web",
    schema={
        "name": "web_search",
        "description": "搜索互联网获取实时信息",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "搜索关键词"}
            },
            "required": ["query"]
        }
    },
    handler=lambda args, **kw: web_search(args["query"]),
    check_fn=check_web_search,
)
```

**工具发现**：`tools/registry.py` 的 `discover_builtin_tools()` 扫描 `tools/` 目录下所有 `.py` 文件，通过 AST 分析检测哪些文件包含顶层 `registry.register()` 调用，然后动态 import：

```python
def discover_builtin_tools(tools_dir=None):
    tools_path = Path(tools_dir) if tools_dir else Path(__file__).resolve().parent
    for path in sorted(tools_path.glob("*.py")):
        if _module_registers_tools(path):
            importlib.import_module(f"tools.{path.stem}")
```

**可用性检查**：每个工具可以定义 `check_fn`，返回 `False` 时该工具在 LLM 视角中不可见。`check_fn` 有 30 秒 TTL 缓存：

```python
_CHECK_FN_TTL_SECONDS = 30.0
_check_fn_cache: Dict[Callable, tuple[float, bool]] = {}
```

**并行执行**：当 LLM 一次返回多个 `tool_calls` 时，Hermes 可以并行执行它们：

| 工具类型 | 并行策略 | 条件 |
|----------|----------|------|
| 只读工具（web_search, read_file） | 无限制并行 | `_PARALLEL_SAFE_TOOLS` |
| 路径作用域工具（write_file, patch） | 路径不冲突时并行 | `_PATH_SCOPED_TOOLS` |
| 交互工具（clarify） | 强制串行 | `_NEVER_PARALLEL_TOOLS` |
| 其他 | 串行 | 默认 |

```python
_MAX_TOOL_WORKERS = 8  # 最大并行线程数

def _should_parallelize_tool_batch(tool_calls):
    names = [tc.function.name for tc in tool_calls]
    if any(name in _NEVER_PARALLEL_TOOLS for name in names):
        return False
    if all(name in _PARALLEL_SAFE_TOOLS for name in names):
        return True
    # 路径作用域工具：检查冲突
    if all(name in _PATH_SCOPED_TOOLS for name in names):
        paths = [_extract_parallel_scope_path(n, a) for n, a in ...]
        return not _any_paths_overlap(paths)
    return False
```

#### 7.6.2 结果回注

工具执行完成后，结果被组装成 OpenAI-format 的 `tool` 角色消息，追加到消息列表：

```python
# 工具执行后
tool_result = handle_function_call(tool_name, arguments)
messages.append({
    "role": "tool",
    "tool_call_id": tool_call.id,
    "content": tool_result
})
# 继续 loop，LLM 看到之前调用的结果
```

#### 7.6.3 大尺寸结果裁剪

工具返回结果可能非常庞大（如读取大文件、网页抓取）。Hermes 通过以下机制控制：

| 机制 | 阈值 | 行为 |
|------|------|------|
|  | `max_tool_result_size_chars` | 超过后截断 |
| 工具级限制 | `max_result_size_chars` | 单工具自定义上限 |
| Turn Budget | `enforce_turn_budget()` | 工具结果占用超过阈值时触发压缩 |

```python
# tools/tool_result_storage.py
def maybe_persist_tool_result(result: str, max_size: int = 50000) -> str:
    if len(result) > max_size:
        # 截断并添加提示
        return result[:max_size] + "\n[Tool result truncated...]"
    return result
```

---

### 7.7 流式输出

#### 7.7.1 Block Chunker

当 LLM 以流式（SSE）方式返回文本时，Hermes 将连续的字符流组装成有意义的"块"（Chunk），逐块发送到客户端。

流式处理的完整路径：

```
LLM Stream (SSE)
    │
    ▼
_interruptible_api_call()
    │ stream=True
    ▼
Stream delta chunks
    │ per content block
    ▼
StreamingContextScrubber (移除敏感信息)
    │
    ▼
StreamingThinkScrubber (处理 think 块)
    │
    ▼
stream_callback(delta) → 客户端 UI / TTS
```

StreamingThinkScrubber 负责处理 LLM 的思考过程（如 DeepSeek R1 的 `reasoning_content` 或 Anthropic 的 `thinking` 块），将其与最终输出分离。

#### 7.7.2 重试策略

Hermes 实现了多层重试策略，通过 `agent/retry_utils.py` 的 `jittered_backoff()` 控制：

| 错误类型 | 重试次数 | 退避策略 | 行为 |
|----------|----------|----------|------|
| 网络超时 | 3 次 | 指数退避 + 抖动 | 重建 HTTP 客户端 |
| API 限流 (429) | 5 次 | 30s → 60s → 2m → 4m → 8m | 等待后重试 |
| 服务端错误 (5xx) | 3 次 | 10s → 30s → 60s | 回退模型 |
| 认证错误 (401/403) | 1 次 | 立即 | 轮换凭证池 |
| 上下文溢出 | 1 次 | 立即 | 压缩后自动重试 |

**凭证池轮换**：当 API Key 触发 429 或 402 时，自动切换到池中下一个 Key：

```python
def _recover_with_credential_pool(self, status_code, ...):
    if status_code in (429, 402) and self._credential_pool:
        new_cred = self._credential_pool.rotate(self.provider)
        if new_cred:
            self._rebuild_client(api_key=new_cred)
            return True  # 继续重试
    return False
```

#### 7.7.3 提前终止

以下情况会触发 Agent Loop 提前终止：

| 条件 | 触发方式 | 行为 |
|------|----------|------|
| 用户中断 | `Ctrl+C` / `/stop` | 立即终止当前 loop |
| 迭代超限 | `max_turns` (默认 90) | 返回当前结果，提示用户 |
| 工具守卫拦截 | `ToolGuardrailController` | 阻断危险工具调用 |
| 模型沉默 | `_empty_content_retries > N` | 重试 3 次后放弃 |

```python
# 工具守卫示例 (agent/tool_guardrails.py)
class ToolCallGuardrailController:
    def check(self, tool_name, arguments):
        if tool_name in DANGEROUS_TOOLS:
            decision = self._classify_tool(tool_name, arguments)
            if decision == ToolGuardrailDecision.HALT:
                return "Tool call blocked: destructive pattern detected"
        return None  # 放行
```

---

### 7.8 本章小结

本章深入剖析了 Hermes Agent 的底层架构。核心要点：

1. **四层架构设计**：入口（Gateway）→ 控制（AIAgent）→ 执行（Agent Loop）→ 能力（Tool + Provider），层间通过标准接口解耦。

2. **Agent Loop 核心**：`run_conversation()` 是一个同步循环，每次迭代调用 LLM → 处理 tool_calls → 回注结果，直到 LLM 返回文本响应或迭代预算耗尽。

3. **系统提示词按10层组装**：从身份标识到环境信息，按顺序静态构建，缓存复用，仅在压缩事件后重建。

4. **工具系统自注册**：每个工具文件独立声明，通过 AST 分析自动发现。check_fn 控制可见性，TTL 缓存避免频繁检查。

5. **上下文压缩由事件驱动**：接近 Token 限制时自动触发，保护首尾信息，中间内容由辅助 LLM 摘要化。

6. **错误恢复多层化**：凭证池轮换、模型回退、上下文压缩、指数退避四层防御，确保系统韧性。

7. **流式输出支持实时处理**：Context Scrubber + Think Scrubber 双清洗通道，确保输出安全和格式正确。

---

**延伸阅读：** 继续学习第8章《高级特性》了解语音模式、API 服务器、MCP 集成和 RL 训练。遇到问题时参考第9章《排障与实战》。
