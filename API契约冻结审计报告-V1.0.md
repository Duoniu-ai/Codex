# 网址发现平台 — API 契约 V1.0 最终冻结审计报告

> 审计对象:
> ① `docs/api/API契约设计-V1.0-候选版.md`
> ② `docs/audit/API契约设计-内部一致性检查报告.md`
> 审计基线(只读,不修改):
> ③ 产品规划 V2.1.2 最终冻结版(docs/product/)
> ④ Schema 与数据同步契约 V1.0 冻结版(docs/architecture/,24 表,下称 Schema 冻结版)
> ⑤ 技术架构 V1.0-rev2 候选版(docs/architecture/)
> ⑥ PROJECT_STATUS.md / PROJECT_BOUNDARY.md
> 审计日期:2026-08-17;审计方式:字段级逐项对照(非"是否写了"的表面检查)。
> **结论:PASS WITH 5 FIXES** —— 0 阻断(P0);5 项 P1 建议修复后冻结;7 项 P2 不阻塞冻结;无 Schema Change Request。

---

# 1. 审计方法覆盖(任务书二,15 项)

| # | 方法项 | 执行情况 |
|---|---|---|
| 1 | 字段级检查 | ✅ 逐响应字段对照 Schema 表字段(§2/§4 逐表) |
| 2 | Schema 对照 | ✅ 24 表逐表核对;接口-表标注全部落 24 表内 |
| 3 | API 请求参数检查 | ✅ §4/§5 全部端点参数核对(类型/长度/枚举/必填) |
| 4 | API 响应字段检查 | ✅ 逐响应字段确认 Schema 来源或计算来源(**发现问题 P1-2/P2-6**) |
| 5 | 错误码检查 | ✅ 与架构 §10 错误码基础约定对照(发现差异登记 D-1) |
| 6 | 幂等性检查 | ✅ submissions/sync_queue/Admin 创建类逐项核对 |
| 7 | 鉴权检查 | ✅ 匿名/Admin 单角色/Key 三层核对 |
| 8 | 限流检查 | ✅ 逐端点核对(发现新增约定登记 D-2) |
| 9 | P0/P1 边界检查 | ✅ related 门控三层执行核对(**发现 P1-5 关系写通道缺口**) |
| 10 | SiteIntel 数据边界检查 | ✅ §8 白名单与 Schema §11.2 逐行对照(**发现 P1-3**) |
| 11 | 同步队列状态机检查 | ✅ §9 与 Schema §9 逐条对照(off-by-one 消除,一致) |
| 12 | 推荐系统数据污染检查 | ✅ recommendations 写入通道唯一性核对 |
| 13 | 收藏逻辑检查 | ✅ 无表/无端点/白名单无收藏事件核对 |
| 14 | normalized_domain 全链路检查 | ✅ 输入→normalize→lookup→route→submission→queue→logs→SiteIntel→upsert 逐环核对 |
| 15 | 管理后台 API 权限边界检查 | ✅ 单角色/无 RBAC/9 项能力覆盖/门控绕过检查 |

---

# 2. 重点审计项逐项发现(A-M)

## A. API 与 Schema 一致性

**结论:整体一致,发现 4 处字段级问题**(详见 §3 问题清单 P1-1/P1-2/P1-3/P1-4):

- ✅ 全部接口涉及表 ∈ 24 表;无接口引用不存在的表。
- ✅ 未发现任何 API 隐式要求新增表。
- ✅ 未发现任何 API 需要修改冻结 Schema 才能实现(全部问题可在 API 契约层修复)。
- ❌ **P1-1** §8.5 summary_hash 参与/排除字段与 Schema §12.1 不一致且文档自相矛盾。
- ❌ **P1-2** §4.3 详情响应 `changes[].severity` 无 Schema 来源。
- ❌ **P1-3** §8.3 将 SiteIntel `history[]` 映射进 website_changes,违反 Schema §11.2(history[] 忽略)。
- ❌ **P1-4** §4.10 reports `field` 示例(`title|description|category`)不在 Schema §2.18 冻结白名单内。

