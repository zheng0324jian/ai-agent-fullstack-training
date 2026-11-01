# 第一章 2-3 课后作业：Streaming Agent Gateway

> 课程：极客时间《AI Agent 全栈工程师训练营》
> 对应课件：`course_materials/week01/1-3.pdf`（1.3 Streaming，第 45 页「课后练习」）
> 对应笔记：`doc/chapter1/内容大纲.md` 第 2 节「流式输出」2-3
> 参考代码：`course_code/week01/1-3/`
> 整理日期：2026-11-02

## 本目录内容

| 文件 | 用途 |
| --- | --- |
| `README.md` | 作业要求整理：目标、功能拆解、验收标准、常见坑 |
| `验收清单.md` | 逐项自查表（P0/P1/P2），可直接填结果与证据 |
| `实现路线.md` | 分阶段实施顺序、每阶段任务与完成标准 |
| `提交模板.md` | 作业项目 README 模板，复制到项目根目录使用 |

## 0. 一句话任务

把「同步调用模型、等全部生成完再一次性返回」的 Demo，升级成一条**可观察、可取消、可恢复、可审计**的 Agent 流式事件通道：

```text
Model Adapter → Agent Loop → Event Store → SSE Gateway → Web / CLI Client
```

作业的重点不是前端“打字机效果”，而是让一次 Agent Run 的执行过程成为**稳定、可重放、可恢复的事实流**。

## 1. 作业定位

### 1.1 要解决的问题

一次 Agent Run 可能经历「模型推理 → 文件检索 → 工具执行 → 再次推理 → 结果校验」多个阶段。如果只在最后返回一行 JSON，用户无法判断：

- 正在生成，还是正在排队？
- 正在调用工具，还是已经卡死？
- 仍在执行，还是早已失败？
- 方向正确，还是应该立即停止？

### 1.2 四条能力主线

| 能力 | 含义 | 最终落点 |
| --- | --- | --- |
| 可观察 | 事件可追踪 | `RunEvent` 协议、`seq`、SSE、TTFT |
| 可取消 | 显式终止 | Cancel API、`cancelling` 状态机、取消传播 |
| 可恢复 | Replay + Checkpoint | Event Store 重放、Checkpoint 续跑 |
| 可审计 | Trace + 失败协议 | 14 项 Trace、稳定错误码、测试证据 |

### 1.3 关键边界

- **Streaming ≠ 让模型更快**：它只改善感知延迟（TTFT），不减少推理时间。
- **SSE 是传输协议，Harness Event 是业务协议**：不要让客户端依赖供应商 SDK 的 chunk 结构。
- **断线 ≠ 取消**：关闭订阅只停止“看”，停止业务 Run 必须走 Cancel API。
- **文本结束 ≠ 任务完成**：`model.finished` 不是 `run.completed`，还要校验 `finish_reason`。
- **取消即存档点，不等于可执行点**：半截 JSON / SQL / Patch 可以重放，不能直接提交。

## 2. 课件原文要求（第 45 页）

> 课后练习：**Streaming Agent Gateway**
> 将同步模型调用升级为 · 可观察 · 可取消 · 可恢复 · 可审计。

| 维度 | 课件要求 |
| --- | --- |
| 目标 | 将同步模型调用升级为：可观察 · 可取消 · 可恢复 · 可审计 |
| 4 项能力升级 | 可观察（事件可追踪）· 可取消（显式终止）· 可恢复（Replay + Checkpoint）· 可审计（Trace + 失败协议） |
| 8 项系统能力 | 创建独立 Run · 流式调用模型 · 转换统一 Harness Event · SSE 推送 · 用户取消 · 断线恢复 · checkpoint 保存 · TTFT 统计 |
| 实现顺序 | 标题写「11 步」，页面实际列出 10 步（见第 4 节；第 11 步建议补端到端验收） |
| 验收标准 | 标题写「10 标准」，页面明示 5 条（见第 5 节；其余 5 条按课程正文补齐） |

