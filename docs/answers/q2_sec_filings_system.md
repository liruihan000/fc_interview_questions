# 问题2：SEC Filings 自动化系统设计

> **原题**：公司每天会通过人工的方式获取客户在SEC上公开的filing数据，识别出具有中国背景的公司，并提取公司负责人联系方式，用于后续商务拓展。请你针对以上业务场景，设计一个基于AI的自动化系统方案，并回答以下问题：如何实现对 SEC filings 的每日自动获取与增量更新？如何定义并识别中国背景公司？如何从非结构化文本中准确抽取公司负责人姓名与联系方式？如何控制误判率与漏判率？如何设计数据质量监控与人工校验流程？该系统的核心技术风险与合规风险是什么？如何规避？如果模型成本过高，你会如何优化整体成本结构？

---

## 前置分析：SEC数据结构 — 先搞清楚数据长什么样（并非所有的情景都需要llm来解决，尤其当hardcode可以更快更低成本实现时）

> **设计AI方案之前，理解数据源**。SEC EDGAR的数据相对结构化——很多信息可以直接通过API字段获取，不需要LLM。

### EDGAR核心API与数据结构

**Submissions API** (`data.sec.gov/submissions/CIK{cik}.json`)：

| 字段 | 类型 | 示例 | 用途 |
|------|------|------|------|
| `stateOfIncorporation` | string | `"E9"`(开曼) / `"F4"`(中国) / `"K3"`(香港) | **直接判断注册地** |
| `stateOfIncorporationDescription` | string | `"Cayman Islands"` | 人类可读的注册地 |
| `addresses.business.countryCode` | string | `"K3"`(香港) / `"L2"`(爱尔兰) | **实际办公地** |
| `addresses.business.country` | string | `"Hong Kong"` | 人类可读 |
| `phone` | string | `"(408) 996-1010"` | **公司主电话，结构化字段** |
| `name` | string | `"PDD Holdings Inc."` | 公司法律名称 |
| `tickers` | list | `["PDD"]` | 股票代码 |
| `exchanges` | list | `["Nasdaq"]` | 上市交易所 |
| `sic` / `sicDescription` | string | `"5961"` / `"Catalog & Mail-Order Houses"` | 行业分类 |
| `ein` | string | `"000000000"` | 外国公司EIN全0 |
| `entityType` | string | `"operating"` | 实体类型 |

**关键国家/地区编码**（`stateOfIncorporation` 和 `countryCode` 共用）：

| 编码 | 国家/地区 | 含义 |
|------|----------|------|
| `F4` | China (中国大陆) | **直接命中** |
| `K3` | Hong Kong (香港) | **直接命中** |
| `E9` | Cayman Islands (开曼) | **需进一步判断**（可能是VIE结构的中国公司，也可能不是） |
| `D0` | Bermuda (百慕大) | 同上 |
| `D8` | British Virgin Islands | 同上 |
| `1T` | Marshall Islands | 通常是航运公司，非中国背景 |

**Submissions API中NOT结构化的数据**（需要解析Filing文本）：
- 高管/董事姓名 → 在DEF 14A (proxy statement) 的**非结构化HTML**中
- 高管邮箱 → 偶尔出现在Filing文本中，无结构化字段
- 子公司列表 → 在Exhibit 21的**非结构化文本**中
- VIE结构披露 → 在10-K注释的**非结构化文本**中
- 审计师信息 → 在10-K audit report的**非结构化文本**中

### 核心结论：什么需要LLM，什么不需要

| 任务 | 需要LLM？ | 理由 |
|------|----------|------|
| 判断注册地是否中国/香港 | **不需要** | `stateOfIncorporation` = `F4`/`K3`，一行代码 |
| 判断开曼/百慕大公司是否有中国背景 | **需要** | 需阅读10-K文本判断VIE结构、业务描述、子公司 |
| 获取公司主电话 | **不需要** | `phone` 字段直接返回 |
| 获取公司地址 | **不需要** | `addresses.business` 结构化返回 |
| 提取高管姓名和职位 | **需要** | DEF 14A是非结构化HTML，需从文本中抽取 |
| 提取高管个人联系方式 | **需要** | 散布在多个Filing section的非结构化文本中 |
| 公司名称含"China/Sino" | **不需要** | `name` 字段 + 正则匹配 |

