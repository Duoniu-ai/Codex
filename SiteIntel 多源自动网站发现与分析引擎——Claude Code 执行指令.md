# SiteIntel 多源自动网站发现与分析引擎
## Claude Code + DeepSeek V4 Pro 执行指令

你现在需要对 SiteIntel（https://www.siteintel.cc/）现有的“自动繁衍/自动调查”机制进行一次产品级升级。

这不是简单增加一个数据源。

我要把 SiteIntel 的自动繁衍重新定义为：

> **多源自动网站发现与持续分析引擎。**

最终目标是：

> 在有限服务器分析资源下，持续获得大量高质量、可访问、具有数据价值的独立网站，并让这些网站的分析结果继续发现新的候选网站，从而形成长期自动数据增长闭环。

---

# 一、首先理解我要解决的问题

当前 SiteIntel 已经具备网站调查、实体、关系、快照等基础能力。

但目前自动繁衍存在明显问题：

- 数据源太少
- 初始网站数量有限
- 自动繁衍容易停止
- 新发现域名没有形成稳定的分析队列
- 不同域名没有进行合理优先级排序
- 无法访问的网站可能浪费分析资源
- 子域名、API、CDN 等可能抢占真正网站的分析资源
- 分析能力没有最大化转化为“新增独立网站数据”
- 数据冷启动问题明显

因此，本次任务不是简单修复一个 Bug。

而是建立一套：

# 多源网站种子 → 候选池 → 智能排序 → 持续分析 → 新候选 → 再分析

的数据增长机制。

---

# 二、不要把 URIRANK 当成唯一来源

我提到：

https://urirank.com/

只是因为它可以提供大量网站排名数据。

它只是第一批 Seed Source。

SiteIntel 必须设计成：

> **可插拔、多源的网站种子与候选发现系统。**

第一阶段优先研究并接入以下来源：

## 1. URIRANK

用途：

- 全球网站排名
- 国家/地区网站排名
- 网站主域名种子

---

## 2. Cloudflare Radar

重点研究：

- Global Top Domains
- Country Top Domains
- Top 100
- Top 1K
- Top 10K
- Top 100K
- Top 1M
- Trending Domains

用途：

> 高质量全球网站种子 + 国家地区网站种子 + 趋势网站发现。

官方数据文档：

https://developers.cloudflare.com/radar/investigate/domain-ranking-datasets/

---

## 3. Majestic Million

重点研究：

- Top 1 Million Domains

用途：

> 从 Web 链接生态角度提供另一套大型网站种子。

官方：

https://majestic.com/reports/majestic-million

---

## 4. Common Crawl

Common Crawl 不要简单当成排行榜。

重点研究：

- URL Index
- Host/Domain 数据
- Web Graph
- Domain Nodes
- Domain Edges

用途：

> 大规模长尾网站发现 + 网站关系发现。

官方：

https://commoncrawl.org/

---

# 三、架构不能写死数据源

不要出现这种设计：

```text
if source == "urirank"
```

然后以后每增加一个数据源就修改大量代码。

应该抽象成：

```text
SeedSource
```

例如：

```text
SeedSource
├── URIRankSource
├── CloudflareRadarSource
├── MajesticSource
├── CommonCrawlSource
└── FutureSource
```

所有来源统一输出标准化 Candidate。

例如：

```text
CandidateDomain
{
    domain,
    source,
    source_rank,
    source_country,
    source_category,
    discovered_at,
    metadata
}
```

具体字段以现有项目架构为准，不要机械照搬。

---

# 四、建立统一 Candidate Pool

所有来源的数据不能直接进入 Investigation。

必须经过：

```text
External Sources
       ↓
Candidate Pool
```

Candidate Pool 是自动繁衍的核心缓冲层。

它负责：

- 标准化
- 去重
- 来源合并
- 可访问性状态
- 优先级
- 分析状态
- 重试状态
- 失败记录
- 数据价值
- 繁殖潜力

---

# 五、域名必须标准化

例如：

```text
https://example.com/
http://example.com
https://www.example.com
www.example.com
```

需要正确归一化。

