# 网址发现平台 — API 契约设计 V1.0(候选版)

> 依据基线(均为冻结/候选基线,本契约不得与之冲突):
> ①《网址发现平台-完整产品规划-V2.1.2-最终冻结版》(docs/product/);
> ②《网址发现平台-技术架构设计-V1.0-候选版》(V1.0-rev2,docs/architecture/);
> ③《网址发现平台-数据库Schema与数据同步契约-V1.0-冻结版》(docs/architecture/,24 表,下称"Schema 冻结版")。
> 状态:**候选版,等待外部审计复核后再冻结**;未写 schema.prisma / route.ts / Worker / 前端。
> 内部一致性检查见《docs/audit/API契约设计-内部一致性检查报告.md》。

---

# 0. 版本与基线声明

1. 本契约是"接口边界定义",不是业务代码;开发阶段按本契约实现 route.ts 时,不得越过本契约定义的边界。
2. 本契约与架构 V1.0-rev2 §10 的关系:架构 §10 已定义响应/分页/错误基础约定与端点清单,本契约在其上**细化**,不改变架构约定(分页参数 `page_size`、错误体 `{ error, details? }` 均沿用架构,不引入 v2/v3 版本号)。
3. 本契约与 Schema 冻结版的关系:每个接口标注涉及数据表,全部落在 24 表内;**不新增实体**;若发现 Schema 无法支持某需求 → 输出 Schema Change Request(§10),不修改冻结版。
4. SiteIntel 侧:本契约只定义"导航站如何消费 SiteIntel",**不设计、不修改 SiteIntel API**。
5. 数据边界硬规则:任何接口不得返回 SiteIntel 内部数据(evidence.rawData / 原始 DNS 明细 / 完整 IP 明细 / 完整证书链 / SiteIntel 内部字段);只返回导航站自身持久化的摘要。

---

# 1. 通用约定

## 1.1 API 版本策略

- 统一前缀 `/api`;暂不引入 v2/v3 版本段(单版本演进,变更走冻结后流程)。
- 页面路径(SEO 实体页)不在本契约范围,属前后台信息架构阶段。

## 1.2 响应格式(沿用架构 §10 冻结约定,不改变)

- **成功**:按接口定义的 JSON 结构返回;列表接口带分页对象 `{ items, page, page_size, total }`。
- **失败**:HTTP 状态码表达类别 + 错误体:

```json
{ "error": "RATE_LIMITED: 请求过于频繁,请稍后再试", "details": {} }
```

- `error` 为字符串,格式固定为 **`CODE: message`**(错误码大写下划线,冒号+空格分隔;机器可读取 CODE 段)。
- `details` 可选对象,携带字段级信息(如 `{ "field": "q", "reason": "length_exceeded" }`)。

## 1.3 HTTP 状态映射

| 状态码 | 含义 | 典型错误码 |
|---|---|---|
| 200 | 成功(GET/读) | — |
| 201 | 创建成功(POST) | — |
| 400 | 参数/请求体非法 | VALIDATION_ERROR |
| 401 | 未登录(Admin) | UNAUTHORIZED |
| 403 | 已登录但越权(非 ADMIN_EMAILS)/ P0 门控拒绝 | FORBIDDEN |
| 404 | 资源不存在(含未达索引阈值) | NOT_FOUND |
| 409 | 冲突(重复提交等) | DUPLICATE_SUBMISSION / ALREADY_EXISTS |
| 422 | 业务校验失败 | RELATION_TYPE_FORBIDDEN / TAG_TYPE_INVALID / FIELD_NOT_ALLOWED |
| 429 | 限流 | RATE_LIMITED |
| 500 | 服务端错误(含同步失败回执) | INTERNAL_ERROR / SYNC_FAILED |
| 502/503 | 上游 SiteIntel 不可达(仅同步相关) | UPSTREAM_UNAVAILABLE |

## 1.4 限流(全部内存滑动窗口,无独立组件)

| 层 | 限流 | 实现 |
|---|---|---|
| 公开只读(GET) | 60/min/IP | 内存滑动窗口(同 SiteIntel 实现) |
| 提交类(POST submissions/reports) | 3/min/IP(submissions)、10/min/IP(reports) | 同上 |
| POST /api/events | 60/min/IP | 同上 |
| Admin | 不按 IP;登录 session 会话级节流 120/min | 内存计数 |
| 同步侧(SiteIntel 调用) | 30/h 硬顶(上游 v1/analyze),report 按 navigation-sync Key 配额 | 内存令牌桶(架构 §4.2) |

限流超限一律 429 + RATE_LIMITED;限流只拒绝不排队(P0 无队列需求)。

## 1.5 幂等

- GET 天然幂等,不产生副作用(除 search_logs 行为记录)。
- POST 提交类:submissions 以 `normalized_domain` 幂等(UNIQUE 约束 → 409);reports 不做去重(重复纠错允许,管理端合并);events 允许重复(行为统计,不保证精确去重)。
- Admin 写操作:PATCH 幂等(按 id);POST 创建以业务唯一键幂等(slug / alias_normalized / UNIQUE 组合 → 409)。

## 1.6 鉴权

