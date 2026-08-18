# SiteIntel.cc — Claude Code Master Prompt

## 你的角色

你现在不是一个单纯的前端开发助手。

你是：

- Senior Product Architect
- Full Stack Engineer
- Data Platform Architect
- Data Intelligence Engineer
- UI/UX Product Designer
- Database Architect
- Systems Designer

你的任务是从零开始设计和开发一个完整的网站：

# SiteIntel.cc

## Website Data Intelligence Platform

产品定位：

> Analyze websites. Connect data. Discover insights.

中文：

> 分析网站，关联数据，发现洞察。

---

# 一、最高优先级：理解产品本质

在开始编写任何代码之前，你必须先理解：

**SiteIntel 不是传统的网站工具集合。**

它不是简单的：

- WHOIS 查询
- DNS 查询
- IP 查询
- SSL 查询
- HTTP Header 查询
- 网站技术栈检测
- SEO 工具
- 网络工具箱

这些功能可以存在。

但是它们只是：

> SiteIntel 的数据来源和基础能力。

SiteIntel 真正的核心是：

```text
用户输入：

Domain
Website
URL

↓

SiteIntel 自动进行：

Data Collection
数据采集

↓

Data Processing
数据处理

↓

Data Normalization
数据标准化

↓

Entity Extraction
实体提取

↓

Relationship Analysis
关系分析

↓

Snapshot Comparison
历史快照对比

↓

Event Detection
事件检测

↓

Insight Generation
洞察生成

↓

最终输出：

Website Intelligence Report
网站情报报告
```

---

# 二、SiteIntel 最终解决的问题

用户输入：

```text
example.com
```

SiteIntel 不应该只返回：

```text
IP:
1.2.3.4

DNS:
Cloudflare

SSL:
Let's Encrypt

Technology:
Next.js
```

这只是原始数据。

真正的产品应该继续回答：

### 1. 这个网站是什么？

例如：

- 网站类型
- 网站主要用途
- 所属行业
- 产品类型
- 商业模式推测
- 内容结构
- 技术特征

注意：

所有自动推测都必须明确标记：

```text
Inference
推测

Confidence
置信度

Evidence
证据
```

不能把推测伪装成事实。

---

### 2. 这个网站是如何构建的？

分析：

```text
Frontend

Backend

Framework

CMS

Analytics

CDN

Hosting

Database

Authentication

Payment

Third-party Services
```

最终形成：

```text
Website Technology Profile
网站技术画像
```

---

### 3. 它背后使用了什么基础设施？

包括：

```text
Domain

Nameserver

DNS Provider

IP

ASN

ISP

Hosting Provider

Cloud Provider

CDN

SSL Certificate

Email Provider
```

并且建立：

```text
Infrastructure Graph
基础设施关系图
```

例如：

```text
example.com
    │
    ├── Cloudflare
    │
    ├── AWS
    │
    ├── Vercel
    │
    └── Google Analytics
```

---

### 4. 它和哪些数据存在关联？

SiteIntel 必须逐步建立实体关系。

核心实体：

```text
Website
Domain
Subdomain
IP
ASN
Organization
DNS Provider
Hosting Provider
Cloud Provider
CDN
SSL Certificate
Technology
Tracking ID
Analytics Provider
Email Provider
```

关系示例：

```text
Website
   │
   ├── resolves_to → IP
   │
   ├── hosted_by → Provider
   │
   ├── protected_by → CDN
   │
   ├── uses → Technology
   │
   ├── shares → SSL Certificate
   │
   └── belongs_to → Organization
```

未来系统应该能够回答：

```text
这个 IP 被哪些网站使用？

这些网站是否共享相同 CDN？

哪些网站使用相同 Tracking ID？

哪些网站使用相同 SSL Certificate？

这些网站是否可能属于同一个组织？

某个基础设施变化影响了哪些网站？
```

注意：

关系必须基于真实可验证数据。

对于推测性关系必须区分：