同时区分：

```text
Root / Apex Domain
Subdomain
URL
```

例如：

```text
example.com
www.example.com
api.example.com
cdn.example.com
blog.example.com
```

不能全部当成五个完全独立的网站。

---

# 六、自动分析必须“主域名优先”

这是本次需求的核心原则之一。

如果发现：

```text
example.com
www.example.com
api.example.com
cdn.example.com
blog.example.com
```

默认优先：

```text
example.com
```

而不是：

```text
cdn.example.com
api.example.com
```

推荐逻辑：

```text
P0
Apex / Root Domain

P1
www 主站

P2
明显独立网站性质的子域名

P3
普通子域名

P4
API / CDN / Static / Image / Tracking / Infrastructure
```

但不要简单依靠字符串判断。

需要结合：

- DNS
- HTTP
- 页面内容
- Content-Type
- Title
- Redirect
- Technology
- hostname pattern

判断。

---

# 七、增加轻量级网站可访问性探测

这是非常重要的优化。

不要让完整网站分析任务去浪费在大量打不开的网站上。

在进入完整 Investigation 前，增加轻量探测：

```text
DNS
↓
TCP / TLS
↓
HTTP / HTTPS
↓
Status Code
↓
Redirect
↓
Response Time
↓
基础页面可获取性
```

目标不是完整分析。

目标只是回答：

> “这个网站现在是否值得消耗完整分析资源？”

---

# 八、优先分析可以打开的网站

这是自动分析调度的核心。

例如：

```text
A.com → HTTP 200
B.com → HTTP 200
C.com → HTTP 403
D.com → Timeout
E.com → DNS Failed
F.com → HTTP 200
```

默认应该：

```text
A
B
F
```

优先于：

```text
C
D
E
```

但：

> 无法访问 ≠ 永久删除。

必须保留：

```text
last_probe_at
last_status
failure_count
last_failure_reason
next_retry_at
```

失败网站降低优先级。

未来重新 Probe。

---

# 九、建议建立透明的 Priority Score

第一阶段：

**不要引入复杂 AI 决策。**

先建立可解释、可调节的规则评分系统。

核心因素：

```text
Accessibility
+
Root Domain Priority
+
New Domain Bonus
+
External Ranking Signal
+
Propagation Potential
+
Historical Success Rate
+
Data Diversity
-
Recent Analysis Penalty
-
Failure Penalty
-
Duplicate Penalty
```

具体权重由你结合现有数据库和代码结构设计。

但必须：

> 每个候选网站都能解释“为什么它排在这里”。

---

# 十、可访问性应该是高权重因素

可以参考：

```text
HTTP 200 / 2xx
高

HTTP 301 / 302
较高

HTTP 403
中低

HTTP 401
中低

HTTP 429
低

Timeout
很低

DNS Failure
很低

Connection Failure
很低
```

具体分值不要机械照搬。

请根据实际业务测试后设计。

---

# 十一、外部排名成为“信号”，不是绝对规则

例如：

```text
google.com
```

可能同时来自：

```text
URIRANK
Cloudflare Radar
Majestic
Common Crawl
```

不要产生四条网站记录。

应该：

```text
google.com
```

只有一个 Domain Entity。

但保留多个来源信号：

```text
source_count
urirank_rank
cloudflare_rank
majestic_rank
commoncrawl_seen
```

这样以后可以形成综合网站价值评分。

---

# 十二、建立“繁殖潜力”

一个网站分析之后可能发现：

- 新域名
- IP
- ASN
- DNS
- SSL
- CNAME
- Technology
- 关联网站
- 页面链接
- 第三方服务
- Web Graph 关系

如果一个网站历史上能产生大量高质量候选：

> 它的 Propagation Potential 应该提高。

这样系统可以优先分析：

> “既容易访问，又容易产生下一批数据”的网站。

这比单纯追求网站排名更符合 SiteIntel 的产品目标。

---

# 十三、不要让系统只分析大站

例如不能永远：

```text
Google
Microsoft
Apple
Amazon
Meta
```

不断围绕大站繁殖。

需要加入：

