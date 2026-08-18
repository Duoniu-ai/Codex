# GAP-01 专项裁定审计报告 — 首页聚合数据缺少 Public API

> 审计日期:2026-08-17
> 审计范围:GAP-01 真实性核验 + 方案 A/B 对比 + 最终裁定
> 审计基线:产品 V2.1.2 冻结版 / Schema V1.0 冻结版(24 表)/ API 契约 V1.0 冻结版 / 信息架构 V1.0 候选版(14/15 PASS)
> 裁定结论:**方案 B(服务端只读直查冻结 Schema),不新增 API;GAP-01 可关闭**
> 本报告不修改任何冻结版文档。

---

## 1. GAP 描述

首页 `/` 需要展示"编辑精选 / 今日推荐"与"可 SEO 索引任务列表"等站点自有聚合数据,但 API 契约 V1.0 冻结版的 Public 11 端点中不存在对应端点(curated_picks 无 Public 读取端点、tasks 无列表端点),信息架构检查 14/15 PASS 中唯一未闭环项。

## 2. 当前冻结基线

| # | 基线 | 状态 | 相关条款 |
|---|---|---|---|
| 1 | 产品规划 V2.1.2 | 已冻结 | §12 用户端 P0 含"首页"与"编辑精选 / 今日推荐" |
| 2 | Schema V1.0(24 表) | 已冻结 | §2.16 curated_picks、§2.5 tasks、§10 生命周期 |
| 3 | API 契约 V1.0 | 已冻结 | §3.1 Public 11 端点、§4.1 空 q 行为契约、§5.8 recommendations context 白名单(含 home)、文末冻结后变更规则(7 步) |
| 4 | 技术架构 V1.0-rev2 | 候选版 | §6.1 curated_picks 小节、§10.1 Public API、§10.3 缓存 |
| 5 | 信息架构 V1.0 | 候选版 | §4.A 首页模块、§13 GAP-01 登记、检查报告 14/15 PASS |

## 3. 首页数据需求(产品 §12 P0 用户端)

| 首页模块 | 数据需求 | 展示语义 |
|---|---|---|
| 搜索框(首屏) | 无数据(交互入口) | — |
| 编辑精选 / 今日推荐 | 已发布精选条目(站点名 / 编辑理由 / 可选任务挂载) | 站点自有内容展示 |
| 任务入口 | 可 SEO 索引任务列表(任务名 / slug / 候选数) | 引导式发现入口 |
| 分类导航 | 分类树(active) | 已有 API 承载 |

## 4. Schema 来源(冻结字段,全部只读)

| 首页数据 | Schema 实体 | 冻结字段 | 冻结索引(展示位设计) |
|---|---|---|---|
| 编辑精选 / 今日推荐 | curated_picks(§2.16) | website_id / task_id? / reason / sort_order / status(`published`)/ published_at | `INDEX(status, sort_order)`(首页/任务页展示位查询) |
| 任务入口 | tasks(§2.5) | name / slug / description / status(`active`)/ seo_indexable / candidate_count | `INDEX(status, seo_indexable)`(阈值复核扫描) |
| 分类导航 | categories(§2.2) | slug / name / children | 经 API |

**要点**:Schema 冻结版两处索引注释即为"展示位查询"与"阈值扫描"设计——首页聚合本就是 Schema 设计时预期的**内部查询场景**,并非预留第三方接口的语义。

## 5. API 现状(逐条核实)

| 可候选端点 | 结论 |
|---|---|
| GET /api/search(§4.1) | `q` 必填(1~200 字符);空 q → 400;**行为契约原文:"不返回热门兜底,首页热门由 curated_picks 承担"** → 契约明确预期首页展示 curated_picks,却无承载端点(契约内部自引用缺口,铁证) |
| GET /api/tasks/{slug}(§4.2) | 仅单任务实体页;无任务列表端点 |
| GET /api/categories(§4.5) | 分类树,可承载首页分类导航(已有) |
| GET /api/tags / tags/{slug}(§4.7/4.8) | 标签语义,非首页聚合 |
| GET /api/websites/{normalized_domain}(§4.3) | 详情实体,非聚合 |
| Admin curated-picks(§5.9) | 管理写通道,非 Public 读通道 |
| recommendations context 白名单含 `home`(§5.8) | 固定推荐集定义层支持 home 上下文,但该上下文无公开读取端点 |