## 3. 功能要求拆解

### 3.1 Run 生命周期与三个接口

课件要求把「创建 / 订阅 / 控制」拆成三个互不重叠的职责：

| 接口 | 职责 | 关键要求 |
| --- | --- | --- |
| `POST /v1/runs` | 创建任务并立即返回 `run_id` | 后台启动执行；课件要求立即 `202` 返回；不要等模型跑完才返回 |
| `GET /v1/runs/{run_id}/events` | SSE 持续推送事件 | 支持 `Last-Event-ID` 断线重连；重连不得重新发起模型请求 |
| `POST /v1/runs/{run_id}/cancel` | 显式停止任务 | 进入 `cancelling`；传播到 Loop / Model / Tool；最终写入 `run.cancelled` |

> 课件接口写作 `/runs`，参考代码使用 `/v1/runs`。作业里统一一种风格即可，建议保留 `/v1` 前缀。

**状态机要求：**

```text
created → running → completed
                  → failed
                  → cancelling → cancelled
```

- 一个 Run 只能有一个终态：`completed` / `failed` / `cancelled`。
- 取消不能从 `running` 直接跳到 `cancelled`，必须有 `cancelling` 过渡态。
- 对已经进入终态的 Run 再次取消，应幂等返回当前状态。
- 完成与取消发生竞态时，以第一条终态事件为准，不能出现两条终态。
- 客户端断开只结束订阅，不自动取消 Run。

### 3.2 Harness Event 协议

对外统一使用版本化、类型化的 `RunEvent`：

| 字段 | 类型/约束 | 作用 |
| --- | --- | --- |
| `schema_version` | `Literal["1"]` | 协议演化护栏；未知类型时旧客户端不崩溃 |
| `run_id` | `str` | Run 标识 |
| `seq` | `int >= 0`，Run 内单调递增 | 排序、去重、重放、Trace 的稳定位置键 |
| `type` | `EventType` | 事件类型；编译期可发现拼写错误 |
| `created_at` | `datetime` | 客户端重排、计算 E2E 延迟 |
| `data` | `dict[str, Any]` | 可演化的业务负载，不破坏旧客户端 |

**对外 8 类事件（以课件第 14 页为准）：**

```text
run.started
text.delta
tool.started
tool.completed
run.retrying
run.completed
run.failed
run.cancelled
```

**口径说明：**

- `model.finished` / `model.usage` 是 **Model Adapter 的内部事件**，不要直接当成对外 RunEvent；需要由 Agent Loop 转换成 `run.completed` / `run.failed` 等业务事实。
- 课件其他页出现过的 `task.started`、`message.completed` 属于早期/概念命名；落地时统一到上面的 8 类，避免同一语义两套事件。
- 未知事件类型应被忽略或降级，而不是让客户端崩溃。
- 心跳 `: heartbeat` 是 SSE 注释行，**不得占用业务 `seq`**。

### 3.3 SSE 编解码与客户端消费

**服务端编码要求：**

- `Content-Type: text/event-stream`，UTF-8。
- 每条事件包含 `id:`（用 `seq`）、`event:`、`data:`，以空行结束。
- 心跳使用注释行，例如 `: heartbeat\n\n`。
- 建议响应头：`Cache-Control: no-cache, no-transform`、`X-Accel-Buffering: no`、`Connection: keep-alive`。
- 把 `encode_sse` / `encode_heartbeat` 写成**纯函数**，单独做单元测试。

**客户端消费要求：**

- 不能假设 `1 Token = 1 SSE = 1 Chunk`；一次 `read()` 可能只有半条事件，也可能包含多条事件。
- 正确姿势：`bytes → buffer → lines → SSE frames → RunEvent`。
- 使用 `TextDecoder(..., { stream: true })` 处理 UTF-8 边界。
- 按 `seq` 去重；保存 `Last-Event-ID`，重连时回传。
- 渲染节流：网络层可以高频收，UI 层用 `requestAnimationFrame` 或 debounce 合并刷新。
- `EventSource` 适合简单 GET 订阅；需要 POST、`Authorization` 头、精细错误处理时用 `fetch + ReadableStream`。

