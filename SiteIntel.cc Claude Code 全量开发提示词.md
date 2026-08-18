# SiteIntel.cc — Website & Internet Intelligence Platform

你现在是这个项目的首席产品架构师、全栈工程师、UI/UX 设计师和数据工程师。

我要你从零开始开发一个生产级网站：

**SiteIntel**
**Domain:** https://siteintel.cc

产品定位：

> Website & Internet Intelligence

中文：

> 网站与互联网基础设施情报平台

---

# 1. 核心产品理念

SiteIntel 不是普通的 WHOIS / DNS / IP 查询工具。

它必须把：

- Domain
- URL
- IP
- ASN
- DNS
- RDAP / WHOIS
- HTTP
- TLS
- Website Technology
- CDN
- WAF
- Hosting
- Security Headers
- Reputation
- Subdomains
- Certificates
- Historical Snapshots
- Infrastructure Relationships

整合成一个完整的：

**Intelligence Report**

核心用户体验：

用户输入：

example.com

↓

SiteIntel 自动调查

↓

实时显示调查进度

↓

收集多个数据源

↓

标准化数据

↓

建立 Domain / IP / ASN / Technology 关系

↓

计算可解释的 Intelligence Score

↓

生成完整报告

---

# 2. 产品原则

必须遵守以下原则：

1. 不伪造数据。
2. 没有配置 Provider 时必须显示 `Not configured`，不能显示 Clean。
3. 所有外部数据必须记录 source/provider。
4. 所有数据必须记录 collected_at / observed_at。
5. 所有技术识别必须尽可能提供 evidence。
6. 所有评分必须可以解释。
7. AI 不能制造事实数据。
8. AI 只能负责解释已经采集并结构化的数据。
9. 前端不能硬编码假数据作为真实结果。
10. 所有扫描任务必须有明确状态。
11. 扫描失败必须允许部分成功。
12. 一个 Provider 失败不能导致整个 Investigation 失败。
13. 所有 Provider 必须采用 Adapter 模式。
14. 不要把所有数据塞进一个 JSON。
15. PostgreSQL 是核心数据存储。
16. Raw JSON 只能作为 raw_response / provider_metadata / evidence 等辅助字段。
17. 所有 API 必须有 schema validation。
18. 所有敏感 API Key 只能存在服务端环境变量。
19. 不允许把第三方 API Key 暴露给浏览器。
20. 所有扫描必须考虑 SSRF 风险。

---

# 3. 技术栈

优先使用：

Frontend:

- Next.js
- TypeScript
- Tailwind CSS
- shadcn/ui

Backend:

- Next.js Route Handlers
- Node.js

Database:

- PostgreSQL

ORM:

- Prisma

Validation:

- Zod

Queue:

- Redis
- BullMQ

Graph:

- React Flow

Charts:

- Recharts

Icons:

- Lucide

Authentication:

- Auth.js 或成熟的 Next.js Auth 方案

Deployment:

- Vercel 用于 Web/API
- Worker 使用独立运行环境

---

# 4. 架构

整体架构：

Browser
↓
Next.js
↓
API
↓
Investigation Service
↓
Job Queue
↓
Worker
↓
Provider Adapters
↓
Normalizer
↓
Correlation Engine
↓
Scoring Engine
↓
Snapshot
↓
Report

不要让 Vercel Function 承担长时间、大规模扫描。

Vercel：

- 页面
- API
- Auth
- SSE
- 轻量任务

Worker：

- DNS
- HTTP
- TLS
- Technology Detection
- Screenshot
- Subdomain Discovery
- Bulk Scan
- Monitoring

---

# 5. 首页

首页必须极简、专业、现代。

参考设计语言：

- Linear
- Vercel
- Stripe
- Cloudflare
- Security Research SaaS

不要做传统中国工具站风格。

不要：

- 大量渐变
- 花哨动画
- 大面积彩色背景
- 复杂卡片堆砌
- 廉价 SaaS 感

应该：

- 大量留白
- 清晰 typography
- 黑白/灰色为主
- 少量品牌 Accent
- 精致边框
- 专业数据产品感觉

Hero：

SITEINTEL

Website & Internet Intelligence

Understand any website, domain and infrastructure in seconds.

输入框：

Enter a domain, IP or URL

按钮：

Investigate

支持：

Domain
IP
URL
ASN

---

# 6. 首页内容

Hero 后展示：

## What can SiteIntel tell you?

六个模块：

Identity
Who is behind this domain?

Infrastructure
Where is it hosted?

