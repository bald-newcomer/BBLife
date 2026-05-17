
# 如何设计的一个AI Agent的代码助手
前后端分离

## 特性
### Server-Sent Events (SSE)
允许服务器主动向浏览器（客户端）推送数据的单向通信

SSE申明协议头：Content-Type', 'text/event-stream'

服务端通过不断的res.write发送信息给客户端，客户端通过res.on监听消息发送时间完成，并呈现。

### Hono
Hono 是为现代云端环境设计的“超轻量 Express”。快速启动、跨平台。

## 项目结构
核心入口：src/index.ts，在项目启动时，会加载并注册所有类型的command，服务启动原则上就是执行命令。

### packages/opencode/src/server/server.ts
该文件是OpenCode 的 HTTP API 总入口和服务容器

* Default
一个懒加载的默认 app 实例。server.ts (line 57)
* createApp(opts)
负责构建整棵 HTTP 路由树。还有上下文的创建
* listen(opts)
负责把 app 变成真正监听端口的 Bun server。server.ts (line 625)
  * 先记录 url
  * 用 createApp(opts) 得到 app
  * 把 app.fetch 交给 Bun.serve(...)
  * 包装 server.stop()，在停止时顺便取消 mDNS


## 项目启动
opencode是前后端分离项目，服务端处理AI对话、工具执行，客户端处理页面展示
### 启动服务端
bun run dev -- --print-logs --log-level DEBUG serve
### 启动客户端
bun run dev:web
### 执行测试指令参考
bun run dev --cwd packages/opencode --conditions=browser src/index.ts run 列出当前目录

## 常用API
### prompts.text
在命令行里弹一个文本输入框，等用户输入字符串。

### Instance.state
Instance.state 是一个“按当前项目实例隔离的状态工厂”。

以 Instance.directory 作为 key，给每个项目实例创建一份独立 state，后续在同一实例里复用这份 state。通常用来存储：
* session 运行中的内存状态
* callbacks
* client 连接
* 缓存对象
* 每个项目独立的运行时资源

### fn
创建一个带有运行时输入校验和类型安全保障的函数。

当调用返回的函数时，它先拿 schema 去校验传入的参数。校验通过才执行 cb，否则抛出格式化的错误。

## 核心能力
### ACP
opencode可以作启动一个 ACP 子进程，通过 stdio 与 IDE 通信，作为IDE的插件运行。

ACP是一个协议适配器：它让外部 ACP 客户端通过标准协议来驱动 OpenCode，而不是直接调 OpenCode 私有 API。

### MCP
注册MCP服务，MCP服务会保存在config文件中

在一次 prompt 执行过程中，SessionPrompt 会调用 MCP.tools()，把 MCP 提供的工具并入模型可用工具列表：prompt.ts (line 927)

### TuiThread
TuiThreadCommand 负责执行 opencode 命令（即不带任何子命令时）来启动终端用户界面（TUI）

启动参数指令：
* --continue 或 -c：直接继续上一次的对话会话。
* --session 或 -s：恢复一个指定的历史会话。
* --fork：基于一个已有的会话创建一个新的分支会话。
* --model 或 -m：指定启动后要使用的 AI 模型。

### attach
将本地的终端用户界面（TUI）连接到一个已经在运行中的 OpenCode 后端服务器

基于前后端分离，可以做到跨设备远程开发、多窗口会话管理、团队资源共享

示例：opencode serve --hostname 0.0.0.0 --port 4096

### Run
RunCommand 是 OpenCode 的非交互式命令执行引擎。它的设计目标是让 AI 代理能够直接在命令行中执行一次性任务，无需进入聊天界面，适合脚本编写、CI/CD 集成和自动化工作流。

#### 参数
--model <model> / -m	指定 AI 模型	opencode run --model openai/gpt-4 "分析架构"

--agent <name>	指定使用的代理	opencode run --agent build "编译项目"

--command <name>	执行预定义的自定义命令	opencode run --command review "审查代码"

--share	自动分享会话并生成链接	opencode run --share "生成报告"

--timeout <秒>	设置最大执行时间	opencode run --timeout 300 "深度分析"

--verbose	输出详细执行日志	opencode run --verbose "调试任务"

-f <file>	将文件内容附加到指令	opencode run -f ./error.log "分析这个错误"

--quiet / -q	静默模式（无动画，纯输出）	opencode run -q "快速回答"

--output-format json	输出 JSON 格式，便于解析	opencode run "分析" --output-format json

#### 使用场景示例
##### 场景1：多步骤代码审查与修复
opencode run "审查当前代码变更"           # 第一步：审查
opencode run --continue "根据审查意见修复" # 第二步：延续会话进行修复

##### 场景2：使用特定模型处理不同任务
opencode run --model openai/gpt-3.5-turbo "快速解释这个函数"
opencode run --model anthropic/claude-3 "对这个复杂模块进行深度分析"

##### 场景3：自动化脚本集成（JSON 输出）
result=$(opencode run "分析测试覆盖率" --output-format json --quiet)
echo $result | jq '.response'

##### 场景4：团队协作与分享
opencode run --share "生成项目架构文档"  # 自动生成分享链接
总的来说，opencode run 的参数系统让你能像使用瑞士军刀一样，灵活地组合出适合各种自动化场景的 AI 任务执行方式。

#### execute() (line 413)

### GenerateCommand
导出 API 描述文件”的内部工具，通常用于：生成 SDK、生成文档、给 OpenAPI 补 SDK 示例代码

### ProvidersCommand
符合管理和配置AI模型商

### AgentCommand
管理agents

