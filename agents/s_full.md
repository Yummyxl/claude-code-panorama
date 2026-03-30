# s_full - Claude Code 全景图

## 如果你只读一章，就读这一章

这一章的任务很简单：

**把前面所有分章重新拼成一整台机器。**

你在单章里会分别看到：

- agent loop
- prompt 组装
- tools
- compaction
- memory
- task
- subagent
- team
- worktree
- MCP
- plugins
- telemetry

但如果这些东西一直分开看，你很容易产生一个错觉：

“Claude Code 好像只是有很多功能。”

这其实是最妨碍学习的一种理解。

Claude Code 不是“很多功能的集合”。  
Claude Code 是一台机器。  
这些功能之所以存在，不是因为产品菜单需要它们，而是因为这台机器要长期工作，就必须有这些部件。

---

## 这一章最好怎么读

这一章不是替代前面的细章。  
它更像总装图。

最好的读法是：

- 如果你还没有整体感，先用这一章建立总图
- 如果你已经读过分章，再回到这一章把零散部件重新装回一台机器

所以它的作用不是“再讲一遍”，而是“把前面所有机制重新拼成一个整体运行时”。

---

## Claude Code 到底是什么

我们先把最终结论放在最前面。

**Claude Code 是一个 agent harness。**

这个定义里每个词都重要。

### 1. 它是“本地”的

因为它直接运行在你的机器上，直接看你的目录，直接调用你的 shell，直接维护你的 worktree、文件、任务、消息历史。

### 2. 它是“agent”的

因为它不是一次性补全文本，而是在持续做这件事：

- 看当前状态
- 决定下一步动作
- 调工具
- 读取结果
- 再决定下一步动作

### 3. 它是“harness”

因为真正做决策的主体仍然是模型。  
Claude Code 的工作，是把模型放进一个能持续工作的环境里。

这个环境包括：

- prompt
- tools
- state
- permissions
- context management
- team coordination
- isolation
- extension runtime

所以理解 Claude Code，不能只问“它会不会调工具”。  
更要问：

**它是怎么把模型和真实工作环境接上的。**

---

## 把 Claude Code 压缩成一条循环

如果你把所有复杂机制暂时剥掉，Claude Code 最小的本质其实非常朴素：

```text
messages
  -> model
  -> response blocks
  -> tool_use?
  -> execute tools
  -> tool_result
  -> append back to messages
  -> next turn
```

这就是最小 agent harness。

一切复杂度都来自一个事实：

这个循环如果要在真实编程工作里长期跑下去，就不能只靠这几行伪代码。

因为真实世界里会立刻冒出一堆问题：

- prompt 要怎么拼
- 工具要怎么定义
- 命令太慢怎么办
- 历史太长怎么办
- 用户中途改主意怎么办
- 多个 agent 怎么协作
- 并行修改代码怎么避免互相踩踏
- 外部服务怎么接进来

Claude Code 的全部工程价值，就在于它把这些问题逐个做成了运行时机制。

---

## Claude Code 的九层结构

为了让整台机器一眼看清，我们把它分成九层。

### 第一层：模型交互层

这是最核心的一层。

它负责：

- 维护消息历史
- 组装 system prompt
- 把 tools 作为能力面暴露给模型
- 调 Messages API
- 接收流式 content blocks

这里决定了：模型这一轮到底看见什么。

### 第二层：工具执行层

当模型要求 `tool_use` 时，Claude Code 要负责：

- 找到对应工具
- 校验参数
- 执行动作
- 把结果包装回 `tool_result`

这里决定了：模型的意图如何变成真实动作。

### 第三层：上下文管理层

真实工作不会只跑一轮，所以 Claude Code 必须处理：

- 长消息历史
- compaction
- continuation state
- tool result 摘要

这里决定了：系统怎样在长会话里不崩掉。

### 第四层：长期背景层

除了“刚刚发生了什么”，模型还需要知道一些长期事实：

- 用户偏好
- 项目背景
- 团队共享知识
- 记忆文件

这里就是 memory 的位置。

### 第五层：任务状态层

Claude Code 不只是在调工具，它还在维护工作过程：

- task
- todo
- plan mode
- background commands

这里决定了：模型是不是能把一个大任务拆开并持续追踪。

### 第六层：多 agent 协作层

一旦任务变大，一个 agent 不够，就会出现：

- subagent
- teammate
- mailbox
- approval protocol
- shutdown protocol

这里决定了：多个执行单元如何协调，而不是互相污染。

### 第七层：隔离层

多个 agent 并行工作时，最危险的问题之一是目录和 git 状态冲突。  
Claude Code 用 worktree、tmux、in-process 模式来处理这个问题。

这里决定了：并行执行是不是可靠。

### 第八层：扩展层

Claude Code 不把所有能力硬编码死在 bundle 里。它还支持：

- MCP
- skills
- plugins
- output styles
- remote / channel / bridge

这里决定了：这台机器是不是可扩展。

### 第九层：产品化平台层

最后还有那些表面不那么显眼、但产品运行必不可少的部分：

- auth
- voice
- telemetry

这里决定了：Claude Code 是不是一个可交付、可运营、可观察的系统，而不只是一个玩具 loop。

