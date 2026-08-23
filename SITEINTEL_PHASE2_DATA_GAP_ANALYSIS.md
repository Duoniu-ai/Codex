# SiteIntel Phase 2 数据缺口分析

> 审计配套文档 ｜ 日期：2026-08-23 ｜ 只读 ｜ 生产数据截至 2026-08-23
> 依据：生产只读 SQL（2026-08-23 采集）+ Phase 1 Step 5/6/7 报告 + 2.0 Data Architecture 冻结文档

---

## 1. 生产数据快照

| 表 | 行数 | 说明 |
|---|---|---|
| Target | 407 | 分析主体 |
| Investigation | 682 | 分析运行 |
| Fact | 3237 | schema0=3202（legacy）/ schema1=35（canonical） |
| Evidence | 5298 | 证据 |
| Event | 80 | 对外时间线（唯一） |
| Signal | 3481 | 内部迁移日志 |
| Job / JobRun | 4 / 10 | running=1, failed=2, success=7；running_per_job_max=1 |
| DiscoveryCandidate | 1195 | pending=1032 / analyzed=163；probe ok=874 / fail=158 |
| RawCollection | 4760 | 证据根 |
| Snapshot | 4227 | 时间点观测 |
| Entity | 4016 | subdomain=1390（34.6%） |
| EntityRelationship | 8633 | 关系资产 |
| 孤儿（Fact/Snapshot/Raw） | 0 / 0 / 0 | 无 orphan |

---

## 2. 十二层数据模型逐层分析

### 2.1 Raw（RawCollection）

- 现状：EXISTING，4760 行，不可变证据根；写入路径=collectAll providers；读取=Evidence 溯源/重放。
- 与 Master 一致：✅（保留策略 90 天滚动 PLANNED）。
- Gap：无保留策略；日增约千行级（规模推演 1M Target 时 TB 级）。
- 建议：Phase 2.5 实施 90 天滚动归档（备份后），CLEANUP 纳入 Job System。

### 2.2 Fact

- 现状：EXISTING，3237 行；schemaVersion 0/1；9 类 zod schema；报告唯一读源。
- 与 Master 一致：✅。
- Gap：legacy 188 条无效 Fact（140 metadata + 48 infrastructure）未迁移；sitemap/insights 读取未接入统一 gate；Evidence 指针截断语义待固化。
- 建议：读取 gate 统一（Step 9 候选）；legacy 迁移单独授权。

### 2.3 Entity

- 现状：EXISTING，4016 行；type+normalized 唯一；幂等 upsert。
- 与 Master 一致：✅。
- Gap：SAN 子域噪声（1390 子域，多数源自 SAN 证书列表）；无 Page 模型。
- 建议：SAN 降噪入池（Phase 2 D-2）；Page 模型（Phase 3）。

### 2.4 Relationship

- 现状：EXISTING，8633 行；9 类 type + confidence + evidence。
- 与 Master 一致：✅（status 字段 PLANNED）。
- Gap：evidence 为 JSON 内嵌未规范化；关系 status（active/superseded）未加。
- 建议：保留内嵌聚合，不急于规范化；status 随规模需求再加。

### 2.5 Observation / Snapshot

- 现状：EXISTING，4227 行；aspect 级 hash 去重；Snapshot=Observation 存储实现。
- 与 Master 一致：✅（无重复表）。
- Gap：无保留策略（aspect 最近 N 条 PLANNED）；无 zod 校验（快照载荷）。
- 建议：保留策略 + 快照校验（Step 9 候选）。

### 2.6 Event

- 现状：EXISTING，80 行；**唯一对外时间线**；12 类事件。
- 与 Master 一致：✅（Step 6 收敛后）。
- Gap：未来 Security/SEO/Ranking 变化事件类型未定义（随领域落地）；Event(targetId,type,detectedAt) 复合索引为规模化前置项。
- 建议：索引按需加（10 万 Target 前）。

### 2.7 Finding

- 现状：PLANNED（不存在）。
- 需求：状态化问题（open/fixed/verified/reopened/false_positive/accepted_risk）。
- 数据缺口：安全头值、依赖清单、版本覆盖、CVE 数据。
- 建议：Phase 4 建 SecurityFinding；Phase 5 复用同一 Finding 语义；不提前建表。

### 2.8 Insight

