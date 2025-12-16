[根目录](../../CLAUDE.md) > [src](../) > **utils**

# Utils Module

## 模块职责

工具模块为 Obsidian Copilot 提供了各种通用的工具函数和辅助功能，包括聊天历史处理、项目管理、速率限制、工具结果处理等。这些工具函数被整个应用程序复用，确保了代码的一致性和可维护性。

## 入口与启动

### 核心工具文件

- **chatHistoryUtils.ts** - 聊天历史处理工具
  - 格式化聊天历史记录
  - 转换聊天消息格式
  - 处理消息的序列化和反序列化
  - 支持多种输出格式（Markdown、JSON 等）

- **projectUtils.ts** - 项目管理工具
  - 处理项目配置和元数据
  - 管理项目的创建、切换和删除
  - 提供项目上下文信息
  - 处理项目文件的路径和命名

- **rateLimitUtils.ts** - 速率限制工具
  - 实现 API 调用的速率限制
  - 支持多种限制策略（令牌桶、滑动窗口等）
  - 提供请求队列和延迟机制
  - 处理不同提供商的特定限制

- **toolResultUtils.ts** - 工具结果处理工具
  - 统一工具输出的格式化
  - 处理结果的摘要和预览
  - 支持结果的过滤和转换
  - 管理错误的处理和显示

- **base64.ts** - Base64 编码/解码工具
  - 提供高效的 Base64 编解码
  - 支持 Unicode 和二进制数据
  - 处理大文件的流式编解码
  - 优化性能和内存使用

## 工具函数详解

### 1. 聊天历史工具 (chatHistoryUtils)

```typescript
export interface ChatMessage {
  id: string
  role: 'user' | 'assistant' | 'system'
  content: string
  timestamp: number
  metadata?: ChatMessageMetadata
}

// 格式化聊天历史为 Markdown
export function formatChatHistoryAsMarkdown(
  messages: ChatMessage[],
  options?: MarkdownFormatOptions
): string {
  const { includeTimestamps = true, includeMetadata = false } = options || {}

  return messages.map(msg => {
    let line = `**${formatRole(msg.role)}**: `
    line += msg.content

    if (includeTimestamps) {
      line += `\n\n_时间: ${formatTimestamp(msg.timestamp)}_`
    }

    if (includeMetadata && msg.metadata) {
      line += `\n\n元数据: ${JSON.stringify(msg.metadata, null, 2)}`
    }

    return line
  }).join('\n\n---\n\n')
}

// 格式化聊天历史为 JSON
export function formatChatHistoryAsJSON(
  messages: ChatMessage[],
  options?: JSONFormatOptions
): string {
  const { prettyPrint = true, includeMetadata = true } = options || {}

  const data = {
    version: '1.0',
    exportTime: new Date().toISOString(),
    messages: messages.map(msg => ({
      id: msg.id,
      role: msg.role,
      content: msg.content,
      timestamp: msg.timestamp,
      ...(includeMetadata && msg.metadata && { metadata: msg.metadata })
    }))
  }

  return JSON.stringify(data, null, prettyPrint ? 2 : 0)
}

// 转换为 LangChain 格式
export function convertToLangChainMessages(
  messages: ChatMessage[]
): BaseMessage[] {
  return messages.map(msg => {
    switch (msg.role) {
      case 'user':
        return new HumanMessage(msg.content)
      case 'assistant':
        return new AIMessage(msg.content)
      case 'system':
        return new SystemMessage(msg.content)
      default:
        throw new Error(`Unknown role: ${msg.role}`)
    }
  })
}
```

### 2. 项目工具 (projectUtils)

