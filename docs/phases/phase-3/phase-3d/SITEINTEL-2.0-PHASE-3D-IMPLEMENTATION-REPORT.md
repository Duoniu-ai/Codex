# SITEINTEL 2.0 PHASE 3D IMPLEMENTATION REPORT

> 日期：2026-08-23
> Provenance：`conversation-evidence-only` — 本报告无原始报告文件；内容依据 commit `de22a92` 真实代码树、测试套件与 2026-08-23 会话验收证据重建。
> Commit：`de22a92196966f03d0464b5e88ffd94ca7546bca`（feat(report): complete phase 3D intelligence layer）
> 规模：30 文件，+4,885 / −207，全部位于 `src/`；无 prisma / package / config / env 变化。

---

## 1. Phase 3D 目标

把 Website Report 从"数据展示"升级为"可追溯情报层"：稳定实体身份、发现溯源、事实证据链、数据可信度、历史概览、健康观察。Phase 3D 不新增数据库结构、不新增 Event Type、不改变采集/规则引擎。

---

## 2. 子阶段交付

### 3D-1 — Report UI 冻结与新区块 shell

- 冻结 `/website/[domain]` 组件边界：第一屏 → Identity + 概览 → 基础设施/技术 → 历史 → 洞察 → 关联 → 深入数据。
- 新增 UI shell：`data-trust-badge.tsx`、`website-identity.tsx`、`website-provenance.tsx`；重构 `report-page.tsx`（+276/−207 净减）。

### 3D-2 — 稳定实体身份 + 发现溯源

- `identity-assembly.ts`（+175）与 `entity-keys.ts`（+49）：Identity 为纯函数装配（零新查询），实体引用直接来自 `EntityRelationship.sourceId/targetId` 与 `Entity.normalized`，不做 value 反查/重编码。
- Provenance：仅当 `Target.sourceCandidateId` 非空才查询 `DiscoveryCandidate`（生产 100% NULL → 零新增查询）；缺失时 UI 走"无溯源"空状态。

### 3D-3 — 事实证据链 + 数据可信度

- `evidence-assembly.ts`（+142）：Fact → Evidence → RawCollection 证据链，批量 +2 查询（无 N+1）；`safeRawDataProjection` 保证页面/API 绝不返回完整 RawCollection JSON。
- `confidence.ts`（+72）：Data Trust 分档（sufficient / limited / conflict / stale / none），展示层纯函数，不持久化、不评分。

### 3D-4 — 历史概览（Event 主时间线 + Snapshot 辅助）

- `history-summary.ts`（+190）与 `history-section.tsx`（+333）：Event 时间线按 dimension 聚合，Snapshot 仅在 Event 缺 before/after 时补充（+1 查询，复用既有索引）；不建立第二条时间线。

### 3D-5 — 健康概览 / What to watch

- `health-summary.ts`（+230）与 `health-overview.tsx`（+97）：多维度独立状态（availability / TLS / redirects / technology / infrastructure / observations），观察性语言、无总分；纯函数，**零新增数据库查询**。

---

## 3. 范围控制

- 未修改：prisma schema / migrations、Event Type、package.json / lockfile、next.config、tsconfig、.env、Discovery/Collector/Pipeline/规则引擎。
- `git diff 8e5d9a8 de22a92 -- prisma/` = 空；Event Type 无新增（代码注释显式声明"无新 Event type"）。

---

## 4. 验证

- Phase 3D 测试：`report-3d2.test.ts`（8）+ `report-3d3.test.ts`（9）= **17/17 PASS**（2026-08-23 复核，vitest 734ms）。
- Query guard：`report-3d3` 断言 evidence/rawCollection 批量次数、`total ≤ 25`、无 N+1；`report-3d2` 断言 sourceCandidateId NULL → 0 次候选查询。
- Health：`report-3d3` 断言 health 基于既有报告数据、零新增查询。
- 关联测试：`identity-assembly`（254 行）、`evidence-assembly`（164 行）、`confidence`（111 行）、`history-summary`（330 行）、`health-summary`（330 行）、`entity-keys`（52 行）、`types`（112 行）、`i18n`（48 行）。

---

## 5. 产物清单（commit de22a92）

```text
src/components/report/data-trust-badge.tsx
src/components/report/evidence-chain.tsx
src/components/report/health-overview.tsx
src/components/report/history-section.tsx
src/components/report/insight-card.tsx
src/components/report/report-page.tsx
src/components/report/website-identity.tsx
src/components/report/website-provenance.tsx
src/lib/entity-keys.ts (+test)
src/lib/evidence-assembly.ts (+test)
src/lib/i18n.ts (+test)
src/lib/identity-assembly.ts (+test)
src/lib/intelligence/confidence.ts (+test)
src/lib/intelligence/event-diff.ts
src/lib/intelligence/health-summary.ts (+test)
src/lib/intelligence/history-summary.ts (+test)
src/lib/report-3d2.test.ts
src/lib/report-3d3.test.ts
src/lib/report-identity.ts (+test)
src/lib/report.ts
src/lib/types.ts (+test)
```

---

*归档说明：本报告为 Phase 3A–3D Documentation Consolidation 的一部分；原始对话证据未另存文件，以本文档为正式记录。*