## B. normalized_domain 全链路

**结论:一致,无回退。**

| 环节 | 候选版 | Schema 冻结版 | 判定 |
|---|---|---|---|
| 输入 | submissions body 仅收 `domain`,服务端折叠(§4.9 项12) | §2.17 submissions.normalized_domain 服务端派生 | ✅ |
| normalize | §8.2 折叠规则:小写/去协议/去 www/去尾斜杠/去端口/punycode | §5 折叠规则(www 折叠,子域名保留) | ✅ 一致 |
| entity lookup | §8.2 按 normalized_domain 查找,禁按裸 domain(§8.2/§11.1) | §11.1 契约强制 | ✅ 一致 |
| route | §4.3 详情路由 `{normalized_domain}` | §2.1 实体键 | ✅ |
| submission | §4.9 UNIQUE 冲突 409;www/裸域同站去重 | §2.17 UNIQUE(normalized_domain) | ✅ |
| sync_queue | §9.1 enqueue(normalized_domain);UNIQUE(normalized_domain, job_type) 幂等 | §2.21 同约束 | ✅ |
| sync_logs | §9 关联用 normalized_domain | §2.22 INDEX(normalized_domain, created_at) | ✅ |
| SiteIntel 查询 | §8.1 以 target.domain 出站;折叠后查找(§8.2) | §11.1 normalize 流程 | ✅ |
| upsert | §8.2 存在则 upsert 原站,不新建 | §11.1 同 | ✅ |
| Admin | §5.2 normalized_domain 服务端派生、PATCH 不可改(实体键) | §2.1 unique | ✅ 更严 |

- domain 在契约中仅出现于:原始输入(§4.9 body/submissions.domain)、展示/日志(§4.3 响应 domain 字段、sync_queue/sync_logs domain 列)、SiteIntel 出站原始值(§8.2)——**未承担任何唯一性/去重/关联/幂等职责**。✅

## C. P0/P1 边界(website_relations)

**结论:门控语义正确,发现 1 处能力缺口(P1-5)。**

- ✅ related 创建门控:候选版 §5.2/§5.8 无任何 relation_type 创建端点;§4.4 读取过滤 + 防御性 422。
- ✅ related 自动生成:规则引擎不产出(§7)。
- ✅ related 前端展示:响应无 related 字段 + p0_gate 声明(§4.4)。
- ✅ 恶意绕过防御:relation_type 白名单校验 → 422 RELATION_TYPE_FORBIDDEN(§4.4/§7)。
- ❌ **P1-5** similar/alternative 的 **Admin 写通道缺失**:产品冻结版 §12 用户端 P0 明确"相似网站;人工精选替代网站",数据来源 = 编辑人工(Schema §1 website_relations"编辑人工(替代优先)/规则")。候选版 Admin 端点组(§5.2-§5.12)无 relations 创建/更新端点 → P0 闭环缺数据写入通道;且门控"P0 只允许 similar/alternative"缺少正面承载端点。非门控绕过,属能力缺口。

## D. recommendations 数据污染

**结论:无污染风险。**

- ✅ 唯一写通道 = §5.8 Admin 推荐依据维护(编辑决策/固定推荐集/人工覆盖)。
- ✅ §4.1 搜索(项16)/§4.2 任务页(项16)明确"不写 recommendations"。
- ✅ §6 硬规则 3:"任何一次用户搜索或任务页访问不得新增 recommendations 行"。
- ✅ 生命周期只 superseded 不硬删(§5.8 supersede 端点;无 DELETE)。
- ✅ context 枚举(task_page/search/detail/home)、confidence(high/medium/low)、evidence_type 五枚举与 Schema §2.14/§2.15 一致。

## E. 收藏系统

**结论:无服务端收藏。**

- ✅ 无 favorites 表(Schema 全文无;候选版无)。
- ✅ 无 POST/DELETE /api/favorites(候选版无任何收藏端点;§4.12 明确"无端点、无表、无服务端参与")。
- ✅ user_events 白名单 4 项,无 favorite/unfavorite(§4.11 与 Schema §2.19 CHECK 一致)。
- ✅ visitor_id 仅匿名行为统计、禁止参与收藏(§1.7/§4.11 与 Schema §3 一致)。