```typescript
export interface ProjectConfig {
  id: string
  name: string
  path: string
  description?: string
  settings?: Partial<CopilotSettings>
  createdAt: number
  updatedAt: number
  tags?: string[]
}

// 创建新项目
export function createProject(
  name: string,
  path: string,
  options?: CreateProjectOptions
): ProjectConfig {
  const id = generateProjectId(name)
  const now = Date.now()

  return {
    id,
    name,
    path,
    description: options?.description,
    settings: options?.settings,
    createdAt: now,
    updatedAt: now,
    tags: options?.tags || []
  }
}

// 验证项目配置
export function validateProjectConfig(
  config: Partial<ProjectConfig>
): ValidationResult {
  const errors: string[] = []
  const warnings: string[] = []

  if (!config.name || config.name.trim().length === 0) {
    errors.push('项目名称不能为空')
  }

  if (config.name && config.name.length > 100) {
    errors.push('项目名称不能超过 100 个字符')
  }

  if (!config.path || !isValidPath(config.path)) {
    errors.push('项目路径无效')
  }

  if (config.path && !fs.existsSync(config.path)) {
    warnings.push('项目路径不存在，将自动创建')
  }

  return {
    valid: errors.length === 0,
    errors,
    warnings
  }
}

// 获取项目相对路径
export function getProjectRelativePath(
  projectPath: string,
  absolutePath: string
): string {
  if (!absolutePath.startsWith(projectPath)) {
    return absolutePath
  }

  return path.relative(projectPath, absolutePath)
}

// 检查文件是否属于项目
export function isFileInProject(
  project: ProjectConfig,
  filePath: string
): boolean {
  const normalizedProjectPath = path.resolve(project.path)
  const normalizedFilePath = path.resolve(filePath)

  return normalizedFilePath.startsWith(normalizedProjectPath)
}
```

### 3. 速率限制工具 (rateLimitUtils)

```typescript
export interface RateLimitConfig {
  maxRequests: number      // 最大请求数
  windowMs: number        // 时间窗口（毫秒）
  strategy?: LimitStrategy // 限制策略
}

enum LimitStrategy {
  TOKEN_BUCKET = 'token_bucket',    // 令牌桶
  SLIDING_WINDOW = 'sliding_window', // 滑动窗口
  FIXED_WINDOW = 'fixed_window',     // 固定窗口
  LEAKY_BUCKET = 'leaky_bucket'      // 漏桶
}

export class RateLimiter {
  private requests: Map<string, RequestRecord[]> = new Map()
  private config: RateLimitConfig
  private timer?: NodeJS.Timeout

  constructor(config: RateLimitConfig) {
    this.config = config
    this.setupCleanup()
  }

  // 检查是否允许请求
  async checkLimit(key: string): Promise<LimitResult> {
    const now = Date.now()
    const requests = this.requests.get(key) || []

    switch (this.config.strategy || LimitStrategy.TOKEN_BUCKET) {
      case LimitStrategy.TOKEN_BUCKET:
        return this.checkTokenBucket(key, now)
      case LimitStrategy.SLIDING_WINDOW:
        return this.checkSlidingWindow(key, now)
      default:
        return this.checkFixedWindow(key, now)
    }
  }

  // 记录请求
  recordRequest(key: string): void {
    const now = Date.now()
    const requests = this.requests.get(key) || []

    requests.push({ timestamp: now })
    this.requests.set(key, requests)
  }

  // 获取剩余请求数
  getRemainingRequests(key: string): number {
    const requests = this.requests.get(key) || []
    const now = Date.now()
    const windowStart = now - this.config.windowMs

    const validRequests = requests.filter(r => r.timestamp > windowStart)
    return Math.max(0, this.config.maxRequests - validRequests.length)
  }

  // 重置计数器
  reset(key?: string): void {
    if (key) {
      this.requests.delete(key)
    } else {
      this.requests.clear()
    }
  }

  // 令牌桶实现
  private checkTokenBucket(key: string, now: number): LimitResult {
    const bucket = this.getOrCreateBucket(key)
    const tokensToAdd = (now - bucket.lastRefill) * bucket.refillRate / 1000

    bucket.tokens = Math.min(bucket.capacity, bucket.tokens + tokensToAdd)
    bucket.lastRefill = now

    if (bucket.tokens >= 1) {
      bucket.tokens--
      return { allowed: true, tokensLeft: bucket.tokens }
    }

    const waitTime = (1 - bucket.tokens) * 1000 / bucket.refillRate
    return {
      allowed: false,
      tokensLeft: 0,
      retryAfter: Math.ceil(waitTime)
    }
  }

  // 滑动窗口实现
  private checkSlidingWindow(key: string, now: number): LimitResult {
    const requests = this.requests.get(key) || []
    const windowStart = now - this.config.windowMs

    // 清理过期请求
    const validRequests = requests.filter(r => r.timestamp > windowStart)

    if (validRequests.length < this.config.maxRequests) {
      return { allowed: true, tokensLeft: this.config.maxRequests - validRequests.length }
    }

    // 计算最早请求的剩余时间
    const oldestRequest = validRequests[0]
    const retryAfter = oldestRequest.timestamp + this.config.windowMs - now

    return {
      allowed: false,
      tokensLeft: 0,
      retryAfter: Math.ceil(retryAfter)
    }
  }
}

// 创建全局速率限制器
export function createRateLimiters(): {
  chat: RateLimiter
  embedding: RateLimiter
  fileOperation: RateLimiter
} {
  return {
    chat: new RateLimiter({
      maxRequests: 100,
      windowMs: 60 * 1000, // 1分钟
      strategy: LimitStrategy.TOKEN_BUCKET
    }),
    embedding: new RateLimiter({
      maxRequests: 1000,
      windowMs: 60 * 1000, // 1分钟
      strategy: LimitStrategy.FIXED_WINDOW
    }),
    fileOperation: new RateLimiter({
      maxRequests: 50,
      windowMs: 60 * 1000, // 1分钟
      strategy: LimitStrategy.SLIDING_WINDOW
    })
  }
}
```

