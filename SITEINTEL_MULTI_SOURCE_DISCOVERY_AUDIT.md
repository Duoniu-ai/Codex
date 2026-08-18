# SiteIntel 多源自动网站发现与分析引擎——现状审计报告

> **SITEINTEL_MULTI_SOURCE_DISCOVERY_AUDIT.md**
> 审计日期：2026-08-17 ｜ 依据：《SiteIntel 多源自动网站发现与分析引擎——Claude Code 执行指令.md》§20/§21/§22/§23/§25
> 性质：**只读白盒审计，零代码修改**。所有结论均附证据与等级（CONFIRMED/HIGH/MEDIUM/LOW/UNKNOWN）。
> 前置证据链：`SITEINTEL_AUTO_PROPAGATION_PRODUCTION_DIAGNOSIS.md`（2026-08-17，25 节完整诊断，根因 F+G 种子闭包）

---

## 一、执行摘要

SiteIntel 生产环境（154.204.176.66:3003，git c74dfd1，systemd `siteintel`）当前的"自动繁衍"是一个**单源、内循环、不可自愈、无可观测性的最小实现**：

| 维度 | 现状（实测） | 指令要求（§1-§18） |
|---|---|---|
| 候选来源 | 仅 5 种**内源**（SAN/CNAME/Redirect/共享 IP/共享 Infra），**零外部种子** | URIRANK + Cloudflare Radar + Majestic Million + Common Crawl 四外部源 |
| 调度 | 72 分钟 1 节拍、**每节拍仅分析 1 条**、每日上限 20 | 连续消费、2 分钟/站目标、稳定优先 |
| 单站耗时 | **completed 平均 4.6s（中位数 2.3s，n=334）** | 假设约 2 分钟（相差约 20 倍——实际资源余量巨大） |
| Probe | **不存在**：pending 直接进完整分析 | 轻量可访问性预检（DNS→TCP→HTTP） |
| 排序 | 静态源优先级 + 老化，无解释 | 透明可解释评分（可访问性/主域/排名/传播潜力/多样性） |
| 失败处理 | **无 Retry、无失败记录、失败即永久丢弃** | 失败降级 + 指数退避 + 保留重探 |
| 主域分级 | 仅 strip www，无子域分层 | P0-P4 主域优先级 |
| 循环 | **种子闭包（2026-08-16 09:48 起候选创建 = 0）** | 多源 → 大量候选 → 分析 → 新候选 → 循环增长 |
| 可观测性 | 无候选池/队列页面（admin 仅 data-quality/seo） | 后台仪表盘，能回答"过去 24h 新增多少独立网站" |

**核心结论**：停摆不是故障而是**设计饱和**（种子闭包）；真正的差距在"输入侧"（无外部源）与"调度侧"（20/天 vs 实测容量 700+/天，差两个数量级）。升级的关键是：**现有 pipeline（分析引擎本体）零改动**，把改造全部放在候选层——多源注入、预检、可解释评分、连续消费、失败自愈，最终以仪表盘闭环。这符合 §19"不要推翻 SiteIntel"与 §25 最终原则。

---

## 二、审计方法与证据链

| 方法 | 范围 | 证据 |
|---|---|---|
| 生产代码通读 | collector.ts(134 行)/candidates.ts(163 行)/domain.ts(83 行)/monitoring.ts/pipeline.ts/prisma schema | 全部关键路径逐行核对 |
| 生产 SQL 只读查询 | step_a~step_g 七组，`sudo -u postgres psql -f` 方式执行 | 537 Investigation / 926 Candidate / 289 Target 全量统计 |
| 生产日志/环境 | systemd 日志、env 键存在性（TELEGRAM_BOT_TOKEN/AUTO_DISCOVER/DISCOVER_DAILY_CAP 均 EXISTS） | 只报 EXISTS/MISSING |
| 服务器资源实测 | `nproc`/`free`/`uptime`/`df` | 4 vCPU / 7.9GB RAM(3.2 用) / load 0.05 / 48G 空闲 |
| 外部数据源实证 | curl 直连 Majestic 官方页、Common Crawl collinfo.json/CDX API、Cloudflare Radar 官方文档、urirank.com 首页 | 4 源访问方式逐一确认（等级分列） |
| 时间线/回归 | git log 传播相关 6 commit + c74dfd1 差异确认（SEO P0 不触碰 discovery/pipeline） | 无回归 |

**证据等级定义**：CONFIRMED=代码行号或 SQL 实测直接证明；HIGH=多证据一致推断；MEDIUM=单一证据或官方文档；LOW=存在但未实证；UNKNOWN=未验证。