**编码器至少覆盖 6 项测试（TC01–TC06）：**

| 编号 | 用例 |
| --- | --- |
| TC01 | 中文/Unicode 往返不丢字 |
| TC02 | 换行在 payload 内部正确转义 |
| TC03 | 引号、反斜杠不破坏 SSE 结构 |
| TC04 | 空字符串 delta 不漂移 `seq` |
| TC05 | 大 Payload（> 64 KB）单事件 |
| TC06 | 拆包/粘包任意字节切分都能恢复 |

### 3.4 Model Adapter

用 `Protocol` 把模型供应商差异挡在 Adapter 层：

```python
@dataclass(frozen=True)
class ModelStreamEvent:
    type: str
    data: dict[str, Any]

class StreamingModel(Protocol):
    async def stream(
        self,
        messages: list[dict[str, str]],
    ) -> AsyncIterator[ModelStreamEvent]: ...
```

**Adapter 必须处理的 4 类边界：**

1. Chunk 可能没有 `choices`（例如末尾的 usage-only chunk）。
2. `delta.content` 可能为 `None`（只有 role 元数据），应丢弃。
3. 文本结束 ≠ 业务任务完成；`model.finished` ≠ `run.completed`。
4. Harness 仍需校验 `finish_reason`：`length` / `content_filter` 需要业务层处理。

**其他要求：**

- `AsyncOpenAI(..., max_retries=0)` 关闭 SDK 自动重试，由 Harness 决定何时、重试几次。
- Adapter 只产生内部 `ModelStreamEvent`，不拼 SSE 字符串、不渲染 UI。
- 切换模型只替换 Adapter，Agent Loop、Event Store、Client 不改。
- 不要把供应商原始 chunk 直接透传给前端，避免泄露内部字段和绑定具体 SDK。

### 3.5 Run Store 与 Agent Loop

**Run Store 的职责：**

- 任务状态不能依赖单次 HTTP 请求的生命周期。
- 保存 Run 状态、事件列表、事件序号、取消信号。
- 支撑 4 项能力：断线恢复、事件订阅、任务取消、状态查询。
- 课堂实现可以先用**内存 Run Store** 打通协议，再接 PostgreSQL + Redis。

**Agent Loop 的职责：**

- 只负责驱动任务、调用工具、产生业务事实，不直接操作 HTTP。
- 不拼 SSE、不管理连接、不决定客户端怎么渲染。
- 产生的事件至少覆盖：`run.started`、`text.delta`、`tool.started`、`tool.completed`、`run.completed`、`run.failed`、`run.cancelled`。
- `CancelledError` 是控制信号，不是普通异常，必须单独处理，不能被 `except Exception` 吞掉。

**六层责任对位：**

| 组件 | 负责 | 不负责 |
| --- | --- | --- |
| Model Adapter | 模型流转换、错误归一化 | UI 输出、SSE 编码 |
| Agent Loop | 执行任务、调工具、产生事件 | HTTP 连接、页面展示 |
| Tool Runtime | 工具执行、权限与超时 | 页面展示、模型解析 |
| Event Store | 保存事件、排序、重放 | 重新执行模型 |
| SSE Gateway | 编码、心跳、订阅管理 | 决定任务是否完成 |
| Client | 展示、去重、取消、重连 | 模型调用、工具执行、重试逻辑 |

### 3.6 取消

**两种取消方式：**

- **协作式取消（优先）**：组件在每一步 `await` 之前检查取消信号；安全、可保存状态、可释放资源。
- **强制 Task Cancel**：用于长时间阻塞任务；中断点不确定，副作用可能未保存。

**生产级取消流程：**

1. 设置取消状态（`cancelling`）。
2. 等待清理。
3. 未退出则强制取消。
4. 终止外部副作用（如工具进程）。
5. 超时记录。

**取消状态机：**

