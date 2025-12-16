[根目录](../../CLAUDE.md) > [src](../) > **commands**

# Commands Module

## 模块职责

命令模块负责管理 Obsidian Copilot 的所有自定义命令功能，包括命令注册、执行、管理和用户界面。该模块提供了完整的命令生态系统，允许用户创建自定义命令、管理命令设置，并通过聊天界面执行各种操作。

## 入口与启动

### 核心文件

- **index.ts** - 命令模块主入口
  - 导出所有命令相关功能
  - 提供统一的命令访问接口
  - 管理命令生命周期

- **customCommandManager.ts** - 自定义命令管理器
  - 管理用户创建的自定义命令
  - 处理命令的存储和加载
  - 提供命令执行和验证功能

- **customCommandRegister.ts** - 命令注册器
  - 注册命令到 Obsidian 命令面板
  - 管理命令的元数据和配置
  - 处理命令的启用/禁用状态

### 核心组件

- **CustomCommandChatModal.tsx** - 自定义命令聊天模态框
  - 提供命令执行界面
  - 支持参数输入和预览
  - 集成聊天上下文

- **CustomCommandSettingsModal.tsx** - 自定义命令设置模态框
  - 管理命令配置
  - 提供命令编辑界面
  - 支持导入/导出功能

- **customCommandUtils.ts** - 命令工具函数
  - 提供命令处理的各种工具
  - XML 转义和安全处理
  - 命令验证和格式化

## 对外接口

### CustomCommandManager API

```typescript
export class CustomCommandManager {
  // 单例模式
  static getInstance(): CustomCommandManager

  // 命令管理
  addCommand(command: CustomCommand): Promise<void>
  removeCommand(commandId: string): Promise<void>
  updateCommand(commandId: string, updates: Partial<CustomCommand>): Promise<void>

  // 命令执行
  executeCommand(
    commandId: string,
    params?: Record<string, any>,
    context?: CommandContext
  ): Promise<CommandResult>

  // 命令查询
  getCommand(commandId: string): CustomCommand | undefined
  getAllCommands(): CustomCommand[]
  getEnabledCommands(): CustomCommand[]

  // 持久化
  saveCommands(): Promise<void>
  loadCommands(): Promise<void>
}
```

### 命令类型定义

```typescript
interface CustomCommand {
  id: string                    // 唯一标识符
  name: string                  // 显示名称
  description: string           // 描述信息
  command: string               // 命令内容（可以是提示词模板）
  type: CommandType            // 命令类型
  enabled: boolean             // 是否启用
  category?: string            // 命令分类
  icon?: string                // 图标标识
  shortcut?: string            // 快捷键
  parameters?: CommandParameter[]  // 参数定义
  metadata?: CommandMetadata   // 元数据
}

enum CommandType {
  PROMPT = 'prompt',           // 提示词命令
  ACTION = 'action',           // 操作命令
  QUERY = 'query',             // 查询命令
  CUSTOM = 'custom'            // 自定义命令
}

interface CommandParameter {
  name: string                 // 参数名
  type: ParameterType         // 参数类型
  required: boolean           // 是否必需
  description?: string        // 参数描述
  default?: any               // 默认值
  validation?: ValidationRule // 验证规则
}
```

### 命令执行流程

```typescript
// 1. 注册命令
const command: CustomCommand = {
  id: 'summarize-note',
  name: '总结笔记',
  description: '使用 AI 总结当前笔记',
  command: '请总结以下笔记内容：{{content}}',
  type: CommandType.PROMPT,
  parameters: [
    {
      name: 'content',
      type: ParameterType.TEXT,
      required: true,
      description: '要总结的内容'
    }
  ]
}

await commandManager.addCommand(command)

// 2. 执行命令
const result = await commandManager.executeCommand('summarize-note', {
  content: activeFile.content
})

// 3. 处理结果
if (result.success) {
  // 显示结果或执行后续操作
  showResult(result.output)
}
```

## 命令特性

### 1. 模板系统

命令支持模板变量，允许动态内容插入：

```typescript
// 支持的模板变量
const templateVars = {
  '{{content}}': 当前选中的文本或文件内容
  '{{title}}': 当前笔记标题
  '{{path}}': 当前文件路径
  '{{date}}': 当前日期
  '{{time}}': 当前时间
  '{{user}}': 当前用户名
  '{{project}}': 当前项目名
  // 自定义变量...
}
```

### 2. 参数系统

支持多种参数类型：

```typescript
enum ParameterType {
  TEXT = 'text',               // 文本输入
  NUMBER = 'number',           // 数字输入
  BOOLEAN = 'boolean',         // 布尔值
  SELECT = 'select',           // 下拉选择
  MULTI_SELECT = 'multiSelect', // 多选
  FILE = 'file',               // 文件选择
  FOLDER = 'folder',           // 文件夹选择
  DATE = 'date',               // 日期选择
  COLOR = 'color'              // 颜色选择
}
```

### 3. 命令分类

```typescript
const commandCategories = {
  'note': '笔记操作',
  'search': '搜索查询',
  'ai': 'AI 交互',
  'file': '文件管理',
  'project': '项目管理',
  'custom': '自定义'
}
```

