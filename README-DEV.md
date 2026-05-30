# NocoBase AI 模块开发笔记

> 本文档记录 AI 员工模块的架构、核心流程与关键代码位置，供开发者快速上手。

---

## 一、项目整体架构

NocoBase 是一个开源无代码/低代码平台，用于构建企业管理系统（CRM、ERP、OA 等）。

### 核心设计

- **插件化架构**：几乎所有功能都以插件形式实现，核心只是一个骨架
- **Application（服务端）**：继承 Koa，组装 Database、Resourcer、ACL、PluginManager、AIManager 等子系统
- **Database**：基于 Sequelize 的 ORM，通过声明式 Collection 定义（JSON → 表）
- **Resourcer**：自动将 Collection 映射为 RESTful API（list/create/get/update/destroy）
- **ACL**：基于角色的访问控制，配合 Snippets 机制
- **双前端运行时**：v1（SchemaComponent, JSON Schema 驱动）和 v2（FlowEngine/FlowModel）；v1 可引用 v2，反之不行

### 插件注册机制

两条注册路径：

1. **预设插件（Preset）**：硬编码在应用配置中，`PluginManager.initPresetPlugins()` 启动时加载
2. **数据库注册插件**：从 `applicationPlugins` 表读取 `enabled: true` 的记录，`PluginManagerRepository.init()` 加载

统一流程：
```
add("plugin-name")
  → resolvePlugin() → parseName() → importModule()
  → new PluginClass(app, options)
  → pluginInstances Map 存储
  → instance.afterAdd()
```

加载阶段（`pm.load()`）两遍遍历：
```
第一遍: 所有插件 beforeLoad()
第二遍: 按依赖序逐个
  ├── loadCollections()  → 注册表结构
  ├── loadAI()           → 自动注册 AI 工具（扫描 lib/ai/tools/）
  └── load()             → 主业务逻辑
```

依赖排序：通过 `peerDependencies` + `Topo.Sorter` 拓扑排序。

---

## 二、AI 模块分层

### `@nocobase/ai`（核心框架层）

位置：`packages/core/ai/`

提供 AI 基础设施，不含业务逻辑：

| 模块 | 文件 | 说明 |
|------|------|------|
| AIManager | `ai-manager.ts` | 持有 DocumentManager + ToolsManager |
| ToolsManager | `tools-manager/index.ts` | 工具注册表（静态 Registry + 动态 providers） |
| DocumentManager | `document-manager/index.ts` | FlexSearch 全文索引 |
| ToolsLoader | `loader/tools.ts` | 约定式扫描目录，自动注册工具 |
| DocumentLoader | `document-loader/index.ts` | Worker 线程解析文档（PDF/Word/Excel） |

### `plugin-ai`（业务逻辑层）

位置：`packages/plugins/@nocobase/plugin-ai/`

完整的 AI 员工实现：

```
plugin.ts（入口）
  └── load()
        ├── registerLLMProviders()     ← 注册 10+ LLM 提供者
        ├── registerTools()            ← 注册文档搜索 + 工作流调用工具
        ├── defineResources()          ← 注册 API 资源
        ├── setPermissions()           ← 配置 ACL 权限
        ├── registerWorkflow()         ← 注册工作流触发器和节点
        └── registerWorkContextResolveStrategy()
```

### 扩展插件（如 `plugin-ai-gigachat`）

位置：`packages/plugins/@nocobase/plugin-ai-gigachat/`

纯 LLM Provider 扩展，`load()` 中只做一件事：
```typescript
this.aiPlugin.aiManager.registerLLMProvider('gigachat', gigaChatProviderOptions);
```

通过 `peerDependencies` 声明对 `plugin-ai` 的依赖，拓扑排序保证加载顺序。

---

## 三、代码层次与职责

```
plugin.ts                    — 插件入口，组装所有模块
  ↓ 注册
resource/*.ts                — HTTP API 层（Controller），处理请求/响应
  ↓ 调用
AIEmployee 类                — 业务逻辑层，执行对话
  ↓ 使用
middleware/*.ts              — LangGraph 中间件，拦截 Agent 运行周期
  ↓ 读写
ai-chat-conversation.ts     — 数据访问层，操作 DB
```

