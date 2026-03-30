# s12 - Tools Catalog

## 这一章怎么读

前面的章节已经解释了 Claude Code 为什么需要工具面。  
这一章不再讲抽象原则，而是做一件更直接的事：

**把当前交付实现里的工具定义整理成一份可学习、可查表、可模仿的手册。**

这里的依据主要是：

- `工具定义`
- 少量 `实现主干` 运行时锚点，例如 Bash 的后台行为

这一章的目标不是把类型文件机械抄写一遍，而是让你读完之后能回答：

- 这个工具是干什么的
- 模型调用它时要提供哪些字段
- 运行时会返回什么
- 这个工具为什么是单独存在的
- 如果我要自己做一个类似 harness，应该怎么设计 demo

你可以把这一章当成两份材料一起读：

### 第一份：给使用者的“工具说明书”

帮助你理解：

- 模型到底能调用什么
- 每个动作的输入输出边界是什么
- 哪些能力是内建的，哪些能力是开放扩展的

### 第二份：给 harness 工程师的“接口设计样本”

帮助你学习：

- 为什么 `FileEdit` 和 `FileWrite` 要拆开
- 为什么 `AskUserQuestion` 不是普通文本提问
- 为什么 `EnterWorktree` 是正式工具而不是 Bash 命令模板
- 为什么 `TaskOutput` 这种“过程管理能力”也被做成工具

如果你带着第二种眼光读，这一章的学习价值会高很多。

---

## 这一章最好怎么用

这章很长，所以最好不要线性硬读。

更高效的读法是：

### 第一步：先看工具分组

先看六大组的划分，弄清：

- 哪些工具是在感知世界
- 哪些工具是在执行动作
- 哪些工具是在治理过程
- 哪些工具是在协作和切环境

### 第二步：再挑你最关心的工具深读

比如你关心：

- 文件编辑，就先读 `FileRead / FileEdit / FileWrite`
- 多 agent，就先读 `Agent / EnterWorktree / ExitWorktree`
- 人机澄清，就先读 `AskUserQuestion`

### 第三步：最后把它当查表手册

等你已经理解整体结构之后，这章最好的用法就变成：

- 查字段
- 查输出
- 查 demo
- 查设计意图

这样读，这章会更像一份高密度参考手册，而不是需要一次性吞完的长文。

---

## Claude Code 工具面的完整清单

这里要先把口径说清楚。

本章这份清单讲的是：

**`工具定义` 里显式导出的 SDK schema 工具面。**

它非常重要，但它不等于“prompt 里对模型可见的全部 runtime 工具名”。

例如在其他章节里你还会看到：

- `Read / Edit / Write`
- `TaskCreate`
- `Skill`

这些名字出现在 prompt runtime 或运行时映射层里。  
所以最准确的理解是：

- `s12` 主要整理 SDK schema 暴露出来的工具定义
- `s13` 会继续解释 prompt 里可见的 runtime 工具名映射

只要先把这层边界记住，后面你看到 `FileRead` 和 `Read`、`TodoWrite` 和 `TaskCreate` 这些名字并存时，就不会误以为文档前后矛盾。

从工具输入定义集合可以直接确认，SDK schema 至少定义了这些输入工具：

- `AgentInput`
- `BashInput`
- `TaskOutputInput`
- `ExitPlanModeInput`
- `FileEditInput`
- `FileReadInput`
- `FileWriteInput`
- `GlobInput`
- `GrepInput`
- `TaskStopInput`
- `ListMcpResourcesInput`
- `McpInput`
- `NotebookEditInput`
- `ReadMcpResourceInput`
- `TodoWriteInput`
- `WebFetchInput`
- `WebSearchInput`
- `AskUserQuestionInput`
- `ConfigInput`
- `EnterWorktreeInput`
- `ExitWorktreeInput`

锚点：`工具定义锚点`

相应的输出定义至少包括：

