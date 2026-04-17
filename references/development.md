# 阶段 2：开发实施

本文件定义代码生成的核心规则。**所有生成的代码必须严格遵循 Project Profile 中记录的风格指纹。**

---

## 2.1 开发前：需求拆解

在写第一行代码之前，先做以下分析并输出给用户确认：

### 任务拆解模板

```
## 开发计划

**任务类型：** 新功能 / BUG修复 / 存量改造 / 数据库变更

**影响范围：**
- 文件变更：[列出需要新增/修改的文件]
- 数据库变更：[是/否，如是则列出表/字段]
- 接口变更：[是/否，如是则列出接口路径和方法]
- 依赖变更：[是/否，如是则列出新增依赖]

**开发步骤：**
1. [步骤1]
2. [步骤2]
...

**风险点：**
- [可能的边界情况或已知风险]

确认后开始开发，还是需要调整计划？
```

---

## 2.2 代码生成规则

### 黄金规则：风格一致性优先于"最佳实践"

如果项目现有代码和通用最佳实践冲突，**优先遵循项目现有风格**。例如：
- 项目用 `callback` 而非 `async/await` → 继续用 callback
- 项目注释是中文 → 新代码也用中文注释
- 项目用 `any` 类型 → 不强行加类型（但可以在 Review 阶段建议改进）

### 代码生成检查清单

生成每个文件后，自我检查：

- [ ] 命名规范与 Profile 一致（函数名前缀、变量名大小写）
- [ ] 注释风格与 Profile 一致（JSDoc/docstring/Javadoc，中/英文）
- [ ] 错误处理方式与项目一致
- [ ] 返回值格式与项目一致
- [ ] 日志调用方式与项目一致
- [ ] 导入顺序与项目一致
- [ ] 分页参数/格式与项目一致
- [ ] 数据库字段命名与现有表一致

### 新功能开发模板

按项目架构层次生成，从下到上（DB → Service → Controller）：

```
1. 数据库层（如需）
   - Migration 脚本（严格遵循项目 Migration 框架语法）
   - ORM Model / Entity 定义

2. Repository / DAO 层
   - 基础 CRUD 方法
   - 复杂查询（参考项目中已有的 QueryBuilder 或 SQL 风格）

3. Service 层
   - 业务逻辑
   - 事务处理（参考项目事务使用方式）
   - 调用其他 Service 的依赖注入

4. Controller / Router 层
   - 路由定义（路径命名规范与项目一致）
   - 入参校验（项目用的校验库：class-validator / joi / zod / marshmallow 等）
   - 权限校验（参考项目现有的鉴权装饰器或中间件）
   - 响应格式包装
```

### BUG 修复输出格式

BUG 修复类任务，**优先输出 diff 格式**，最小化改动：

```diff
文件：src/services/user.service.ts
行号：第 87 行

- const users = await this.userRepo.find({ where: { status: 1 } });
+ const users = await this.userRepo.find({
+   where: { status: 1, deletedAt: IsNull() }
+ });

修复说明：原代码未过滤软删除记录，导致已删除用户仍被查询返回。
```

如果涉及多个文件，每个文件单独一个 diff 块，注明文件路径。

---

## 2.3 多模态输入处理

### 输入：截图 / 原型图

1. 描述识别到的 UI 元素（表单字段、按钮、列表列等）
2. 推断数据结构（字段名、类型、校验规则）
3. 生成对应接口（遵循项目 RESTful / GraphQL 风格）
4. 如有前端框架，同步生成前端组件骨架

**示例推断过程：**
```
截图识别：用户列表页面，包含 姓名/手机号/状态/创建时间/操作列
↓
推断接口：GET /users?page=1&pageSize=20&status=&keyword=
↓
推断返回：{ code: 0, data: { list: User[], total: number }, message: "success" }
↓
生成：Controller + Service + 类型定义
```

### 输入：Postman Collection / OpenAPI YAML

1. 解析接口定义（路径、方法、入参、出参）
2. 以接口为入口，逆向生成实现代码
3. 补全缺失的校验逻辑、错误处理
4. 生成对应的 Service / Repository 骨架

### 输入：错误日志 / Stack Trace

1. 定位错误发生的文件和行号
2. 分析根因（类型错误 / 空指针 / 业务逻辑 / 数据库查询等）
3. 生成最小修复方案（diff 格式）
4. 说明为什么这样修复，以及是否有隐藏的同类问题

---

## 2.4 变更影响图

每次开发完成后，输出变更影响图，格式如下：

