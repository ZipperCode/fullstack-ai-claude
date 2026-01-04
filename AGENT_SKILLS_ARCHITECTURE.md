# Monorepo 全栈开发 Agent & Skills 架构方案

## 📋 项目概览

**技术栈：**
- 前端：Vue 3 + TypeScript
- 后端：Python + FastAPI
- 移动端：Android (JetpackCompose + Kotlin)
- 数据库：MongoDB + Redis
- 部署：Docker + CI/CD

**架构模式：**
- 1 个总控 Agent (Orchestrator)
- 8 个专职 Agent
- 14 个共享 Skills

---

## 🎯 Agent 架构总览

### 总控层
- **Orchestrator** - 全能协调者（任务分解、架构决策、进度追踪）

### 开发层
- **Frontend Specialist** - Vue 3 前端开发
- **Backend Specialist** - Python 后端开发
- **Android Specialist** - Android 移动端开发
- **Database Specialist** - MongoDB + Redis 数据库专家

### 支撑层
- **DevOps Engineer** - Docker + CI/CD 部署
- **Test Engineer** - 全栈测试策略
- **Security Auditor** - 安全审计
- **Performance Engineer** - 全栈性能优化

---

## 🛠️ Skills 架构总览

### 代码理解类 (1个)
1. **codebase-analysis** - 代码库理解与分析

### 代码审查类 (3个)
2. **code-review** - 通用代码审查
3. **security-review** - 安全审查
4. **performance-review** - 性能审查

### 工作流程类 (4个)
5. **commit-messages** - 提交信息规范
6. **api-design** - API 设计规范
7. **contract-sync** - API 契约同步与校验 🆕
8. **documentation** - 技术文档生成

### 专业技能类 (6个)
9. **vue-best-practices** - Vue 3 最佳实践
10. **python-best-practices** - Python 最佳实践
11. **android-guidelines** - Android 开发规范
12. **mongodb-optimization** - MongoDB 优化
13. **docker-deployment** - Docker 部署
14. **testing-strategy** - 测试策略

---

## 📊 Agent 与 Skills 关系矩阵

| Agent | Skills |
|-------|--------|
| **Orchestrator** | codebase-analysis, contract-sync, api-design, code-review, security-review, performance-review, documentation, commit-messages |
| **Frontend Specialist** | codebase-analysis, vue-best-practices, contract-sync, code-review, performance-review, documentation |
| **Backend Specialist** | codebase-analysis, python-best-practices, api-design, contract-sync, code-review, security-review, documentation |
| **Android Specialist** | codebase-analysis, android-guidelines, contract-sync, code-review, performance-review, documentation |
| **Database Specialist** | codebase-analysis, mongodb-optimization, code-review, performance-review, documentation |
| **DevOps Engineer** | codebase-analysis, docker-deployment, code-review, security-review, documentation |
| **Test Engineer** | codebase-analysis, testing-strategy, code-review, documentation |
| **Security Auditor** | codebase-analysis, security-review, code-review, documentation |
| **Performance Engineer** | codebase-analysis, performance-review, mongodb-optimization, code-review, documentation |

---

## 🔗 Agent 协作规则

### 协作模式 A：新功能开发
```
用户请求新功能
  ↓
Orchestrator 分析需求
  ↓
Orchestrator 拆解任务
  ↓
[并行] Frontend Specialist + Backend Specialist + Android Specialist
  ↓
[串行] Test Engineer 测试
  ↓
[串行] Security Auditor 审查
  ↓
Orchestrator 集成验收
```

### 协作模式 B：修改现有功能
```
用户请求修改
  ↓
相应的 Specialist 分析
  ↓
[可选] 如需其他模块配合，使用 Task 工具调用
  ↓
实现修改
  ↓
contract-sync 验证（如涉及 API）
```

### Task 工具调用权限矩阵

| Agent | 可以调用的 Agent |
|-------|----------------|
| **Orchestrator** | 所有 Agent（协调职责） |
| **Frontend Specialist** | Backend Specialist, Test Engineer |
| **Backend Specialist** | Frontend Specialist, Android Specialist, Database Specialist, Test Engineer |
| **Android Specialist** | Backend Specialist, Test Engineer, Performance Engineer |
| **Database Specialist** | Backend Specialist, Performance Engineer |
| **DevOps Engineer** | Frontend/Backend/Android Specialists, Database Specialist, Security Auditor |
| **Test Engineer** | Frontend/Backend/Android Specialists, Performance Engineer, Security Auditor |
| **Security Auditor** | 所有开发类 Agent（报告安全问题） |
| **Performance Engineer** | 所有开发类 Agent（报告性能问题） |

