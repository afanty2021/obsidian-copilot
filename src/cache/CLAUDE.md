[根目录](../../CLAUDE.md) > [src](../) > **cache**

# Cache Module

## 模块职责

缓存模块为 Obsidian Copilot 提供了高性能的缓存解决方案，用于存储和管理各种临时数据，包括文件内容、搜索结果、嵌入向量、项目上下文等。该模块通过多层次的缓存策略，显著提高了应用的响应速度，减少了重复计算和 API 调用。

## 入口与启动

### 核心组件

- **fileCache.ts** - 文件内容缓存
  - 缓存文件读取的文本内容
  - 支持文件变更检测和自动失效
  - 提供大文件的分块缓存
  - 集成 LRU 淘汰策略

- **pdfCache.ts** - PDF 文件缓存
  - 专门处理 PDF 文件的解析和缓存
  - 存储提取的文本和元数据
  - 支持页面级别的缓存粒度
  - 优化 PDF 处理性能

- **projectContextCache.ts** - 项目上下文缓存
  - 缓存项目的上下文信息
  - 管理上下文的构建和更新
  - 支持增量更新和依赖追踪
  - 提供上下文的版本控制

## 缓存架构

### 缓存层次结构

```typescript
interface CacheHierarchy {
  // L1 - 内存缓存（最快）
  memoryCache: MemoryCache

  // L2 - 本地存储缓存（中等）
  localStorageCache: LocalStorageCache

  // L3 - 文件系统缓存（持久化）
  fileSystemCache: FileSystemCache
}
```

### 缓存策略

```typescript
enum CacheStrategy {
  LRU = 'lru',              // 最近最少使用
  LFU = 'lfu',              // 最少使用频率
  TTL = 'ttl',              // 生存时间
  SIZE_BASED = 'size_based', // 基于大小
  MANUAL = 'manual'         // 手动控制
}

interface CacheConfig {
  strategy: CacheStrategy   // 缓存策略
  maxSize: number          // 最大缓存大小
  ttl: number             // 生存时间（毫秒）
  persist: boolean        // 是否持久化
  compression: boolean     // 是否压缩
  encryption: boolean      // 是否加密
}
```

## 核心缓存实现

### 1. 文件缓存 (FileCache)

```typescript
class FileCache {
  private cache: Map<string, CacheEntry> = new Map()
  private watcher: FileSystemWatcher
  private config: CacheConfig

  constructor(config: CacheConfig) {
    this.config = config
    this.watcher = new FileSystemWatcher()
    this.setupWatcher()
  }

  // 获取文件内容
  async get(filePath: string): Promise<string | null> {
    const cached = this.cache.get(filePath)

    // 检查缓存是否有效
    if (cached && this.isValid(cached)) {
      return cached.content
    }

    // 缓存未命中或已过期，读取文件
    const content = await this.readFile(filePath)
    if (content) {
      this.set(filePath, content)
    }

    return content
  }

  // 设置文件缓存
  set(filePath: string, content: string): void {
    const stat = fs.statSync(filePath)

    this.cache.set(filePath, {
      content,
      mtime: stat.mtime.getTime(),
      size: stat.size,
      timestamp: Date.now(),
      accessCount: 0
    })

    // 检查缓存大小限制
    this.evictIfNeeded()
  }

  // 检查缓存有效性
  private isValid(entry: CacheEntry): boolean {
    // 检查 TTL
    if (Date.now() - entry.timestamp > this.config.ttl) {
      return false
    }

    // 检查文件是否修改
    try {
      const stat = fs.statSync(entry.path)
      return stat.mtime.getTime() === entry.mtime
    } catch {
      return false
    }
  }
}
```

### 2. PDF 缓存 (PDFCache)

```typescript
class PDFCache {
  private cache: Map<string, PDFCacheEntry> = new Map()
  private parser: PDFParser

  // 获取 PDF 文本
  async getText(pdfPath: string, page?: number): Promise<string> {
    const key = this.generateKey(pdfPath, page)
    const cached = this.cache.get(key)

    if (cached && this.isValid(cached)) {
      return cached.text
    }

    // 解析 PDF
    const result = await this.parser.parse(pdfPath, {
      pages: page ? [page] : undefined,
      includeMetadata: true
    })

    // 缓存结果
    this.cache.set(key, {
      text: result.text,
      metadata: result.metadata,
      pageCount: result.pageCount,
      timestamp: Date.now()
    })

    return result.text
  }

  // 获取 PDF 元数据
  async getMetadata(pdfPath: string): Promise<PDFMetadata | null> {
    const cached = this.cache.get(pdfPath)

    if (cached && cached.metadata) {
      return cached.metadata
    }

    // 解析元数据
    const metadata = await this.parser.getMetadata(pdfPath)
    if (metadata) {
      // 更新缓存条目
      const existing = this.cache.get(pdfPath) || {}
      this.cache.set(pdfPath, {
        ...existing,
        metadata,
        timestamp: Date.now()
      })
    }

    return metadata
  }
}
```

