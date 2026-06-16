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
/code-to-req <需求描述>                    # 完整流程（当前目录）
/code-to-req scan                         # 仅扫描：项目画像 + 架构图 + 知识图谱
/code-to-req plan <需求描述>               # 扫描 + 规划：生成阅读计划
/code-to-req --feature <功能名>            # 需求细化：深入某个功能的详细需求
/code-to-req --feature <功能名> --req <需求名>  # 实现细化：精确到函数级的代码实现细节
/code-to-req --path <项目路径> [子命令]     # 指定目标项目路径（可与其他子命令组合）
```

**--path 参数**：默认分析当前工作目录。传入 `--path` 时，以该路径为项目根目录执行分析，输出仍写入该项目下的 `docs/code-analysis/`。

示例：
```
/code-to-req --path /root/ai-pm-workbench scan
/code-to-req --path /root/ai-pm-workbench --feature "用户认证"
```

## 输出物

所有产出写入 `docs/code-analysis/` 目录：

```
📦 docs/code-analysis/
├── README.md                  ← 报告入口（健康度评分 + 关键发现 + 目录）
├── project-profile.md         ← 项目画像（技术栈/规模/模块/健康度）
├── architecture-diagrams.md   ← 架构图集（5种 Mermaid 图）
├── knowledge-graph.md         ← 知识图谱（实体/关系/JSON 数据）
├── feature-map.md             ← 功能清单（用户视角的能力地图）
├── reading-plan.md            ← 智能阅读计划
├── requirements-draft.md      ← 需求草稿
├── tech-proposal.md           ← 技术方案
└── features/                  ← 功能需求细化（按需生成）
    └── {功能名}.md            ← 单功能详细需求文档
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

7. **健康度评估**（基于扫描数据的可量化指标，禁止主观推断）

   | 维度 | 评判标准（纯客观） | 数据来源 |
   |------|-------------------|---------|
   | 架构分层 | 是否存在 routes/controllers/services/models 等分层目录 | find 结果 |
   | 代码组织 | 有无超大文件（>500行），统计文件行数分布 | wc -l |
   | 测试覆盖 | 测试文件数 / 源码文件数的比值，是否有测试框架配置 | find + package.json |
   | 文档完善度 | README 是否存在及字节数，.md 文件数量 | find + wc -c |
   | 依赖健康 | lockfile 是否存在，运行时依赖数量 | find + package.json |

   **评分规则**：每个维度用具体数字说话（如"测试比 13:1"），评级仅作为数字的标签（A≥优秀阈值, B≥良好, C≥及格, D=不及格）。禁止没有数据支撑的定性描述。

**输出**：将结果写入 `docs/code-analysis/project-profile.md`，格式见模板。

输出进度：`📊 阶段一完成：{N} 个文件，{M} 个模块，{K} 个 API 端点，健康度 {评级}`

---

### 阶段 1.5：验证性读取 + 架构图 + 知识图谱

**目标**：读取关键代码片段验证阶段一的推断，然后基于**已验证数据**生成架构图和知识图谱。

#### 步骤 A：验证性代码读取

阶段一只获取了文件名和 import 行。在画图之前，必须读取关键文件以验证调用关系：

1. **入口文件**：读取 main/index/app 入口文件的前 50 行（确认模块加载和初始化逻辑）
2. **路由注册**：读取路由定义文件中实际的 handler 注册代码（确认路由 → handler 映射）
3. **核心调用链**：选取 1 个典型 API handler，读取其完整实现（确认 controller → service → model 的调用路径）
4. **数据层**：如果存在 migration/schema 文件，读取其内容（确认表结构）

```bash
# 示例：读取入口文件前 50 行
head -50 <入口文件>

# 示例：读取一个路由文件确认 handler 映射
head -80 <路由文件>

# 示例：读取一个核心 handler 确认调用链
cat <handler文件> | head -100
```

