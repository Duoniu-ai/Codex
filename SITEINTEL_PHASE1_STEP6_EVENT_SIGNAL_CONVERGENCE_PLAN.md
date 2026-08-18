# SiteIntel Phase 1 — Step 6 Event/Signal 语义收敛（实施前方案）

> 日期：2026-08-18 ｜ 性质：**只分析，不修改**（未改代码/数据库/生产/历史数据）
> 依据：真实代码（facts.ts/correlation.ts/contradictions.ts/events.ts/search/events.ts）+ 生产只读统计
> 冻结语义：Event = 唯一对外时间线；Signal = 内部检测/触发机制；禁止 Event≈Signal；禁止 Signal 直接作为产品时间线

---

# 1. 当前 Signal 架构

```text
Signal（表：id/targetId/factId/type/previous/current/detectedAt）
  职责：Fact 状态迁移与内部检测的中间记录（内部信号）
  写入方（仅 2 处）：
    ① facts.ts syncOneFact：fact_created（新建 Fact）、fact_changed（值变化）
    ② contradictions.ts：fact_contradicted（矛盾开启）
  读取方（仅 2 处）：
    ① correlation.ts runSignalCorrelation（仅读 fact_changed → 生成复合 Event）
    ② admin/data-quality 页（type 分布展示，内部诊断）
```

生产统计：Signal 总计 3480 = fact_created 3237 / fact_changed 204 / fact_contradicted 39；24h 新增 502。

# 2. 当前 Event 架构

```text
Event（表：id/targetId/investigationId/type/category/severity/previous/current/evidence/detectedAt）
  职责：用户可理解、可追踪、可进入 Timeline 的事实变化（对外）
  写入方（仅 3 处）：
    ① facts.ts diffFact → storeEvents（类型化事实事件）
    ② correlation.ts → storeEvents（复合事件）
    ③ search/sync.ts → storeSearchEvents（搜索事件）
  读取方：report.ts（history/Timeline）、intelligence/engine（规则）、monitoring（告警）、
          search/report、event-diff/insight-text（渲染）、admin/data-quality（计数）
```

生产统计：Event 总计 79，全部为 fact-derived（evidence.factType 非空）；provider_changed 21 / infrastructure_migration 19 / dns_changed 14 / ip_changed 10 / technology_added 5 / metadata_changed 4 / ssl_changed 3 / website_overhaul 2 / rdap_changed 1；24h 新增 4（搜索事件尚未出现——无已连接属性）。

# 3. 全部写入路径（Signal / Event）

| # | 路径 | Signal | Event | 说明 |
|---|---|---|---|---|
| 1 | facts.ts：Fact 新建 | ✅ fact_created | ❌ | 首次观测，无变化 |
| 2 | facts.ts：Fact 值变化 | ✅ fact_changed | ✅ diffFact→typed event(s) | **双写核心** |
| 3 | facts.ts：值变化但 diff 为空 | ✅ fact_changed | ❌ | **孤儿 Signal（无 Event）** |
| 4 | contradictions.ts：矛盾开启 | ✅ fact_contradicted | ❌ | Contradiction 表承载展示 |
| 5 | correlation.ts：≥2 基础设施事实同变 | ❌ | ✅ infrastructure_migration | 由 fact_changed Signal 聚合 |
| 6 | correlation.ts：技术+元数据同变 | ❌ | ✅ website_overhaul | 同上 |
| 7 | search/sync.ts：搜索 diff | ❌ | ✅ 10 类搜索事件 | 无 Signal |

# 4. Signal → Event 映射矩阵

| Signal type | 产生 Event？ | Event type(s) | 关系 | 分类 |
|---|---|---|---|---|
| fact_created | 否 | — | — | **B（Signal-only，内部）** |
| fact_changed（resolved_ips） | 是 | ip_changed | 1:1 | A（重复事实）+ D 边界 |
| fact_changed（dns_records） | 是 | nameserver_changed + dns_changed | **1:2** | A + **D（一对多）** |
| fact_changed（tls_certificate） | 部分 | ssl_changed（仅 fingerprint 变） | 1:1 或 0 | A + B 边界（孤儿） |
| fact_changed（http_status） | 部分 | website_status_changed（仅在线↔离线） | 1:1 或 0 | A + B 边界（孤儿） |
| fact_changed（technologies） | 是 | technology_added / technology_removed ×N | **1:N** | A + D（一对多） |
| fact_changed（metadata） | 部分 | metadata_changed（仅 6 个 key） | 1:1 或 0 | A + B 边界（孤儿） |
| fact_changed（registrar） | 部分 | rdap_changed（不含 nameservers） | 1:1 或 0 | A + B 边界（孤儿） |
| fact_changed（infrastructure） | 是 | provider_changed | 1:1 | A |
| fact_contradicted | 否 | — | — | **B（Signal-only，内部）** |
| （无 Signal） | 是 | infrastructure_migration | 多 Signal→1 Event | **E（多对一）** |
| （无 Signal） | 是 | website_overhaul | 多 Signal→1 Event | E（多对一） |
| （无 Signal） | 是 | 10 类搜索事件 | — | **C（Event-only）** |

