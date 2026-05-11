## 第七章：Agent Loop 内核剖析

> 本章定位：第三部分·精通 | 预估篇幅：~4000行 | 前置阅读：第1-3章

**内容导读**

```
○ 7.1 请求流转
  - ○ 7.1.1 消息完整生命周期追踪
  - ○ 7.1.2 四层架构详解
  - ○ 7.1.3 端到端消息链路图
  - ○ 7.1.4 分层排障策略

○ 7.2 核心循环结构
  - ○ 7.2.1 主循环代码解析
  - ○ 7.2.2 迭代预算控制

○ 7.3 嵌入式集成
  - ○ 7.3.1 Hermes 如何嵌入 LLM 运行时
  - ○ 7.3.2 工具注入机制
  - ○ 7.3.3 事件订阅与插件钩子

○ 7.4 事件驱动模型
  - ○ 7.4.1 事件总线设计
  - ○ 7.4.2 Tick 机制
  - ○ 7.4.3 状态恢复

○ 7.5 入口与排队
  - ○ 7.5.1 协议归一化
  - ○ 7.5.2 幂等控制（防重放）
  - ○ 7.5.3 分道排队（Session/Global/Background）
  - ○ 7.5.4 反压与超时

○ 7.6 提示词装配
  - ○ 7.6.1 动态 System Prompt
  - ○ 7.6.2 上下文预算与裁剪
  - ○ 7.6.3 注入防御

○ 7.7 工具执行
  - ○ 7.7.1 工具注册机制
  - ○ 7.7.2 工具调度与参数转换
  - ○ 7.7.3 执行策略：顺序与并发
  - ○ 7.7.4 结果回注与裁剪

○ 7.8 流式输出
  - ○ 7.8.1 Block Chunker
  - ○ 7.8.2 重试策略
  - ○ 7.8.3 错误处理
  - ○ 7.8.4 提前终止

○ 7.9 状态管理
  - ○ 7.9.1 SQLite 数据库结构
  - ○ 7.9.2 FTS5 全文搜索
  - ○ 7.9.3 会话链追踪

○ 7.10 本章小结
```

---

### 7.1 请求流转

#### 7.1.1 消息完整生命周期追踪

每条用户消息进入 Hermes 后，经历一条从入口到输出的完整流水线。理解这条流水线是调优和排障的基础。

**消息生命周期时序**

```
用户输入
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Gateway Layer                             │
│  Telegram/Discord/Slack/飞书 → 协议归一化 → NormalizedMessage    │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Session Layer                               │
│  历史消息加载 → 上下文组装 → sanities_message() 清理            │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Agent Loop (核心循环)                       │
│  while 迭代预算剩余:                                             │
│    ├── _prepare_api_messages()     注入记忆/插件上下文          │
│    ├── streaming_api_call()        调用 LLM                     │
│    ├── tool_calls 存在?            是→执行工具                   │
│    │     ├── _execute_tool_calls() 顺序/并发执行                │
│    │     ├── 上下文压缩检查        超过阈值触发压缩              │
│    │     └── continue              继续下一轮                   │
│    └── tool_calls 空?             是→返回最终响应               │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
流式输出 → 返回上游平台
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

#### 7.1.2 四层架构详解

Hermes 的四层架构设计使其能够支持 15+ 消息平台、20+ LLM 供应商、80+ 内置工具，而核心逻辑保持单一。

| 层级 | 组件 | 说明 |
|------|------|------|
| **Layer 1：入口层** | Gateway | 消息入口、协议归一化、会话管理 |
| **Layer 2：控制层** | AIAgent | 提示词装配、上下文管理、流式输出 |
| **Layer 3：执行层** | Agent Loop | 工具调度、环境抽象、安全审批 |
| **Layer 4：能力层** | Tool + Provider | 工具注册表、工具集、记忆系统 |

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

即 `run_conversation()` 方法中的主循环。详见 7.2 节。

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

### 7.2 核心循环结构

#### 7.2.1 主循环代码解析

**源码**：`run_agent.py` 第 8464 行 `def run_conversation()`

Agent Loop 是 Hermes 的"心脏"——一个同步循环，每次迭代调用 LLM、处理工具调用、回注结果，直到 LLM 返回文本响应或迭代预算耗尽。

```python
while (api_call_count < self.max_iterations and self.iteration_budget.remaining > 0) \
      or self._budget_grace_call:

    # 1. 检查中断请求
    if self._interrupt_requested:
        interrupted = True
        break

    # 2. 消耗迭代预算
    api_call_count += 1
    if not self.iteration_budget.consume():
        break

    # 3. 构建 API 请求消息
    api_messages = self._prepare_api_messages(messages)

    # 4. 调用模型 API（可中断流式）
    response = self._interruptible_streaming_api_call(api_kwargs)

    # 5. 处理响应
    if assistant_message.tool_calls:
        # 5a. 执行工具调用
        self._execute_tool_calls(assistant_message, messages, effective_task_id)

        # 5b. 上下文压缩检查
        if self.compression_enabled and should_compress(...):
            messages = self._compress_context(...)

        # 5c. 继续循环
        continue
    else:
        # 5d. 最终响应
        final_response = assistant_message.content
        break