## F. SiteIntel 数据边界

**结论:基本一致,发现 1 处白名单违反(P1-3)。**

- ✅ 仅稳定 API 消费(v1/report + navigation-sync Key,§8.1);无直连 DB/共享 Prisma schema。
- ✅ evidence.rawData 禁止持久化(§8.4 与 Schema §11.2 一致)。
- ✅ 完整 DNS/IP 明细/证书链不持久化(§8.4 与 Schema §11.2 一致)。
- ✅ SiteIntel 内部字段不作为产品依赖:audit 仅健康摘要、口径"规则审计非质量评分"(§4.3/§8.4 与 Schema §11.2 一致)。
- ❌ **P1-3** §8.3 映射表将 `history[]`(仅 type/severity/detectedAt)→ website_changes。Schema §11.2 冻结:**history[] ❌ 忽略**;P0 变化信号 = 本地 summary_hash diff(Schema §13);website_changes.change_type CHECK 仅 `siteintel_sync`(§2.13,"上游 events API 上线后扩展")。候选版此行为为冻结契约之外的字段准入。

## G. 网站详情 API

**结论:边界正确,发现 2 处字段级问题(P1-2/P2-6)。**

- ✅ 只返回导航站持久化摘要;禁止字段清单(evidence.rawData/原始 DNS/完整 IP/完整证书链/上游内部字段)在 §4.3 数据边界硬规则落实。
- ✅ 无"活跃度"表述;status_signal 语义 = "最近检测正常访问/数据更新时间/近期变化"(§4.3 与 Schema §2.11 口径一致);online 不作"活跃"解释。
- ❌ **P1-2** 响应 `changes: [{change_type, severity, detected_at}]`:Schema website_changes 字段 = `change_type/delta/hash_prev/hash_next/created_at`,**无 severity 字段**;且 SiteIntel history 的 severity 语义不在冻结白名单。`severity` 无 Schema 来源。
- ⚠️ **P2-6** 响应 `website.name` ↔ Schema `websites.display_name` 别名未声明(搜索/详情/任务页/分类页/标签页多处使用 name)。

## H. 搜索 API

**结论:主链路正确,发现 1 处遗漏(P2-9)、1 处状态码差异登记(D-1)。**

- ✅ 空 q → 400 VALIDATION_ERROR(§4.1)。
- ✅ alias 词典优先(Task 识别:全等 > 前缀/包含 > 多 Token,§4.1 行为契约)。
- ✅ Constraint Mapper(constraint.dictionary → tag 映射,settings 承载)。
- ✅ ILIKE 兜底(websites.name/title/description + tasks.name)。
- ✅ Top-1 ≥85% 验收引用(§4.1 行为契约)。
- ✅ 五因子排序(权重 settings.ranking.weights,零黑盒,§4.1/§11)。
- ✅ page/page_size ≤100(§1.2 与架构 §10 冻结约定)。
- ✅ 不写 recommendations(§4.1 项16)。
- ✅ 无未定义 Schema 字段(constraints = tag slug 列表;task = tasks)。
- ⚠️ **P2-9** Schema §2.7 冻结 task_related_tags 用途:"仅用于搜索意图辅助与 Constraint Mapper 辅助"。候选版 §4.1 行为契约/涉及表未接入该用途(数据来源仅 task_aliases/tasks/tags/website_tasks/websites/website_status/settings),该表在搜索执行层无落点。
- ⚠️ **D-1** submissions 重复提交:架构 §10.1(候选版)写"422 重复提交",候选版用 409 DUPLICATE_SUBMISSION(统一错误码体系语义更准)——见 §4 差异登记。

## I. submissions 幂等性

**结论:一致。**

