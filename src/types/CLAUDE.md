[根目录](../../CLAUDE.md) > [src](../) > **types**

# Types Module

## 模块职责

类型模块定义了 Obsidian Copilot 的核心类型系统和接口，确保整个应用程序的类型安全。该模块包含了消息类型、API 响应类型、配置类型、事件类型等核心类型定义，为模块间的数据交换提供了统一的契约。

## 入口与启动

### 核心文件

- **message.ts** - 消息类型定义
  - 定义聊天消息的数据结构
  - 包含用户消息、AI 响应、系统消息等类型
  - 支持消息的元数据和上下文信息
  - 处理消息的状态和生命周期

## 类型系统架构

### 消息类型体系

```typescript
// 基础消息接口
interface BaseMessage {
  id: string                      // 唯一标识符
  timestamp: number               // 时间戳
  metadata?: MessageMetadata     // 元数据
}

// 聊天消息
interface ChatMessage extends BaseMessage {
  sender: MessageSender           // 发送者
  content: string                 // 消息内容
  displayContent?: string         // 显示内容（可能与实际内容不同）
  context?: MessageContext        // 消息上下文
  state?: MessageState           // 消息状态
}

// 消息发送者
enum MessageSender {
  USER = 'user',                  // 用户
  ASSISTANT = 'assistant',        // AI 助手
  SYSTEM = 'system'              // 系统
}

// 消息状态
enum MessageState {
  SENDING = 'sending',           // 发送中
  SENT = 'sent',                 // 已发送
  PROCESSING = 'processing',     // 处理中
  COMPLETED = 'completed',       // 已完成
  ERROR = 'error',              // 错误
  EDITED = 'edited'             // 已编辑
}

// 消息元数据
interface MessageMetadata {
  tokenCount?: number            // Token 数量
  model?: string                 // 使用的模型
  temperature?: number           // 温度参数
  processingTime?: number        // 处理时间
  cost?: number                  // 成本
  sources?: SourceInfo[]         // 引用来源
  tools?: ToolExecutionInfo[]    // 工具执行信息
  edits?: MessageEdit[]          // 编辑历史
}
```

### 上下文类型

```typescript
// 消息上下文
interface MessageContext {
  notes?: string[]               // 引用的笔记
  tags?: string[]               // 引用的标签
  folders?: string[]            // 引用的文件夹
  urls?: string[]               // 引用的 URL
  files?: FileReference[]       // 引用的文件
  selections?: TextSelection[]  // 选中文本
  timestamp?: number            // 上下文时间戳
}

// 文件引用
interface FileReference {
  path: string                  // 文件路径
  name: string                  // 文件名
  type: FileType               // 文件类型
  size?: number                 // 文件大小
  mtime?: number                // 修改时间
  excerpt?: string              // 内容摘要
}

enum FileType {
  MARKDOWN = 'markdown',
  PDF = 'pdf',
  IMAGE = 'image',
  VIDEO = 'video',
  AUDIO = 'audio',
  OTHER = 'other'
}

// 文本选择
interface TextSelection {
  file: string                  // 文件路径
  start: number                 // 开始位置
  end: number                   // 结束位置
  text: string                  // 选中的文本
  line?: number                 // 行号
}

// 来源信息
interface SourceInfo {
  type: SourceType              // 来源类型
  path: string                  // 路径
  title?: string                // 标题
  excerpt?: string              // 摘要
  score?: number                // 相关性分数
  page?: number                 // 页码（PDF）
  timestamp?: number            // 时间戳
}

enum SourceType {
  NOTE = 'note',
  WEBPAGE = 'webpage',
  PDF = 'pdf',
  YOUTUBE = 'youtube',
  DATABASE = 'database',
  CUSTOM = 'custom'
}
```

### API 类型

```typescript
// API 请求基础接口
interface BaseAPIRequest {
  id?: string                   // 请求 ID
  timestamp?: number            // 时间戳
  metadata?: Record<string, any>
}

// API 响应基础接口
interface BaseAPIResponse<T = any> {
  success: boolean              // 是否成功
  data?: T                      // 响应数据
  error?: APIError              // 错误信息
  metadata?: ResponseMetadata   // 响应元数据
}

// API 错误
interface APIError {
  code: string                  // 错误代码
  message: string               // 错误消息
  details?: any                 // 错误详情
  stack?: string                // 堆栈信息
  retryable?: boolean           // 是否可重试
  retryAfter?: number           // 重试延迟
}

// 响应元数据
interface ResponseMetadata {
  requestId?: string            // 请求 ID
  duration?: number             // 响应时间
  rateLimit?: RateLimitInfo     // 速率限制信息
  usage?: UsageInfo             // 使用量信息
}

// 速率限制信息
interface RateLimitInfo {
  limit: number                 // 限制
  remaining: number             // 剩余
  reset: number                 // 重置时间
  retryAfter?: number           // 重试延迟
}

// 使用量信息
interface UsageInfo {
  promptTokens?: number         // 提示词 tokens
  completionTokens?: number     // 完成 tokens
  totalTokens?: number          // 总 tokens
  cost?: number                 // 成本
}
```

