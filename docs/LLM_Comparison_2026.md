# 2026 主流大语言模型（LLM）综合对比

> 更新时间：2026年3月 | 数据来源：LMSYS Chatbot Arena、BenchLM.ai、llm-stats.com、各厂商官方定价

---

## 一、综合排名概览（按 Chat Arena ELO 降序）

| 排名 | 模型 | 厂商 | 类型 | Chat Arena ELO | Code Arena | 上下文窗口 | 参数规模 |
|:---:|------|------|:----:|:-----------:|:----------:|:--------:|:-------:|
| 1 | Claude Opus 4.6 | Anthropic | 闭源 | 1,503 | 1,560 | 200K (1M可用) | — |
| 2 | Gemini 3.1 Pro | Google | 闭源 | 1,500 | — | 1M | — |
| 3 | Grok 4 | xAI | 闭源 | 1,495 | — | 128K | — |
| 4 | GPT-5.2 (Thinking) | OpenAI | 闭源 | 1,486 | — | 128K | — |
| 5 | GLM-5 (Reasoning) | 智谱AI | 开源 | 1,451 | 1,595 | 200K | — |
| 6 | Qwen3.5 397B (Reasoning) | 阿里巴巴 | 开源 | 1,450 | 1,214 | 128K | 397B (17B活跃) |
| 7 | Kimi K2.5 (Reasoning) | 月之暗面 | 闭源 | 1,447 | — | 128K | — |
| 8 | Claude Sonnet 4.6 | Anthropic | 闭源 | — | 1,531 | 200K | — |
| 9 | MiMo-V2-Pro | 小米 | 闭源(API) | 1,426 | — | 1M | 1T+ |
| 10 | MiniMax M2.5 | MiniMax | 闭源 | 1,421 | — | 205K | 230B |
| 11 | DeepSeek V3.2 | DeepSeek | 开源 | 1,421 | — | 128K | — |
| 12 | MiniMax M2.7 | MiniMax | 闭源 | — | — | 205K | — |
| 13 | GPT-5.4 | OpenAI | 闭源 | 1,146 | 1,679 | 128K | — |
| 14 | GPT-5 | OpenAI | 闭源 | — | — | 128K | — |
| 15 | o3 | OpenAI | 闭源 | — | — | 200K | — |
| 16 | o4-mini | OpenAI | 闭源 | — | — | 200K | — |
| 17 | MiMo-V2-Flash | 小米 | 开源 | — | — | 256K | 309B (15B活跃) |
| 18 | MiMo-V2-Omni | 小米 | 闭源(API) | — | — | 256K | — |
| 19 | Mistral Large 3 | Mistral | 开源 | — | — | 128K | — |
| 20 | Llama 4 Maverick | Meta | 开源 | — | — | 1M (Scout 10M) | 400B+ |
| 21 | Claude Haiku 4.5 | Anthropic | 闭源 | — | — | 200K | — |
| 22 | Gemini 2.5 Flash | Google | 闭源 | — | — | 1M | — |
| 23 | GPT-5 Mini | OpenAI | 闭源 | — | — | 128K | — |
| 24 | Grok 4.1 Fast | xAI | 闭源 | — | — | 128K | — |

---

## 二、关键 Benchmark 对比（按 Chat Arena ELO 降序）

