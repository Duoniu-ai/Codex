# WEBSITES_D6_PUBLIC_API_COMPLETION_REPORT

> 项目:**Websites**(网址发现平台/网址导航站;独立产品,SiteIntel 仅为上游数据源)
> 报告日期:2026-08-18(+0800)
> Git 基线:D5 Commit = `b1b92e763a038cb51976bb518a0540238333ae21`
> D6 实施范围:`src/app/api/**`(Public 路由)、`src/lib/search/**`、`src/lib/ranking/**`、`src/lib/reasons/**`(冻结任务拆解 D6)
> 数据库:nav_disc @ 154(本轮**零结构变更、零生产写库**,仅只读验证)

---

## 1. D6 Summary

**D6 = PASS(30/30 验收矩阵;Search Top-1 Coverage = 100%)**

- 实现 Public 11 端点(6.1-6.12)+ Search / Ranking / Reasons 模块
- D6 测试 **76 项**全绿;全量 `npm test` **235/235 PASS**;`tsc --noEmit` PASS;`eslint` PASS;`git diff --check` PASS
- 复用 D1 基础(errors / pagination / sliding-window),复用 D4 normalize(服务端折叠);未复制 D4/D5 实现
- 真实数据验证:90 别名 × 4 变体 = 360 条种子查询,**Top-1 命中 360/360(100% ≥ 85%)**;读端点冒烟全 200

---

## 2. 11 Endpoint Matrix(API 契约 §3.1/§4)

| # | Endpoint | Method | Read/Write | Auth | Rate Limit | Result |
|---|---|---|---|---|---|---|
| 01 | 6.1 GET /api/search | GET | 写 search_logs | 匿名 | 60/min/IP | PASS |
| 02 | 6.2 GET /api/tasks/{slug} | GET | 只读 | 匿名 | 60/min/IP | PASS |
| 03 | 6.3 GET /api/websites/{normalized_domain} | GET | 只读 | 匿名 | 60/min/IP | PASS |
| 04 | 6.4 GET .../relations | GET | 只读 | 匿名 | 60/min/IP | PASS |
| 05 | 6.5 GET /api/categories | GET | 只读 | 匿名 | 60/min/IP | PASS |
| 06 | 6.6 GET /api/categories/{slug} | GET | 只读 | 匿名 | 60/min/IP | PASS |
| 07 | 6.7 GET /api/tags | GET | 只读 | 匿名 | 60/min/IP | PASS |
| 08 | 6.8 GET /api/tags/{slug} | GET | 只读 | 匿名 | 60/min/IP | PASS |
| 09 | 6.9 POST /api/submissions | POST | 写 submissions | 匿名 | 3/min/IP | PASS |
| 10 | 6.10 POST /api/reports | POST | 写 reports | 匿名 | 10/min/IP | PASS |
| 11 | 6.11 POST /api/events | POST | 写 user_events | 匿名 + vid cookie | 60/min/IP | PASS |

> 6.12 集成验收 = 本报告 30 项矩阵 + 真实数据冒烟(§10)。

---

## 3. Search

- `src/lib/search/query.ts`:归一化(小写/全角转半角/停用词过滤,保留 免费/AI/在线/中文/无需注册)/ 连续 CJK+ASCII token 化
- `src/lib/search/task-matcher.ts`:§7.2 全等 > 前缀/包含 > 多 Token 联合;同分取候选数多者;Top-1 { task_id, confidence }
- `src/lib/search/constraint-mapper.ts`:§7.3 settings.constraint.dictionary 子串映射
- `src/lib/search/search.ts`:候选(website_tasks / ILIKE 兜底)→ 约束过滤 → 五因子排序 → 分页 → search_logs

---

## 4. Ranking

- `src/lib/ranking/weights.ts`:权重读 settings.ranking.weights(实测 `{task_match:0.35, evidence:0.25, quality:0.2, curated:0.1, recency:0.1}`),缺失权重按 0,全 0 安全失败
- `src/lib/ranking/five-factor.ts`:五因子 0-100 加权;确定性排序,平局按 websiteId 升序;无随机/无隐式 DB 顺序

---

## 5. Reasons

- `src/lib/reasons/reasons.ts`:模板生成(§8.4);每条 reason 可回溯(task_match→website_tasks / evidence→match_basis / quality→data_quality_score≥80 / curated→curated_picks / recency→website_status)
- 无依据不输出(no hallucination);quality<80 不出质量理由

---

## 6. Rate Limit

- 复用 `sliding-window.ts`:PUBLIC_READ_60(共享 60/min/IP)/ SUBMISSIONS_3 / REPORTS_10 / EVENTS_60
- 测试:正常 / 第 61 次 429(真实用例)/ 窗口语义沿用 D1 既有测试

---

## 7. Pagination

- 复用 `pagination.ts`:page≥1、page_size 1-100 默认 20;非法 → 400 VALIDATION_ERROR
- 测试:首页 / 越界空 items / total 正确

---

## 8. Related P1 Gate

- `relations/route.ts`:数据中存在 related 行 → **422 RELATION_TYPE_FORBIDDEN** + 告警日志;响应无 related 字段,`p0_gate: "related=disabled"`
- 测试:纯函数门控 + 路由 200/422

---

## 9. Recommendations No-Write

- Search 与 Task 页**不写 recommendations / recommendation_evidence**(§6 硬规则);PublicDb 结构上无推荐写方法
- 测试:断言仅 search_logs 写入,无任何 recommendations 副作用

---

## 10. Search Top-1 Coverage(真实数据,只读)

```text
Seed Queries: 360(90 别名 × 4 变体:全等 / 免费前缀 / 最好用包装 / 网站后缀)
Top-1 Hits: 360
Top-1 Misses: 0
Coverage: 100.00%
Result: PASS(≥85%)
```

