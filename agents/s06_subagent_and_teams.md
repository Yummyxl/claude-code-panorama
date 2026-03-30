# s06 - Subagent And Teams

## 这一章要解决什么问题

当一个任务开始变大，Claude Code 不能只回答“模型会不会用工具”，而必须回答另一个更难的问题：

**工作怎么分出去，而且分出去之后怎么不失控。**

这就是这一章要处理的核心。

从实现看，Claude Code 没有把所有“多 agent”都混成一个概念。  
它至少区分了两层：

- `subagent`
- `team`

如果你不把这两层分开，后面关于 mailbox、permission relay、shutdown、worktree 的很多设计都会看起来像一团乱麻。

---

## 这一章最好怎么读

最好的读法，是始终带着一个具体判断：

```text
什么时候系统只是把工作“派出去”，什么时候它已经进入“持续组织协作”？
```

前者是 subagent。  
后者是 team。

只要你抓住这条分界线，后面出现的 mailbox、leader、shutdown、permission relay 才不会重新混成一团“多 agent 功能”。

---

## 先把结论说死

**subagent 是一次正式委派。**  
**team 是一套持续协作组织。**

这两个东西都叫“多 agent”，但它们解决的问题完全不是一个量级。

---

## 为什么这两个概念必须分开

先想一个具体场景。

用户说：

```text
Trace the failing login flow, patch the bug, run the tests, and summarize the root cause.
```

这时候主 agent 至少有三种可能做法：

### 做法 1：全都自己做

问题是上下文会越来越脏：

- 测试日志塞满历史
- 试错过程淹没主线
- 一个子问题的噪音污染整个对话

### 做法 2：派一个 worker 去查某个子问题

这就是 subagent。

主 agent 说：

- 你去搜 auth flow
- 你去整理测试失败点
- 你去查 token refresh 分支

这还是“一次委派”。

### 做法 3：系统里长期存在多个 agent，各自有身份和职责

这就是 team。

此时不再是“我临时派了一个人出去”，而是：

- 有 lead
- 有 teammate
- 有 mailbox
- 有权限转发
- 有 shutdown 协议
- 有长期协作边界

这已经不是单次委派，而是组织问题。

---

## subagent 到底是什么

subagent 的本质是：

**当前 agent 正式派生出另一个执行单元，让它在相对独立的工作上下文里处理某个子问题。**

这不是：

- 在脑子里想象另一个自己
- 换一个 prompt 继续答
- 在同一份 messages 里开个新话题

从工具定义 可以直接确认，它是正式运行时对象，因为 `AgentOutput` 里真的会返回：

- `agentId`
- `prompt`
- `status`
- `usage`
- `totalTokens`
- `totalDurationMs`

如果后台启动，还会给：

- `outputFile`
- `status: "async_launched"`

锚点：

- `工具定义锚点`
- `工具定义锚点`

所以最准确的理解是：

**subagent 是 Claude Code 里的正式 worker 机制。**

---

## team 又是什么

team 不是“我多开几个 subagent”这么简单。

team 的本质是：

**系统里存在多个长期可寻址的 agent，它们通过消息、协议和运行模式协作。**

一旦进入 team 模式，系统立刻多出一整套 subagent 不需要面对的问题：

- 谁是 lead
- 队友之间怎么通信
- 谁来做权限审批
- 谁能批准 shutdown
- 一个 agent 空闲了要不要自动接活
- teammate 用 `tmux` 还是 `in-process`

也就是说，team 处理的是组织层问题，不只是执行层问题。

---

## 用一句话把两者分开

你可以把它们先压缩成这组对照：

```text
subagent = 派一个出去做一段工作
team     = 让一组 agent 长期一起工作
```

或者更严格一点：

```text
subagent = delegation
team     = organization
```

---

## Claude Code 为什么两层都要有

因为这两个机制各自解决不同的痛点。

### subagent 解决“容量不够”

主 agent 一口气做所有子问题，会很快把消息历史搞脏。  
这时最有效的办法，是把一个问题拆出去，让另一个执行单元独立做。

所以 subagent 最直接解决的是：

- 上下文容量
- 子问题隔离
- 并行探索

### team 解决“组织不稳定”

当系统里已经不是临时派一个 worker，而是多个 agent 需要持续协作时，真正的难点不再是“能不能派出去”，而是：

- 谁和谁说话
- 谁能批准什么
- 谁在等谁
- 谁该停
- 谁接下一个任务

所以 team 最直接解决的是：

- 协作秩序
- 角色边界
- 生命周期管理

这就是为什么 Claude Code 不能只做一个 `Agent` 工具就结束。  
它还必须继续长出 team runtime。

---

## 实现能直接确认什么

### 1. subagent 是正式工具，而不是提示词技巧

`AgentInput` / `AgentOutput` 直接存在于本地工具 schema 中。

`AgentInput` 里还能看到这些重要字段：

- `description`
- `prompt`
- `subagent_type`
- `model`
- `run_in_background`
- `name`
- `team_name`
- `mode`
- `isolation`

