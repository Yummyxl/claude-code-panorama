# s03 - System Prompt

## 为什么这一章很关键

很多人学 Claude Code，最容易犯的错误之一，就是把 system prompt 理解成：

- 一段人格设定
- 一段语气要求
- 一段“你是谁”的开场白

这种理解太浅了。

从本地 `cli.js` 看，Claude Code 的 system prompt 更准确的定义是：

**运行时把当前工作状态翻译给模型的一段结构化文本。**

这句话可以拆成两层意思：

### 第一层：它当然还是 prompt

最后喂给模型的，确实是一段文本。

### 第二层：但这段文本不是“文案”

它承载的是运行时状态。  
也就是说，system prompt 在 Claude Code 里不是装饰，不是品牌语气，也不是写作风格模板。  
它是控制面。

---

## 如果没有这套 prompt 机制，会发生什么

先不要急着看函数名。

你先想一个最朴素的 coding agent。如果它没有精心设计的 system prompt，它很快就会开始猜：

- 猜当前在哪个目录
- 猜有没有 git
- 猜什么工具应该优先用
- 猜当前是否处在 brief 模式
- 猜当前有没有记忆需要遵守
- 猜是不是连着 MCP server

而 Claude Code 恰恰在做相反的事：

**它尽量不让模型猜。**

它会把很多“如果不说清楚，模型就只能猜”的状态，主动写进 system prompt。

这就是为什么这一章不能只讲“提示词技巧”。  
Claude Code 的 prompt 机制，本质上是在解决运行时状态同步问题。

---

## 先把最重要的结论钉住

Claude Code 的 system prompt 有三个核心特征：

### 1. 它不是固定文案，而是装配结果

不是把一整段字符串常量永远塞给模型。  
而是根据当前运行时状态，把不同 section 拼起来。

### 2. 它不只管语气，还管行为

它不仅告诉模型“怎么说”，还告诉模型：

- 该怎么用工具
- 现在环境长什么样
- 有哪些长期背景需要遵守
- 当前有哪些运行模式在生效

### 3. 它是每轮重建的

Claude Code 的 prompt 不是“程序启动时写一次，以后一直沿用”。  
每轮都要基于当前状态重新拼。

这意味着：

- prompt 是 runtime state 的镜像
- prompt 是当前系统状态的一次序列化
- prompt 是这一轮 agent 行为的控制边界

---

## Claude Code 为什么要把 prompt 拆成 section

如果系统很简单，一段大 prompt 当然也能跑。

但 Claude Code 面临的是不断变化的状态：

- 有没有 memory
- 当前是不是在 worktree 里
- MCP 有没有连上
- output style 是什么
- scratchpad 是否存在
- brief 模式有没有开启
- 有没有附加的自定义 system prompt

如果这些东西都堆在一大段不可拆分的文本里，工程上会立刻出现几个问题：

### 问题 1：很难维护

你很难知道某一条规则属于哪一层。

### 问题 2：很难条件注入

某一节只该在特定条件下出现，如果 prompt 是整块文本，就只能做非常脆弱的字符串拼接。

### 问题 3：很难缓存

有些 section 是稳定的，有些是动态的。  
如果不拆 section，就很难对“稳定部分”做缓存。

### 问题 4：很难解释系统现在为什么这么做

一旦 prompt 出问题，你甚至不知道是哪一段动态内容把模型行为带偏了。

所以 Claude Code 选择的是：

**先把 prompt 拆成 section，再按当前状态组装。**

锚点：`cli.js:1480`

---

## 可以把它想成“给模型发一份任务前简报”

如果你不喜欢“prompt section”这个词，可以先用一个更直观的类比。

每一轮 Claude Code 真正做的事情，有点像在给模型发一份任务前简报：

- 你现在是谁
- 你应该怎么工作
- 你手上有什么工具
- 你现在处在什么环境
- 你有哪些长期记忆要遵守
- 当前有没有特殊模式

区别只在于，这份“简报”最后不是给人看，而是给模型看。

所以 system prompt 在 Claude Code 里，更像：

**运行时发给模型的一份操作前 briefing。**

---

## 本地代码里能直接确认哪些 section

从 `PD(...)` 的装配过程里，可以直接看到这些 section 名称：

- `memory`
- `ant_model_override`
- `env_info_simple`
- `language`
- `output_style`
- `mcp_instructions`
- `scratchpad`
- `frc`
- `summarize_tool_results`
- `brief`

