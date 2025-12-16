[根目录](../../CLAUDE.md) > [src](../) > **tools**

# Tools Module

## 模块职责

工具模块为 AI 模型提供了丰富的工具集，使其能够与 Obsidian 环境进行深度交互。该模块实现了各种工具，包括文件操作、搜索查询、笔记管理、时间处理等，让 AI 能够执行具体任务、访问用户数据并提供实际的帮助。

## 入口与启动

### 核心组件

- **ToolRegistry.ts** - 工具注册表
  - 管理所有可用工具的注册和发现
  - 提供工具查询和过滤功能
  - 处理工具的启用/禁用状态
  - 维护工具的元数据和配置

- **toolManager.ts** - 工具管理器
  - 协调工具的执行流程
  - 处理工具调用和结果返回
  - 管理工具权限和安全控制
  - 提供工具执行的上下文

- **ToolResultFormatter.ts** - 工具结果格式化器
  - 统一工具输出的格式
  - 支持多种输出类型（文本、JSON、Markdown 等）
  - 处理错误和异常情况的显示
  - 提供结果的摘要和预览

### 核心工具

#### 1. 文件操作工具

- **FileParserManager.ts** - 文件解析管理器
  - 支持多种文件格式的解析（Markdown、PDF、图片等）
  - 提取文本内容和元数据
  - 处理大文件的分块读取
  - 缓存解析结果以提高性能

- **FileTreeTools.ts** - 文件树工具
  - 浏览和搜索 Obsidian 库的文件结构
  - 创建、删除、移动文件和文件夹
  - 获取文件属性和统计信息
  - 支持批量操作

#### 2. 笔记管理工具

- **NoteTools.ts** - 笔记操作工具
  - 创建、更新、删除笔记
  - 管理笔记的前置信息（frontmatter）
  - 处理标签和链接
  - 支持笔记模板

#### 3. 搜索工具

- **SearchTools.ts** - 搜索工具集
  - 全文搜索笔记内容
  - 按标签、文件夹、日期搜索
  - 语义搜索集成
  - 搜索结果去重和排序

#### 4. 时间工具

- **TimeTools.ts** - 时间处理工具
  - 日期解析和格式化
  - 时区转换
  - 时间计算和比较
  - 日程和提醒功能

#### 5. 标签工具

- **TagTools.ts** - 标签管理工具
  - 创建和管理标签
  - 查找标签使用情况
  - 标签层次和关系
  - 标签清理和优化

#### 6. 其他工具

- **CanvasLoader.ts** - Canvas 加载器
  - 读取和处理 Obsidian Canvas 文件
  - 提取节点和连接信息
  - 转换为其他格式

- **YoutubeTools.ts** - YouTube 工具
  - 获取视频信息
  - 提取字幕和元数据
  - 生成笔记摘要

- **memoryTools.ts** - 记忆工具
  - 与记忆系统交互
  - 存储和检索用户记忆
  - 管理记忆上下文

## 工具架构

### 工具接口定义

```typescript
interface Tool {
  name: string                    // 工具唯一名称
  description: string             // 工具描述
  parameters: ToolParameter[]     // 参数定义
  execute: ToolExecutor          // 执行函数
  permission?: ToolPermission    // 权限要求
  category?: ToolCategory        // 工具分类
  enabled?: boolean             // 是否启用
}

interface ToolParameter {
  name: string                  // 参数名
  type: ParameterType          // 参数类型
  required: boolean            // 是否必需
  description: string          // 参数描述
  default?: any                // 默认值
  validation?: ValidationRule  // 验证规则
}

type ToolExecutor = (
  input: ToolInput,
  context: ToolContext
) => Promise<ToolResult>
```

### 工具上下文

```typescript
interface ToolContext {
  // 用户信息
  userId: string
  userName: string

  // 当前环境
  vaultPath: string
  currentFile?: TFile
  currentFolder?: TFolder

  // 权限信息
  permissions: UserPermissions

  // 会话信息
  sessionId: string
  conversationId: string

  // 辅助服务
  apiClient: APIClient
  searchEngine: SearchEngine
  memoryManager: MemoryManager
}
```

### 工具结果

```typescript
interface ToolResult {
  success: boolean              // 执行是否成功
  output: any                   // 输出内容
  type: ResultType             // 输出类型
  metadata?: ResultMetadata    // 元数据
  error?: ToolError           // 错误信息
  warnings?: string[]         // 警告信息
}

enum ResultType {
  TEXT = 'text',
  JSON = 'json',
  MARKDOWN = 'markdown',
  FILE_PATH = 'file_path',
  FILE_LIST = 'file_list',
  BINARY = 'binary'
}
```

