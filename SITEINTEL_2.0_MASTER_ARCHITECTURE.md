# SiteIntel 2.0 Master Architecture（架构冻结）

> 文档性质：只读架构冻结（未修改任何代码/数据库/生产/服务器/Git 工作树）
> 日期：2026-08-18 ｜ 依据：真实代码（Duoniu-ai/siteintel @ 27bbf93）+ Schema + API + 测试（297 用例）+ 生产状态 + 既有规划/Gap Analysis/Phase 1 报告
> 状态标记：**EXISTING**（已实现且稳定）/ **PARTIAL**（部分实现）/ **PLANNED**（规划中，未实现）/ **RECOMMENDED**（建议设计）/ **DEPRECATED**（应淘汰）/ **CONFLICT**（语义冲突，待收敛）

---

# 1. 定义与核心闭环

**SiteIntel = Website Data Intelligence Platform（网站数据智能平台）。**

不是 SEO 工具集合、漏洞扫描器、IP/DNS 查询工具、技术栈查询站或报告生成器。用户输入一个网站只是入口，产品对象是持续演化的**数据资产**。

核心闭环（冻结）：

```text
Discover → Collect → Normalize → Understand → Relate → Observe
→ Detect → Explain → Recommend → Act → Verify → Monitor → Rediscover
```

当前真实状态：Discover（EXISTING，多源 Phase 0）→ Collect/Normalize/Understand/Relate/Observe/Detect/Explain（EXISTING）→ Recommend（PARTIAL，计算式不落库）→ Act/Verify（PLANNED）→ Monitor（EXISTING）→ Rediscover（PARTIAL，无价值回填）。

---

# 2. 分层架构总览

## 2.1 现状（EXISTING）

```text
Frontend（Next.js 16 App Router SSR + 少量客户端组件，双语）
  ↓
API Route Handlers（/api/*、/api/v1/*，SSE 进度流）
  ↓
Application/Domain Services（src/lib：providers、pipeline、discovery、
  intelligence、search、monitoring、seo、security/safe-fetch）
  ↓
Prisma → PostgreSQL 14（单体）
  ↓
systemd 单进程（:3003）+ nginx + Cloudflare；进程内 setInterval 调度器
```

## 2.2 目标分层（RECOMMENDED，演进不重写）

```text
Presentation（页面/组件；后续按区块拆分）
  ↓
API Layer（现有 route handlers；v1 收敛；未来 entity/relationship API）
  ↓
Application Service（pipeline、monitor、discovery、search sync）
  ↓
Domain Service（fact/entity/relationship/event/finding/insight/recommendation）
  ↓
Repository / Data Layer（Prisma + 类型化 Fact 校验 + Source Registry）
  ↓
PostgreSQL（结构化业务表 + 受约束 JSONB 载荷）
  ↓
Job Consumer（未来最小 Job/JobRun 统一调度；单进程内消费，先不引入队列）
```

原则：**保持模块化单体 + 单 worker**；任何新能力都从“能否复用现有底座”开始评估。

---

# 3. 六大产品领域冻结

## 3.1 Website Intelligence（网站是什么）

- 范围：Domain / URL / Identity / IP / ASN / Organization / DNS / CDN / Hosting / Certificate / Technology / Infrastructure / Pages / Geography。
- 现状：**EXISTING（核心）**——Target/Fact/Entity/Relationship/Snapshot/Event/Insight + 报告页 + 实体反查页（/ip /asn /certificate /organization /technology）。
- 真实实现：7 providers（dns/rdap/ip/ssl/http/metadata/technology）+ infrastructure analyzer + website-summary；报告读 Fact。
- 缺口：Page 级模型（PLANNED）；Geography 仅 IP 元数据（PARTIAL）；URL 规范化仅域名级（PARTIAL）。

## 3.2 Technology Intelligence（如何构建/运行）

- 现状：**PARTIAL**——指纹库 57+ 技术实体、分类、部分版本提取、共享技术反查页。
- 缺口：依赖清单/依赖树（PLANNED）、TechnologyVersion 实体（DERIVED，可由 tech+version 计算）、版本→已知漏洞（PLANNED，Phase 4 依赖）。

## 3.3 Security Intelligence（哪里存在风险）