| 角色 | 鉴权方式 | 范围 |
|---|---|---|
| 匿名用户 | 无(公开只读 + 限流) | Public GET / submissions / reports / events |
| 管理员(单角色) | 登录 session + `ADMIN_EMAILS` 白名单交集校验(架构 §9.2);**无 RBAC、无多角色** | 全部 /api/admin/* |
| 导航站同步服务(内部) | 非 HTTP 接口;服务内直接调用(§9) | 无网络端点 |
| SiteIntel 消费 | `X-API-Key: si_<navigation-sync Key>`,仅存服务器 .env | 仅同步服务出站调用 |

## 1.7 visitor_id 规则(冻结)

- 由服务端从 cookie 生成/读取(HttpOnly cookie `vid`);**客户端不得传入 visitor_id**;
- 仅用于匿名、非敏感行为统计;禁止参与收藏;禁止在 P0 形成"匿名用户收藏数据库"。

## 1.8 收藏边界(冻结,非 API)

- P0 收藏 = 纯前端本地数据(localStorage / IndexedDB),**无服务端端点、无数据表**;
- 本契约不存在 favorites 相关端点;user_events 白名单**不含** favorite/unfavorite。

## 1.9 P0/P1 标注

- 每个接口标注 P0(Phase 1 必须)/ P1(Phase 2);P1 接口只定义契约,不进入 P0 开发范围。

---

# 2. 错误码体系(按模块分类,克制不膨胀)

## 2.1 公共错误码(所有接口共用)

| 错误码 | 触发场景 | HTTP |
|---|---|---|
| VALIDATION_ERROR | 参数缺失/格式错/超长/枚举外值 | 400 |
| NOT_FOUND | 资源不存在 / 未达索引阈值视同不存在 | 404 |
| RATE_LIMITED | 任何限流超限 | 429 |
| UNAUTHORIZED | 未登录访问 Admin | 401 |
| FORBIDDEN | 登录但不在 ADMIN_EMAILS 白名单 | 403 |
| INTERNAL_ERROR | 未预期服务端错误 | 500 |
| METHOD_NOT_ALLOWED | 方法不允许 | 405 |

## 2.2 业务错误码(按域)

| 域 | 错误码 | 触发场景 | HTTP |
|---|---|---|---|
| 提交 | DUPLICATE_SUBMISSION | submissions.normalized_domain 已存在且未拒绝 | 409 |
| 提交 | SUBMISSION_CLOSED | 提交入口被后台关闭(settings 开关,P0 默认开) | 403 |
| 纠错 | FIELD_NOT_ALLOWED | reports.field 不在白名单 | 422 |
| 事件 | EVENT_TYPE_INVALID | event_type 不在白名单 | 400 |
| 关系 | RELATION_TYPE_FORBIDDEN | 尝试创建/返回 related(P1 门控) | 422 |
| 标签 | TAG_TYPE_INVALID | tag_type 非四枚举(后台) | 422 |
| 管理 | SLUG_CONFLICT | slug / alias_normalized 冲突 | 409 |
| 管理 | TASK_INDEX_NOT_READY | 任务未达索引阈值 | 404 |
| 同步 | SYNC_FAILED | 同步任务最终失败(Admin 查询时透出) | 200(状态字段)/ 500(触发时) |
| 同步 | UPSTREAM_UNAVAILABLE | SiteIntel 不可达/超时(仅同步侧) | 502 |

## 2.3 错误码纪律

- 优先复用公共码,业务码只在"客户端需要差异化处理"时新增;
- 新增业务码须在本节登记(冻结后走变更流程);
- 错误消息面向开发与排障,不面向用户文案(前端自行映射)。

---

# 3. 接口总览(11 域)

## 3.1 Public API(匿名)

| # | 端点 | P0/P1 | 写库 | 触发 SiteIntel |
|---|---|---|---|---|
| 1 | GET /api/search | P0 | 仅 search_logs | 否 |
| 2 | GET /api/tasks/{slug} | P0 | 否 | 否 |
| 3 | GET /api/websites/{normalized_domain} | P0 | 否 | 否 |
| 4 | GET /api/websites/{normalized_domain}/relations | P0 | 否 | 否 |
| 5 | GET /api/categories | P0 | 否 | 否 |
| 6 | GET /api/categories/{slug} | P0 | 否 | 否 |
| 7 | GET /api/tags | P0 | 否 | 否 |
| 8 | GET /api/tags/{slug} | P0 | 否 | 否 |
| 9 | POST /api/submissions | P0 | 是(submissions) | 否(审核通过后由后台入队) |
| 10 | POST /api/reports | P0 | 是(reports) | 否 |
| 11 | POST /api/events | P0 | 是(user_events) | 否 |

## 3.2 Admin API(登录 + ADMIN_EMAILS,全部 P0)

| 能力域(产品冻结版 §12 九项) | 端点组 | 写库 |
|---|---|---|
| 总览 | GET /api/admin/overview | 否 |
| 网站管理 | GET/POST /api/admin/websites;GET/PATCH /api/admin/websites/{id};POST /api/admin/websites/{id}/sync;POST /api/admin/websites/{id}/relations;PATCH /api/admin/website-relations/{id} | 是 |
| Category 管理 | GET/POST /api/admin/categories;PATCH /api/admin/categories/{id} | 是 |
| Task 管理 | GET/POST /api/admin/tasks;PATCH /api/admin/tasks/{id}(含 task_aliases / task_related_tags) | 是 |
| Tag 管理 | GET/POST /api/admin/tags;PATCH /api/admin/tags/{id}(改名/合并) | 是 |
| 提交审核队列 | GET /api/admin/submissions;POST /api/admin/submissions/{id}(批准/拒绝/重复) | 是 |
| 纠错处理队列 | GET /api/admin/reports;POST /api/admin/reports/{id}(接受/拒绝/解决) | 是 |
| 推荐依据维护 | GET/POST /api/admin/recommendations;PATCH /api/admin/recommendations/{id};POST /api/admin/recommendations/{id}/supersede | 是 |
| 精选管理 | GET/POST /api/admin/curated-picks;PATCH /api/admin/curated-picks/{id} | 是 |
| 设置 | GET/PATCH /api/admin/settings | 是 |
| 同步队列 | GET /api/admin/sync;POST /api/admin/sync/manual;POST /api/admin/sync/{id}/retry;POST /api/admin/sync/{id}/skip;GET /api/admin/sync/logs | 是 |
| SEO 索引状态 | GET /api/admin/seo | 否 |

## 3.3 内部契约(非 HTTP)

| 域 | 对象 | P0 |
|---|---|---|
| SiteIntel 同步边界 | 出站消费 `GET /api/v1/report/{domain}?lang=zh`(navigation-sync Key) | P0 |
| Worker / Sync Queue | enqueue / claim / process / complete / retry / fail / skip / recover | P0 |

---

# 4. Public API 定义

## 4.1 GET /api/search — P0(搜索主链路)

| 定义项 | 内容 |
|---|---|
| 1 Method | GET |
| 2 Path | `/api/search` |
| 3 用途 | Query → Task 识别 + Constraint 提取 → 候选网站 → 可解释排序(产品 §4/§19);实时搜索结果**默认不写 recommendations** |
| 4 鉴权 | 匿名 |
| 5 Rate Limit | 60/min/IP(内存滑动窗口) |
| 6 Request 参数 | `q`(必填,1~200 字符);`task`(可选,指定 Task slug,跳过错配直接取候选集);`tag`(可选,逗号分隔 tag slug,与 query 解析的 constraint 合并过滤);`page`(默认 1);`page_size`(默认 20,≤100,沿用架构 §10 命名) |
| 7 Request Body | 无 |
| 8 Response 成功 | `200 { query, task: { slug, name } \| null, constraints: [ "free", "ai" ], items: [ { website: { normalized_domain, domain, name, slug, description }, score, confidence, reasons: [ { type, text } ], status_signal: { online, checked_at } } ], page, page_size, total }`(结构沿用架构 §10.1,website 标识改为 normalized_domain) |
| 9 Response 错误 | `400 VALIDATION_ERROR`(q 缺失/超长)、`429 RATE_LIMITED`、`500 INTERNAL_ERROR` |
| 10 HTTP Status | 200 / 400 / 429 / 500 |
| 11 Pagination | `page` 从 1 起;`page_size` ≤100;`total` 为过滤后总量 |
| 12 Idempotency | GET 幂等;唯一副作用是 search_logs 行为记录(query/识别结果/点击由前端后续上报) |
| 13 数据来源 | 全部导航站库:task_aliases(内存词典)、tasks、tags(constraint.dictionary 映射)、website_tasks、websites、website_status、settings(权重/阈值) |
| 14 涉及数据表 | task_aliases / tasks / tags / task_related_tags(意图扩展辅助,读) / website_tasks / websites / website_status / search_logs(写) / settings(读) |
| 15 P0/P1 | P0 |
| 16 是否允许写数据库 | 否(排序与推荐理由内存计算;**不写 recommendations / recommendation_evidence**;仅 search_logs 落库) |
| 17 是否可能触发 SiteIntel | 否(离线优先,页面永不实时调 SiteIntel) |
| 18 缓存策略 | 响应不缓存(个性化/实时性);高频热词可应用层 30s 内存缓存(架构 §10.3) |

**行为契约(冻结)**:
- Task 识别:alias 词典包含匹配(全等 > 前缀/包含 > 多 Token 联合),Top-1 输出 `{ task_id, confidence }`;验收 = 种子覆盖范围 Top-1 ≥85%(产品 §19);
- 未命中 Task → ILIKE 兜底(websites.name/title/description + tasks.name);
- 兜底仍空 → 200 空 items + 热门任务入口提示(不展示"伪热门");
- Constraint Mapper:免费→tag(constraint)、AI→tag(feature)等,存 settings(`constraint.dictionary`);未识别 token 不进 constraint;**task_related_tags 辅助意图扩展**:命中 Task 后取其关联标签提示扩展约束/特征匹配(如 Task=制作PPT 关联"免费/AI"标签,搜索"免费做PPT的AI网站"时扩展约束;仅辅助,不替代 website_tasks 主匹配关系,不改变实时搜索不落库原则);
- 排序:五因子加权(权重取 settings,架构 §8),**不得引入黑盒模型**;
- 空 q → 400 VALIDATION_ERROR(不返回热门兜底,首页热门由 curated_picks 承担)。

## 4.2 GET /api/tasks/{slug} — P0(任务实体页数据)

| 定义项 | 内容 |
|---|---|
| 1 Method | GET |
| 2 Path | `/api/tasks/{slug}` |
| 3 用途 | Task 实体页(SEO 优先,产品 §8):候选网站 + 推荐依据 + 状态信号 |
| 4 鉴权 | 匿名 |
| 5 Rate Limit | 60/min/IP |
| 6 Request 参数 | `page` / `page_size`(≤100);无查询语义参数(排序由服务端按冻结五因子,参数页由前端 noindex,不提供排序 API 参数——架构 §8 参数页 noindex) |
| 7 Request Body | 无 |
| 8 Response 成功 | `200 { task: { slug, name, description, aliases_count /* 来自 task_aliases */ }, websites: [ 同 4.1 items 结构 ], page, page_size, total }` |
| 9 Response 错误 | `404 NOT_FOUND`(不存在或**未达索引阈值**——候选 ≥5 且 ≥3 完整推荐依据,产品 §3)、`429 RATE_LIMITED` |
| 10 HTTP Status | 200 / 404 / 429 / 500 |
| 11 Pagination | 同上 |
| 12 Idempotency | GET 幂等,零写 |
| 13 数据来源 | tasks / task_aliases / website_tasks / recommendations(固定推荐集,见 §6) / websites / website_status |
| 14 涉及数据表 | tasks / task_aliases / website_tasks / websites / website_status / recommendations / recommendation_evidence(读) |
| 15 P0/P1 | P0 |
| 16 是否允许写数据库 | 否(页面访问**不得**创建 recommendations 行——架构 §8.6 硬规则) |
| 17 是否可能触发 SiteIntel | 否 |
| 18 缓存策略 | ISR 60s;编辑发布时 revalidate 该 slug(架构 §10.3) |

## 4.3 GET /api/websites/{normalized_domain} — P0(网站详情)

| 定义项 | 内容 |
|---|---|
| 1 Method | GET |
| 2 Path | `/api/websites/{normalized_domain}` |
| 3 用途 | 网站详情页数据:基础信息 + 状态信号 + 近期变化 + 标签/任务/关系 + 推荐理由 + 质量分(**架构 §10.1 原 `:domain`,本契约明确为 normalized_domain —— 实体键贯穿原则**) |
| 4 鉴权 | 匿名 |
| 5 Rate Limit | 60/min/IP |
| 6 Request 参数 | 无(路径参数即查询键) |
| 7 Request Body | 无 |
| 8 Response 成功 | `200 { website: { normalized_domain, domain, name, title, description, summary_text, category, tags: [{slug,name,tag_type}], tasks: [{slug,name}], relations: { similar: [...], alternative: [...] /* related 不返回,见 4.4 */ }, status_signal: { online, http_status, checked_at }, changes: [{change_type, delta, detected_at}], quality_score, reasons: [...] } }`(changes 的 `delta` 为 `[{field, from, to}]`,仅摘要字段集合内,见 Schema §2.13;website_changes 无 severity 字段,响应不得包含) |
| 9 Response 错误 | `404 NOT_FOUND`、`429 RATE_LIMITED` |
| 10 HTTP Status | 200 / 404 / 429 / 500 |
| 11 Pagination | 无(单对象) |
| 12 Idempotency | GET 幂等,零写 |
| 13 数据来源 | 仅导航站自身持久化的摘要:websites / categories / website_tags / tags / website_tasks / website_relations / website_status / website_changes / website_metrics / recommendations |
| 14 涉及数据表 | 上述 10 张(全读) |
| 15 P0/P1 | P0 |
| 16 是否允许写数据库 | 否 |
| 17 是否可能触发 SiteIntel | 否(离线优先) |
| 18 缓存策略 | ISR 60s |

**数据边界(硬规则,泄漏即缺陷)**:
- **禁止**返回:evidence.rawData、原始 DNS 明细、完整 IP 明细、完整证书链、SiteIntel 内部字段(investigationId / 完整 history 全量 / insights 全量 / audit 原样);
- `audit` 为 SiteIntel 规则审计,**不得**解读为"质量评分";详情页可展示的仅:最近检测时间 / 可访问状态 / 近期变化类型(透传 change_type)/ SSL 到期;
- **禁止**一切"活跃度/运营活跃/业务增长/质量更高"表述(产品 §7)。

## 4.4 GET /api/websites/{normalized_domain}/relations — P0(similar / alternative,P1 门控)

| 定义项 | 内容 |
|---|---|
| 1 Method | GET |
| 2 Path | `/api/websites/{normalized_domain}/relations` |
| 3 用途 | 相似 / 替代网站(产品 §5);**related 为 P1 门控:P0 不允许创建、不允许自动生成、不允许前端展示** |
| 4 鉴权 | 匿名 |
| 5 Rate Limit | 60/min/IP |
| 6 Request 参数 | 无 |
| 7 Request Body | 无 |
| 8 Response 成功 | `200 { relations: { similar: [ { normalized_domain, name, slug, basis } ], alternative: [ ... ] }, p0_gate: "related=disabled" }`(**响应中不存在 related 字段**;`p0_gate` 为明确的接口层门控声明) |
| 9 Response 错误 | `404 NOT_FOUND`、`429 RATE_LIMITED`、`422 RELATION_TYPE_FORBIDDEN`(防御性:若数据中存在 related 行,接口拒绝输出并记日志告警——P0 数据层也不应存在 related 行,见下) |
| 10 HTTP Status | 200 / 404 / 422 / 429 / 500 |
| 11 Pagination | 无 |
| 12 Idempotency | GET 幂等,零写 |
| 13 数据来源 | website_relations(仅 `relation_type IN ('similar','alternative')`) |
| 14 涉及数据表 | website_relations / websites |
| 15 P0/P1 | P0(similar/alternative);related 为 P1 门控 |
| 16 是否允许写数据库 | 否 |
| 17 是否可能触发 SiteIntel | 否 |
| 18 缓存策略 | ISR 60s |

**P0 门控执行(三层,接口层必须执行)**:
1. **不允许创建**:Admin API 创建关系时 `relation_type` 白名单校验仅 similar/alternative(§5.2 relations 端点);related 请求 → 422 RELATION_TYPE_FORBIDDEN;用户端无关系写端点;
2. **不允许自动生成**:规则引擎(架构 §8)关系生成逻辑只输出 similar/alternative;related 不实现;
3. **不允许前端展示**:本接口响应不含 related 字段;前端无 related 渲染分支。

## 4.5 GET /api/categories — P0

| 定义项 | 内容 |
|---|---|
| 1 Method | GET |
| 2 Path | `/api/categories` |
| 3 用途 | 分类导航(树) |
| 4 鉴权 | 匿名 |
| 5 Rate Limit | 60/min/IP |
| 6 Request 参数 | 无 |
| 7 Request Body | 无 |
| 8 Response 成功 | `200 { categories: [ { slug, name, description, children: [...] } ] }`(仅 active) |
| 9 Response 错误 | `429 RATE_LIMITED` |
| 10 HTTP Status | 200 / 429 / 500 |
| 11 Pagination | 无(全量树,数据量小) |
| 12 Idempotency | GET 幂等,零写 |
| 13 数据来源 | categories |
| 14 涉及数据表 | categories |
| 15 P0/P1 | P0 |
| 16 是否允许写数据库 | 否 |
| 17 是否可能触发 SiteIntel | 否 |
| 18 缓存策略 | ISR 60s(架构 §10.3) |

## 4.6 GET /api/categories/{slug} — P0

| 定义项 | 内容 |
|---|---|
| 1 Method | GET |
| 2 Path | `/api/categories/{slug}` |
| 3 用途 | 分类页数据(SEO 实体页,产品 §8:Category ≠ Task,不互 canonical) |
| 4 鉴权 | 匿名 |
| 5 Rate Limit | 60/min/IP |
| 6 Request 参数 | `page` / `page_size`(≤100) |
| 7 Request Body | 无 |
| 8 Response 成功 | `200 { category: { slug, name, description, children[] }, websites: [ 同 4.1 items 结构(仅分类内) ], page, page_size, total }` |
| 9 Response 错误 | `404 NOT_FOUND`(不存在或 hidden)、`429 RATE_LIMITED` |
| 10 HTTP Status | 200 / 404 / 429 / 500 |
| 11 Pagination | 标准分页 |
| 12 Idempotency | GET 幂等,零写 |
| 13 数据来源 | categories / websites / website_status |
| 14 涉及数据表 | categories / websites / website_status |
| 15 P0/P1 | P0 |
| 16 是否允许写数据库 | 否 |
| 17 是否可能触发 SiteIntel | 否 |
| 18 缓存策略 | ISR 60s |

## 4.7 GET /api/tags — P0

| 定义项 | 内容 |
|---|---|
| 1 Method | GET |
| 2 Path | `/api/tags` |
| 3 用途 | 标签列表(筛选/发现;**tag_type 严格四枚举** attribute/feature/constraint/scenario,无第五种) |
| 4 鉴权 | 匿名 |
| 5 Rate Limit | 60/min/IP |
| 6 Request 参数 | `tag_type`(可选,过滤四枚举之一,非枚举值 → 400 VALIDATION_ERROR);`group`(**不接受为查询参数**——tags.group 仅后台管理分组,不参与匹配语义) |
| 7 Request Body | 无 |
| 8 Response 成功 | `200 { tags: [ { slug, name, tag_type } ], groups: { /* 仅后台用,公开接口不返回 */ } }`(公开响应只含 slug/name/tag_type/description) |
| 9 Response 错误 | `400 VALIDATION_ERROR`、`429 RATE_LIMITED` |
| 10 HTTP Status | 200 / 400 / 429 / 500 |
| 11 Pagination | 无(数据量小) |
| 12 Idempotency | GET 幂等,零写 |
| 13 数据来源 | tags(仅 active) |
| 14 涉及数据表 | tags |
| 15 P0/P1 | P0 |
| 16 是否允许写数据库 | 否 |
| 17 是否可能触发 SiteIntel | 否 |
| 18 缓存策略 | ISR 60s |

## 4.8 GET /api/tags/{slug} — P0

| 定义项 | 内容 |
|---|---|
| 1 Method | GET |
| 2 Path | `/api/tags/{slug}` |
| 3 用途 | 标签页(网站列表,按标签过滤;tag_type 随响应透出) |
| 4 鉴权 | 匿名 |
| 5 Rate Limit | 60/min/IP |
| 6 Request 参数 | `page` / `page_size`(≤100) |
| 7 Request Body | 无 |
| 8 Response 成功 | `200 { tag: { slug, name, tag_type, description }, websites: [ 同 4.1 items 结构 ], page, page_size, total }` |
| 9 Response 错误 | `404 NOT_FOUND`、`429 RATE_LIMITED` |
| 10 HTTP Status | 200 / 404 / 429 / 500 |
| 11 Pagination | 标准分页 |
| 12 Idempotency | GET 幂等,零写 |
| 13 数据来源 | tags / website_tags / websites / website_status |
| 14 涉及数据表 | tags / website_tags / websites / website_status |
| 15 P0/P1 | P0 |
| 16 是否允许写数据库 | 否 |
| 17 是否可能触发 SiteIntel | 否 |
| 18 缓存策略 | ISR 60s |

## 4.9 POST /api/submissions — P0(用户提交网站)

| 定义项 | 内容 |
|---|---|
| 1 Method | POST |
| 2 Path | `/api/submissions` |
| 3 用途 | 匿名用户提交网站(产品 §12 用户端 P0);**不引入账号体系** |
| 4 鉴权 | 匿名 |
| 5 Rate Limit | 3/min/IP(架构 §10.1 冻结) |
| 6 Request 参数 | 无 |
| 7 Request Body | `{ "domain": "www.figma.com", "note": "可选,≤500 字符" }` |
| 8 Response 成功 | `201 { id, status: "pending" }` |
| 9 Response 错误 | `400 VALIDATION_ERROR`(域名格式非法/note 超长)、`409 DUPLICATE_SUBMISSION`(normalized_domain 已存在且未拒绝)、`403 SUBMISSION_CLOSED`、`429 RATE_LIMITED` |
| 10 HTTP Status | 201 / 400 / 403 / 409 / 429 / 500 |
| 11 Pagination | 无 |
| 12 Idempotency | 以 **normalized_domain** 幂等:服务端先折叠(§5 规则:小写/去协议/去 www/去尾斜杠/去端口/punycode)→ UNIQUE 冲突 → 409;重复点击不产生重复行 |
| 13 数据来源 | 用户输入;normalized_domain 由服务端派生(禁止信任客户端) |
| 14 涉及数据表 | submissions(写) |
| 15 P0/P1 | P0 |
| 16 是否允许写数据库 | 是(submissions 一行) |
| 17 是否可能触发 SiteIntel | **否**(仅审核通过后由 Admin 同步端点入队触发,见 5.x / §8) |
| 18 缓存策略 | 不缓存 |

**normalized_domain 处理(冻结)**:`www.example.com` 与 `example.com` 视为同一实体,二次提交 → 409;同提交重复请求(网络重试)幂等无害。

## 4.10 POST /api/reports — P0(网站数据纠错)

| 定义项 | 内容 |
|---|---|
| 1 Method | POST |
| 2 Path | `/api/reports` |
| 3 用途 | 匿名纠错(产品 §12;与提交审核队列分离,管理端处理) |
| 4 鉴权 | 匿名 |
| 5 Rate Limit | 10/min/IP |
| 6 Request 参数 | 无 |
| 7 Request Body | `{ "website_id": "可选", "normalized_domain": "或提供 domain 由服务端解析", "field": "display_name|description|category_id|homepage_url|online|http_status|technologies|ssl_expires_at|cdn|hosting", "current_value": "可选", "expected_value": "必填", "note": "可选" }`(field 严格为 Schema §2.18 冻结白名单 10 项,无 title;category 语义 = category_id) |
| 8 Response 成功 | `201 { id, status: "pending" }` |
| 9 Response 错误 | `400 VALIDATION_ERROR`(参数缺失)、`422 FIELD_NOT_ALLOWED`(field 不在白名单)、`404 NOT_FOUND`(网站不存在)、`429 RATE_LIMITED` |
| 10 HTTP Status | 201 / 400 / 404 / 422 / 429 / 500 |
| 11 Pagination | 无 |
| 12 Idempotency | 不去重(重复纠错允许,管理端合并处理);同请求重试无害(各自成行) |
| 13 数据来源 | 用户输入 + websites 解析;**匿名**,无 visitor 绑定(行为事件才用 visitor_id) |
| 14 涉及数据表 | reports(写) / websites(读) |
| 15 P0/P1 | P0 |
| 16 是否允许写数据库 | 是(reports 一行) |
| 17 是否可能触发 SiteIntel | 否(管理端"接受"时按 field 回写导航站字段,不回写上游) |
| 18 缓存策略 | 不缓存 |

## 4.11 POST /api/events — P0(匿名行为事件)

| 定义项 | 内容 |
|---|---|
| 1 Method | POST |
| 2 Path | `/api/events` |
| 3 用途 | 匿名、非敏感行为统计(visitor_id 服务端 cookie 管理) |
| 4 鉴权 | 匿名(必须带服务端签发的 `vid` cookie;无 cookie 则由服务端生成并 Set-Cookie) |
| 5 Rate Limit | 60/min/IP |
| 6 Request 参数 | 无 |
| 7 Request Body | `{ "event_type": "visit|reason_expand|submit|search_click", "website_id": "可选", "task_id": "可选", "payload": "可选" }` |
| 8 Response 成功 | `201 { ok: true }` |
| 9 Response 错误 | `400 EVENT_TYPE_INVALID`(白名单外)/ `400 VALIDATION_ERROR`(结构非法)/ `429 RATE_LIMITED` |
| 10 HTTP Status | 201 / 400 / 429 / 500 |
| 11 Pagination | 无 |
| 12 Idempotency | 不保证精确去重(行为统计,重复上报可接受);同一会话聚合由分析侧处理 |
| 13 数据来源 | 客户端上报 + 服务端 visitor_id(cookie);**客户端禁止传入 visitor_id,服务端忽略/拒绝客户端提供的该字段** |
| 14 涉及数据表 | user_events(写) |
| 15 P0/P1 | P0 |
| 16 是否允许写数据库 | 是(user_events 一行) |
| 17 是否可能触发 SiteIntel | 否 |
| 18 缓存策略 | 不缓存 |

**白名单冻结**:`visit / reason_expand / submit / search_click`(架构 §6.1)。**不含 favorite / unfavorite**——收藏为纯前端本地数据,不产生服务端事件。

## 4.12 本地收藏(冻结声明,非 API)

- 无端点、无表、无服务端参与;localStorage / IndexedDB 由前端实现(信息架构阶段定);
- P1 账号体系后提供云端收藏 + 本地迁移(P1 接口届时另立契约)。

---

# 5. Admin API(单角色,登录 session + ADMIN_EMAILS 白名单)

## 5.0 Admin 通用约定(以下端点共用,不再逐条重复)

| 定义项 | 通用值 |
|---|---|
| 4 鉴权 | 登录 session;ADMIN_EMAILS 白名单交集校验;**401 UNAUTHORIZED / 403 FORBIDDEN** |
| 5 Rate Limit | 120/min/会话(内存计数) |
| 8/9 响应 | 成功按接口结构;错误体 `{ error: "CODE: message", details? }` 同 §1.2 |
| 11 Pagination | 列表统一 `page` / `page_size`(≤100)→ `{ items, page, page_size, total }` |
| 12 Idempotency | PATCH 按 id 幂等;POST 创建以业务唯一键幂等(slug / alias_normalized / UNIQUE 组合),冲突 → 409 SLUG_CONFLICT |
| 16 是否允许写数据库 | 全部 Admin 写端点:是;读端点:否 |
| 17 是否可能触发 SiteIntel | 仅 sync 端点:是;其余:否 |
| 18 缓存策略 | 不缓存;写操作触发任务页/分类页 revalidate(按 slug) |

## 5.1 GET /api/admin/overview — 总览

- **用途**:待审提交数 / 待处理纠错数 / 同步队列摘要(pending/retrying/failed 计数)/ 索引状态摘要;
- **响应**:`200 { stats: { submissions_pending, reports_pending, sync: { pending, retrying, failed }, seo: { below_threshold, indexed } } }`;
- **涉及表**:submissions / reports / sync_queue / websites;P0。

## 5.2 网站管理

| 端点 | 用途 | 关键请求/响应 | 涉及表 |
|---|---|---|---|
| GET /api/admin/websites | 列表(搜索/筛选 status,is_indexed) | `?q&status&is_indexed&page&page_size` → items | websites |
| POST /api/admin/websites | 新建(可带 category/tags/tasks 关联) | body: name/domain(原始 Host)/normalized_domain(服务端折叠派生)/category_id/tags[]/tasks[];slug 生成或显式 | websites / categories / website_tags / website_tasks |
| GET /api/admin/websites/{id} | 详情(字段/标签/任务/关系/状态/同步信息/纠错记录) | → 全量字段 + 关联 + sync_queue 最近状态 + reports 相关行 | websites + 关联表 + sync_queue + reports |
| PATCH /api/admin/websites/{id} | 更新字段/状态(active↔archived)/索引开关;触发 revalidate | body: 可改字段白名单(domain 仅展示字段可改,normalized_domain 不可改——实体键) | websites |
| POST /api/admin/websites/{id}/sync | 手动触发同步入队 | → `201 { job: { id, status: "pending" } }`;幂等:已有 pending/processing 同实体任务 → 409 ALREADY_EXISTS | sync_queue(写) |
| POST /api/admin/websites/{id}/relations | 创建网站关系(P0 写通道) | body: `{ website_id_to, relation_type: "similar"\|"alternative", basis, confidence }`;relation_type 白名单校验,**related → 422 RELATION_TYPE_FORBIDDEN(P1 门控)**;source=manual;UNIQUE(website_id_from, website_id_to, relation_type) 冲突 → 409 ALREADY_EXISTS;仅管理员可写,用户端无关系写端点 | website_relations(写) |
| PATCH /api/admin/website-relations/{id} | 关系更新/失效 | body: `{ status: "superseded" }`(软失效不硬删,Schema §10)或 basis/confidence 修正;不允许改 relation_type 为 related | website_relations |

## 5.3 Category 管理

| 端点 | 用途 | 关键请求/响应 | 涉及表 |
|---|---|---|---|
| GET /api/admin/categories | 列表(树) | → items 含 parent_id / sort_order / is_indexed | categories |
| POST /api/admin/categories | 新建 | body: name/slug/parent_id?/sort_order;slug 冲突 → 409 | categories |
| PATCH /api/admin/categories/{id} | 更新(改名/slug/父子/排序/索引);归档=hidden | body: 白名单字段;父子变更需环检测(400 VALIDATION_ERROR) | categories |

## 5.4 Task 管理(含 task_aliases / task_related_tags)

| 端点 | 用途 | 关键请求/响应 | 涉及表 |
|---|---|---|---|
| GET /api/admin/tasks | 列表(候选数/索引状态) | → items 含 candidate_count / is_indexed | tasks + website_tasks 聚合 |
| POST /api/admin/tasks | 新建 | body: name/slug/description/aliases[]/related_tag_ids[];服务端事务写 tasks + task_aliases + task_related_tags;alias_normalized 冲突 → 409 SLUG_CONFLICT;tag_type 校验(related_tags 必须为已存在 tag) | tasks / task_aliases / task_related_tags / tags |
| PATCH /api/admin/tasks/{id} | 更新(名称/说明/状态 draft↔active) | body: 白名单字段 | tasks |
| POST /api/admin/tasks/{id}/aliases | 别名整体替换(事务:旧行置 `hidden` 停用 + 新行插入,**不硬删**,保留治理历史,符合 task_aliases.status 冻结语义"隐藏=词典停用";alias_normalized 冲突 → 409 SLUG_CONFLICT) | body: `{ aliases: [...] }` | task_aliases |
| POST /api/admin/tasks/{id}/related-tags | 关联标签整体替换(UNIQUE(task_id,tag_id) 幂等) | body: `{ tag_ids: [...] }` | task_related_tags |

**语义边界(冻结)**:task_aliases 仅支撑 Task Query 识别与 Top-1 验收;task_related_tags 仅辅助搜索意图识别与 Constraint Mapper,**不替代 website_tasks**(本端点组不产生 website_tasks 行)。

## 5.5 Tag 管理

| 端点 | 用途 | 关键请求/响应 | 涉及表 |
|---|---|---|---|
| GET /api/admin/tags | 列表(tag_type 过滤) | `?tag_type` 非四枚举 → 422 TAG_TYPE_INVALID | tags |
| POST /api/admin/tags | 新建 | body: name/slug/tag_type/description/group?;**tag_type 仅四枚举,新增第五种 → 422 TAG_TYPE_INVALID** | tags |
| PATCH /api/admin/tags/{id} | 改名/合并/归档(hidden) | 合并:body `{ merge_into_id }` → 事务迁移 website_tags/task_related_tags 引用后归档原 tag;改名不影响 slug 时允许 | tags / website_tags / task_related_tags |

**group 语义(冻结)**:`tags.group` 仅为可选管理分组维度(如 price/tech/platform),**绝不参与匹配/过滤语义**;公开 API 不接受 group 参数(§4.7)。

## 5.6 提交审核队列

| 端点 | 用途 | 关键请求/响应 | 涉及表 |
|---|---|---|---|
| GET /api/admin/submissions | 队列 | `?status=pending|approved|rejected|duplicate&page&page_size` | submissions |
| POST /api/admin/submissions/{id} | 审核 | body: `{ action: "approve|reject|duplicate", review_note? }`;approve → 建 website 草稿(折叠为 normalized_domain,已存在 → 409 提示合并);reject/duplicate 仅状态流转 | submissions / websites(approve 时) |

## 5.7 纠错处理队列

| 端点 | 用途 | 关键请求/响应 | 涉及表 |
|---|---|---|---|
| GET /api/admin/reports | 队列 | `?status=pending|accepted|rejected|resolved&page&page_size` | reports |
| POST /api/admin/reports/{id} | 处理 | body: `{ action: "accept|reject|resolve", resolution_note? }`;accept → 按 field 白名单回写 websites 字段并置 resolved | reports / websites(accept 时) |

## 5.8 推荐依据维护(稳定推荐定义层,冻结)

| 端点 | 用途 | 关键请求/响应 | 涉及表 |
|---|---|---|---|
| GET /api/admin/recommendations | 列表 | `?website_id&task_id&context&status` | recommendations + recommendation_evidence |
| POST /api/admin/recommendations | 创建(编辑决策) | body: website_id/task_id?/context(白名单 task_page/search/detail/home)/score/confidence/source=manual/evidence[];事务写 recommendations + recommendation_evidence;同一 (website_id, task_id, context) 已存在 active → 409 | recommendations / recommendation_evidence |
| PATCH /api/admin/recommendations/{id} | 更新(评分/依据) | body: score/confidence/evidence 替换(证据行旧删新插) | recommendations / recommendation_evidence |
| POST /api/admin/recommendations/{id}/supersede | 作废(不硬删) | → 旧行 status=superseded,可带新推荐 id | recommendations |

**冻结边界(与实时搜索隔离)**:本组端点**唯一**允许写入 recommendations 的通道(编辑决策/固定推荐集/人工覆盖);`GET /api/search` 与 `GET /api/tasks/{slug}` 永不写此表(架构 §8.6)。

## 5.9 精选管理

| 端点 | 用途 | 关键请求/响应 | 涉及表 |
|---|---|---|---|
| GET /api/admin/curated-picks | 列表 | `?status=draft|published|archived` | curated_picks |
| POST /api/admin/curated-picks | 新建 | body: website_id/task_id?/reason/sort_order | curated_picks |
| PATCH /api/admin/curated-picks/{id} | 排序/发布/归档 | body: sort_order/status;发布触发 revalidate | curated_picks |

## 5.10 设置

| 端点 | 用途 | 关键请求/响应 | 涉及表 |
|---|---|---|---|
| GET /api/admin/settings | 读全部配置 | → `{ settings: { "ranking.weights": {...}, ... } }` | settings |
| PATCH /api/admin/settings | 更新 | body: key 白名单校验(`ranking.weights / quality.weights / task.index_thresholds / constraint.dictionary / website_changes.retention / search_logs.retention_days / sync_queue.max_attempts`),白名单外 → 400 VALIDATION_ERROR;sync_queue.max_attempts 须 ≥1 且为整数 | settings |

## 5.11 同步队列管理

| 端点 | 用途 | 关键请求/响应 | 涉及表 |
|---|---|---|---|
| GET /api/admin/sync | 队列状态 | `?status&page&page_size` → items 含 attempts/max_attempts/next_retry_at/last_error | sync_queue |
| POST /api/admin/sync/manual | 手动入队(插队 priority=-10) | body: `{ normalized_domain 或 domain, job_type: "sync" }`;UNIQUE 冲突 → 409 | sync_queue |
| POST /api/admin/sync/{id}/retry | 人工重置 failed → pending(attempts 清零) | 仅 failed 可重置;否则 400 VALIDATION_ERROR | sync_queue |
| POST /api/admin/sync/{id}/skip | 人工永久放弃 → skipped | 仅 pending/retrying/failed 可跳过 | sync_queue |
| GET /api/admin/sync/logs | 同步日志 | `?action&status&page&page_size`(保留 30 天) | sync_logs |

## 5.12 SEO 索引状态

| 端点 | 用途 | 关键请求/响应 | 涉及表 |
|---|---|---|---|
| GET /api/admin/seo | 索引状态检查(低于阈值自动降级清单) | → `{ tasks: [ { slug, candidate_count, complete_evidence_count, is_indexed, below: true } ] }` | tasks / website_tasks / recommendations 聚合 |

---

# 6. 稳定推荐层 API 边界(域 5 细化)

1. **定义层**(可读可写,仅 §5.8 写通道):recommendations + recommendation_evidence;内容 = 编辑决策 / 固定推荐集 / 人工覆盖;证据链可审计(each recommendation 可追溯 task/constraint/quality/status/editorial 因子);
2. **执行层**(只读不写):GET /api/search、GET /api/tasks/{slug} 实时计算推荐理由(内存),**默认不落库**;
3. 硬规则:**任何一次用户搜索或任务页访问不得新增 recommendations 行**(违反 = 接口缺陷);
4. 生命周期:只 superseded,不硬删(Admin 端点无 DELETE);
5. P0 审计能力:`GET /api/admin/recommendations` 可回答"为什么推荐这个网站"。

---

# 7. 网站关系 API 边界(域 4 细化)

1. P0 allowlist:`similar / alternative`(创建、自动生成、前端展示三通道全部允许;Admin 创建通道 = §5.2 relations 端点);
2. `related` = **P1 门控**:P0 禁止创建(§5.8/§5.2 无此枚举)、禁止自动生成(规则引擎不产出)、禁止前端展示(§4.4 响应无此字段);
3. 接口层防御:关系创建端点 relation_type 白名单校验;读取端点过滤 relation_type;发现库中异常 related 行 → 422 RELATION_TYPE_FORBIDDEN + 告警日志(数据治理修复,不回退门控)。

---

# 8. SiteIntel 同步接口边界(域 10;非 SiteIntel API 设计,仅消费契约)

## 8.1 消费端点与鉴权

- 出站:仅同步服务调用 `GET https://siteintel.cc/api/v1/report/{domain}?lang=zh`(阶段 1 审计确认覆盖需求 90%+);
- 鉴权:`X-API-Key: si_<navigation-sync 专用 Key>`(SiteIntel 管理端创建,rateLimitPerHour 按需;仅存服务器 .env);
- 404(未分析)→ 入队触发 `POST /api/v1/analyze`(同 Key,受 30/h 硬顶)→ 回填后重拉;
- **禁止**使用无鉴权 `/api/report/{domain}` 公开端点;
- 导航站侧无"同步回调"监听端点(单向拉取)。

