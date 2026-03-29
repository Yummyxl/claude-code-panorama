# s14 - ToolRunner Callgraph

## 这一章要解决什么问题

前面的章节已经把 Claude Code 拆成了：

- prompt
- tools
- memory
- team
- worktree

但如果你还看不清“这些东西在一轮执行里到底怎么串起来”，那你依然没有真正抓住运行时。

这一章只做一件事：

**把 Claude Code 一轮消息处理的真实调用链，从入口一路追到最终结果。**

目标不是罗列所有内部函数，而是回答这几个工程问题：

1. 一条用户输入从哪里进入
2. system prompt 在哪一步组装
3. 哪一步决定“这轮要不要真的去问模型”
4. 哪一步进入真正的 agent loop
5. `tool_use` / `tool_result` / `stream_event` / `compact_boundary` 在哪里回流
6. 最终 `result` 是怎样合成的

主要证据来自：

- `cli.js:16476`
- `cli.js:7974`
- `cli.js:7806`
- `cli.js:1495`
- `cli.js:297`

---

## 这一章最好怎么读

不要把这一章当成“函数索引”来读。  
最好的读法是始终带着一个具体问题：

```text
用户说了一句话之后，Claude Code 到底怎样一步步把这句话变成：
- prompt
- 模型调用
- tool_use
- tool_result
- 最终结果
```

如果你一直抓住这条主线，后面出现的 `RtK`、`LtK.submitMessage`、`tU8`、`uF8`、`_b` 就不会变成无意义的名字。

---

## 先给最终图：一轮消息真正经过哪些层

先把整条链压成一张图：

```text
RtK(...)
  -> new LtK(...)
  -> LtK.submitMessage(...)
       1. 读取 app state / model / thinking mode
       2. 组装 system prompt: PD(...) + WA() + uO()
       3. 预处理本轮输入: tU8(...)
       4. 发出 init/system metadata: uF8(...)
       5. 若 shouldQuery = false: 直接返回
       6. 进入主循环: for await (event of _b(...))
            -> assistant
            -> user
            -> progress
            -> stream_event
            -> attachment
            -> system(compact_boundary / api_error)
            -> tool_use_summary
       7. 汇总 stop_reason / usage / structured_output
       8. 生成最终 result
```

你可以把它理解成三层：

- 外层包装：`RtK`
- 单轮协调器：`LtK.submitMessage`
- 真正的 agent loop：`_b(...)`

---

## 第一层：入口为什么是 `RtK(...)`

本地代码里，`RtK(...)` 很简单：

```text
async function* RtK(...) {
  let C = new LtK({...});
  try {
    yield* C.submitMessage(...)
  } finally {
    P(C.getReadFileState())
  }
}
```

锚点：`cli.js:16476`

### 它在架构里的作用

`RtK` 不是主逻辑本体。  
它更像一个会话级包装器：

- 创建 `LtK`
- 把调用参数注入实例
- 把读取缓存写回外层
- 把 `submitMessage()` 的事件流原样往外吐

所以如果你在追“真正这一轮发生了什么”，重点应该放在 `LtK.submitMessage(...)`。

---

## 先用一个具体例子把整章串起来

假设用户发来一句话：

```text
Fix the failing login test and tell me what was broken.
```

这一轮在 Claude Code 里，不是直接跳进“模型回答”。

它会依次经历：

1. `RtK(...)`
   建一个本轮执行器
2. `LtK.submitMessage(...)`
   取当前模型、权限、tools、MCP、app state
3. `PD(...)`
   按这一轮状态拼 system prompt
4. `tU8(...)`
   预处理这条用户输入，决定是否真的 query 模型
5. `uF8(...)`
   先向外层发一份 “这一轮 runtime 长什么样” 的 init 事件
6. `_b(...)`
   正式进入 agent loop，模型开始：
   - 读文件
   - 跑测试
   - 改代码
   - 回流 tool result
7. `submitMessage(...)`
   一边消费这些事件，一边维护消息历史、usage、compaction、structured output
8. 最终合成 `result`
   把本轮收口成一个稳定的终局对象

