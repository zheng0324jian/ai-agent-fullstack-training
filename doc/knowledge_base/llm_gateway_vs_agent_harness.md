# LLM Gateway vs Agent Harness

> 分类：Architecture / Agent 运行时
> 相关课程：第一章 1-6 / 1-7 LLM Gateway；第三章 3-5 Agent Harness
> 整理日期：2026-11-06

## 定义

一句话区别：

> **LLM Gateway 管“模型怎么被调用”，Agent Harness 管“任务怎么被完成”。**

| 概念 | 定位 | 解决的问题 |
| --- | --- | --- |
| LLM Gateway | 模型访问层 | 稳定、便宜、安全地调用模型 |
| Agent Harness | 任务执行层 | 让模型通过工具、状态、验证和恢复真正完成任务 |

课程对两者的定义：

> **LLM Gateway**：统一模型调用入口与出口，把模型调用规则集中到一个可治理的边界中。
>
> **Agent Harness**：模型之外，组织工具、状态、上下文与执行环境的支撑程序；**Agent = Model + Harness**。

## 架构位置

```text
用户任务
   ↓
Agent Harness                         ← 任务执行层
   ├─ Agent Loop                      何时继续 / 何时停止
   ├─ Context / Prompt                给模型看什么
   ├─ Tool Runtime                    工具发现、调用、结果回写
   ├─ State / Checkpoint              状态保存、中断恢复
   ├─ Governance                      权限、审批、参数校验
   ├─ Sandbox                         文件、命令、网络边界
   └─ LLM Gateway                     模型访问层
         ├─ 鉴权 / 限流 / 配额
         ├─ 模型路由 / fallback
         ├─ Prompt 模板
         ├─ Structured Output
         ├─ Streaming
         └─ Token / Cost / Latency Trace
                ↓
         模型供应商（DeepSeek / OpenAI / ...）
```

Gateway 可以在 Harness 内部，也可以是一个被多个 Agent 共享的独立服务。

## 核心区别

| 维度 | LLM Gateway | Agent Harness |
| --- | --- | --- |
| 边界 | 一次模型请求 / 流式响应 | 一次完整任务 / Run |
| 核心问题 | 用哪个模型、密钥怎么管、怎么路由、怎么重试、怎么计费 | 任务怎么拆、上下文怎么给、工具怎么调、失败怎么恢复 |
| 状态 | 基本是请求级，不保存任务状态 | 有计划、步骤、证据、Checkpoint、终态 |
| 工具 | 不知道工具的存在 | 组织 Tool Runtime、工具调用与结果回写 |
| 失败处理 | 重试、切换备用模型、fallback | 重新规划、恢复、回滚、人工审批 |
| 安全 | API Key、租户、限流、配额 | 工具权限、沙箱、HITL 审批 |
| 可观测 | 单次调用 Trace：Token、延迟、成本、模型 | 整次 Run Trace：步骤、工具、证据、Checkpoint、终态 |
| 复用性 | 可被多个 Agent / RAG / 批处理共享 | 通常面向某个 Agent 或某类任务 |
| 输出 | 模型响应或流式文本 | 经过验证的任务结果 / 交付物 |

再简化一点：

```text
LLM Gateway：把“模型调用”做成可治理的基础设施
Agent Harness：把“任务执行”做成可控制、可验证、可恢复的闭环
```

## Gateway、Loop、Harness 三层职责

三者容易混淆，可以这样分：

| 层 | 负责 | 不负责 |
| --- | --- | --- |
| LLM Gateway | 模型访问、路由、鉴权、限流、重试、fallback、结构化输出、用量与成本 | 任务计划、工具执行、状态恢复 |
| Agent Loop | 多轮“模型 → 工具 → 观察”循环，判断继续还是停止 | 供应商密钥、模型路由、完整任务状态 |
| Agent Harness | 把 Loop、Context、Tool Runtime、State/Checkpoint、Governance、Sandbox、验证与交付组织成完整闭环 | 供应商 API 细节、模型计费 |

