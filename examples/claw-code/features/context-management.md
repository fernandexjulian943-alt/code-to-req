# 上下文管理 — 需求细节

> 生成时间：2026-06-16 | 代码版本：`d229a9b` | 基于 feature-map「项目理解 > 上下文管理」

## 功能概述

- **所属功能域**：项目理解
- **用户价值**：查看/调整 AI 的项目上下文，了解当前会话包含了哪些信息，按需管理
- **当前状态**：部分实现（命令解析已完成，执行逻辑标记为 "not yet implemented in this build"）

## 用户故事

| # | 角色 | 行为 | 目的 |
|---|------|------|------|
| US-1 | 作为开发者 | 我想查看当前会话的上下文构成 | 以便了解 AI "知道"哪些项目信息 |
| US-2 | 作为开发者 | 我想清除当前上下文 | 以便从干净状态重新开始，避免旧信息干扰 |
| US-3 | 作为开发者 | 我想知道上下文占用了多少 token | 以便判断是否需要 compact 或清理 |

## 验收标准

| # | 场景 | Given | When | Then |
|---|------|-------|------|------|
| AC-1 | 查看上下文 | 会话已有对话历史和项目文件 | 用户输入 `/context show` | 展示：instruction files 列表、git 分支、暂存文件、最近提交、estimated tokens |
| AC-2 | 默认行为 | 无参数 | 用户输入 `/context` | 等同于 `/context show` |
| AC-3 | 清除上下文 | 会话有项目上下文注入 | 用户输入 `/context clear` | 清除动态上下文（保留系统提示骨架），确认消息告知用户 |
| AC-4 | 无效参数 | - | 用户输入 `/context foo` | 报错提示合法参数为 `[show|clear]` |
| AC-5 | 空会话 | 刚启动，无历史 | 用户输入 `/context show` | 展示基础环境信息（cwd、日期、model），标注"无项目指令文件" |

## 数据模型

从代码中提取的核心结构：

```rust
// runtime/src/prompt.rs:86-93
pub struct ProjectContext {
    pub cwd: PathBuf,
    pub current_date: String,
    pub git_status: Option<String>,
    pub git_diff: Option<String>,
    pub git_context: Option<GitContext>,       // 分支 + 最近提交 + 暂存文件
    pub instruction_files: Vec<ContextFile>,   // CLAUDE.md/.claw/rules 等
}

// runtime/src/prompt.rs:67-70
pub struct ContextFile {
    pub path: PathBuf,
    pub content: String,
}

// runtime/src/git_context.rs:12-17
pub struct GitContext {
    pub branch: Option<String>,
    pub recent_commits: Vec<GitCommitEntry>,  // 最多 5 条
    pub staged_files: Vec<String>,
}

// runtime/src/session.rs:117-133
pub struct Session {
    pub messages: Vec<ConversationMessage>,
    pub compaction: Option<SessionCompaction>,
    pub workspace_root: Option<PathBuf>,
    pub prompt_history: Vec<SessionPromptEntry>,
    pub model: Option<String>,
    // ...
}
```

## API 接口

| 方法 | 路径 | 参数 | 响应 | 说明 |
|------|------|------|------|------|
| Slash 命令 | `/context` | 无 | 展示当前上下文摘要 | 等同于 show |
| Slash 命令 | `/context show` | `action = "show"` | 渲染上下文各组成部分 | 主功能 |
| Slash 命令 | `/context clear` | `action = "clear"` | 确认消息 | 清除动态上下文 |

命令解析入口：`rust/crates/commands/src/lib.rs:1500`
```rust
"context" => SlashCommand::Context { action: remainder }
```

命令规格定义：`rust/crates/commands/src/lib.rs:366-372`
```rust
SlashCommandSpec {
    name: "context",
    aliases: &[],
    summary: "Inspect or manage the conversation context",
    argument_hint: Some("[show|clear]"),
    resume_supported: true,
}
```

## 业务规则 & 边界条件

- **上下文组成**（从 `SystemPromptBuilder::build()` 提取，`prompt.rs:211-233`）：
  1. 静态系统提示（intro + system + doing_tasks + actions）
  2. 动态边界符 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`
  3. 环境信息（model family、cwd、date、platform）
  4. 项目上下文（git status/diff/context + instruction files）
  5. 运行时配置
  6. 追加段落（append_sections）

- **指令文件发现规则**（`prompt.rs:289-314`）：
  - 从 git root 到 cwd 逐层处理（收集时从 cwd 向上遍历，reverse 后从 root 开始加载）
  - 每层查找：`CLAUDE.md` / `CLAW.md` / `AGENTS.md` / `CLAUDE.local.md` / `.claw/CLAUDE.md` / `.claude/CLAUDE.md` / `.claw/instructions.md`
  - 加载 `.claw/rules/` 和 `.claw/rules.local/` 目录下的规则文件
  - 单文件上限 4,000 字符，总计上限 12,000 字符（`prompt.rs:43-44`）

- **Git diff 上限**：50,000 字符（`prompt.rs:45`）

- **Token 估算**：`compact.rs:35` 的 `estimate_session_tokens()` 按消息逐条估算

- **相关命令**：
  - `/compact` — 压缩上下文（已实现）
  - `/files` — 列出上下文中的文件
  - `/focus` / `/unfocus` — 聚焦/取消聚焦特定路径
  - `/add-dir` — 添加目录到分析范围

## 依赖关系

- **上游**（本功能依赖）：
  - `runtime::prompt::ProjectContext` — 上下文数据来源
  - `runtime::prompt::SystemPromptBuilder` — 上下文渲染
  - `runtime::git_context::GitContext` — Git 信息采集
  - `runtime::compact::estimate_session_tokens()` — Token 估算
  - `runtime::session::Session` — 会话状态

- **下游**（依赖本功能）：
  - `/compact` 命令 — 基于 context 体积判断是否需要压缩
  - `/focus` / `/unfocus` — 修改上下文聚焦范围
  - 多轮对话 — 上下文是对话质量的基础

## 当前实现摘要

- **核心文件**：
  - `rust/crates/commands/src/lib.rs` — 命令解析（枚举定义 + 参数提取）
  - `rust/crates/rusty-claude-cli/src/main.rs:8235` — 执行分发（当前返回 "not yet implemented"）
  - `rust/crates/runtime/src/prompt.rs` — 上下文构建逻辑（`ProjectContext` + `SystemPromptBuilder`）
  - `rust/crates/runtime/src/git_context.rs` — Git 上下文采集
  - `rust/crates/runtime/src/compact.rs` — Token 估算

- **代码行数**：命令框架 ~20 行（解析+分发），上下文基础设施 ~600 行（prompt.rs + git_context.rs + compact.rs）
- **测试覆盖**：`git_context.rs` 有 6 个单元测试；`/context` 命令本身无执行测试（因为未实现）