- 现状：**PARTIAL（数据层）**——TLS 证书/有效期、5 项安全头存在性、技术/版本(有限)、被动洞察（tls_expiry/domain_expiry/status）；无 Finding/Vulnerability/Remediation 模型。
- 边界（冻结）：**Passive / Safe / Non-destructive**；禁止攻击平台化。
- 未来模型：SecurityFinding / SecurityEvidence / Vulnerability / Remediation / FixVerification（PLANNED，Phase 4 前不建表）。

## 3.4 SEO Intelligence（为什么搜索表现不足）

- 现状：**PARTIAL（Technical SEO）**——title/desc/H1/robots/sitemap/JSON-LD 计数/内链数 + 8 项 audit；无 canonical/noindex 采集；无 Keyword/Ranking/Intent/Opportunity。
- 未来模型：Keyword / KeywordObservation / KeywordRanking / SearchEngine / SearchIntent / KeywordOpportunity / ContentGap / ContentOpportunity（PLANNED，Phase 5 前不建表）。
- 定位（冻结）：SEO 是 Website Intelligence 的一个维度，不是独立工具。

## 3.5 Change Intelligence（发生了什么变化）

- 现状：**EXISTING**——12 类事件 + 13 条洞察规则 + 信号相关性 + 矛盾检测 + 统一时间线。
- 缺口：SEO/Security/Ranking 变化事件（PLANNED，随对应领域落地）；Event/Signal 双轨（CONFLICT，见 §5）。

## 3.6 Growth Intelligence（下一步做什么）

- 现状：**PARTIAL**——22 项 audit → 公式化推荐（P0/P1/P2）+ SWOT-lite + AI 解释（GLM）。
- 缺口：Recommendation/Action 持久化与生命周期、Action Center、验证回环（PLANNED，Phase 7）。

---

# 4. 统一数据底座（十二层职责冻结）

```text
Raw               原始采集（RawCollection，不可变，证据根）
Fact              规范化当前状态（Fact，每 target+type 唯一，报告唯一读源）
Entity            实体（Entity，type+normalized 唯一，幂等 upsert）
Relationship      关系（EntityRelationship，observed/probable/inferred）
Observation/Snapshot  时间点观测（Snapshot，aspect 级 hash 去重，历史存储）
Event             变化事件（Event，对外时间线）
Finding           状态化问题（SecurityFinding/SEOFinding，PLANNED）
Insight           规则解释（Insight，证据链 + 置信度）
Recommendation    行动建议（PLANNED）
Action            已跟踪任务（PLANNED）
Verification      修复验证（PLANNED）
```

每一层职责、输入、输出、持久化、生命周期在 Data Architecture 中详述。

---

# 5. Canonical Semantic Dictionary（概念消歧，冻结）

| 概念对 | 决议 | 状态 |
|---|---|---|
| Event vs Signal | Event = 对外变化事件（唯一时间线事实源）；Signal = 事实内部状态迁移日志（仅内部，不对外展示）。**Phase 1 收敛：以 Event 为准，Signal 降为内部过程记录**。 | CONFLICT → 收敛 |
| Snapshot vs Observation | Snapshot = 已持久化的 aspect 级时间点载荷（当前唯一实现）；Observation = 语义观测单元（概念层）。**现阶段不新增 Observation 表；Snapshot 即 Observation 的存储实现**。 | 兼容（无重复表） |
| Insight vs Hypothesis | Insight = 基于证据的规则解释；Hypothesis = 未证实的多事实推断。当前 Insight 内嵌 certainty/supporting/contradicting 字段 → **语义越界**。决议：Insight 只承载解释；假设语义保留字段但仅搜索类规则使用，未来如需一等公民再拆表。 | CONFLICT |
| Fact vs Observation | Fact = 规范化当前状态（唯一）；Observation = 原始/语义测量。当前 Fact + Evidence + RawCollection 已覆盖，无需新表。 | 清晰 |
| Finding vs Event | Finding = 状态化问题（open/fixed/verified/reopened/false_positive/accepted_risk）；Event = 单点变化。**未来 Finding 表由规则+证据生成，事件作为其变化历史**。 | PLANNED |
| Recommendation vs Insight | Recommendation = 可行动建议（priority/impact/effort/evidence/status）；Insight = 解释。当前推荐由 audit 计算不落库（PARTIAL）；未来独立表。 | PARTIAL |
| Action vs Recommendation | Action = 被跟踪执行的任务（状态/截止/验证）；Recommendation = 建议本身。未来 ActionItem 关联 Recommendation。 | PLANNED |
| Candidate vs Target | Candidate = 潜在目标队列行（可丢弃）；Target = 已分析主体。当前清晰（DiscoveryCandidate → Investigation → Target）。 | 清晰 |
| Investigation vs Job | Investigation = 对某 Target 的一次分析运行；Job = 通用任务包装。未来 Job 系统以 Job/JobRun 包裹 Investigation 及全部后台任务，Investigation 表保留。 | PLANNED |

