# SiteIntel X — Search Intelligence 产品与技术升级方案

> 项目：siteintel.cc  
> 产品：SiteIntel X  
> 核心定位：**Website Data Intelligence + Search Intelligence**  
> 核心目标：在现有 SiteIntel 数据采集与情报分析基础上，建立以 Google / Baidu / Bing 为第一梯队的多搜索引擎数据体系，最终形成“搜索表现 + 网站基础设施 + 技术 + 历史变化 + 情报洞察”的统一分析平台。

---

# 1. 项目总原则

SiteIntel X **不是推倒重做**。

现有 SiteIntel 已经具备：

- 真实 Provider 数据采集
- PostgreSQL + Prisma
- Investigation Pipeline
- Raw Collections
- Entity / Relationship
- Snapshot
- Event
- Insight Engine
- Evidence
- Confidence
- Website Intelligence Report
- Related Websites
- Relationship Graph
- Monitor
- Dashboard
- Bulk Investigation
- API
- AI Explanation

当前 MVP、第二阶段和第三阶段均已完成，未完成项目主要是计费、Redis、独立 Worker、截图 Provider、部分监控频率等非核心能力。fileciteturn4file0L21-L21 fileciteturn4file0L346-L354

因此升级原则：

> **保留已经验证的数据底座，重新强化产品价值与情报分析能力。**

---

# 2. SiteIntel X 的核心定位

不要定位成：

- SEO Checker
- DNS Checker
- IP Lookup
- Website Scanner
- Technology Detector

这些都是底层能力。

最终定位：

# Website Data Intelligence & Search Intelligence

中文：

# 网站数据分析与搜索情报平台

核心回答：

```text
这个网站是什么？
它是怎么运行的？
它依赖什么基础设施？
它发生了什么变化？
它在搜索引擎中的表现怎么样？
搜索表现为什么发生变化？
哪些变化值得关注？
```

---

# 3. 最重要的产品认知变化

站长真正长期关心的 SEO 结果，不是：

- Title 字符数
- Meta Description 长度
- Canonical 是否存在
- Structured Data 是否存在

这些只是原因层和技术层。

站长更关心：

```text
收录量
关键词数量
关键词排名
Top 3 / Top 10 / Top 20
搜索曝光
搜索点击
CTR
自然搜索流量
页面表现
排名涨跌
收录涨跌
流量涨跌
```

因此 SiteIntel 的 SEO 产品核心必须从：

> Technical SEO Checker

升级为：

# Search Performance Intelligence

---

# 4. Search Intelligence 产品结构

Search Intelligence 分成六层：

```text
SEARCH PERFORMANCE
    ↓
SEARCH VISIBILITY
    ↓
INDEX INTELLIGENCE
    ↓
PAGE INTELLIGENCE
    ↓
TECHNICAL SIGNALS
    ↓
SEARCH EVENT CORRELATION
```

---

# 5. Search Performance

用于回答：

> 搜索流量到底发生了什么？

核心指标：

- Organic Clicks
- Impressions
- CTR
- Average Position
- Organic Traffic
- Search Engine Share

支持：

- Today
- 7 Days
- 30 Days
- 90 Days
- 12 Months

核心展示：

```text
Organic Search

Clicks       +18.4%
Impressions  +12.2%
CTR           +5.7%
Position      +2.1
Traffic      +21.3%
```

重点展示变化，而不是只展示静态数字。

---

# 6. Search Visibility

用于回答：

> 网站在搜索结果里的可见程度有没有变化？

指标：

```text
Keywords
Top 3
Top 10
Top 20
Top 50
Top 100
```

趋势：

```text
New
Rising
Stable
Falling
Lost
```

建议引入 SiteIntel 自有指标：

# Search Visibility

而不是模糊的“网站权重”。

---

# 7. Search Visibility 的设计原则

“权重”不是一个所有搜索引擎都统一定义的官方指标。

因此 SiteIntel 不应该直接告诉用户：

> 网站权重 = 82

而应该使用可解释的：

- Search Visibility
- Search Authority
- Ranking Distribution

其中：

### Search Visibility

基于真实搜索表现计算。

可考虑：

```text
Keyword Distribution
+
Ranking Position
+
Impressions
+
Clicks
+
Indexed Pages
```

