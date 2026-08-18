# SiteIntel X — Search Intelligence 正式实施方案

> 项目：siteintel.cc  
> 产品：SiteIntel X  
> 核心定位：**Website Data Intelligence + Search Intelligence**  
> 目标用户：**中国站长、独立站站长、跨境电商、外贸站、英文站、SaaS、出海网站运营者**  
> 本阶段核心搜索引擎：**百度 → Bing → Google**  
> 核心产品目标：把“收录、关键词、排名、点击、曝光、流量、页面表现”与 SiteIntel 已有的“网站、IP、DNS、ASN、SSL、HTTP、Technology、Events、Evidence、Insights”连接起来，形成真正的 **Search Intelligence**。

---

# 1. 这不是重做 SiteIntel

现有 SiteIntel 已经完成 MVP、第二阶段和第三阶段，现有系统已经拥有：

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

当前完成情况中，核心数据链路已经实现：

```text
Collect
↓
Raw
↓
Entities
↓
Relationships
↓
Snapshots
↓
Events
↓
Insights
↓
Report
```

因此本项目采取：

> **增量演化，不推翻重建。**

现有系统继续作为 Website Intelligence 的基础层，Search Intelligence 作为新的 Observation / Event / Insight 数据来源加入。

---

# 2. Search Intelligence 的核心重新定义

不要把新功能定位为：

- SEO Checker
- SEO Score
- Search Engine Checker
- Keyword Tracker

而是：

# Search Intelligence

真正回答：

```text
网站在搜索引擎中的表现怎么样？

收录量发生了什么变化？

哪些关键词在上涨？

哪些关键词在下降？

哪些页面获得了流量？

哪些页面正在掉量？

Google / Bing / 百度之间有什么差异？

搜索表现变化之前发生了什么？

这种变化是否与网站基础设施、技术、内容或搜索配置变化有关？

有哪些证据？

置信度是多少？
```

---

# 3. 产品核心模型

最终：

```text
Website
│
├── Website Intelligence
│   ├── Infrastructure
│   ├── Technology
│   ├── Security
│   └── Relationships
│
└── Search Intelligence
    ├── Search Performance
    ├── Search Visibility
    ├── Index Intelligence
    ├── Keyword Intelligence
    ├── Page Intelligence
    ├── Search Events
    └── Search Insights
```

然后：

```text
Website Events
+
Search Events
+
Evidence
+
History
        ↓
Correlation
        ↓
Insight
```

---

# 4. 搜索引擎优先级

## Phase 1

# Baidu

原因：

- 用户群体是中国站长
- 中文网站和中国市场站长首先需要百度
- 百度搜索表现对目标用户有直接价值

但是必须严格区分：

### Baidu Official

官方可编程能力与公开站长工具能力不是同一概念。

SiteIntel 不得因为百度站长后台网页端存在某项数据，就宣称“官方 API 可以直接获取该数据”。

百度官方页面目前可以确认其站长工具包含：

- 索引量
- 抓取异常
- 流量与关键词
- 链接提交

其中“流量与关键词”网页工具展示热门关键词来源、展现、点击等信息；链接提交用于主动向百度推送 URL，但官方明确说明提交并不保证收录。citeturn889220search4turn889220search2

因此 SiteIntel 对百度拆成两个 Provider：

```text
Baidu Official
├── Index / available official data
├── Submission
├── Crawl / diagnostics where programmatically available
└── Site management

Baidu SERP Intelligence
├── Keyword ranking
├── Ranking changes
├── SERP presence
├── Target URL
└── Search result observations
```

### 严禁

```text
把网页端“流量与关键词”
直接描述为
“百度官方 API 返回的关键词/点击/CTR 数据”
```

---

## Phase 2

# Bing

Bing Webmaster API 官方支持程序化访问网站在 Bing 搜索和索引中的信息，包括 Rank & Traffic Stats、Keyword Details、Crawl Stats 等，并支持 URL / Sitemap 等提交。citeturn889220search0turn889220search1

Bing Provider 可作为第二个完整 Search Provider。

---

## Phase 3

# Google

