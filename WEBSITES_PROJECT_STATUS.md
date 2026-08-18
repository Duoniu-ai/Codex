# PROJECT STATUS — 网址发现平台(网址导航站)

> 独立产品项目,非 SiteIntel 子项目;SiteIntel 仅为上游数据来源(见 PROJECT_BOUNDARY.md)。
> 更新日期:2026-08-17

---

## 已完成

- ✅ 产品规划 **V2.1.2 冻结**(外部三轮审计收敛,2026-08-17)
- ✅ SiteIntel 上游数据能力审计(PASS WITH 2 GAPS,架构阶段 1)
- ✅ 技术架构 **V1.0-rev2**(外部审计 5 修复 + 与 Schema 冻结版基线同步,**自身仍为候选版**)
- ✅ 数据库 Schema 与数据同步契约 **V1.0 正式冻结**(2026-08-17,24 表,18 项检查 PASS,验收记录 docs/audit/Schema冻结验收记录-V1.0.md)
- ✅ 基线文档整理入库(docs/product、docs/architecture、docs/audit)
- ✅ **API 契约设计 V1.0 正式冻结**(2026-08-17:外部冻结审计 PASS WITH 5 FIXES → P1-1~P1-5 + P2-9/P2-10 修复 → 重检 20/20 + 专项 S1-S7 PASS → 冻结;冻结版 docs/api/API契约设计-V1.0-冻结版.md,验收记录 docs/audit/API契约冻结验收记录-V1.0.md)
- ✅ **前后台信息架构设计 V1.0 候选版**(2026-08-17:候选版 docs/architecture/前后台信息架构设计-V1.0-候选版.md + 检查报告 docs/audit/前后台信息架构-内部一致性检查报告.md;检查 14/15 PASS,登记 INFORMATION_ARCHITECTURE_GAP-01(首页聚合数据无公开端点),无 SCR、无 API Change Request 落地,零触碰冻结基线)
- ✅ **GAP-01 已裁定关闭**(2026-08-17:专项审计 docs/audit/GAP-01-首页聚合数据专项审计报告.md,裁定方案 B——首页服务端组件只读直查冻结 Schema(curated_picks published + tasks active&seo_indexable,共 2 个只读查询),不新增 Public API,零触碰冻结基线;信息架构内部一致性检查收口为 **15/15 PASS**)
- ✅ **前后台信息架构设计 V1.0 正式冻结**(2026-08-17:最终冻结审计 PASS WITH FIXES(0 P0/1 P1/4 P2)→ P1-1 修复(/tasks、/topic 直查权限表述)→ 专项复检通过 → 冻结;冻结版 docs/architecture/前后台信息架构设计-V1.0-冻结版.md,验收记录 docs/audit/前后台信息架构冻结验收记录-V1.0.md)
- ✅ **P0 开发任务拆解 V1.0 正式冻结**(2026-08-17:候选版 D1-D10 共 73 项任务 → 内部一致性检查 **10/10 PASS**(A-J)→ 冻结收口(冻结版 docs/development/P0开发任务拆解-V1.0-冻结版.md,验收记录 docs/audit/P0开发任务拆解冻结验收记录-V1.0.md);冻结范围 21 项 + 文末冻结后变更规则;零新增/零删除/零调整任务与依赖;无 SCR、无 ACR)
- ✅ **D1 项目与基础设施初始化**(2026-08-17:8 项任务全部完成;22/22 测试 + tsc + eslint + next build + dev 冒烟全 PASS;验收报告 docs/audit/D1-项目与基础设施初始化-开发验收报告.md)
- ✅ **D2 冻结 Schema 实现**(2026-08-17:24 表 Prisma 模型逐字落地,字段级机械对照 24/24 表、221/221 字段、16/16 Unique、31/31 Index、25/25 FK、30 项 CHECK 登记 D3 Migration 实现;对照清单 docs/audit/D2-冻结Schema字段级对照清单.md;验收报告 docs/audit/D2-冻结Schema实现-开发验收报告.md,**PASS — 正式验收通过**)
- ✅ **D3 Migration 与数据库验收**(2026-08-17:**D3 PASS — COMPLETE**。154 服务器建库 nav_disc(owner=nav_disc_user)+ 建用户(LOGIN 最小权限,64 位 hex 密码仅落服务器 /www/wwwroot/websites/.env);修复 D2 两层映射偏差(24 表级 @@map + 124 字段级 @map,零语义变更,决定性验证 PASS);首迁 20260817_init 应用成功(24 表 / 221 字段 / 15 UNIQUE+settings PK / 31 INDEX / 25 FK / 30 CHECK,全部 snake_case 物理结构);失败迁移官方 resolve --rolled-back;**seed 已执行并验收**(30 任务 × 150 网站 + 分类 7 + 标签 14 + 别名 90 + settings 7 + curated_picks 3;幂等二次执行零重复;别名登记勘误 94→90 经用户裁决以实际脚本为准;首次失败 P2022=client 过期,prisma generate 修复,零数据写入);报告 docs/audit/D3-正式部署-数据库初始化与Migration验收报告.md + docs/audit/D3-Seed执行与最终闭环验收报告.md)
- ✅ **D4 SiteIntel 数据同步实现**(2026-08-18:**D4 PASS — COMPLETE**。`src/lib/sync` 六模块(client/normalize/mapping/summary-hash/persist/throttle)+ index 出口;D4 测试 68 项,全量 90/90 PASS,tsc/eslint/git diff --check PASS;30/30 D4 验收矩阵 PASS;报告 SITEINTEL_D4_SYNC_COMPLETION_REPORT.md;生产 e2e(真实 SiteIntel + nav_disc 落库)按冻结任务书 D4.2 留待人工确认)

