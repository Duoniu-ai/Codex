# SiteIntel 2.0 — Phase 2.4 Technology Historical Intelligence 验收报告

> 日期：2026-08-19 ｜ 范围：Phase 2.4（Technology Historical Intelligence only）
> 授权：按《SITEINTEL 2.0 — Phase 2.4 Technology Historical Intelligence 执行指令》执行
> 结论：✅ 完成（历史概览 + 时间线深化 + 证据标识 + 分页；无 Schema 变更；未部署生产；未 push）

---

# 1. Scope（范围）

在 Phase 2.3 Technology Profile 页面中深化 History 区块，将现有真实 Event / Relationship 数据转化为用户可理解、带明确 observation coverage 的 Technology History：

```text
History summary（First observed / Last observed / Observed events / Coverage）
Timeline（newest → oldest，Event + Website + Evidence 标识）
```

未建立全站 Historical Intelligence、未做 Backfill、未做互联网历史重建。

---

# 2. Existing Event Model（既有事件模型）

- Event 表：`type / severity / previous / current / evidence / detectedAt / targetId / investigationId`。
- `technology_added`：`current` = 指纹 slug，`previous` = null。
- `technology_removed`：`previous` = 指纹 slug，`current` = null。
- `evidence` 为 JSON：`{ detail, factType, snapshotType, previousSnapshotId, currentSnapshotHash }`（快照/事实指针）。
- Event **无 confidence 字段** —— 本阶段不伪造 confidence，时间线以 evidence 存在性 + severity 呈现。
- 生产实测（只读）：6 条技术事件（5 added / 1 removed），severity=low，evidence 全部非空。

---

# 3. Historical Data Sources（历史数据来源）

| 来源 | 用途 | 本阶段使用 |
|---|---|---|
| Event（canonical） | Timeline 唯一事件源 | ✅ |
| EntityRelationship firstSeenAt / lastSeenAt | First observed / Last observed summary | ✅（仅摘要，不构成事件） |
| Snapshot | 事件 evidence 指针（不直接展示） | ✅（通过 event.evidence 标识） |
| Evidence observedAt | 不直接用于 Timeline | 未使用（保留） |

未重新采集数据、未修改 crawler、未创建任何 fake historical records。

---

# 4. Timeline Implementation（时间线实现）

- 复用 Phase 2.2 统一 profile payload 的 `events`（后端一次查询，`orderBy [detectedAt desc, id asc]`，上限 200）。
- 前端在视图层对返回事件做分页（pageSize 20，`?historyPage=`），**不新增独立后端查询体系**。
- 每项展示：日期（YYYY-MM-DD）、事件（+ 检测到 / − 移除，文案由真实 Event 类型生成）、Website（可进入 /website/[domain]）、证据标识（Evidence attached，当 event.evidence 非空）。
- 顺序：newest → oldest（确定性排序）。

---

# 5. Event Semantics（事件语义）

- 只渲染 `technology_added` / `technology_removed` 两类真实事件。
- 文案：`Technology observed on {domain}` / `Technology removed from {domain}`（zh/en 双语）。
- `observedAt` = Event.detectedAt（真实字段）。
- Evidence：事件存在 evidence 时显示“含证据”标识，不输出内部指针/ID。
- Confidence：Event 无该字段 → 不显示、不伪造。

---

# 6. First / Last Observed（首次 / 最近观测）

- 语义严格限定为：**SiteIntel 当前记录中的最早 / 最近观察时间**（Relationship firstSeenAt / lastSeenAt 聚合）。
- UI 文案改为：`First observed` / `Last observed`（zh：首次观测 / 最近观测），不再使用“First seen”；
  明确不使用 `Created` / `Updated`（页面无此类文案，单测强制校验）。
- 空数据 → “—”。

---

# 7. Deduplication（去重）

- **Canonical historical event source = Event 行（主键唯一）**。
- 后端在组装时按 event.id 防御性去重：同一真实事件无论被多少个 payload 值命中，只出现一次。
- Relationship firstSeen/lastSeen 只用于 summary，不产生 Timeline 条目；Snapshot 仅作为 evidence 指针，
  三者不会让同一事件重复出现在 Timeline（单测覆盖：同一 id 两行 → 1 条）。
- 说明：同一网站对同一技术多次真实变化（added → removed → added）是不同真实事件，保留各自条目。

---

# 8. Coverage（覆盖边界）

- 页面明确展示：
  - First observed / Last observed / Observed events
  - `Historical coverage is limited to websites observed by SiteIntel.`
  - `This is SiteIntel's observation history, not the technology's full internet history.`
- 不展示：100% history / Complete history / All historical usage / global trend / market share。

---

# 9. UI（UI）

- 仅深化 Phase 2.3 已有 History 区块（section 04），未重设计整个 Profile：
  - 新增 History summary（3 个 StatCard：First observed / Last observed / Observed events）+ coverage 说明。
  - Timeline 每项增加证据标识；空态文案保持。
  - 时间线分页（pageCount > 1 时显示 Prev/Next + Page X of Y），与网站分页互不干扰、互相保留参数。

