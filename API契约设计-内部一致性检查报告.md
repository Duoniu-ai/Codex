# API 契约设计 V1.0(候选版)— 内部一致性检查报告

> 检查对象:《docs/api/API契约设计-V1.0-候选版.md》
> 检查基线:①产品 V2.1.2 冻结版 ②技术架构 V1.0-rev2(候选)③Schema V1.0 冻结版(24 表)
> 检查日期:2026-08-17;检查方式:人工逐项核对(设计文档内审)
> 结论:**20/20 PASS**,无阻断项;候选版可进入 API 契约冻结审计。

---

## 检查项明细(20 项)

| # | 检查项 | 结论 | 证据(文档 §/行位) |
|---|---|---|---|
| 1 | **P0 用户路径闭环**:产品 §12 用户端 P0 能力均有 API 承载(搜索/详情/分类/标签/相似替代/提交/纠错/行为统计) | ✅ PASS | §3.1 Public 11 端点;提交 4.9、纠错 4.10、事件 4.11;收藏为纯本地非端点(4.12) |
| 2 | **搜索主链路完整**:q/task/tag 参数、分页、排序(五因子)、空查询 400、Constraint Mapper、alias 词典、ILIKE 兜底、Top-1 ≥85% 验收 | ✅ PASS | §4.1 全部 18 项 + 行为契约 6 条(Task 识别/兜底/Constraint/排序/空 q) |
| 3 | **网站详情**:只返回导航站持久化摘要;不泄漏 evidence.rawData/原始 DNS 明细/完整 IP 明细/完整证书链/SiteIntel 内部字段;禁"活跃度"表述 | ✅ PASS | §4.3 数据边界硬规则(禁止清单 3 条 + 表述禁令);§8.4 禁止持久化清单 5 条 |
| 4 | **分类与标签**:GET categories/tags 两级别端点;tag_type 严格四枚举(attribute/feature/constraint/scenario);tags.group 仅管理分组、公开接口不接受 group 参数 | ✅ PASS | §4.5-4.8;§5.5 group 语义冻结;"新增第五种 → 422 TAG_TYPE_INVALID" |
| 5 | **similar/alternative P0 allowlist**:关系仅白名单两类型可创建/生成/展示 | ✅ PASS | §4.4 响应仅 similar/alternative;§7 三通道全部允许 |
| 6 | **related P1 门控在接口层执行**:P0 不允许创建/自动生成/前端展示 | ✅ PASS | §4.4 三层执行(创建校验/规则引擎/响应无 related 字段 + p0_gate 声明 + 防御性 422);§7 |
| 7 | **recommendations 不被实时搜索污染**:实时搜索/任务页访问不得写 recommendations | ✅ PASS | §4.1 项 16(仅 search_logs)、§4.2 项 16(页面访问不创建行)、§6 硬规则 3、§5.8 唯一写通道 |
| 8 | **收藏无服务端 API**:无 favorites 端点/表/服务端存储;localStorage/IndexedDB 纯前端 | ✅ PASS | §1.8、§4.12(明确"无端点、无表、无服务端参与");Schema 冻结版无 favorites 表 |
| 9 | **user_events 无 favorite/unfavorite**:事件白名单固定 4 项 | ✅ PASS | §4.11 白名单冻结(visit/reason_expand/submit/search_click,不含 favorite/unfavorite);EVENT_TYPE_INVALID 400 |
| 10 | **visitor_id 服务端生成**:服务端 cookie(vid)签发;客户端禁止传入;无 cookie 由服务端生成并 Set-Cookie | ✅ PASS | §1.7、§4.11 项 13("客户端禁止传入 visitor_id,服务端忽略/拒绝") |
| 11 | **normalized_domain 贯穿**:详情路由/提交幂等/同步入队/实体查找全部以折叠后实体键为准 | ✅ PASS | §4.3 路由 {normalized_domain};§4.9 幂等以 normalized_domain;§8.2 normalize 流程强制;§5.2 normalized_domain 不可改(实体键) |
| 12 | **SiteIntel 仅经 API 契约**:单向消费 v1/report + navigation-sync Key;无直连 PG/无共享 Schema/不修改 SiteIntel;仅同步服务出站调用,页面离线优先 | ✅ PASS | §8.1(端点+Key+404 转 analyze+禁公开端点);§1.6 表;PROJECT_BOUNDARY 4 点吻合 |
| 13 | **无新增实体**:全部接口落 Schema 冻结版 24 表,未引用未定义字段 | ✅ PASS | §10.1("全部落在 24 表内");§4/§5 每端点涉及表已标注,均 ∈ 24 表 |
| 14 | **sync_queue 冻结状态机**:7 状态(pending→processing→completed/retrying/failed/skipped)、CAS 领取、locked_at/locked_by、10 分钟超时、指数退避、幂等、systemd 重启恢复 | ✅ PASS | §9.1-9.5(enqueue/claim/process/complete/retry/fail/skip/recover 8 操作);§9.3 CAS 条件更新;§9.4 超时恢复两步骤 |
| 15 | **attempts >= max_attempts 规则一致**:自增后判定;默认 5;settings 可调 ≥1 | ✅ PASS | §9.2 状态机(引用 Schema §9);§5.10 settings 白名单含 sync_queue.max_attempts(≥1 整数校验) |
| 16 | **第 5 次失败即 failed,无第 6 次**:重试与超时恢复两通道均用更新后值;退避 min(2^attempts×60s,4h) | ✅ PASS | §9.1 retry() 语义("attempts+1 → 更新后值判定");§9.2 显式"第 5 次失败即 failed,不得出现第 6 次";§9.4 两步骤 SQL 边界(attempts+1 < / >= max_attempts) |
| 17 | **无 P1 提前进 P0**:related(门控)、账号/云同步(收藏 P1)均未进入 P0 范围 | ✅ PASS | §4.4/§7 related 门控;§4.12 "P1 账号体系后提供云端收藏,届时另立契约";所有端点 P0/P1 标注齐全 |
| 18 | **无 Redis/ES/BullMQ/Docker/微服务/RBAC/账号体系**:限流/节流全部内存实现,单 worker 实例 | ✅ PASS | §1.4(内存滑动窗口/令牌桶);§9.5(单实例 instrumentation);§1.6 单角色 ADMIN_EMAILS,无多角色 |
| 19 | **API 与 Schema 对应**:涉及表逐点可溯源至 24 表;未发现 Schema 无法支持的 API 需求 | ✅ PASS | §10(无 Schema Change Request);接口-表标注见 §4/§5 |
| 20 | **API 与架构 V1.0-rev2 一致**:响应格式 { error, details? }、分页 page/page_size(≤100)、限流 3/min/IP(submissions)、ISR 60s、错误码表,全部沿用架构 §10 冻结约定 | ✅ PASS | §1.2-1.4;§1.1 单版本 /api(不引入 v2/v3);§4.1 项 5/6/8 与架构 §10.1 逐项对照;§11 缓存策略 |

