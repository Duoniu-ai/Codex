# SiteIntel X — SEO Architecture & Page Specification

## 1. SEO 总原则

SiteIntel SEO 不是关键词页面生成器，而是：

> 一个搜索意图 → 一个有价值页面 → 真实产品能力 → 真实数据 → 独立解释 → Evidence → Unique Intelligence

目标不是页面数量，而是**真正有用、可验证、有独立价值的页面**。

---

## 2. SEO Engine

建立统一 SEO Engine，不允许每个页面自行硬编码 SEO。

```text
SEO Engine
├── Metadata Engine
│   ├── Title
│   ├── Description
│   ├── Canonical
│   ├── Robots
│   ├── OpenGraph
│   └── Twitter/X
├── Structured Data Engine
├── Sitemap Engine
├── Internal Link Engine
├── Index Quality Gate
├── Dynamic Metadata Engine
└── Search Intent Registry
```

所有页面进入统一 `SEO Page Registry`：

```text
route
page_type
search_intent
primary_keyword
secondary_keywords
title_template
description_template
h1_template
h2_structure
canonical_strategy
schema_type
index_policy
sitemap_policy
internal_link_rules
```

---

# 3. 全站页面 SEO 规划

## `/`

**Title**

```text
SiteIntel - 网站数据分析与情报洞察平台
```

**Description**

```text
SiteIntel 是网站数据分析与情报洞察平台，分析网站基础设施、IP、DNS、技术栈、搜索表现与历史变化，帮助站长发现网站变化并理解搜索表现。
```

**H1**

```text
网站数据分析与情报洞察平台
```

**H2**

```text
网站基础设施分析
搜索引擎数据分析
网站技术栈分析
网站历史变化
网站监控与预警
Search Intelligence
```

核心词：

```text
网站数据分析
网站分析
网站情报
网站监控
SEO数据分析
```

Schema：

```text
Organization
WebSite
SoftwareApplication
```

---

## `/website-intelligence`

**Title**

```text
网站数据分析与网站情报 - SiteIntel
```

**Description**

```text
全面分析网站基础设施、IP、DNS、SSL、技术栈、HTTP 与历史变化，结合事件、证据和情报帮助站长了解网站状态。
```

**H1**

```text
网站数据分析与情报
```

**H2**

```text
网站基础设施分析
IP 与 ASN 信息
DNS 与域名分析
SSL 与安全信息
网站技术栈识别
历史变化与事件
网站关系与关联分析
```

核心词：

```text
网站数据分析
网站分析
网站情报
网站信息查询
网站基础设施分析
```

---

## `/search-intelligence`

这是 SiteIntel 最重要的产品 SEO 页面之一。

**Title**

```text
SEO搜索数据分析与搜索情报 - SiteIntel
```

**Description**

```text
分析百度、Bing、Google 搜索表现，包括收录、关键词、排名、点击、曝光、CTR、页面表现和历史变化，帮助站长发现搜索变化及潜在原因。
```

**H1**

```text
Search Intelligence
```

副标题：

```text
搜索表现分析与搜索情报
```

**H2**

```text
搜索流量分析
关键词排名分析
网站收录分析
搜索页面分析
搜索排名变化
搜索事件与历史趋势
搜索表现关联分析
```

核心词：

```text
SEO数据分析
搜索数据分析
SEO分析工具
关键词排名分析
网站排名分析
网站收录查询
```

注意：只能展示实际 Provider 能提供的数据，绝不能伪造。

---

## `/infrastructure-intelligence`

**Title**

```text
网站基础设施分析 - IP、DNS、ASN、SSL | SiteIntel
```

**Description**

```text
分析网站 IP、ASN、DNS、CDN、SSL、网络基础设施及历史变化，帮助站长识别网站架构变化、迁移和基础设施风险。
```

**H1**

```text
网站基础设施情报
```

**H2**

```text
IP 与 ASN 分析
DNS 分析
CDN 与网络基础设施
SSL 证书
基础设施变化
网站迁移检测
```

---

## `/technology-intelligence`

**Title**

```text
网站技术栈分析与技术识别 - SiteIntel
```

**Description**

```text
识别网站使用的 CMS、框架、Web Server、CDN、分析工具和第三方技术，并追踪技术栈历史变化。
```

**H1**

```text
网站技术栈分析
```

核心词：

```text
网站技术分析
网站技术栈
网站技术识别
网站框架检测
CMS检测
网站技术检测
```

