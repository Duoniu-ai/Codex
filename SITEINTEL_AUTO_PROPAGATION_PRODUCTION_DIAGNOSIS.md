# SiteIntel 自动繁殖生产故障诊断报告

- **日期**: 2026-08-17
- **性质**: 生产环境只读故障诊断（Read-Only Incident Diagnosis）
- **指令依据**: `/www/wwwroot/siteintel.cc/SiteIntel 自动繁殖停止——生产环境只读故障诊断指令.md`
- **执行范围**: 仅阅读代码/配置、查询数据库/日志/Git、对比线上行为。**零修改**（未改业务代码/DB/环境/Cron/Worker/部署/配置；未重启；未顺手优化）
- **证据等级**: 每个结论标注 CONFIRMED / HIGH / MEDIUM / LOW / UNKNOWN

---

## 1. 执行摘要

**核心结论：自动繁殖链路没有"卡死"，而是在 `2026-08-16 09:48:37` 后进入「候选产出为零」的饱和态。**

- 链路各环节（触发器/调度器/消费 Worker/调查管道/实体提取/洞察引擎）**全部正常运行**，唯独最后一环——候选发现（candidate discovery）——持续产出 0。
- 停摆后（08-16 09:48 → 08-17 06:29）共执行 **7 次**自动分析，**全部 0 新增候选**；每次分析提取代码都真实执行（journal 日志为证），但提取出的域名**全部已在 Target 或 DiscoveryCandidate 表中**（去重命中率 100%）。
- 决胜证据（§12）：wordpress.org 证书 SAN = `{*.wordpress.org, wordpress.org}` → 通配跳过 1 条、`wordpress.org` 已是 Target → **accepted = 0**；pm.sitevance.tw SAN = `{sitevance.tw, pm.sitevance.tw, *.pm.sitevance.tw}` → **accepted = 0**。
- **性质：设计性饱和（种子闭包），非代码故障、非回归。** 系统唯一的新域来源是"已分析域名的 SAN/CNAME/重定向/实体图邻居"；当已分析集合的自证闭包 ⊆ 已知集合时，产出必然为 0。增长只发生在"新种子注入"时（最后一次大注入 = 08-16 08:16-08:23 的手动品牌批量分析：reuters.com 单次 +131 候选、paypal +93、airbnb +78…）。
- **用户引用的六宫格数字（58/84/511/891/726/0）是 08-14 晚间旧首页的实时渲染快照**（84 = 08-14 当日调查总数，精确相等；洞察 0 = Insight 表当时为空——引擎 08-15 才产出首行）。3A 用户化首页（commit 605f8ab，08-16 09:16 上线）已移除该统计条，当前首页（`已分析 289 个网站`）与数据库实时一致。当前真实数据：**289 Target / 537 调查 / 3230 实体 / 5888 关系 / 3147 快照 / 65 事件 / 58 洞察**。
- **独立发现（P0 级资源缺陷，与繁殖停摆无关）**：wordpress.org 监视器死循环。3 个用户（test@siteintel.cc、gaoye2020@gmail.com、353876401@qq.com）**全部未配置任何通知通道**（notifyTelegramChatId / notifyWebhookUrl 均 NULL）→ 每次投递失败 → `lastRunAt` 永不推进 → **每 30 分钟全管道重跑一次**，自 08-15 04:39 起已空转 **141 次**（08-15: 59 / 08-16: 65 / 08-17 持续，今日至 06:29 已 15 次），每次重报同样的 10 事件 + 5 洞察。

**根因分类（§20 五选一/复合）**：**F（Discovery 无产出）+ G（Dedup 饱和）复合**，机制按设计工作；另附 **K（Multiple）**：monitor 死循环为独立调度逻辑缺陷。

---

## 2. 当前生产版本

| 项 | 实测 | 证据等级 |
|---|---|---|
| Git HEAD (production master) | `c74dfd1`（本会话早期对齐验收完成，16/16 PASS） | CONFIRMED |
| GitHub main | `c74dfd1` | CONFIRMED（前序验证，本会话零远程写） |
| Production running code | `.next` 构建产物 = c74dfd1（SEO P0），2026-08-17 05:23:06 UTC 启动，`NRestarts=0`，PID 677080 | CONFIRMED |
| Working tree | clean（仅 tsconfig.json 空白行尾差异，已定性为零修改） | CONFIRMED |
| Database | `siteintel` @ PostgreSQL 14（localhost，Prisma） | CONFIRMED |
| Scheduler | 进程内（`src/instrumentation.ts`）：monitoring.ts 30-min tick + collector.ts 72-min tick + search/sync | CONFIRMED |
| Queue | **无外部队列**（无 Redis/BullMQ/DB job 表）；`DiscoveryCandidate` 表承担待办队列语义 | CONFIRMED |
| Worker | 单 systemd 进程（`siteintel.service`），全部逻辑 in-process | CONFIRMED |
| Deployment | Git bundle 通道 + systemd restart；最近一次上线 = SEO P0（08-17 05:23:06） | CONFIRMED |