```text
running → cancelling → cancelled
```

- 不要直接从 `running` 写 `cancelled`。
- 用 CAS 更新，避免完成与取消竞争：`WHERE status IN ('created','running')`。
- 取消必须贯穿 Client → Loop → Model → Tool 四级。
- `AbortController` 只能停止当前 HTTP 读取，不能保证后台 Agent 停止计费；必须调用 Cancel API。

**取消时必须保存的 6 项：**

1. 已输出文本
2. 最后事件序号 `last_seq`
3. 当前 Loop 步骤
4. 工具状态（`call_id`、是否已发出）
5. Token 使用量
6. 取消原因（`cancelled_at`）

半成品不能直接提交，但必须可被客户端重放。

### 3.7 断线恢复、重试与 Checkpoint

**三种恢复不是同一件事：**

| 类型 | 恢复什么 | 做法 | 约束 |
| --- | --- | --- | --- |
| Transport Resume | 事件传输 | 客户端带 `Last-Event-ID`，服务端按 `seq` 重放 Event Store | 不重新调用模型 |
| Model Retry | 模型调用 | 仅在**尚未产生业务事件**时有限重试 | 已产生业务事件则禁止盲目重跑 |
| Task Resume | 任务执行 | 从 Checkpoint 继续下一步 | 需要保存可恢复状态 |

**Streaming 失败阶段：**

| 阶段 | 已产生事实 | 策略 |
| --- | --- | --- |
| A. 连接/响应阶段 | 无事实 | DNS、网络失败、429、503 可有限重试 |
| B. 首事件后网络断 | 中等事实 | 禁止盲目拼接重跑；优先重放/查状态 |
| C. 大量事件后 | 大量事实 | 先查询上次状态，再决定重放还是重跑 |

**允许重试的白名单：** DNS 错误、网络失败、429、503。
**不能机械重试的黑名单：** 认证失败、参数错误、Schema 错误、内容拒答。

**重试必须限制 6 项：**

- `max_attempts=N`
- `max_total_seconds=T`
- `whitelist={429, 5xx, DNS}`
- exponential backoff
- jitter
- Run budget

**Checkpoint 至少保存 6 个字段：**

| 字段 | 含义 |
| --- | --- |
| `loop_step` | 当前步骤 |
| `completed[]` | 已完成内容 |
| `next_cursor` | 下一步位置 |
| `context_digest` | 上下文摘要 |
| `tool_state[]` | 工具状态 |
| `version` | checkpoint 版本 |

> Checkpoint 保存的是“可恢复状态”，不是把全部历史重新塞回 Prompt。

**断线与崩溃恢复矩阵：**

| 情况 | 恢复方式 |
| --- | --- |
| 客户端断线 + 短暂 | 重新订阅；服务端无需动作 |
| 客户端断线 + 长期 | 服务端后台清理 Run；通知前端已回收 |
| Worker 崩溃 + 未落库 | 读 Checkpoint 恢复，重跑未完成步骤 |
| Worker 崩溃 + 涉及外部副作用 | 标记 `unknown`，必要时人工确认 |

### 3.8 错误协议与 Trace

**对外错误事件示例：**

```json
{
  "type": "run.failed",
  "data": {
    "code": "TOOL_TIMEOUT",
    "stage": "tool.completed",
    "retryable": false,
    "hint": "需要确认是否属于 unknown 状态"
  }
}
```

**客户端需要看到 4 项：** 稳定错误码、是否可重试、恢复方式、失败阶段定位。
**服务端需要保留 5 项：** 原始异常、Trace ID、Request ID、Token 使用、工具副作用状态。
**禁止暴露：** API Key、Prompt 原文、内部堆栈。

**Trace 至少记录 14 个字段：**

```text
trace_id · run_id · 模型版本 · Prompt 版本
首 Token 时间 · 总耗时 · Token 使用
重试次数 · 重连次数 · 取消延迟 · 工具调用 · checkpoint
最终状态 · 错误码
```

**TTFT 口径：**