```

#### 7.2.2 迭代预算控制

```python
class IterationBudget:
    """管理最大迭代次数"""
    total: int           # 总预算
    remaining: int       # 剩余次数
    _refunds: int        # 退款计数

    def consume(self) -> bool:
        """消耗一次迭代，返回是否还有剩余"""
        if self.remaining <= 0:
            return False
        self.remaining -= 1
        return True

    def refund(self):
        """execute_code 等程序化工具调用可退还预算"""
        if self._refunds < self.total:
            self._refunds += 1
            self.remaining = min(self.remaining + 1, self.total)
```

**默认配置**：
- `max_iterations`: 90
- `budget_grace_call`: 1（允许超出预算一次返回最终响应）

---

### 7.3 嵌入式集成

#### 7.3.1 Hermes 如何嵌入 LLM 运行时

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

#### 7.3.2 工具注入机制

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

#### 7.3.3 事件订阅与插件钩子

Hermes 支持插件系统，通过事件钩子（Event Hook）机制扩展功能。

**钩子系统**：

```python
# hermes_cli/plugins.py
HOOKS = [
    "pre_tool_call",           # 工具调用前
    "post_tool_call",          # 工具调用后
    "transform_tool_result",   # 结果转换
    "on_message",              # 消息接收
    "on_response",             # 响应发送
    "get_context",             # 获取额外上下文
    "on_session_start",        # 全新会话创建时
    "on_tool_complete",        # 工具执行完成后
    "on_message_send",         # 消息投递前
]

def invoke_hook(hook_name, **kwargs):
    """调用所有注册的钩子"""
    results = []
    for plugin in plugins:
        if hook_name in plugin.hooks:
            result = plugin.hooks[hook_name](**kwargs)
            if result:
                results.append(result)
    return results
```

**插件注册方式**：

```python
# 在 plugin.py 中
from hermes_cli.plugins import register_hook

@register_hook("on_session_start")
def my_hook(session_id, model, platform):
    print(f"Session started: {session_id}")
```

**钩子使用示例**：

```python
class AuditPlugin:
    """审计插件示例"""
    hooks = ["pre_tool_call", "post_tool_call"]

    def pre_tool_call(self, tool_name, args, **kwargs):
        """记录工具调用"""
        log(f"Tool: {tool_name}, Args: {args}")

    def post_tool_call(self, tool_name, args, result, **kwargs):
        """记录结果"""
        log(f"Result: {result[:100]}...")

# 注册插件
register_plugin(AuditPlugin())
```

---

### 7.4 事件驱动模型

#### 7.4.1 事件总线设计

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

#### 7.4.2 Tick 机制

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

#### 7.4.3 状态恢复

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

### 7.5 入口与排队

#### 7.5.1 协议归一化

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

#### 7.5.2 幂等控制（防重放）

消息平台（特别是 Webhook 模式）可能重复投递同一条消息。Hermes 通过以下机制防止重复处理：

| 机制 | 实现 | 适用场景 |
|------|------|----------|
| 消息 ID 去重 | `update_id` 缓存 | Telegram Webhook |
| 幂等 Key | 计算消息哈希 | Discord |
| 时间窗口 | 5秒内相同内容丢弃 | 所有平台 |

#### 7.5.3 分道排队（Session/Global/Background）

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

#### 7.5.4 反压与超时

当系统负载超过容量时，反压机制防止雪崩：

| 反压策略 | 实现 | 触发条件 |
|----------|------|----------|
| 背压（Backpressure） | 队列满时拒绝新请求 | 队列容量 > 100 |
| 超时（Timeout） | API 调用可配超时 | `timeout: 180` (默认) |
| 限流（Rate Limit） | 每分钟请求数限制 | `requests_per_minute: 60` |

---

### 7.6 提示词装配

#### 7.6.1 动态 System Prompt

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

#### 7.6.2 上下文预算与裁剪

上下文预算管理是 Agent 长时间运行的关键。Hermes 使用 `ContextCompressor` 类（`agent/context_compressor.py`）自动管理上下文。

**压缩触发条件**：

| 条件 | 阈值 | 行为 |
|------|------|------|
| 接近上下文限制 | 达到 `max_context_tokens` 的 70% | 自动提示压缩 |
| 上下文即将溢出 | 超过 `context_compression_threshold` | 自动执行压缩 |
| 手动触发 | `/compress` 命令 | 用户主动压缩 |

**压缩算法**：

```python
class ContextCompressor:
    def __init__(self, threshold=0.5, target_ratio=0.2):
        self.threshold = threshold        # 50% 时触发
        self.target_ratio = target_ratio  # 压缩到 20%
        self.protect_last_n = 20         # 保留最近 20 条

    def should_compress(self, messages, max_tokens) -> bool:
        """判断是否需要压缩"""
        used_ratio = count_tokens(messages) / max_tokens
        return used_ratio > self.threshold

    def compress(self, messages, max_tokens):
        """执行压缩"""
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