---

## 为什么说它是一台机器，而不是一堆 feature

因为这些层彼此不是松散并列关系，而是严格耦合的。

举个最典型的例子。

假设模型这一轮决定去跑测试。

表面上看，只发生了一件事：  
“Bash 跑了一个命令。”

但对 Claude Code 来说，至少同时有这些层在参与：

- system prompt 告诉模型尽量优先用专门工具
- tool registry 暴露了 Bash
- permission system 判断这个命令是否允许
- background runtime 决定是否自动后台化
- todo/task 层更新当前任务状态
- telemetry 记录命令执行
- messages 层准备把结果再交回模型

你看到的是一个命令。  
Claude Code 实际运行的是一整条状态链。

这就是“机器思维”和“功能思维”的区别。

---

## Claude Code 的核心哲学

如果把实现中展现出来的架构倾向浓缩一下，可以总结成四句话。

### 1. 把结构化的事情结构化

能做成工具 schema 的，不放任成自然语言。  
能做成 typed protocol 的，不放任成自由文本。  
能做成 runtime state 的，不让模型自己硬记。

### 2. 把长期工作和单轮问答区分开

Claude Code 从设计上就不是一次性问答器，所以它必须有：

- compaction
- memory
- task
- background
- team

### 3. 把协作和隔离同时做出来

多 agent 不是只要“能发消息”就够了。  
如果没有 worktree 这类隔离层，协作越强，互相踩踏就越严重。

### 4. 把 prompt 当成运行时的一部分

Claude Code 的 prompt 不是开场白。  
它是系统控制面的一部分：

- 环境怎么注入
- memory 怎么注入
- tools 怎么使用
- 输出怎么约束

这些都是运行时，而不只是文案。

---

## 一次真实工作是怎么在这台机器里流动的

我们把整个系统放进一个具体场景里看。

用户说：

“修掉测试失败，并告诉我根因。”

Claude Code 不会直接把这句话扔给模型就结束。它大致会做下面这些事。

### 第一步：准备这一轮输入

运行时先收集当前状态：

- 当前消息历史
- 当前 system prompt sections
- 当前 tools
- 当前工作目录和是否处于 worktree
- 当前 memory
- 当前 permission mode

然后把这些东西一起发给模型。

### 第二步：模型决定动作

模型看到这些状态后，可能会先做两件事：

- 写一个 todo
- 调 `Bash` 跑测试

此时 Claude Code 不会“脑补执行”，而是进入工具执行路径。

### 第三步：运行时执行工具

如果测试命令很长，Claude Code 可能会选择把它后台化。  
如果命令结果很大，Claude Code 可能会把输出落盘。  
如果命令受权限限制，Claude Code 可能会先发起审批。

这些都不是模型自己做的，而是 runtime 在执行模型动作时附加的系统行为。

### 第四步：结果回流给模型

测试失败的输出不会只显示给用户看。  
更关键的是，它会变成 `tool_result`，重新进入消息历史。

于是模型下一轮才能继续分析：

- 是哪个测试失败
- 根因更可能在什么位置
- 下一步应该读哪些文件

### 第五步：如果任务变大，系统继续放大能力

如果问题太复杂，模型可能再派一个 subagent 去调查某个模块。  
如果多个 agent 同时改代码，系统可能给它们进入不同 worktree。  
如果历史太长，系统可能触发 compaction。  
如果用户有长期偏好，memory section 会继续影响之后的每一轮。

整个过程不是一连串孤立功能，而是一条被不断放大的主循环。

---

## 每一章在整台机器里的位置

你可以把这个仓库理解成对这台机器的拆机顺序。

### `s00` 到 `s03`

先教你看见机器的骨架：

- 总图
- 主循环
- 工具面
- prompt 组装

### `s04` 到 `s05`

再教你看见“长期工作”所需的状态层：

- compaction
- memory
- task
- todo
- background

### `s06` 到 `s07`

接着教你看见“并行执行”的代价与解法：

- subagent
- team
- protocol
- worktree

### `s08` 到 `s09`

再看这台机器怎样和外部世界接起来、怎样被产品化：

- MCP
- skills
- plugins
- remote
- auth
- voice
- telemetry

### `s12` 到 `s15`

最后把最关键的四类材料单独收束：

- tools 的定义手册
- prompt 的原文和解释手册
- 单轮运行时调用链
- 完整渲染 prompt 场景手册

这样你读完全仓库之后，不会只记住“这里有个概念”，而是知道：

- 这个概念在整台机器里的位置
- 它为什么存在
- 它依赖什么
- 它影响什么

---

## 这一章真正想让你学会的东西

如果读完这一章，你只记住一句话，我希望是这句：

**Claude Code 的本质，不是一个会聊天的 CLI，而是一台把模型接进真实工作环境的 agent harness。**

如果你记住两句话，那第二句应该是：

**你看到的每一个“功能”，其实都是为了让主循环在真实世界里长期工作下去。**

一旦按这个角度看，后面的所有章节都会突然变得非常清楚。  
因为它们不再是分散的功能笔记，而是同一台机器的不同部件。