# Data Diversity

目标是扩大：

- 国家
- 地区
- 行业
- 网站类型
- 技术栈
- 企业规模

例如：

```text
科技
电商
媒体
教育
金融
政府
工具
博客
社区
开发者
企业
内容网站
```

这样未来 SiteIntel 的：

- 网站画像
- 技术情报
- 网站对比
- 关联分析
- 聚类
- 趋势分析

才有真正的数据基础。

---

# 十四、分析资源采用持续消费模式

当前我希望：

> **一个网站完整分析平均约 2 分钟。**

因此不要采用：

```text
每天一次批量分析
```

而应该采用：

```text
Candidate Pool
       ↓
Priority Queue
       ↓
Worker
       ↓
持续消费
```

只要候选池存在合格候选：

```text
Worker
↓
取最高优先级
↓
完整分析
↓
完成
↓
立即取下一个
```

目标：

> 在现有服务器能力允许的情况下，尽量接近每约 2 分钟完成一个网站分析。

不要为了达到“2 分钟”而牺牲系统稳定性。

实际吞吐量必须通过生产数据验证。

---

# 十五、自动繁衍真正应该形成这个闭环

```text
              ┌──────────────────────────────┐
              │        External Seeds         │
              │                              │
              │ URIRANK                      │
              │ Cloudflare Radar              │
              │ Majestic Million              │
              │ Common Crawl                  │
              │ Future Sources                │
              └──────────────┬───────────────┘
                             ↓
                      Candidate Pool
                             ↓
                    Domain Normalization
                             ↓
                       Deduplication
                             ↓
                     Root Domain Priority
                             ↓
                    Lightweight Probe
                             ↓
                     Priority Scoring
                             ↓
                     Analysis Queue
                             ↓
                     Site Investigation
                             ↓
                  Entity / Relationship
                             ↓
                    New Domain Discovery
                             ↓
                       Candidate Pool
                             ↓
                          循环
```

这才是 SiteIntel 的自动数据增长飞轮。

---

# 十六、不要忘记用户主动输入

用户输入的网站必须拥有更高优先级。

建议：

```text
用户主动分析
    ↓
最高优先级

SiteIntel 自动繁衍
    ↓
次高优先级

外部 Seed
    ↓
普通优先级
```

不能因为自动繁衍正在运行，就让用户提交的网站排队很久。

---

# 十七、建立来源质量体系

以后每个候选都需要知道：

```text
这个网站是从哪里来的？
```

例如：

```text
source = user
source = urirank
source = cloudflare_radar
source = majestic
source = commoncrawl
source = siteintel_discovery
source = related_domain
source = certificate
source = ip
source = dns
source = technology
```

同时记录：

```text
first_discovered_at
last_seen_at
source_count
```

这样 SiteIntel 自己的数据网络才能逐渐形成。

---

# 十八、必须考虑数据规模

不要一次把几百万甚至千万候选全部塞入：

```text
analysis queue
```

正确方式：

```text
External Dataset
        ↓
Incremental Import
        ↓
Candidate Pool
        ↓
按需 Probe
        ↓
合格候选
        ↓
Priority Queue
```

必须控制：

- 数据库增长
- Queue 增长
- Probe 请求
- 分析任务
- 外部数据导入
- 去重成本

---

# 十九、不要破坏现有 SiteIntel

当前项目已经完成过生产验收。

因此本次开发必须：

1. 先完整阅读现有架构；
2. 找到现有自动繁衍代码；
3. 找到 Investigation Pipeline；
4. 找到 Queue / Worker；
5. 找到现有 Domain / Candidate 数据结构；
6. 判断哪些能力已经存在；
7. 能复用就复用；
8. 尽量增量改造；
9. 不重复建立已有能力；
10. 不创建概念重复的数据表。

特别注意：

不要因为这个需求重新创建一套完全独立的：

```text
domains
events
signals
insights
snapshots
investigations
```

必须先映射现有模型。

---

# 二十、第一阶段必须先做白盒审计

开始编码之前：

### 不要立即修改代码。

先输出：