所有评分必须可解释。

---

# 8. Index Intelligence

这是站长最关注的核心模块之一。

展示：

```text
Indexed Pages
Index Growth
Index Loss
New Indexed Pages
Deindexed Pages
Index Coverage
```

例如：

```text
Google
12,482
↑ 8.4%

Baidu
8,731
↓ 3.1%

Bing
11,902
↑ 6.2%
```

重点不是单一收录数字。

而是：

# Index Change Intelligence

例如：

> 过去 7 天 Google 收录量增加 8.4%，百度收录量下降 3.1%。

---

# 9. Index Event

系统需要将收录变化事件化：

```text
INDEX_GROWTH
INDEX_DROP
INDEX_REGRESSION
INDEX_RECOVERY
PAGE_NEWLY_INDEXED
PAGE_DEINDEXED
```

这些事件可以与现有 SiteIntel `events` 表保持一致。

**不要新建一个平行的 search_events 表，除非现有 events 无法表达。**

---

# 10. Page Intelligence

站长最终需要知道：

> 到底哪些页面在涨，哪些页面在掉？

页面维度：

```text
URL
Clicks
Impressions
CTR
Position
Index Status
Traffic Change
Ranking Change
Last Observed
```

页面分类：

```text
Rising Pages
Declining Pages
New Pages
Lost Pages
High Traffic Pages
High Impression / Low CTR Pages
```

---

# 11. Keyword Intelligence

关键词页面展示：

```text
Keyword
Search Engine
Position
Previous Position
Change
Clicks
Impressions
CTR
URL
```

排名变化：

```text
+ New
+ Rising
→ Stable
- Falling
- Lost
```

核心统计：

```text
Top 3
Top 10
Top 20
Top 50
Top 100
```

---

# 12. Multi-Search Engine

第一阶段只优先做三家：

# Tier 1

1. Google
2. Baidu
3. Bing

这是第一阶段必须完成的 Search Intelligence 基础。

---

# 13. Tier 2 搜索引擎

第二阶段：

- Yandex
- Naver
- Shenma
- Sogou

目标：

形成多地区搜索数据体系。

---

# 14. Tier 3 搜索引擎

未来根据用户与数据价值再考虑：

- Seznam
- DuckDuckGo
- Brave Search
- 其他区域搜索引擎

不要一开始为了“覆盖搜索引擎数量”而开发。

优先顺序：

```text
数据质量
>
API 稳定性
>
用户价值
>
市场覆盖
>
数量
```

---

# 15. Search Data 的两种来源必须严格区分

这是架构上非常重要的一点。

## A. Owned Search Data

用户授权自己的网站。

例如：

```text
Google Search Console
Baidu Webmaster / Search Resource
Bing Webmaster
Yandex Webmaster
Naver Search Advisor
...
```

可以获得：

- 实际点击
- 实际曝光
- CTR
- 实际排名
- 查询词
- 页面表现
- 索引相关数据
- 抓取数据（视平台能力）

这是：

# 第一方搜索表现数据

---

## B. SERP Intelligence

不要求网站所有权。

用于：

> 查询任意关键词和任意网站的公开搜索排名。

例如：

```text
Keyword:
website analytics

Google
#5

Bing
#3

Baidu
#9
```

这是：

# 第三方公开 SERP 数据

两者不能混为一谈。

---

# 16. SiteIntel 的真正差异化

不能只做：

```text
Google 数据搬运
+
Baidu 数据搬运
+
Bing 数据搬运
```

真正的价值是：

# Search Data Fusion

例如：

```text
Google
+
Baidu
+
Bing
+
Infrastructure
+
Technology
+
History
+
Events
```

形成：

# Search Intelligence

---

# 17. Cross-Engine Intelligence

一个关键词：

```text
website analytics
```

可以展示：

```text
Google      #8
Bing        #5
Baidu       #23
Yandex      #11
```

然后：

```text
Cross-engine Visibility
72%
```

并分析：

- 哪个搜索引擎表现最好
- 哪个搜索引擎表现最差
- 哪些关键词在不同搜索引擎产生明显差异
- 哪些页面具有跨搜索引擎稳定表现

---

# 18. Search Engine Comparison

Dashboard 增加：