---

## 三、现状审计：§20 二十问逐题应答

### Q1. 当前自动繁衍在哪里实现？
- **现状**：`src/lib/discovery/collector.ts`（in-process scheduler，`initDiscoveryCollector` 在 boot 后 5 分钟启动 + `setInterval` 72 分钟节拍，unref）。每节拍从 DiscoveryCandidate 取 1 条 → `startInvestigation` 全量分析。注册入口 `src/instrumentation.ts`。与监控循环（monitoring.ts，30 分钟节拍）、搜索同步（6 小时）并列，**无独立 Queue/Worker/Cron**。
- **证据**：collector.ts TICK_MS=72*60_000；tick() 每节拍 1 条。【CONFIRMED】

### Q2. 当前 Candidate 是什么？
- **现状**：Prisma 模型 `DiscoveryCandidate`：`domain @unique`、`source`、`priority`、`status (pending|analyzed|skipped)`、`createdAt`、`analyzedAt`，索引 `@@index([status, priority])`。全库 926 条：868 pending（813×priority60 + 45×50 + 4×45 + 6×40）、58 analyzed、0 skipped。来源分布：san 869 / cname 45 / shared_infra 7 / redirect 4 / shared_ip 1。
- **证据**：schema.prisma:383-393；step_g SQL。【CONFIRMED】

### Q3. 当前 Domain 是什么？
- **现状**：Prisma 模型 `Target`（即"Domain Entity"）：`domain @unique`、`firstSeenAt`、`lastSeenAt`。全库 289 条，按日新增 57/133/91/8（今日 8 条均为用户手动分析）。子域分布：www 11 / 基建类 0 / 其他 279（基建子域已被过滤，未进入）。
- **证据**：schema.prisma:17-28；step_g。【CONFIRMED】

### Q4. 当前 Investigation 是什么？
- **现状**：Prisma 模型 `Investigation`：`targetId`、`userId`（可空=自动）、`status (running|completed|partial|failed)`、`progress Json`、`error`、`startedAt`、`completedAt`。全库 537 条：333 completed / 202 partial / 2 failed（均"Interrupted by server restart"）。按发起者：用户 227 + 自动 179 + 监控 132（wordpress.org 死循环，见 Q18）。
- **证据**：schema.prisma:40-54；step_g。【CONFIRMED】

### Q5. 当前 Queue 是什么？
- **现状**：**无独立队列表**。`DiscoveryCandidate.status='pending'` 即队列（`@@index([status,priority])` 支持按优先级扫描）。collector 每节拍 `fetchAll` pending → `effectivePriority DESC → createdAt ASC → id ASC` 取 1 条。无入队时间戳以外的排队语义、无状态流转记录。
- **证据**：collector.ts:46 pickNextCandidate 纯函数。【CONFIRMED】

### Q6. 当前 Worker 是什么？
- **现状**：**无独立 worker 进程**。in-process `setInterval` 即 worker；`RUNNING Map`（pipeline.ts:26）保证同域并发去重。语义等价于"单 worker、每 72 分钟消费 1 条"。
- **证据**：collector.ts + pipeline.ts:26。【CONFIRMED】

### Q7. 当前有哪些外部数据源？
- **现状**：**零外部数据源**。所有候选来自分析产物内源：SAN 证书域（priority 60）、CNAME（50）、Redirect（45）、共享 IP（75）、共享证书（90）、共享跟踪（95）、共享基建（40）。无排名/榜单/爬虫类输入。
- **证据**：candidates.ts SOURCE_PRIORITY 全量；候选 source 分布（step_g）。【CONFIRMED】

### Q8. 当前能否直接加入 Seed Source 抽象？
- **现状**：**可以，且底子很好**。`addCandidate(domain, source, priority)`（candidates.ts:54-73）已是统一入口（normalize → stripWww → 三重去重 → upsert，永不 throw），任何新来源只要调它即进入现有队列。但 `source` 目前是裸字符串、无类型层次；需要新增 SeedSource 接口层 + 各源实现 + 导入器，将"外部抓取"与"候选入库"解耦。
- **证据**：candidates.ts:54-73。【CONFIRMED】

### Q9. 当前是否已有 Probe？
- **现状**：**不存在**。collector 拿到 pending 直接 `startInvestigation` 全量分析（7 步 pipeline + 证书/基建/洞察全量采集），无 DNS/TCP/HTTP 预检、无可访问性状态。grep discovery/pipeline 全目录 probe/retry/backoff = 0 命中。
- **证据**：collector.ts tick()；grep 0 命中。【CONFIRMED】

