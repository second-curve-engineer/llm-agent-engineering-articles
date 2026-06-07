# 【深度篇】Multi-Agent 工程实践：从「能跑」到「稳定」

系列的第 09 篇讲了 Multi-Agent 的基础：三种模式、通信方式、四个难点和主流框架。

但知道这些还不够。

真正做 Multi-Agent 系统的时候，你会遇到更具体的问题：任务怎么拆才算合理，拆多细是过度设计？Sub-Agent 之间怎么传递信息，用什么格式？某个 Sub-Agent 失败了，整个任务怎么处理？多个 Sub-Agent 并行跑，出了问题怎么追溯到具体是哪个 Agent 的哪一步？

这篇来回答这些问题。

---

## 一、任务拆解：什么样的拆法是好的

Multi-Agent 最核心的设计决策，是任务怎么拆。拆得好，系统清晰高效；拆得不好，比单 Agent 更难调试。

### 1.1 什么时候该拆，什么时候不该拆

不是所有任务都适合 Multi-Agent。拆分是有成本的：增加了协调开销、状态同步复杂度和调试难度。在以下情况才值得拆：

**子任务之间真正独立**：子任务 A 和子任务 B 没有执行顺序依赖，可以并行跑。如果所有子任务都必须串行，拆了没有收益。

**子任务超出单 Agent 的 Context 承载能力**：单个 Agent 跑完整任务会导致 Context 爆炸，或者任务需要同时处理太多不同领域的工具和知识。

**子任务有明确的专业化边界**：「分析日志」和「查数据库」和「生成报告」，三件事有清晰的输入输出边界，可以独立设计和测试。

**不应该拆的情况**：
- 子任务之间有强依赖，后一个任务必须等前一个完成才能开始
- 协调成本超过并行收益（子任务本身很短，协调开销反而更大）
- 团队还没有 Trace、Eval、状态管理的基础设施

### 1.2 任务拆解的基本原则

**单一职责**：每个 Sub-Agent 只做一件事，输入输出边界清晰。「分析日志并查数据库并生成报告」不是一个 Sub-Agent 的职责，而是三个。

**最小权限**：每个 Sub-Agent 只拥有完成自己任务所需的工具权限，不多给。日志分析 Agent 不需要有写数据库的权限。这不只是安全考虑，也是减少 Sub-Agent 做出意外操作的工程保障。

**幂等设计**：Sub-Agent 的执行应该是幂等的——同样的输入，执行多次，产生相同的结果，没有重复的副作用。这对失败重试和精准 Resume 至关重要。

**明确的终止条件**：每个 Sub-Agent 都应该有清晰的完成标准，不应该依赖 Orchestrator 来判断"它是不是跑完了"。

### 1.3 Orchestrator 的设计

Orchestrator 是 Multi-Agent 系统里最关键的组件，也是最容易设计过度的地方。

**Orchestrator 应该做什么：**
- 把用户任务分解成子任务，分配给对应的 Sub-Agent
- 收集 Sub-Agent 的结果，做合并和冲突处理
- 监控整体任务进度，处理超时和失败

**Orchestrator 不应该做什么：**
- 不应该包含具体的业务逻辑——那是 Sub-Agent 的职责
- 不应该直接操作工具——Orchestrator 本身不应该有工具权限，除非它需要做一些协调性的操作（比如写入共享状态）

**静态 Orchestrator vs 动态 Orchestrator**

静态 Orchestrator：任务拆解逻辑写死在代码里，按固定流程分发。实现简单、行为可预期、容易测试。适合任务类型固定、拆解逻辑不变的场景。

动态 Orchestrator：用 LLM 来做任务拆解——根据用户输入，动态决定拆成哪些子任务、分配给哪些 Sub-Agent。灵活，但不可预期，拆解结果难以测试，出了问题难以排查。

**建议**：除非你的任务多样性确实要求动态拆解，否则优先用静态 Orchestrator。可预期的系统比灵活的系统更容易维护。