**结论：生产版本与诊断前提一致（c74dfd1），无版本漂移。**

---

## 3. 当前线上数据状态

| 指标 | 用户引用的旧首页数值 | 当前实际（08-17 06:29 快照） | 说明 |
|---|---|---|---|
| 已分析域名 | 58 | **289**（Target 总数） | 58 = 08-14 晚间 Target≈58，或已分析候选数（恰好 58） |
| 调查次数 | 84 | **537**（08-14:84 / 08-15:243 / 08-16:186 / 08-17:24） | 84 = 08-14 当日总数，精确相等 |
| 实体 | 511 | **3230** | 08-14 时代数值 |
| 关系 | 891 | **5888** | 08-14 时代数值 |
| 快照 | 726 | **3147**（08-14:740 / 08-15:1401 / 08-16:933 / 08-17:73） | 726 = 08-14 晚间的即时值（当日终值 740） |
| 洞察 | 0 | **58**（08-15:12 / 08-16:39 / 08-17:7） | 08-14 时 Insight 表为空；引擎 08-15 才产出首行 |

- 当前线上首页实测（curl https://www.siteintel.cc/，HTTP 200，105 KB）：3A 用户化首页，信任行"已分析 289 个网站"= 实时 `prisma.target.count()`（`src/app/page.tsx:18-25`），4 张示例卡计数全部实时查询——**无六宫格统计条**。
- 缓存头：`cf-cache-status: DYNAMIC` + `Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate` → **Cloudflare 不缓存首页，不存在陈旧 HTML 分发**（CONFIRMED）。
- 六宫格移除提交：`605f8ab feat(homepage): SiteIntel 2.0 Phase 3A - user-centric homepage`（2026-08-16 07:42 commit，~09:16 部署上线）。用户在旧版首页时代看到的数值即上述 6 项。

---

## 4. 自动繁殖架构

```
Seed Domain（手动/自动/API）
   ↓
startInvestigation (src/lib/pipeline.ts)          ← RUNNING map 防并发重复
   ↓
runPipeline → upsertTarget → Investigation(running)
   ↓
executePipeline（单进程内顺序执行）
   ① validate → ② collect providers（dns/rdap/http/ssl/metadata/technology/ip，RawCollection 535 行）→
   ③ extractEntities（Entity/EntityRelationship 表）→ ④ infrastructure → ⑤ snapshots/compare（Snapshot 表）→
   ⑥ runInsightEngine（Insight 表）→ ⑦ finalize
   ⑦.5 extractCandidatesFromBundle (src/lib/discovery/candidates.ts)   ← ★ 繁殖最后一环
   ↓
addCandidate 去重守卫: domain ∈ Target ∪ DiscoveryCandidate → skip
   ↓
DiscoveryCandidate(status=pending)
   ↓
collector.ts（自动消费，72-min tick，20/天上限，优先级老化）
   /  monitoring.ts（监视器重跑，30-min tick）
   ↓
下一轮 Investigation → 下一代
```

- 消费端两个调度器（`src/instrumentation.ts` register() 顺序：monitoring → search/sync → discovery collector），全部 `setInterval` in-process，服务运行即存在。

---

## 5. 完整调用链（节点 → 文件/函数/表/触发条件）

| 节点 | 实现 | 状态字段 | 触发条件 |
|---|---|---|---|
| Seed | 手动表单 / API / 自动收集器 / 监视器 | Target 表 | 任意来源 |
| Investigation Created | `pipeline.ts: startInvestigation` | Investigation.status=running | 并发去重：RUNNING map |
| Investigation Queued | **无队列**——创建即执行（`void executePipeline` 分离式 promise，crash→failed） | status=running → completed/partial/failed | 立即 |
| Worker Picks | 无 Worker——同进程 | — | 立即 |
| Collectors Execute | `executePipeline` 7 步 | progress 字段 | 顺序 |
| Entities Extracted | `extractEntities`（管道 ③） | Entity / EntityRelationship | 每次调查 |
| Snapshots | 管道 ⑤ | Snapshot（今日 73 条创建，活跃） | 每次调查 |
| Events | 管道 ⑤ 对比 | Event（65 行） | 数据变化 |
| Insight Engine | 管道 ⑥ `runInsightEngine` | Insight（58 行） | 每次调查 |
| **Candidate Discovery** | **管道 ⑦.5 `extractCandidatesFromBundle`（`src/lib/discovery/candidates.ts`）** | — | 每次调查 |
| Candidate Sources | ① SSL SAN（非通配）② CNAME 目标 ③ 重定向主机 ④ 实体图共享邻居（ip/ssl_certificate/tracking_id/provider/technology） | source 字段（san 869 / cname 45 / shared_infra 7 / redirect 4 / shared_ip 1） | 每次调查 |
| Dedup | `addCandidate`: `if (target || existing) return;` | — | 入库前 |
| 优先级 | 60/50/45/40（san/cname/redirect/shared_infra）；`effectivePriority = priority + min(waitingDays×5, 40)` | DiscoveryCandidate.priority | 消费时 |
| New Investigation | `collector.ts tick()` → `startInvestigation(domain, {userId:null})` | status → analyzed（成功）/ skipped（失败） | 72-min tick，20/天上限 |
| Next Generation | 上述闭环 | — | — |