- TTFT = 请求开始 → **首个非空 `text.delta`** 到达客户端。
- 首个 SDK chunk 不一定包含文本：`role.chunk → empty delta → empty delta → text.delta != ""`。
- 不要把“首事件”当 TTFT。
- 同时区分 GEN（首 Token → 生成结束）与 E2E（用户操作 → 任务最终完成）。

### 3.9 客户端（Web / CLI）

- Web、CLI、IDE 共享同一套 Harness Event 协议，差异只停留在渲染层/IO 层。
- 客户端负责：展示文本、事件去重、自动重连、状态展示、触发取消。
- 客户端不负责：模型调用、工具执行、重试逻辑、解析供应商 chunk。
- Web 可以用 `EventSource`（简单 GET）或 `fetch + ReadableStream`（POST / 鉴权 / 精细控制）。
- CLI 只接收事件、去重、展示结果；不要自己调模型或执行业务。
- 页面至少展示：当前 Run ID、连接状态、TTFT、拼接后的文本、SSE 原始事件日志、取消/重连按钮。

### 3.10 持久化与部署

**课堂版：**

- 内存 Run Store：保存 Run 状态、事件序号、事件列表、取消控制。

**生产版：**

- PostgreSQL 保存事实事件：`run_id + seq` 唯一约束，保证可恢复、可追踪。
- Redis Stream / PubSub 只做实时通知，不作为事实来源；通知可丢，事实事件不能丢。
- Checkpoint 独立表或独立存储，保存第 3.7 节的 6 个字段。
- 可选 Docker / docker-compose 启动 API、PostgreSQL、Redis。
- 数据库写入策略：状态事件、工具调用、审批事件立即保存；文本 delta 批量保存，避免 IO 放大。

## 4. 实现顺序（课件 11 步）

课件标题写「11 步」，页面实际列出 10 步；第 11 步建议补「端到端验收与交付证据」。

| # | 步骤 | 产出 | 完成标准 |
| --- | --- | --- | --- |
| 1 | 定义 `RunEvent` | 事件模型与 `EventType` | 字段、`seq`、`schema_version` 可校验 |
| 2 | 实现 SSE 编解码 | `encode_sse` / `encode_heartbeat` / 解析器 | TC01–TC06 通过 |
| 3 | 抽象 `StreamingModel` | Protocol + `ModelStreamEvent` | 下游不依赖具体供应商 |
| 4 | 接入模型 Adapter | DeepSeek Adapter | 处理 choices/delta/finish_reason/usage 四类边界 |
| 5 | 实现 Run Store | 状态、事件、序号、取消信号 | 断线后仍能查询/订阅/取消 |
| 6 | 实现 Agent Loop | 事件生产者 | 产生 8 类业务事件，不操作 HTTP |
| 7 | 创建订阅取消接口 | 三个 API | 创建/订阅/控制分离；`Last-Event-ID` 可用 |
| 8 | 开发 Web/CLI 客户端 | 两个消费者 | 去重、重连、取消、渲染节流 |
| 9 | 增加异常测试 | 边界测试集 | 覆盖第 6 节测试矩阵 |
| 10 | 接入数据库与 checkpoint | 持久化与恢复 | 进程重启后可从 checkpoint 续跑 |
| 11 | 端到端验收与交付 | 演示、日志、测试报告 | 第 5 节 10 条验收全部有证据 |

## 5. 验收标准

课件第 45 页明示 5 条；其余 5 条按课程正文补齐，作为自查口径。

