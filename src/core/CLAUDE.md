[根目录](../../CLAUDE.md) > [src](../) > **core**

# Core Architecture Module

## 模块职责

核心架构模块是 Obsidian Copilot 的中央业务逻辑层，负责消息管理、聊天协调、上下文处理和持久化存储。该模块实现了单一真实来源（Single Source of Truth）模式，确保数据一致性和完整性。

## 入口与启动

### 核心组件

- **ChatManager.ts** - 中央业务逻辑协调器
  - 协调 MessageRepository、ContextManager 和 LLM 操作
  - 处理所有消息 CRUD 操作（创建、读取、更新、删除）
  - 管理项目聊天隔离，每个项目维护独立的消息历史
  - 与链内存同步，维护对话上下文
  - 集成 ChatPersistenceManager 处理聊天保存/加载

- **MessageRepository.ts** - 消息的唯一真实来源
  - 存储每条消息的 `displayText`（用户看到的内容）和 `processedText`（LLM 处理的内容）
  - 提供计算视图：`getDisplayMessages()` 给 UI，`getLLMMessages()` 给 AI
  - 支持消息编辑、截断、批量加载等操作
  - 存储上下文信封（context envelope）用于分层上下文系统

- **ContextManager.ts** - 上下文处理引擎
  - 处理消息上下文（笔记、URL、选中文本）
  - 当消息被编辑时重新处理上下文，确保新鲜度
  - 支持 L1-L5 分层上下文系统
  - 生成 PromptContextEnvelope 用于结构化上下文表示

- **ChatPersistenceManager.ts** - 聊天持久化管理
  - 将聊天历史保存为 Markdown 文件
  - 支持项目感知的文件命名（项目 ID 前缀）
  - 解析和格式化聊天内容用于存储
  - 与 ChatManager 无缝集成

## 对外接口

### ChatManager 主要方法

```typescript
// 消息操作
async sendMessage(message: string, context?: MessageContext): Promise<void>
editMessage(messageId: string, newMessage: string): Promise<void>
regenerateMessage(messageId: string): Promise<void>
deleteMessage(messageId: string): Promise<void>

// 上下文管理
async reprocessMessageContext(messageId: string): Promise<void>

// 项目隔离
private getCurrentMessageRepo(): MessageRepository  // 自动检测项目切换
```

### MessageRepository 主要方法

```typescript
// 基础操作
addMessage(displayText: string, processedText: string, sender: string): string
editMessage(id: string, newDisplayText: string): boolean
updateProcessedText(id: string, processedText: string): boolean

// 视图获取
getDisplayMessages(): ChatMessage[]    // UI 显示用
getLLMMessages(): ChatMessage[]       // LLM 处理用

// 批量操作
truncateAfterMessageId(messageId: string): void
loadMessages(messages: ChatMessage[]): void
```

## 关键依赖与配置

### 依赖模块

- `src/LLMProviders/chainManager` - LLM 链管理和模型调用
- `src/tools/FileParserManager` - 文件解析和内容提取
- `src/context/` - 分层上下文系统
- `src/memory/UserMemoryManager` - 用户记忆管理
- `obsidian` - Obsidian API 集成

### 配置项

- 项目隔离通过 `ProjectManager.getCurrentProjectId()` 自动检测
- 上下文处理规则通过 `settings.model` 配置
- 持久化路径通过 `settings.defaultSaveFolder` 配置

## 数据模型

### StoredMessage

```typescript
interface StoredMessage {
  id: string
  displayText: string      // 用户在 UI 中看到的内容
  processedText: string    // LLM 处理的内容（可能包含上下文）
  sender: string           // "user" | "assistant" | "system"
  timestamp: number
  context?: MessageContext  // 原始上下文信息
  content?: MessageContent[] // 结构化内容（如文件引用）
  contextEnvelope?: PromptContextEnvelope // 分层上下文表示
}
```

### PromptContextEnvelope

用于 L1-L5 分层上下文系统的结构化表示：

```typescript
interface PromptContextEnvelope {
  l1: ChatMessage[]        // 核心上下文
  l2?: ChatMessage[]       // 自动提升的相关内容
  l3?: ChatMessage[]       // 手动添加的上下文
  l4?: ChatMessage[]       // 工具使用结果
  l5?: ChatMessage[]       // 系统指令
}
```

## 测试与质量

### 测试文件

- `ChatManager.test.ts` - 测试业务逻辑协调
- `MessageRepository.test.ts` - 测试消息存储和视图
- `ChatPersistenceManager.test.ts` - 测试持久化功能
- `MessageLifecycle.test.ts` - 测试消息生命周期
- `MessageLifecycle.xmltags.test.ts` - 测试 XML 标签处理

### 质量保证

- 每个消息都有唯一的 ID
- 所有操作都有适当的错误处理
- 上下文重新处理确保数据一致性
- 项目隔离防止数据污染

## 常见问题 (FAQ)

### Q: 如何实现项目聊天隔离？
A: ChatManager 维护一个 `projectMessageRepos` Map，键为项目 ID，值为对应的 MessageRepository。`getCurrentMessageRepo()` 方法自动检测当前项目并返回对应的仓库。

### Q: 消息编辑时如何保证上下文更新？
A: ChatManager 在编辑消息后自动调用 `ContextManager.reprocessMessageContext()`，确保上下文始终是最新的。

### Q: 如何处理大量消息的性能问题？
A: MessageRepository 使用计算视图模式，只在需要时生成消息数组。建议定期调用 `truncateAfterMessageId()` 清理旧消息。

### Q: 上下文信封（context envelope）的作用是什么？
A: 它保存了 L1-L5 分层上下文的结构化表示，让 chain runners 在执行时能够动态构建完整的提示词结构。

## 相关文件清单

```
src/core/
├── ChatManager.ts              # 中央业务逻辑协调器
├── MessageRepository.ts        # 消息存储和计算视图
├── ContextManager.ts           # 上下文处理引擎
├── ChatPersistenceManager.ts   # 聊天持久化管理
├── index.ts                    # 模块导出
└── __tests__/
    ├── ChatManager.test.ts
    ├── MessageRepository.test.ts
    ├── ChatPersistenceManager.test.ts
    ├── MessageLifecycle.test.ts
    └── MessageLifecycle.xmltags.test.ts
```

## 变更记录 (Changelog)

### 2025-12-07 14:10:46
- ✨ 创建核心架构模块文档
- 📊 详细说明 ChatManager、MessageRepository 等核心组件
- 🔗 添加了与相关模块的链接
- 📝 记录了关键设计决策和实现细节