- `AgentOutput`
- `BashOutput`
- `ExitPlanModeOutput`
- `FileEditOutput`
- `FileReadOutput`
- `FileWriteOutput`
- `GlobOutput`
- `GrepOutput`
- `TaskStopOutput`
- `ListMcpResourcesOutput`
- `McpOutput`
- `NotebookEditOutput`
- `ReadMcpResourceOutput`
- `TodoWriteOutput`
- `WebFetchOutput`
- `WebSearchOutput`
- `AskUserQuestionOutput`
- `ConfigOutput`
- `EnterWorktreeOutput`
- `ExitWorktreeOutput`

锚点：`工具定义锚点`

为了便于学习，我们按“用途”把它们分成六组。

---

## 最小伪代码

```python
def dispatch_tool(name, raw_input):
    schema = sdk_schema[name]
    parsed_input = validate_against_schema(raw_input, schema)
    output = tool_handlers[name](parsed_input)
    return coerce_to_typed_output(name, output)
```

这段伪代码对应的是 s12 的核心视角：

- 工具先有 schema
- 输入先过校验
- 再进入具体 handler
- 最后返回有形状的输出对象

---

## 第一组：委派与协作工具

### 1. Agent

#### 它解决什么问题

当前 agent 发现某个子问题需要独立执行单元时，不能只在脑子里“分身”，而要真正派生一个 worker。  
`Agent` 工具就是这个正式委派接口。

#### 输入定义

`AgentInput` 的关键字段有：

- `description: string`
  一个很短的任务名，3 到 5 个词。
- `prompt: string`
  真正交给子 agent 的完整任务描述。
- `subagent_type?: string`
  要求使用专门化 agent 类型。
- `model?: "sonnet" | "opus" | "haiku"`
  子 agent 的模型覆盖。
- `run_in_background?: boolean`
  是否后台运行。
- `name?: string`
  给 agent 起名，便于后续寻址。
- `team_name?: string`
  这个 agent 应该属于哪个 team。
- `mode?: "acceptEdits" | "bypassPermissions" | "default" | "dontAsk" | "plan"`
  子 agent 的权限模式。
- `isolation?: "worktree"`
  是否要求 worktree 隔离。

锚点：`工具定义锚点`

#### 输出定义

`AgentOutput` 有两种主要形态。

**同步完成形态**

- `agentId`
- 可选 `agentType`
- `content`
- `totalToolUseCount`
- `totalDurationMs`
- `totalTokens`
- `usage`
- `status: "completed"`
- `prompt`

**异步启动形态**

- `status: "async_launched"`
- `agentId`
- `description`
- `prompt`
- `outputFile`
- 可选 `canReadOutputFile`

锚点：`工具定义锚点`

#### Demo

```json
{
  "description": "Search auth flow",
  "prompt": "Find all OAuth entry points and summarize which files define token exchange and callback handling.",
  "subagent_type": "general-purpose",
  "run_in_background": true,
  "name": "auth-worker",
  "mode": "plan",
  "isolation": "worktree"
}
```

#### 怎么理解这个设计

这里最值得学的是：Claude Code 把“再派一个 agent”提升成了正式动作。  
这说明它不是单 agent 脚本，而是带 worker 概念的 runtime。

---

## 第二组：本地执行工具

### 2. Bash

#### 它解决什么问题

很多现实动作暂时不值得单独造专门工具，例如：

- 跑测试
- 跑构建
- 跑 git 命令
- 跑项目脚本

所以 Claude Code 仍然保留了 Bash 作为通用执行面。

#### 输入定义

- `command: string`
  要执行的命令。
- `timeout?: number`
  毫秒超时，最大 600000。
- `description?: string`
  要用主动语态、清晰地说明命令在做什么。
- `run_in_background?: boolean`
  是否后台运行。
- `dangerouslyDisableSandbox?: boolean`
  是否危险地关闭 sandbox。

锚点：`工具定义锚点`

#### 输出定义

- `stdout: string`
- `stderr: string`
- `rawOutputPath?: string`
- `interrupted: boolean`
- `isImage?: boolean`
- `backgroundTaskId?: string`
- `backgroundedByUser?: boolean`
- `assistantAutoBackgrounded?: boolean`
- `dangerouslyDisableSandbox?: boolean`
- `returnCodeInterpretation?: string`
- `noOutputExpected?: boolean`
- `structuredContent?: unknown[]`
- `persistedOutputPath?: string`
- `persistedOutputSize?: number`