- ✅ normalized_domain 服务端计算(§4.9 项12"禁止信任客户端";请求体仅收 domain)。
- ✅ www 折叠:www.example.com 与 example.com 视为同一实体,二次提交 409(§4.9 行为契约)。
- ✅ 409 DUPLICATE_SUBMISSION 定义一致(§1.3 表 ↔ §4.9)。
- ✅ 3/min/IP(§1.4 与架构 §10.1 冻结值一致)。
- ✅ 同请求重试幂等无害(UNIQUE 约束,不产生重复行)。

## J. reports 纠错接口

**结论:机制正确,发现 1 处白名单不一致(P1-4)。**

- ✅ 匿名(§4.10,无账号要求)。
- ✅ 10/min/IP(§1.4;架构未冻结该值,登记 D-2)。
- ✅ 非法字段 422 FIELD_NOT_ALLOWED(§4.10)。
- ✅ 进入管理审核(§5.7 处理队列,accept/reject/resolve)。
- ✅ 不直接修改网站事实数据:仅管理端 accept 时按 field 白名单回写,resolved_value 留痕(Schema §2.18)。
- ❌ **P1-4** §4.10 Request Body 示例 `"field": "title|description|category|online|..."`:Schema §2.18 冻结白名单 = `display_name/description/category_id/homepage_url/online/http_status/technologies/ssl_expires_at/cdn/hosting`。示例中的 `title`(Schema 无此字段可纠错;title 为 S 来源仅供编辑参考)与 `category`(Schema 字段为 category_id)不成立。

## K. events

**结论:一致。**

- ✅ visitor_id 仅服务端 HttpOnly cookie(`vid`)生成/读取,客户端提交被忽略/拒绝(§1.7/§4.11 项13)。
- ✅ 事件白名单仅 visit/reason_expand/submit/search_click,无 favorite/unfavorite(§4.11 ↔ Schema §2.19)。
- ✅ 60/min/IP(§1.4,登记 D-2);不保证精确去重(行为统计)。

## L. Admin API

**结论:权限边界正确,9 项能力覆盖完整;发现 P1-5(关系写通道)与 P2-10(alias 替换语义)。**

- ✅ 单角色 ADMIN_EMAILS 白名单交集校验;401/403 区分;无 RBAC/无多角色(§5.0)。
- ✅ 9 项能力逐项覆盖(产品 §12 管理端):网站管理 §5.2 / Category §5.3 / Task §5.4 / Tag §5.5 / 提交审核 §5.6 / 纠错队列 §5.7 / 推荐依据维护 §5.8 / 精选管理 §5.9 / 基础发布状态 §5.10(settings)+ §5.2(status 流转)。
- ✅ 无门控绕过:无 relation_type 创建端点(因此无 related 绕过通道;但参见 P1-5 写通道缺失)。
- ✅ 删除策略:全 Admin 无 DELETE;推荐只 superseded、Tag/Category 只归档(符合 Schema §10)。
- ⚠️ **P2-10** §5.4 aliases 整体替换"旧行删/新行插":Schema §2.6 有 status=hidden("隐藏=词典停用")、§10 未冻结 task_aliases 删除策略;硬删旧行丢失 alias 治理历史(alias_normalized 冲突追溯)。

## M. sync_queue 状态机

**结论:与 Schema §9 完全一致,无 off-by-one。**

| 检查点 | 候选版 §9 | Schema 冻结版 §9 | 判定 |
|---|---|---|---|
| 状态枚举 | pending/processing/completed/retrying/failed/skipped | 同(§2.21 CHECK) | ✅ 一致 |
| attempts 先 +1 再判断 | §9.1 retry()/fail() 语义 | §9 规则 1-2 | ✅ |
| attempts >= max_attempts → failed | §9.2 | §9 规则 2 | ✅ |
| 第 5 次失败即 failed,无第 6 次 | §9.2 显式 | §9 规则 3 | ✅ |
| 超时计入 attempts | §9.4 两步骤(attempts+1 < / >= max) | §9.4 SQL 两步骤同边界 | ✅ |
| CAS 原子领取 | §9.3 单条条件更新,count==1 | §9.3 同 | ✅ |
| locked_at / locked_by | §9.3 | §2.21 | ✅ |
| 10 分钟超时恢复 | §9.4 | §9.4 | ✅ |
| systemd 重启恢复 | §9.4 断点续跑 | §9.4 | ✅ |
| 指数退避 | §9.2 min(2^attempts×60s, 4h) | §9.2 同 | ✅ |
| 幂等 | §9.5 UNIQUE + upsert + hash 零写 | §9.4 | ✅ |

