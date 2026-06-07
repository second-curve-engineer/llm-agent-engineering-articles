# Agent 的分工：一文讲透 Multi-Agent

一个 Agent 接到任务：全面排查一次线上故障。

它需要分析应用日志、查监控指标、梳理服务调用链，最后生成一份根因报告。

单个 Agent 串行跑下来：40 轮对话，Context 越来越长，到后面开始"忘事"——前面收集的证据，后面推理时已经挤出了窗口。

如果任务拆分合理，多个 Sub-Agent 可以并行处理日志、指标和调用链，整体耗时可能明显下降，同时每个 Agent 的 Context 也更干净。

Multi-Agent 不是默认更高级，而是在任务复杂度、并行度和专业化需求超过单 Agent 承载能力时，才值得引入。



## 一、为什么需要 Multi-Agent

单个 Agent 有三个天花板，复杂任务一碰就到顶：

**Context 长度有限**：Agent 跑的轮次越多，积累的上下文越长。超出窗口之后，早期收集的证据开始被截断，推理质量下降。这不是模型不够聪明，是物理限制。

**串行效率低**：一个 Agent 一次只能做一件事。如果任务的多个子问题之间没有依赖关系，串行跑完全是浪费——本来可以并行的事情，硬是排成了队。

**单一 Prompt 难以覆盖复杂任务**：让一个 Agent 又分析日志、又查数据库、又写报告，Prompt 里要塞的背景知识和工具集越来越多，反而让每一块都做不精。专业的事交给专业的 Agent，效果更好。

Multi-Agent 解决的核心问题是三个字：**分治、并行、专业化**。



## 二、Multi-Agent 的基本模式

Multi-Agent 没有固定的拓扑结构，但有三种常见模式，可以单独使用，也可以组合。

**Orchestrator + Sub-Agent**

最常见的模式。Orchestrator 是调度层，负责拆解任务、分发给 Sub-Agent、收集结果、汇总输出。它可以是 LLM Agent，也可以是代码写死的 workflow 或状态机。Sub-Agent 各司其职，只处理分配给自己的那一块，不需要了解全局。

Orchestrator 的任务拆解质量决定了整体效果，是这个模式的核心难点。

**Pipeline（流水线）**

Agent A 的输出是 Agent B 的输入，依次流转。适合有明确先后依赖的任务——比如先做信息提取，再做分析，再做报告生成。每个 Agent 只关心自己的输入和输出，链路清晰，容易调试。

**并行**

多个 Agent 同时执行互相独立的子任务，最后由一个汇总节点合并结果。适合子任务之间没有依赖的场景。并行能显著降低整体延迟，但需要处理结果冲突——两个 Agent 得出了不同结论时，谁说了算。

> 📌 【插图一：Multi-Agent 三种模式示意图】



## 三、Agent 之间怎么通信

Agent 之间需要传递信息，但不是所有 Agent 共享一个大 Context——那样 Context 会膨胀得更快。

常见的通信方式有三种：

**消息传递**：Agent 之间通过结构化消息交互，每条消息包含发送方、内容、意图。Orchestrator 发指令，Sub-Agent 返回结果。边界清晰，但需要设计好消息格式。

**共享存储**：所有 Agent 读写同一个 Evidence Store 或 Memory Store。Agent A 写入发现的证据，Agent B 读取并在此基础上继续分析。适合需要跨 Agent 共享中间结果的场景。

**Handoff（控制权移交）**：Agent A 完成自己的部分后，把控制权连同上下文一起移交给 Agent B。不是并行，而是接力——B 拿到 A 留下的状态，继续往下跑。

常见组合是：Orchestrator 用消息分发任务，Sub-Agent 通过共享存储沉淀中间结果，跨阶段再用 Handoff 做衔接。



## 四、Multi-Agent 的难点

Multi-Agent 解决了单 Agent 的天花板，但也引入了新的复杂度。

**任务拆解**：怎么拆、拆多细，决定了整体效果。拆太粗，Sub-Agent 还是要处理复杂逻辑；拆太细，协调开销反而超过了并行收益。好的拆解需要对任务本身有深入理解，不是自动化就能解决的。

**结果一致性**：多个 Agent 并行，可能得出互相矛盾的结论。汇总时需要明确的冲突处理策略——是取置信度最高的那个，还是交给 Orchestrator 二次判断，还是触发 HITL 让人来决定。

**可观测性**：一次任务会产生多条执行链路。最好有一个 root run / trace，把每个 Sub-Agent 的执行链路、handoff、工具调用都关联起来；否则复盘时很难还原完整因果链。

**HITL 与权限边界**：Sub-Agent 遇到高危操作时，审批权在哪里？是 Sub-Agent 自己暂停等待，还是上报给 Orchestrator 统一处理？权限边界如果不清晰，Multi-Agent 的风险比单 Agent 更难控制。

