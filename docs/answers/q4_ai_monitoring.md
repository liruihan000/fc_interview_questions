# 问题4：如何监控AI系统质量？如何设计指标？

> 如何监控一个AI系统的质量？如何设计指标？

---

## 核心观点

监控和评估是同一问题的两面——评估是上线前的质量门禁，监控是上线后的持续保障。二者共享指标体系，区别在于触发时机和响应方式。

我的整体方案分两部分：**指标设计（监控什么）** 和 **评估方法（怎么算这些指标）**。

---

## 一、指标设计：四层递进

我会从四个层次设计指标体系，从基础设施到业务价值逐层递进。每一层回答一个不同的问题，服务于不同的stakeholder。

### 第一层：基础性能指标——"系统还活着吗？"

这是所有软件系统都需要的基础监控，AI系统也不例外。

| 指标 | 说明 | 目标值参考 |
|------|------|----------|
| Latency (P50/P95/P99) | 端到端响应时间 | P95 < 3s |
| Error Rate | 系统错误率 | < 0.1% |
| Token Usage | 每次请求消耗token数 | 持续优化 |
| Cost per Query | 单次查询成本 | 根据业务ROI设定上限 |

这层的价值在于**快速发现"系统挂了"或"成本炸了"**，是告警的第一道防线。目标值不是固定的，应根据SLA和业务ROI倒推。

### 第二层：AI质量指标——"模型输出靠谱吗？"

这是AI系统特有的层，传统软件监控覆盖不到。

| 指标 | 说明 | 评估方法 |
|------|------|---------|
| **Faithfulness** | 输出是否基于给定上下文，而非模型自行编造 | LLM-as-Judge自动打分 + 人工抽查校准 |
| **Hallucination Rate** | 幻觉比例 | 与Ground Truth对比 |
| **Relevance** | 回答是否切题 | 语义相似度 + LLM评分 |
| **Safety** | 是否包含有害/PII内容 | Guard model实时检测 |

这层最容易被忽略的是Faithfulness。模型输出"看起来流畅"但内容是编造的，用户很难靠肉眼察觉，必须依赖自动化检测机制。这也是RAG系统中最常见的质量问题。

### 第三层：业务指标——"对业务有帮助吗？"

技术指标再好，业务没改善也没意义。业务指标是向上汇报的语言——技术团队看Faithfulness，但VP和客户只关心"帮我省了多少时间、减少了多少人工"。

| 指标 | 说明 |
|------|------|
| Task Completion Rate | 用户任务完成率 |
| Human Escalation Rate | 转人工比例（越低越好） |
| Time Saved | 相比人工节省的时间 |
| User Satisfaction (CSAT) | 用户满意度评分 |

### 第四层：Agent行为指标——"Agent的推理过程正常吗？"

传统AI系统是单次请求-响应，但Agent系统有多步推理、工具调用、自主决策，需要专门的监控维度。这是我认为**最容易被遗漏但非常重要**的一层。

| 指标 | 说明 | 告警信号 |
|------|------|---------|
| **Tool Call Success Rate** | 工具调用成功率 | < 95% 说明工具接口或参数构造有问题 |
| **Avg ReAct Steps** | 平均推理-行动循环次数 | 突然增加说明Agent在"打转" |
| **Loop Detection Rate** | Agent陷入死循环的比例 | > 1% 需要立即排查 |
| **pass^k Consistency** | 同一输入多次执行结果的一致性 | 不一致说明输出不可靠 |
| **Token Waste Ratio** | 无效推理步骤消耗的token占比 | 持续优化 |

前三层告诉你"结果好不好"，但第四层告诉你"过程有没有问题"。一个Agent可能最终给出了正确答案，但中间绕了10步、调了5次无效工具——这在成本和延迟上是隐患，也是下一次出错的前兆。

### 四层总览