完整定义（生命周期/持久化/可删/可重建/关系）见 Data Architecture §3。

---

# 6. 核心 Entity 设计（冻结）

| Entity | 类别 | 现状/决议 |
|---|---|---|
| Target | Core Existing | 分析主体（去 www），保留 |
| Website | Derived | 由 Target + 关联实体组合的语义视图，不建表 |
| Domain / Subdomain | Core Existing | Entity type=domain/subdomain |
| URL / Page | Core Future | Page 未来成为核心（SEO/内容维度）；URL 由 Page 派生 |
| IP | Core Existing | Entity type=ip（含 geo 元数据） |
| ASN | Core Existing | Entity type=asn（AS0 哨兵） |
| Organization | Core Existing | Entity type=organization + OrganizationAlias 治理 |
| Certificate | Core Existing | Entity type=ssl_certificate |
| Technology | Core Existing | Entity type=technology |
| TechnologyVersion | Derived | tech + version 组合，可计算，不独立建表（未来需要时再评估） |
| Nameserver | Core Existing | Entity type=nameserver（注意 SAN 派生噪声治理） |
| MX / EmailProvider | Derived/Existing | email_provider 实体已有；MX 记录在 Fact |
| CDN / HostingProvider | Derived | provider 实体 + metadata.kind（cdn/hosting/dns），不重复建表 |
| Country / Region / Location | Derived | 从 IP 元数据/ASN 组织派生；未来地理聚合再评估 |
| SearchEngine | Core Future | 枚举级实体（Baidu/Bing/Google/Shenma/Sogou） |
| Keyword | Core Future | 规范化关键词实体 |
| SecurityFinding / Vulnerability | Core Future | Phase 4 建表 |
| Recommendation / Action | Core Future | Phase 7 建表 |

原则：**不为未来功能提前建表**；Derived 一律计算；Temporary（progress/步骤/sync 状态）不进入核心模型。

---

# 7. Relationship 核心资产（冻结）

既有 9 类关系（EXISTING）：resolves_to / hosted_by / uses / protected_by / shares / belongs_to / redirects_to / associated_with。

未来语义对齐（PLANNED 映射，不新增表，仅规范 type 值）：

```text
Domain → resolves_to → IP          EXISTING
IP → belongs_to → ASN              EXISTING
ASN → belongs_to → Organization    EXISTING
Domain → uses → Technology         EXISTING
Domain → protected_by → CDN        EXISTING（provider kind=cdn）
Domain → has_certificate → Certificate  EXISTING（shares）
Certificate → contains → Domain    DERIVED（SAN 解析）
Domain → belongs_to → Organization EXISTING（registrar，inferred）
```

关系必须携带：type / direction / first_seen / last_seen / confidence / source / evidence / status（前 7 项 EXISTING；status 未来加：active/superseded，PLANNED）。

---

# 8. Evidence / Source / Confidence（冻结）

统一证据链：

```text
Source → Raw Data → Fact → Evidence → Entity/Relationship → Observation → Finding/Insight
```

- Source Registry（PLANNED，Phase 1 轻量表）：source/provider/category/endpoint/rate_limit/cost/priority/reliability/coverage/last_success/last_failure。
- Evidence（EXISTING，PARTIAL）：Fact 级证据已建；Relationship/Insight 证据为内嵌 JSON；rawValue 目前多为占位 → Phase 1 渐进补 source_url/raw_reference。
- Confidence（EXISTING，需统一展示层）：0-100 贯穿 fact/relationship/evidence/insight；未来加 source_count / source_quality / conflict_count / verified_at（PLANNED 字段级）。