锚点：`工具定义锚点`

#### Demo

```json
{
  "command": "npm test -- --runInBand",
  "description": "Run the project's test suite serially",
  "timeout": 120000,
  "run_in_background": false
}
```

后台版：

```json
{
  "command": "npm run dev",
  "description": "Start the local development server",
  "run_in_background": true
}
```

#### 运行时补充

本地 `实现主干` 明确表明 Bash 不是“裸命令”，还带有后台任务和输出持久化逻辑。

锚点：`实现锚点`

#### 怎么理解这个设计

Claude Code 不是拒绝 Bash。  
它只是把 Bash 放进了可托管的 runtime 里。

---

## 第三组：文件与代码感知工具

### 3. FileRead

#### 它解决什么问题

把本地文件读成模型可消费的结构。

#### 输入定义

- `file_path: string`
  绝对路径。
- `offset?: number`
  从第几行开始。
- `limit?: number`
  读多少行。
- `pages?: string`
  PDF 页码范围，例如 `"1-5"`。

锚点：`工具定义锚点`

#### 输出定义

`FileReadOutput` 不是单一文本，而是五种返回类型：

**文本**

- `type: "text"`
- `file.filePath`
- `file.content`
- `file.numLines`
- `file.startLine`
- `file.totalLines`

**图片**

- `type: "image"`
- `file.base64`
- `file.type`
- `file.originalSize`
- 可选 `file.dimensions`

**notebook**

- `type: "notebook"`
- `file.filePath`
- `file.cells`

**PDF**

- `type: "pdf"`
- `file.filePath`
- `file.base64`
- `file.originalSize`

**PDF 拆页**

- `type: "parts"`
- `file.filePath`
- `file.originalSize`
- `file.count`
- `file.outputDir`

锚点：`工具定义锚点`

#### Demo

读取源码：

```json
{
  "file_path": "/abs/project/src/auth.ts",
  "offset": 1,
  "limit": 200
}
```

读取 PDF 指定页：

```json
{
  "file_path": "/abs/project/specs/api.pdf",
  "pages": "1-3"
}
```

#### 怎么理解这个设计

Claude Code 的“读取”不是文本专用，而是多模态感知接口。

---

### 4. FileEdit

#### 它解决什么问题

对已有文件做局部、精确替换，而不是整份覆盖。

#### 输入定义

- `file_path`
- `old_string`
- `new_string`
- `replace_all?: boolean`

锚点：`工具定义锚点`

#### 输出定义

- `filePath`
- `oldString`
- `newString`
- `originalFile`
- `structuredPatch`
- `userModified`
- `replaceAll`
- 可选 `gitDiff`

锚点：`工具定义锚点`

#### Demo

```json
{
  "file_path": "/abs/project/src/auth.ts",
  "old_string": "const timeout = 1000;",
  "new_string": "const timeout = 5000;",
  "replace_all": false
}
```

#### 怎么理解这个设计

`FileEdit` 的重点是“局部可验证”。  
这有利于 UI 呈现 diff，也有利于避免整份文件被粗暴覆盖。

---

### 5. FileWrite

#### 它解决什么问题

整份写入文件，适合新建文件或彻底重写。

#### 输入定义

- `file_path`
- `content`

锚点：`工具定义锚点`

#### 输出定义

- `type: "create" | "update"`
- `filePath`
- `content`
- `structuredPatch`
- `originalFile`
- 可选 `gitDiff`

锚点：`工具定义锚点`

#### Demo

```json
{
  "file_path": "/abs/project/docs/auth-notes.md",
  "content": "# Auth Notes\n\nThis file documents the login flow.\n"
}
```

#### 怎么理解这个设计

Claude Code 把“局部编辑”和“整份写入”拆成两个工具，是为了让动作语义更清楚。

---

### 6. NotebookEdit

#### 它解决什么问题

Jupyter notebook 不是普通文本文件。  
Claude Code 为它单独做了编辑工具。

#### 输入定义

