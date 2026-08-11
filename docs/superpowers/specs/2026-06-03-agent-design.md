# AI Agent 全流程研发管线 · Agent 详细设计

> **版本:** v0.1 · **状态:** 设计稿
>
> 本文档精确描述每个 Agent 的边界、工作范围、交付物和工作方式。
> 先定义清楚再编码，避免后续返工。

---

## 目录

1. [Agent 全景概览](#1-agent-全景概览)
2. [需求 Agent (Requirement Agent)](#2-需求-agent-requirement-agent)
3. [设计 Agent (Design Agent)](#3-设计-agent-design-agent)
4. [编码 Agent (Coding Agent)](#4-编码-agent-coding-agent)
5. [测试 Agent (Testing Agent)](#5-测试-agent-testing-agent)
6. [上线 Agent (Deployment Agent)](#6-上线-agent-deployment-agent)
7. [编排引擎 (Orchestrator)](#7-编排引擎-orchestrator)
8. [Agent 间协作协议](#8-agent-间协作协议)

---

## 1. Agent 全景概览

### 1.1 流水线总图

```
 输入               Agent 流水线              输出
──────┼──── 需求 ──→ 设计 ──→ 编码 ──→ 测试 ──→ 上线 ──→
      │     Agent    Agent    Agent    Agent    Agent
      │       │         │        │        │        │
      │       │   上下文传递（PipelineContext）   │
      │       └──────────────────────────────────┘
      │                    │
      │            Agent 编排引擎
      │          (注册 / 调度 / 审批 / 重试)
```

### 1.2 设计原则

| 原则 | 说明 |
|------|------|
| **阶段松耦合** | 每个 Agent 独立部署、独立演進。需求 Agent 升级不影响编码 Agent |
| **上下文标准化** | Agent 的输出格式是下一个 Agent 的标准输入，保证流水线不断链 |
| **人工节点可插拔** | 阶段间的审批节点可配置。小团队关掉，大团队打开 |
| **渐进深度** | 每个 Agent 有"简版"和"完整版"。简版快速出结果，完整版多轮推理 |
| **可观测** | 每个 Agent 的输出包含完整的 metadata（耗时、模型、token 数、错误） |

### 1.3 通用术语

- **Task**：一个 Agent 接收的最小工作单元（如"分析这个 PRD"）
- **Capability**：Agent 的能力标识（如 `analyze-prd`、`generate-code`）
- **Artifact**：Agent 产出的交付件（文档 / 代码 / 测试 / 报告）
- **Manifest**：Agent 的"身份证"，声明名字、版本、能力列表和资源需求

---

## 2. 需求 Agent (Requirement Agent)

### 2.1 定位

**Agent 名称:** `requirement-agent`

需求 Agent 是整个流水线的**第一入口**。它把模糊的原始需求（PRD / Issue / 口头描述）转化为结构化的、可执行的开发任务列表。

### 2.2 边界

| ✅ 在范围内 | ❌ 不在范围内 |
|------------|-------------|
| PRD 解析与结构化 | 业务可行性判断（该不该做） |
| 需求模糊点识别与追问清单 | 产品方向决策 |
| 功能点的任务拆解（Epic → Story → Task） | UI/UX 设计 |
| 变更影响范围分析（关联模块、服务、API） | 项目管理排期（不替代 Jira 等） |
| 工作量估算（基于代码复杂度 + 历史数据） | 替代产品经理 |

### 2.3 Capabilities

```
analyze-prd
  └─ 输入: PRD 文本 / Issue 描述 / 语音转录
  └─ 输出: 结构化需求文档 + 模糊点清单 + 影响范围报告

decompose-tasks
  └─ 输入: 结构化的功能点列表
  └─ 输出: Epic → Story → Task 的拆解树 + 依赖关系 + 估算
```

### 2.4 交付物 (Artifacts)

| Artifact | 类型 | 内容 |
|----------|------|------|
| `requirement-analysis` | doc | 需求分析报告：提取的功能点、业务规则、验收标准 |
| `ambiguity-list` | doc | 模糊点清单：Agent 无法确定的问题（给产品经理确认） |
| `impact-report` | doc | 影响范围：变更会影响到哪些模块/服务/API |
| `task-decomposition` | doc | 任务拆解树：Epic → User Story → 子任务，含依赖和估算 |

### 2.5 工作流

```
                  输入 (PRD / Issue / 语音)
                           │
                           ▼
              ┌───────────────────────┐
              │  Step 1: PRD 解析      │
              │  ───────────────────── │
              │  • 提取功能点列表       │
              │  • 提取业务规则         │
              │  • 提取验收标准         │
              │  • 识别需求间依赖关系   │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 2: 模糊点识别    │  ← 需要 LLM 推理
              │  ───────────────────── │
              │  • 标记每个功能点的确定性  │
              │  • 对不确定点生成追问问题  │
              │  • 输出 ambiguity-list  │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 3: 影响分析      │  ← 需要代码库检索 (RAG)
              │  ───────────────────── │
              │  • 搜索代码库中相关模块   │
              │  • 识别受影响的服务      │
              │  • 识别受影响的 API     │
              │  • 评估变更风险等级      │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 4: 任务拆解      │
              │  ───────────────────── │
              │  • 功能点 → User Story  │
              │  • User Story → 子任务  │
              │  • 标注任务间依赖关系    │
              │  • 预估工作量（人天）    │
              └──────────┬────────────┘
                         ▼
              输出 (结构化工单 + 影响报告)
```

### 2.6 输入/输出 Schema

```python
# ==== 输入 (analyze-prd) ====
{
    "prd_text": str,              # PRD 原文或 Issue 描述
    "project_context": str,       # 项目背景（可选）
    "attachments": [str],         # 附件路径（可选）
}

# ==== 输出 (requirement-analysis artifact) ====
{
    "summary": str,                # 需求摘要
    "features": [
        {
            "id": str,
            "name": str,
            "description": str,
            "business_rules": [str],
            "acceptance_criteria": [str],
            "priority": "P0" | "P1" | "P2" | "P3",
            "depends_on": [str],    # 依赖的其他 feature id
        }
    ],
    "ambiguities": [
        {
            "feature_id": str,
            "question": str,
            "suggested_answer": str | None,
        }
    ],
    "impact_scope": {
        "services": [str],
        "apis": [str],
        "data_entities": [str],
    },
    "task_decomposition": {
        "epics": [
            {
                "id": str,
                "title": str,
                "stories": [
                    {
                        "id": str,
                        "title": str,
                        "tasks": [
                            {
                                "id": str,
                                "title": str,
                                "estimate_hours": float,
                                "dependencies": [str],
                            }
                        ],
                    }
                ],
            }
        ],
    },
}
```

---

## 3. 设计 Agent (Design Agent)

### 3.1 定位

**Agent 名称:** `design-agent`

设计 Agent 把需求 Agent 输出的任务拆解转化为**可执行的技術方案**。它是需求到代码之间的桥梁。一个需求可以有多个技术方案，设计 Agent 需要给出 trade-off 分析。

### 3.2 边界

| ✅ 在范围内 | ❌ 不在范围内 |
|------------|-------------|
| 多方案技术选型（如新服务 vs 改现有 vs 集成） | 写业务代码 |
| 架构设计（模块划分、交互关系） | 测试用例设计 |
| 接口定义（REST / gRPC / GraphQL Schema） | 部署方案 |
| 数据库模型设计（表结构、索引、迁移策略） | 具体的 UI 组件设计 |
| 安全设计（权限、认证、脱敏、合规） | 替代架构师决策（最终由人拍板） |

### 3.3 Capabilities

```
generate-tech-design
  └─ 输入: 任务拆解 + 现有代码库索引
  └─ 输出: 技术设计文档 (ADR + API Spec + 数据模型)

review-design
  └─ 输入: 技术设计文档
  └─ 输出: 设计审查意见
```

### 3.4 交付物 (Artifacts)

| Artifact | 类型 | 内容 |
|----------|------|------|
| `tech-design` | doc | 技术设计文档：方案对比、选型理由、整体架构 |
| `api-spec` | doc | 接口定义：OpenAPI / gRPC proto / GraphQL schema |
| `data-model` | doc | 数据模型：ER 图、表结构、索引方案、迁移计划 |
| `design-review` | doc | 设计审查意见：潜在问题、优化建议 |

### 3.5 工作流

```
                输入 (任务拆解 + 代码库索引)
                           │
                           ▼
              ┌───────────────────────┐
              │  Step 1: 需求理解      │
              │  ───────────────────── │
              │  • 合并需求 Agent 的输出  │
              │  • 理解业务上下文        │
              │  • 确定约束条件          │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 2: 方案设计      │  ← 核心推理
              │  ───────────────────── │
              │  • 生成 2-3 个候选方案   │
              │  • 每个方案标注:         │
              │    - 优点 / 缺点        │
              │    - 实现成本（人天）    │
              │    - 技术风险           │
              │    - 扩展性评估         │
              │  • 给出推荐方案 + 理由  │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 3: 接口设计      │
              │  ───────────────────── │
              │  • 定义 API 端点/方法   │
              │  • 定义请求/响应 Schema │
              │  • 生成 OpenAPI Spec   │
              │  • 标注权限要求         │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 4: 数据模型      │
              │  ───────────────────── │
              │  • 定义实体和关系       │
              │  • 设计表结构 + 字段   │
              │  • 设计索引策略        │
              │  • 设计迁移方案        │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 5: 安全自检      │
              │  ───────────────────── │
              │  • 权限设计检查        │
              │  • 数据脱敏检查        │
              │  • 合规风险标记        │
              └──────────┬────────────┘
                         ▼
              输出 (技术设计 + API + 数据模型)
```

### 3.6 输入/输出 Schema

```python
# ==== 输入 (generate-tech-design) ====
{
    "tasks": [...],       # 从需求 Agent 继承
    "codebase_index": {   # 代码库索引（由编排引擎注入）
        "services": [str],
        "interfaces": [str],
        "data_entities": [str],
    },
    "constraints": {       # 约束条件
        "tech_stack": [str],    # 如 ["Python 3.12", "PostgreSQL 16"]
        "deadline": str | None,
        "compliance": [str],   # 合规要求
    },
}

# ==== 输出 (tech-design artifact) ====
{
    "candidate_solutions": [
        {
            "name": str,          # 方案名称
            "description": str,   # 方案描述
            "pros": [str],
            "cons": [str],
            "cost_estimate_days": float,
            "risk_level": "low" | "medium" | "high",
            "scalability_notes": str,
        }
    ],
    "recommended_solution": str,  # 推荐的方案名称
    "architecture": {
        "components": [
            {"name": str, "responsibility": str, "dependencies": [str]},
        ],
        "data_flow": [  # 关键数据流
            {"from": str, "to": str, "data": str},
        ],
    },
    "interfaces": [
        {
            "name": str,
            "protocol": "REST" | "gRPC" | "GraphQL" | "Event",
            "endpoints": [
                {"path": str, "method": str, "request": {}, "response": {}},
            ],
        }
    ],
    "data_model": {
        "entities": [
            {
                "name": str,
                "fields": [
                    {"name": str, "type": str, "constraints": [str]},
                ],
                "indexes": [{"fields": [str], "unique": bool}],
            }
        ],
        "migration_plan": str,
    },
    "security_review": {
        "issues": [{"severity": str, "description": str, "suggestion": str}],
        "passed": bool,
    },
}
```

---

## 4. 编码 Agent (Coding Agent)

### 4.1 定位

**Agent 名称:** `coding-agent`

编码 Agent 是**开发者最直接交互**的 Agent。它把设计 Agent 的方案变成真实代码。这里的关键是"人机协作"——不是 AI 完全替代人，而是 AI 生成初稿，人在此基础上修改和完善。

### 4.2 边界

| ✅ 在范围内 | ❌ 不在范围内 |
|------------|-------------|
| 按任务粒度生成代码框架 | 替代开发者做架构决策（架构由设计 Agent 出） |
| 生成核心业务逻辑 | 运行/调试代码（那是 CI 的事情） |
| 代码审查（正确性/安全/性能/风格） | 部署代码 |
| 自动修复安全检查问题 | 编写完整的大型系统（按 Task/Story 粒度） |
| 生成 Commit Message + PR 描述 | 与产品经理沟通需求 |

### 4.3 Capabilities

```
generate-code
  └─ 输入: 技术设计 + 任务定义
  └─ 输出: 代码文件 (Diff / 新文件)

review-code
  └─ 输入: PR Diff + 代码库上下文
  └─ 输出: Review 意见（多维: 正确性/安全/性能/风格）

suggest-refactor
  └─ 输入: 代码文件
  └─ 输出: 重构建议 + Diff
```

### 4.4 交付物 (Artifacts)

| Artifact | 类型 | 内容 |
|----------|------|------|
| `code-generation` | code | 生成的代码文件/代码变更 |
| `code-review` | doc | 多维代码审查意见 |
| `refactor-suggestion` | doc | 重构建议 + 示例代码 |
| `commit-message` | doc | 自动生成的 Commit Message + PR Description |

### 4.5 "人机协作"工作流（核心）

```
             开发者标记任务范围 (Task/Story)
                           │
                           ▼
              ┌───────────────────────┐
              │  Step 1: 上下文理解    │
              │  ───────────────────── │
              │  • 从编排引擎获取设计文档  │
              │  • 读取现有代码库结构    │
              │  • 理解任务边界和约束    │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 2: 生成初版代码  │
              │  ───────────────────── │
              │  • 生成框架代码/骨架    │
              │  • 实现核心逻辑         │
              │  • 遵守项目代码规范     │
              │  • 添加必要注释         │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 3: 开发者 Review │  ← 人工环节
              │  ───────────────────── │
              │  • 开发者阅读并修改 AI 代码 │
              │  • 调整不符合预期的地方    │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 4: AI 二次审查   │  ← 编码 Agent 再次介入
              │  ───────────────────── │
              │  • 检查开发者修改是否引入问题 │
              │  • 给出最终优化建议       │
              │  • 生成 Commit Message   │
              └──────────┬────────────┘
                         ▼
              开发者提交 PR → 触发 Code Review
```

### 4.6 Code Review 模型（多维审查）

```
PR Diff
    │
    ├──→ 正确性审查 (Correctness)
    │     • 逻辑是否正确？边界条件是否覆盖？
    │     • 是否存在并发问题？
    │     • 错误处理是否完备？
    │
    ├──→ 安全审查 (Security)
    │     • 是否存在注入风险？（SQL/XSS/命令注入）
    │     • 权限检查是否到位？
    │     • 敏感数据是否泄露？
    │
    ├──→ 性能审查 (Performance)
    │     • 是否存在 N+1 查询？
    │     • 是否有不必要的资源消耗？
    │     • 缓存策略是否合理？
    │
    └──→ 风格审查 (Style)
          • 是否符合项目代码规范？
          • 命名是否清晰？
          • 是否有重复代码？
```

### 4.7 输入/输出 Schema

```python
# ==== 输入 (generate-code) ====
{
    "task": {
        "id": str,
        "title": str,
        "description": str,
        "acceptance_criteria": [str],
    },
    "design": {...},          # 从设计 Agent 继承
    "existing_code": {        # 相关现有代码
        "files": [{"path": str, "content": str}],
    },
    "code_standards": {       # 项目代码规范
        "language": str,
        "framework": str,
        "style_guide": str,
        "patterns": [str],
    },
}

# ==== 输出 (code-generation artifact) ====
{
    "changes": [
        {
            "file_path": str,
            "action": "create" | "modify" | "delete",
            "content": str,          # 完整文件内容
            "diff": str,             # Git diff 格式
        }
    ],
    "commit_message": {
        "title": str,                # commit 标题 (< 72 chars)
        "body": str,                 # commit 正文
        "type": "feat" | "fix" | "refactor" | "test" | "docs" | "chore",
    },
    "notes": [str],                  # 对开发者的说明/提醒
}

# ==== 输出 (code-review artifact) ====
{
    "summary": str,               # 总体评价
    "findings": [
        {
            "category": "correctness" | "security" | "performance" | "style",
            "severity": "blocker" | "critical" | "major" | "minor" | "nit",
            "file": str,
            "line": int,
            "message": str,
            "suggestion": str,      # 建议修复的代码
        }
    ],
    "verdict": "approve" | "request-changes" | "block",
}
```

---

## 5. 测试 Agent (Testing Agent)

### 5.1 定位

**Agent 名称:** `testing-agent`

测试 Agent 在代码变更完成后介入，自动生成测试用例、分析覆盖率、识别未覆盖路径。目标是把测试覆盖率从"看天吃饭"变成"机械可保证"。

### 5.2 边界

| ✅ 在范围内 | ❌ 不在范围内 |
|------------|-------------|
| 增量单元测试自动生成 | 手动测试替代 |
| 集成测试生成（基于接口定义） | 生产环境测试 |
| 覆盖率分析 + 自动补充 | E2E 测试编排 |
| 性能回归测试标记 | 性能测试结果解读 |
| 依赖漏洞扫描 | 渗透测试 |

### 5.3 Capabilities

```
generate-unit-tests
  └─ 输入: 代码变更 + 测试框架配置
  └─ 输出: 测试用例代码 + 预期覆盖率报告

generate-integration-tests
  └─ 输入: 接口定义 + 代码变更
  └─ 输出: 集成测试代码

analyze-coverage
  └─ 输入: 代码文件 + 已有测试
  └─ 输出: 覆盖分析报告 + 待补充用例
```

### 5.4 交付物 (Artifacts)

| Artifact | 类型 | 内容 |
|----------|------|------|
| `unit-tests` | test | 自动生成的单元测试代码 |
| `integration-tests` | test | 自动生成的集成测试代码 |
| `coverage-report` | report | 覆盖率分析：已覆盖 / 未覆盖的路徑 |
| `supplemental-tests` | test | 补充测试（覆盖未覆盖路径） |
| `vulnerability-scan` | report | 依赖漏洞扫描报告 |

### 5.5 工作流

```
         输入 (代码变更 + 接口定义 + 现有测试)
                           │
                           ▼
              ┌───────────────────────┐
              │  Step 1: 变更分析      │
              │  ───────────────────── │
              │  • 识别新增/修改的函数   │
              │  • 分析函数复杂度       │
              │  • 标记需要测试的路径   │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 2: 单元测试生成  │
              │  ───────────────────── │
              │  • 为每个新增/修改函数生成 │
              │  • 正常路径测试          │
              │  • 边界条件测试          │
              │  • 异常路径测试          │
              │  • 遵循项目的测试框架和风格│
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 3: 集成测试生成  │
              │  ───────────────────── │
              │  • 基于接口定义生成     │
              │  • Mock 外部依赖       │
              │  • 测试交互场景        │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 4: 覆盖率分析    │
              │  ───────────────────── │
              │  • 运行所有测试         │
              │  • 分析覆盖率报告       │
              │  • 标记未覆盖的分支     │
              │  • 补充缺失的测试       │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 5: 安全扫描      │
              │  ───────────────────── │
              │  • 依赖版本检查         │
              │  • 已知漏洞匹配         │
              └──────────┬────────────┘
                         ▼
              输出 (测试代码 + 覆盖率报告)
```

### 5.6 输入/输出 Schema

```python
# ==== 输入 (generate-unit-tests) ====
{
    "changed_files": [
        {
            "path": str,
            "content": str,
            "language": str,
        }
    ],
    "test_framework": "pytest" | "jest" | "unittest" | "go-test" | ...,
    "coverage_threshold": float,   # 目标覆盖率（如 80）
}

# ==== 输出 (unit-tests artifact) ====
{
    "test_files": [
        {
            "path": str,           # 测试文件路径
            "content": str,        # 测试代码全文
            "coverage_target": str,  # 覆盖的目标函数
        }
    ],
    "coverage_analysis": {
        "estimated_coverage": float,
        "uncovered_branches": [{"file": str, "line": int, "condition": str}],
        "supplemental_tests": [{"file": str, "content": str}],
    },
}

# ==== 输出 (coverage-report artifact) ====
{
    "overall_coverage": float,
    "file_coverage": [
        {"path": str, "coverage": float, "missed_lines": [int]},
    ],
    "recommendations": [str],
}
```

---

## 6. 上线 Agent (Deployment Agent)

### 6.1 定位

**Agent 名称:** `deployment-agent`

上线 Agent 是流水线的**最后一站**。它确保每次发布是安全、可控、可回滚的。它不直接操作生产环境（那是 CI/CD 工具的事），而是提供**决策支持和风险控制**。

### 6.2 边界

| ✅ 在范围内 | ❌ 不在范围内 |
|------------|-------------|
| 变更影响评估（影响哪些服务/API/数据） | 直接操作生产服务器 |
| 发布策略建议（蓝绿/灰度/金丝雀/全量） | 管理 CI/CD 流水线（只给建议） |
| 回滚方案生成 | 创建基础设施 |
| Release Notes 自动生成 | 监控告警配置 |
| 上线前检查清单 | 替代 SRE 做最终决策 |

### 6.3 Capabilities

```
assess-impact
  └─ 输入: 代码变更 + 测试报告 + 线上拓扑
  └─ 输出: 变更影响评估 + 风险等级

generate-release-plan
  └─ 输入: 变更信息 + 影响评估
  └─ 输出: 发布策略 + 回滚方案 + 检查清单

generate-release-notes
  └─ 输入: 变更历史 + Commit Messages
  └─ 输出: Release Notes
```

### 6.4 交付物 (Artifacts)

| Artifact | 类型 | 内容 |
|----------|------|------|
| `impact-assessment` | report | 变更影响评估：受影响服务/API/数据 + 风险等级 |
| `release-plan` | report | 发布计划：策略、窗口、检查清单 |
| `rollback-plan` | report | 回滚方案：步骤、影响、预计时间 |
| `release-notes` | doc | 自动生成的 Release Notes |
| `preflight-checklist` | report | 上线前检查清单（逐项确认） |

### 6.5 工作流

```
         输入 (代码变更 + 测试报告 + 线上架构图)
                           │
                           ▼
              ┌───────────────────────┐
              │  Step 1: 变更影响评估  │
              │  ───────────────────── │
              │  • 识别变更影响的模块    │
              │  • 标记下游依赖方       │
              │  • 评估回滚复杂度       │
              │  • 标注变更风险等级     │
              │    (低/中/高/灾难)     │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 2: 发布策略     │
              │  ───────────────────── │
              │  • 根据风险推荐发布方式  │
              │    低风险 → 全量发布    │
              │    中风险 → 灰度发布    │
              │    高风险 → 金丝雀发布  │
              │    灾难  → 暂缓 + 人工  │
              │  • 建议发布时间窗口     │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 3: 回滚方案      │
              │  ───────────────────── │
              │  • 回滚步骤（逐条）     │
              │  • 回滚影响范围        │
              │  • 预计回滚时间        │
              │  • 回滚验证方式        │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 4: 上线前检查    │
              │  ───────────────────── │
              │  • 测试是否全部通过？   │
              │  • Code Review 是否通过？│
              │  • 安全扫描是否通过？   │
              │  • 性能基准是否达标？   │
              │  • 是否有相关团队通知？ │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │  Step 5: Release Notes │
              │  ───────────────────── │
              │  • 汇总本次变更        │
              │  • 标注 Breaking Change │
              │  • 生成 Release Notes  │
              └──────────┬────────────┘
                         ▼
              输出 (发布计划 + 回滚方案 + Release Notes)
```

### 6.6 输入/输出 Schema

```python
# ==== 输入 (assess-impact) ====
{
    "changes": [
        {
            "file_path": str,
            "change_type": "add" | "modify" | "delete",
            "service": str,
            "api_changed": bool,
            "db_migration": bool,
            "config_changed": bool,
        }
    ],
    "test_results": {
        "unit_pass": bool,
        "integration_pass": bool,
        "coverage_pct": float,
    },
    "review_results": {
        "approved": bool,
        "blockers": [str],
    },
    "topology": {           # 线上服务拓扑
        "services": [str],
        "dependencies": [{"from": str, "to": str}],
    },
}

# ==== 输出 (release-plan artifact) ====
{
    "risk_level": "low" | "medium" | "high" | "disaster",
    "recommended_strategy": "full-release" | "canary" | "blue-green" | "hold",
    "release_window": str,
    "affected_services": [str],
    "affected_apis": [str],
    "rollback": {
        "steps": [str],
        "estimated_time_minutes": int,
        "verification_steps": [str],
    },
    "preflight_checklist": [
        {"item": str, "status": "pass" | "fail" | "manual-check"},
    ],
    "notifications": [str],          # 需要通知的团队/个人
}


# ==== 输出 (release-notes artifact) ====
{
    "version": str,
    "date": str,
    "summary": str,
    "features": [str],
    "bug_fixes": [str],
    "breaking_changes": [{"description": str, "migration_guide": str}],
    "dependency_changes": [str],
    "credits": [str],                # 贡献者
}
```

---

## 7. 编排引擎 (Orchestrator)

### 7.1 定位

编排引擎不是"一个 Agent"，而是所有 Agent 的**运行环境**。它负责：
- Agent 的注册和发现
- 流水线的调度和执行
- Agent 间上下文传递
- 人工审批节点的管理
- 审计日志

### 7.2 职责

| 职责 | 说明 |
|------|------|
| **Agent 注册中心** | 管理所有 Agent 实例，通过 manifest.name 定位 |
| **流水线调度** | 按配置顺序执行 stages，支持跳過/并發 |
| **上下文传递** | 自动将前一个 Agent 的输出作为后一个的上下文 |
| **审批节点** | 在指定 stage 前暂停、等待人工确认 |
| **状态追踪** | 记录每个 stage 的执行状态、耗时、结果 |
| **审计日志** | 记录所有 Agent 的输入输出 |

### 7.3 关键设计

```
编排引擎的核心是 PipelineConfig:

PipelineConfig {
    stages: [
        { stage: "requirement", enabled: true,  agent_name: "requirement-agent" },
        { stage: "design",      enabled: true,  agent_name: "design-agent" },
        { stage: "coding",      enabled: true,  agent_name: "coding-agent" },
        { stage: "testing",     enabled: true,  agent_name: "testing-agent" },
        { stage: "deployment",  enabled: false, agent_name: "deployment-agent" },
    ],
    human_approval: {
        requirement: true,   # 需求阶段完成后需要人确认
        design: true,        # 设计评审需要人拍板
        coding: false,       # 编码后直接进测试
        testing: false,      # 测试后直接进上线
        deployment: true,    # 上线前需要审批
    }
}
```

### 7.4 "拼装"的实现

拼装 = 改变 `PipelineConfig` + 决定注册哪些 Agent。

```python
# 小型团队（10-50人）
orchestrator.register_agent(RequirementAgent())
orchestrator.register_agent(CodingAgent())
orchestrator.register_agent(DeploymentAgent())
# 不注册 DesignAgent 和 TestingAgent，流水线自动跳過

# 中型团队（50-100人）
orchestrator.register_agent(RequirementAgent())
orchestrator.register_agent(DesignAgent())
orchestrator.register_agent(CodingAgent())
orchestrator.register_agent(TestingAgent())
orchestrator.register_agent(DeploymentAgent())
```

---

## 8. Agent 间协作协议

### 8.1 上下文传递链

```
需求 Agent ──→ 设计 Agent ──→ 编码 Agent ──→ 测试 Agent ──→ 上线 Agent
    │              │              │              │              │
    │ task-        │ tech-        │ code-        │ test-        │ release-
    │ decomposi-   │ design       │ changes      │ reports      │ plan +
    │ tion         │              │              │              │ rollback
    ▼              ▼              ▼              ▼              ▼
PipelineContext.previous_outputs = {
    "requirement": { ... AgentOutput ... },
    "design":      { ... AgentOutput ... },
    "coding":      { ... AgentOutput ... },
    "testing":     { ... AgentOutput ... },
    "deployment":  { ... AgentOutput ... },  # ← 最终输出
}
```

### 8.2 通信规则

1. **不跨阶段反向依赖**：编码 Agent 不依赖测试 Agent 的输出，上线 Agent 不依赖需求 Agent 的原始输出（已通过上下文链传递）
2. **输出不可变性**：Agent 一旦产生输出，不允许修改。如果需要修正在前一阶段的问题，使用新 Task 而非修改已有输出
3. **失败传染**：如果需求 Agent 失败，整个流水线终止。如果测试 Agent 失败（测试没通过），编码阶段需要重新执行（人工决定）
4. **审批节点语义**：当 `human_approval.requirement = true` 时，需求 Agent 执行完毕后编排引擎暂停，等待外部确认信号。确认方式后续定义（CLI 确认 / API 确认 / Webhook）

### 8.3 错误处理策略

| 场景 | 行为 |
|------|------|
| Agent 内部错误 | 标记 stage 为 failed，流水线终止 |
| 超时 | AgentErrorCode.TIMEOUT，标记失败 |
| LLM API 错误 | AgentErrorCode.MODEL_ERROR，自动重试 2 次 |
| 人工审批超时 | 保持等待（不自动跳过） |
| 输入校验失败 | AgentErrorCode.VALIDATION_ERROR，需要上游修复 |

---

## 后续步骤

1. **确认这个设计** — 你觉得每个 Agent 的边界和交付物定义是否合理？有没有要调整的？
2. **选择首个实现目标** — 确认后，从哪个 Agent 开始实现？（建议从编码 Agent 开始，价值最直接）
3. **按照实现计划** — 回到 `docs/superpowers/plans/2026-06-03-agent-scaffolding.md` 开始编码
