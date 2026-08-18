# SiteIntel 2.0 Gap Analysis Report（差距审计报告）

> 审计日期：2026-08-18
> 审计方式：**只读审计**（未修改任何代码 / 数据库 / 环境变量 / 生产配置 / Git / 部署 / Cron / Queue；未执行 Migration；未触发任何写操作）
> 审计依据：
> - 《SiteIntel 2.0 - 后期产品与技术总规划.md》（`C:\Users\deepo\Downloads\SiteIntel 2.0 - 后期产品与技术总规划.md`，下称“规划”）
> - 本地镜像 `C:\Users\deepo\siteintel`（master @ `1bdfcb4`，含全部历史报告）
> - GitHub 私有仓库 `Duoniu-ai/siteintel`（main @ `c74dfd1`；`multi-source/phase0-cfradar` @ `06f2486`，只读 API 复核 + 临时浅克隆核对）
> - 生产线上实测（siteintel.cc HTTP 200 / sitemap 255 URL / HSTS / Cloudflare）
> - 历史生产报告：`SITEINTEL-CURRENT-DATA-SNAPSHOT.md`、`SITEINTEL_AUTO_PROPAGATION_PRODUCTION_DIAGNOSIS.md`、`SITEINTEL_MULTI_SOURCE_DISCOVERY_AUDIT.md`、`SITEINTEL_MULTI_SOURCE_PHASE0_CFRADAR_ACCEPTANCE_REPORT.md`、`SITEINTEL_SEED_DAILY_BATCH_50_REPORT.md` 等
> - 本地只读测试：`pnpm test` → 189/189 通过（master 镜像）；multi-source 分支记录 258/258 通过

---

# 1. Executive Summary

SiteIntel 当前**不是**“工具集合”，而是一个真实具备“七层数据流 + 证据链 + 实体图谱 + 变化检测 + 洞察引擎 + 监控告警”的**单体数据情报平台**。相对于 2.0 规划，它的底座（数据模型、证据、安全网络层、前端信息架构）是健康的，且已经走过两轮生产治理（SSRF 修复、www 实体合并、org 别名、Domain 创建守卫、多源 Phase 0）。

**总体判断：架构方向与规划一致，无需推翻；真正的差距集中在五处：**

1. **数据资产层**：Event/Signal 并行重复记录、大量 JSONB 无约束、候选数据不回流为价值反馈——数据库尚未成为“可复利资产”。
2. **Job / Worker 体系**：全部任务由 3 个进程内 `setInterval` 调度，无统一 Job System、无外部队列、无可观测性；这是 Phase 0 稳定性的核心短板。
3. **Security / SEO 业务模型**：被动检测数据已具备，但 `SecurityFinding / Vulnerability / Remediation / Keyword / Ranking / Opportunity` 等业务模型**完全未建**；现有数据足够支撑其自然形成，缺的是“模型 + 规则 + 页面”。
4. **Discovery 质量闭环**：供给（>500/天）与消费（50/天）失衡，SAN 子域自我繁殖占 94%，候选缺“价值回填”机制——仍停留在“发现更多”，未到“发现更有价值”。
5. **生产工程化**：多源代码未合入 main、本地镜像落后于生产、staging 未清理、无 deploy.sh、无 CI/CD、无服务自身监控——基线分裂风险高。

**最重要的一句话**：不要开发新功能，先做 **P0 安全收口（probe SSRF）+ 数据一致性收口（Event/Signal、Job System 雏形）+ Discovery 质量闭环**，然后再进入 Security / SEO 业务模型阶段。

---

# 2. 当前 SiteIntel 真实状态

## 2.1 版本与部署基线

| 项 | 实测值 | 说明 |
|---|---|---|
| 本地镜像 | `C:\Users\deepo\siteintel`，master @ `1bdfcb4`（legacy 66 commits） | 工作树含未提交报告/文档（与 SITEINTEL_STATE.md 一致） |
| GitHub main | `c74dfd1`（2026-08-17，SEO Phase 1 P0） | 私有仓库 `Duoniu-ai/siteintel`，default branch `main` |
| 生产代码基线 | main `c74dfd1` + `multi-source/phase0-cfradar`（`e32bd93`+`06f2486`）文件 | 多源文件在生产落位，但**未 merge 进 main** |
| 生产服务 | systemd `siteintel`，端口 3003，nginx + Cloudflare 反代 | 线上实测 200 OK；staging（:3004）仍 active，待删 |
| 数据库 | PostgreSQL 14，库 `siteintel`（127.0.0.1:5432），约 41 MB | 生产 DB 无法本地直连；数据口径以 08-17 只读报告为准 |
| 测试 | 本地 master 189/189 通过；multi-source 分支记录 258/258 通过 | 只读运行确认 |

## 2.2 数据规模（生产，08-16/08-17 只读快照）

| 指标 | 08-16 | 08-17 | 说明 |
|---|---:|---:|---|
| Target | 279 | 289 | 已分析网站（去 www） |
| Investigation | 480 | 537 | completed/partial/failed 混合 |
| Entity | 3196 | 3230 | 其中 subdomain 1129 + nameserver 433 ≈ 48%（SAN 派生） |
| Relationship | 5659 | 5888 | 9 类关系 |
| Snapshot | 3047 | 3147 | 10 种 aspect，hash 去重 |
| Event | 62 | 65 | 12 类网站事件 + 搜索事件（category 维度） |
| Insight（active） | 55（49 active） | 58 | 13 条规则产物，证据链 0 断裂 |
| Fact / RawCollection | 2246 / 3346 | — | 报告读 Fact，快照存历史 |
| DiscoveryCandidate | 926（pending 875） | 926+（pending ~856-875） | SAN 源占 ~94% |
| 自动分析 | 18/天（旧 20 上限时代） | 50/天（DISCOVER_DAILY_CAP=50） | SEED_DAILY_BATCH=50 已生效 |
| 候选净增 | +570/天 | — | 供给远超消费，积压加速 |

## 2.3 Git / 部署卫生状态

- GitHub：main=`c74dfd1`；`multi-source/phase0-cfradar` @ `06f2486`（未 merge）；`seo/phase-1-p0`；`legacy-prod-history`；3 个 production tag 与记录一致。
- 本地镜像无 remote、无 tag；工作树 33+ 项未跟踪（报告/文档/diag_sql/well-known），按约束不清理。
- 服务器：`.env` 手工管理（SEED_DAILY_BATCH=50、DISCOVER_DAILY_CAP=50、CLOUDFLARE_API_TOKEN 等）；部署通道 = Git bundle + systemd restart；`deploy.sh` 尚未落库；staging 服务与只读角色待删。

---

# 3. 当前架构

## 3.1 技术栈

Next.js 16.3（App Router，SSR）+ React 19 + TypeScript strict（`noUncheckedIndexedAccess`）+ Tailwind 4 + Prisma 6 + PostgreSQL 14。单进程 systemd 服务（:3003）。依赖极少（@prisma/client、jose、next、react 四件套）。零外部前端运行时依赖（图谱为自研 SVG 力导向）。双语（zh/en，cookie 记忆）。

## 3.2 分层

```
Frontend（App Router SSR 页面 + 少量客户端组件）
   ↓
API Route Handlers（/api/* 与 /api/v1/*）
   ↓
Application / Domain Services（lib/ 下各引擎）
   ├── providers/（dns rdap ip ssl http metadata technology — 失败隔离）
   ├── pipeline.ts（调查编排：validate→collect→raw→entity→snapshot→fact→event→insight→candidate）
   ├── entities.ts / entity-resolution.ts（实体/关系/组织治理）
   ├── facts.ts / snapshots.ts / events.ts / correlation.ts / contradictions.ts
   ├── intelligence/（13 条规则 + 可解释置信度）
   ├── discovery/（候选提取 + 多源种子 + probe + 评分 + 退避）
   ├── monitoring.ts / notify.ts（监控 + Telegram/Webhook）
   ├── search/（Baidu 官方适配 + 同步调度）
   ├── seo/（页面注册表 + 质量门 + 实体页组装）
   └── security/（safe-fetch 统一网络层）
   ↓
Prisma → PostgreSQL（schema.prisma，27+ 模型）
```

## 3.3 安全基线（已有）

- 统一网络层 `safe-fetch`：解析 → 逐地址校验（IPv4/IPv6/内嵌 IPv4/NAT64/6to4）→ IP 钉扎 + Host/SNI → 逐跳重定向再校验。webhook 同规则。
- 密码 scrypt + timingSafeEqual；JWT（jose HS256）httpOnly + SameSite=Lax；API Key `si_` 前缀、sha256 存储、可撤销、按小时限流 + 月度配额。
- 搜索凭据 AES-256-GCM 加密（密钥由 AUTH_SECRET 派生）。
- 生产无 source maps、`poweredByHeader: false`、robots 屏蔽私有区、SSR 默认转义。

---

# 4. 当前数据库结构

## 4.1 模型清单（prisma/schema.prisma，本地 master）

| 分组 | 模型 |
|---|---|
| 分析主体 | Target、Investigation |
| 原始证据 | RawCollection |
| 图谱 | Entity、EntityRelationship |
| 时间历史 | Snapshot（10 aspect）、Event（+category/severity） |
| 解释层 | Insight（+certainty/supporting/contradicting/recommendedNextStep） |
| 事实层 | Fact、Evidence、Signal、Contradiction |
| 监控 | Monitor |
| 用户/API | User、ApiKey、DomainOwnership |
| 搜索 | SearchConnection、SearchProperty、SearchDailyMetric、SearchQueryDaily、SearchPageDaily、SearchSubmission、SearchSyncJob |
| Discovery | DiscoveryCandidate |
| 组织治理 | OrganizationAlias、OrganizationDenyLog |

生产增量（multi-source/phase0-cfradar，migration `20260817_multi_source_candidate_pool` 已部署）：`Target.isApex`；DiscoveryCandidate +18 列（probeStatus/lastProbeAt/lastStatusCode/probeLatencyMs/probeFailureCount/failureCount/lastFailureReason/nextRetryAt/score/scoreReason/sourceCount/sourceRank/sourceCountry/sourceCategory/cloudflareRank/domainType/rootDomainId/metadata）+ 3 个索引。

