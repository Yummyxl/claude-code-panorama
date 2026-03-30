# s08 - MCP Plugins Remote

## 这一章要解决什么问题

如果 Claude Code 想长期有生命力，它就不能把所有能力都写死在自己的 bundle 里。

它必须回答四个更大的问题：

- 外部工具怎么接进来
- 新知识怎么按需加载
- 输出和命令怎么继续扩展
- 外部系统怎么反过来驱动我

Claude Code 运行时里，对应的就是这四组东西：

- MCP
- skills
- plugins
- remote / channel / bridge

这一章讲的不是“又多了几个 feature”，而是：

**Claude Code 怎样从一个 agent，长成一个可扩展 runtime。**

---

## 先把扩展层的角色分清

很多人会把这些词混在一起，觉得都是“插件机制”。

其实不是。

### MCP

把外部 server 的资源和工具接进 Claude Code。

它解决的是：

- 外部能力如何进入当前工具面

### skills

把额外工作知识、工作模板、工作习惯按条件注入。

它解决的是：

- 模型什么时候该拿到什么额外工作经验

### plugins

把运行时的命令、内容命令、输出样式继续模块化。

它解决的是：

- 当前能力如何被继续扩展和定制

### remote / channel / bridge

让 Claude Code 不只是自己向外调用，还能被外部系统驱动。

它解决的是：

- 外部世界如何与 Claude Code 建立双向连接

一旦把这四层分开，你就会发现 Claude Code 的扩展能力是有层次的，而不是一锅“支持插件”。

---

## 为什么这章对学习特别重要

因为很多 agent demo 的生命周期都停在这一步之前。

它们通常是：

- 一份固定 prompt
- 一组固定工具
- 一个固定主循环

这当然能工作，但很快就会变成封闭系统。

而 Claude Code 实现展示的是另一条路线：

**把 runtime 做成有正式扩展边界的机器。**

这也是 Claude Code 和一次性 demo 之间非常重要的差别。

---

## MCP 在 Claude Code 里到底代表什么

MCP 的意义，不是“能多调一个工具”，而是：

**Claude Code 允许外部 server 把自己的资源和能力正式挂进来。**

这件事的架构含义很大。

因为它意味着 Claude Code 并不坚持“所有能力必须是内建工具”。  
相反，它承认：

- 外部资源是一级对象
- 外部工具也是一级对象
- 外部 server 的存在会影响 prompt 和工具面

这就是为什么工具定义 里会同时出现：

- `ListMcpResourcesInput`
- `ReadMcpResourceInput`
- `McpInput`

它不是把 MCP 看成一个单一接口，而是拆成：

- 发现资源
- 读取资源
- 调用外部工具

---

## 为什么 skills 很关键

很多系统一说“技能”，就喜欢把更多说明永远塞进主 prompt。  
Claude Code 实现显示，它没有这么粗暴。

它会：

- 动态发现 skill
- 去重
- 暂存 conditional skill
- 在文件或路径匹配后再激活

锚点：

- `实现锚点`
- `实现锚点`

这很重要，因为它说明 Claude Code 在处理的不是“多塞一点知识”，而是：

**什么时候把哪份知识给模型。**

这正是成熟 harness 和粗暴 prompt 堆料之间的差别。

换句话说，skills 的真正价值不是“内容更多”，而是“内容按条件进入运行时”。

---

## plugin 为什么不只是“加命令”

从实现看，plugin 不只是在加工具，还会影响：

- command 加载
- inline content command
- output style

锚点：

- `实现锚点`
- `实现锚点`

这说明 plugin 能进入的不只是动作面，还包括表达面。

这点非常值得注意，因为它说明 Claude Code 的插件系统不是：

```text
加一个新按钮
```

而更像：

```text
进入命令系统
进入内容生成系统
进入输出控制系统
```

也就是说，plugin 是能摸到运行时控制面的。

---

## remote / channel / bridge 说明了什么

一旦一个系统支持：

- `--remote`
- `--remote-control`
- `--channels`

它就已经不是单纯的“本地手工使用 CLI”了。

它已经开始变成：

- 可以接外部控制
- 可以接外部通知
- 可以接外部协作入口

锚点：`实现锚点`

这意味着 Claude Code 不只是自己向外接能力，  
还允许外部系统反过来把它当成一个 runtime endpoint。

这件事的学习价值在于：

Claude Code 不是只把自己设计成一把刀，  
而是把自己设计成一个可嵌入、可编排、可被驱动的 agent runtime。

---

## 最小伪代码

```python
def extension_runtime(state):
    tools = load_builtin_tools()
    tools += load_mcp_tools(state.connected_servers)

    prompt_sections = []
    prompt_sections += load_matching_skills(state.cwd, state.files)
    prompt_sections += load_plugin_output_style(state.output_style_name)

    commands = load_plugin_commands()
    channels = connect_remote_channels_if_enabled(state.remote_mode)

    return RuntimeExtensions(
        tools=tools,
        prompt_sections=prompt_sections,
        commands=commands,
        channels=channels,
    )
```

这段伪代码对应的是本章的核心结构：

- MCP 扩工具面
- skills 扩工作知识
- plugins 扩命令和输出样式
- remote/channel 扩外部驱动入口

---

## 把四层关系压成一张图

```text
MCP
  -> 接外部工具和资源

skills
  -> 在正确时机补工作知识

plugins
  -> 扩展本地命令、内容命令、输出风格

remote / channel / bridge
  -> 让外部系统反向控制和通知 Claude Code
```

如果你把这张图记住，再看整台 Claude Code 机器时，就不会把扩展层误解成某个孤立功能点。

---

## 这一章和前后章节怎么接

前面几章主要讲的是 Claude Code 内部怎样工作：

- prompt
- tools
- task
- team
- isolation

而这一章开始讲的是：

**这台机器怎样和外部世界建立正式接口。**

后面你再去看：

- [s12_tools_catalog.md](./s12_tools_catalog.md)
- [s13_prompt_catalog.md](./s13_prompt_catalog.md)

会更容易理解为什么：

- MCP 既影响工具面，也影响 prompt
- skills 既不是普通工具，也不是普通 memory
- plugin 能碰到 output style

---

## 这一章真正想让你学会什么

一句话总结：

**Claude Code 不是封闭产品，而是可扩展运行时。**

它的强点不只是“现在有什么能力”，更在于：

- 新能力怎么被接进来
- 已有能力怎么按需激活
- 外部系统怎么反向接入它

这就是它和很多一次性 agent demo 最大的差别之一。