```text
Search Engines

Google
Baidu
Bing
Yandex
Naver
Shenma
Sogou
```

每个平台显示：

```text
Index
Keywords
Clicks
Impressions
CTR
Average Position
Visibility
Trend
```

未连接平台：

```text
Not Connected
```

不得伪造数据。

---

# 19. Search Event Correlation

这是 SiteIntel 最重要的创新方向之一。

把：

```text
Infrastructure Events
Technology Events
Search Events
Index Events
Ranking Events
Traffic Events
```

放进同一时间轴。

例如：

```text
2026-08-10
IP Changed

2026-08-11
Technology Changed

2026-08-12
Indexed Pages -12%

2026-08-13
Top 10 Keywords -8%

2026-08-14
Organic Traffic -17%
```

然后产生：

# Possible Causal Chain

> 检测到基础设施变化后，网站索引覆盖率下降，随后关键词排名和自然搜索流量同步下降。

必须使用：

- Possible
- Likely
- Probable

等非绝对表达。

不得把相关性自动声明为因果关系。

---

# 20. Search Insight

Search Insight 必须基于现有：

```text
events
+
evidence
+
insights
```

扩展，而不是创建一套平行架构。

例如：

```json
{
  "type": "search_regression",
  "status": "probable",
  "confidence": 87,
  "supporting_evidence": [
    "index_drop",
    "keyword_drop",
    "traffic_drop"
  ],
  "contradicting_evidence": [
    "impressions_stable"
  ]
}
```

---

# 21. Technical SEO 的正确位置

Technical SEO 不是主产品。

它属于：

# Technical Signals

包括：

- Robots
- Sitemap
- Canonical
- Meta
- Structured Data
- Crawlability
- Indexability
- HTTP
- Redirect
- Performance
- Internal Links

它的作用是：

> **解释 Search Performance 为什么发生变化。**

---

# 22. Search Health

增加一个：

# Search Health

不是网站“好坏分”。

而是搜索表现健康程度。

示例：

```text
Search Health
91

Index Coverage        94
Ranking Momentum      88
Traffic Stability      93
CTR Efficiency         82
Technical Crawl        95
```

每项必须有依据。

---

# 23. Search Visibility Dashboard

报告顶部增加：

```text
SEARCH INTELLIGENCE

Search Visibility
82.4
↑ 12.7%

Organic Traffic
+18.4%

Indexed Pages
+6.2%

Keywords
+9.8%

Top 10 Keywords
+14.3%
```

下面：

```text
What Changed?

↑ 183 keywords gained
↓ 42 keywords lost
↑ 128 pages indexed
↓ 17 pages removed
↑ 12 pages gained traffic
↓ 8 pages lost traffic
```

---

# 24. Why Changed

SiteIntel 的核心价值：

```text
What happened?
↓
Why?
↓
What evidence supports it?
```

例如：

```text
Traffic
↓ 23%

Possible causes:

Index Coverage
↓ 8%

Top 10 Keywords
↓ 13%

Robots.txt
Changed

Canonical
Changed
```

最终：

> 可能与索引覆盖下降以及近期抓取配置变化有关。

---

# 25. Search Intelligence 与现有 SiteIntel 的关系

现有：

```text
Website
↓
DNS
↓
IP
↓
ASN
↓
Technology
↓
SSL
↓
HTTP
↓
Snapshot
↓
Event
↓
Insight
```

升级：

```text
Website
├── Infrastructure
├── Technology
├── Security
├── Search
└── History
        ↓
      Events
        ↓
      Evidence
        ↓
      Insights
```

Search 是新的 Observation 类别，不是独立产品。

---

# 26. 数据模型原则

不要立即新建：

```text
signals
patterns
hypotheses
intelligences
search_events
```

如果现有：

```text
events
insights
snapshots
evidence
```

可以表达，就继续演化现有结构。

例如：

## events 增加

```text
source
category
observed_at
previous
current
```

category：

```text
infrastructure
technology
search
index
ranking
traffic
performance
```

---

# 27. Search Observation

可以在现有 Snapshot 模型上增加：

```text
search
index
ranking
traffic
```

等快照类别。

例如：

```text
snapshot.type = search
snapshot.type = index
snapshot.type = ranking
```

