# 问题1：设计一个基于大模型的"智能报表生成助手"

> **原题**：请设计一个基于大模型（如 OpenAI API）的"智能报表生成助手"。要求：用户输入自然语言需求（例如：生成本月客户增长分析），系统自动调用数据库查询数据，通过 LLM 生成结构化报表总结，返回可下载内容（Markdown / PDF）。

## 系统架构设计（ReAct Agent + Skills）

**核心思路**：一个 ReAct Agent + 4个Skills。Agent只负责推理（理解意图、决定查什么、分析数据、决定用什么图表和报告结构），安全/校验/图表渲染/PDF构建全部下沉到Skill层。Schema不塞Prompt，让Agent通过树状目录自己探索。Skills内置了各类图表的生成方式和PDF的设计规范，Agent自行决定如何组合。

```
ReAct Agent（推理-行动循环）
├── Skill: explore_schema()                 树状目录逐层探索数据库结构
├── Skill: query_database(sql)              通过API查询数据（API层负责安全）
├── Skill: create_chart(type, data, style)  生成图表（bar/line/pie/heatmap/scatter/...）
└── Skill: build_report(sections, format)   组装Markdown/PDF（封装布局、封面、目录、样式规范）
```

### 设计哲学：关注点分离

```
Agent层：只做推理（理解意图、决定查什么、分析数据、撰写报告）
Skill/API层：封装所有确定性逻辑（安全校验、权限控制、格式化）
Agent不需要知道也不应该知道安全细节——那是API的事。
```

| 层 | 职责 | 需要LLM？ |
|----|------|-----------|
| Agent | 理解用户意图、推理查询策略、分析数据、生成报告 | YES |
| Skill: explore_schema | 返回树状目录，Agent逐层探索 | NO |
| Skill: query_database | API层做AST校验+只读连接+超时+行数限制 | NO |
| Skill: generate_report | Agent生成内容，Skill做图表渲染 | 部分 |
| Skill: export_file | 纯代码：Jinja2模板+WeasyPrint导出 | NO |

### Schema探索策略：按数据库规模选择

> **💡 关键区分**：渐进式树状探索只在大型数据库（表多、schema复杂）时才有必要。如果数据库规模较小（表<20张），完整schema可以一次性通过Skill输入给Agent，无需逐层探索。

| 数据库规模 | Schema获取方式 | 理由 |
|-----------|--------------|------|
| 小型（<20张表） | **一次性注入**：Skill直接返回完整schema，Agent一次看全 | 上下文窗口装得下，无需多轮交互，更快更简单 |
| 大型（数十~数百张表） | **渐进式树状探索**：Agent逐层drill-down | 完整schema塞不进上下文/会稀释注意力，需要Agent自主推理哪些表相关 |

#### 大型数据库：为什么用树状目录而非Embedding？

Embedding方式会漏掉"语义距离远但业务相关"的表。比如用户问"客户增长分析"，Embedding可能检索到`customers`表但漏掉`orders`表，因为"订单"和"增长"语义距离远——但业务上需要交叉分析。

**树状目录方式**：让Agent自己探索、自己推理哪些表有关（仅大型DB需要）。

```
explore_schema()
→ 返回: [business_db, analytics_db, logs_db]

explore_schema("business_db")
→ 返回: [customers, orders, payments, products, ...]

explore_schema("business_db.customers")
→ 返回: {columns: [id, name, level, created_at, churned_at, ...],
         row_count: 50000, description: "客户主表"}
```

Agent看到完整目录后自己判断："增长分析可能需要orders做交叉验证"——这是Embedding做不到的推理。