Technology
What technologies does it use?

Security
How secure is the configuration?

Relationships
What infrastructure is connected?

History
What changed over time?

---

# 7. Investigation 流程

用户输入 Domain / URL / IP / ASN 后：

POST /api/v1/investigations

返回：

{
  "id": "...",
  "status": "queued"
}

然后前端通过 SSE：

GET /api/v1/investigations/:id/events

实时接收进度。

进度必须显示：

Investigating example.com

✓ Input normalized
✓ Domain resolved
✓ DNS records collected
✓ IP addresses identified
✓ ASN identified
✓ RDAP analyzed
✓ TLS analyzed
✓ HTTP analyzed
● Detecting technologies
○ Security analysis
○ Reputation analysis
○ Building relationships
○ Generating report

不能只显示：

Loading...

---

# 8. Investigation 状态

定义：

QUEUED
RUNNING
PARTIAL
COMPLETED
FAILED
CANCELLED

每一个 Provider Task：

QUEUED
RUNNING
COMPLETED
FAILED
SKIPPED

允许 partial success。

例如：

DNS 成功
TLS 成功
Technology Provider 失败

最终：

PARTIAL

而不是整个 Investigation FAILED。

---

# 9. Domain Report

最终页面：

/domain/[domain]

顶部：

example.com

Website & Domain Intelligence

Overall Intelligence Score
87 / 100

LOW RISK

然后显示：

Country
Organization
ASN
Hosting
CDN
Technology

---

# 10. Report Tabs

必须有：

Overview
Identity
Infrastructure
Technology
Security
Reputation
Subdomains
Relationships
History

---

# 11. Overview

展示：

Intelligence Score

Identity Score
Infrastructure Score
Security Score
Reputation Score
Technology Score

并且提供：

Why this score?

例如：

+ TLS 1.3 enabled
+ HSTS enabled
+ DMARC configured
- CSP missing

所有分数必须可解释。

---

# 12. Identity

字段：

Domain
Status
Registrar
Created At
Updated At
Expires At
Privacy
Nameservers

优先使用 RDAP。

WHOIS 作为 fallback。

---

# 13. DNS

支持：

A
AAAA
CNAME
MX
NS
TXT
CAA
SOA

另外分析：

SPF
DMARC
DKIM

DNS 数据必须保存：

record_type
name
value
ttl
source
observed_at

---

# 14. Infrastructure

展示：

IP
Country
Region
City
ASN
Organization
Network
CDN
WAF
Hosting Provider

必须支持多个 IP。

不能假设一个域名只有一个 IP。

---

# 15. IP Intelligence

每个 IP 都可以点击：

/ip/[ip]

展示：

IP
Version
Country
Region
City
ASN
Organization
Network
Reverse DNS
Hostname
CDN
Datacenter
First Seen
Last Seen

---

# 16. ASN Intelligence

页面：

/asn/[asn]

展示：

ASN
Organization
Country
Prefixes
Known Domains
Known IPs

---

# 17. HTTP Analysis

采集：

Status Code
Final URL
Redirect Chain
Response Time
Server
Content Type
Content Length
Compression
HTTP Version
HTTP/2
HTTP/3
Cookies
Headers

同时记录：

favicon hash
body hash
header hash

---

# 18. TLS

展示：

Certificate
Issuer
Subject
SAN
Valid From
Valid To
Serial Number
Fingerprint
TLS Version
Cipher
ALPN

计算：

Certificate Status
Certificate Expiration
TLS Security

---

# 19. Security Headers

检查：

Strict-Transport-Security
Content-Security-Policy
X-Frame-Options
X-Content-Type-Options
Referrer-Policy
Permissions-Policy
Cross-Origin-Opener-Policy
Cross-Origin-Resource-Policy

每项：

PASS
WARN
FAIL
NOT_DETECTED

并提供 explanation。

---

# 20. Technology Detection

必须采用证据驱动。

TechnologyDetection：

{
  technology,
  category,
  version,
  confidence,
  evidence,
  source,
  observed_at
}

Technology categories：

Frontend
Backend
Framework
CMS
JavaScript
CSS
Analytics
Advertising
CDN
Cloud
Hosting
Payment
Security
Monitoring
Web Server
Database

Provider Adapter：

TechnologyProvider

可以支持：

Custom Fingerprint
Wappalyzer 等第三方 Provider

第三方 API 没有 Key 时：

不要伪造。

可以使用 Custom Fingerprint 作为基础。

---

# 21. Custom Technology Fingerprint