| 排名 | 模型 | GPQA Diamond | SWE-bench Verified | SWE-bench Pro | AIME 2025 | MATH 500 | LiveCodeBench | MMLU-Pro |
|:---:|------|:----------:|:-----------------:|:------------:|:---------:|:--------:|:-------------:|:-------:|
| 1 | Claude Opus 4.6 | — | **80.8%** | — | — | — | — | — |
| 2 | Gemini 3.1 Pro | **94.3%** | 76.2% | — | ~95 | — | — | — |
| 3 | Grok 4 | 88.4% | — | — | — | — | — | — |
| 4 | GPT-5.2 (Thinking) | 93.2% | 80.0% | 55.6% | **100** | — | — | — |
| 5 | GLM-5 (Reasoning) | — | 77.8% | — | 98 | **92** | — | — |
| 6 | Qwen3.5 397B (Reasoning) | — | — | — | — | — | — | — |
| 7 | Kimi K2.5 (Reasoning) | — | 76.8% | **70** | — | — | **85** | — |
| 8 | Claude Sonnet 4.6 | — | — | — | — | — | — | — |
| 9 | MiMo-V2-Pro | — | — | — | — | — | — | — |
| 10 | MiniMax M2.5 | 79.9% | 80.2% | — | — | — | — | 85.0 |
| 11 | DeepSeek V3.2 | 79.9% | 72–74% | — | 89.3 | — | — | — |
| 12 | MiniMax M2.7 | — | 78% | 56.2% | — | — | — | — |
| 13 | GPT-5.4 | 92.8% | — | — | — | — | — | — |
| 14 | GPT-5 | — | — | — | — | — | — | — |
| 15 | o3 | — | — | — | — | — | — | — |
| 16 | o4-mini | — | — | — | — | — | — | — |
| 17 | MiMo-V2-Flash | — | 73.4% | — | — | — | 80.6 | — |
| 18 | MiMo-V2-Omni | — | — | — | — | — | — | — |
| 19 | Mistral Large 3 | — | — | — | — | — | — | — |
| 20 | Llama 4 Maverick | — | — | — | — | — | — | — |

> **注**：各厂商公布的 benchmark 侧重不同，"—" 表示未公开或未收录该项数据。**加粗**为该项最高分。

---

## 三、API 定价对比（USD / 百万 token，按 Chat Arena ELO 降序）

| 排名 | 模型 | 厂商 | 类型 | 输入价格 | 输出价格 | 性价比评级 |
|:---:|------|------|:----:|:-------:|:-------:|:--------:|
| 1 | Claude Opus 4.6 | Anthropic | 闭源 | $5.00 | $25.00 | ⭐⭐ |
| 2 | Gemini 3.1 Pro | Google | 闭源 | $2.00–4.00 | $12.00–18.00 | ⭐⭐⭐ |
| 3 | Grok 4 | xAI | 闭源 | $3.00 | $15.00 | ⭐⭐⭐ |
| 4 | GPT-5.2 | OpenAI | 闭源 | $1.75 | $14.00 | ⭐⭐⭐ |
| 5 | GLM-5 | 智谱AI | 开源 | $1.00 | $3.20 | ⭐⭐⭐⭐ |
| 6 | Qwen3.5 397B | 阿里巴巴 | 开源 | 自部署 | 自部署 | ⭐⭐⭐⭐⭐ |
| 7 | Kimi K2.5 | 月之暗面 | 闭源 | $0.60 | $2.50 | ⭐⭐⭐⭐⭐ |
| 8 | Claude Sonnet 4.6 | Anthropic | 闭源 | $3.00 | $15.00 | ⭐⭐⭐ |
| 9 | MiMo-V2-Pro | 小米 | 闭源(API) | $1.00 | $3.00 | ⭐⭐⭐⭐ |
| 10 | MiniMax M2.5 | MiniMax | 闭源 | $0.30 | $1.20 | ⭐⭐⭐⭐⭐ |
| 11 | DeepSeek V3.2 | DeepSeek | 开源 | $0.28 | $0.42 | ⭐⭐⭐⭐⭐ |
| 12 | MiniMax M2.7 | MiniMax | 闭源 | $0.30 | $1.20 | ⭐⭐⭐⭐⭐ |
| 13 | GPT-5.4 | OpenAI | 闭源 | — | — | — |
| 14 | GPT-5 | OpenAI | 闭源 | $1.25 | $10.00 | ⭐⭐⭐ |
| 15 | o3 | OpenAI | 闭源 | $2.00 | $8.00 | ⭐⭐⭐ |
| 16 | o4-mini | OpenAI | 闭源 | $1.10 | $4.40 | ⭐⭐⭐⭐ |
| 17 | MiMo-V2-Flash | 小米 | 开源 | $0.10 | $0.30 | ⭐⭐⭐⭐⭐ |
| 18 | MiMo-V2-Omni | 小米 | 闭源(API) | $0.40 | $2.00 | ⭐⭐⭐⭐ |
| 19 | Mistral Large 3 | Mistral | 开源 | $2.00 | $6.00 | ⭐⭐⭐ |
| 20 | Llama 4 Maverick | Meta | 开源 | 自部署 | 自部署 | ⭐⭐⭐⭐⭐ |
| 21 | Claude Haiku 4.5 | Anthropic | 闭源 | $1.00 | $5.00 | ⭐⭐⭐⭐ |
| 22 | Gemini 2.5 Flash | Google | 闭源 | $0.30 | $2.50 | ⭐⭐⭐⭐ |
| 23 | GPT-5 Mini | OpenAI | 闭源 | $0.25 | $2.00 | ⭐⭐⭐⭐ |
| 24 | Grok 4.1 Fast | xAI | 闭源 | $0.20 | $0.50 | ⭐⭐⭐⭐⭐ |