---

## 子问题1：如何实现对SEC filings的每日自动获取与增量更新？

### 数据源

| API | 端点 | 用途 | 限制 |
|-----|------|------|------|
| **Submissions API** | `data.sec.gov/submissions/CIK{cik}.json` | 获取公司元数据+filing历史 | 10 req/s，需User-Agent |
| **Full-Text Search** | `efts.sec.gov/LATEST/search-index?q=...` | 按日期/关键词搜索新filing | 同上 |
| **XBRL API** | `data.sec.gov/api/xbrl/companyfacts/CIK{cik}.json` | 结构化财务数据 | 同上 |
| **Bulk Data** | `companyfacts.zip` / `submissions.zip` | 每夜全量更新 | 文件大 |
| **RSS Feed** | EDGAR RSS | 实时推送新filing | 无 |

### 增量更新方案

```
Airflow DAG（每日定时 + RSS实时触发）
├── 1. 调用EDGAR Full-Text Search API，查询昨日新filing
├── 2. 按 accession number 去重（唯一标识）
├── 3. 对每条新filing，调用Submissions API获取公司元数据
├── 4. 增量入库（PostgreSQL），记录 last_sync_timestamp
└── 5. 触发下游筛选Pipeline
```

**注意**：SEC要求User-Agent header包含联系邮箱，限制10 req/s。无需API Key。

---

## 子问题2：如何定义并识别"中国背景"公司？

### 分层识别策略：结构化优先，LLM兜底

```
新Filing入库
│
├── 第1层：结构化字段过滤（零成本，代码规则）
│   ├── stateOfIncorporation ∈ {F4, K3} → 直接标记"中国背景" ✅
│   ├── addresses.business.countryCode ∈ {F4, K3} → 直接标记 ✅
│   ├── name 含 "China|Chinese|Sino|中国" → 直接标记 ✅
│   └── ein == "000000000" + stateOfIncorporation ∈ {E9, D0, D8}
│       → 外国公司注册在离岸地 → 进入第2层
│
├── 第2层：规则+关键词（低成本，代码规则）
│   ├── 10-K全文搜索 "operations in China|PRC|VIE|Variable Interest"
│   ├── Exhibit 21子公司列表中含中国地名/实体
│   └── 审计师匹配已知中国审计所名单
│   → 命中2+信号 → 标记"中国背景" ✅
│   → 命中1个 → 进入第3层
│
└── 第3层：LLM判断（仅对边界案例，量极少）
    ├── 用LLM阅读10-K相关section，判断是否有实质中国业务
    └── 输出: {is_chinese_background: bool, confidence: 0-100, reason: "..."}
```

**关键点**：预估70-80%的公司可以在第1层直接判断，15-20%在第2层解决，只有5-10%需要LLM。这意味着5000条/天中，只有~250-500条需要LLM调用。

### 模型选择（参考 [LLM_Comparison_2026](../LLM_Comparison_2026.md)）

- 第1-2层：**不用LLM**，纯代码规则
- 第3层：**MiMo-V2-Flash**（小米，$0.10/$0.30，开源 MoE 309B/15B活跃）— 分类任务 pattern 固定，不需要强推理，极低成本即可覆盖
- 不需要ReAct Agent——流程完全固定，一次structured output调用即可

---

## 子问题3：如何从非结构化文本中准确抽取负责人姓名与联系方式？

### 先用结构化数据，再用LLM补充