> 注:任务书称"7 状态",Schema 冻结版 CHECK 为 6 枚举(pending/processing/completed/retrying/failed/skipped);以冻结版为准,候选版一致。

---

# 3. 问题清单

## P0 阻断问题(必须修复才能冻结)

**无。**

## P1 重要问题(建议修复后冻结)

### P1-1 summary_hash 参与/排除字段描述与 Schema §12.1 不一致(且自相矛盾)

- **严重等级**:P1
- **所在文档章节**:候选版 §8.5
- **具体冲突**:①§8.5 写"参与字段(固定 10 项)"但实际列出 12 个字段(online/http_status/ip/asn/asn_org/country/technologies/ssl_expires_at/cdn/hosting/seo_summary/health_summary),"10 项"与列出的 12 项自相矛盾;②排除字段仅列 4 项(checked_at/last_sync_at/title/description),Schema §12.1 排除 7 项(另含 updated_at/summary_text/display_name)。
- **影响**:实现阶段按 §8.5 实现会得出与冻结版不同的 hash 输入集合 → 变化判定结果偏离冻结契约;该规则是同步去重与 website_changes 触发源,偏差影响数据正确性。
- **修复建议**:§8.5 改为直接引用 Schema §12.1:12 参与字段(与快照 data 固定字段集一致)+ 7 排除字段,删除"10 项"表述。
- **是否涉及 Schema Change Request**:否(仅修正契约文档引用,不改 Schema)。

### P1-2 网站详情响应 changes[].severity 无 Schema 来源

- **严重等级**:P1
- **所在文档章节**:候选版 §4.3(Response 成功)
- **具体冲突**:`changes: [{change_type, severity, detected_at}]` 的 `severity` 不在 Schema website_changes 字段集(change_type/delta/hash_prev/hash_next/created_at,§2.13);delta 才是变化内容载体。
- **影响**:响应字段无 Schema 来源,实现阶段无从取值(或从禁止持久化的 SiteIntel history 取,违反 F 边界)。
- **修复建议**:响应改为 `changes: [{change_type, delta, detected_at}]`(delta 为 `[{field, from, to}]`,Schema §2.13);页面展示层在信息架构阶段决定如何呈现 delta。
- **是否涉及 Schema Change Request**:否。

### P1-3 §8.3 将 SiteIntel history[] 映射进 website_changes,违反 Schema §11.2

- **严重等级**:P1
- **所在文档章节**:候选版 §8.3(字段白名单表)
- **具体冲突**:Schema §11.2 冻结映射:`history[] ❌ 忽略`(P0 变化信号 = 本地 summary_hash diff,§13;website_changes.change_type CHECK 仅 `siteintel_sync`,§2.13,上游 events API 上线后扩展)。候选版写"history[](仅 type/severity/detectedAt)→ website_changes"。
- **影响**:这是冻结契约之外的 SiteIntel 字段准入 —— 违反"只允许冻结白名单字段进入导航站库"的 SiteIntel 数据边界;实现时还会引入 severity 语义(与 P1-2 联动)。
- **修复建议**:§8.3 删除 history 行,改为"history[]:忽略(变化信号由本地 summary_hash diff 产生,引用 Schema §13;上游 events API 上线后再扩展)"。
- **是否涉及 Schema Change Request**:否(回到冻结白名单,是契约文档修正)。

### P1-4 reports.field 示例与 Schema §2.18 冻结白名单不一致

