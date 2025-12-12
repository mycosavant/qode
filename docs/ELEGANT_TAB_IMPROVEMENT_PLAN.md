# 优雅的Tab补全改进计划

## 一、当前架构分析

### 核心数据结构（保持不变）
```typescript
interface UnifiedSuggestion        // ✅ 完美，不需要改动
interface CompletionContext        // ✅ 完美，不需要改动  
```

### 状态管理（需要增强）
```typescript
// 当前状态
const [suggestions, setSuggestions]       // ✅ 保持
const [selectedIndex, setSelectedIndex]   // ✅ 保持
const [isActive, setIsActive]             // ✅ 保持
const lastTabContext = useRef()           // ✅ 保持

// 需要添加的状态（最小化）
const tabState = useRef<TabState>()       // 🆕 Tab按键状态
```

### 关键函数（大部分保持）
- `getWordAtCursor()` ✅ 完美，不改
- `generateCommandSuggestions()` ✅ 完美，不改
- `generateAgentSuggestions()` ✅ 完美，不改
- `generateFileSuggestions()` ✅ 完美，不改
- `generateSuggestions()` ✅ 完美，不改
- Tab处理逻辑 ❌ 需要重构

## 二、最小化改动方案

### 1. 添加Tab状态跟踪（新增数据结构）
```typescript
// 添加到文件顶部，与其他interface并列
interface TabState {
  lastTabTime: number
  consecutiveTabCount: number
  lastPrefix: string
  lastSuggestions: UnifiedSuggestion[]
}
```

### 2. 添加公共前缀计算（纯函数，无副作用）
```typescript
// 添加为独立的utility函数
const findCommonPrefix = (suggestions: UnifiedSuggestion[]): string => {
  if (suggestions.length === 0) return ''
  if (suggestions.length === 1) return suggestions[0].value
  
  const values = suggestions.map(s => s.value)
  let prefix = values[0]
  
  for (let i = 1; i < values.length; i++) {
    while (prefix && !values[i].startsWith(prefix)) {
      prefix = prefix.slice(0, -1)
    }
    if (!prefix) break
  }
  
  return prefix
}
```

### 3. 重构Tab处理逻辑（核心改动）

将现有的Tab处理（185-237行）替换为新的智能处理：

```typescript
// Handle Tab key - Terminal-compliant behavior
useInput(async (_, key) => {
  if (!key.tab || key.shift) return false
  
  const context = getWordAtCursor()
  if (!context) return false
  
  const now = Date.now()
  const isDoubleTab = tabState.current && 
    (now - tabState.current.lastTabTime) < 500 &&
    tabState.current.lastPrefix === context.prefix
  
  // 如果菜单已显示，Tab选择下一个
  if (isActive && suggestions.length > 0) {
    // 保持原有逻辑
    const selected = suggestions[selectedIndex]
    // ... 完成逻辑
    return true
  }
  
  // 生成建议（只在需要时）
  let currentSuggestions = suggestions
  if (!isDoubleTab || suggestions.length === 0) {
    currentSuggestions = await generateSuggestions(context)
  }
  
  // 决策树 - 完全符合终端行为
  if (currentSuggestions.length === 0) {
    // 无匹配：蜂鸣
    return false
    
  } else if (currentSuggestions.length === 1) {
    // 唯一匹配：立即完成
    completeWith(currentSuggestions[0], context)
    resetTabState()
    return true
    
  } else {
    // 多个匹配
    const commonPrefix = findCommonPrefix(currentSuggestions)
    
    if (commonPrefix.length > context.prefix.length) {
      // 可以补全到公共前缀
      partialComplete(commonPrefix, context)
      updateTabState(now, context.prefix, currentSuggestions)
      return true
      
    } else if (isDoubleTab) {
      // 第二次Tab：显示菜单
      setSuggestions(currentSuggestions)
      setIsActive(true)
      setSelectedIndex(0)
      return true
      
    } else {
      // 第一次Tab但无法补全：记录状态
      updateTabState(now, context.prefix, currentSuggestions)
      return false // 蜂鸣
    }
  }
})
```