Google Search Console 作为跨境、英文站、SaaS、独立站等场景的扩展。

Google 不作为第一阶段依赖，避免 OAuth / Token / Sync / Quota 过早成为项目最大工程。

---

## Phase 4

扩展：

- Google SERP
- Bing SERP
- Baidu SERP

---

## Phase 5

根据真实用户与数据价值决定是否增加：

- Yandex
- Naver
- Sogou
- Shenma
- 其他区域搜索引擎

原则：

> 数据质量 > 用户价值 > API 稳定性 > 市场覆盖 > 搜索引擎数量

---

# 5. 第一阶段必须做到什么

第一阶段不是“做完百度 + Bing + Google”。

只做：

# Baidu Search Intelligence

但百度分两块：

```text
Official Data
+
SERP Observation
```

最终用户看到统一的：

```text
Index
Keywords
Ranking
Pages
Search Visibility
Changes
Insights
```

但每一项必须显示真实来源。

例如：

```text
Source: Baidu Official
Source: SERP Observation
```

绝不能混淆。

---

# 6. Search Intelligence 六大模块

## 6.1 Search Performance

关注：

- Clicks
- Impressions
- CTR
- Average Position
- Organic Traffic

但只有实际 Provider 能提供的数据才展示。

---

## 6.2 Search Visibility

关注：

- Keywords
- Top 3
- Top 10
- Top 20
- Top 50
- Top 100
- New Keywords
- Rising Keywords
- Falling Keywords
- Lost Keywords

SiteIntel 可定义：

# Search Visibility

但必须有明确公式。

第一阶段禁止使用无法解释的：

> 权重 82

---

## 6.3 Index Intelligence

关注：

- Indexed Pages
- Index Growth
- Index Loss
- New Indexed Pages
- Deindexed Pages
- Index Trend

这是国内站长最直接的 SEO 经营指标之一。

---

## 6.4 Keyword Intelligence

字段：

```text
keyword
search_engine
position
previous_position
change
clicks
impressions
ctr
target_url
observed_at
```

数据来源明确：

```text
Official Search Data
或
SERP Observation
```

---

## 6.5 Page Intelligence

字段：

```text
page_url
clicks
impressions
ctr
position
index_status
traffic_change
ranking_change
observed_at
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

## 6.6 Search Events

沿用现有 `events` 模型。

不要创建平行的 `search_events` 表，除非现有 events 无法表达。

新增 event category：

```text
search
index
ranking
traffic
```

事件示例：

```text
INDEX_GROWTH
INDEX_DROP
RANKING_GAIN
RANKING_DROP
KEYWORD_GAIN
KEYWORD_LOSS
TRAFFIC_SPIKE
TRAFFIC_DROP
CTR_CHANGE
SEARCH_VISIBILITY_CHANGE
```

---

# 7. Search Intelligence 不只是展示数字

最重要的业务链：

```text
What happened?
↓
How much?
↓
Where?
↓
When?
↓
Why might it have happened?
↓
What evidence supports it?
↓
How confident are we?
```

例如：

```text
Organic Traffic
↓ 23%

Index
↓ 8%

Top 10 Keywords
↓ 13%

Robots
Changed

Canonical
Changed
```

最终：

> 搜索流量下降与索引覆盖下降、关键词减少同时发生；此前检测到搜索配置变化，存在潜在关联。

不得自动宣称确定因果。

---

# 8. Search Event Correlation

这是 SiteIntel Search Intelligence 的核心差异化。

把现有：

```text
IP_CHANGED
ASN_CHANGED
SSL_CHANGED
NAMESERVER_CHANGED
TECHNOLOGY_CHANGED
INFRASTRUCTURE_MIGRATION
```

与新的：

```text
INDEX_DROP
RANKING_DROP
KEYWORD_LOSS
TRAFFIC_DROP
```

放进统一 Timeline。

例如：

```text
08-10
IP Changed

08-11
Technology Changed

08-12
Indexed Pages -12%

08-13
Top 10 Keywords -8%