> 📌 【插图一：静态 vs 动态 Orchestrator 对比图】

---

## 二、Sub-Agent 之间的协议设计

Sub-Agent 之间需要传递信息，协议设计的质量直接影响系统的可维护性和可调试性。

### 2.1 消息格式

Sub-Agent 之间传递的消息，应该是结构化的，而不是自然语言。自然语言在机器之间传递会引入歧义，也难以做格式校验。

一个 Sub-Agent 消息的基本结构：

```json
{
  "message_id": "msg_20240115_001",
  "session_id": "session_abc123",
  "from_agent": "orchestrator",
  "to_agent": "log_analyzer",
  "message_type": "task_request",
  "payload": {
    "task_id": "task_001",
    "task_type": "analyze_logs",
    "input": {
      "service": "order-service",
      "time_range": "2024-01-15T14:00:00Z/2024-01-15T15:00:00Z",
      "error_pattern": "connection timeout"
    },
    "constraints": {
      "max_results": 50,
      "timeout_seconds": 30
    }
  },
  "metadata": {
    "created_at": "2024-01-15T14:23:00Z",
    "priority": "high",
    "correlation_id": "root_task_xyz"
  }
}
```

几个关键字段：

**session_id**：把整个 Multi-Agent 任务的所有消息关联起来，是可观测性的基础。

**correlation_id**：根任务的 ID，在所有子任务的消息里都带上，便于追溯一个完整任务的所有执行链路。

**message_type**：消息类型（`task_request` / `task_result` / `task_error` / `status_update`），接收方可以根据类型路由处理逻辑。

**constraints**：调用约束，包括超时时间、结果数量限制等。每个子任务都应该有明确的超时，不能无限等待。

### 2.2 结果的标准化

Sub-Agent 返回结果时，格式应该标准化，便于 Orchestrator 统一处理：

```json
{
  "message_id": "msg_20240115_002",
  "in_reply_to": "msg_20240115_001",
  "from_agent": "log_analyzer",
  "to_agent": "orchestrator",
  "message_type": "task_result",
  "status": "success",
  "payload": {
    "task_id": "task_001",
    "result": {
      "found_errors": 3,
      "patterns": ["connection pool exhausted", "timeout after 30s"],
      "evidence": [
        { "timestamp": "2024-01-15T14:23:01Z", "log": "..." }
      ]
    },
    "confidence": 0.92
  },
  "metadata": {
    "execution_time_ms": 1240,
    "tools_used": ["query_logs", "parse_log_pattern"],
    "token_usage": { "input": 840, "output": 320 }
  }
}
```

**status 字段**：`success` / `partial_success` / `failed` / `timeout`。Orchestrator 根据 status 决定后续处理逻辑。

**confidence**：Sub-Agent 对自己结果的置信度。Orchestrator 在合并多个 Sub-Agent 结果时，可以用 confidence 做加权。

**metadata.token_usage**：每个 Sub-Agent 的 Token 消耗，Orchestrator 汇总后可以做整体成本监控。

---

## 三、失败处理与降级策略

Multi-Agent 系统最容易被忽视的工程问题，是失败处理。单 Agent 失败了，任务失败。Multi-Agent 里某个 Sub-Agent 失败了，不一定意味着整个任务失败——但要设计好降级策略。

### 3.1 失败的分类

不同类型的失败，处理方式不同：

**超时**：Sub-Agent 在规定时间内没有返回结果。处理方式：先重试一次（可能是临时网络问题），重试仍然超时则标记为失败，看是否影响其他子任务的进行。

**工具调用失败**：Sub-Agent 调用的工具返回了错误。处理方式：如果是可重试的错误（网络抖动），重试；如果是不可重试的错误（权限不足、资源不存在），标记失败，上报给 Orchestrator。