**关键约束**：后续所有图表中的关系，必须能在本步骤读到的代码中找到对应证据。无法验证的关系不得写入。

#### 步骤 B：架构图生成

**反幻觉规则**（适用于全部 5 张图）：
- 每条连线必须对应一个已验证的事实（import 行 / 函数调用 / 配置项）
- 禁止基于"这种框架通常是这样工作的"进行推断
- 如果某张图的数据不足（如无数据库、无 migration），跳过该图并说明原因

1. **系统层级图**（graph TB）
   - 检测到前端框架（grep 到 React/Vue/Svelte 等 import） → 加入"客户端"层
   - 检测到 API 路由（grep 匹配到路由定义） → 加入"应用层"
   - 检测到数据库连接（grep 到 DB 驱动 import 或连接配置） → 加入"数据层"
   - 检测到外部 API 调用（代码中有 fetch/axios 调用外部域名） → 加入"外部服务"层
   - 节点数 ≤ 8
   - **每层必须标注判断依据**（如"检测到 `require('mongoose')` in db.js:3"）

2. **模块依赖图**（graph LR）
   - 数据来源：阶段一的 import/require grep 结果（**纯事实，无推断**）
   - **归并规则**：同目录下的文件归并为一个节点
   - 只保留项目内部模块间关系，过滤 node_modules
   - 节点数 ≤ 20

3. **数据流图**（flowchart LR）
   - 基于步骤 A 中实际读取的那条 API 调用链
   - 每个节点标注 `文件名:行号`
   - 只画已读取验证过的路径，不补全"应该有"的环节

4. **API 调用时序图**（sequenceDiagram）
   - 基于步骤 A 中实际读取的那个典型 handler
   - 只展示在代码中明确看到的调用关系
   - 每个调用标注来源（如 `app.js:45 调用 userService.create()`）

5. **ER 图**（erDiagram）
   - **前提**：步骤 A 已读取到 migration/schema 文件内容
   - 如果没有 migration/schema 文件 → 跳过此图，注明"未检测到数据库 schema"
   - 从实际读取的 SQL/ORM 定义中提取表结构和外键关系
   - ≤ 10 张表：全部画出
   - > 10 张表：只画有外键关系的核心表，其余列清单
   - 实体数 ≤ 15

#### 步骤 C：知识图谱生成

1. **实体提取**（从阶段一扫描数据）：
   - module：每个一级目录/功能模块
   - service：被多处 import 的核心服务文件
   - api：按资源聚合的 API 端点组（来自 grep 结果）
   - data-model：数据库表/ORM 模型（来自 schema 文件读取）
   - external：外部 npm 包/第三方服务

2. **关系提取**（分级置信度）：

   | 关系类型 | 数据来源 | 写入条件 |
   |---------|---------|---------|
   | depends_on | import/require grep 结果 | 直接写入（确定性 100%） |
   | calls | 步骤 A 读取的代码中明确看到的函数调用 | 标注 `文件:行号` 后写入 |
   | reads_from / writes_to | 步骤 A 读取的代码中明确看到的 DB 操作 | 标注 `文件:行号` 后写入 |
   | extends | grep 到的 extends/implements 关键字 | 直接写入 |

   **禁止项**：不得写入无法标注来源的关系。宁可图谱稀疏，不可编造关系。

3. **聚合规则**：
   - 顶层图 ≤ 30 个节点（模块级）
   - 标注每个节点的子实体数
   - 同时输出 Mermaid 文本 + JSON 数据

#### 步骤 D：功能清单生成

**目标**：从用户/产品视角描述系统能力，让非技术人员能直接基于此提出需求。

**提取信号**（按优先级）：

1. **CLI 命令/子命令**：argparse subcommand、clap subcommand、cobra command 注册
2. **Slash 命令**：注册表中的命令名 + 描述
3. **API 端点**：路由定义按资源分组，翻译为用户能力（如 `POST /users` → "用户注册"）
4. **UI 页面/组件**：前端路由定义、页面级组件
5. **配置能力**：环境变量、feature flag、插件系统
6. **README/文档中的 feature 描述**：项目自带的功能说明