## 《SiteIntel 多源自动网站发现与分析引擎——现状审计报告》

必须回答：

1. 当前自动繁衍在哪里实现？
2. 当前 Candidate 是什么？
3. 当前 Domain 是什么？
4. 当前 Investigation 是什么？
5. 当前 Queue 是什么？
6. 当前 Worker 是什么？
7. 当前有哪些外部数据源？
8. 当前能否直接加入 Seed Source 抽象？
9. 当前是否已有 Probe？
10. 当前是否已有 Domain Normalization？
11. 当前是否已有 Dedup？
12. 当前是否已有 Priority？
13. 当前是否已有 Retry？
14. 当前是否已有失败记录？
15. 当前是否已有繁殖深度限制？
16. 当前调查平均耗时是多少？
17. 当前每天实际可以分析多少网站？
18. 当前自动繁衍为什么停止？
19. 当前哪些能力可以直接复用？
20. 最小改造方案是什么？

---

# 二十一、然后再提出实施方案

审计完成以后再设计：

### Phase 1

恢复现有自动繁衍。

### Phase 2

建立统一 Candidate Pool。

### Phase 3

接入多源 Seed：

- URIRANK
- Cloudflare Radar
- Majestic Million
- Common Crawl

### Phase 4

加入：

- Root Domain Priority
- Accessibility Probe
- Priority Score
- Retry
- Failure Backoff

### Phase 5

持续 Queue + Worker。

### Phase 6

加入：

- Propagation Potential
- Data Diversity
- External Ranking Signals

### Phase 7

建立自动数据增长 Dashboard。

---

# 二十二、必须建立可观测性

后台至少能够看到：

```text
Candidate Total
Candidate Pending
Candidate Probed
Candidate Accessible
Candidate Inaccessible

Queue Pending
Queue Running
Queue Success
Queue Failed

Investigations Today
Investigations / Hour
Average Investigation Duration

New Domains Today
New Entities Today
New Relationships Today

Propagation Success Rate
Candidate Acceptance Rate
Accessibility Rate
```

尤其要能够回答：

> **过去 24 小时 SiteIntel 到底新增了多少独立网站？**

这是比“调查次数”更重要的指标。

---

# 二十三、核心 KPI

本次升级不要把：

> Investigation Count

作为唯一 KPI。

核心 KPI 应该是：

### 1. 新增独立网站数量

### 2. 每小时成功分析网站数

### 3. 网站分析成功率

### 4. 可访问候选比例

### 5. 候选 → Investigation 转化率

### 6. Investigation → 新候选转化率

### 7. 平均分析耗时

### 8. 自动繁衍持续时间

### 9. 每个网站平均产生的新候选数量

### 10. 数据来源多样性

---

# 二十四、最重要的产品目标

请始终记住：

> SiteIntel 自动繁衍不是为了让机器“无限分析”。

而是：

> **让有限的分析资源尽可能持续转化成新的、高质量、可访问、独立的网站数据。**

所以：

```text
不是：
发现一个 → 分析一个

而是：
发现大量
↓
筛选
↓
排序
↓
优先分析最值得分析的
↓
产生更多候选
↓
继续筛选
↓
继续分析
```

---

# 二十五、最终要求

现在开始：

## 第一步

只做白盒审计。

## 第二步

生成：

```text
SITEINTEL_MULTI_SOURCE_DISCOVERY_AUDIT.md
```

## 第三步

给出：

```text
现状
↓
问题
↓
根因
↓
架构方案
↓
数据模型方案
↓
调度方案
↓
评分方案
↓
数据源接入方案
↓
实施阶段
↓
风险
↓
验收标准
```

## 第四步

等待确认后再开始代码修改。

---

# 最终原则

**不要为了这个需求推翻 SiteIntel。**

优先：

> 在现有 SiteIntel 架构基础上，把“自动繁衍”升级成“多源自动网站发现与分析引擎”。

重点是：

**多源 → 大量候选 → 主域名优先 → 可访问性优先 → 智能排序 → 持续分析 → 新候选 → 循环增长。**

先审计，后设计，最后实施。

禁止跳过审计直接重构。