```text
Observed Relationship
观察到的关系

Probable Relationship
可能关系

Inferred Relationship
推断关系
```

---

# 三、核心产品模块

整个 SiteIntel 产品分为五个核心系统。

```text
ANALYZE
```

网站分析。

```text
DISCOVER
```

数据发现。

```text
INTELLIGENCE
```

数据关联。

```text
INSIGHTS
```

自动洞察。

```text
MONITOR
```

持续监控。

---

# 四、ANALYZE — 网站分析系统

用户输入：

```text
domain.com
```

系统创建：

```text
Investigation
```

一次完整的网站分析任务。

分析流程：

```text
Validate Target
      ↓
Normalize Domain
      ↓
Create Investigation
      ↓
Collect Data
      ↓
Store Raw Data
      ↓
Normalize Data
      ↓
Extract Entities
      ↓
Create Relationships
      ↓
Create Snapshot
      ↓
Compare History
      ↓
Detect Events
      ↓
Generate Insights
      ↓
Generate Intelligence Report
```

---

# 五、数据采集系统

必须设计为：

```text
Provider Adapter Architecture
```

不要把所有 API 和采集逻辑直接写死。

设计统一接口。

例如：

```typescript
interface DataProvider {
  name: string

  collect(target: Target): Promise<CollectionResult>
}
```

Provider 示例：

```text
DNS Provider

WHOIS Provider

IP Intelligence Provider

ASN Provider

SSL Provider

HTTP Provider

Technology Detection Provider

Screenshot Provider

Website Metadata Provider
```

目录结构建议：

```text
src/
 ├── providers/
 │
 │   ├── dns/
 │   ├── whois/
 │   ├── ip/
 │   ├── ssl/
 │   ├── http/
 │   ├── technology/
 │   ├── metadata/
 │   └── screenshot/
```

每个 Provider：

```text
Input
↓
Collection
↓
Raw Response
↓
Validation
↓
Normalization
↓
Normalized Result
```

原始数据必须保留。

---

# 六、核心数据架构

整个 SiteIntel 的数据架构必须遵循：

```text
RAW DATA
```

原始数据。

↓

```text
NORMALIZED DATA
```

标准化数据。

↓

```text
ENTITIES
```

实体。

↓

```text
RELATIONSHIPS
```

关系。

↓

```text
SNAPSHOTS
```

快照。

↓

```text
EVENTS
```

变化事件。

↓

```text
INSIGHTS
```

自动洞察。

---

# 七、数据库设计

使用：

```text
PostgreSQL
```

数据库设计必须支持长期扩展。

不要把所有数据塞进：

```text
websites
```

一张表。

建议核心结构：

```text
users
projects
targets
investigations
```

---

## Raw Data

```text
raw_collections
```

字段建议：

```text
id

investigation_id

provider

data_type

target

raw_data JSONB

status

collected_at

expires_at
```

---

## Entities

建立统一实体表。

```text
entities
```

字段：

```text
id

entity_type

entity_value

normalized_value

metadata JSONB

first_seen_at

last_seen_at

created_at
```

实体类型：

```text
domain

subdomain

ip

asn

organization

provider

technology

ssl_certificate

tracking_id

email_provider
```

---

## Relationships

```text
entity_relationships
```

字段：

```text
id

source_entity_id

target_entity_id

relationship_type

confidence

evidence JSONB

first_seen_at

last_seen_at

status
```

relationship_type：

```text
resolves_to

hosted_by

uses

protected_by

shares

belongs_to

redirects_to

associated_with
```

---

## Snapshots

```text
snapshots
```

字段：

```text
id

target_id

snapshot_type

data JSONB

hash

created_at
```

快照类型：

```text
dns

ip

infrastructure

technology

ssl

website_metadata

full
```

---

## Events

```text
events
```

字段：

```text
id

target_id

event_type

previous_value

current_value

evidence JSONB

detected_at

severity
```

event_type：

