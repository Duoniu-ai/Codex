# SiteIntel V2 产品与技术架构总纲

> 项目：siteintel.cc  
> 定位：Website & Internet Intelligence Platform  
> 版本：V2 Master Plan  
> 日期：2026-08-15  
> 用途：长期产品、技术、数据、SEO、商业化升级依据  
> 执行对象：Claude Code + DeepSeek V4 Pro

---

# 1. 最终产品定位

SiteIntel 不应只是“网站扫描器”“网站技术检测工具”或“AI SEO 工具”。

最终定位：

> **以域名为入口，以 Evidence 为基础，以 Entity Graph 为骨架，以 History 为时间轴，以 Intelligence 为核心，并进一步为网站所有者提供 Strategy 与 Roadmap 的网站与互联网情报平台。**

核心理念：

```text
DATA
↓
INFORMATION
↓
RELATION
↓
INTELLIGENCE
↓
STRATEGY
↓
ACTION
```

产品最终回答两类问题：

### Public Intelligence

> 这个网站是什么？它在哪里运行？使用什么技术？和谁有关？发生过什么？

### Owner Intelligence

> 这是我的网站。它现在怎么样？有什么问题？应该往哪里发展？下一步应该做什么？

---

# 2. 当前 V1 基础

现有 SiteIntel 已经形成 V1 雏形，包括：

- Domain Investigation
- Analysis Job
- Website Report
- Infrastructure
- DNS
- IP / ASN
- CDN
- TLS
- RDAP
- HTTP / Security Headers
- Technology
- History / Snapshot
- Relationship Graph
- Related Websites
- Raw Evidence
- Technology Intelligence
- Technology Tool
- 登录 / 注册
- Monitor / Dashboard / API Key 等方向

现有后端与 PostgreSQL 不应被轻易推翻。

V2 的原则：

> **在真实代码、数据库和采集能力之上渐进式升级，而不是重新开发一个新项目。**

---

# 3. V2 核心架构

统一为：

```text
Collectors
↓
Raw Evidence
↓
Canonical Facts
↓
Entities
↓
Relationships
↓
Snapshots
↓
Signals
↓
Events
↓
Insights
↓
Intelligence Report
↓
Owner Audit
↓
Strategy
↓
Roadmap
```

核心数据层必须统一，不能让每个页面各自解释数据。

---

# 4. Evidence Layer

这是最高优先级。

所有事实必须能够追溯到证据：

```text
Insight
↓
Signal
↓
Fact
↓
Evidence
↓
Collector
↓
Timestamp
```

建议 Evidence 至少包含：

```text
evidence_id
source
collector
observed_at
raw_value
normalized_value
entity_id
fact_type
confidence
expires_at
```

禁止：

- 不同模块各自解释 DNS / IP / TLS
- 没有 Evidence 的确定性结论
- 把推测写成事实

必须建立：

## Contradiction Detector

自动发现：

```text
Module A: MX = none
Module B: MX = Google
```

并阻止矛盾数据继续进入报告。

---

# 5. Canonical Facts

建立统一事实层：

```text
Raw Evidence
↓
Parser
↓
Normalizer
↓
Canonical Fact
```

所有：

- Report
- Graph
- Search
- Insight
- Audit
- Strategy

都读取 Canonical Facts。

这样可以避免“DNS Records 有 MX，但 Email 卡片说没有 MX”之类的数据冲突。

---

# 6. Entity Model

SiteIntel 不应该把 IP、ASN、Technology 当作报告里的普通字段。

它们必须成为一等实体。

优先实体：

```text
Website / Domain
IP
ASN
Organization
Technology
Certificate
Provider
Nameserver
Mail Service
Tracking ID
Subdomain
```

建议：

```text
Entity
├── id
├── type
├── canonical_value
├── display_name
├── metadata
├── first_seen
├── last_seen
└── confidence
```

---

# 7. Entity Detail Pages

结果页面不是终点。

重要实体应该可以继续深入。

例如：

```text
/website/{domain}
/ip/{ip}
/asn/{asn}
/organization/{id}
/technology/{technology}
/certificate/{fingerprint}
```

不是要求第一阶段全部实现。

优先：

1. Website
2. IP
3. ASN
4. Technology
5. Certificate
6. Organization

---

# 8. IP Intelligence

点击网站报告中的 IP，不应该只显示一个 IP 卡片。

应该进入：

