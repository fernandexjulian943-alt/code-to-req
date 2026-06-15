---
name: code-to-req
description: >
  Scan a codebase, understand its architecture, and generate professional audit reports with architecture diagrams, knowledge graphs, and structured requirement documents.
  Use when: the user wants to understand an existing codebase, plan changes, or generate requirements/tech proposals based on code analysis.
  Trigger: /code-to-req
---

# Code-to-Req: AI 代码架构审计引擎

代码仓库的自动化架构审计报告生成器。通过三层扫描（项目索引 → 智能规划 → 渐进读取）、自动图表生成（架构图 + 知识图谱）和多角色协作（PM / 架构师 / 开发工程师），输出专业级审计报告。

## 环境要求

**无需额外配置**：Skill 使用你当前 Claude Code 会话的模型，无需单独配 API Key。

| 场景 | 推荐模型 | 说明 |
|---|---|---|
| **最佳体验** | Claude Opus 4.6+ | 多角色讨论深度好，架构分析准确 |
| **性价比** | Claude Sonnet 4.6 | 可完成全流程，多角色讨论质量略降 |
| **国内用户（首选）** | DeepSeek V4 Pro | 代码理解强，中文输出好，成本低 |
| **国内用户（备选）** | Qwen 3.6 Plus | 中文理解最强，代码能力稍弱于 DeepSeek |
| **不建议** | Haiku / 小参数模型 | 多角色讨论和深度分析质量不足 |

> 国内用户可通过 `claude --model` 切换模型，或使用 OpenAI 兼容端点接入 DeepSeek/Qwen。

## 使用方式

```
/code-to-req <需求描述>          # 完整流程：扫描 → 图表 → 分析 → 讨论 → 输出报告
/code-to-req scan               # 仅扫描：生成项目画像 + 架构图 + 知识图谱
/code-to-req plan <需求描述>     # 扫描 + 规划：生成阅读计划
```

## 输出物

所有产出写入 `docs/code-analysis/` 目录：

```
📦 docs/code-analysis/
├── README.md                  ← 报告入口（健康度评分 + 关键发现 + 目录）
├── project-profile.md         ← 项目画像（技术栈/规模/模块/健康度）
├── architecture-diagrams.md   ← 架构图集（5种 Mermaid 图）
├── knowledge-graph.md         ← 知识图谱（实体/关系/JSON 数据）
├── reading-plan.md            ← 智能阅读计划
├── requirements-draft.md      ← 需求草稿
└── tech-proposal.md           ← 技术方案
```

## 执行流程

收到用户指令后，按以下流程执行。每个阶段完成后输出进度提示。

---

### 阶段零：规模评估 + 策略选择

**目标**：快速判断项目规模，选择对应的扫描策略。

**执行步骤**：

1. **获取项目基本信息**
   ```bash
   # git commit hash
   git rev-parse --short HEAD 2>/dev/null || echo "non-git"
   
   # 统计源码文件数（排除常见无关目录）
   find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" \
     -o -name "*.py" -o -name "*.java" -o -name "*.go" -o -name "*.rs" \
     -o -name "*.rb" -o -name "*.php" -o -name "*.cs" -o -name "*.vue" -o -name "*.svelte" \) \
     -not -path "*/node_modules/*" -not -path "*/.git/*" -not -path "*/dist/*" \
     -not -path "*/build/*" -not -path "*/vendor/*" -not -path "*/__pycache__/*" \
     -not -path "*/.venv/*" -not -path "*/target/*" -not -path "*/.next/*" \
     -not -path "*/generated/*" | wc -l
   ```

2. **选择扫描策略**

| 文件数 | 策略 | 说明 |
|--------|------|------|
| < 100 | 全量扫描 | 所有文件进入分析 |
| 100-300 | 全量结构 + 重点代码 | 结构全扫，代码只读入口+核心服务 |
| 300-1000 | 采样扫描 | 每模块取 top 3 文件（按 import 数排序） |
| > 1000 | 模块级画像 | 先生成模块级概览，提示用户可指定模块深入 |

3. **智能文件过滤**（所有策略通用）

排除规则（除 .gitignore 外）：
- 测试文件：`**/*.test.*` / `**/*.spec.*` / `**/__tests__/**`
- 类型声明：`**/*.d.ts`（保留 `types/index.d.ts` 等定义文件）
- 生成代码：`**/generated/**` / `**/.next/**` / `**/dist/**` / `**/build/**`
- 锁文件：`package-lock.json` / `yarn.lock` / `pnpm-lock.yaml`
- 静态资源：`**/*.png/jpg/svg/ico/woff/woff2/mp3/mp4`
- 避免循环：`**/docs/code-analysis/**`（不读自己的输出）

输出进度：`📊 规模评估：{N} 个源码文件，采用「{策略名}」策略`

---

### 阶段一：项目索引扫描（不调 AI，纯代码分析）

**目标**：快速生成项目全局画像。

**执行步骤**：