### Agent权限设计
permission: PermissionNext.merge(
          defaults,
          PermissionNext.fromConfig({
            "*": "deny",
          }),
          user,
        ),

等价于：

先加载 defaults
项目默认权限
再加一条总规则 * = deny
所有权限默认拒绝
也就是这个 agent 默认什么工具都不能用
最后再叠加 user
用户配置里如果显式允许某些权限，会覆盖前面的 deny
对 compaction 这种 agent 来说，含义就是：

默认它是“只做总结/压缩”的
不应该随便调用工具、读写文件、跑命令
除非用户配置里特地给它开权限

## 待定
内置agent：https://blog.csdn.net/qq_54406969/article/details/157504680
### UpgradeCommand
###  UninstallCommand
### ServeCommand
### WebCommand
### ModelsCommand
### StatsCommand
### ExportCommand
### ImportCommand
### GithubCommand
### PrCommand
### SessionCommand
### DbCommand

## 上下文管理和token控制
管理和优化 Token 使用，核心在于两点(开源节流)：一是从源头减少进入上下文的内容，二是让 AI 只记住最重要的信息。

### 内置自动压缩（compaction）
位置：agent.ts (line 164)

不是等 token 爆了才处理，而是先计数、再裁减旧 tool 输出、最后用 compaction agent 生成摘要，用“分层压缩”维持上下文可持续运行。

#### Agent压缩promot

```
You are a helpful AI assistant tasked with summarizing conversations.

When asked to summarize, provide a detailed but concise summary of the conversation.
Focus on information that would be helpful for continuing the conversation, including:
- What was done
- What is currently being worked on
- Which files are being modified
- What needs to be done next
- Key user requests, constraints, or preferences that should persist
- Important technical decisions and why they were made

Your summary should be comprehensive enough to provide context but concise enough to be quickly understood.

Do not respond to any questions in the conversation, only output the summary.

```

#### 压缩流程
每条 assistant message 都记录 tokens.input/output/reasoning/cache，后续流程据此判断是否接近模型上下文上限

先做轻量裁剪 prune，优先裁掉“旧的 tool 输出”，不删消息，只给 tool part 打 state.time.compacted

再做真正压缩 process，当 isOverflow() 判断快超限时触发
用 compaction_agent，把旧对话总结成一条可继续工作的摘要，后续不必带完整历史，只带 summary + 最近上下文

## Agent执行流程设计
Agent执行流程通常采用感知-规划-行动-反馈的循环架构
* 目标解析与任务分解
* 上下文感知与记忆检索
* 规划与推理（ReAct、CoT、Tree-of-Thought）
* 工具选择与行动执行
* 结果评估与反馈

### opencode中的流程设计
#### 入口层（SessionPrompt.prompt()）
用户目标解析和任务分解

CLI/TUI/Web 最终都会落到 session 接口
普通输入：sdk.session.prompt(...)
命令输入：sdk.session.command(...)
服务端入口：session.ts
#### prompt 组装层
核心在 prompt.ts，这里会整理：
* 当前 user message
* 历史消息
* system prompt
* agent 配置
* tools / MCP tools
* session 权限
#### agent 选择层
agent 定义在 agent.ts，有 build / plan / general / explore / compaction / summary / title 等。每个 agent 带自己的：
* prompt
* model
* permission
* mode
#### 执行层（SessionPrompt.loop()）
* 维护运行状态
* 防并发
* 处理中断
* 执行主 agent
* 调子任务
* 调 compaction
* 处理结构化输出
* 兼容用户中途追加消息
* 收尾并返回最终结果

SessionProcessor实际调用模型，位置在 processor.ts，它负责：
* 调 provider 模型
* 处理流式输出
* 处理 tool call
* 更新 message / parts / token / cost
#### 工具调用
模型如果决定调工具，就走 tool 执行流程，执行前会过 PermissionNext，工具结果再写回消息流，继续喂给模型。形成“模型 -> 工具 -> 模型”的 loop。
#### 压缩与续跑
如果上下文快超限，会进入 compaction.ts，先 prune，不够再用 compaction agent 做总结，然后继续后续执行。

## MCP、SKILL 能力模块化
MCP和SKILL都是对Agent能力的补充

MCP 管理的是"工具"，那么 Skill 管理的则是"如何使用工具的标准化流程"。Skill 本身就是一个高度模块化的文件夹

一个 Skill 就是一个文件夹，其核心是 SKILL.md 文件，还可以包含 scripts/（脚本）、references/（参考文档）等资源

SKILL.md 开头的 YAML 格式 name 和 description 是 Agent 判断何时调用该 Skill 的唯一依据，实现了"声明式"的模块注册

### MCP
配置来源：config.mcp

在prompt.ts (line 857)的resolveTools()中将MCP工具注入到工具表中

MCP tool 和内置tool很像，只是背后为外部服务

### SKILLS
配置来源：config.skills.paths、config.skills.urls

SystemPrompt.skills(agent)：会将skill注入到system skill中

tool/skill.ts：实际的按需加载，加载目录中定义的所有skill文件，主要是为了让模型了解到skill的存在

## AI调用过程的稳定性

### 自动故障转移和重试

### 上下文管理与记忆优化

### 状态持久化

## AI调用过程的可观测性

### 专用追踪和可视化
@braintrust/trace-opencode

### 本地化的指标与日志
使用统计：opencode-assistant

健康状态：opencode-rate-limit

开启debug日志：@tarquinen/opencode-dcp

## Promot Engeering 在IDE场景中的设计
## tool call 和 function call
## codebase理解与RAG