> 📌 【插图二：Multi-Agent 难点示意图】



## 五、工业界怎么实现

目前主流框架对 Multi-Agent 各有不同的抽象方式。

**LangGraph**

把 Multi-Agent 建模成有向图：节点是 Agent 或工具，边是状态流转规则。Orchestrator 的逻辑可以建模为图中的控制流和状态转移，支持条件分支、循环和并行节点，也支持 supervisor、handoff、swarm 等多种 Multi-Agent 架构。Agent 之间通过 shared state 传递数据。适合需要精确控制执行流程的场景，图结构天然可视化，便于调试。

**AutoGen（微软）**

以"对话"为核心抽象。Agent 之间通过消息互相调用，支持 Group Chat 模式——多个 Agent 在同一个对话上下文里协作，由 GroupChatManager 决定下一个发言的是谁。适合需要多轮协商、Agent 之间有复杂交互的场景。AutoGen 0.4 对架构做了较大重构，引入了更清晰的 Agent Runtime 抽象。

**CrewAI**

角色化的 Multi-Agent 框架。每个 Agent 有明确的 role、goal 和 backstory，像给每个 Agent 写一份 JD。支持 sequential、hierarchical、hybrid 等流程形态，也可以通过异步执行提升并发能力。上手简单，适合快速搭建有明确角色分工的 Agent 团队。

**OpenAI Agents SDK**

OpenAI 官方把 Agents SDK 定位为此前 Swarm 实验的 production-ready upgrade，核心能力包括 Agent Loop、Tools、Handoffs、Guardrails 和 Tracing。Handoff 成为官方支持的原语——Agent A 把控制权连同上下文移交给 Agent B，Tracing 覆盖 LLM 调用、工具调用、handoff 和 guardrail 的完整链路。

**Google ADK（Agent Development Kit）**

Agent 作为可组合单元，支持嵌套调用——一个 Agent 可以把另一个 Agent 当作工具来调用（AgentTool）。支持 multi-agent orchestration、基于图的 workflow 和 evaluation。虽然对 Gemini 和 Google Cloud 生态有较好的优化，但 ADK 也强调更广泛的 LLM 支持和本地开发能力。

这五个框架虽然抽象方式不同，但解决的核心问题是一致的：**如何让多个 Agent 分工协作，同时保持整体可控、可观测、可调试。**



## 六、什么时候不该用 Multi-Agent

Multi-Agent 不是默认选项。

如果任务本身很短、依赖关系很强、结果需要高度一致，或者团队还没有 Trace、Eval、权限和状态管理的基础，单 Agent 加清晰 workflow 往往更稳。Multi-Agent 引入的协调开销、冲突处理和可观测性成本是真实存在的——拆错了比不拆更麻烦。

Multi-Agent 的收益来自合理分工，不来自 Agent 数量本身。



## 七、没有 Multi-Agent 的代价

在任务复杂度确实超出单 Agent 承载能力的场景下，强行用单 Agent 跑会面临三个问题：

**Context 膨胀，质量下降**：任务越复杂，上下文越长，模型推理质量越差。强行用单 Agent 跑复杂任务，到后期基本靠运气。

**串行跑，速度慢**：本来可以并行的子任务排成了队，用户等待时间线性增长。

**什么都能做，什么都做不精**：一个 Agent 塞了太多工具和背景知识，反而在每个子任务上都表现平庸。



## 最后说一句

Multi-Agent 不是银弹，拆得越细不代表越好。

分工的前提是每个 Agent 的边界清晰、通信成本可控、整体可观测。一个拆解混乱的 Multi-Agent 系统，比单 Agent 更难调试、更难排查、更难信任。

把前面几篇讲的东西串起来：Context Engineering 决定每个 Agent 看到什么，Tool Calling 决定它能做什么，Trace 让执行过程可见，Eval 让质量可量化，HITL 划定操作边界——这些是 Multi-Agent 系统能跑起来的基础设施，缺一不可。

下一篇，我们来聊 MCP（Model Context Protocol）——它为什么在短时间内被广泛采用，解决了什么真实问题。



## 参考资料

1. LangGraph 官方文档 — Multi-Agent 架构设计：https://langchain-ai.github.io/langgraph/concepts/multi_agent/
2. AutoGen 0.4 架构介绍：https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/index.html
3. CrewAI 官方文档：https://docs.crewai.com/introduction
4. OpenAI Agents SDK 文档：https://openai.github.io/openai-agents-python/
5. OpenAI Agents SDK Tracing：https://github.com/openai/openai-agents-python/blob/main/docs/tracing.md
6. Google ADK 官方文档：https://google.github.io/adk-docs/
7. Anthropic：Building effective agents（Multi-agent systems 章节）：https://www.anthropic.com/research/building-effective-agents
