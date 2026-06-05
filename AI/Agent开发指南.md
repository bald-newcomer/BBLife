以OMO为例，分析Agent调度与规划

# 主agent和子agent关系

使用同步队列保证子agent的消费

## 任务编排主流程

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Sisyphus (主编排器)                              │
│  mode: primary | claude-opus-4-7 max | thinking: 32k                         │
└─────────────────────────────────────────────────────────────────────────────┘
│                    │                    │
task(category=)        call_omo_agent()      task(subagent_type=)
│                    │                    │
▼                    ▼                    ▼
┌─────────────────────────────────┬─────────────────────────────────────────────┐
│     delegate-task 工具          │         call-omo-agent 工具                  │
│  src/tools/delegate-task/       │   src/tools/call-omo-agent/                 │
│  - Category解析                 │   - 直接named agent                          │
│  - Skill加载                    │   - 无skill注入                               │
│  - Model fallback链             │   - Agent fallback链                         │
│  - 8+ 内置Category              │   - 仅explore/librarian                      │
└─────────────────────────────────┴─────────────────────────────────────────────┘
                       │                    │                    │
                       └──────────┬─────────┘                    │
                                  ▼                              ▼
                    ┌────────────────────────┐      ┌────────────────────────┐
                    │    BackgroundManager   │      │     BackgroundManager  │
                    │  (后台任务调度引擎)       │      │    (同上，共享)          │
                    │  src/features/         │      │                        │
                    │  background-agent/     │      │                        │
                    └────────────────────────┘      └────────────────────────┘
                                 │                              │
                ┌────────────────┴────────────────┐             │
                ▼                                 ▼             ▼
        ┌────────────────┐               ┌────────────────┐  ┌──────────────┐
        │ BackgroundTask │               │  SyncSession   │  │ Background   │
        │ (pending→      │               │  (同步执行链)   │  │ Task         │
        │  running→      │               │                │  │ (异步执行)    │
        │  completed)    │               └────────────────┘  └──────────────┘
        └────────────────┘


用户: task(category="deep", load_skills=["git-master"], run_in_background=false, prompt="...")

┌──────────────────────────────────────────────────────────────────────────┐
│ tools.ts: createDelegateTask().execute() (第63-194行)                    │
│                                                                          │
│  1. prepareDelegateTaskArgs() → 验证参数                                  │
│  2. resolveSkillContent() → 加载skills                                   │
│  3. buildSystemContent() → 构建system prompt                             │
│  4. resolveParentContext() → 获取父会话上下文                             │
│                                                                          │
│  ┌─ category参数 ─────────────────────────────────────────────────────┐ │
│  │ category-resolver.ts: resolveCategoryExecution()                   │ │
│  │   1. mergeCategories() 合并用户+默认配置                             │ │
│  │   2. resolveCategoryConfig() 获取category的模型配置                 │ │
│  │   3. resolveModelForDelegateTask() 应用fallback链                   │ │
│  │   4. 检测unstable agent (gemini/minimax) → 强制后台                 │ │
│  │   5. 返回 agentToUse = "sisyphus-junior"                           │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌─ subagent_type参数 ────────────────────────────────────────────────┐ │
│  │ subagent-resolver.ts: resolveSubagentExecution()                   │ │
│  │   1. sanitizeSubagentType() 验证agent名称                          │ │
│  │   2. 检查plan/coordinator agent限制                                │ │
│  │   3. 从服务器agent列表匹配                                         │ │
│  │   4. 解析模型和fallback链                                          │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
                    │
         ┌──────────┴──────────┐
         ▼                     ▼
  run_in_background=true  run_in_background=false
         │                     │
         ▼                     ▼
┌────────────────────┐  ┌────────────────────────────────────────────────────┐
│ executeBackground  │  │ executeSyncTask() (sync-task.ts, 第50-384行)        │
│ Task()             │  │                                                     │
│                    │  │  while(true) {                                      │
│ 1.manager.launch() │  │    1. sendSyncPrompt() → 发送prompt                │
│ 2.等待session创建  │  │       ↓                                            │
│ 3.注册session元数据│  │    2. pollSyncSession() → 轮询直到idle             │
│ 4.返回 bg_xxx      │  │       ↓ (失败则retrySyncPromptWithFallbacks)        │
│                    │  │    3. fetchSyncResult() → 获取结果                 │
└────────────────────┘  │       ↓                                            │
                        │    4. 有nextFallbackModel? → continue               │
                        │  }                                                  │
                        │                                                     │
                        └────────────────────────────────────────────────────┘