## 8.2 normalize 流程(契约强制)

```text
target.domain → 折叠(小写/去协议/去 www./去尾斜杠/去端口/punycode)
  → normalized_domain → 按 normalized_domain 查找已有 website
  → 存在:upsert 原站(不新建) | 不存在:新建
```

**禁止按裸 domain 判断实体存在性**(否则 www.example.com 与 example.com 建成两个 Website)。

## 8.3 字段白名单(允许进入导航站库,以 Schema 冻结版 §11.2 为准)

| SiteIntel 字段 | 导航站落点 |
|---|---|
| target.domain | websites.domain(原始值) |
| —(派生) | websites.normalized_domain / slug |
| overview.title | websites.title(仅创建时回填) |
| overview.summaryText(zh) | websites.summary_text |
| websiteStatus.{online,status,checkedAt} | website_status(online/http_status/checked_at)+ website_snapshots |
| infrastructure.ips[0](主 IP 单值) | website_status(ip/asn/asn_org/country)+ snapshots |
| infrastructure.ssl.validTo/daysRemaining | website_status + snapshots |
| infrastructure.cdn.name / hosting.name | website_status + snapshots |
| technology[].name(名称列表) | website_status.technologies + snapshots |
| history[] | 忽略 | **P0 变化信号 = 本地 summary_hash diff 产生**(Schema §13);上游 events API 上线后再接入(届时按冻结后变更流程扩展 change_type CHECK) | ❌ |