---

# 10. Mobile（移动端）

- 沿用响应式布局（flex-wrap / sm:grid-cols-3 / 无宽表）。
- Timeline 行式布局，证据标识与日期自动换行，无横向滚动。
- 视觉验证待生产部署后执行（项目无 E2E 基础设施，按既有约定不引入）。

---

# 11. SEO（SEO）

- 未新增独立 URL（无 /technology/[slug]/history）。
- History 属于既有 /technology/[slug] 内容，SEO gate / canonical / robots 逻辑不变（Phase 2.3 已校准 0 回归）。
- 时间线事件文案含真实网站名，内链指向 /website/[domain]（既有路由）。

---

# 12. Performance（性能）

```text
Cached:     0 database queries（ttlMemo，key=slug+page+pageSize）
Uncached:   同一 profile 请求（≤10 条索引查询），无新增事件查询体系
No N+1:     ✅ 事件为单次批量查询；evidence 为事件 JSON 字段，无额外查询
History pagination: 视图层内存分页（上限 200 条事件，当前生产全量 6 条）
```

---

# 13. Tests（测试）

```text
existing tests: 396
new tests:      5
total:          401 / 401 PASS（28 files）
```

新增覆盖：

- Data：事件 evidenceAvailable 真/假、severity、id、newest-first 排序、同一真实事件去重（同一 id 两行 → 1 条）、查询 orderBy 断言
- UI：History summary（first/last observed + event count）、时间线分页（25 事件 → 2 页；第 2 页 5 条）、越界页不触发空态、证据标识透出
- Semantic：`First observed` / `Last observed` 文案（非 Created/Updated）、coverage 文案存在

---

# 14. Typecheck（类型检查）

```text
tsc --noEmit: 0 错误（PASS）
```

---

# 15. Build（构建）

```text
next build（Turbopack，Next 16.3.1）: PASS
路由：ƒ /technology/[slug]（dynamic）
唯一 warning：jobs.ts crypto/Edge 历史警告（与本阶段无关）
```

---

# 16. Database Impact（数据库影响）

**NO。** 无 Schema / Prisma / migration / seed / 数据修改。唯一数据库动作：生产**只读**验证事件查询形状（临时脚本已清理）。

---

# 17. Known Limitations（已知限制）

1. 时间线上限 200 条事件（后端 take 上限），超出部分截断——当前生产 6 条，远低于上限。
2. Event 无 confidence 字段，时间线不展示置信度（如实）。
3. 历史 = SiteIntel 观测历史，不代表技术全局历史（UI 已明确）。
4. 版本变化（technology_version_changed）仍不存在（既有 diff 语义，Phase 2.4 不扩展）。
5. History summary 的 First/Last observed 来自关系时间戳，随重新观测滚动更新。

---

# 18. Future Backfill Boundary（未来 Backfill 边界）

本阶段**未实现**、且明确记录为未来独立 Historical Intelligence Phase 的内容：

- Wayback 自动重建
- 第三方历史数据 / AI 推断 / inferred historical events
- 根据 firstSeen 之前的数据补事件
- Global adoption curve / market share / worldwide growth / forecast

任何未来 Backfill 必须单独授权并严格区分 observed / reconstructed / inferred。

---

# 19. Acceptance Matrix（验收矩阵）

```text
Technology History implemented        PASS
Uses real Event data                  PASS
technology_added supported            PASS
technology_removed supported          PASS
First observed supported              PASS
Last observed supported               PASS
Correct chronological ordering        PASS（detectedAt desc + id asc）
No duplicate historical events        PASS（canonical Event + id 去重）
Website links work                    PASS（/website/[domain]）
Evidence displayed when available     PASS（Evidence attached 标识）
Historical coverage clearly stated    PASS
No fabricated history                 PASS
No Historical Backfill                PASS
No database change                    PASS
No crawler change                     PASS
No global UI redesign                 PASS
Mobile PASS                           PASS（响应式布局 + build；视觉验证待部署）
Tests PASS                            401/401
Typecheck PASS                        0 错误
Build PASS                            next build
Performance PASS                      缓存 0 查询 / 无 N+1 / 无重复事件查询
```

---

# 20. Commit（提交）

```text
Repository: Duoniu-ai/siteintel（本地 master，未 push）
Message:    feat(technology): add technology historical intelligence
Files:      src/lib/technology/profile.ts · profile.test.ts · profile-view.ts ·
            profile-view.test.ts · src/lib/i18n.ts · src/app/technology/[slug]/page.tsx
```

Commit: 2755bd3（feat(technology): add technology historical intelligence，6 files，+284/−42）

---

*Phase 2.4 完成，立即停止。未进入 Phase 2.5；未部署生产；未 push。*