## 当前阶段

**D4 COMPLETE — 进入 D5:Sync Queue 与 Worker**(产品/Schema/API/信息架构/开发任务拆解五基线全部冻结;D1-D4 已完成,数据库含种子数据可用)

- 下一执行包:**D5**(Sync Queue 与 Worker:状态机/CAS 领取/锁与超时恢复/退避/调度;允许目录 src/lib/worker、src/lib/queue、src/instrumentation.ts)
- D5 后按依赖链推进:D6∥D7 API → D8∥D9 前端 → D10 测试验收部署(人工确认)
- 开发前置未决项:正式域名(D10.5);D4 生产 e2e(真实同步)待人工确认

## 下一阶段建议

D3(Migration 与数据库验收)→ D4∥D5(同步实现 / Sync Queue 与 Worker)→ D6∥D7(Public API / Admin API)→ D8∥D9(Public Frontend / Admin Frontend)→ D10(测试、验收与部署)

- **禁止**在开发过程中重新修改已冻结的 Schema / API 契约 / 信息架构 / 任务拆解语义
- 发现 Schema 不满足需求:**不得直接修改冻结版,必须登记 Schema Change Request**(按 Schema 冻结版文末流程流转)
- 发现 API 契约不满足需求:**不得直接修改冻结版,必须走 API Change Request**(按 API 契约冻结版文末流程流转)
- 开发任务无法执行:先确认是实现问题还是冻结基线问题;实现问题按各包失败回滚原则处理,不得修改冻结文档;拆解自身确需调整 → 新版本任务拆解 + 审计 + 新冻结版(冻结版 §3.2)

## 禁止(开发阶段红线)

- 不修改任何既有冻结版(产品/Schema/API/信息架构/任务拆解)
- 不超出当前开发包的允许修改范围(按 D1-D10 每包第 4/5 项)
- 不新增 Schema 表/字段/枚举/索引(24 表红线)
- 不新增/修改 API 端点与错误码(Public 11 + Admin 17 端点组红线)
- 不实现 P1 功能(/tag /tasks /topic /compare /related 渲染)
- 不扩大首页直查范围(仅 GAP-01 两个只读查询);P1 页面不得继承直查权限
- 首页/搜索不得写入 recommendations;favorites 保持纯本地
- 不直连 SiteIntel PG;同步只走冻结消费契约(v1/report + Key)
- 不修改 SiteIntel / 不新增产品功能 / 不引入未授权新技术组件