继续沿用当前 Hash 去重与历史机制。

---

# 28. 数据时效

第一阶段不做复杂 Evidence Decay 数学模型。

只增加：

```text
observed_at
last_verified_at
freshness
```

状态：

```text
fresh
aging
stale
expired
```

不同数据类型使用不同 TTL。

---

# 29. 搜索数据增长策略

这是必须提前解决的问题。

不能只等用户每天手工查询。

第一阶段：

```text
User-connected Sites
+
Seed Sites
+
User Investigations
+
Related Domain Candidates
```

数据成长：

```text
100
↓
1,000
↓
10,000
```

先建立有价值的观察池。

---

# 30. Candidate Discovery

当一个网站被分析后：

```text
IP
SSL
Nameserver
Technology
Tracking ID
Organization
```

可以发现：

```text
Candidate Websites
```

但：

> Candidate 不等于永久观察。

使用：

```text
priority score
```

决定是否进一步分析。

---

# 31. Similarity 暂不全量开发

当前数据规模较小时：

# On-demand Similarity

用户查看网站时才计算：

```text
Top Similar Websites
```

不要一开始做：

```text
N × N
```

全库两两比较。

---

# 32. Cluster 延后

Cluster 需要：

- 大量网站
- 大量关系
- 大量历史
- 稳定 Feature

数据量不足时不开发。

未来达到规模再使用：

```text
Feature Vector
+
Candidate Retrieval
+
Approximate Nearest Neighbor
+
Worker
```

---

# 33. Redis / Worker

当前不立即引入。

保持现有：

- Next.js
- PostgreSQL
- Prisma
- 异步 Pipeline

只有出现：

```text
大量 Monitor
大量 Search Refresh
大量 SERP Query
大量 Similarity
大量 Batch
```

再引入：

```text
Redis
Queue
Worker
```

---

# 34. SEO Growth

SiteIntel 自身也需要 SEO。

但不能做：

```text
百万个空壳 Domain Pages
```

不能批量生成低价值 AI 内容。

采用：

# Intelligence-driven SEO

页面必须拥有：

```text
真实数据
+
真实变化
+
真实关系
+
独特 Insight
```

---

# 35. SEO 页面分层

## 第一层：产品页

```text
/website-intelligence
/search-intelligence
/infrastructure-intelligence
/technology-intelligence
/website-monitoring
```

---

## 第二层：工具页

```text
/tools/website-analysis
/tools/dns
/tools/ip
/tools/ssl
/tools/technology
```

---

## 第三层：真实数据页

未来：

```text
/website/[domain]
/entity/[entity]
/technology/[technology]
```

但必须通过 Index Quality Gate。

---

# 36. Index Quality Gate

一个动态页面只有达到条件才能：

```text
INDEXABLE
```

至少检查：

```text
真实数据数量
Evidence 数量
独立 Insight
历史深度
关系信息
页面独特 Summary
```

否则：

```html
<meta name="robots" content="noindex">
```

---

# 37. Search Intelligence SEO 页面

一个网站页面可以成为：

# Live Intelligence Object

示例：

```text
example.com

Search Visibility
82.4

Indexed Pages
12,482

Keywords
8,231

Top 10
312

Organic Trend
+18.2%

Last Observed
2 hours ago
```

下面显示：

```text
Search Changes
Infrastructure Changes
Technology Changes
Key Insights
Evidence
```

不是一篇静态文章。

---

# 38. Internal Linking

建立：

```text
Website
↓
Search Intelligence
↓
Technology
↓
Infrastructure
↓
Related Websites
↓
Insights
```

让不同 Intelligence 页面形成知识网络。

---

# 39. JSON-LD / Metadata

所有可索引页面提供合理的：

- title
- description
- canonical
- OpenGraph
- JSON-LD
- Breadcrumb

动态页面必须根据真实数据生成 Metadata。

---

# 40. Search Console / Webmaster 接入

第一阶段：

```text
Google Search Console
Baidu Webmaster
Bing Webmaster
```

第二阶段：

```text
Yandex
Naver
Shenma
Sogou
```

用户连接自己的网站后，SiteIntel 将搜索引擎数据导入统一 Search Intelligence 模型。

---

# 41. 授权原则

