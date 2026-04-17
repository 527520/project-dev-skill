# 阶段 1：项目感知

本文件指导如何建立项目的完整上下文 Profile。**每次新任务开始时必须执行，已有缓存 Profile 时可跳过扫描，直接复用。**

---

## 1.0 项目隔离机制（Profile 持久化）

### 存储路径规则

Profile 文件按项目名称隔离存储，路径格式：
```
references/projects/{projectName}/profile.json
```

- `projectName`：项目根目录名称（通过当前工作目录获取）
- 每个项目独立维护自己的 Profile 文件

### 首次访问流程

```
1. 获取当前项目名称
   → 从当前工作目录提取根目录名称作为 projectName
   → 例如：/Users/xxx/projects/my-app → projectName = "my-app"

2. 检查是否存在缓存的 Profile
   → 读取 references/projects/{projectName}/profile.json
   → 如果存在，检查过期状态（详见 references/profile-management.md）
   → 未过期且有效，直接加载使用，跳过扫描

3. 如果不存在缓存或已过期
   → 执行完整扫描流程
   → 扫描完成后，将 Profile 写入对应项目目录
```

### Profile 过期检查

加载 Profile 时，执行以下检查：

```javascript
// 过期判断逻辑
const isExpired = (profile) => {
  const lastUpdated = new Date(profile.lastUpdated);
  const now = new Date();
  const daysDiff = (now - lastUpdated) / (1000 * 60 * 60 * 24);
  return daysDiff > 7; // 默认 7 天过期
};
```

**过期处理：**
- 已过期 → 提示用户："⏰ Profile 缓存已过期（超过 7 天），建议刷新。请说 '刷新 Profile' 或继续使用当前缓存。"
- 用户选择刷新 → 重新扫描
- 用户继续 → 加载 Profile，标记为可能过时

### 获取项目名称的方法

```bash
# 方法1：从工作目录提取
# 当前工作目录为 /Users/xxx/projects/my-app/src/components
# 项目名称 = 工作目录中第一个非标准路径段的名称

# 方法2：查找项目根目录标识文件
# 优先查找 .git 目录所在的父目录名
```

**实现步骤：**
1. 使用 Bash 执行 `pwd` 获取当前工作目录
2. 解析目录路径，提取项目根目录名称
3. 如果是 git 项目，可用 `git rev-parse --show-toplevel` 获取项目根目录

### Profile 文件结构

```json
{
  "projectName": "my-app",
  "projectPath": "/Users/xxx/projects/my-app",
  "lastUpdated": "2024-01-15T10:30:00Z",
  "version": 1,
  "profile": {
    // 实际的 Profile 内容
    "language": "TypeScript",
    "framework": "NestJS",
    // ... 其他字段
  }
}
```

---

## 1.1 项目结构扫描

### 扫描目标

按以下顺序扫描，获取项目基本信息：

```
1. 根目录文件（package.json / pyproject.toml / pom.xml / go.mod / Cargo.toml / composer.json）
   → 识别：语言、框架、依赖库、版本

2. 目录树（深度 ≤ 3 层，忽略 node_modules / .git / dist / __pycache__）
   → 识别：项目分层结构（MVC / 分层架构 / 模块化 / 微服务）

3. 配置文件（.eslintrc / .prettierrc / pyproject.toml[tool.black] / .editorconfig）
   → 识别：缩进、引号风格、行尾、最大行宽

4. CI 配置（.github/workflows / .gitlab-ci.yml / Jenkinsfile）
   → 识别：测试框架、构建流程、代码检查工具
```

### 技术栈识别速查

| 文件特征 | 技术栈 |
|---------|-------|
| `package.json` + `next.config.*` | Next.js |
| `package.json` + `vue.config.*` | Vue.js |
| `requirements.txt` + `manage.py` | Django |
| `requirements.txt` + `app.py` / `main.py` | FastAPI / Flask |
| `pom.xml` + `src/main/java` | Spring Boot |
| `go.mod` | Go |
| `Cargo.toml` | Rust |
| `composer.json` + `artisan` | Laravel |

---

## 1.2 代码风格提取

### 提取样本策略

从项目中选取 **3～5 个最具代表性的现有文件** 作为风格样本，优先选择：
- 与本次任务同类型的文件（如要写 Controller，就看现有 Controller）
- 代码量适中（200～500 行），非脚手架生成的模板文件
- 最近修改过的文件（反映当前团队风格）