---

## 四、数据模型

### `aiConversations` — 对话会话

| 字段 | 类型 | 说明 |
|------|------|------|
| `sessionId` | UUID (PK) | 会话唯一 ID |
| `userId` | FK→users | 属于哪个用户 |
| `aiEmployeeUsername` | FK→aiEmployees | 和哪个 AI 员工对话 |
| `thread` | integer | LangGraph 线程版本号，每次重发/编辑消息时 fork+1 |
| `title` | string | 对话标题（取自第一条用户消息前 30 字符） |
| `options` | jsonb | 对话级配置：systemMessage、skillSettings、conversationSettings |

定义文件：`src/server/collections/ai-conversations.ts`

### `aiMessages` — 对话消息

| 字段 | 类型 | 说明 |
|------|------|------|
| `messageId` | bigInt (PK) | Snowflake 生成，天然有序 |
| `sessionId` | FK→aiConversations | 属于哪个会话 |
| `role` | string | `user` / `assistant`（AI 员工 username）/ `tool` |
| `content` | jsonb | `{ type: "text", content: "..." }` |
| `toolCalls` | jsonb | AI 回复中包含的工具调用 |
| `attachments` | jsonb | 附件（图片、文件） |
| `workContext` | jsonb | 客户端注入的工作上下文 |
| `metadata` | jsonb | model、provider、usage 等元数据 |

定义文件：`src/server/collections/ai-messages.ts`

### `aiToolMessages` — 工具调用执行记录

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigInt (PK) | Snowflake 生成 |
| `sessionId` | UUID | 会话 ID |
| `messageId` | bigInt | 关联的 AI 消息 |
| `toolCallId` | string (唯一索引) | 工具调用 ID |
| `toolName` | string | 工具名称 |
| `status` | string | 执行结果：`success` / `error` |
| `content` | jsonb | 执行结果内容 |
| `invokeStatus` | string | 生命周期：`init → interrupted → waiting → pending → done → confirmed` |
| `auto` | boolean | 是否自动执行（false 需要人工审批） |
| `userDecision` | jsonb | 用户对 interrupted 工具的决策（approve/edit/reject） |

定义文件：`src/server/collections/ai-tool-messages.ts`

### `lcCheckpoints` / `lcCheckpointBlobs` / `lcCheckpointWrites` — LangGraph 状态

存储 Agent 运行时内部状态（非对话历史），用于：
- human-in-the-loop（工具被 interrupt 后恢复执行）
- 中断恢复

定义文件：`src/server/collections/lc-checkpoints.ts` 等

---

## 五、对话核心流程

### 5.1 API 入口

文件：`src/server/resource/aiConversations.ts`

主要 action：

| Action | 行号 | 说明 |
|--------|------|------|
| `sendMessages` | 324 | 发送消息（主入口） |
| `getMessages` | 170 | 分页加载历史消息（每页 10 条，游标分页） |
| `resumeToolCall` | 662 | 用户审批工具后恢复 Agent 执行 |
| `updateUserDecision` | 565 | 用户对 interrupted 工具做出决策 |
| `resendMessages` | 477 | 重新生成回复 |
| `abort` | 454 | 中止对话 |

### 5.2 sendMessages 流程

```
POST aiConversations:sendMessages
  ├── 验证 userId + sessionId
  ├── 首次对话时自动设标题（前 30 字符）
  ├── 查找 AI 员工（验证角色权限）
  ├── new AIEmployee(ctx, employee, sessionId, ...)
  ├── 如果有未完成的 toolCall → cancelToolCall()
  └── aiEmployee.stream({ userMessages, messageId })
```

### 5.3 AIEmployee.stream() 主流程

文件：`src/server/ai-employees/ai-employee.ts`

