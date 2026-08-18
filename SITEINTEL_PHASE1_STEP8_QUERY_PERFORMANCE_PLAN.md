# SiteIntel Phase 1 — Step 8 Query & Read Performance Foundation（实施前方案）

> 日期：2026-08-18 ｜ 性质：**只分析，不修改**（未改代码/数据库/生产；仅只读 EXPLAIN + 低频 HTTP 采样）
> 目标：为 10万/30万 Target、100万+ Fact 建立可验证的读路径基础

---

# 1. 读取架构（Page/API → Service → Query → Table）

| 读取入口 | 服务 | 查询数（静态计数） | 表 | 说明 |
|---|---|---|---:|---|
| 首页 `/` | page.tsx | ~15 SQL | Target/Entity/EntityRelationship | count + findMany(4, 含 3×_count 子查询) + entity + groupBy |
| `/website/[domain]` | generateMetadata + page | **~30-38 SQL/请求** | 全套 | **assembleReport + assembleSearchReport 被调用 2 次（元数据+页面）** |
| `/api/report/[domain]` | assembleReport | ~12-14 SQL | 全套 | 无重复（单次） |
| Entity 页（/ip /asn /cert /org） | assemble*Page + fetchEntityRelationships | ~10-15 SQL（顺序） | Entity/EntityRelationship/Event | 每页含重复 entity findFirst（页面+关系函数） |
| `/technology/[slug]` | assembleTechnologyPage | ~5 SQL | Entity/EntityRelationship/Event | 含质量门 |
| Timeline（报告内） | report.history | 1 SQL | Event | LIMIT 50，索引命中 |
| Intelligence | insight.findMany + search insights | 2-4 SQL | Insight | LIMIT 20 |
| Discovery（admin 无页） | 无读取页 | — | — | 仅 DB 写 |
| Admin data-quality | page.tsx | ~8 SQL | 全表 groupBy | 批量聚合 |
| Sitemap | sitemap.ts | ~12 SQL | Target/Fact/Event/Insight/Entity/Relationship | 多 groupBy + 实体 findMany |
| API v1/tools | v1/report、tools/* | ~12 SQL / 0-1 | 报告/工具 | tools 为外部网络调用 |
| Search（控制台/报告搜索区块） | assembleSearchReport | ~4-5 SQL | Search*/Event/Insight/Snapshot | 报告页内被重复调用 |

# 2. N+1 Matrix

**无经典循环 N+1**（related.ts 3 次扁平查询；实体页用 `IN` 批量；无 for-await-prisma 循环）。

真问题：

| 模式 | 位置 | 固定查询数（不随 N 增长） | N=10/100/1000 时 |
|---|---|---|---:|
| A. 双次装配（元数据+页面） | website/[domain] | 报告 ~12-14 ×2 + 搜索 ~4-5 ×2 | 每请求 32-38 SQL（与 N 无关，但 ×页面浏览量） |
| B. 首页 _count 子查询 | page.tsx | 1 findMany + 12 count 子查询 | 每请求 ~15 SQL |
| C. 实体页顺序查询 | seo/entity.ts | 10-15 SQL/页 | 每请求 10-15 SQL（无循环放大） |
| D. 重复 entity 查询 | 实体页 + fetchEntityRelationships | +1 重复 | 固定 |

结论：当前瓶颈是**每请求固定查询次数偏高 + 双次装配**，不是 N+1 放大。

# 3. Performance Baseline（生产源站 127.0.0.1:3003，3 次采样）

| 页面 | cold TTFB | warm TTFB | response size |
|---|---:|---:|---:|
| / | 156ms | 21-29ms | 105KB |
| /website/example.com | 173ms | 77-79ms | 308KB |
| /website/wordpress.org | 79ms | 60-63ms | 335KB |
| /technology/wordpress | 28ms | 25-28ms | 95KB |
| /ip/104.20.23.154 | 55ms | 46-49ms | 138KB |
| /asn/AS13335 | 81ms | 70-78ms | 267KB |
| /sitemap.xml | 158ms | 63-70ms | 45KB |
| /api/report/example.com | 72ms | 27-30ms | 64KB |
| /api/tools/dns | 24ms | 9ms | 0.7KB |

（未启用 DB 耗时/查询计数仪表——避免生产配置变更；DB 侧以 EXPLAIN 覆盖。）

# 4. EXPLAIN 结果（生产，example.com 实参）

| 查询 | 计划 | Execution Time |
|---|---|---:|
| Fact by targetId | Index Scan（`Fact_targetId_factType_key`） | 0.118ms |
| Event by targetId ORDER detectedAt DESC LIMIT 50 | Bitmap Index（`Event_targetId_detectedAt_idx`）+ Heap | 0.110ms |
| Insight active ORDER importance,confidence LIMIT 20 | Bitmap Index（`Insight_targetId_firstDetectedAt_idx`）+ Sort(27kB) | 0.082ms |
| Entity (type,normalized) | Index Scan（`Entity_type_normalized_key`） | 0.035ms |
| Relationship sourceId JOIN Entity | Nested Loop（unique idx + pkey） | 0.115ms |