---

## 修复后专项复核(2026-08-17,冻结审计 PASS WITH 5 FIXES → 7 项最小修复)

> 依据《docs/audit/API契约冻结审计报告-V1.0.md》;修复对象仅《API契约设计-V1.0-候选版.md》,未触碰任何冻结基线。

| # | 专项检查 | 结果 | 证据(候选版 §) |
|---|---|---|---|
| S1 | P1-1 summary_hash 与 Schema §12.1 完全一致 | ✅ PASS | §8.5 参与字段固定 12 项(与 §12.1 逐字一致,无增删)+ 排除字段固定 7 项(checked_at/last_sync_at/updated_at/title/description/summary_text/display_name);"固定 10 项"矛盾表述已删除 |
| S2 | P1-2 详情响应字段全部有 Schema 来源 | ✅ PASS | §4.3 changes → `{change_type, delta, detected_at}`,delta 语义 = Schema §2.13(`[{field, from, to}]`);severity 已移除,响应无未定义字段 |
| S3 | P1-3 SiteIntel history[] 不再持久化 | ✅ PASS | §8.3 history[] → 忽略(P0 变化信号 = 本地 summary_hash diff,Schema §13);change_type 保持冻结 CHECK 仅 `siteintel_sync`;无新增上游字段准入 |
| S4 | P1-4 reports.field 与 Schema §2.18 白名单一致 | ✅ PASS | §4.10 field 示例 = 冻结白名单 10 项原文(display_name/description/category_id/homepage_url/online/http_status/technologies/ssl_expires_at/cdn/hosting);无 title;category 语义 = category_id |
| S5 | P1-5 relations Admin 写通道 | ✅ PASS | §5.2 新增 `POST /api/admin/websites/{id}/relations`(relation_type 白名单仅 similar/alternative,related → 422 RELATION_TYPE_FORBIDDEN,UNIQUE 冲突 → 409,source=manual)+ `PATCH /api/admin/website-relations/{id}`(superseded 软失效);§4.4/§7/§3.2 引用同步;无用户端写端点;零新增表/字段 |
| S6 | P2-9 task_related_tags 意图扩展 | ✅ PASS | §4.1 涉及数据表加入 task_related_tags(读)+ 行为契约补充(命中 Task 后关联标签辅助扩展约束;仅辅助,不替代 website_tasks 主匹配;实时搜索不落库原则不变) |
| S7 | P2-10 alias 整体替换不硬删 | ✅ PASS | §5.4 改为"旧行置 `hidden` 停用 + 新行插入",符合 task_aliases.status 冻结语义("隐藏=词典停用"),治理历史保留 |