### 风格提取清单

从样本文件中提取以下信息，形成风格指纹：

**命名规范**
- 变量名：camelCase / snake_case / PascalCase
- 函数名：命名动词前缀习惯（如 `get` / `fetch` / `query` / `find`）
- 常量：UPPER_SNAKE / const camelCase
- 文件名：kebab-case / snake_case / PascalCase
- 数据库字段：snake_case（几乎普遍）/ camelCase（少数 NoSQL 项目）

**注释风格**
- 是否有文件头注释（版权、作者、日期）
- 函数注释：JSDoc / Python docstring / Javadoc / 行内注释
- 注释语言：中文 / 英文 / 混合
- 注释密度：每函数必有 / 关键逻辑处 / 极少注释

**代码结构**
- 错误处理模式：try/catch / Result 类型 / 错误码返回 / 异常抛出
- 日志调用：`logger.info` / `console.log` / `log.Printf` / 自定义封装
- 返回值格式：`{ code, data, message }` / `{ success, result }` / 直接返回数据
- 分页模式：offset/limit / cursor / page/pageSize
- 事务使用：装饰器 / 手动开关 / ORM 链式

**导入组织**
- 导入顺序（标准库 → 第三方 → 内部模块）
- 是否有 barrel 文件（`index.ts` / `__init__.py`）

---

## 1.3 数据库表结构感知

### 方案 A：MCP 直连数据库（最优）

**前置条件：** 用户已在 Claude 设置中连接了数据库 MCP Server。

常见 MCP Server：
- PostgreSQL：`@modelcontextprotocol/server-postgres`
- MySQL：`mcp-server-mysql`
- SQLite：官方内置

**使用方法：**
```
通过 MCP 工具执行以下查询获取表结构：

-- PostgreSQL
SELECT table_name, column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'public'
ORDER BY table_name, ordinal_position;

-- 获取索引信息
SELECT indexname, indexdef FROM pg_indexes WHERE schemaname = 'public';

-- 获取外键信息
SELECT tc.table_name, kcu.column_name, ccu.table_name AS foreign_table_name
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage AS ccu ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY';
```

从中提取：
- 字段命名规范（下划线 / 前缀规律）
- 主键类型（自增 int / UUID / 雪花 ID）
- 时间字段命名（`created_at` / `create_time` / `gmt_create`）
- 软删除字段（`is_deleted` / `deleted_at` / `status`）
- 通用字段规律（`tenant_id` / `operator_id` 等）

**如果没有 MCP 连接，告知用户：**
> "检测到未配置数据库 MCP 连接，将降级到 Migration 文件扫描。如需实时表结构感知，可在设置中添加对应数据库的 MCP Server。"

### 方案 B：Migration 文件扫描

扫描以下目录（按框架自动选择）：

| 框架 | Migration 目录 |
|-----|--------------|
| Django | `*/migrations/*.py` |
| Laravel | `database/migrations/*.php` |
| Rails | `db/migrate/*.rb` |
| Flyway | `src/main/resources/db/migration/V*.sql` |
| Liquibase | `src/main/resources/db/changelog/` |
| Prisma | `prisma/migrations/` |
| TypeORM | `src/migrations/` |
| Alembic | `alembic/versions/*.py` |

**重点扫描最近 10 个 Migration 文件**，提取：
- 最新表结构状态（累积应用后的结果）
- 历史变更模式（字段如何增加、如何废弃）

### 方案 C：ORM Model 定义

| ORM | 扫描路径 |
|-----|---------|
| Django ORM | `*/models.py` / `*/models/*.py` |
| SQLAlchemy | 含 `Base` / `Column` 的文件 |
| Eloquent | `app/Models/*.php` |
| Prisma | `prisma/schema.prisma` |
| TypeORM Entity | `src/**/*.entity.ts` |
| GORM | 含 `gorm:` tag 的 struct 文件 |
| MyBatis | `src/main/resources/**/*Mapper.xml` |
| JPA Entity | `@Entity` 注解的 Java 文件 |

---

## 1.4 架构模式识别

根据扫描结果识别项目采用的架构模式，后续代码生成必须遵循：