```text
IP Intelligence

IP
ASN
Organization
ISP / Provider
Country
Region
Datacenter
Reverse DNS
First Seen
Last Seen

Hosted Websites
Related IPs
Related ASNs
Certificates
Technologies
Historical Changes
Evidence
```

尤其是：

> Hosted Websites

回答：

> 这个 IP 上观察到多少网站？哪些网站？关系强度是什么？证据是什么？

必须避免把“共享 IP”直接解释为“属于同一主体”。

---

# 9. ASN Intelligence

ASN 页面应逐步支持：

- Organization
- Country
- Network / Prefix
- Observed IPs
- Observed Domains
- Related Providers
- Technologies
- Historical Changes
- Evidence

最终用户可以：

```text
Website
↓
IP
↓
ASN
↓
Organization
↓
Other Domains
```

持续探索。

---

# 10. Technology Intelligence

当前技术识别是重要短板。

宣传能力不能明显超过实际检测能力。

建议检测：

- CMS
- Frontend
- Framework
- Backend
- Server
- CDN
- Analytics
- Tracking
- Payment
- Security
- Fonts
- Hosting
- Third-party services

数据输出：

```text
technology
category
version
evidence
confidence
first_seen
last_seen
```

---

# 11. Technology Intelligence 与 Technology Tool

## /technology-intelligence

定位：

> Technology Intelligence 产品能力页

强调：

- Technology Stack
- Technology History
- Technology Changes
- Technology Relationships
- Technology Signals
- Technology Insights

## /tools/technology

定位：

> 免费 SEO 工具 / Technology Stack Checker

强调：

- 输入域名
- 快速检测
- 免费
- 可索引
- 真实结果
- 引导完整 Website Intelligence

漏斗：

```text
Search
↓
/tools/technology
↓
Free Check
↓
Full Intelligence
↓
/website/{domain}
↓
Register
↓
Monitor / API
```

---

# 12. Relationship Model

关系也必须成为一等数据。

建议：

```text
relationship_id
source_entity
target_entity
relationship_type
strength
confidence
evidence_ids
first_seen
last_seen
```

关系类型：

- resolves_to
- hosted_by
- belongs_to
- uses
- protected_by
- issued_to
- served_by
- shared_with
- related_to
- uses_technology

---

# 13. Relationship Intelligence

不能只显示：

> Cloudflare · 35

必须解释：

```text
Relationship Score
Evidence
Relationship Type
Confidence
Observed Time
```

例如：

```text
example.com

Score: 35 / 100

Evidence:
✓ Same CDN
✓ Same Provider

Not observed:
× Same IP
× Same Certificate
× Same Tracking ID

Assessment:
Weak infrastructure relationship
```

建议可解释评分示例：

```text
Shared IP             95
Shared SSL            90
Shared Tracking ID    85
Same ASN              55
Same CDN              20
Same DNS Provider     15
```

最终公式必须通过真实数据验证。

---

# 14. Graph Intelligence

Relationship Graph 是 SiteIntel 最有潜力的差异化能力之一。

图谱应支持：

- 节点详情
- 关系详情
- Evidence
- Confidence
- First Seen
- Last Seen
- 时间变化
- 节点类型筛选
- Relationship Type 筛选
- 路径探索

例如：

```text
Website
↓
IP
↓
ASN
↓
Organization
↓
Other Domains
↓
Technology
↓
Certificate
```

---

# 15. Investigation Trail

用户每次探索都可以形成调查路径：

```text
openai.com
↓
IP
↓
ASN
↓
Provider
↓
Related Domains
↓
Certificate
```

未来可以保存：

```text
Investigation
Domains discovered
Entities discovered
Relationships discovered
Changes
Insights
```

让用户可以继续调查，而不是每次重新开始。

---

# 16. Website Report V2

核心页面：

```text
/website/{domain}
```

不要只是数据库信息展示。

建议顺序：

```text
1. Executive Summary
2. Key Insights
3. Signals / Events
4. Infrastructure
5. Technology
6. Relationships
7. Related Entities
8. Historical Changes
9. Search Intelligence
10. Evidence
```

原则：

> **先结论，后数据。**

---

# 17. Executive Summary

顶部应显示：

- Online Status
- Infrastructure
- Technology Count
- Related Entities
- Detected Changes
- Insights
- Confidence
- Last Analyzed

然后：

## Executive Summary

告诉用户：