---

## 6. Scheduler / Cron 检查

| 检查项 | 实测 | 判定 |
|---|---|---|
| crontab | 无任何 crontab 条目（root + 服务用户） | ✅ 无外部 Cron（PASS） |
| Collector Scheduler | `[discovery] collector armed (20 candidates/day, tick every 72m, priority aging +40 cap)`（08-17 05:23:06 日志）→ 运行中，tick 准点（00:52 / 02:04 / 03:16 / 04:28 / 05:28） | ✅ PASS |
| Monitor Scheduler | `[monitor] scheduler armed (tick every 30m, daily re-analysis + change alerts)` 同刻 → 运行中，tick 准点（05:23 / 05:53 / 06:23） | ✅ PASS |
| 锁/重复保护 | 无分布式锁（单进程无需）；`RUNNING` map 防同一域名并发重复调查 | ✅ PASS |
| 历史抖动 | 08-15 02:10-04:07 曾 crash-loop ~15 次（`boot tick failed: PrismaClientKnownRequestError`）——开发期部署动荡，随后恢复，与本次停摆无关（MEDIUM） | 历史记录 |

**结论：Scheduler 正常（CONFIRMED）。**

---

## 7. Queue 检查

| 检查项 | 实测 | 判定 |
|---|---|---|
| 外部队列 | 无 Redis / BullMQ / job 表（`\d` 全部 53 表核对） | ✅ N/A |
| 候选池（事实队列） | `DiscoveryCandidate`：926 总 = **868 pending / 58 analyzed / 0 skipped** | ✅ 未枯竭 |
| pending 优先级分布 | 60:813 / 50:45 / 45:4 / 40:6 | ✅ |
| 消费速率 vs 存量 | ~5-20 次/天 vs 868 pending → **约 43 天余量** | ✅ 无饥饿 |
| dead-letter / retry | 无此概念（失败 → status=skipped，历史 0 次） | ✅ |

**结论：队列侧正常（CONFIRMED）——候选池远未耗尽，不是"队列空了导致停止"。**

---

## 8. Worker 检查

| 检查项 | 实测 | 判定 |
|---|---|---|
| Worker 运行 | 单进程（PID 677080），服务 active，NRestarts=0 | ✅ PASS |
| 最近消费 | 08-17 05:28:06 pm.sitevance.tw（collector 第 5/20 次） | ✅ PASS |
| 最近成功 | 同次调查 completed（2-25 秒内完成；wordpress.org 3 秒级） | ✅ PASS |
| 最近失败 | 全局 537 调查 = 333 completed / 202 partial / **2 failed**；2 failed 均为 `Interrupted by server restart`（08-14 google.com、08-16 virtualearth，重启中断，非故障） | ✅ PASS |
| 卡死/超时 | 最近 3 次 wordpress.org 调查均在秒级完成；无 stuck running | ✅ PASS |
| 权限/环境 | 无权限错误；TELEGRAM_BOT_TOKEN / AUTO_DISCOVER / DISCOVER_DAILY_CAP 均 EXISTS | ✅ PASS |

**结论：Worker 正常（CONFIRMED）。**

---

## 9. Investigation 检查

| 指标 | 值 |
|---|---|
| 总调查 | **537**（08-14: 84 / 08-15: 243 / 08-16: 186 / 08-17: 24 至 06:29） |
| 成功 | 333 completed + 202 partial（partial = 部分收集器失败仍完成分析） |
| 失败 | 2（均为服务重启中断，自动可重跑） |
| 今日新发现域名 | +8（Target.firstSeenAt 08-17: 8） |
| 今日新候选 | **0** |
| 今日新入队 | 5（collector 消费）/ 15（wordpress.org 监视器循环） |

**结论：调查管道完全健康（CONFIRMED）。**

---

## 10. Entity / Relationship 检查

| 指标 | 值 | 证据等级 |
|---|---|---|
| Entity | **3230**（用户旧值 511） | CONFIRMED |
| EntityRelationship | **5888**（用户旧值 891） | CONFIRMED |
| 提取器运行 | 每次调查执行（今日 24 次调查 → 提取持续发生） | CONFIRMED |
| 今日新增量 | UNKNOWN——Entity 无 createdAt 列（仅 lastSeenAt），无法直接按日度量新增 | UNKNOWN |

实体图持续增长，与繁殖停摆无因果（实体增长靠调查数驱动，调查数未停）。

---

## 11. Candidate Discovery 检查