- 数据来源:nav_disc 真实 90 条 task_aliases × 30 tasks × website_tasks 候选数(只读查询,未修改任何数据)
- 端点冒烟(真实 DB,只读):GET /api/tasks/ppt-maker → 200(5 站点,理由可追溯);GET /api/categories → 200(树);GET /api/tags → 200(14 标签);GET /api/websites/ai-image-gen-01.example.com → 200(六层响应)

---

## 11. Tests

- D6 新增 76 项:search 基础 17 / search 编排 7 / ranking 7 / reasons 4 / 路由+纯函数 32 / 其他 9
- 全量 `npm test` **235/235 PASS**(D1-D5 保持全绿)

---

## 12. Security

- 所有 API 错误响应经 `handleError`:未知错误只返回通用 500,不泄露 SQL/stack/DB 信息/Secret
- 无新密钥/凭证/回调;vid cookie HttpOnly + SameSite=Lax;客户端伪造 visitor_id 被忽略
- `NAVIGATION_SYNC_KEY` 未进入任何 D6 代码/日志/测试

---

## 13. Database

```text
Schema Change = NO
Migration = NO
Data Migration = NO
```

- 仅运行时写 submissions / reports / user_events / search_logs(按契约);验证阶段未写入生产库

---

## 14. Scope

```text
D6 = PASS
D7+ = NOT EXECUTED
D10 = NOT EXECUTED

D4 = UNCHANGED
D5 = UNCHANGED
D4 Production E2E = BLOCKED(HTTP 401,单独待处理)
```

---

## 15. 30 CHECK Matrix

| # | D6 Check | Result | Evidence |
|---|---|---|---|
| 01 | API foundation | PASS | `_lib/http.ts` / `_lib/data.ts` 冻结错误体 + 惰性 Prisma |
| 02 | endpoint contract | PASS | 11 端点矩阵(§2),路径/方法/响应字段按 §4 |
| 03 | request validation | PASS | q 必填/超长、page/page_size、tag_type、body JSON 校验测试 |
| 04 | error contract | PASS | VALIDATION_ERROR/NOT_FOUND/RATE_LIMITED/RELATION_TYPE_FORBIDDEN/EVENT_TYPE_INVALID 测试 |
| 05 | rate limit | PASS | 第 61 次 search → 429;submissions/reports/events 桶复用 |
| 06 | pagination | PASS | 复用 parsePagination;越界空 items 测试 |
| 07 | search normalization | PASS | query.test 归一化/停用词/token 化 |
| 08 | candidate selection | PASS | search.test 命中 Task/ILIKE 兜底/约束过滤 |
| 09 | deterministic ranking | PASS | five-factor 同输入同顺序 + 平局稳定 |
| 10 | ranking weights | PASS | settings 权重解析 + 缺失安全失败 |
| 11 | reasons | PASS | reasons.test 四场景 |
| 12 | reasons traceability | PASS | 理由与 task/match_basis/quality/curated/status 数据绑定 |
| 13 | 6.1 endpoint | PASS | search route 200/400/429 |
| 14 | 6.2 endpoint | PASS | tasks route 200/404 阈值 |
| 15 | 6.3 endpoint | PASS | websites route 六层响应 200/404 |
| 16 | 6.4 endpoint | PASS | relations route 200 + p0_gate |
| 17 | 6.5 endpoint | PASS | categories route 树 200 |
| 18 | 6.6 endpoint | PASS | categories/{slug} 200/404 |
| 19 | 6.7 endpoint | PASS | tags route 200/400(tag_type/group) |
| 20 | 6.8 endpoint | PASS | tags/{slug} 200/404 |
| 21 | 6.9 endpoint | PASS | submissions 201/409/400 |
| 22 | 6.10 endpoint | PASS | reports 201/422/404 |
| 23 | 6.11/6.12 endpoint | PASS | events 201+vid cookie/400;集成冒烟 |
| 24 | related P1 gate = 422 | PASS | related 行 → 422 + 告警(纯函数+路由测试) |
| 25 | recommendations no-write | PASS | 仅 search_logs 写入断言;结构无推荐写方法 |
| 26 | API security | PASS | 未知错误不泄露;无 Secret 输出 |
| 27 | search Top-1 ≥85% | PASS | 真实数据 360/360 = 100%(§10) |
| 28 | integration flow | PASS | 真实 DB 端点冒烟 4 个全 200 + 全量测试 |
| 29 | scope / diff check | PASS | git 仅 D6 文件;diff --check 0 |
| 30 | final D6 acceptance | PASS | 以上 29 项 + 235/235 + tsc/eslint 全过 |

**结果:30 / 30 PASS(0 FAIL / 0 BLOCKED / 0 NOT_RUN)**

---

## 16. Git

实际修改/新增文件:

```text
src/lib/search/{types,query,task-matcher,constraint-mapper,search}.ts + 4 测试
src/lib/ranking/{weights,five-factor}.ts + 2 测试
src/lib/reasons/reasons.ts + 测试
src/app/api/_lib/{db,data,http,relations,category-tree,items}.ts
src/app/api/{search,tasks/[slug],websites/[normalized_domain],websites/[normalized_domain]/relations,
  categories,categories/[slug],tags,tags/[slug],submissions,reports,events}/route.ts
src/lib/api/d6-{routes,relations,category-tree}.test.ts
WEBSITES_D6_PREFLIGHT_REPORT.md
WEBSITES_D6_PUBLIC_API_COMPLETION_REPORT.md
```

D6 Commit:`feat(websites): implement public api`(提交后 HEAD 见 git log)