```
stream()                              ← 第 164 行
  └── buildChatContext()              ← 第 133 行
        ├── getLLMService()            — 获取 LLM Provider 实例
        ├── initSession()              ← 第 95 行
        │     ├── getAIEmployeeTools() — 收集工具（GENERAL + 员工绑定的 CUSTOM）
        │     ├── forkCurrentThread()  — fork LangGraph thread
        │     └── listMessages()       — 从 DB 加载历史消息（≤50 条）
        │
        └── getChatContext()
              ├── getSystemPrompt()    — 组装完整 system prompt
              └── formatMessages()     — 将 DB 消息转换为 LLM 格式

  └── prepareChatStream()             ← 第 281 行
        ├── createAgent()              — 创建 LangGraph Agent
        └── agent.stream(input, config)

  └── processChatStream()             ← 第 317 行
        — 遍历 stream 事件
        — 通过 SSE 推送到前端
        — 处理 content / reasoning / tool_calls / interrupt 等
```

### 5.4 消息格式化

`formatMessages()`（1013 行）将 DB 中的消息转换为 LLM 可理解的格式：

- **user 消息**：包裹在 `<user_query>` 标签中，WorkContext 包裹在 `<work_context>` 标签中
- **tool 消息**：附带 `tool_call_id`
- **assistant 消息**：附带 `tool_calls` 和 `additional_kwargs`
- 内容超过 50000 字符会被截断

---

## 六、对话中间件 — 消息持久化

文件：`src/server/ai-employees/middleware/conversation.ts`

LangGraph 中间件，拦截 Agent 运行的三个阶段：

### `beforeAgent`（103 行）— 保存用户消息

```
1. 如果是编辑消息（messageId 存在）→ 删除该 messageId 之后的所有消息
2. 将新的用户消息写入 aiMessages 表
3. 更新 conversation 的 thread 版本号
```

### `beforeModel`（122 行）— 保存工具执行结果

```
1. 收集本轮新的 tool 结果消息
2. 将 tool 消息写入 aiMessages 表
3. 更新 aiToolMessages 的 invokeStatus 为 'confirmed'
4. 通过 SSE 发送 beforeSendToolMessage 事件
```

### `afterModel`（145 行）— 保存 AI 回复

```
1. 取出 LangGraph state 中最后一条 AI 消息
2. 将 AI 消息写入 aiMessages 表（包含 content、toolCalls、metadata）
3. 如果有 toolCalls → initToolCall()：在 aiToolMessages 表创建记录
4. 通过 SSE 发送 initToolCalls 事件
5. 更新 lastMessageIndex
```

关键设计：`lastMessageIndex` 跟踪已处理的消息索引，确保中间件只处理增量消息。

---

## 七、工具调用与 Human-in-the-Loop

### 工具中间件

文件：`src/server/ai-employees/middleware/tools.ts`

两个中间件：

1. **`toolInteractionMiddleware`**：基于 LangGraph 的 `humanInTheLoopMiddleware`，根据工具权限配置决定哪些工具需要 interrupt
2. **`toolCallStatusMiddleware`**：包装工具执行，追踪状态（pending → done），处理错误

### 工具调用生命周期

```
init          — afterModel 保存 AI 回复时创建
  ↓
interrupted   — humanInTheLoop 中断（需要用户审批）
  ↓
waiting       — 用户做出决策（approve/edit/reject）
  ↓
pending       — 开始执行
  ↓
done          — 执行完成
  ↓
confirmed     — 结果已发送给 LLM
```

### 权限判断

```typescript
// ai-employee.ts 第 1000 行
isAutoCall(tools: ToolsEntry): boolean {
  // defaultPermission === 'ALLOW' → 自动执行
  // defaultPermission === 'ASK'   → 需要用户审批
  // CUSTOM 类型的工具可以在 AI 员工的 skillSettings 中覆盖
}
```

---

## 八、对话历史与"记忆"

### 历史消息加载 — 50 条窗口

文件：`src/server/manager/ai-chat-conversation.ts`，第 91-106 行

```typescript
async listMessages(query: AIMessageQuery): Promise<AIMessage[]> {
    const messages = await this.aiConversationMessagesRepo.find({
      sort: ['-messageId'],   // 倒序取最新的
      limit: 50,              // 最多 50 条
      filter,
    });
    return messages.reverse(); // 反转回正序
}
```