### 4. 工具结果工具 (toolResultUtils)

```typescript
export interface ToolResult {
  success: boolean
  output: any
  type?: ResultType
  metadata?: ResultMetadata
  error?: ToolError
  warnings?: string[]
}

// 格式化工具结果
export function formatToolResult(
  result: ToolResult,
  format: 'text' | 'markdown' | 'json' = 'text'
): string {
  if (!result.success) {
    return `错误: ${result.error?.message || '未知错误'}`
  }

  switch (format) {
    case 'markdown':
      return formatAsMarkdown(result)
    case 'json':
      return JSON.stringify(result, null, 2)
    default:
      return formatAsText(result)
  }
}

// 创建结果摘要
export function createResultSummary(
  result: ToolResult,
  maxLength: number = 200
): string {
  if (!result.success) {
    return `执行失败: ${result.error?.message || '未知错误'}`
  }

  const output = result.output
  let summary = ''

  if (typeof output === 'string') {
    summary = output
  } else if (Array.isArray(output)) {
    summary = `返回 ${output.length} 个项目`
  } else if (typeof output === 'object' && output !== null) {
    const keys = Object.keys(output)
    summary = `返回包含 ${keys.length} 个属性的对象: ${keys.join(', ')}`
  } else {
    summary = String(output)
  }

  // 截断过长的摘要
  if (summary.length > maxLength) {
    summary = summary.substring(0, maxLength - 3) + '...'
  }

  return summary
}

// 过滤敏感信息
export function filterSensitiveData(data: any): any {
  const sensitiveKeys = [
    'password', 'token', 'key', 'secret',
    'apiKey', 'accessToken', 'privateKey'
  ]

  if (Array.isArray(data)) {
    return data.map(item => filterSensitiveData(item))
  }

  if (typeof data === 'object' && data !== null) {
    const filtered: any = {}
    for (const [key, value] of Object.entries(data)) {
      if (sensitiveKeys.some(sensitive =>
        key.toLowerCase().includes(sensitive.toLowerCase())
      )) {
        filtered[key] = '[FILTERED]'
      } else {
        filtered[key] = filterSensitiveData(value)
      }
    }
    return filtered
  }

  return data
}

// 合并多个工具结果
export function mergeToolResults(
  results: ToolResult[],
  mergeStrategy: 'concat' | 'merge' | 'override' = 'concat'
): ToolResult {
  const hasError = results.some(r => !r.success)

  if (hasError) {
    const errors = results
      .filter(r => !r.success)
      .map(r => r.error?.message)
      .filter(Boolean)

    return {
      success: false,
      output: null,
      error: {
        message: `部分工具执行失败: ${errors.join('; ')}`
      }
    }
  }

  const outputs = results.map(r => r.output)
  let mergedOutput: any

  switch (mergeStrategy) {
    case 'concat':
      mergedOutput = outputs.flat()
      break
    case 'merge':
      mergedOutput = outputs.reduce((acc, curr) => ({ ...acc, ...curr }), {})
      break
    case 'override':
      mergedOutput = outputs[outputs.length - 1]
      break
  }

  return {
    success: true,
    output: mergedOutput,
    metadata: {
      toolCount: results.length,
      mergeStrategy
    }
  }
}
```

### 5. Base64 工具 (base64)

