# s10 - Team Protocols

## 这一章要解决什么问题

一旦系统里有多个 agent，协作就不能再停留在：

- “你们自己商量”
- “你们互相看看彼此输出”

因为那样的协作根本不稳定。

Claude Code 走的是另一条路：

**把多 agent 协作做成协议。**

这就是这一章要讲的东西。

---

## 这一章最好怎么读

不要把这一章读成“消息类型总表”。  
更好的读法是始终抓住一个问题：

```text
如果一个 teammate 不能只靠自然语言协作，那它还需要什么正式机制？
```

带着这个问题往下看，`permission_request`、`shutdown_request`、`idle_notification` 就不会只是名词，而会变成一套必要的 runtime 协议。

---

## 先把最核心的判断说死

**多 agent 一旦进入真实工作，就必须区分三种东西：**

- 给用户看的自然语言
- 给队友看的控制消息
- 会触发状态变化的协议消息

如果这三种东西混在一起，你就会得到一个看起来“很智能”、但实际上完全不可控的 swarm。

Claude Code 这套 team 设计最值得学的地方，不是它有 mailbox，  
而是它很早就承认了一个工程现实：

**队友之间不能只靠自然语言互相理解，必须靠协议。**

---

## 为什么 team 不能靠普通文本协作

这是整个 team 设计的出发点。

如果多个 teammate 只是互相看普通 assistant 文本，会立刻出问题：

- 哪些消息是给用户看的，哪些消息是给队友看的
- 哪些消息需要触发状态变化，哪些只是普通解释
- 哪些消息需要审批，哪些只是通知

一旦这些边界不清，多 agent 协作就会变成一锅噪音。

所以 Claude Code 的做法很明确：

- 面向用户的文本输出是一条通道
- 队友之间的消息协议是另一条通道

本地 team 提示甚至直接说了：

```text
Just writing a response in text is not visible to others on your team - you MUST use the SendMessage tool.
```