- **严重等级**:P1
- **所在文档章节**:候选版 §4.10(Request Body)
- **具体冲突**:示例 `"field": "title|description|category|online|..."`:`title` 不在 Schema §2.18 白名单(白名单 = display_name/description/category_id/homepage_url/online/http_status/technologies/ssl_expires_at/cdn/hosting);`category` 在 Schema 中为 `category_id`。
- **影响**:实现阶段可能放开 title 纠错(回写 S 来源字段,违反 §1 覆盖规则),或误用 category 与 category_id 映射错位。
- **修复建议**:示例改为冻结白名单原文:`"field": "display_name|description|category_id|homepage_url|online|http_status|technologies|ssl_expires_at|cdn|hosting"`。
- **是否涉及 Schema Change Request**:否。

### P1-5 similar/alternative 关系缺少 Admin 写通道(P0 闭环缺口)

- **严重等级**:P1
- **所在文档章节**:候选版 §5.2-§5.12(Admin 端点组)
- **具体冲突**:产品冻结版 §12 用户端 P0 = "相似网站;人工精选替代网站";Schema §1 website_relations 数据来源 = "编辑人工(替代优先)/规则"。候选版 Admin 端点组无 relations 创建/更新端点(§4.4 仅读取;§5.2 网站管理可带 category/tags/tasks,无 relations)。
- **影响**:P0 用户端"相似/替代"无数据写入通道,闭环不成立;且 P1 门控"P0 仅允许 similar/alternative"缺乏正面创建端点的承载(当前只有防御性 422)。
- **修复建议**:§5.2 增加 `POST /api/admin/websites/{id}/relations`(body: `{ website_id_to, relation_type: "similar"|"alternative", basis, confidence }`,relation_type 白名单校验,related → 422 RELATION_TYPE_FORBIDDEN,source=manual)+ `PATCH /api/admin/website-relations/{id}`(状态流转 superseded)。这是对现有 9 项能力中"网站管理"的端点补充,非新增产品功能。
- **是否涉及 Schema Change Request**:否(website_relations 表已冻结,仅 API 层补端点)。

## P2 优化问题(不阻塞冻结)

| # | 问题名称 | 章节 | 冲突/说明 | 修复建议 | SCR |
|---|---|---|---|---|---|
| P2-6 | 响应字段 `name` ↔ Schema `display_name` 别名未声明 | §4.1/4.2/4.3/4.6/4.8 | 响应 `website.name` 实际取值 websites.display_name;响应字段需"Schema 或明确计算来源" | §4 增加响应字段别名说明(或在 §1 通用约定声明 name=display_name 展示别名) | 否 |
| P2-7 | reasons.type 未声明与 evidence_type 映射 | §4.1 | `reasons: [{ type, text }]` 的 type 未绑定架构 §8.4 五枚举 | 声明 type ∈ {task_match, constraint_match, quality, status_signal, editorial}(内存生成,不落库) | 否 |
| P2-8 | submissions 重复提交状态码与架构 §10.1 差异 | §1.3/4.9 | 架构(候选版)写"422 重复提交",契约用 409 DUPLICATE_SUBMISSION | 在契约中显式登记差异及理由(409 冲突语义;架构为候选版未冻结,不阻塞) | 否 |
| P2-9 | 搜索执行层未接入 task_related_tags 意图扩展 | §4.1 | Schema §2.7 冻结用途"辅助搜索意图/Constraint Mapper";§4.1 数据来源未列该表 | §4.1 行为契约补一句(task_related_tags 参与意图扩展,读取),涉及表加入 | 否 |
| P2-10 | task_aliases 整体替换采用硬删旧行 | §5.4 | Schema §2.6 status=hidden"词典停用";§10 未冻结 alias 删除策略 | 改为"旧行置 hidden + 新行插入"(保留治理历史)或声明为开发阶段决策 | 否 |
| P2-11 | 新增限流值为契约新增约定 | §1.4 | reports 10/min/IP、events 60/min/IP、Admin 120/min 架构未冻结 | 在契约登记为 API 层新增约定(不违反任何冻结基线) | 否 |
| P2-12 | events 不保证精确去重 | §4.11 | 行为统计允许重复上报 | 登记为契约约定(分析侧聚合去重),非缺陷 | 否 |