如果你一直记住这条具体路径，后面每一节就不会飘。

---

## 第二层：`LtK.submitMessage(...)` 是真正的单轮协调器

`LtK.submitMessage(...)` 一开头就把当前这一轮需要的关键运行时全拿出来：

- `cwd`
- `commands`
- `tools`
- `mcpClients`
- `thinkingConfig`
- `maxTurns`
- `maxBudgetUsd`
- `taskBudget`
- `customSystemPrompt`
- `appendSystemPrompt`
- `userSpecifiedModel`
- `fallbackModel`
- `jsonSchema`
- `getAppState / setAppState`
- `replayUserMessages`
- `includePartialMessages`
- `agents`

锚点：`cli.js:16476`

这一步很重要，因为它说明：

**Claude Code 每一轮都不是“只拿 messages 去问模型”。它会把这一轮可能影响行为的所有 runtime 状态重新抽出来。**

---

## 第三层：真正的预处理顺序

`submitMessage(...)` 的前半段，顺序其实非常清楚。

---

### 第 1 步：确定模型和 thinking mode

源码里先做：

- `I = Z()` 读取 app state
- `p = D ? oK(D) : K5()` 决定主模型
- `u = O ? O : (settings/default-derived thinking config)` 决定 thinking config

你可以把它理解成：

- 先决定“这一轮到底用哪个模型”
- 再决定“这一轮是不是带 thinking / adaptive thinking”

这一步是整个 loop 的计算基线。

---

### 第 2 步：组装 system prompt 相关输入

接着会做：

```text
[g, F, Q] = await Promise.all([
  customSystemPrompt ? [] : PD(...),
  WA(),
  customSystemPrompt ? {} : uO()
])
```

锚点：`cli.js:16476`

这里三样东西分别对应：

- `g`
  `PD(...)` 产出的 system prompt sections
- `F`
  `WA()` 产出的 user context
- `Q`
  `uO()` 产出的 system context

如果用户直接给了 `customSystemPrompt`，那么：

- 不再跑默认 `PD(...)`
- 不再跑默认 `uO()`

这说明 Claude Code 把“完全自定义 system prompt”当成一种硬覆盖模式，而不是普通 append。

---

### 第 3 步：把 system prompt 真正拼成 `e`

源码又做了两步：

```text
l = { ...F, ...xFY($, iF()?R76():void 0) }
K6 = customSystemPrompt exists && oD8() ? await ZV8() : null
e = O5([
  ...customSystemPrompt ? [customSystemPrompt] : g,
  ...K6 ? [K6] : [],
  ...appendSystemPrompt ? [appendSystemPrompt] : []
])
```

锚点：`cli.js:16476`

这几行非常关键：

- `e`
  才是这一轮最后真正喂给模型的 `systemPrompt`
- `l`
  是这一轮 user-context 层的数据对象
- `K6`
  是某种特殊路径下额外插进去的 memory prompt

这解释了一个常见误解：

Claude Code 不是“有一个固定 system prompt 字符串”。  
它是先生成很多块，再在这里总装。

---

### 第 4 步：把本轮用户输入预处理成 messages

然后进入：

```text
let { messages:r, shouldQuery:_6, allowedTools:D6, model:J6, resultText:E6 } = await tU8(...)
```

锚点：`cli.js:7974`、`cli.js:16476`

`tU8(...)` 是一个非常重要但经常被忽略的层。

它做的不是“发模型请求”，而是：

- 处理用户输入
- 处理 slash command / hook / prompt 扩展
- 决定这轮是否需要真的 query 模型
- 在某些情况下直接返回本轮结果

换句话说，`tU8(...)` 是真正 agent loop 之前的“输入编排层”。

---

## 第四层：为什么 `shouldQuery` 这么关键

`tU8(...)` 返回的 `shouldQuery` 是整个调用链里的第一个硬分叉。

源码里有一个非常明确的短路路径：

```text
if (!shouldQuery) {
  ... emit prebuilt messages and replayed messages ...
  yield { type:"result", subtype:"success", result: E6 ?? "", ...metadata }
  return
}
```