---

# 9. 时间模型（冻结）

| 数据类 | 表示 | 字段 |
|---|---|---|
| Current State | Fact / Entity / Relationship / Monitor | lastSeenAt / updatedAt |
| Historical State | Snapshot（aspect 级）+ Event + Insight 历史 | createdAt / detectedAt / firstDetectedAt / lastDetectedAt |
| Immutable Record | RawCollection / Evidence（只增不改） | collectedAt / observedAt |
| Derived State | 评分/聚合/健康度（计算不落库或短期缓存） | computedAt |

未来为关系与事实引入 valid_from/valid_to（PLANNED）；先不加，避免过度设计。

---

# 10. Discovery 架构（冻结，不实施）

目标管线：

```text
Discovery Sources（SeedSource 抽象，EXISTING 1 源 + 未来扩展）
→ Candidate → Normalize → Deduplicate → Quality → Novelty → Value → Priority
→ Investigation → Entity/Relation → New Discovery
```

当前真实问题：SAN 子域自我繁殖约 94%（pending 1000+，analyzed 163）。

冻结决议：
1. **domain lineage**：rootDomainId 已有（soft link），消费端按根域限额。
2. **root-domain constraint**：同根域候选合并/降权（classifyDomain tier 已有）。
3. **novelty**：候选价值回填（analyzedValue）→ 学习“哪个候选带来好数据”。
4. **source diversity**：来源国家/类别饱和惩罚已有（score.ts），继续扩展。
5. **duplicate suppression**：SAN 源入池时子域/基建降权（classifyDomain isInfra 已有，入池逻辑 Phase 2 补）。
6. **diminishing-return control**：每根域候选上限 + 低价值标记（人工复核队列）。

禁止：无限扩量；任何“增加数量”类改动。

---

# 11. Job 架构（冻结，不实施）

目标模型：Job / JobRun / JobAttempt / JobError / JobLog（Phase 1 最小版先建 Job+JobRun 两表）。

任务类型（冻结枚举）：DISCOVERY / INVESTIGATION / DNS_SCAN / CERT_SCAN / TECH_SCAN / SECURITY_SCAN / SEO_SCAN / RANKING_SCAN / MONITOR / REANALYSIS / CLEANUP / DATA_QUALITY。

属性：queue / priority / retry（指数退避）/ timeout / concurrency（单实例预算制）/ idempotency（RUNNING map + 唯一约束）/ observability（JobLog + 指标）/ cost（行数/耗时/外部调用次数）。

现状：monitor(30m)/search-sync(6h)/discovery(15s) 三个进程内 setInterval（EXISTING，PARTIAL）；SearchSyncJob 仅覆盖搜索。

---

# 12. 安全架构（冻结）

## Inbound（EXISTING/PARTIAL）
- Rate Limit：内存滑动窗口（analyze 20/hr、tools 20/hr、report 30/hr、login 10/15min）EXISTING；多实例前不换 Redis。
- Auth：scrypt + JWT(httpOnly/SameSite=Lax) EXISTING；API Key sha256 + 撤销 + 配额 EXISTING。
- SSE Auth：**PLANNED（Phase 1 Step 4 待授权）**——当前 /api/analyze/[id] 与 /stream 无鉴权（CONFLICT/已知风险）。
- Abuse：注册无验证（PARTIAL）；请求体/分页限制（PARTIAL）。

## Outbound（EXISTING，Step 2 已收口）
- Safe Fetch 唯一网络层：DNS 校验 + IP 钉扎 + 逐跳 redirect 复验 + 端口白名单 80/443 + timeout + body 上限 + 明确错误码。
- 固定可信 URL 通道（Telegram/AI/Radar）：记录在案，未走 safeFetch（低风险，后续统一）。

## Data Security（PLANNED 基线）
- Stored XSS：SSR 默认转义 EXISTING；证据 JSON 渲染需持续审计。
- Source/Entity/Relationship Poisoning：治理层（org alias/deny、Domain 创建守卫）EXISTING；未来加“来源可信度 + 冲突标记”。