锚点：`工具定义锚点`

这说明 subagent 不是自由发挥，而是带：

- 模型覆盖
- 权限模式
- 背景执行
- team 归属
- worktree 隔离

的正式委派动作。

### 2. team 有明确运行模式

本地 CLI 选项里能看到：

- `--agent-id`
- `--agent-name`
- `--team-name`
- `--teammate-mode auto|tmux|in-process`

锚点：`实现锚点`

这说明 team 不是抽象概念，而是启动级 runtime 形态。

### 3. team 有自己的一套提示与消息层

运行时实现 里可以直接看到：

- team 提示文本
- 协议消息构造器
- in-process teammate loop
- inbox / mailbox 轮询

锚点：

- `实现锚点`
- `实现锚点`
- `实现锚点`

这几处证据放在一起，足以排除一种常见误解：

**team 不是“prompt 里写一句大家合作吧”，而是有真实运行时支撑的。**

---

## 一个更贴近真实工作的拆解例子

还是看登录故障这个例子：

```text
Fix the failing login flow, run the tests, and summarize the root cause.
```

Claude Code 可能这样分层处理：

### 第一步：主 agent 保留主线

主 agent 负责：

- 维护总体目标
- 决定拆哪些子问题
- 汇总最终结果

### 第二步：派 subagent 做独立研究

比如：

- worker A 查 auth entry points
- worker B 复现测试失败
- worker C 查 token refresh 逻辑

这一步的目标是把探索噪音从主线里隔离出去。

### 第三步：如果这些 worker 不是一次性，而是长期存在

那系统就不再只是“发出几个 worker 然后等回来”，而会开始面对：

- 哪个 worker 空闲
- 哪个 worker 可以继续接活
- 哪个 worker 卡在审批上
- 谁能批准 shutdown

此时 subagent 模式已经不够。  
系统必须升级成 team 模式。

这就是为什么你不能把 team 理解成“subagent 的数量更多”。

---

## 从工程角度看，两者的边界在哪里

为了避免概念再次混淆，这里直接按工程问题来划分。

### subagent 关注的是：

- 如何创建一个新的执行单元
- 给它什么 prompt
- 要不要后台运行
- 要不要切 worktree
- 结束后拿回什么结果

### team 关注的是：

- agent 如何命名和寻址
- 内部消息如何传递
- 审批如何转发
- 空闲状态如何广播
- shutdown 如何完成
- teammate 在什么运行模式下存在

所以你会发现：

- subagent 更接近“任务执行接口”
- team 更接近“组织运行时”

---

## 最容易犯的误解

### 误解 1：subagent 就是 team 的简化版

不对。  
subagent 和 team 不是“强弱关系”，而是“层级关系”。

subagent 先解决委派问题。  
team 再解决组织问题。

### 误解 2：team 只是多了个 mailbox

也不对。  
mailbox 只是其中一个部件。

team 真正多出来的是：

- 角色
- 协议
- 审批路径
- 生命周期

### 误解 3：只要模型够聪明，team 就不需要协议

这正好是 Claude Code 本地设计在反对的东西。  
一旦进入真实协作，光靠自然语言是不够稳定的。

后面的 [s10_team_protocols.md](./s10_team_protocols.md) 就是在把这个问题彻底拆开。

---

## 最小伪代码

先看 subagent：

```python
def handle_large_task():
    worker = spawn_subagent(
        description="Trace auth flow",
        prompt="Find all login entry points and token refresh paths."
    )
    return collect_worker_result(worker)
```

它的重点是：

- 主 agent 派出去
- worker 独立做
- 最后把结果拿回来

再看 team：

```python
def team_runtime():
    lead = start_teammate("lead")
    workers = [
        start_teammate("worker-a"),
        start_teammate("worker-b"),
    ]

    while True:
        relay_mailbox_messages()
        process_permission_requests()
        assign_new_work_if_needed()
        handle_shutdown_if_requested()
```

它的重点已经不是“派了几个人”，而是：

- 持续协作
- 持续同步
- 持续治理

---

## 这一章和后面几章的关系

这一章负责把“委派”和“组织”分开。

接下来：

- [s07_worktree_and_isolation.md](./s07_worktree_and_isolation.md)
  解释并行执行为什么需要目录隔离
- [s10_team_protocols.md](./s10_team_protocols.md)
  解释 team 内部为什么必须协议化

也就是说：

- `s06` 先告诉你系统里有哪些层
- `s07` 讲并行时怎么不互相踩
- `s10` 讲协作时怎么不互相乱

---

## 这一章真正想让你学会什么

一句话总结：

**subagent 是委派，team 是组织。**

Claude Code 的成熟之处，不只是它会“再派一个 agent”，而是它继续把这个能力往前推进成了：

- 独立执行单元
- 长期协作结构
- 协议化消息
- 权限与生命周期治理

从这里开始，多 agent 就不再只是“多开几个模型”，而是正式 runtime 设计问题。
