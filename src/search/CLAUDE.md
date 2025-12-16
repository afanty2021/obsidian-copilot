[根目录](../../CLAUDE.md) > [src](../) > **search**

# Search Engine Module

## 模块职责

搜索引擎模块为 Obsidian Copilot 提供强大的搜索功能，包括语义搜索、词法搜索和混合搜索。该模块实现了 v3 版本的搜索引擎，支持分层检索、查询扩展、评分归一化等高级功能，同时保持向后兼容的 Orama 支持。

## 入口与启动

### 核心组件

- **vectorStoreManager.ts** - 向量存储管理器（已弃用，保留兼容性）
  - 管理 Orama 索引（遗留）
  - 新版本使用 MemoryIndexManager
  - 处理索引同步和重建

- **IndexManager.ts** - 索引管理器
  - 统一的索引管理接口
  - 协调多个搜索引擎
  - 处理索引生命周期

### v3 搜索引擎组件

- **SearchCore.ts** - 搜索核心引擎
  - 统一的搜索接口
  - 集成多种检索器
  - 结果合并和排序

- **MergedSemanticRetriever.ts** - 合并语义检索器
  - 结合语义和词法搜索
  - 智能结果合并
  - 评分权重调整

- **QueryExpander.ts** - 查询扩展器
  - 同义词扩展
  - 相关性增强
  - 查询重写

- **TieredLexicalRetriever.ts** - 分层词法检索器
  - 多级检索策略
  - 文件名优先级
  - 路径权重计算

### 存储和索引

- **ChunkedStorage.ts** - 分块存储
  - 大文档高效处理
  - 智能分块策略
  - 元数据保留

- **MemoryIndexManager.ts** - 内存索引管理器
  - JSONL 快照格式
  - 增量更新
  - 快速加载

## 子系统模块

### 检索器 (retrievers/)

- **FullTextEngine.ts** - 全文搜索引擎
- **SemanticEngine.ts** - 语义搜索引擎
- **HybridRetriever.ts** - 混合检索器
- **FolderBoostRetriever.ts** - 文件夹增强检索器

### 扫描器 (scanners/)

- **GrepScanner.ts** - Grep 扫描器
- **FileScanner.ts** - 文件扫描器
- **TagScanner.ts** - 标签扫描器

### 评分和排序 (scoring/)

- **ScoreNormalizer.ts** - 评分归一化
- **FolderBoostCalculator.ts** - 文件夹权重计算
- **RelevanceScorer.ts** - 相关性评分

### 工具函数 (utils/)

- **FuzzyMatcher.ts** - 模糊匹配
- **ScoreNormalizer.ts** - 分数归一化
- **TagUtils.ts** - 标签工具
- **ChunkUtils.ts** - 分块工具

### 数据结构

- **SearchResult.ts** - 搜索结果
- **SearchQuery.ts** - 搜索查询
- **SearchContext.ts** - 搜索上下文

## 对外接口

### SearchCore 主要方法

```typescript
class SearchCore {
  // 执行搜索
  async search(query: SearchQuery): Promise<SearchResult[]>

  // 添加文档
  async addDocument(doc: Document): Promise<void>

  // 删除文档
  async removeDocument(docId: string): Promise<void>

  // 重建索引
  async rebuildIndex(): Promise<void>
}
```

### IndexManager 主要方法

```typescript
class IndexManager {
  // 索引管理
  async initialize(): Promise<void>
  async addDocuments(docs: Document[]): Promise<void>
  async removeDocument(path: string): Promise<void>

  // 搜索接口
  async search(query: string, options?: SearchOptions): Promise<SearchResult[]>

  // 生命周期
  async save(): Promise<void>
  async load(): Promise<void>
}
```

### MergedSemanticRetriever 主要方法

```typescript
class MergedSemanticRetriever {
  // 检索
  async retrieve(query: string, k?: number): Promise<Document[]>

  // 配置
  setSemanticWeight(weight: number): void
  setLexicalWeight(weight: number): void
}
```

## 关键依赖与配置

### 搜索引擎

- `@orama/orama` ^3.0.0-rc-2 - Orama 搜索引擎（遗留）
- `flexsearch` ^0.8.205 - FlexSearch 快速搜索
- `fuzzysort` ^3.1.0 - 模糊搜索

### 文本处理

- `@langchain/textsplitters` ^1.0.0 - 文本分割
- `chrono-node` ^2.7.7 - 时间解析

### 配置项

```typescript
interface SearchSettings {
  enableSemanticSearchV3: boolean  // 启用 v3 语义搜索
  enableIndexSync: boolean        // 启用索引同步
  maxSourceChunks: number        // 最大源块数
  qaExclusions: string[]         // 排除路径
  chatNoteContextPath: string    // 聊天上下文路径
  chatNoteContextTags: string[]  // 聊天上下文标签
}
```