## 工具注册和管理

### 注册工具

```typescript
// 注册单个工具
ToolRegistry.register(new SearchTool())

// 批量注册
ToolRegistry.registerBatch([
  new FileReadTool(),
  new FileWriteTool(),
  new NoteCreateTool()
])

// 条件注册
if (userHasPermission('canvas')) {
  ToolRegistry.register(new CanvasTool())
}
```

### 查询工具

```typescript
// 获取所有工具
const allTools = ToolRegistry.getAll()

// 按类别筛选
const searchTools = ToolRegistry.getByCategory('search')

// 按权限筛选
const allowedTools = ToolRegistry.getByPermission(userPermissions)

// 搜索工具
const foundTools = ToolRegistry.search('read file')
```

### 工具配置

```typescript
interface ToolConfig {
  enabled: boolean           // 是否启用
  parameters: Record<string, any>  // 参数默认值
  permissions: string[]      // 所需权限
  rateLimit?: RateLimit     // 速率限制
  timeout?: number         // 超时时间
}

// 更新工具配置
ToolRegistry.updateConfig('read-file', {
  enabled: true,
  timeout: 10000,
  rateLimit: {
    max: 10,
    window: 60000
  }
})
```

## 安全机制

### 权限系统

```typescript
enum Permission {
  READ_FILES = 'read:files',
  WRITE_FILES = 'write:files',
  DELETE_FILES = 'delete:files',
  READ_VAULT = 'read:vault',
  MODIFY_VAULT = 'modify:vault',
  ACCESS_NETWORK = 'access:network',
  EXECUTE_CODE = 'execute:code'
}

// 检查权限
function checkPermission(
  tool: Tool,
  userPermissions: Permission[]
): boolean {
  return tool.permissions?.every(p => userPermissions.includes(p)) ?? true
}
```

### 沙箱执行

```typescript
class ToolSandbox {
  // 在沙箱中执行工具
  async executeSafely(
    tool: Tool,
    input: ToolInput,
    context: ToolContext
  ): Promise<ToolResult> {
    // 1. 权限检查
    if (!this.checkPermissions(tool, context.permissions)) {
      return {
        success: false,
        error: new PermissionError('Insufficient permissions')
      }
    }

    // 2. 参数验证
    const validation = this.validateParameters(tool, input)
    if (!validation.valid) {
      return {
        success: false,
        error: new ValidationError(validation.errors)
      }
    }

    // 3. 执行工具
    try {
      const result = await Promise.race([
        tool.execute(input, context),
        this.timeout(tool.timeout || 30000)
      ])

      // 4. 结果过滤
      return this.filterResult(result, context)
    } catch (error) {
      return this.handleError(error)
    }
  }
}
```

## 性能优化

### 工具缓存

```typescript
class ToolCache {
  private cache: Map<string, CacheEntry> = new Map()
  private ttl: number = 5 * 60 * 1000 // 5分钟

  // 缓存结果
  set(key: string, result: ToolResult): void {
    this.cache.set(key, {
      result,
      timestamp: Date.now()
    })
  }

  // 获取缓存
  get(key: string): ToolResult | null {
    const entry = this.cache.get(key)
    if (entry && Date.now() - entry.timestamp < this.ttl) {
      return entry.result
    }
    return null
  }
}
```

### 批量操作

```typescript
// 批量读取文件
const results = await Promise.all(
  files.map(file => FileReadTool.execute({ path: file.path }))
)

// 并行搜索
const searchResults = await Promise.allSettled([
  SearchTools.search('query1'),
  SearchTools.search('query2'),
  SearchTools.search('query3')
])
```

### 延迟加载

```typescript
class LazyToolLoader {
  private toolLoaders: Map<string, () => Promise<Tool>> = new Map()

  // 注册延迟加载工具
  register(name: string, loader: () => Promise<Tool>): void {
    this.toolLoaders.set(name, loader)
  }

  // 按需加载
  async loadTool(name: string): Promise<Tool | null> {
    if (!this.toolLoaders.has(name)) {
      return null
    }

    const loader = this.toolLoaders.get(name)!
    return await loader()
  }
}
```

## 工具开发

### 创建自定义工具

