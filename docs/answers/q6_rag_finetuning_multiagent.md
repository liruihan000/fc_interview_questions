# 问题6：RAG vs Fine-tuning & Multi-Agent

> **原题**：RAG vs Fine-tuning，你如何选择？多Agent架构是否真的有必要？

---

## 一、RAG vs Fine-tuning

### 个人判断标准

知识不够用RAG，只有当rag和harness engineer以及上下文工程都达不到要求时才会考虑Fine-tuning。

```
Prompt Engineering + Few-shot 能解决吗？
│
├── 能 → 到此为止
│
└── 不能 → 问题出在哪？
            │
            ├── 知识不够（不知道最新信息/内部数据）→ RAG
            ├── 行为不对（格式不符、分类不准）→ Fine-tuning
            └── 两者都有 / 生产级要求 → 混合方案（FT底座 + RAG动态知识）
```

倾向先从RAG开始：不需要训练数据、可实时更新、可追溯到具体文档段落。Fine-tuning门槛更高——需要标注数据、训练周期长、知识更新要重新训练。

---

## 二、单Agent vs 多Agent

80%场景不需要多Agent。单Agent + Tools/Skills 是最优解。

多Agent的必要条件很苛刻，需同时满足：大量可并行子任务 + 超出单上下文窗口 + 工具>20 + 任务价值覆盖15倍成本。全满足的场景很少。

Livins AI的租房Agent用单Agent + 14个Tool + 6个Skill，够用。FC的SEC系统是"两个独立单Agent + Airflow调度"，不是多Agent架构——调度交给Airflow（确定性），每个Agent独立运行，通过任务队列传递数据。

**如果犹豫 → 用单Agent。** 多Agent是"单Agent明确不够用"之后的选择。

---

## 三、总结

| 问题 | 答案 |
|------|------|
| RAG还是Fine-tuning？ | 先试Prompt Engineering；知识问题用RAG；行为问题用FT；生产级用混合方案 |
| 多Agent有必要吗？ | 80%场景不需要，默认单Agent + Skills |
| FC项目该怎么选？ | SEC筛选→FT MiMo-V2-Flash，SEC抽取→RAG+MiniMax M2.5，报表→单ReAct Agent，知识库→单Agent+RAG |
