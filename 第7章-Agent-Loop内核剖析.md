## 第七章：Agent Loop 内核剖析

### 章节引言

理解 Hermes 的内核机制是成为高级用户和开发者的必经之路。本章将深入解析 Agent Loop 的核心循环结构、工具注册与调度系统、上下文管理机制和状态持久化原理。通过学习，你将能够理解 Hermes 如何处理用户请求、如何调用工具、如何管理对话历史，以及如何实现自动压缩。前置知识包括第三章的工具系统和第四章的消息网关。

---

### 7.1 消息完整生命周期

#### 7.1.1 端到端流程图

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

#### 7.1.2 四层架构详解

| 层级 | 组件 | 说明 |
|------|------|------|
| **Layer 1** | Gateway | 消息入口、协议归一化、会话管理 |
| **Layer 2** | Control | 提示词装配、上下文管理、流式输出 |
| **Layer 3** | Execution | 工具调度、环境抽象、安全审批 |
| **Layer 4** | Capability | 工具注册表、工具集、记忆系统 |

---

### 7.2 核心循环结构

#### 7.2.1 主循环代码

**源码**：`run_agent.py` 第 8464 行 `def run_conversation()`

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

### 7.3 工具注册机制

#### 7.3.1 自注册模式

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
        check_fn=check_web_available,  # 可选可用性检查
    )

# 模块导入时自动注册
register()
```

#### 7.3.2 ToolRegistry 类

```python
class ToolRegistry:
    """单例模式"""
    _instance = None
    _lock = threading.RLock()  # 线程安全

    _tools: Dict[str, ToolEntry] = {}  # name -> ToolEntry

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

#### 7.3.3 工具发现流程

```python
# model_tools.py
def discover_builtin_tools():
    """导入所有工具模块，触发自注册"""
    import tools.web_tools
    import tools.file_tools
    import tools.terminal_tool
    import tools.browser_tool
    # ... 更多工具
```

---

### 7.4 工具调度

#### 7.4.1 handle_function_call 入口

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

#### 7.4.2 参数类型强制转换

```python
def coerce_tool_args(tool_name: str, args: Dict) -> Dict:
    """LLM 返回 "42" 而不是 42，自动转换类型"""
    schema = registry.get_schema(tool_name)
    properties = schema["parameters"]["properties"]

    for key, value in args.items():
        if not isinstance(value, str):
            continue

        expected = properties.get(key, {}).get("type")

        if expected == "integer":
            args[key] = int(value)  # "42" -> 42
        elif expected == "number":
            args[key] = float(value)  # "3.14" -> 3.14
        elif expected == "boolean":
            args[key] = value.lower() in ("true", "1")  # "true" -> True

    return args
```

#### 7.4.3 异步事件循环管理

```python
# 避免 "Event loop is closed" 错误
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
        # 在 async 上下文中，使用线程池
        with concurrent.futures.ThreadPoolExecutor(max_workers=1) as pool:
            return pool.submit(asyncio.run, coro).result(timeout=300)

    tool_loop = _get_tool_loop()
    return tool_loop.run_until_complete(coro)
```

---

### 7.5 工具执行策略

#### 7.5.1 顺序执行 vs 并发执行

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
    # 工具分类
    parallel_safe = set(_PARALLEL_SAFE_TOOLS)
    never_parallel = set(_NEVER_PARALLEL_TOOLS)
    path_scoped = set(_PATH_SCOPED_TOOLS)

    # 按路径隔离的文件操作可并行
    path_groups = group_by_file_path(tool_calls, path_scoped)
    # ...
```

#### 7.5.2 工具分类

| 类别 | 工具 | 执行方式 |
|------|------|----------|
| **Parallel Safe** | `web_search`, `web_extract`, `read_file`, `session_search`, `vision_analyze` | 可并发 |
| **Never Parallel** | `clarify` | 必须顺序 |
| **Path Scoped** | `read_file`, `write_file`, `patch` | 按路径隔离并行 |
| **其他** | `terminal`, `delegate_task` 等 | 顺序执行 |

#### 7.5.3 execute_code 预算退款

```python
def execute_code(..., enabled_tools=None):
    """
    程序化工具调用（execute_code）执行后，
    允许退还迭代预算，避免消耗过多。
    """
    result = _execute_code_impl(...)

    # 退还预算
    if iteration_budget:
        iteration_budget.refund()

    return result
