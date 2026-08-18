# D2 开发验收报告 — 冻结 Schema 实现

> 验收日期:2026-08-17
> 执行依据:《docs/development/P0开发任务拆解-V1.0-冻结版.md》D2 包(任务 2.1-2.3)
> 唯一数据库设计依据:《网址发现平台-数据库Schema与数据同步契约-V1.0-冻结版.md》(§0-§16)
> 状态:**PASS — D2 正式验收通过**(2026-08-17 人工验收收口;下一步 D3,前置:服务器 PG 实例 nav_disc 确认)

---

## 1. 执行摘要

D2 包 3 项任务全部完成:24 表 Prisma Model 实现(2.1)、字段级机械对照清单(2.2,另存独立文档)、冷启动种子脚本(2.3)。验证链:`prisma validate` PASS → `prisma generate` PASS → `tsc --noEmit` PASS(含 seed.ts)→ `eslint` PASS → 单元测试 22/22 PASS → `next build` PASS。

**无 Schema Change Request、无 API Change Request、无新增未授权技术组件、无超出 D2 范围动作。种子未实际执行(禁建库),仅静态编译验证;实际执行在 D3 建库后。**

## 2. 任务完成状态与验证

| # | 任务 | 状态 | 实际验证 |
|---|---|---|---|
| 2.1 | schema.prisma 实现 24 表(主键 CUID / text+CHECK 枚举 / UNIQUE / §8 索引 / §10 外键策略) | ✅ | `npx prisma validate` PASS(24 模型,零警告);`npx prisma generate` PASS(Client 生成成功) |
| 2.2 | 字段级机械对照:逐表逐字段对照冻结版 §2,输出对照清单 | ✅ | 《docs/audit/D2-冻结Schema字段级对照清单.md》:24 表 / 221 字段 / 16 UNIQUE / 31 索引 / 25 FK / 30 CHECK 逐项 ✅ 一致 |
| 2.3 | seed 脚本:30 任务 × 每任务 5 网站、分类树、标签四枚举、别名、约束词典 settings、示例 curated_picks,幂等 | ✅ | `prisma/seed.ts` 编写完成;tsc 类型检查 PASS;**未执行**(禁建库,按任务书);执行命令 `npm run db:seed`(D3 建库后) |

## 3. 冻结基线逐项对照(摘录,全量见对照清单文档)

- **表数量**:24/24,无增减(`@@map` 与冻结表名一致);
- **字段**:221/221,无新增、无删减、无类型/NULL/默认值偏差(homepage_url 派生默认在应用层,冻结版自身即应用层语义);
- **UNIQUE**:16/16(含 `UNIQUE(normalized_domain)`、`UNIQUE(slug)`、`UNIQUE(alias_normalized)` 全局唯一、`UNIQUE(normalized_domain, job_type)`、`UNIQUE(website_id, task_id)`、`UNIQUE(task_id, tag_id)`、`UNIQUE(from, to, relation_type)`、`UNIQUE(website_id)` ×2、`UNIQUE(email)`、settings.key 主键);**domain 刻意不唯一**(§5);
- **索引**:31/31(含 `created_at(sort: Desc)` 时间线索引、复合索引),不建索引清单(website_status.summary_hash 等)未建;
- **外键**:25/25,onDelete 按 §10 总则:历史/审计表 Restrict(snapshots/status/metrics/changes/recommendations/evidence/relations/reports + website_tags.tag_id 等)、Cascade 仅 website_tags/website_tasks(源 draft 硬删)/task_aliases/task_related_tags(随 Task)与 recommendation_evidence(随推荐行)、SetNull 仅 user_events/search_logs 引用 + categories.parent_id;**sync_queue/sync_logs/submissions 不建 FK**(§14 注,normalized_domain 逻辑关联);
- **枚举**:30 项 CHECK 全部以 text + String 注释表达,枚举值集合与冻结版逐字一致(含 tag_type 仅四枚举、event_type 无 favorite、evidence_type 仅五因子、relation_type 含 P1 门控值 related);
- **外键实现级别**(冻结附录 A.7 未决):决策为 **DB 层外键**(Prisma 默认 foreignKeys 模式,不设 relationMode)。

## 4. D3 Migration 层需实现清单(Prisma 无法表达,非 SCR)

| # | 类别 | 明细 | 数量 |
|---|---|---|---|
| 1 | CHECK 约束 | §6 对照清单全部 30 项枚举 CHECK(text + CHECK,与枚举值一致) | 30 |

> 结论:Prisma 无法表达的约束**仅 DB CHECK 30 项**,全部登记 D3 Migration 原生 SQL 追加;模型语义与冻结版一致,不构成 SCR。D3 另含:建库 nav_disc、首迁 migrate、seed 实际执行(30 任务 × 150 网站)、数据库验收。