08-14
Organic Clicks -17%
```

生成：

```text
Search Correlation Insight
```

字段：

```text
type
status
confidence
supporting_evidence
contradicting_evidence
first_detected_at
last_detected_at
```

继续复用现有 `insights` 表。

---

# 9. 不新增 Hypothesis Engine

现阶段不创建：

```text
signals
patterns
hypotheses
intelligences
```

四套新的平行表。

现有：

```text
events
evidence
insights
confidence
```

已经可以表达绝大部分需要。

因此：

```text
Event
↓
Evidence
↓
Insight
```

即可。

Insight 增加：

```text
status
supporting_evidence
contradicting_evidence
recommended_next_step
```

状态：

```text
observed
probable
inferred
unconfirmed
unknown
```

---

# 10. Search Health 第一阶段取消

不要制造：

```text
Search Health = 91
```

因为这会与“反对无依据权重分”的原则冲突。

第一阶段直接展示真实指标：

```text
Clicks
+18.4%

Impressions
+11.2%

CTR
+5.7%

Average Position
+2.1

Indexed Pages
+8.2%
```

以后如果真的需要综合分：

# Search Visibility

并公开计算公式。

---

# 11. 百度第一阶段数据结构

必须明确区分：

## Baidu Official

例如：

```text
Index
Submission
Crawl
Diagnostic
```

官方能力之外的字段一律：

```text
Unavailable from current official integration
```

---

## Baidu SERP

用于：

```text
Keyword
Position
Target URL
SERP Feature
Ranking Change
```

这是 SiteIntel 自己的搜索观察能力。

---

# 12. 数据库设计

第一阶段不建立数据仓库。

只增加最少必要的数据模型。

## 12.1 search_connections

```text
id
user_id
provider
account_identifier
access_token_encrypted
refresh_token_encrypted
token_expires_at
status
created_at
updated_at
```

Baidu / Bing / Google 的授权方式不同，允许 Provider 使用不同连接字段。

---

## 12.2 search_properties

```text
id
connection_id
provider_property_id
site_url
verification_status
status
last_sync_at
next_sync_at
created_at
updated_at
```

---

## 12.3 search_daily_metrics

网站级指标：

```text
id
property_id
date
clicks
impressions
ctr
position
```

适用于 Provider 能提供对应指标的情况。

---

## 12.4 search_query_daily

```text
id
property_id
date
query
clicks
impressions
ctr
position
source
```

其中：

```text
source =
official
serp
```

---

## 12.5 search_page_daily

```text
id
property_id
date
page_url
clicks
impressions
ctr
position
index_status
source
```

---

# 13. 不把每日关键词和页面数据塞进 Snapshot JSONB

Snapshot 继续用于：

> 网站整体状态快照。

Search Query / Search Page 是大量行数据。

因此：

```text
search_query_daily
search_page_daily
```

使用关系型表。

优点：

- 趋势查询快
- Top keyword 排序容易
- 排名变化查询容易
- 页面分页容易
- 数据去重容易
- 未来做统计更容易

---

# 14. Search Provider Adapter

统一接口：

```ts
interface SearchProvider {
  id: string
  name: string

  connect(): Promise<void>

  listProperties(): Promise<SearchProperty[]>

  getOverview(
    property: SearchProperty,
    range: DateRange
  ): Promise<SearchOverview>

  getQueries(
    property: SearchProperty,
    range: DateRange
  ): Promise<SearchQuery[]>

  getPages(
    property: SearchProperty,
    range: DateRange
  ): Promise<SearchPage[]>

  getIndexStatus?(
    property: SearchProperty
  ): Promise<IndexStatus>

  submitUrl?(
    property: SearchProperty,
    url: string
  ): Promise<SubmissionResult>
}
```

Provider：

```text
baidu-official
baidu-serp

bing-webmaster

google-search-console
google-serp
```

未来继续扩展。

---

# 15. 数据归一化

所有 Provider 最终转换到统一 SiteIntel 数据模型。

例如：

```text
provider_clicks
→ clicks

provider_impressions
→ impressions

provider_ctr
→ ctr