```

---

### 7.6 上下文管理

#### 7.6.1 消息准备

```python
def _prepare_api_messages(messages: List[Dict]) -> List[Dict]:
    """构建发送给 LLM 的消息"""
    api_messages = []

    # 1. System Prompt
    api_messages.append({"role": "system", "content": build_system_prompt()})

    # 2. 记忆上下文（如果启用）
    if self.memory_enabled:
        memory_context = self._get_memory_context(messages)
        api_messages.append({"role": "system", "content": memory_context})

    # 3. 插件上下文
    if plugin_context := invoke_hook("get_context", ...):
        api_messages.append({"role": "system", "content": plugin_context})

    # 4. 历史消息
    api_messages.extend(messages)

    return api_messages
```

#### 7.6.2 上下文压缩

```python
class ContextCompressor:
    """上下文压缩器"""

    def __init__(self, threshold=0.5, target_ratio=0.2):
        self.threshold = threshold      # 50% 时触发
        self.target_ratio = target_ratio  # 压缩到 20%
        self.protect_last_n = 20       # 保留最近 20 条

    def should_compress(self, messages, max_tokens) -> bool:
        """判断是否需要压缩"""
        used_ratio = count_tokens(messages) / max_tokens
        return used_ratio > self.threshold

    def compress(self, messages) -> List[Dict]:
        """执行压缩"""
        # 1. 保留 system prompt
        # 2. 保留最近 N 条消息
        # 3. 中间消息 LLM 摘要后保留
        # 4. 更新 parent_session_id 追踪链
```

#### 7.6.3 压缩流程

```bash
压缩前: [S, M1, M2, M3, M4, ..., M100, M101, M102]
                 ↓
压缩后: [S, SUMMARY(M1-M100), M101, M102]
                              ↑
                        新会话开始
```

---

### 7.7 状态管理

#### 7.7.1 SQLite 数据库结构

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

#### 7.7.2 FTS5 全文搜索

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

#### 7.7.3 会话链追踪

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

### 7.8 事件驱动模型

#### 7.8.1 插件钩子系统

```python
# hermes_cli/plugins.py
HOOKS = [
    "pre_tool_call",       # 工具调用前
    "post_tool_call",      # 工具调用后
    "transform_tool_result",  # 结果转换
    "on_message",         # 消息接收
    "on_response",        # 响应发送
    "get_context",        # 获取额外上下文
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

#### 7.8.2 钩子使用示例

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

### 7.9 错误处理与重试

#### 7.9.1 API 调用重试

```python
def _streaming_api_call_with_retry(...)
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

#### 7.9.2 工具执行错误处理

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
        return json.dumps({
            "error": f"Unexpected error: {str(e)}",
        })
```

---

### 7.10 本章小结

#### 核心概念总结

| 概念 | 源码位置 | 说明 |
|------|----------|------|
| Agent Loop | `run_agent.py:8464` | 主循环：请求→LLM→工具→响应 |
| 迭代预算 | `IterationBudget` | 控制最大迭代次数，支持退款 |
| 工具注册 | `tools/registry.py` | 自注册模式，check_fn 可用性检查 |
| 工具调度 | `model_tools.py` | handle_function_call 统一入口 |
| 参数转换 | `coerce_tool_args()` | 字符串自动转 int/bool |
| 上下文压缩 | `ContextCompressor` | 超过阈值时摘要历史 |
| 状态持久化 | `hermes_state.py` | SQLite + FTS5 |
| 插件钩子 | `plugins.py` | pre/post_tool_call 等 |

#### 关键源码文件

| 文件 | 功能 |
|------|------|
| `run_agent.py` | Agent Loop 主循环、消息处理、流式输出 |
| `model_tools.py` | 工具发现、调度、参数转换 |
| `tools/registry.py` | 工具注册表、单例模式 |
| `hermes_state.py` | SQLite 状态、消息持久化、FTS5 |
| `hermes_cli/plugins.py` | 插件系统、钩子管理 |

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

*源码索引：`run_agent.py`、`tools/registry.py`、`model_tools.py`、`hermes_state.py`*