锚点：`cli.js:16476`

### 这意味着什么

不是每轮输入都会真正打到模型。

有些输入在进入主 loop 之前就被 runtime 吃掉了，例如：

- 某些本地命令扩展
- 某些无需再次 query 的变换
- 某些直接产出预构造消息的路径

所以 Claude Code 的真实执行模型是：

```text
raw input
  -> preprocess
  -> maybe query model
  -> maybe return immediately
```

而不是：

```text
raw input -> always query model
```

---

## 第五层：`uF8(...)` 为什么在这里发

在真正进入 `_b(...)` 之前，源码会先 `yield uF8(...)`：

```text
yield uF8({
  tools,
  mcpClients,
  model,
  permissionMode,
  commands,
  agents,
  skills,
  plugins,
  fastMode
})
```

锚点：`cli.js:7806`、`cli.js:16476`

`uF8(...)` 产出的是一个 `system/init` 型事件，里面会带：

- 当前工具名集合
- MCP server 状态
- 当前模型
- permission mode
- slash commands
- output style
- agents
- skills
- plugins
- fast mode

### 它的意义

Claude Code 在真正开始这一轮执行前，会先把“这轮 runtime 的外观”发给外层观察面。

这一步对产品层非常重要，因为它让：

- SDK
- transcript
- UI
- debug / telemetry

都能在模型真正开始说话前，先知道本轮的系统状态。

---

## 第六层：真正的主循环在 `_b(...)`

到这里，才正式进入：

```text
for await (let L6 of _b({
  messages: $6,
  systemPrompt: e,
  userContext: l,
  systemContext: Q,
  canUseTool: x,
  toolUseContext: t,
  fallbackModel: P,
  querySource: "sdk",
  maxTurns: w,
  taskBudget: H
}))
```

锚点：`cli.js:16476`

虽然我们这里没有把 `_b(...)` 的完整函数体整段再摘出来，但从调用点已经能严格确认：

`_b(...)` 是真正消费：

- `messages`
- `systemPrompt`
- `userContext`
- `systemContext`
- `toolUseContext`
- `canUseTool`

并持续产出事件流的主 generator。

也就是说，前面的所有东西，都是在给 `_b(...)` 备料。

---

## 第七层：`_b(...)` 吐出来的不是单一 assistant message，而是一整条事件流

`submitMessage(...)` 在消费 `_b(...)` 的时候，明确处理了这些类型：

- `assistant`
- `progress`
- `user`
- `stream_event`
- `attachment`
- `system`
- `tool_use_summary`
- `tombstone`

锚点：`cli.js:16476`

这非常关键，因为它说明：

**Claude Code 的主 loop 不是“等模型返回一个最终回答”。它是在消费一个多种事件混合的执行流。**

下面逐个看这些事件的意义。

---

### 1. `assistant`

这是最直观的类型：

- 模型产出的 assistant message
- 里面可能带 `stop_reason`

`submitMessage(...)` 会：

- 把它推入 `mutableMessages`
- `yield* gu8(L6)` 往外发
- 记录最新 `stop_reason`

这说明 assistant message 既是 transcript 内容，也是后续 result synthesis 的输入。

---

### 2. `progress`

`progress` 也会被放进消息历史，并对外 `yield`。

这类事件的存在说明 Claude Code 的 runtime 不满足于“只有最终文本”。  
它要让外层 UI 在中间阶段也能看到工作推进。

---

### 3. `user`

这看起来奇怪，因为我们明明已经有用户输入了。  
但在这个阶段吐出的 `user` 往往不是“原始用户敲的那条”，而是 runtime 回流进消息历史的用户侧事件，例如：

- 某些 replay
- 某些 injected prompt
- 某些 queued command

这说明消息历史里的 `user` 不等于“用户键盘直输”，而是更广义的 user-role message。

---

### 4. `stream_event`

这类事件专门服务于流式 API 细节。

源码里会根据：

- `message_start`
- `message_delta`
- `message_stop`

去累计 usage，并更新 `stop_reason`。

这说明 usage 统计不是在最终结果里一次性拿到，而是随着流事件逐步累加。