- 网站主要基础设施
- 关键技术
- 主要关系
- 最近变化
- 重要异常
- 置信度

---

# 18. Insight Engine

这是 V2 最重要的核心能力之一。

模型：

```text
Facts
↓
Signals
↓
Correlation
↓
Events
↓
Insights
```

示例：

```text
IP Changed
+
ASN Changed
+
Hosting Provider Changed
↓
Infrastructure Migration Event
↓
Insight
Website likely migrated infrastructure
Confidence: High
Evidence: ...
```

---

# 19. Fact / Signal / Event / Insight 必须区分

### Fact

> IP = 1.2.3.4

### Signal

> IP changed

### Event

> Infrastructure migration

### Insight

> The website likely migrated to a different hosting infrastructure.

---

# 20. History / Snapshot / Diff

已有 Snapshot 应继续利用。

建立：

```text
Snapshot A
↓
Snapshot B
↓
Diff Engine
↓
Changes
↓
Events
↓
Insights
```

检测：

- IP change
- ASN change
- CDN change
- DNS change
- MX change
- NS change
- TLS change
- Technology change
- HTTP header change
- Hosting change
- Provider change
- Subdomain change
- Tracking ID change

---

# 21. Search Intelligence

逐步建立：

- Indexed Pages
- Search Presence
- Search Signals
- Search History
- Search-derived Relationships

所有推导必须有 Evidence。

---

# 22. 双视角产品体系

这是本次新增的核心方向。

同一个域名分析完成后：

```text
Website Intelligence
↓
“这是你的站点吗？”
↓
┌─────────────────────────────┐
│ 不是                         │
│ Public Intelligence         │
└─────────────────────────────┘

┌─────────────────────────────┐
│ 是                           │
│ Owner Intelligence           │
└─────────────────────────────┘
```

---

# 23. Public Intelligence

回答：

> “这个网站是什么？”

包括：

- Website
- Infrastructure
- IP
- ASN
- DNS
- TLS
- Technology
- Organization
- Certificate
- Related Websites
- Relationships
- History
- Events
- Evidence
- Insights

核心：

```text
Discover
↓
Understand
↓
Explore
```

---

# 24. Owner Intelligence

回答：

> “我的网站现在怎么样？下一步应该怎么发展？”

包括：

- Website Audit
- Website Health
- Technical Health
- SEO Health
- Content Health
- Performance
- Security Signals
- Infrastructure Health
- Technology Assessment
- Content Opportunities
- Information Architecture
- Growth Opportunities
- Recommendations
- Roadmap

核心：

```text
Audit
↓
Diagnose
↓
Prioritize
↓
Plan
↓
Act
```

---

# 25. Website Health

建议建立综合健康度：

```text
Website Health
78 / 100

Technical       86
SEO             62
Performance     74
Content         51
Security        83
Infrastructure  91
Growth          58
```

评分必须可解释：

```text
Score
↓
Why?
↓
Evidence
↓
Recommendation
```

不能凭空生成评分。

---

# 26. Website Audit

至少覆盖：

## Technical

- HTTP
- HTTPS
- Redirect
- Headers
- TLS
- DNS
- Robots
- Sitemap
- Canonical
- Indexability
- Page structure

## SEO

- Title
- Meta Description
- H1/H2
- Canonical
- Robots
- Structured Data
- Internal Links
- Broken Links
- Duplicate Signals
- Content Coverage

## Performance

在拥有可靠数据后评估：

- Response time
- Resource count
- Page size
- Image signals
- Script signals
- Cache signals
- Core Web Vitals

## Security

- HTTPS
- Security Headers
- Certificate
- Exposed service signals
- Misconfiguration signals

## Infrastructure

- IP
- ASN
- CDN
- Hosting
- DNS
- Provider
- TLS

---

# 27. Content Intelligence

回答：

> 网站现在在讲什么？覆盖了什么？缺什么？

分析：

```text
已有页面
↓
页面主题
↓
主题覆盖度
↓
内容重复
↓
内容缺口
↓
内容机会
```

---

# 28. Content Strategy

逐步建立：

```text
Core Topic
│
├── Pillar
├── Supporting Content
├── Tutorial
├── Comparison
├── FAQ
├── Case Study
└── Tool / Resource
```

首先基于真实网站数据：

- 页面
- Title
- H1/H2
- 文本
- 内链
- 主题分布
- 网站定位

未来再接入搜索数据、竞争网站等。

---

# 29. Content Gap