课程原话：

> Harness 不是重新写一个“大类”，而是把已有模块按职责连接起来。

## 课程中的对应

### LLM Gateway（Week 01，1-6 / 1-7）

`course_code/week01/1-6/gateway.py` 和 `course_code/week01/1-7/llm-gateway` 主要处理：

- 统一模型入口，隔离供应商差异
- Provider Adapter、模型别名、路由
- 鉴权、限流、配额
- Prompt 模板与版本
- Structured Output 校验与纠错
- Streaming 转发
- 重试、fallback、熔断
- Token、成本、延迟、调用状态 Trace

它**不负责**：

- 任务计划
- 工具执行
- 任务状态与 Checkpoint
- 结果验证
- 人工审批
- 交付物生成

### Agent Harness（Week 03，3-5）

`course_code/week03/3-5/harness_agent` 主要处理：

- Agent Loop：多轮模型与工具交互
- Planning：任务拆解与计划管理
- Context：每轮给模型准备什么信息
- Tool Runtime：工具调用、结果回写
- State / Checkpoint：任务状态、恢复
- Governance / Approval：权限与人工审批
- Sandbox：文件、命令、网络边界
- Verification：测试证据、完成契约
- Delivery：最终报告与交付物

它**不负责**：

- 供应商密钥管理
- 多模型路由与计费
- 限流与配额
- 模型 API 的统一协议适配

这些通常交给 LLM Gateway。

## 两者的关系

不是替代关系，而是上下游协作关系：

```text
Agent Harness
   ├─ 负责“任务怎么完成”
   └─ 需要模型能力时 → 调用 LLM Gateway
                          └─ 负责“模型怎么调用”
```

- **Gateway 是模型访问的“统一窗口”**：所有 Agent 都通过它调用模型，密钥、路由、限流、成本集中管理。
- **Harness 是任务执行的“项目经理 + 操作系统”**：它决定任务怎么拆、工具怎么用、失败怎么办、什么时候算完成。
- **Gateway 不需要知道 Harness 的计划、工具和状态。**
- **Harness 通常需要一个模型访问层**，可以自己直连模型，也可以调用 Gateway；生产环境更常见的是走 Gateway。

## 为什么需要分开

1. **关注点分离**：模型访问规则和任务执行规则变化频率不同，分开后更容易维护。
2. **多 Agent 共享**：一个 Gateway 可以服务多个 Agent、RAG、批处理任务，统一密钥、配额和计费。
3. **独立演进**：换模型供应商只改 Gateway；改任务流程只改 Harness。
4. **安全边界不同**：Gateway 管 API Key、租户、限流；Harness 管工具权限、沙箱和审批。
5. **独立观测**：Gateway 记录单次调用 Trace；Harness 记录整次 Run 的步骤与证据。

## 常见误解

| 误解 | 实际情况 |
| --- | --- |
| “Gateway 就是 Agent 的大脑” | 大脑是 Model；Gateway 只是模型访问层 |
| “Harness 就是 Agent Loop” | Loop 只是 Harness 的一个组件；Harness 还包括 Context、Tools、State、Sandbox、Governance |
| “有了 Gateway 就有了 Agent” | Gateway 只解决模型调用，不解决任务完成 |
| “Harness 必须包含 Gateway” | 不一定；Harness 需要模型访问能力，可以直连，也可以走 Gateway |
| “Gateway 和 Harness 是竞品” | 不是；它们解决不同层的问题，通常组合使用 |

## 一句话总结

> **Gateway 让模型“调得稳”；Harness 让 Agent“做得完”。**

- **Model**：专家，负责思考和提出方案。
- **LLM Gateway**：公司统一的外联 / 采购 / 结算窗口，管账号、额度、供应商、路由、账单。
- **Agent Harness**：项目经理 + 流程 + 工具 + 档案 + 风控，负责把方案真正落地、验证、恢复和交付。

两者结合，才是一个可运行、可治理、可恢复的生产级 Agent 系统。