**组织结构**：按"功能域"（用户可感知的能力分组）而非代码模块组织。

```markdown
## 功能域：{域名}

| # | 功能 | 用户价值 | 实现位置 |
|---|------|---------|---------|
| 1 | {功能名} | {一句话说明用户能做什么} | {文件/模块路径} |
```

**生成规则**：
- 功能名用用户语言（"创建会话"而非"instantiate session"）
- 实现位置精确到文件或模块，不写行号（功能粒度比代码行粗）
- 按用户使用频率/重要性排序，不按代码位置排序
- 如果功能域下只有 1 个功能，合并到相近的域
- 功能清单是"这个产品能做什么"的完整回答

**反幻觉规则**：
- 每个功能必须有对应的代码注册点（命令注册、路由注册、组件导出）
- 禁止从 README 营销描述中提取"计划中"的功能
- 只列已实现的能力，标注来源（如 "来自 commands.py 注册表"）

**输出**：`docs/code-analysis/feature-map.md`

---

**阶段 1.5 输出汇总**：
- `docs/code-analysis/architecture-diagrams.md`
- `docs/code-analysis/knowledge-graph.md`
- `docs/code-analysis/feature-map.md`

输出进度：`📊 阶段 1.5 完成：读取 {N} 个关键文件，生成 {X}/5 张架构图 + 知识图谱（{E} 实体，{R} 关系）+ 功能清单（{F} 个功能域）`

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

### --feature 模式：功能需求细化

**触发**：`/code-to-req --feature <功能名>`

**前置条件**：`docs/code-analysis/feature-map.md` 必须已存在。如果不存在，提示用户先执行 `/code-to-req scan`。

**目标**：针对一个具体功能，输出产品级需求文档（用户故事、验收标准、边界条件、数据流、接口定义）。

#### 步骤 1：模糊匹配 + 用户确认

1. 读取 `docs/code-analysis/feature-map.md`
2. 用用户输入的 `<功能名>` 做模糊匹配（关键词包含匹配，不区分大小写）
3. 匹配结果处理：
   - **0 个匹配**：列出所有功能域及功能，请用户重新指定
   - **1 个匹配**：展示匹配结果，请用户确认"是否分析这个功能？"
   - **多个匹配**：列出候选项（编号），请用户选择

```
🔍 找到以下匹配功能：
1. [用户管理] 用户注册 — 实现位置：src/controllers/auth.ts
2. [用户管理] 用户登录 — 实现位置：src/controllers/auth.ts
3. [会话管理] 创建会话 — 实现位置：src/controllers/session.ts

请确认要细化哪个功能（输入编号或功能名）：
```

**等待用户确认后再继续。**

#### 步骤 2：深入代码读取

确认功能后，基于 feature-map 中记录的「实现位置」深入读取：

1. **主文件**：读取功能的核心实现文件（完整读取，不截断）
2. **调用链追溯**：从该文件的 import 追溯上下游依赖（最多 2 层）
3. **数据模型**：如果功能涉及 CRUD，找到对应的 model/schema 定义
4. **API 契约**：提取路由定义、请求参数、响应结构
5. **配置项**：扫描相关环境变量 / feature flag

#### 步骤 3：生成需求细节文档

输出文件：`docs/code-analysis/features/{功能名-slug}.md`

文档结构：