**写入规则**:摘要 upsert;summary_hash 未变 → 零写;变化 → website_changes 事件 + status 更新;snapshots 永久保留(历史事实)。

## 8.4 禁止持久化清单(泄漏即缺陷)

- evidence.rawData(全量原始数据);
- 原始 DNS 记录明细 / 完整 IP 明细列表 / 完整证书链 / redirects;
- overview.categories(仅编辑参考,分类由导航站定义);
- overview.description(编辑主权,不覆盖);
- insights 全量(必要时仅取事件类型);
- audit 原样(SiteIntel 规则审计,非导航站质量分)。

## 8.5 summary_hash 规则(冻结,以 Schema 冻结版 §12.1 为唯一基线,不增不减)

- **参与字段(固定 12 项,Schema 冻结版 §12.1)**:`online / http_status / ip / asn / asn_org / country / technologies(排序后)/ ssl_expires_at(ISO 固定时区)/ cdn / hosting / seo_summary(规范化)/ health_summary(规范化)`;禁止自行增加或删除任何参与字段;
- **排除字段(固定 7 项,Schema 冻结版 §12.1)**:`checked_at / last_sync_at / updated_at / title / description / summary_text / display_name`(时间与内容编辑字段的变化**不产生**变化信号);
- canonicalize:键按字典序排序;嵌套 JSON(technologies/seo_summary/health_summary)内部数组排序后序列化;时间统一 `YYYY-MM-DDTHH:MM:SSZ`;数值归一化 → 确定性 JSON(无空白差异)→ `SHA-256(UTF-8 bytes)` → hex 小写 64 字符;
- 变化判定:`new_hash != old_hash` → 更新 website_status + 追加 website_snapshots + 写 website_changes(逐字段 diff,仅 §12.1 集合内字段);`new_hash == old_hash` → 零写(仅刷新 last_sync_at / last_seen_at)。