### 配置类型

```typescript
// 插件设置接口（部分）
interface CopilotSettings {
  // API 配置
  openAIApiKey?: string
  anthropicApiKey?: string
  googleApiKey?: string
  azureOpenAIApiKey?: string
  azureOpenAIApiVersion?: string
  azureOpenAIApiInstanceName?: string

  // 模型配置
  defaultModelKey?: string
  embeddingModelKey?: string
  temperature?: number
  maxTokens?: number
  topP?: number

  // 功能开关
  enableStream?: boolean
  enableAutosaveChats?: boolean
  enableTokenCount?: boolean
  enableEncryption?: boolean

  // UI 设置
  defaultSaveFolder?: string
  userName?: string
  userBio?: string

  // 高级设置
  debugMode?: boolean
  logLevel?: LogLevel
  requestTimeout?: number
  maxRetries?: number
}

enum LogLevel {
  ERROR = 'error',
  WARN = 'warn',
  INFO = 'info',
  DEBUG = 'debug'
}

// 项目配置
interface ProjectConfig {
  id: string                    // 项目 ID
  name: string                  // 项目名称
  path: string                  // 项目路径
  description?: string          // 描述
  settings?: Partial<CopilotSettings>  // 项目特定设置
  createdAt: number             // 创建时间
  updatedAt: number             // 更新时间
  tags?: string[]               // 标签
  active?: boolean              // 是否激活
}
```

### 事件类型

```typescript
// 事件类型枚举
enum EventType {
  // 聊天事件
  MESSAGE_SEND = 'message:send',
  MESSAGE_RECEIVE = 'message:receive',
  MESSAGE_EDIT = 'message:edit',
  MESSAGE_DELETE = 'message:delete',

  // 插件事件
  PLUGIN_LOAD = 'plugin:load',
  PLUGIN_UNLOAD = 'plugin:unload',
  SETTINGS_CHANGE = 'settings:change',

  // 搜索事件
  SEARCH_START = 'search:start',
  SEARCH_COMPLETE = 'search:complete',
  INDEX_UPDATE = 'index:update',

  // 工具事件
  TOOL_EXECUTE = 'tool:execute',
  TOOL_COMPLETE = 'tool:complete',
  TOOL_ERROR = 'tool:error',

  // 缓存事件
  CACHE_HIT = 'cache:hit',
  CACHE_MISS = 'cache:miss',
  CACHE_CLEAR = 'cache:clear'
}

// 基础事件接口
interface BaseEvent {
  type: EventType               // 事件类型
  timestamp: number             // 时间戳
  source: string                // 事件源
  data?: any                    // 事件数据
}

// 具体事件类型
interface MessageEvent extends BaseEvent {
  type: EventType.MESSAGE_SEND | EventType.MESSAGE_RECEIVE | EventType.MESSAGE_EDIT | EventType.MESSAGE_DELETE
  data: {
    messageId: string
    message?: ChatMessage
    oldContent?: string
    newContent?: string
  }
}

interface SettingsEvent extends BaseEvent {
  type: EventType.SETTINGS_CHANGE
  data: {
    key: string
    oldValue: any
    newValue: any
  }
}

interface ToolEvent extends BaseEvent {
  type: EventType.TOOL_EXECUTE | EventType.TOOL_COMPLETE | EventType.TOOL_ERROR
  data: {
    toolId: string
    input?: any
    output?: any
    error?: Error
    duration?: number
  }
}
```

### 工具类型

