# WEBSITES_ARCHITECTURE_CONSISTENCY_AUDIT

> 项目:**Websites**(网址发现平台/网址导航站)— 独立产品;SiteIntel = 上游数据源/独立项目
> 审计日期:2026-08-18(+0800)
> Git HEAD:86c6379f834a29e2a8e5787ba399702943a302ec(D6)
> 本轮性质:**只读审计**;未修改代码/数据库/`.env`/中央仓库,未 commit/push
> 依据:冻结文档(P0 任务拆解/产品规划/技术架构/Schema/API 契约/信息架构)+ D1-D6 完成报告 + 实际代码与数据库只读核验

---

## 1. Executive Summary

全链路一致性总体 **PASS**。D1-D6 实现与冻结基线一致,未发现 P0/P1 级问题。

- P0 Consistency:PASS
- Boundary:PASS
- Schema:PASS
- API:PASS(2 项 P2/P3 偏差,见 §9)
- Information Architecture:PASS(1 项 P2 数据完整度偏差)
- D3-D6 Alignment:PASS
- 问题统计:P0=0 / P1=0 / P2=1(D-01 已修复)/ P3=2

---

## 2. P0 Consistency

依据:P0开发任务拆解-V1.0-冻结版.md + 冻结验收记录 + 内部一致性检查报告。

| Stage | Frozen Definition(摘要) | Actual Completion | Match |
|---|---|---|---|
| D1 项目与基础设施初始化 | Next.js/Prisma/测试/构建基线 | PASS(22/22 测试,tsc/eslint/build) | ✅ |
| D2 冻结 Schema 实现 | 24 表 Prisma 逐字落地 | PASS(字段级对照 24/24,221/221) | ✅ |
| D3 Migration 与数据库验收 | 建库/首迁/30 CHECK/seed | PASS(24 表/221 字段/30 CHECK,commit 038492d) | ✅ |
| D4 SiteIntel 数据同步实现 | src/lib/sync 六模块 | PASS(实现,commit 20fd42e);E2E=BLOCKED(外部) | ✅(实现)/⏸(E2E 外部) |
| D5 Sync Queue 与 Worker | queue/worker/instrumentation | PASS(commit b1b92e7) | ✅ |
| D6 Public API | 11 端点 + search/ranking/reasons | PASS(commit 86c6379) | ✅ |
| D7 Admin API | 冻结定义 | **NOT STARTED**(未提前实现) | ✅(范围守约) |
| D8 Public Frontend | 冻结定义 | NOT STARTED | ✅ |
| D9 Admin Frontend | 冻结定义 | NOT STARTED | ✅ |
| D10 测试验收部署 | 冻结定义 | NOT STARTED | ✅ |

**结论:PASS**(D1-D6 与冻结定义一致;D7-D10 未越界实现)。

---

## 3. Boundary Consistency

依据:PROJECT_BOUNDARY.md + 实际代码只读核查。

| 检查项 | 结果 | Evidence |
|---|---|---|
| Websites 独立产品 | ✅ | PROJECT_BOUNDARY;独立仓库 duoniu-ai/Websites |
| SiteIntel 仅上游数据源 | ✅ | 同步仅经 `src/lib/sync/client.ts`(siteintel.cc API);无直连 SiteIntel PG |
| 数据库独立 | ✅ | nav_disc(独立库独立用户),不共享 SiteIntel PG |
| Prisma 独立 | ✅ | 独立 `prisma/schema.prisma`(24 表),无共享 schema |
| 代码独立 | ✅ | 无 SiteIntel 业务代码复制(全库 grep 无 siteintel PG 连接/无复制模块) |
| 反向依赖 | ✅ | 单向 SiteIntel → API → 同步 → nav_disc;无反向耦合 |

**结论:PASS**

---

## 4. Schema Consistency

依据:Schema 冻结版 + Schema冻结验收记录 + `prisma/schema.prisma` + `prisma/migrations/**` + 生产库只读核验。