**结论:GAP-01 真实存在。** 三重证据:(1) §3.1 Public 11 端点无聚合端点;(2) §4.1 空 q 行为契约自引用 curated_picks 但无端点;(3) 信息架构检查 14/15 PASS 唯一未闭环项。

## 6. 方案 A — 新增 GET /api/home 首页聚合接口

- **内容**:新增 Public 端点,返回 `{ curated_picks: [...], tasks: [...], categories: [...] }` 聚合;18 项定义(限流/错误码/缓存/分页/幂等……);ISR 60s + 编辑发布 revalidate。
- **前提**:新增 API 路径属 API 契约文末 15 类禁止修改范围 → 必须走**冻结后变更流程 7 步**(API Change Request → 影响分析 → SCR 判定 → 新版本号 V1.1 → 内部一致性检查 → 外部/专项审计 → 新冻结版)。
- **优点**:Public API 面完整;页面数据一律经 API,架构 §10 统一;未来第三方消费可能。
- **缺点**:
  1. V1.0 冻结当日即触发改版,全量审计流程成本高;
  2. 聚合端点开创先例,P1 页面(/tasks、/topic)将连锁要求列表端点,契约面持续膨胀,与 §2.3"错误码纪律·克制不膨胀"同源哲学相悖;
  3. 首页聚合是"站点自有内容展示"(非实体、非交互),与 Public 11 端点全部"实体/交互"语义的契约定位不一致;
  4. 新增端点本身需要 18 项定义,契约篇幅膨胀,且多一跳 HTTP 层,无性能收益(缓存策略与方案 B 相同)。

## 7. 方案 B — 首页服务端组件只读直查冻结 Schema,不新增 Public API

- **内容**:首页为 Next.js 服务端组件(RSC,技术栈冻结 §11),服务端直接以 Prisma 只读查询冻结库:curated_picks(`status='published'` 按 sort_order)+ tasks(`status='active' AND seo_indexable=true`)+ 分类树走现有 GET /api/categories;首页 ISR 60s,编辑发布精选/任务时 revalidate(与 API 契约 §11 页面层缓存策略一致)。
- **与冻结基线的关系(逐条对照 API 契约文末 15 类禁止)**:

| 禁止类别 | 方案 B 是否触碰 |
|---|---|
| API 路径 / 请求字段 / 响应字段 / 错误码 / HTTP 状态码 / 鉴权 / 限流 | 否(零端点变更) |
| normalized_domain 规则 | 否 |
| P0/P1 功能边界 | 否(首页精选与任务入口本就是产品 §12 P0 功能,仅取数方式不同) |
| related 门控 / recommendations 落库边界 / 收藏本地化 / SiteIntel 消费边界 / summary_hash / sync_queue | 否 |

- **法律地位**:API 契约冻结的是**对外 HTTP 接口面**;服务端组件内部只读取数是应用内部实现(Next.js + Prisma 同栈直查,与 SiteIntel 模式一致),不在契约面内;Schema 冻结版§2.16/§2.5 索引注释明示"首页/任务页展示位查询"为预期内部场景。
- **风险与护栏**(必须写死):
  1. 直查范围锁定:仅首页 2 个查询(curated_picks published + tasks active&seo_indexable),只读,零写操作;
  2. 防蔓延:未来任何新聚合/列表需求 → 必须走 API Change Request(不得继续扩大直查面);
  3. 缓存纳入既有策略:ISR 60s + 编辑发布 revalidate 首页。

## 8. 对比

| 维度 | 方案 A(GET /api/home) | 方案 B(服务端直查) |
|---|---|---|
| 冻结基线触碰 | 必须走 7 步变更流程(API V1.1) | 零触碰(15 类禁止零涉及) |
| 流程成本 | 高(影响分析 + 内部检查重跑 + 外部审计 + 新冻结版) | 无 |
| 契约哲学 | 聚合端点与"实体/交互"契约定位不符,开膨胀先例 | 契约面保持纯净 |
| 数据语义 | curated_picks 是站点自有展示数据,非第三方消费面 | 内部取数天然契合展示语义 |
| 缓存 | ISR 60s + revalidate | 相同(ISR 60s + revalidate) |
| Schema 适配 | 无需改表 | 无需改表(冻结索引即为直查设计) |
| 未来扩展 | 第三方可直接消费精选列表 | 第三方需要时再按流程新增端点(YAGNI) |
| 先例风险 | P1 页面连锁要求列表端点 | 直查面锁定,蔓延受护栏约束 |