用户连接搜索引擎必须：

```text
OAuth / 官方授权 / 官方 API Key
```

不允许：

- 抓取用户账号密码
- 保存用户密码
- 伪造授权状态
- 混淆第一方数据与公开 SERP 数据

---

# 42. Search Provider Adapter

统一接口：

```ts
interface SearchProvider {
  id: string
  name: string

  connect(): Promise<void>

  getOverview(site): Promise<SearchOverview>

  getQueries(site): Promise<SearchQuery[]>

  getPages(site): Promise<SearchPage[]>

  getIndexStatus(site): Promise<IndexStatus>

  getPerformance(site): Promise<SearchPerformance>
}
```

Provider：

```text
google-search-console
baidu-search
bing-webmaster
yandex-webmaster
naver-search-advisor
shenma
sogou
```

没有实现的 Provider：

```text
Not Connected / Not Available
```

绝不能用假数据填充。

---

# 43. 统一 Search Data Model

所有 Provider 最终归一化为：

```text
SearchOverview
SearchQuery
SearchPage
SearchPerformance
IndexObservation
SearchEvent
```

不同 Provider 缺失的字段：

```text
null / unavailable
```

不允许伪造。

---

# 44. Multi-Engine Dashboard

统一：

```text
Search Intelligence

Google
Baidu
Bing
Yandex
Naver
Shenma
Sogou
```

每个引擎：

```text
Clicks
Impressions
CTR
Position
Keywords
Indexed Pages
Visibility
Trend
```

---

# 45. Cross-Engine Divergence

新增一个非常有价值的分析：

# Search Engine Divergence

例如：

```text
Google
Visibility +18%

Baidu
Visibility -7%

Bing
Visibility +4%
```

SiteIntel 自动分析：

> 不同搜索引擎表现出现明显分化。

然后继续检查：

- Content
- Indexability
- Technical Signals
- Regional differences
- Ranking distribution
- Crawl signals

---

# 46. Ranking Intelligence

排名变化：

```text
NEW
RISING
STABLE
FALLING
LOST
```

支持：

```text
Keyword
URL
Search Engine
Country
Device
Time
```

---

# 47. Traffic Intelligence

显示：

```text
Clicks
Impressions
CTR
Traffic
Change Rate
```

并支持：

```text
Page
Keyword
Search Engine
Country
Device
```

---

# 48. Search Alerts

监控用户可以选择：

```text
☑ Index Drop
☑ Traffic Drop
☑ Keyword Drop
☑ Ranking Drop
☑ CTR Drop
☑ Search Visibility Drop
☑ Search Recovery
```

例如：

```text
HIGH PRIORITY

Google Search Traffic
-28%

Indexed Pages
-11%

Top 10 Keywords
-16%

Detected 2 technical changes
```

---

# 49. Search Correlation Report

最终报告回答：

```text
发生了什么？

什么时候发生？

哪些搜索引擎发生？

哪些页面受影响？

哪些关键词受影响？

是否伴随基础设施变化？

是否伴随技术变化？

是否伴随技术 SEO 变化？

证据是什么？

置信度是多少？
```

---

# 50. SiteIntel 最终产品结构

```text
SITEINTEL

ANALYZE
├── Website
├── Infrastructure
├── Technology
├── Security
└── Search

DISCOVER
├── Related Websites
├── Entities
└── Relationships

UNDERSTAND
├── Insights
├── Events
├── History
└── Search Changes

MONITOR
├── Website
├── Infrastructure
├── Search
└── Alerts

INVESTIGATE
├── Timeline
├── Compare
├── Evidence
└── AI Investigation
```

---

# 51. 开发优先级

## Phase 1 — Search Foundation

优先级：★★★★★

只做：

```text
Search Provider Adapter
Google Search Console
Baidu
Bing
Search Data Normalization
Search Snapshot
Search Event
Search Report
```

目标：

> 让一个站长连接自己的 Google / 百度 / Bing 数据，并在 SiteIntel 一个页面看懂。

---

## Phase 2 — Search Intelligence

优先级：★★★★★

实现：

```text
Search Visibility
Index Intelligence
Keyword Intelligence
Page Intelligence
Traffic Intelligence
Search Health
```

目标：

> 不只是展示数据，而是告诉用户变化。