- `notebook_path`
- `cell_id?`
- `new_source`
- `cell_type?: "code" | "markdown"`
- `edit_mode?: "replace" | "insert" | "delete"`

锚点：`工具定义锚点`

#### 输出定义

- `new_source`
- `cell_id?`
- `cell_type`
- `language`
- `edit_mode`
- 可选 `error`
- `notebook_path`
- `original_file`
- `updated_file`

锚点：`工具定义锚点`

#### Demo

```json
{
  "notebook_path": "/abs/project/analysis.ipynb",
  "cell_id": "cell-12",
  "new_source": "df.groupby('user_id').size().head()",
  "cell_type": "code",
  "edit_mode": "replace"
}
```

#### 怎么理解这个设计

这说明 Claude Code 不愿把 notebook 退化成 JSON 字符串编辑。

---

## 第四组：搜索与检索工具

### 7. Glob

#### 它解决什么问题

按路径模式找文件。

#### 输入定义

- `pattern`
- 可选 `path`

锚点：`工具定义锚点`

#### 输出定义

- `durationMs`
- `numFiles`
- `filenames`
- `truncated`

锚点：`工具定义锚点`

#### Demo

```json
{
  "pattern": "**/*auth*.ts",
  "path": "/abs/project/src"
}
```

---

### 8. Grep

#### 它解决什么问题

按内容搜索代码或文本。

#### 输入定义

关键字段包括：

- `pattern`
- `path`
- `glob`
- `output_mode`
- `-B`
- `-A`
- `-C`
- `context`
- `-n`
- `-i`
- `type`
- `head_limit`
- `offset`
- `multiline`

锚点：`工具定义锚点`

#### 输出定义

- `mode?`
- `numFiles`
- `filenames`
- `content?`
- `numLines?`
- `numMatches?`
- `appliedLimit?`
- `appliedOffset?`

锚点：`工具定义锚点`

#### Demo

```json
{
  "pattern": "OAuth|token exchange",
  "path": "/abs/project/src",
  "glob": "*.{ts,tsx}",
  "output_mode": "content",
  "-n": true,
  "context": 2,
  "head_limit": 20
}
```

#### 怎么理解这个设计

Claude Code 希望“查找内容”成为结构化动作，而不是默认退回 shell。

---

### 9. WebFetch

#### 它解决什么问题

抓取网页内容后，再对内容执行一个处理 prompt。

#### 输入定义

- `url`
- `prompt`

锚点：`工具定义锚点`

#### 输出定义

- `bytes`
- `code`
- `codeText`
- `result`
- `durationMs`
- `url`

锚点：`工具定义锚点`

#### Demo

```json
{
  "url": "https://example.com/docs/auth",
  "prompt": "Extract the login callback requirements and summarize the error cases."
}
```

---

### 10. WebSearch

#### 它解决什么问题

进行网络搜索，并把结果组织成可继续推理的结构。

#### 输入定义

- `query`
- `allowed_domains?`
- `blocked_domains?`

锚点：`工具定义锚点`

#### 输出定义

- `query`
- `results`
  其中每个元素要么是字符串，要么是带 `tool_use_id` 和 `content[]` 的搜索命中集合
- `durationSeconds`

锚点：`工具定义锚点`

#### Demo

```json
{
  "query": "RFC 9728 protected resource metadata",
  "allowed_domains": ["rfc-editor.org", "ietf.org"],
  "blocked_domains": ["reddit.com"]
}
```

---

## 第五组：任务治理与用户澄清工具

### 11. TaskOutput

#### 它解决什么问题

从后台任务或异步任务读取结果。

#### 输入定义

`TaskOutputInput` 在本地类型文件里只有三个字段，而且都是真字段，不是装饰性参数：

- `task_id: string`
  要读取哪个后台任务。
- `block: boolean`
  是“立即拿现有结果”还是“可以阻塞等待”。
- `timeout: number`
  最长等待多少毫秒。

锚点：`工具定义锚点`

#### 输出定义

本地 `工具定义` 里**没有**单独导出 `TaskOutputOutput` 或同名输出 interface。

这点必须明确记下来，因为它意味着：