示例：

```text
Topic A
Coverage: High
Status: Strong

Topic B
Coverage: Medium
Status: Needs expansion

Topic C
Coverage: Low
Status: Opportunity

Topic D
Coverage: None
Status: Potential opportunity
```

目标不是告诉站长“多写文章”，而是告诉他：

> **应该补什么。**

---

# 30. SEO Strategy

不要只告诉：

> Meta Description 缺失。

应该输出：

```text
Problem
↓
Impact
↓
Priority
↓
Recommendation
↓
Implementation Direction
```

例如：

```text
Problem
核心页面缺少独立 Meta Description

Impact
搜索摘要控制能力较弱

Priority
Medium

Recommendation
为核心页面建立独立 Description

Next Action
优先：首页、核心分类页、核心产品页
```

---

# 31. Website Architecture Strategy

分析：

- 页面层级
- URL 结构
- 分类结构
- 内链结构
- 核心页面
- 孤立页面
- 内容集群
- 首页到核心页面距离

最终给出：

> 网站信息架构应该如何调整。

---

# 32. Growth Intelligence

分析：

- 当前优势
- 当前弱点
- 内容机会
- 技术机会
- 产品机会
- 用户路径问题
- 转化问题
- 网站结构问题

最终输出：

```text
Strengths
Weaknesses
Opportunities
Priorities
```

未来接入真实流量、搜索、转化、收入数据后再扩大。

---

# 33. Recommendation Engine

建议必须结构化：

```text
recommendation_id
category
title
problem
impact
priority
evidence_ids
related_entities
estimated_effort
expected_value
status
```

每条建议必须有：

- 问题
- 证据
- 影响
- 优先级
- 建议
- 执行方向

禁止空泛的：

> “建议加强 SEO。”

---

# 34. Priority Scoring

建议考虑：

```text
Impact
×
Confidence
×
Urgency
÷
Effort
```

但具体公式必须通过真实数据验证。

输出：

```text
P0
立即处理

P1
近期处理

P2
增长建设
```

---

# 35. 30 / 60 / 90 Day Roadmap

最终 Owner Intelligence 应输出：

```text
0-30 Days
Foundation

31-60 Days
Optimization

61-90 Days
Growth
```

### P0

基础、影响大、优先级高的问题。

### P1

结构和优化。

### P2

长期增长、内容、产品建设。

---

# 36. Website Ownership Verification

公开分析与 Owner Intelligence 必须分开。

用户选择：

> “这是我的网站”

之后进行验证。

推荐：

### DNS TXT

```text
siteintel-verification=xxxx
```

### 文件

```text
/.well-known/siteintel-verification.txt
```

### HTML

```html
<meta name="siteintel-verification" content="xxxx">
```

未来可接：

- Google Search Console
- Bing Webmaster
- Analytics

---

# 37. Owner Data

验证后未来允许连接：

- Google Search Console
- Google Analytics
- Bing Webmaster
- 其他合法数据源

形成：

```text
Public Data
+
Owner Data
↓
Deeper Owner Intelligence
```

---

# 38. AI 原则

当前没有 API Key，因此：

> **SiteIntel 核心功能不能依赖外部 AI API。**

第一阶段：

```text
Crawler
+
Rules
+
Structured Data
+
Graph
+
Diff
+
Scoring
```

未来 AI 作为增强层：

```text
Structured Audit
↓
AI Explanation
↓
Natural Language Strategy
```

AI 可以：

- 解释问题
- 总结现状
- 生成内容方向
- 生成策略说明
- 生成 Roadmap 描述
- 生成 Content Brief

AI 不可以：

- 凭空创造事实
- 修改 Evidence
- 替代数据采集
- 把推测写成确定事实

---

# 39. SEO 总体策略

SEO 分两大类。

## 工具型 SEO

例如：

```text
/tools/technology
/tools/ip
/tools/dns
/tools/tls
/tools/headers
```

要求：

- 独立 Title
- 独立 Description
- H1
- 真实工具
- 可索引结果
- FAQ
- Structured Data
- Internal Links

## Intelligence / Entity SEO

例如：

```text
/technology-intelligence
/website/{domain}
/ip/{ip}
/asn/{asn}
/technology/{technology}
/certificate/{fingerprint}
```

只有具有真实数据和独立价值的实体页面才应索引。

禁止批量生成无价值 SEO 页面。

---

# 40. Entity SEO