**分层架构（最常见）**
```
Controller/Router → Service → Repository/DAO → DB
```
生成规则：每层只做本层的事，不跨层调用。

**DDD（领域驱动）**
```
Interface层 → Application层 → Domain层 → Infrastructure层
```
生成规则：业务逻辑集中在 Domain，Application 做编排，Repository 接口在 Domain 定义。

**CQRS**
```
Command Handler / Query Handler 分离
```
生成规则：写操作和读操作走不同路径，不共用 Service。

**函数式 / 无状态**
```
纯函数 + 副作用隔离
```
生成规则：避免类，优先高阶函数，副作用推到边界。

---

## 1.5 建立 Project Profile

扫描完成后，建立 Profile 对象并持久化存储：

### 1.5.1 获取项目名称

首先获取当前项目名称，用于隔离存储：

```bash
# 步骤1：获取项目根目录
# 优先使用 git 根目录（如果存在）
git rev-parse --show-toplevel 2>/dev/null || pwd

# 步骤2：提取项目名称
# 从根目录路径中提取最后一个路径段作为项目名称
# 例如：/Users/xxx/projects/my-app → my-app
```

### 1.5.2 构建 Profile 对象

```json
{
  "projectName": "my-app",
  "projectPath": "/Users/xxx/projects/my-app",
  "lastUpdated": "2024-01-15T10:30:00Z",
  "version": 1,
  "profile": {
    "language": "TypeScript",
    "framework": "NestJS",
    "frameworkVersion": "11.0.0",
    "architecture": "分层架构（Controller → Service → Repository）",
    "naming": {
      "variable": "camelCase",
      "function_prefix": ["get", "find", "create", "update", "delete"],
      "file": "kebab-case",
      "db_field": "snake_case"
    },
    "comment": {
      "style": "JSDoc",
      "language": "中文",
      "density": "每个 public 方法必有"
    },
    "error_handling": "抛出自定义 HttpException，统一 ExceptionFilter 捕获",
    "response_format": "{ code: number, data: T, message: string }",
    "pagination": "page + pageSize，返回 { list, total }",
    "logging": "this.logger.log / warn / error（NestJS Logger）",
    "db": {
      "orm": "TypeORM",
      "pk_type": "UUID",
      "soft_delete": "deletedAt（TypeORM @DeleteDateColumn）",
      "common_fields": ["createdAt", "updatedAt", "deletedAt", "createdBy"]
    },
    "test_framework": "Jest + Supertest",
    "db_source": "MCP直连（PostgreSQL）",
    "dependencies": {
      "core": {
        "nestjs": "11.0.0",
        "typeorm": "0.3.20",
        "typescript": "5.3.0"
      },
      "notable": {
        "class-validator": "0.14.0",
        "zod": "3.22.0"
      }
    }
  }
}
```

### 版本信息获取（用于 Context7 文档查询）

从依赖配置文件中提取关键版本信息：

| 语言 | 配置文件 | 提取字段 |
|-----|---------|---------|
| Node.js | `package.json` | `dependencies`, `devDependencies` |
| Python | `pyproject.toml` / `requirements.txt` | `dependencies` |
| Java | `pom.xml` | `version`, `dependencies.version` |
| Go | `go.mod` | `require` |
| Rust | `Cargo.toml` | `dependencies` |
| PHP | `composer.json` | `require` |

**提取策略：**
1. 核心框架版本（如 nestjs, next, react, vue）
2. ORM 版本（如 typeorm, prisma, sqlalchemy）
3. 语言版本（如 typescript, python, java version）
4. 关键工具库（如 class-validator, zod）

### 1.5.3 持久化存储

将 Profile 写入项目专属目录：

```
存储路径：references/projects/{projectName}/profile.json
```

**执行步骤：**
1. 确保目录存在：`references/projects/{projectName}/`
2. 将 Profile 对象写入 `profile.json`
3. 后续访问同一项目时，直接读取该文件

### 1.5.4 确认通知

**Profile 建立后，向用户简要确认：**
> "已感知项目上下文：[语言/框架]，[架构模式]，数据库通过 [获取方式] 感知。Profile 已缓存至 `references/projects/{projectName}/profile.json`，下次可复用。开始开发。"

若关键信息缺失（无法确定框架或数据库），先询问用户补充，再继续。
