[根目录](../../CLAUDE.md) > [src](../) > **mentions**

# Mentions Module

## 模块职责

提及（@mentions）模块为 Obsidian Copilot 提供了智能的提及功能，允许用户在聊天中通过 `@` 符号快速引用笔记、标签、文件夹等内容。该模块实现了自动补全、上下文感知和智能搜索，极大地提升了用户输入效率和交互体验。

## 入口与启动

### 核心文件

- **Mention.ts** - 提及功能核心实现
  - 处理 `@` 触发的提及逻辑
  - 管理不同类型的提及项（笔记、标签、文件夹等）
  - 实现提及项的渲染和格式化
  - 提供提及的自定义和扩展接口

## 提及系统架构

### 提及类型

```typescript
enum MentionType {
  NOTE = 'note',           // 笔记文件
  TAG = 'tag',             // 标签
  FOLDER = 'folder',       // 文件夹
  VAULT = 'vault',         // 整个库
  DATETIME = 'datetime',   // 日期时间
  USER = 'user',           // 用户
  COMMAND = 'command',     // 命令
  CUSTOM = 'custom'        // 自定义类型
}

interface MentionItem {
  type: MentionType        // 提及类型
  id: string               // 唯一标识符
  name: string             // 显示名称
  path?: string            // 路径（文件、文件夹）
  description?: string     // 描述信息
  icon?: string            // 图标
  metadata?: any          // 元数据
}
```

### 提及提供者

```typescript
interface MentionProvider {
  type: MentionType        // 提供的提及类型
  name: string             // 提供者名称
  priority: number         // 优先级（用于排序）

  // 获取建议列表
  getSuggestions(
    query: string,
    context: MentionContext
  ): Promise<MentionItem[]>

  // 渲染提及项
  render(item: MentionItem): MentionRender

  // 格式化提及文本
  format(item: MentionItem): string

  // 验证提及项
  validate(item: MentionItem): boolean
}

interface MentionContext {
  // 当前位置
  position: {
    line: number
    column: number
    offset: number
  }

  // 当前文件
  currentFile?: TFile

  // 项目信息
  projectId?: string

  // 用户偏好
  userPreferences?: UserPreferences
}
```

## 内置提供者实现

### 1. 笔记提及提供者

```typescript
class NoteMentionProvider implements MentionProvider {
  type = MentionType.NOTE
  name = 'Notes'
  priority = 1

  async getSuggestions(
    query: string,
    context: MentionContext
  ): Promise<MentionItem[]> {
    const vault = context.currentFile?.vault
    if (!vault) return []

    // 获取所有笔记
    const files = vault.getMarkdownFiles()

    // 搜索匹配的笔记
    const matches = files.filter(file =>
      file.basename.toLowerCase().includes(query.toLowerCase())
    )

    // 转换为提及项
    return matches.slice(0, 10).map(file => ({
      type: MentionType.NOTE,
      id: file.path,
      name: file.basename,
      path: file.path,
      description: file.path,
      icon: 'file-text',
      metadata: {
        size: file.stat.size,
        mtime: file.stat.mtime,
        tags: this.extractTags(file)
      }
    }))
  }

  render(item: MentionItem): MentionRender {
    return {
      text: item.name,
      element: this.createNoteElement(item)
    }
  }

  format(item: MentionItem): string {
    return `[[${item.path}]]`
  }

  private createNoteElement(item: MentionItem): HTMLElement {
    const container = createDiv('mention-item')

    // 图标
    const icon = container.createEl('span', { cls: 'mention-icon' })
    icon.innerHTML = this.getIconElement(item.icon || 'file-text')

    // 名称
    const name = container.createEl('span', { cls: 'mention-name' })
    name.textContent = item.name

    // 路径
    if (item.path && item.path !== item.name) {
      const path = container.createEl('span', { cls: 'mention-path' })
      path.textContent = item.path
    }

    // 标签
    if (item.metadata?.tags?.length) {
      const tags = container.createEl('span', { cls: 'mention-tags' })
      tags.textContent = item.metadata.tags.slice(0, 3).join(', ')
    }

    return container
  }
}
```

### 2. 标签提及提供者

```typescript
class TagMentionProvider implements MentionProvider {
  type = MentionType.TAG
  name = 'Tags'
  priority = 2

  async getSuggestions(
    query: string,
    context: MentionContext
  ): Promise<MentionItem[]> {
    const vault = context.currentFile?.vault
    if (!vault) return []

    // 获取所有标签
    const tags = this.getAllTags(vault)

    // 搜索匹配的标签
    const matches = tags.filter(tag =>
      tag.toLowerCase().includes(query.toLowerCase())
    )

    return matches.slice(0, 10).map(tag => ({
      type: MentionType.TAG,
      id: tag,
      name: `#${tag}`,
      icon: 'tag',
      description: `${this.getTagUsage(vault, tag)} 个使用`
    }))
  }

  format(item: MentionItem): string {
    return item.name // 已经包含 # 前缀
  }
}
```

### 3. 文件夹提及提供者

```typescript
class FolderMentionProvider implements MentionProvider {
  type = MentionType.FOLDER
  name = 'Folders'
  priority = 3

