# 架构图集：Claw Code

> 生成时间：2026-06-16 | 代码版本：`d229a9b` | 分析工具：code-to-req v2
> 反幻觉：所有连线标注 Cargo.toml 或代码行号

---

## 1. 系统层级图

```mermaid
graph TB
    subgraph CLI 层
        CLI[rusty-claude-cli<br/>22,146行 · 用户交互]
    end

    subgraph 命令/工具层
        Commands[commands<br/>7,183行 · slash commands]
        Tools[tools<br/>11,633行 · 工具实现]
    end

    subgraph 核心运行时
        Runtime[runtime<br/>41,145行 · 会话/权限/MCP]
        Plugins[plugins<br/>4,500行 · 插件系统]
    end

    subgraph API 层
        API[api<br/>9,397行 · LLM 客户端]
        Telemetry[telemetry<br/>526行]
    end

    subgraph 服务层
        RAG[claw-rag-service<br/>1,149行 · 向量搜索]
        Analog[claw-analog<br/>4,832行 · 兼容层]
    end

    CLI --> Commands
    CLI --> Tools
    CLI --> Runtime
    CLI --> Plugins
    CLI --> API
    Commands --> Runtime
    Commands --> Plugins
    Tools --> Runtime
    Tools --> API
    Tools --> Commands
    Tools --> Plugins
    Runtime --> Plugins
    Runtime --> Telemetry
    API --> Runtime
    API --> Telemetry
    Analog --> API
    Analog --> Runtime
```

**层级判断依据**：
- CLI 层：rusty-claude-cli/Cargo.toml 依赖 5 个内部 crate（最多）
- 命令/工具层：commands + tools 依赖 runtime + plugins
- 核心运行时：runtime 依赖 plugins + telemetry
- API 层：api 依赖 runtime + telemetry
- 无前端框架、无传统数据库（Qdrant 在 RAG 服务中）

---

## 2. 模块依赖图（Rust Crate 间）

```mermaid
graph LR
    cli[rusty-claude-cli] -->|Cargo.toml| api
    cli -->|Cargo.toml| commands
    cli -->|Cargo.toml| runtime
    cli -->|Cargo.toml| plugins
    cli -->|Cargo.toml| tools
    commands -->|Cargo.toml| plugins
    commands -->|Cargo.toml| runtime
    tools -->|Cargo.toml| api
    tools -->|Cargo.toml| commands
    tools -->|Cargo.toml| plugins
    tools -->|Cargo.toml| runtime
    api -->|Cargo.toml| runtime
    api -->|Cargo.toml| telemetry
    runtime -->|Cargo.toml| plugins
    runtime -->|Cargo.toml| telemetry
    analog[claw-analog] -->|Cargo.toml| api
    analog -->|Cargo.toml| runtime
    compat[compat-harness] -->|Cargo.toml| commands
    compat -->|Cargo.toml| tools
    compat -->|Cargo.toml| runtime
    mock[mock-anthropic-service] -->|Cargo.toml| api
```

**数据来源**：每个 crate 的 `Cargo.toml` 中 `path = "../xxx"` 依赖声明。

---

## 3. 数据流图（Python 移植层）

**场景：用户执行 `claw turn-loop "write tests"` 的调用链（基于 main.py + runtime.py 实际代码）**

```mermaid
flowchart LR
    User[用户] -->|"argv"| Main["main.py:94<br/>main(argv)"]
    Main -->|"L96: parse_args"| Parser[argparse]
    Main -->|"L154: PortRuntime()"| RT["runtime.py:89<br/>PortRuntime"]
    RT -->|"L182: QueryEnginePort.from_workspace()"| QE["query_engine.py<br/>QueryEnginePort"]
    RT -->|"L184: route_prompt()"| Route["runtime.py:90<br/>route_prompt()"]
    Route -->|"L94-95: PORTED_COMMANDS/TOOLS"| Registry["commands.py + tools.py"]
    RT -->|"L190: engine.submit_message()"| QE
    QE -->|返回 TurnResult| RT
    RT -->|"L191: results.append()"| Main
    Main -->|"L157: print(result.output)"| User
```

**验证证据**：
- `main.py:154` → `results = PortRuntime().run_turn_loop(args.prompt, ...)`
- `runtime.py:182` → `engine = QueryEnginePort.from_workspace()`
- `runtime.py:184` → `matches = self.route_prompt(prompt, limit=limit)`
- `runtime.py:190` → `result = engine.submit_message(turn_prompt, command_names, tool_names, ())`

---

## 4. API 调用时序图

**场景：`PortRuntime.bootstrap_session()` 的内部执行链（基于 runtime.py 实际代码）**

```mermaid
sequenceDiagram
    participant Main as main.py:151
    participant RT as PortRuntime
    participant QE as QueryEnginePort
    participant Reg as execution_registry

    Main->>RT: bootstrap_session(prompt, limit)
    Note over RT: L137: build_port_context()
    Note over RT: L138: run_setup(trusted=True)
    RT->>RT: L144: route_prompt(prompt)
    Note over RT: 匹配 PORTED_COMMANDS + PORTED_TOOLS
    RT->>Reg: L145: build_execution_registry()
    RT->>Reg: L146: registry.command(name).execute(prompt)
    RT->>Reg: L147: registry.tool(name).execute(prompt)
    RT->>QE: L149-154: stream_submit_message(prompt, commands, tools, denials)
    RT->>QE: L155-160: submit_message(prompt, commands, tools, denials)
    QE-->>RT: TurnResult
    RT->>QE: L161: persist_session()
    QE-->>RT: persisted_session_path
    RT-->>Main: RuntimeSession
```

**验证证据**：所有行号来自 `src/runtime.py` 实际代码（136-179行）。

---

## 5. ER 图

> **跳过**：项目无传统数据库 migration/schema 文件。claw-rag-service 使用 Qdrant 向量数据库（通过 API 交互，无 SQL schema 文件）。
> `find . -path "*/migration*" -name "*.sql" -o -path "*/schema*" -name "*.sql"` 返回空。

---

## 图表阅读指南

| 图表 | 看什么 | 回答什么问题 |
|------|--------|------------|
| 系统层级图 | 分层 | Claw Code 整体架构怎么分层？ |
| 模块依赖图 | crate 关系 | 改一个 crate 会影响哪些下游？ |
| 数据流图 | 请求路径 | 用户输入从 CLI 到 AI 响应怎么走？ |
| 时序图 | bootstrap | 一次会话启动涉及哪些步骤？ |
| ER 图 | - | 无传统数据库，已跳过 |