| 指标 | 值 |
|---|---|
| 候选总数 | 926（创建：08-14: 53 / 08-15: 293 / **08-16: 580** / **08-17: 0**） |
| 最后创建 | **2026-08-16 09:48:37** `blog.csdn.net` → `2734556a.csdn.net.cname.yunduns.com`（cname 源，priority 50） |
| 最后大注入 | 08-16 08:16-08:23 手动品牌批量分析：reuters.com **+131**、paypal +93、airbnb +78、bloomberg +50、pinterest +47、ebay +36、amazon +20、yahoo +11…（journal 逐条为证） |
| 08-16 收集器驱动产出 | kimi.com +1（02:37）/ doubao.com +1（05:59）/ isimlens.com +1（07:17）/ ip.cn +1（09:25:06）/ blog.csdn.net +1（09:48:37） |
| 停摆后分析 | **7 次**（08-16 10:28 app-analytics-services-att、11:40 region1.app-analytics-services、08-17 00:52 app-analytics-services、02:04 service.urchin、03:16 fps.goog、04:28 googleoptimize、05:28 pm.sitevance）→ **全部 0 产出**（journal 无 `+N candidate(s)` 行） |

**结论：Discovery 提取代码一直在执行（CONFIRMED），但产出 = 0（CONFIRMED）。**

---

## 12. Dedup / Filter 检查（决胜证据）

**代码逻辑**（`src/lib/discovery/candidates.ts` `addCandidate`）：域 ∈ Target **或** DiscoveryCandidate → 直接 skip（`if (target || existing) return;`）。

**实际统计（candidate_count vs accepted_count，指令 §12 要求的格式）**：

| 最近分析 | candidate_count（SAN 条目） | 通配/IP 跳过 | accepted_count |
|---|---|---|---|
| wordpress.org（08-17 06:23 第 141 次重跑） | 2（`*.wordpress.org`, `wordpress.org`） | 1 | **0**（wordpress.org ∈ Target） |
| pm.sitevance.tw（08-17 05:28） | 3（`sitevance.tw`, `pm.sitevance.tw`, `*.pm.sitevance.tw`） | 1 | **0**（sitevance.tw 08-15 已入库；pm.sitevance.tw 当日已分析） |

- 停摆后 7 次分析 → 7×「candidate_count > 0, accepted_count = 0」模式（CONFIRMED）。
- 过滤层全貌：① 去重守卫（Target ∪ DiscoveryCandidate）② INFRA_SKIP_PATTERNS（15 个基础设施族，如 google-analytics 系）③ 通配符 SAN ④ IP SAN。**注意：停摆后消费的候选多为 Google/analytics 基础设施族（app-analytics-services*、service.urchin、fps.goog、googleoptimize）——它们自身就是基础设施死胡同**（分析成功但 SAN 为 Google 通配证书 → 跳过）。
- 文档 §7 情况分类：**情况 B**（候选已产生 → 全部被去重/过滤 → 0 accepted）。但需强调：**非过滤条件变化，而是已知集合饱和**——去重按设计工作（CONFIRMED）。`domain exists ≠ domain investigated` 的区分：本项目的"候选池"语义 = 已发现待调查清单，去重命中即"已发现"（合理设计，非误伤）。

**结论：Dedup 按设计工作；饱和是结构性结果（CONFIRMED）。**

---

## 13. Recursive Investigation 检查

- 递归曾经工作（历史证据链，CONFIRMED）：08-15 日志 `[discovery] analyzing candidate g.co → goo.gl → googlecommerce → urchin → youtu.be → youtubeeducation → youtubekids → baidu.com.hk → baidu.hk → baidu.net.au`…… 每轮分析产生新候选（`+13 from urchin.com` 02:29、`+1 from sitevance.tw` 02:51），形成多代链。
- 触发条件：**所有调查（手动/自动/API/监视器）走同一 pipeline，无条件分支**（`extractCandidatesFromBundle` 对每次调查执行；代码级 CONFIRMED）。不存在 `if source === "auto"` 之类限制。
- 当前断裂点：递归前沿（frontier）已闭合——已分析域名的 SAN/CNAME/重定向闭包 ⊆ 已知集合（§12 实测）。

---

## 14. Generation / Depth Limit 检查

- **无** maxDepth / maxGeneration / generation 字段 / parent_id / root_id（代码与 `\d` 核对，CONFIRMED）。
- 存在的限速（均未达到或未限制繁殖深度）：
  - `DISCOVER_DAILY_CAP=20`（消费限速；今日 5/20，未触顶）
  - monitor 频率预算 daily:10 / weekly:3 / monthly:2（仅作用于监视器重跑频率；**注意：死循环的 wordpress.org 已突破该预算——预算未拦截 lastRunAt 为空的情形**）
  - INFRA_SKIP_PATTERNS（15 族）+ 通配/IP SAN 跳过（候选质量过滤）
  - 优先级老化 `+ min(waitingDays×5, 40)`（防饥饿，工作正常：今日日志 `priority 60, effective 70`）
- **结论：非 I 类（Generation Limit）**——不存在繁殖深度上限（CONFIRMED）。

---

## 15. 最近 30 天运行时间线

