# Agent Trace

> 分类：Observability / Agent 可观测性
> 相关课程：第一章 2-3 流式输出；1-6 LLM Gateway；第二章 2-4 工具治理；第三章 3-7 人工审批
> 整理日期：2026-11-05

## 定义

Trace（链路追踪 / 运行轨迹）是**一次 Run 从开始到结束的完整执行记录**。它用 `trace_id` 把模型调用、工具调用、重试、Checkpoint、错误等串成一条有因果关系的链路，回答：

> 这次任务到底发生了什么？为什么慢、为什么贵、为什么失败？

一句话：

> **Trace 是 Agent 的黑匣子 / 行车记录仪。**

### 先分清两个 “trace”

| 含义 | 英文 | 例子 |
| --- | --- | --- |
| 调用栈 | Stack Trace / Traceback | Python 报错时打印的一层层函数调用 |
| 链路追踪 | Distributed Tracing / Run Trace | 一次 Agent Run 从开始到结束的完整执行记录 |

课程里说的“Trace + 失败协议”“Trace 14 字段”，都是第二种。

## 一条 Agent Trace 的结构

```text
trace_id = trace_20261105_001
run_id   = run_abc123

└─ POST /v1/runs                      12 ms
   └─ agent.run                       8.4 s
      ├─ model.call #1                2.1 s
      │  ├─ TTFT                     1.2 s
      │  ├─ input_tokens             820
      │  └─ output_tokens            96
      ├─ tool.call get_order          120 ms
      │  ├─ args                     {"order_id":"ord_1002"}
      │  └─ status                   ok
      ├─ model.call #2                3.4 s
      ├─ checkpoint.save              15 ms
      └─ run.completed                8.5 s
```

核心概念：

| 概念 | 含义 |
| --- | --- |
| `trace_id` | 一次完整链路的唯一标识；跨服务、跨日志都靠它关联 |
| `span` | 链路中的一个操作，例如一次模型调用、一次工具调用、一次数据库查询 |
| `span_id` | 当前操作的唯一标识 |
| `parent_span_id` | 父操作标识，用来还原调用树 |
| `name` | 操作名称，如 `model.call`、`tool.call`、`checkpoint.save` |
| `start_time` / `end_time` | 操作开始与结束时间，用于计算耗时 |
| `attributes` | 关键属性：模型版本、Prompt 版本、Token、工具名、错误码等 |
| `status` | 成功 / 失败 / 取消 |
| `events` | 操作内部的关键事件，如重试、超时、审批 |

一条 Trace 是一棵树；每个 Span 是树上的一个节点。`trace_id` 相同，`parent_span_id` 决定父子关系。

## Trace、Log、Metric、Audit、Event Store 的区别

| 类型 | 回答的问题 | 形态 |
| --- | --- | --- |
| **Log** | 某一刻发生了什么？ | 一条条离散日志 |
| **Metric** | 整体趋势如何？ | 聚合数字：QPS、P95 延迟、Token 成本 |
| **Trace** | **这一次请求/任务为什么这样？** | 有 `trace_id`、父子关系的调用链 |
| **Audit** | 谁在什么时候做了什么？是否合规？ | 审计记录：主体、动作、结果、审批 |
| **Event Store** | 任务经过哪些业务事实？能否重放？ | `RunEvent`：`run.started`、`text.delta`、`tool.completed` 等 |

可以这样理解：

- **Log 是原料**；
- **Metric 是统计**；
- **Trace 是案件卷宗**；
- **Audit 是合规档案**；
- **Event Store 是业务事实账本**。

### Trace 与 Event Store 的分工

| 对比项 | Event Store | Trace |
| --- | --- | --- |
| 目的 | 保存业务事实，支持重放和恢复 | 记录执行轨迹，支持诊断和分析 |
| 内容 | `RunEvent` 与 `seq` | 跨层元数据：耗时、Token、模型版本、错误码、Span |
| 是否是事实来源 | 是，重放以它为准 | 通常不是，主要用于观测 |
| 关注点 | 任务状态与事件顺序 | 性能、成本、因果链、失败定位 |

两者可以共享 `run_id` / `trace_id`，但职责不同：

- **Event Store 让任务“能恢复”**；
- **Trace 让人“能看懂为什么”**。

## 课程要求记录什么

课件要求 Trace 至少记录 14 个字段：

```text
trace_id · run_id · 模型版本 · Prompt 版本
首 Token 时间 · 总耗时 · Token 使用
重试次数 · 重连次数 · 取消延迟 · 工具调用 · checkpoint
最终状态 · 错误码
```

按用途分类：

| 类别 | 字段 | 用途 |
| --- | --- | --- |
| 身份 | `trace_id`、`run_id` | 把一次任务的所有记录串起来 |
| 版本 | 模型版本、Prompt 版本 | 出问题时知道用的是哪套配置 |
| 性能 | 首 Token 时间、总耗时 | 区分 TTFT、生成时间、端到端延迟 |
| 成本 | Token 使用 | 估算调用成本 |
| 可靠性 | 重试次数、重连次数、取消延迟 | 判断网络抖动、频繁重试、取消是否及时 |
| 执行 | 工具调用、checkpoint | 还原任务做到哪一步 |
| 结果 | 最终状态、错误码 | 判断成功、失败、取消及失败原因 |

## 为什么 Agent 项目特别需要 Trace

### 1. 定位问题

没有 Trace 时，你只知道：

> “Agent 不工作。”

有 Trace 后，你能定位到：