### 4. 添加辅助函数（与现有风格一致）

```typescript
// 完成补全
const completeWith = useCallback((suggestion: UnifiedSuggestion, context: CompletionContext) => {
  const completion = context.type === 'command' ? `/${suggestion.value} ` :
                    context.type === 'agent' ? `@${suggestion.value} ` :
                    suggestion.value
  
  const newInput = input.slice(0, context.startPos) + completion + input.slice(context.endPos)
  onInputChange(newInput)
  setCursorOffset(context.startPos + completion.length)
}, [input, onInputChange, setCursorOffset])

// 部分补全
const partialComplete = useCallback((prefix: string, context: CompletionContext) => {
  const newInput = input.slice(0, context.startPos) + prefix + input.slice(context.endPos)
  onInputChange(newInput)
  setCursorOffset(context.startPos + prefix.length)
}, [input, onInputChange, setCursorOffset])

// Tab状态管理
const updateTabState = useCallback((time: number, prefix: string, suggestions: UnifiedSuggestion[]) => {
  tabState.current = {
    lastTabTime: time,
    consecutiveTabCount: (tabState.current?.consecutiveTabCount || 0) + 1,
    lastPrefix: prefix,
    lastSuggestions: suggestions
  }
}, [])

const resetTabState = useCallback(() => {
  tabState.current = null
}, [])
```

## 三、实施步骤

### Phase 1: 基础设施（不影响现有功能）
1. 添加 `TabState` interface
2. 添加 `tabState` useRef
3. 添加 `findCommonPrefix` 函数
4. 添加辅助函数

### Phase 2: 核心逻辑替换（原子操作）
1. 备份现有Tab处理代码
2. 替换为新的决策树逻辑
3. 测试所有场景

### Phase 3: 细节优化
1. 调整超时时间（500ms vs 300ms）
2. 优化菜单显示格式
3. 添加蜂鸣反馈（可选）

## 四、影响评估

### 不变的部分（90%）
- 所有数据结构
- 所有生成函数
- 箭头键处理
- Effect清理逻辑
- 与PromptInput的接口

### 改变的部分（10%）
- Tab按键处理逻辑
- 新增4个小函数
- 新增1个状态ref

### 风险评估
- **低风险**：改动集中在一处
- **可回滚**：逻辑独立，易于回滚
- **向后兼容**：接口不变

## 五、测试场景

### 场景1: 多个文件补全
```bash
# 文件: package.json, package-lock.json
输入: p[Tab]
期望: 补全到 "package"
输入: package[Tab][Tab]
期望: 显示菜单
```

### 场景2: 唯一匹配
```bash
输入: READ[Tab]
期望: 补全到 "README.md"
```

### 场景3: 连续补全
```bash
输入: src/[Tab]
期望: 可以继续Tab补全
```

## 六、代码风格指南

### 保持一致性
- 使用 `useCallback` 包装所有函数
- 使用 `as const` 断言类型
- 保持简洁的注释风格

### 命名规范
- 动词开头：`completeWith`, `updateTabState`
- 布尔值：`isDoubleTab`, `isActive`
- 常量大写：`TAB_TIMEOUT`

### 错误处理
- 保持静默失败（符合现有风格）
- 使用 try-catch 包装文件操作

## 七、预期效果

### Before
```
cat p[Tab]
▸ package.json     # 立即显示菜单 ❌
  package-lock.json
```

### After  
```
cat p[Tab]
cat package        # 补全公共前缀 ✅
cat package[Tab][Tab]
package.json  package-lock.json  # 双Tab显示 ✅
```

## 八、总结

这个方案：
1. **最小化改动** - 90%代码不变
2. **原子操作** - 可以一次性替换
3. **风格一致** - 像原生代码
4. **100%终端兼容** - 完全匹配bash行为

准备好实施了吗？