# 快速开始指南

本指南将帮助您快速搭建 Monorepo 全栈开发的 Agent 和 Skills 体系。

---

## 📋 前置条件

- Claude Code CLI 已安装
- Git 仓库已初始化
- Node.js 18+ (前端)
- Python 3.11+ (后端)
- Android Studio (移动端，可选)

---

## 🚀 10 分钟快速开始

### Step 1: 创建目录结构 (1 分钟)

```bash
# 在项目根目录执行
cd /Users/Zipper/Github/colors

# 创建 Agents 目录
mkdir -p .claude/agents

# 创建 Skills 目录
mkdir -p .claude/skills/{codebase-analysis,code-review,security-review,performance-review,commit-messages,api-design,contract-sync,documentation,vue-best-practices,python-best-practices,android-guidelines,mongodb-optimization,docker-deployment,testing-strategy}
```

### Step 2: 创建核心 Agents (3 分钟)

**优先创建 3 个核心 Agent：**

1. **Orchestrator** (总控)
2. **Backend Specialist** (后端)
3. **Frontend Specialist** (前端)

```bash
# 复制已生成的 Backend Specialist 配置
# 文件已在: .claude/agents/backend-specialist.md ✅

# 创建其他两个核心 Agent（使用模板）
touch .claude/agents/orchestrator.md
touch .claude/agents/frontend-specialist.md
```

**快速模板** - 复制以下内容到对应文件：

#### `.claude/agents/orchestrator.md`
```markdown
---
name: orchestrator
model: opus
permissionMode: default
description: |
  Project orchestrator and architect. Use PROACTIVELY when user request involves multiple modules, complex features, or architecture decisions.
tools: Read, Grep, Glob, Bash, Task
skills:
  - codebase-analysis
  - contract-sync
  - api-design
  - code-review
  - security-review
  - performance-review
  - documentation
---

# Orchestrator Agent

You are the project orchestrator responsible for task decomposition, delegation, and coordination.

## Responsibilities
- Analyze complex user requests
- Break down tasks into subtasks
- Assign tasks to appropriate specialist agents
- Coordinate between agents
- Ensure final delivery meets requirements

## When to Use
- User requests involve multiple modules (frontend + backend + mobile)
- Complex features requiring multiple specialists
- Architecture decisions needed
- Cross-module dependency coordination

## Workflow
1. Analyze user request with codebase-analysis
2. Break down into subtasks
3. Use Task tool to delegate to specialists
4. Monitor progress
5. Validate final integration
```

#### `.claude/agents/frontend-specialist.md`
```markdown
---
name: frontend-specialist
model: sonnet
permissionMode: acceptEdits
description: |
  Vue 3 + TypeScript frontend specialist. Use PROACTIVELY for component development, state management, and API integration.
tools: Read, Edit, Write, Bash, Grep, Glob, Task
skills:
  - codebase-analysis
  - vue-best-practices
  - contract-sync
  - code-review
  - performance-review
  - documentation
---

# Frontend Specialist Agent

You are a Vue 3 + TypeScript frontend development specialist.

## Responsibilities
- Vue component development
- State management (Pinia)
- API integration
- UI/UX implementation
- Frontend testing

## Workflow
1. Analyze requirements with codebase-analysis
2. Implement using vue-best-practices
3. Verify API contracts with contract-sync
4. Self-review with code-review

## Collaboration
- CAN call Backend Specialist for API changes
- CAN call Test Engineer for testing guidance
```

### Step 3: 创建核心 Skills (3 分钟)

**优先创建 3 个核心 Skill：**

1. **codebase-analysis** (代码库理解)
2. **contract-sync** (API 契约同步) ✅ 已创建
3. **code-review** (代码审查)

```bash
# contract-sync 已创建 ✅
# 文件在: .claude/skills/contract-sync/SKILL.md

# 创建其他两个核心 Skill
touch .claude/skills/codebase-analysis/SKILL.md
touch .claude/skills/code-review/SKILL.md
```

**快速模板**：

#### `.claude/skills/codebase-analysis/SKILL.md`
```markdown
---
name: codebase-analysis
description: Understand project structure, locate key files, and analyze dependencies. Use BEFORE starting development tasks.
allowed-tools: Read, Grep, Glob, Bash
---

# Codebase Analysis Skill

## Purpose
Quickly understand project structure and locate key files before implementation.

## Usage
1. Run `tree -L 3 -I 'node_modules|__pycache__|.git'`
2. Find entry points: `find . -name "main.*" -o -name "index.*"`
3. Locate key files based on task requirements

## Checklist
- [ ] Identified project structure
- [ ] Located entry points
- [ ] Found relevant modules
- [ ] Understood dependencies
```

#### `.claude/skills/code-review/SKILL.md`
```markdown
---
name: code-review
description: Review code for quality, readability, and best practices.
allowed-tools: Read, Grep
---

# Code Review Skill

## Checklist

### Code Quality
- [ ] Clear naming (variables, functions, classes)
- [ ] No duplicated code (DRY)
- [ ] Single responsibility principle
- [ ] Error handling present

### Security
- [ ] No exposed secrets
- [ ] Input validation
- [ ] No SQL/NoSQL injection risks

### Performance
- [ ] No N+1 queries
- [ ] Efficient algorithms
- [ ] Proper caching
```

