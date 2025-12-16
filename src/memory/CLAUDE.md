[根目录](../../../CLAUDE.md) > [src](../../) > **memory**

# Memory System Module

## 模块职责

内存系统模块实现了用户记忆管理功能，通过 AI 驱动的摘要和智能存储，为对话提供长期上下文支持。该系统包含两个主要功能：近期对话历史跟踪和用户明确要求保存的记忆，确保 AI 能够记住重要信息并提供个性化的响应。

## 入口与启动

### 核心文件

- **UserMemoryManager.ts** - 用户记忆管理器
  - 单例模式管理所有内存操作
  - 异步处理，不阻塞主流程
  - 支持对话摘要和显式记忆存储
  - 提供记忆检索和格式化功能

- **memory-design.md** - 内存系统设计文档
  - 详细的架构设计说明
  - 实现细节和决策记录
  - 性能考虑和优化策略

- **UserMemoryManager.test.ts** - 单元测试
  - 测试记忆的创建、存储和检索
  - 验证异步操作和错误处理

## 功能特性

### 1. 近期对话历史 (Recent Conversations)

自动跟踪和分析最近的对话：

```typescript
// 添加最近对话到内存
addRecentConversation(
  messages: ChatMessage[],
  chatModel?: BaseChatModel
): void
```

特性：
- **异步处理**：在后台运行，不阻塞对话流程
- **智能摘要**：使用 AI 模型提取关键信息
- **时间戳管理**：自动记录对话时间
- **可配置**：通过设置启用/禁用

### 2. 保存记忆 (Saved Memories)

保存用户明确要求记住的信息：

```typescript
// 更新保存的记忆
async updateSavedMemory(
  query: string,
  chatModel: BaseChatModel
): Promise<{ content?: string; error?: string }>
```

特性：
- **显式触发**：只有用户明确要求时才保存
- **智能处理**：AI 提取和格式化重要信息
- **持久化存储**：保存在 Obsidian 库中的专用文件
- **即时可用**：保存后立即可用于后续对话

### 3. 记忆检索

为当前对话提供相关的历史上下文：

```typescript
// 获取格式化的记忆提示
async getUserMemoryPrompt(): Promise<string | null>
```

## 文件存储结构

### 存储位置

```
[Obsidian Vault]/
└── .obsidian/
    └── plugins/
        └── copilot/
            └── userMemory/
                ├── recentConversations.md    # 近期对话摘要
                └── savedMemories.md          # 用户保存的记忆
```

### 文件格式

**recentConversations.md**:
```markdown
# Recent Conversations Summary

## 2025-12-07 14:00:00
- 用户询问了关于 Obsidian 插件开发的问题
- 讨论了 TypeScript 类型系统的最佳实践
- 提到了 React Hooks 的使用场景

## 2025-12-07 10:30:00
- 用户分享了项目进展
- 讨论了下一阶段的开发计划
- 记录了技术决策和理由
```

**savedMemories.md**:
```markdown
# Saved Memories

## User Preferences
- 用户偏好使用深色主题
- 喜欢在早晨进行代码审查
- 重要：对 LaTeX 公式过敏，尽量避免

## Project Context
- 当前项目是 Obsidian Copilot 插件
- 使用 TypeScript + React 技术栈
- 集成了多个 LLM 提供商

## Personal Notes
- 用户的宠物猫叫 "Mittens"
- 正在学习 Rust 编程语言
```

## API 接口详解

### UserMemoryManager 类

```typescript
export class UserMemoryManager {
  constructor(app: App)

  // 近期对话管理
  addRecentConversation(messages: ChatMessage[], chatModel?: BaseChatManager): void

  // 保存记忆管理
  updateSavedMemory(query: string, chatModel: BaseChatModel): Promise<{content?: string; error?: string}>

  // 记忆检索
  getUserMemoryPrompt(): Promise<string | null>

  // 私有方法
  private async loadMemory(): Promise<void>                    // 加载现有记忆
  private async ensureMemoryFolderExists(): Promise<void>       // 确保文件夹存在
  private async updateMemory(messages: ChatMessage[], chatModel?: BaseChatModel): Promise<void>
  private async updateRecentConversationsFile(filePath: string, messages: ChatMessage[], chatModel?: BaseChatModel): Promise<string>
  private async updateSavedMemoryFile(filePath: string, query: string, chatModel: BaseChatModel): Promise<{content?: string; error?: string}>
  private getRecentConversationFilePath(): string              // 获取文件路径
  private getSavedMemoriesFilePath(): string                  // 获取文件路径
  private getTimestamp(): string                              // 获取时间戳
}
```

### 配置选项

在设置中可以控制：