锚点：[`cli.js:2790`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js#L2790)

这几乎就是整套 protocol 的总纲。

---

## 一条最小协作事故，你就会明白为什么要协议

先假设没有协议，只有自由文本。

现在有三个 agent：

- 主 agent
- 测试 agent
- 修复 agent

测试 agent 发现：

- 测试失败
- 怀疑是 token refresh 分支有问题

如果它只是发一段普通文本：

```text
I think the auth flow is broken around token refresh.
```

系统会立刻遇到这些歧义：

- 这是不是正式结论，还是中间猜测
- 这是发给用户的，还是发给主 agent 的
- 主 agent 现在该不该暂停当前任务
- 修复 agent 能不能直接接手
- 这条消息是不是需要写入团队状态

如果你再加上权限请求、shutdown、后台任务完成、sandbox host allow 这些场景，歧义会爆炸。

所以 Claude Code 不是让 teammate 自己“解释清楚”，  
而是直接要求：

- 这种消息是什么类型
- 应该带什么字段
- 收到后触发什么状态变化

这就是 protocol 的意义。

---

## Claude Code 的 team protocol 至少包含什么

从本地代码能直接确认，它不是一条消息类型，而是一组 typed messages。

至少包括：

- `idle_notification`
- `permission_request`
- `permission_response`
- `sandbox_permission_request`
- `sandbox_permission_response`
- `shutdown_request`
- `shutdown_approved`
- `shutdown_rejected`

锚点：

- [`cli.js:2790`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js#L2790)
- [`cli.js:7979`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js#L7979)

这已经说明团队协作不是自然语言习惯，而是正式协议面。

---

## 把这些消息按问题类型重新分组

单纯背消息名没有学习价值。  
更重要的是看它们各自解决哪类协作问题。

### 第一组：状态类消息

- `idle_notification`

它要表达的不是“我想说句话”，而是：

- 我空闲了
- 我刚完成一个任务
- 我现在可以继续接活

这类消息的价值在于让 team lead 或其他 agent 知道：

- 现在有没有空闲 worker
- 要不要再派任务
- 某个任务是不是已经进入“可衔接”阶段

### 第二组：权限协商类消息

- `permission_request`
- `permission_response`
- `sandbox_permission_request`
- `sandbox_permission_response`

这组消息把“某个 teammate 自己没有权限，但又不能直接卡死”的问题协议化了。

它们解决的是：

- worker 如何向 leader 正式提出审批
- leader 的批准 / 拒绝如何回流
- sandbox host 级权限如何异步确认

### 第三组：生命周期类消息

- `shutdown_request`
- `shutdown_approved`
- `shutdown_rejected`

这组消息解决的是：

- 谁可以宣布自己想停
- 谁有权批准这个停机
- 拒绝停机时如何把原因带回去

也就是说，它们在管理 team 的退出顺序，而不是单次任务本身。

---

## 这套协议到底在解决什么

如果把上面再抽象一层，你可以把 team protocol 看成在解决四类基础问题。

你可以把它看成在解决四类问题。

### 1. 状态同步

谁空闲了，谁忙着，谁做完了，谁该接手。

### 2. 权限协商

某个 teammate 遇到需要审批的动作怎么办。  
是自己停住，还是发请求给 team lead。

### 3. 生命周期管理

什么时候该停。  
谁批准停。  
停之前怎样收尾。

### 4. 组织边界

哪些消息是用户看的，哪些消息是队友之间传递的内部控制信息。

一旦把这四类问题都做成显式协议，team 才能稳定工作。

---

## 协议消息真正长什么样

本地代码并不是只给了“消息名字”，还直接暴露了这些 helper：

- `uF1(...)`
  把权限请求打包成 `permission_request`
- `mF1(...)`
  把权限响应打包成 `permission_response`
- `BF1(...)`
  打包 `sandbox_permission_request`
- `pF1(...)`
  打包 `sandbox_permission_response`
- `Lk6(...)`
  打包 `shutdown_request`
- `gF1(...)`
  打包 `shutdown_approved`
- `FF1(...)`
  打包 `shutdown_rejected`

锚点：[`cli.js:2797`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js#L2797)

这点非常关键。

因为这说明 team protocol 不是 prompt 里的一套“行为建议”，  
而是 bundle 内部有专门构造器和解析器的正式消息结构。

---

## 一条权限请求是怎么跑完整的

这是最值得学的一条协作链路。

你可以把它压成下面这条因果链：

```text
worker 想调用某个高风险工具
  -> 本地权限检查发现不能直接放行
  -> 组装 permission_request
  -> 通过 mailbox 发给 leader
  -> leader 处理并返回 permission_response
  -> worker 恢复执行 / 或终止当前动作
```

这条链说明了一个非常成熟的设计思路：

**权限不是 UI 提示，而是 team runtime 里的正式协商协议。**

如果没有协议，所谓“team lead 审批”就只能靠自然语言猜测，根本没法稳定接回 worker 的执行流。

---

## shutdown protocol 为什么也必须存在

很多多 agent 系统会漏掉这一层。

它们会做：

- 任务分发
- 消息互发
- maybe mailbox

但它们往往不会认真设计：

- 一个 worker 什么时候可以停
- 谁批准它停
- 停之前有没有最后状态需要同步

Claude Code 明显更谨慎。

从本地消息集合就能看出，shutdown 不是一个单向通知，而是 request / approved / rejected 三段式。

这说明它把“退出”也视为一种必须协商的状态迁移，而不是随便结束线程。

这点很工程。

因为多 agent 系统里最容易出现的隐性 bug 之一，就是：

- 任务还没真正收尾
- worker 以为自己可以停
- leader 以为对方还在工作

shutdown protocol 就是用来消灭这类歧义的。

---

## 本地代码说明 team 是“运行时协议”，不是“prompt 魔法”

从本地 bundle 可见：

- 有 team prompt
- 有 in-process teammate loop
- 有 inbox / mailbox poller
- 有 hook events

锚点：

- [`cli.js:2790`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js#L2790)
- [`cli.js:2800`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js#L2800)
- [`cli.js:7979`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js#L7979)
- [`cli.js:14796`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js#L14796)

这说明 Claude Code 对 team 的理解是：

不是“让多个模型自由聊天”。  
而是“给多个 agent 一条可控的协作总线”。

---

## 最小运行图

把 team protocol 压成最小运行图，就是这样：

```text
teammate
  -> mailbox.poll()
  -> parse typed message
  -> update local state
  -> maybe emit protocol response
  -> continue work / stop / request approval / announce idle
```

如果再展开一点：

```python
def teammate_loop():
    while True:
        for msg in mailbox.poll():
            if msg.type == "permission_request":
                handle_permission_request(msg)
            elif msg.type == "permission_response":
                handle_permission_response(msg)
            elif msg.type == "shutdown_request":
                handle_shutdown_request(msg)
            elif msg.type == "idle_notification":
                update_team_state(msg)

        maybe_pick_task()
        maybe_request_permission()
        maybe_send_idle_notification()
```

这个伪代码的重点不是精确函数名，  
而是让你看到：

- 协议消息是持续轮询处理的
- 它们会改 state
- 改完 state 之后，才回到下一轮行为选择

这就是“运行时协议”的真正含义。

---

## 最小伪代码

```python
def teammate_loop():
    while True:
        for msg in mailbox.poll():
            handle_protocol_message(msg)

        maybe_pick_task()
        maybe_request_permission()
        maybe_send_status()
```

这个伪代码很短，但已经表达出 team 的本质：

它不是一个普通工具调用，  
而是一个带 mailbox、带消息类型、带状态转移的持续运行系统。

---

## 这一章真正想让你学会什么

一句话总结：

**Claude Code 的 team 之所以能工作，不是因为 prompt 写得聪明，而是因为它把协作做成了协议。**

如果你再往前多记一句，那就是：

**协议不是为了“形式化”而形式化，而是为了把多 agent 协作里的歧义砍掉。**

这是多 agent 系统里最值得偷学的一类设计。  
因为真正可靠的协作，从来都不能只靠自由文本。
