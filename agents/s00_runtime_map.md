# s00 - Runtime Map

## 这一章的任务

这一章不负责解释每个细节。  
这一章只负责一件事：

**让你先看见 Claude Code 这台机器的整体轮廓。**

如果没有这张总图，后面每一章都会变成碎片：

- prompt 像一段提示词
- task 像一个附加 UI
- memory 像一个“顺便有的功能”
- team 像一个神秘的多助手模式

但从本地代码看，它们其实都是同一套运行时的不同层。

---

## 这一章最好怎么读

不要把这一章当成“摘要页”。  
它更像一张地铁图。

你第一次读时，不需要试图记住每一个函数名。  
你只需要先回答两个问题：

- Claude Code 这台机器大致分成哪几层
- 后面每一章分别在解释哪一层

只要这两件事先建立起来，后面章节就不会变成零散知识点。

---

## Claude Code 的总图

先把整个流程压成一幅图：

```text
User input
   |
   v
Runtime reads current state
   |
   +--> system prompt sections
   +--> tools
   +--> memory
   +--> task / todo state
   +--> permissions
   +--> worktree / team state
   |
   v
Anthropic Messages API
   |
   v
Streamed content blocks
   |
   +--> text
   +--> tool_use
   +--> thinking / compaction-related blocks
   |
   v
Tool execution / state updates
   |
   v
tool_result appended back into messages
   |
   v
Next turn
```

这张图里最重要的一点是：

Claude Code 并不是“用户说一句，模型答一句”。  
它真正做的是：

**把当前系统状态打包给模型，让模型决定动作，再把动作结果重新变成下一轮状态。**

---

## 为什么这张图重要

因为很多人对 Claude Code 的理解都停在错误的层级。

他们会说：

- “Claude Code 就是 Anthropic API 外面包了一层工具。”
- “Claude Code 就是一个会读文件、会跑命令的 CLI。”
- “Claude Code 就是 prompt + tools + shell。”

这些说法都有一点点对，但都太浅。

从本地代码看，Claude Code 至少还有下面这些你不能忽略的层：

- 长上下文管理
- 任务状态管理
- 多 agent 协作
- 隔离工作区
- 扩展加载
- 产品级平台能力

一旦把这些层拿掉，Claude Code 就不再是现在这台机器，而只剩一个“能调用工具的 loop”。

---

## 这台机器的九层结构

为了让后面的章节有位置感，我们先给 Claude Code 画一张九层剖面图。

### 第一层：API client 层

本地 bundle 直接构造了 Anthropic 相关资源对象，例如：

- `models`
- `messages`
- `files`
- `skills`

锚点：`cli.js:39`

这意味着 Claude Code 不是“借别人再包一层”，它本身就是正式客户端。

### 第二层：prompt assembly 层

这一层负责把：

- 角色约束
- 工具使用规范
- 环境信息
- memory
- MCP instructions
- scratchpad

拼成这一轮真正给模型看的 system prompt。

锚点：`cli.js:1480`

### 第三层：agent loop 层

这是整个系统的发动机。

它负责：

- 发请求
- 收流
- 识别 `tool_use`
- 执行工具
- 把 `tool_result` 回流

锚点：`cli.js:36`

### 第四层：tool surface 层

这层定义“模型到底能做哪些动作”。

本地 schema 可以直接看到的工具包括：

- Agent
- Bash
- FileRead / FileEdit / FileWrite
- Glob / Grep
- TodoWrite
- WebFetch / WebSearch
- AskUserQuestion
- MCP tools
- Worktree tools

锚点：`sdk-tools.d.ts:9`

### 第五层：context management 层

只要会话不是一次性，Claude Code 就必须管理：

- 历史长度
- compaction
- continuation summary
- tool result 的可持续利用

这一层的存在，是因为真实工作不会在两轮内结束。

### 第六层：task / memory 层

Claude Code 还维护两种不同的“长期状态”：

- 工作过程状态：task、todo、background
- 背景知识状态：memory、team memory

这两者都不是临时输出，而是会持续影响下一轮行为。

### 第七层：coordination 层

一旦有多个执行单元，就必须解决：

- 谁在做什么
- 谁能看见什么
- 谁来审批
- 谁来收尾

这就是 subagent、team、mailbox、protocol 的位置。

### 第八层：isolation 层

一旦多个 agent 并行修改同一个仓库，共享目录就会出问题。  
worktree、tmux、in-process teammate 都是为了解这个问题。

### 第九层：extension / platform 层

最后是那些让系统真正成为产品级 runtime 的能力：

- MCP
- skills
- plugins
- remote / channels
- auth
- voice
- telemetry

---

## 最小伪代码

如果把上面九层都压缩掉，Claude Code 可以被写成这样：

```python
def claude_code_runtime(user_input, state):
    tools = load_tools(state)
    system_prompt = build_system_prompt(state, tools)
    messages = state.messages + [user_input]

    while True:
        response = model(
            system=system_prompt,
            messages=messages,
            tools=tools,
        )

        messages.append(response.content)

        if not contains_tool_use(response):
            return messages

        results = execute_tools(response, state)
        messages.append(tool_results_to_message(results))

        state = update_runtime_state(messages, state)
```

这个伪代码故意写得很短，因为它想表达的是：

Claude Code 的复杂度不是来自“主循环难写”。  
复杂度来自 `load_tools`、`build_system_prompt`、`execute_tools`、`update_runtime_state` 这几步背后到底藏了多少系统。

---

## 本地代码锚点

这章不展开细节，只给你看总图对应的核心锚点。

### Anthropic client 资源

- `cli.js:39`

### system prompt 组装

- `cli.js:1480`

### 主 loop / ToolRunner

- `cli.js:36`

### tool schema

- `sdk-tools.d.ts:9`

### memory / task / team / worktree / skills

- memory：`cli.js:1256`
- task / todo：`cli.js:1286`
- team：`cli.js:2790`
- skills：`cli.js:3345`
- bash runtime：`cli.js:4588`
- worktree / CLI options：`cli.js:16726`

---

## 这一章真正要你记住什么

如果这一章读完，你脑子里只形成一张图，那应该是这张：

```text
Claude Code
  = 一个 agent loop
  + 一套状态系统
  + 一套协作系统
  + 一套隔离系统
  + 一套扩展系统
```

这张图一旦建立，后面每一章你都知道它在讲哪一层。  
没有这张图，后面每一章都只是零件说明书。