| # | 验收标准 | 判定方式 | 来源 |
| --- | --- | --- | --- |
| 1 | 首个 delta 快速展示 | 记录 TTFT；不能等完整回答才渲染 | 课件明示 |
| 2 | 不依赖模型 SDK | 客户端只依赖 `RunEvent`；换 Adapter 不改 Client | 课件明示 |
| 3 | Run 单一终态 | 构造完成/取消竞态，只能出现一条终态事件 | 课件明示 |
| 4 | 取消真停止 | 取消后模型/工具停止，已输出内容与 6 项状态被保存 | 课件明示 |
| 5 | 断线不重跑 | 重连带 `Last-Event-ID`，只重放事件，不再调用模型 | 课件明示 |
| 6 | SSE 编解码正确 | 中文、换行、引号、>64KB、拆包/粘包测试通过 | 整理补全 |
| 7 | `seq` 单调、事件可去重 | 重复投递不重复渲染；可按 `seq` 重放 | 整理补全 |
| 8 | 取消状态机完整 | `cancelling` 过渡态、CAS、幂等取消、部分输出归档 | 整理补全 |
| 9 | 失败阶段可判定 | A/B/C 阶段对应重试、重放、checkpoint 策略正确 | 整理补全 |
| 10 | Trace 与错误协议可审计 | 14 项 Trace、稳定错误码、测试报告可复现 | 整理补全 |

## 6. 测试要求

### 6.1 必测边界（课件列 10 项，标题写 11 项）

- [ ] SSE 编码
- [ ] 拆包恢复
- [ ] 粘包处理
- [ ] `seq` 顺序
- [ ] 事件去重
- [ ] 断线续传
- [ ] 取消传播
- [ ] 完成/取消竞态
- [ ] 流失败恢复
- [ ] checkpoint 恢复
- [ ] 第 11 项建议：TTFT 首事件统计或错误事件协议（课件未列出，二选一或都补）

### 6.2 课件示例：拆包恢复

```python
def test_sse_chunk_split():
    data = "id: 17\nevent: text.delta\nda|ta: {\"seq\":17}"
    events = list(parser.feed_bytes(data.encode("utf-8")))
    assert events[0].seq == 17
```

### 6.3 不要只测这些

- 只打开 `index.html` 看“页面会不会逐字出现”。
- 只用一个 `EventSource` 手工点一遍。
- 只做类型检查或语法检查，不验证真实事件顺序与失败路径。

> 课程结论：Harness 的 bug 多是“小颗粒度边界条件”，不是大架构错误。

## 7. 交付物与提交

### 7.1 必交

- [ ] 可运行源码（服务端 + Web/CLI 客户端）
- [ ] `README.md`：环境变量、启动方式、接口说明、事件协议
- [ ] 测试代码与测试结果（`pytest -q` 输出或报告）
- [ ] 演示证据：首 delta 截图/录屏、取消前后日志、断线重连日志、checkpoint 恢复日志
- [ ] 一份 `验收清单.md` 的自查结果（可用本目录模板）

### 7.2 建议提交结构

```text
streaming-agent-gateway/
├─ app/
│  ├─ main.py              # FastAPI 入口
│  ├─ events.py            # RunEvent / EventType
│  ├─ sse.py               # encode / parse / heartbeat
│  ├─ model/
│  │  ├─ protocol.py       # StreamingModel / ModelStreamEvent
│  │  └─ deepseek.py       # DeepSeek Adapter
│  ├─ store/
│  │  ├─ memory.py         # 课堂版 Run Store
│  │  └─ postgres.py       # 生产版（可选）
│  ├─ loop.py              # Agent Loop
│  ├─ checkpoint.py        # Checkpoint 读写
│  ├─ trace.py             # Trace 记录
│  └─ errors.py            # 稳定错误码 / 错误事件
├─ clients/
│  ├─ web/                 # EventSource 或 fetch 客户端
│  └─ cli.py               # CLI 消费者
├─ tests/
│  ├─ test_sse.py
│  ├─ test_events.py
│  ├─ test_cancel.py
│  ├─ test_replay.py
│  └─ test_checkpoint.py
├─ artifacts/              # 演示日志、测试报告、截图
├─ .env.example
└─ README.md
```

### 7.3 环境与命令（参考）

```bash
# Python 3.11+
pip install fastapi uvicorn httpx openai pydantic pytest pytest-asyncio
# 可选：asyncpg redis

export DEEPSEEK_API_KEY=...
uvicorn app.main:app --reload

pytest -q
python clients/cli.py "请解释 SSE 的 Last-Event-ID"
```