```text
ip_changed

dns_changed

nameserver_changed

ssl_changed

provider_changed

technology_added

technology_removed

website_status_changed
```

---

## Insights

```text
insights
```

字段：

```text
id

target_id

insight_type

title

summary

details

confidence

importance

evidence JSONB

status

first_detected_at

last_detected_at
```

---

# 八、最重要的系统：Insight Engine

Insight Engine 是 SiteIntel 的核心竞争力。

绝对不能只是：

```text
IP changed
```

系统应该进一步分析：

```text
IP changed

+
CDN changed

+
SSL certificate changed

+
Hosting provider changed

↓

Infrastructure Migration Detected
```

---

## Insight Engine 输入

```text
Current Snapshot

Historical Snapshots

Detected Events

Entity Relationships

Infrastructure Data

Technology Data
```

---

## Insight Engine 输出

统一格式：

```json
{
  "type": "infrastructure_migration",
  "title": "Possible Infrastructure Migration Detected",
  "summary": "Multiple infrastructure components changed within a short period.",
  "confidence": 0.87,
  "importance": "high",
  "evidence": [],
  "first_detected_at": ""
}
```

---

## 洞察必须分级

### Critical

可能严重影响：

```text
Website Offline

DNS Failure

Certificate Expired

Major Infrastructure Change
```

---

### High

值得重点关注：

```text
Hosting Migration

CDN Migration

Major Technology Change
```

---

### Medium

正常变化：

```text
IP Changed

SSL Renewed

New Technology Detected
```

---

### Low

一般信息：

```text
Minor Metadata Change
```

---

# 九、Insight Rules Engine

第一阶段不要过度依赖 AI。

优先建立：

```text
Rule-based Insight Engine
```

规则示例：

```text
IF

ip_changed

AND

ssl_changed

WITHIN 7 days

THEN

possible_infrastructure_migration
```

---

```text
IF

nameserver_changed

AND

cdn_provider_changed

THEN

possible_dns_cdn_migration
```

---

```text
IF

technology_removed

AND

technology_added

WITHIN 30 days

THEN

possible_technology_stack_change
```

---

每个 Insight 必须包含：

```text
Observation

Interpretation

Evidence

Confidence

Time Range
```

例如：

```text
Observation

The website changed IP addresses three times
within 14 days.

Interpretation

This may indicate infrastructure migration
or traffic routing changes.

Confidence

78%

Evidence

IP change detected on:
2026-08-01
2026-08-07
2026-08-12
```

---

# 十、Evidence System

所有洞察必须可追溯。

禁止：

```text
AI says:
This website migrated to AWS.
```

除非存在证据。

正确方式：

```text
Possible AWS Infrastructure Migration

Confidence
82%

Evidence

• IP ASN changed to Amazon
• CDN fingerprint changed
• SSL infrastructure changed
```

因此建立：

```text
Evidence Chain
```

结构：

```text
Insight
    ↓
Event
    ↓
Snapshot
    ↓
Entity
    ↓
Raw Collection
```

用户应该能够点击：

```text
View Evidence
```

查看洞察依据。

---

# 十一、Confidence System

所有推断必须使用统一置信度。

范围：

```text
0 — 100
```

建议：

```text
90-100
High Confidence

70-89
Strong Signal

40-69
Possible

0-39
Weak Signal
```

Confidence 计算必须可解释。

例如：

```text
Base Score
+
Multiple Evidence Bonus
+
Historical Consistency
-
Conflicting Evidence
```

不要使用随机值。

---

# 十二、Website Intelligence Report

每个网站都应该生成一个完整情报报告。

页面结构：

```text
Website Overview
```

```text
What is this website?
```

AI / Rules 根据公开数据生成简短描述。

---

```text
Key Insights
```

展示最重要的：

```text
Infrastructure Migration

Technology Change

Certificate Change

Suspicious Change Pattern

New Provider Detected
```

---

```text
Infrastructure
```

展示：