系统上线仅 4 天（2026-08-14 起），30 天窗口即全部生命周期：

| 日期 | 调查数 | 成功 | 失败 | 候选创建 | 候选消费(analyzedAt) | 新 Target | 事件 | 洞察 |
|---|---|---|---|---|---|---|---|---|
| 08-14 | 84 | 82 | 2* | 53 | 13 | 57 | 0 | 0 |
| 08-15 | 243 | 243 | 0 | 293 | 20 | 133 | 35 | 12 |
| 08-16 | 186 | 186 | 0 | **580** | 20 | 91 | 30 | 39 |
| 08-17(至06:29) | 24 | 24 | 0 | **0** | 5 | 8 | 0 | 7 |

\* 2 失败均为"Interrupted by server restart"（08-14 google.com、08-16 virtualearth），重启中断可重跑，非故障。

**断点明确：候选创建在 08-16 09:48:37 后归零；其余一切照常。**

---

## 16. 最近一次成功繁殖证据

```
Parent Investigation:  blog.csdn.net（08-16 09:48 前）
Parent Domain:         blog.csdn.net
Discovered Candidate:  2734556a.csdn.net.cname.yunduns.com（source=cname, priority=50）
Candidate Created At:  2026-08-16 09:48:37
Candidate Accepted:    pending（入池）
New Investigation:     （消费端 868 排队中未轮到——按 20/天 + 优先级，需等待）
```

向前追溯（08-16 全链，journal 逐条 CONFIRMED）：kimi.com +1 (02:37) → doubao.com +1 (05:59) → isimlens.com +1 (07:17) → 品牌批量注入 08:16-08:23（reuters +131 / paypal +93 / airbnb +78 / bloomberg +50 / pinterest +47 / ebay +36 / amazon +20 / yahoo +11 / wikipedia +14 / snapchat +17 / netflix +12 / khanacademy +22 / bbc +17 / office +4 / gitlab +5 / spotify +1 / nytimes +5 / indeed +4 / quora +3 / facebook +1 / instagram +2 / whatsapp +1 / threads.net +1 / stackoverflow +3 / groupon +1）→ ip.cn +1 (09:25:06) → **blog.csdn.net +1 (09:48:37) ← 最后一次**

**历史上自动繁殖确实成功运行过（CONFIRMED，多代链条完整）。**

---

## 17. 最后一次正常运行时间

- **候选产出维度（繁殖）**：2026-08-16 09:48:37（blog.csdn.net → yunduns CNAME）。
- **系统整体维度**：持续至今——collector 今日 5/20、洞察引擎今日 7 条、监视器循环今日 15 次（均为"正常运行"），服务 active / NRestarts=0 / 线上 200。

---

## 18. 第一次异常时间

- **2026-08-16 10:28:37**：停摆后首次自动分析（app-analytics-services-att.com）→ 0 候选产出。此为可证的首个异常时刻。
- 观察窗口 = 08-16 09:48:37（最后成功）之后（CONFIRMED）。
- 旁证：08-16 09:16 部署（3A + 优先级老化上线）后提取仍工作到 09:48——**部署本身未破坏提取**（重要排除项）。

---

## 19. Git 回归分析

| Commit | 日期 | 涉及 | 与停摆关系 |
|---|---|---|---|
| 0f86811 Data growth engine | 08-14 10:38 | 候选钩子/优先级/自动收集器 | 停摆前部署，链条运转至 08-16 09:48 |
| 7c73050 监控通知 | 08-14 06:05 | Telegram/Webhook 通道 | 停摆前 |
| d8cefdb 监控调度修复 | 08-15 03:06 | monitor 修复 | 停摆前 |
| 6e09049 频率预算 | 08-15 07:08 | monitor 预算/手动触发 | 停摆前 |
| 467c736 优先级老化 | 08-16 07:06 | collector 防饥饿 | ~09:16 部署；**之后提取仍成功到 09:48** → 排除 |
| 605f8ab 3A 首页 | 08-16 07:42 | 首页改版 | 移除六宫格统计条（解释用户观感）；与提取无关 |
| c74dfd1 SEO P0 | 08-17 05:23 部署 | Gate/404/canonical/metadata/低价值过滤 | **不触碰** discovery/pipeline（`git log -- src/lib/discovery/ src/lib/pipeline.ts` 于 master 线仅 b52ec84 基线）→ 排除 |

**结论：无高度相关回归（CONFIRMED）**——所有繁殖相关代码均于停摆前部署，且部署后提取实测仍在工作（09:25/09:48 产出 +1），停摆后唯一部署（SEO P0）不涉及繁殖链路。

---

## 20. Production Log 分析

关键日志逐条（journalctl -u siteintel）：