```typescript
// 工具定义
interface Tool {
  name: string                  // 工具名称
  description: string           // 描述
  parameters: ToolParameter[]   // 参数定义
  execute: ToolExecutor         // 执行函数
  category?: ToolCategory       // 分类
  permission?: Permission[]     // 权限要求
  enabled?: boolean            // 是否启用
}

// 工具参数
interface ToolParameter {
  name: string                  // 参数名
  type: ParameterType          // 参数类型
  required: boolean            // 是否必需
  description: string          // 描述
  default?: any                // 默认值
  validation?: ValidationRule  // 验证规则
  enum?: any[]                 // 枚举值
}

enum ParameterType {
  STRING = 'string',
  NUMBER = 'number',
  BOOLEAN = 'boolean',
  ARRAY = 'array',
  OBJECT = 'object',
  FILE = 'file',
  FOLDER = 'folder'
}

// 工具执行器
type ToolExecutor = (
  input: any,
  context: ToolContext
) => Promise<ToolResult>

// 工具上下文
interface ToolContext {
  vault: Vault                 // Obsidian 库
  app: App                    // Obsidian 应用
  settings: CopilotSettings   // 插件设置
  projectId?: string           // 当前项目 ID
  sessionId: string            // 会话 ID
}

// 工具结果
interface ToolResult {
  success: boolean             // 是否成功
  output: any                  // 输出
  type?: ResultType           // 结果类型
  metadata?: any              // 元数据
  error?: Error               // 错误
  warnings?: string[]         // 警告
}

enum ResultType {
  TEXT = 'text',
  JSON = 'json',
  MARKDOWN = 'markdown',
  FILE_PATH = 'file_path',
  BINARY = 'binary'
}
```

### 搜索类型

```typescript
// 搜索查询
interface SearchQuery {
  query: string                // 查询文本
  type?: SearchType           // 搜索类型
  filters?: SearchFilter[]    // 过滤器
  options?: SearchOptions     // 搜索选项
}

enum SearchType {
  FULLTEXT = 'fulltext',      // 全文搜索
  SEMANTIC = 'semantic',      // 语义搜索
  HYBRID = 'hybrid',          // 混合搜索
  FUZZY = 'fuzzy'            // 模糊搜索
}

// 搜索过滤器
interface SearchFilter {
  field: FilterField          // 过滤字段
  operator: FilterOperator    // 操作符
  value: any                  // 值
}

enum FilterField {
  PATH = 'path',
  TAGS = 'tags',
  FOLDER = 'folder',
  MTIME = 'mtime',
  SIZE = 'size'
}

enum FilterOperator {
  EQUALS = 'eq',
  NOT_EQUALS = 'ne',
  CONTAINS = 'contains',
  STARTS_WITH = 'starts',
  ENDS_WITH = 'ends',
  GREATER_THAN = 'gt',
  LESS_THAN = 'lt'
}

// 搜索选项
interface SearchOptions {
  limit?: number              // 结果限制
  offset?: number             // 偏移量
  sortBy?: SortField          // 排序字段
  sortOrder?: SortOrder       // 排序顺序
  includeContent?: boolean    // 包含内容
  highlight?: boolean         // 高亮
}

enum SortField {
  RELEVANCE = 'relevance',
  MTIME = 'mtime',
  PATH = 'path',
  SIZE = 'size'
}

enum SortOrder {
  ASC = 'asc',
  DESC = 'desc'
}

// 搜索结果
interface SearchResult {
  document: Document          // 文档
  score: number               // 相关性分数
  matches?: SearchMatch[]     // 匹配信息
  excerpt?: string            // 摘要
  highlights?: string[]       // 高亮片段
}

// 文档
interface Document {
  id: string                  // 文档 ID
  path: string                // 路径
  title: string               // 标题
  content: string             // 内容
  metadata: DocumentMetadata  // 元数据
  embeddings?: number[]       // 嵌入向量
  chunks?: DocumentChunk[]    // 分块
}

// 文档元数据
interface DocumentMetadata {
  tags?: string[]             // 标签
  folder?: string             // 文件夹
  mtime: number               // 修改时间
  size: number                // 大小
  fileType: FileType         // 文件类型
}
```

### 记忆类型

