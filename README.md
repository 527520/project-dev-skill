# Project Dev Skill

> 项目感知型开发自动化 Skill，自动感知项目上下文生成一致的代码、测试用例、接口文档，执行 Code Review。

**作者：吴奇安**

---

## ✨ 功能特性

### 🔄 项目感知与隔离
- **自动扫描**：扫描项目结构、技术栈、代码风格，建立 Project Profile
- **多项目隔离**：Profile 按项目独立存储，切换项目自动加载缓存
- **风格一致性**：生成的代码严格遵循项目现有风格，而非通用最佳实践

### 📝 代码生成
- **多任务类型**：支持新功能开发、BUG 修复、存量改造、数据库变更
- **多模态输入**：文字描述、截图/原型图、OpenAPI YAML、错误日志
- **完整产物**：代码文件、Migration 脚本、单元测试、接口文档、DB Change Log

### 🔍 Code Review
- **分级报告**：Block（必须修复）、Warning（建议修复）、Suggestion（可选优化）
- **检查清单**：命名规范、错误处理、安全漏洞、性能问题等

### 📚 Context7 集成
- 自动获取新版本框架/库的官方文档和代码示例
- 确保使用最新版本的正确语法，避免废弃 API

### ⚡ 快捷命令
- `刷新 Profile` / `重新分析项目` - 重新扫描项目
- `查看 Profile` - 显示当前项目风格指纹
- `导出 Profile` - 导出 Profile JSON 文件供团队共享
- `对比文件风格` - 分析指定文件与 Profile 的风格差异

---

## 📦 安装教程

### Claude Code 安装

#### 方法一：通过 .skill 文件安装

1. 下载 `project-dev-skill.skill` 文件
2. 将文件放入 Claude Code 的 skills 目录：
   ```bash
   # macOS / Linux
   cp project-dev-skill.skill ~/.claude/skills/

   # Windows
   copy project-dev-skill.skill %USERPROFILE%\.claude\skills\
   ```
3. 重启 Claude Code 或执行 `/skills` 刷新

#### 方法二：手动安装

1. 创建 skill 目录：
   ```bash
   mkdir -p ~/.claude/skills/project-dev-skill/references
   ```

2. 下载并解压文件：
   ```bash
   cd ~/.claude/skills/project-dev-skill
   unzip /path/to/project-dev-skill.zip
   ```

3. 目录结构应为：
   ```
   ~/.claude/skills/project-dev-skill/
   ├── SKILL.md
   └── references/
       ├── project-sensing.md
       ├── development.md
       ├── artifacts.md
       ├── code-review.md
       ├── profile-update.md
       ├── profile-management.md
       └── commands.md
   ```

### Cursor 安装

1. 找到 Cursor 的配置目录：
   - **macOS**: `~/.cursor/`
   - **Windows**: `%APPDATA%\Cursor\`
   - **Linux**: `~/.config/cursor/`

2. 创建 skills 目录并安装：
   ```bash
   # macOS / Linux
   mkdir -p ~/.cursor/skills
   cp project-dev-skill.skill ~/.cursor/skills/

   # Windows
   mkdir %APPDATA%\Cursor\skills
   copy project-dev-skill.skill %APPDATA%\Cursor\skills\
   ```

3. 重启 Cursor

### Windsurf / 其他 IDE 安装

参考上述方法，将 skill 文件放入对应 IDE 的 skills 目录即可。

---

## 🚀 使用方法

### 自动触发

当你说出以下内容时，skill 会自动激活：

- "帮我写一个用户登录功能"
- "修复这个 BUG"
- "重构这段代码"
- "新增一个订单接口"
- "查看 src/services/user.ts"

### 手动触发

```
/project-dev-skill
```

### 快捷命令

| 命令 | 功能 |
|------|------|
| `刷新 Profile` | 重新扫描项目，更新 Profile |
| `查看 Profile` | 显示当前项目的风格指纹 |
| `导出 Profile` | 导出 Profile 供团队共享 |
| `对比文件风格 src/xxx.ts` | 分析文件与项目风格的差异 |
| `清除缓存` | 删除当前项目的 Profile 缓存 |

---

## 📋 工作流程

```
输入（需求/BUG/截图）
  │
  ▼
[阶段 1] 项目感知 ← 建立/加载 Profile
  │
  ▼
[阶段 2] 开发实施 ← 代码生成 + Context7 文档查询
  │
  ▼
[阶段 3] 产物生成 ← 测试/文档
  │
  ▼
[阶段 4] Code Review ← 分级审查报告
  │
  ▼
[阶段 5] Profile 自更新 ← 学习新风格
```

---

## 🗂️ 文件结构

```
project-dev-skill/
├── SKILL.md                      # Skill 主文件
└── references/
    ├── project-sensing.md        # 项目感知：扫描、风格提取
    ├── development.md            # 开发实施：代码生成规则
    ├── artifacts.md              # 产物生成：测试、文档
    ├── code-review.md            # Code Review：检查清单
    ├── profile-update.md         # Profile 自更新规则
    ├── profile-management.md     # Profile 过期与刷新管理
    └── commands.md               # 快捷命令支持
```

---

## ⚙️ 配置要求

### 可选：数据库 MCP 连接

如需实时获取数据库表结构，可配置数据库 MCP Server：

```json
// ~/.claude/settings.json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://user:pass@localhost:5432/db"]
    }
  }
}
```

### 可选：Context7 MCP

已内置 Context7 MCP 集成，无需额外配置即可自动获取新技术栈文档。

---

## 📝 示例

### 示例 1：新功能开发

```
用户：帮我新增一个订单管理模块，包含创建订单、查询订单、取消订单功能

Skill 响应：
1. 扫描项目，识别使用 NestJS + TypeORM + PostgreSQL
2. 生成开发计划，确认后开始开发
3. 生成 OrderController、OrderService、OrderRepository
4. 生成 Migration 脚本
5. 生成单元测试
6. 输出 Code Review 报告
```

### 示例 2：BUG 修复

```
用户：修复这个报错：TypeError: Cannot read property 'id' of undefined

Skill 响应：
1. 分析错误日志，定位到 src/services/user.service.ts:87
2. 生成 diff 格式的修复方案
3. 说明修复原因和潜在影响
```

### 示例 3：新技术栈查询

```
用户：帮我用 Next.js 15 的 App Router 写一个页面

Skill 响应：
1. 检测到 Next.js 15，调用 Context7 获取最新文档
2. 使用官方推荐的 App Router 语法生成代码
3. 避免使用已废弃的 Pages Router 方式
```

---

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

---

## 📄 License

MIT License

---

## 👤 作者

**吴奇安**

如果这个 Skill 对你有帮助，欢迎 Star ⭐