**压缩流程示意**：

```
压缩前: [S, M1, M2, M3, M4, ..., M100, M101, M102]
                 ↓
压缩后: [S, SUMMARY(M1-M100), M101, M102]
                              ↑
                        新会话开始
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

#### 7.6.3 注入防御

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

### 7.7 工具执行

#### 7.7.1 工具注册机制

**自注册模式**

**源码**：`tools/registry.py`

每个工具文件在模块级别调用 `register()`：

```python
# tools/web_tools.py
def register():
    registry.register(
        name="web_search",
        toolset="web",
        schema={
            "type": "function",
            "function": {
                "name": "web_search",
                "description": "搜索网络获取信息",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "query": {"type": "string"},
                        "num_results": {"type": "integer", "default": 10}
                    }
                }
            }
        },
        handler=web_search_handler,
        check_fn=check_web_available,
    )

# 模块导入时自动注册
register()
```

**ToolRegistry 单例**：

```python
class ToolRegistry:
    """单例模式"""
    _instance = None
    _lock = threading.RLock()  # 线程安全
    _tools: Dict[str, ToolEntry] = {}

    def register(self, name, toolset, schema, handler, check_fn=None):
        with self._lock:
            self._tools[name] = ToolEntry(...)

    def dispatch(self, name, args, **kwargs):
        tool = self._tools.get(name)
        if not tool:
            raise ToolNotFoundError(name)
        return tool["handler"](args, **kwargs)

    def get_definitions(self, tool_names):
        """返回工具 schema 列表（仅包含 check_fn 通过的工具）"""
        return [t.schema for t in self._tools.values()
                if t.name in tool_names and t.check_fn()]
```

**工具发现流程**：`tools/registry.py` 的 `discover_builtin_tools()` 扫描 `tools/` 目录下所有 `.py` 文件，通过 AST 分析检测哪些文件包含顶层 `registry.register()` 调用，然后动态 import：

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

#### 7.7.2 工具调度与参数转换

**handle_function_call 入口**

**源码**：`model_tools.py`

```python
def handle_function_call(
    function_name: str,
    function_args: Dict[str, Any],
    task_id: Optional[str] = None,
    tool_call_id: Optional[str] = None,
    session_id: Optional[str] = None,
    user_task: Optional[str] = None,
    enabled_tools: Optional[List[str]] = None,
    skip_pre_tool_call_hook: bool = False,
) -> str:
    # 1. 类型强制转换（"42" -> 42）
    function_args = coerce_tool_args(function_name, function_args)

    # 2. 插件钩子检查（pre_tool_call）
    if not skip_pre_tool_call_hook:
        if block_message := get_pre_tool_call_block_message(...):
            return json.dumps({"error": block_message})

    # 3. 调用注册表分发
    if function_name == "execute_code":
        result = registry.dispatch(..., enabled_tools=sandbox_enabled)
    else:
        result = registry.dispatch(function_name, function_args, task_id=task_id)

    # 4. 后置钩子（post_tool_call, transform_tool_result）
    invoke_hook("post_tool_call", ...)
    if transformed := invoke_hook("transform_tool_result", ...):
        result = transformed

    return result
```

**参数类型强制转换**：LLM 有时返回字符串 `"42"` 而不是整数 `42`，自动转换：

```python
def coerce_tool_args(tool_name: str, args: Dict) -> Dict:
    schema = registry.get_schema(tool_name)
    properties = schema["parameters"]["properties"]

    for key, value in args.items():
        if not isinstance(value, str):
            continue
        expected = properties.get(key, {}).get("type")
        if expected == "integer":
            args[key] = int(value)
        elif expected == "number":
            args[key] = float(value)
        elif expected == "boolean":
            args[key] = value.lower() in ("true", "1")
    return args