---

## Phase 3 — Search Event Correlation

优先级：★★★★★

实现：

```text
Infrastructure Event
+
Technology Event
+
Search Event
+
Index Event
+
Ranking Event
```

形成：

```text
Search Insight
```

目标：

> **解释为什么搜索表现发生变化。**

---

## Phase 4 — SEO Growth

优先级：★★★★★

实现：

```text
Product SEO
Tool SEO
Intelligence Pages
Index Quality Gate
Canonical
Sitemap
JSON-LD
Internal Linking
```

目标：

> SiteIntel 自己产生自然搜索流量。

---

## Phase 5 — Search Engine Expansion

优先级：★★★★☆

加入：

```text
Yandex
Naver
Shenma
Sogou
```

---

## Phase 6 — Data Growth

优先级：★★★★☆

实现：

```text
Seed Import
Candidate Discovery
Relationship Expansion
Observation Priority
```

目标：

> 建立自己的互联网观察池。

---

## Phase 7 — Similarity

优先级：★★★☆☆

先做：

```text
On-demand Similarity
```

达到数据规模后再：

```text
Vector
ANN
Worker
Batch
```

---

## Phase 8 — Cluster / Advanced Intelligence

优先级：★★★☆☆

只有数据量和历史数据足够后再开发：

```text
Cluster
Genome
Web Pulse
Global Intelligence Feed
```

---

# 52. AI Investigation

最后增加自然语言调查。

用户：

> 最近这个网站 Google 流量为什么下降？

处理：

```text
User Question
↓
Intent Parser
↓
Strict Schema
↓
Function Calling
↓
Search Data
↓
Infrastructure Data
↓
Technical Data
↓
Events
↓
Evidence
↓
AI Explanation
```

AI 只能解释真实数据。

---

# 53. AI 安全原则

AI：

可以：

- 解析意图
- 选择查询函数
- 关联 Evidence
- 解释变化
- 提出 Investigation Direction

不能：

- 创建不存在的事实
- 修改数据库事实
- 伪造排名
- 伪造流量
- 伪造收录
- 把相关性说成因果关系

---

# 54. 最终产品飞轮

```text
Search Users
      ↓
Connect Search Engines
      ↓
More Search Data
      ↓
More Events
      ↓
More Insights
      ↓
Better Intelligence
      ↓
Better SEO Pages
      ↓
More Organic Traffic
      ↓
More Users
      ↓
More Connected Sites
```

同时：

```text
Website Investigation
      ↓
Infrastructure Data
      ↓
Search Data
      ↓
Correlation
      ↓
Better Diagnosis
```

最终形成 SiteIntel 的核心壁垒：

# Search + Website + Infrastructure Intelligence

---

# 55. 最终一句话

SiteIntel X 不做：

> “又一个 SEO 工具。”

而做：

> **一个把 Google、百度、Bing 等搜索表现，与网站基础设施、技术栈、历史变化和站长数据连接起来的 Website Intelligence Platform。**

核心不是：

> 你的排名是多少？

而是：

> **你的排名、收录、流量发生了什么变化，以及 SiteIntel 能否用网站数据、技术数据、基础设施数据和历史证据解释这种变化。**

---

# 56. 第一阶段验收标准

第一阶段完成后，用户能够：

1. 添加网站。
2. 连接 Google Search Console。
3. 连接百度。
4. 连接 Bing。
5. 查看三个搜索引擎的数据。
6. 查看收录量。
7. 查看关键词。
8. 查看 Top 3 / 10 / 20 / 50。
9. 查看点击。
10. 查看曝光。
11. 查看 CTR。
12. 查看平均排名。
13. 查看页面表现。
14. 查看历史趋势。
15. 查看变化事件。
16. 将 Search Event 与现有 Infrastructure / Technology Event 对齐。
17. 查看 Evidence。
18. 查看 Confidence。
19. 获取 Search Insight。

最终用户真正得到的是：

```text
WHAT CHANGED?
        ↓
WHERE?
        ↓
WHEN?
        ↓
HOW MUCH?
        ↓
WHY?
        ↓
EVIDENCE?
        ↓
CONFIDENCE?
```

这才是 SiteIntel 的 Search Intelligence 核心。
