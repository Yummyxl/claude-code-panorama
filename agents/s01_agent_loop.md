# s01 - Agent Loop

## 这一章为什么最重要

Claude Code 再复杂，也复杂不过这一章。

因为后面所有章节，无论讲的是：

- prompt
- tools
- compaction
- memory
- team
- worktree

它们最后都必须回到同一个地方：

**主循环到底怎么跑。**

如果你不先把这件事吃透，后面所有高级机制都会看起来像：

- 一堆零散功能
- 一堆奇怪 prompt
- 一堆工具名字

而不是一套有内在逻辑的系统。

---

## 这一章最好怎么读

最好的读法，不是盯着 `while`。  
而是一直抓住一个具体例子：

```text
用户说：修复登录失败测试，并告诉我根因。
```

然后不断追问：

- 这句话什么时候进模型
- 模型什么时候不再输出文本，而是发出 `tool_use`
- 工具结果怎样重新回到下一轮

只要你一直抓着这条线，agent loop 就不会被你读成抽象伪代码。

---

## Claude Code 的核心循环到底是什么

把所有外层复杂性先拿掉，Claude Code 的循环本质上只有四步：

1. 把当前状态发给模型
2. 看模型返回了什么
3. 如果模型要求调工具，就执行工具
4. 把工具结果重新交给模型

用一句话说就是：

**模型决定动作，运行时执行动作，再把动作结果还给模型。**

这就是 agent loop。

---

## 为什么它不是“普通聊天”

普通聊天系统通常长这样：

```text
user text -> model -> assistant text
```

Claude Code 不是。

Claude Code 更接近：

```text
messages + tools + system prompt + state
    -> model
    -> structured content blocks
    -> tool_use?
    -> real-world action
    -> tool_result
    -> messages again
```

区别就在这里：

Claude Code 的响应不是一句“最终文本”，而是一组结构化内容块。  
而这组内容块里，可能既有文本，也有 `tool_use`，也有别的运行时相关块。

所以 Claude Code 从第一天起就不是“文本进，文本出”的系统。

---

## 这个循环的真正难点不在 while，而在边界

很多人第一次看到 agent loop，会以为重点是那句 `while True`。

其实不是。

真正的难点在三个边界上：

### 边界一：模型和运行时的边界

模型只能决定“下一步想做什么”。  
运行时负责决定“这件事在本机上如何真的被执行”。

### 边界二：文本和动作的边界

不是所有输出都是给用户看的。  
有些输出是运行时控制信息，例如 `tool_use`。

### 边界三：当前轮和下一轮的边界

一轮执行结束之后，最重要的不是打印了什么，而是：

**下一轮模型到底能看到什么。**

Claude Code 的全部工程，都是在稳住这三个边界。

---

## Claude Code 如何跑完一轮

我们顺着真实执行过程走一遍。

### 第一步：准备 request

在模型被调用之前，Claude Code 要先准备一大堆东西：

- 当前消息历史
- system prompt
- 当前可用 tools
- 用户环境上下文
- 系统上下文
- 可能还有 memory 等补充背景

这一步说明：  
Claude Code 发给模型的，从来不是“用户最后一句话”这么简单。

### 第二步：接收流式返回

Claude Code 收到的也不是一句纯文本。  
从实现看，它会处理多种 delta / block，例如：

- `text_delta`
- `input_json_delta`
- `thinking_delta`
- `compaction_delta`

锚点：`实现锚点`

这说明 Claude Code 的基本单位不是字符串，而是 block。

### 第三步：发现 `tool_use`

如果模型在返回里发出了 `tool_use`，Claude Code 不会把这轮当作结束，而是切换到工具执行路径。

这一步非常关键，因为它意味着：

“模型回复”不是终点。  
“模型发出动作指令”才是很多轮里的真正中间态。

### 第四步：执行工具

运行时拿到 `tool_use` 后，会：

- 找到这个工具对应的 handler
- 校验输入
- 判断权限
- 执行动作

这时候 Claude Code 才真正接触真实世界：

- 读文件
- 改文件
- 跑命令
- 发起搜索
- 调 MCP

### 第五步：把结果包装成 `tool_result`

工具跑完之后，Claude Code 不会只把结果显示给用户。  
更关键的是，它会把这些结果重新包装成结构化消息，塞回 `messages`。

这一步是整个系统最关键的闭环。

因为模型下一轮之所以还能继续工作，靠的不是“它记住了自己刚才干了什么”，而是运行时把刚才干完的结果重新喂给了它。

### 第六步：进入下一轮

到这里，主循环就闭合了。

只要模型还没停，Claude Code 就会继续下一轮：

- 用更新后的消息历史
- 用同样或更新后的 prompt
- 用当前状态下的 tools

再跑一次。

---

## 最小伪代码

把上面的过程压缩成最小可实现版本，可以写成这样：

```python
def agent_loop(messages, system_prompt, tools):
    while True:
        response = client.messages.create(
            system=system_prompt,
            messages=messages,
            tools=tools,
        )

        messages.append({
            "role": "assistant",
            "content": response.content,
        })

        tool_calls = [
            block for block in response.content
            if block.type == "tool_use"
        ]

        if not tool_calls:
            return messages

        results = []
        for call in tool_calls:
            output = run_tool(call.name, call.input)
            results.append({
                "type": "tool_result",
                "tool_use_id": call.id,
                "content": output,
            })

        messages.append({
            "role": "user",
            "content": results,
        })
```

这段伪代码已经足够解释 Claude Code 为什么能成为 agent harness。  
后面的所有复杂机制，本质上都是在增强这段循环。

---

## 实现锚点

### 主 loop 承载体

主 loop 相关锚点在：

- `实现锚点`

从实现可以确认，这里至少涉及：

- model 请求
- assistant 内容追加
- `tool_result` 注入
- `max_iterations`
- `compactionControl`
- `pushMessages`

### 流式 block 处理

- `实现锚点`

本地可见的 block/delta 类型说明 Claude Code 不是只看文本。

### compaction block 必须保留完整 content

运行时实现 明确强调：

- 不能只拿 response 里的纯文本
- 必须保留完整 `response.content`

锚点：

- `实现锚点`
- `实现锚点`

这说明在 Claude Code 里，“历史”本身也是结构化状态，而不只是文字记录。

---

## 这一章真正想让你学会什么

如果你读完这一章，只学会一句话，我希望是这句：

**Claude Code 的本质，不是模型会不会调工具，而是工具结果能不能重新进入模型的下一轮推理。**

很多人学 agent，只看到“调用工具”。  
真正的工程关键是“回流闭环”。

一旦这个闭环建立起来，后面的 prompt、memory、task、team、worktree 才有地方挂。  
没有这个闭环，后面的一切都只是产品表面。