> **补充低成本选项**：GPT-5 Nano ($0.05/$0.40)、Gemini 2.5 Flash-Lite ($0.10/$0.40)、Mistral Nemo ($0.02/$0.02)

---

## 四、模型特点分析（按 Chat Arena ELO 降序）

| 排名 | 模型 | 类型 | 核心优势 | 适用场景 |
|:---:|------|:----:|---------|---------|
| 1 | Claude Opus 4.6 | 闭源 | Code Arena #1（1560），SWE-bench 80.8%，复杂架构重构 | 高端编程、复杂推理、长文档 |
| 2 | Gemini 3.1 Pro | 闭源 | GPQA 94.3% 全场最高，1M 上下文，多模态 | 科研、长文档、多模态分析 |
| 3 | Grok 4 | 闭源 | 实时网络搜索集成，响应速度快 | 实时信息、快速问答 |
| 4 | GPT-5.2 (Thinking) | 闭源 | AIME 满分 100，GPQA 93.2%，ARC-AGI >90% | 数学、推理、学术研究 |
| 5 | GLM-5 (Reasoning) | 开源 | AIME 98 数学近满分，Agent 能力全场领先（Terminal-Bench 81） | 数学推理、自主 Agent |
| 6 | Qwen3.5 397B | 开源 | MoE 397B/17B活跃，均衡通用，多语言支持强 | 通用任务、多语言、国内部署 |
| 7 | Kimi K2.5 | 闭源 | SWE-bench Pro 70 全场最高，LiveCodeBench 85 | 代码生成、实际工程 |
| 8 | Claude Sonnet 4.6 | 闭源 | GDPval 1633 全场最高（实际办公任务），Code Arena #3 | 企业日常、编程性价比 |
| 9 | MiMo-V2-Pro | 闭源(API) | 1T+ 参数，1M 上下文，价格仅顶级模型 1/6 | 长文档、通用推理 |
| 10 | MiniMax M2.5 | 闭源 | SWE-bench 80.2% 全场最高，230B 高效部署 | 软件工程、自动修复 |
| 11 | DeepSeek V3.2 | 开源 | MIT 协议无限制，$0.28/$0.42 极致低价，AIME 89.3 | 推理、数学、预算有限团队 |
| 12 | MiniMax M2.7 | 闭源 | SWE-bench 78%，自主迭代能力提升 30%，极低定价 | 批量代码任务、成本敏感 |
| 13 | GPT-5.4 | 闭源 | Code Arena 1679 排名高，GPQA 92.8% | 编程、企业集成 |
| 14 | GPT-5 | 闭源 | 生态完善，插件丰富，稳定性高 | 通用任务、企业集成 |
| 15 | o3 | 闭源 | 推理优先架构，复杂逻辑链 | 深度推理、数学证明 |
| 16 | o4-mini | 闭源 | 推理能力 + 低成本平衡 | 轻量推理、日常推理任务 |
| 17 | MiMo-V2-Flash | 开源 | MoE 309B/15B活跃，SWE 73.4%，$0.10/$0.30 极低价 | 自部署编程助手、低延迟 |
| 18 | MiMo-V2-Omni | 闭源(API) | 多模态（语音+视觉+文本），Agent 和机器人场景优化 | 多模态应用、机器人 |
| 19 | Mistral Large 3 | 开源 | 开源 LMArena #2，欧洲合规友好 | 欧洲部署、GDPR 合规 |
| 20 | Llama 4 Maverick | 开源 | Scout 版本 10M 超长上下文，社区生态最大 | 超长文档、社区二次开发 |