建立：

technology_definitions

每一个技术包含：

name
slug
category
website
fingerprints
confidence_rules

Fingerprints 可以检测：

HTML
Script
CSS
Headers
Cookies
Meta
URLs
JS globals

每次识别必须保存 evidence。

例如：

Next.js

evidence:

__NEXT_DATA__
_next/static/
x-powered-by

---

# 22. Screenshot

支持 Website Screenshot。

但截图必须在 Worker 执行。

不能在前端直接执行。

保存：

screenshot_url
captured_at
viewport

第一版可以做：

Desktop Screenshot

以后：

Mobile Screenshot

---

# 23. Reputation

设计统一 Provider Interface：

ReputationProvider

支持：

checkDomain()
checkIp()

Provider 可以包括：

VirusTotal
AbuseIPDB
URLhaus
Spamhaus
Google Safe Browsing

但是如果 Provider 没有 API Key：

显示：

Not configured

不能显示：

Clean

---

# 24. Subdomain

保存：

subdomain
ip
status
technology
first_seen
last_seen

页面：

/domain/example.com/subdomains

表格：

Subdomain
IP
HTTP Status
Technology
First Seen
Last Seen

---

# 25. Relationship Engine

这是 SiteIntel 的核心壁垒。

建立：

relationships

字段：

source_type
source_id
target_type
target_id
relationship_type
confidence
evidence
first_seen
last_seen

关系类型：

SHARED_IP
SHARED_ASN
SHARED_CERTIFICATE
SHARED_NAMESERVER
SHARED_CDN
SHARED_WAF
SHARED_ANALYTICS_ID
SHARED_FAVICON_HASH
SHARED_TECHNOLOGY
SAME_ORGANIZATION

注意：

共享 Cloudflare 不能直接证明两个域名属于同一组织。

Relationship 必须有：

type
confidence
evidence

---

# 26. Infrastructure Graph

使用 React Flow。

节点：

Domain
Subdomain
IP
ASN
Organization
Certificate
Technology
CDN
WAF

边：

RESOLVES_TO
HOSTED_ON
ANNOUNCED_BY
USES_CERTIFICATE
USES_TECHNOLOGY
PROTECTED_BY
RELATED_TO

图必须支持：

Zoom
Pan
Node Click
Edge Click
Hide/Show Node Types

---

# 27. History

每一次完整 Investigation 都创建 Snapshot。

保存：

domain_snapshot
dns_snapshot
ip_snapshot
certificate_snapshot
technology_snapshot
security_snapshot
http_snapshot

比较：

previous snapshot
current snapshot

生成：

DNS_CHANGED
IP_CHANGED
ASN_CHANGED
CERTIFICATE_CHANGED
TECHNOLOGY_CHANGED
HTTP_CHANGED
SECURITY_CHANGED
SUBDOMAIN_CHANGED

---

# 28. Change Event

ChangeEvent：

{
  type,
  severity,
  previous_value,
  current_value,
  detected_at,
  evidence
}

严重级别：

INFO
LOW
MEDIUM
HIGH
CRITICAL

---

# 29. Domain History UI

例如：

Technology Changes

2026-08-14

Next.js 14
→
Next.js 15

DNS Changes

104.xxx.xxx.xxx
→
172.xxx.xxx.xxx

Certificate Changes

Certificate A
→
Certificate B

必须提供 Timeline。

---

# 30. Monitoring

用户可以：

Monitor Domain

选：

DNS
IP
ASN
Certificate
Technology
Website
Security
Subdomains

频率：

Daily
Every 6 hours
Every hour

不同计划限制频率。

---

# 31. Monitoring Worker

定期：

monitor_runs

↓

run investigation

↓

compare snapshot

↓

generate change events

↓

如果有重要变化：

Notification

---

# 32. Notification

第一版：

Email

以后：

Webhook
Discord
Slack
Telegram

---

# 33. Search

以后支持：

Domain Search
IP Search
ASN Search
Technology Search

例如：

technology:next.js

返回：

Domains using Next.js

例如：

asn:AS13335

返回：

Domains / IPs

---

# 34. Domain Compare

页面：

/compare

输入：

example.com
competitor.com

比较：

Infrastructure
Technology
Security
DNS
TLS
ASN
Hosting
CDN

视觉上：

左右两列。

---

# 35. Database

使用 PostgreSQL + Prisma。

核心表：