1. **识别技术栈**
   ```bash
   find . -maxdepth 2 -name "package.json" -o -name "pom.xml" -o -name "build.gradle" \
     -o -name "go.mod" -o -name "requirements.txt" -o -name "pyproject.toml" \
     -o -name "Cargo.toml" -o -name "composer.json" -o -name "Gemfile" \
     -o -name "*.csproj" -o -name "*.sln" | head -20
   ```

2. **扫描目录结构**（应用过滤规则后的文件列表）
   ```bash
   find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" \
     -o -name "*.py" -o -name "*.java" -o -name "*.go" -o -name "*.rs" \
     -o -name "*.rb" -o -name "*.php" -o -name "*.cs" -o -name "*.vue" -o -name "*.svelte" \) \
     -not -path "*/node_modules/*" -not -path "*/.git/*" -not -path "*/dist/*" \
     -not -path "*/build/*" -not -path "*/vendor/*" -not -path "*/__pycache__/*" \
     -not -path "*/.venv/*" -not -path "*/target/*" -not -path "*/.next/*" \
     -not -path "*/*.test.*" -not -path "*/*.spec.*" -not -path "*/__tests__/*" \
     -not -path "*/generated/*" | head -500
   ```

3. **提取 API 路由/端点**
   ```bash
   # Express/Koa/Fastify
   grep -rn "router\.\(get\|post\|put\|delete\|patch\)\|app\.\(get\|post\|put\|delete\)" \
     --include="*.ts" --include="*.js" -l 2>/dev/null
   
   # Flask/FastAPI/Django
   grep -rn "@app\.\(route\|get\|post\)\|@router\.\|path(" \
     --include="*.py" -l 2>/dev/null
   
   # Spring Boot
   grep -rn "@\(Get\|Post\|Put\|Delete\|Request\)Mapping\|@RestController" \
     --include="*.java" -l 2>/dev/null
   
   # Go (gin/echo/chi)
   grep -rn "\.GET\|\.POST\|\.PUT\|\.DELETE\|HandleFunc" \
     --include="*.go" -l 2>/dev/null
   ```

4. **提取数据模型/表结构**
   ```bash
   find . -path "*/migration*" -name "*.sql" -o -path "*/schema*" -name "*.sql" | head -20
   grep -rn "CREATE TABLE\|class.*Model\|@Entity\|Schema(\|define(" \
     --include="*.sql" --include="*.ts" --include="*.py" --include="*.java" -l 2>/dev/null
   ```

5. **分析模块间依赖**
   ```bash
   grep -rn "^import \|^from \|require(" --include="*.ts" --include="*.js" \
     --include="*.py" --include="*.java" 2>/dev/null | head -500
   ```

6. **读取关键配置文件**
   - 构建配置的依赖列表
   - 入口文件（main/index/app）
   - 环境变量模板（.env.example）

7. **健康度评估**（基于扫描数据，不调 AI）
   - 架构合理性：检测是否有明确分层（routes/controllers/services/models）
   - 代码组织：目录结构是否规范，有无超大文件（>500行）
   - 测试覆盖：测试文件占比，是否有测试框架配置
   - 文档完善度：README 是否存在，注释密度
   - 依赖健康：lockfile 是否存在，依赖数量

**输出**：将结果写入 `docs/code-analysis/project-profile.md`，格式见模板。

输出进度：`📊 阶段一完成：{N} 个文件，{M} 个模块，{K} 个 API 端点，健康度 {评级}`

---

### 阶段 1.5：架构图 + 知识图谱生成

**目标**：基于阶段一的扫描数据，生成 5 种架构图 + 知识图谱。

**架构图生成规则**：

1. **系统层级图**（graph TB）
   - 检测到前端框架 → 加入"客户端"层
   - 检测到 API 路由 → 加入"应用层"
   - 检测到数据库连接 → 加入"数据层"
   - 检测到外部 API 调用（fetch/axios/http 向外部域名请求） → 加入"外部服务"层
   - 节点数 ≤ 8

2. **模块依赖图**（graph LR）
   - 从 import 分析结果提取模块间引用
   - **归并规则**：同目录下的文件归并为一个节点
   - 只保留项目内部模块间关系，过滤 node_modules
   - 节点数 ≤ 20

3. **数据流图**（flowchart LR）
   - 选取 API 端点最多的核心模块
   - 追踪请求从 route → controller → service → model/DB 的路径
   - 标注文件名便于定位

4. **API 调用时序图**（sequenceDiagram）
   - 选取一个典型写操作（如创建资源）
   - 展示 Client → Router → Controller → Service → DB → Response

5. **ER 图**（erDiagram）
   - 从 migration/schema 文件提取表结构
   - ≤ 10 张表：全部画出
   - > 10 张表：只画有外键关系的核心表，其余列清单
   - 实体数 ≤ 15

**知识图谱生成规则**：

1. **实体提取**（从阶段一数据）：
   - module：每个一级目录/功能模块
   - service：被多处 import 的核心服务文件
   - api：按资源聚合的 API 端点组
   - data-model：数据库表/ORM 模型
   - external：外部 API/第三方服务

