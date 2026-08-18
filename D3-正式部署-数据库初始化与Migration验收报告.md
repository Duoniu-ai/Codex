# D3 正式部署 — 数据库初始化与 Migration 验收报告

> 验收日期:2026-08-17
> 执行依据:《D3 正式部署前最终修复与执行授权》+《D3 字段级 @map 映射修复与失败 Migration 恢复执行授权》(用户下发)
> 唯一数据库设计依据:《网址发现平台-数据库Schema与数据同步契约-V1.0-冻结版.md》(§2 逐字段物理列名)
> 结论:**D3 PASS**

---

## 1. 原 P3018 / 42703 失败原因

首次 `migrate deploy`(20260817_init)失败:`P3018 / PostgreSQL 42703 — column "tag_type" does not exist`(HINT: 应为 `tags.tagType`)。

**根因(两层映射缺失,均为 D2 实现偏差)**:
1. **表级**:24 个 Prisma model(PascalCase)无 `@@map` → 物理表名将为大写形式,与冻结版 snake_case 表名不符;
2. **字段级**:221 个字段中 124 个 camelCase 字段无 `@map` → 物理列名 camelCase(如 `tagType`),与冻结版 §2 逐字段物理列名不符;CHECK 语句引用冻结版列名 `tag_type` 触发 42703。

失败迁移整体事务回滚,业务表 0 残留,仅 `_prisma_migrations` 留 1 条失败记录(finished=false)。

## 2. 表级 `@@map` 修复情况

- 24 个 model 逐一补齐 `@@map("snake_case 表名")`,以冻结版 §2 标题为唯一标准;
- 交叉核对三方:冻结版 §2、D2 对照清单 §1、schema.prisma → 24/24 一致;
- 核验:24 模型 ↔ 24 `@@map` 一一对应,无重复、无遗漏;`prisma validate` PASS。

## 3. 字段级 `@map` 修复情况

- 以冻结版 §2 逐字段物理列名为唯一标准(非自动 camelCase→snake_case 转换;特殊名如 `asn_org`、`ssl_expires_at`、`http_status`、`next_retry_at`、`clicked_website_id`、`duration_ms` 均按冻结版原文);
- **共 124 个字段级 `@map`**(按表:websites 12 / categories 3 / tags 1 / website_tags 3 / tasks 6 / task_aliases 3 / task_related_tags 3 / website_tasks 7 / website_relations 5 / website_snapshots 3 / website_status 11 / website_metrics 6 / website_changes 5 / recommendations 4 / recommendation_evidence 5 / curated_picks 6 / submissions 6 / reports 5 / user_events 5 / search_logs 6 / sync_queue 11 / sync_logs 5 / settings 1 / admin_users 2);
- 其余 97 字段本身即为单字母词或已符合 snake_case(如 id/domain/ip/data/payload/status/source 等),按"不需要映射不重复添加"原则不加;
- 无同表重复映射、无两字段映射同一物理列;124 = 148 总 `@map` − 24 表级。

## 4. 是否存在其他 Schema 语义修改

**无。** 决定性验证:HEAD 基线 vs 当前 schema 的 24 模型 × 全部字段「字段名+类型」序列**逐字全等**(脚本比对 PASS),即零字段增删、零类型/nullable/default/relation/index/unique/enum 变更;`git diff` 增量全部为 `@@map` / `@map` 行。

## 5. 失败 Migration 恢复方式

- 官方机制:`npx prisma migrate resolve --rolled-back 20260817_init` → "Migration 20260817_init marked as rolled back";
- 未手工删除/修改 `_prisma_migrations`、未 DROP、未重建数据库、未使用 `db push`。

## 6. 正式 Migration 执行结果