## 4.2 关键索引

已有：Target.domain unique；Entity (type,normalized) unique + normalized；EntityRelationship (source,target,type) unique + targetId；Snapshot (targetId,type,createdAt)；Event (targetId,detectedAt)；Insight (targetId,firstDetectedAt)；Fact (targetId,factType) unique；DiscoveryCandidate (status,priority)、(status,probeStatus)、(rootDomainId)、(probeStatus,status,nextRetryAt)；Monitor (targetId,userId,frequency) unique。

缺失（规模化前可缓）：Event (targetId,type,detectedAt)、Insight (targetId,status)、EntityRelationship 按 source.type/target.type 的复合查询索引、Snapshot 按 created_at 的清理索引、Fact 按 lastSeenAt 的陈旧扫描索引。

---

# 5. 当前数据流

```
用户输入 / Discovery 候选 / Seed / 监控 tick / 搜索同步
   ↓
startInvestigation（RUNNING map 防重）
   ↓
collectAll（7 providers，失败隔离 partial）
   ↓
RawCollection（原文持久化，证据链根）
   ↓
extractEntities（Entity / EntityRelationship，幂等 upsert，组织治理）
   ↓
Snapshot（10 aspect，hash 去重，历史存储）
   ↓
syncFacts（Fact 单值 upsert + Evidence 行 + Signal）
   ↓
diffFact → Event（12 类）+ runSignalCorrelation（复合事件）+ contradiction detector
   ↓
runInsightEngine（13 条规则，active/resolved 生命周期）
   ↓
extractCandidatesFromBundle（SAN/CNAME/redirect/共享实体 → Candidate 池）
   ↓
Report 组装（报告读 Fact，不读 aspect 快照）
```

这条链路的设计与规划 §14（Raw→Parser→Fact→Normalizer→Entity）高度一致，**是当前项目最大的资产**。

---

# 6. 当前 Discovery 架构

生产实际运行的是 multi-source 引擎（`src/lib/discovery/collector.ts` @ `06f2486`）：

- **调度**：15s tick（`DISCOVER_TICK_MS`），每 tick 一个动作（分析优先 → probe 次之），单飞互斥。
- **预算**：`DISCOVER_DAILY_CAP=50` 分析/天；`SEED_DAILY_BATCH=50` 种子导入/天（6h 窗口）。
- **种子源**：Cloudflare Radar 全球 Top100（`sources/cloudflareRadar.ts`），源头过滤基建域，只入池不建 Target。
- **预检**：`probe.ts` DNS→HTTPS→HTTP，失败按 1h→4h→12h→24h→72h→168h 指数退避，7 次后 skipped 保留。
- **评分**：`score.ts` 透明可解释（可访问性/主域级别/新颖性/外部排名/历史成功率/繁殖潜力/多样性 − 近期/失败/重复惩罚），`scoreReason[]` 落库。
- **内源**：SAN / CNAME / redirect / 共享 IP / 共享证书 / 共享跟踪 / 共享基建（`candidates.ts`）。
- **去重**：Target ∪ Candidate 双重守卫；`sourceCount` 累计多源信号。
- **反向发现**：共享实体（IP/证书/跟踪 ID）已实现；“数据缺口→候选”尚未实现。

**核心矛盾**：供给（SAN 自我繁殖 + 种子）≫ 消费（50/天）；候选无“分析后价值回填”，无法回答“哪些候选最终成为高价值 Target”。

---

# 7. 当前 Job / Worker 架构

- **无 Job System**：无 Job / JobRun / JobAttempt / JobError / JobLog 表，无外部队列（无 Redis/BullMQ/DB queue）。
- 三个进程内调度器（`src/instrumentation.ts`）：monitoring 30min、search-sync 6h、discovery 15s（多源）/ 72min（master）。
- 单 worker 语义：单 systemd 进程；pipeline `RUNNING` Map 做同域去重；重启时 >1h running 调查标记 failed。
- `SearchSyncJob` 是唯一的“任务表”，仅覆盖搜索同步。
- 可观测性：只有 console.log / systemd journal；无任务级状态/重试/成本记录。

**结论**：与规划 §18（统一 Job System）差距明确；但当前规模下单进程可用，属“该建但不必一步到位”的项。

---

# 8. 当前 Security 能力

## 8.1 已具备的被动检测数据

| 能力 | 状态 | 代码位置 |
|---|---|---|
| HTTP/HTTPS | ✅ | `providers/http.ts`（状态/跳转链/安全头） |
| TLS 握手 + 证书 | ✅ | `providers/ssl.ts`（链、有效期、SAN、指纹、签名算法） |
| Security Headers | ✅（仅存在性，5 项） | `http.ts` securityHeaders |
| 技术栈检测 | ✅ | `providers/technology.ts` + `fingerprints.ts`（header/cookie/HTML/generator/URL） |
| 版本检测 | ⚠️ 有限 | 仅 header/asset URL 中的少数版本 |
| 依赖检测 | ❌ | 无 JS 依赖清单/CVE 映射 |
| CMS 检测 | ✅ | fingerprints category=cms（WordPress 等） |
| CDN / Server / Hosting | ✅ | `analyzers/infrastructure.ts` + fingerprints |
| DNS / NS / MX | ✅ | `providers/dns.ts` |
| IP / ASN / 地理 | ✅ | `providers/ip.ts`（ipwho.is→uapis.cn→rdap 降级） |
| SSRF 防护（主链路） | ✅ 强 | `security/safe-fetch.ts` + `ssrf-guard.ts` |
| 安全洞察 | ⚠️ 基础 | tls_expiry / domain_expiry / website_status 规则 |
| Security Finding 模型 | ❌ | 无表 |
| CVE / Advisory / Remediation / FixVerification | ❌ | 无 |

## 8.2 SSRF 专项检查（对照规划 §48/§49）

| 检查项 | 主采集链路（safe-fetch） | Discovery probe（生产，multi-source） |
|---|---|---|
| 内网 IP / localhost / 127.0.0.1 | ✅ 阻断 | ❌ **不校验** |
| IPv6 / 内嵌 IPv4 / NAT64 / 6to4 | ✅ 阻断 | ❌ 不校验 |
| DNS rebinding | ✅ IP 钉扎 | ❌ fetch 自行二次解析 |
| redirect bypass | ✅ 逐跳再校验 | ⚠️ fetch redirect:follow 直接跟随 |
| cloud metadata（169.254.169.254） | ✅ 阻断 | ❌ 不校验 |
| 非标准端口 | ⚠️ 地址校验后允许任意端口 | ❌ 无限制 |
| 超大响应 | ✅ 1.5MB 上限 | ⚠️ fetch 无 body 上限 |
| 超时 | ✅ 12s | ✅ 8s |
| 超多 redirect | ✅ 6 跳 | ⚠️ 依赖 fetch 默认（20 跳） |
| 并发 | ⚠️ 仅 IP 级限流 + 单域去重，无全局并发上限 | ⚠️ 单飞（单进程） |

> **P0 安全缺口**：`src/lib/discovery/probe.ts` 是生产实际运行的**第二套 Fetch 实现**，未走 safe-fetch。攻击路径：分析一个由攻击者控制的域名（其 DNS/redirect/SAN 指向内网主机名或私有 IP）→ 候选入池 → probe 直接 `fetch` 该地址。违反规划 §49“禁止业务层自行实现第二套 Fetch”，必须优先修复。

---

# 9. 当前 SEO 能力

## 9.1 已实现（技术 SEO 检测，站点维度）

title / description / H1（数量+文本）/ robots.txt 存在性 / sitemap 存在性 / 结构化数据块数量 / 内链外链数量 / 字数 / favicon / lang / OG title/description/image / generator。

代码位置：`providers/metadata.ts`；报告投影 `report.ts` overview；Audit 检查 `lib/audit.ts`（SEO 维度 8 项）。

## 9.2 未实现（规划 §28-§34）

canonical、hreflang、robots meta indexability（noindex 判定）、OpenGraph type/twitter card、image alt、URL 结构、404/redirect 分析、Keyword、KeywordObservation、KeywordRanking、SearchIntent、KeywordOpportunity、ContentGap、ContentOpportunity、SearchEngine/IndexObservation 模型。

搜索数据现状：仅 Baidu 官方链接提交（已验证可用）；Baidu 索引量/关键词/排名**无公开 API，诚实标记 absent**（`search/registry.ts` 仅注册 baidu-official，`sync.ts` 的能力探测全部走 provider capability）。`SearchDailyMetric / SearchQueryDaily / SearchPageDaily` 是官方数据行表，为未来 KeywordRanking 提供了正确的行级基础（关键点：不进 JSONB 快照）。

> **规划要点确认**：当前 SEO 主要实现了“SiteIntel 自身站点的 SEO 营销”（registry/sitemap/OG/质量门），而“对分析网站的 SEO Intelligence”仅到 Technical SEO 检测。二者不能混淆；前者是营销资产，后者才是产品能力。

---

# 10. 当前 Technology Intelligence

- 指纹库 `fingerprints.ts`（~19KB 规则），57 个 technology 实体（生产），分类覆盖 frontend/backend/cms/cdn/hosting/analytics/tag_manager/security/payments/fonts/javascript/other。
- 版本：header（server/x-powered-by）与 asset URL 模式提取，有限。
- 关系：domain → uses → technology；共享技术反查页 `/technology/[slug]`（≥3 站才索引）。
- 未实现：依赖树/依赖清单（JS 包、CDN 引用文件清单）、版本→已知漏洞映射、TechnologyVersion 实体、技术版本分布统计页。

---

# 11. 当前 Monitoring

- 模型：Monitor（frequency daily/weekly/monthly，lastRunAt/lastNotifiedAt，unique per user+target+freq）。
- 调度：30min tick，按频率预算（daily 10 / weekly 3 / monthly 2，合计 ≤15/拍）。
- 流程：到期重跑 pipeline → 取新鲜事件/洞察 → Telegram / Webhook 告警（HMAC 签名 + SSRF 复检），无通知通道时不推进 lastNotifiedAt（保留重试）。
- 已修缺陷：wordpress.org 无通知通道死循环（141 次空转）已在多源 Phase 0 修复。
- 缺口：无邮件通道；无 SEO/安全/排名维度监控；无告警历史表；无监控目标健康度视图（dashboard 仅列表）。