**结果质量不达标**：Sub-Agent 返回了结果，但 confidence 低于阈值，或者格式不符合预期。处理方式：可以触发 HITL，让人工判断是否接受这个结果。

**Sub-Agent 崩溃**：Sub-Agent 进程异常退出。处理方式：Orchestrator 检测到心跳超时，把任务重新分配给新的 Sub-Agent 实例（需要任务是幂等的）。

### 3.2 降级策略

**部分结果继续**：如果某个 Sub-Agent 失败了，但它的结果对整体任务不是必须的，Orchestrator 可以用其他 Sub-Agent 的结果继续生成最终输出，并在输出中标注「日志分析部分因超时未完成，以下结论基于监控数据和调用链分析」。

**结果替代**：如果某个 Sub-Agent 失败，尝试用另一个能力相似的 Sub-Agent 替代。比如「精细日志分析 Agent」超时，降级用「快速日志摘要 Agent」。

**人工介入（HITL）**：关键 Sub-Agent 失败时，不自动降级，而是暂停整个任务，通知人工处理。适合高风险场景，宁可等待也不要用不完整的结果继续。

**任务终止**：某些 Sub-Agent 的结果是后续所有步骤的前提，它失败了后续无法继续。直接终止任务，返回明确的失败原因，而不是用空结果或默认值蒙混过关。

> 📌 【插图二：失败处理决策树】

### 3.3 重试设计

重试是失败处理的第一道防线，但重试设计不当会放大问题。

**重试次数限制**：最多重试 N 次（通常 2-3 次），超过次数直接走降级逻辑。无限重试会导致任务卡死。

**退避策略**：重试之间加指数退避（第一次重试等 1s，第二次等 2s，第三次等 4s），避免瞬间大量重试打垮下游服务。

**幂等前提**：只有 Sub-Agent 的执行是幂等的，重试才安全。有副作用的操作（写数据库、发请求）需要先确认幂等性，或者用幂等键控制。

**重试范围**：重试应该是 Sub-Agent 级别的，而不是整个 Multi-Agent 任务级别的。某个 Sub-Agent 失败了重试它，不要把整个任务从头跑一遍。

---

## 四、Multi-Agent 的可观测性

Multi-Agent 的 Trace 比单 Agent 复杂得多：一次任务产生多条执行链路，每个 Sub-Agent 有自己的 Loop，工具调用分散在不同 Agent 里。如果没有专门设计，出了问题完全无从追溯。

### 4.1 Trace 的串联设计

核心原则：**所有 Sub-Agent 的执行链路，必须能被串联到同一个根任务下。**

实现方式：在根任务启动时生成一个 `root_run_id`，所有 Sub-Agent 的 Trace 都携带这个 ID。查询时，用 `root_run_id` 可以拉出整个 Multi-Agent 任务的完整执行链路。

Trace 结构示例：

```
root_run_id: "run_abc123"
├── Orchestrator
│   ├── 任务拆解：生成 3 个子任务
│   ├── 分发给 log_analyzer（task_001）
│   ├── 分发给 metrics_checker（task_002）
│   └── 分发给 trace_analyzer（task_003）
├── log_analyzer（task_001）
│   ├── Loop 1: LLM Span → Tool Span（query_logs）
│   ├── Loop 2: LLM Span → Tool Span（parse_pattern）
│   └── 结果返回给 Orchestrator
├── metrics_checker（task_002）
│   ├── Loop 1: LLM Span → Tool Span（check_metrics）
│   └── 结果返回给 Orchestrator（超时，标记失败）
└── Orchestrator
    ├── 合并结果（task_002 失败，降级处理）
    └── 生成最终报告
```

### 4.2 关键监控指标

除了执行链路，还需要对 Multi-Agent 系统建立整体的监控指标：

**任务级别**：
- 端到端成功率（包含降级成功的任务）
- 平均完成时间
- Sub-Agent 失败率（分 Agent 类型统计）
- 降级触发率（多少任务触发了降级）