### Q10. 当前是否已有 Domain Normalization？
- **现状**：有基础版。`normalizeDomain`（domain.ts）：去 scheme/userinfo/路径/端口/尾点、拒 IPv4/IPv6、DOMAIN_RE 校验、内置 PUBLIC_SUFFIXES 约 45 条（含 com.cn/co.uk 等）、未知后缀强制 2 段最小；`stripWww` 仅剥 www。**无子域分级**（api./cdn./blog. 与根域同等对待），无 root-apex 归属。
- **证据**：domain.ts 全文；pipeline.ts:44 stripWww。【CONFIRMED】

### Q11. 当前是否已有 Dedup？
- **现状**：**有，且是三重保险**：① normalizeDomain 规范化后再比较；② addCandidate 内 `target.findUnique` + `discoveryCandidate.findUnique` 双查（Promise.all）；③ DB `UNIQUE(domain)` 兜底。实测 DiscoveryCandidate 重复域 = **0 行**；926 候选无一条与 289 Target 重复（analyzed 58 条与 Target 一一对应）。
- **证据**：candidates.ts:54-73；step_g 去重查询 0 行。【CONFIRMED】

### Q12. 当前是否已有 Priority？
- **现状**：有静态优先级 + 老化：源固定分（共享跟踪 95 … 共享基建 40）+ `effectivePriority = priority + min(waitingDays×5, 40)`（老化上限 40）。**无**：可访问性因子、主域级别因子、外部排名因子、多样性因子、传播潜力因子、失败惩罚；且分数不可解释（无"为什么排这里"）。
- **证据**：collector.ts:35、candidates.ts SOURCE_PRIORITY。【CONFIRMED】

### Q13. 当前是否已有 Retry？
- **现状**：**不存在**。分析失败 → 候选 `status='skipped'` 永久退出队列（`status` 枚举仅 pending/analyzed/skipped 三态，无 retry 概念）；监控失败也永不推进 lastRunAt（导致 Q18 死循环）。
- **证据**：collector.ts catch → skipped；monitoring.ts:132 注释"run failed — do NOT touch lastRunAt"。【CONFIRMED】

### Q14. 当前是否已有失败记录？
- **现状**：**极弱**。仅 Investigation 表 `status='failed'` + `error` 文本（全库 2 条，均为重启中断，说明真正的分析失败反而大多以 partial 收场）；候选层面无 failureCount/lastFailureReason/nextRetryAt 列。即"失败"不可见、不可追溯、不可恢复。
- **证据**：schema Investigation 字段；0 skipped 行。【CONFIRMED】

### Q15. 当前是否已有繁殖深度限制？
- **现状**：**无显式深度限制**（无"第 N 代停止"逻辑）。实际约束只有两个隐式项：① 每日 20 条节流（`DISCOVER_DAILY_CAP` env，上限 200）；② **种子闭包**——内源候选全部来自已分析站点，当已分析站点不再产出新候选时，繁殖自然停止（这正是停摆根因，Q18）。
- **证据**：collector.ts daily budget；诊断报告根因 F+G。【CONFIRMED】

### Q16. 当前调查平均耗时是多少？
- **现状**：**completed 平均 4.6s、中位数 2.3s、最大 25.4s（n=334）；partial 平均 9.9s、中位数 5.2s、最大 50.4s（n=202）**；今日 25 次运行 1-27s/次。即完整分析远比指令假设的"约 2 分钟"快。
- **证据**：step_g SQL 聚合（completedAt - startedAt）。【CONFIRMED】

### Q17. 当前每天实际可以分析多少网站？
- **现状**：配置 20/天（env 上限 200）。实测容量：服务器 4 vCPU / load 0.05（基本空闲）/ 内存 7.9GB（3.2 用）；按实测 4.6s/站、24h 连续、单并发，**理论容量 ≈ 18,000 站/天**；即使按指令的 2 分钟预算，720 站/天毫无压力。当前实际 08-17 全天 25 次调查。**瓶颈 100% 在调度配置，不在资源。**
- **证据**：step_g + 服务器实测。【CONFIRMED】