---

# 12. 当前前端产品结构

| 公开区 | 页面 |
|---|---|
| 首页 | `/`（3A 用户化首页，真实示例卡） |
| 产品页 | /how-it-works、/website-intelligence、/search-intelligence、/infrastructure-intelligence、/technology-intelligence、/relationship-intelligence、/website-monitoring |
| 工具页 | /tools/website-analysis、/tools/dns、/tools/ip、/tools/ssl、/tools/technology |
| 搜索中心入口 | /search/baidu、/search/bing、/search/google |
| 指南/案例 | /guides/* ×3、/cases/* ×2 |
| 动态情报页 | /website/[domain]（质量门）、/report/[domain]（308 迁入）、/technology/[slug]、/ip/[ip]、/asn/[asn]、/certificate/[fp]、/organization/[slug]、/report（探索索引） |
| 私有区 | /dashboard、/bulk、/search（控制台）、/login、/register、/admin/seo、/admin/data-quality、/claim/[domain]、/analyze/[id] |

报告页结构（`components/report/report-page.tsx`，64KB 单体组件）：Executive Summary（一句话结论+首屏 4 指标）→ Technology → How It Runs（Infra）→ History → Insights → Relationships（图谱）→ Related → Deep Dive（Audit + Recommendations）→ Search（若有）→ Evidence。

实体页跳转链：website → IP → ASN → Organization；technology → websites；certificate → domains。**数据间互跳已基本打通**。

---

# 13. 2.0 规划对照表

| 规划维度 | 状态 | 证据摘要 |
|---|---|---|
| Website Intelligence | ✅ 已实现（基础） | Target/Fact/Entity/报告页，7 provider |
| Security Intelligence | ⚠️ 部分（被动数据） | 无 Finding/Remediation/Verification 模型 |
| SEO Intelligence | ⚠️ 部分（Technical SEO） | 无 Keyword/Ranking/Intent/Opportunity |
| Technology Intelligence | ✅ 基础已实现 | 指纹 + 版本(有限) + 反查页；无依赖/CVE |
| Change Intelligence | ✅ 已实现 | 12 类事件 + 13 规则 + correlation + contradiction |
| Growth Intelligence | ⚠️ 部分 | audit 评分 + 推荐 + AI 解释；无 Action Center |
| Entity/Relationship 数据体系 | ✅ 已实现 | Entity + EntityRelationship + 治理层 |
| Evidence/Source/Confidence | ⚠️ 部分 | Evidence 表存在但 rawValue 占位；无 Source Registry；Confidence 多模型但未统一 |
| Discovery 自动繁衍 | ⚠️ 部分 | 多源 Phase 0 上线；无价值回填/缺口驱动 |
| Job System | ❌ 未实现 | 仅进程内 setInterval + SearchSyncJob |
| Security 检测边界 | ✅ 符合（被动安全） | 无攻击性检测 |
| SEO 数据体系 | ❌ 未实现 | 无 Keyword/Ranking 模型 |
| Recommendation / Action Center | ⚠️ 部分 | 推荐计算不落库，无 Action 生命周期 |
| Monitoring | ⚠️ 部分 | 基础设施监控可用；无 SEO/安全/排名维度 |
| Explore | ⚠️ 部分 | 实体页存在；无全局 Explore/检索 |
| API Platform | ⚠️ 部分 | v1 analyze/report + API Key；无 entity/relationship API |
| 数据库规划 | ✅ 方向一致 | PG 单体 + 合理索引 + JSONB 有节制；规模尚小 |
| 后期 Phase 路线 | ✅ 记录在案 | Phase 0-9 与规划一致 |
| 暂时不要做的事情 | ✅ 已遵守 | 未引入 GraphDB/微服务/K8s/AI Agent |

---

# 14. Gap Analysis（对照规划的分类差距）

> 分类：A 已实现 / B 部分实现 / C 尚未实现 / D 架构不合理 / E 数据模型问题 / F 安全问题 / G 性能问题 / H 规划冲突 / I 暂时不要开发

## A. 已实现

### A1. 七层数据流 + 证据链
- 当前状态：Raw → Fact → Entity → Relationship → Snapshot → Event → Insight → Report，每层持久化。
- 代码位置：`src/lib/pipeline.ts`、`src/lib/facts.ts`、`src/lib/snapshots.ts`、`src/lib/events.ts`、`src/lib/intelligence/engine.ts`。
- 数据库位置：RawCollection / Fact / Evidence / Entity / EntityRelationship / Snapshot / Event / Insight。
- API：`/api/analyze`、`/api/report/[domain]`、`/api/v1/*`、`/api/insights/[id]/explain`。
- 实现方式：单次调查内顺序执行，失败隔离（partial），报告只读 Fact。
- 与 2.0 差距：Fact.value 为无约束 JSONB；历史仅 Snapshot + Event，无“事实版本表”。
- 风险：中（JsonB 形状漂移）。
- 建议：为 9 类 Fact 建立 TS 校验器（zod）并在写入前校验；暂不建事实版本表。
- 优先级：P1。

### A2. 实体图谱 + 关系反查
- 当前状态：11 类实体、9 类关系、IP/ASN/证书/组织/技术反查页、报告图谱组件。
- 代码位置：`src/lib/entities.ts`、`src/lib/seo/entity.ts`、`src/lib/seo/technology.ts`、`src/components/relationship-graph.tsx`。
- 数据库位置：Entity / EntityRelationship。
- API：页面直连 DB（无独立 entity API）。
- 实现方式：幂等 upsert；evidence 内嵌；kind=observed/probable/inferred。
- 与 2.0 差距：Relationship 无 valid_from/valid_to，历史关系变化不可查；无统一 entity API。
- 风险：低。
- 建议：保留现状；后续加“关系变化事件”时再引入关系历史。
- 优先级：P2。

### A3. Change Intelligence
- 当前状态：12 类事件 + 13 条洞察规则 + 信号相关性 + 矛盾检测。
- 代码位置：`src/lib/facts.ts`、`src/lib/correlation.ts`、`src/lib/contradictions.ts`、`src/lib/intelligence/rules.ts`。
- 数据库位置：Fact / Signal / Event / Insight / Contradiction。
- 实现方式：事实 diff → 类型化事件；同次多事实变化 → 复合事件；矛盾标记事实状态。
- 与 2.0 差距：Event 与 Signal 双轨重复（见 E1）；无变化分类学（无 SEO_CHANGE/SECURITY_CHANGE/RANKING_CHANGE）。
- 风险：中（双轨漂移）。
- 建议：Phase 1 统一事件源（Signal 收敛为内部过程记录或移除）。
- 优先级：P0。

### A4. Safe Fetch / SSRF 主链路
- 当前状态：统一解析→校验→钉扎→逐跳复验；IPv4/IPv6/内嵌/NAT64/6to4/metadata 全覆盖；webhook 同规则。
- 代码位置：`src/lib/security/safe-fetch.ts`、`src/lib/ssrf-guard.ts`。
- 测试：17+63 项单测通过。
- 与 2.0 差距：probe.ts 绕过（F1）；端口未限制。
- 风险：主链路低；probe 高（见 F1）。
- 建议：P0 先修 probe。
- 优先级：P0。

### A5. 监控基础
- 当前状态：daily/weekly/monthly 重跑 + Telegram/Webhook + 去重基线。
- 代码位置：`src/lib/monitoring.ts`、`src/lib/notify.ts`。
- 与 2.0 差距：无邮件、无维度监控、无告警历史。
- 建议：P2 补邮件通道 + 告警历史表（AlertDelivery）。
- 优先级：P2。

### A6. 用户 / 权限 / API Key / 配额
- 当前状态：scrypt + JWT；API Key 哈希 + 撤销 + 限流 + 月度配额；所有权验证（DNS TXT / well-known / meta）。
- 代码位置：`src/lib/auth.ts`、`src/lib/api-keys.ts`、`src/lib/ownership.ts`。
- 与 2.0 差距：注册无验证/风控；限流内存态；无 RBAC。
- 建议：Phase 0 仅加“注册节流”，不做完整 RBAC。
- 优先级：P2。

### A7. SEO 站内营销体系
- 当前状态：SEO_PAGE_REGISTRY、质量门（INDEX/复核/NOINDEX）、sitemap 255 URL、robots、OG、实体页。
- 代码位置：`src/lib/seo/`、`src/app/sitemap.ts`、`src/app/robots.ts`。
- 与 2.0 差距：这属于“SiteIntel 自己的 SEO”，与“网站的 SEO Intelligence”不同。
- 建议：保持；不得因营销 SEO 挤占产品 SEO 预算。
- 优先级：P3。

### A8. 数据治理历史工程
- 当前状态：www 合并（294→210）、OrganizationAlias + DenyLog、Domain 创建守卫、孤立清理、AS0 哨兵。
- 代码位置：`src/lib/entity-resolution.ts`、`src/lib/entities.ts`、`scripts/`。
- 建议：保留；Organization 历史合并（21 实体）等待授权后执行。
- 优先级：P2（执行待授权）。

## B. 部分实现

### B1. Security Intelligence（被动数据 → 无业务模型）
- 当前状态：TLS/Headers/技术/版本(有限)/证书/域名到期 可检测；洞察仅 tls_expiry/domain_expiry/status。
- 代码位置：`lib/audit.ts`（security 5 项）、`lib/intelligence/rules.ts`。
- 数据库：无 SecurityFinding/Vulnerability/Remediation 表。
- 与 2.0 差距：规划 §21-§26 全部未落地。
- 风险：产品承诺风险（用户以为在“安全分析”，实际只有头/证书检查）。
- 建议：Phase 3 再建模型；先把“现有被动数据能否自然形成 Finding”的映射表做出来（本报告 §18 已给）。
- 优先级：P2。

### B2. SEO Intelligence（Technical SEO 部分）
- 当前状态：title/desc/H1/robots/sitemap/JSON-LD 计数/内链数 等 8 项 audit + 元数据采集。
- 代码位置：`lib/audit.ts`、`providers/metadata.ts`。
- 缺口：canonical、noindex、hreflang、alt、URL 结构、关键词、排名、意图、机会。
- 建议：Phase 4 前不建 Keyword 表；先把 Technical SEO 字段（canonical/noindex/hreflang）补齐到现有 metadata fact——**这是“现在必须改”的数据模型项**。
- 优先级：P1（字段级）。

### B3. Growth Intelligence / Recommendation
- 当前状态：22 项 audit 检查 → 公式化推荐（P0/P1/P2）+ SWOT-lite + AI 解释（GLM）。
- 代码位置：`lib/audit.ts`、`lib/recommendations.ts`、`lib/ai.ts`、`components/report/report-page.tsx`。
- 缺口：推荐不落库、无 Action 生命周期、无验证回环。
- 建议：Phase 5 再建 Recommendation/ActionItem 表；当前保持计算式推荐。
- 优先级：P2。

### B4. Monitoring
- 缺口：无邮件；无 SEO/安全/排名监控；无告警历史。
- 建议：P2 先补 AlertDelivery 历史表 + 邮件通道（Nodemailer/SMTP）。
- 优先级：P2。

### B5. Explore
- 现状：IP/ASN/Certificate/Organization/Technology 五类实体页 + 报告图谱；无全局 Explore 与实体检索。
- 建议：Phase 7 前不做；可用“site search”（简单 SQL 检索）替代。
- 优先级：P3。

### B6. Job System
- 现状：SearchSyncJob 仅覆盖搜索；DiscoveryCandidate 承担队列语义。
- 建议：Phase 1 建最小 Job/JobRun（不建 6 张表，先 2 张：Job + JobRun），把 monitor/discovery/search 三个调度器统一登记。
- 优先级：P0。

### B7. Data Quality
- 现状：contradiction 检测 + admin/data-quality 页 + 一次性脚本；无自动化 TTL/孤儿/陈旧扫描。
- 建议：Phase 1 加 cron 型“数据质量巡检”Job（只读检查 + 报告），不做自动删除。
- 优先级：P1。

### B8. History
- 现状：Snapshot 逐 aspect 历史 + Event 时间线 + Entity/Relationship first/lastSeen。
- 缺口：Relationship 变化无历史；Fact 值历史仅靠 Signal/Event 推断。
- 建议：Phase 1 加 `FactHistory`（或复用 Signal 正规化）——**这是“现在必须改”之一**（E1 合并方案）。
- 优先级：P1。

### B9. Candidate Intelligence Queue
- 现状：透明评分 + probe + 退避 + 老化；无价值/新颖性/风险分；无队列观测页。
- 建议：Phase 2 加“候选→Target 价值回填”列（见 §20），并在 admin 加队列页。
- 优先级：P1。

### B10. API Platform
- 现状：v1 analyze/report + API Key + 配额 + docs 页。
- 缺口：无 entity/relationship/technology/search API；/api/report 公开无鉴权（F2）。
- 建议：Phase 8 前不扩 API；先封 /api/report。
- 优先级：P2。

## C. 尚未实现

| # | 能力 | 规划出处 | 建议时机 |
|---|---|---|---|
| C1 | SecurityFinding / Vulnerability / Remediation / FixVerification | §21-§26 | Phase 3 |
| C2 | 依赖检测 + CVE/Advisory 映射 | §22 Dependency | Phase 3 |
| C3 | Keyword / KeywordRanking / SearchIntent / KeywordOpportunity / ContentGap | §30-§34 | Phase 4 |
| C4 | Recommendation / ActionItem 持久化 + Action Center | §36-§37 | Phase 5 |
| C5 | Website Compare | §59 | Phase 6+（P3） |
| C6 | SiteIntel Search（多条件实体检索） | §42 | Phase 7（P3） |
| C7 | 全维度 Monitoring（SEO/安全/排名 + 告警历史） | §39 | Phase 6 |
| C8 | Data Source Registry（source/rate_limit/cost/reliability） | §13 | Phase 1 轻量版（表 + 登记 7 provider + Radar） |
| C9 | Raw 数据重放/重解析管线 | §14 | Phase 1 后（能力预留，不建表） |
| C10 | Cache 层（Redis 或 Next 缓存策略） | §47 | 数据规模/并发达到阈值后（P2 观察） |
| C11 | 数据 TTL / 清理 / 分区 | §52 | Phase 1 清理策略（不分区） |
| C12 | 通用 Job/JobRun/JobAttempt/JobError/JobLog/JobCost | §18 | Phase 1 最小版 |
| C13 | 服务自身监控 / 日志聚合 / CI/CD | Phase 0 | Phase 0 |
| C14 | 商业化（计费/用量自助） | §71 | Phase 9 |

## D. 已有但架构不合理

### D1. 三个进程内调度器替代 Job System
- 当前状态：monitoring 30min / search 6h / discovery 15s，各自 setInterval + 各自 running 标志。
- 代码位置：`src/instrumentation.ts` + `monitoring.ts` + `search/sync.ts` + `discovery/collector.ts`。
- 问题：重启即丢状态；无重试/退避统一；单实例绑定；曾出现 monitor 死循环缺陷。
- 建议：Phase 1 建最小 Job 表，三调度器登记为 Job 定义，tick 消费 JobRun。不引入 Redis。
- 优先级：P0。

### D2. probe.ts 第二套 Fetch
- 见 F1。违反规划 §49。
- 优先级：P0。

### D3. 双 Change 记录（Signal + Event）
- 见 E1。
- 优先级：P0。

### D4. Insight 承担 Hypothesis 角色
- 当前状态：Insight 同时有 certainty/supportingEvidence/contradictingEvidence/recommendedNextStep（假设字段）与 evidence/details（解释字段）。
- 问题：语义越界（规划 §54 要求 Insight=解释、避免 Insight≈Hypothesis）。
- 建议：保留字段但明确规则——非搜索类洞察不写 certainty/recommendedNextStep；Phase 1 后若 Hypothesis 成为一等公民再拆表。
- 优先级：P2。

### D5. /api/report 公开 JSON 与 v1 API 并存
- 见 F2。
- 优先级：P1。

### D6. 生产代码基线分裂
- 当前状态：main=`c74dfd1`；生产运行 c74dfd1 + multi-source 未合入文件；本地镜像 @1bdfcb4。
- 问题：回滚/审计/新开发都在“三个世界”里。
- 建议：Phase 0 把 multi-source 合入 main（评审后），本地镜像同步，后续以 main 为唯一基线；deploy.sh 落库。
- 优先级：P1。

### D7. DiscoveryCandidate 承担队列 + 状态机
- 问题：无入队历史/分析结果/价值反馈字段；pending 全表内存排序。
- 建议：加 `analyzedResult`（summary/价值）列 + admin 队列页；保持表结构。
- 优先级：P1。

### D8. 报告页单体组件
- 当前状态：`report-page.tsx` 64KB，单文件包含 10+ 区块。
- 建议：Phase 2 拆分区块组件（security/seo/history/recommendations/evidence），不做页面重构。
- 优先级：P2。

## E. 已有但存在数据模型问题

### E1. Event 与 Signal 双轨重复（重点）
- 现象：一次事实变化同时写 Signal（fact_changed, previous/current）与 Event（ip_changed 等）；fact_created 只写 Signal；fact_contradicted 只写 Signal。
- 代码位置：`facts.ts`（Signal 写入）+ `events.ts`/`correlation.ts`（Event 写入）。
- 数据库：Signal、Event 两表内容重叠。
- 风险：双轨漂移、洞察/监控各读一边、未来历史查询不确定以谁为准。
- 建议：**Event 为唯一外部事件源；Signal 降级为内部过程日志（不对外）**；fact_created 也产出一个低严重度 Event（或明确不产 Event 并在文档固化）。Phase 1 迁移存量 Signal。
- 优先级：P0。

### E2. 关键 JSONB 无形状约束
- 对象：Fact.value（9 类形状）、Snapshot.data（10 aspect）、Event.previous/current、Insight.evidence/details、Entity.metadata（异构）、RawCollection.rawData、DiscoveryCandidate.scoreReason/metadata。
- 风险：报告 shape helpers（tuple 假设如 `[slug,name,category,version,confidence]`）脆弱；未来 Keyword/Ranking 若塞 JSONB 会重蹈覆辙。
- 建议：对 Fact.value 与 Snapshot 加 zod 校验器（写入前 validate + 失败降级）；**新业务模型（Keyword/Ranking/Finding）一律建表，不进 JSONB**。
- 优先级：P1。

### E3. Relationship.evidence 与 Insight.evidence 内嵌，不与 Evidence 表关联
- 现象：关系证据是 JSON 数组；洞察证据是 JSON 数组；Evidence 表只覆盖 Fact 级。
- 风险：无法按 source/时间统一查询“全部证据”。
- 建议：Phase 1 只在 Fact 级维持 Evidence 表；关系/洞察证据暂保持内嵌（规模小，不强制外键）。
- 优先级：P2。

### E4. DiscoveryCandidate.rootDomainId 无外键
- 现象：soft link TEXT，Target 删除后孤儿。
- 建议：加 FK（ON DELETE SET NULL）或接受 soft link 并在巡检中清理。**现在必须改（一行 migration）**。
- 优先级：P1。

### E5. subdomain / nameserver 实体占 48%（SAN 派生噪声）
- 现象：subdomain 1129 + nameserver 433 / Entity 3230。
- 风险：图谱信噪比下降；Explore/反查页未来被噪声淹没。
- 建议：Phase 2 引入实体价值分级（core/supplementary/noise），subdomain/nameserver 降级为 supplementary，不进核心 Explore；不删除数据。
- 优先级：P1。

### E6. Evidence.rawValue 多为占位符
- 现象：`facts.ts` 写入 `rawValue: { rawCollectionProvider: source }`，未引用具体原始片段；无 source_url/raw_reference。
- 与 2.0 §11 差距明确。
- 建议：Phase 1 为关键 Fact（resolved_ips/tls_certificate/http_status）补 evidence 字段映射（source_url + raw 指针），渐进式。
- 优先级：P2。

### E7. Fact.evidenceIds 截断
- 现象：`evidenceIds.slice(-20)`——超过 20 次更新后早期证据指针丢失。
- 建议：改为 FactEvidence 关联表（证据只增不改）；或明确“证据窗口 20 条”为产品语义并文档化。
- 优先级：P1。

### E8. 无更新时间戳模型
- 现象：除 SearchConnection/SearchProperty 外，多数表无 updatedAt；治理操作（如 OrganizationAlias 状态变更）不可审计。
- 建议：关键治理表（OrganizationAlias、DenyLog、DiscoveryCandidate）加 updatedAt + 操作来源字段。
- 优先级：P2。

### E9. Monitor 无告警投递历史
- 现象：只有 lastNotifiedAt；重试/失败不可追溯。
- 建议：AlertDelivery 表（monitorId, channel, eventIds, status, error, sentAt）。
- 优先级：P2。

### E10. DiscoveryCandidate 无价值反馈字段
- 现象：status=analyzed 后即“结束”，不记录该候选的价值（是否完成/产出多少候选/是否高价值站点）。
- 建议：加 `analyzedValue`（0-100）或保留候选→Target 关联（candidateId on Target）——**“现在必须改”之一**，让 Discovery 形成数据复利。
- 优先级：P1。

### E11. Search 数据未与图谱打通
- 现象：SearchProperty.domain 为字符串，无 FK 到 Target；search 事件走 Target.domain upsert。
- 建议：保持字符串 + 唯一约束（领域正确：搜索属性属“用户资产”而非“实体”）；Phase 4 在 KeywordRanking 处再关联 Target。
- 优先级：P3。

## F. 已有但存在安全问题

### F1.【P0】Discovery probe 绕过 safe-fetch（SSRF）
- 代码位置：`src/lib/discovery/probe.ts`（生产 multi-source 分支）。
- 风险：候选域名（可被用户提交域名间接注入）可触发对任意地址的原始 fetch——内网探测、云元数据（若上云）、DNS rebinding。
- 建议：probe 改用 safe-fetch（resolveSafeAddresses + 钉扎 + 重定向逐跳校验 + body 上限），删除第二套 fetch；补测试。
- 优先级：P0（第 1 优先）。

### F2.【P1】/api/report/[domain] 公开无鉴权、无限流
- 代码位置：`src/app/api/report/[domain]/route.ts`。
- 风险：完整报告 JSON（含 RawCollection 原始数据）可匿名批量拉取；与 v1 付费通道矛盾。
- 建议：加 API Key 或限流 + 降级字段（去掉 rawData 细节），与 v1 合并契约。
- 优先级：P1。

### F3.【P1】SSE 流端点无鉴权
- 代码位置：`src/app/api/analyze/[id]/stream/route.ts`、`/api/analyze/[id]`。
- 风险：CUID 可猜度低，但任何知悉 id 者可看进度/错误；调查 id 已出现在公开页面链接中。
- 建议：进度详情要求 owner 或一次性 token（分析创建时返回）。
- 优先级：P1。

### F4.【P2】注册开放 + 内存限流
- 现象：/api/auth/register 无邮箱验证/验证码；限流表在进程内存，重启清零。
- 风险：垃圾账号、API 滥用。
- 建议：注册加邮件验证或邀请码（产品规模小，P2 观察）；限流若多实例/重启敏感再上 Redis。
- 优先级：P2。

### F5.【P2】safeFetch 允许非标准端口
- 现象：`url.port` 原样连接（地址已校验，但可访问公网主机上任意端口服务）。
- 建议：白名单 80/443/8443 或至少拒绝 1-1023 非标准端口；redirect 同样约束。
- 优先级：P2。

### F6.【P2】AI 解释无成本上限与缓存持久化
- 现象：内存缓存（>500 清空）；每次点击可触发 GLM 调用；无用户级配额。
- 建议：加每日 AI 调用配额 + 持久缓存表（可选）。
- 优先级：P2。

### F7.【P2】搜索凭据与 JWT 共用 AUTH_SECRET
- 现象：AES 密钥由 AUTH_SECRET sha256 派生，无密钥版本；轮换即全部失效。
- 建议：引入 `ENCRYPTION_KEY` 独立密钥 + key version 字段（v1/v2），渐进迁移。
- 优先级：P2。

## G. 已有但存在性能问题

### G1. 候选全量分析成本
- 现状：probe 已降低无效分析；每次完整分析 ~4.6s（实测 334 站）。
- 风险：预算 50/天 × 4.6s ≈ 4 分钟/天计算量，当前可忽略；放大到 200+/天仍可接受。
- 建议：观察即可；若到 500/天再引入并发上限（2-3 并发）与资源隔离。
- 优先级：P3。

### G2. 报告/实体页 N+1 查询
- 现象：`report.ts` 报告组装串行多次查询；`seo/entity.ts` 各实体页 10-20 次顺序查询；`fetchFacts` 全量加载。
- 风险：页面在数据规模增大后变慢；无缓存兜底。
- 建议：Phase 1 用 `Promise.all` 并行 + 关键页加短 TTL 缓存（如 Next `unstable_cache` / 内存 60s），不做复杂缓存。
- 优先级：P2。

### G3. Discovery 每 15s 全表拉取 pending 内存排序
- 现象：`collector.ts` findMany 全量 pending（当前 ~900，可接受）。
- 风险：候选上万后每 15s 一次全表扫描。
- 建议：加 `ORDER BY score DESC NULLS LAST, createdAt` 数据库排序 + LIMIT，或引入 SQL 侧选取。
- 优先级：P2。

### G4. Snapshot / RawCollection 无保留策略
- 现象：每日新增 ~1000+ Snapshot / ~1300 RawCollection；无 TTL/归档。
- 风险：DB 增长（当前 41MB，300 天 ≈ 数 GB）。
- 建议：Phase 1 定义保留策略（Raw 保留 N 天、Snapshot 每 aspect 保留最近 M 条，hash 去重已控量），先只读巡检再执行。
- 优先级：P1。

## H. 规划与当前产品存在冲突

### H1. “不要孤立工具” vs /tools/* 工具页
- 现状：/tools/dns、/tools/ip、/tools/ssl、/tools/technology 独立查询入口。
- 判断：作为“漏斗页”可接受（均导向 /website/[domain] 报告）；但若各自独立深化（如独立 DNS 历史页）则偏离定位。
- 建议：保持漏斗定位，工具结果页一律附“生成完整情报报告”CTA；不再扩展孤立工具。
- 优先级：P2（原则约束）。

### H2. “SEO 是 Website Intelligence 维度” vs 当前 SEO=营销
- 现状：SEO 代码 90% 用于 SiteIntel 自身 SEO（registry/sitemap/OG/质量门）；对目标网站的 SEO Intelligence 仅 Technical SEO。
- 建议：区分两条线，产品 SEO 走 Phase 4；营销 SEO 保持现状。
- 优先级：P2。

### H3. “Insight=解释” vs Insight 含 Hypothesis 字段
- 见 D4。
- 优先级：P2。

### H4. “统一 Source Registry” vs source 硬编码
- 现状：provider names 硬编码于 `providers/registry.ts`；发现源注册于 `discovery/sources/index.ts`；无 registry 表。
- 建议：Phase 1 轻量 SourceRegistry 表（登记 7 provider + cloudflare_radar，含 rate_limit/reliability/last_success/failure），接入统计日志。
- 优先级：P1。

### H5. “质量>数量” vs 当前供给失衡
- 现状：SAN 子域自我繁殖（94%）+ seed 50/天；消费 50/天。
- 建议：Phase 2 实施质量闭环（见 §20）；不无限扩量。
- 优先级：P1。

## I. 规划中暂时不应该开发的内容

1. Graph Database（Neo4j）——当前 PG 足够。
2. 微服务 / Kubernetes。
3. 复杂 AI Agent / 自主修复。
4. 大规模商业化（计费、Bulk 付费）。
5. 一次性创建几十张未来表。
6. 大规模数据库重构/重写。
7. 全维度 Monitoring（SEO/安全/排名）——Phase 6 再议。
8. 完整 Explore / SiteIntel Search——Phase 7。
9. Majestic / URIRANK / Common Crawl 接入——等待 Phase 0/2 质量闭环后确认。
10. 搜索集群（Elasticsearch/Meilisearch 等）。

---

# 15. Database Gap（数据库专项审计）

## 15.1 当前实体关系图（PostgreSQL 逻辑 ER）

```mermaid
erDiagram
    TARGET ||--o{ INVESTIGATION : "analysis runs"
    TARGET ||--o{ SNAPSHOT : "history"
    TARGET ||--o{ EVENT : "changes"
    TARGET ||--o{ INSIGHT : "interpretation"
    TARGET ||--o{ FACT : "current state"
    TARGET ||--o{ SIGNAL : "fact changes"
    TARGET ||--o{ CONTRADICTION : "conflicts"
    TARGET ||--o{ MONITOR : "subscriptions"
    TARGET ||--o{ DOMAIN_OWNERSHIP : "claims"
    INVESTIGATION ||--o{ RAW_COLLECTION : "verbatim"
    INVESTIGATION ||--o{ EVIDENCE : "provenance"
    FACT ||--o{ SIGNAL : "signals"
    FACT ||--o{ EVIDENCE : "supports"
    ENTITY }o--o{ ENTITY : "RELATIONSHIP (source,target,type)"
    USER ||--o{ MONITOR : "owns"
    USER ||--o{ API_KEY : "owns"
    USER ||--o{ SEARCH_CONNECTION : "owns"
    SEARCH_CONNECTION ||--o{ SEARCH_PROPERTY : "properties"
    SEARCH_PROPERTY ||--o{ SEARCH_DAILY_METRIC : "daily"
    SEARCH_PROPERTY ||--o{ SEARCH_QUERY_DAILY : "queries"
    SEARCH_PROPERTY ||--o{ SEARCH_PAGE_DAILY : "pages"
    SEARCH_PROPERTY ||--o{ SEARCH_SUBMISSION : "submits"
    SEARCH_PROPERTY ||--o{ SEARCH_SYNC_JOB : "sync jobs"
    DISCOVERY_CANDIDATE }o--o| TARGET : "rootDomainId (soft, no FK)"
    ENTITY ||--o{ ORGANIZATION_ALIAS : "canonicalization"
    ORGANIZATION_DENY_LOG : "rejected org values"
```

## 15.2 规划 §5 数据库专项 20 问逐题结论

| # | 问题 | 结论 |
|---|---|---|
| 1 | Entity 是否重复定义 | **否**。Entity 统一表 + type/normalized unique；domain/subdomain 语义分离清晰。 |
| 2 | Relationship 是否合理 | **基本合理**。9 类关系 + kind + confidence + evidence；缺 valid_from/to（历史关系不可查）。 |
| 3 | Snapshot/Observation/Event 是否概念重叠 | **部分重叠**。Snapshot=历史快照、Event=变化，边界清楚；但 Snapshot 与“Observation”语义近似（无独立 Observation 概念，可接受）；真正重叠的是 **Event 与 Signal（E1）**。 |
| 4 | Event / Signal 是否重复 | **是（重点）**。事实变化同时写 Signal 与 Event（E1）。 |
| 5 | Insight / Hypothesis 是否重复 | **未重复但语义越界**。无 Hypothesis 表，但 Insight 内嵌 hypothesis 字段（certainty/supporting/contradicting/recommendedNextStep）→ D4。 |
| 6 | Candidate / Investigation 是否合理 | **合理**。Candidate 入池 → Investigation 消费 → Target 落库；缺“分析后价值回填”（E10）。 |
| 7 | Evidence 是否真正存在 | **存在但浅**。Evidence 表覆盖 Fact 级；Relationship/Insight 证据内嵌 JSON；Evidence.rawValue 多为占位（E6）。 |
| 8 | Source 是否统一 | **否**。无 Source Registry；来源硬编码（H4）。 |
| 9 | Confidence 是否统一 | **部分统一**。0-100 Int 贯穿 relationship/fact/evidence/insight，但缺少统一展示/聚合层（source_count、conflict_count 等规划字段未建模）。 |
| 10 | History 能否支持时间维度 | **能支持当前需求**。Snapshot 时间线 + Event + first/lastSeen 可用；Fact 历史与 Relationship 历史缺失（B8）。 |
| 11 | 大量 JSONB 代替正式结构的问题 | **存在但可接受，需约束**。核心 JsonB：Fact.value、Snapshot.data、Event 前后值、RawCollection、Entity.metadata、scoreReason。**原则：只允许“采集原文/展示载荷”用 JSONB；业务状态（Finding/Keyword/Ranking/Action）必须建表**。 |
| 12 | 是否存在重复字段 | **少量**。Event.previous/current 与 Signal.previous/current 重复（E1）；Insight.details 与 evidence 部分重复；Entity.metadata 与 Fact 部分重叠（如 cert 信息）。 |
| 13 | 是否存在孤儿数据 | 08-16 巡检 **0 孤立**（Fact/Snapshot/RawCollection/Entity/Relationship 全绿）；**潜在孤儿**：DiscoveryCandidate.rootDomainId 无 FK（E4）。 |
| 14 | 是否存在缺失索引 | 见 §4.2：Event(targetId,type,detectedAt)、Insight(targetId,status)、EntityRelationship 类型查询、Fact(lastSeenAt) 等缺失。当前规模无影响，规模化前补。 |
| 15 | 是否存在不合理外键 | **无硬伤**。Investigation.userId 无 FK（匿名/删除用户语义，可接受）；DiscoveryCandidate.rootDomainId 无 FK（应补或接受 soft）；Evidence 双可空 FK（investigation/rawCollection）合理。 |
| 16 | 是否存在未来无法扩展的问题 | **基本可扩展**。最大隐患：Fact.value/Snapshot.data 无 schema 版本字段——未来形状变更无法迁移回放；建议加 `schemaVersion`。 |
| 17 | Discovery 数据能否形成长期数据资产 | **目前不能**。候选无价值回填、无分析结果留存、无质量反馈（E10）；有评分与 scoreReason（透明），方向正确。 |
| 18 | Security Finding 如何接入 | 建议新表：`SecurityFinding(targetId, category, severity, confidence, status, firstSeenAt, lastSeenAt, evidenceIds, remediationId)` + `SecurityEvidence` + `Remediation`。现有 Entity/Fact 数据（tls/dns/http/tech）可直接作为 finding 的证据来源；**无需改现有表**。 |
| 19 | SEO Keyword/Ranking 如何接入 | 建议沿用 SearchQueryDaily 的行级思路：`Keyword`（规范化 + 意图）→ `KeywordObservation`（来源/日期/位置/设备）→ `KeywordRanking`（position/change）→ `KeywordOpportunity`（评分）；与 Target 通过 domain 关联，search_engine 用枚举表。**Phase 4 建**。 |
| 20 | Recommendation/Action 是否统一模型 | **是**。Phase 5 建 `Recommendation(targetId, category, priority, impact, effort, evidenceIds, status)` + `ActionItem(recommendationId, title, status, dueAt)` + `ActionVerification`；当前计算式推荐迁移到表。 |

## 15.3 “现在必须改” vs “未来需要预留”

**现在必须改（数据模型层）**
1. Event/Signal 双轨统一（E1）——一次性迁移。
2. DiscoveryCandidate.rootDomainId 加 FK 或纳入巡检（E4）——一行 migration。
3. 9 类 Fact.value 增加形状校验（zod），写入前验证（E2）。
4. DiscoveryCandidate 增加价值回填字段（E10）——先加列，消费端 Phase 2 接。
5. Snapshot/RawCollection 保留策略巡检脚本（G4）——只读出报告，授权后清理。

**未来需要预留（不建表，只留语义）**
- SecurityFinding / Keyword / KeywordRanking / Recommendation 的命名与归属（§18/§19/§20 结论），不提前建表。
- Fact.schemaVersion 字段（一行 migration，值得现在就加）。
- Source Registry（一张小表 + 登记），Phase 1 轻量落地。

---

# 16. Backend Gap

| 项 | 现状 | 差距 | 优先级 |
|---|---|---|---|
| Job System | 无 | 最小 Job/JobRun + 三调度器统一 | P0 |
| 统一网络层 | safe-fetch 主链路 ✅；probe ❌ | probe 并入 safe-fetch | P0 |
| API 契约 | /api/report 公开 JSON | 与 v1 合并/封口 | P1 |
| 数据校验 | 无 | Fact/Snapshot zod 校验 | P1 |
| Source Registry | 无 | 轻量表 + 登记 | P1 |
| 缓存 | 无 | 关键页短 TTL | P2 |
| 密钥管理 | AUTH_SECRET 派生 | 独立 ENCRYPTION_KEY + 版本 | P2 |
| 错误处理 | 失败隔离 ✅、pipeline 恢复 ✅ | 无统一错误码/可观测 | P2 |
| 日志 | console.log / journald | 结构化日志（JSON）+ 请求 id | P2 |
| Rate limit | 内存态 | 多实例前可保持 | P3 |

---

# 17. Frontend Gap

| 项 | 现状 | 差距 | 优先级 |
|---|---|---|---|
| 报告页 | 64KB 单体组件 | 拆分区块组件 | P2 |
| 探索 | 实体页互跳 ✅ | 无全局 Explore / 检索 | P3 |
| Action Center | 报告内联推荐 | 无“下一步做什么”中心 | Phase 5 |
| 健康度解释 | 评分 + 一句话结论 | 无维度解释（为什么 82） | P2 |
| 监控视图 | dashboard 列表 | 无告警历史/趋势 | P2 |
| 候选队列 | 无 | admin 队列仪表盘 | P1 |
| 对比 | 无 | Website Compare | P3 |
| 数据跳转 | IP/ASN/技术/证书 ✅ | Organization↔Website 弱（经 ASN） | P2 |

---

# 18. Security Gap

## 18.1 现有被动数据 → Security Finding 的自然映射

| 现有数据 | 可形成的 Finding | 需要新增 |
|---|---|---|
| TLS 证书 + daysRemaining | tls_expiry、弱签名算法、自签证书 | TLS 协议版本/密码套件采集 |
| Security Headers（5 项存在性） | 缺 HSTS/CSP/XCTO/Referrer-Policy/XFO | 头值解析（max-age、CSP 策略内容） |
| server / x-powered-by + 版本 | 已知漏洞（若版本命中） | CVE 数据库/映射表 |
| 技术栈 + 版本(有限) | Framework/CMS 风险 | 依赖清单采集（JS/CSS 引用） |
| HTTP 跳转链 | 开放重定向迹象、过多跳转 | 目标 URL 校验（已有） |
| DNS MX/NS/TXT | 无 SPF/DMARC、NS 不一致（已有 contradiction） | TXT SPF/DMARC 解析 |
| 证书 SAN / 共享 IP | 证书滥用/共享基础设施风险 | 关系分析规则 |
| 无数据 | 暴露服务、端口扫描、Source Map 泄露 | 被动层不引入端口扫描；Source Map 可用 GET 探测 |

**结论**：现有数据能自然形成 Finding/Evidence/Remediation 链，缺的是“模型 + 规则 + 版本知识库”，不是新扫描能力。Phase 3 建议从“Security Headers 值解析 + TLS 配置 + Source Map 探测”三个被动项起步。

## 18.2 攻击面清单（审计结论）

- ✅ 已加固：主采集 SSRF、webhook SSRF、凭据加密、会话 Cookie、API Key 哈希、限流、Source Map 关闭。
- ❌ 待修：probe SSRF（F1）、/api/report 匿名（F2）、SSE 无鉴权（F3）、端口白名单（F5）。
- ⚠️ 观察：注册无验证（F4）、AI 成本（F6）、密钥共享（F7）。

---

# 19. SEO Gap

## 19.1 Technical SEO 缺口（字段级，Phase 1 可补）

| 字段 | 现状 | 建议 |
|---|---|---|
| canonical | ❌ 未采集 | metadata fact 增加 canonical |
| robots meta（noindex/nofollow） | ❌ 未采集 | indexability 判定 |
| hreflang / OG type / twitter | ❌ | 采集入 metadata fact |
| image alt / 图片数 | ❌ | 低优先 |
| 404 / redirect 目标状态 | ⚠️ 仅跳转链 | redirect 目标 HTTP 状态探测 |
| 页面级索引状态 | ❌ | Phase 4 |

## 19.2 Keyword / Ranking 体系（Phase 4 前不建表，方向已定）

```
SearchQueryDaily（已有行级基础）
   ↓
Keyword（规范化 + intent）
   ↓
KeywordObservation（来源/日期/引擎/设备/位置）
   ↓
KeywordRanking（position + change）
   ↓
KeywordOpportunity（demand × competition × relevance × gap）
```

原则：SEO 必须是 Website Intelligence 维度（挂在 Target/website 页面），不是独立工具；只记录实际可获取数据（Baidu 无公开排名 API 时诚实 absent，规划 §29 已明确）。

---

# 20. Discovery Gap

## 20.1 六个关键问题实测

| 问题 | 现状 |
|---|---|
| 每天发现多少 Candidate | 峰值 +588/天（08-16，SAN 自我繁殖）；现 SEED 50/天 + 内源续供 |
| Candidate 来源 | SAN 94%、CNAME、redirect、共享 IP/证书/跟踪、Cloudflare Radar |
| 去重机制 | Target ∪ Candidate 双守卫 + sourceCount 累计 ✅ |
| Quality Score | 10 因子透明评分 + scoreReason ✅（08-17 上线） |
| Priority | score DESC + 老化加成（≤40）✅ |
| Investigation 消费能力 | 50/天（DISCOVER_DAILY_CAP）；单站 ~4.6s，理论上限远超 |
| Worker 消费能力 | 单进程单飞，15s tick；可扩 |
| 数据库增长速度 | Entity +1500/天（SAN 派生主导）、Candidate 净增数百/天、Snapshot/Raw +1000+/天 |
| Candidate 堆积 | **是**（pending ~870，日净增仍为正） |
| 低价值网站大量进入 | **是**（SAN 子域/tiles 类自我繁殖；probe 已过滤不可达，但“可达但低价值”仍入池） |
| 是否形成数据复利 | **否**——候选无价值回填，无法回答“哪个候选带来了好数据” |
| SEED_DAILY_BATCH 工作方式 | 6h 窗口导入，预算 50/天，去重记账（31/50 对账 PASS） |
| Cloudflare Radar 参与 | Top100 日更，源头过滤基建域，只入池 |
| 反向发现 | 共享实体反向 ✅；数据缺口驱动 ❌ |

## 20.2 质量>数量的具体改造（Phase 2，非“加数量”）

1. **价值回填**：Target 增加 `sourceCandidateId`，候选增加 `analyzedValue`（完成度/产出候选数/后续被引用数）——让队列学会“什么候选值得”。
2. **SAN 降噪**：SAN 候选按“根域去重 + 子域降权”在入池时处理（classifyDomain 已有 tier，入池逻辑补 isInfra/子域过滤），SAN 子域不再批量入池。
3. **去重增强**：同根域候选合并（rootDomainId 已有列，消费时按根域限额）。
4. **队列观测**：admin Discovery 仪表盘（pending/analyzed/skipped、score 分布、source 分布、价值 Top）。
5. **饱和退出**：候选 7 次失败 skipped 保留（已有）→ 增加“分析后低价值”标记（可选，人工复核）。

---

# 21. Data Quality Gap

| 项 | 现状 | 差距 |
|---|---|---|
| 孤立/重复/自环 | 08-16 全绿 | 无自动巡检 Job（一次性脚本） |
| 假阳性 | provider_changed 已治理 | 无回归测试持续防线 |
| 证据链 | 0 断裂 | 无定期证据链校验 Job |
| 新鲜度 | 无 TTL | 无动态 TTL/陈旧扫描 |
| 冲突 | contradiction 4 类 | 规则可扩展 |
| 治理队列 | DenyLog / OrganizationAlias pending | 无 admin 治理队列页（人工复核靠 DB） |

建议 Phase 1：把“数据质量巡检”做成 Job System 的第一个 Job（只读检查 + 生成报告 + 钉钉/Telegram 通知）。

---

# 22. Performance Gap

- 当前规模（289 Target / 3230 Entity / 41MB DB）**远未到性能瓶颈**；服务器 4 vCPU / 7.8GB，load 0.05-0.13。
- 主要风险是**增长形态**（Snapshot/Raw/Candidate 每日千行级）与**查询形态**（N+1、全表排序），不是当前延迟。
- 建议：Phase 1 只做“报告页短 TTL 缓存 + sitemap 静态化权衡”，Phase 2 再评估 Redis/分区。**不要提前引入搜索集群/OLAP**。

---

# 23. Architecture Risks

| # | 风险 | 等级 | 说明 |
|---|---|---|---|
| 1 | 生产代码基线分裂（main / 生产文件 / 本地镜像三态） | 高 | 回滚与审计困难，新开发可能基于错误基线 |
| 2 | 单进程调度器无 Job 记录 | 高 | 重启丢状态、缺陷难复现（monitor 死循环先例） |
| 3 | probe 第二套 Fetch | 高 | SSRF 攻击面 |
| 4 | 双轨事件（Signal/Event） | 中 | 历史数据口径漂移 |
| 5 | JsonB 形状漂移 | 中 | 报告 shape helpers 脆弱 |
| 6 | 候选积压 + SAN 噪声 | 中 | 数据资产信噪比下降 |
| 7 | 无缓存 + force-dynamic | 低-中 | 规模化后页延迟 |
| 8 | 密钥共享/无轮换 | 低-中 | 凭据迁移成本 |

---

# 24. Product Risks

| # | 风险 | 说明 |
|---|---|---|
| 1 | “安全分析”名不副实 | 当前只有被动配置检查；用户若期待漏洞发现会失望——建议 UI 明示“被动安全配置检查” |
| 2 | “SEO 分析”未成型 | 报告 SEO 维度只有 Technical 8 项，无关键词/排名——避免对外宣传 SEO Intelligence |
| 3 | 报告仍是“数据堆积”边缘 | 已有 3A-3C 改善（首屏指标/阅读路径/证据链），但 Recommendation 未落库，无法形成“下一步行动”闭环 |
| 4 | 数据复利未闭环 | Discovery 有量无价值反馈；长期数据壁垒未建立 |
| 5 | 用户留存依赖监控 | 监控仅 Telegram/Webhook；无邮件/站内通知中心 |

---

# 25. Technical Debt

1. `report-page.tsx` 64KB 单体（拆分 P2）。
2. 12 个历史 lint 错误未清理（记录在案）。
3. tsconfig 归一化未做；baidu_verify 大小写双文件；49 个基线排除文件待处置。
4. 12+ 报告/文档未入库（Git 卫生）。
5. `diag_sql/`、`tmp_sql_seedtest_*.sql` 残留未清理（记录在案，不自行删）。
6. 本地镜像落后于生产基线（无 remote，同步需授权）。
7. Organization 历史合并（21 实体）脚本 dry-run 就绪未执行（等授权）。
8. staging（:3004）与 `siteintel_staging_ro` 待删。

---

# 26. P0 问题（系统安全 / 数据正确性 / 生产稳定）

| # | 问题 | 证据 | 处置 |
|---|---|---|---|
| P0-1 | Discovery probe SSRF（第二套 Fetch） | `src/lib/discovery/probe.ts`（生产分支）未走 safe-fetch；DNS 解析不校验、fetch 自行解析、无 body 上限 | 改造为 safe-fetch + 测试；唯一网络层 |
| P0-2 | 生产代码基线分裂 | main=`c74dfd1`，生产跑 multi-source 未合入文件；本地镜像 @1bdfcb4 | multi-source 合入 main；本地同步；以 main 为唯一基线 |
| P0-3 | Event/Signal 双轨记录 | `facts.ts` 同时写 Signal 与 Event | 统一事件源迁移 |
| P0-4 | 无 Job System，调度器 in-process 无状态 | `instrumentation.ts` + 三调度器；monitor 死循环先例 | 最小 Job/JobRun 落地 |
| P0-5 | 候选积压 + SAN 噪声威胁数据资产 | pending ~870，SAN 94%，净增为正 | 入池过滤 + 价值回填 + admin 队列页 |
| P0-6 | /api/report 匿名无限流 | `api/report/[domain]/route.ts` 无鉴权 | 加鉴权/限流或并入 v1 |

# 27. P1 问题（核心产品能力）

| # | 问题 | 处置 |
|---|---|---|
| P1-1 | SSE 流端点无鉴权 | owner token |
| P1-2 | Fact.value 形状无校验 | zod 校验器 |
| P1-3 | DiscoveryCandidate.rootDomainId 无 FK | 补 FK/巡检 |
| P1-4 | Snapshot/RawCollection 无保留策略 | 巡检 + 授权后清理 |
| P1-5 | Technical SEO 字段缺失（canonical/noindex/hreflang） | metadata fact 扩展 |
| P1-6 | Source Registry 缺失 | 轻量表 + 登记 |
| P1-7 | 候选无价值回填 | 加列 + Phase 2 消费 |
| P1-8 | 报告/实体页 N+1 与无缓存 | 并行化 + 短 TTL |
| P1-9 | Organization 历史合并（21 实体）未执行 | 等授权执行（dry-run 已就绪） |
| P1-10 | Fact.evidenceIds 截断丢证据 | FactEvidence 关联或固化窗口语义 |

# 28. P2 问题（数据资产增强 / 体验）

1. 报告页组件拆分。
2. 邮件通知通道 + AlertDelivery 表。
3. safeFetch 端口白名单。
4. AI 调用配额 + 持久缓存。
5. ENCRYPTION_KEY 独立密钥与版本化。
6. 注册验证/风控（观察）。
7. 实体价值分级（core/supplementary/noise）。
8. 关键治理表 updatedAt。
9. 数据质量巡检 Job。
10. admin Discovery 队列页。
11. 健康度维度解释（为什么 82）。
12. Git 卫生收口（12 文档入库、tsconfig、baidu_verify、49 排除文件、删 staging）。

---

# 29. 暂时不要做的事情（明确定义）

1. **不引入 Graph Database / 微服务 / K8s / 搜索集群 / OLAP**。
2. **不接入 Majestic / URIRANK / Common Crawl**——直到 Phase 2 质量闭环完成。
3. **不建 SecurityFinding/Keyword/Recommendation 等未来表**（Phase 3-5 前）。
4. **不做漏洞扫描/端口扫描/任何攻击性检测**。
5. **不把 SEO 做成独立工具站**。
6. **不做完整 Explore/Compare/Action Center**。
7. **不做大规模商业化**。
8. **不无限扩大 Discovery 数量**（先质量后数量）。
9. **不重写现有系统/大规模重构**。
10. **不做 AI Agent / 自主修复**。

---

# 30. 建议开发顺序

```
Phase 0（稳定收口）→ Phase 1（Data Foundation 2.0 最小版）
→ Phase 2（Discovery Intelligence）→ Phase 3（Security Intelligence）
→ Phase 4（SEO Intelligence）→ Phase 5（Growth/Action）→ 后续
```

顺序依据：
- **价值**：数据正确性/安全 > Discovery 质量 > Security/SEO 业务模型 > 前端体验。
- **风险**：P0 安全项必须在任何新功能之前。
- **依赖**：Finding/Keyword 模型依赖 Fact/Evidence 稳定；Job System 依赖一切后台任务。
- **成本**：P0 收口均为小改动（probe 改造、合分支、双轨迁移），成本低于任何新功能。

---

# 31. Phase 1 建议（Data Foundation 2.0 最小版）

**目标：把数据库真正变成资产（规划 §63），但只做最小闭环。**

1. **P0 安全收口**：probe 并入 safe-fetch；/api/report 鉴权；SSE owner token。
2. **数据一致性**：Event/Signal 统一；Fact zod 校验 + schemaVersion；DiscoveryCandidate.rootDomainId FK；证据指针加固。
3. **最小 Job System**：`Job` + `JobRun` 两表；monitor/search/discovery 三调度器统一登记；数据质量巡检作为第一个 Job。
4. **Discovery 质量闭环起步**：入池过滤（子域/基建降权）、Target.sourceCandidateId、候选价值回填列。
5. **Source Registry 轻量表**：7 provider + cloudflare_radar 登记（rate_limit/reliability/last_success/failure）。
6. **保留策略**：Snapshot/Raw 保留策略巡检报告（只读 → 授权后执行）。
7. **工程基线**：multi-source 合入 main、本地镜像同步、deploy.sh 落库、删 staging。

**验证方式**：每项带单元测试 + staging 验证 + 生产验收（沿用既有 3A-3C 验收流程）。

---

# 32. Phase 2 建议（Discovery Intelligence）

1. 候选价值回填消费：analyzedValue → 队列排序加价值权重。
2. 同根域合并/限额；SAN 源降噪规则上线。
3. admin Discovery 队列仪表盘（pending/score/source/价值分布）。
4. 反向发现 v2：数据缺口（缺 cert/缺 geo/缺 tech）→ 定向再分析。
5. 观察期 KPI：pending 增速、低价值候选占比、候选→Target 转化率、每 Target 平均候选产出。

---

# 33. Phase 3 建议（Security Intelligence）

1. 建 `SecurityFinding` / `SecurityEvidence` / `Remediation` / `FixVerification` 四表（Phase 3 才建）。
2. 三条被动规则起步：Security Headers 值解析（HSTS max-age/CSP 内容）、TLS 协议与密码套件采集、Source Map 探测。
3. 版本→已知漏洞映射：先人工维护小知识库（如 WordPress/OpenResty/JQuery 常见 CVE），不接外部收费源。
4. 报告页 Security 区块 + P0/P1/P2 修复计划（复用 audit 的 P0-P2 桶）。
5. Fix Verification：重分析 → 对比 Finding 状态（open/fixed/verified）。

---

# 34. 数据库未来演进建议

- **维持 PostgreSQL 单体**；不分区、不引入图数据库（实体规模 <10 万前）。
- 表级演进顺序：Job/JobRun → SourceRegistry → SecurityFinding 组 → Keyword 组 → Recommendation 组 → AlertDelivery → FactEvidence。
- 每个新业务模型都遵循：**建表（结构化列）+ 时间字段（first/lastSeen）+ 证据指针 + 状态机**，不进 JSONB。
- 索引策略：在 Phase 2 数据规模达到 ~2000 Target 时补 §4.2 缺失索引；每月只读巡检一次。
- 保留策略：Raw 保留 90 天 / Snapshot 每 aspect 保留最近 200 条 / Evidence 只增不改（历史归档）。

---

# 35. 后端未来演进建议

- 保持模块化单体 + 单 worker；进程内调度器升级为 Job 消费器（不引入 Redis，直到多实例需求出现）。
- safe-fetch 成为唯一出站通道（含 probe、种子源、通知、验证）。
- API 契约收敛：公开 JSON 只有 v1（鉴权 + 配额）；页面内部读库不变。
- 日志：console.log → JSON 结构化日志 + 请求 id（成本低）。
- 缓存：先内存/Next 短 TTL；Redis 仅在多实例后引入。

---

# 36. 前端未来演进建议

- 报告页按区块拆分（Security/SEO/History/Recommendations/Evidence），数据不变。
- Website Profile 第一屏增加“健康度解释 + 下一步 3 件事”（Action Center 的 Phase 5 雏形）。
- admin 增加 Discovery 队列页与治理队列页（DenyLog/Alias pending 复核）。
- 监控增加告警历史视图（依赖 AlertDelivery 表）。
- 新增页面都挂在 Website Intelligence 体系下，禁止孤立工具页。

---

# 37. 最终架构建议

```
Next.js 单体（保持）
  ├── App Router 页面 + Route Handlers
  ├── Application Services（lib/ 引擎）
  ├── 最小 Job System（Job + JobRun，in-process consumer）
  ├── 统一网络层 safe-fetch（唯一出站通道）
  ├── 数据层：PostgreSQL（结构化业务表 + 受约束 JSONB 载荷）
  └── 可选：内存短 TTL 缓存（规模化后再上 Redis）
```

**不引入**：微服务、K8s、GraphDB、搜索集群、OLAP、AI Agent。

---

# 38. 风险与注意事项

1. **本报告所有生产数据口径以 08-16/08-17 只读报告为准**；生产 DB 实时状态（今日计数、服务日志细目）需授权后只读核验。
2. 任何生产 DB 变更（即使一条 migration）必须遵循：备份 → dry-run → 影响统计 → 授权 → 执行 → 验证 → 记录。
3. Organization 历史合并（21 实体）、Majestic、SEO Phase 2、删 staging 均处于“等授权”状态，不自动执行。
4. multi-source 合入 main 属于生产基线变更，需先评审（分支仅 16 文件，改动集中在 discovery 层）。
5. 本地镜像 33+ 未跟踪文件为历史报告/文档，保留不清理；入库决策待用户。
6. probe 修复后需生产回归：候选预检链路（probeStatus/score/退避）全套测试。
7. 不要把“安全分析”或“SEO Intelligence”作为对外宣传口径，直到 Phase 3/4 落地。
8. 服务器无 IPv6 路由；DNS 解析返回 IPv6 时 safe-fetch 会优先 IPv4（设计如此），IPv6 站点分析可能 partial——记录在案。

---

# 39. 现在最应该做的 5 件事（按价值/风险/依赖/成本排序）

| 排序 | 事项 | 价值 | 风险下降 | 依赖 | 成本 |
|---|---|---|---|---|---|
| 1 | **修复 probe SSRF（并入 safe-fetch）** | 高（唯一网络层闭环） | 消除最高攻击面 | 无 | 小（1-2 天 + 测试） |
| 2 | **生产基线统一（multi-source 合入 main + 本地同步）** | 高（消除三态分裂） | 回滚/审计风险归零 | 评审 | 小（代码评审 + git 操作） |
| 3 | **Event/Signal 双轨统一 + Fact 校验 + rootDomainId FK** | 高（数据正确性） | 历史口径漂移 | 需 1 次授权 migration | 中 |
| 4 | **最小 Job System（Job+JobRun）+ 三调度器登记 + 数据质量巡检 Job** | 高（后台任务可观测） | 调度缺陷可复现可恢复 | Phase 1 | 中 |
| 5 | **Discovery 质量闭环（入池过滤 + 价值回填 + admin 队列页）** | 高（数据复利起点） | 数据资产劣化 | 依赖 #3/#4 | 中 |

**排序理由**：#1 无依赖且风险最高，先做；#2 让一切后续改动有唯一基线；#3 是任何模型扩展的前提；#4 支撑所有后台任务；#5 开始把“量”变成“资产”。

---

# 40. 第一阶段应该开发什么 / 现在绝对不要开发什么

## 第一阶段应该开发什么（Phase 1 范围）

1. probe 并入 safe-fetch（P0-1）。
2. 生产基线统一（P0-2）。
3. Event/Signal 统一 + Fact 形状校验 + schemaVersion + rootDomainId FK（P0-3 数据一致性子集）。
4. 最小 Job System（Job + JobRun）并把 monitor/search/discovery 登记入册（P0-4）。
5. Discovery 入池过滤与候选价值回填列（P0-5 起步）。
6. /api/report 鉴权 + SSE owner token（P1-1）。

## 现在绝对不要开发什么

1. SecurityFinding / Vulnerability / CVE 映射（Phase 3）。
2. Keyword / Ranking / SearchIntent / ContentGap（Phase 4）。
3. Recommendation / Action 表与 Action Center（Phase 5）。
4. Monitoring 全维度（SEO/安全/排名）、Compare、Explore、全局搜索。
5. Majestic / URIRANK / Common Crawl。
6. 任何“增加发现数量”类改动。
7. 任何攻击性安全检测。
8. 商业化 / 计费。
9. GraphDB / 微服务 / 搜索集群 / AI Agent。
10. 大规模数据库重构或一次性建大量未来表。

---

> **结论**：SiteIntel 的方向与规划一致，底座健康。接下来 6-8 周应专注“安全收口 + 数据一致性 + Job 可观测 + Discovery 质量闭环”，之后才值得进入 Security / SEO 业务模型开发。每一步开发前应回答规划 §75 的 Before/Goal/Scope/Non-goals/Risk/Implementation/Verification/Rollback 八问。

---

*本报告由 Codex 以只读方式生成，未修改任何代码、数据库、环境变量、部署或 Git 状态。所有路径基于本地镜像 `C:\Users\deepo\siteintel`。*