```text
IP

ASN

Hosting

Cloud

CDN

DNS

Nameserver

SSL
```

---

```text
Technology
```

展示：

```text
Frontend

Backend

CMS

Analytics

Tag Manager

Framework

Third-party Services
```

---

```text
Relationships
```

展示实体关系。

例如：

```text
Domain
   ↓
IP
   ↓
ASN
   ↓
Organization
```

---

```text
History
```

展示：

```text
Timeline
```

例如：

```text
Aug 12

IP Changed

Aug 10

SSL Certificate Renewed

Aug 05

New Technology Detected
```

---

```text
Evidence
```

所有重要结论必须查看原始证据。

---

# 十三、DISCOVER 系统

未来支持主动发现。

例如：

```text
Find websites sharing this IP
```

```text
Find domains using this Nameserver
```

```text
Find websites using this technology
```

```text
Find related infrastructure
```

第一阶段可以先实现有限版本。

不要一开始尝试构建全球互联网扫描系统。

---

# 十四、INTELLIGENCE 系统

建立：

```text
Entity Graph
```

核心逻辑：

```text
Website
     │
     ├──── IP
     │       │
     │       └── ASN
     │
     ├──── Technology
     │
     ├──── SSL Certificate
     │
     ├──── DNS Provider
     │
     └──── Organization
```

关系数据必须可以长期积累。

未来可以扩展：

```text
Shared Infrastructure

Related Websites

Provider Clusters

Technology Clusters

Organization Networks
```

---

# 十五、MONITOR 系统

用户可以监控：

```text
Domain

Website

Infrastructure

Technology
```

监控周期：

```text
Daily

Weekly

Monthly
```

第一阶段可以只实现：

```text
Daily
```

监控系统：

```text
Scheduled Job

↓

Collect Data

↓

Create Snapshot

↓

Compare Snapshot

↓

Detect Event

↓

Generate Insight

↓

Notify User
```

---

# 十六、产品页面架构

---

## 1. Homepage

目标：

让用户立即理解：

> SiteIntel 能分析一个网站，并发现隐藏在数据中的变化和关联。

首页核心：

```text
Hero

Enter a domain to investigate

[ example.com ]
[ Analyze ]
```

下方展示：

```text
What SiteIntel analyzes
```

五个模块：

```text
Website
Infrastructure
Technology
Relationships
Insights
```

---

## 2. Search / Investigate

用户输入：

```text
domain.com
```

创建：

```text
Investigation
```

实时展示分析进度：

```text
Validating domain

✓ DNS collected

✓ IP resolved

✓ SSL analyzed

✓ Technology detected

✓ Infrastructure analyzed

✓ Historical data compared

✓ Insights generated
```

可以使用：

```text
SSE
```

实现实时进度。

---

## 3. Website Intelligence Report

URL：

```text
/report/example.com
```

页面：

```text
Header

Domain
Status
Last Updated
Analyze Again
Monitor
```

---

### Section 1

```text
Overview
```

---

### Section 2

```text
Key Insights
```

必须成为视觉重点。

---

### Section 3

```text
Infrastructure
```

使用卡片。

---

### Section 4

```text
Technology
```

技术分类。

---

### Section 5

```text
Relationships
```

第一阶段可以：

```text
Relationship List
```

第二阶段：

```text
Interactive Graph
```

不要第一版就为了炫酷强行使用复杂图谱。

---

### Section 6

```text
History
```

Timeline。

---

### Section 7

```text
Evidence
```

展开查看。

---

# 十七、Dashboard

登录用户可以看到：

```text
My Investigations
```

```text
Monitored Websites
```

```text
Recent Changes
```

```text
Important Insights
```

---

# 十八、商业模式

不要把所有数据免费开放。

建议：

## Free

允许：

```text
Basic Website Analysis

Current Infrastructure

Basic Technology

Limited History

Limited Investigations
```

---

## Pro

提供：

```text
Full History

Advanced Insights

Relationship Discovery

More Investigations

Monitoring

Change Alerts
```

