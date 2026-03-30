# s05 - Task Todo Background

## 这一章要解决什么问题

一个真正能连续工作的 coding agent，不能只会回答：

**“我这一秒钟下一步做什么。”**

它还必须同时回答另外三个问题：

- 我现在到底在做哪件大事
- 这件大事拆成了哪几步
- 某个动作如果很慢，我怎么不停下来干等

Claude Code 运行时里，对应的是三层不同能力：

- `task`
- `todo`
- `background command`

这一章不是把它们当成同义词，而是把它们彻底拆开。

---

## 先用一个真实工作例子把三层区分开

假设用户说：

```text
Fix the failing login flow, run the tests, and summarize the root cause.
```

这时候 Claude Code 运行时里其实会同时存在三种不同粒度的状态。

### 第一层：task

这整件事本身就是一个 task。

比如：

- 修复登录失败

它回答的是：

- 我现在这轮大工作到底是什么

### 第二层：todo

为了完成这个 task，当前会拆出一些步骤。

比如：

- 复现失败
- 定位根因
- 修改代码
- 跑验证
- 写结论

它回答的是：

- 我当前的执行路线是什么

### 第三层：background

其中某一步可能很慢，比如跑整套测试。

这时候就不应该把整个 agent loop 卡死。  
所以系统会把它放进后台执行。

它回答的是：

- 慢动作怎样异步继续推进

如果你用这个例子去看 Claude Code，就会很容易明白：

**task、todo、background 不是三个名字相近的 feature，而是三种不同粒度的运行时控制。**

---

## 为什么很多人会把这三者学混

因为它们都和“工作过程”有关。

但工程上，它们处理的是完全不同的问题。

### task 处理的是目标对象

当前系统正在处理哪一件正式工作。

### todo 处理的是执行分解

为了推进这件工作，现在的步骤序列是什么。

### background 处理的是执行节奏

某个动作太慢时，怎样不让主循环停摆。

所以如果把它们全归成“任务系统”，你就会失去最重要的结构感。

---

## 为什么 Claude Code 必须三层同时存在

只要少一层，系统就会失去一种关键能力。

### 只有 task，没有 todo

模型知道自己在做什么大事，但很容易在过程里漂移。

### 只有 todo，没有 task

模型知道自己下一步做什么，但缺少更高层的目标对象，长期工作容易丢主线。

### 只有同步执行，没有 background

任何慢命令都会把整个 agent loop 卡住。

所以这三层不是“功能越做越多”，而是：

**Claude Code 想要长期工作，就必须同时拥有目标层、路径层和节奏层。**

---

## 实现能确认什么

从运行时实现 里可以直接看到这些名字：

- `TaskCreate`
- `TaskOutput`
- `TaskStop`
- `TodoWrite`
- `ExitPlanMode`
- `AskUserQuestion`

锚点：

- `实现锚点`
- `实现锚点`

从工具定义 又能看到对应输入定义：

- `工具定义锚点`
- `工具定义锚点`
- `工具定义锚点`

这说明：

- task / todo 不是 UI 标签
- 它们是正式 runtime object
- 它们已经进入 Claude Code 的工具面

这里也要注意命名口径：

- 这一章这段先说的是 runtime 名字，例如 `TaskCreate`
- 工具定义层里未必用同一个名字暴露 schema

所以这段是在讲“运行时里发生了什么”，不是在给 schema 做逐字段列表。

---

## Claude Code 为什么连 Bash 都要“任务化”

这一点非常值得学。

运行时里的 Bash 路径能看到这些字段：

- `backgroundTaskId`
- `assistantAutoBackgrounded`
- `persistedOutputPath`
- `persistedOutputSize`

锚点：`实现锚点`

这说明 Claude Code 对 shell 的理解不是：

```text
执行一条命令
-> 打印点输出
-> 结束
```

而更像：

```text
执行一个动作
-> 这个动作可能很慢
-> 可能要后台化
-> 可能要持久化输出
-> 后面还要继续追踪它
```

这其实就是把 Bash 也纳入了过程管理框架。

换句话说，Claude Code 不是“额外有个后台任务功能”，  
而是它从设计上就把执行动作视为可跟踪、可异步、可回收的 runtime object。

---

## `TodoWrite` 为什么不是可有可无的小工具

很多人会误以为 todo 只是“顺手记个笔记”。

但 Claude Code 这里的 `TodoWrite` 不是普通文本记录。  
它是把当前执行路线显式写入运行时。

这件事的价值在于：

- 主线不会只存在于模型短期注意力里
- 用户可以看到当前步骤结构
- 后续 agent / teammate 可以接手时更容易对齐状态

所以 todo 的意义不是“漂亮 UI”，而是：

**把过程从模型脑内拉到运行时显式状态里。**

---

## `TaskOutput` / `TaskStop` 为什么也很重要

一旦 background task 存在，系统就必须继续回答两个问题：

- 我怎样重新拿回它的结果
- 我怎样在必要时停掉它

这就是：

- `TaskOutput`
- `TaskStop`

存在的原因。

如果没有这两个接口，后台任务就会沦为“扔出去之后靠运气找回来”的半成品机制。

而 Claude Code 明确不接受这种松散设计。

---

## 最小运行图

你可以把这一章压成下面这条过程线：

```text
task
  -> 定义当前大目标
  -> todo
       -> 把目标拆成当前步骤
       -> background command
            -> 慢动作异步跑
            -> TaskOutput 取回结果
            -> TaskStop 中断任务
```

这里最重要的一点是：

这不是三套无关功能，而是一条层层递进的过程治理链。

---

## 最小伪代码

```python
def handle_big_task():
    task_id = create_task("Fix auth failure")

    write_todo([
        "Reproduce failing test",
        "Find root cause",
        "Patch auth flow",
        "Run verification",
        "Summarize result",
    ])

    run = bash("npm test", run_in_background=True)

    if run.backgroundTaskId:
        result = read_task_output(run.backgroundTaskId, block=True, timeout=30000)
        if failed(result):
            stop_task(run.backgroundTaskId)

    complete_task(task_id)
```

这个伪代码的重点不是字段细节，而是三层关系：

- task 定义目标
- todo 组织路径
- background 处理慢动作

---

## 这一章和后面几章怎么连起来

这一章学会之后，你再去看：

- [s06_subagent_and_teams.md](./s06_subagent_and_teams.md)
  会更容易理解为什么多 agent 需要共享过程状态
- [s07_worktree_and_isolation.md](./s07_worktree_and_isolation.md)
  会更容易理解为什么并行执行不仅要分目录，也要分任务

也就是说：

- `s05` 先解决“工作过程怎样被运行时显式管理”
- 后面几章再解决“这些过程怎样被多人并行推进”

---

## 这一章真正想让你学会什么

一句话总结：

**task 管目标，todo 管路径，background 管节奏。**

把这三层分清楚，你后面再看多 agent、team、worktree，就会知道为什么 Claude Code 必须把“执行过程”本身做成一级运行时状态。