- 不要把 API Key 提交到仓库；用 `.env` 或环境变量。
- 如果使用代理，确认关闭响应缓冲，避免“伪流式”。

## 8. 常见坑（来自课件）

1. **伪 Streaming**：等完整回答再一次性返回；`yield` 写了不等于真的流式。
2. **同步 SDK 阻塞**：在异步接口里调用同步 SDK，阻塞事件循环；优先异步 SDK 或受控线程池。
3. **代理缓冲**：Nginx / Gateway / CDN 重新聚合响应；关闭缓冲、检查 timeout 与压缩。
4. **每个 delta 写数据库**：小事件数量巨大，IO 放大；状态/工具/审批事件立即存，文本 delta 批量存。
5. **断线直接取消**：用户刷新页面就取消 Agent；应区分“订阅断开”和“Run 取消”。
6. **首个 chunk 当 TTFT**：首个 SDK chunk 可能没有文本；要等首个非空 `text.delta`。
7. **透传供应商 chunk**：前端绑定具体 SDK，换模型全部重写；必须经过 Adapter。
8. **已产生业务事件后盲目重试**：会生成第二份独立输出，语义冲突；只能重放或走 checkpoint。
9. **吞掉 `CancelledError`**：`except Exception` 会把控制信号当普通异常；必须单独处理。
10. **心跳占用 `seq`**：心跳是注释行，不应进入业务事件序列。

## 9. 与参考代码的差距

`course_code/week01/1-3/app.py` 已经提供了很好的起点：

| 已具备 | 位置/说明 |
| --- | --- |
| `RunEvent` / `RunState` | 事件、状态、`seq`、取消信号 |
| SSE 编码与心跳 | `encode_sse` / `encode_heartbeat` |
| 创建/订阅/取消接口 | `POST /v1/runs`、`GET /v1/runs/{id}/events`、`POST /v1/runs/{id}/cancel` |
| `Last-Event-ID` | 订阅时按 `after_seq` 重放 |
| DeepSeek Adapter 雏形 | `StreamingModel` Protocol + `DeepSeekChatAdapter` |
| Web 客户端 | 内嵌 HTML + `EventSource` |

需要补齐/升级的部分：

- [ ] `POST /v1/runs` 返回 `202 Accepted`（当前是 200）。
- [ ] 对外事件补齐 `tool.started` / `tool.completed` / `run.retrying`。
- [ ] 把 `model.finished` / `model.usage` 与对外 RunEvent 的边界写清楚。
- [ ] 增加 `run.failed` 的稳定错误码、`stage`、`retryable`、`hint`。
- [ ] 增加 14 项 Trace 与 TTFT 统计。
- [ ] 增加 Checkpoint（6 字段）与进程重启恢复。
- [ ] 内存 Store 换成 PostgreSQL + Redis（或提供可切换实现）。
- [ ] 增加完整测试矩阵（SSE、拆包、粘包、seq、去重、重放、取消、竞态、恢复）。
- [ ] 增加 CLI 客户端与渲染节流。
- [ ] 取消状态机补 `cancelling` 与 CAS 幂等。
- [ ] 不要直接提交课堂 Demo；作业要求的是“升级后的网关”。

## 10. 备注

- 本目录按「第一章 2-3 = 流式输出」整理，来源是 `course_materials/week01/1-3.pdf` 第 45 页的「课后练习：Streaming Agent Gateway」。
- 仓库里的 `course_materials/week02/2-3.pdf` 是**第二章 MCP** 的课堂练习（扩展 `get_logistics` 工具），与本作业不是同一份。如果你实际指的是那份 MCP 练习，可以再单独建一份整理。
- 课件第 45 页的「11 步 / 10 标准 / 11 项测试」存在标题与列项数量不一致的情况；本整理已保留原文数量，并把缺失项标为「建议补全」，避免把推测当成课件硬性要求。