这组名字本身就很说明问题。

如果 prompt 只是人格文案，你不会看到：

- `memory`
- `env_info_simple`
- `mcp_instructions`
- `scratchpad`
- `summarize_tool_results`

这些 section 说明，Claude Code 的 prompt 系统正在承载：

- 当前环境信息
- 长时背景
- 扩展运行时信息
- 输出模式
- 工具结果总结策略

换句话说，它不是“角色设定卡”。  
它是运行时状态装配线。

---

## 一轮 prompt 到底是怎么装出来的

这一节是整章的核心。  
我们不用先陷进函数细节，先按运行时真正发生的顺序讲。

### 第 1 层：固定主干

无论当前是不是在 worktree、有没有 memory、有没有 MCP，Claude Code 都要先给模型一套稳定的基础约束。

这层通常包括：

- 身份定义
- 工具使用原则
- 输出风格
- 输出长度/效率约束
- 环境信息

这层的目标是先把模型固定成“会用工具做事的 agent”，而不是普通聊天模型。

### 第 2 层：条件 section

然后再看当前运行时状态，决定要不要往上加：

- memory section
- MCP instructions
- scratchpad
- brief
- output style override

这层的目标不是定义 Claude Code 是谁，而是定义：

**Claude Code 这一轮正处在什么上下文里。**

### 第 3 层：外部附加项

最后，运行时还可能再叠加：

- `customSystemPrompt`
- `appendSystemPrompt`
- 某些特殊路径下额外 memory prompt

所以最终 system prompt，不是某个写死常量，而是多层装配结果。

---

## 直接用自然语言把主干 section 讲透

这一节不做类型表，而是把 system prompt 真的在干什么讲明白。

### 第一段：角色定义

本地代码里有非常关键的一句：

```text
You are an agent for Claude Code, Anthropic's official CLI for Claude. Given the user's message, you should use the tools available to complete the task. Do what has been asked; nothing more, nothing less.
```

这段话的作用不是“品牌介绍”，而是把模型的基本身份钉死：

- 你不是普通聊天助手
- 你是 Claude Code 的 agent
- 你的任务是完成用户要求
- 不能擅自加戏

这就是 agent 行为的起点。

### 第二段：工具使用规则

Claude Code 不只是把 tool schema 暴露给模型。  
它还会在 prompt 里给出工具使用规则。

比如从本地 prompt 文本可以直接看到，系统会强调：

- 有专门工具时不要默认退回 Bash
- 文件读取要优先走文件工具
- 搜索要优先走结构化搜索工具
- 没有依赖关系的工具调用要尽量并行

这一步非常关键，因为它在做的不是“工具介绍”，而是“工具政策”。

你可以把 schema 理解成“这把扳手长什么样”，  
把 prompt 规则理解成“什么时候该先用扳手，什么时候不该直接上万能电钻”。

### 第三段：语气与输出效率

Claude Code 把“怎么说”和“说多少”拆开管理。

从本地代码能看到：

- `a4z()` 负责 tone and style
- `o4z()` 负责 output efficiency

这意味着系统在 prompt 层明确区分：

- 表达风格
- 响应密度

这种拆分的好处是：

- 你可以保持同一种 agent 行为
- 同时针对不同场景压缩或放宽输出

### 第四段：环境信息

Claude Code 会把当前环境写进 prompt。

这通常包括：

- 当前目录
- shell
- OS
- git / worktree 状态
- 当前模型

这一步的价值非常直接：

**把环境事实显式告诉模型，而不是让模型自己猜。**

### 第五段：动态背景

最后接上的，是那些每轮可能不同的 section：

- memory
- MCP instructions
- scratchpad
- brief

这些内容的共同特点是：

- 不是每轮都有
- 一旦有，就会显著改变 agent 行为

所以它们必须由运行时动态决定，而不是被写进一份永远不变的 prompt 模板。

---

## 从 `submitMessage(...)` 反看 prompt 装配

如果你把 `s14` 里的调用链和这一章连起来看，会更清楚。

`LtK.submitMessage(...)` 不是一上来就把一段固定 prompt 交给模型。  
它会先做一轮装配准备：

```text
[g, F, Q] = await Promise.all([
  customSystemPrompt ? [] : PD(...),
  WA(),
  customSystemPrompt ? {} : uO()
])
```

这里至少可以严格确认三件事：

