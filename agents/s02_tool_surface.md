# s02 - Tool Surface

## 为什么第二章必须讲工具面

如果第一章讲的是“Claude Code 的发动机”，那这一章讲的就是它的四肢和感官。

因为一个 agent loop 再漂亮，如果它没有动作接口，就什么也做不了。  
模型可以推理、可以计划、可以判断，但如果它最终不能：

- 读文件
- 改文件
- 跑命令
- 搜索代码
- 问用户
- 派子 agent

那它就只能停留在“解释问题”的层面，而不是“解决问题”的层面。

所以这一章的真正问题不是：

“Claude Code 有哪些工具？”

而是：

**Claude Code 为什么要把动作接口设计成现在这个样子。**

---

## 这一章最好怎么读

不要把这一章当成“工具列表”。  
更好的读法是一直问：

- 为什么这个动作被做成独立工具
- 为什么它没有退回 Bash
- 为什么它的输入输出要被结构化

这样你读到的就不只是“Claude Code 有什么工具”，而是“Claude Code 如何把世界做成可被模型稳定操作的接口层”。

---

## Claude Code 不把所有动作都塞进 Bash

这是理解工具面的第一关键点。

很多人第一次设计 coding agent，会有一个很直接的想法：

“反正 shell 什么都能做，那给模型一个 Bash 不就够了。”

Claude Code 没有走这条路。

从本地 schema 看，它显式定义了很多工具：

- Agent
- Bash
- FileRead / FileEdit / FileWrite
- Glob / Grep
- TodoWrite
- WebFetch / WebSearch
- AskUserQuestion
- MCP 相关工具
- Worktree 相关工具

锚点：[`sdk-tools.d.ts:9`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L9)

这说明它做了一个非常清晰的架构选择：

**能结构化表达的动作，就不要先退化成 shell。**

原因很简单。

结构化动作比 Bash 更适合：

- 参数校验
- 权限控制
- UI 呈现
- 结果回流
- 教模型稳定使用

---

## Claude Code 的工具面分成哪几类

为了看清设计意图，我们不按文件名列工具，而按功能角色来分。

### 第一类：感知工具

这是模型“看世界”的入口：

- `FileRead`
- `Glob`
- `Grep`
- `WebFetch`
- `WebSearch`
- `ListMcpResources`
- `ReadMcpResource`

这类工具的作用，是把外部世界的信息转成模型可读内容。

### 第二类：行动工具

这是模型“动手”的入口：

- `FileEdit`
- `FileWrite`
- `Bash`
- `NotebookEdit`
- `Mcp`

这类工具的作用，是把模型的决策变成真实修改或真实操作。

### 第三类：治理工具

这是模型“管理过程”的入口：

- `TodoWrite`
- `TaskOutput`
- `TaskStop`
- `ExitPlanMode`
- `Config`

这类工具不直接改项目代码，但会改变运行时过程。

### 第四类：协作工具

这是模型“组织其他执行单元”的入口：

- `Agent`
- `EnterWorktree`
- `ExitWorktree`
- `AskUserQuestion`

这里已经不是单人单轮工作模式，而是运行时层面的协作与切换。

---

## 为什么 `FileRead` 这么重要

如果只看名字，`FileRead` 很普通。  
但从本地 schema 看，它实际上非常关键。

因为它说明 Claude Code 并不是只把“文件”理解成纯文本。

`FileReadOutput` 可以返回：

- 文本
- 图片
- notebook
- PDF
- PDF 拆页结果

锚点：[`sdk-tools.d.ts:88`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L88)

这意味着 Claude Code 在输入层已经是多模态的。  
模型不是只会读 `.ts` 和 `.py`。  
它还能把 notebook、图片、PDF 统一纳入自己的工作流。

这是一个很重要的架构信号：

Claude Code 的“感知层”比传统代码助手宽得多。

---

## 为什么 `Bash` 仍然不可替代

既然 Claude Code 已经做了这么多专门工具，那 Bash 还有什么不可替代的？

答案是：

总会有很多真实动作，暂时没有专门工具能很好覆盖，例如：

- 跑测试
- 跑构建
- 跑 git 命令
- 跑包管理器
- 跑项目脚本

所以 Claude Code 并没有排斥 Bash。  
它只是把 Bash 放在一个更成熟的运行时环境里。

从本地输出定义看，`BashOutput` 不只是：

- `stdout`
- `stderr`

它还可以带：

- `backgroundTaskId`
- `assistantAutoBackgrounded`
- `persistedOutputPath`
- `persistedOutputSize`

锚点：[`sdk-tools.d.ts:2165`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L2165)

这说明 Bash 在 Claude Code 里不是“裸命令”，而是被 runtime 托管的动作。

---

## 为什么 `Agent` 是最能暴露系统野心的工具

如果一个系统只提供文件和 shell 工具，那它还是单 agent 视角。  
一旦它把 `Agent` 本身变成工具，事情就变了。

因为这等于告诉模型：

“你不只是能做事，你还能再组织另一个执行单元去做事。”

从 `AgentInput` 可以直接看到：

- 有 `description`
- 有 `prompt`
- 有 `subagent_type`
- 有 `run_in_background`
- 有 `team_name`
- 有 `mode`
- 有 `isolation`

锚点：[`sdk-tools.d.ts:249`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L249)

这已经不是简单工具调用了。  
这是在把“委派”做成正式运行时动作。

---

## 最小伪代码

如果你自己实现一个最小工具层，核心大概长这样：

```python
TOOL_REGISTRY = {
    "FileRead": file_read,
    "FileEdit": file_edit,
    "FileWrite": file_write,
    "Glob": glob_search,
    "Grep": grep_search,
    "Bash": bash_run,
    "TodoWrite": todo_write,
    "Agent": spawn_agent,
}


def execute_tool(tool_use):
    name = tool_use.name
    input_data = validate_against_schema(name, tool_use.input)
    handler = TOOL_REGISTRY[name]
    return handler(input_data)
```

但 Claude Code 真正比这个更成熟的地方，在于：

- 它不是只有 registry
- 它还有 schema
- 还有权限
- 还有后台化
- 还有 UI 语义
- 还有结果持久化

也就是说，它不是“会调函数”，而是“有完整动作接口层”。

---

## 本地代码锚点

### 工具总表

- [`sdk-tools.d.ts:9`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L9)

### AgentInput

- [`sdk-tools.d.ts:249`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L249)

### BashInput

- [`sdk-tools.d.ts:287`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L287)

### FileEdit / FileRead / FileWrite

- [`sdk-tools.d.ts:349`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L349)
- [`sdk-tools.d.ts:367`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L367)
- [`sdk-tools.d.ts:385`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L385)

### Todo / Web / AskUserQuestion / Worktree

- [`sdk-tools.d.ts:514`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L514)
- [`sdk-tools.d.ts:524`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L524)
- [`sdk-tools.d.ts:534`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L534)
- [`sdk-tools.d.ts:548`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L548)
- [`sdk-tools.d.ts:2135`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L2135)

---

## 这一章真正想让你学会什么

如果这一章读完，你只记住一句话，我希望是这句：

**Claude Code 的工具面不是“让模型有更多按钮”，而是“把现实世界动作做成结构化接口”。**

这件事一旦成立，模型才能稳定地：

- 感知
- 行动
- 委派
- 协作

也正因为如此，工具面才是 Claude Code 这台机器的四肢，而不是附加插件。