2. **关系提取**：
   - depends_on：A 的代码 import 了 B
   - calls：运行时调用关系（controller 调 service）
   - reads_from / writes_to：服务与数据层的交互
   - extends：继承/实现关系

3. **聚合规则**：
   - 顶层图 ≤ 30 个节点（模块级）
   - 标注每个节点的子实体数
   - 同时输出 Mermaid 文本 + JSON 数据

**输出**：
- `docs/code-analysis/architecture-diagrams.md`
- `docs/code-analysis/knowledge-graph.md`

输出进度：`📊 阶段 1.5 完成：生成 5 张架构图 + 知识图谱（{N} 实体，{M} 关系）`

如果用户指令是 `scan`，到此结束，输出报告入口 README.md。

---

### 阶段二：智能阅读规划

**目标**：根据用户需求 + 项目索引，精确定位需要深入阅读的代码范围。

**执行步骤**：

1. **理解用户需求**：解析需求描述，提取关键词（功能模块、业务概念、技术组件）

2. **索引匹配**：在项目画像中查找相关模块：
   - 按 API 端点匹配
   - 按数据模型匹配
   - 按模块名匹配
   - 按依赖链追溯

3. **生成阅读计划**：按"由外到内"组织轮次：
   - 第 1 轮：入口层（路由定义、控制器）
   - 第 2 轮：业务层（服务、核心逻辑）
   - 第 3 轮：数据层（模型、数据库交互）
   - 第 4 轮（可选）：基础设施层（中间件、工具类）

4. **Token 预算控制**：
   - 每轮读取预算：~50K tokens
   - 超出预算 → 拆成多轮，先读核心文件
   - 标注每个文件的预估 token 数

**输出**：`docs/code-analysis/reading-plan.md`

如果用户指令是 `plan`，到此结束。

---

### 阶段三：渐进代码读取 + 理解

**目标**：按计划逐轮读取代码，构建深度理解。

**执行步骤**：

对阅读计划中的每一轮：

1. **读取本轮文件**：使用 Read 工具（大文件只读关键段落）
2. **生成本轮理解摘要**：
   - 模块职责
   - 核心数据流
   - 与其他模块的交互点
   - 发现的潜在问题或限制
3. **动态调整**：发现计划外的重要依赖 → 追加到后续轮次
4. **累积上下文**：每轮摘要叠加，形成完整理解文档

输出进度：`📖 阶段三进度：第 {X}/{Y} 轮读取中，已理解 {模块名} 模块...`

---

### 阶段四：多角色协作讨论

**目标**：从三个专业视角分析需求的可行性和实现方案。

**角色**：

#### 产品经理视角
基于代码现状 + 用户需求，评估：
- 需求解决什么问题？价值多大？
- 现有系统已支持哪些相关能力？
- 需要新增哪些功能点？优先级？
- 边界场景和异常场景？
- 验收标准？

#### 架构师视角
- 现有架构能否支撑？需要哪些改造？
- 涉及哪些模块？影响传播路径？
- 技术风险（性能、安全、兼容性）？
- 推荐技术方案 + 备选方案？

#### 开发工程师视角
- 具体改哪些文件？每个文件改什么？
- 预估工作量（人天）？
- 实现顺序建议？
- 技术债务影响？测试覆盖？

**执行方式**：
1. 将代码分析结果 + 用户需求注入三个角色 prompt
2. 依次生成三个角色的分析
3. 交叉讨论：识别共识点、分歧点，生成综合建议

---

### 阶段五：报告生成

**目标**：整合所有分析结果，生成完整审计报告。

**生成文件**：

1. **`README.md`** — 报告入口（格式见 report-index 模板）
   - 健康度概览（基于阶段一的评估数据）
   - 关键发现 Top 3
   - 报告目录 + 阅读时间

2. **`requirements-draft.md`** — 需求草稿
   - 基于三角色讨论的综合结果
   - 每个需求项含：标题、背景、目标、验收标准、优先级、工作量

3. **`tech-proposal.md`** — 技术方案
   - 改动概览 + 详细设计 + 数据库/API 变更 + 测试策略 + 风险

**完成后**：展示报告入口 README.md 的内容摘要，告知用户文档位置。

输出进度：`✅ 报告生成完毕，共 7 个文件。入口：docs/code-analysis/README.md`

---

## 输出文档规范

- 所有文档使用中文，Markdown 格式
- 每个文件统一头部：`> 生成时间 | 代码版本 | 分析工具：code-to-req v2`
- Mermaid 图表使用 ```mermaid 代码块
- JSON 数据使用 ```json 代码块
- 文件间使用相对链接互相引用

## 关键原则

1. **不过度读取**：先评估规模，按策略扫描，不盲目全读
2. **图表优先**：能用图表表达的不用文字墙
3. **用户可控**：阅读计划展示后可调整再继续
4. **模块级聚合**：大项目自动归并，保持图表可读性（节点 ≤ 20-30）
5. **健康度量化**：给出 A/B/C/D 评分，不只是描述
6. **可操作的输出**：需求和方案可直接用于立项和开发