```

## 任务同步执行过程

```text

executeSyncTask
    │
    ├─① createSyncSession()        创建子会话
    │   ├─ 获取父会话directory
    │   ├─ client.session.create({ parentID, model, ... })
    │   └─ 返回 { ok, sessionID }
    │
    ├─② sendSyncPrompt()           发送任务prompt  
    │   ├─ buildTaskPrompt() 构建任务prompt
    │   ├─ buildSyncPromptTools() 构建工具列表
    │   ├─ setSessionTools() 注册工具到session
    │   └─ promptWithModelSuggestionRetry() 发送
    │
    ├─③ pollSyncSession()          轮询直到idle
    │   ├─ 无限循环直到:
    │   │   - 超时(30min)
    │   │   - session idle + 有有效输出
    │   │   - 检测到terminal error
    │   └─ 每秒检查一次 (POLL_INTERVAL_MS=1000)
    │
    ├─④ fetchSyncResult()          获取结果
    │   ├─ session.messages() 获取消息
    │   ├─ 提取assistant/tool消息内容
    │   └─ 返回 { ok, textContent }
    │
    └─⑤ formatResult()             格式化返回
        └─ "Task completed in ${duration}.\n\nAgent:${agentToUse}\n\n---\n\n${textContent}"

```

## 任务异步执行过程

```text
BackgroundManager.launch(LaunchInput)
        │
        ▼
┌─────────────────────────────────────────────────────┐
│ 1. reserveSubagentSpawn()  检查subagent深度限制      │
│ 2. 创建 BackgroundTask { status: "pending" }        │
│ 3. startAttempt() 创建attempt记录                    │
│ 4. addTask() 注册到内存Map                           │
│ 5. queuesByKey.get(key).push(task) 加入FIFO队列      │
│ 6. ConcurrencyManager.acquire() 等待/获取并发槽位    │
│    └─ key = "${providerID}/${modelID}"              │
│    └─ 默认5个并发/模型                               │
│ 7. processKey() fire-and-forget触发队列处理         │
└─────────────────────────────────────────────────────┘
                    │
                    ▼
        startTask() → 启动任务
        │
        ├─→ 创建子session: client.session.create({ parentID })
        ├─→ 注册subagent session: subagentSessions.add(sessionID)
        ├─→ 设置session工具限制: setSessionTools(sessionID, ...)
        ├─→ 注册bootstrap: registerDelegatedChildSessionBootstrap()
        ├─→ 发送prompt: promptWithRetryInDirectory() (fire-and-forget)
        └─→ 启动轮询: startPolling()
                    │
                    ▼
        pollRunningTasks() (每3秒)
        │
        ├─→ client.session.status() 批量获取状态
        ├─→ 检查running任务:
        │   ├─ status="retry" → tryFallbackRetry()
        │   ├─ isActiveSession → 继续等待
        │   ├─ isTerminalSession → tryCompleteTask()
        │   └─ session消失 → validateSessionHasOutput() → 完成
        │
        ▼
    双重完成信号:
    1. session.idle事件 (事件驱动)
    2. 轮询检测到terminal状态 (轮询兜底)
    
    完成条件:
    ✓ session idle
    ✓ 距离开始>5秒 (防抖)
    ✓ 有有效输出 (assistant/tool消息含内容)
    ✓ 无未完成todos

```

### ConcurrencyManager.acquire() 的实现逻辑

使用map+Promise实现的Promise 延迟映射，也可称为同步队列

1. acquire()：
    1. 未超过最大并发数，acquire成功，并发计数+1
    2. 超过最大并发数，新建Promise并返回，同时Promise会加入FIFO队列。因为 await acquire(),此时会等待
    3. 等待release()释放，acquire()完成，任务继续运行
2. 执行subAgent逻辑：startTask
3. release(): 释放队列中的Promise，激活其他线程的acquire()

## 父子session数据传递

```text

Parent Session                          Child Session
┌─────────────────────────┐           ┌─────────────────────────┐
│ sessionID                │──launch──│ parentID = 父sessionID  │
│ parentMessageId          │          │ rootSessionId           │
│ parentAgent              │          │ spawnDepth              │
│ parentModel              │          │                         │
└─────────────────────────┘           └─────────────────────────┘
         ↑                                       │
         │                                       ▼
         │                      ┌────────────────────────────────┐
         │                      │ 传递的数据:                     │
         │                      │ - prompt                       │
         │                      │ - skillContent (技能注入)       │
         │                      │ - categoryPromptAppend         │
         │                      │ - fallbackChain (模型降级链)    │
         │                      │ - model config                 │
         │                      └────────────────────────────────┘
         │                                       │
         │                      ┌────────────────────────────────┐
         └─system-reminder──────│ 结果返回:                       │
           (background_output)  │ - 任务状态通知                  │
           + ParentWakeNotifier │ - 消息内容                      │
                                │ - 光标状态恢复                  │
                                └────────────────────────────────┘