| 信息 | 来源 | 方式 |
|------|------|------|
| 公司主电话 | Submissions API `phone` 字段 | **直接取，不需要LLM** |
| 公司地址 | Submissions API `addresses.business` | **直接取** |
| 高管姓名+职位 | DEF 14A proxy statement | **需要LLM**（非结构化HTML表格） |
| 高管个人电话/邮箱 | 10-K cover page、proxy statement | **需要LLM**（散布在文本中） |

### 抽取架构

```
对每家已确认的中国背景公司：
│
├── 1. 结构化数据直取（代码Pipeline，无LLM）
│   ├── phone ← Submissions API
│   ├── address ← Submissions API
│   └── company_name, ticker, exchange ← Submissions API
│
├── 2. LLM抽取高管信息（ReAct Agent）
│   ├── Skill: extract_officers(filing_html)
│   │   → 从DEF 14A中抽取姓名、职位、薪酬
│   │   → Agent可动态决定去哪些section找（Cover Page → DEF 14A → 10-K Item 10）
│   └── 输出: [{name, title, phone?, email?}]
│
└── 3. 后处理Pipeline（代码写死，无LLM）
    ├── cross_verify: LinkedIn/Bloomberg API交叉验证
    ├── verify_quality: 格式校验、去重、完整度检查
    └── 入库 / 低置信度 → 人工审核队列
```

**为什么Step 2用ReAct Agent而非写死？**
高管信息可能分散在Filing的不同section，且格式不统一。Agent需要自主判断"DEF 14A没找到邮箱→去10-K cover page找"——这是需要动态决策的部分。但cross_verify和verify_quality是确定性操作，写死即可。

**模型选择**（参考 [LLM_Comparison_2026](../LLM_Comparison_2026.md)）：**MiniMax M2.5**（$0.30/$1.20，SWE-bench Verified 80.2% 全场最高，MMLU-Pro 85.0）+ structured output。联系方式抽取不需要顶级推理能力，M2.5 配合 JSON schema 约束足够且成本极低。低置信度 case 升级 **MiMo-V2-Pro**（$1.00/$3.00，1M 上下文，适合阅读完整长 filing）。

---

## 子问题4：如何控制误判率与漏判率？

### 分层指标策略

| | 筛选（子问题2） | 抽取（子问题3） |
|--|---------------|----------------|
| **核心指标** | Recall ≥ 95%（不能漏） | Precision + 字段级F1（不能错） |
| **容忍** | 高FP（多选只是多花后续成本） | 低FP（错误数据入库影响商务拓展） |
| **原因** | 漏掉 = 永远丢失客户 | 错误联系方式 = 商务信誉损失 |

### 控制策略

1. **结构化优先**：第1-2层用代码规则，准确率接近100%，零成本
2. **高召回筛选**：第3层LLM置信度>30即放行，宁多勿漏
3. **置信度分层入库**：高置信度自动入库，低置信度进人工审核队列
4. **多源交叉验证**：SEC数据 + LinkedIn + Bloomberg
5. **反馈循环**：人工校正结果回流，优化规则/Prompt

### Benchmark设计（本项目专用）

**Golden Dataset**：
```
50-100份SEC Filing人工标注：
├── 30份 明确中国背景（F4/K3直接命中）
├── 20份 离岸注册的中国VIE公司（E9+中国业务）
├── 20份 离岸注册的非中国公司（E9但非中国）
├── 15份 边界案例（华裔创始人但非中国业务等）
└── 15份 对抗性案例（格式异常、信息缺失）
```

**评估指标**：

| 阶段 | 指标 | 方法 |
|------|------|------|
| 筛选 | Recall, FNR | 确定性检查：golden label vs 系统输出 |
| 筛选 | pass^k一致性 | 同一输入跑k次，结果必须全部一致 |
| 抽取 | 字段级F1 | 姓名精确匹配 + 电话格式校验 + 完整度 |
| 抽取 | LLM-as-Judge | 对"理由"质量打分（1-4整数scale） |
| 整体 | 人工修正率 | 越低越好，目标<5% |

**CI/CD集成**：每次Prompt/规则变更自动回归跑Golden Dataset。

