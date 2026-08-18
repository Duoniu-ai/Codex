# WEBSITES_D6_PREFLIGHT_REPORT

> 项目:**Websites**(网址发现平台/网址导航站;独立产品,SiteIntel 仅为上游数据源)
> 预检日期:2026-08-18(+0800)
> Git HEAD:b1b92e763a038cb51976bb518a0540238333ae21(D5,worktree clean)
> 本轮性质:**只读预检**;未修改代码/数据库/`.env`,未提交 Git

---

## 1. D6 Definition

| 项 | 内容 |
|---|---|
| D6 名称 | **Public API**(P0 任务拆解 V1.0 冻结版 D6) |
| D6 目标 | 按 API 契约 §4 实现 Public 11 端点(18 项定义逐项)并完成集成验收 |
| D6 任务编号 | 6.1 GET /api/search → 6.11 POST /api/events + 6.12 集成验收核对表 |
| D6 输入 | API 契约 §1(通用)/§2(错误码)/§3.1(总览)/§4(11 端点全文)/§6(稳定推荐层)/§7(关系边界);信息架构 §9.1;架构 §7/§8 |
| D6 输出 | `src/app/api/`(Public 路由)+ `src/lib/search` / `src/lib/ranking` / `src/lib/reasons` 实现 + 测试 |
| D6 依赖 | D4+D5(联调完成,同步数据可用)——冻结任务书前置 |
| D6 验收标准 | 6.12 核对表全 PASS;集成冒烟通过;搜索验收口径准备(供 D10 测试集) |
| D6 DoD | 端点矩阵核对表 PASS + 测试覆盖 |

**允许修改目录(冻结)**:`src/app/api/`(Public 路由)、`src/lib/search`、`src/lib/ranking`、`src/lib/reasons`

---

## 2. Current State

### 代码

| 对象 | 状态 |
|---|---|
| `src/app/api/` | 目录存在,**0 个路由文件**(待 D6 新建) |
| `src/lib/search` / `src/lib/ranking` / `src/lib/reasons` | 空目录(待 D6 新建) |
| `src/lib/api/errors.ts` / `pagination.ts` | 已实现(D1),D6 只读复用 |
| `src/lib/rate-limit/sliding-window.ts` | 已实现(D1),含 Public 速率位(60/min 搜索等),D6 只读复用 |
| `src/lib/sync`(D4) / `queue`+`worker`+`instrumentation`(D5) | 已提交;D6 不修改 |

### 数据库(2026-08-18 实时只读复核)

| 对象 | 状态 |
|---|---|
| D6 涉及 18 张表 | 全部存在(24 表基线);关键读表:tasks=30 / websites=150 / categories=7 / tags=14 / task_related_tags=77 / website_tasks=150 / recommendations=0 / website_relations=0 |
| D6 写表 | submissions=0 / reports=0 / user_events=0 / search_logs=0(空,待写入) |
| settings | 7 键齐全,含 D6 所需:`constraint.dictionary`、`ranking.weights`、`task.index_thresholds` |
| 约束 | 30 CHECK / 25 FK / 16 UNIQUE / 31 INDEX 与冻结版一致 |

> 备注(记录不处理):`PROJECT_STATUS.md` 尚未更新 D5 状态(该文件 D5 轮按范围未改);D5 完成状态以 `SITEINTEL_D5_SYNC_QUEUE_WORKER_COMPLETION_REPORT.md` 为准。

---

## 3. D5 → D6 Dependency

| Existing Module | D6 Dependency | Read/Write | Required |
|---|---|---|---|
| D4 `src/lib/sync/**` | 数据产出依赖(D6 读 website_status/snapshots 等同步数据) | Read(数据) | **YES**(数据层面) |
| D5 `src/lib/queue/**` | Public API 不暴露队列端点(§3.1 无 sync) | — | NO |
| D5 `src/lib/worker/**` | 不直接调用 | — | NO |
| D5 `src/instrumentation.ts` | 不相关 | — | NO |

> 冻结任务书要求 D4+D5 联调完成作为前置;D6 代码不调用 D4/D5 模块,只读同步产生的数据。

---

## 4. Database Assessment

```text
New table / New fields / New FK / New UNIQUE / New INDEX / New CHECK = NO
Migration = NO
Data migration = NO
```

D6 全部基于现有 24 表结构;本轮零写入。

---

## 5. Code Assessment

- 需新建:`src/app/api/` 11 个路由(search/tasks/websites/categories/tags/submissions/reports/events)、`src/lib/search`(Task 识别/Constraint Mapper/ILIKE 兜底/Query 解析)、`src/lib/ranking`(五因子加权)、`src/lib/reasons`(理由模板)
- 可复用:`errors.ts`(冻结错误码/错误体)、`pagination.ts`、`sliding-window.ts`(限流)、D4/D5 不触碰
- D6 与 D5 职责边界:Public API 只读数据 + 写 submissions/reports/user_events/search_logs;队列/Worker 完全隔离

---

## 6. API / Contract Assessment

| 项 | 结论 |
|---|---|
| 端点 | 11 个 Public 端点(§3.1),路径/方法/请求/响应字段冻结 |
| 鉴权 | 匿名;无新 API key/token/webhook/callback(安全风险低) |
| 限流 | 复用现有内存滑动窗口:search 60/min/IP、submissions 3/min、reports 10/min、events 60/min |
| 幂等 | submissions 服务端折叠幂等(409,UNIQUE normalized_domain);search_logs GET 副作用单行写 |
| 错误处理 | 复用冻结错误码(VALIDATION_ERROR/NOT_FOUND/RATE_LIMITED 等) |
| 硬规则 | 搜索/任务页**不得写 recommendations**(§6);related=P1 门控,发现异常 related 行 → 422 + 告警日志(§7) |
| 外部 API | 无新外部依赖;页面永不实时调 SiteIntel(离线优先) |