```typescript
class CustomTool implements Tool {
  name = 'custom-action'
  description = '执行自定义操作'

  parameters: ToolParameter[] = [
    {
      name: 'input',
      type: ParameterType.STRING,
      required: true,
      description: '输入参数'
    }
  ]

  async execute(
    input: ToolInput,
    context: ToolContext
  ): Promise<ToolResult> {
    try {
      // 执行自定义逻辑
      const result = await this.performAction(input.input, context)

      return {
        success: true,
        output: result,
        type: ResultType.TEXT
      }
    } catch (error) {
      return {
        success: false,
        error: {
          message: error.message,
          code: 'CUSTOM_ERROR'
        }
      }
    }
  }

  private async performAction(
    input: string,
    context: ToolContext
  ): Promise<string> {
    // 实现具体逻辑
    return `处理结果: ${input}`
  }
}

// 注册工具
ToolRegistry.register(new CustomTool())
```

### 工具装饰器

```typescript
// 权限装饰器
@Tool({
  name: 'secure-tool',
  permissions: [Permission.READ_FILES],
  category: ToolCategory.FILE
})
class SecureTool {
  @Parameter({
    type: ParameterType.STRING,
    required: true,
    description: '文件路径'
  })
  async execute(@Input('path') path: string): Promise<ToolResult> {
    // 实现
  }
}

// 缓存装饰器
@CacheResult(ttl: 60000)
async expensiveOperation(input: string): Promise<string> {
  // 耗时操作
}
```

## 测试与质量

### 测试策略

- **单元测试** - 每个工具的独立功能测试
- **集成测试** - 工具组合和流程测试
- **性能测试** - 工具执行时间和资源使用
- **安全测试** - 权限控制和输入验证

### 测试文件

- `ComposerTools.test.ts` - 作曲器工具测试
- `FileTreeTools.test.ts` - 文件树工具测试
- `NoteTools.test.ts` - 笔记工具测试
- `ReadNoteTool.test.ts` - 读取笔记工具测试
- `SearchTools.test.ts` - 搜索工具测试
- `SimpleTool.test.ts` - 简单工具测试
- `TagTools.test.ts` - 标签工具测试
- `TimeTools.test.ts` - 时间工具测试
- `ToolResultFormatter.test.ts` - 结果格式化测试
- `allTools.validation.test.ts` - 所有工具验证测试

### 测试覆盖率

- 功能测试：100%
- 边界条件：95%
- 错误处理：90%
- 性能测试：80%

## 常见问题 (FAQ)

### Q: 如何添加新工具？
A: 实现 Tool 接口，然后注册到 ToolRegistry。

### Q: 工具如何访问 Obsidian API？
A: 通过 ToolContext 中的 app 属性访问完整的 Obsidian API。

### Q: 如何处理工具的错误？
A: 工具应该返回包含错误信息的 ToolResult，系统会统一处理和显示。

### Q: 工具支持异步操作吗？
A: 是的，所有工具都是异步的，支持 Promise 和 async/await。

## 相关文件清单

```
src/tools/
├── ToolRegistry.ts                  # 工具注册表
├── toolManager.ts                   # 工具管理器
├── ToolResultFormatter.ts           # 结果格式化器
├── builtinTools.ts                  # 内置工具集合
├── CanvasLoader.ts                  # Canvas 加载器
├── ComposerTools.ts                 # 作曲器工具
├── FileParserManager.ts             # 文件解析管理器
├── FileTreeTools.ts                 # 文件树工具
├── memoryTools.ts                   # 记忆工具
├── NoteTools.ts                     # 笔记工具
├── SearchTools.ts                   # 搜索工具
├── SimpleTool.ts                    # 简单工具基类
├── TagTools.ts                      # 标签工具
├── TimeTools.ts                     # 时间工具
├── YoutubeTools.ts                  # YouTube 工具
└── __tests__/                       # 测试文件
    ├── allTools.validation.test.ts
    ├── ComposerTools.test.ts
    ├── FileTreeTools.test.ts
    ├── NoteTools.test.ts
    ├── ReadNoteTool.test.ts
    ├── SearchTools.dedupe.test.ts
    ├── SearchTools.schema.test.ts
    ├── SearchTools.test.ts
    ├── SimpleTool.test.ts
    ├── TagTools.test.ts
    ├── TimeTools.test.ts
    ├── TimeTools.timezone.test.ts
    └── ToolResultFormatter.test.ts
```

## 变更记录 (Changelog)

### 2025-12-16 16:20:00
- ✨ 创建工具模块文档
- 📚 详细说明工具系统架构和接口
- 🔗 记录核心工具和安全机制
- 📝 提供工具开发指南和示例