```

**异步事件循环管理**：避免 "Event loop is closed" 错误

```python
_tool_loop = None
_tool_loop_lock = threading.Lock()

def _get_tool_loop():
    """返回持久化事件循环"""
    global _tool_loop
    with _tool_loop_lock:
        if _tool_loop is None or _tool_loop.is_closed():
            _tool_loop = asyncio.new_event_loop()
        return _tool_loop

def _run_async(coro):
    """从同步上下文运行异步协程"""
    try:
        loop = asyncio.get_running_loop()
    except RuntimeError:
        loop = None

    if loop and loop.is_running():
        with concurrent.futures.ThreadPoolExecutor(max_workers=1) as pool:
            return pool.submit(asyncio.run, coro).result(timeout=300)

    tool_loop = _get_tool_loop()
    return tool_loop.run_until_complete(coro)
```

#### 7.7.3 执行策略：顺序与并发

当 LLM 一次返回多个 `tool_calls` 时，Hermes 可以并行执行它们：

```python
def _execute_tool_calls_sequential(tool_calls, ...):
    """顺序执行所有工具调用"""
    for tool_call in tool_calls:
        result = _invoke_tool(function_name, function_args, task_id, ...)
        messages.append({
            "role": "tool",
            "tool_call_id": tool_call.id,
            "content": result
        })

def _execute_tool_calls_concurrent(tool_calls, ...):
    """自动并行化只读工具"""
    parallel_safe = set(_PARALLEL_SAFE_TOOLS)
    never_parallel = set(_NEVER_PARALLEL_TOOLS)
    path_scoped = set(_PATH_SCOPED_TOOLS)
    path_groups = group_by_file_path(tool_calls, path_scoped)
    # ...
```

**工具分类与并行策略**：

| 类别 | 工具 | 执行方式 |
|------|------|----------|
| **Parallel Safe** | `web_search`, `web_extract`, `read_file`, `session_search`, `vision_analyze` | 无限制并行 |
| **Never Parallel** | `clarify` | 强制串行 |
| **Path Scoped** | `read_file`, `write_file`, `patch` | 路径不冲突时并行 |
| **其他** | `terminal`, `delegate_task` 等 | 默认串行 |

```python
_MAX_TOOL_WORKERS = 8  # 最大并行线程数

def _should_parallelize_tool_batch(tool_calls):
    names = [tc.function.name for tc in tool_calls]
    if any(name in _NEVER_PARALLEL_TOOLS for name in names):
        return False
    if all(name in _PARALLEL_SAFE_TOOLS for name in names):
        return True
    if all(name in _PATH_SCOPED_TOOLS for name in names):
        paths = [_extract_parallel_scope_path(n, a) for n, a in ...]
        return not _any_paths_overlap(paths)
    return False
```

**execute_code 预算退款**：程序化工具执行后允许退还迭代预算

```python
def execute_code(..., enabled_tools=None):
    result = _execute_code_impl(...)
    if iteration_budget:
        iteration_budget.refund()
    return result
```

#### 7.7.4 结果回注与裁剪

**结果回注**：工具执行完成后，结果被组装成 tool 角色消息，追加到消息列表：

```python
tool_result = handle_function_call(tool_name, arguments)
messages.append({
    "role": "tool",
    "tool_call_id": tool_call.id,
    "content": tool_result
})
# 继续 loop，LLM 看到之前调用的结果
```

**大尺寸结果裁剪**：工具返回结果可能非常庞大，通过以下机制控制：

| 机制 | 阈值 | 行为 |
|------|------|------|
| 全局限制 | `max_tool_result_size_chars` | 超过后截断 |
| 工具级限制 | `max_result_size_chars` | 单工具自定义上限 |
| Turn Budget | `enforce_turn_budget()` | 工具结果占用超过阈值时触发压缩 |

```python
# tools/tool_result_storage.py
def maybe_persist_tool_result(result: str, max_size: int = 50000) -> str:
    if len(result) > max_size:
        return result[:max_size] + "\n[Tool result truncated...]"
    return result
```

---

### 7.8 流式输出

#### 7.8.1 Block Chunker

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

#### 7.8.2 重试策略

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
            return True
    return False
```

#### 7.8.3 错误处理

**API 调用重试**：

```python
def _streaming_api_call_with_retry(...):
    max_retries = 3
    base_delay = 1

    for attempt in range(max_retries):
        try:
            return _streaming_api_call(...)
        except RateLimitError:
            delay = base_delay * (2 ** attempt)
            time.sleep(delay)
        except APIError as e:
            if not e.is_retryable():
                raise
            delay = base_delay * (2 ** attempt)
            time.sleep(delay)

    raise MaxRetriesExceeded()
```