---

## `/website-monitoring`

**Title**

```text
网站监控与变化检测 - SiteIntel
```

**Description**

```text
持续监控网站基础设施、技术栈、DNS、SSL和搜索表现变化，在重要变化发生时及时发现并提供事件与证据。
```

**H1**

```text
网站监控与变化检测
```

**H2**

```text
基础设施监控
技术栈监控
DNS监控
SSL监控
搜索表现监控
变化事件
智能预警
```

---

# 4. 工具页面

## `/tools/website-analysis`

Title：

```text
网站分析工具 - 网站信息与技术栈查询 | SiteIntel
```

Description：

```text
在线分析网站信息、IP、DNS、SSL、技术栈、服务器和基础设施，快速了解网站当前状态及重要技术信息。
```

H1：

```text
网站分析工具
```

---

## `/tools/dns`

Title：

```text
DNS查询与解析分析工具 - SiteIntel
```

Description：

```text
在线查询网站 DNS 记录，分析 A、AAAA、CNAME、MX、NS 等解析信息，帮助排查域名解析和网站基础设施变化。
```

H1：

```text
DNS查询工具
```

---

## `/tools/ip`

Title：

```text
IP地址查询与网站IP分析工具 - SiteIntel
```

Description：

```text
查询网站 IP 地址、ASN、网络组织及相关基础设施信息，帮助分析网站服务器和网络环境。
```

H1：

```text
IP地址查询工具
```

---

## `/tools/ssl`

Title：

```text
SSL证书查询与HTTPS安全分析 - SiteIntel
```

Description：

```text
查询网站 SSL/TLS 证书、有效期、颁发机构和 HTTPS 配置，帮助发现证书变化与安全配置问题。
```

H1：

```text
SSL证书查询工具
```

---

## `/tools/technology`

Title：

```text
网站技术栈检测工具 - CMS、框架与服务器识别 | SiteIntel
```

Description：

```text
在线检测网站使用的 CMS、Web 框架、服务器、CDN、统计工具及其他技术组件。
```

H1：

```text
网站技术栈检测
```

---

# 5. 搜索引擎页面

## `/search/baidu`

Title：

```text
百度SEO数据分析与排名监测 - SiteIntel
```

Description：

```text
SiteIntel 百度 Search Intelligence 提供网站收录、关键词排名、排名变化、页面表现及搜索事件分析，并区分官方数据与SERP观测数据。
```

H1：

```text
百度SEO数据分析
```

H2：

```text
百度收录分析
百度关键词排名
百度排名变化
百度页面表现
百度SERP监测
百度搜索事件
```

**关键规则：**

百度官方数据和百度 SERP Observation 必须始终分开。

不得把 SERP 观测数据描述为百度官方 API 数据。

---

## `/search/bing`

Title：

```text
Bing SEO数据分析与关键词排名监测 - SiteIntel
```

Description：

```text
分析Bing搜索流量、关键词、排名、页面表现和抓取数据，帮助跨境网站、英文网站和独立站持续监测搜索表现。
```

H1：

```text
Bing SEO数据分析
```

---

## `/search/google`

Title：

```text
Google SEO数据分析与搜索表现监测 - SiteIntel
```

Description：

```text
分析Google搜索点击、曝光、CTR、平均排名、关键词和页面表现，帮助独立站、SaaS和出海网站了解自然搜索变化。
```

H1：

```text
Google SEO数据分析
```

Google 尚未接入时可以保留产品介绍页，但绝不能显示假数据。

---

# 6. 动态网站页面 `/website/[domain]`

这是未来最重要的 Programmatic SEO 页面。

例如：

```text
/website/example.com
```

Title：

```text
example.com 网站分析、技术栈与基础设施信息 - SiteIntel
```

Description：

```text
查看 example.com 的网站基础设施、IP、DNS、SSL、技术栈、搜索表现及历史变化。
```

H1：

```text
example.com 网站分析
```

推荐内容：

```text
网站概览
基础设施
IP 与 ASN
DNS
SSL
技术栈
搜索表现
历史变化
事件
关联网站
Insights
Evidence
```

最低必须存在有意义的真实数据：

```text
Domain
IP
ASN
DNS
SSL
Technology
Last Observed
Events
```

---

# 7. Dynamic Page Index Quality Gate

不能把所有域名自动开放给搜索引擎。

必须评估：