```typescript
interface CopilotSettings {
  enableRecentConversations: boolean    // 启用近期对话跟踪
  enableSavedMemory: boolean           // 启用保存记忆功能
  maxRecentConversations: number       // 最大保存的对话数量
  memoryRetentionDays: number          // 记忆保留天数
}
```

## 使用流程

### 1. 初始化

```typescript
// 在插件初始化时
const memoryManager = new UserMemoryManager(app);
```

### 2. 对话后更新

```typescript
// 在消息发送后
chatManager.onMessageSent((messages) => {
  memoryManager.addRecentConversation(messages, currentChatModel);
});
```

### 3. 获取记忆上下文

```typescript
// 在创建新的对话上下文时
const memoryPrompt = await memoryManager.getUserMemoryPrompt();
if (memoryPrompt) {
  contextPrompt += memoryPrompt;
}
```

### 4. 保存用户记忆

```typescript
// 当用户说"记住这个"时
const result = await memoryManager.updateSavedMemory(userRequest, chatModel);
if (result.error) {
  showError(result.error);
}
```

## 性能优化

### 1. 异步处理

- 所有记忆操作都是异步的
- 使用 "fire and forget" 模式避免阻塞
- 错误处理不会影响主流程

### 2. 防重复机制

```typescript
private isUpdatingMemory: boolean = false;

async updateMemory(messages: ChatMessage[], chatModel?: BaseChatModel): Promise<void> {
  if (this.isUpdatingMemory) {
    return; // 防止并发更新
  }

  this.isUpdatingMemory = true;
  try {
    // 处理逻辑
  } finally {
    this.isUpdatingMemory = false;
  }
}
```

### 3. 懒加载

- 记忆文件只在需要时加载
- 缓存机制减少文件读取
- 时间戳检查避免无效更新

## 安全和隐私

### 1. 本地存储

- 所有记忆数据存储在用户本地
- 不会发送到外部服务
- 用户完全控制数据的删除和修改

### 2. 权限控制

```typescript
// 检查功能是否启用
if (!settings.enableRecentConversations) {
  logWarn("Recent history is disabled");
  return;
}
```

### 3. 数据最小化

- 只存储必要的信息
- 定期清理过期记忆
- 支持用户手动清理

## 测试与质量

### 测试策略

```typescript
describe('UserMemoryManager', () => {
  test('should create memory folder if not exists')
  test('should load existing memories on initialization')
  test('should add recent conversations asynchronously')
  test('should update saved memories with AI processing')
  test('should format memory prompt correctly')
  test('should handle errors gracefully')
  test('should respect disabled settings')
})
```

### 质量保证

- 完整的错误处理
- 详细的日志记录
- TypeScript 严格模式
- 异步操作超时保护

## 扩展性

### 1. 插件化记忆处理器

```typescript
interface MemoryProcessor {
  canProcess(content: string): boolean
  process(content: string): Promise<string>
}

// 可以添加特定的处理器
const codeSnippetProcessor: MemoryProcessor = {
  canProcess: (content) => content.includes('```'),
  process: async (content) => {
    // 处理代码片段的记忆
  }
}
```

### 2. 记忆分类和标签

```typescript
interface MemoryEntry {
  id: string
  type: 'preference' | 'context' | 'fact' | 'decision'
  tags: string[]
  content: string
  timestamp: number
  relevance: number  // AI 评估的相关性分数
}
```

## 常见问题 (FAQ)

### Q: 记忆会占用多少存储空间？
A: 记忆以纯文本格式存储，通常只有几 KB。系统会自动清理过期内容，保持存储空间在合理范围内。

### Q: AI 如何决定记住什么？
A: 对于近期对话，AI 会提取关键信息、决策和模式。对于保存的记忆，AI 会理解用户意图并结构化存储重要信息。

### Q: 可以删除特定的记忆吗？
A: 可以。记忆存储在标准 Markdown 文件中，用户可以直接编辑或删除特定条目。

### Q: 记忆会影响隐私吗？
A: 所有数据都存储在本地，完全由用户控制。插件不会上传任何记忆数据到外部服务。

### Q: 如何禁用记忆功能？
A: 在插件设置中可以独立控制近期对话和保存记忆的开关。

## 相关文件清单

```
src/memory/
├── UserMemoryManager.ts          # 核心管理器
├── UserMemoryManager.test.ts     # 单元测试
└── memory-design.md              # 设计文档

# 生成的文件
.obsidian/plugins/copilot/userMemory/
├── recentConversations.md        # 近期对话摘要
└── savedMemories.md             # 保存的记忆
```

## 变更记录 (Changelog)

### 2025-12-07 14:15:17
- ✨ 创建内存系统模块文档
- 📊 详细说明两种记忆类型
- 🔗 记录文件存储结构
- 📝 提供完整的 API 文档