---

## 五、性价比推荐

### 按使用场景推荐

| 场景 | 首选推荐 | 高性价比替代 | 理由 |
|------|---------|------------|------|
| 极致编程 | Claude Opus 4.6 | Kimi K2.5 | Opus Code Arena 断层领先；Kimi SWE-bench Pro 70 最高 |
| 编程高性价比 | DeepSeek V3.2 | MiMo-V2-Flash | 价格仅为 Opus 的 1/50，编程能力优秀 |
| 数学推理 | GPT-5.2 (Thinking) | GLM-5 | GPT-5.2 AIME 满分；GLM-5 AIME 98 且开源 |
| 超长文档 | Gemini 3.1 Pro | Llama 4 Scout / MiMo-V2-Pro | 1M–10M 上下文 |
| 企业通用 | Claude Sonnet 4.6 | GPT-5 | GDPval 实际办公 #1；GPT 生态完善 |
| 极低成本 | DeepSeek V3.2 | MiMo-V2-Flash / Gemini Flash-Lite | 输入低至 $0.10/M，输出低至 $0.30/M |
| 国内部署 | Qwen3.5 397B | DeepSeek V3.2 / GLM-5 | 开源、中文优化、合规 |
| Agent 任务 | GLM-5 | Kimi K2.5 | Terminal-Bench 81、BrowseComp 80 领先 |
| 多模态 | Gemini 3.1 Pro | MiMo-V2-Omni | 视觉+文本+长上下文 |
| 软件工程自动修复 | MiniMax M2.5 | Claude Opus 4.6 | SWE-bench Verified 80.2% 全场最高 |

---

## 六、趋势观察

1. **价格暴跌**：2025→2026 API 价格整体下降约 80%，DeepSeek V3.2 以 $0.28/$0.42 刷新底价
2. **中国模型崛起**：开源 TOP10 中 8 个来自中国，Hugging Face 下载量占比超 60%
3. **MoE 架构普及**：MiMo-V2-Flash（309B/15B活跃）、Qwen3.5（397B/17B活跃）大幅降低推理成本
4. **上下文窗口竞赛**：Llama 4 Scout 达到 10M token，Gemini/MiMo-V2-Pro 达到 1M
5. **推理模型分化**：Reasoning 模式（GLM-5、Kimi K2.5、GPT-5.2 Thinking）在数学和代码上显著超越标准模式
6. **小米异军突起**：MiMo-V2 系列三款模型（Pro/Flash/Omni）覆盖全场景，Flash 以 $0.10/$0.30 极低价进入第一梯队
7. **SWE-bench 成为核心指标**：MiniMax M2.5（80.2%）、Claude Opus 4.6（80.8%）、GPT-5.2（80.0%）三强鼎立

---

## 数据来源

- [LMSYS Chatbot Arena (lmarena.ai)](https://lmarena.ai/)
- [BenchLM.ai - Chinese LLM Rankings](https://benchlm.ai/blog/posts/best-chinese-llm)
- [LLM Stats - AI Leaderboards 2026](https://llm-stats.com/)
- [TLDL - LLM API Pricing 2026](https://www.tldl.io/resources/llm-api-pricing-2026)
- [PricePerToken.com](https://pricepertoken.com/)
- [OpenRouter](https://openrouter.ai/)
- [Artificial Analysis](https://artificialanalysis.ai/)
- 各厂商官方定价页面（Anthropic、OpenAI、Google、DeepSeek、MiniMax、小米、智谱、月之暗面）