**Sub-Agent 级别**：
- 各 Sub-Agent 的平均执行时间
- 各 Sub-Agent 的超时率和错误率
- 各 Sub-Agent 的 Token 消耗
- 各 Sub-Agent 的工具调用成功率

这些指标能帮你发现系统瓶颈：某个 Sub-Agent 超时率特别高，说明它的工具或 Prompt 有问题；某个 Sub-Agent 的 Token 消耗异常高，说明 Context 管理有问题。

### 4.3 调试 Multi-Agent 的几个技巧

**隔离测试每个 Sub-Agent**：在接入 Orchestrator 之前，先用固定输入单独测试每个 Sub-Agent，确保它能正确处理正常 case 和边界 case。Multi-Agent 系统的问题，很多时候根源在某个 Sub-Agent 本身，而不是协调逻辑。

**录制和回放**：记录真实任务里每个 Sub-Agent 的输入和输出，复现问题时可以用录制的输入直接喂给 Sub-Agent，不需要重跑整个 Multi-Agent 流程。

**逐步放开并行**：先用串行模式跑通整个流程，确认每个步骤都正确，再改成并行。并行引入的问题（竞态条件、结果冲突）比串行更难调试。

---

## 五、几个实践坑

**坑一：Orchestrator 包含了太多业务逻辑**

Orchestrator 越来越胖，开始直接处理各种边界情况、做业务判断。结果是：Orchestrator 既是调度层又是业务层，逻辑纠缠在一起，测试和维护都很困难。业务逻辑应该在 Sub-Agent 里，Orchestrator 只负责调度。

**坑二：Sub-Agent 之间直接通信**

Sub-Agent A 调用了 Sub-Agent B，绕过了 Orchestrator。一旦 A-B 之间的通信出问题，Orchestrator 完全不知道，也没有办法介入处理。Sub-Agent 之间不应该直接通信，所有消息都通过 Orchestrator 或共享存储传递。

**坑三：没有超时，没有重试上限**

某个 Sub-Agent 开始无限等待工具返回，整个 Multi-Agent 任务卡死。每个子任务必须有超时时间，重试必须有上限。没有超时的系统，迟早会卡死。

**坑四：并行任务的结果冲突没有处理策略**

两个 Sub-Agent 并行运行，各自得出了不同的根因结论，Orchestrator 不知道该用哪个。在设计阶段就要明确：结果冲突时，是用 confidence 加权、是触发 HITL、还是以某个 Sub-Agent 的结论为准。没有策略的冲突处理，会让最终输出不可信。

**坑五：所有错误都触发全局重试**

某个非关键的 Sub-Agent 超时，Orchestrator 把整个任务从头跑了一遍。代价巨大，而且重新跑一遍不一定能解决原来的问题。重试应该是 Sub-Agent 级别的，降级策略应该是任务级别的，不要混在一起。

**坑六：没有 root_run_id，Trace 无法串联**

每个 Sub-Agent 各自有自己的 Trace，但没有一个公共的根 ID 把它们关联起来。出了问题，要把五个 Sub-Agent 的日志手工对齐时间戳来排查。这是 Multi-Agent 系统里最痛的可观测性问题，也是最容易在一开始就设计好的事情。

---

## 最后说一句

Multi-Agent 系统的难点，不在于「怎么让多个 Agent 同时跑起来」——那一两天就能做到。

真正的难点在于：**某个 Sub-Agent 失败了，系统怎么优雅地处理；出了问题，怎么快速定位到具体是哪个 Agent 的哪一步；业务需求变了，怎么只改一个 Sub-Agent 而不影响整个系统。**

这些问题，靠框架解决不了，要靠工程设计。

任务拆解的边界、Sub-Agent 的协议、失败处理的策略、Trace 的串联——这四件事设计好了，Multi-Agent 系统才能从「能跑」变成「稳定」。