  async getSuggestions(
    query: string,
    context: MentionContext
  ): Promise<MentionItem[]> {
    const vault = context.currentFile?.vault
    if (!vault) return []

    // 获取所有文件夹
    const folders = this.getAllFolders(vault)

    // 搜索匹配的文件夹
    const matches = folders.filter(folder =>
      folder.name.toLowerCase().includes(query.toLowerCase())
    )

    return matches.slice(0, 10).map(folder => ({
      type: MentionType.FOLDER,
      id: folder.path,
      name: folder.name,
      path: folder.path,
      icon: 'folder',
      description: `${folder.fileCount} 个文件`
    }))
  }

  format(item: MentionItem): string {
    return `[${item.path}/]`
  }
}
```

## 提及管理器

```typescript
class MentionManager {
  private providers: Map<MentionType, MentionProvider> = new Map()
  private activeProvider?: MentionProvider
  private isVisible = false

  // 注册提供者
  register(provider: MentionProvider): void {
    this.providers.set(provider.type, provider)
  }

  // 注销提供者
  unregister(type: MentionType): void {
    this.providers.delete(type)
  }

  // 触发提及
  async trigger(
    query: string,
    context: MentionContext
  ): Promise<MentionItem[]> {
    if (!query.startsWith('@')) return []

    const queryWithoutAt = query.substring(1)
    const suggestions: MentionItem[] = []

    // 从所有提供者获取建议
    for (const provider of this.providers.values()) {
      try {
        const providerSuggestions = await provider.getSuggestions(
          queryWithoutAt,
          context
        )
        suggestions.push(...providerSuggestions)
      } catch (error) {
        console.error(`Error in mention provider ${provider.name}:`, error)
      }
    }

    // 排序和限制结果
    return suggestions
      .sort((a, b) => this.compareItems(a, b, queryWithoutAt))
      .slice(0, 10)
  }

  // 选择提及项
  select(item: MentionItem): string {
    const provider = this.providers.get(item.type)
    if (!provider) return ''

    return provider.format(item)
  }

  // 自定义排序
  private compareItems(
    a: MentionItem,
    b: MentionItem,
    query: string
  ): number {
    // 精确匹配优先
    const aExact = a.name.toLowerCase() === query.toLowerCase()
    const bExact = b.name.toLowerCase() === query.toLowerCase()
    if (aExact && !bExact) return -1
    if (!aExact && bExact) return 1

    // 前缀匹配优先
    const aPrefix = a.name.toLowerCase().startsWith(query.toLowerCase())
    const bPrefix = b.name.toLowerCase().startsWith(query.toLowerCase())
    if (aPrefix && !bPrefix) return -1
    if (!aPrefix && bPrefix) return 1

    // 按提供者优先级排序
    const aProvider = this.providers.get(a.type)
    const bProvider = this.providers.get(b.type)
    if (aProvider && bProvider && aProvider.priority !== bProvider.priority) {
      return aProvider.priority - bProvider.priority
    }

    // 按名称排序
    return a.name.localeCompare(b.name)
  }
}
```

## 集成到编辑器

### Lexical 编辑器集成

```typescript
class MentionPlugin extends LexicalPlugin {
  private mentionManager: MentionManager

  constructor(mentionManager: MentionManager) {
    super()
    this.mentionManager = mentionManager
  }

  populateEditor(editor: LexicalEditor): void {
    // 注册提及节点
    editor.registerNodes([MentionNode])

    // 添加提及命令
    editor.registerCommand(
      INSERT_MENTION_COMMAND,
      (payload: MentionPayload) => {
        const mentionNode = new MentionNode(payload.type, payload.id, payload.name)
        editor.update(() => {
          const selection = $getSelection()
          if ($isRangeSelection(selection)) {
            selection.insertNodes([mentionNode])
          }
        })
        return true
      },
      COMMAND_PRIORITY_LOW
    )

    // 监听文本变化
    editor.registerUpdateListener(({ editorState }) => {
      editorState.read(() => {
        const selection = $getSelection()
        if (!$isRangeSelection(selection)) return

        const text = selection.getTextContent()
        const mentionTrigger = this.findMentionTrigger(text)

        if (mentionTrigger) {
          this.showMentionSuggestions(mentionTrigger)
        } else {
          this.hideMentionSuggestions()
        }
      })
    })
  }

  private findMentionTrigger(text: string): MentionTrigger | null {
    const index = text.lastIndexOf('@')
    if (index === -1) return null

    // 检查 @ 前的字符
    const prevChar = text[index - 1]
    if (prevChar && !/\s/.test(prevChar)) {
      return null // @ 必须是词的开始
    }

    // 提取查询文本
    const query = text.substring(index)
    const spaceIndex = query.indexOf(' ')
    const actualQuery = spaceIndex === -1 ? query : query.substring(0, spaceIndex)

    return {
      index,
      query: actualQuery,
      fullText: text
    }
  }
}
```

### 提及节点定义

```typescript
class MentionNode extends LexicalNode {
  __type: MentionType
  __id: string
  __name: string

