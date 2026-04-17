---
name: project-dev-skill
user_invocable: true
description: "项目感知型开发自动化 Skill，自动感知项目上下文生成一致的代码、测试用例、接口文档，执行 Code Review。支持多项目隔离，Profile 按项目独立存储；集成 Context7 MCP 自动获取新技术栈官方文档。"
TRIGGER when: 用户请求读取代码、编写代码、修改代码、新增功能、修复BUG、重构代码，或提供需求描述、接口定义、原型图等开发相关任务。
DO NOT TRIGGER when: 纯粹的问答、文档编写、非代码相关的咨询。
---

# Project Dev Skill — 项目感知型开发自动化

## 项目隔离机制

Profile 按项目隔离存储，支持多项目无缝切换：

```
references/projects/
├── {projectName-1}/
│   └── profile.json    ← 项目1的 Profile 缓存
├── {projectName-2}/
│   └── profile.json    ← 项目2的 Profile 缓存
└── ...
```

- **自动识别**：根据当前工作目录自动获取项目名称
- **自动加载**：切换项目时自动加载对应的 Profile 缓存
- **无需重扫**：已扫描过的项目直接复用，无需重复分析

---

## 快捷命令

用户可通过自然语言触发管理命令：

| 命令短语 | 功能 |
|---------|------|
| "刷新 Profile" / "重新分析项目" | 重新扫描项目 |
| "查看 Profile" / "显示 Profile" | 查看当前 Profile 内容 |
| "导出 Profile" | 导出 Profile JSON 文件 |
| "对比文件风格" | 分析指定文件与 Profile 的风格差异 |
| "清除缓存" | 删除当前项目的 Profile 缓存 |

详见 `references/commands.md`

---

## Profile 过期管理

Profile 缓存默认有效期 **7 天**，过期后建议刷新：

- 配置文件变更自动标记过期
- 核心依赖版本变更触发更新
- 用户可手动触发刷新

详见 `references/profile-management.md`

---

## 新技术栈文档获取

当涉及新版本框架/库时，**自动调用 Context7 MCP** 获取最新官方文档和代码示例：

- **新版本框架**：Next.js 15、React 19、NestJS 11 等
- **首次使用的库**：项目中未出现过的新依赖
- **API 变更的库**：库的新版本有 breaking changes

详见 `references/development.md` → "2.5 新技术栈文档获取"

---

## 总体流程

```
输入（需求/BUG/截图）
  │
  ▼
[阶段 1] 项目感知        ← 读 references/project-sensing.md
  │
  ▼
[阶段 2] 开发实施        ← 读 references/development.md
  │
  ▼
[阶段 3] 产物生成        ← 读 references/artifacts.md
  │
  ▼
[阶段 4] Code Review     ← 读 references/code-review.md
  │
  ▼
[阶段 5] Profile 自更新  ← 读 references/profile-update.md
```

每个阶段的详细指令在对应 reference 文件中。**进入每个阶段前必须先读取对应文件。**

---

## 快速判断：任务类型

在开始之前，先判断任务类型，后续阶段的侧重点不同：

| 任务类型 | 特征关键词 | 开发策略 |
|---------|-----------|---------|
| **新功能** | "新增"、"实现"、"做一个" | 参考风格从零生成，完整产物 |
| **BUG 修复** | "报错"、"BUG"、"修复"、"fix" | 定位最小改动范围，输出 diff 格式 |
| **存量改造** | "重构"、"优化"、"改造" | 分析影响面，渐进式变更 |
| **数据库变更** | "加字段"、"新建表"、"Migration" | 强制走表结构感知流程 |

---

## 输入支持

本 Skill 支持多种输入形式：

- **文字描述**：需求描述、BUG 描述（最常见）
- **截图 / 原型图**：自动解析 UI 意图，生成对应后端接口和前端组件
- **Postman Collection / OpenAPI YAML**：以接口定义为入口，逆向生成实现代码
- **错误日志 / Stack Trace**：定位 BUG 根因，生成最小修复方案

---

## 数据库表结构获取策略（优先级降级）

```
优先级 1：MCP 直连数据库（实时准确）
    ↓ 如果没有配置 MCP
优先级 2：扫描 Migration 文件目录
    ↓ 如果没有 Migration
优先级 3：扫描 ORM Model 定义文件
    ↓ 如果都没有
优先级 4：请用户提供建表 SQL 或 Schema 描述
```

MCP 配置示例见 `references/project-sensing.md` → "数据库感知"章节。

---

## 输出产物一览

| 产物 | 触发条件 | 格式 |
|-----|---------|-----|
| 代码文件 | 始终 | 与项目语言一致 |
| Migration 脚本 | 涉及表结构变更 | 项目 Migration 框架格式 |
| 单元测试 | 始终 | 项目测试框架风格 |
| 集成测试 | 有新接口时 | — |
| 接口文档 | 有新增/修改接口 | OpenAPI 或 Markdown |
| 功能说明文档 | 新功能类任务 | Markdown |
| DB Change Log | 表结构变更 | Markdown 表格 |
| 变更影响图 | 始终 | 文字拓扑 + 受影响文件列表 |
| Code Review 报告 | 始终 | 分级报告（见下） |

---

## Code Review 分级标准

```
🔴 Block     — 必须修复才能提交（安全漏洞、逻辑错误、数据丢失风险）
🟡 Warning   — 强烈建议修复（性能问题、缺失错误处理、潜在竞态）
🔵 Suggestion — 可选优化（可读性、风格一致性、扩展性）
```

有任何 🔴 Block 问题时，必须在报告末尾明确提示：**"存在 Block 级问题，建议修复后再提交"**。

---

## 参考文件索引

| 文件 | 内容 | 何时读取 |
|-----|-----|---------|
| `references/project-sensing.md` | 项目扫描、风格提取、DB 感知、Profile 建立 | 阶段 1 开始前 |
| `references/development.md` | 代码生成规则、diff 格式、多模态输入处理 | 阶段 2 开始前 |
| `references/artifacts.md` | 各类产物生成模板和规范 | 阶段 3 开始前 |
| `references/code-review.md` | Review 检查清单、报告格式 | 阶段 4 开始前 |
| `references/profile-update.md` | Profile 自更新规则和触发条件 | 阶段 5 开始前 |
| `references/profile-management.md` | Profile 过期策略、刷新管理 | 管理 Profile 时 |
| `references/commands.md` | 快捷命令列表和执行流程 | 用户触发命令时 |
| `references/projects/{projectName}/profile.json` | 各项目的 Profile 缓存 | 阶段 1 自动加载 |