- 现状：EXISTING（PARTIAL），规则引擎 13 条 + contradiction。
- 与 Master 一致：⚠️ hypothesis 字段语义越界（CONFLICT 保留）。
- Gap：Insight(targetId,status) 索引为规模化前置项；evidence 内嵌未规范化。
- 建议：保留字段约束，仅搜索类规则使用；不拆表。

### 2.9 Recommendation

- 现状：PARTIAL（计算式不落库）。
- 数据缺口：无 Finding 状态机、无影响/成本/验证字段持久化。
- 建议：Phase 7 建表（priority/impact/effort/evidence/status）。

### 2.10 Action

- 现状：PLANNED（不存在）。
- 需求：被跟踪任务（status/dueAt/owner）+ 验证回环。
- 建议：Phase 7 与 Recommendation 同批。

### 2.11 Verification

- 现状：PLANNED（不存在）。
- 需求：before/after 对比（re-scan → verified）。
- 建议：Phase 7。

### 2.12 DiscoveryCandidate / Target 关联

- 现状：EXISTING（1195 行），pending=1032。
- 与 Master 一致：✅（analyzedValue/rootYield 回填 PLANNED）。
- Gap：**无价值回填**（不知道哪个候选带来好数据）；无候选→Target 关联列（sourceCandidateId）；无根域限额消费端；SAN 降噪入池未做；队列查询全表扫描（每 15s 拉全量 pending）。
- 建议：Phase 2 D-1/D-2/D-3（唯一推荐首选）。

---

## 3. 未来扩展能力验证

| 未来事件 | 当前模型能否承载 | 结论 |
|---|---|---|
| IP_CHANGED | Fact resolved_ips diff → Event | ✅ 可承载 |
| DNS_CHANGED | Fact dns_records diff → Event | ✅ 可承载 |
| ASN_CHANGED | Fact resolved_ips diff → Event | ✅ 可承载 |
| CERTIFICATE_CHANGED | Fact tls_certificate（fingerprint）diff → ssl_changed | ✅ 可承载（daysRemaining 变化=空 diff，已收敛） |
| TECHNOLOGY_CHANGED | Fact technologies diff → Event | ✅ 可承载 |
| INFRASTRUCTURE_CHANGED | Fact infrastructure diff → Event | ✅ 可承载 |
| PAGE_CHANGED | ❌ 无 Page 模型 | Phase 3 建 Page 后承载 |
| SECURITY_CHANGED | ⚠️ 被动 Fact 可 diff | Finding 状态机 Phase 4 补 |
| SEO_CHANGED | ⚠️ Technical SEO Fact 可 diff | Keyword/Ranking Phase 5-6 补 |
| RANKING_CHANGED | ❌ 无观测数据 | Phase 6 引擎观测后承载 |

结论：Event 模型（type/previous/current/evidence/detectedAt）**足够支持全部未来事件类型**，无需新增结构；只缺上游数据（Page/引擎观测/Finding）。

---

## 4. 数据增长与保留风险

| 数据 | 增长 | 保留策略 | 风险 |
|---|---|---|---|
| RawCollection | ~1300 日增（早期观察） | 无（90 天 PLANNED） | P1 |
| Snapshot | ~1000 日增（早期观察） | 无（aspect 最近 N 条 PLANNED） | P1 |
| DiscoveryCandidate | 数百/批 | analyzed 保留；pending 治理 | P1（积压 1032） |
| JobRun | 极少（10 行） | `pruneJobRuns(30d)` helper 未自动执行 | P2 |
| Event/Signal | 少量 | 长期保留 | ✅ |

---

## 5. 优先级排序（数据层面）

1. **P1**：Candidate 价值回填 + SAN 降噪 + 队列 SQL 化（Phase 2 D）。
2. **P1**：Raw/Snapshot 保留策略（Phase 2.5 基础设施）。
3. **P2**：Fact 读取 gate 统一 + legacy 迁移授权。
4. **P2**：Event/Insight 规模化索引（10 万 Target 门禁）。
5. **P2**：Evidence source_url 补全。
6. **P3**：Finding/Recommendation/Action/Verification（Phase 4-7）。

---

## GitHub Publication

- Repository: Duoniu-ai/Codex
- Branch: main
- Commit: （提交后回填）
- Tag: 无
- File: SITEINTEL_PHASE2_DATA_GAP_ANALYSIS.md
- Remote verification: PASS（提交推送后回填）