```
# 繁殖正常期（08-15）
Aug 15 02:29:47 [discovery] +13 candidate(s) from urchin.com
Aug 15 02:51:50 [discovery] +1 candidate(s) from sitevance.tw
Aug 15 04:39:01 [monitor] re-analyzing 1 monitored target(s)
Aug 15 04:39:01 [monitor] wordpress.org → investigation cmstvz4lg004ds5h9qz9j51w2 (reused: false)
Aug 15 04:39:16 [monitor] wordpress.org: 2 event(s), 0 insight(s) — telegram:false webhook:false

# 最后一次繁殖（08-16）
Aug 16 09:48:37 [discovery] +1 candidate(s) from blog.csdn.net          ← 最后成功

# 停摆期（08-16 09:48 之后，每 72 分钟一次消费，全部 0 产出）
Aug 16 10:28:37 [discovery] analyzing candidate app-analytics-services-att.com (priority 60, effective 65)
...
Aug 17 05:28:06 [discovery] analyzing candidate pm.sitevance.tw (priority 60, effective 70)
Aug 17 05:28:06 [discovery] pm.sitevance.tw done — 5/20 today          ← 无 +N 行 = 0 候选

# 监视器死循环（每 30 分钟一轮，自 08-15 04:39 起 141 轮）
Aug 17 06:23:06 [monitor] re-analyzing 1 monitored target(s)
Aug 17 06:23:06 [monitor] wordpress.org → investigation cmswukooc00dts5fsuh4vsntm (reused: false)
Aug 17 06:23:11 [monitor] wordpress.org: 10 event(s), 5 insight(s) — telegram:false webhook:false
Aug 17 06:23:11 [monitor] wordpress.org alert delivery failed on all channels — retry next tick
```

- 停摆期间无 ERROR/WARN/timeout/rate-limit/数据库连接错误（grep 核对，CONFIRMED）。
- 历史抖动：08-15 02:11:42 `boot tick failed: PrismaClientKnownRequestError`（开发期部署动荡，恢复后无复发）。
- 可观测性缺口：管道 ⑦.5 仅在 `candidatesAdded > 0` 时打日志 → **0 产出的每次分析均静默**（P2 建议）。

**环境变量（§18，仅名称）**：TELEGRAM_BOT_TOKEN = EXISTS；AUTO_DISCOVER = EXISTS；DISCOVER_DAILY_CAP = EXISTS；CRON_*/QUEUE_*/WORKER_*/REDIS_*/SCHEDULER_* = 无此基础设施（N/A）。

---

## 21. Insight = 0 独立分析

指令 §13 A-G 逐项：

| 问题 | 答案 | 等级 |
|---|---|---|
| A. 是否真产生过 Event？ | **是**——Event 表 65 行（08-15: 35 / 08-16: 30），wordpress.org 10 条（infra_migration 3、provider_changed 3、dns 2、ip 2、metadata 1） | CONFIRMED |
| B. Event → Insight 是否存在？ | **是**——管道 ⑥ runInsightEngine + insight 规则（frequent_changes 等 6 类） | CONFIRMED |
| C. 规则是否命中？ | **是**——Insight 表 58 行（08-15: 12 / 08-16: 39 / 08-17: 7），今日 7 条 lastDetectedAt 刷新（wordpress.org 6 + bbs.asiadvb.net 1） | CONFIRMED |
| D. Insight Worker 是否运行？ | **是**——管道内同步执行；monitor 日志每轮 "5 insight(s)"（重检测既有模式） | CONFIRMED |
| E. 写入是否失败？ | 否——无相关错误 | CONFIRMED |
| F. 表是否有数据？ | **有，58 行** | CONFIRMED |
| G. 前端是否错误显示 0？ | **旧首页曾真实显示 0**（08-14 时表为空——引擎 08-15 才产出首行，与旧值"洞察 0"精确吻合）；3A 改版后统计条已移除；当前首页无此指标 | CONFIRMED |

**结论：Insight = 0 是 08-14 时代的真实快照，与繁殖停摆无因果，且当前洞察引擎健康（非 J 类故障——不存在故障）。** 今日事件 = 0 属正常（重跑集合无真实变化；08-15/16 的事件来自大量新域分析）。

---

## 22. 故障树

```
自动繁殖
│
├── Trigger                      → PASS  （今日 24 次调查，手动/自动/监视器三源全触发）
├── Scheduler                    → PASS  （collector 72m + monitor 30m 双 armed，tick 准点）
├── Queue Producer               → PASS  （候选入池工作至 08-16 09:48:37）
├── Queue                        → PASS  （868 pending，43 天余量）
├── Worker                       → PASS  （单进程消费正常，今日 5/20，秒级完成）
├── Investigation                → PASS  （537 次，333 completed / 202 partial / 2 重启中断）
├── Entity Extraction            → PASS  （每次调查执行；实体图 3230 持续增长）
├── Candidate Discovery          → ★ FAIL（提取执行但产出 0；7/7 次分析 0 新增）
├── Deduplication                → PASS  （按设计工作——全部命中即饱和，非误伤）
├── New Investigation Creation   → PASS  （候选持续被消费为调查）
└── Next Generation              → ★ FAIL（无新候选可进入下一代）
```

---

## 23. 根因判断