## Detection（冻结）
- 只规划 Passive / Safe / Non-destructive；禁止 Exploit/Brute-force/端口扫描/入侵。

---

# 13. Security Intelligence 数据体系（映射优先）

冻结决议：先判断现有 Fact/Evidence/Entity/Relationship/Event 能否承载：

| 安全事实 | 现有承载 | 结论 |
|---|---|---|
| TLS 配置/证书 | Fact(tls_certificate) + Entity(cert) | ✅ 可承载 |
| Security Headers | Fact(http_status) 仅存在性 | ⚠️ 需扩展字段（头值） |
| 技术/版本 | Fact(technologies) | ⚠️ 版本覆盖有限 |
| 依赖风险 | 无 | ❌ 需新采集（依赖清单） |
| 已知漏洞 | 无 | ❌ 需 Vulnerability 实体（Phase 4） |
| Finding 状态机 | 无 | ❌ 需 SecurityFinding（Phase 4） |
| Remediation/Verification | 无 | ❌ 需新表（Phase 4） |

结论：**只有明确无法表达时才建新实体**；Phase 4 前不建表。

---

# 14. SEO 数据体系（Observation/Entity/Derived 分类）

| 数据 | 类别 | 说明 |
|---|---|---|
| SearchEngine | Entity（Core Future） | Baidu/Bing/Google/Shenma/Sogou 枚举 |
| Keyword | Entity（Core Future） | 规范化关键词 |
| KeywordObservation | Observation | 某引擎某日某关键词的观测行（可复用 SearchQueryDaily 行级思路） |
| KeywordRanking | Derived/Observation | position/change 由观测计算 |
| SearchIntent | Derived | 规则/模型分类 |
| KeywordOpportunity / ContentGap / ContentOpportunity | Derived | 多因子评分计算，不落库或短期缓存 |

Phase 5 前不建任何表；现有 SearchQueryDaily/SearchPageDaily 是正确行级基础（不进 JSONB 快照）。

---

# 15. Search Engine Capability Matrix（冻结，禁止虚构）

| Search Engine | Index | Keyword | Ranking | CTR | Traffic | API | Capability |
|---|---|---|---|---|---|---|---|
| Baidu | ⚠️ 网页端有，无公开 API | ❌ 无公开 API | ❌ 无公开 API | ❌ 无公开 API | ❌ 无公开 API | ✅ 链接提交（已验证） | PARTIAL（submission only） |
| Bing | ⚠️ Webmaster 部分 | ⚠️ 需验证 | ⚠️ 需验证 | ⚠️ 需验证 | ⚠️ 需验证 | ✅ IndexNow/Webmaster | PARTIAL（待接入验证） |
| Google | ✅ GSC 官方 | ✅ GSC 官方（查询级） | ✅ GSC 官方（position） | ✅ GSC 官方 | ✅ GSC 官方 | ✅ Search Console API | STRONG（用户验证后） |
| Shenma | ❌ 无可靠公开 API | ❌ | ❌ | ❌ | ❌ | ❌ | UNSUPPORTED（仅估计） |
| Sogou | ❌ 无可靠公开 API | ❌ | ❌ | ❌ | ❌ | ❌ | UNSUPPORTED（仅估计） |

原则：只记录实际可获取数据；能力标记 supported/partial/unsupported/estimated。

---

# 16. Keyword Intelligence（冻结）

```text
Website → Search Observation → Keyword → Intent → Current Ranking
→ Opportunity → Content Gap → Content Opportunity
```

目标：帮用户发现“值得做什么”，不是 Keyword Tool。

---

# 17. Growth Intelligence 统一链（冻结）

```text
Finding → Explanation → Recommendation → Action → Verification
```

安全与 SEO 共用同一条链：

- Security：SecurityFinding → Remediation → Fix → Re-scan → Verified
- SEO：SEOFinding → Recommendation → Content/Technical Action → Re-analysis → Verified

禁止把安全和 SEO 做成两个完全独立系统。

---

# 18. Website Health（冻结）

维度：Technical / Security / SEO / Performance / Infrastructure / Content。

要求：**可解释**。不能简单平均；每个维度给出依据（哪些 check、哪些证据、缺失数据不计分——当前 audit 已实现此原则），并回答“为什么是 82”。

