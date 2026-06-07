# Agent 为什么会"自己干活"？一文讲透 Agent Loop

你有没有注意到，当你把一个任务丢给 Cursor、Claude Code，或者国内的 Trae、通义灵码，它不会只给你一段代码就完事。

它会先读文件，再改代码，然后跑测试，发现测试挂了，再去分析报错，再改，再跑……几分钟后，问题解决了。

你没有一步一步指挥它。它自己就干完了。

这背后到底发生了什么？

答案是：**Agent Loop**。

---

## Agent 和 ChatGPT，差在哪里

要理解 Agent Loop，先得明白 Agent 和普通 ChatGPT 有什么本质区别。

普通的 ChatGPT 是这样工作的：

```
用户提问 → LLM → 输出回答 → 结束
```

一问一答，每次对话都是独立的。它没有能力主动采取行动，也没有办法根据结果调整自己的行为。

Agent 不一样：

```
用户提问 → LLM → 决定行动 → 调用工具 → 获取结果 → LLM → 再决定 → …… → 任务完成
```

最关键的变化是多了一个**循环**：LLM 不只是输出文字，而是可以调用工具，看到工具返回的结果，然后再次思考，再次行动。

这个循环，就是 Agent Loop。

Agent 之所以能"自己干活"，靠的就是这个循环不断地转下去。

> 📌 【插图一：ChatGPT vs Agent 对比图】

---

## Agent Loop 长什么样

Agent Loop 的理论基础，来自一篇 2022 年的论文——**ReAct**。

提出这个方法的是姚顺雨，清华姚班出身，普林斯顿计算机博士，博士期间就在 Language Agent 方向做出了奠基性贡献。毕业后加入 OpenAI，是 Operator 和 Deep Research 的核心贡献者，入选 MIT 科技评论 2024 年"35 岁以下科技创新 35 人"中国区名单，也是其中最年轻的一位。2025 年底，27 岁的他被腾讯延揽，出任首席 AI 科学家，兼任 AI Infra 和大语言模型部门负责人。

ReAct 把 Agent Loop 拆解成三个步骤，循环往复：

**Reasoning（推理）**：现在情况是什么？我下一步该做什么？

**Acting（行动）**：执行一个具体的动作，比如搜索、读文件、调接口。

**Observation（观察）**：动作执行完了，结果是什么？

然后回到 Reasoning，开始下一轮。

用人话说就是：**想清楚 → 动手干 → 看结果 → 再想清楚**。

> 📌 【插图二：ReAct 循环示意图】

这个循环听起来简单，但它是今天几乎所有 Agent 框架——LangGraph、OpenAI Agents SDK、Claude Code——背后共同的底层逻辑。

---

## 一个真实的例子

光说概念不够直观，来看一个具体的例子。

假设线上出现了报错，你把一段错误日志丢给 Agent，让它帮你排查原因。它不会一次性给你答案，而是一轮一轮地推进：

**第 1 轮**
- Reasoning：先看看报错信息是什么
- Action：读取错误日志
- Observation：`NullPointerException`，出在 `UserService.getProfile()` 第 42 行

**第 2 轮**
- Reasoning：去看看这段代码
- Action：读取 `UserService.java` 第 42 行附近
- Observation：代码里没有判断 `user` 是否为 null，直接调用了 `user.getId()`

**第 3 轮**
- Reasoning：为什么 user 会是 null？去看调用方
- Action：搜索 `getProfile` 的调用位置
- Observation：发现调用方传入了一个不存在的 userId，数据库查询返回了 null

**第 4 轮**
- Reasoning：根因找到了，整理结论
- Action：生成根因分析报告
- Observation：输出完成

**结束。**

整个过程，Agent 自己决定每一步做什么，根据上一步的结果决定下一步的行动。你只是在最开始丢了一段报错日志。

> 📌 【插图三：4 轮 Loop 过程图】

---

## Loop 什么时候停

Agent Loop 不会无限转下去。它有终止条件。

常见的有三种：

**1. 模型主动宣布完成**：LLM 判断任务已经达成，不再输出 Action，直接给出最终回答。这是最理想的情况。

**2. 达到最大轮数**：作为兜底机制，防止 Agent 陷入死循环。OpenAI Agents SDK 里这个参数叫 `max_turns`，默认是 10。

**3. 工具调用出错**：某个工具抛出了异常，或者返回了无法处理的结果，Agent 中止并报告错误。

这三种终止逻辑在工程实现里都要显式处理，是 Agent 开发绕不过去的基础设计。

---

## Loop 本身不是难点

等你真正动手实现一个 Agent Loop，会发现：核心逻辑其实很简单。

```python
while not done:
    action = llm.think(context)
    result = tool.run(action)
    context.update(result)
```

一个下午就能写出来。

真正难的部分在别处：**每次调用 LLM 时，"context"里该放什么？**

第 1 轮的日志要不要传？第 3 轮的中间结果要不要保留？随着 Loop 不断执行，传给模型的上下文会越来越长，最终超出 Token 限制。

这就是所谓的 **Context Engineering** 问题——怎么在有限的窗口里，把最有用的信息喂给模型。

这是 Agent 工程里真正有技术含量的部分，也是下一篇要专门讲的内容。

---

## 最后说一句

理解了 Agent Loop，你就理解了所有 Agent 框架的核心抽象。

不管是 LangGraph 的节点流转，OpenAI Agents SDK 的 `Runner.run()`，还是 Claude Code 在你的终端里自动跑命令——本质上都是同一件事：**一个 LLM 在一个循环里，不断思考、行动、观察，直到任务完成。**

Agent 工程师，本质上是在给 LLM 写 Runtime 的人。

这是系列的第一篇，后面会继续拆 Context Engineering、Tool Calling 和 Agent Runtime。感兴趣可以关注。