**A（重复事实）**：fact_changed 与其派生 typed event 描述同一事实变化（职责上 Signal=触发记录、Event=对外记录；语义不重复，但**记录粒度重复**——同一次变化写两行）。

**B（Signal-only）**：fact_created、fact_contradicted，以及 **fact_changed 孤儿（diff 为空）**——后者的存在是缺陷（见 §5）。

**C（Event-only）**：复合事件（infrastructure_migration / website_overhaul）与全部搜索事件——天然无 Signal。

**D（一对多）**：dns_records（1→2）、technologies（1→N）。

**E（多对一）**：correlation 聚合多 fact_changed → 1 复合事件。

**F（无法安全归并）**：无“必须保留双写”的硬冲突；唯一需要产品决策的是 fact_created 是否未来产生“首次观测”事件（当前 B 保留）。

# 5. 重复事实统计（生产实测）

- fact_changed Signal：204；fact-derived Event：79（含复合 21=19+2）。
- **孤儿 Signal：约 125 个（204 − 79）**——值变化但 diff 无可对外事件（如 metadata 的 faviconUrl/wordCount/h1 等非 diff key 变化、tls 的 daysRemaining/issuerOrg 变化、http 的状态码/头变化不跨在线线、registrar 的 nameservers 变化）。
- 结论：当前“Signal 与 Event 同步生成”逻辑存在**非关键字段变化产生 Signal 而无 Event**的中间态；收敛后此类变化应既不产生 Signal 也不产生 Event（内部 no-op），或按产品需要升级为低严重度 Event。

# 6. Signal-only 分类

1. `fact_created`（3237）——首次观测，内部（供 correlation/审计；未来可选“first seen”事件，产品决策）。
2. `fact_contradicted`（39）——矛盾检测内部信号（对外由 Contradiction 表 + 报告横幅承载）。
3. **孤儿 fact_changed（~125）**——**收敛后应消除**（见 §9 方案 A）。

# 7. Event-only 分类

1. 复合事件：infrastructure_migration（19）、website_overhaul（2）。
2. 搜索事件：traffic_spike/drop、ctr_change、keyword_gain/loss、ranking_gain/drop、index_growth/drop、visibility_change（生产当前 0）。

# 8. 推荐最终语义（冻结）

```text
Detection（值变化检测）
  ↓
Signal（内部触发记录：fact_created / fact_changed / fact_contradicted）
  ↓
dedupe / classification（diff + 幂等门）
  ↓
Event（对外时间线唯一事实源）
  ↓
Timeline / Insight / Monitoring
```

- Signal = 内部；绝不作为产品时间线（当前报告/API/前端已不读 Signal，admin 页仅内部诊断展示）。
- Event = 唯一对外；时间线、洞察规则、监控告警全部基于 Event。
- **同一事实变化只产生一个有效 Event**：值相等门（现有 factValuesEqual）阻止重复；同一次调查内去重由 diff 集合语义保证；不同时间真实变化分别产生事件（历史窗口语义）。

# 9. 最小修改方案（实施阶段，本次不执行）

**方案 A（推荐，最小）**——仅调整 `facts.ts syncOneFact` 内部顺序，消除孤儿：

```text
现状：值变化 → 写 fact_changed Signal → diffFact →（可能 0 个 Event）
改为：值变化 → diffFact → 若 diff 为空：不写 Signal、不写 Event（内部 no-op，记 debug）
      → 若 diff 非空：写 fact_changed Signal → storeEvents（1 个 Signal 触发 1..N Event）
```