```markdown
# {功能名} — 需求细节

> 生成时间 | 代码版本 | 基于 feature-map 中的功能项

## 功能概述

- **所属功能域**：{域名}
- **用户价值**：{一句话}
- **当前状态**：已实现 / 部分实现 / 桩代码

## 用户故事

| # | 角色 | 行为 | 目的 |
|---|------|------|------|
| US-1 | 作为{角色} | 我想要{行为} | 以便{目的} |

## 验收标准

| # | 场景 | Given | When | Then |
|---|------|-------|------|------|
| AC-1 | {正常场景} | {前置条件} | {操作} | {预期结果} |
| AC-2 | {异常场景} | ... | ... | ... |

## 数据模型

{相关表/模型结构，从代码中提取}

## API 接口

| 方法 | 路径 | 参数 | 响应 | 说明 |
|------|------|------|------|------|

## 业务规则 & 边界条件

- {从代码逻辑中提取的校验规则、限制条件、特殊处理}

## 依赖关系

- 上游：{本功能依赖哪些其他功能/服务}
- 下游：{哪些功能依赖本功能}

## 当前实现摘要

- **核心文件**：{路径列表}
- **代码行数**：{统计}
- **测试覆盖**：{有/无，测试文件路径}
```

**生成规则**：
- 用户故事从代码行为反推，每个独立的操作路径 = 一个故事
- 验收标准从代码中的 if/else 分支、错误处理、校验逻辑反推
- 边界条件从 validation、limit、edge case 处理中提取
- 所有内容必须有代码证据，标注来源文件

#### 步骤 3.5：幻觉验证（写入前必做）

文档草稿生成后、写入文件前，逐项验证以下易错点：

1. **行号验证**：文档中每个 `文件:行号` 引用，用 `sed -n '{行号}p' {文件}` 确认内容匹配
2. **计数验证**：测试数量、文件数量等，用 `grep -c` 或 `wc -l` 确认，不靠记忆
3. **字段完整性**：引用的 struct/enum 定义，重新 grep 原始定义确认字段无遗漏
4. **逻辑方向**：涉及"顺序""方向""先后"的描述，回读代码确认（特别注意 reverse/sort/swap）
5. **常量值**：引用的常量值和行号，用 `grep -n "常量名" 文件` 一次性确认

**执行方式**：将所有待验证项合并为一批 grep/sed 命令并行执行，对比结果修正文档。不通过验证的内容要么修正要么删除，不得带着"可能对"的心态写入。

**验证完成标志**：在文档末尾的生成信息中标注 `验证状态：已通过`。

#### 步骤 4：输出

1. 写入 `docs/code-analysis/features/{slug}.md`
2. 展示文档摘要给用户
3. 提示：可继续 `--feature` 其他功能，或基于此文档提出修改需求

输出进度：`✅ 功能需求细化完成：{功能名}，输出 docs/code-analysis/features/{slug}.md`

---

### --req 模式：需求实现细化

**触发**：`/code-to-req --feature <功能名> --req <需求名>`

**前置条件**：`docs/code-analysis/features/{功能名}.md` 必须已存在。如果不存在，提示用户先执行 `--feature`。

**目标**：针对某个功能下的具体需求（验收标准/用户故事），输出开发者可直接执行的实现级文档：精确到函数、行号、调用链、改动插入点。

#### 步骤 1：需求定位 + 用户确认

1. 读取 `docs/code-analysis/features/{功能名}.md`
2. 从用户故事（US-x）和验收标准（AC-x）中模糊匹配 `<需求名>`
3. 匹配结果处理（同 --feature 逻辑：0/1/多分别处理）
4. 展示匹配结果，等待用户确认

```
🔍 在「会话恢复」中找到以下匹配需求：
1. [US-4] 跨工作区恢复会话
2. [AC-9] 别名引用时允许跨 workspace，打印 note

请确认要细化哪个需求（输入编号）：
```

#### 步骤 2：深度代码追踪

确认需求后，从 feature doc 中记录的「核心文件」和「数据模型」出发，**追踪到函数级**：