## 8.6 同步错误处理

| 场景 | 行为 |
|---|---|
| 404(未分析) | 转 analyze 分支 → 回填后重拉 |
| 429 | 记 last_http_status=429,退避重试(§9) |
| 5xx/网络错 | 退避重试 |
| SiteIntel 不可达 | 本地缓存兜底,UI 显示"数据更新时间:X",不阻塞页面 |
| Key 失效 | 同步日志告警,人工换 Key |

---

# 9. Worker / Sync Queue 内部契约(域 11;非 HTTP,服务内部调用边界)

## 9.1 内部操作(函数级契约,实现阶段按此签名语义实现,不写代码)

| 操作 | 契约(输入 → 输出/副作用) |
|---|---|
| `enqueue(normalized_domain, job_type="sync", priority=0)` | 折叠入队;UNIQUE(normalized_domain, job_type) 冲突 → 已存在则忽略(幂等);写 sync_queue(pending) |
| `claim()` | CAS 单条条件更新(§9.3);成功 → processing + locked_at/locked_by;返回任务 |
| `process(job)` | 调 §8 同步契约;成功 → `complete(job)`;失败 → `retry(job, error)` |
| `complete(job)` | processing → completed;写 sync_logs(success);清理锁字段 |
| `retry(job, error, http_status?)` | `attempts+1`;用更新后值判定:`attempts >= max_attempts` → `fail(job)`;**否则** → retrying + next_retry_at=now()+min(2^attempts×60s,4h);写 sync_logs(retry) |
| `fail(job, error)` | retrying/processing → failed;写 sync_logs(failed);留待人工处置(Admin 5.11) |
| `skip(job)` | → skipped(人工触发,Admin 5.11) |
| `recover()` | 每个 tick 最先执行(§9.4):扫描 locked_at 超期 processing 行 → 按重试边界分派 |