users
organizations
domains
domain_scans
domain_snapshots
dns_records
ip_addresses
asns
certificates
technologies
technology_detections
subdomains
http_observations
security_observations
reputation_observations
relationships
change_events
monitors
monitor_runs
api_keys
usage_records

不要把所有数据放在：

domains.intelligence JSONB

结构化数据必须使用独立表。

JSONB 仅用于：

raw_response
provider_metadata
evidence

---

# 36. Domain Schema

至少包含：

id
name
normalized_name
status
first_seen
last_seen
created_at
updated_at

增加唯一索引：

normalized_name

---

# 37. Snapshot Schema

每次调查必须有：

id
domain_id
investigation_id
observed_at
version

所有数据都必须能够关联回 Snapshot。

---

# 38. Provider Adapter

统一接口：

interface Provider<TInput, TOutput> {
  name: string
  check(input: TInput): Promise<TOutput>
}

Provider 不允许直接修改数据库。

Provider：

采集数据

↓

返回 normalized result

↓

Service 保存

这样方便以后替换供应商。

---

# 39. Normalization

所有 Provider 输出必须进入 Normalizer。

例如：

Provider A：

country_code

Provider B：

countryCode

最终统一：

country_code

同理：

IP
ASN
Organization
Timestamp

必须统一。

---

# 40. Scoring Engine

建立：

scoring/

例如：

identity-score.ts
security-score.ts
reputation-score.ts
infrastructure-score.ts
technology-score.ts
overall-score.ts

评分必须 deterministic。

输入：

normalized data

输出：

score
factors
warnings
positives

例如：

{
  score: 82,
  factors: [
    {
      type: "positive",
      code: "TLS_1_3",
      points: 10,
      explanation: "TLS 1.3 is enabled."
    }
  ]
}

---

# 41. SSRF 安全

这是必须认真处理的。

任何用户输入 URL / IP 都必须：

1. Normalize
2. Validate
3. Resolve DNS
4. Block private IP
5. Block loopback
6. Block link-local
7. Block metadata endpoints
8. Block localhost
9. Block internal network
10. Re-check after DNS resolution
11. Limit redirects
12. Validate redirect destination

禁止：

127.0.0.1
localhost
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
169.254.0.0/16
::1
fc00::/7
metadata service IP

必须防止 DNS rebinding。

---

# 42. Rate Limit

游客：

3 investigations/day

注册用户：

20/day

Pro：

1000/month

API：

按照 API Plan

所有限制必须服务端执行。

不能只依赖前端。

---

# 43. Cache

缓存：

DNS
RDAP
Technology
IP Intelligence
ASN

但必须显示：

Data freshness

例如：

Updated 4 minutes ago

---

# 44. Freshness

所有数据必须显示：

Observed
Collected
Updated

例如：

Last observed:

2026-08-14 10:21 UTC

---

# 45. Error Handling

不能出现：

Something went wrong

必须：

DNS Provider unavailable

或者：

Technology detection timed out

同时允许 Retry。

---

# 46. API

设计：

POST /api/v1/investigations
GET /api/v1/investigations/:id
GET /api/v1/domains/:domain
GET /api/v1/domains/:domain/dns
GET /api/v1/domains/:domain/ips
GET /api/v1/domains/:domain/technology
GET /api/v1/domains/:domain/security
GET /api/v1/domains/:domain/history
GET /api/v1/domains/:domain/relationships
GET /api/v1/ip/:ip
GET /api/v1/asn/:asn

API 返回标准 JSON。

所有输入使用 Zod。

---

# 47. API Authentication

API Key：

si_live_xxxxxxxxx

数据库：

api_keys

保存 hash。

不要保存明文 API Key。

只在创建时显示一次。

支持：

rate limit
usage
revoked_at
created_at

---

# 48. Billing

第一阶段不要过度实现。

设计：

Free
Pro
Intelligence
API

Free：

3 investigations/day

Pro：

1000 investigations/month

Intelligence：

高级历史、关系图谱、监控

API：

按 credits / usage

先把权限系统做好。

支付系统可以后接 Stripe。

---

# 49. SEO

第一版：

/
 /domain/[domain]
 /ip/[ip]
 /asn/[asn]
 /technology/[technology]

但是：

只有有真实数据的页面允许 index。

禁止大量生成空页面。

Metadata：

title
description
canonical
OG image

---

# 50. robots.txt

必须：

robots.txt
sitemap.xml

动态 sitemap。

---

# 51. 页面

必须完成：

/
 /investigate
 /domain/[domain]
 /ip/[ip]
 /asn/[asn]
 /compare
 /search
 /pricing
 /docs
 /login
 /dashboard
 /settings
 /api-docs