现状：audit 已有维度化评分（EXISTING，PARTIAL——无 Performance 真实数据不评分，符合原则）。

---

# 19. Website Timeline（冻结）

所有变化（安全/SEO/技术/基础设施）进入同一时间线：Current State → Historical State → Events → Changes。现状：Event 统一时间线 EXISTING；未来 Finding 状态变化、Ranking 变化作为事件类型扩展（PLANNED）。

---

# 20. Explore / Relationship Graph（冻结）

```text
Website → IP → ASN → Organization → Domains → Technology
Technology → Websites → Infrastructure
```

优先 PostgreSQL relationship model（EXISTING 实体页 + 报告图谱）；**不引入 Graph Database**（规模阈值：关系 >500 万或查询模式证明 PG 不足时再评估）。

---

# 21. Search 架构（冻结）

不引入 OpenSearch/Elasticsearch/Meilisearch/Typesense。

PostgreSQL 成为瓶颈的触发条件（任一）：Target >20 万 或 Entity >200 万 或 全文检索延迟 >500ms P95 且 PG FTS 优化无效。届时再评估专用搜索。

---

# 22. API 架构（冻结）

分层：Frontend → API → Application Service → Domain Service → Repository/Data Layer。

未来能力（PLANNED，按阶段开放）：Website / Entity / Relationship / Search / History / Events / Security / SEO / Keywords / Rankings / Recommendations / Monitoring。

现状：v1 analyze/report EXISTING；/api/report 限流 EXISTING；SSE 鉴权 PLANNED。

---

# 23. Caching（冻结）

可缓存：Website Profile / Entity / Technology / Search 结果 / Derived Intelligence / Rate Limit 状态。
必须实时：Security Verification / Live Status / Active Job 状态。

TTL/失效：短期内存/Next 缓存（60s-5min）+ 写入失效；Redis 仅在多实例后引入。

---

# 24. Data Freshness（冻结）

| 数据 | TTL 策略 |
|---|---|
| IP / DNS | 短（小时级） |
| Technology | 中（天级） |
| Certificate | 中（天级） |
| Organization | 长（周-月级） |
| Historical | 不可变 |

最终采用 **TTL + Change-Driven Refresh**（监测到变化才重扫），不是全量日扫。

---

# 25. Data Quality（冻结）

检查项：duplicate / orphan / invalid / self-loop / stale / conflict / low-confidence / missing-evidence / broken relationship → 形成 Data Quality Score。

现状：矛盾检测（4 规则）+ admin/data-quality 页 + 一次性脚本（PARTIAL）；未来作为 DATA_QUALITY Job 定期只读巡检（PLANNED）。

---

# 26. AI 架构（冻结）

```text
Data → Rules/Detection → Evidence → AI Interpretation
```

AI 只负责：摘要 / 解释 / SEO 策略建议 / 内容策略 / 修复说明 / 模式解读。基础事实必须来自数据（现状 explainInsight/explainStrategy 已遵守严格契约）。

---

# 27. Frontend Information Architecture（冻结）

```text
Website → Profile → Health → Security → SEO → Technology → Infrastructure
→ History → Relationships → Recommendations → Monitoring
```

用户可从一个实体持续钻取（EXISTING 实体页互跳 + PLANNED 全局 Explore）。

---

# 28. 核心页面逻辑八层（冻结）

1. 这个网站怎么样？（第一屏：状态/技术/网络/关注点）
2. 为什么？（一句话结论 + 维度评分解释）
3. 证据是什么？（证据链展开）
4. 发生过什么？（Timeline）
5. 存在什么问题？（Findings）
6. 怎么解决？（Recommendations/Remediation）
7. 下一步做什么？（Action Center）
8. 修复后有没有改善？（Verification/Re-scan）

现状：1-5 层 EXISTING（部分），6 层 PARTIAL，7-8 层 PLANNED。

---

# 29. 现在不要做的事（冻结 + 原因）