- 影响面：仅 facts.ts 一个函数；不删表、不迁移、不改历史；Signal 表继续保留（fact_created/fact_contradicted 不变）。
- 语义收益：Signal 严格成为“产生 Event 的触发记录”；双写从“并行双写”变为“触发链”。

**可选增强（需另行授权）**：`Event.dedupeHash`（sha256(targetId+type+previous+current)）+ 唯一索引，作为“同一事实只产生一个有效 Event”的数据库级强制；当前应用层值相等门已覆盖，非必须。

# 10. Schema 是否需要变更

- 最小方案 A：**不需要**任何 Schema 变更（Signal 表保留，Event 表不变）。
- 可选 dedupeHash：需 1 个 additive 字段 + 唯一索引（另行授权，本次不做）。

# 11. 历史兼容方案

- 历史 Signal/Event **一律不改、不迁移、不重建**（原则冻结）。
- 历史孤儿 Signal（约 125 个）保持原样（内部记录，不影响时间线/报告）。
- 新行为只影响新写入；报告/Timeline 读取路径零变化。

# 12. 去重方案

1. **值相等门**（现有）：factValuesEqual 对 9 类事实做语义比较（集合/规范化 hash）——相同 before/after 重复调查不产生 Signal/Event（已实证：监控每日重跑无重复事件）。
2. **diff 集合语义**：dns/technology 多事件按集合差生成，天然无重复项。
3. **同一次 Observation**：一次调查内 fact 只 upsert 一次 → diff 只执行一次。
4. **不同时间真实变化**：previous/current 以最近一次事实为准，时间维自然区分（Event.detectedAt）。
5. （可选）dedupeHash 唯一索引兜底。

# 13. 测试方案（实施阶段）

1. 一个真实变化（如 IP 集合变）→ 恰好 1 个 Signal + 1 个 Event。
2. 同值重跑 → 0 Signal + 0 Event（幂等）。
3. 非关键字段变化（metadata faviconUrl / tls daysRemaining）→ 0 Signal + 0 Event（无孤儿）。
4. dns 变化 → 1 Signal + ≤2 Event；technology 变化 → 1 Signal + N Event。
5. Signal-only（fact_created/fact_contradicted）→ 0 Event，不进 Timeline。
6. 复合事件：≥2 基础设施事实同变 → infrastructure_migration 恰 1 个。
7. 不同时间两次真实变化 → 2 个 Event（历史保留）。
8. 回归：Report/Intelligence/Discovery/监控告警全量测试（现 329 例）+ staging + production。
9. 历史数据校验：迁移前后 Signal/Event 行数与内容不变（只读对比）。

# 14. 回滚方案

- 代码：`git revert <commit>`（仅 facts.ts 顺序调整）；无数据变更。
- 历史数据：零改动，无回滚需求。
- 可选 dedupeHash（若授权实施）：DROP COLUMN + DROP INDEX 可逆。

# 15. 未来扩展能力验证（Event 模型是否足够）

| 未来事件 | 现模型承载 | 说明 |
|---|---|---|
| IP_CHANGED | ✅ | 已有 ip_changed |
| DNS_CHANGED | ✅ | 已有 dns_changed / nameserver_changed |
| ASN_CHANGED | ✅ | 可经 resolved_ips 派生（type 自由扩展） |
| CERTIFICATE_CHANGED | ✅ | 已有 ssl_changed |
| TECHNOLOGY_CHANGED | ✅ | 已有 technology_added/removed |
| INFRASTRUCTURE_CHANGED | ✅ | 已有 provider_changed / infrastructure_migration |
| PAGE_CHANGED | ✅ | type 自由扩展（未来 Page 模型接入） |
| SECURITY_CHANGED | ✅ | type 自由扩展（Phase 4 Finding 状态变化事件） |
| SEO_CHANGED | ✅ | type 自由扩展（Phase 5） |
| RANKING_CHANGED | ✅ | 已有 ranking_gain/drop |

结论：Event 表结构（type/category/severity/previous/current/evidence/detectedAt）**无需 Schema 变更即可承载全部未来事件**；只需按类型命名规范（snake_case）扩展 type 值，并在接入新维度时补 category。

---

## GitHub Publication

- Repository: Duoniu-ai/Codex
- Branch: main
- Commit: 见提交结果（推送后 local HEAD = origin/main）
- Tag: 无
- File: SITEINTEL_PHASE1_STEP6_EVENT_SIGNAL_CONVERGENCE_PLAN.md
- Remote verification: PASS（推送后验证）
