# 上下文理解（CodeBase和RAG）

在opencode中，主要通过**init初始化**、**内置检索工具**、**深度代码理解**等能力来实现代码及上下文理解

## 内置SKILL命令

### /init

执行/init命令,基于grep获取代码仓的核心文件，分析当前代码库并生成一份 AGENTS.md 文件

### 内置检索tools

* glob src/tool/glob.ts 文件名匹配搜索

* grep src/tool/grep.ts 文件内容正则搜索

* read src/tool/read.ts 读取文件内容

* codesearch src/tool/codesearch.ts 基于exa mcp的外部代码搜索

### 增强型工具 [Graphify](https://github.com/safishamsi/graphify/blob/v7/docs/translations/README.zh-CN.md)

为 opencode等助手扩展SKILL能力，输入 '/graphify' . 即可为文件夹构建图谱

增强型理解处理流程：

1. 基础层：AST 提取提供确定性的结构信息（tree-sitter）
2. 增强层：LLM 语义提取增加深度理解：采用SKILL从代码注释和命名推断隐式关系等
3. 分析层：社区检测（Leiden）和图分析，将生成的关系进行社区分析并转储为图
4. 输出层：结构化图数据和可读报告

## 深度代码理解

通过图谱化，进行深度代码理解

### 云端代码索引架构设计

在插件中可以授权云端获取自己的代码仓，生成云端索引

云端索引基于个人改动分支增量索引与更新

![img.png](img.png)

* CSS Code Semantic Search（代码语义搜索库） 存储代码的向量表示，支持语义搜索
* GES Graph Engine Service（图引擎服务） 存储代码的调用关系图，支持依赖分析
* AST (抽象语法树)：理解单个文件内的代码结构。
* CPG (代码属性图)：融合多种图结构，进行更深层的代码分析。

| 插件名称     | 输入          | 输出         | 示例                                 |
|----------|-------------|------------|------------------------------------|
| AST 解析插件 | 源代码文件       | AST 语法树    | 将 def add(a,b): return a+b 解析成函数节点 |
| 代码切片插件   | AST         | 独立代码片段     | 提取单个函数、类、方法作为独立单元                  |
| 摘要提取插件   | 代码切片        | 补充自然语言描述   | "这是一个加法函数"                         |
| 向量化插件    | 代码切片 + 摘要   | 向量（浮点数数组）  | [0.123, -0.456, 0.789, ...]        |
| 图谱构建插件   | 代码切片 + 调用关系 | 图节点 + 边    | Function A -> calls -> Function B  |
| 索引入库插件   | 向量 + 图数据    | 写入 CSS/GES | 批量写入数据库                            |

### 实用开源工具