如果数据质量足够：

```text
Website
↓
IP
↓
ASN
↓
Organization
↓
Technology
↓
Certificate
```

形成内部链接网络。

这会形成 SiteIntel 的：

> Entity SEO + Intelligence Graph

---

# 41. Internal Linking

建立：

```text
Home
↓
Tools
↓
Technology
↓
Technology Intelligence
↓
Website Report
↓
Entity Detail
↓
Related Entities
↓
Other Reports
```

每个实体页面都应该可以继续探索。

---

# 42. Monitor

Owner Intelligence 最终应该支持：

```text
Add Domain
↓
Set Frequency
↓
Run Analysis
↓
Compare Snapshot
↓
Detect Changes
↓
Create Event
↓
Generate Insight
↓
Notify
```

监控：

- IP
- DNS
- CDN
- ASN
- TLS
- Technology
- Subdomains
- Provider
- Headers
- Relationships

---

# 43. API

API 未来支持：

```text
Domain Analysis
Technology Detection
Infrastructure
Entities
Relationships
Snapshots
Changes
Insights
Evidence
```

API 必须有稳定 Schema。

API Key 应支持：

- Usage
- Quota
- Rate Limit
- Request Log
- Error Rate
- Paid quota
- Batch
- Webhook

---

# 44. 商业化

建议：

```text
Free
↓
Pro
↓
Monitor
↓
Intelligence
↓
API
↓
Enterprise
```

核心商业价值：

- 历史
- 监控
- 洞察
- 批量
- API
- Owner Strategy

而不是单纯依赖广告。

---

# 45. 数据库建议

已有 PostgreSQL 不要推翻。

逐步形成：

```text
domains
investigations
snapshots
evidence
facts
entities
relationships
technology_detections
signals
events
insights
recommendations
monitoring_targets
api_keys
usage_logs
```

采用 migration-first。

禁止直接破坏生产数据。

---

# 46. 安全

必须重点防御：

- SSRF
- URL validation
- DNS rebinding
- Private IP access
- Internal network access
- Redirect abuse
- Resource exhaustion
- Rate limiting
- Authentication
- API Key leakage
- Sensitive evidence exposure

禁止访问内部地址空间。

---

# 47. 前端原则

不要为了“高级感”堆动画。

优先：

1. 数据可信度
2. 信息层级
3. Evidence
4. Confidence
5. 可读性
6. 可操作性
7. 性能

Report 应专业、克制、偏 Intelligence Console。

---

# 48. 移动端

重点检查：

- Report
- Entity pages
- Graph
- Tables
- Evidence
- Technology
- Analyze progress
- Navigation

移动端不能只是桌面缩小。

---

# 49. 性能

优先：

- Cache
- Incremental loading
- Lazy Graph
- Lazy Evidence
- Pagination
- Background analysis
- Streaming progress
- Server-side fetching where appropriate

---

# 50. Analyze 页面

```text
/analyze/{analysisId}
```

必须支持：

```text
CREATED
↓
QUEUED
↓
RUNNING
↓
PARTIAL
↓
COMPLETED
↓
FAILED
```

刷新、断网、重新打开 URL 后能够恢复状态。

建议显示：

- Domain
- Progress
- Collector Status
- Preliminary Findings
- Errors / Warnings
- Final Report

---

# 51. Collector 架构

统一：

```text
Collector
↓
Raw Evidence
↓
Parser
↓
Normalizer
↓
Fact
```

Collector 不直接写 UI。

建议：

- DNS
- RDAP
- IP
- ASN
- TLS
- HTTP
- Headers
- Technology
- Subdomain
- Certificate
- Mail
- Relationship

---

# 52. 数据质量

建立 Data Quality Dashboard：

```text
Collector Success Rate
Parser Error Rate
Missing Fields
Contradictory Facts
Stale Evidence
Duplicate Entities
Duplicate Relationships
Confidence Distribution
```

这是情报平台必须长期维护的内部系统。

---

# 53. V2 开发阶段

## Phase 0
全仓库、数据库、API、Collector 审计。

## Phase 1
Evidence + Canonical Facts + 数据一致性。

## Phase 2
Entity + Relationship 基础完善。

## Phase 3
Entity Detail Pages。

## Phase 4
Technology Engine。

## Phase 5
Snapshot Diff + Signals + Events。

## Phase 6
Insight Engine。

## Phase 7
Website Report V2 + Graph Exploration。