- 当前公开类型文件只给了“怎么取任务输出”
- 但没有像 `BashOutput`、`TodoWriteOutput` 那样，给出独立、稳定的结果对象 schema

锚点：`工具定义锚点`、`工具定义锚点`

#### Demo

```json
{
  "task_id": "task-123",
  "block": true,
  "timeout": 30000
}
```

#### 怎么理解这个设计

这说明 Claude Code 里“任务结果”本身就是一个正式读接口，不是靠日志瞎翻。

更进一步说：

- 任务是 runtime 一级对象
- 任务结果读取是一级工具
- 但它的返回体并没有在公开类型文件里完全暴露

这属于“接口有、结果 schema 暂未公开完整类型”的典型信号。

---

### 12. TaskStop

#### 它解决什么问题

停止一个后台任务。

#### 输入定义

- `task_id?`
- `shell_id?` 已标记 deprecated

锚点：`工具定义锚点`

#### 输出定义

- `message`
- `task_id`
- `task_type`
- 可选 `command`

锚点：`工具定义锚点`

#### Demo

```json
{
  "task_id": "bg-serve-01"
}
```

---

### 13. TodoWrite

#### 它解决什么问题

把当前工作步骤正式写入任务板。

#### 输入定义

`todos` 是对象数组，每项至少有：

- `content`
- `status: "pending" | "in_progress" | "completed"`
- `activeForm`

锚点：`工具定义锚点`

#### 输出定义

- `oldTodos`
- `newTodos`
- 可选 `verificationNudgeNeeded`

锚点：`工具定义锚点`

#### Demo

```json
{
  "todos": [
    {
      "content": "Reproduce failing login test",
      "status": "completed",
      "activeForm": "Reproducing failing login test"
    },
    {
      "content": "Patch token refresh logic",
      "status": "in_progress",
      "activeForm": "Patching token refresh logic"
    }
  ]
}
```

---

### 14. ExitPlanMode

#### 它解决什么问题

正式结束 plan mode，并附带允许的 prompt-based permissions。

#### 输入定义

关键可见字段：

- `allowedPrompts?`
  每项包含：
  - `tool: "Bash"`
  - `prompt`

锚点：`工具定义锚点`

#### 输出定义

- `plan`
- `isAgent`
- 可选 `filePath`
- `hasTaskTool?`
- `planWasEdited?`
- `awaitingLeaderApproval?`
- `requestId?`

锚点：`工具定义锚点`

#### Demo

```json
{
  "allowedPrompts": [
    {
      "tool": "Bash",
      "prompt": "run tests"
    },
    {
      "tool": "Bash",
      "prompt": "install dependencies"
    }
  ]
}
```

#### 怎么理解这个设计

这说明 plan mode 不是聊天语气，而是正式运行模式。

---

### 15. AskUserQuestion

#### 它解决什么问题

运行过程中向用户发起结构化澄清，而不是靠随便打一段文本。

#### 输入定义

`AskUserQuestionInput` 是当前工具面里约束最细的一类 schema 之一。

顶层字段至少有：

- `questions`
  长度必须是 1 到 4。
- 可选 `answers`
  本地注释写的是 “User answers collected by the permission component”。
- 可选 `annotations`
  记录预览选择或补充说明。
- 可选 `metadata`
  不展示给用户，只做追踪和分析用途。

`questions[]` 中每个问题对象必须包含：

- `question: string`
  完整问题句，应该清晰、具体，并且以问号结尾。
- `header: string`
  超短标签，最大 12 字符。
- `options`
  选项数必须是 2 到 4。
- `multiSelect: boolean`
  是否允许多选。

每个 `options[]` 元素包含：

- `label: string`
  用户看到的短标签，推荐 1 到 5 个词。
- `description: string`
  解释这个选项意味着什么。
- `preview?: string`
  可选预览内容，适合放 mockup、代码片段、视觉对比内容。

锚点：`工具定义锚点`

#### 输出定义

`AskUserQuestionOutput` 会返回：

- `questions`
  也就是实际展示给用户的问题集合
- `answers`
  一个 map，键是问题文本，值是答案字符串