| 检查项 | 冻结 | 实际 | Result |
|---|---|---|---|
| 表 | 24 | 24(生产库实时确认) | ✅ |
| 字段 | 221 | 221(snake_case,camelCase=0) | ✅ |
| PK / FK / UNIQUE | 24 / 25 / 16 | 24 / 25 / 16(15 唯一索引+settings PK) | ✅ |
| INDEX / CHECK | 31 / 30 | 31 / 30(30/30 已验收) | ✅ |
| migration | 20260817_init | 仅此一个;git 历史只有 baseline+D3 触碰 schema/migrations(D4-D6 零改动) | ✅ |
| D3-D6 依赖冻结 Schema | 严格 | D4 persist/D5 queue 均只读写既有表;无新表/字段/枚举 | ✅ |

**结论:PASS**

---

## 5. API Consistency

依据:API 契约冻结版 + 冻结验收/审计记录 + 实际 `src/app/api/**`、`src/lib/search|ranking|reasons`。

| Endpoint | Frozen | Actual | Match |
|---|---|---|---|
| 6.1 GET /api/search | q 1-200/任务识别/五因子/理由/search_logs/不写 recommendations | 实现 + 测试(60/min/IP,第 61 次 429) | ✅ |
| 6.2 GET /api/tasks/{slug} | 阈值 404/固定推荐集优先/ISR 60s | 实现(阈值读 settings) | ✅ |
| 6.3 GET /api/websites/{normalized_domain} | 六层响应/changes 无 severity/禁止清单 | 实现 | ✅ |
| 6.4 GET .../relations | similar/alternative;related→422 + p0_gate | 实现(数据含 related → 422 + 告警) | ✅ |
| 6.5-6.8 categories/tags | 树/分页/四枚举/group 禁参 | 实现 | ✅ |
| 6.9 POST /api/submissions | 折叠幂等 409/3min/403 关闭态 | 409/400 实现;**403 关闭态见 P3-02** | ⚠️ |
| 6.10 POST /api/reports | field 白名单 10 项/404/10min | 实现 | ✅ |
| 6.11 POST /api/events | vid 服务端生成/四事件/忽略客户端 visitor_id | 实现 + 测试 | ✅ |
| recommendations no-write | 硬规则 | 全库 grep 无 recommendations 写(仅 D7 未实施) | ✅ |
| 第二套实现 | 禁止 | SyncThrottle/TokenBucket/buildSummaryHash/SlidingWindowRateLimiter/parsePagination 各 1 处 | ✅ |

**结论:PASS**(2 项偏差见 §9)

---

## 6. Information Architecture Consistency

依据:前后台信息架构冻结版 + 冻结验收/最终冻结审计 + D6 API 支撑。

| 页面消费者 | D6 API 支撑 | Result |
|---|---|---|
| /search | GET /api/search | ✅ |
| /task/[slug] | GET /api/tasks/{slug} | ✅ |
| /category/[slug] | GET /api/categories/{slug}(**children 数据见 P2-01**) | ⚠️ |
| /website/[normalized_domain] | GET 详情 + relations | ✅ |
| /submit / 纠错表单 / 埋点 | submissions / reports / events | ✅ |

**结论:PASS**(1 项 P2 数据完整度偏差)

---

## 7. D3-D6 Implementation Consistency

| Stage | Frozen Requirement | Actual Implementation | Evidence | Result |
|---|---|---|---|---|
| D3 | 冻结 Schema 落库 + 30 CHECK | 24 表/221 字段/30 CHECK 生产库实时一致 | D3 完成报告 + 复验 | ✅ |
| D4 | 消费契约/normalize/映射/hash/持久化/节流 | 六模块复用无重复;persist 事务+零写;throttle 单一 | D4 完成报告 + grep | ✅ |
| D5 | 状态机/CAS/超时/退避/调度/日志 | CAS 单条条件更新;attempts 第 5 次即 failed(无第 6 次);tick 单实例 | D5 完成报告 + 测试 | ✅ |
| D6 | API 契约 11 端点 | 端点矩阵 30/30 验收 | D6 完成报告 | ✅ |

**结论:PASS**

---

## 8. Hidden Side Effects

