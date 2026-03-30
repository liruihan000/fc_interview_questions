# 问题1：设计一个基于大模型的"智能报表生成助手"

> **原题**：请设计一个基于大模型（如 OpenAI API）的"智能报表生成助手"。要求：用户输入自然语言需求（例如：生成本月客户增长分析），系统自动调用数据库查询数据，通过 LLM 生成结构化报表总结，返回可下载内容（Markdown / PDF）。

> **Demo 代码和完整架构文档**：[liruihan000/livinsai_chart_report_demo_for_FC](https://github.com/liruihan000/livinsai_chart_report_demo_for_FC)
>
> 以 Livins AI 房源数据库为数据源，做了一个可运行的 Demo。以下是设计思路和关键决策的总结，详细文档见 repo 的 `docs/` 目录。

## 设计哲学

```
Agent层：只做推理（理解意图、写SQL、分析数据、设计报告结构）
Skill/API层：封装所有确定性逻辑（安全校验、权限控制、格式化）
Agent不需要知道也不应该知道安全细节——那是API的事。
尽可能的简化架构而不是复杂化
```

## 系统架构

一个 ReAct Agent + 2 个 Tool + Code Execution 沙盒。

```
┌───────────────────────────────────────────────────┐
│  Browser (localStorage: messages + session_id)    │
└──────────────────────┬────────────────────────────┘
                       │ POST /chat/stream (SSE)
┌──────────────────────▼────────────────────────────┐
│  FastAPI Backend (Stateless)                      │
│  ┌──────────────────────────────────────────────┐ │
│  │  ReAct Agent (LangGraph)                     │ │
│  │  Thought → Action → Observation → ...        │ │
│  │      │          │          │                  │ │
│  │      ▼          ▼          ▼                  │ │
│  │  load_skill  query_db   Code Execution       │ │
│  │  (按需加载   (Text-to   (Anthropic沙盒:      │ │
│  │   Schema/    SQL,API    matplotlib图表+       │ │
│  │   规范)      层校验)    reportlab PDF)        │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────┬────────────────────────────┘
                       │
┌──────────────────────▼────────────────────────────┐
│  PostgreSQL + PostGIS (via Data Service API)      │
│  buildings, listings, ml_listings, isochrones     │
└───────────────────────────────────────────────────┘
```

### 为什么用单 Agent 不用 Multi-Agent？

- Pipeline 多 Agent 串行会丢失上下文（用户说"重点关注流失率"在传递中丢失）
- 报表生成是顺序+工具密集型任务，Google Research 数据：工具密集型多 Agent 慢 2-6 倍
- 所有步骤共享同一个上下文（Agent 需要知道之前查了什么才能决定下一步）

## 关键设计决策

### 1. Text-to-SQL：Agent 写 SQL，API 层校验

业界标准范式（Databricks AI/BI、Snowflake Cortex Analyst、AWS QuickSight Q 都用）。

- **灵活**：任意聚合/JOIN/子查询，不需要为每种分析需求预定义 endpoint
- **安全可控**：校验逻辑下沉到 API 层，Agent 不感知

```python
# data_service 侧 — POST /query/execute
def validate_sql(sql: str) -> None:
    tree = sqlglot.parse_one(sql)
    if not isinstance(tree, exp.Select):
        raise ValueError("Only SELECT queries allowed")
    # 禁止写操作、白名单表、只读连接 + 超时 + 行数限制
```


> **规模扩展**：如果表增长到几十~几百张，Schema 一次加载会稀释注意力。此时改为渐进式树状探索（`explore_schema` Tool 逐层 drill-down），让 Agent 自己推理哪些表相关——比 Embedding 检索更准，因为 Embedding 会漏掉"语义距离远但业务相关"的表。

### 2. 图表+PDF：Code Execution 沙盒

Agent 在 Anthropic 托管沙盒中执行 matplotlib 绘图 + reportlab 组装 PDF，通过 Files API 取回文件。图表和 PDF 合并为一次 Code Execution 调用，Agent 自行决定图表类型和报告结构。

| 组件 | Demo（当前） | 生产升级方案（10x+ 提速） |
|------|-------------|--------------------------|
| 图表 | LLM 沙盒 + matplotlib（5-15s） | [antvis/mcp-server-chart](https://github.com/antvis/mcp-server-chart) MCP Server，26+ 图表类型，~1-2s |
| PDF | LLM 沙盒 + reportlab（同上） | Jinja2 → HTML → Playwright，warm 模式 13ms |

### 3. SSE 流式推送工具调用步骤

Agent 执行报表生成通常较久，纯等待体验差。`POST /chat/stream` 通过 LangGraph `astream_events(v2)` 实时推送事件：

| SSE event | 说明 |
|-----------|------|
| `thinking` | Agent 工具调用前的推理文本 |
| `tool_start` / `tool_end` | 工具调用开始/完成（含 SQL、代码片段） |
| `token` | 最终回复文本 |
| `done` | 完成，附带生成的文件列表 |

前端实时显示推理过程和工具步骤，完成后折叠保留在历史消息中可展开查看。

### 4. 对话历史：浏览器管状态，服务端无状态

demo的对话历史存浏览器 localStorage，每轮请求携带完整 messages 数组，服务端不持久化。后续需要跨设备同步时加 LangGraph Checkpointer（Redis / PostgreSQL）。

## ReAct 推理过程示例

```
用户: "分析曼哈顿一居室过去3个月的租金趋势，对比布鲁克林"

Thought: 需要查数据库，先加载 Schema
Action: load_skill("data_query")
→ (返回完整 Schema：4张表字段+关系+SQL规范)

Thought: 按月聚合曼哈顿和布鲁克林一居室的均价
Action: query_database("SELECT b.borough, DATE_TRUNC('month', l.listed_at) AS month,
        AVG(l.price) FROM listings l JOIN buildings b ON l.building_id = b.id
        WHERE b.borough IN ('Manhattan','Brooklyn') AND l.bedrooms = 1
        AND l.listed_at >= CURRENT_DATE - INTERVAL '3 months'
        GROUP BY b.borough, month ORDER BY month")
→ [{borough: "Manhattan", month: "2026-01", avg_price: 3450}, ...]

Thought: 加载图表和报告规范，然后在沙盒中一次性生成图表+PDF
Action: load_skill("chart_generation"), load_skill("report_building")
Action: code_execution("""
  import matplotlib.pyplot as plt
  from reportlab.platypus import SimpleDocTemplate, Image, Paragraph
  # 趋势折线图 + 区域对比柱状图 + 组装 PDF
  ...
""")
→ 文件生成完毕，通过 Files API 取回 → 前端展示下载按钮 + PDF 预览
```

## 技术栈

| 层 | 选择 |
|---|------|
| 前端 | Next.js 15 (静态导出) + React 19 + Tailwind CSS v4 + @react-pdf-viewer/core |
| 后端 | FastAPI + LangGraph ReAct Agent + LangChain |
| LLM | Anthropic / OpenAI（`init_chat_model("provider:model")` 可切换） |
| 图表/PDF | LLM Code Execution 沙盒（Demo）→ antvis MCP + Playwright（Prod） |
| 数据 | PostgreSQL + PostGIS，通过 Data Service API 的 `/query/execute` 端点 |
| 可观测性 | LangSmith / Arize AI 追踪 Thought-Action-Observation （还未实现）|

## API 设计

| 端点 | 方法 | 说明 |
|------|------|------|
| `/chat` | POST | 同步对话（发送完整消息历史，返回回复+文件引用） |
| `/chat/stream` | POST | SSE 流式对话（实时推送工具步骤+文本） |
| `/reports/{file_id}` | GET | 从 Anthropic Files API 流式转发文件下载 |
| `/health` | GET | 健康检查 |

## 项目结构

```
├── src/livins_report_agent/    # 后端
│   ├── agent/                  # ReAct Agent（LangGraph）
│   ├── api/                    # FastAPI 端点
│   ├── tools/                  # query_database, load_skill
│   ├── skills/                 # SKILL.md（Schema、图表规范、报告规范）
│   └── apartment_client/       # 数据服务客户端（Protocol-based，Mock/Http 可切换）
├── frontend/                   # Next.js 前端
│   └── src/
│       ├── components/         # Chat + PDF 预览 split-panel
│       ├── hooks/              # useChat（SSE流式）, usePdf, useLocalStorage
│       └── lib/                # API 调用、类型定义
└── docs/                       # 详细架构文档
    └── architecture/           # overview, agent, api, frontend, decisions
```

> 更多设计决策详见 repo 的 [`docs/architecture/decisions.md`](https://github.com/liruihan000/livinsai_chart_report_demo_for_FC/blob/main/docs/architecture/decisions.md)。