- 可选 `annotations`
  记录 preview / notes 等附加信息

注意输出里 `answers` 是必填，而输入里 `answers` 是可选。  
这很合理，因为输入阶段只是“发问”，输出阶段则已经完成“收答”。

锚点：`工具定义锚点`

#### Demo

```json
{
  "questions": [
    {
      "question": "Which fix direction should we take?",
      "header": "Fix path",
      "multiSelect": false,
      "options": [
        {
          "label": "Minimal patch",
          "description": "Patch only the failing branch and keep behavior stable."
        },
        {
          "label": "Refactor flow",
          "description": "Clean up the full auth flow while fixing the bug."
        }
      ]
    }
  ]
}
```

带 preview 和 metadata 的完整 demo：

```json
{
  "questions": [
    {
      "question": "Which review mode should I use for this pass?",
      "header": "Review mode",
      "multiSelect": false,
      "options": [
        {
          "label": "Bug hunt",
          "description": "Prioritize bugs, regressions, and missing tests.",
          "preview": "Check auth redirect, token refresh edge cases, and failing test coverage."
        },
        {
          "label": "Architecture",
          "description": "Prioritize system structure and long-term design quality.",
          "preview": "Check module boundaries, ownership, layering, and hidden coupling."
        }
      ]
    }
  ],
  "metadata": {
    "source": "review-disambiguation"
  }
}
```

#### 运行时补充

本地说明还强调：

- 用户总能选 `Other`
- 推荐项应放第一项并标 `(Recommended)`
- plan mode 下不要拿它做 plan approval

锚点：`实现锚点`

#### 怎么理解这个设计

`AskUserQuestion` 不是普通文本提问，而是一个 UI-aware elicitation tool：

- 它有硬性问题数量限制
- 有硬性选项数量限制
- 支持 preview
- 支持 annotations
- 支持 metadata

这说明 Claude Code 不是把“澄清”当普通聊天，而是把它做成了正式人机交互协议。

---

## 第六组：MCP 与环境切换工具

### 16. ListMcpResources

#### 它解决什么问题

列出某个 MCP server 当前暴露了哪些资源。

#### 输入定义

- `server?`

锚点：`工具定义锚点`

#### 输出定义

数组元素包含：

- `uri`
- `name`
- 可选 `mimeType`
- 可选 `description`
- `server`

锚点：`工具定义锚点`

#### Demo

```json
{
  "server": "docs"
}
```

#### 怎么理解这个设计

这不是“调用 MCP 工具”，而是“先看资源目录”。  
Claude Code 把资源发现和资源读取拆成了两个独立动作。

---

### 17. ReadMcpResource

#### 它解决什么问题

读取一个已经知道 URI 的 MCP resource。

#### 输入定义

- `server`
- `uri`

锚点：`工具定义锚点`

#### 输出定义

`contents[]` 中每项可包含：

- `uri`
- `mimeType?`
- `text?`
- `blobSavedTo?`

锚点：`工具定义锚点`

#### Demo

```json
{
  "server": "docs",
  "uri": "doc://auth/login-flow"
}
```

#### 怎么理解这个设计

MCP resource 被当成“外部只读知识对象”，不是普通网页也不是普通本地文件。

---

### 18. Mcp

#### 它解决什么问题

调用某个 MCP server 暴露出来的动态工具。

#### 输入定义

`McpInput` 当前在 schema 上是开放对象：

- `[k: string]: unknown`

锚点：`工具定义锚点`

#### 输出定义

- `McpOutput = string`

锚点：`工具定义锚点`

#### Demo

这个工具的具体字段取决于对应 MCP server 提供的工具 schema，所以 demo 只能给抽象形态：

```json
{
  "server": "docs",
  "tool": "lookup",
  "arguments": {
    "query": "token exchange"
  }
}
```

#### 怎么理解这个设计

Claude Code 在总工具面里给 MCP 留了一个开放入口，因为真正的字段由外部 server 决定。

这也意味着：

- `工具定义` 只能约束到“这是个开放对象”
- 真正的参数契约来自外部 MCP tool schema
- 所以 `Mcp` 天然比 `FileRead`、`TodoWrite` 这类内建工具更动态