### 4. 快捷键支持

```typescript
interface CommandShortcut {
  key: string                  // 按键
  modifiers?: Modifier[]       // 修饰键
  context?: ShortcutContext    // 上下文限制
}

enum Modifier {
  CTRL = 'ctrl',
  ALT = 'alt',
  SHIFT = 'shift',
  META = 'meta'  // Cmd on Mac, Win key on Windows
}
```

## 安全机制

### 1. XML 转义

防止 XML 注入攻击：

```typescript
// 在 customCommandUtils.xmlescape.ts 中
export function escapeXML(text: string): string {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;')
}
```

### 2. 命令验证

```typescript
function validateCommand(command: CustomCommand): ValidationResult {
  const errors: string[] = []

  // 验证必需字段
  if (!command.name) errors.push('命令名称不能为空')
  if (!command.command) errors.push('命令内容不能为空')

  // 验证参数
  command.parameters?.forEach(param => {
    if (param.required && !param.default) {
      errors.push(`必需参数 ${param.name} 缺少默认值`)
    }
  })

  return { valid: errors.length === 0, errors }
}
```

### 3. 权限控制

```typescript
interface CommandPermissions {
  canReadFiles: boolean        // 是否可以读取文件
  canWriteFiles: boolean       // 是否可以写入文件
  canExecuteSystem: boolean    // 是否可以执行系统命令
  canAccessNetwork: boolean    // 是否可以访问网络
}
```

## 数据持久化

### 存储位置

```
[Obsidian Vault]/
└── .obsidian/
    └── plugins/
        └── copilot/
            └── commands/
                ├── customCommands.json    # 用户自定义命令
                ├── commandHistory.json     # 命令执行历史
                └── commandSettings.json    # 命令相关设置
```

### 数据格式

```json
{
  "version": "1.0.0",
  "commands": [
    {
      "id": "summarize-note",
      "name": "总结笔记",
      "description": "使用 AI 总结当前笔记",
      "command": "请总结以下笔记内容：{{content}}",
      "type": "prompt",
      "enabled": true,
      "category": "note",
      "icon": "file-text",
      "parameters": [
        {
          "name": "content",
          "type": "text",
          "required": true,
          "description": "要总结的内容"
        }
      ],
      "metadata": {
        "createdAt": "2025-12-07T14:00:00Z",
        "updatedAt": "2025-12-07T14:00:00Z",
        "usageCount": 0
      }
    }
  ]
}
```

## 内置命令

### 1. 笔记操作命令

- `summarize-note` - 总结当前笔记
- `extract-keywords` - 提取关键词
- `generate-tags` - 生成标签
- `format-note` - 格式化笔记

### 2. 搜索命令

- `search-notes` - 搜索笔记
- `search-by-tag` - 按标签搜索
- `search-by-date` - 按日期搜索

### 3. AI 交互命令

- `ask-ai` - 向 AI 提问
- `translate` - 翻译文本
- `explain-code` - 解释代码
- `generate-code` - 生成代码

## 测试与质量

### 测试文件

- `customCommandUtils.test.ts` - 工具函数测试
- `customCommandUtils.xmlescape.test.ts` - XML 转义测试
- `allTools.validation.test.ts` - 命令验证测试

### 测试覆盖

- 命令创建和管理
- 参数验证和类型检查
- 模板渲染和变量替换
- XML 转义和安全处理
- 命令执行和错误处理

## 常见问题 (FAQ)

### Q: 如何创建自定义命令？
A: 通过设置界面或编程方式：
```typescript
const command = await CustomCommandManager.getInstance().addCommand({
  name: '我的命令',
  command: '处理：{{content}}',
  type: CommandType.PROMPT
})
```

### Q: 命令可以访问系统资源吗？
A: 默认情况下，命令在沙箱环境中运行，需要明确授予权限才能访问文件系统或网络。

### Q: 如何调试命令？
A: 使用开发者工具查看控制台输出，或启用命令的调试模式查看详细日志。

### Q: 命令支持异步操作吗？
A: 是的，命令可以包含异步操作，如 API 调用或文件读写。

## 相关文件清单

```
src/commands/
├── index.ts                       # 模块入口
├── constants.ts                   # 常量定义
├── contextMenu.ts                 # 上下文菜单处理
├── CustomCommandChatModal.tsx     # 命令执行界面
├── customCommandManager.ts        # 命令管理器
├── customCommandRegister.ts       # 命令注册器
├── CustomCommandSettingsModal.tsx # 命令设置界面
├── customCommandUtils.ts          # 工具函数
├── customCommandUtils.test.ts     # 工具函数测试
├── customCommandUtils.xmlescape.test.ts # XML 转义测试
├── migrator.ts                    # 数据迁移
├── state.ts                       # 状态管理
└── type.ts                        # 类型定义
```

## 变更记录 (Changelog)

### 2025-12-16 16:20:00
- ✨ 创建命令模块文档
- 📚 详细说明自定义命令系统架构
- 🔗 记录命令类型、参数系统和安全机制
- 📝 提供完整的 API 文档和使用示例