**分类：F（Discovery 无产出）+ G（Dedup/Filter 饱和）——复合，机制性饱和。**

- **本质**：递归种子闭包（frontier closure）。系统唯一的新域来源 = 已分析域名的 SAN/CNAME/重定向/实体图邻居；当已分析集合的闭包 ⊆ 已知集合（Target 289 ∪ DiscoveryCandidate 926）时，提取结果 100% 被去重命中 → 产出 = 0。**这不是故障，是设计的自然收敛边界**——机制全部按设计工作（CONFIRMED）。
- **增长只在"新种子注入"时发生**：最后一次注入 = 08-16 08:16-08:23 手动品牌批量分析（单日 +580 候选）；停摆后无新种子，自动链只能反复重遇到已知域。
- 排除项（逐一）：A 触发器（三源全触发）/ B 调度器（双 armed 准点）/ C 队列（868 存量）/ D Worker（消费正常）/ E Investigation（全部成功）/ H 递归创建（历史多代链完整）/ I 深度上限（无 maxDepth）/ J Insight（58 行健康）——全部 PASS，证据见 §6-§21。

**附带独立问题（K/Multiple）**：wordpress.org 监视器死循环（B 类调度逻辑缺陷）：
- 机制：3 用户全部无通知通道 → `notifyUser` 返回 `{telegram:false, webhook:false}` → `processMonitor` **仅在投递成功时推进 lastRunAt/lastNotifiedAt**，失败则"retry next tick" → lastRunAt 永为 NULL → 每次 tick 都视为到期 → 每 30 分钟全管道重跑。
- 实证：Monitor 表 wordpress.org 行 `lastRunAt = NULL`（141 次重跑后仍为空）；journal 141 轮 "alert delivery failed on all channels — retry next tick"；08-15: 59 次 / 08-16: 65 次 / 08-17 持续 ~48 次/天；RawCollection 中 wordpress.org 证书行 142 条（循环的可视化铁证）。
- 影响：全管道空转（6 收集器 × 48 次/天 ≈ 288 API 调用/天 + 快照/原始存储膨胀 + 每次重报同批 10 事件 5 洞察）；监视器"每日重分析+变更告警"核心价值归零（永不投递成功）。
- 根因位置：`src/lib/monitoring.ts` `processMonitor` 推进条件过严 + 用户通道缺失无兜底；`notify.ts` 对无通道用户的静默失败。

---

## 24. 证据等级与 12 个验收问题

### 证据等级总表

| 结论 | 等级 |
|---|---|
| 候选创建停摆于 2026-08-16 09:48:37（journal + DB 双证） | **CONFIRMED** |
| 停摆后 7 次分析全部 0 产出（journal 无 +N 行） | **CONFIRMED** |
| 去重饱和机制（wordpress.org 0/2、pm.sitevance.tw 0/3 accepted） | **CONFIRMED** |
| 无代码回归（部署时序 + git log + 停摆后提取仍执行） | **CONFIRMED** |
| 用户六宫格 = 08-14 晚间旧首页快照（84 精确相等、洞察 0 精确吻合） | **CONFIRMED** |
| 当前首页无六宫格（3A）+ CF 不缓存（DYNAMIC/no-store） | **CONFIRMED** |
| monitor 死循环机制（lastRunAt NULL + 141 轮日志 + 3 用户无通道） | **CONFIRMED** |
| 监控循环每日空转量级（~48 次/天全管道） | **HIGH** |
| 08-14/15 时代实体/关系数值（511/891 无 createdAt 列，间接推断） | **MEDIUM** |
| 08-16 09:16 部署窗口细节（08:47 构建报错 → 09:16 恢复） | **MEDIUM** |
| 其余各项 | CONFIRMED |

### 12 个验收问题