每次发送新消息时，系统最多从 DB 加载最近 50 条消息作为上下文发给 LLM。超过 50 条的早期消息被截断。

### 跨对话记忆

**没有跨对话记忆。** 每次创建新对话是一个全新的 sessionId，历史消息从零开始。

用户唯一的"持久化个性设置"是 **个人提示词**（`usersAiEmployees` 表的 `prompt` 字段），通过 `aiEmployees:updateUserPrompt` API 设置，注入到 system prompt 的 `<personal>` 标签中。这不是自动记忆，是用户手动配置的。

### Checkpoint 清理

文件：`src/server/ai-employees/checkpoints/cleaner.ts`

- 每天凌晨 2 点运行定时任务
- 清理 48 小时无活动且没有 pending tool call 的对话的 checkpoint
- 清理后将 `conversation.thread` 重置为 0（legacy 模式）

---

## 九、System Prompt 结构

文件：`src/server/ai-employees/prompts.ts`

```
你是 **{nickname}**，NocoBase 的 AI 员工...

<instructions>
  <global>         — 安全规则、数据完整性、SQL 规范、沟通标准、工具集成
  <ai_employee>    — 该 AI 员工的角色描述（about 字段）
  <personal>       — 用户个人提示词（可选）
</instructions>

<task>
  <background>     — 对话级 systemMessage + 数据源上下文 + WorkContext 背景
  <context>        — 具体情境（可选）
</task>

<environment>
  <main_database>  — 数据库类型（影响 SQL 语法）
  <locale>         — 语言
</environment>

<knowledgeBase>    — 知识库 RAG 检索结果（企业版）
```

`AIEmployee.getSystemPrompt()`（569 行）的组装逻辑：
1. 从 `usersAiEmployees` 表查用户个人提示词
2. 解析 `about` 字段中的变量
3. 对话级 `systemMessage` → background
4. `workContextHandler.background()` → 注入 WorkContext 背景信息
5. 知识库检索 → 格式化后注入

---

## 十、SSE 流式协议

文件：`ai-employee.ts` 第 1275 行 `ChatStreamProtocol` 类

事件类型：

| 事件 | 说明 |
|------|------|
| `stream_start` | 流开始 |
| `stream_end` | 流结束 |
| `content` | 文本内容块 |
| `reasoning` | 推理内容（深度思考） |
| `tool_calls` | 工具调用列表（含 messageId、toolCallId） |
| `tool_call_chunks` | 工具调用流式块 |
| `tool_call_status` | 工具状态变更（pending/done/interrupted/confirmed） |
| `web_search` | 网页搜索动作 |
| `new_message` | 新消息开始（工具结果确认后） |

---

## 十一、重点文件索引

按阅读优先级排列：

| 优先级 | 文件路径 | 说明 |
|--------|----------|------|
| 1 | `src/server/ai-employees/ai-employee.ts` | 对话核心逻辑 |
| 2 | `src/server/ai-employees/middleware/conversation.ts` | 消息持久化 |
| 3 | `src/server/manager/ai-chat-conversation.ts` | 50 条消息窗口、消息 CRUD |
| 4 | `src/server/resource/aiConversations.ts` | API 入口 |
| 5 | `src/server/ai-employees/prompts.ts` | System Prompt 结构 |
| 6 | `src/server/ai-employees/middleware/tools.ts` | Human-in-the-loop、工具执行追踪 |
| 7 | `src/server/ai-employees/checkpoints/saver.ts` | LangGraph 状态持久化 |
| 8 | `src/server/ai-employees/checkpoints/cleaner.ts` | Checkpoint 清理 |
| 9 | `src/server/plugin.ts` | 插件入口，模块组装 |
| 10 | `src/server/manager/ai-manager.ts` | LLM Provider 注册与管理 |
| 11 | `src/server/tools/workflow-caller.ts` | AI 工具 → 工作流桥接 |
| 12 | `src/server/manager/work-context-handler.ts` | WorkContext 策略模式 |

> 以上路径相对于 `packages/plugins/@nocobase/plugin-ai/`