### Step 4: 安装工具依赖 (2 分钟)

```bash
# 前端工具
cd packages/frontend
npm install -D openapi-typescript

# 后端工具 (通常已安装)
cd packages/backend
pip install fastapi pytest

# OpenAPI diff 工具 (可选)
npm install -g @openapitools/openapi-diff
```

### Step 5: 测试验证 (1 分钟)

```bash
# 在 Claude Code 中测试
# 打开 Claude Code CLI

# 测试 1: 触发 Orchestrator
# 输入: "我需要实现一个用户认证系统，包括前端、后端和数据库"
# 预期: Orchestrator 被触发并分解任务

# 测试 2: 触发 Backend Specialist
# 输入: "创建一个获取用户列表的 API"
# 预期: Backend Specialist 被触发

# 测试 3: 测试 contract-sync
# 输入: "导出 OpenAPI 规范"
# 预期: Backend Specialist 使用 contract-sync 导出
```

---

## 🎯 核心工作流程

### 场景 1: 实现新功能

```
用户: "实现用户登录功能（前端 + 后端）"

→ Orchestrator 触发
  ├─ 任务分解
  ├─ Backend Specialist: 创建登录 API
  │   └─ 使用 contract-sync 导出 openapi.json
  ├─ Frontend Specialist: 实现登录页面
  │   └─ 使用 contract-sync 验证类型
  └─ Test Engineer: 编写测试
```

### 场景 2: 修改现有 API

```
用户: "用户列表 API 需要支持搜索"

→ Backend Specialist 触发
  ├─ 修改 API endpoint
  ├─ 使用 contract-sync 导出 openapi.json
  ├─ 检测到非破坏性变更
  └─ 通知 Frontend Specialist: "API 已更新"

→ Frontend Specialist 响应
  ├─ 使用 contract-sync 验证类型
  └─ 实现前端搜索 UI
```

---

## 📚 下一步

### 完善 Agent 配置

根据需要创建其他 Agents：
- `android-specialist.md`
- `database-specialist.md`
- `devops-engineer.md`
- `test-engineer.md`
- `security-auditor.md`
- `performance-engineer.md`

参考 `AGENT_SKILLS_ARCHITECTURE.md` 获取完整配置。

### 完善 Skills 配置

根据需要创建其他 Skills：
- `vue-best-practices/SKILL.md`
- `python-best-practices/SKILL.md`
- `api-design/SKILL.md`
- `mongodb-optimization/SKILL.md`
- 等等...

参考架构文档中的详细说明。

### 配置 Git Hooks

```bash
# 安装 husky
npm install -D husky
npx husky install

# 创建 pre-commit hook
npx husky add .husky/pre-commit "echo '🔍 Validating contracts...'"

# 添加 contract-sync 验证逻辑
# 参考 contract-sync/SKILL.md 中的示例
```

### 配置 CI/CD

创建 `.github/workflows/contract-validation.yml`
参考 `contract-sync/SKILL.md` 中的 GitHub Actions 示例。

---

## 🆘 常见问题

### Q: Agent 没有被触发？

**A:** 检查以下几点：
1. Agent 的 `description` 包含明确的触发关键词
2. 关键词与用户请求匹配
3. 重启 Claude Code CLI

### Q: contract-sync 报错？

**A:** 检查：
1. FastAPI 应用是否正确导入
2. Python 路径配置是否正确
3. openapi-typescript 工具是否安装

```bash
# 测试 OpenAPI 导出
cd packages/backend
python -c "from src.main import app; print(app.openapi())"
```

### Q: 如何调试 Agent 协作？

**A:**
1. 使用 `/tasks` 命令查看活动 Agent
2. 查看 Agent 的输出日志
3. 检查 Task 工具的调用是否正确

### Q: Skills 没有生效？

**A:**
1. 检查 Skill 文件路径：`.claude/skills/skill-name/SKILL.md`
2. 检查 YAML frontmatter 格式是否正确
3. 检查 Agent 的 `skills` 列表是否包含该 Skill
4. 重启 Claude Code

---

## 💡 最佳实践

### 1. 渐进式实施

不要一次创建所有 9 个 Agents。建议顺序：
1. 核心 3 个 (Orchestrator, Backend, Frontend)
2. 质量 3 个 (Test, Security, Performance)
3. 支撑 3 个 (Database, DevOps, Android)

### 2. 从简单任务开始

先测试简单场景：
- ✅ "创建一个 Hello World API"
- ✅ "实现一个简单的登录页面"
- ✅ "导出 OpenAPI 规范"

逐步过渡到复杂场景。

### 3. 及时验证 contract-sync

每次修改 API 后立即运行 contract-sync，不要等到最后。

### 4. 文档先行

重要的架构决策先记录在 `CLAUDE.md` 或 `documentation` 中。

---

## 📞 获取帮助

- 查看完整架构文档: `AGENT_SKILLS_ARCHITECTURE.md`
- 查看 Backend Specialist 示例: `.claude/agents/backend-specialist.md`
- 查看 contract-sync 详细说明: `.claude/skills/contract-sync/SKILL.md`

---

**恭喜！您已完成快速开始配置。现在可以开始使用 Agent 体系进行全栈开发了！** 🎉