```text
Data Coverage
Evidence Coverage
Historical Depth
Unique Intelligence
Content Quality
```

示例：

```text
>= 80
INDEX

60-79
REVIEW / CONDITIONAL

< 60
NOINDEX
```

注意：

这个分数只是 SiteIntel 内部的**发布质量门槛**，不能叫 Google SEO Score、Baidu SEO Score 或 Search Health。

不合格页面：

```html
<meta name="robots" content="noindex,follow">
```

只有真正可索引页面才能进入 Sitemap。

---

# 8. `/technology/[technology]`

例如：

```text
/technology/wordpress
/technology/cloudflare
/technology/next-js
```

Title 示例：

```text
WordPress网站技术分析与使用情况 - SiteIntel
```

Description：

```text
查看WordPress技术信息、相关网站、技术关系及SiteIntel观测到的网站技术变化。
```

H1：

```text
WordPress 网站技术情报
```

内容：

```text
Technology Overview
Detected Websites
Technology Relationships
Recent Changes
Related Technologies
```

只有真实数据足够才允许 Index。

---

# 9. `/entity/[entity]`

例如：

```text
/entity/cloudflare
/entity/amazon-web-services
```

只为真正有价值、数据充分的实体创建可索引页面。

Title 示例：

```text
Cloudflare 网站基础设施与技术情报 - SiteIntel
```

H1：

```text
Cloudflare 技术情报
```

内容：

```text
Entity Overview
Technology Information
Infrastructure Relationships
Observed Websites
Historical Changes
Evidence
```

---

# 10. `/relationship/[relationship]`

例如：

```text
/relationship/cloudflare-wordpress
```

必须有充分关系证据才能 Index。

内容：

```text
Relationship Overview
Evidence
Observed Websites
Technology
Infrastructure
History
```

禁止为所有可能的实体组合批量生成页面。

---

# 11. `/insight/[insight]`

Insight 是 Intelligence Content，不是普通博客。

内容：

```text
Insight Summary
Observed Pattern
Evidence
Affected Websites
Timeline
Confidence
Related Technology
Related Infrastructure
```

没有真实证据：

```text
NOINDEX
```

---

# 12. `/cases/[case]`

案例页是高质量 SEO 内容。

结构：

```text
发生了什么
什么时候发生
网站发生哪些变化
基础设施发生什么
搜索表现发生什么
Evidence
分析
结论
```

必须基于真实观察，或明确标记为方法论示例。

---

# 13. `/guides/[topic]`

知识库示例：

```text
/guides/website-seo
/guides/website-migration
/guides/dns
/guides/cdn
/guides/ssl
/guides/seo-ranking
/guides/indexing
/guides/baidu-seo
/guides/bing-seo
```

每篇文章必须真正解决一个问题：

```text
定义
问题
原因
检测方法
分析方法
解决方法
相关 SiteIntel 工具
相关 Intelligence
```

禁止批量生成同质化 AI 文章。

---

# 14. Breadcrumb

所有深层页面使用真实层级 BreadcrumbList：

```text
首页
>
搜索情报
>
百度
>
关键词排名
```

或：

```text
首页
>
网站分析
>
example.com
```

---

# 15. Canonical

所有 Indexable 页面必须有 canonical。

示例：

```html
<link rel="canonical"
      href="https://siteintel.cc/website/example.com">
```

规范化：

```text
/?domain=example.com
?tab=search
?tab=technology
?page=2
?sort=...
```

避免参数 URL 产生大量重复页面。

---

# 16. Robots

允许公开内容抓取。

私有区域：

```text
/api/
/admin/
/dashboard/
/account/
/login/
/settings/
/internal/
```

禁止作为主要的 noindex 手段。需要抓取但不索引的页面使用：

```html
<meta name="robots" content="noindex,follow">
```

---

# 17. Sitemap

推荐：

```text
/sitemap.xml

/sitemap-pages.xml
/sitemap-tools.xml
/sitemap-guides.xml
/sitemap-websites-1.xml
/sitemap-websites-2.xml
/sitemap-technologies.xml
/sitemap-entities.xml
```

只有：

```text
indexable = true
```

的页面进入 Sitemap。

只有发生实际内容变化时才更新 `lastmod`。

---

# 18. Internal Link Architecture

核心：

```text
首页
↓
产品 Intelligence
↓
Website
↓
Technology
↓
Infrastructure
↓
Search
↓
Insight
↓
Case
↓
Guide
```

例如：

