# Pydantic

> 分类：Python / 数据校验
> 相关课程：第一章 1-5 Structured Output；第二章 2-2 Tool Runtime
> 整理日期：2026-11-03

## 定义

Pydantic 是 Python 的**数据校验与序列化库**。你用普通 Python 类型注解定义“数据应该长什么样”，Pydantic 在运行时负责：

- 检查字段是否存在、类型是否正确；
- 检查长度、范围、正则等约束；
- 做必要的类型转换；
- 校验失败时给出结构化错误；
- 把模型转成 dict / JSON；
- 根据模型自动生成 JSON Schema。

一句话：

> **Pydantic 把“不可信的原始数据”变成“程序可以放心使用的类型化对象”。**

## 最小示例

```python
from pydantic import BaseModel, Field, ValidationError


class OrderQuery(BaseModel):
    order_id: str = Field(
        pattern=r"^ord_\d{4}$",
        description="订单号，例如 ord_1002",
    )
    limit: int = Field(default=20, ge=1, le=50)


query = OrderQuery.model_validate({
    "order_id": "ord_1002",
    "limit": 20,
})

print(query.order_id)      # ord_1002
print(query.limit)         # 20
print(query.model_dump())  # {'order_id': 'ord_1002', 'limit': 20}
```

校验失败：

```python
try:
    OrderQuery.model_validate({
        "order_id": "1002",  # 不符合 ^ord_\d{4}$
        "limit": 999,        # 超过 50
    })
except ValidationError as e:
    print(e.errors())
```

输出类似：

```python
[
    {
        "type": "string_pattern_mismatch",
        "loc": ("order_id",),
        "msg": "String should match pattern '^ord_\\d{4}$'",
        "input": "1002",
    },
    {
        "type": "less_than_equal",
        "loc": ("limit",),
        "msg": "Input should be less than or equal to 50",
        "input": 999,
    },
]
```

核心价值：不是等程序跑到后面才报 `KeyError` / `TypeError`，而是在数据进入系统时就把问题拦住，并给出具体位置。

## 常用 API

| API | 作用 |
| --- | --- |
| `BaseModel` | 所有模型的基类 |
| `Field(...)` | 给字段加约束和说明，如 `min_length`、`pattern`、`ge`、`le`、`description` |
| `model_validate(dict)` | 把 dict 校验并转成模型对象 |
| `model_validate_json(str)` | 把 JSON 字符串校验并转成模型对象 |
| `model_dump()` | 模型对象转 dict |
| `model_dump_json()` | 模型对象转 JSON 字符串 |
| `model_json_schema()` | 生成 JSON Schema |
| `ValidationError` | 校验失败时抛出，包含结构化错误列表 |
| `ConfigDict(extra="forbid")` | 禁止未定义字段，工具参数常用 |
| `@field_validator` | 单字段自定义校验 |
| `@model_validator` | 跨字段 / 整体校验 |
| `TypeAdapter` | 校验非 `BaseModel` 类型，如 `list[OrderQuery]` |

示例：禁止多余字段并做跨字段校验。

```python
from typing import Literal

from pydantic import BaseModel, ConfigDict, Field, model_validator


class AgentDecision(BaseModel):
    model_config = ConfigDict(extra="forbid")

    action: Literal["search", "finish"]
    query: str | None = None
    answer: str | None = None

    @model_validator(mode="after")
    def check_combination(self):
        if self.action == "search" and not self.query:
            raise ValueError("action=search 时必须提供 query")
        if self.action == "finish" and not self.answer:
            raise ValueError("action=finish 时必须提供 answer")
        return self
```

## 在 Agent 项目中的用法

### 1. 约束 LLM 的结构化输出

```text
Pydantic 模型
  → model_json_schema()
  → 交给模型做 Structured Output
  → 模型返回 JSON
  → model_validate_json()
      ├─ 成功：得到类型化对象
      └─ 失败：ValidationError → 纠错 / 重试 / 降级
```

课程 1-5「Structured Output」的主线就是：**Pydantic 定义契约，Schema 约束模型，校验失败进入修复循环。**

### 2. 校验 Tool Call 的参数

