# Server-Sent Events (SSE)

> 分类：Web / Streaming
> 相关课程：第一章 2-1 Streaming 概述与 SSE
> 整理日期：2026-10-31

## 定义

SSE（Server-Sent Events）是一种基于 HTTP 的**服务器单向推送**协议。客户端发起一个普通 HTTP 请求后，服务器保持响应不关闭，按固定文本格式持续把数据推给客户端。

## 通信过程

1. 客户端发起请求（浏览器可用 `EventSource`）
2. 服务器返回 `Content-Type: text/event-stream`，并保持连接
3. 服务器不断写入一条条事件
4. 连接断开后，浏览器默认会自动重连

## 协议格式

响应头：

```http
Content-Type: text/event-stream
Cache-Control: no-cache
```

响应体中的每条消息以空行结束：

```text
data: 第一段

event: token
data: {"text":"你"}

id: 42
data: 第二段

: heartbeat
```

字段：

- `data:`：消息内容，可写多行，客户端会拼接
- `event:`：自定义事件名，配合 `addEventListener`
- `id:`：事件 ID，断线重连时浏览器会带上 `Last-Event-ID`
- `retry:`：重连间隔（毫秒）
- 以 `:` 开头的是注释，常用来做心跳

## 客户端示例

```js
const es = new EventSource("/api/chat/stream");

es.onmessage = (e) => {
  console.log(e.data);
};

es.addEventListener("done", () => {
  es.close();
});
```

注意：原生 `EventSource` 只支持 GET，不能自定义请求头和请求体。

## 服务端示例（FastAPI）

```python
from fastapi.responses import StreamingResponse

async def event_stream():
    yield "data: 你好\n\n"
    yield "data: 世界\n\n"
    yield "data: [DONE]\n\n"

@app.get("/stream")
async def stream():
    return StreamingResponse(
        event_stream(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
    )
```

## 特点

- 单向：只能服务器推给客户端；客户端要发消息，需要再发请求
- 纯文本、UTF-8，不能直接传二进制
- 基于 HTTP，穿过代理和防火墙通常比 WebSocket 容易
- 自带断线重连和事件 ID 续传机制

## 与 WebSocket 对比

| 对比项 | SSE | WebSocket |
| --- | --- | --- |
| 方向 | 服务器 → 客户端 | 双向 |
| 协议 | HTTP | 独立协议（握手升级） |
| 数据 | 文本 | 文本 + 二进制 |
| 重连 | 浏览器内置 | 需要自己实现 |
| 适用场景 | LLM 流式输出、通知、进度推送 | 聊天室、游戏、协同编辑 |

只需要服务器持续推送文本时，SSE 更简单；需要双向实时通信时，选 WebSocket。

## 在 LLM 流式输出中的应用

OpenAI 兼容接口的流式响应普遍采用 SSE 格式：

```text
data: {"choices":[{"delta":{"content":"你"}}]}

data: {"choices":[{"delta":{"content":"好"}}]}

data: [DONE]
```

但请求通常是 POST + JSON，原生 `EventSource` 只支持 GET，因此前端一般使用：

- `fetch` + `ReadableStream` 手动解析
- 或官方 SDK 的流式接口

## EventSource 与 fetch + ReadableStream

`EventSource` 是浏览器内置的 SSE 客户端；`fetch + ReadableStream` 则是用通用 Fetch API 拿到响应流后自行解析。二者对比如下：

| 维度 | EventSource | fetch + ReadableStream |
| --- | --- | --- |
| API 定位 | SSE 专用 | 通用流式请求 |
| 请求方法 | 仅 GET | 任意方法 |
| 请求头/请求体 | 不能自定义 | 完全可控 |
| 响应格式 | 必须是 `text/event-stream`，浏览器自动解析 | 任意流式响应，自行解析 |
| 事件分发 | 自动按 `event:` 分发 | 自行解析并分发 |
| 断线重连 | 自动重连，支持 `Last-Event-ID` | 需自行实现 |
| 取消 | `es.close()` | `AbortController` |
| 错误处理 | 只能拿到 `onerror`，看不到状态码/响应体 | 可检查 `response.status` 和响应体 |
| 二进制 | 不支持 | 支持 |
| 认证 | 依赖 Cookie，不能加 `Authorization` 头 | 可加任意认证头 |
| 适用场景 | 简单 GET 推送 | LLM 对话、POST、鉴权、自定义协议 |

### 为什么 LLM 场景更常用 fetch

- 请求是 POST + JSON，`EventSource` 只支持 GET
- 需要 `Authorization` 等自定义请求头
- 需要检查 HTTP 状态码，区分 401/429/500 与正常流
- 需要 `AbortController` 主动中断生成
- 需要解析 `data: [DONE]`、多行 `data:`、跨 chunk 的半个 JSON
- 自动重连可能重复触发模型生成，通常希望手动控制重连

### 选择建议

- 只是接收服务端通知、进度、简单文本流，且是 GET：用 `EventSource`
- LLM 聊天、需要 POST/鉴权/中断/精细错误处理：用 `fetch + ReadableStream`
- 想要 SSE 语义又需要 POST/headers：可以用 `fetch` 自己封装，或使用 `@microsoft/fetch-event-source` 这类库

## 常见问题

- **代理缓冲**：Nginx 等默认可能缓冲响应，导致内容“攒一批才吐出来”；需要关闭缓冲（`proxy_buffering off` 或 `X-Accel-Buffering: no`）
- **格式错误**：消息之间必须有空行，否则客户端不会派发
- **连接超时**：长连接需要心跳，避免中间设备超时断开
- **连接数限制**：HTTP/1.1 下浏览器对同域名连接数有限制，多个 SSE 标签页可能互相挤占