## 9. 推荐方案

**推荐方案 B(服务端组件只读直查冻结 Schema,不新增 Public API)。**

理由(优先级从高到低):
1. **零冻结基线触碰**:不触发 API Change Request、不触发 Schema Change Request、不产生 V1.1;
2. **与冻结设计意图一致**:Schema §2.16/§2.5 索引注释即为首页展示位查询预留;API 契约 §4.1"首页热门由 curated_picks 承担"在方案 B 下完全成立;
3. **语义匹配**:首页聚合 = 站点自有内容展示,非实体/交互契约语义,不值得为单一页面开放契约面;
4. **成本最低**:2 个只读查询 + 既有缓存策略,无 18 项端点定义、无审计流程;
5. **护栏可控**:直查范围锁定 + 未来聚合需求强制走 API Change Request,防蔓延。

## 10. 是否触发 API Change Request

**否。** 方案 B 零新增/零修改 API;首页数据取数为服务端内部只读实现,不涉及 API 契约文末 15 类禁止清单任何一项。未来若出现第二个聚合需求,再按 7 步流程启动 API Change Request(届时建议一次性评估聚合端点族)。

## 11. 是否触发 Schema Change Request

**否。** 直查全部落在冻结 24 表既有字段(curated_picks.status/sort_order、tasks.seo_indexable/candidate_count),无新增表/字段/枚举/索引需求;不产生 SCR。

## 12. 对 P0/P1 边界影响

**无影响。**
- 首页"编辑精选 / 今日推荐 + 任务入口"按产品 §12 P0 用户端原样实现,功能边界未变,仅取数方式由(不存在的)API 改为服务端直查;
- 未引入任何 P1 功能(账号/云端收藏/related/对比/专题列表均未触碰);
- 信息架构页面矩阵 P0/P1 标记不变,仅首页"数据来源"列由"GAP-01"改为"服务端只读直查(Schema 冻结)"。

## 13. recommendations 数据边界检查

- 首页"今日推荐 / 编辑精选"数据源 = **curated_picks 表(published)**,独立于 recommendations(稳定推荐定义层);首页**不读取、不写入** recommendations / recommendation_evidence;
- recommendations context 白名单含 `home` 属于定义层允许范围(Admin §5.8 写通道可挂 home 上下文的固定推荐),方案 B 下 home 上下文推荐的读取 = 服务端直查 curated_picks + 关联(如有),不触发任何实时计算落库;
- 双层隔离(定义层写入 vs 实时搜索执行层不落库)不破坏:首页零 recommendations 写入,零搜索执行层计算;
- **结论:recommendations 边界零污染。**

## 14. 最终裁定

| 裁定项 | 结论 |
|---|---|
| GAP-01 是否真实存在 | **是**(三重证据,见 §5) |
| 推荐方案 | **方案 B:首页服务端组件只读直查冻结 Schema(2 个查询),不新增 Public API** |
| 是否需要 API Change Request | **否**(方案 B 零触碰 API 契约) |
| 是否需要 Schema Change Request | **否**(零触碰冻结 24 表) |
| 对 P0/P1 边界影响 | 无 |
| recommendations 边界 | 零污染(curated_picks 独立于 recommendations,零写入) |
| GAP-01 是否可关闭 | **可以关闭**(按方案 B 处置;需执行 §15 文档动作后正式关闭) |
| 本报告是否修改冻结版 | 否(仅审计落档) |

## 15. 关闭条件(后续文档动作,非本次执行)

1. 信息架构候选版 §4.A 首页模块"数据来源"列:标记服务端只读直查(curated_picks + tasks,冻结 Schema);
2. 信息架构候选版 §9.1 Page→API Mapping 首页行:数据来源改为"服务端只读直查(冻结 Schema)+ GET /api/categories",并注明直查护栏(只读/范围锁定/未来聚合需求走 API Change Request);
3. 信息架构候选版 §13 GAP-01 处置记录:填"已裁定(2026-08-17):方案 B,报告 docs/audit/GAP-01-首页聚合数据专项审计报告.md";
4. 内部一致性检查报告:结论更新为 15/15 PASS(GAP-01 已处置);
5. PROJECT_STATUS.md:待裁定行移除,记录 GAP-01 已裁定;
6. 以上全部为候选版/状态文档修改,**零触碰三个冻结版**。