  static getType(): string {
    return 'mention'
  }

  static clone(node: MentionNode): MentionNode {
    return new MentionNode(node.__type, node.__id, node.__name)
  }

  constructor(type: MentionType, id: string, name: string) {
    super('mention')
    this.__type = type
    this.__id = id
    this.__name = name
  }

  createDOM(): HTMLElement {
    const element = document.createElement('span')
    element.className = `mention mention-${this.__type}`
    element.contentEditable = 'false'
    element.textContent = this.__name
    return element
  }

  exportJSON(): MentionJSON {
    return {
      type: 'mention',
      version: 1,
      mentionType: this.__type,
      id: this.__id,
      name: this.__name
    }
  }

  static importJSON(json: MentionJSON): MentionNode {
    return new MentionNode(json.mentionType, json.id, json.name)
  }

  getTextContent(): string {
    const provider = mentionManager.providers.get(this.__type)
    if (provider) {
      return provider.format({
        type: this.__type,
        id: this.__id,
        name: this.__name
      })
    }
    return this.__name
  }
}
```

## 自定义提供者

### 创建自定义提及提供者

```typescript
class CustomMentionProvider implements MentionProvider {
  type = MentionType.CUSTOM
  name = 'Custom Items'
  priority = 10

  // 自定义数据源
  private items: CustomItem[] = []

  async getSuggestions(
    query: string,
    context: MentionContext
  ): Promise<MentionItem[]> {
    return this.items
      .filter(item => item.name.toLowerCase().includes(query.toLowerCase()))
      .slice(0, 10)
      .map(item => ({
        type: MentionType.CUSTOM,
        id: item.id,
        name: item.name,
        description: item.description,
        icon: item.icon,
        metadata: item
      }))
  }

  // 添加自定义项
  addItem(item: CustomItem): void {
    this.items.push(item)
  }

  // 移除自定义项
  removeItem(id: string): void {
    this.items = this.items.filter(item => item.id !== id)
  }
}

// 注册自定义提供者
const customProvider = new CustomMentionProvider()
customProvider.addItem({
  id: 'meeting-template',
  name: '会议模板',
  description: '标准的会议纪要模板',
  icon: 'calendar'
})

mentionManager.register(customProvider)
```

## 性能优化

### 1. 防抖搜索

```typescript
class DebouncedMentionSearch {
  private debounceTimer?: NodeJS.Timeout
  private lastQuery = ''

  search(query: string, callback: (results: MentionItem[]) => void): void {
    // 清除之前的定时器
    if (this.debounceTimer) {
      clearTimeout(this.debounceTimer)
    }

    // 如果查询没有变化，不重新搜索
    if (query === this.lastQuery) return
    this.lastQuery = query

    // 防抖延迟
    this.debounceTimer = setTimeout(() => {
      this.performSearch(query, callback)
    }, 200)
  }

  private async performSearch(
    query: string,
    callback: (results: MentionItem[]) => void
  ): Promise<void> {
    const results = await this.mentionManager.trigger(query, this.context)
    callback(results)
  }
}
```

### 2. 缓存结果

```typescript
class MentionCache {
  private cache = new Map<string, CachedMentionResult>()

  get(query: string): MentionItem[] | null {
    const cached = this.cache.get(query)
    if (!cached) return null

    // 检查缓存是否过期
    if (Date.now() - cached.timestamp > 5000) {
      this.cache.delete(query)
      return null
    }

    return cached.results
  }

  set(query: string, results: MentionItem[]): void {
    this.cache.set(query, {
      results,
      timestamp: Date.now()
    })

    // 限制缓存大小
    if (this.cache.size > 100) {
      const oldestKey = this.cache.keys().next().value
      this.cache.delete(oldestKey)
    }
  }
}
```

## 测试与质量

### 测试覆盖

- 提及触发和搜索逻辑
- 各种提供者的功能测试
- 自定义提供者的扩展测试
- 性能基准测试（大量提及项）
- 边界条件测试（特殊字符、空查询等）

## 常见问题 (FAQ)

### Q: 如何添加新的提及类型？
A: 实现 `MentionProvider` 接口并注册到 `MentionManager`。

### Q: 提及项的渲染可以自定义吗？
A: 可以，通过 `render` 方法返回自定义的 HTML 元素。

### Q: 提及支持键盘导航吗？
A: 支持，可以使用上下箭头导航，回车选择，ESC 取消。

### Q: 如何处理重复的提及？
A: 提供者应该返回唯一的 ID，系统会自动去重。

## 相关文件清单

```
src/mentions/
└── Mention.ts                   # 提及功能核心实现
```

## 变更记录 (Changelog)

### 2025-12-16 16:20:00
- ✨ 创建提及模块文档
- 📚 详细说明提及系统和提供者架构
- 🔗 记录编辑器集成和自定义扩展方法
- 📝 提供性能优化和最佳实践指南