# Schema 冻结验收记录 — 数据库 Schema 与数据同步契约 V1.0

> 冻结验收日期:2026-08-17
> 冻结对象:《网址发现平台-数据库Schema与数据同步契约-V1.0-冻结版.md》
> 验收结论:**PASS**

---

## 一、冻结对象

- 产品基线:《网址发现平台-完整产品规划-V2.1.2-最终冻结版》(上游依据,冻结版不得修改)
- 架构输入:《网址发现平台-技术架构设计-V1.0-候选版》(V1.0-rev1,含 A1/A2/B1/B2/B3 修订)
- 冻结文档:《网址发现平台-数据库Schema与数据同步契约-V1.0-冻结版.md》
  - 24 张数据表(字段级:主键/时间戳/枚举/索引/删除策略/外键总则)
  - SiteIntel → 导航站数据同步契约(字段级映射 / summary_hash / normalize 流程)

## 二、冻结日期

2026-08-17(Schema 与数据同步契约 V1.0 正式冻结,状态:已冻结)

## 三、审计过程

| 阶段 | 日期 | 结论 |
|---|---|---|
| Schema 初稿完成(23 表) | 2026-08-17 | 候选版 V1.0 |
| 外部深度审计 | 2026-08-17 | **PASS WITH 4 FIXES** |
| A1–A4 修复回写 | 2026-08-17 | V1.0-rev1 候选版(修订记录入文档文末) |
| 冻结前最小修复 M1/M2 | 2026-08-17 | 消除关联语义歧义 + 重试 off-by-one |
| 机械一致性检查 | 2026-08-17 | **18 项全 PASS** |
| 正式冻结 | 2026-08-17 | **V1.0 冻结** |

## 四、A1–A4 修复记录(外部审计)

| # | 审计问题 | 修复 |
|---|---|---|
| A1 | website_relations 的 related 必须 P0/P1 门控 | relation_type 行冻结 **P0 allowlist 仅 similar/alternative**;P0 禁止创建/自动生成/展示 related,related 为 P1 门控,保留枚举值仅为未来兼容(§2.9) |
| A2 | website_metrics.month_favorites 彻底删除 | 字段/索引/说明/检查项引用全部移除,全文零残留(§2.12) |
| A3 | tasks.related_tag_slugs JSONB → 结构化表 | 新增 `task_related_tags`(id/task_id/tag_id/source/created_at,UNIQUE(task_id,tag_id),source CHECK manual/rule);仅辅助搜索意图/Constraint Mapper,**不替代 website_tasks**(§2.7);表数量 23→24 |
| A4 | www 折叠统一到 normalized_domain | websites 删除 UNIQUE(domain),domain 非实体唯一键;submissions UNIQUE(normalized_domain)、sync_queue UNIQUE(normalized_domain, job_type)、sync_logs 增加 normalized_domain 列;§11.1 契约强制 normalize 流程(target.domain → normalize → normalized_domain → 查找 → upsert) |

## 五、M1/M2 最终修复记录(冻结前最小修复)

| # | 修复 | 内容 |
|---|---|---|
| M1 | normalized_domain 关联语义统一 | sync_queue ↔ sync_logs 逻辑关联与索引统一为 normalized_domain(ER 图/§2.22/§8);domain 仅承担原始输入/展示/日志,不承担实体唯一性/去重/关联/幂等 |
| M2 | sync_queue 最大重试边界冻结(off-by-one 消除) | 统一规则:每次失败 attempts+1,用更新后值判定,`attempts >= max_attempts` → failed(默认 5,第 5 次失败即 failed,无第 6 次);`attempts < max_attempts` → retrying + next_retry_at;超时恢复计入 attempts;§9.4 SQL 拆两步骤;settings.sync_queue.max_attempts 可调(≥1) |

## 六、24 表确认

章节编号 2.1–2.24 连续无重复,表清单(与冻结版 §1 矩阵一致):

websites / categories / tags / website_tags / tasks / task_aliases / task_related_tags / website_tasks / website_relations / website_snapshots / website_status / website_metrics / website_changes / recommendations / recommendation_evidence / curated_picks / submissions / reports / user_events / search_logs / sync_queue / sync_logs / settings / admin_users

## 七、机械一致性检查(18 项全 PASS)

| 检查项 | 结果 |
|---|---|
| 1 实体全部可追溯(冻结版/架构/细化拆分) | ✅ |
| 2 无新增账号体系 | ✅ |
| 3 无新增 P1/P2 功能(month_favorites 已删) | ✅ |
| 4 tag_type 严格四枚举 | ✅ |
| 5 无重复实体 | ✅ |
| 6 snapshots/status/metrics/changes 职责不重叠 | ✅ |
| 7 recommendations 不被实时搜索污染 | ✅ |
| 8 favorites 不进服务器数据库 | ✅ |
| 9 sync_queue 完整状态恢复 | ✅ |
| 10 SiteIntel 无数据库耦合 | ✅ |
| 11 summary_hash 无误判 | ✅ |
| 12 索引支撑 P0 查询 | ✅ |
| 13 删除策略不毁历史 | ✅ |
| 14 website_relations P0 allowlist 门控 | ✅ |
| 15 month_favorites 彻底删除 | ✅ |
| 16 related_tag_slugs JSONB 已替代 | ✅ |
| 17 normalized_domain 为唯一实体键(含 sync_queue↔sync_logs 关联) | ✅ |
| 18 sync_queue 重试边界冻结(off-by-one 消除) | ✅ |

## 八、最终结论

**PASS。** 冻结版文档满足:与产品基线 V2.1.2 一致、无新增产品功能、无 P0/P1 边界变更、机械检查 18 项全通过、未执行 schema.prisma / migrate / API / 业务代码。

## 九、冻结范围

- 24 张数据表;
- normalized_domain 实体规则;
- Task / Tag 数据模型;
- task_aliases / task_related_tags / website_tasks;
- recommendations / recommendation_evidence;
- website_snapshots / website_status / website_metrics / website_changes / website_relations;
- sync_queue / sync_logs(原子领取与 Worker Lock、最大重试边界);
- SiteIntel → 导航站数据同步契约;
- summary_hash 计算规则;
- P0 / P1 数据边界。

冻结后任何修改必须走:变更提案 → 影响分析 → 新版本号 → 审计 → 新冻结版(见冻结版文末"冻结后变更规则")。

## 十、已知未决项(仅记录,不影响冻结,不修改 Schema)

| # | 未决项 | 说明 |
|---|---|---|
| 1 | 网站正式域名 | 上线前须完成,不影响 Schema 冻结 |
| 2 | SiteIntel navigation-sync 专用 API Key | 需 SiteIntel 管理员创建(rateLimitPerHour 按需) |
| 3 | SiteIntel analyze 30/h 硬顶 | 缺口 1,冷启动节奏影响,小票排期 |
| 4 | SiteIntel v1/events 增量端点 | 缺口 2,P1 收藏更新提醒前优化项 |
| 5 | PostgreSQL 独立数据库 nav_disc 部署确认 | 服务器实例/权限,开发前确认 |
| 6 | Prisma relationMode 与数据库 FK 实现方式 | 开发阶段确定(不改变实体语义) |
