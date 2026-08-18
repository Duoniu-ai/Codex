# SiteIntel Phase 1 — Step 6 Event/Signal 语义收敛完成报告

> 日期：2026-08-18 ｜ 授权：用户正式授权（采用方案 A）｜ 结论：✅ **完成（PASS）**

---

## 1. 修改前 Signal/Event 流程

```text
值变化 → 写 fact_changed Signal → diffFact →（可能 0 个 Event → 孤儿 Signal）
```

已知缺陷：非关键字段变化（metadata faviconUrl/wordCount、tls daysRemaining、registrar nameservers、http 非在线状态变化）产生 fact_changed Signal 但无对应 Event（生产历史约 125 个孤儿）。

## 2. 修改后流程

```text
值变化 → factValuesEqual 门 → diffFact（先分类）
  ├─ diff 为空 → 不写 Signal、不写 Event（内部 no-op，孤儿消除）
  └─ diff 非空 → 写 fact_changed Signal → storeEvents（1 Signal → 1..N Event）
fact_created / fact_contradicted：保持 Signal-only（不变）
```

## 3. diff 逻辑（真实验证）

- metadata：仅 title/description/language/generator/hasRobots/hasSitemap 参与 diff → faviconUrl/wordCount/h1/内链/结构化数据变化 = 空 diff。
- tls_certificate：仅 fingerprint 变化产生 ssl_changed → daysRemaining/issuerOrg/有效期变化 = 空 diff。
- registrar：registrar/expiresAt/updatedAt/status 参与 diff → nameservers 变化 = 空 diff。
- http_status：仅在线↔离线产生 website_status_changed → 状态码/头变化 = 空 diff。
- dns_records：nameservers+records 均参与 → 最多 2 个事件（一对多）。
- 决策（记录在案）：上述非关键字段 = **C 完全忽略**（Fact 值照常更新，当前状态正确；无产品可见事件；孤儿 Signal 消除）。未凭猜测改变任何事件语义。

## 4. 孤儿 Signal 原因

修改前“先写 Signal 再 diff”导致值变化但 diff 为空时产生无 Event 的 fact_changed（历史约 125 个；本步只消除未来写入，历史保留）。

## 5. 实际修改文件

```text
src/lib/facts.ts                     （syncOneFact：diff 前置，空 diff 不写 Signal/Event）
src/lib/facts-event-signal.test.ts   （新增 10 例）
```

## 6. Schema 是否变化

**否**。无 Migration、无字段、无新表；Signal/Event 表保留。

## 7. 历史数据处理

历史 Signal/Event **零修改、零迁移、零清洗**；新行为只影响未来写入。

## 8. 测试

- 新增 10 例：diffFact 语义（真实变化/非关键字段/tls daysRemaining/registrar nameservers/dns 一对多）；syncFacts 收敛行为（同值无 Signal 无 Event、真实变化 1 Signal+1 Event、非关键变化无 Signal 无 Event、一次 sync 不重复、fact_created Signal-only）。
- 全量：**339/339 通过（23 文件）**；typecheck 0；build 通过（生产 BUILD_ID `QXfPGv3BKNN2_t93WqIHv`）。

## 9. staging

代码部署成功（build/active/home 200）；只读库无法新建 Fact，收敛行为由单测+生产实证覆盖。

## 10. production

| 验证 | 结果 |
|---|---|
| 同值重分析（example.com） | 0 新 Signal / 0 新 Event（幂等） |
| 同值重分析（wordpress.org） | 0 新 Signal / 0 新 Event |
| 真实变化（ntp.org/icloud.com 批次） | **1 fact_changed ↔ 1 ssl_changed（1:1）** |
| 空变化 Event | 0（无 orphan、无空事件） |
| Report/Timeline | /website/example.com 200 |
| 调度器/fatal | monitor/search/discovery 正常；0 fatal；NRestarts=0 |
| HEAD | 对齐 `559134c`，tracked 树零差异 |

## 11. Signal 行为对比

| 场景 | Step 5 前 | Step 6 后 |
|---|---|---|
| 同值重跑 | 无 Signal | 无 Signal（不变） |
| 真实变化 | 1 fact_changed + Event | 1 fact_changed + Event（不变） |
| 非关键字段变化 | 1 孤儿 fact_changed（无 Event） | **0 Signal / 0 Event** |
| fact_created / fact_contradicted | Signal-only | Signal-only（不变） |

## 12. Event 行为对比

| 场景 | Step 5 前 | Step 6 后 |
|---|---|---|
| 真实变化 | 对应 typed Event | 对应 typed Event（不变） |
| 非关键字段变化 | 无 Event（有孤儿 Signal） | 无 Event（且无孤儿 Signal） |
| 复合/搜索事件 | correlation/search 独立写入 | 不受影响（未触碰） |

## 13. Git commit

`559134c1ecad12b37af47ac976bf1086acec9291`（fix(data): converge signal and event emission (Plan A)）；siteintel main = 生产 HEAD = 559134c。

## 14. tag

`production-phase1-step6-event-signal-2026-08-18` → 559134c（annotated，已推送）。

## 15. GitHub 文件

- 本报告与 Step 6 方案均推送至 Duoniu-ai/Codex main（见 GitHub Publication）。

## 16. 回滚

- `git revert 559134c` 或恢复 tag `production-phase1-step5-fact-validation-2026-08-18`=61e75bf → build/restart。
- 零 Schema/零历史数据变更，回滚无数据风险。

## 17. 已知限制

1. 历史约 125 个孤儿 Signal 保留（不迁移；内部记录，不影响时间线/报告）。
2. 非关键字段变化（faviconUrl/wordCount/tls daysRemaining/registrar nameservers 等）现在既无 Signal 也无 Event——Fact 当前状态仍正确；如未来产品需要“元数据微变”时间线，需单独设计低严重度事件类型（另行授权）。
3. 数据库级去重（Event.dedupeHash 唯一索引）未实施（本步禁止；应用层值相等门已覆盖）。

---

## GitHub Publication

- Repository: Duoniu-ai/Codex
- Branch: main
- Commit: 见提交结果（推送后 local HEAD = origin/main）
- Tag: production-phase1-step6-event-signal-2026-08-18（siteintel 仓库）
- File: SITEINTEL_PHASE1_STEP6_EVENT_SIGNAL_CONVERGENCE_COMPLETION_REPORT.md
- Remote verification: PASS（推送后验证）

---

*完成，停止。未开始 Step 7。*
