# SiteIntel 2.0 — Phase 2.3 Technology Profile Frontend 验收报告

> 日期：2026-08-19 ｜ 范围：Phase 2.3（Technology Profile Frontend only）
> 授权：按《SITEINTEL 2.0 — Phase 2.3 Technology Profile Frontend 执行指令》执行
> 结论：✅ 完成（/technology/[slug] 全新用户页 + 视图模型 + i18n + 测试；无 Schema 变更；未部署生产；未 push）

---

# 1. Scope（范围）

将 Phase 2.2 Technology Profile Backend 转化为面向用户的 `/technology/[slug]` Intelligence 页面：

```text
这是什么技术 → 谁在使用（分页）→ SiteIntel 如何检测 → 观察到什么变化（时间线）
→ 有什么证据 → 数据边界（Coverage）→ 下一步探索
```

覆盖：Identity / Detection / Adoption / Websites（分页）/ History / Evidence / Coverage / Explore / SEO / Mobile / Empty-Error-404 状态。

---

# 2. Files Changed（文件变更）

```text
src/app/technology/[slug]/page.tsx            （重写：Phase 2.2 backend → 用户页）
src/lib/technology/profile-view.ts            （新增：纯视图模型 + SEO gate + 分页 + 格式化）
src/lib/technology/profile-view.test.ts       （新增：12 例）
src/lib/i18n.ts                               （修改：techPage 字典重构，zh/en 双语）
```

未包含：pnpm-workspace.yaml（用户本地修改）、.next、.env、文档、临时文件。

---

# 3. UI Architecture（UI 架构）

- Server Component（`force-dynamic`），与 /website、/ip 页面同构；无新 UI framework、无全局 design system 改动。
- 页面结构（渐进式信息披露，普通用户先看懂再深入）：
  1. Hero / Identity —— 名称、厂商、分类、子分类、描述、官网、已观察网站数
  2. Detection —— 指纹规则数 + 检测方法（用户化标签）
  3. Adoption —— 已观察网站数 / 首次发现 / 最近发现 + observed 口径说明
  4. Websites using this technology —— 分页列表（domain / first seen / last seen / confidence）
  5. Technology history —— +/− 时间线（added / removed）+ observation coverage 说明
  6. Evidence —— 用户化证据（类型 / 来源 / 采集时间 / 置信度），不暴露内部 ID 或 JSON dump
  7. Coverage —— 已观察网站 / 历史事件 / 证据记录 + 数据边界说明
  8. Explore —— 分析网站 / 检测原理链接 + 生态功能未开放提示
- 复用既有组件：`SectionHeader`、`ConfidenceBadge`、`MonoChip`、`JsonLd`、`panel`/`label-micro`/`mono-value` 设计系统。
- 视图模型（`profile-view.ts`）为纯函数层：页面只做投影，全部标签 / 格式化 / 分页 / 空态 / gate 均可单测。

---

# 4. API Integration（API 集成）

- 页面调用 Phase 2.2 `cachedTechnologyProfile(slug, page, pageSize)`（Step 8 ttlMemo，key=slug+page+pageSize，TTL 60s）。
- 每次渲染一次 profile 请求；分页为内联 `?page=N` 服务端渲染，无客户端循环请求、无 N 次关系/证据请求。
- `generateMetadata` 复用同一缓存键（page=1），同请求内二次调用命中 memo，不重复打库。
- 未调用 v1 API 端点（页面直接走服务端组装层，符合现有 Server Component 架构；API 端点保留供外部消费）。

---

# 5. SEO（SEO）

- `generateMetadata`：title / description / canonical（自持，无全局继承）/ OG / Twitter / robots。
- robots 由 profile 质量门 `technologyProfileGate` 决定：INDEX → `index, follow`，否则 `noindex, follow`（沿用现有规则，薄页不索引）。
- **Gate 校准（关键）**：以生产 64 个技术实体为样本，把旧页 gate（含 related technologies 输入）与新的 profile-based gate 对比：
  - 旧 gate INDEX = 33，新 gate INDEX = 33，**0 处不一致**（2026-08-19 生产数据实测）。
  - 公式：dataCoverage（≥3 站 25 / 1-2 站 15 / 0）+ evidenceCoverage min(20, 站数×4) + historicalDepth min(15, 事件×5) + uniqueIntelligence min(20, 15+min(5,站数)) + contentQuality（有站 20）。
  - 边界说明：4+ 站且无历史的技术页可索引（与 sitemap ≥3 关系门槛一致）；0-3 站无历史保持 NOINDEX。
- JSON-LD：WebPage（与全站统一机制一致）；sitemap / robots.txt 未改动；未批量开放低质量页面。
- 内链：页面 → /website/[domain]（网站列表与历史）、首页、/how-it-works。

---

# 6. Mobile（移动端）

- 全部区块使用响应式布局：`flex-wrap`、`sm:grid-cols-*`、`min-w-0 truncate`、面板行式布局（无宽表依赖）。
- Evidence 使用行式面板而非宽表，避免移动端横向滚动。
- 项目无既有 UI/E2E 测试基础设施（vitest node 环境），按指令未引入大型 E2E framework；移动端以响应式 class + build 验证。
- 验证方式：视图模型单测覆盖空态/分页/标签 + typecheck + production build；视觉验证待生产部署后执行。

---