```

# 开发一个agent的核心元素

* 创建 src/agents/my-agent.ts 工厂函数
* 定义 MODE: AgentMode
* 定义 MY_AGENT_PROMPT_METADATA
* 在 src/agents/builtin-agents.ts 的 agentSources 中注册
* 在 src/agents/builtin-agents.ts 的 agentMetadata 中注册（可选）
* 在 src/config/schema/agent-names.ts 添加AgentName（如果需要）
* 在 src/shared/agent-tool-restrictions.ts 添加工具限制（可选）
* 在 src/shared/model-requirements.ts 添加fallback链（可选）

## Agent工厂

```js
// src/agents/my-agent.ts
import type {AgentConfig} from "@opencode-ai/sdk";
import type {AgentMode, AgentPromptMetadata} from "./types";

const MODE: AgentMode = "subagent";  // 或 "primary"

export const createMyAgent: AgentFactory = (model: string) => ({
    instructions: "Your agent prompt here...",
    model,
    temperature: 0.1,
    // ...AgentConfig其他字段
})
createMyAgent.mode = MODE  // 静态属性

```
```text
Agent的核心字段定义与作用

AgentConfig
├── 身份定义
│   ├── model         (使用哪个模型)
│   └── mode          (primary/subagent - 模型选择策略)
│
├── 行为控制
│   ├── instructions  (system prompt - 做什么)
│   ├── description   (UI显示)
│   └── temperature   (创造性程度)
│
├── 模型调优
│   ├── variant       (变体选择)
│   ├── maxTokens     (输出限制)
│   ├── top_p         (采样策略)
│   └── thinking      (思考预算)
│
└── 安全边界
    ├── tools         (允许的工具)
    └── permission    (权限配置)
```

## Prompt元数据

```js

export const MY_AGENT_PROMPT_METADATA: AgentPromptMetadata = {
    category: "advisor",       // exploration | specialist | advisor | utility
    cost: "EXPENSIVE",         // FREE | CHEAP | EXPENSIVE
    promptAlias: "MyAgent",    // 显示名称
    triggers: [                // Sisyphus委托表的触发条件
        {
            domain: "领域",
            trigger: "触发描述",
        },
    ],
    useWhen: ["使用场景1", "使用场景2"],
    avoidWhen: ["避免场景1", "避免场景2"],
}

```
```text
AgentPromptMetadata 的核心信息
├── 分组标识
│   ├── category      (在prompt中哪个section显示)
│   └── cost          (工具选择表的成本标记)
│
├── 委托决策
│   ├── triggers      (何时应该委托给这个agent)
│   ├── useWhen       (应该使用的场景)
│   └── avoidWhen     (应该避免的场景)
│
├── 展示控制
│   ├── promptAlias   (prompt中显示的名称)
│   └── dedicatedSection (自定义prompt段落)
│
└── 意图识别
    └── keyTrigger    (Phase 0关键词检测)
```

## 工具注册到builtin-agents
```js
// src/agents/builtin-agents.ts
import {createMyAgent} from "./my-agent";

const agentSources: Record<BuiltinAgentName, AgentSource> = {
   // ...其他agent
   "my-agent": createMyAgent,
}

// 添加元数据
const agentMetadata: Partial<Record<BuiltinAgentName, AgentPromptMetadata
>>
= {
   "my-agent": MY_AGENT_PROMPT_METADATA,
}
```

## 额外定义
```js
// src/shared/agent-tool-restrictions.ts
export const AGENT_TOOL_RESTRICTIONS: Record<BuiltinAgentName, string[]> = {
  // ...其他agent
  "my-agent": ["write", "edit", "task", "call_omo_agent"],
}

// src/shared/model-requirements.ts
export const AGENT_MODEL_REQUIREMENTS: Record<string, ModelRequirement> = {
   "my-agent": {
      fallbackChain: [
         { providers: ["openai"], model: "gpt-5.5", variant: "high" },
         { providers: ["anthropic"], model: "claude-opus-4-7", variant: "max" },
      ],
   },
}
```