第一阶段优先：

/
 /investigate
 /domain/[domain]

---

# 52. Dashboard

登录用户：

Recent Investigations

Monitored Domains

Recent Changes

Usage

API Usage

---

# 53. UI 组件

建立：

DomainSearchInput
InvestigationProgress
ScoreCard
ScoreBreakdown
DataFreshness
DnsRecordTable
IpCard
AsnCard
TechnologyCard
SecurityCheck
CertificateCard
RelationshipGraph
Timeline
ChangeEvent
SubdomainTable
ProviderStatus
EmptyState
ErrorState
LoadingState

---

# 54. Loading UI

所有数据页面都需要 skeleton。

不能出现页面完全空白。

---

# 55. Responsive

必须支持：

Desktop
Tablet
Mobile

Mobile 优先保证：

Search
Overview
Score
Infrastructure
Technology

体验完整。

---

# 56. Accessibility

必须支持：

Keyboard navigation
Focus states
ARIA labels
Contrast
Semantic HTML

---

# 57. Dark Mode

默认：

Dark

同时支持：

Light

设计不要依赖纯黑。

使用：

background
surface
border
muted
foreground
accent

统一 design tokens。

---

# 58. Design Tokens

不要在组件里面到处写颜色。

建立：

--background
--foreground
--surface
--surface-hover
--border
--muted
--accent
--success
--warning
--danger

---

# 59. Animations

只做非常轻量：

fade
slide
progress

不要：

大量 bouncing
复杂粒子
3D
炫酷背景

SiteIntel 应该像专业情报软件。

---

# 60. Project Structure

建议：

src/
  app/
  components/
  lib/
    db/
    providers/
    scanners/
    normalizers/
    scoring/
    security/
    queue/
    cache/
  server/
  workers/
  types/

prisma/
  schema.prisma

tests/
  unit/
  integration/
  e2e/

---

# 61. Testing

必须建立：

Unit tests

测试：

domain normalization
IP validation
CIDR blocking
SSRF protection
score calculation
DNS normalization
technology detection
snapshot diff

Integration tests：

investigation pipeline

E2E：

输入 example.com

↓

创建 investigation

↓

得到 report

---

# 62. Mock Provider

开发阶段建立：

MockDNSProvider
MockRDAPProvider
MockIPProvider
MockTechnologyProvider

方便没有 API Key 时测试。

但：

Mock 数据必须明确标记：

environment = test

生产环境不能返回 Mock 数据。

---

# 63. Environment Variables

建立：

DATABASE_URL
REDIS_URL
NEXTAUTH_SECRET

以及：

WAPPALYZER_API_KEY
VIRUSTOTAL_API_KEY
ABUSEIPDB_API_KEY

等等。

不要把真实 Key 写入代码。

同时生成：

.env.example

---

# 64. Logging

统一 logger。

每次 Investigation：

investigation_id

每个 Job：

job_id

每个 Provider：

provider

所有错误都可以追踪。

---

# 65. Observability

记录：

scan_duration
provider_duration
provider_error
queue_latency
cache_hit
cache_miss

以后方便优化成本。

---

# 66. 数据成本控制

必须考虑：

API Provider 有费用。

所以：

优先：

Cache

其次：

Cheap Provider

最后：

Expensive Provider

例如：

DNS

↓

HTTP

↓

Custom Technology

↓

Paid Technology Provider

↓

Reputation Provider

只有用户需要高级数据时才调用昂贵 Provider。

---

# 67. Investigation Pipeline

标准流程：

1. Normalize input
2. Validate input
3. Create investigation
4. Resolve domain
5. DNS
6. IP
7. RDAP
8. ASN
9. HTTP
10. TLS
11. Technology
12. Security
13. Reputation
14. Subdomain
15. Relationships
16. Score
17. Snapshot
18. Change detection
19. Generate report
20. Mark completed

其中可以并行：

DNS
RDAP
HTTP
TLS

后续依赖：

IP → ASN

HTTP → Technology

Snapshot → Change Detection

---

# 68. Parallel Processing

尽量：

Promise.allSettled()

不要：

Promise.all()

因为一个 Provider 失败不能影响其他 Provider。

---

# 69. Report Generation

Report 不是重新扫描。

Report 必须从：

normalized database data

生成。

这样：

页面
API
PDF
AI Summary

都可以复用同一份 Intelligence Model。

---

# 70. Intelligence Model

建立统一：

