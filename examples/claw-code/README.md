# 代码架构审计报告：Claw Code

> 分析时间：2026-06-16 | 代码版本：`d229a9b` | 分析工具：code-to-req v2
> 项目规模：88 Rust 文件 + 68 Python 文件 / 106K 行 / 11 crates
> 验证级别：Rust 依赖来自 Cargo.toml，Python 依赖来自 import 行号

---

## 健康度概览

| 维度 | 评分 | 关键数据 |
|------|------|---------|
| 架构分层 | A | 11 crate workspace，职责边界清晰 |
| 代码组织 | B | runtime 41K 行偏大，commands/lib.rs 7183 行单文件 |
| 测试覆盖 | C | Python 仅 5 个 test 文件；Rust 内联 #[cfg(test)] |
| 文档完善度 | A | README 14KB + USAGE 33KB + PHILOSOPHY + ROADMAP 1MB |
| 依赖健康 | A | workspace.dependencies 统一版本，unsafe_code = "forbid" |

---

## 关键发现

1. **双语言架构**：Rust 是主实现（104K 行），Python `src/` 是"移植工作区"——按模块镜像 Rust CLI 的命令/工具/运行时能力（main.py:22 描述 "Python porting workspace for the Claude Code rewrite effort"）
2. **runtime crate 过大**：41,145 行 / 40+ 个 mod，承载了会话、权限、MCP、LSP、bash、git、config、sandbox 等全部核心逻辑
3. **rusty-claude-cli 是唯一 CLI 入口**：依赖全部 5 个核心 crate（api/commands/runtime/plugins/tools），扇出最广

---

## 报告目录

| # | 文件 | 内容 |
|---|------|------|
| 1 | [project-profile.md](./project-profile.md) | 项目画像（技术栈/规模/crate清单） |
| 2 | [architecture-diagrams.md](./architecture-diagrams.md) | 架构图集（4/5 图，ER 图因无 SQL 跳过） |
| 3 | [knowledge-graph.md](./knowledge-graph.md) | 知识图谱（21 实体 / 50+ 关系） |
| 4 | [feature-map.md](./feature-map.md) | 功能清单（10 域 / 42 个用户功能） |