## 数据模型

### SearchResult

```typescript
interface SearchResult {
  document: Document
  score: number
  metadata: {
    path: string
    mtime: number
    size: number
    tags?: string[]
    folder?: string
  }
  highlights?: string[]
  excerpt?: string
}
```

### Document

```typescript
interface Document {
  id: string
  content: string
  metadata: DocumentMetadata
  embeddings?: number[]
  chunks?: DocumentChunk[]
}
```

### DocumentChunk

```typescript
interface DocumentChunk {
  id: string
  content: string
  index: number
  metadata: ChunkMetadata
  embeddings?: number[]
}
```

## 测试与质量

### 测试文件

- `SearchCore.test.ts` - 搜索核心测试
- `MergedSemanticRetriever.test.ts` - 语义检索测试
- `QueryExpander.test.ts` - 查询扩展测试
- `TieredLexicalRetriever.test.ts` - 词法检索测试
- `chunks.test.ts` - 分块测试
- `engines/FullTextEngine.test.ts` - 全文引擎测试
- `scanners/GrepScanner.test.ts` - Grep 扫描器测试
- `scoring/FolderBoostCalculator.test.ts` - 权重计算测试
- `utils/FuzzyMatcher.test.ts` - 模糊匹配测试
- `utils/ScoreNormalizer.test.ts` - 分数归一化测试
- `utils/tagUtils.test.ts` - 标签工具测试

### 测试覆盖

- 搜索算法准确性
- 性能基准测试
- 索引一致性
- 边界条件处理
- 错误恢复机制

## 常见问题 (FAQ)

### Q: v3 搜索引擎与旧版本的区别？
A: v3 使用 MemoryIndexManager 和 JSONL 快照，支持增量更新，性能更好，内存使用更少。

### Q: 如何处理大型文档？
A: 使用 ChunkedStorage 将大文档分割成小块，支持并行处理和部分匹配。

### Q: 搜索结果如何排序？
A: 使用 ScoreNormalizer 归一化不同检索器的分数，然后结合文件夹权重、时间衰减等因素排序。

### Q: 如何添加自定义检索器？
A: 实现 BaseRetriever 接口，并在 SearchCore 中注册新的检索器。

## 相关文件清单

```
src/search/
├── vectorStoreManager.ts          # 向量存储管理器（已弃用）
├── IndexManager.ts                # 索引管理器
├── ChunkedStorage.ts             # 分块存储
├── dbOperations.ts               # 数据库操作（Orama）
├── indexEventHandler.ts          # 索引事件处理
├── indexOperations.ts            # 索引操作
├── searchUtils.ts                # 搜索工具
├── v3/                           # v3 搜索引擎
│   ├── SearchCore.ts             # 搜索核心
│   ├── MergedSemanticRetriever.ts
│   ├── QueryExpander.ts
│   ├── TieredLexicalRetriever.ts
│   ├── chunks.ts                 # 分块逻辑
│   ├── engines/                  # 搜索引擎
│   │   └── FullTextEngine.ts
│   ├── retrievers/               # 检索器
│   ├── scanners/                 # 扫描器
│   │   └── GrepScanner.ts
│   ├── scoring/                  # 评分系统
│   │   ├── FolderBoostCalculator.ts
│   │   └── ScoreNormalizer.ts
│   ├── utils/                    # 工具函数
│   │   ├── FuzzyMatcher.ts
│   │   ├── ScoreNormalizer.ts
│   │   └── tagUtils.ts
│   ├── SearchResult.ts           # 搜索结果类型
│   ├── SearchQuery.ts            # 搜索查询类型
│   └── SearchContext.ts          # 搜索上下文类型
└── __tests__/                    # 测试文件
    ├── searchUtils.test.ts
    └── v3/                       # v3 测试
        ├── SearchCore.test.ts
        ├── MergedSemanticRetriever.test.ts
        ├── QueryExpander.test.ts
        ├── TieredLexicalRetriever.test.ts
        ├── chunks.test.ts
        ├── engines/FullTextEngine.test.ts
        ├── scanners/GrepScanner.test.ts
        ├── scoring/FolderBoostCalculator.test.ts
        └── utils/
            ├── FuzzyMatcher.test.ts
            ├── ScoreNormalizer.test.ts
            └── tagUtils.test.ts
```

## 变更记录 (Changelog)

### 2025-12-07 14:10:46
- ✨ 创建搜索引擎模块文档
- 📊 详细说明了 v3 搜索引擎架构
- 🔗 记录了索引管理和检索器系统
- 📝 列出了完整的测试覆盖情况