| 层 | 回答什么问题 | 主要受众 |
|----|-------------|---------|
| 第一层：性能 | 系统还活着吗？成本可控吗？ | SRE / 运维 |
| 第二层：AI质量 | 模型输出靠谱吗？ | AI工程师 |
| 第三层：业务 | 对用户/业务有帮助吗？ | PM / 业务方 |
| 第四层：Agent行为 | Agent推理过程正常吗？ | AI工程师 / 架构师 |

补充一个行业参考框架——**CLASSIC五维度**（2025年企业级Agent评估共识），从另一个角度归纳了同样的关切：**C**ost（成本）、**L**atency（延迟）、**A**ccuracy（准确性）、**S**tability（稳定性/一致性）、**S**ecurity（安全性）。与我的四层体系互补：CLASSIC按维度切分，四层体系按stakeholder切分。

---

## 二、评估方法：三层自动化检查

指标定义好了，接下来的核心问题是**怎么高效地算出这些指标**。全靠人工审核不现实（成本太高、速度太慢），全靠自动化又有盲区。我的方案是三层检查，95%自动化，人工只做校准和边界case。

### 第1层：确定性自动检查（Code-Based Grader）

最可靠、最便宜、零LLM成本。适合有明确对错标准的指标：

```python
# 字符串精确匹配
assert agent_output["name"] == golden_label["name"]

# 正则格式校验
assert re.match(r'\+\d{1,3}-\d+-\d+', agent_output["phone"])

# 数据库状态对比（τ-bench的做法）
# Agent说"已完成操作" → 不信它说的，直接检查DB中状态是否真的变了
assert db.query("SELECT status FROM orders WHERE id=?", order_id) == "cancelled"
```

这层的关键原则来自Anthropic的建议：**评估Outcome（最终状态），而非Trajectory（执行路径）**。Agent可能走不同的推理路径但都达到正确结果，所以应该检查它"产出了什么"，而不是"怎么走到那里的"。

### 第2层：LLM-as-Judge（Model-Based Grader）

处理确定性规则覆盖不了的"模糊"判断——比如回答质量、推理合理性、摘要完整度等没有唯一正确答案的指标。

核心思路：用一个强模型（如GPT-4o）评估目标模型的输出质量。研究表明与人类评估达80%一致率，成本低500-5000倍。

**三种评估结构：**

| 结构 | 做法 | 适用场景 |
|------|------|---------|
| **Pointwise（打分式）** | 对单个回答按评分标准打分 | 通用质量评估 |
| **Pairwise（对比式）** | 给Judge两个回答，判断哪个更好 | A/B测试、模型对比 |
| **Reference-guided** | 提供参考答案辅助评分 | 有Golden Answer的场景 |

**提升Judge准确率的关键实践：**

1. **用小整数scale（1-4）而非0-10浮点**——减少模型在连续空间上的随机性
2. **强制Chain-of-Thought**——要求Judge先输出推理过程再给分，而非直接输出分数
3. **提供具体等级描述**——每个分值对应什么质量水平

这三个改进可以将Judge与人类评估的Pearson相关性从0.567提升到0.843（来源：Hugging Face LLM-as-Judge研究）。

```python
# 改进后的Judge Prompt示例
JUDGE_PROMPT = """
Your task is to evaluate the quality of the AI assistant's answer.

Score scale:
1: Terrible — completely irrelevant or very partial
2: Mostly not helpful — misses key aspects
3: Mostly helpful — provides support, could be improved
4: Excellent — relevant, direct, detailed, addresses all concerns

You MUST provide values for both fields:
Evaluation: (your reasoning process)
Total rating: (1-4)

Question: {question}
Answer: {answer}

Evaluation: """
```

**已知偏差及消除方法：**
- **位置偏差**：GPT-4在Pairwise评估中有~40%的位置不一致性（把A放前面和放后面，给出不同判断）。解决方法是对(A,B)和(B,A)两种顺序各评一次，只取一致的结果。
- **单一Judge盲区**：多个不同LLM组成"评审团"，2025年研究表明可提高6.7个百分点准确率。