---

# 4. 与架构 V1.0-rev2 §10.1 的差异登记(候选版间差异,均以 Schema 冻结版/契约统一体系为准)

| # | 差异点 | 候选版 | 架构 §10.1 | 判定 |
|---|---|---|---|---|
| D-1 | submissions 重复提交状态码 | 409 DUPLICATE_SUBMISSION | 422 重复提交 | 契约统一错误码体系语义更准;架构为候选版未冻结 → 候选版成立,登记留痕 |
| D-2 | 详情页路由与响应标识 | `/api/websites/{normalized_domain}`,website 标识含 normalized_domain | `:domain` | normalized_domain 实体键贯穿优先(§B 审计);架构为候选版 → 成立,登记留痕 |
| D-3 | 详情 relations 响应 | 仅 similar/alternative,无 related + p0_gate 声明 | `relations: { similar[], alternative[], related[] }` | 契约按 Schema §2.9 P1 门控收紧(related 禁止前端展示)→ 成立,登记留痕 |
| D-4 | 端点细化 | /api/categories、/api/tags 两级别 + relations 端点 + Admin overview/seo/sync 组 | 仅 categories/:slug | 契约细化,产品 P0 能力要求,登记留痕 |

---

# 5. 结论(任务书六,10 项判定)

| # | 判定项 | 结论 |
|---|---|---|
| 1 | 是否 PASS | **PASS WITH 5 FIXES**(0 阻断;5 项 P1;7 项 P2) |
| 2 | 是否可以冻结 | **建议修复 5 项 P1 后冻结**(同 Schema V1.0"PASS WITH 4 FIXES → 修复 → 冻结"流程);若立即冻结,5 项 P1 须登记为冻结后首批变更提案 |
| 3 | 是否存在 Schema Change Request | **否**——全部问题可在 API 契约文档层修复,无一处触碰冻结 Schema |
| 4 | 是否存在 P0/P1 边界冲突 | 无门控冲突;唯一边界相关项为 P1-5(关系写通道缺失,属 P0 能力缺口,非边界违反) |
| 5 | 是否存在 SiteIntel 耦合 | 候选版 §8.3 history 行违反冻结白名单(P1-3);修复后无耦合(仅 v1/report + Key 单向消费,无直连/无共享 schema/无 rawData 持久化) |
| 6 | 是否存在 recommendations 污染 | **否**(唯一写通道 = §5.8;搜索/任务页/实时计算均不落库) |
| 7 | 是否存在服务端收藏 | **否**(无表/无端点/白名单无收藏事件/visitor_id 不参与) |
| 8 | 是否存在 normalized_domain 语义回退 | **否**(全链路 10 环一致;domain 未承担唯一性/去重/关联/幂等) |
| 9 | sync_queue 是否存在 off-by-one | **否**(attempts 先 +1 再判断;>= max_attempts → failed;默认 5;第 5 次失败即 failed;超时计入;§9.4 两步骤与 Schema 逐字一致) |
| 10 | 是否允许进入 API 冻结 | **允许——修复 5 项 P1 后冻结**(P2 项登记不阻塞) |

---

# 6. 冻结建议

1. **推荐路径(与 Schema V1.0 冻结流程一致)**:按"最小修复原则"对候选版执行 5 项 P1 修正(全部为文档字段级修正,无架构重构/无新功能/无 Schema 变更)→ 重跑内部一致性检查 → 正式冻结为《API契约设计-V1.0-冻结版》+ 冻结验收记录(含本审计报告归档)。
2. 7 项 P2 登记为冻结版已知项,不阻塞;其中 P2-9(task_related_tags 意图扩展)与 P2-10(alias 替换语义)建议在冻结时一并修正(成本为零,均一行级修改)。
3. 冻结时同步:P0 相关差异登记(D-1~D-4)写入冻结版 §0;PROJECT_STATUS.md 阶段推进为"API 契约 V1.0 冻结";记忆文件更新。