DomainIntelligence

包含：

identity
infrastructure
dns
http
tls
technology
security
reputation
subdomains
relationships
history
score

这是前端和 API 的核心数据模型。

---

# 71. PDF

以后：

Export PDF

内容：

Overview
Score
Identity
Infrastructure
Technology
Security
Reputation
Relationships
Changes

第一版可以预留接口。

---

# 72. AI Summary

暂时不要依赖 AI 才能生成报告。

以后增加：

/api/v1/domains/:domain/summary

输入：

DomainIntelligence

输出：

Executive Summary

例如：

- Infrastructure overview
- Technology overview
- Security findings
- Significant changes
- Risk explanation

AI 不允许增加原始数据。

---

# 73. 产品文案

首页不要写：

“全球领先”
“最强”
“革命性”

保持专业。

推荐：

Understand websites.
Discover infrastructure.
Investigate relationships.
Track changes.

---

# 74. 不要做的事情

第一阶段禁止：

- 用户社区
- 评论
- 博客 CMS
- 新闻
- 社交功能
- 复杂后台
- AI Chat
- 多语言
- 大量营销页面
- 虚假的数据
- 虚假的实时状态
- 大量假 API

先把核心 Investigation 做好。

---

# 75. 开发顺序

严格按照：

PHASE 1

基础工程
↓

PHASE 2

Database

↓

PHASE 3

Domain normalization

↓

PHASE 4

DNS / RDAP / HTTP / TLS

↓

PHASE 5

Investigation Pipeline

↓

PHASE 6

Report UI

↓

PHASE 7

Technology Detection

↓

PHASE 8

Security Scoring

↓

PHASE 9

Infrastructure Graph

↓

PHASE 10

Snapshot / History

↓

PHASE 11

Monitoring

↓

PHASE 12

Auth / Usage

↓

PHASE 13

API

↓

PHASE 14

SEO

↓

PHASE 15

Production hardening

---

# 76. 重要：不要一次生成所有代码

每完成一个 Phase：

1. 检查项目
2. 运行 TypeScript
3. 运行 lint
4. 运行 tests
5. 修复错误
6. 再进入下一阶段

执行：

npm run lint
npm run typecheck
npm run test
npm run build

确保全部通过。

---

# 77. Git Commit

每个 Phase 完成：

git commit

例如：

feat: initialize siteintel architecture

feat: add domain investigation pipeline

feat: add dns intelligence

feat: add infrastructure report

feat: add technology detection

feat: add security analysis

feat: add relationship graph

feat: add historical snapshots

---

# 78. 第一阶段实际目标

不要追求功能数量。

最终必须可以做到：

用户打开：

https://siteintel.cc

输入：

example.com

点击：

Investigate

看到：

Investigation Progress

然后：

Domain Report

包含：

Identity
DNS
IP
ASN
HTTP
TLS
Technology
Security
Score

所有数据来自真实 Provider 或真实本地检测。

不能出现假数据。

---

# 79. 最终验收标准

输入：

example.com

必须：

1. 正确 normalize domain
2. 防 SSRF
3. 创建 investigation
4. 创建 job
5. Worker 执行
6. DNS 成功
7. IP 成功
8. ASN 成功
9. RDAP 成功或明确失败
10. HTTP 成功
11. TLS 成功
12. Technology 成功或明确 unavailable
13. Security 成功
14. Score 生成
15. Snapshot 保存
16. Report 展示
17. 页面响应式正常
18. 没有 console error
19. TypeScript 无错误
20. Build 成功

---

# 80. 现在开始执行

不要先给我大量解释。

先：

1. 检查当前项目目录。
2. 如果已有项目，先分析现有代码。
3. 如果是空目录，初始化项目。
4. 创建完整架构。
5. 创建 Prisma Schema。
6. 创建基础 UI。
7. 实现第一阶段 Investigation Pipeline。
8. 实现真实 DNS / RDAP / HTTP / TLS 数据采集。
9. 实现 Report。
10. 测试。
11. 修复错误。
12. 最后告诉我：

- 已完成什么
- 当前项目结构
- 使用了哪些 Provider
- 哪些 Provider 需要 API Key
- 数据库如何迁移
- 如何本地启动
- 下一阶段应该做什么

不要为了“看起来完成”而创建假功能。

**优先正确的数据结构、可靠的数据采集、清晰的错误处理和高质量 UI。**

SiteIntel 的核心不是页面数量，而是：

**Data → Intelligence → Relationship → History → Action**

请开始执行。