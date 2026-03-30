# 问题4：如何监控AI系统质量？如何设计指标？

> 如何监控一个AI系统的质量？如何设计指标？

方案基于我的个人经验和行业调研，分三部分：**指标设计**，**评估方法** 和 **监控系统**。

---

## 一、指标设计：四层递进

### 第一层：基础性能指标

所有软件系统都需要的基础监控。

| 指标 | 说明 | 目标值参考 |
|------|------|----------|
| Latency (P50/P95/P99) | 端到端响应时间 | P95 < 3s |
| Error Rate | 系统错误率 | < 0.1% |
| Token Usage | 每次请求消耗token数 | 持续优化 |
| Cost per Query | 单次查询成本 | 根据业务ROI设定上限 |

快速发现"系统挂了"或"成本炸了"，告警的第一道防线。目标值根据SLA和业务ROI倒推。

### 第二层：AI质量指标

AI系统特有，传统软件监控覆盖不到。

| 指标 | 说明 | 评估方法 |
|------|------|---------|
| **Faithfulness** | 输出是否基于给定上下文，而非编造 | LLM-as-Judge + 人工抽查校准 |
| **Hallucination Rate** | 幻觉比例 | 与Ground Truth对比 |
| **Relevance** | 回答是否切题 | 语义相似度 + LLM评分 |
| **Safety** | 是否包含有害/PII内容 | Guard model实时检测 |

最容易被忽略的是 Faithfulness。模型输出"看起来流畅"但内容是编造的，肉眼难以察觉，必须自动化检测。

### 第三层：业务指标

技术指标再好，业务没改善也没意义。

| 指标 | 说明 |
|------|------|
| Task Completion Rate | 用户任务完成率 |
| Human Escalation Rate | 转人工比例（越低越好） |
| Time Saved | 相比人工节省的时间 |
| User Satisfaction (CSAT) | 用户满意度评分 |

### 第四层：Agent行为指标

Agent系统有多步推理、工具调用、自主决策，需要专门监控。容易被遗漏但很重要。

| 指标 | 说明 | 告警信号 |
|------|------|---------|
| **Tool Call Success Rate** | 工具调用成功率 | < 95% 说明工具接口或参数有问题 |
| **Avg ReAct Steps** | 平均推理-行动循环次数 | 突然增加说明Agent在"打转" |
| **Loop Detection Rate** | 陷入死循环的比例 | > 1% 需立即排查 |
| **pass@k / pass^k** | pass@k衡量能力上限（k次至少对1次），pass^k衡量生产可靠性（k次全部对）。90%准确率的Agent pass^8只有43% | pass^k下降是模型退化或prompt漂移的早期信号 |
| **Token Waste Ratio** | 无效推理步骤消耗的token占比 | 持续优化 |

前三层看结果，第四层看过程。Agent可能最终给出正确答案，但中间绕了10步、调了5次无效工具——成本和延迟上是隐患。

在 Livins AI 的折扣提取系统中，我实际用的是确定性检查优先——info_hash 对比 + 字段格式校验（金额是否为正数、free rent months 是否合理）覆盖大部分 case，只有格式校验通过但内容存疑的才需人工抽查。

---

## 二、评估方法：怎么算这些指标

全靠人工审核不现实，全靠自动化有盲区。三层检查叠加，能用代码算的用代码，模糊判断用LLM，人工只做校准。

**确定性自动检查（Code-Based Grader）** — 覆盖第一层（延迟、错误率直接从日志算）、第三层（任务完成率从埋点算）、第四层大部分指标（工具成功率、ReAct步数、循环检测从Trace算）。最可靠、最便宜、零LLM成本。关键原则：**评估Outcome而非Trajectory**——检查Agent产出了什么，而不是怎么走到那里的。

**LLM-as-Judge（Model-Based Grader）** — 覆盖第二层（Faithfulness、Relevance等模糊判断）。用强模型评估目标模型输出质量，与人类评估达80%一致率。提升准确率的关键：用小整数scale（1-4），强制CoT先输出推理再给分，提供具体等级描述。已知偏差：位置偏差（A/B两种顺序各评一次取一致结果）、单一Judge盲区（多个LLM组成评审团）。工具实现用 DeepEval 的 GEval 指标，与 pytest 集成。

**人工介入（校准而非逐条检查）** — 覆盖第三层的主观指标（CSAT）+ 校准前两层。构建 Golden Dataset（50-100个真实case），定期抽样校准 LLM Judge，低置信度case路由给人工审核，失败案例分析反哺规则改进。

三层叠加：每层都有盲区，但问题同时穿过所有层的概率极低。

---

## 三、监控系统：在哪跑这些评估

**线上实时**：Production Traces 接入 Langfuse（开源可自托管），第一层和第四层指标实时 Dashboard + 阈值告警。第二层 Faithfulness 通过 Langfuse 内置的 LLM-as-Judge 在线打分。

**CI/CD 门禁**：每次 Prompt/Agent 代码变更自动跑 DeepEval 回归（与 pytest 集成），不通过则阻止合并。RAG 场景额外加 RAGAS 检测 Faithfulness/Relevancy。

**离线批量**：定期对第三层业务指标和第四层 pass^k 做批量评估，生成周/月质量报告。第三层的 CSAT 通过用户反馈持续收集。

---

## 四、起步路线

构建评估体系不需要一步到位：

1. 从20-50个真实失败案例开始，真实failure case价值远高于人造数据
2. 把手动QA检查转化为自动化test case
3. 设计无歧义的Task——两个专家独立评估应得出相同pass/fail
4. Grader优先用确定性规则，能用代码判断的不要用LLM
5. 评估Outcome而非Trajectory
6. 监控eval饱和度——通过率长期100%时应补充新edge case