**工具执行错误处理**：

```python
def handle_function_call(...):
    try:
        result = registry.dispatch(...)
    except ToolExecutionError as e:
        return json.dumps({
            "error": f"Tool execution failed: {str(e)}",
            "tool": function_name,
        })
    except Exception as e:
        logger.error(f"Unexpected error in {function_name}: {e}")
        return json.dumps({"error": f"Unexpected error: {str(e)}"})
```

#### 7.8.4 提前终止

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

### 7.9 状态管理

#### 7.9.1 SQLite 数据库结构

**源码**：`hermes_state.py`

```python
# 数据库表结构
TABLES = """
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    created_at INTEGER,
    updated_at INTEGER,
    parent_session_id TEXT,
    source TEXT,
    metadata TEXT
);

CREATE TABLE messages (
    id TEXT PRIMARY KEY,
    session_id TEXT,
    role TEXT,
    content TEXT,
    tool_calls TEXT,
    tool_results TEXT,
    created_at INTEGER,
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

CREATE VIRTUAL TABLE messages_fts USING fts5(
    content, session_id UNINDEXED
);
"""

# WAL 模式：支持并发读 + 单写
PRAGMA journal_mode=WAL
```

#### 7.9.2 FTS5 全文搜索

```python
def session_search(query: str, limit: int = 10):
    """跨会话全文搜索"""
    results = db.execute("""
        SELECT session_id, content,
               highlight(messages_fts, 0, '<mark>', '</mark>') as snippet
        FROM messages_fts
        WHERE messages_fts MATCH ?
        ORDER BY rank
        LIMIT ?
    """, (query, limit))
    return results
```

#### 7.9.3 会话链追踪

压缩触发的会话分割会创建父子会话链：

```python
# 压缩触发的会话分割
original_session_id = current_session.id

# 创建压缩后的新会话
new_session = Session(
    id=generate_session_id(),
    parent_session_id=original_session_id,  # 指向父会话
)

# 通过 parent_session_id 可以回溯完整历史
def get_session_chain(session_id):
    chain = []
    while session_id:
        session = db.get_session(session_id)
        chain.append(session)
        session_id = session.parent_session_id
    return list(reversed(chain))
```

---

### 7.10 本章小结

本章深入剖析了 Hermes Agent 的底层架构。核心要点：

1. **四层架构设计**：入口（Gateway）→ 控制（AIAgent）→ 执行（Agent Loop）→ 能力（Tool + Provider），层间通过标准接口解耦。

2. **Agent Loop 核心**：`run_conversation()` 是一个同步循环，每次迭代调用 LLM → 处理 tool_calls → 回注结果，直到 LLM 返回文本响应或迭代预算耗尽。

3. **系统提示词按10层组装**：从身份标识到环境信息，按顺序静态构建，缓存复用，仅在压缩事件后重建。

4. **工具系统自注册**：每个工具文件独立声明，通过 AST 分析自动发现。check_fn 控制可见性，TTL 缓存避免频繁检查。

5. **上下文压缩由事件驱动**：接近 Token 限制时自动触发，保护首尾信息，中间内容由辅助 LLM 摘要化。

6. **错误恢复多层化**：凭证池轮换、模型回退、上下文压缩、指数退避四层防御，确保系统韧性。

7. **流式输出支持实时处理**：Context Scrubber + Think Scrubber 双清洗通道，确保输出安全和格式正确。

#### 关键源码文件

| 文件 | 功能 |
|------|------|
| `run_agent.py` | Agent Loop 主循环、消息处理、流式输出 |
| `model_tools.py` | 工具发现、调度、参数转换 |
| `tools/registry.py` | 工具注册表、单例模式 |
| `hermes_state.py` | SQLite 状态、消息持久化、FTS5 |
| `hermes_cli/plugins.py` | 插件系统、钩子管理 |
| `agent/context_compressor.py` | 上下文压缩 |
| `agent/retry_utils.py` | 重试策略与退避算法 |
| `agent/tool_guardrails.py` | 工具守卫与安全控制 |

#### 调试命令

```bash
# 追踪完整请求流程
hermes logs --trace --request-id <id>

# 查看工具调用详情
hermes logs --filter "dispatch" --verbose

# 检查工具注册状态
hermes tools --check

# 模拟 Agent Loop
python run_agent.py --prompt "test"
```

---

**延伸阅读：** 继续学习第8章《高级特性》了解语音模式、API 服务器、MCP 集成和 RL 训练。遇到问题时参考第9章《排障与实战》。