## 9.2 状态机与重试边界(冻结,引用 Schema 冻结版 §9)

```text
pending → processing → completed
              ↘ retrying →(next_retry_at 到期)→ pending
              ↘ failed(attempts >= max_attempts;人工重置 pending / 跳过 skipped)
              ↘ skipped(人工永久放弃)
```

- 每次失败(主动失败或超时恢复):`attempts = attempts + 1` → 用**更新后值**判定;
- `attempts >= max_attempts` → **failed**(默认 max_attempts=5:**第 5 次失败即 failed,不得出现第 6 次**);
- `attempts < max_attempts` → retrying + next_retry_at;
- 超时恢复同样计入 attempts;退避 `min(2^attempts × 60s, 4h)`(attempts 为更新后值);
- settings.sync_queue.max_attempts 可调(≥1)。

## 9.3 CAS 原子领取与锁

- 领取 = 单条条件更新(Compare-And-Set),`status='pending' AND (next_retry_at IS NULL OR next_retry_at <= now())` 的 updateMany,count==1 才成功;禁止"先 SELECT 再 UPDATE";
- 锁字段:`locked_at`(领取时间)/ `locked_by`(worker 标识,单实例=进程 ID);锁持有期 10 分钟;
- 同任务防重:processing 不再满足领取条件。