> “第 2 次模型调用返回了工具名 `get_order`，但参数校验失败；Runtime 拒绝执行；Loop 又重试了 3 次；最后预算耗尽。”

问题从“Agent 坏了”变成“哪一层、哪一步、什么原因”。

### 2. 分析性能

Agent 一次任务可能耗时几十秒。Trace 能拆开：

- 模型推理占多少？
- 工具执行占多少？
- 排队、重试、Checkpoint 占多少？
- TTFT 是多少？用户多久看到第一个字？

### 3. 控制成本

Trace 记录 Token 使用、模型、重试次数，可以回答：

- 这次任务为什么贵？
- 是 Prompt 太长，还是重试太多？
- 换模型后成本变化多少？

### 4. 评估质量

把 Trace 和 Prompt 版本、模型版本关联起来，才能做回归：

- 新 Prompt 上线后，成功率有没有下降？
- 某个模型是不是更容易乱调工具？
- 失败集中在哪些任务类型？

### 5. 支持恢复和审计

Trace 里的 `checkpoint`、`last_seq`、工具调用状态，能帮助判断：

- 任务断在哪里？
- 哪些步骤已经完成？
- 外部副作用有没有发生？
- 是否需要人工确认？

### 6. 支持安全与合规

Trace 可以记录谁在什么时候调用了什么工具、是否经过审批、结果如何。它是审计的基础之一，但不能替代专门的审计系统。

## 课程中的 Trace 例子

### 1. LLM Gateway 的 CallTrace

`course_code/week01/1-6/gateway.py`：

```python
class CallTrace(BaseModel):
    request_id: str
    timestamp: datetime
    requested_model: str
    actual_model: str | None = None
    prompt_name: str | None = None
    prompt_version: str | None = None
    input_tokens: int
    output_tokens: int
    cost_usd: float
    latency_ms: int
    attempts: int
    status: Literal["success", "failed"]
    error_code: str | None = None
```

它记录的是“一次模型调用”的 Trace；到了 Agent 项目，就把模型调用、工具调用、Checkpoint 等 Span 串成一棵更大的 Trace 树。

### 2. Tool Runtime 的 trace_id

`course_code/week02/2-4/tool_governance_demo.py` 的 `ExecutionContext` 携带 `trace_id`，审计记录可以按 `trace_id` 查询：

```python
@dataclass(frozen=True)
class ExecutionContext:
    trace_id: str
    user_id: str
    tenant_id: str
    permissions: frozenset[str]
    allowed_tools: frozenset[str]
    approval_id: str | None = None
    idempotency_key: str | None = None
```

这样一次工具调用链路上的决策记录、执行记录和错误记录都能关联起来。

### 3. Streaming Gateway 的 14 字段

课程 1-3 的流式作业要求记录：TTFT、总耗时、Token、重试、重连、取消延迟、工具调用、Checkpoint、最终状态、错误码等。这样一次 Run 的“慢、贵、失败、取消、恢复”都有证据。

## 最小实现

### 1. 生成并传递 trace_id

请求进入时生成 `trace_id`，传给模型调用、工具调用、数据库、Checkpoint：

```python
trace_id = new_trace_id()
ctx = ExecutionContext(trace_id=trace_id, ...)
```

### 2. 用 Span 包裹关键操作

```python
with span(trace_id, "model.call", model="deepseek-v4-flash"):
    response = await model.stream(messages)

with span(trace_id, "tool.call", tool="get_order"):
    result = await runtime.execute(call)
```

### 3. 记录属性

每个 Span 至少记录：

- `name`、`start_time`、`end_time`
- `status`、`error_code`
- 模型版本、Prompt 版本
- Token 使用
- 工具名、参数摘要、结果状态
- 重试次数、Checkpoint 版本

### 4. 存储与查询

- 小项目：JSON 日志，按 `trace_id` 检索。
- 中等项目：SQLite / PostgreSQL 表。
- 分布式系统：OpenTelemetry + Jaeger / Tempo / Zipkin。

### 5. 采样与隐私

- **失败和慢请求优先全量记录**；成功请求可以按比例采样。
- **不要记录 API Key、完整 Prompt、用户隐私**；需要时做脱敏或哈希。
- **记录版本**：模型版本、Prompt 版本、代码版本。
- **Trace 要能按 `trace_id` / `run_id` 查询**，否则等于没记。

## 常见误区

1. **把 Trace 当成 Stack Trace**：课程说的是执行链路，不是异常调用栈。
2. **把 Trace 当成 Log**：日志是离散的；Trace 强调一次任务内部的因果关系和耗时。
3. **把 Trace 当成 Metric**：Metric 是聚合趋势；Trace 是单次请求的案卷。
4. **把 Trace 当成 Audit**：Audit 关注“谁做了什么、是否合规”；Trace 还包含性能、成本、调试信息。
5. **只记录成功/失败**：没有中间 Span，就不知道慢在哪、错在哪。
6. **没有 `trace_id`**：跨服务、跨日志无法关联。
7. **记录敏感信息**：API Key、完整 Prompt、用户隐私不应进入 Trace。
8. **没有版本信息**：出问题时无法判断是模型、Prompt 还是代码变化导致的。

## 相关条目

- [Agent Prompt Engineering](./agent_prompt_engineering.md)：Prompt 版本是 Trace 的关键字段
- [SSE](./sse.md)：流式事件传输协议
- [Pydantic](./pydantic.md)：Trace 记录的结构化建模