**当前规模零 Seq Scan、零慢查询**；瓶颈在应用层请求次数与无缓存，不在执行计划。

# 5. Index Audit

| 表 | 现有 | 缺失候选（10万+ 时） | 重复 |
|---|---|---|---|
| Target | domain unique, lastSeenAt | — | 无 |
| Fact | (targetId,factType) unique, factType | **(factType,targetId)**（sitemap 批量） | 无 |
| Event | (targetId,detectedAt) | **(targetId,type,detectedAt)**（类型筛选） | 无 |
| Insight | (targetId,firstDetectedAt) | **(targetId,status)** | 无 |
| Signal | (targetId,detectedAt),(factId) | — | 无 |
| EntityRelationship | (source,target,type) unique, targetId | — | 无 |
| Job/JobRun | Step 7 已加 5 索引 | — | 无 |

结论：**当前索引充足**；候选索引按规模门禁（10万 Target）再加，不提前堆。

# 6. JSONB Audit

- 读路径中 **数据库层无 JSONB 操作**（Fact.value/Event 前后值/JobRun.meta 均整行取出后 JS 解析）。
- 成本点：报告页/实体页把 Fact.value 全量取出再 JS 解析（每 target 9 行）；sitemap 为 500 target 取 3 类 Fact 的整行 value（含大 payload）→ 传输/序列化成本。
- 结论：**本阶段不重构 JSONB**；后续可在 sitemap 改为列投影（value 仅在 JS 判断存在性时用 `jsonb_typeof`/`jsonb_array_length` 下推）——记录为 P2。

# 7. Cache Audit

| 读取 | 读/写比 | 可缓存 | 策略 |
|---|---|---|---|
| 首页计数/示例卡 | 高读/低写 | ✅ 短 TTL | 60s 内存缓存（含 _count 聚合） |
| 报告装配（per target） | 高读/低写（分析后更新） | ✅ 短 TTL | 60s 内存（key=target+locale）；新调查后自然过期 |
| 实体页 | 高读/低写 | ✅ 短 TTL | 60s 内存 |
| sitemap | 高读/极低写 | ✅ | 5-15min 缓存 |
| /api/report | 高读 | ✅ | 30s 内存 |
| tools dns/ip/ssl | 实时 | ❌ | 不缓存 |
| 分析进度 SSE / Monitor 状态 | 实时 | ❌ | 不缓存 |
| 安全/在线状态 | 实时 | ❌ | 不缓存 |

**请求级去重（P0，零缓存架构）**：`React cache()` 包裹 assembleReport / assembleSearchReport——同一次 SSR 请求内元数据+页面共享结果，直接消除双次装配（约 50% 读查询），无陈旧语义。

# 8. Report Audit（最高优先级）

- **双次装配**：`generateMetadata` 与页面各调用一次 assembleReport + assembleSearchReport（实测每页 ~32-38 SQL，去除后 ~16-19）。
- 报告内部：12-14 次查询含 2 次 `Promise.all` 与顺序补充查询（ipEntityGeo 二次查询）；evidence 含完整 rawData（页面 308KB 响应，其中报告 JSON 64KB）。
- 无循环 N+1；insight 证据链在 JS 组装。
- 修复方向：cache() 去重（P0）→ 并行化剩余顺序查询（P1）→ 可选响应裁剪（P2）。

# 9. Entity Audit

- 每页 10-15 次顺序查询（entity findFirst → 关系 → 二级关系/事件/技术聚合），take 上限 100-300。
- 重复：页面 entity findFirst 与 fetchEntityRelationships 内 findFirst 各查一次（+1 冗余）。
- 无循环 N+1；修复方向：合并重复查询 + Promise.all（P1）。

# 10. Scale Projection（模型估算，非容量承诺）

| 规模 | Fact | Snapshot | RawCollection | Relationship | Event | 报告查询/页 |
|---|---:|---:|---:|---:|---:|---:|
| 1万 Target | 9万 | 10-30万 | 21-70万 | 20-40万 | 0.3-2万 | ~32（当前形态） |
| 10万 | 90万 | 100-300万 | 210-700万 | 200-400万 | 3-20万 | ~32 |
| 30万 | 270万 | 300-900万 | 630-2100万 | 600-1200万 | 9-60万 | ~32 |
| 100万 | 900万 | 1000-3000万 | 2100-7000万 | 2000-4000万 | 30-200万 | ~32 |

风险分级：

