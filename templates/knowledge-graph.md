# 知识图谱：{项目名}

> 生成时间：{时间} | 代码版本：`{git commit hash}` | 分析工具：code-to-req v2
> 实体数：{N} | 关系数：{N} | 聚合层级：模块级

---

## 实体清单

| ID | 类型 | 名称 | 描述 | 子实体数 |
|----|------|------|------|---------|
| m01 | module | {模块名} | {一句话职责} | {N 个文件} |
| s01 | service | {服务名} | {一句话职责} | - |
| a01 | api | {端点组} | {N 个端点} | - |
| d01 | data-model | {表/模型名} | {核心字段} | - |
| e01 | external | {外部服务} | {用途} | - |

**实体类型说明**：
- `module`：代码目录级模块
- `service`：业务逻辑服务（被多处调用的核心单元）
- `api`：API 端点组（按资源聚合）
- `data-model`：数据库表或 ORM 模型
- `external`：外部 API / 第三方服务

---

## 关系清单

| 源 | 关系类型 | 目标 | 说明 |
|----|---------|------|------|
| {源ID} | depends_on | {目标ID} | {代码层面：import/require} |
| {源ID} | calls | {目标ID} | {运行时调用} |
| {源ID} | reads_from | {目标ID} | {读取数据} |
| {源ID} | writes_to | {目标ID} | {写入数据} |
| {源ID} | extends | {目标ID} | {继承/实现} |

**关系类型说明**：
- `depends_on`：代码级依赖（A 的文件里 import 了 B）
- `calls`：运行时调用（A 在执行过程中调用 B 的方法）
- `reads_from`：数据读取（服务从数据源读数据）
- `writes_to`：数据写入（服务向数据源写数据）
- `extends`：继承或接口实现

---

## Mermaid 可视化

> 顶层图限制 ≤ 30 个节点（模块级聚合）

```mermaid
graph TD
    subgraph 前端
        FE[前端应用]
    end

    subgraph 后端服务
        {服务节点...}
    end

    subgraph 数据层
        {数据节点...}
    end

    subgraph 外部
        {外部节点...}
    end

    %% 关系连线
    FE -->|calls| API
    API -->|depends_on| Service
    Service -->|reads_from| DB
    Service -->|calls| External
```

---

## JSON 数据（供前端渲染）

以下 JSON 可直接用于 sigma.js / D3.js / Cytoscape.js 渲染交互式图谱。

```json
{
  "nodes": [
    {
      "id": "{id}",
      "label": "{名称}",
      "type": "{module|service|api|data-model|external}",
      "description": "{一句话描述}",
      "size": {子实体数或文件数，用于节点大小},
      "group": "{所属分组，用于着色}"
    }
  ],
  "edges": [
    {
      "source": "{源ID}",
      "target": "{目标ID}",
      "type": "{depends_on|calls|reads_from|writes_to|extends}",
      "label": "{可选说明}"
    }
  ],
  "metadata": {
    "project": "{项目名}",
    "generated_at": "{时间}",
    "commit": "{hash}",
    "total_nodes": {N},
    "total_edges": {N}
  }
}
```

---

## 图谱解读

**核心节点**（连接数最多）：
1. {节点名} — {为什么重要：被 N 个模块依赖}
2. {节点名} — {为什么重要}
3. {节点名} — {为什么重要}

**孤立节点**（无连接或仅单向）：
- {节点名} — {可能的问题：死代码？独立工具？}

**潜在风险**：
- {高耦合点：某模块被过多依赖，改动影响面大}
- {循环依赖：A→B→C→A，需要解耦}
