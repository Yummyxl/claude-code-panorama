# s07 - Worktree And Isolation

## 这一章要解决什么问题

只要 Claude Code 开始支持：

- subagent
- team
- 并行任务
- 长时间修改

系统就会立刻撞上一个非常现实的问题：

**多个 agent 在同一个目录里工作，怎么不互相毁掉彼此的现场。**

Claude Code 的回答不是：

- 大家小心一点
- 提醒模型别冲突
- 靠 git diff 最后再收拾

它的回答是：

**直接做隔离。**

这就是这一章真正要讲的东西。

---

## 先想一个不做隔离的灾难场景

假设主 agent 负责总体修复，两个 worker 并行处理：

- worker A 查 auth flow
- worker B 改测试

如果三者都在同一个目录里工作，很快就会出现这些问题：

- git status 混在一起，看不清谁改了什么
- 一个 agent 改完文件，另一个 agent 还在读旧版本
- hook、临时文件、构建输出互相污染
- 当前目录已经不再代表“某一个执行单元的工作空间”

注意，这里最要命的一点不是“可能 merge 冲突”，而是：

**每个 agent 对自己所处世界的认知会变得不稳定。**

而 agent runtime 最怕的就是环境语义漂移。

---

## 为什么隔离不是礼貌，而是基础设施

很多系统把隔离理解成一种“更稳一点的增强项”。  
Claude Code 本地设计不是这样。

它更接近这个逻辑：

```text
单 agent 时，共享目录有时还能勉强工作
多 agent / 并行 / 长任务时，共享目录会系统性失效
所以隔离不是优化，而是前置条件
```

这就是为什么 worktree 在 Claude Code 里不是附加功能，而是并行执行的基础设施。

---

## Claude Code 到底在隔离什么

很多人一听到 worktree，会下意识只想到 git。

但本地代码说明，Claude Code 真正抽象的不是单一命令，而是：

**给 agent 一个独立、稳定、可追踪的工作空间。**

这个工作空间至少要隔离：

- 当前目录语义
- git 状态
- branch / worktree path
- 某些本地设置
- hook 行为

所以 worktree 的本质不是“多开一个仓库副本”，  
而是“把执行环境做成一级 runtime 对象”。

---

## Claude Code 至少支持哪两种隔离路径

从本地代码看，它至少支持两种思路。

### 1. 标准 git worktree

这是最直接、最容易理解的路径：

- 创建隔离工作副本
- 切到新的 branch / worktree path
- 让这个 agent 在那个目录继续工作

### 2. hook-based fallback

如果不在标准 git 场景里，Claude Code 仍然预留了基于 hook 的创建 / 删除路径。

这说明系统真正要的不是“必须有 git worktree 命令”，而是：

**必须有一个隔离 backend。**

这个判断非常重要，因为它暴露了 Claude Code 的设计层级：

- 外层抽象是 isolation runtime
- git worktree 只是其中一种实现

---

## 本地代码能确认什么

本地代码里可以直接看到完整的 worktree 生命周期逻辑，包括：

- 创建 worktree
- 复制 `settings.local.json`
- 配置 hooks path
- sparse checkout
- 删除 worktree
- 删除 worktree branch

锚点：[`cli.js:1280`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js#L1280)

本地 schema 里还能直接看到：

- `EnterWorktreeInput`
- `ExitWorktreeInput`

锚点：

- [`sdk-tools.d.ts:2135`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L2135)
- [`sdk-tools.d.ts:2141`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts#L2141)

这说明两件事：

- worktree 不是启动时顺手做一下
- worktree 是正式工具面能力

也就是说，模型可以通过受控接口进入和退出隔离环境，而不是自己乱写一串 git 命令。

---

## `EnterWorktree` / `ExitWorktree` 为什么要做成正式工具

这是最值得 harness 工程师学习的一点。

如果 Claude Code 只是让模型直接写：

```bash
git worktree add ...
```

那运行时就很难稳定跟踪：

- 当前 agent 到底切到哪个目录了
- 原始 cwd 是什么
- 当前 worktree branch 是什么
- 删除时到底会丢什么

而现在它做成正式工具之后，运行时就能把这些状态显式建模。

这也是为什么 `ExitWorktreeOutput` 会暴露：

- `originalCwd`
- `worktreePath`
- `worktreeBranch`
- `discardedFiles`
- `discardedCommits`

因为这不再是“git 命令副作用”，而是 runtime 管理对象。

---

## worktree 和 subagent / team 的关系

这一章最好和 [s06_subagent_and_teams.md](/Users/unicorn/aiprojects/claude/agents/s06_subagent_and_teams.md) 连起来看。

`s06` 解释了：

- subagent 是正式委派
- team 是持续组织

而这一章解释的是：

**这些委派和组织如果要安全并行，底层执行空间就必须被隔离。**

所以可以把关系压成一张图：

```text
subagent / team
  -> 带来并行执行
  -> 并行执行带来环境冲突风险
  -> worktree 负责把环境冲突变成系统上不成立的事
```

---

## 最小运行图

```text
main agent
  -> create isolated workspace
  -> spawn worker in that workspace
  -> worker reads / edits / runs commands
  -> worker exits workspace
  -> runtime keeps or removes worktree
```

这里最重要的是：

agent 不是被提醒“记得别互相冲突”。  
而是被直接放进不同的世界里。

---

## 最小伪代码

```python
def enter_isolated_workspace(name):
    if in_git_repo():
        return create_git_worktree(name)
    if has_worktree_hooks():
        return run_worktree_create_hook(name)
    raise RuntimeError("no isolation backend")


def run_worker_in_isolation(name, prompt):
    env = enter_isolated_workspace(name)
    worker = spawn_agent(prompt=prompt, cwd=env.worktree_path)
    return worker
```

真正的 Claude Code 比这复杂得多，但最核心的设计思想就是：

不要让多个执行单元共享未经隔离的工作空间。

---

## 这一章真正想让你学会什么

一句话总结：

**在 Claude Code 里，隔离不是行为约束，而是系统事实。**

它不是让模型“尽量别互相冲突”。  
它是直接给模型不同的工作环境。

这就是成熟 agent harness 和“多开几个线程”之间的差别。
