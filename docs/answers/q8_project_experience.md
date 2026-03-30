# 问题8：请讲一个你主导的AI项目

> 包括项目背景、决策过程、技术选型、数据来源、指标变化、遇到最大问题、如何解决等。

---

## 项目背景

我主导的项目是 **Livins AI**，一个面向纽约租客的AI驱动租房平台。纽约租房市场有两个核心痛点：第一，折扣信息极度不透明——free rent、OP（owner pays broker fee）、security deposit减免等优惠散落在数千条listing的非结构化描述中，租客几乎无法高效比较；第二，传统leasing agent人工回复咨询，响应慢且不可规模化。

我负责两个核心AI模块：**折扣智能提取系统**和**租房服务Agent**。

---

## 模块一：折扣智能提取

### 决策过程与技术选型

首先考虑过传统NLP方案（正则+规则引擎），但折扣表述方式高度多样——"2 months free"、"$4000 rent credit"、"OP"、"reduced deposit to $1000 normally $3500"——变体几乎无穷，规则引擎的维护成本会随之指数增长。最终选择了LLM方案，天然具备语义理解能力。

模型选的是 `xiaomi/mimo-v2-flash`（通过OpenRouter调用），原因是这个任务本质上是结构化信息提取，不需要强推理能力，小模型+精心设计的prompt完全够用，且成本极低。

### 核心设计

- **Prompt工程**：设计了4步Chain-of-Thought——关键词扫描 → 上下文分类 → 数值计算 → 交叉验证，配合6个few-shot examples覆盖边界case。最关键的一条规则是"Don't guess"：金额必须在文本中明确出现才提取，杜绝幻觉。
- **并行处理**：多线程批量处理数千条listing。
- **幂等机制**：对 description + model 做 hash，内容未变化的 listing 直接跳过，避免重复 API 调用，大幅节省成本。
- **调度**：Airflow DAG，由上游 fetcher DAG 触发（非定时），保证数据新鲜度。

### 数据来源

PostgreSQL中已抓取的listing数据（description + price），来源为 Livins AI 开发的房源数据获取渠道。提取后生成12+维度的结构化折扣数据（free rent months、OP金额、security deposit savings等），并计算排序分：`ranking_score = max_discount + OP_amount × 2.0`，OP加权2倍是因为对租客的隐性价值更高。

### 迭代过程

- v1：DeepInfra API，粗粒度字段
- v2：迁移OpenRouter，增加security_deposit_savings等细粒度字段
- v3（当前）：增加few-shot examples提升边界case准确率，timeout保护，worker从10提升到50，增加retry机制

---

## 模块二：租房服务Agent

这是经历了**完整技术迭代**的部分。

### V1：自建Workflow Agent（2024年）

最初设计了一个自研DAG workflow引擎。架构是：Triage节点做意图分类 → 路由到6个专用节点（搜索、QA、Onboarding、Tour、地图、对比），每个节点独立配置model和prompt，通过JSON邻接表定义路由关系。

**遇到的最大问题**——V1暴露了四个严重缺陷：

1. **慢**：每轮对话需要3-4次LLM串行调用（triage → API generator → search & rank → response generator）
2. **准确性差**：一旦Triage分类错误，整条链路就错了，且没有自我纠正能力
3. **架构过重**：8个节点、8个独立prompt文件、自定义DAG引擎、JSON配置文件，维护成本极高
4. **扩展困难**：新增一个能力需要加节点+配路由+写prompt，改动面大

### 如何解决：V2 ReAct Agent

本质问题是我在"用workflow模拟agent"。租房对话需要灵活决策（用户可能随时切换话题、追问、比较），这不是一个确定性流程。Anthropic和LangChain的研究也验证了这一点：**当任务需要灵活决策时，单ReAct Agent + Tools远优于Workflow**。

于是完全重写，采用 **LangGraph ReAct Agent**：

- **模型**：`claude-haiku-4-5`（高性价比），可按需切换Sonnet
- **14个工具**：原子操作 + 组合操作，覆盖搜索、详情、预约、FAQ、lead capture 等
- **6个Skill（Markdown文件）**：自动加载为 system prompt，指导 agent 执行复杂流程。agent 知道"怎么做"，但保留了"做不做"的自主决策权
- **双LLM架构**：Agent LLM 负责推理决策 + Formatter LLM 将回复结构化为 UI 组件
- **Protocol-based客户端抽象**：MockClient / HttpClient 可切换，开发测试不依赖真实后端

### 指标变化

| 维度 | V1 Workflow | V2 ReAct Agent |
|------|------------|----------------|
| 响应速度 | 3-4次串行，体感慢 | 1-2次调用，响应显著加快 |
| 错误恢复 | Triage误分类则整链失败 | Agent可observe→think→retry |
| 新增能力 | 加节点+路由+prompt | 新增一个tool函数 |
| 代码维护 | 8节点+DAG引擎+JSON配置 | 1个graph + tools + markdown skills |

架构支持multi-agent扩展，当前以单agent为主，按需可拆分为多agent协作。

---

## 衍生部分：Crawler（未完成）

原计划自建crawler拓展更多数据源，目前已有初始方案和benchmark，但**爬取成本与数据准确性的平衡问题**尚未完全解决——提高爬取频率和覆盖面会显著推高成本，而降低频率又会影响数据时效性和准确性。这个trade-off仍在迭代优化中。

---

## 核心收获

1. **Start simple, stay simple**：V1最大教训是过度工程化。Workflow适合确定性流程，但需要灵活决策的场景，ReAct Agent才是正确抽象层。
2. **Prompt工程 > 模型大小**：折扣提取用小模型+CoT prompt+few-shot，效果不输大模型，成本低一个数量级。
3. **幂等设计是生产必备**：info_hash机制在数千条listing规模下节省了大量重复API调用成本。
4. **架构决策要数据驱动**：从Workflow到ReAct的转型基于响应速度、准确率、维护成本的具体度量，不是跟风。
5. **成本意识贯穿全程**：模型选型（mimo-flash / haiku）、幂等跳过、crawler搁置，每一步都在算ROI。