- 重新生成首迁 SQL:`prisma migrate diff --from-empty --to-schema-datamodel`(官方机制,非 db push);
- 迁移文件 `prisma/migrations/20260817_init/migration.sql`:24 表 snake_case、25 FK 内联 REFERENCES、31 CREATE INDEX、15 CREATE UNIQUE INDEX、**30 项枚举 CHECK 原生 SQL**;
- SQL 审查:表名/列名/CHECK/FK/INDEX/UNIQUE 全部引用真实 snake_case 物理列名;`tags.tag_type` 正确,无 `tags.tagType`;
- `npx prisma migrate deploy` → **"All migrations have been successfully applied."**。

## 7. 24 表验收结果

| 项 | 结果 |
|---|---|
| 24 张目标表全部存在 | ✅(另有 `_prisma_migrations` 1 张 = 25) |
| 表名与冻结版完全一致(snake_case) | ✅ 24/24 |
| 意外 PascalCase 表 | ✅ 0 |

## 8. 字段级物理列名验收结果

| 项 | 结果 |
|---|---|
| 全部列 snake_case(无 camelCase 列) | ✅ `column_name ~ '[a-z][A-Z]'` 查询 0 命中 |
| 总列数 | 229 = 221 业务字段 + `_prisma_migrations` 8 列 |
| 关键映射抽查 `normalized_domain` / `tag_type` / `display_name` / `alias_normalized` / `website_id_from` / `next_retry_at` | ✅ 均正确 |

## 9. 约束验收结果

| 约束 | 预期(D2 对照) | 实测 | 结果 |
|---|---|---|---|
| PK | 24 | 25(24 + _prisma_migrations) | ✅ |
| FK | 25 | 25 | ✅ |
| UNIQUE 语义 | 16(15 显式 + settings.key PK) | 15 UNIQUE INDEX + settings_pkey(key) | ✅(Prisma 以 UNIQUE INDEX 实现 @unique,故 pg_constraint.contype='u'=0 属正常) |
| CHECK | 30 | **30** | ✅ |
| INDEX | 31 | 31 | ✅ |
| CHECK/FK/UNIQUE/INDEX 引用列 | snake_case | 全部 snake_case | ✅ |

## 10. `_prisma_migrations` 最终状态

| migration_name | finished | rolled_back | 说明 |
|---|---|---|---|
| 20260817_init(失败记录) | false | true | 官方 resolve 标记,结构零残留 |
| 20260817_init(成功记录) | **true** | false | 本次正式应用 |

无残留 failed、无异常 rolled_back、迁移历史与实际结构一致。

## 11. 工作区状态

- `git status`:`M prisma/schema.prisma`(映射修复)、`?? prisma/migrations/`(首迁)、`?? docs/audit/D3-154服务器-PostgreSQL只读核验报告.md`(前序报告);
- `.env` 未被 git 跟踪(仅 `.env.example`),密码未入 Git / 聊天 / 本报告;
- 服务器 `/PRIVATE_PATH/.env`(600 权限,host=127.0.0.1)与本地开发隧道连接方式均正常;无临时脚本残留。

## 12. D3 最终结论

**D3 PASS**

- 建库建用户:✅ nav_disc(owner=PRIVATE_DATABASE_USER,UTF8)+ PRIVATE_DATABASE_USER(LOGIN,最小权限);密码 64 位 hex 仅落服务器 `.env`;
- 权限实查:✅ CONNECT/CREATE/TEMP、public USAGE/CREATE、表 owner 能力(全部表 owner=PRIVATE_DATABASE_USER);
- 连接实测:✅ PRIVATE_DATABASE_USER → nav_disc → Prisma Client → SELECT 1(经 SSH 隧道,公网 5432 保持防火墙关闭——安全惯例);
- 表级 + 字段级映射修复:✅(仅映射,零语义变更,决定性验证 PASS);
- 首迁 + 30 CHECK:✅ 全部成功;
- 24 表 / 221 字段 / 16 UNIQUE / 31 INDEX / 25 FK / 30 CHECK:✅ 与冻结版逐项一致;
- Migration 历史:✅ 成功记录 + 失败记录官方回滚标记。

**下一步**:依赖链进入 D4∥D5(同步实现 / Sync Queue 与 Worker)。未决项:navigation-sync Key(D4)、正式域名(D10.5)。