```typescript
// 用户记忆
interface UserMemory {
  recentConversations?: ConversationSummary[]   // 近期对话摘要
  savedMemories?: SavedMemory[]                 // 保存的记忆
  preferences?: UserPreferences                 // 用户偏好
  lastUpdated: number                           // 最后更新时间
}

// 对话摘要
interface ConversationSummary {
  id: string                  // 对话 ID
  timestamp: number           // 时间戳
  topic: string               // 主题
  keyPoints: string[]         // 关键点
  decisions?: string[]        // 决策
  actionItems?: string[]      // 行动项
}

// 保存的记忆
interface SavedMemory {
  id: string                  // 记忆 ID
  type: MemoryType           // 记忆类型
  content: string            // 内容
  tags?: string[]            // 标签
  importance?: number        // 重要性 (1-5)
  createdAt: number          // 创建时间
  lastAccessed: number       // 最后访问时间
  accessCount: number        // 访问次数
}

enum MemoryType {
  FACT = 'fact',             // 事实
  PREFERENCE = 'preference', // 偏好
  DECISION = 'decision',     // 决策
  RELATIONSHIP = 'relationship', // 关系
  EVENT = 'event',          // 事件
  CUSTOM = 'custom'         // 自定义
}

// 用户偏好
interface UserPreferences {
  language?: string          // 语言
  timezone?: string          // 时区
  dateFormat?: string        // 日期格式
  theme?: ThemeType         // 主题
  notifications?: NotificationSettings
  privacy?: PrivacySettings
}

enum ThemeType {
  LIGHT = 'light',
  DARK = 'dark',
  AUTO = 'auto'
}
```

### 流类型

```typescript
// 流事件
interface StreamEvent {
  type: StreamEventType       // 事件类型
  data: any                  // 数据
  timestamp: number           // 时间戳
  id?: string                // 事件 ID
}

enum StreamEventType {
  START = 'start',           // 开始
  DATA = 'data',             // 数据
  THINKING = 'thinking',     // 思考中
  ACTION = 'action',         // 动作
  ERROR = 'error',           // 错误
  END = 'end'               // 结束
}

// 流处理器
type StreamHandler = (event: StreamEvent) => void

// 流配置
interface StreamConfig {
  enabled?: boolean          // 是否启用
  buffer?: boolean           // 是否缓冲
  timeout?: number           // 超时时间
  retries?: number           // 重试次数
}
```

## 类型守卫和验证

```typescript
// 类型守卫
export function isChatMessage(obj: any): obj is ChatMessage {
  return (
    obj &&
    typeof obj.id === 'string' &&
    Object.values(MessageSender).includes(obj.sender) &&
    typeof obj.content === 'string'
  )
}

export function isToolResult(obj: any): obj is ToolResult {
  return (
    obj &&
    typeof obj.success === 'boolean' &&
    (obj.output !== undefined || obj.error !== undefined)
  )
}

// 运行时验证
export function validateMessageContext(
  context: any
): context is MessageContext {
  const errors: string[] = []

  if (context.notes && !Array.isArray(context.notes)) {
    errors.push('notes must be an array')
  }

  if (context.tags && !Array.isArray(context.tags)) {
    errors.push('tags must be an array')
  }

  if (errors.length > 0) {
    throw new ValidationError(errors)
  }

  return true
}

// 验证错误
export class ValidationError extends Error {
  constructor(public errors: string[]) {
    super(`Validation failed: ${errors.join(', ')}`)
  }
}
```

## 使用示例

```typescript
// 创建消息
const message: ChatMessage = {
  id: generateId(),
  sender: MessageSender.USER,
  content: 'Hello, AI!',
  timestamp: Date.now(),
  context: {
    notes: ['note1.md', 'note2.md'],
    tags: ['important', 'work']
  }
}

// 处理 API 响应
async function handleAPIResponse(response: BaseAPIResponse) {
  if (response.success && response.data) {
    const data = response.data as ChatMessage
    return data
  } else if (response.error) {
    throw new Error(response.error.message)
  }
}

// 类型守卫使用
function processUnknown(obj: unknown) {
  if (isChatMessage(obj)) {
    // TypeScript 知道这是 ChatMessage
    console.log(`Message from ${obj.sender}: ${obj.content}`)
  }
}
```

## 测试与质量

### 测试策略

- 使用 TypeScript 编译器进行静态类型检查
- 编写类型守卫的单元测试
- 验证运行时类型转换的正确性
- 测试类型定义的完整性和一致性

## 常见问题 (FAQ)

### Q: 如何扩展类型定义？
A: 创建新的接口或扩展现有接口，确保保持向后兼容性。

### Q: 类型定义如何版本化？
A: 使用语义化版本控制，破坏性更改需要升级主版本号。

### Q: 如何处理可选字段？
A: 使用 `?` 标记可选字段，运行时检查是否存在。

## 相关文件清单

```
src/types/
└── message.ts                 # 消息类型定义
```

## 变更记录 (Changelog)

### 2025-12-16 16:20:00
- ✨ 创建类型模块文档
- 📚 详细说明核心类型系统和接口定义
- 🔗 记录类型守卫和验证机制
- 📝 提供使用示例和最佳实践