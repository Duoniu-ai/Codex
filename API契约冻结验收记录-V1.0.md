# API 契约冻结验收记录 — V1.0

> 验收日期:2026-08-17
> 验收结论:**PASS —— API 契约设计 V1.0 正式冻结**

---

## 1. 冻结对象

《docs/api/API契约设计-V1.0-冻结版.md》(版本 V1.0,状态:已冻结,冻结日期:2026-08-17)

- 冻结范围 30 项:Public/Admin API 契约、SiteIntel 上游消费契约、搜索行为契约、normalized_domain 规则、Task/Alias/Tag 搜索语义、task_related_tags 辅助匹配边界、网站详情数据返回边界、similar/alternative P0 allowlist、related P1 门控、recommendations 双层隔离、submissions 幂等与限流、reports 字段白名单、events visitor_id 服务端规则、收藏纯前端、ADMIN_EMAILS 单角色、navigation-sync Key 边界、上游字段白名单与禁止持久化、summary_hash 固定 12 字段、sync_queue 状态机(六枚举)、CAS 原子领取、Worker Lock、10 分钟超时恢复、attempts 边界(先 +1 再判断 / >= max_attempts 即 failed / 默认 5 / 第 5 次失败即 failed / 指数退避 / 幂等与断点续跑)、P0/P1 功能边界。
- 候选版《API契约设计-V1.0-候选版.md》保留(不删除、不覆盖),作为审计追溯。

## 2. 冻结日期

2026-08-17

## 3. 输入基线

| # | 基线 | 状态 |
|---|---|---|
| 1 | 《网址发现平台-完整产品规划-V2.1.2-最终冻结版》 | 已冻结 |
| 2 | 《网址发现平台-数据库Schema与数据同步契约-V1.0-冻结版》(24 表) | 已冻结 |
| 3 | 《网址发现平台-技术架构设计-V1.0-候选版》(V1.0-rev2) | 候选版(已完成 Schema 基线同步) |
| 4 | 《API契约设计-V1.0-候选版》(docs/api/) | 候选版(冻结前) |
| 5 | 《API契约冻结审计报告-V1.0》(docs/audit/) | 审计结论 PASS WITH 5 FIXES |
| 6 | 《API契约设计-内部一致性检查报告》(docs/audit/) | 20/20 PASS(含修复后专项复核) |

## 4. 审计过程

**冻结审计结论:PASS WITH 5 FIXES**(0 阻断 / 5 项 P1 / 7 项 P2)。审计方法:字段级逐项对照(15 项方法 + A-M 重点审计项),基线只读。

修复记录(冻结前最小修复,全部仅修改 API 契约候选版,零触碰冻结基线):

| # | 修复 | 内容 |
|---|---|---|
| P1-1 | summary_hash 修复 | §8.5 参与字段固定 12 项、排除字段固定 7 项,与 Schema §12.1 完全一致;删除"固定 10 项"矛盾表述 |
| P1-2 | changes.severity 修复 | §4.3 详情响应删除无 Schema 来源的 severity,改为 {change_type, delta, detected_at}(delta = Schema §2.13) |
| P1-3 | history[] 不再写入 website_changes | §8.3 映射表 history[] → 忽略;P0 变化信号 = 本地 summary_hash diff(Schema §13);change_type 保持冻结 CHECK 仅 siteintel_sync |
| P1-4 | reports.field 白名单修复 | §4.10 field 示例改为 Schema §2.18 冻结白名单 10 项原文;无 title;category 语义 = category_id |
| P1-5 | Admin relations 写通道补齐 | §5.2 新增 POST /api/admin/websites/{id}/relations(relation_type 白名单 similar/alternative,related → 422,UNIQUE 冲突 409)+ PATCH /api/admin/website-relations/{id}(superseded 软失效);引用同步(§4.4/§7/§3.2) |
| P2-9 | task_related_tags 搜索辅助修复 | §4.1 涉及表加入 task_related_tags(读)+ 行为契约补充意图扩展(仅辅助,不替代 website_tasks,不落库原则不变) |
| P2-10 | alias hidden 停用替代硬删除 | §5.4 别名整体替换改为"旧行置 hidden + 新行插入",符合 task_aliases.status 冻结语义 |

## 5. 修复后验证

| # | 验证项 | 结果 |
|---|---|---|
| 1 | 内部一致性检查重跑(20 项) | **20/20 PASS** |
| 2 | 修复后专项检查(S1-S7) | **7/7 PASS** |
| 3 | Schema Change Request | **无**(7 项修复零触碰冻结 Schema) |
| 4 | 无新增 Schema 实体 | ✅(全部接口落 24 表;P1-5 仅补 API 端点,website_relations 冻结语义原样沿用) |
| 5 | 无 P1 功能提前进入 P0 | ✅(related 仍为 P1 门控,接口层三层执行 + 防御性 422;用户端无关系写端点) |
| 6 | 无 recommendations 污染 | ✅(唯一写通道 = Admin §5.8;搜索/任务页/实时计算不落库) |
| 7 | 无服务端收藏 | ✅(无表/无端点/白名单无收藏事件/visitor_id 不参与) |
| 8 | SiteIntel 数据边界符合冻结规则 | ✅(history[] 移除后字段准入与 Schema §11.2 冻结映射完全一致;仅 v1/report + navigation-sync Key 单向消费;evidence.rawData 禁止持久化) |
| 9 | normalized_domain 全链路未回退 | ✅(输入→normalize→lookup→route→submission→queue→logs→SiteIntel→upsert 10 环一致;domain 未承担唯一性/去重/关联/幂等) |
| 10 | sync_queue attempts 边界符合 Schema V1.0 | ✅(attempts 先 +1 再判断;>= max_attempts → failed;默认 5;第 5 次失败即 failed 无第 6 次;超时计入;§9.4 两步骤与 Schema §9 逐字一致) |

## 6. 最终验收结论

**PASS**

API 契约设计 V1.0 正式冻结。API 开发(开发阶段)以《docs/api/API契约设计-V1.0-冻结版.md》为唯一依据;冻结后变更走"API Change Request → 影响分析 → Schema Change Request 判定 → 新版本号 → 内部一致性检查 → 外部/专项审计 → 新冻结版"流程。

## 7. 冻结后已知未决项(不阻塞冻结)

| # | 事项 | 处置 |
|---|---|---|
| 1 | 7 项 P2 遗留(P2-6 name↔display_name 别名声明 / P2-7 reasons.type 枚举绑定 / P2-8 409 差异登记 / P2-11 新增限流值登记 / P2-12 events 不去重约定) | 开发阶段落实,不改变冻结范围 |
| 2 | 网站正式域名未定 | 上线前完成(沿用 Schema 冻结验收记录未决项) |
| 3 | SiteIntel navigation-sync 专用 Key 创建 | 需 SiteIntel 管理员操作 |
| 4 | SiteIntel 缺口 1(analyze 30/h 硬顶)排期 | 小票 |
| 5 | 服务器 PostgreSQL 实例 nav_disc 确认 | 开发阶段 |
| 6 | 技术架构 V1.0-rev2 自身未冻结(仅基线同步) | 待信息架构/开发阶段复核后单独处理 |

## 8. 归档

- 冻结版:docs/api/API契约设计-V1.0-冻结版.md
- 候选版(保留):docs/api/API契约设计-V1.0-候选版.md
- 冻结审计报告:docs/audit/API契约冻结审计报告-V1.0.md
- 内部一致性检查报告:docs/audit/API契约设计-内部一致性检查报告.md