```text
Website → Technology
Website → Infrastructure
Website → Search
Technology → Related Websites
Search → Search Events
Search Event → Website Event
Insight → Evidence
Guide → Tool
```

必须是上下文相关内链，禁止随机大量 footer 链接。

---

# 19. Data Source Display

数据页面显示：

```text
Source: SiteIntel Observation
```

或：

```text
Source: Baidu Official
Source: Baidu SERP Observation
Source: Bing Webmaster
Source: Google Search Console
```

并显示：

```text
Last Observed
```

或：

```text
Last Synced
```

不能把历史数据伪装成实时数据。

---

# 20. Search Data SEO Rules

只有 Provider 实际提供的数据才能展示：

```text
Clicks
Impressions
CTR
Average Position
Keywords
Indexed Pages
Ranking Changes
```

不可用字段：

```text
Not available from current source
```

禁止伪造：

```text
Search Volume
CTR
Clicks
Impressions
Position
Traffic
Index Count
```

---

# 21. Structured Data

首页：

```text
Organization
WebSite
SoftwareApplication
```

工具 / 产品：

```text
WebPage
SoftwareApplication
BreadcrumbList
```

指南：

```text
Article
BreadcrumbList
```

网站情报：

```text
WebPage
BreadcrumbList
```

技术情报：

```text
WebPage
BreadcrumbList
```

只有页面真实符合 Dataset 时才使用 Dataset。

结构化数据必须与页面可见内容一致。

---

# 22. OpenGraph

所有 Indexable 页面：

```text
og:title
og:description
og:url
og:type
og:site_name
og:image
```

动态网站页面可生成：

```text
example.com
Website Intelligence
IP / DNS / Technology / Search
Last Observed
```

不得生成误导性图片。

---

# 23. Rendering

SEO 核心内容必须存在于搜索引擎可访问的 HTML。

禁止完全依赖：

```text
Canvas
Client-only rendering
首次加载后才请求全部内容的纯客户端页面
```

优先：

```text
SSR
SSG
Prerendering
```

交互式 Dashboard 可以继续使用 Client-side UI。

---

# 24. Performance

SEO 实现不得破坏性能。

要求：

```text
快速初始 HTML
最小阻塞 JS
优化 CSS
图片优化
非关键图片懒加载
字体优化
缓存
压缩
```

SEO 内容不能等待完整 Dashboard JS 加载后才出现。

---

# 25. Index Policy

统一：

```text
INDEX
NOINDEX
INDEX_AFTER_QUALITY_GATE
PRIVATE
```

建议：

```text
Homepage
→ INDEX

Product pages
→ INDEX

Tool pages
→ INDEX

Guides
→ INDEX

User dashboard
→ PRIVATE

Account
→ PRIVATE

Website intelligence
→ INDEX_AFTER_QUALITY_GATE

Technology entity
→ INDEX_AFTER_QUALITY_GATE

Relationship
→ INDEX_AFTER_QUALITY_GATE

Insight
→ INDEX_AFTER_QUALITY_GATE

Thin dynamic page
→ NOINDEX
```

---

# 26. Duplicate / Thin Page Protection

不能出现：

```text
example123.com
IP: 1.2.3.4
Technology: nginx
```

这种页面就直接 Index。

动态页面至少需要足够的：

```text
真实数据
历史
Evidence
事件
关系
独立分析
```

如果多个页面高度相似：

```text
canonical
或
noindex
```

禁止仅仅替换一个域名名称就批量生产几乎完全相同的页面。

---

# 27. Search Intent Mapping

```text
网站分析
→ /tools/website-analysis

DNS查询
→ /tools/dns

网站技术栈
→ /tools/technology

SEO搜索数据分析
→ /search-intelligence

百度SEO
→ /search/baidu

Bing SEO
→ /search/bing

Google SEO
→ /search/google

example.com分析
→ /website/example.com
```

避免多个页面争夺完全相同的核心搜索意图。

---

# 28. Title Rules

推荐：

```text
核心主题 + 用户价值 + 品牌
```

正确：

```text
网站技术栈检测工具 - CMS、框架与服务器识别 | SiteIntel
```

错误：

```text
网站技术检测_网站技术栈检测_CMS检测_网站分析_SEO工具_SiteIntel
```

禁止关键词堆砌。

---

# 29. Description Rules

描述：

```text
页面是什么
+
用户能得到什么
```

例如：

