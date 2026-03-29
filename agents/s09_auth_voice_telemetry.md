# s09 - Auth Voice Telemetry

## 这一章要解决什么问题

如果你只盯着 agent loop、tools、prompt，很容易忽略另一类同样重要的东西：

- 认证
- 输入通道
- 遥测

它们看起来不像“智能”本身，甚至不够酷。  
但没有它们，Claude Code 很难从“能工作的原型”长成“真正可交付的系统”。

这一章要讲的正是这件事：

**agent harness 不只有执行层，还有产品运行层。**

---

## 先把三者的角色分清

这三者虽然经常一起出现，但它们解决的问题不同。

### auth

回答的是：

- 你是谁
- 你能访问什么
- 这一轮请求代表哪个身份

### voice

回答的是：

- 输入是不是只能来自键盘
- Claude Code 是否已经有了非文本输入通道

### telemetry

回答的是：

- 这台机器现在运行得怎样
- 哪些能力被高频使用
- 哪些路径在失败

所以这一章不是在讲三个边角 feature，而是在讲：

- 身份层
- 输入层
- 可观测层

---

## auth 为什么不是外围功能

很多人一看到 auth，就容易把注意力放到：

- 登录页
- 账号状态
- token 有没有过期

但从运行时角度看，auth 决定的是更底层的问题：

- 当前请求到底走哪个后端
- 当前账户能用什么能力
- OAuth / token 怎么进入请求链路

也就是说，auth 不是产品边角，而是运行时身份层。

从本地 bundle 可以直接看到认证和 OAuth 相关端点与 scope：

- `https://api.anthropic.com`
- `claude.com/cai/oauth/authorize`
- `user:mcp_servers`

锚点：

- [`cli.js:40`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js#L40)
- [`cli.js:124`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js#L124)

这说明 Claude Code 并不是“最后再补一个登录壳”，  
而是从本地交付物层面就把身份接入作为正式运行时路径。

---

## voice 为什么说明 Claude Code 不是“纯文本终端工具”

只要本地交付物里已经带了多平台音频原生模块，就说明语音不是 UI 假动作。

`vendor/` 里能直接看到音频采集模块，例如：

- [`audio-capture.node`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/vendor/audio-capture/arm64-darwin/audio-capture.node)

而在 `cli.js` 里还能看到：

- `voiceEnabled`
- 深链注册相关字段

锚点：[`cli.js:307`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js#L307)

这说明 Claude Code 已经把输入通道扩展到了键盘之外。

更重要的是，这件事的架构含义是：

Claude Code 并不把“用户输入”限定成一段文本框里的字符串。  
它承认输入层本身也可以扩展。

---

## telemetry 为什么对 agent runtime 特别重要

一个普通脚本没有 telemetry，还能勉强活。  
一个复杂 agent runtime 没 telemetry，基本等于瞎子。

因为它必须持续回答很多只有运行后才能知道的问题：

- 哪些工具被高频调用
- 哪些动作最常失败
- token 用量怎样分布
- 多 agent 协作是否有效
- 哪些路径最耗时

本地 bundle 顶部能直接看到一批 telemetry 指标名字，例如：

- `claude_code.pull_request.count`
- `claude_code.commit.count`
- `claude_code.token.usage`

锚点：[`cli.js:8`](/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js#L8)

这说明 Claude Code 从一开始就把可观测性当成正式系统能力。

也就是说，它不是“先把系统做出来，再想办法埋点”，  
而是把“系统必须能被观察”视为运行时前提。

---

## 为什么这三者应该放在同一章里

表面上看，auth、voice、telemetry 没什么共同点。

但从 runtime 角度看，它们都属于这台机器对外可用、可运营、可持续的基础条件：

- auth 决定它有没有正式身份
- voice 决定它是不是只有单一输入通道
- telemetry 决定它是不是可观测

如果没有这三者，Claude Code 仍然可能是个很强的 demo。  
但很难是个成熟产品。

---

## 把这一章压成一张图

```text
auth
  -> 给 runtime 一个正式身份

voice
  -> 给 runtime 扩展输入通道

telemetry
  -> 给 runtime 提供可观测性
```

这三层一起把 Claude Code 从：

```text
会执行的 agent
```

推进成：

```text
可使用、可运营、可调试的 agent product
```

---

## 这一章和全书主线的关系

前面很多章节都在讲：

- 这台机器怎么思考
- 这台机器怎么行动
- 这台机器怎么维持上下文

而这一章在补另一块经常被忽略的图景：

**这台机器怎样以产品的身份存在。**

这就是为什么这一章虽然不直接讲“智能动作”，但仍然非常重要。

---

## 这一章真正想让你学会什么

一句话总结：

**agent harness 不是只有“智能执行层”，还必须有“产品运行层”。**

auth 让它有身份。  
voice 让它有更多输入通道。  
telemetry 让它有可观测性。

这三者合在一起，才让 Claude Code 从“会工作的原型”变成“能交付的系统”。