**工具实现**：DeepEval框架提供了开箱即用的GEval指标，支持自定义criteria，与pytest集成：

```python
from deepeval.metrics import GEval
from deepeval.test_case import LLMTestCase, LLMTestCaseParams

metric = GEval(
    name="Correctness",
    criteria="Determine if the actual output is correct based on the expected output.",
    evaluation_params=[LLMTestCaseParams.ACTUAL_OUTPUT, LLMTestCaseParams.EXPECTED_OUTPUT],
    threshold=0.5
)
test_case = LLMTestCase(
    input="What is the CEO's contact info?",
    actual_output="Zhang San, CEO, zhang@xyz.com, +86-21-1234-5678",
    expected_output="Zhang San, CEO, zhang@xyz.com, +86-21-1234-5678"
)
assert_test(test_case, [metric])
```

### 第3层：人工介入（校准而非逐条检查）

人工不是逐条审核（那样成本不可控），而是作为**校准层**，确保前两层自动化检查本身是可信的。

| 介入时机 | 目的 | 频率 |
|---------|------|------|
| 构建Golden Dataset | 标注50-100个真实case作为ground truth | 一次性，持续补充 |
| 校准LLM Judge | 抽样检查Judge评分是否与人类判断一致 | 每周/每月 |
| 低置信度审核 | Agent输出置信度低于阈值时路由给人工 | 实时但量少 |
| 失败案例分析 | 分析自动检查漏掉或误判的case，改进规则 | 按需 |

### 三层检查的设计哲学

这个方案的核心思想来自Anthropic提出的**"瑞士奶酪模型"**——每一层检查都有自己的盲区（像瑞士奶酪上的孔洞），但多层叠加后，问题同时穿过所有层的概率极低。

```
确定性检查（快、便宜、有明确对错标准的指标）
    ↓ 漏掉的模糊判断
LLM-as-Judge（灵活、可扩展、80%与人类一致）
    ↓ Judge自身的盲区
人工校准（金标准、高成本、低频次）
    ↓ 校准结果反哺上两层
```

---

## 三、可靠性度量：pass^k

在Agent系统中，单次准确率不够——生产环境需要的是**每次都对**，而不是"多试几次总有一次对"。

τ-bench（Sierra AI）引入了一个关键区分：

- **pass@k**：k次尝试中至少成功1次的概率。公式 `1-(1-p)^k`，k越大越趋近100%。这衡量的是"能力上限"。
- **pass^k**：k次尝试全部成功的概率。公式 `p^k`，要求p足够接近100%才能保持高分。这衡量的是"生产可靠性"。

| 单次准确率 p | pass@8（至少对1次） | pass^8（全部对） |
|-------------|-------------------|-----------------|
| 90% | 99.99% | 43% |
| 80% | 99.97% | 17% |
| 70% | 99.94% | 6% |

实验结果：GPT-4o在零售客服场景中pass@1不到50%，pass^8不到25%。

**启示**：对于金融合规、医疗等高风险场景，必须用pass^k来评估Agent的可靠性。一个90%准确率的Agent听起来不错，但连续跑8次全部正确的概率只有43%——这在每天自动处理数千条数据的生产系统中是不可接受的。

在监控体系中，我会对关键Agent定期跑pass^k测试：用同一批测试输入重复执行k次，检查结果一致性。pass^k下降是模型退化或prompt漂移的早期信号。

---

## 四、监控架构与工具选型

### 整体架构

```
Production Traces → Observability平台 → Real-time Dashboard → Alerting（异常检测）
                                                                    ↓
                                                            Offline Eval（定期批量评估）
                                                                    ↓
                                                            Quality Report（周/月报）
```

完整的质量保障不止线上监控，还包括CI/CD阶段的评估门禁：

