# D3 Seed 执行与最终闭环验收报告

> 执行日期:2026-08-17
> 执行依据:《D3 最终闭环:Seed 实际执行与验收授权》(用户下发,授权 D3 任务书第 3 项)
> 前置状态:数据库结构与 Migration 已验收 PASS(见《D3-正式部署-数据库初始化与Migration验收报告.md》,D3 PASS)
> 结论:**D3 PASS — COMPLETE**

---

## 1. 执行时间

2026-08-17(本地时区 +0800;数据库 154 服务器 UTC)

## 2. 实际执行命令

```bash
# 连接方式:SSH 隧道(已验证方案,未改动)
# ssh -L PRIVATE_PORT:127.0.0.1:5432 -p REDACTED_PORT root@PRIVATE_SERVER
# 密码策略:从服务器 .env 拉取至 shell 变量,经进程环境变量注入,未落盘/未进聊天/未进报告

DATABASE_URL="postgresql://PRIVATE_DATABASE_USER:${PW}@127.0.0.1:PRIVATE_PORT/nav_disc?schema=public" npm run db:seed
```

- 执行 2 次(第 2 次为幂等性验证,见 §7);
- 未修改数据库 Host/Port/用户/密码/连接策略;未改 PostgreSQL 配置;公网 5432 保持防火墙关闭。

## 3. 目标数据库确认(执行前检查 6 项)

| # | 检查项 | 结果 |
|---|---|---|
| 1 | nav_disc 可连接 | ✅ 预检 `SELECT current_database(), current_user, 1` OK |
| 2 | DATABASE_URL 指向 nav_disc | ✅ `current_database() = nav_disc` |
| 3 | 非错误环境 | ✅ 服务器 PG active;nav_disc 存在(owner=PRIVATE_DATABASE_USER);siteintel/ip_query_db/backup_check 均未触碰 |
| 4 | 运行用户 | ✅ `current_user = PRIVATE_DATABASE_USER`(项目验收的正确运行用户) |
| 5 | 工作区无未授权 Schema 修改 | ✅ git status 仅 D3 交付物(schema.prisma @@map 修复为已验收变更;无其他改动) |
| 6 | seed 脚本与已验收版本一致 | ✅ `prisma/seed.ts` 与 Git 基线 de6e954 逐字一致,未修改 |

## 4. Seed 执行结果

| 阶段 | 结果 |
|---|---|
| 首次执行 | ❌ 失败 `P2022: column parentId does not exist`(根因与修复见 §4.1;失败时首条 upsert 即抛错,**零数据写入**,现场保留) |
| 修复后执行 | ✅ 成功:`[seed] 完成:任务 30(种子 30)/ 网站 150(本次 150)/ 别名 90/ 标签 14/ 分类 7/ settings 7/ curated_picks 3` |
| 幂等复跑(§7) | ✅ 成功,数量与首次完全一致 |

### 4.1 首次失败根因与修复(环境问题,非数据/脚本问题)

- **现象**:`categories.upsert()` 报 P2022,column `parentId` 不存在;
- **根因**:@prisma/client 生成产物**过期**——D3 阶段修复 124 个字段级 `@map`(15:24:46)前,client 最后一次 generate 在 15:13:38;旧 client 无 `@map` 映射,将逻辑字段名 `parentId` 直作物理列名,而数据库物理列为 `parent_id`(冻结版 §2.2)。SELECT 1 预检不涉及列引用,故未提前暴露;
- **证据**:client 产物中 `parent_id` 引用 0 次、`parentId` 3 次;schema.prisma 修改时间晚于 client 生成时间;
- **修复**:`npx prisma generate`(纯本地重新生成 client;零数据库接触、非 migration、非 db push、不修改 Schema);修复后 client `parent_id` 映射生效(2 次引用);
- **现场保留**:失败时未删除任何数据/表,未重建数据库,未使用任何破坏性手段重试。

## 5. 各类数据实际数量(只读验收,SQL 与数据模型一致)

| 表 | 实际数量 | 说明 |
|---|---|---|
| categories | **7** | 全部存在 |
| tags | **14** | tag_type 四枚举全覆盖 |
| tasks | **30** | 30/30 status=active |
| websites | **150** | 150/150 status=active |
| task_aliases | **90** | 90/90 status=active |
| settings | **7** | — |
| curated_picks | **3** | 3/3 status=published,3/3 带 `[seed] ` 标记 |
| website_tasks(关联) | 150 | 150/150 source=manual(每网站 1 任务) |
| task_related_tags(关联) | 77 | 任务×标签辅助意图映射 |