### Q18. 当前自动繁衍为什么停止？
- **现状**：**设计饱和（种子闭包），非故障**（诊断报告根因 F+G）：
  - 候选创建最后一条：2026-08-16 09:48:37（blog.csdn.net → 2734556a.csdn.net.cname.yunduns.com），此后 **0 条新候选**；
  - 08-16 09:48 后 7 次自动/手动分析全部 0 输出（已分析站点的 SAN/CNAME/Redirect/共享基建不再产生任何未见过域）；
  - 决定性测试：wordpress.org 最新证书 2 个 SAN、pm.sitevance.tw 3 个 SAN，**全部已存在（0 接受）**——内源提取器逻辑正确，是"池子干了"；
  - 用户六宫格数字为 08-14 旧首页快照（84 恰为 08-14 调查数），3A 首页（605f8ab）已移除统计栏。
- **另发现独立 P0**：monitoring.ts 死循环——因"投递失败不推进 lastRunAt"（:132/:148），3 用户均无通知通道 → wordpress.org 自 08-15 04:39 起每 30 分钟重复全量分析（141 次），每次重报同样的 10 事件 + 5 洞察，RawCollection 堆积 142 行证书数据。
- **证据**：诊断报告全证据链（step_a~step_f）；monitoring.ts:22-23/58-69/132/148/179/188。【CONFIRMED】

### Q19. 当前哪些能力可以直接复用？
- **现状**：可直接复用（零改动）：
  1. **addCandidate 统一入库入口** + 三重去重（新来源天然复用）；
  2. **normalizeDomain/stripWww** 规范化（子域分级在其上增量扩展）；
  3. **pending 队列 + @@index 查询**（只需扩展取数逻辑）；
  4. **RUNNING 并发防重**（多源多 worker 安全）；
  5. **executePipeline 7 步分析引擎本体**（不动）；
  6. **extractCandidatesFromBundle**（⑦.5 闭环产出，不动）；
  7. **DISCOVER_DAILY_CAP env 节流**（连续消费沿用同一预算语义）；
  8. **用户输入立即执行路径**（用户分析不排队、天然最高优先，§16 已满足）；
  9. **rawData 原始数据存证**（RawCollection，审计可追溯）；
  10. **全部 Prisma 模型**（新能力只在现有表上加列，不建重复表）。
- **证据**：上述文件行号。【CONFIRMED】

### Q20. 最小改造方案是什么？
- **现状**：见第十一节 Phase 0→7。**最小起步** = ① 修 monitor 死循环（Phase 0，P0 级）；② 接入第一个外部种子源（URIRANK 或 Majestic 取其一，Phase 1/3）打破种子闭包，候选复用现有 pending 队列流入分析。两者都不触碰 pipeline 本体。
- **证据**：第十一节全表。【CONFIRMED】

---

## 四、问题清单（按严重度排序）

| # | 级别 | 问题 | 证据 | 影响 |
|---|---|---|---|---|
| P0-1 | P0 | monitor 死循环：wordpress.org 每 30 分钟重分析 ×141 次，重复事件/洞察刷屏 | monitoring.ts:132/148；141 次计数 | 浪费吞吐、数据污染、监控功能实际失效 |
| P0-2 | P0 | 种子闭包：候选创建 08-16 09:48 后归零 → 自动繁衍**永久停滞**（产品核心目标失效） | 诊断报告 F+G | 系统变为"手动分析器" |
| P1-1 | P1 | 无 Probe：不可达/低质域直接消耗完整分析资源，产出低、成本高 | grep 0 命中 | 资源浪费、转化率低（58/926=6.3%） |
| P1-2 | P1 | 无 Retry/失败记录：瞬时故障（DNS 抖动等）即永久弃候选，且不可追溯 | collector.ts catch→skipped | 候选损耗、无自愈 |
| P1-3 | P1 | 无子域分级：种子放大后 api./cdn./blog. 子域将与根域平等竞争，冲淡数据质量 | domain.ts 仅 stripWww | 独立网站口径失真 |
| P1-4 | P1 | 优先级不可解释、无外部信号：队列近似先进先出，"最值得分析的"无排序依据 | collector.ts:35 | 资源错配 |
| P2-1 | P2 | 无可观测性页面：无法回答"过去 24h 新增多少独立网站"（§22 核心） | admin 仅 data-quality/seo | 运营盲区 |
| P2-2 | P2 | 吞吐配置 20/天 vs 实测容量 700+/天：数量级浪费 | Q16/Q17 | 增长太慢 |

---

## 五、根因

