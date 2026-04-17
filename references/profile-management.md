# Profile 过期与刷新管理

本文件定义 Profile 缓存的生命周期管理策略。

---

## 1. Profile 过期策略

### 默认有效期

Profile 缓存默认有效期：**7 天**

过期判断逻辑：
```javascript
const isExpired = (profile) => {
  const lastUpdated = new Date(profile.lastUpdated)
  const now = new Date()
  const daysDiff = (now - lastUpdated) / (1000 * 60 * 60 * 24)
  return daysDiff > 7
}
```

### 自动过期触发条件

以下情况 Profile 自动标记为过期，需重新扫描：

| 触发条件 | 检测方式 | 过期范围 |
|---------|---------|---------|
| 配置文件变更 | package.json / pyproject.toml 等修改时间晚于 lastUpdated | 全部过期 |
| 核心依赖版本变更 | 主要框架版本号变化（如 next 14 → 15） | 依赖信息过期 |
| 用户手动触发 | 用户说"刷新 Profile" / "重新分析" | 全部过期 |
| Profile 版本不兼容 | skill 更新后 profile.version 不匹配 | 需迁移或重扫 |

### 过期提示消息

```
⏰ 检测到 Profile 缓存已过期（超过 7 天），建议刷新以获取最新项目风格。
如需刷新，请说 "刷新 Profile" 或 "重新分析项目"。
```

---

## 2. Profile 加载流程（含过期检查）

```
[开始]
   │
   ▼
获取项目名称
   │
   ▼
检查 Profile 文件是否存在
   │
   ├─ 不存在 → 执行完整扫描 → 创建 Profile → [结束]
   │
   └─ 存在 → 读取 Profile
              │
              ▼
           检查版本兼容性
              │
              ├─ 版本不兼容 → 提示用户 → 执行扫描 → 更新 Profile → [结束]
              │
              └─ 版本兼容
                    │
                    ▼
                 检查是否过期
                    │
                    ├─ 已过期 → 提示用户
                    │           │
                    │           ├─ 用户确认刷新 → 执行扫描 → 更新 Profile
                    │           │
                    │           └─ 用户继续使用 → 加载 Profile（标记为可能过时）
                    │
                    └─ 未过期 → 直接加载 → [结束]
```

---

## 3. 增量更新策略

当检测到局部变更时，只更新受影响的部分：

### 依赖变更

```
检测到 package.json 变更：
→ 只更新 dependencies 字段
→ 检查是否有新框架需要 Context7 查询
→ 不影响已有风格指纹
```

### 新增文件

```
检测到新增文件：
→ 不影响现有 Profile
→ 如有用户修改，在 Profile 自更新阶段补充
```

### 关键文件修改

```
检测到关键文件修改（如 .eslintrc、tsconfig.json）：
→ 重新提取相关风格指纹
→ 更新 Profile 对应字段
```

---

## 4. Profile 版本管理

### 版本号规则

Profile 结构中的 `version` 字段：

```json
{
  "version": 2,
  "profileVersion": "1.0.0",
  ...
}
```

| 版本类型 | 说明 | 变更时处理 |
|---------|------|-----------|
| `version` | Profile 结构版本（整数） | 不兼容时需要重新扫描 |
| `profileVersion` | Skill 版本（语义化） | 小版本变更可自动迁移 |

### 版本迁移

当 skill 更新后，检测到 Profile 版本不兼容：

```javascript
const CURRENT_PROFILE_VERSION = 2;

const migrateProfile = (oldProfile) => {
  if (oldProfile.version < CURRENT_PROFILE_VERSION) {
    // 尝试迁移
    if (canMigrate(oldProfile.version, CURRENT_PROFILE_VERSION)) {
      return doMigrate(oldProfile);
    }
    // 迁移失败，需要重新扫描
    return null;
  }
  return oldProfile;
};
```

### 迁移提示

```
📋 检测到 Profile 版本已更新（v1 → v2），正在迁移...

迁移成功！新增字段：dependencies
如需完整重新扫描，请说 "刷新 Profile"。
```

---

## 5. 强制刷新触发词

用户可通过以下短语触发强制重新扫描：

| 触发词 | 动作 |
|-------|------|
| "刷新 Profile" | 删除缓存，重新扫描 |
| "重新分析项目" | 删除缓存，重新扫描 |
| "项目重构了" | 删除缓存，重新扫描 |
| "重新感知项目" | 删除缓存，重新扫描 |
| "清除缓存" | 删除 Profile 文件 |
| "重置 Profile" | 删除缓存，重新扫描 |

### 刷新流程

```
1. 删除 references/projects/{projectName}/profile.json
2. 执行完整项目扫描
3. 创建新的 Profile 文件
4. 通知用户
```

### 刷新确认消息

```
🔄 正在重新分析项目...
✅ Profile 已刷新，发现以下更新：
- 框架版本：NestJS 10.0.0 → 11.0.0
- 新增依赖：@nestjs/microservices
- 代码风格：无变化

Profile 已缓存，开始开发。
```

---

## 6. Profile 状态标识

在 Profile 中记录状态信息：

```json
{
  "projectName": "my-app",
  "projectPath": "/Users/xxx/projects/my-app",
  "lastUpdated": "2024-01-15T10:30:00Z",
  "version": 2,
  "status": {
    "isExpired": false,
    "lastCheck": "2024-01-16T08:00:00Z",
    "configChanges": [],
    "needsRefresh": false
  },
  "profile": {
    ...
  }
}
```

| 字段 | 说明 |
|-----|------|
| `isExpired` | 是否已过期 |
| `lastCheck` | 上次检查时间 |
| `configChanges` | 检测到的配置变更列表 |
| `needsRefresh` | 是否需要刷新 |