---

## 📁 目录结构

```
/Users/Zipper/Github/colors/
├── .claude/
│   ├── agents/                          # Agent 配置
│   │   ├── orchestrator.md
│   │   ├── frontend-specialist.md
│   │   ├── backend-specialist.md
│   │   ├── android-specialist.md
│   │   ├── database-specialist.md
│   │   ├── devops-engineer.md
│   │   ├── test-engineer.md
│   │   ├── security-auditor.md
│   │   └── performance-engineer.md
│   │
│   ├── skills/                          # Skills 配置
│   │   ├── codebase-analysis/
│   │   │   └── SKILL.md
│   │   ├── code-review/
│   │   │   └── SKILL.md
│   │   ├── security-review/
│   │   │   └── SKILL.md
│   │   ├── performance-review/
│   │   │   └── SKILL.md
│   │   ├── commit-messages/
│   │   │   └── SKILL.md
│   │   ├── api-design/
│   │   │   └── SKILL.md
│   │   ├── contract-sync/
│   │   │   ├── SKILL.md
│   │   │   └── tools-setup.md          # 工具安装说明
│   │   ├── documentation/
│   │   │   └── SKILL.md
│   │   ├── vue-best-practices/
│   │   │   └── SKILL.md
│   │   ├── python-best-practices/
│   │   │   └── SKILL.md
│   │   ├── android-guidelines/
│   │   │   └── SKILL.md
│   │   ├── mongodb-optimization/
│   │   │   └── SKILL.md
│   │   ├── docker-deployment/
│   │   │   └── SKILL.md
│   │   └── testing-strategy/
│   │       └── SKILL.md
│   │
│   ├── CLAUDE.md                        # 项目级文档
│   └── settings.json                    # 项目配置
│
├── packages/
│   ├── frontend/
│   ├── backend/
│   └── mobile/
│
├── openapi.json                         # API 契约（contract-sync 维护）
└── AGENT_SKILLS_ARCHITECTURE.md        # 本文档
```

---

## 🚀 实施步骤

### Phase 1: 创建目录结构
```bash
mkdir -p .claude/agents
mkdir -p .claude/skills/{codebase-analysis,code-review,security-review,performance-review,commit-messages,api-design,contract-sync,documentation,vue-best-practices,python-best-practices,android-guidelines,mongodb-optimization,docker-deployment,testing-strategy}
```

### Phase 2: 创建 Agents (按照附录的配置)
创建 9 个 Agent 配置文件到 `.claude/agents/`

### Phase 3: 创建 Skills (按照附录的配置)
创建 14 个 Skill 配置文件到 `.claude/skills/*/SKILL.md`

### Phase 4: 配置 contract-sync 工具
```bash
# 前端工具
cd packages/frontend
npm install -D openapi-typescript

# 后端工具（确保已安装）
cd packages/backend
pip install fastapi

# 可选：OpenAPI Diff 工具
npm install -g @openapitools/openapi-diff
```

### Phase 5: 创建根级文档
创建 `CLAUDE.md` 描述 monorepo 结构和开发规范

### Phase 6: 测试验证
- 测试每个 Agent 的触发条件
- 验证 Agent 之间的协作
- 验证 contract-sync 工作流

---

## 🔑 关键设计原则

### 1. 职责单一原则
- 每个 Agent 有明确的职责范围
- 审查类 Agent（Test、Security、Performance）**只审查，不修复**
- 开发类 Agent 负责实现

### 2. 协作而非冲突
- 新功能：Orchestrator 统筹规划（模式 A）
- 修改功能：Specialist 主动协调（模式 B）
- 使用 Task 工具跨 Agent 通信

### 3. 契约驱动开发
- `contract-sync` Skill 确保 API 契约是 Single Source of Truth
- 前后端通过 `openapi.json` 保持同步
- 手动触发验证（修改 API 后）

### 4. 质量内建
- 每个 Specialist 拥有 `code-review` 进行自查
- Test Engineer 全面测试
- Security Auditor 安全审查
- Performance Engineer 性能验证