---

### 5. `attachment`

`attachment` 在这里至少承担三类职责：

- structured output 回传
- max turns reached
- queued command replay

也就是说，attachment 不是“文件附件”这么简单，它还是 runtime 特殊信号承载层。

---

### 6. `system`

这里最重要的两个 subtype 是：

- `compact_boundary`
- `api_error`

#### `compact_boundary`

一旦收到它，`submitMessage(...)` 会直接裁剪掉历史前半段，只保留压缩后的边界后内容。

这说明 compaction 不是离线批处理，而是主 loop 中实时发生的历史替换动作。

#### `api_error`

则会被转成外层可见的 `api_retry` 系统事件。

---

### 7. `tool_use_summary`

Claude Code 不只保存原始 tool use / tool result，还会产出摘要事件：

- `summary`
- `preceding_tool_use_ids`

这再次说明 runtime 在主动做工具结果压缩，而不是把所有原始工具输出永久塞在历史里。

---

## 第八层：compaction 在调用链里的真实位置

很多人以为 compaction 是“后台优化”。  
从这条调用链看，不是。

它在 `submitMessage(...)` 里的处理是一级逻辑：

- 一旦收到 `compact_boundary`
- 立刻裁切 `mutableMessages`
- 同步裁切当前工作副本 `$6`
- 对外 `yield` 一个新的 compact boundary 事件

锚点：`cli.js:16476`

这意味着：

**compaction 改的不是副本，而是这轮后续继续执行所依赖的真实消息历史。**

所以它绝不是可有可无的小优化，它是长会话主循环的一部分。

---

## 第九层：最终 `result` 是怎样合成的

`submitMessage(...)` 的尾部会统一收束成一个 `result` 事件。

它至少会处理这些终局情况：

- `success`
- `error_max_turns`
- `error_max_budget_usd`
- `error_max_structured_output_retries`
- `error_during_execution`

成功路径里，最终 `result` 至少会带：

- `duration_ms`
- `duration_api_ms`
- `num_turns`
- `result`
- `stop_reason`
- `total_cost_usd`
- `usage`
- `modelUsage`
- `permission_denials`
- 可选 `structured_output`

锚点：`cli.js:16476`

### 这里最重要的设计点

Claude Code 把“这轮执行的最终答案”做成了单独的 result synthesis 阶段。

这意味着：

- 中途事件可以很多、很碎
- 但最终对外仍然有统一收口

这对 SDK / transcript / automation 都很重要，因为它们不必自己猜“这轮是不是结束了”。

---

## 第十层：把整条链再压成最小伪代码

如果把这章内容压成一份接近工程现实的伪代码，大致是这样：

```python
async def run_turn(input):
    state = get_app_state()
    model = resolve_model(state)
    thinking = resolve_thinking(state)

    system_sections, user_ctx, system_ctx = await prepare_prompt_context(
        tools, model, state, mcp_clients
    )
    system_prompt = assemble(system_sections, custom_system_prompt, append_system_prompt)

    pre = await preprocess_input(input, messages, state)
    messages.extend(pre.messages)

    emit_init_metadata(...)

    if not pre.should_query:
        return synthesize_result_from_preprocess(pre)

    async for event in main_loop(
        messages=messages,
        system_prompt=system_prompt,
        user_context=user_ctx,
        system_context=system_ctx,
        can_use_tool=permission_fn,
        tool_use_context=runtime_ctx,
    ):
        apply_event_to_history(event)
        emit_event(event)
        update_usage_and_stop_reason(event)
        if event is compact_boundary:
            rewrite_history_with_compaction_boundary()

    return synthesize_final_result()
```

---

## 这一章真正想让你学会什么

如果这一章只能记一句话，我希望是这句：

**Claude Code 的“agent loop”不是一个简单 while 循环，而是“输入预处理 + system prompt 装配 + 主事件流 + result synthesis”四段式运行时。**

理解这四段，你再看：

- prompt
- tools
- compaction
- memory
- background task
- team

它们就不会再像散件，而会自动回到同一条调用链上。
