# SITEINTEL_D3_DATABASE_CHECK_COMPLETION_REPORT

> 项目:网址发现平台(网址导航站)— 独立产品,SiteIntel 仅为上游数据源
> 报告日期:2026-08-18(+0800)
> 执行基线 Git HEAD:de6e954524f1ec4f698cfbc764ec32ad78e16d04
> 数据库目标:PRIVATE_SERVER PostgreSQL 14.23 / 数据库 nav_disc(owner=PRIVATE_DATABASE_USER)/ SSH 隧道只读连接
> 本报告全部结论基于 2026-08-18 对生产库的**实时只读复核**(pg_constraint / information_schema / _prisma_migrations),不依赖历史报告文字

---

## 1. D3 Execution Summary

| 项 | 结果 |
|---|---|
| D3 状态 | COMPLETE / **PASS** |
| Git HEAD(基线) | de6e954524f1ec4f698cfbc764ec32ad78e16d04 |
| 验收时间 | 2026-08-18(+0800),实时只读复核 |
| 数据库目标 | nav_disc @ PRIVATE_SERVER:5432(SSH 隧道,只读) |
| Migration 状态 | `20260817_init` 两条记录:失败记录已官方 `resolve --rolled-back`;成功记录 `applied`;成功记录 checksum 与本地 migration.sql 逐字节一致(`6496160ea6135267d2169087b72ead3ee2e07730cacc571fd79a22bfe9021d21`) |

---

## 2. Database Change Summary

- **prisma/schema.prisma**:D3 映射修复 = 24 表级 `@@map` + 124 字段级 `@map`。逐行去空白比对:除 `@map`/`@@map` 与 24 处对齐空行外无任何差异;无字段增删、无类型/nullable/default/关系/索引/唯一变化(零语义变更)。
- **Migration**:`prisma/migrations/20260817_init/migration.sql` = 24 CREATE TABLE + 31 CREATE INDEX + 15 CREATE UNIQUE INDEX + 25 FK + 30 枚举 CHECK;无 DROP TABLE / TRUNCATE / DELETE / UPDATE 等破坏性语句;migration 目录仅此一个,顺序正常。
- **实测约束**:PK 24 / FK 25 / CHECK 30 / UNIQUE 16(15 唯一索引 + settings.key PK)/ INDEX 31(另有 39 唯一/PK 索引)。
- **物理结构**:24 表、221 字段,全部 snake_case,camelCase 列 0。

---

## 3. 30 CHECK Matrix

evidence 说明:全部 30 项于 2026-08-18 从生产库 `pg_constraint`(`contype='c'`)实时拉取,定义与《D2-冻结Schema字段级对照清单.md》§6 冻结枚举值逐字一致。