provider_position
→ position
```

但如果某 Provider 没有某字段：

```text
null
```

不伪造。

---

# 16. Search Connection System

搜索引擎接入不能只是 Provider。

必须建立：

# Search Connection System

流程：

```text
Connect
↓
Authenticate
↓
Verify Property
↓
Select Property
↓
Initial Sync
↓
Store
↓
Daily Sync
↓
Calculate Events
↓
Generate Insights
```

---

# 17. Google OAuth

Google 放到 Phase 3。

必须单独设计：

```text
OAuth
Refresh Token
Token Encryption
Property Selection
Sync Job
Quota / Load Control
Retry
Backoff
Cache
```

不要把 Google OAuth 当成普通 API Key。

---

# 18. Bing 接入

Bing Webmaster API 官方支持：

- Rank & Traffic Stats
- Keyword Details
- Crawl Stats
- Link Details
- URL / Sitemap 提交

因此 Bing 可作为第二阶段较完整的 Search Provider。citeturn889220search0turn889220search3

---

# 19. Search Sync Scheduler

第一阶段保持现有 SiteIntel 单实例架构。

暂时不引入 Redis。

增加：

```text
search_sync_jobs
```

状态：

```text
queued
running
completed
failed
retry
quota_blocked
```

每个 Property：

```text
last_sync_at
next_sync_at
priority
```

优先级：

```text
HIGH
NORMAL
LOW
```

---

# 20. 数据同步策略

不要每次打开 Dashboard 调 API。

流程：

```text
Provider
↓
Scheduled Sync
↓
Local DB
↓
Dashboard
```

优点：

- 减少 API 调用
- 防止配额问题
- 页面更快
- 历史数据可查询
- 可以做自己的趋势分析

---

# 21. Search Data Freshness

所有 Search Data 标记：

```text
observed_at
synced_at
source
```

UI 显示：

```text
Last synced
2 hours ago
```

不要伪装实时数据。

---

# 22. 数据增长策略

Search Intelligence 不是靠大量用户人工输入才能成长。

数据来源：

```text
User-owned connected sites
+
User investigations
+
Seed sites
+
Related website candidates
```

第一阶段目标：

```text
100
↓
1,000
↓
10,000
```

重点是：

> **高质量观察对象。**

不是追求数量本身。

---

# 23. Candidate Discovery

分析一个网站以后：

```text
IP
Certificate
Nameserver
Technology
Tracking ID
Organization
```

发现：

```text
Candidate Websites
```

这些网站先成为：

```text
candidate
```

再根据：

```text
priority
relationship strength
search demand
```

决定是否进一步采集。

---

# 24. Similarity 暂缓

当前数据规模不足时：

# On-demand Similarity

只在用户查询时计算。

不要一开始做全库：

```text
N × N
```

比较。

---

# 25. Cluster 暂缓

只有观察池达到足够规模后才考虑：

```text
Feature Vector
Candidate Retrieval
Approximate Nearest Neighbor
Worker
Batch Processing
```

当前不做。

---

# 26. SEO Growth

SiteIntel 自身 SEO 不做内容农场。

第一阶段：

```text
/website-intelligence
/search-intelligence
/infrastructure-intelligence
/technology-intelligence
/website-monitoring
```

第二阶段：

```text
/tools/website-analysis
/tools/dns
/tools/ip
/tools/ssl
/tools/technology
```

第三阶段才考虑真实数据：

```text
/website/[domain]
/entity/[entity]
/technology/[technology]
```

---

# 27. Intelligence Page Index Gate

动态页面只有满足质量阈值才允许：

```text
INDEXABLE
```

必须检查：

```text
真实数据覆盖
+
Evidence 数量
+
独立 Insight
+
历史信息
+
关系信息
+
独立 Summary
```

否则：

```text
NOINDEX
```

---

# 28. SEO 的真正产品飞轮

```text
Search
↓
Website Intelligence Page
↓
用户分析网站
↓
产生真实数据
↓
Events / Insights
↓
页面获得更多内容
↓
更多搜索流量
↓
更多用户
↓
更多网站观察
```

SEO 数据必须来自真实产品行为。

---

# 29. Search Dashboard

首页采用站长真正关心的指标：

```text
SEARCH INTELLIGENCE

                    Baidu      Bing      Google

