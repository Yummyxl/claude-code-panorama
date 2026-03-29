# s04 - Compaction And Memory

## 为什么这一章特别容易学偏

只要一个系统里同时出现：

- 长会话
- memory
- summary
- compact

很多人就会自然把它们混成一团，觉得都是“记忆机制”。

Claude Code 这套本地代码非常清楚地告诉你：

**不是。**

在 Claude Code 里，至少有两种完全不同的东西：

- `compaction`
- `memory`

它们长得都像“给模型补背景”，但解决的是完全不同的问题。

---

## 这一章最好怎么读

最有效的读法，是始终把两个问题分开：

- “刚刚这一大段会话，下一轮怎么继续带下去？”
- “有哪些长期事实，以后每轮最好都知道？”

前者是 compaction。  
后者是 memory。

只要你读这章时始终不把这两个问题混在一起，Claude Code 后面的 prompt、TeamMem、continuation summary 就都会清晰很多。

---

## compaction 解决的是什么问题

compaction 解决的是：

**当前会话历史太长了，下一轮再原样塞给模型不现实。**

也就是说，compaction 面对的是“刚刚这段会话怎么继续带下去”。

它本质上处理的是：

- 当前对话的历史
- 上下文窗口的压力
- 下一轮 request 的可持续性

所以它是“会话维护机制”。

从本地 bundle 可见，Claude Code 非常强调：

- 不要只保留文本
- 必须保留完整 `response.content`
- compaction block 本身就是下一轮状态的一部分

锚点：

- `cli.js:9689`
- `cli.js:12624`

---

## memory 解决的是什么问题

memory 解决的是：

**有哪些长期事实，模型以后每轮最好都知道。**

它处理的不是“刚刚这一轮聊了什么”，而是更长期的背景，例如：

- 用户偏好
- 项目背景
- 团队共享信息
- 本地记忆文件

所以它是“长期背景注入机制”。

从本地 memory 区域可以直接看到，它至少区分这些来源：

- `Managed`
- `User`
- `Project`
- `Local`
- `AutoMem`
- `TeamMem`

锚点：`cli.js:1256`

---

## 这两个机制最大的区别

我们把它们并排写出来：

```text
Compaction
  = 为了让“当前会话”继续跑下去
  = 处理历史消息
  = 属于上下文维护

Memory
  = 为了让“长期背景”持续存在
  = 处理稳定事实
  = 属于 prompt 注入
```

这就是整章最重要的一张对照表。

如果你把两者混掉，后面几乎所有机制都会理解错：

- 你会误解为什么 memory 放在 system prompt
- 你会误解为什么 compaction 要保留 response.content
- 你会误解 TeamMem 的意义

---

## Claude Code 是怎么把它们接进系统的

这才是工程上最值得学的地方。

### compaction 的接入点

compaction 接在消息历史这一侧。  
也就是说，它属于主循环内部的上下文维护。

简单说就是：

- 这一轮消息太长了
- 系统需要把它压缩成还能继续使用的形式
- 下一轮继续基于压缩后的状态跑

所以 compaction 的位置更靠近 `messages[]`。

### memory 的接入点

memory 则接在 system prompt 这一侧。

从本地 `PD(...)` 可以直接看到：

- `memory` 是一个正式 section

锚点：`cli.js:1480`

这意味着 memory 不是“从历史里自动推理出来的东西”，而是运行时在每轮 request 前主动注入的背景材料。

---

## 最小伪代码

为了把这个区别彻底固定下来，我们把两者放进同一个伪代码里：

```python
def build_system_prompt(state):
    sections = []
    sections.append(base_role_prompt())
    sections.append(environment_prompt(state))
    sections.append(memory_section(state.memory_sources))
    return sections


def next_turn(messages, state):
    response = model(
        system=build_system_prompt(state),
        messages=messages,
        tools=state.tools,
    )

    messages.append(response.content)

    if contains_tool_use(response):
        messages.append(execute_tools(response))

    if context_too_large(messages):
        messages = compact(messages)

    return messages
```

一眼就能看出：

- memory 在 `build_system_prompt`
- compaction 在 `next_turn` 的历史维护里

这就是 Claude Code 里两者的结构差别。

---

## 本地代码锚点

### memory 是 system prompt section

- `cli.js:1480`

### memory 来源枚举

- `cli.js:1256`

### compaction block 必须保留完整内容

- `cli.js:9689`
- `cli.js:12624`

### compaction 也是正式 hook 生命周期

本地还能看到：

- `PreCompact`
- `PostCompact`

锚点：

- `cli.js:8162`
- `cli.js:14796`

这说明 compaction 不是内部小技巧，而是正式生命周期事件。

---

## 这一章真正想让你学会什么

这一章最重要的结论也是一句话：

**memory 是长期背景，compaction 是会话维护。**

这两个机制都在帮助模型持续工作。  
但它们帮的是完全不同的事情。

把这件事彻底想明白，你后面看：

- TeamMem
- continuation summary
- system prompt injection
- 长会话行为

就都不会再混乱。