```text
在线检测网站使用的 CMS、Web 框架、服务器、CDN、统计工具及其他技术组件。
```

每个重要页面使用独立 description。

---

# 30. Heading Rules

每个 Indexable 页面：

```text
1 × H1
multiple H2
H3 when needed
```

示例：

```text
H1
example.com 网站分析

H2
网站概览

H2
网站基础设施

H2
网站技术栈

H2
搜索表现

H2
历史变化
```

---

# 31. Content Freshness

动态 Intelligence 页面显示：

```text
Last Observed
Last Updated
Data Sources
Historical Timeline
```

不要为了 SEO 人为改变 `lastmod`。

只有真实数据变化才更新。

---

# 32. SEO Flywheel

```text
搜索
↓
SiteIntel Intelligence Page
↓
用户分析网站
↓
真实数据生成
↓
Events
↓
Evidence
↓
Insights
↓
页面价值增加
↓
搜索可见性提升
↓
更多用户
↓
更多观察网站
↓
更多 Intelligence
```

优先使用真实产品数据形成 SEO 飞轮，而不是大量 AI 内容。

---

# 33. 页面优先级

## S级：第一批

```text
/
/website-intelligence
/search-intelligence
/infrastructure-intelligence
/technology-intelligence
/website-monitoring

/tools/website-analysis
/tools/dns
/tools/ip
/tools/ssl
/tools/technology

/search/baidu
/search/bing
/search/google
```

## A级：核心产品稳定后

```text
/website/[domain]
/technology/[technology]
/guides/[topic]
/cases/[case]
```

## B级：数据规模起来后

```text
/entity/[entity]
/relationship/[relationship]
/insight/[insight]
```

---

# 34. SEO Validation

实现内部：

```text
/admin/seo
```

必须 PRIVATE / NOINDEX。

展示：

```text
Route
Title
Description
H1
Canonical
Robots
Schema
Index Decision
Sitemap Status
Internal Links
Quality Gate
```

示例：

```text
/website/example.com

Index:
YES

Quality:
87

Title:
OK

Description:
OK

Canonical:
OK

Schema:
OK

Sitemap:
INCLUDED

Internal Links:
14

Evidence:
27

Last Observed:
2026-08-14
```

---

# 35. Technical SEO Checklist

自动检查：

```text
[ ] Unique title
[ ] Unique description
[ ] One H1
[ ] Canonical
[ ] Robots
[ ] Breadcrumb
[ ] JSON-LD
[ ] Sitemap inclusion
[ ] Internal links
[ ] SSR/HTML content
[ ] OpenGraph
[ ] No accidental noindex
[ ] No accidental canonical
[ ] No duplicate URL
[ ] No broken internal links
[ ] No orphan pages
[ ] Lastmod correctness
```

---

# 36. Claude Code 实施要求

把 SEO 做成**架构能力**，不要散落在页面代码里。

必须：

1. 建立 SEO Page Registry。
2. 建立 Metadata Engine。
3. 建立 Structured Data Engine。
4. 建立 Sitemap Engine。
5. 建立 Internal Link Engine。
6. 建立 Index Quality Gate。
7. 建立 Dynamic Metadata。
8. 建立 Search Intent Registry。
9. 为 SEO 核心内容实现 SSR / SSG / Prerender。
10. 建立 SEO Validation / Debug 页面。
11. 不伪造 SEO 数据。
12. 不伪造搜索引擎数据。
13. 不声称 Provider 没有提供的官方 API 字段。
14. 官方搜索数据与 SERP Observation 必须分开。
15. 禁止批量生成薄 Programmatic SEO 页面。
16. 动态页面必须通过质量门槛。
17. 只有 Indexable 页面进入 Sitemap。
18. 每个 Indexable 页面必须有唯一 Metadata。
19. 每个页面必须有明确 Search Intent。
20. 每个页面必须有合理内链关系。
21. 保留现有 SiteIntel 功能。
22. 不推翻现有 Website Intelligence 数据模型。
23. 优先扩展现有 `events`、`evidence`、`insights`、`snapshots`，避免建立重复概念。
24. SEO 是产品能力，不是一堆静态 Meta 标签。

最终目标：

```text
SiteIntel
=
Website Intelligence
+
Search Intelligence
+
Evidence
+
History
+
Relationships
+
SEO Knowledge Graph
```

SEO 层必须把真实 SiteIntel Intelligence 转化为长期搜索价值，同时不能牺牲产品质量。