---

## Intelligence

面向专业用户：

```text
Advanced Relationship Analysis

Bulk Investigation

Export

Long-term History

Advanced Monitoring

Priority Collection
```

---

## API

提供：

```text
Website Intelligence API

Infrastructure API

Technology API

Insight API
```

---

# 十九、付费核心原则

免费展示：

```text
事实数据
```

例如：

```text
Current IP

Current DNS

Basic Technology
```

收费展示：

```text
历史

变化

关联

批量

监控

高级洞察
```

因为真正有持续价值的是：

```text
Time
History
Relationships
Insights
Monitoring
```

而不是单次查询。

---

# 二十、技术架构

推荐：

```text
Frontend

Next.js
TypeScript
Tailwind CSS
```

---

```text
Backend

Next.js API / Route Handlers

Server Actions where appropriate
```

---

```text
Database

PostgreSQL
```

建议：

```text
Prisma or Drizzle
```

在项目开始时选择一个，并保持统一。

---

```text
Cache

Redis
```

用于：

```text
API Cache

Rate Limit

Collection Cache

Temporary Job State
```

---

```text
Background Jobs

Queue / Worker Architecture
```

任务：

```text
DNS Collection

Technology Detection

Historical Comparison

Monitoring

Insight Generation
```

不要把耗时任务全部放进一次 HTTP Request。

---

# 二十一、推荐项目架构

```text
siteintel/
│
├── apps/
│
│   └── web/
│
├── packages/
│
│   ├── database/
│   │
│   ├── core/
│   │
│   ├── collectors/
│   │
│   ├── analyzers/
│   │
│   ├── intelligence/
│   │
│   └── ui/
│
├── workers/
│
│   ├── collector/
│   │
│   ├── analyzer/
│   │
│   └── monitor/
│
└── docs/
```

如果项目当前规模不需要 Monorepo：

不要为了架构复杂化强行使用。

可以使用：

```text
src/
 ├── app/
 ├── components/
 ├── lib/
 ├── providers/
 ├── collectors/
 ├── analyzers/
 ├── intelligence/
 ├── db/
 └── jobs/
```

优先选择：

> 当前开发效率最高，同时未来可以扩展的方案。

---

# 二十二、Collection Pipeline

所有采集任务统一进入 Pipeline。

```text
Target
   ↓
Job Created
   ↓
Provider Selection
   ↓
Data Collection
   ↓
Raw Storage
   ↓
Validation
   ↓
Normalization
   ↓
Entity Extraction
   ↓
Relationship Update
   ↓
Snapshot Creation
   ↓
Event Detection
   ↓
Insight Generation
```

---

# 二十三、Analyzer Architecture

不要把所有分析逻辑写在：

```text
analyzeWebsite()
```

里面。

拆分：

```text
analyzers/

infrastructureAnalyzer

technologyAnalyzer

relationshipAnalyzer

changeAnalyzer

insightAnalyzer
```

统一接口：

```typescript
interface Analyzer {
  name: string

  analyze(input: AnalysisContext): Promise<AnalysisResult>
}
```

---

# 二十四、Insight Engine Architecture

建议：

```text
intelligence/

rules/

scorers/

generators/

evidence/

types/
```

例如：

```text
rules/

detectInfrastructureMigration

detectTechnologyMigration

detectProviderChange

detectFrequentIPChange

detectCertificateChange
```

---

# 二十五、第一阶段 MVP

第一阶段绝对不要开发过多功能。

目标：

> 用户输入一个 Domain，可以获得真正有价值的网站情报报告。

MVP 必须包括：

```text
1.

Domain Input
```

```text
2.

DNS Analysis
```

```text
3.

IP Analysis
```

```text
4.

ASN Analysis
```

```text
5.

SSL Analysis
```

```text
6.

HTTP Analysis
```

```text
7.

Basic Technology Detection
```

```text
8.

Infrastructure Profile
```

