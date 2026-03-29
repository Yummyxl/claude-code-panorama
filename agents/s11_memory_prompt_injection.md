# s11 - Memory Prompt Injection

## 这一章要解决什么问题

当一个系统说自己“有 memory”时，最容易引发两种误解：

- 误以为它有一个神秘的长期人格内核
- 误以为它只是把历史对话存起来

Claude Code 本地代码说明，两种都不对。

它的 memory 更准确地说是：

**一套受控的长期背景材料，再通过 prompt section 注入给模型。**

这一章就是把这个过程拆开。

---

## 先用一句最重要的话把它钉住

**Claude Code 的 memory，不是模型自己“记住了”，而是运行时每轮重新把长期背景告诉模型。**

只要你先抓住这一点，后面很多细节都会顺很多：

- 为什么 memory 会进入 system prompt
- 为什么 memory 和 compaction 不是一回事
- 为什么 TeamMem 会单独存在

---

## Claude Code 的 memory 不是黑盒

从本地代码看，Claude Code 没有表现出一个神秘的“隐形记忆脑”。  
它做的事情更工程化，也更可解释：

1. 枚举记忆来源
2. 读取并校验内容
3. 组织成 memory section
4. 在每轮请求前注入 system prompt

这意味着 memory 在 Claude Code 里首先是运行时机制，其次才是产品语义。

---

## 先想一个最直观的例子

假设你长期在同一个仓库里工作，Claude Code 需要反复记住这些事实：

- 这个项目默认用 pnpm
- 测试只跑某个目录更快
- 团队约定提交信息要符合某个格式
- 某个 teammate 已经确认 auth 模块有历史坑

这些事实和“刚刚上一轮发生了什么”不是一类东西。

它们更像：

- 长期背景
- 稳定约定
- 项目上下文
- 团队共享认知

这就是为什么 Claude Code 不把它们混进普通消息历史里，而是单独走 memory 注入路径。

---

## 它有哪些 memory 来源

本地代码里可以直接看到至少这些来源名：

- `Managed`
- `User`
- `Project`
- `Local`
- `AutoMem`
- `TeamMem`

锚点：`cli.js:1256`

这说明 Claude Code 从一开始就没有把 memory 设计成“一坨统一文本”。

它在区分：

- 用户私人背景
- 项目背景
- 本地上下文
- 自动积累的信息
- 团队共享信息

这是一种非常成熟的设计，因为不同来源天然就应该有不同生命周期和不同作用范围。

---

## TeamMem 为什么特别重要

只要系统支持多 agent，记忆就不能只有“我知道什么”，还必须有“团队应该共同知道什么”。

本地代码里可以直接看到：

- `TeamMem` 是单独来源
- 有单独开关和路径逻辑
- 还会以显式标签插入 prompt

例如本地可以看到：

- `<team-memory-content source="shared">`

锚点：`cli.js:1256`

这说明 TeamMem 不是“聊天记录顺手共享一下”，而是正式的共享背景层。

这点特别关键，因为它表明：

Claude Code 已经承认，多 agent 协作不仅要共享任务，还要共享长期背景。

---

## memory 是怎么进入模型的

这一步才是最值得学的。

Claude Code 并不是把 memory 临时塞到用户消息里。  
它也不是把 memory 和历史压缩混在一起。

从本地 `PD(...)` 可以确认：

- `memory` 是 system prompt 的正式 section

锚点：`cli.js:1480`

也就是说，Claude Code 的做法是：

**每轮请求前，运行时重新准备 memory section，再把它作为 prompt 一部分交给模型。**

所以 memory 的本质不是“模型神奇记住了”。  
而是“运行时稳定地再告诉它一次”。

---

## 这和 compaction 有什么本质区别

这是这一章必须彻底钉死的一点。

```text
memory
  = 长期背景
  = 进入 system prompt

compaction
  = 当前会话的压缩延续
  = 进入 message continuation state
```

它们都在帮助模型持续工作。  
但一个面向长期背景，一个面向会话延续。

所以你可以把两者理解成：

- memory 解决“以后每轮最好都知道什么”
- compaction 解决“刚刚这段会话怎样继续带下去”

这就是为什么 Claude Code 同时需要两者，而且不能合并成一个机制。

---

## 为什么 memory 要放在 prompt，而不是普通消息里

因为它表达的不是“对话里刚发生的内容”，而是：

- 持续有效的背景事实
- 当前轮也应该遵守的长期约束

把这种内容放进 system prompt，会产生两个好处：

### 1. 语义位置更稳定

它不会和普通 user / assistant 消息混在一起。

### 2. 每轮都能稳定重建

只要运行时在请求前重新注入，memory 就不会依赖模型“自己回忆起之前某条消息”。

这就是 Claude Code memory 设计里最重要的工程性。

---

## 最小运行图

```text
memory sources
  -> read and validate
  -> classify by source
  -> render as memory section
  -> inject into system prompt
  -> model sees it again on the next turn
```

这里最重要的不是“存了多久”，而是：

memory 在 Claude Code 里是一个主动注入链，而不是被动残留物。

---

## 最小伪代码

```python
def build_memory_section():
    pieces = []
    pieces += load_user_memory()
    pieces += load_project_memory()
    pieces += load_local_memory()
    pieces += load_automem()
    pieces += load_teammem()
    return render_memory_text(pieces)


def build_system_prompt():
    return [
        base_role_prompt(),
        environment_prompt(),
        build_memory_section(),
    ]
```

这个伪代码想表达的是：

memory 不是“历史里自己长出来的”，  
而是运行时主动构造出来的 prompt section。

---

## 这一章和前后章节怎么连

这章最好和两章一起看：

- [s04_compaction_and_memory.md](./s04_compaction_and_memory.md)
  那一章负责把 compaction 和 memory 的边界钉死
- [s13_prompt_catalog.md](./s13_prompt_catalog.md)
  那一章会继续展开 memory section 在 prompt 装配线中的真实位置

所以：

- `s04` 讲边界
- `s11` 讲注入机制
- `s13` 讲 prompt 装配细节

---

## 这一章真正想让你学会什么

一句话总结：

**Claude Code 的 memory，本质上是长期背景的受控 prompt 注入。**

它不是黑盒人格。  
不是自动历史替代。  
不是神秘长期脑。

它是可枚举、可校验、可注入的运行时层。