Clicks              —         —         —
Impressions         —         —         —
CTR                 —         —         —
Avg Position        —         —         —
Keywords            —         —         —
Top 10              —         —         —
Indexed Pages       —         —         —
Visibility          —         —         —
```

未连接：

```text
Not Connected
```

不允许假数据。

---

# 30. Baidu Dashboard

第一阶段：

```text
Baidu Search Intelligence

Index
Indexed Pages
Index Change

Crawl
Crawl Status
Crawl Changes

Submission
Submitted URLs
Submission History

SERP Observation
Keywords
Positions
Ranking Changes
Target URLs
```

明确区分：

```text
Official
SERP Observation
```

---

# 31. Bing Dashboard

第二阶段：

```text
Bing Search Intelligence

Traffic
Clicks
Impressions
CTR
Average Position

Keywords
Top 3
Top 10
Top 20
Top 50

Pages
Rising
Declining

Crawl
Crawl Changes
```

---

# 32. Google Dashboard

第三阶段：

```text
Google Search Intelligence

Clicks
Impressions
CTR
Average Position

Queries
Pages
Countries
Devices

Index
Search Appearance
```

Google Search Console API 的 Search Analytics 能力包括 clicks、impressions、CTR、position 等字段，适合作为第三阶段的完整搜索表现 Provider。 

---

# 33. Cross-Engine Intelligence

当至少有两个搜索引擎数据后，加入：

```text
Cross-Engine Comparison
```

例如：

```text
Keyword:
website analytics

Baidu
#23

Bing
#8
```

显示：

```text
Search Divergence
```

以及：

```text
Best Engine
Worst Engine
Difference
Trend
```

---

# 34. Search Events

统一事件：

```text
INDEX_GROWTH
INDEX_DROP

KEYWORD_GAIN
KEYWORD_LOSS

RANKING_GAIN
RANKING_DROP

TRAFFIC_SPIKE
TRAFFIC_DROP

CTR_CHANGE

VISIBILITY_CHANGE
```

现有 Infrastructure / Technology Event 与 Search Event 进入统一 Timeline。

---

# 35. Search Alerts

监控可以选择：

```text
☑ Index Drop
☑ Traffic Drop
☑ Keyword Loss
☑ Ranking Drop
☑ CTR Drop
☑ Search Visibility Drop
```

示例：

```text
HIGH PRIORITY

Baidu Indexed Pages
-18%

Top 10 Keywords
-13%

Traffic
-24%

Detected:
Technology Change
36 hours earlier
```

---

# 36. Search Correlation Insight

示例：

```text
Observed:

Baidu index decreased 18%
↓
Top 10 keywords decreased 13%
↓
Traffic decreased 24%

Supporting evidence:

Technology changed
Infrastructure remained stable
Robots unchanged

Status:
Probable

