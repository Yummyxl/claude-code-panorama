# Claude Code 本地运行时拆解

## 先把最大的误解拆掉

很多人一看到 Claude Code，就会把它理解成三选一里的某一种：

- 一个会聊天的 CLI
- 一个会调工具的脚本壳
- 一堆 prompt 和命令的拼装器

这三种理解都不够。

**从实现上看，Claude Code 更准确的身份是：一个围绕大模型构建出来的 agent harness。**

它不是模型本身。  
它也不是“替模型思考”的控制器。  
它做的事情，是把模型放进一个可以真正工作的环境里。

这个环境至少包括：

- 可以读写代码的工具
- 可以执行 shell 命令的终端
- 可以维持长会话的上下文机制
- 可以分解任务和维护 todo 的状态层
- 可以派生子 agent、组织 teammate 的协作层
- 可以隔离目录的 worktree 机制
- 可以接入 MCP、skills、plugins 的扩展层
- 可以处理认证、语音、遥测的产品层

换句话说，Claude Code 的价值不在于“它比模型更聪明”，而在于：  
**它给模型准备好了一个能持续感知、持续行动、持续协作的工作空间。**

---

## 这个仓库真正想教你的是什么

这个仓库不是在教你“怎么背出 Claude Code 有哪些功能”。

它要教的是更底层的东西：

**一个能长期工作的 coding agent，到底需要哪些运行时机制。**

你如果只看产品表面，很容易得出一些非常浅的结论：

- Claude Code 会读文件
- Claude Code 会跑命令
- Claude Code 会改代码
- Claude Code 能多 agent

这些结论都不假，但学习价值很低。  
真正有价值的问题是：

- 为什么它不是直接把所有事都扔给 Bash
- 为什么它要把 prompt 拆成 section
- 为什么 memory 和 compaction 必须分开
- 为什么 team 不能只靠自然语言协作
- 为什么 worktree 是多 agent 的基础设施而不是附加功能
- 为什么 tool result 必须回流进消息历史

你把这些问题想明白，才算真正学会 Claude Code。

---

## 什么是 harness

为了避免后面所有章节都漂在半空里，这里先给出一个工作定义。

在本项目里，`harness` 指的是模型工作的外部环境总和。

可以把它压缩成下面这条公式：

```text
Claude Code Harness
  = Prompt Assembly
  + Tools
  + Runtime State
  + Permissions
  + Context Management
  + Team Coordination
  + Isolation
  + Extensions
```

把每一项翻成人话：

- `Prompt Assembly`
  这一轮到底给模型看什么
- `Tools`
  模型这一轮到底能做什么动作
- `Runtime State`
  会话现在处在什么状态
- `Permissions`
  哪些动作允许，哪些动作要拦
- `Context Management`
  长会话怎样不爆炸
- `Team Coordination`
  多个 agent 怎样配合
- `Isolation`
  并行执行怎样不互相踩踏
- `Extensions`
  外部工具和能力怎样接进来

模型负责决策。  
harness 负责把这个决策变成真实世界中的动作，并把动作结果重新带回模型。

所以你可以把 Claude Code 看成：

```text
Claude Code
  = 一个 agent loop
  + 一套工具面
  + 一套状态管理
  + 一套协作协议
  + 一套隔离机制
  + 一套扩展运行时
```

---

## 这个仓库的分析对象是什么

这点要说得非常严格。

本仓库只分析当前交付实现，不做线上仓库对照，不拿公开文档反推实现。

主分析材料包括：

- `实现主干`
- `工具定义`
- `包定义`
- `vendor 目录`

这些都是实现引用，所以文档里保留为代码引用，不做仓库内超链接。

因此，这个仓库里所有靠谱的结论，都应该满足一条标准：

**要么能在实现里直接看到，要么能从实现里稳定推出。**

---

## Claude Code 的最小本质

如果你把所有功能都剥掉，只剩一个不可再简化的核心，那么 Claude Code 的最小本质就是这条循环：

```python
def agent_loop(messages):
    while True:
        response = model(messages, tools, system_prompt)
        messages.append(response.content)

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                results.append(run_tool(block))

        messages.append(tool_results_to_message(results))
```

这就是最小 agent harness。

后面你会看到，Claude Code 所有高级功能都只是往这个循环外挂机制：

- prompt sections
- background tasks
- compaction
- memory
- task system
- subagents
- mailbox
- worktree
- MCP
- plugins

它们再复杂，也都必须回到同一个问题：

**这一轮给模型什么，模型做了什么，结果怎么再回到下一轮。**

---

## 学这个仓库，你应该建立的心智转换

很多人学 agent 产品，关注点总是错的。

他们问：

- 这个 prompt 怎么写
- 这个工具怎么调
- 这个多 agent 怎么发消息

这些问题当然有用，但都不是第一性问题。

你更应该问：

- 这个系统把哪些东西当成“一等运行时状态”
- 哪些能力是 prompt 驱动，哪些能力是 runtime 驱动
- 哪些地方靠文本约束，哪些地方靠结构化协议
- 哪些地方靠约定，哪些地方靠隔离

这就是从“会用 Claude Code”到“会理解 Claude Code”的区别。  
也是从“会做一个 AI 功能”到“会做一个 agent harness”的区别。

---

## 这个仓库最好怎么学

不要把它当成一本需要从头背到尾的手册。  
更好的方法是分三轮：

### 第一轮：先建立总图

先看：

- [s00_runtime_map.md](./agents/s00_runtime_map.md)
- [s01_agent_loop.md](./agents/s01_agent_loop.md)
- [s02_tool_surface.md](./agents/s02_tool_surface.md)
- [s03_system_prompt.md](./agents/s03_system_prompt.md)

