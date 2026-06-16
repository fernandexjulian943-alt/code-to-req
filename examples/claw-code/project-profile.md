# 项目画像：Claw Code

> 生成时间：2026-06-16 | 代码版本：`d229a9b` | 分析工具：code-to-req v2
> 分析范围：/root/test-projects/claw-code | 策略：全量结构 + 重点代码

## 技术栈

| 维度 | 内容 | 依据 |
|------|------|------|
| 语言 | Rust (主) + Python (移植层) | rust/Cargo.toml workspace + src/*.py |
| 构建工具 | Cargo workspace (resolver=2) | rust/Cargo.toml |
| 包管理 | Cargo (Rust) | workspace.package.version = "0.1.3" |
| 数据库 | Qdrant (向量数据库，RAG 服务用) | claw-rag-service/src/qdrant_index.rs |
| 测试框架 | cargo test (Rust) + pytest (Python) | CLAUDE.md 验证命令 |

## 项目规模

| 指标 | 数值 | 来源 |
|------|------|------|
| Rust 源码文件 | 88 个 | find rust/crates -name "*.rs" |
| Rust 代码行数 | 104,432 行 | wc -l 汇总 |
| Python 源码文件 | 68 个 | find src -name "*.py" |
| Python 代码行数 | 1,933 行 | wc -l src/*.py |
| Rust crates | 11 个 | rust/crates/ 子目录 |
| 测试文件 | 5 个 (Python tests/) | find tests -name "*.py" |

## 健康度评分

| 维度 | 评分 | 数据 |
|------|------|------|
| 架构分层 | A | 11 个 crate 职责分明，workspace 组织 |
| 代码组织 | B | runtime crate 41,145 行偏大；commands 7,183 行单文件(lib.rs) |
| 测试覆盖 | C | Python 侧仅 5 个测试文件；Rust 有内联 #[cfg(test)] 模块 |
| 文档完善度 | A | README 13.8KB + USAGE 33KB + CONTRIBUTING + PHILOSOPHY + ROADMAP |
| 依赖健康 | A | Cargo.toml 管理，workspace.dependencies 共享版本 |

## Rust Crate 清单

| Crate | 行数 | 职责 |
|-------|------|------|
| **runtime** | 41,145 | 核心运行时：会话/权限/MCP/LSP/bash/git/config/sandbox |
| **rusty-claude-cli** | 22,146 | CLI 前端：输入/渲染/setup wizard/主循环 |
| **tools** | 11,633 | 工具实现：lane completion / PDF extract |
| **api** | 9,397 | API 客户端：Anthropic+OpenAI 兼容/SSE/prompt cache |
| **commands** | 7,183 | 命令处理：slash commands/skills/agents |
| **claw-analog** | 4,832 | 类似工具兼容层：agents/config/doctor |
| **plugins** | 4,500 | 插件系统：hooks/lifecycle/isolation |
| **claw-rag-service** | 1,149 | RAG 服务：chunk/embed/ingest/qdrant |
| **mock-anthropic-service** | 1,157 | 测试用 mock Anthropic API |
| **telemetry** | 526 | 遥测数据收集 |
| **compat-harness** | 363 | 兼容性测试工具 |

## Python 移植层（src/）

| 模块 | 文件 | 职责（from import 分析） |
|------|------|------|
| main.py | CLI 入口 | argparse 子命令路由 |
| runtime.py | 运行时 | PortRuntime: route/bootstrap/turn-loop |
| query_engine.py | 查询引擎 | QueryEnginePort: session/submit/persist |
| commands.py | 命令 | PORTED_COMMANDS 注册表 |
| tools.py | 工具 | PORTED_TOOLS 注册表 |
| models.py | 数据模型 | PortingModule/PermissionDenial/UsageSummary |
| session_store.py | 会话存储 | load_session/save_session |
| permissions.py | 权限 | ToolPermissionContext/PathScope |
| setup.py | 启动 | deferred_init + prefetch |
| execution_registry.py | 执行注册 | command/tool 执行器绑定 |

## 入口文件

| 文件 | 说明 |
|------|------|
| `rust/crates/rusty-claude-cli/src/main.rs` | Rust CLI 入口 |
| `src/main.py` | Python 移植层入口 |