1. **根因 A（停摆）——种子闭包**：候选全部来自"已分析站点"的内源。去重三保险使池子只出不进；当已分析集收敛（SAN/CNAME/共享关系全部命中）时，新候选 = 0。**设计使然，非代码故障**。修复不在 pipeline，在**输入侧**（外部种子）。
2. **根因 B（低吞吐）——调度设计**：72 分钟 × 1 条/节拍 = 20/天，而实测 4.6s/站、服务器 load 0.05。调度粒度与执行成本差 2 个数量级。
3. **根因 C（无自愈）——失败即弃**：skipped 永久化 + 无失败字段 + 监控 lastRunAt 不推进，短暂故障即永久损失。
4. **根因 D（无排序信号）——无 Probe/排名/多样性数据**：队列无从分辨"高价值可访问新站"与"僵尸不可达域"，FIFO 近似。

---

## 六、架构方案

> 原则：**分析引擎本体（pipeline 7 步 + 实体关系 + 洞察）零改动**；全部新能力落在"候选层"。

```
┌────────────────────────────── 外部种子（§2 四源）──────────────────────────────┐
│  URIRANK         Cloudflare Radar    Majestic Million    Common Crawl           │
└──────────┬──────────────────┬──────────────────┬──────────────────┬─────────────┘
           ▼                  ▼                  ▼                  ▼
      ┌─────────────────────── SeedSource 抽象（§3 可插拔）────────────────────────┐
      │  interface SeedSource { name; fetchIncrement(budget): RawSeed[] }         │
      │  URIRankSource / CloudflareRadarSource / MajesticSource /                 │
      │  CommonCrawlSource / FutureSource        （无 if source=="urirank"）      │
      └──────────────────────────────┬────────────────────────────────────────────┘
                                     ▼ 增量导入（§18 规模控制：日批上限、去重、采样）
      ┌────────────────────────── 统一 Candidate Pool（§4）───────────────────────┐
      │  DiscoveryCandidate（扩展现表，不建新表）                                   │
      │  pending ─► probed ─► analyzed   │   failed ─(指数退避)─► 重新 pending     │
      └──────────────────────────────┬────────────────────────────────────────────┘
                                     ▼
      ① Domain Normalization（§5） → ② Root Domain Priority（§6，P0-P4 判定）
      ③ Lightweight Probe（§7：DNS→TCP/TLS→HTTP→Status→Redirect→TTFB→可抓取）
      ④ Priority Score（§9-§10，可解释）→ ⑤ 分析队列（§14 连续消费）
                                     ▼
      startInvestigation（用户输入直连此入口，最高优先，§16 不动）＋ Monitor（Phase 0 修复）
                                     ▼
      executePipeline 7 步 + ⑦.5 extractCandidatesFromBundle（现成，零改动）
                                     ▼
      Target / Entity / Relationship / Snapshot / Insight / Event
                                     │
                    新域（san/cname/redirect/共享关系）──► 回到 Candidate Pool（闭环）
```

**闭环要点（§15）**：外部种子 → 池 → 规范化 → 去重 → 主域分级 → 轻量预检 → 评分 → 队列 → 调查 → 实体关系 → 新域发现 → 回池。每轮分析既是消费也是生产，且外部种子持续供给，闭包永不成立。

---

## 七、数据模型方案（全部扩展现表，零新表，§19）

### DiscoveryCandidate 加列（一次 migration，全部可空带默认值）

| 新列 | 类型 | 用途 |
|---|---|---|
| `domainType` | String? `apex\|www\|subdomain` | 主域分级（§6） |
| `rootDomainId` | String? → Target.id | 子域归属根域（P2/P3 排序与统计） |
| `probeStatus` | String? `pending\|ok\|fail` | 轻量预检状态（§7） |
| `lastProbeAt` | DateTime? | 最近预检时间 |
| `lastStatusCode` | Int? | 最近 HTTP 状态 |
| `probeLatencyMs` | Int? | 预检总耗时（DNS→HTTP） |
| `failureCount` | Int @default(0) | 分析失败次数（§8 降级依据） |
| `lastFailureReason` | String? | 失败原因文本 |
| `nextRetryAt` | DateTime? | 指数退避下次可重试时间 |
| `sourceCount` | Int @default(1) | 被多少来源发现（§17） |
| `sourceRank` | Int? | 各源统一排名位（跨源可比） |
| `sourceCountry` / `sourceCategory` | String? | 多样性维度（§13） |
| `urirankRank` / `cloudflareRank` / `majesticRank` / `commoncrawlSeen` | Int? / Int? / Int? / Boolean? | 外部排名信号（§11，一域多信号） |
| `score` | Int? | 优先级总分（§9-§10） |
| `scoreReason` | Json? | 可解释文本数组（"为什么排这里"） |
| `metadata` | Json? | 源特定原始信息（国家/分类/快照链接） |