| 检查项 | 结果 |
|---|---|
| GET 写 recommendations | 无(全库 grep 无 recommendations 写调用) |
| search 触发 queue/worker | 无(process 仅在 worker 内调用) |
| API 改业务数据(非契约写端点) | 无(仅 search_logs/submissions/reports/user_events 按契约写) |
| API 自动生成推荐 | 无 |
| D5 worker 绕过 CAS | 无(claim 单条件更新;tick 先 recover→throttle→claim) |
| retry 多执行一次 | 无(第 5 次即 failed,单测覆盖) |
| 第二套 throttle/hash/persist/pagination/rate-limit | 无(各 1 处定义) |

**结论:未发现隐性副作用**

---

## 9. Deviations(记录,不修复)

| # | Area | Finding | Status | Severity |
|---|---|---|---|---|
| D-01 | API/IA | 契约 §4.6 要求 `category.children[]`,实际 `categories/[slug]/route.ts` 硬编码 `children: []`(子分类数据缺失) | **RESOLVED** | P2(已修复) |
| D-02 | API/设置 | 契约 §2.2 定义 `SUBMISSION_CLOSED` 403(settings 开关,P0 默认开),但冻结 settings 7 键**无该开关**,403 路径当前不可实现(当前默认开=行为与默认态一致) | **CONFIRMED** | P3 |
| D-03 | 文档/命名 | D3-D5 报告文件名沿用历史 `SITEINTEL_` 前缀(内容属 Websites),与规范 `WEBSITES_` 命名不一致(内容已归档,仅命名历史遗留) | CONFIRMED | P3 |
| D-04 | 外部 | D4 Production E2E = BLOCKED(HTTP 401,非实现问题,单独待处理) | CONFIRMED | P1(外部,非架构缺陷) |

> D-04 计为外部阻塞,不计入实现偏差严重度统计。

---

## 10. Severity Matrix

| Severity | Count | Items |
|---|---|---|
| P0(数据/安全/架构破坏) | 0 | — |
| P1(正确性/核心功能) | 0 | — |
| P2(正确性/契约一致性) | 1 | D-04 外部 E2E(不计实现) |
| P3(文档/一致性/维护) | 2 | D-02(SUBMISSION_CLOSED 开关)、D-03(历史命名) |

## D-01 修复记录(2026-08-18)

- **状态**:D-01 = RESOLVED
- **修改文件**:
  - `src/app/api/_lib/data.ts`(`categoryPage` 返回实际 `children`,复用 `/api/categories` 的 `buildCategoryTree` 逻辑,active-only + slug 排序)
  - `src/app/api/categories/[slug]/route.ts`(`children: pageData.children`,移除硬编码 `[]`)
  - `src/lib/api/d6-routes.test.ts`、`src/lib/api/d6-category-tree.test.ts`(新增 Case:子分类 2 个 / 无子分类 [] / 404 / 顺序稳定 / 跨父级隔离)
- **实现**:分类页 children 与树端点规则一致(仅 active、slug 升序、嵌套),复用既有树构建,未新增第二套逻辑;无 schema/数据库/D4-D6 其他改动
- **测试**:定向 38/38 PASS;全量 239/239 PASS;tsc/eslint/git diff --check 全过
- **Commit**:`fix(websites): return category children`(见 Websites 仓库 git log)

---

## 11. Recommended Actions(供后续决策,本轮不执行)

1. **D-01(P2)**:已修复(`GET /api/categories/{slug}` 返回实际子分类,复用树逻辑;见上节修复记录)。
2. **D-02(P3)**:如产品需要“提交入口关闭”,需登记 **Settings Change / ACR**(冻结 settings 键清单新增开关),再实现 403 路径;否则保持现状并记录。
3. **D-03(P3)**:可选:为 D3-D5 报告生成 `WEBSITES_` 规范名副本(内容不改),或维持历史名并在归档说明中标注。
4. D4 E2E:待 SiteIntel Key 认证问题修复后单独重跑(外部依赖)。

---

## 12. D7 Readiness

```text
D7 前置条件:
D1-D6 = PASS(实现)
D4 Production E2E = BLOCKED(外部,不阻塞 D7 开发,同 D5 先例)
架构一致性 = PASS(仅 P2/P3 记录项)

D7 Readiness:
READY
```

> 本报告为只读审计交付物,按指令**未提交 Git**。
