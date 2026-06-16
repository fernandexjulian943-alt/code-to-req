# Code-to-Req: AI 代码架构审计引擎

> 将任意代码仓库转化为专业的架构审计报告 —— 包含架构图、知识图谱和结构化需求文档。

**为什么不直接问 AI？** 这个 Skill 产出的是标准化、多文档的审计报告，不是聊天回复。包含架构图（Mermaid）、知识图谱（JSON + 可视化）、健康度评分和可执行的需求建议。输出可直接用于汇报和立项，不只是开发笔记。

## 快速开始

### 1. 安装

```bash
# 克隆到 Claude Code 的 skills 目录
git clone https://github.com/fernandexjulian943-alt/code-to-req.git ~/.claude/skills/code-to-req
```

安装完成后，在任何 Claude Code 会话中输入 `/code-to-req` 即可使用。

### 2. 基本用法

```bash
# 在你要分析的项目目录下启动 Claude Code，然后：

/code-to-req scan                          # 快速扫描：项目画像 + 架构图 + 知识图谱 + 功能清单
/code-to-req plan 添加用户权限管理           # 扫描 + 针对需求的阅读规划
/code-to-req 添加用户权限管理               # 完整流程：扫描 → 分析 → 多角色讨论 → 报告
```

### 3. 进阶用法：三层钻取

```bash
# 第一层：全局骨架（适合新人/PM 快速了解项目）
/code-to-req scan

# 第二层：某个功能的详细需求（适合需求评审/工作量评估）
/code-to-req --feature "用户认证"

# 第三层：某个需求的代码级实现细节（适合开发者改造评估）
/code-to-req --feature "用户认证" --req "token刷新"
```

**指定其他项目**（不在当前目录时）：
```bash
/code-to-req --path /path/to/project scan
/code-to-req --path /path/to/project --feature "某功能"
```

## 输出物

所有产出写入目标项目的 `docs/code-analysis/` 目录：

```
📦 docs/code-analysis/
├── README.md                  ← 报告入口（健康度评分 + 关键发现 + 目录索引）
├── project-profile.md         ← 项目画像（技术栈/规模/模块/健康度评分）
├── architecture-diagrams.md   ← 5 张架构图（系统层级/模块依赖/数据流/时序/ER）
├── knowledge-graph.md         ← 知识图谱（实体/关系/JSON 数据）
├── feature-map.md             ← 功能清单（用户视角的能力地图）
├── reading-plan.md            ← 智能阅读计划（按需求定制）
├── requirements-draft.md      ← 需求草稿（多角色讨论结论）
├── tech-proposal.md           ← 技术方案（改动范围/工作量/风险）
└── features/                  ← 功能钻取（按需生成）
    ├── {功能名}.md            ← 用户故事 + 验收标准 + 数据模型 + API
    └── {功能名}/
        └── req-{需求}.md     ← 函数级调用链 + 改动点 + 工作量
```

## 三层能力说明

| 层级 | 命令 | 适合谁 | 产出 |
|------|------|--------|------|
| **scan** | `/code-to-req scan` | PM、新人、技术负责人 | 项目全貌：画像 + 5张图 + 知识图谱 + 功能清单 |
| **--feature** | `/code-to-req --feature "xx"` | PM、技术评审 | 单功能需求文档：用户故事 + 验收标准 + 边界条件 |
| **--req** | `--feature "xx" --req "yy"` | 开发者、架构师 | 代码级实现：调用链 + 函数清单 + 改造方案 + 工作量 |

## 推荐模型

无需额外配置 API Key，Skill 使用你当前 Claude Code 会话的模型。

| 场景 | 推荐模型 | 说明 |
|------|---------|------|
| **最佳体验** | Claude Opus 4.6+ | 多角色讨论深度好，架构分析准确 |
| **性价比** | Claude Sonnet 4.6 | 全流程可用，讨论深度略降 |
| **国内首选** | DeepSeek V4 Pro | 代码理解强，中文输出好，成本低 |
| **国内备选** | Qwen 3.6 Plus | 中文理解最强，代码能力稍弱 |
| **不建议** | Haiku / 小参数模型 | 多角色讨论质量不足 |

## Demo：claw-code 项目分析

`examples/claw-code/` 目录包含对一个 Rust CLI 项目（~1000 文件）的完整分析产出：

- **scan 层**：[项目画像](examples/claw-code/project-profile.md) | [架构图](examples/claw-code/architecture-diagrams.md) | [知识图谱](examples/claw-code/knowledge-graph.md) | [功能清单](examples/claw-code/feature-map.md)
- **feature 层**：[上下文管理](examples/claw-code/features/context-management.md) | [会话恢复](examples/claw-code/features/session-resume.md)
- **req 层**：[Token 估算实现细节](examples/claw-code/features/context-management/req-token-estimation.md)

## 工作流程

```
阶段 0：规模评估 → 自动选择扫描策略（全量/采样/模块级）
阶段 1：代码扫描（无 AI 调用，纯静态分析）
阶段 1.5：验证性读取 → 架构图 + 知识图谱 + 功能清单
   ↑ /code-to-req scan 到此结束
阶段 2：智能阅读规划（AI 根据需求定制阅读路径）
   ↑ /code-to-req plan 到此结束
阶段 3：渐进代码读取 + 深度理解
阶段 4：多角色协作讨论（PM / 架构师 / 开发工程师）
阶段 5：报告生成
```

## 反幻觉机制

所有输出经过验证：
- 架构图每条连线必须有代码证据（import / 函数调用 / 配置项）
- 知识图谱关系标注置信度来源（grep 行号 / 文件读取验证）
- `--feature` 和 `--req` 模式在写入前执行 Step 3.5 批量验证（sed/grep 确认行号、计数、字段）
- 无法验证的信息不写入，宁可报告内容少也不编造

## 适合谁

- **产品经理**：接手老系统，快速理解全貌，产出改进需求
- **技术负责人**：新项目评估，快速出架构报告用于汇报
- **架构师**：系统改造前的现状审计，改动影响评估
- **新人入职**：比看 wiki 快 10 倍的项目理解方式
- **开发者**：改造前评估工作量，定位精确修改点

## 常见问题

**Q: 分析一个项目大概需要多久？**
- scan：2-5 分钟（取决于项目大小）
- 完整流程：10-20 分钟
- --feature / --req：3-8 分钟

**Q: 大项目（1000+ 文件）能用吗？**
能。自动切换采样策略，只读关键模块，不会超 token 限制。

**Q: 报告准确率如何？**
经反幻觉机制验证后，行号/调用关系/数据结构准确率 ~98%。剩余 2% 主要是跨文件间接引用可能遗漏。

**Q: 支持哪些语言/框架？**
TypeScript/JavaScript、Python、Java、Go、Rust、Ruby、PHP、C#、Vue、Svelte。框架自动识别（Express、FastAPI、Spring Boot、Gin 等）。

## License

MIT