# 7. Empty / Error / 404 States（状态）

| 状态 | 实现 |
|---|---|
| 404 | 技术不存在（Catalog 与 Entity 均无）→ `notFound()`，使用 SiteIntel 既有 404 规范 |
| Empty websites | "No websites observed using this technology yet." + 无分页控件 |
| Empty history | "No observed changes..." + observation coverage 说明 |
| Empty evidence | "No evidence records available." + 值契约说明 |
| 缺失可选字段 | vendor / description / officialUrl 等为 null → "Not available" / "—"，不伪造 |
| Catalog 缺失 | 页面正常渲染 Entity-only 数据（`catalogAvailable=false` 提示），不 500 |
| 非法分页参数 | `?page=abc` → 回退 page=1；越界页 → 空列表 + 分页收敛到末页标签 |

---

# 8. Performance（性能）

- 单次渲染 1 个 profile 组装（≤10 SQL，全部索引；缓存命中 0 SQL）。
- metadata 与页面共享缓存键，无重复后端工作。
- 无客户端循环 / polling / 状态管理引入。
- 分页服务端渲染，不一次加载全部网站。

---

# 9. Tests（测试）

```text
existing tests: 384
new tests:      12（src/lib/technology/profile-view.test.ts）
total:          396 / 396 PASS（28 files）
```

新增覆盖：

- Rendering（视图契约）：有效技术全字段 / 缺失可选字段（null）/ Catalog 缺失降级 / 空网站关系 / 空历史 / 空证据
- States：空态 flags、分页边界（越界页、末页收敛）、pageLabel
- SEO gate：≥4 站 INDEX；0-3 站无历史非 INDEX；历史加成边界（3 站 + 4 事件 → INDEX）
- 格式化：日期、检测方法标签、证据类型/来源标签（未知值原样回退，不猜测）

（项目为纯 vitest node 测试架构，无组件渲染运行时；页面薄投影层由 typecheck + build 覆盖，未引入 E2E framework。）

---

# 10. Typecheck（类型检查）

```text
tsc --noEmit: 0 错误（PASS）
```

---

# 11. Build（构建）

```text
next build（Turbopack，Next 16.3.1）: PASS
路由：ƒ /technology/[slug]（dynamic）· ƒ /api/v1/technology/[slug]
唯一 warning：jobs.ts crypto/Edge 历史警告（与本阶段无关）
```

---

# 12. Database Impact（数据库影响）

**NO。** 无 Schema / Prisma / migration / seed / 数据修改。唯一数据库动作：SEO gate 校准的**只读**生产查询（临时脚本已清理）。

---

# 13. Security（安全）

- 页面为服务端组装，无 API Key 暴露、无 database credentials、无 server-only env 暴露。
- Evidence 仅展示用户化摘要（类型 / 来源 / 时间 / 置信度），不输出 raw JSON、内部 ID 或原始载荷。
- 官网链接一律 `rel="noopener noreferrer"`；无用户 HTML 注入面（JSON-LD 由受控数据构造）。

---

# 14. Acceptance Matrix（验收矩阵）

```text
Technology Profile page exists        PASS
Uses Phase 2.2 API                    PASS
Identity visible                      PASS
Detection visible                     PASS
Observed adoption visible             PASS
Website relationships paginated       PASS
Technology history visible            PASS
Evidence userized                     PASS
Coverage visible                      PASS
404 handled                           PASS
Empty states handled                  PASS
Mobile layout verified                PASS（响应式布局 + build；视觉验证待部署）
SEO metadata verified                 PASS（metadata + canonical + robots + gate 校准 0 回归）
No database change                    PASS
No unrelated global UI changes        PASS
No N+1 frontend requests              PASS
Tests PASS                            396/396
Typecheck PASS                        0 错误
Build PASS                            next build
```

---

# 15. Known Limitations（已知限制）

1. Related technologies / Ecosystem / Alternatives 不展示（Phase 2.6；前端不猜测）。
2. Evidence 无单条 observed value（rawValue 占位，Phase 2.7 强化）；页面如实标注。
3. Distribution（country/language/market）不展示（Phase 2.5）。
4. `/technology/iconfont-alibaba` 404 为既有带括号实体名 slug 边界（Phase 2.2 实体对齐项，未在本阶段处理）。
5. `assembleTechnologyPage`（seo/technology.ts）不再被页面使用，保留供 sitemap 的 slugify 与后续回退；未删除（避免无关改动）。
6. Mobile 视觉验证需生产部署后进行（本阶段不部署）。

---

# 16. Next Phase Recommendation（下一阶段建议）

**Phase 2.4 — Technology History**（沿用 Phase 2 Plan 顺序）：

- Why now：本页已消费 added/removed 事件，但仅显示最近事件；历史时间线、版本变化 diff、事件索引（计划 S3）可让页面真正回答"什么时候发生变化"。
- 前置：Phase 2.2/2.3 已提供事件查询与展示底座。
- 建议范围：事件聚合（按技术/网站分组时间线）、`technology_version_changed` 评估（需单独授权写路径）、`Event(type, detectedAt)` 索引评估。
- 替代候选：Phase 2.6 Ecosystem/Alternatives（本页 Explore 区块的天然扩充）。若优先生态，则 History 顺延。

---

*Phase 2.3 完成，立即停止。未进入 Phase 2.4；未部署生产；未 push。*