| 事项 | 原因 |
|---|---|
| GraphDB | PG 关系模型足够；规模未到 |
| Microservices / Kubernetes | 单机单进程远未饱和；增加运维复杂度 |
| Large Search Cluster | 无检索压力；PG 足够 |
| Complex AI Agent | 数据与证据未成体系 |
| Attack Platform / Exploit Scanner | 法律/伦理/产品定位 |
| Massive Discovery Expansion | 供给已过剩（SAN 94% 自我繁殖） |
| Commercialization / Bulk paid | 数据资产未建立 |
| 大规模重写 | 现有底座健康 |

---

# 30. 阶段路线（按真实依赖重排）

| 阶段 | 名称 | 依赖 | 理由 |
|---|---|---|---|
| Phase 0 | 生产稳定 | — | P0/P1 安全与数据一致性（probe ✅、report 限流 ✅、SSE 待授权） |
| Phase 1 | Data Foundation | Phase 0 | Fact 校验/Event-Signal 收敛/Job 最小版/Source Registry/保留策略 |
| Phase 2 | Discovery Intelligence | Phase 1 | 候选价值回填/SAN 降噪/队列观测 |
| Phase 3 | Website Intelligence | Phase 1 | Page 模型/画像深化（先完善“是什么”） |
| Phase 4 | Security Intelligence | Phase 3 | 依赖技术/版本/依赖数据 |
| Phase 5 | SEO Intelligence | Phase 3 | 依赖页面与观测数据 |
| Phase 6 | Search/Ranking Intelligence | Phase 5 | 依赖 SEO 观测 |
| Phase 7 | Growth Intelligence | 4+5+6 | 统一 Finding→Action 链 |
| Phase 8 | Monitoring / Explore | 4+5+6+7 | 长期服务与探索 |
| Phase 9 | API / Platform | 数据资产成熟 | 对外能力 |
| Phase 10 | Commercialization | 9 | 最后 |

---

# 31. 300 → 100 万网站：第一个真正会崩的地方

逐项评估（按崩溃时序）：

1. **API 延迟与连接（最先崩）**：报告组装大量串行/顺序查询（report.ts 多次查询、实体页 10-20 次顺序查询）+ 全部 force-dynamic + 零缓存；网站数 ×1000 后单页可能 >5s，DB 连接池与 CPU 先饱和。**触发阈值：约 1-2 万 Target 即明显。**
2. **单进程调度器（第二）**：discovery/monitor/search 全在 Next.js 进程内单飞；1M 网站监控与发现任务无法横向扩展；重启即丢状态。
3. **DiscoveryCandidate 全表扫描（第三）**：每 15s 拉全量 pending 内存排序；100 万网站候选可能千万级 → 每 tick 全表扫描拖垮 PG。
4. **Snapshots/RawCollections/Events 存储增长（第四）**：无保留策略，日增千行级 ×1M 网站 → TB 级；JSONB 查询变慢。
5. **Relationship 图查询（第五）**：fetchRelationships/实体页 one-hop 查询在亿级关系下无索引可撑。
6. **内存限流（第六）**：多实例部署即失效（需 Redis）。
7. **Search（最后）**：无检索能力，Explore 不可用。

**结论：第一个真正崩的是“报告/实体页的查询形态 + 无缓存 + 单进程”，不是数据库容量本身。** 治理顺序：查询并行化/缓存（Phase 1）→ Job 消费器（Phase 1）→ 候选队列 SQL 化（Phase 2）→ 保留策略（Phase 1）→ 关系索引/分区（Phase 3+）。

---

# 32. 新数据源接入成本（Security 源 + SEO 源）

现状（新源需改动的文件）：providers/registry.ts（注册）+ pipeline.ts（raw 存储列表）+ facts.ts（fact 映射/校验）+ report.ts（shape helpers）+ 可能 insight 规则 —— **约 5-7 个文件，且触碰核心流程**。

目标（冻结设计）：Source Registry + 适配器 + Normalizer + 类型化 Fact schema → 新源 = 新增 adapter + registry 行 + 类型化 fact 定义；**核心 pipeline 零改动**。该项是 Phase 1 Data Foundation 的验收标准之一。

---

# 33. 结论

SiteIntel 的底座方向正确、无需重写。冻结后的主架构 = **PostgreSQL 单体 + 统一证据链 + 六大领域共享数据模型 + 最小 Job 系统 + 唯一安全网络层**；所有未来模块都从“现有层能否承载”开始评估，禁止平行概念与提前建表。