1. **自动繁殖现在到底有没有运行？** → 部分运行（CONFIRMED）：消费端（collector 5/20 今日、监视器 15 轮今日）全活；**生产端（新候选产出）自 08-16 09:48:37 起为 0**。
2. **最后一次运行是什么时候？** → 采集器今日 05:28:06 仍在消费（pm.sitevance.tw，5/20）；监视器 06:23:06 仍重跑（CONFIRMED）。
3. **最后一次成功是什么时候？** → 候选产出维度：2026-08-16 09:48:37（blog.csdn.net → yunduns CNAME）（CONFIRMED）。
4. **最近一次失败是什么时候？** → 无调查失败（2 failed 均为重启中断，最近一次 08-16 05:34 virtualearth，非故障）；"0 产出"首次出现 = 08-16 10:28:37（CONFIRMED）。
5. **Scheduler 是否正常？** → 正常（CONFIRMED）：collector 72min + monitor 30min 双 armed，tick 准点无缺失。
6. **Queue 是否正常？** → 正常（CONFIRMED）：无外部队列；候选池 868 pending 未枯竭（43 天余量）。
7. **Worker 是否正常？** → 正常（CONFIRMED）：单进程消费，今日调查全部秒级完成。
8. **Investigation 是否正常？** → 正常（CONFIRMED）：537 次，今日 24 次全部 completed/partial。
9. **Candidate Discovery 是否正常？** → 代码执行正常但**产出 0**（CONFIRMED）：停摆后 7 次分析 0 新增；机制 = 去重饱和。
10. **Candidate 是否成功变成下一代 Investigation？** → 是（CONFIRMED）：历史持续如此（08-14~08-16 09:48 多代链），停摆前最后 580 个候选仍在按 20/天被消费为调查（今日 5 个）。
11. **哪一个具体环节断了？** → 第 9 环 **Candidate Discovery 的产出端**（提取→去重间）：提取执行、去重按设计全部命中 → 0 产出。根因 = 种子闭包 + 去重饱和（F+G 复合）（CONFIRMED）。
12. **为什么现在 58 个域名 / 84 次调查之后没有继续增长？** → **前提不成立**：该组数字是 08-14 晚间旧首页快照（84 = 08-14 当日调查总数精确相等；洞察 0 = 当时 Insight 表为空；726 = 当日快照即时值）。实际数据早已大幅增长（289 Target / 537 调查 / 3230 实体 / 5888 关系 / 3147 快照 / 58 洞察），当前首页（3A 改版）实时渲染且 Cloudflare 不缓存。**真正停止增长的是"新候选域产出"**（08-16 09:48 后 = 0），而非全局数据增长——候选域增长依赖新种子注入，自动链已到设计性饱和边界（CONFIRMED）。

---

## 25. 修复建议

> 仅记录问题与建议，**本轮不执行任何修复**（遵守指令 §25）。

### P0（当前持续空转的资源浪费 — 与繁殖停摆独立，但应立即处理）

**wordpress.org 监视器死循环**（每 30 分钟全管道重跑，自 08-15 起 141 次，持续中）：

1. **立即止血（非代码，可选其一）**：
   - 在仪表盘为任一用户配置 `notifyTelegramChatId` 或 `notifyWebhookUrl`（服务器 TELEGRAM_BOT_TOKEN 已 EXISTS，缺的只是用户通道）——配置后首次成功投递即推进 lastRunAt，循环自愈；或
   - 将 wordpress.org 监视器 `active` 置为 false（数据行更新，非代码）——立即止住 ~48 次/天空转。
2. **代码修复（后续发布）**：`src/lib/monitoring.ts` `processMonitor` 应在"用户无任何通知通道"时跳过投递**但仍推进 lastRunAt/lastNotifiedAt**（无通道 = 无可送达者，不应视为失败重试）；或为连续失败加熔断/降频（如 3 次失败后改为每日一次并告警）。修复后同时解决"每次重报同批 10 事件 5 洞察"的基线问题（lastNotifiedAt ?? epoch 语义）。
3. 若 1 与 2 均未排期，至少在每次失败时记录一条可观测的 WARN（当前"retry next tick"无任何升级路径）。

### P1（繁殖恢复 — 种子补充机制）

1. **种子注入**（效果已实证）：手动/批量分析新品牌域（08-16 实证单次 +1~131 候选）；或建立"种子清单"（如 Tranco top-200 未入库项）作为候选池兜底源，当"连续 N 天候选产出 = 0"时由 collector 自动启用。
2. **候选池结构优化**：当前 868 pending 中 813 个 priority=60 且头部多为 Google/analytics 基础设施族（app-analytics-services*、service.urchin、fps.goog、googleoptimize）——分析即死胡同（SAN 为 Google 通配证书，INFRA_SKIP 命中）。建议对命中 INFRA_SKIP_PATTERNS 的域降分或提前标记，把每日 20 次的消费额度让给更可能产生新域的源（cname 50/redirect 45/shared_infra 40）。
3. **长期**：实体图共享邻居（shared_tracking/shared_cert/shared_ip 75-95 分）是最高产出源，但依赖新实体产生——种子问题解决后此源自然激活。

### P2（可观测性）

1. 管道 ⑦.5 目前仅 `>0` 时打日志——建议输出 `0 candidates (dedup=N, skip_infra=M)` 摘要，使"静默饱和"可观测（本次诊断正是靠 DB 反推才定位）。
2. 候选创建/消费速率、去重命中率仪表（后台管理页已有队列视图，补充趋势计数）。
3. 首页/文档注明数据口径与快照时点（六宫格已随 3A 移除，但用户观察基线仍易混淆——建议在关于页说明数据统计口径）。
4. Insight 每轮"5 insight(s)"为重检测语义——日志区分 新建/重检测。

---

## 附：遵守声明

本轮**零修改**：未修改业务代码 / 数据库结构 / 数据 / Cron / Scheduler / Worker / Queue / 环境变量 / Cloudflare 配置；未重新部署；未重启服务（NRestarts=0 保持）；未"顺手优化"；所有结论基于证据，UNKNOWN 处已明示。仅交付本报告 + 终端关键结论摘要。

**发现即记录，未修复。**