| # | CHECK 约束 | 冻结枚举值(目标) | Evidence(实查定义) | Result |
|---|---|---|---|---|
| 01 | chk_websites_status | draft/pending/active/archived | `status = ANY (ARRAY['draft','pending','active','archived'])` | PASS |
| 02 | chk_websites_source | manual/submission | `source = ANY (ARRAY['manual','submission'])` | PASS |
| 03 | chk_categories_status | active/hidden | `status = ANY (ARRAY['active','hidden'])` | PASS |
| 04 | chk_tags_tag_type | attribute/feature/constraint/scenario | `tag_type = ANY (ARRAY['attribute','feature','constraint','scenario'])` | PASS |
| 05 | chk_tags_status | active/hidden | `status = ANY (ARRAY['active','hidden'])` | PASS |
| 06 | chk_website_tags_source | manual/rule | `source = ANY (ARRAY['manual','rule'])` | PASS |
| 07 | chk_tasks_status | draft/active/pending_review/archived | `status = ANY (ARRAY['draft','active','pending_review','archived'])` | PASS |
| 08 | chk_task_aliases_source | manual/rule | `source = ANY (ARRAY['manual','rule'])` | PASS |
| 09 | chk_task_aliases_status | active/hidden | `status = ANY (ARRAY['active','hidden'])` | PASS |
| 10 | chk_task_related_tags_source | manual/rule | `source = ANY (ARRAY['manual','rule'])` | PASS |
| 11 | chk_website_tasks_source | manual/rule | `source = ANY (ARRAY['manual','rule'])` | PASS |
| 12 | chk_website_tasks_status | active/archived | `status = ANY (ARRAY['active','archived'])` | PASS |
| 13 | chk_website_relations_relation_type | similar/alternative/related | `relation_type = ANY (ARRAY['similar','alternative','related'])` | PASS |
| 14 | chk_website_relations_source | rule/manual | `source = ANY (ARRAY['rule','manual'])` | PASS |
| 15 | chk_website_relations_status | active/superseded | `status = ANY (ARRAY['active','superseded'])` | PASS |
| 16 | chk_website_snapshots_type | sync | `type = 'sync'` | PASS |
| 17 | chk_recommendations_context | task_page/search/detail/home | `context = ANY (ARRAY['task_page','search','detail','home'])` | PASS |
| 18 | chk_recommendations_confidence | high/medium/low | `confidence = ANY (ARRAY['high','medium','low'])` | PASS |
| 19 | chk_recommendations_source | rule/manual | `source = ANY (ARRAY['rule','manual'])` | PASS |
| 20 | chk_recommendations_status | active/superseded | `status = ANY (ARRAY['active','superseded'])` | PASS |
| 21 | chk_recommendation_evidence_evidence_type | task_match/constraint_match/quality/status_signal/editorial | `evidence_type = ANY (ARRAY['task_match','constraint_match','quality','status_signal','editorial'])` | PASS |
| 22 | chk_recommendation_evidence_generated_by | rule/manual | `generated_by = ANY (ARRAY['rule','manual'])` | PASS |
| 23 | chk_curated_picks_status | draft/published/archived | `status = ANY (ARRAY['draft','published','archived'])` | PASS |
| 24 | chk_submissions_status | pending/approved/rejected/duplicate | `status = ANY (ARRAY['pending','approved','rejected','duplicate'])` | PASS |
| 25 | chk_reports_status | pending/accepted/rejected/resolved | `status = ANY (ARRAY['pending','accepted','rejected','resolved'])` | PASS |
| 26 | chk_user_events_event_type | visit/reason_expand/submit/search_click | `event_type = ANY (ARRAY['visit','reason_expand','submit','search_click'])` | PASS |
| 27 | chk_sync_queue_job_type | sync | `job_type = 'sync'` | PASS |
| 28 | chk_sync_queue_status | pending/processing/completed/retrying/failed/skipped | `status = ANY (ARRAY['pending','processing','completed','retrying','failed','skipped'])` | PASS |
| 29 | chk_sync_logs_action | sync/analyze/report/diff | `action = ANY (ARRAY['sync','analyze','report','diff'])` | PASS |
| 30 | chk_sync_logs_status | success/failed/retry | `status = ANY (ARRAY['success','failed','retry'])` | PASS |

---

## 4. Data Integrity

- 不合规数据:**无**(30 项 CHECK 全部生效,种子数据各枚举值均合法)
- 修改业务数据:**无**(本轮为纯只读复核;种子数据由前序授权执行)
- 删除数据:**无**
- Destructive operation:**无**
- 种子实测:categories 7 / tags 14 / tasks 30 / websites 150 / task_aliases 90 / settings 7 / curated_picks 3 / website_tasks 150 / task_related_tags 77,与 D3 Seed 验收一致

---

## 5. Server Boundary

- `/PRIVATE_PATH` 当前仅包含 `.env`(600 权限,root 属主),**无代码目录**。
- Migration 由本地经 SSH 隧道应用,checksum 证明与本地 migration.sql 逐字节一致。
- 服务器无代码目录属于 **D10 部署范围,不是 D3 failure**;本轮未部署代码、未修改服务器目录、未修改 `.env`。

---

## 6. Git Safety

- `.env` 未被 Git 跟踪(`.gitignore` 覆盖),仅 `.env.example` 被跟踪。
- 无 secrets / 数据库 dump / 临时文件入库(64 位 hex 密码仅存服务器 `.env`)。
- `git diff --check`:**PASS**(无 whitespace error)。
- 仅 D3 交付物提交:`prisma/schema.prisma`、`prisma/migrations/`、`docs/audit/D3-*` 3 份、`PROJECT_STATUS.md`(D3 状态更新)、本报告。

---

## 7. Scope

```text
D3 = completed
D4 = not executed
D5 = not executed
D6 = not executed
D7 = not executed
D8 = not executed
D9 = not executed
D10 = not executed
```

---

## 8. Final Verdict

```text
CHECK 01-30 → 30/30 PASS
FAIL: 0
BLOCKED: 0
NOT_RUN: 0

D3 = PASS
```