```typescript
// 异步 Base64 编码
export async function encodeBase64(data: string | Uint8Array): Promise<string> {
  if (typeof data === 'string') {
    // 处理 Unicode 字符
    const encoder = new TextEncoder()
    data = encoder.encode(data)
  }

  // 使用浏览器或 Node.js 的原生实现
  if (typeof btoa !== 'undefined') {
    return btoa(String.fromCharCode(...data))
  } else if (typeof Buffer !== 'undefined') {
    return Buffer.from(data).toString('base64')
  } else {
    // 纯 JavaScript 实现
    return base64Encode(data)
  }
}

// 异步 Base64 解码
export async function decodeBase64(
  base64: string,
  asString: boolean = true
): Promise<string | Uint8Array> {
  let data: Uint8Array

  // 使用浏览器或 Node.js 的原生实现
  if (typeof atob !== 'undefined') {
    const binary = atob(base64)
    data = new Uint8Array(binary.length)
    for (let i = 0; i < binary.length; i++) {
      data[i] = binary.charCodeAt(i)
    }
  } else if (typeof Buffer !== 'undefined') {
    data = new Uint8Array(Buffer.from(base64, 'base64'))
  } else {
    // 纯 JavaScript 实现
    data = base64Decode(base64)
  }

  if (asString) {
    const decoder = new TextDecoder()
    return decoder.decode(data)
  }

  return data
}

// 流式 Base64 编码（用于大文件）
export async function encodeBase64Stream(
  stream: ReadableStream,
  chunkSize: number = 64 * 1024
): Promise<string> {
  const reader = stream.getReader()
  const encoder = new TextEncoder()
  let result = ''

  while (true) {
    const { done, value } = await reader.read()
    if (done) break

    const chunk = value.subarray(0, chunkSize)
    result += await encodeBase64(chunk)
  }

  return result
}

// Base64 URL 编码（URL 安全）
export function encodeBase64URL(data: string | Uint8Array): Promise<string> {
  return encodeBase64(data).then(base64 => {
    return base64
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=/g, '')
  })
}
```

## 使用示例

### 组合使用工具

```typescript
import {
  formatChatHistoryAsMarkdown,
  createProject,
  RateLimiter,
  formatToolResult,
  encodeBase64
} from '@/utils'

// 创建和配置项目
const project = createProject('我的项目', '/path/to/project', {
  description: '这是一个测试项目',
  tags: ['test', 'demo']
})

// 设置速率限制
const rateLimiter = new RateLimiter({
  maxRequests: 100,
  windowMs: 60000
})

// 在操作中使用
async function processChatHistory(messages: ChatMessage[]): Promise<void> {
  // 检查速率限制
  const limitCheck = await rateLimiter.checkLimit('chat')
  if (!limitCheck.allowed) {
    throw new Error(`Rate limit exceeded. Retry after ${limitCheck.retryAfter}ms`)
  }

  // 格式化聊天历史
  const markdown = formatChatHistoryAsMarkdown(messages, {
    includeTimestamps: true,
    includeMetadata: false
  })

  // 编码附件（如果有的话）
  if (project.attachments) {
    const encoded = await encodeBase64(project.attachments[0])
    // 处理编码后的数据...
  }

  // 记录请求
  rateLimiter.recordRequest('chat')
}
```

## 测试与质量

### 测试文件

- `toolResultUtils.test.ts` - 工具结果处理测试

### 测试覆盖

- 格式化函数的输入输出验证
- 边界条件和错误处理
- 性能基准测试
- Unicode 和特殊字符处理

## 常见问题 (FAQ)

### Q: 速率限制是否支持不同的策略？
A: 是的，支持令牌桶、滑动窗口、固定窗口和漏桶四种策略。

### Q: Base64 编码如何处理大文件？
A: 提供了流式编码功能，可以分块处理大文件，避免内存溢出。

### Q: 项目工具支持嵌套项目吗？
A: 支持，可以通过路径配置和 `isFileInProject` 函数管理嵌套结构。

## 相关文件清单

```
src/utils/
├── base64.ts                    # Base64 编解码工具
├── chatHistoryUtils.ts          # 聊天历史处理
├── projectUtils.ts              # 项目管理工具
├── rateLimitUtils.ts            # 速率限制工具
├── toolResultUtils.test.ts      # 工具结果测试
└── toolResultUtils.ts           # 工具结果处理
```

## 变更记录 (Changelog)

### 2025-12-16 16:20:00
- ✨ 创建工具模块文档
- 📚 详细说明各类工具函数的功能和用法
- 🔗 记录速率限制和结果处理机制
- 📝 提供完整的使用示例和最佳实践