## 9.4 超时恢复(每 tick 最先执行,两步骤)

```text
步骤 1:status='processing' AND locked_at < now()-10min AND attempts+1 < max_attempts
        → retrying, attempts+1, next_retry_at = now()+retry_interval(attempts+1)
步骤 2:同条件 AND attempts+1 >= max_attempts → failed, attempts+1
```

systemd 重启后 processing 行必然超期 → 按边界分派 → **断点续跑**。

## 9.5 幂等与节流

- 幂等:`UNIQUE(normalized_domain, job_type)` + 摘要 upsert + summary_hash 零写;失败重试无害;
- 节流:内存令牌桶适配上游 30/h(analyze)与 report 配额;令牌耗尽快照常,不阻塞其他功能;
- 单实例:instrumentation 单进程 tick;锁机制防御 tick 重入与重启竞态(未来多实例直接复用,无需架构变更)。

---

# 10. 与 Schema 的对应关系与 Schema Change Request 声明区

1. 本契约全部接口落在 Schema 冻结版 24 表内,未引用任何未定义字段;
2. 本阶段**未发现** Schema 无法支持的 API 需求 → **无 Schema Change Request**;
3. 若后续发现:必须按《Schema 冻结版》文末"冻结后变更规则"输出 SCR(问题 / API 需求 / 受影响实体 / 是否阻塞 P0 / 替代方案),**不得直接修改冻结版**;
4. 接口与表对应已逐条标注(§4/§5);worker 契约仅使用 sync_queue / sync_logs / settings / websites / website_status / website_snapshots / website_changes。

---

# 11. 缓存策略汇总(沿用架构 §10.3)

| 层 | 策略 |
|---|---|
| 页面 | 任务页/分类页 ISR 60s;编辑发布时按 slug revalidate;参数页 noindex 且不缓存 |
| API | /api/tasks/*、/api/categories/*、/api/tags/*、/api/websites/* ISR 60s;搜索不缓存(或热词 30s 内存);提交/纠错/事件不缓存 |
| 数据库 | 常规索引 + 查询优化;无独立缓存层(数据量小) |
| 上游 | 同步摘要落库,页面不实时调 SiteIntel(离线优先) |

---

# 12. 结论

1. **契约覆盖完整**:Public 11 端点 + Admin 12 能力域 + SiteIntel 消费契约 + Worker 内部契约,全部 P0 用户路径与管理路径有接口承载(产品 §12 对照见一致性检查报告);
2. **边界冻结**:related P1 门控(接口层三层执行)、recommendations 双层层隔离、收藏纯本地、visitor_id 服务端生成、normalized_domain 实体键贯穿、SiteIntel 仅经 API 消费——全部在本契约逐条落实;
3. **状态**:候选版,等待外部审计复核后再冻结;不写 schema.prisma / route.ts / Worker / 前端。