```python
args = CreateRefundArgs.model_validate(tool_call.arguments)
```

校验通过才进入 handler；校验失败直接拒绝，**不会产生业务副作用**。课程 2-2 Tool Runtime 中的 `input_model.model_validate(...)` 就是做这件事。

### 3. 校验工具返回值

```python
output = OrderResult.model_validate(raw_output)
```

工具返回的数据也可能缺字段、类型不对。校验后再放回 Context，避免脏数据污染下一轮推理。

### 4. FastAPI 的请求 / 响应模型

```python
class CreateRunRequest(BaseModel):
    prompt: str = Field(min_length=1)


@app.post("/v1/runs")
async def create_run(body: CreateRunRequest):
    ...
```

FastAPI 会自动校验请求体、生成 OpenAPI 文档，并返回 422 错误。课程 1-3 的 `CreateRunRequest` 就是这种用法。

### 5. 配置管理

`pydantic-settings` 可以把环境变量映射成类型化配置对象，例如 `DEEPSEEK_API_KEY`、超时时间、模型名。它和 `BaseModel` 是同一套思路，只是数据来源从 JSON 变成了环境变量。

## 校验失败怎么处理

在 Agent 项目里，`ValidationError` 不只是异常，还可以作为反馈信号：

```python
try:
    decision = AgentDecision.model_validate_json(raw)
except ValidationError as e:
    repair_prompt = f"""
    上一次输出不符合约定，请只修正格式后重新输出。
    错误：{e.errors()}
    """
```

常见处理顺序：

1. **可修复错误**：把结构化错误交回模型，要求只修正格式，重试 1～2 次。
2. **不可修复错误**：返回稳定错误码，走降级或人工处理。
3. **工具参数错误**：直接拒绝执行，记录审计事件。
4. **工具返回值错误**：标记为工具失败，不要把脏数据放入 Context。

## Pydantic 不负责什么

| Pydantic 能做 | Pydantic 不能做 |
| --- | --- |
| 检查字段是否存在 | 判断订单是否真的存在 |
| 检查类型、格式、范围 | 判断用户是否有权限退款 |
| 把 JSON 转成对象 | 保证模型一定输出合法 JSON |
| 给出结构化校验错误 | 替代数据库约束、业务规则、沙箱 |
| 生成 JSON Schema | 保证模型按 Schema 执行 |

所以它是**“数据进入程序时的安检门”**，不是“业务大脑”，也不是“安全边界”。

一个订单号格式合法，不代表这个订单真实存在；一个退款参数合法，不代表当前用户有权退款。前者靠 Pydantic，后者靠业务查询、权限系统和 Harness。

## Pydantic v2 注意事项

- v2 使用 `model_validate` / `model_dump` / `model_json_schema`；如果看到 `parse_obj` / `.dict()` / `.schema()`，那是 v1 写法。
- 工具参数建议 `ConfigDict(extra="forbid")`，拒绝模型多传的字段。
- 默认会做类型转换，例如 `"20"` 可能转成 `20`；需要严格时用 `strict=True`。
- `model_json_schema()` 可能生成 `$defs`、`title` 等字段；不同模型对 JSON Schema 的支持有差异，必要时裁剪或内联。
- `Optional[T]` 不等于“可以不传”，默认值要显式设置 `= None`。
- Pydantic 不是 ORM；数据库建模用 SQLAlchemy 等工具。
- 校验通过只代表“结构合法”，业务合法性仍要单独检查。

## 与课程的关系

| 课程位置 | 用途 |
| --- | --- |
| 1-5 Structured Output | 定义输出模型、生成 JSON Schema、校验与纠错重试 |
| 2-2 Tool Runtime | `input_model` / `output_model` 校验工具参数和结果 |
| 2-4 工具治理 | 严格参数模型、`extra="forbid"`、校验失败转稳定错误码 |
| 1-3 Streaming Gateway | `CreateRunRequest` 校验 HTTP 请求体 |
| 1-6 LLM Gateway | FastAPI 请求 / 响应模型 |

## 相关条目

- [Agent Prompt Engineering](./agent_prompt_engineering.md)：Prompt 作为行为契约，以及它与 Harness 的边界
