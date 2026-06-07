# Agent 的骨架：一文讲透 Agent Runtime

前三篇讲了 Agent Loop、Context Engineering 和 Tool Calling。

如果你把这三篇的内容拼在一起，会发现一件事：Loop 要转、Context 要组装、工具要调用、结果要处理——这些事情不会自己发生。

背后有一个东西在协调，它就是 Agent Runtime。


## 一、Runtime 是什么

先从一个真实的困境说起。

你写好了 Agent Loop，LLM 接好了，工具也写好了。跑起来之后发现：

- 工具调用失败了，没人处理，Loop 直接崩
- 跑了十几轮还没结束，没有终止条件
- 出了问题想排查，没有任何日志
- 某个敏感操作被 LLM 随意调用，没有权限控制

代码能跑，但像一辆没有底盘的车——引擎有了，但传动、刹车、仪表盘全都缺。

这些"脏活"，就是 Agent Runtime 负责的事。

**一句话定义：Runtime 是驱动 Agent Loop 运转的执行框架，负责把 LLM、工具、状态、权限和日志串成一个可运行的系统。**

LLM 是引擎，Runtime 是底盘。没有底盘，引擎再好也上不了路。

> 📌 【插图一：Runtime 在 Agent 中的位置示意图】


## 二、Runtime 具体负责什么

如果说 Agent Loop 是循环，Context Engineering 是每轮喂给模型的信息，Tool Calling 是模型伸向外部的手，那么 Runtime 就是负责调度这一切的执行层。

把 Runtime 的职责拆开来看，通用 Agent 里至少有这七件事：

**工具管理（ToolRegistry）**
Runtime 统一注册和管理工具，并把当前可用工具的 schema 提供给模型；模型只需要基于这些描述决定是否调用，不需要知道工具怎么实现的。

**上下文组装**
每轮调用 LLM 之前，Runtime 负责把 System Prompt、执行历史、工具定义、当前状态拼成完整的 Context 塞给模型。这是 Context Engineering 的执行层。

**状态管理（State）**
当前跑到第几轮、已经收集了哪些信息、任务进展如何——这些运行时状态由 Runtime 维护，不是 LLM 的事。LLM 每次调用都是无状态的，状态要靠外部系统保存。

**终止判断**
Agent Loop 什么时候停？达到最大轮数、模型返回最终答案、不再请求工具调用、工具连续失败——Runtime 负责执行这些终止逻辑。没有明确的终止条件，Loop 就可能无限跑下去。

**风险控制**
哪些操作需要人工审批？哪些工具有调用频率限制？哪些参数需要脱敏处理？Runtime 在 LLM 和工具执行之间做一层拦截，防止 Agent 做出不该做的事。

**Trace（链路追踪）**
每一轮发生了什么、LLM 输出了什么、工具返回了什么——Runtime 把这些信息记录成完整的执行链路。没有 Trace，出了问题就是黑盒，什么都查不了。

**可观测性（Observability）**
基于 Trace 数据，能实时看到 Agent 的运行状态、历史行为、异常信号。Trace 是记录，Observability 是在这些记录上建立的"眼睛"。这两者会在后续文章单独展开。

对于复杂的 Agent，还会多一层：

**路由（Router）**
当 Agent 需要处理多种不同类型的任务时，Router 负责判断用户意图、分发到对应的 workflow。简单 Agent 可以不单独设计 Router，由提示词和工具选择承担简单分发；复杂 Agent 有多个 workflow 时，才需要显式路由。

> 📌 【插图二：Runtime 职责全景图】


## 三、没有 Runtime 会怎样

来做一个直接的对比。

**没有 Runtime 的 Agent：**

```python
while True:
    action = llm.call(context)
    result = tool.run(action)
    context.append(result)
```

能跑，但：工具失败直接抛异常、没有日志、没有权限控制、不知道什么时候结束、出了问题无从排查。这是一个脚本，不是一个系统。

**有 Runtime 的 Agent：**

同样的 Loop，但每一步都有人兜底——工具调用前校验权限，执行结果写入 Trace，异常触发重试或终止，敏感操作等待人工确认。

这才是一个可以交付、可以维护、可以信任的系统。

> 📌 【插图三：有无 Runtime 的对比图】


## 四、主流框架的 Runtime 长什么样

你不一定需要从零实现 Runtime。现在已经有不少框架在帮你处理这些问题，比如：

**LangGraph**：以图结构组织 workflow，节点是 LLM 调用或工具执行，边是状态转移。适合流程复杂、分支多的 Agent。

**OpenAI Agents SDK**：Runner 驱动 Loop，tools 统一注册，内置 handoff 机制支持多 Agent 协作。接口简洁，上手快。

**自研 Runtime**：可控性最强，可以按需裁剪每一个模块，代价是要自己处理所有工程细节。适合对系统有深度定制需求的场景。

框架选哪个，取决于你的场景复杂度和对可控性的要求。但无论选哪个，背后解决的都是同一批问题。


## 最后说一句

理解了 Runtime，你就理解了为什么 Agent 开发不只是"调 LLM API"。

LLM 可以被替换，但不是无成本的替换：不同模型的工具调用格式、上下文窗口、推理风格和稳定性都会影响系统表现。Runtime 的价值在于，把这些差异尽量封装起来，让 Agent 系统更稳、更安全、更好维护。

下一篇，我们来拆 Memory 体系——Agent 怎么记住东西，短期记忆、Working Memory 和长期存储各自怎么设计。
