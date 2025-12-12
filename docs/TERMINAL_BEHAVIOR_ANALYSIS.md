# 终端Tab补全行为深度分析

## 一、标准终端（bash/zsh）的Tab补全行为

### 1. **单次Tab行为**
```bash
$ cat pa[Tab]
# 场景A：唯一匹配
$ cat package.json  # 立即补全，光标在末尾

# 场景B：多个匹配
$ cat p[Tab]
# 无反应，需要按第二次Tab
```

### 2. **双击Tab行为**
```bash
$ cat p[Tab][Tab]
package.json  package-lock.json  public/
$ cat p█  # 光标保持原位，显示所有可能选项
```

### 3. **公共前缀补全**
```bash
$ cat pac[Tab]
$ cat package  # 补全到公共前缀 "package"
$ cat package[Tab][Tab]
package.json  package-lock.json
```

### 4. **路径补全特性**
```bash
# 自动添加斜杠
$ cd src[Tab]
$ cd src/  # 目录自动加斜杠

# 连续补全
$ cd src/[Tab]
components/  hooks/  utils/  # 显示目录内容

# 隐藏文件
$ ls .[Tab][Tab]  # 需要以.开头才显示隐藏文件
.env  .gitignore  .npmrc
```

### 5. **上下文感知**
```bash
# 命令后的第一个参数
$ git [Tab][Tab]
add  commit  push  pull  status  # 显示git子命令

# 不同命令的不同行为
$ cd [Tab]  # 只显示目录
$ cat [Tab]  # 显示文件
$ chmod [Tab]  # 显示可执行文件
```

### 6. **特殊字符处理**
```bash
# 空格转义
$ cat My\ Documents/[Tab]
$ cat "My Documents/"[Tab]

# 通配符
$ cat *.js[Tab]  # 展开所有.js文件
$ cat test*[Tab]  # 展开所有test开头的文件
```

## 二、现代终端增强功能（fish/zsh with plugins）

### 1. **实时建议（灰色文本）**
```bash
$ cat p
$ cat package.json  # 灰色显示建议
# 右箭头接受，Tab完成
```

### 2. **智能历史**
```bash
$ npm run
$ npm run dev  # 基于历史的建议
```

### 3. **模糊匹配**
```bash
$ cat pjs[Tab]
$ cat package.json  # 模糊匹配p...j...s
```

### 4. **语法高亮**
```bash
$ cat existing.txt  # 绿色，文件存在
$ cat missing.txt   # 红色，文件不存在
```

## 三、我们当前实现的差距

### ❌ 缺失的核心功能

1. **双击Tab显示所有选项**
   - 当前：第一次Tab就显示菜单
   - 应该：第一次Tab尝试补全，第二次显示选项

2. **公共前缀补全**
   - 当前：直接显示菜单
   - 应该：先补全到公共前缀

3. **无需显式触发**
   - 当前：必须Tab才触发
   - 应该：输入时就准备好建议

4. **连续路径补全**
   - 当前：补全后停止
   - 应该：目录补全后继续等待下一次Tab

5. **通配符展开**
   - 当前：不支持
   - 应该：*.js展开为所有js文件

### ✅ 已有但需优化

1. **上下文检测**
   - 有基础实现，但不够智能

2. **文件类型区分**
   - 有图标，但行为未区分

3. **即时响应**
   - 已实现，但交互模式不对

## 四、理想的Tab补全交互流程

### 阶段1：输入时（无Tab）
```
用户输入: cat pa
内部状态: 准备suggestions ["package.json", "package-lock.json"]
显示: 无变化（或灰色提示）
```

### 阶段2：第一次Tab
```
用户操作: [Tab]
判断逻辑:
  - 唯一匹配 → 直接补全
  - 多个匹配但有公共前缀 → 补全到公共前缀
  - 多个匹配无公共前缀 → 蜂鸣/无反应
```

### 阶段3：第二次Tab
```
用户操作: [Tab][Tab]
行为: 显示所有可能的补全选项
格式: 水平排列，按列对齐
```

### 阶段4：选择
```
继续输入: 缩小范围
方向键: 选择（可选）
Tab: 循环选择（可选）
Enter: 确认选择
```

## 五、改进建议

### 必须实现（核心体验）
1. **双Tab机制** - 第一次补全，第二次显示
2. **公共前缀** - 智能补全到最长公共前缀
3. **连续补全** - 目录后继续补全
4. **更智能的上下文** - 命令感知

### 应该实现（提升体验）
1. **灰色建议** - 输入时显示
2. **历史感知** - 基于使用频率排序
3. **模糊匹配** - 支持简写
4. **路径缓存** - 提升性能

### 可以实现（锦上添花）
1. **语法高亮** - 文件存在性
2. **自定义补全** - 用户定义规则
3. **异步加载** - 大目录优化
4. **补全预览** - 显示文件内容预览

## 六、技术实现要点

### Tab计数器
```typescript
interface TabState {
  lastTabTime: number
  tabCount: number
  lastContext: string
}

// 双击检测：300ms内的第二次Tab
if (Date.now() - lastTabTime < 300) {
  tabCount++
} else {
  tabCount = 1
}
```

### 公共前缀算法
```typescript
function findCommonPrefix(strings: string[]): string {
  if (!strings.length) return ''
  return strings.reduce((prefix, str) => {
    while (!str.startsWith(prefix)) {
      prefix = prefix.slice(0, -1)
    }
    return prefix
  })
}
```

### 智能补全决策
```typescript
function handleTab(suggestions: string[]): Action {
  if (suggestions.length === 0) {
    return 'beep'
  }
  if (suggestions.length === 1) {
    return 'complete'
  }
  const prefix = findCommonPrefix(suggestions)
  if (prefix.length > currentWord.length) {
    return 'complete-to-prefix'
  }
  if (isSecondTab()) {
    return 'show-menu'
  }
  return 'beep'
}
```

## 七、优先级路线图

### Phase 1: 核心终端行为（必须）
- [ ] 双Tab机制
- [ ] 公共前缀补全
- [ ] 连续路径补全
- [ ] 更准确的上下文检测

### Phase 2: 现代增强（应该）
- [ ] 实时灰色建议
- [ ] 历史/频率排序
- [ ] 模糊匹配支持

### Phase 3: 高级功能（可选）
- [ ] 通配符展开
- [ ] 自定义补全规则
- [ ] 异步性能优化