1. **入口函数定位**：找到该需求对应的触发入口（命令分发、路由 handler、事件监听）
2. **完整调用链**：从入口向下追踪每一层函数调用（最多 4 层），读取每个函数体
3. **数据流**：追踪入参如何变换、传递、存储
4. **分支逻辑**：找到该需求对应的 if/match 分支，读取完整条件和处理逻辑
5. **错误路径**：该需求失败时的错误处理链
6. **关联测试**：grep 测试文件中引用该函数/该场景的用例

#### 步骤 3：生成实现细节文档

输出文件：`docs/code-analysis/features/{功能名}/req-{需求slug}.md`

文档结构：

```markdown
# {需求名} — 实现细节

> 生成时间 | 代码版本 | 基于 feature doc 中的 {US-x / AC-x}
> 验证状态：{已通过/未通过}

## 需求描述（从 feature doc 引用）

{一句话描述}

## 调用链路图

```mermaid
graph LR
    A[入口: 文件:行号] --> B[函数A: 文件:行号]
    B --> C[函数B: 文件:行号]
    C --> D[最终操作: 文件:行号]
```

## 关键函数清单

| # | 函数签名 | 文件:行号 | 职责 | 输入 | 输出 |
|---|---------|----------|------|------|------|
| 1 | fn xxx(...) | path:line | 一句话 | 参数类型 | 返回类型 |

## 核心逻辑（代码片段）

{逐个关键函数贴出精简代码片段（去掉注释和非核心分支），标注行号}

## 分支与边界

| 条件 | 文件:行号 | 处理 |
|------|----------|------|
| {if/match 条件} | path:line | {走哪条路径} |

## 错误处理

| 错误场景 | 触发条件 | 错误类型 | 用户可见信息 |
|---------|---------|---------|------------|

## 关联测试

| 测试函数 | 文件:行号 | 覆盖场景 |
|---------|----------|---------|

## 改造评估（如需修改此需求）

- **改动点**：{需要修改的函数列表 + 行号范围}
- **插入点**：{新逻辑应该插入的位置}
- **影响范围**：{改动会影响哪些上下游调用者}
- **需同步修改的测试**：{测试文件:函数名}
- **预估工作量**：{行数/复杂度}
```

**生成规则**：
- 调用链路图中每个节点必须标注 `文件:行号`，行号已验证
- 代码片段只保留与该需求直接相关的逻辑，省略无关代码用 `// ...` 标注
- 改造评估基于代码结构推断，标注"基于当前代码结构，未考虑未提交的变更"

#### 步骤 3.5：幻觉验证（同 --feature 模式）

所有行号、函数签名、分支条件用 `sed -n` / `grep` 批量验证。

#### 步骤 4：输出

1. 写入 `docs/code-analysis/features/{功能名}/req-{slug}.md`
2. 展示调用链路图 + 改造评估摘要
3. 提示：可基于此文档直接提出改造需求，或继续 `--req` 其他需求

输出进度：`✅ 需求实现细化完成：{需求名}，输出 docs/code-analysis/features/{功能名}/req-{slug}.md`

---

- 所有文档使用中文，Markdown 格式
- 每个文件统一头部：`> 生成时间 | 代码版本 | 分析工具：code-to-req v2`
- Mermaid 图表使用 ```mermaid 代码块
- JSON 数据使用 ```json 代码块
- 文件间使用相对链接互相引用

## 关键原则

1. **零幻觉**：所有输出的事实（文件名、行数、依赖关系、调用路径）必须有代码证据。未验证的信息不写入报告。宁可报告内容少，不可编造。行号/计数/常量值写入前必须用 grep/sed 再确认一次，不靠读取时的记忆
2. **不过度读取**：先评估规模，按策略扫描，不盲目全读
3. **图表优先**：能用图表表达的不用文字墙
4. **用户可控**：阅读计划展示后可调整再继续
5. **模块级聚合**：大项目自动归并，保持图表可读性（节点 ≤ 20-30）
6. **健康度量化**：给出 A/B/C/D 评分，不只是描述
7. **可操作的输出**：需求和方案可直接用于立项和开发