```
代码/Prompt提交 → CI Automated Evals（每次commit）
               → Production Monitoring（实时）
               → A/B Testing（发版时1-2周验证）
               → User Feedback（持续收集）
               → Manual Review（每周抽样校准）
```

### 工具选型

**评估框架：**

| 框架 | 定位 | 核心能力 |
|------|------|---------|
| **DeepEval** (14K+ Stars) | 通用LLM Eval | pytest集成，50+内置指标，GEval支持自定义Judge |
| **Inspect AI** (UK AI安全研究所) | 学术/安全评估 | Dataset+Solver+Scorer架构，100+预构建eval |
| **RAGAS** | RAG专用 | Faithfulness/Relevancy等RAG核心指标 |
| **Bloom** (Anthropic官方) | 行为安全评估 | 4阶段pipeline自动生成对抗性测试 |
| **Promptfoo** | Prompt测试+红队 | CI/CD集成最完整，GitHub Actions原生支持 |

**可观测性平台：**

| 平台 | 开源 | 核心能力 |
|------|------|---------|
| **Langfuse** | Yes（可自托管） | Trace/Score/Session三级评分，内置LLM-as-Judge |
| **LangSmith** | No（SaaS） | LangChain官方，Agent链路追踪最完整 |
| **Arize Phoenix** | Yes | 开源追踪 + 商业平台 |
| **Helicone** | Yes | 轻量级，专注请求日志和成本追踪 |

**我的推荐组合**：Langfuse（开源可自托管，做生产监控和Trace）+ DeepEval（做CI/CD评估门禁，与pytest集成）+ 每周人工抽样校准。

### CI/CD集成示例

将评估嵌入PR流程，Prompt或Agent代码变更时自动跑评估，不通过则阻止合并：

```yaml
name: AI Quality Gate
on:
  pull_request:
    paths: ['prompts/**', 'agents/**', 'config/**']
jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install deepeval
      - name: Run evals
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: deepeval test run tests/eval_*.py --exit-on-first-failure
```

---

## 五、构建评估体系的起步路线

参考Anthropic "Demystifying Evals for AI Agents" 的建议，构建评估体系不需要一步到位：

1. **从20-50个真实失败案例开始**——不需要数百个测试用例，真实failure case的价值远高于人造数据
2. **把手动QA检查转化为自动化test case**——团队日常review中发现的问题就是最好的测试素材
3. **设计无歧义的Task**——两个领域专家独立评估应得出相同的pass/fail判定
4. **Grader优先用确定性规则**——能用代码判断对错的就不要用LLM
5. **评估Outcome而非Trajectory**——Agent可能走不同路径达到相同正确结果
6. **监控eval饱和度**——当某个eval的通过率长期100%时，它已经不再提供信息，应转为regression suite，同时补充新的edge case

---

## 参考来源

- [Anthropic: Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [Anthropic: Bloom - Automated Behavioral Evaluations](https://alignment.anthropic.com/2025/bloom-auto-evals/)
- [Sierra AI: τ-Bench - Benchmarking AI Agents](https://sierra.ai/blog/benchmarking-ai-agents)
- [Evidently AI: LLM-as-a-Judge Complete Guide](https://www.evidentlyai.com/llm-guide/llm-as-a-judge)
- [AWS: Evaluating AI Agents - Real-World Lessons from Amazon](https://aws.amazon.com/blogs/machine-learning/evaluating-ai-agents-real-world-lessons-from-building-agentic-systems-at-amazon/)
- [Hugging Face: LLM-as-a-Judge Cookbook](https://huggingface.co/learn/cookbook/en/llm_judge)
- [GitHub: confident-ai/deepeval](https://github.com/confident-ai/deepeval)
- [GitHub: UKGovernmentBEIS/inspect_ai](https://github.com/UKGovernmentBEIS/inspect_ai)
- [GitHub: sierra-research/tau2-bench](https://github.com/sierra-research/tau2-bench)