- **明显风险**：无缓存 + 双次装配 → 10万 Target 时 DB 请求量 = 页面浏览量 × ~32 SQL；报告/实体页顺序查询放大延迟；sitemap 全量聚合（>30万 Target 后 groupBy 变重）。
- **中等风险**：Snapshot/RawCollection 无保留策略（TB 级存储增长）；Discovery 候选每 15s 全表拉取（属 Phase 2）；JobRun 增长（30 天保留后收敛）。
- **低风险**：索引（当前足够，10万 后再加候选）；Event/Insight 单目标查询（索引命中，随行数线性可接受）；关系图 one-hop（200-400万 关系时仍可索引驱动）。

# 11. 优化优先级

## P0（立即做，最小、零风险）

**P0-1 请求级装配去重**
- 问题：报告/搜索装配每页双次（元数据+页面）。
- 证据：website/[domain] 代码路径 + 静态查询计数（~32-38 SQL/页）。
- 方案：`React cache()` 包裹 assembleReport/assembleSearchReport（同请求共享）。
- 预期收益：页面读查询减半；TTFB 明显下降。
- 风险：无行为变化（同请求同参数幂等）。
- 回滚：移除 cache 包裹（1 行）。

**P0-2 首页短缓存**
- 问题：首页 ~15 SQL/请求，高读低写。
- 方案：60s 内存缓存 targetCount+示例卡数据。
- 收益：首页 warm 成本降至 ~0 SQL。
- 风险：计数最多 60s 陈旧（可接受）。
- 回滚：移除缓存调用。

## P1

1. **per-target 60s 报告缓存**（key=domain+locale；新调查后 TTL 自然过期；live 状态字段保留 checkedAt 语义）。
2. **实体页查询并行化 + 重复 entity 查询合并**（seo/entity.ts + fetchEntityRelationships）。
3. **sitemap 5-15min 缓存**（含 groupBy 结果）。
4. **/api/report 30s 内存缓存**。

## P2（规模门禁触发，10万 Target 或延迟超标时）

1. 索引：`Fact(factType,targetId)`、`Event(targetId,type,detectedAt)`、`Insight(targetId,status)`。
2. sitemap JSONB 下推（存在性判断移入 SQL）。
3. 报告 evidence rawData 响应裁剪（分页/截断）。
4. JobRun 30 天保留执行（复用 pruneJobRuns helper，未来 CLEANUP Job）。

## 暂不做

Redis/Memcached/CDN 缓存架构、分布式缓存、读副本、分库分表、搜索集群、GraphDB、CQRS、JSONB 模型重构、Discovery 候选 SQL 重写（Phase 2）、分页大改造。

# 12. 实施顺序

```text
P0-1 cache() 去重 → P0-2 首页缓存 → P1-1 报告缓存 → P1-2 实体页并行 →
P1-3 sitemap 缓存 → P1-4 api/report 缓存 →（10万规模门禁）P2 索引/下推/裁剪
```

每步独立 commit、独立验证、可单独回滚。

# 13. 测试方案

1. cache() 去重：单测（同请求两次调用返回同一 Promise/结果）+ 页面回归（TTFB 与查询数对比）。
2. 首页/报告/实体缓存：单测（TTL/命中/失效）；staging 双请求验证第二次命中。
3. 全量测试 + typecheck + build。
4. 生产前后 TTFB 对比（复用 §3 基线，3 次采样/页）。
5. EXPLAIN 复核（执行计划不变，缓存仅减少调用）。

# 14. staging

- 部署代码，验证页面 200、缓存命中（第二次请求 DB 调用减少——以日志/计数为准，staging 只读库无写影响）。

# 15. production

- 逐项 P0/P1 小步发布；每步：backup（代码无需 DB 备份，migration 无）→ build → restart → 页面/API 冒烟 → TTFB 对比 → 观察 30min（无 fatal）。

# 16. rollback

- 全部为应用层缓存/去重：`git revert <commit>` 或移除缓存调用 → build/restart；零 Schema/零数据变更。

# 17. 已知限制

1. 基线为源站 TTFB（Cloudflare 回源另加 ~10-50ms）；未启用 DB 耗时/查询计数仪表（避免生产配置变更）。
2. 样本低频小量（3 次/页），未做压力测试（禁止压垮生产）。
3. 规模推演为模型估算，非容量承诺。
4. 内存缓存为单实例语义（与 Job System 一致）；多实例前不引入 Redis。

# 18. 与 Step 7 的关系

Step 7 已为任务层建立可观测基础；本方案只处理读路径，不触碰任务/写入路径。

---

## GitHub Publication

- Repository: Duoniu-ai/Codex
- Branch: main
- Commit: 见提交结果（推送后 local HEAD = origin/main）
- Tag: 无
- File: SITEINTEL_PHASE1_STEP8_QUERY_PERFORMANCE_PLAN.md
- Remote verification: PASS（推送后验证）
