# Agent Architecture Decision Guide

> 基于 Anthropic、Google Research、LangChain 2026年最新研究与实战总结

---

## 核心原则

**尽可能用最简单的方案，只在必要时增加复杂度。**
— Anthropic, "Building Effective Agents"

大多数生产系统最终都**简化**了Agent数量。行业经验表明"From 12 Agents to 1"是常态，不是例外。

---

## 复杂度光谱

```
尽量待在左边，只在必要时向右移动 →

Prompt → Workflow → 单Agent(ReAct) → 多Agent(Supervisor/Swarm) → 层级系统
 最简                                                            最复杂
```

**每向右一步，你需要回答："为什么左边的方案不够？"**

---

## 决策树

```
你的任务需要LLM自主决策吗？
│
├── NO → 流程固定，步骤明确
│        → 用 Workflow（代码/Airflow写死流程）
│        → 不需要Agent
│
└── YES → 需要推理、工具调用、动态路径
         │
         ├── 工具 < 20个，上下文够用？
         │   → 单Agent (ReAct + Tools/Skills)
         │   → 这是80%场景的最优解
         │
         └── 单Agent不够？为什么？
             │
             ├── 子任务可大量并行
             │   → 多Agent (Swarm / Scatter-Gather)
             │   → 例：多源数据采集、多视角分析
             │
             ├── 需要路由到不同专业领域
             │   → 多Agent (Supervisor)
             │   → 例：客服系统路由到退款/技术/投诉
             │
             ├── 信息量超出单个上下文窗口
             │   → 多Agent（拆分上下文）
             │   → 例：分析100页文档
             │
             ├── 需要对接大量复杂工具（>20）
             │   → 多Agent（按工具域拆分）
             │   → 例：全栈开发Agent（前端/后端/数据库）
             │
             └── 大型多层级项目
                 → Hierarchical（层级系统）
                 → 例：自动化软件开发、大型研究报告
```

---

## 各架构详解

### 1. Workflow（工作流）

```
Step A → Step B → Step C → 输出
（代码写死，无LLM决策）
```

| 项目 | 说明 |
|------|------|
| **适用** | 流程固定、步骤明确、无需判断 |
| **优势** | 确定性100%、成本最低、最好调试 |
| **劣势** | 无灵活性，路径写死 |
| **实现** | Airflow / Temporal / 普通代码 |
| **Token消耗** | 0（不用LLM）或极低 |
| **典型场景** | ETL管道、合规审批流程、定时数据同步 |

**判断标准**：如果你能用 if/else 写出所有分支，不要用Agent。

---

### 2. 单Agent (ReAct + Tools/Skills)

```
ReAct Agent（推理-行动循环）
├── Tool/Skill A
├── Tool/Skill B
└── Tool/Skill C
```

| 项目 | 说明 |
|------|------|
| **适用** | 需要推理、工具<20个、任务顺序执行或轻度分支 |
| **优势** | 上下文完整、简单、好维护 |
| **劣势** | 不能并行、受上下文窗口限制 |
| **实现** | LangGraph ReAct / OpenAI Assistants / Claude Agent |
| **Token消耗** | ~4x（相比单轮对话） |
| **典型场景** | 报表生成、代码调试、文档QA、数据分析 |

**判断标准**：这是80%场景的最优解。**默认从这里开始。**

---

### 3. 多Agent — Supervisor（监督者）

```
        ┌────────────┐
        │ Supervisor │ ← 路由、分发、汇总
        └──┬───┬───┬─┘
           │   │   │
           ▼   ▼   ▼
         Agent Agent Agent
          A     B     C
```

| 项目 | 说明 |
|------|------|
| **适用** | 任务需路由到不同专业Agent |
| **优势** | 清晰的职责分工、可审计 |
| **劣势** | Supervisor有"翻译"开销，Token消耗高于Swarm |
| **实现** | LangGraph Supervisor / AutoGen Group Chat |
| **Token消耗** | ~15x |
| **典型场景** | 客服路由、多领域问答、审批流程 |

**判断标准**：子Agent之间不需要互相通信，只需要中心节点分发。

---

### 4. 多Agent — Swarm（群体）

```
      ┌─────┐ ┌─────┐ ┌─────┐
      │  A  │ │  B  │ │  C  │  ← 自主运行，可互相handoff
      └──┬──┘ └──┬──┘ └──┬──┘
         │       │       │
         └───────┼───────┘
                 ▼
            汇总结果
```

| 项目 | 说明 |
|------|------|
| **适用** | 可并行的子任务、需要多视角 |
| **优势** | 并行提速、Agent自主handoff、比Supervisor省Token |
| **劣势** | 行为不可预测、调试困难 |
| **实现** | LangGraph Swarm / OpenAI Agents SDK handoff |
| **Token消耗** | ~10-15x（比Supervisor略低） |
| **典型场景** | 多源数据采集、多模型交叉验证、协作研究 |

**判断标准**：子任务独立、可并行、Agent之间需要灵活交接。