### 5. 知识复用
- Skills 可被多个 Agents 共享
- `codebase-analysis` 是所有 Agent 的基础技能
- 专业 Skills 聚焦特定技术栈

---

## 📖 附录：完整配置

详细的 Agent 和 Skill 配置见独立文件：

### Agents 配置文件
- [orchestrator.md](docs/agents/orchestrator.md)
- [frontend-specialist.md](docs/agents/frontend-specialist.md)
- [backend-specialist.md](docs/agents/backend-specialist.md)
- [android-specialist.md](docs/agents/android-specialist.md)
- [database-specialist.md](docs/agents/database-specialist.md)
- [devops-engineer.md](docs/agents/devops-engineer.md)
- [test-engineer.md](docs/agents/test-engineer.md)
- [security-auditor.md](docs/agents/security-auditor.md)
- [performance-engineer.md](docs/agents/performance-engineer.md)

### Skills 配置文件
- [codebase-analysis](docs/skills/codebase-analysis.md)
- [contract-sync](docs/skills/contract-sync.md) 🆕
- [其他 Skills...](docs/skills/)

---

## 💡 使用示例

### 示例 1: 实现用户认证系统
```
用户: 实现完整的用户认证系统（前端、后端、移动端）

→ Orchestrator 触发
  [分析] 使用 codebase-analysis
  [拆解任务]
    Task 1: 数据库设计 → Database Specialist
    Task 2: 后端 API → Backend Specialist
    Task 3: 前端登录页 → Frontend Specialist
    Task 4: Android 登录 → Android Specialist
    Task 5: 测试 → Test Engineer
    Task 6: 安全审查 → Security Auditor

  [协调]
    - Backend 完成 API 后，触发 contract-sync
    - Frontend/Android 使用 contract-sync 验证类型

  [验收]
    - 所有模块集成测试通过
    - 安全审查通过
    - 性能符合标准
```

### 示例 2: 修改用户列表 API（增加搜索）
```
用户: 用户列表需要支持搜索功能

→ Frontend Specialist 触发（修改现有功能）
  [分析] 当前 API 不支持 search 参数

  [协作] 使用 Task 调用 Backend Specialist
    "请在 GET /api/v1/users 添加 search 查询参数"

→ Backend Specialist 实现
  - 修改 API endpoint
  - 更新 Pydantic Schema
  - 使用 contract-sync 导出新的 openapi.json

  [协作] 使用 Task 通知 Frontend Specialist
    "API 已更新，请验证类型定义"

→ Frontend Specialist 验证
  - 使用 contract-sync 检查类型一致性
  - 实现前端搜索 UI
```

### 示例 3: 性能优化
```
用户: 首页加载太慢

→ Performance Engineer 触发
  [分析]
    - Lighthouse: Performance 45/100
    - Bundle size: 2.5MB
    - LCP: 5.8s

  [生成报告]
    - Critical: 路由懒加载、依赖库优化
    - High: 图片优化、并行 API
    - Medium: HTTP/2、资源预加载

  [协作] 使用 Task 调用 Frontend Specialist
    "性能分析完成，优化方案已生成，请实施 Priority 1"

→ Frontend Specialist 实施
  - 实现路由懒加载
  - 替换 moment.js 为 day.js
  - lodash 按需导入

→ Performance Engineer 验证
  - Lighthouse: 45 → 85
  - LCP: 5.8s → 2.0s ✅
```

---

## 🎓 最佳实践

### 1. Agent 使用建议
- **明确任务类型**：新功能用 Orchestrator，修改功能用 Specialist
- **避免过度调用**：简单任务直接实现，不要每次都调用 Agent
- **及时验证**：API 修改后立即运行 contract-sync

### 2. Skills 使用建议
- **主动使用 codebase-analysis**：在不熟悉代码时先分析
- **遵循 code-review**：每次实现后自查
- **contract-sync 是关键**：前后端协作的核心

### 3. 协作建议
- **清晰的沟通**：使用 Task 工具时提供详细上下文
- **结果反馈**：完成任务后通知协作的 Agent
- **文档先行**：重要决策记录在 documentation

---

## 📞 支持与反馈

如有问题或建议，请：
1. 查看本文档的相关章节
2. 查阅具体 Agent/Skill 的配置文档
3. 测试验证后提出改进建议

---

**文档版本**: v1.0
**最后更新**: 2026-01-04
**维护者**: AI 架构团队