关联完整性:curated_picks 孤儿引用 0、website_tasks 孤儿 0、task_aliases 孤儿 0。

## 6. 与预期数量的对比

| 数据 | 授权书预期 | 实际 | 结果 |
|---|---|---|---|
| 任务 | 30 | 30 | ✅ |
| 网站 | 150 | 150 | ✅ |
| 分类树 | 7 | 7 | ✅ |
| 标签 | 14 | 14 | ✅ |
| 别名 | **94** | **90** | ⚠️ 差 4,已按授权停止并报告,经用户裁决以实际脚本为准 |
| settings | 7 | 7 | ✅ |
| curated_picks | 3 | 3 | ✅ |

**别名差异裁决记录**(授权书 §3"立即停止,报告差异,不自行修改"已执行):
- 执行前程序化核对发现实际脚本 90 条别名(ppt-maker 4 + browser-tool 2 + 其余 28 任务各 3);
- "94"来源:`docs/audit/D2-冻结Schema实现-开发验收报告.md:53` 登记值;实际 `prisma/seed.ts` 与 Git 基线 de6e954 一致、从未修改,即 D2 登记时脚本即为 90——**D2 验收报告登记统计偏差**;
- 冻结文档(P0 任务拆解 §2.3)仅要求"别名",未规定精确数量;
- **用户裁决:以实际脚本 90 为准继续执行**,报告登记勘误。

## 7. 幂等性验证方式与结果

- **方式**:seed 设计为安全幂等(全部 upsert:categories/tags/tasks/settings 按 slug/key;task_aliases 按 alias_normalized;task_related_tags 按 UNIQUE(task_id,tag_id);websites 按 normalized_domain;website_tasks 按 UNIQUE(website_id,task_id);curated_picks 无自然唯一键→先删 `[seed] ` 标记行再重建)→ 按授权 §7 确认设计幂等后**实际二次执行**;
- **结果**:二次执行成功,全部数量与首次完全一致(7/14/30/150/90/77/150/7/3);
- 已确认无项目自带 dry-run/统计模式,二次执行为授权允许路径。

## 8. 是否产生重复数据

**无。** 二次执行后:
- categories slug distinct=7、tags slug distinct=14、tasks slug distinct=30、websites normalized_domain distinct=150、task_aliases alias_normalized distinct=90——全部与总数相等(零重复);
- curated_picks 仍 3 条(先删后建,无残留);
- 关联表无孤儿、无冗余(website_tasks 150 恰为 1 对 1 关系,无重复 pair)。

## 9. 是否发生 Schema/Migration 修改

**无。**
- schema.prisma:未修改(仅保留 D3 已验收的 @@map/@map 修复);
- 无新 migration、未 `prisma db push`、无 DROP、无 TRUNCATE、未清库、未删表;
- 唯一动作:`prisma generate`(重新生成 client 以匹配已验收 Schema,纯本地、零数据库接触)。

## 10. 工作区状态

```
M PROJECT_STATUS.md        # D3 闭环状态更新(本报告登记)
M prisma/schema.prisma     # D3 已验收的 @@map/@map 修复(前序交付,非本轮变更)
?? docs/audit/D3-154服务器-PostgreSQL只读核验报告.md   # 前序交付
?? docs/audit/D3-正式部署-数据库初始化与Migration验收报告.md  # 前序交付
?? docs/audit/D3-Seed执行与最终闭环验收报告.md  # 本报告
?? prisma/migrations/      # D3 首迁(前序交付)
```

- 无临时脚本残留(tmp_seed_verify.mjs / tmp_seed_precheck.mjs 已删除);
- .env 未跟踪,密码未入 Git/聊天/报告。

## 11. 最终 D3 状态

**D3 PASS — COMPLETE**

- 建库建用户/权限/连接:✅(D3 前序验收)
- 24 表 / 221 字段 / 16 UNIQUE / 31 INDEX / 25 FK / 30 CHECK:✅(D3 前序验收)
- 首迁 20260817_init:✅ 已应用
- **Seed(D3 任务书第 3 项):✅ 已执行并验收(30 任务 × 150 网站 + 分类 7 + 标签 14 + 别名 90 + settings 7 + curated_picks 3)**
- 幂等性:✅ 二次执行零重复
- 数据库进入可用状态,可承载 D4∥D5 同步实现所需基础数据

**下一阶段**:D4 ∥ D5(同步实现 / Sync Queue 与 Worker),待 SiteIntel navigation-sync 专用 API Key 就绪后继续。