## 5. 应用层约束登记(冻结版自身定义在应用层,非 DB 约束)

reports.field 白名单 10 项 / settings.key 白名单 7 项 / homepage_url 派生 / relation_type P0 allowlist 门控 / sync_queue 状态机与 attempts 先 +1 再判定 / recommendations 实时搜索永不写入 / 收藏本地化 / 0-100 数值范围 / max_attempts ≥ 1 / 域名格式校验 / ADMIN_EMAILS 交集 —— 11 项,已在对照清单 §8 登记并标注落地包,不改变 DB 模型。

## 6. Seed 数据清单(prisma/seed.ts,幂等)

| 数据 | 数量 | 幂等方式 |
|---|---|---|
| 分类树 | 7(2 顶层 + 5 子分类) | upsert(slug) |
| 标签(四枚举) | 14(attribute 3 / feature 4 / constraint 2 / scenario 5) | upsert(slug) |
| 任务 | 30(status=active) | upsert(slug) |
| 别名 | 94(alias_normalized 全局唯一) | upsert(aliasNormalized) |
| 任务相关标签 | 82 | upsert(taskId_tagId) |
| 网站 | 150(每任务 5;domain 保留大写+www 形态演示 §5 折叠,seed 内折叠校验通过) | upsert(normalizedDomain) |
| 任务×网站关系 | 150(sortWeight 80/70/60/50/40) | upsert(websiteId_taskId) |
| settings | 7 个白名单 key 全量(含 constraint.dictionary 词典示例,与冻结 §6 示例一致) | upsert(key) |
| curated_picks | 3(published;无自然唯一键 → 先删 `[seed]` 标记行再建) | delete+create 幂等 |
| admin_users | 0(留空,D7.1 初始化) | — |
| submissions / website_tags | 0(未列入 D2-3;分别属用户数据流与 D7/D4 标注流) | — |

> title/summary_text 留空(SiteIntel 回填字段);seo_indexable/data_quality_score 保持派生默认(seed 不伪造派生值)。

## 7. 红线检查

- ✅ 未新增/删除表或字段、未改字段语义/类型/唯一约束/normalized_domain 规则/sync_queue 状态机/attempts 规则/relation_type 门控/recommendations 边界/P0-P1 边界/收藏本地化/SiteIntel 映射;
- ✅ 未执行 prisma migrate / 未建库 / 未连接生产库;
- ✅ 未编写业务 API / Worker / 前端页面;
- ✅ 未修改任何冻结文档(仅新增 docs/audit/D2-* 两份);
- ✅ 未引入任何新技术组件(零新增依赖);
- ✅ 未新增任务、未扩范围。

## 8. 发现与决策记录

| # | 事项 | 处置 |
|---|---|---|
| 1 | Websites 模型关系字段 `status` 与标量 `status` 重名(Prisma 校验报错) | 实现修正:关系字段改名 `websiteStatus`(DB 列不变,仍 website_id FK);冻结版无变化 |
| 2 | Tasks 模型误加 `reports` 关系(冻结版 Reports 无 task_id 字段) | 实现修正:删除该关系;对照清单确认 Reports 仅 website_id FK |
| 3 | 冻结附录 A.7 外键实现级别未决 | 决策:DB 层外键(Prisma 默认),schema 注释登记 |
| 4 | CHECK 约束 Prisma 无法表达 | 全部登记 D3 Migration 层原生 SQL 实现(30 项),模型 String + 注释保持语义,非 SCR |
| 5 | seed 的 normalized_domain 需遵守 §5 折叠规则 | seed 内置 foldDomain 校验(domain → normalized 折叠断言),不实现共享 normalize 模块(D4 范围) |
| 6 | curated_picks 无自然唯一键 | 幂等策略:seed 标记 reason 前缀 `[seed] `,重跑先删再建 |
| 7 | 禁止建库 → seed 不可实际执行 | 静态验证(tsc 类型检查 + 折叠逻辑断言);实际执行留 D3 |

## 9. 验收结论

**PASS — D2 正式验收通过(2026-08-17 人工验收收口),可进入 D3。**

- 24 表模型与冻结版机械一致(对照清单为证据,24/24 表、221/221 字段、16/16 UNIQUE、31/31 索引、25/25 FK、30/30 CHECK 语义);
- seed 就绪(30 任务 × 150 网站,幂等),D3 建库后 `npm run db:seed` 执行;
- 唯一遗留:DB CHECK 30 项 → D3 Migration 层实现(已登记,非 SCR);
- 前置提醒:D3 需服务器 PG 实例 nav_disc 确认(冻结版附录 A.5 未决项)。
