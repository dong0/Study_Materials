# AI Agent Rules

本文件是从 `.cursor/rules/*.mdc` 汇总而来的通用 AI 规则文件，适用于本仓库内的 AI Agent 协作。Cursor 专用规则仍保留在 `.cursor/rules/` 中；如果后续修改规则，请尽量同步更新两处。

## 全局规则

以下规则默认适用于整个仓库。

### Git 规范

#### Commit Message 格式

```text
type(scope): description

[可选] BREAKING CHANGE: 描述破坏性变更
```

#### type 枚举

- `feat` -- 新功能
- `fix` -- 修复 bug
- `docs` -- 文档内容新增或修改
- `style` -- 代码格式调整（不影响功能）
- `refactor` -- 代码重构（不影响功能）
- `test` -- 测试相关变更
- `chore` -- 构建、配置、杂项变更
- `perf` -- 性能优化
- `ci` -- CI/CD 配置变更

#### scope

使用文档主题名或功能模块名（小写英文），如：

- `controllers`、`pipes`、`guards`、`interceptors`
- `modules`、`providers`、`middleware`、`exception-filters`
- `custom-decorators`、`custom-providers`
- `config`、`build`、`deps`（依赖管理）

非文档变更可省略 scope。

#### description

- 使用中文描述变更内容，简洁明了
- 长度控制在 50 个字符以内
- 以动词开头，使用祈使句语气（如：添加、修改、修复、删除）

#### Breaking Changes

对于破坏性变更，在 commit message 末尾添加：

```text
BREAKING CHANGE: 描述具体的破坏性变更内容
```

#### 分支命名规范

```text
type/description 或 feature/description 或 bugfix/issue-number
```

- `main` -- 主分支，稳定版本
- `develop` -- 开发分支
- `feature/*` -- 新功能分支，如 `feature/add-custom-providers-doc`
- `bugfix/*` -- 修复分支，如 `bugfix/fix-translation-error`
- `hotfix/*` -- 紧急修复分支

#### 版本控制（可选）

遵循 [Semantic Versioning](https://semver.org/)：

- `MAJOR.MINOR.PATCH`（如：`1.0.0`）
- `feat` 提交增加 MINOR 版本
- `fix` 提交增加 PATCH 版本
- BREAKING CHANGE 增加 MAJOR 版本

#### Emoji 支持（可选）

可选择性在 type 前添加 emoji：

- ✨ `feat` -- 新功能
- 🐛 `fix` -- 修复 bug
- 📚 `docs` -- 文档
- 🎨 `style` -- 格式
- 🔄 `refactor` -- 重构
- ✅ `test` -- 测试
- 🔧 `chore` -- 配置
- ⚡ `perf` -- 性能

#### 示例

- `feat(custom-providers): 添加自定义提供者文档翻译`
- `fix(controllers): 修正路由参数示例代码错误`
- `docs(pipes): 更新管道章节的翻译说明`
- `style: 统一代码块语言标识格式`
- `refactor(config): 重构配置文件的目录结构`
- `test: 添加单元测试用例`
- `chore: 更新依赖包版本`

带破坏性变更的示例：

```text
feat(auth): 重构认证模块接口

BREAKING CHANGE: AuthService.validateUser() 方法签名已更改
```

带 emoji 的示例：

```text
✨ feat(custom-providers): 添加自定义提供者文档翻译
🐛 fix(controllers): 修正路由参数示例代码错误
```

### 项目结构与内容审核

#### 目录结构

- 学习文档按学习对象存放在对应的 `docs/` 目录下，如 `NestJS/docs`
- 文件命名格式：`{学习对象} - {English Topic Name}.zh.md`
- 文档主题应对应官方文档的章节划分

#### 内容审核要点

- 翻译须与官方英文文档保持一致，不可随意增删原文内容
- 代码示例须与官方示例匹配，确保可运行
- 如有自定义补充说明，需用明显标记与原文翻译区分（如使用 blockquote 或加粗标注）
- 新增文档前先确认 `docs/` 下是否已存在同主题文件，避免重复

## Markdown 文档规则

适用范围：`**/docs/**/*.md`

### 标题与元信息

- 一级标题格式：`# English Title（中文标题）`
- 文档开头必须包含来源链接，如：`来源：[NestJS xxx 文档](https://docs.nestjs.com/xxx)`
- 子标题双语并列，英文在前中文在后，用两个空格分隔：`### Routing  路由`

### 代码块

- 代码块必须指定语言标识，如 ```` ```typescript ````
- 代码块前应单独一行标注文件名，如 `` `cats.controller.ts` ``

### 排版

- 中文与英文、数字之间加一个空格，如“使用 `@Get()` 装饰器”
- 段落之间用空行分隔
- 列表项之间保持一致的缩进
- blockquote 用于提示和警告，格式为 `> **提示**` 或 `> **警告**`

## NestJS 文档规则

适用范围：`NestJS/docs/**/*.md`

### 内容规范

#### 代码示例

- 代码示例必须使用 TypeScript，且须为可运行的完整代码片段
- 如果 NestJS 官方文档同时提供 TS 和 JS 两种版本，两种都保留
- 装饰器用法必须附带完整的 import 语句

#### 概念解释

- 重要概念需给出简明的中文解释，而不仅仅是逐句翻译
- 涉及的 NestJS CLI 命令保留原文，如 `nest g controller [name]`

#### 内容来源

- 内容基于 NestJS 官方英文文档翻译整理
- 每篇文档对应官方文档的一个章节

### 语言规范

- 所有文档使用**简体中文**撰写
- 标题采用“英文原文 + 中文”双语并列格式，如 `### Routing  路由`
- 提示使用 `> **提示**`，警告使用 `> **警告**`

### NestJS 专有名词

以下专有名词保留英文：

Controller, Module, Provider, Guard, Pipe, Interceptor, Middleware, Decorator, Filter, Injectable, NestFactory, DynamicModule, ExecutionContext

### 通用术语对照表

| 英文 | 中文 |
|------|------|
| request | 请求 |
| response | 响应 |
| handler | 处理器 |
| endpoint | 端点 |
| route / routing | 路由 |
| dependency injection | 依赖注入 |
| middleware | 中间件 |
| exception | 异常 |
| decorator | 装饰器 |
| pipe | 管道 |
| guard | 守卫 |
| interceptor | 拦截器 |
| provider | 提供者 |
| module | 模块 |
| controller | 控制器 |
| serialization | 序列化 |
| validation | 验证 |
| payload | 载荷 |
| scope | 作用域 |