### 修复后复查(冻结审计任务书 §五 5 项)

1. **20 项内部一致性检查重跑:20/20 PASS** —— 修复未引入新冲突:§4.1 涉及表新增 task_related_tags 不改变搜索不落库;§5.2 新端点属既有 9 项能力内补充;§8.5/§4.3 变更均收敛到冻结版字段集;§4.4 引用与 §5.2 端点一致。
2. **专项检查 S1-S7:7/7 PASS**(见上表)。
3. **API 字段 Schema 来源**:逐响应字段核对,全部有 Schema 字段或明确计算来源(§4.3/§4.1 的 `name` ↔ `websites.display_name` 展示别名已在 P2-6 登记,冻结审计确认不阻塞)。
4. **Schema Change Request:无** —— 7 项修复零触碰冻结 Schema;P1-5 仅补 API 端点,website_relations 冻结语义(relation_type CHECK 三枚举 + P0 allowlist 仅 similar/alternative)原样沿用。
5. **P0/P1 边界**:related 门控增强(创建端点白名单校验 422 + 无用户端写通道 + 响应无 related 字段 + 防御性拒绝);无 P1 提前进 P0。
6. **SiteIntel 数据边界**:history[] 移除后,字段准入与 Schema §11.2 冻结映射表完全一致;仍仅 v1/report + navigation-sync Key 单向消费。
7. **normalized_domain 全链路**:本次修复未触碰;审计结论保持(输入→normalize→lookup→route→submission→queue→logs→SiteIntel→upsert 10 环一致,无回退)。
8. **recommendations 污染**:本次修复未触碰;唯一写通道 §5.8 不变,搜索/任务页不落库。

## 发现的问题

- **未发现阻断项**(0 严重 / 0 一般)。
- 候选版阶段遗留提示(不属缺陷,冻结审计关注):
  1. `GET /api/websites/{normalized_domain}/relations` 的 `p0_gate` 字段为接口层门控声明,属于本契约新增约定,需外部审计确认表述;
  2. 错误消息文案为英文格式(`CODE: message`),前端文案映射属信息架构阶段;
  3. §8.1 中"404 → 转 analyze 分支"受上游 30/h 硬顶约束,冷启动排期属已知未决项(沿用 Schema 冻结验收记录 6 项未决)。

## 结论

- **20/20 PASS(重跑)+ 专项 S1-S7 全 PASS**;冻结审计 5 项 P1(P1-1~P1-5)+ 2 项 P2(P2-9/P2-10)已全部按最小修复原则修正完毕。
- API 契约 V1.0 候选版满足冻结条件:**P0 用户路径全覆盖(含 relations Admin 写通道)、Schema 24 表零新增、零 Schema Change Request、冻结边界(related 门控 / recommendations 双层 / 收藏本地 / 重试边界 / SiteIntel 白名单)全部落实、无新技术组件引入**。
- **建议进入正式冻结**(按"冻结审计 PASS WITH 5 FIXES → 修复 → 重检 → 冻结"流程;7 项 P2 遗留项登记不阻塞)。