---

## 7. Risk Assessment

| # | 风险 | 评估 |
|---|---|---|
| 1 | 并发:D6 与 D5 worker 竞争 | 低(写集不重叠:D6 写 4 张业务表,D5 写 sync_queue/sync_logs;读多写少的普通并发) |
| 2 | 幂等 | 低(submissions 折叠 409;D4 hash/persist 零写不受影响) |
| 3 | 数据一致性 | 低(单行插入无 partial write;GET 读已提交数据;唯一风险 = recommendations 误写,由硬规则 + 测试断言防) |
| 4 | 安全 | 低(匿名只读 + 3 个受控写端点,无新密钥/凭证/回调;限流防滥用) |
| 5 | 性能 | 低(search 内存词典 + 索引查询 + page_size≤100;ILIKE 兜底有索引;高频热词可选 30s 内存缓存) |
| 6 | 搜索质量 | 中(Task Top-1 ≥85% 种子覆盖为验收项,需按种子集验证;属验收而非阻塞) |

**总体 D6 Risk:LOW**

---

## 8. D6 Difference Matrix

| # | D6 Requirement(冻结) | Current State | Target | Difference | Risk |
|---|---|---|---|---|---|
| 1 | 6.1 GET /api/search | 无路由 | search 路由 + Task 识别/约束/排序/理由/search_logs | 需新建 | MEDIUM(搜索质量) |
| 2 | 6.2 GET /api/tasks/{slug} | 无路由 | 阈值 404 + 固定推荐集优先 + 五因子 + ISR 60s | 需新建 | LOW |
| 3 | 6.3 GET /api/websites/{normalized_domain} | 无路由 | 六层响应 + 禁止清单硬边界 | 需新建 | LOW |
| 4 | 6.4 GET .../relations | 无路由 | similar/alternative only;related → 422 | 需新建 | LOW |
| 5 | 6.5-6.8 categories/tags | 无路由 | 树/分页/四枚举过滤 | 需新建 | LOW |
| 6 | 6.9 POST /api/submissions | 无路由 | 折叠幂等 409 + 3/min + 403 关闭态 | 需新建 | LOW |
| 7 | 6.10 POST /api/reports | 无路由 | field 白名单 10 项 + 404 | 需新建 | LOW |
| 8 | 6.11 POST /api/events | 无路由 | vid 服务端生成/校验 + 四事件白名单 | 需新建 | LOW |
| 9 | 6.12 集成验收 | 无 | 11×18 核对表 + 冒烟 + search_logs 验证 | 需新建 | LOW |
| 10 | search/ranking/reasons 库 | 空目录 | 纯函数实现 + 单测 | 需新建 | LOW |
| 11 | 数据库 | 18 表就绪、settings 齐 | 无变更 | 无差异 | LOW |
| 12 | D4/D5 模块 | 已提交 | 只读复用 | 无冲突 | LOW |

---

## 9. D6 File Scope

| File | Existing | Required Change | Reason |
|---|---|---|---|
| `src/app/api/search/route.ts` | 无 | 新建 | 6.1 |
| `src/app/api/tasks/[slug]/route.ts` | 无 | 新建 | 6.2 |
| `src/app/api/websites/[normalized_domain]/route.ts` | 无 | 新建 | 6.3 |
| `src/app/api/websites/[normalized_domain]/relations/route.ts` | 无 | 新建 | 6.4 |
| `src/app/api/categories/route.ts`、`categories/[slug]/route.ts` | 无 | 新建 | 6.5/6.6 |
| `src/app/api/tags/route.ts`、`tags/[slug]/route.ts` | 无 | 新建 | 6.7/6.8 |
| `src/app/api/submissions/route.ts` | 无 | 新建 | 6.9 |
| `src/app/api/reports/route.ts` | 无 | 新建 | 6.10 |
| `src/app/api/events/route.ts` | 无 | 新建 | 6.11 |
| `src/lib/search/**`(task-matcher/constraint-mapper/fallback/query) | 空目录 | 新建 | 6.1 输入处理 |
| `src/lib/ranking/**`(五因子) | 空目录 | 新建 | 排序 |
| `src/lib/reasons/**`(模板) | 空目录 | 新建 | 可解释理由 |
| `src/lib/api/errors.ts` / `pagination.ts` / `rate-limit/*` | 已实现 | **不修改(只读复用)** | D1 基线 |
| D4/D5 模块 / schema / migrations / `.env` | 已提交 | **不修改** | 范围红线 |

---

## 10. D6 Acceptance Test Plan

- **Unit**:Task 识别(全等/前缀/包含/多 Token Top-1)、Constraint Mapper(settings 词典)、五因子排序权重、理由模板、normalize/折叠、vid 生成校验、field 白名单
- **API 集成**:11 端点 × 18 项定义核对表;200/400/404/409/422/429/500 路径;分页边界(page_size≤100)
- **硬规则断言**:搜索/任务页后 recommendations 行数不变(零写);relations 响应不含 related;p0_gate 异常 related → 422
- **限流冒烟**:search 60/min、submissions 3/min、reports 10/min、events 60/min
- **落库验证**:search_logs / submissions / reports / user_events 写入正确
- **回归**:全量 `npm test`(现 159/159)+ tsc + eslint

---

## 11. Scope Boundary

```text
D6 = PREFLIGHT COMPLETE
D6 Implementation = NOT STARTED

D7+ = NOT EXECUTED
D10 = NOT EXECUTED

D4 Production E2E = BLOCKED(HTTP 401,单独待处理)
Codex Report Push = BLOCKED(github.com:443 网络,单独待处理)
```

> 本报告为预检交付物,按指令**未提交 Git**。