新索引：`@@index([status, priority, nextRetryAt])`、`@@index([probeStatus])`、`@@index([rootDomainId])`。

### Target 加列

| 新列 | 类型 | 用途 |
|---|---|---|
| `isApex` | Boolean @default(true) | 独立网站口径（§23 KPI-1 的精确定义） |

Insight / Event / Snapshot / Relationship / RawCollection **完全不动**。

---

## 八、调度方案（§14 连续消费）

| 项 | 现状 | 方案 |
|---|---|---|
| 节拍 | 72 分钟固定 | **完成一条后间隔 `DISCOVER_TICK_MS`（默认 10-30s）再取下一条**，预算内连续消费；实测 4.6s/站 → 等效每小时 ~5-25 站（由日预算约束） |
| 并发 | 1（RUNNING 防重） | 保持 1（稳定优先 §14；4 核服务器单并发已足够，需要时可 env 提 2） |
| 每日预算 | `DISCOVER_DAILY_CAP`=20（上限 200） | **沿用同一 env，逐步调大**（20 → 50 → 100 → 200 阶梯灰度，每档观察 48h） |
| 用户优先（§16） | 用户输入直接执行、不排队 | **保持不动**（天然最高优先，自动消费永不抢占用户路径） |
| 失败退避 | 永久弃 | 指数退避：1h → 4h → 12h → 24h → 72h → 168h，7 次后暂停（保留记录，不删除）；每次失败 `failureCount+1`、分数 −15×count |
| 不可达域 | 无概念 | 预检失败 → `probeStatus=fail`、降权、按 nextRetryAt 定期重探（§8：不可达 ≠ 永久删除） |
| Boot 恢复 | 5 分钟延迟已有 | 保持；RUNNING 随重启清空自动恢复 |

**节拍事实修正**：指令假设"约 2 分钟/站"；实测 completed 平均 4.6s。实施时以实测值参数化，**容量目标定在 ≥5 站/小时持续（§23 KPI-2），远低于理论上限**，留足安全余量。

---

## 九、评分方案（§9-§10 透明可解释）

```
score = 100×可访问性(0-100)
      + 40×主域级别(P0 根域100 / P1 www90 / P2 独立子域70 / P3 普通子域50 / P4 基建10)
      + 30×新站奖励(未分析1.0 / 已分析按天数衰减)
      + 30×外部排名信号(min(1, log10(topN)/log10(1e6)))  ← URIRANK/CF/Majestic 取最优
      + 25×传播潜力(该根域历史 新候选产出数/调查数，0-1)
      + 20×数据多样性(国家/行业/技术栈配额余量)
      − 20×近期分析惩罚(≤7 天内再分析)
      − 15×failureCount(失败惩罚)
      − 10×近亲域惩罚(与已分析域同根/同 IP 簇)
```

- 每候选落库时同时写 `scoreReason[]` 文本数组，逐项说明加减分原因（"HTTP 200，P0 根域，URIRANK top 5k，历史产出 3 候选"）→ **任何候选都能回答"为什么排这里"**；
- 现有 `priority` 列保留（映射为 rootPriority 项），aging（+5/天 cap 40）保留，新评分在其上叠加；
- 可访问性权重最高（§7：先问"现在是否值得消耗完整分析资源"）。

---

## 十、数据源接入方案（§2 四源，2026-08-17 实证）

| 源 | 访问方式（实证） | 数据规模/频率 | 许可 | 证据等级 | 风险 |
|---|---|---|---|---|---|
| **URIRANK** | 网页 Top-100 列表（全球/国家）；**首页实证无公开批量 API**，需抓取列表页 | 100-200 域/批 | 页面抓取需核对条款 | LOW | 排名质量一般（Alexa 替代类） |
| **Cloudflare Radar** | REST API：`GET /radar/ranking/top?name=top`（全球/国家 Top 100，日更）+ `GET /radar/datasets?datasetType=RANKING_BUCKET`（桶 200→100万，周更，2023 起有存档）；Bearer token（免费 CF 账号）。**官方文档逐字实证** | Top100 日更 / 桶周更 | CF 条款（署名/重分发限制实施前核对） | CONFIRMED | token 需用户提供；条款 |
| **Majestic Million** | `downloads.majestic.com/majestic_million.csv`（官方报告页实证链接）；CSV 列：GlobalRank/TldRank/Domain/RefSubNets/RefIPs/IDN/Prev* | Top 1M，免费版周更 | 免费非商业 + 署名；商用需采购 | HIGH | 免费版商用受限 |
| **Common Crawl** | CDX API `index.commoncrawl.org/CC-MAIN-2026-30-index`（collinfo.json 实证，2026-07 索引在线）；Web Graph/domain graph（data.commoncrawl.org） | 每月快照 | 开放数据（CC0 类） | CONFIRMED | 量大需采样；旧快照滞后 |