这一轮的目标不是记细节，而是先看清 Claude Code 这台机器到底分哪几层。

### 第二轮：再学长期工作机制

再看：

- compaction / memory
- task / todo / background
- subagent / team / worktree

这一轮的目标，是理解 Claude Code 为什么能连续工作，而不是只会做一轮。

### 第三轮：最后啃高密度手册

最后再去看：

- [s12_tools_catalog.md](./agents/s12_tools_catalog.md)
- [s13_prompt_catalog.md](./agents/s13_prompt_catalog.md)
- [s14_toolrunner_callgraph.md](./agents/s14_toolrunner_callgraph.md)
- [s15_complete_rendered_prompts.md](./agents/s15_complete_rendered_prompts.md)

这一轮才是查表、对照源码、确认实现细节。

如果你按这个顺序来，这套文档会更像课程，而不是材料堆。

---

## 阅读路线

这个仓库不是按“功能菜单”组织的，而是按“机制如何层层叠起来”组织的。

### 第一层：先看骨架

先读这几章，建立 Claude Code 的整体画面：

1. [s00_runtime_map.md](./agents/s00_runtime_map.md)
2. [s01_agent_loop.md](./agents/s01_agent_loop.md)
3. [s02_tool_surface.md](./agents/s02_tool_surface.md)
4. [s03_system_prompt.md](./agents/s03_system_prompt.md)
5. [s_full.md](./agents/s_full.md)

你读完这五章，应该能回答：

- Claude Code 的主循环是什么
- 工具面为什么不是一袋零散命令
- prompt 为什么是 section 化的
- 为什么说它是 harness 而不是聊天壳

### 第二层：看状态管理

再读这几章，理解它怎么维持长时工作：

6. [s04_compaction_and_memory.md](./agents/s04_compaction_and_memory.md)
7. [s05_task_todo_background.md](./agents/s05_task_todo_background.md)
8. [s11_memory_prompt_injection.md](./agents/s11_memory_prompt_injection.md)

你读完这一层，应该能回答：

- 为什么 memory 不是 compaction
- 为什么 todo 和 task 不只是 UI
- 为什么后台任务是 runtime 能力而不是 Bash 小技巧

### 第三层：看多 agent 与隔离

接着读：

9. [s06_subagent_and_teams.md](./agents/s06_subagent_and_teams.md)
10. [s10_team_protocols.md](./agents/s10_team_protocols.md)
11. [s07_worktree_and_isolation.md](./agents/s07_worktree_and_isolation.md)

你读完这一层，应该能回答：

- subagent 和 team 有什么本质区别
- 为什么 team 不能靠普通文本协作
- 为什么 worktree 是多 agent 的基础设施

### 第四层：看扩展与产品化能力

最后读：

12. [s08_mcp_plugins_remote.md](./agents/s08_mcp_plugins_remote.md)
13. [s09_auth_voice_telemetry.md](./agents/s09_auth_voice_telemetry.md)
14. [s12_tools_catalog.md](./agents/s12_tools_catalog.md)
15. [s13_prompt_catalog.md](./agents/s13_prompt_catalog.md)
16. [s14_toolrunner_callgraph.md](./agents/s14_toolrunner_callgraph.md)
17. [s15_complete_rendered_prompts.md](./agents/s15_complete_rendered_prompts.md)

这几章解决的是：

- Claude Code 怎么接外部能力
- Claude Code 怎么把 prompt 和 tools 做成正式接口
- Claude Code 一轮执行到底怎样流动
- Claude Code 在不同运行条件下到底给模型看了什么
- Claude Code 怎么从“黑盒体验”变成“可解释系统”

---

## 全部章节

- [agents/s00_runtime_map.md](./agents/s00_runtime_map.md)
- [agents/s01_agent_loop.md](./agents/s01_agent_loop.md)
- [agents/s02_tool_surface.md](./agents/s02_tool_surface.md)
- [agents/s03_system_prompt.md](./agents/s03_system_prompt.md)
- [agents/s04_compaction_and_memory.md](./agents/s04_compaction_and_memory.md)
- [agents/s05_task_todo_background.md](./agents/s05_task_todo_background.md)
- [agents/s06_subagent_and_teams.md](./agents/s06_subagent_and_teams.md)
- [agents/s07_worktree_and_isolation.md](./agents/s07_worktree_and_isolation.md)
- [agents/s08_mcp_plugins_remote.md](./agents/s08_mcp_plugins_remote.md)
- [agents/s09_auth_voice_telemetry.md](./agents/s09_auth_voice_telemetry.md)
- [agents/s10_team_protocols.md](./agents/s10_team_protocols.md)
- [agents/s11_memory_prompt_injection.md](./agents/s11_memory_prompt_injection.md)
- [agents/s12_tools_catalog.md](./agents/s12_tools_catalog.md)
- [agents/s13_prompt_catalog.md](./agents/s13_prompt_catalog.md)
- [agents/s14_toolrunner_callgraph.md](./agents/s14_toolrunner_callgraph.md)
- [agents/s15_complete_rendered_prompts.md](./agents/s15_complete_rendered_prompts.md)
- [agents/s_full.md](./agents/s_full.md)

---

## 最后一句话

如果你只把 Claude Code 学成“一个会用工具的产品”，你学到的是使用方法。  
如果你把它学成“一个围绕模型构建的 agent harness”，你学到的是架构方法。

前者让你会用。  
后者让你会造。  

这个仓库的目标是后者。  
