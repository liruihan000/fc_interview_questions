# 问题4：如何监控AI系统质量？如何设计指标？

> 如何监控一个AI系统的质量？如何设计指标？

四层指标体系，每层解决不同问题。评估方式基于指标引导：能用代码算的用代码，模糊判断用 LLM-as-Judge，主观感受靠用户反馈，人工只做校准。

---

## 第一层：基础性能 — 系统是否正常运行

| 指标 | 目标值参考 |
|------|----------|
| Latency P95 | < 3s |
| Error Rate | < 0.1% |
| Token Usage / Cost per Query | 根据ROI设上限 |

**计算方式**：从日志直接算，确定性检查，零LLM成本。

**监控方式**：实时 Dashboard + 阈值告警。

## 第二层：AI质量 — 模型输出是否靠谱

| 指标 | 说明 |
|------|------|
| Faithfulness | 输出是否基于给定上下文，而非编造 |
| Hallucination Rate | 幻觉比例 |
| Relevance | 回答是否切题 |
| Safety | 是否包含有害/PII内容 |

最容易被忽略的是 Faithfulness——模型输出"看起来流畅"但内容是编造的，肉眼难以察觉。

**计算方式**：LLM-as-Judge 自动打分（用1-4整数scale，强制CoT先输出推理再给分），Safety 用 Guard model 实时检测。已知偏差通过A/B两种顺序各评一次 + 多LLM评审团消除。评分标准和阈值需要人工配置，基于 Golden Dataset 校准（见落地方案）。

**监控方式**：Langfuse 配置 LLM-as-Judge 在线打分 + 人工定期抽样校准 Judge 本身的准确性。

## 第三层：业务 — 对用户是否有帮助

| 指标 | 说明 |
|------|------|
| Task Completion Rate | 用户任务完成率 |
| Human Escalation Rate | 转人工比例 |
| Time Saved | 相比人工节省的时间 |
| CSAT | 用户满意度 |

**计算方式**：完成率和转人工从埋点算，CSAT 从用户反馈收集。

**监控方式**：产品埋点 + 周/月报。

## 第四层：Agent行为 — 推理过程是否正常

| 指标 | 告警信号 |
|------|---------|
| Tool Call Success Rate | < 95% 说明工具接口或参数有问题 |
| Avg ReAct Steps | 突然增加说明Agent在"打转" |
| Loop Detection Rate | > 1% 需立即排查 |
| pass@k / pass^k | pass^k 下降是模型退化或prompt漂移的早期信号 |
| Token Waste Ratio | 持续优化 |

前三层看结果，第四层看过程。Agent可能给出正确答案但中间绕了10步——成本和延迟上是隐患。pass@k 衡量能力上限（k次至少对1次），pass^k 衡量生产可靠性（k次全部对），90%准确率的Agent pass^8只有43%。

**计算方式**：工具成功率、步数、循环检测从 Trace 确定性计算，pass^k 定期批量回归。

**监控方式**：Langfuse Agent Trace + 定期回归测试。

---

## 落地方案

在 Livins AI 的折扣提取系统中，实际用的是确定性检查优先——info_hash 对比 + 字段格式校验覆盖大部分 case，只有内容存疑的才需人工抽查。

工具推荐 Langfuse（开源可自托管，生产 Trace + 实时告警）+ DeepEval（CI/CD 评估门禁，与 pytest 集成）。Prompt/Agent 代码变更自动跑回归，不通过阻止合并。

Golden Dataset（20-50个真实case人工标注）跨层共用：校准第二层 LLM-as-Judge 的评分准确性，同时作为第四层 pass^k 回归测试的固定输入。人工每周抽样校准，失败案例持续补充进 Golden Dataset。

关键原则：**评估 Outcome 而非 Trajectory**——检查Agent产出了什么，而不是怎么走到那里的。