**接入策略（§18 规模控制）**：
- 每源实现 `SeedSource.fetchIncrement(budget)` → 统一 `RawSeed[]` → normalize → 三重去重 → 入库；
- **日批量上限**：`SEED_DAILY_BATCH` env（默认 500 总/日、单源 200）；绝不一次性灌入 1M；
- 外部候选入池后**先 Probe 再进分析队列**（§7），不可达者占池不占分析资源；
- Common Crawl 附加用法：**从已有 Target 反查 CDX 反链** → 天然"分析 → 新域"闭环增强（属 Phase 6 传播潜力数据）；
- 排名信号仅作**评分加分项**，不作强制规则（§11：一域多信号，source_count 累计）。

---

## 十一、实施阶段（§21，含退出标准）

| Phase | 目标 | 主要改动 | 验证/退出标准 |
|---|---|---|---|
| **Phase 0**（前置 P0 修复） | 停止 monitor 死循环 | 推荐：代码修复——投递失败也推进 lastRunAt 并累计投递失败计数；备选：为任一用户配置通知通道或停用无通道 monitor | 48h 内 wordpress.org 重分析 ≤1 次；监控恢复正常语义 |
| **Phase 1** | 恢复自动繁衍（打破种子闭包） | 接入首个外部源（URIRANK 网页源，最小实现：SeedSource + 导入器 + 日批上限），候选流入现有 pending 队列 | 连续 72h：新候选 ≥30、新 Target ≥10、无手动干预 |
| **Phase 2** | 统一 Candidate Pool | 第七节 schema 迁移 + 状态机（pending→probed→analyzed / failed+退避）+ 管理页只读视图 | 候选 8 项指标可查；迁移后现有 926 候选无数据丢失 |
| **Phase 3** | 多源 Seed | Cloudflare Radar + Majestic + Common Crawl 三导入器；sourceCount/rank/国家字段写入 | 4 源全部注入；累计新候选 ≥500；0 重复 |
| **Phase 4** | 主域分级 + Probe + 评分 + Retry | Root Domain Priority 判定器（DNS/HTTP/Content-Type/Title/Redirect/技术栈/hostname 组合，非纯字符串）；轻量 Probe 步骤（DNS→TCP/TLS→HTTP→Status→Redirect→TTFB→可抓取性）；评分函数 + scoreReason；指数退避 | 可访问候选比例 ≥60%；失败重试生效（可观察 nextRetryAt 推进）；分数可解释 |
| **Phase 5** | 持续 Queue + Worker | collector 重写为连续消费（第八节）；DISCOVER_DAILY_CAP 阶梯调大 | 吞吐 ≥5 站/小时持续 72h；服务器 load < 1.5；用户分析延迟无感知变化 |
| **Phase 6** | 传播潜力 + 多样性 + 外部排名信号 | 历史产出统计接入评分；国家/行业配额；rank 信号入 score | 来源多样性 ≥6（4 外 + ≥2 内）；候选→调查转化 ≥40% |
| **Phase 7** | 自动增长 Dashboard | admin 新页：§22 全指标 + "过去 24h 新增独立网站"大字指标 | 页面可实时回答该问题；§23 十 KPI 全部可查 |

**灰度纪律**：每 Phase 独立提交、独立验证、可单独回滚（git 基线 c74dfd1 之上）；多源开关 `MULTI_SOURCE_ENABLED` env 门控；无用户确认不进入下一 Phase。

---

## 十二、可观测性（§22 逐项 → 数据来源）

| 指标 | 数据来源 |
|---|---|
| Candidate Total / Pending / Probed / Accessible / Inaccessible | DiscoveryCandidate 计数（probeStatus/status） |
| Queue Pending / Running / Success / Failed | Investigation status + DiscoveryCandidate status |
| Investigations Today / Hour / Avg Duration | Investigation startedAt/completedAt |
| New Domains / Entities / Relationships Today | Target / Entity / Relationship firstSeenAt 当日 |
| Propagation Success Rate / Candidate Acceptance Rate / Accessibility Rate | 派生指标（成功调查/调查总数；analyzed/总量；probe ok/probed） |