### 3. 项目上下文缓存 (ProjectContextCache)

```typescript
class ProjectContextCache {
  private cache: Map<string, ProjectContext> = new Map()
  private dependencies: DependencyTracker
  private versionManager: VersionManager

  // 获取项目上下文
  async getProjectContext(
    projectId: string,
    options?: ContextOptions
  ): Promise<ProjectContext> {
    const context = this.cache.get(projectId)

    // 检查上下文是否需要更新
    if (context && this.isContextValid(context, options)) {
      return context
    }

    // 构建新的上下文
    const newContext = await this.buildContext(projectId, options)

    // 缓存上下文
    this.cache.set(projectId, {
      ...newContext,
      version: this.versionManager.getNextVersion(),
      timestamp: Date.now(),
      dependencies: this.dependencies.track(projectId)
    })

    return newContext
  }

  // 增量更新上下文
  async updateContext(
    projectId: string,
    changes: ContextChange[]
  ): Promise<ProjectContext> {
    const context = this.cache.get(projectId)
    if (!context) {
      return this.getProjectContext(projectId)
    }

    // 应用增量更新
    const updatedContext = await this.applyChanges(context, changes)

    // 更新缓存
    this.cache.set(projectId, {
      ...updatedContext,
      version: context.version + 1,
      timestamp: Date.now()
    })

    return updatedContext
  }

  // 检查上下文有效性
  private isContextValid(
    context: ProjectContext,
    options?: ContextOptions
  ): boolean {
    // 检查依赖项是否变更
    if (this.dependencies.hasChanges(context.dependencies)) {
      return false
    }

    // 检查选项是否兼容
    if (options && !this.areOptionsCompatible(context.options, options)) {
      return false
    }

    // 检查时间戳
    const maxAge = 30 * 60 * 1000 // 30分钟
    return Date.now() - context.timestamp < maxAge
  }
}
```

## 缓存管理器

### 统一缓存接口

```typescript
interface CacheManager {
  // 基础操作
  get<T>(key: string): Promise<T | null>
  set<T>(key: string, value: T, options?: CacheOptions): Promise<void>
  delete(key: string): Promise<boolean>
  clear(): Promise<void>

  // 批量操作
  mget<T>(keys: string[]): Promise<Map<string, T>>
  mset<T>(entries: Map<string, T>): Promise<void>
  mdelete(keys: string[]): Promise<number>

  // 查询操作
  has(key: string): Promise<boolean>
  keys(pattern?: string): Promise<string[]>
  size(): Promise<number>

  // 统计信息
  stats(): CacheStats
  getHits(): number
  getMisses(): number
  getHitRate(): number
}
```

### 缓存配置

```typescript
interface CacheOptions {
  ttl?: number              // 生存时间
  tags?: string[]          // 标签
  priority?: number        // 优先级
  persist?: boolean        // 是否持久化
  compression?: boolean    // 是否压缩
  encrypt?: boolean        // 是否加密
  metadata?: any          // 元数据
}

// 全局缓存配置
const CACHE_CONFIG = {
  fileCache: {
    maxSize: 100 * 1024 * 1024,  // 100MB
    ttl: 24 * 60 * 60 * 1000,    // 24小时
    strategy: CacheStrategy.LRU
  },
  pdfCache: {
    maxSize: 50 * 1024 * 1024,   // 50MB
    ttl: 7 * 24 * 60 * 60 * 1000, // 7天
    strategy: CacheStrategy.LFU
  },
  contextCache: {
    maxSize: 20 * 1024 * 1024,   // 20MB
    ttl: 30 * 60 * 1000,         // 30分钟
    strategy: CacheStrategy.TTL
  }
}
```

## 性能优化

### 1. 压缩存储

```typescript
class CompressedCache {
  private compressor: Compressor

  async set(key: string, value: any): Promise<void> {
    const serialized = JSON.stringify(value)
    const compressed = await this.compressor.compress(serialized)

    await this.storage.set(key, {
      data: compressed,
      originalSize: serialized.length,
      compressedSize: compressed.length,
      compression: true
    })
  }

  async get<T>(key: string): Promise<T | null> {
    const entry = await this.storage.get(key)
    if (!entry) return null

    if (entry.compression) {
      const decompressed = await this.compressor.decompress(entry.data)
      return JSON.parse(decompressed)
    }

    return entry.data
  }
}
```

### 2. 预加载策略