- `PD(...)` 负责生成默认 system prompt sections
- `WA()` 负责准备 user context
- `uO()` 负责准备 system context

之后再做总装：

```text
e = O5([
  ...customSystemPrompt ? [customSystemPrompt] : g,
  ...K6 ? [K6] : [],
  ...appendSystemPrompt ? [appendSystemPrompt] : []
])
```

锚点：`cli.js:16476`

这几行直接说明：

- 默认 prompt 来自 `PD(...)`
- 自定义 prompt 可以硬覆盖默认主干
- 还可以在最后额外 append
- 某些路径下还有额外 memory prompt 注入

也就是说，真正的 system prompt 是总装结果 `e`，不是 `PD(...)` 单独一个函数的返回值。

---

## 这套设计为什么比“一大段 prompt”更成熟

如果你已经理解前面几节，现在就能看出 Claude Code 这套做法的工程优势。

### 优势 1：行为约束和状态注入可以分层

固定行为约束写进主干 section。  
动态状态只在需要时注入。

### 优势 2：每轮都能按状态变化重建

这让 prompt 真正跟着 runtime 走，而不是跟着初始模板走。

### 优势 3：缓存有了边界

本地 bundle 顶部还能看到 `setSystemPromptSectionCacheEntry`。

锚点：`cli.js:8`

这说明 Claude Code 不只是“会拼 prompt”，还把 section 当成了可缓存对象。

### 优势 4：可解释性更强

当模型行为变化时，你可以追：

- 是哪一段 section 加进来了
- 是哪一个模式开了
- 是哪一个扩展状态改变了 prompt

这比调一大段黑盒 prompt 容易得多。

---

## 最小伪代码

把整套思路压缩成最小结构，大概是这样：

```python
def build_system_prompt(state):
    sections = []

    sections.append(base_role_prompt())
    sections.append(tool_usage_policy())
    sections.append(tone_and_style())
    sections.append(output_efficiency())
    sections.append(environment_section(state))

    if state.memory:
        sections.append(memory_section(state))
    if state.mcp_clients:
        sections.append(mcp_section(state))
    if state.scratchpad:
        sections.append(scratchpad_section(state))
    if state.brief_mode:
        sections.append(brief_section(state))

    if state.custom_system_prompt:
        sections = [state.custom_system_prompt]

    if state.extra_memory_prompt:
        sections.append(state.extra_memory_prompt)
    if state.append_system_prompt:
        sections.append(state.append_system_prompt)

    return join_non_null_sections(sections)
```

最重要的不是这些函数名，而是这个结构本身：

- 主干先装
- 动态 section 条件注入
- 特殊覆盖最后生效

这就是 Claude Code system prompt 的工程思路。

---

## 本地代码锚点

### prompt 总装入口

- `cli.js:1480`

### 单轮提交时的 prompt 组装与覆盖

- `cli.js:16476`

### tone and style

同一区域可以看到 `a4z()` 生成的风格规则，例如：

- 只有用户要求时才用 emoji
- 回答要简洁
- 文件引用用 `file_path:line_number`

### output efficiency

同一区域可以看到 `o4z()` 强调：

- 直奔主题
- 优先最简单做法
- 少铺垫
- 少废话

### environment section

`WVq(...)` 在同一区域负责把环境信息写入 prompt。

### section cache

- `cli.js:8`

可以看到 `setSystemPromptSectionCacheEntry`。

---

## 这一章和后面两章怎么配合

这一章负责建立心智模型。

如果你想继续往下钻：

- 看 [s13_prompt_catalog.md](./s13_prompt_catalog.md)
  这里会把 section、顺序、条件、原文系统展开
- 看 [s15_complete_rendered_prompts.md](./s15_complete_rendered_prompts.md)
  这里会告诉你不同运行条件下，最后真的渲染出来的 prompt 长什么样

也就是说：

- 这一章讲“为什么是这样”
- `s13` 讲“它具体怎么装”
- `s15` 讲“最后模型到底看到了什么”

---

## 这一章真正想让你学会什么

这一章最重要的结论只有一句：

**Claude Code 的 system prompt 不是文案，而是运行时状态的文本装配。**

它当然表现为文本。  
但在架构上，它承担的是：

- 行为约束
- 状态同步
- 环境注入
- 动态模式切换

一旦你用这个角度再去看后面的 memory、MCP、brief、output style，很多东西都会自然很多。