**核心指标（§22 要求必须能回答）**：`SELECT count(*) FROM "Target" WHERE "firstSeenAt" >= now() - interval '24 hours'`——Phase 2 引入 isApex 后精确化为"新增**独立**网站"（exclude 子域/基建）。

---

## 十三、风险与边界

| # | 风险 | 缓解 |
|---|---|---|
| R1 | **回归破坏现有系统**（§19 最高优先） | pipeline 零改动；新能力全部 env 门控；每 Phase 独立提交/回滚；本地镜像 c74dfd1 与生产对齐可随时还原 |
| R2 | 资源耗尽（4 核 8G） | 并发 1；日预算上限；实测容量（load 0.05）证明余量 ≥2 个数量级；load 阈值告警 |
| R3 | 外部种子质量差（站群/垃圾域） | 排名信号仅加分；失败惩罚 + 指数退避天然淘汰；Probe 过滤；人工封禁清单 |
| R4 | 数据许可（Majestic 免费版非商用 / CF 条款 / URIRANK 抓取） | 实施前逐源核对条款并记录到代码注释；商用需要时采购授权；Common Crawl 无风险 |
| R5 | DB 增长（种子导入百万级） | §18 日批上限 + 去重后实际入库量小（估算每日增量 <1 万行 × ~1KB）；按月归档策略 |
| R6 | "独立网站"口径漂移 | isApex/domainType 字段化口径；Dashboard 同时展示 总量/apex 数 |
| R7 | Phase 0 monitor 修复方案被否 | 备选方案（配置通道/停用）任一即可止血；不影响主线 |

---

## 十四、验收标准（§23 十 KPI：基线 → 目标）

| # | KPI | 基线（2026-08-17 实测） | 目标（Phase 7 完成） |
|---|---|---|---|
| 1 | **新增独立网站数量**（主 KPI） | 57/133/91/8/今日 0（且当前停摆） | **≥30/天 连续 7 天**（apex 口径） |
| 2 | 每小时成功分析数 | 自动时段 4-7/时 | ≥5/时 持续 72h |
| 3 | 网站分析成功率 | completed/总数 = 62%（333/537；partial 另有 202） | completed ≥70% |
| 4 | 可访问候选比例 | 无 Probe，无基线 | ≥60% |
| 5 | 候选 → Investigation 转化率 | 6.3%（58 analyzed / 926） | ≥40% |
| 6 | Investigation → 新候选转化率 | ~1.7 候选/调查（历史累计） | ≥0.5 新候选/调查（1 周滑动） |
| 7 | 平均分析耗时 | 4.6s（completed） | ≤15s（含 Probe 后仍远低于 2 分钟预算） |
| 8 | 自动繁衍持续时间 | 上次 08-16 09:48 停摆 | ≥30 天连续不停摆（0 候选日 ≤2 天） |
| 9 | 每站平均新候选数量 | — | ≥0.3/站（滑动窗口） |
| 10 | 数据来源多样性 | 5 内源、0 外源 | ≥6 源（4 外 + ≥2 内），外部候选占比 ≥30% |

**交付物（§22+§23）**：admin Dashboard 一页展示以上全部指标，首页大字显示"过去 24 小时新增独立网站：N"。

---

## 十五、结论与待确认事项

1. **审计结论**：现状是可升级的最小实现——三重去重、统一入库、规范化、并发防重、用户优先路径、日预算节流都是**高质量地基**；缺的是输入侧（0 外部源）、调度侧（20/天 vs 容量 700+/天）、决策侧（无 Probe/评分/失败自愈）与可见侧（无仪表盘）。
2. **停摆定性**：设计饱和（种子闭包），非故障；恢复 = 注入外部种子，而不是修 pipeline。
3. **最大杠杆**：Phase 0（monitor P0 修复，1 处代码）+ Phase 3 首个外部源 = 立刻恢复增长；Phase 4/5 = 质量与吞吐；Phase 7 = 可回答产品主指标。
4. **零修改声明**：本审计全程只读（代码阅读 + SQL 查询 + 外部文档实证），未对生产代码、数据库结构、配置、部署做任何变更。
5. **待确认（进入实施前需用户批准）**：① Phase 0 修复方案（推荐代码修复 vs 配置通道）；② 四源接入顺序与日预算初值（建议 50）；③ 每 Phase 退出标准是否接受；④ Dashboard 口径（apex 独立站优先）。

> 按 §25 第四步：**等待确认后再开始代码修改。**
