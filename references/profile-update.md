# 阶段 5：Profile 自更新

本文件定义在每次开发任务完成后，如何将新发现的项目风格信息更新回 Project Profile，使其随项目演进而成长。

---

## 5.0 Profile 存储机制

### 存储路径

Profile 按项目隔离存储：
```
references/projects/{projectName}/profile.json
```

- `projectName`：项目根目录名称
- 每个项目独立维护自己的 Profile 文件
- 切换项目时自动加载对应项目的 Profile

### 加载流程

```
1. 获取当前项目名称（从工作目录提取）
2. 检查 references/projects/{projectName}/profile.json 是否存在
3. 存在 → 直接加载，跳过扫描
4. 不存在 → 执行完整扫描，创建 Profile 文件
```

---

## 5.1 触发条件

每次完成一个开发任务后，执行以下检查，判断是否需要更新 Profile：

| 发现内容 | 更新动作 |
|---------|---------|
| 用户修改了生成的代码风格 | 记录修改方向，更新命名/注释规范 |
| 本次使用了项目中新出现的设计模式 | 将该模式加入架构模式描述 |
| 发现了之前未识别的通用字段（如 `tenant_id`）| 加入 DB 通用字段列表 |
| 用户明确纠正了某个风格判断 | 更新对应风格规则，标记为"用户确认" |
| 新增了新的表/Model | 更新表结构感知结果 |
| 发现了新的错误处理模式 | 更新错误处理规则 |

---

## 5.2 更新规则

### 自动持久化更新

每次 Profile 更新后，**自动写入项目专属目录**，无需用户手动操作：

```
存储路径：references/projects/{projectName}/profile.json
```

更新时需同时更新以下字段：
- `lastUpdated`：更新时间戳
- `version`：版本号 +1

### 用户修改优先

如果用户对生成的代码做了修改，这是最直接的风格信号：
- 分析用户修改了什么（命名、注释、结构、错误处理等）
- 将修改方向更新到 Profile 对应字段
- 下次生成时直接采用更新后的风格

### 新发现追加，不覆盖

Profile 更新是**追加式**的，不会删除已有信息：
- 如发现新的函数命名前缀，追加到 `naming.function_prefix` 列表
- 如发现新的通用字段，追加到 `db.common_fields` 列表
- 已确认的规则标记为稳定，不会因单次发现而被推翻

### 置信度标记

```json
{
  "naming": {
    "function_prefix": {
      "value": ["get", "find", "create", "update", "delete", "batch"],
      "confidence": "high",       // high: 多次确认 / medium: 推断 / low: 单次发现
      "source": "用户确认",       // 来源：扫描推断 / 用户确认 / 用户修改
      "last_updated": "2024-01-15"
    }
  }
}
```

---

## 5.3 更新后的通知

Profile 更新后，在本次任务结束时简短告知用户（不打断，放在最后）：

```
📝 Profile 已更新：
- 发现新的函数命名前缀 `batch`，已加入风格规则
- 发现通用字段 `org_id`，已加入 DB 通用字段列表
下次任务将使用更新后的风格。
```

如果没有更新，则不输出此段。

---

## 5.4 重置 Profile

如果用户明确说"重新分析项目"或"项目架构有大改动"，则清空当前 Profile，重新执行 `references/project-sensing.md` 中的完整扫描流程。

触发短语示例：
- "项目重构了，重新分析一下"
- "换框架了，从头感知"
- "忘掉之前的风格，重新来"

---

## 5.5 项目隔离与跨会话持久化

### 自动项目隔离

Profile 按项目名称隔离存储，实现多项目支持：

```
references/projects/
├── my-app/
│   └── profile.json
├── another-project/
│   └── profile.json
└── backend-api/
    └── profile.json
```

**工作流程：**
1. 进入阶段 1（项目感知）时，首先获取当前项目名称
2. 检查 `references/projects/{projectName}/profile.json` 是否存在
3. 存在 → 直接加载，跳过扫描
4. 不存在 → 执行完整扫描，创建 Profile 文件

### 切换项目

当用户切换到不同项目目录时：
```
1. 获取新项目名称
2. 尝试加载 references/projects/{newProjectName}/profile.json
3. 已存在 → 复用缓存的 Profile
4. 不存在 → 执行扫描并创建新 Profile
```

**无需用户干预，自动实现项目隔离。**

### 跨会话复用

由于 Profile 文件持久化存储在 skill 目录中：
- 同一项目的新对话会话可直接复用
- 不同项目互不干扰
- Profile 随项目演进持续更新

**提示消息示例：**
```
📋 检测到项目 [my-app] 已有缓存的 Profile，直接复用。
如需重新扫描，请说"重新分析项目"。
```

### 手动导出（可选）

如需将 Profile 分享给团队成员，可手动导出：

```
用户请求 → "导出 Profile"
系统响应 → 提供 references/projects/{projectName}/profile.json 文件路径
```

团队其他成员可将该文件放入自己的 skill 目录对应位置。