```text
9.

Snapshot System
```

```text
10.

Basic Event Detection
```

```text
11.

Basic Insight Engine
```

```text
12.

Website Intelligence Report
```

---

# 二十六、MVP 首批 Insight Rules

至少实现：

### Rule 1

```text
IP Changed
```

---

### Rule 2

```text
Nameserver Changed
```

---

### Rule 3

```text
SSL Certificate Changed
```

---

### Rule 4

```text
Technology Added
```

---

### Rule 5

```text
Technology Removed
```

---

### Rule 6

```text
Infrastructure Migration Signal
```

规则：

```text
Multiple infrastructure events
within a short time window
```

---

### Rule 7

```text
Frequent Infrastructure Changes
```

规则：

```text
Multiple IP or provider changes
within a defined period
```

---

# 二十七、第二阶段

增加：

```text
Monitoring
```

```text
Historical Timeline
```

```text
Related Websites
```

```text
Entity Relationships
```

```text
Advanced Technology Detection
```

```text
User Accounts
```

```text
Dashboard
```

---

# 二十八、第三阶段

增加：

```text
Relationship Graph
```

```text
Bulk Investigation
```

```text
API
```

```text
Advanced Intelligence Rules
```

```text
AI Insight Explanation
```

注意：

AI 只负责：

```text
Explain
Summarize
Interpret
```

不应该直接制造事实。

事实必须来自：

```text
Collected Data

Events

Snapshots

Evidence
```

---

# 二十九、UI / UX 原则

设计风格：

```text
Professional

Data Intelligence

Modern

Technical

Minimal

High Information Density
```

不要设计成：

```text
普通工具站
```

也不要设计成：

```text
廉价 Dashboard Template
```

视觉感觉应该接近：

```text
Data Platform
+
Intelligence Platform
+
Developer Tool
```

---

# 三十、核心 UI 原则

用户访问报告后，必须在 5 秒内理解：

```text
这个网站是什么
```

```text
它的基础设施是什么
```

```text
最重要的变化是什么
```

```text
有什么值得关注
```

因此页面信息优先级：

```text
1.

Key Insights
```

↓

```text
2.

Website Overview
```

↓

```text
3.

Infrastructure
```

↓

```text
4.

Technology
```

↓

```text
5.

History
```

↓

```text
6.

Relationships
```

↓

```text
7.

Raw Data / Evidence
```

---

# 三十一、不要犯这些错误

禁止：

```text
先做漂亮 UI
再想数据结构
```

禁止：

```text
所有数据都放一个 JSON
```

禁止：

```text
Insight 没有 Evidence
```

禁止：

```text
AI 直接生成事实
```

禁止：

```text
所有采集 API 写死在业务代码
```

禁止：

```text
所有功能都同步请求
```

禁止：

```text
第一版就做全球互联网扫描
```

禁止：

```text
为了酷炫加入复杂关系图
但没有真实关系数据
```

禁止：

```text
把 SiteIntel 做成传统网络工具站
```

---

# 三十二、开发顺序

必须严格按照以下顺序。

## Step 1

先分析整个产品。

输出：

```text
Product Architecture
```

---

## Step 2

设计：

```text
Database Schema
```

包括：

```text
Entities

Relationships

Snapshots

Events

Insights
```

---

## Step 3

设计：

```text
Collection Architecture
```

---

## Step 4

实现：

```text
Provider Adapter System
```

---

## Step 5

实现：

```text
Normalization Layer
```

---

## Step 6

实现：

```text
Entity System
```

---

## Step 7

实现：

```text
Relationship System
```

---

## Step 8

实现：

```text
Snapshot System
```

---

## Step 9

实现：

```text
Event Detection
```

---

## Step 10

实现：

```text
Insight Engine
```

---

## Step 11

实现：

```text
Investigation Pipeline
```

---

## Step 12

最后才实现：

```text
Website Intelligence UI
```

---

# 三十三、Claude Code 工作方式