```
## 变更影响分析

### 本次变更文件
- [新增] src/modules/order/order.controller.ts
- [新增] src/modules/order/order.service.ts
- [新增] src/modules/order/order.repository.ts
- [修改] src/modules/user/user.service.ts（新增 getUserOrders 方法）
- [新增] src/migrations/1700000000000-AddOrderTable.ts

### 调用链影响
POST /orders
  └─ OrderController.create()
       └─ OrderService.createOrder()
            ├─ UserService.getUserById()   ← 依赖现有方法，无变更
            └─ OrderRepository.save()

### 潜在影响点
- UserService 被修改，现有调用 UserService 的地方需验证不受影响
- 新增 orders 表，需确认数据库连接配置有建表权限

### 测试建议重点
- [ ] 创建订单 - 正常流程
- [ ] 创建订单 - 用户不存在
- [ ] 创建订单 - 并发创建（是否有重复提交问题）
```

---

## 2.5 新技术栈文档获取（Context7 MCP）

### 触发条件

当涉及以下情况时，**必须调用 Context7 MCP 服务**获取最新官方文档和代码示例：

| 场景 | 示例 |
|-----|-----|
| **新版本框架/库** | Next.js 15、React 19、NestJS 11、Vue 3.5 等 |
| **首次使用的库** | 项目中未出现过的新依赖 |
| **API 变更的库** | 库的新版本有 breaking changes |
| **不熟悉的语法** | 新的 TypeScript 特性、新的 ORM API 等 |

### 调用流程

```
1. 识别技术栈名称和版本
   → 从 package.json / pyproject.toml / go.mod 等获取版本信息

2. 调用 Context7 MCP resolve-library-id
   → mcp__context7__resolve-library-id(libraryName, query)
   → 获取 Context7 兼容的 libraryId

3. 调用 Context7 MCP query-docs
   → mcp__context7__query-docs(libraryId, query)
   → 查询具体的 API 用法、配置方式、代码示例

4. 基于返回的官方文档生成代码
   → 确保使用最新版本的正确语法
   → 避免使用已废弃的 API
```

### 常见技术栈的 Context7 Library ID

| 技术栈 | Library ID 格式 |
|-------|----------------|
| Next.js | `/vercel/next.js` 或 `/vercel/next.js/v15.x` |
| React | `/facebook/react` |
| NestJS | `/nestjs/docs` |
| Vue | `/vuejs/docs` |
| Prisma | `/prisma/docs` |
| Express | `/expressjs/express` |
| Django | `/django/docs` |
| FastAPI | `/fastapi/fastapi` |
| Spring Boot | `/spring-projects/spring-boot` |
| Tailwind CSS | `/tailwindlabs/tailwindcss` |

### 查询示例

```
# 查询 Next.js 15 的 App Router 用法
mcp__context7__query-docs("/vercel/next.js", "App Router server components data fetching")

# 查询 Prisma 的新查询 API
mcp__context7__query-docs("/prisma/docs", "Prisma include select relation query")

# 查询 NestJS 新的装饰器
mcp__context7__query-docs("/nestjs/docs", "NestJS guard interceptor decorator")
```

### 强制规则

1. **版本匹配**：如果项目使用特定版本，优先查询该版本文档
2. **代码示例优先**：Context7 返回的代码示例比通用知识更可靠
3. **API 废弃检查**：检查返回文档中是否有废弃 API 的警告
4. **配置方式**：新版本的配置方式可能变化，必须查询确认

### 通知用户

当调用 Context7 时，简短告知用户：

```
📚 检测到项目使用 [Next.js 15]，正在获取最新官方文档...
```

---

## 2.6 数据库变更规范

### Migration 脚本必须包含

1. **Up 方向**（应用变更）
2. **Down 方向**（回滚变更）—— 即使项目中其他 Migration 没有 Down，也建议生成
3. **幂等性**：使用 `IF NOT EXISTS` / `IF EXISTS`，避免重复执行报错

### 字段命名与现有表保持一致

分析现有表的字段命名规律，新字段严格遵循：
- 时间字段：与现有表一致（`created_at` 或 `create_time` 选一个）
- 主键：与现有表一致（自增 INT 或 UUID）
- 软删除：与现有表一致（`deleted_at` IS NULL 或 `is_deleted = 0`）
- 通用字段：检查是否需要加 `tenant_id`、`org_id` 等多租户字段

### DB Change Log 格式

```markdown
## DB 变更说明

| 变更类型 | 表名 | 字段/索引名 | 说明 |
|---------|-----|-----------|-----|
| 新增表 | orders | — | 订单主表 |
| 新增字段 | orders | user_id | 关联用户ID，外键 users.id |
| 新增索引 | orders | idx_orders_user_id | 提升用户订单查询性能 |

**执行顺序：** 先执行 Migration，再部署代码。
**回滚方式：** 执行 Down Migration 脚本。
**数据迁移：** 无 / [如有，说明步骤]
```