### ReAct 推理过程示例：
```
用户: "生成本月VIP客户增长分析，重点关注流失率"

=== 阶段1：探索数据结构 ===

Thought: 先了解数据库有哪些相关的表
Action: explore_schema("business_db")
Observation: [customers, orders, payments, subscriptions, ...]

Thought: customers和subscriptions可能都相关，看看结构
Action: explore_schema("business_db.customers")
Observation: {columns: [id, name, level, created_at, churned_at, ...]}

Action: explore_schema("business_db.subscriptions")
Observation: {columns: [id, customer_id, plan, started_at, cancelled_at, ...]}

=== 阶段2：查询数据 ===

Thought: 先查本月VIP客户总量和新增
Action: query_database("SELECT COUNT(*) ... WHERE level='VIP' AND created_at >= '2026-03-01'")
Observation: 新增128人，总量3,542人

Thought: 用户强调"重点关注流失率"，查流失数据并与上月对比
Action: query_database("SELECT ... churned_at ... GROUP BY week")
Observation: 本月流失87人，上月流失52人，环比增长67%

Thought: subscriptions数据可以补充分析——流失用户的订阅类型分布
Action: query_database("SELECT plan, COUNT(*) ... JOIN subscriptions ...")
Observation: 流失主要集中在月付用户（72%）

=== 阶段3：生成图表（Agent自行决定图表类型和组合） ===

Thought: 月度增长趋势用折线图，本月vs上月对比用柱状图，流失分布用饼图
Action: create_chart("line", monthly_trend_data, {title: "VIP客户月度增长趋势", x: "月份", y: "人数"})
Observation: chart_1.png 已生成

Action: create_chart("bar", comparison_data, {title: "本月vs上月VIP数据对比", groups: ["新增","流失","净增长"]})
Observation: chart_2.png 已生成

Action: create_chart("pie", churn_by_plan, {title: "流失用户订阅类型分布"})
Observation: chart_3.png 已生成

=== 阶段4：组装报告（Agent自行决定报告结构和内容） ===

Thought: 数据和图表齐了，按"总结→趋势→重点分析→建议"的结构组装PDF
Action: build_report({
  format: "pdf",
  title: "2026年3月VIP客户增长分析报告",
  sections: [
    {type: "summary", content: "本月VIP新增128人，总量3,542人。流失87人，环比增长67%，需重点关注。"},
    {type: "chart", ref: "chart_1.png", caption: "近6个月VIP客户增长趋势"},
    {type: "chart", ref: "chart_2.png", caption: "本月与上月关键指标对比"},
    {type: "analysis", content: "流失率环比增长67%，其中72%为月付用户。年付用户流失率仅8%，说明..."},
    {type: "chart", ref: "chart_3.png", caption: "流失用户订阅类型分布"},
    {type: "recommendation", content: "1. 针对月付VIP推出年付优惠转化方案\n2. 对流失预警用户触发挽留策略\n3. ..."}
  ]
})
Observation: report_2026_03_vip.pdf 已生成，下载链接: /reports/xxx.pdf
```

### Skill/API层安全设计（Agent不感知）：

```python
# query_database Skill 的API层实现
# Agent只调用 query_database(sql)，以下逻辑对Agent透明

def query_database_api(sql: str) -> dict:
    # 1. AST校验（sqlglot解析，不是正则）
    parsed = sqlglot.parse_one(sql)
    if not isinstance(parsed, sqlglot.exp.Select):
        return {"error": "只允许SELECT语句"}

    # 2. 表白名单校验
    tables = extract_tables(parsed)
    if not all(t in ALLOWED_TABLES for t in tables):
        return {"error": f"无权访问: {tables}"}

    # 3. 只读数据库连接（物理隔离，不是应用层限制）
    with readonly_db.connect() as conn:
        # 4. 超时30秒 + 最大10000行
        result = conn.execute(sql, timeout=30, max_rows=10000)
        return {"data": result.to_dict(), "row_count": len(result)}
```

### 关键设计要点：

1. **树状Schema探索**：Agent通过目录逐层发现表结构，比Embedding更准（能推理出间接相关的表）
2. **安全下沉到API层**：AST校验、只读连接、超时限制全在Skill/API层，Agent不感知
3. **缓存层**：相似查询使用语义缓存（Semantic Cache），减少重复LLM调用
4. **流式输出**：支持SSE流式输出，用户可看到推理过程并中途修正
5. **迭代次数限制**：设置max_iterations防止Agent陷入循环（建议上限10轮）

### 技术栈参考（2026最新）：
- **Agent框架**：LangGraph ReAct Agent / OpenAI Assistants API
- **模型选择**：Claude Opus / GPT-4o（需要强推理能力驱动ReAct循环）
- **可观测性**：LangSmith / Arize AI 追踪每轮Thought-Action-Observation