[ContextPlus](https://github.com/ForLoopCodes/contextplus)：将代码库转化为可搜索、层次化的“功能图谱”，以帮助 AI 理解代码

[Joern MCP](https://github.com/Lekssays/codebadger)：fastmcp+Joern 做代码审查和漏洞分析

### LSP插件

可通过语言服务协议获取符号与引用信息，提升代码仓理解与定位能力(代码导航、代码补全、代码重构和格式化、语义理解和诊断)

#### 常见语言服务器

| 语言                      | 推荐的语言服务器                        | 启动命令                                 |
|:------------------------|:--------------------------------|:-------------------------------------|
| Python                  | `pyright` (微软) 或 `basedpyright` | `pyright-langserver --stdio`         |
| TypeScript / JavaScript | `typescript-language-server`    | `typescript-language-server --stdio` |
| Java                    | `jdtls` (Eclipse)               | `java -jar jdtls.jar`                |
| Go                      | `gopls`                         | `gopls serve`                        |
| Rust                    | `rust-analyzer`                 | `rust-analyzer`                      |
| C / C++                 | `clangd`                        | `clangd`                             |

#### LSP客户端

* Python: pygls（实现 LSP 服务器）、python-lsp-jsonrpc（客户端）
* Node.js: vscode-languageserver-node（官方库）

### opencode-autognosis(已废弃)

* 深层代码理解：记忆块、分层推理 、模块摘要
* 智能工作记忆：任务最相关 Chunk Cards 放入ActiveSet,AI在进行思考和生成代码时，聚焦于 ActiveSet
* 性能优化 (Performance)：增量式重新索引、后台处理、内存优化
* 生产级体验 (Production Polish)：监控与指标、操作界面、工具参考集

#### 生态、社区活跃度、性能问题

* 高度依赖四个独立的底层工具：ripgrep、fd、ast-grep、universal-ctags
* 配置与调试成本高、开源组件异常根源排查困难
* 社区活跃度低、CPU、内存资源占用增加、Token 消耗激增

codebase-graph
通过符号索引、AST分析、层级推理，构建代码的静态“知识图谱”，让AI理解项目结构和依赖。

RAG能力
opencode-autognosis (ActiveSet, 分块)
opencode-enhancer-plugin (Librarian)
自建LangChain集成
动态地从代码库或文档中检索最相关的信息片段（Chunk Cards），作为上下文供AI生成答案，实现“检索增强”。
高级编排
opencode-mad 协调多个具有不同角色和权限的AI智能体，并行处理复杂任务，如全栈应用开发。
外部集成
MCP工具 (gatekpr-opencode)
GitHub Actions (/opencode指令)
通过标准协议接入外部文档系统，或在CI/CD流程中自动调用OpenCode完成任务

opencode 不做传统 RAG（向量检索），而是用工具 + Subagent模式来理解代码库：

2. Explore Agent（子代理）
   src/agent/agent.ts:131-157 定义了 explore agent：

"explore agent": 快速搜索代码库的专用代理

- 权限: 只允许 grep, glob, list, read, bash, webfetch, websearch, codesearch
- 任务: 找文件、搜代码、回答代码库问题

3. 工作流（Plan Mode）
   src/session/prompt.ts:1428-1437：

Phase 1: 并行启动最多 3 个 explore agent 搜索代码库
Phase 2: 启动 general agent 设计实现
Phase 3: 审阅
Phase 4: 写最终计划
总结
opencode 用 ripgrep 实时搜索 + LLM Agent 协作 代替传统 RAG 向量索引，更灵活但无持久化索引。



------------------------------ 
https://codehub-g.huawei.com/applicationplatform/cloudbuild/cloudbuild-openAPI/merge_requests/114

https://codehub-g.huawei.com/applicationplatform/CloudBuild2.0/CloudBuild/BuildProject/merge_requests/96

构建工具推送，需要建立流水线，先考虑搭建在公网的流水线

按照以下规则调整

1. 识别用户需要的jdk版本列表
2. 从https://api.adoptium.net/v3/assets/version/ 接口获取到对应jdk发行版本的信息
3. 按照一下格式输出json文件

```json
[
  {
    "version": "3.14.3",
    "stable": true,
    "release_url": "https: //github.com/actions/python-versions/releases/tag/3.14.3-21673711214",
    "files": [
      {
        "filename": "python-3.14.3-linux-22.04-arm64.tar.gz",
        "arch": "arm64",
        "platform": "linux",
        "platform_version": "22.04",
        "download_url": "https: //github.com/actions/python-versions/releases/download/3.14.3-21673711214/python-3.14.3-linux-22.04-arm64.tar.gz"
      },
      {
        "filename": "python-3.14.3-linux-22.04-x64.tar.gz",
        "arch": "x64",
        "platform": "linux",
        "platform_version": "22.04",
        "download_url": "https: //github.com/actions/python-versions/releases/download/3.14.3-21673711214/python-3.14.3-linux-22.04-x64.tar.gz"
      }
    ]
  }
]
```

devrepo.devcloud.br-iaas-icsl1.huaweicloud.com

# AI调用过程的可观测性

为做到可观测性，需要对AI调用的生命周期做数据埋点，埋点协议：OpenTelemetry（OTel）

  ```text
  Session ID (会话层 - 顶层)
  │  说明：代表用户的一次完整会话 
  │
  ├── Trace ID 01 (链路轨迹层1：一次完整的问答交互)
  │   │  说明：代表用户发出的第1条消息及其完整的后台处理流程
  │   │
  │   ├── Attribute: Turn Count = 1 
  │   │
  │   └── Span ID (执行单元层 - 具体的段落)
  │       │  说明：代表代码执行的具体步骤
  │       │
  │       ├── Span: 例如检索知识库 (RAG)
  │       └── Span: 例如调用 LLM 模型 (核心生成)
  │           │
  │           ├── Events: 关联到Prompt ID ## turncount  形如（注意：79082802-9287-4890-a8b2-2ac36198852b#Explore-r6jjfn#1，表示该次对话的第1轮
  │           └── Attributes: Token Stream (生成的每一个字)
  │
  └── Trace ID 02 (链路轨迹层1：一次完整的问答交互)
      │
      ├── Attribute: Turn Count = 2
      └── ...
      
  埋点层级说明
  Trace  —— 一次端到端的Agent事件【不是一个埋点实体，是一个ID】
      Span —— 一次事件周期的Agent行为，比如CodeBase调用阶段【埋点实体，JSON体】
          Event —— 事件（Event，如tool_error） 【埋点实体，JSON体】
              Attributes —— 事件属性（Attributes，对应JSON Schema中的说明，如event.name）【埋点实体，JSON属性】
      
  ```

## OpenTelemetry核心概念

### 链路

#### Session ID (会话 ID)

定位：用户上下文的容器，包括多次会话

* 定义：标识用户与LLM进行的一段连续交互的唯一标识符。
* LLM 场景意义：
    * 将原本无状态的独立请求串联起来，还原完整的对话上下文。
    * 用于分析用户留存、会话时长等业务指标。
* OTEL 标准：通常作为 Resource Attribute 或 Log Field (session.id)。

#### Trace ID (链路 ID)

定位：单次事务的骨架,用户提出的一次对话。在分布式系统中，包含一次请求在跨进程、跨服务、跨线程的完整执行路径记录。

* 定义：标识分布式系统中的一次完整请求/事务（遵循 W3C Trace Context 标准）。
* LLM 场景意义：
    * 代表用户发出的“一句话”及其引发的所有后台动作（API调用 -> 向量检索 -> 模型推理 -> 结果返回）。
    * 是进行性能分析和错误排查的核心索引。
* OTEL 标准：核心追踪字段 trace_id。

#### Span ID (跨度 ID)

定位：最小执行单元 (Atomic Operation)。Span 通过 Parent Span ID 形成调用链结构。

* 定义：标识 Trace 中的一个具体操作步骤。
* LLM 场景意义：
    * 代表具体的LLM调用，如 POST /v1/chat/completions 或 SELECT vector_db。
    * LLM Span：特指调用大模型的那次操作，挂载了 Token 消耗、延迟、模型名称等关键指标。
* OTEL 标准：核心追踪字段 span_id。

#### Event (Span Event)

* 定义：“Span 内部的高光时刻”！Span 内部的时间点记录，包含时间戳、名称和属性。
* 用途：
    * 异常捕获：记录 exception 事件及堆栈信息。
    * 流式输出：记录 Stream 模式下的 token 到达事件。
    * 推理步骤：在 ReAct/Agent 模式下，记录中间的思考（Thought）、动作（Action）和观察（Observation），避免创建过多微小 Span。

#### Turn Count (对话轮次)

* 定义：“会话深度的标尺”。量化当前请求在整个 Session 中的位置（第几轮对话）。
* 用途：
    * 上下文丢失分析：分析随着轮次增加，模型准确率的变化。
    * 成本控制：轮次越后，Context Window 越大，Token 成本越高。
* 位置：通常作为 Attribute (ai.conversation.turn_number) 打在 Trace 的 Root Span 上。

#### Prompt ID (提示词 ID)

定位：资产与版本标识 (Asset & Version Identity)

* 定义：标识所使用的 Prompt 模板版本或特定 ID。通常作为 Attribute 挂载在执行 LLM 调用的 Span 上。
* LLM 场景意义：
    * 属于领域特定（Domain Specific）概念，而非 OTEL 原生字段。用于区分“用的是哪套逻辑/指令”。
    * 支持 A/B 测试分析（例如：对比 v1.0 和 v2.0 模板的幻觉率）。
* 位置：通常作为 Attribute 挂载在执行 LLM 调用的 Span 上。

### 日志

* LogRecord：日志记录，包含时间戳、严重级别、Body（消息体）、Attributes（附加字段）。
* Log 与 Trace 的关联：通过在 LogRecord 的 Attributes 中显式写入 trace_id 和 span_id 来实现。

### Metrics

* Meter：负责创建和收集指标的组件。
* Instrument：具体的测量工具，分为不同类型：
    * Counter：只增不减的计数器（如：请求总数）。
    * UpDownCounter：可增可减的计数器（如：当前活跃连接数）。
    * Histogram：分布统计（如：请求延迟的P99）。
    * Gauge：某一时刻的快照值（如：当前CPU温度）。
* Data Point：指标的具体数值点，包含时间戳、值、Attributes。

## OpenCode 整体交互生命周期

面向“从一次用户输入到会话收尾”的完整执行路径，基于当前代码架构整理。

### 顶层生命周期树（Conversation 视角）

```text
Conversation (session_id)
├─ A. 初始化 / 连接层
│  ├─ API/SSE 建连：/event
│  │  ├─ server.connected
│  │  └─ server.heartbeat (周期)
│  ├─ 会话创建/读取
│  │  ├─ session.created
│  │  └─ session.updated
│  └─ 客户端入口（任选其一）
│     ├─ POST /session/:id/message         (同步 prompt)
│     ├─ POST /session/:id/prompt_async    (异步 prompt)
│     └─ POST /session/:id/command         (命令驱动)
│
├─ B. Turn（一次用户提交）
│  ├─ B1. 接收输入
│  │  ├─ 生成 user message + parts
│  │  ├─ 会话 touch / 权限合并
│  │  └─ 若 noReply=true：直接返回（无 assistant 执行）
│  │
│  ├─ B2. 进入运行循环 loop
│  │  ├─ session.status = busy
│  │  ├─ 装配上下文
│  │  │  ├─ 历史消息（含 compact 过滤）
│  │  │  ├─ system / instruction / agent prompt
│  │  │  ├─ model 解析与检查
│  │  │  └─ tools 解析（内置 + MCP + 插件）
│  │  └─ 创建 assistant message（本轮输出容器）
│  │
│  ├─ B3. 推理与流式输出（Processor）
│  │  ├─ start / start-step
│  │  │  ├─ step-start part（可带 snapshot）
│  │  │  └─ 后续可生成 patch part
│  │  ├─ text 流
│  │  │  ├─ text-start -> message.part.updated
│  │  │  ├─ text-delta -> message.part.delta
│  │  │  └─ text-end   -> message.part.updated
│  │  ├─ reasoning 流
│  │  │  ├─ reasoning-start -> message.part.updated
│  │  │  ├─ reasoning-delta -> message.part.delta
│  │  │  └─ reasoning-end   -> message.part.updated
│  │  ├─ tool 流
│  │  │  ├─ tool-input-start (pending)
│  │  │  ├─ tool-call        (running)
│  │  │  ├─ tool-result       -> completed
│  │  │  └─ tool-error        -> error
│  │  └─ finish-step
│  │     ├─ step-finish part（tokens/cost/reason）
│  │     ├─ assistant message 更新
│  │     └─ summary / compaction 判定
│  │
│  ├─ B4. 人机闸门（按需）
│  │  ├─ Permission
│  │  │  ├─ permission.asked
│  │  │  └─ permission.replied (once/always/reject)
│  │  └─ Question
│  │     ├─ question.asked
│  │     ├─ question.replied
│  │     └─ question.rejected
│  │
│  ├─ B5. 子任务/多代理（按需）
│  │  ├─ subtask part（携带 agent/prompt/model）
│  │  ├─ 主代理继续执行
│  │  └─ command.executed（命令触发链路）
│  │
│  ├─ B6. 异常与重试
│  │  ├─ 可重试错误
│  │  │  ├─ session.status = retry(attempt,next)
│  │  │  └─ sleep(backoff) 后继续 loop
│  │  ├─ 不可重试错误
│  │  │  ├─ session.error
│  │  │  └─ session.status = idle
│  │  └─ 中断/取消
│  │     ├─ abort
│  │     ├─ 未完成 tool part 标记 error
│  │     └─ session.status = idle
│  │
│  └─ B7. 收尾
│     ├─ 可能触发 session.compacted
│     ├─ 回传本轮 assistant message + parts
│     └─ session.status = idle
│
└─ C. Conversation 结束
   ├─ session.deleted（可选）
   └─ global.disposed（进程/实例级）
```

### 事件流视角

```text
用户输入
→ session.prompt / session.prompt_async / session.command
→ session.status: busy
→ message.updated (assistant 初始化)
→ message.part.updated / message.part.delta (text/reasoning/tool/step...)
→ permission.* / question.* (如果触发审批)
→ session.status: retry (如果触发重试)
→ session.error (如果最终失败)
→ session.compacted (如果触发压缩)
→ session.status: idle
→ 返回 assistant 最终消息
```

### 关键状态机

```text
idle
 └─(prompt/command)→ busy
busy
 ├─(retryable error)→ retry
 ├─(done)→ idle
 ├─(fatal error)→ idle
 └─(cancel/abort)→ idle
retry
 ├─(backoff 到期)→ busy
 └─(abort/fatal)→ idle
```

# opencode的checkpoint的回滚机制

## 触发回滚的时机

* 用户点击消息右下角的菜单按钮，选择 "Revert" 时调用
* 快捷键 Undo (index.tsx:517-521)
* 用户按下 messages_redo 快捷键（默认为 Ctrl+Shift+R）时调用，重做到下一个用户消息

## session回退

packages/opencode/src/session/revert.ts

* revert(): 在回退前先创建快照，保存到 session.revert.snapshot

* unrevert(): 恢复到保存的快照，撤销回退操作

## 快照恢复

packages/opencode/src/snapshot/index.ts

* 利用 Git 的 write-tree 创建快照，存储在独立的 .opencode 目录中。

* 使用 git read-tree + git checkout-index 恢复整个工作区。

* 使用 git checkout 从指定 hash 恢复单个文件。

## opencode的一些特性

### opencode的tool工具的定义和使用

工具定义的位置：packages/opencode/src/tool，通过Tool.define定义了工具的名字和行为，其中execute方法是实际工具的执行

* id：工具名字
* init：定义的初始化方法，在register.tools中会调用init初始化工具
* execute：在AgentCommand中，会调用tool.execute做实际工具执行