---

### 5. Hierarchical（层级）

```
           ┌─────────┐
           │ 顶层     │ ← 战略拆解
           └──┬───┬──┘
              │   │
         ┌────┘   └────┐
         ▼             ▼
    ┌─────────┐   ┌─────────┐
    │ 中层主管 │   │ 中层主管 │ ← 战术分配
    └──┬───┬──┘   └──┬───┬──┘
       │   │         │   │
       ▼   ▼         ▼   ▼
     Agent Agent   Agent Agent  ← 执行
```

| 项目 | 说明 |
|------|------|
| **适用** | 大型复杂项目、需要多级拆解 |
| **优势** | 分而治之、可扩展 |
| **劣势** | 复杂度最高、延迟最高、成本最高 |
| **实现** | CrewAI Hierarchical / 自定义LangGraph |
| **Token消耗** | 20x+ |
| **典型场景** | 自动化软件开发、大型研究报告、企业级工作流 |

**判断标准**：只有在其他架构明确不够用时才考虑。

---

## 关键数据

### Token 消耗对比

| 架构 | Token消耗（vs 单轮对话） |
|------|--------------------------|
| 单轮对话 | 1x |
| 单Agent (ReAct) | **4x** |
| 多Agent (Swarm) | **10-15x** |
| 多Agent (Supervisor) | **15x** |
| Hierarchical | **20x+** |

> 多Agent的任务价值必须高到能覆盖15倍的成本开销。
> — Anthropic

### 性能对比（2026研究数据）

| 发现 | 来源 |
|------|------|
| 单Agent准确率>45%后，加Agent收益递减甚至为负 | Google Research |
| 金融分析任务：多Agent比单Agent **+80%** | Mount Sinai / 学术研究 |
| 顺序任务：多Agent比单Agent **-70%** | PlanCraft实验 |
| 工具密集型任务：多Agent **慢2-6倍** | VentureBeat |
| Anthropic多Agent研究系统比单Agent Opus 4 **+90.2%** | Anthropic（研究场景，大量并行） |
| Swarm比Supervisor**省Token**（无翻译开销） | LangChain Benchmark |

---

## Anthropic 的多Agent必要条件

> 多Agent值得用，**当且仅当**同时满足：

| 条件 | 说明 |
|------|------|
| 1. 大量可并行子任务 | 不能并行 → 用单Agent |
| 2. 信息量超出单个上下文窗口 | 上下文够用 → 用单Agent |
| 3. 需要对接大量复杂工具 | 工具<20 → 用单Agent |
| 4. 任务价值足够高 | 覆盖不了15x Token成本 → 用单Agent |

> **反面指标**（不适合多Agent）：
> - 所有Agent需要共享同一个上下文
> - 子任务之间依赖很强
> - 需要严格的执行顺序

---

## 实战决策速查表

| 场景 | 推荐架构 | 理由 |
|------|----------|------|
| 报表生成 | 单Agent (ReAct) | 顺序+工具密集，多Agent反而慢 |
| SEC Filing批量筛选+精准抽取 | **Workflow + 两个独立单Agent** | 按任务性质拆（Haiku筛选+Opus抽取），Agent间不通信，通过任务队列传递数据，类似微服务架构 |
| 客服系统 | Supervisor | 天然路由场景 |
| 多源数据采集 | Swarm | 可并行，各自独立 |
| 代码Debug | 单Agent (ReAct) | 需要完整上下文推理 |
| 大型研究报告 | 多Agent（Anthropic模式） | 大量并行子查询，超出单上下文 |
| ETL/数据管道 | Workflow（不用Agent） | 流程固定，不需要LLM |
| 合规审批 | Workflow（不用Agent） | 步骤确定，不需要推理 |
| 多语言翻译+校审 | Swarm或Pipeline | 可并行翻译，顺序校审 |
| 企业知识库问答 | 单Agent (RAG + ReAct) | 上下文够用，单Agent最简 |

---

## 常见陷阱

### 1. 过度拆分Agent
> "我设计了5个Agent的Pipeline！"
> → 问自己：一个ReAct Agent + 5个Tools能不能做？大概率能。

### 2. 用LLM做调度
> "我的Supervisor Agent负责分配任务和重试。"
> → 调度是确定性工作，交给Airflow/Celery，不要浪费LLM Token。

### 3. 先选框架再选架构
> "我们用CrewAI所以要多Agent。"
> → 先确定任务需要什么架构，再选框架。框架不应该决定架构。

### 4. 忽略成本
> "多Agent效果更好！"
> → 好15%但贵15倍，ROI是负的。先算账。

### 5. 忽略调试成本
> "多Agent上线了但出了bug找不到原因。"
> → 多Agent的调试复杂度是指数级增长的。生产环境必须有链路追踪（LangSmith/Arize）。

---

## 总结：一句话决策

**如果你犹豫用单Agent还是多Agent → 用单Agent。**

多Agent应该是"单Agent明确不够用"之后的选择，不是默认选择。