Confidence:
84
```

结果必须是：

> “存在较强关联”

而不是：

> “系统确定技术变更导致流量下降”。

---

# 37. AI Investigation

后期支持：

> “为什么这周百度流量下降了？”

流程：

```text
User Question
↓
Intent Parsing
↓
Strict JSON Schema
↓
Function Calling
↓
Search Data
↓
Website Data
↓
Events
↓
Evidence
↓
AI Explanation
```

AI 只能解释真实数据。

不得自行生成：

- 排名
- 流量
- 收录
- 搜索量
- 因果关系

---

# 38. 第一阶段完成标准

第一阶段不是“完成所有搜索引擎”。

只要求：

## Baidu

### Official

- 可连接站点
- 可获得实际可编程的官方数据
- 收录 / 提交 / 抓取类能力明确
- 数据来源明确

### SERP

- 关键词排名监测
- 目标 URL
- 排名变化
- 搜索结果观察

### SiteIntel

- Search Snapshot
- Search Event
- Search Insight
- Evidence
- Confidence
- Dashboard
- Timeline
- Alerts

---

# 39. 第二阶段完成标准

Bing：

- Provider
- Site connection
- Ranking / Traffic
- Keywords
- Pages
- Crawl
- Historical sync
- Search Events
- Search Insights

并复用：

```text
Search Connection
Search Provider
Search Snapshot
Search Event
Search Insight
```

---

# 40. 第三阶段完成标准

Google：

- OAuth
- Token storage
- Property selection
- Sync scheduler
- Quota/load-aware sync
- Queries
- Pages
- Clicks
- Impressions
- CTR
- Position
- Historical data
- Search insights

---

# 41. 重要工程约束

1. 不伪造搜索数据。
2. 官方数据与 SERP 数据必须明确区分。
3. 不因网页端存在某项数据就声称官方 API 支持。
4. 不把搜索数据全部塞进 Snapshot JSONB。
5. 不创建大量平行概念和重复表。
6. 不为了覆盖搜索引擎数量而牺牲完成度。
7. 不先做 Google OAuth 再验证产品需求。
8. 不先做全库 Similarity / Cluster。
9. 不使用无依据的 Search Health 综合分。
10. 所有 Search Insight 必须带 Evidence。
11. 所有推断必须带 Confidence。
12. 所有因果性判断必须使用谨慎语言。
13. Search API 数据必须缓存到本地数据库。
14. Dashboard 不直接依赖实时第三方 API。
15. Provider 失败允许 partial success。
16. 每个 Provider 都必须可单独禁用。
17. 用户未连接某搜索引擎时显示 Not Connected。
18. 不得把用户授权数据与公开 SERP 数据混淆。
19. 不为 SEO 批量生成低价值页面。
20. Indexable 页面必须通过 Quality Gate。

---

# 42. 最终产品架构

```text
                         SITEINTEL X
                              │
             ┌────────────────┴────────────────┐
             ↓                                 ↓
      WEBSITE INTELLIGENCE              SEARCH INTELLIGENCE
             │                                 │
      ┌──────┼──────┐                    ┌─────┼─────┐
      ↓      ↓      ↓                    ↓     ↓     ↓
     IP     DNS    Tech                Baidu  Bing  Google
     ASN    SSL    HTTP                  │      │      │
      │      │      │                    └──────┼──────┘
      └──────┼──────┘                           ↓
             ↓                             Search Data
          Events                               ↓
             ↓                           Search Events
          Evidence                              ↓
             ↓                         Search Correlation
          Insights                              ↓
             └──────────────┬───────────────────┘
                            ↓
                       INTELLIGENCE
                            ↓
                     INVESTIGATION
                            ↓
                       MONITOR
                            ↓
                          API
```

---

# 43. 最终产品价值

SiteIntel 不应该告诉站长：

> “你的 SEO Score 是 82。”

而应该告诉：

> **百度过去 7 天收录下降 11%，Top 10 关键词下降 8%，自然搜索流量下降 17%。在变化开始前 36 小时检测到网站技术发生变化。该关联目前置信度为 84%。**

然后用户可以继续点击：

```text
View Timeline
View Evidence
View Pages
View Keywords
View Infrastructure
```

这才是：

# Search Intelligence

---

# 44. 最终路线

```text
NOW
│
├── Phase 1
│   Baidu Search Intelligence
│   Official + SERP
│
├── Phase 2
│   Bing Search Intelligence
│
├── Phase 3
│   Google Search Intelligence
│
├── Phase 4
│   Cross-Engine Intelligence
│   Baidu/Bing/Google SERP
│
├── Phase 5
│   Observation Growth
│   Similarity
│
└── Phase 6
    Cluster / Advanced AI Investigation
```

---

# 45. 最重要的最终判断

SiteIntel 的真正壁垒不是：

> “我们支持多少搜索引擎。”

而是：

> **我们能否把搜索表现的变化与网站本身的变化联系起来，并用真实证据解释。**

因此最终核心能力是：

```text
Search Data
+
Website Data
+
Infrastructure Data
+
Technology Data
+
History
+
Events
+
Evidence
↓
Search Intelligence
```

这就是 SiteIntel X 最合理、最现实、也最适合中国站长用户的产品路线。