```typescript
class PreloadStrategy {
  // 预加载常用文件
  async preloadCommonFiles(): Promise<void> {
    const recentFiles = await this.getRecentFiles(10)
    const tasks = recentFiles.map(file =>
      fileCache.get(file.path).catch(() => null)
    )

    await Promise.allSettled(tasks)
  }

  // 预加载搜索结果
  async preloadSearchResults(query: string): Promise<void> {
    const suggestions = await this.getQuerySuggestions(query)
    const cacheTasks = suggestions.map(suggestion =>
      searchCache.get(suggestion)
    )

    await Promise.allSettled(cacheTasks)
  }
}
```

### 3. 智能预热

```typescript
class WarmupManager {
  private usageStats: UsageStatsCollector

  async warmup(): Promise<void> {
    const patterns = this.usageStats.getCommonPatterns()

    // 根据使用模式预热
    for (const pattern of patterns) {
      switch (pattern.type) {
        case 'file_access':
          await this.warmupFiles(pattern.files)
          break
        case 'search_query':
          await this.warmupSearches(pattern.queries)
          break
        case 'project_context':
          await this.warmupContexts(pattern.projects)
          break
      }
    }
  }
}
```

## 缓存同步

### 跨实例同步

```typescript
class CacheSync {
  private eventBus: EventBus
  private conflictResolver: ConflictResolver

  // 同步缓存更新
  async syncUpdate(
    cacheKey: string,
    value: any,
    source: string
  ): Promise<void> {
    // 本地更新
    await this.localCache.set(cacheKey, value)

    // 广播更新
    this.eventBus.emit('cache:update', {
      key: cacheKey,
      value,
      source,
      timestamp: Date.now()
    })
  }

  // 处理远程更新
  private async handleRemoteUpdate(event: CacheUpdateEvent): Promise<void> {
    const { key, value, source, timestamp } = event

    // 检查是否需要冲突解决
    const local = await this.localCache.get(key)
    if (local && local.lastModified > timestamp) {
      // 发生冲突
      const resolved = await this.conflictResolver.resolve(
        local,
        { value, timestamp }
      )
      await this.localCache.set(key, resolved)
    } else {
      // 直接应用更新
      await this.localCache.set(key, value)
    }
  }
}
```

## 监控和分析

### 缓存统计

```typescript
interface CacheStats {
  hits: number               // 命中次数
  misses: number             // 未命中次数
  hitRate: number           // 命中率
  size: number              // 缓存大小
  entries: number           // 条目数量
  evictions: number         // 淘汰次数
  avgAccessTime: number     // 平均访问时间
  compressionRatio?: number // 压缩比
}

class CacheMonitor {
  private metrics: MetricsCollector

  // 收集统计信息
  async collectStats(): Promise<CacheStats> {
    return {
      hits: await this.metrics.getCounter('cache.hits'),
      misses: await this.metrics.getCounter('cache.misses'),
      hitRate: await this.calculateHitRate(),
      size: await this.calculateSize(),
      entries: await this.getEntryCount(),
      evictions: await this.metrics.getCounter('cache.evictions'),
      avgAccessTime: await this.calculateAvgAccessTime()
    }
  }

  // 生成性能报告
  async generateReport(): Promise<PerformanceReport> {
    const stats = await this.collectStats()

    return {
      summary: {
        hitRate: stats.hitRate,
        size: stats.size,
        performance: this.getPerformanceGrade(stats)
      },
      recommendations: this.getRecommendations(stats),
      topKeys: await this.getTopAccessedKeys(10),
      patterns: await this.analyzeAccessPatterns()
    }
  }
}
```

## 测试与质量

### 测试策略

- **单元测试** - 每个缓存组件的独立测试
- **性能测试** - 缓存命中率和响应时间测试
- **并发测试** - 多线程访问的缓存一致性
- **持久化测试** - 缓存数据的可靠性和恢复

### 测试文件

- 文件缓存测试（在相关模块中）
- PDF 缓存测试（在相关模块中）
- 项目上下文缓存测试（在相关模块中）

## 常见问题 (FAQ)

### Q: 缓存会占用多少存储空间？
A: 缓存大小有配置限制，默认总大小不超过 200MB，会自动清理旧数据。

### Q: 如何手动清理缓存？
A: 可以通过设置页面的"清理缓存"按钮，或删除 `.obsidian/plugins/copilot/cache` 目录。

### Q: 缓存数据会同步到其他设备吗？
A: 默认不会。缓存是本地优化的，每个设备独立管理。

### Q: 如何禁用特定类型的缓存？
A: 在设置中可以单独控制每种缓存类型的启用状态。

## 相关文件清单

```
src/cache/
├── fileCache.ts                  # 文件内容缓存
├── pdfCache.ts                   # PDF 文件缓存
└── projectContextCache.ts        # 项目上下文缓存
```

## 变更记录 (Changelog)

### 2025-12-16 16:20:00
- ✨ 创建缓存模块文档
- 📚 详细说明三级缓存架构和策略
- 🔗 记录核心缓存组件和优化机制
- 📝 提供缓存管理和监控方案