你必须采用：

```text
Architecture First
```

而不是：

```text
UI First
```

在每一个开发阶段：

### 1

先说明：

```text
Current Goal
```

---

### 2

说明：

```text
Architecture Decision
```

---

### 3

说明：

```text
Files to Create / Modify
```

---

### 4

开始开发。

---

### 5

完成后执行：

```text
Type Check
```

```text
Lint
```

```text
Build
```

---

### 6

如果发现问题：

必须主动修复。

不要只告诉我：

```text
There are errors.
```

你应该：

```text
Detect
↓
Diagnose
↓
Fix
↓
Test Again
```

---

# 三十四、代码质量要求

必须：

```text
TypeScript Strict Mode
```

禁止：

```typescript
any
```

除非确实无法避免。

所有核心系统必须：

```text
Strongly Typed
```

---

必须：

```text
Error Handling
```

---

必须：

```text
Logging
```

---

必须：

```text
Provider Failure Isolation
```

例如：

```text
SSL Provider Failed
```

不能导致：

```text
整个 Investigation Failed
```

系统应该：

```text
Partial Success
```

---

# 三十五、Mock Data 策略

在真实 Provider 没有配置 API Key 时：

允许：

```text
Development Mock Provider
```

但必须明确：

```text
mock
```

和：

```text
real
```

数据来源。

禁止：

```text
把 Mock Data 伪装成真实数据。
```

---

# 三十六、最终产品目标

最终用户输入：

```text
example.com
```

SiteIntel 经过完整 Pipeline：

```text
Collect
```

↓

```text
Normalize
```

↓

```text
Analyze
```

↓

```text
Connect
```

↓

```text
Compare
```

↓

```text
Detect
```

↓

```text
Generate Insights
```

最终得到：

# Website Intelligence Report

包括：

```text
What is this website?

How is it built?

What infrastructure does it use?

What technologies does it use?

What entities are connected?

What changed?

What patterns were detected?

What should the user pay attention to?
```

---

# 三十七、现在开始执行

现在不要直接生成几十个页面。

首先执行：

# PHASE 1 — Product & Architecture Foundation

完成以下任务：

```text
1. Analyze the current repository
```

```text
2. Identify the existing tech stack
```

```text
3. Identify reusable code
```

```text
4. Identify architecture problems
```

```text
5. Propose the target architecture
```

```text
6. Design the core data model
```

```text
7. Design the Investigation Pipeline
```

```text
8. Design the Provider Adapter System
```

```text
9. Design the Snapshot System
```

```text
10. Design the Event Detection System
```

```text
11. Design the Insight Engine
```

```text
12. Create a phased implementation plan
```

---

在开始修改大量代码之前：

先输出：

# SITEINTEL ARCHITECTURE PLAN

包含：

```text
1. Product Understanding

2. Current Repository Analysis

3. Target Architecture

4. Core Modules

5. Data Model

6. Collection Pipeline

7. Intelligence Pipeline

8. Database Schema

9. API Architecture

10. Background Jobs

11. UI Architecture

12. Implementation Phases
```

---

只有在完成：

```text
SITEINTEL ARCHITECTURE PLAN
```

之后。

再进入：

# PHASE 2 — Core Data Foundation

然后依次开发。

---

# 最重要的原则

永远记住：

SiteIntel 的核心不是：

> 查询数据。

而是：

> **理解数据。**

不是：

```text
IP = 1.2.3.4
```

而是：

```text
This website recently changed its IP,
CDN and SSL infrastructure within a short
period, indicating a possible infrastructure migration.
```

同时必须能够回答：

```text
Why?
```

并展示：

```text
Evidence.
```

---

# 最终一句话

Build SiteIntel not as a collection of website lookup tools.

Build it as a:

# Website Data Intelligence Platform

Where:

```text
Data becomes Information
```

```text
Information becomes Relationships
```

```text
Relationships become Intelligence
```

```text
Intelligence becomes Actionable Insights
```