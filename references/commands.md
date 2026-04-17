# 快捷命令支持

本文件定义用户可用的快捷命令及其功能。

---

## 1. 快捷命令列表

用户可通过自然语言短语触发特定功能：

| 命令短语 | 功能 | 说明 |
|---------|------|------|
| **刷新 Profile** | 重新扫描项目 | 删除缓存，执行完整扫描 |
| **重新分析项目** | 重新扫描项目 | 同上 |
| **项目重构了** | 重新扫描项目 | 适用于架构大改后 |
| **重新感知项目** | 重新扫描项目 | 同上 |
| **清除缓存** | 删除 Profile 文件 | 只删除，不重新扫描 |
| **重置 Profile** | 重新扫描项目 | 同"刷新 Profile" |
| **查看 Profile** | 显示 Profile 内容 | 展示当前项目风格指纹 |
| **显示 Profile** | 显示 Profile 内容 | 同上 |
| **导出 Profile** | 导出 JSON 文件 | 提供文件路径供分享 |
| **对比文件风格** | 分析风格差异 | 对比指定文件与 Profile |
| **检查风格一致性** | 分析风格差异 | 同上 |

---

## 2. 命令执行流程

### 刷新 Profile

```
用户说："刷新 Profile"

执行步骤：
1. 获取项目名称
2. 删除 references/projects/{projectName}/profile.json
3. 读取 references/project-sensing.md
4. 执行完整项目扫描
5. 创建新 Profile 文件
6. 输出刷新结果摘要

输出格式：
🔄 正在重新分析项目...
✅ Profile 已刷新，发现以下变化：
- [框架版本变更]
- [新增/移除依赖]
- [风格变化（如有）]

Profile 已缓存，开始开发。
```

### 查看 Profile

```
用户说："查看 Profile"

执行步骤：
1. 获取项目名称
2. 检查 Profile 是否存在
   - 不存在 → 提示用户先扫描
   - 存在 → 读取并格式化输出

输出格式：
📋 当前项目 Profile：

**项目信息**
- 名称：{projectName}
- 路径：{projectPath}
- 更新时间：{lastUpdated}
- 状态：{是否过期}

**技术栈**
- 语言：{language}
- 框架：{framework} @ {version}
- ORM：{orm}

**代码风格**
- 变量命名：{variable naming style}
- 函数前缀：{function_prefix list}
- 注释风格：{comment style}，{language}
- 错误处理：{error_handling pattern}

**数据库**
- 主键类型：{pk_type}
- 软删除：{soft_delete field}
- 通用字段：{common_fields list}

如需刷新，请说 "刷新 Profile"。
```

### 导出 Profile

```
用户说："导出 Profile"

执行步骤：
1. 获取项目名称
2. 检查 Profile 是否存在
3. 提供 Profile 文件路径

输出格式：
📁 Profile 文件路径：
references/projects/{projectName}/profile.json

您可以将此文件分享给团队成员，放入他们的 skill 目录对应位置即可复用。
```

### 对比文件风格

```
用户说："对比文件风格 src/services/user.service.ts"

执行步骤：
1. 加载当前 Profile
2. 读取指定文件
3. 提取文件风格指纹
4. 对比差异并输出

输出格式：
🔍 文件风格对比分析：src/services/user.service.ts

**与 Profile 一致**
- ✅ 变量命名：camelCase
- ✅ 函数前缀：使用 find, create, update
- ✅ 注释语言：中文

**与 Profile 不一致**
- ⚠️ 注释风格：文件使用行内注释，Profile 为 JSDoc
- ⚠️ 日志调用：使用 console.log，Profile 为 logger.info

建议：保持与项目整体风格一致，或在 Profile 自更新时采纳此风格。
```

### 清除缓存

```
用户说："清除缓存"

执行步骤：
1. 获取项目名称
2. 删除 Profile 文件（如存在）
3. 确认删除

输出格式：
🗑️ 已清除项目 [{projectName}] 的 Profile 缓存。
下次开发时将重新扫描项目。
如需立即扫描，请说 "重新分析项目"。
```

---

## 3. 命令识别逻辑

### 匹配规则

使用关键词匹配识别命令：

```javascript
const commands = {
  refresh: ['刷新', '重新分析', '重新感知', '项目重构', '重置'],
  view: ['查看', '显示', '看看', '打印'],
  export: ['导出', '输出', '分享'],
  compare: ['对比', '比较', '检查一致性', '分析差异'],
  clear: ['清除', '删除', '清空', '移除']
};

const detectCommand = (userInput) => {
  const input = userInput.toLowerCase();

  for (const [cmd, keywords] of Object.entries(commands)) {
    if (keywords.some(kw => input.includes(kw)) && input.includes('profile') || input.includes('项目')) {
      return cmd;
    }
  }
  return null;
};
```

### 参数提取

部分命令需要额外参数：

| 命令 | 参数 | 提取方式 |
|-----|------|---------|
| 对比文件风格 | 文件路径 | 从输入中提取路径字符串 |
| 查看 Profile | 可选项目名 | 默认当前项目 |

---

## 4. 命令响应模板

### Profile 不存在时的统一响应

```
⚠️ 当前项目尚未建立 Profile。

请先执行项目扫描，方式：
1. 说 "重新分析项目" 触发扫描
2. 或直接提出开发需求，我会自动扫描

扫描完成后可使用快捷命令管理 Profile。
```

### 执行中的状态提示

```
🔄 正在{执行动作}...
```

### 成功后的确认

```
✅ {动作}完成。
{结果摘要}
```

---

## 5. 与开发流程的集成

### 开发任务开始前

自动检查 Profile 状态：

```
检测 Profile →
  ├─ 不存在 → 提示并执行扫描
  ├─ 已过期 → 提示用户，询问是否刷新
  └─ 正常 → 加载并继续
```

### 开发任务完成后

自动更新 Profile（如有新发现）：

```
完成任务 → 检查是否有风格更新 → 更新 Profile → 通知用户
```

---

## 6. 批量操作支持

### 多项目操作

```
用户说："查看所有项目的 Profile"

执行：
1. 扫描 references/projects/ 目录
2. 列出所有项目及其 Profile 状态

输出：
📋 已缓存的项目列表：

| 项目名称 | 技术栈 | 更新时间 | 状态 |
|---------|-------|---------|------|
| my-app | NestJS + TypeScript | 2024-01-15 | 有效 |
| admin-panel | Vue 3 + Vite | 2024-01-10 | 已过期 |
| backend-api | Go + Gin | 2024-01-08 | 有效 |

共 3 个项目已缓存 Profile。
```

### 批量清除

```
用户说："清除所有缓存"

执行：
1. 删除 references/projects/ 下所有 profile.json
2. 确认删除数量

输出：
🗑️ 已清除 3 个项目的 Profile 缓存。
下次切换项目时将重新扫描。
```