## Phase 8
Owner Audit + Website Health。

## Phase 9
Content / SEO / Architecture / Growth Strategy。

## Phase 10
Owner Verification + External Data Connectors。

## Phase 11
Monitor + Alerts。

## Phase 12
API + Billing。

## Phase 13
AI Explanation / Strategy Assistant。

---

# 54. 第一阶段不要做什么

不要：

- 推翻后端
- 重写数据库
- 大规模重做 UI
- 为了 SEO 批量制造页面
- 增加没有数据支持的检测项
- 用 AI 伪造情报
- 把共享 CDN 当成确定关系
- 把弱信号写成事实
- 各模块独立解释同一数据
- 一次性实施整个 V2

---

# 55. Claude Code + DeepSeek V4 Pro 执行规则

每次开发必须：

1. 先扫描现有代码。
2. 先扫描数据库 Schema。
3. 先扫描 API。
4. 先扫描 Collector。
5. 建立 IMPLEMENTED / PARTIAL / BROKEN / MISSING 报告。
6. 禁止猜测现有能力。
7. 禁止删除已有数据。
8. 使用 migration。
9. 一个 Phase 一个 Phase 实施。
10. 每个 Phase 完成后测试。

必须验证：

```text
Build
Lint
Unit Test
Integration Test
Migration
API
Collector
Report
SEO
Mobile
Security
```

---

# 56. Definition of Done

功能必须同时具备：

```text
Backend
+
Database
+
API
+
Frontend
+
Evidence
+
Error Handling
+
Tests
+
SEO
+
Mobile
```

才算完成。

---

# 57. 最终产品架构

```text
                         SITEINTEL
                            │
                Website & Internet Intelligence
                            │
              ┌─────────────┴─────────────┐
              ↓                           ↓
     Public Intelligence           Owner Intelligence
              │                           │
              ↓                           ↓
       What is this?               What should I do?
              │                           │
       Website / IP / ASN             Audit
       Technology / TLS              Health
       Graph / History              Diagnosis
       Events / Insights            Opportunities
              │                       Strategy
              │                       Roadmap
              ↓                           ↓
          Explore                      Act
              │                           │
              └─────────────┬─────────────┘
                            ↓
                         Monitor
                            ↓
                      Analyze Again
```

---

# 58. 最终用户旅程

```text
输入域名
↓
Investigation
↓
Website Intelligence
↓
查看结论
↓
探索 IP / ASN / Technology / Certificate / Organization
↓
继续进入 Entity Intelligence
↓
查看 Graph / History / Events / Evidence
↓
如果是自己的网站
↓
Owner Verification
↓
Website Audit
↓
Website Health
↓
Problems
↓
Opportunities
↓
SEO / Content / Architecture / Growth Strategy
↓
30 / 60 / 90 Day Roadmap
↓
Monitor
↓
Changes
↓
重新分析
```

---

# 59. 最终核心产品原则

SiteIntel 不应该：

```text
后端抓数据
↓
做几十张卡片
↓
用户看完
↓
结束
```

而应该：

```text
Data
↓
Meaning
↓
Relationship
↓
Evidence
↓
Insight
↓
Recommendation
↓
Action
```

任何重要结果都应该成为下一层探索的入口。

例如：

```text
Website
↓
IP
↓
IP Intelligence
↓
Related Domains
↓
Another Website
↓
Technology
↓
Technology Intelligence
↓
Certificate
↓
Certificate Intelligence
```

Owner 路径：

```text
Website
↓
Audit
↓
Problem
↓
Evidence
↓
Recommendation
↓
Roadmap
```

---

# 60. SiteIntel 的最终目标

SiteIntel 最终不是：

> “输入域名，查一些信息。”

而是：

> **“输入一个网站，系统建立它的数字基础设施画像、技术画像、实体关系网络和历史变化，并用可验证证据生成情报；如果这是网站所有者自己的站点，则进一步告诉他网站现在怎么样、问题在哪里、内容和 SEO 应该往哪里发展，以及下一步具体做什么。”**

最终形成两个核心价值：

### 对外

> **理解互联网中的网站。**

### 对站长

> **理解自己的网站，并决定下一步怎么做。**

最终北极星：

```text
Evidence
↓
Understanding
↓
Intelligence
↓
Strategy
↓
Action
↓
Monitoring
↓
Continuous Improvement
```

这就是 SiteIntel V2 的长期产品路线。