---

## 子问题5：如何设计数据质量监控与人工校验流程？

```
筛选 → 抽取 → 置信度评分 → 高置信度(>80) → 自动入库
                          → 中置信度(50-80) → 人工快速审核（只看关键字段）
                          → 低置信度(<50) → 人工详细审核
                                              ↓
                                        人工校正 → 结果回流 → 优化规则/Prompt
                                              ↓
                                        质量仪表板（Langfuse/自建）
```

**监控指标**：
- 每日筛选通过率（异常波动告警）
- 联系方式完整度（非null字段比例）
- 人工修正率趋势（越低越好）
- LLM调用量和成本（per-query追踪）

---

## 子问题6：核心技术风险与合规风险

| 风险类型 | 具体风险 | 规避措施 |
|----------|----------|----------|
| **合规风险** | SEC数据使用条款 | 仅使用公开数据，遵守SEC robots.txt和rate limit |
| **隐私风险** | 个人联系方式的采集和使用 | 遵守CAN-SPAM Act、TCPA；只采集公开filing中的信息 |
| **技术风险** | LLM幻觉导致错误联系方式 | 结构化数据优先 + 多源交叉验证 + 人工校验兜底 |
| **运营风险** | SEC API变更或限流 | 监控API健康、备用数据源(sec-api.io)、本地缓存 |
| **商誉风险** | 错误信息导致商务拓展失败 | 置信度分层 + 低置信度必须人工确认后才用于外联 |

---

## 子问题7：成本优化

### 最大的成本优化：减少LLM调用量

| 优化策略 | 效果 | 说明 |
|---------|------|------|
| **结构化优先** | **减少70-80%的LLM调用** | 5000条中~4000条可通过API字段+规则直接判断，只有~500-1000条需要LLM |
| **模型级联** | 再省60-80% | 分类用 MiMo-V2-Flash（$0.10/$0.30），抽取用 MiniMax M2.5（$0.30/$1.20），复杂 case 才升级 MiMo-V2-Pro（$1.00/$3.00） |
| **Prompt压缩** | 40-60% | 只传入Filing相关section，不传整个文档 |
| **Batch API** | 50% | 非实时任务，使用 MiniMax/小米 Batch API |
| **Prompt Caching** | 20-30% | 系统提示缓存，减少重复输入成本 |

### 成本估算

| 方案 | 每日成本 |
|------|---------|
| 全部5000条用顶级闭源模型（Opus/GPT-5） | ~$50-100/天 |
| 两步Agent（未优化） | ~$5-10/天 |
| **结构化优先 + MiMo/MiniMax 级联（优化后）** | **~$0.3-1/天** |

关键差异：优化后只有500-1000条需要LLM，分类用 MiMo-V2-Flash（$0.10/$0.30），抽取用 MiniMax M2.5（$0.30/$1.20），成本较顶级模型低 1-2 个数量级。

---

## 整体架构总览

```
Airflow 每日触发
│
├── 1. 数据获取（代码Pipeline，无LLM）
│   └── EDGAR API → 增量入库 → 5000条新filing/天
│
├── 2. 中国背景识别（分层，大部分无需LLM）
│   ├── 第1层：结构化字段 → ~70-80%直接判定
│   ├── 第2层：规则+关键词 → ~15-20%判定
│   └── 第3层：LLM分类 → ~5-10%边界case → ~50条候选
│
├── 3. 联系方式抽取（结构化+LLM混合）
│   ├── 结构化直取：phone, address ← API字段
│   ├── LLM抽取：高管姓名/职位/邮箱 ← ReAct Agent读DEF 14A
│   └── 后处理：交叉验证 + 质量校验（代码Pipeline）
│
├── 4. 置信度分层入库
│   ├── 高置信度 → 自动入库
│   └── 低置信度 → 人工审核队列
│
└── 5. 监控 & 反馈循环
    ├── 质量仪表板
    └── 人工校正 → 回流优化规则/Prompt
```