---

### 19. Config

#### 它解决什么问题

读取或修改 Claude Code 自己的设置项。

#### 输入定义

- `setting: string`
  例如 `theme`、`model`、`permissions.defaultMode`
- `value?: string | boolean | number`
  不传就是读取，传了就是写入

锚点：`工具定义锚点`

#### 输出定义

- `success`
- `operation?`
- `setting?`
- `value?`
- `previousValue?`
- `newValue?`
- `error?`

锚点：`工具定义锚点`

#### Demo

读取配置：

```json
{
  "setting": "model"
}
```

写入配置：

```json
{
  "setting": "permissions.defaultMode",
  "value": "plan"
}
```

#### 怎么理解这个设计

`Config` 把“改自身设置”也变成了工具调用。  
这说明 Claude Code 并不把 runtime configuration 当成 CLI 外面的静态东西，而是运行时可访问对象。

---

### 20. EnterWorktree

#### 它解决什么问题

把当前工作上下文切进一个隔离 worktree。

#### 输入定义

- `name?`
  每个 `/` 分段只能包含字母、数字、点、下划线、横线，总长最多 64。
  不给就自动随机生成。

锚点：`工具定义锚点`

#### 输出定义

- `worktreePath`
- `worktreeBranch?`
- `message`

锚点：`工具定义锚点`

#### Demo

```json
{
  "name": "auth-fix-worker"
}
```

#### 怎么理解这个设计

Claude Code 并不是“让 agent 自己写 `git worktree add` 命令”。  
它把进入 worktree 抽象成正式运行时动作，这样后续的 cwd、branch、隔离状态都能被 runtime 追踪。

---

### 21. ExitWorktree

#### 它解决什么问题

离开当前 worktree，并决定保留还是删除它。

#### 输入定义

- `action: "keep" | "remove"`
- `discard_changes?`
  当 `action = "remove"` 且 worktree 里仍有未提交文件或未合并提交时，必须显式给 `true`，否则工具会拒绝。

锚点：`工具定义锚点`

#### 输出定义

- `action`
- `originalCwd`
- `worktreePath`
- `worktreeBranch?`
- `tmuxSessionName?`
- `discardedFiles?`
- `discardedCommits?`
- `message`

锚点：`工具定义锚点`

#### Demo

保留 worktree：

```json
{
  "action": "keep"
}
```

删除 worktree：

```json
{
  "action": "remove",
  "discard_changes": true
}
```

#### 怎么理解这个设计

`ExitWorktree` 暴露了几个很关键的 runtime 事实：

- runtime 知道原始 cwd
- runtime 知道当前 worktree path / branch
- runtime 还能统计被丢弃的文件数和提交数

这说明 worktree 在 Claude Code 里不是“git 命令副作用”，而是被正式建模的执行环境。

---

## 一眼看懂 Claude Code 工具面的设计哲学

把上面所有工具收束一下，Claude Code 的工具面其实在做四件事：

### 1. 把感知做成结构化输入

例如：

- `FileRead`
- `Glob`
- `Grep`
- `WebSearch`
- `ReadMcpResource`

### 2. 把动作做成结构化调用

例如：

- `FileEdit`
- `FileWrite`
- `Bash`
- `NotebookEdit`
- `Mcp`

### 3. 把过程管理也做成工具

例如：

- `TodoWrite`
- `TaskOutput`
- `TaskStop`
- `ExitPlanMode`

### 4. 把协作与环境切换也做成工具

例如：

- `Agent`
- `AskUserQuestion`
- `EnterWorktree`
- `ExitWorktree`

这说明 Claude Code 真正做出来的，不是一组“辅助函数”，而是一整套动作接口系统。

---

## 这一章真正想让你学会什么

如果这一章读完，你只记住一句话，我希望是这句：

**Claude Code 的工具，不只是模型的按钮面板，而是运行时如何把真实世界结构化交给模型的接口层。**

一旦这样理解，工具定义就不再是字段清单，而会变成你构建 agent harness 时最值得借鉴的工程设计。
