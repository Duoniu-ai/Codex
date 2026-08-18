# SITEINTEL SEO PHASE 1 — P0 REPAIR IMPLEMENTATION PLAN

SEO Phase 1 P0 修复实施计划 —— 2026-08-17（规划阶段，**未修改任何代码**）

## 基线声明

| 项 | 值 |
|---|---|
| 生产代码基线 | 服务器 `1bdfcb4`（线上运行构建，PRODUCTION BASELINE CONFIRMED） |
| GitHub 基线 | `Duoniu-ai/siteintel`（PRIVATE）`main` @ `b52ec84`，tag `production-baseline-2026-08-17` |
| 工作仓库 | `C:\Users\deepo\siteintel-github`（GitHub 基线的本地工作副本，有 origin） |
| 工作分支 | `seo/phase-1-p0`（从 main 创建，不在 main 上直接修改） |
| 代码同源性 | 工作仓库代码 = 服务器代码（b52ec84 = 1bdfcb4 的 PUBLIC FILESET 镜像） |
| 服务器仓库 | 冻结不动作（历史完整保留，禁止向其提交本阶段修改） |

**本次审计结论先行**：6 个 P0 全部为**纯代码层问题**，根因均已定位到具体文件行。零数据库 Schema 变更、零迁移、零生产环境触碰。

---

## 总体设计：单一 Indexability 事实来源

```
             ┌──────────────────────────────────────────────┐
             │  src/lib/seo/index-gate.ts (新)                │
             │  classifyWebsiteIndexability(input) → bool    │
             │  ├─ src/lib/seo/low-value-domain.ts (新)       │
             │  │   isLowValueDomain(domain) → bool           │
             │  └─ HOT_DOMAINS / MANUALLY_APPROVED 白名单     │
             └──────┬─────────────────────────────┬──────────┘
                    │ 同一函数                     │ 同一函数
        ┌───────────▼───────────┐     ┌───────────▼───────────┐
        │ /website/[domain]     │     │ sitemap.ts            │
        │ generateMetadata      │     │ 批量加载同一输入字段   │
        │ robots = index/noindex│     │ 过滤 website 条目      │
        └───────────────────────┘     └───────────────────────┘
```

**原则**：页面 metadata、sitemap 调用**同一个纯函数** `classifyWebsiteIndexability`，输入字段完全同构（页面从已装配 report 提取；sitemap 从批量查询提取）。任何一方改判断 = 双方自动一致。

---

## P0-1：不存在域名 Soft 404

### 根因（已定位）

- `src/app/website/[domain]/page.tsx:79-91`：`assembleReport()` 返回 `null`（**仅当 Target 表无该域名**，见 `src/lib/report.ts:16-22`）时，渲染 200 空态引导页（`dict.report.noReportYet` + AnalyzeForm），HTTP 200 + noindex —— 语义上是 Soft 404。
- `assembleReport` 语义（report.ts:15-22）：Target 存在 → **永远返回非 null**（即使事实全空）；`report.latest.status`（running/completed/partial/failed）与 `report.target.investigationCount`（report.ts:25 实时 count）为真实字段。**这提供了区分五态的完整信息，无需新字段。**
- 附带发现小 bug：page.tsx:26 默认 `robots = "index, follow"`，当 `normalized` 为 null（非法域名）时反而输出 index（应为 noindex）。

### 涉及文件

| 文件 | 动作 |
|---|---|
| `src/app/website/[domain]/page.tsx` | 修改：五态分支 + generateMetadata robots 修正 |
| `src/app/website/[domain]/not-found.tsx` | **新增**：路由级 404 页（复用现有空态引导 UI） |
| `src/lib/i18n.ts` | 修改：新增 处理中 状态文案（zh/en 两套） |

### 修改方案（状态机）

```
/website/:domain 五态：
1. normalizeDomain 失败                     → notFound()          (404)
2. Target 不存在（assembleReport = null）   → notFound()          (404) ← 真 404 语义
3. latest.status = running 且事实全空        → 200 + noindex + 「分析处理中」UI（刷新提示）
4. latest.status = failed 且事实全空         → notFound()          (404) ← 分析失败无数据
5. completed/partial 但事实全空              → notFound()          (404) ← 暂无有效数据
6. 其余（有事实数据）                        → 200 正常报告页；robots 由 P0-3 Gate V1 决定
```

- **404 用路由级 `not-found.tsx`**：`app/website/[domain]/not-found.tsx` 渲染当前空态引导内容（保留 AnalyzeForm 引导 = 「明确设计的资源不存在状态」），但 HTTP 状态码为真正的 404。全局 `app/not-found.tsx`（已存在）不动。
- 事实全空的判定：`dns 记录/NS 为空 && ips 为空 && technology 为空`（与 P0-3 数据门槛同源，抽成共享辅助函数）。
- `generateMetadata` 同步五态：null 报告 / 处理中 / 失败 / 暂无数据 → `noindex, follow`；修正 normalized 为 null 时的默认 index 泄漏；有数据 → Gate V1 结果。
- 处理中状态 UI 需要少量 i18n 新键（`dict.report.processingTitle/processingText`，zh/en）。

### 修改风险

- 低。已分析过的域名（Target 存在）行为不变或更清晰；仅「从未分析过 + 分析失败无数据」的域名从 200 变 404，符合 HTTP 语义。
- 风险点：用户从搜索引擎直接点进 404 域名，无引导 —— 已由路由级 not-found 保留 AnalyzeForm 缓解。
- 不涉及采集/分析管线，零数据操作。

### 不修改

- `app/not-found.tsx`（全局 404，保留）
- `src/components/report/report-page.tsx`（报告渲染组件）
- AnalyzeForm 组件、分析管线、Target/Investigation 数据

### 验证

- 新增单元测试：五态判定逻辑抽为纯函数（若抽取出）或页面级 fixture 测试；至少验证状态映射。
- 部署批准后线上验证：`/website/nonexistent-test-domain-123456789.com` → **HTTP 404**；`/website/<真实已分析域>` → 200 + Gate 决定；处理中/失败态用真实数据核对。

---

## P0-2 + P0-3：统一 Indexability + Simple Index Gate V1

（两个问题同根同解：当前「页面一套、sitemap 一套」两套逻辑并存，且页面用的是五维评分。）

### 根因（已定位）

- **页面侧**：`src/app/website/[domain]/page.tsx:27-35` 调 `evaluateWebsitePageGate`（`src/lib/seo/quality-gate.ts:24-68`）= 五维评分（数据 25 + 证据 20 + 历史 15 + 情报 20 + 内容 20，≥80 INDEX），内部用 `registry.evaluateQualityGate`。
- **sitemap 侧**：`src/app/sitemap.ts:27-42` 用**另一套**过滤（`type:"full" 快照 ≥1 && (事件 ≥1 || 洞察 ≥1)`）—— 与页面 gate 不同源。审计实测：sitemap 26 个 website URL 中 10 个页面 robots=noindex（facebook/instagram/taobao/tencent 等热门大站因基础设施稳定、历史变化少 → 五维分低 → noindex，但仍在 sitemap）。
- **字段审计结论（无猜测，已核实）**：`Target` 表**无** analysisCount/investigationCount 列（schema.prisma:17-34）；`investigationCount` 是 `report.ts:25` 的实时 `prisma.investigation.count()` 计算值；`Investigation.status` 取值 `running | completed | partial | failed`（schema.prisma:45）。**全部复用现有数据，零 Schema 变更。**

### 涉及文件

| 文件 | 动作 |
|---|---|
| `src/lib/seo/index-gate.ts` | **新增**：Gate V1 核心（纯函数） |
| `src/lib/seo/low-value-domain.ts` | **新增**：低价值域名判断（P0-6 共用） |
| `src/app/website/[domain]/page.tsx` | 修改：gate 调用替换 + 五态（P0-1 同改） |
| `src/app/sitemap.ts` | 修改：website 条目过滤改用同一函数 |
| `src/app/admin/seo/page.tsx` | 修改：改用 Gate V1 展示（决策原因，替代 breakdown 列） |
| `src/lib/seo/quality-gate.ts` | **删除**（evaluateWebsitePageGate 唯一两个调用方被替换后无引用） |
| `src/lib/seo/registry.ts` | 不动（`evaluateQualityGate:266` 仍被实体页使用：entity.ts ×4、technology.ts） |

### 修改方案（Simple Index Gate V1）

```ts
// src/lib/seo/index-gate.ts（新，纯函数，全部字段可注入 → 可单测）
interface WebsiteGateInput {
  domain: string;
  investigationStatus: "completed" | "partial" | "running" | "failed" | null;
  investigationCount: number;
  hasDns: boolean;        // dns_records 有记录 或 nameservers 非空
  hasIps: boolean;        // resolved_ips 非空
  hasTechnology: boolean; // technologies 非空
}
type GateReason = "indexed" | "low-value-domain" | "not-analyzed" | "insufficient-data" | "below-activity-threshold";

门槛（三项全部满足）：
  1. 分析成功：investigationStatus ∈ {completed, partial}（最近一次调查成功/部分成功）
  2. 数据完整：hasDns + hasIps + hasTechnology 中 ≥2 为 true
  3. 非低价值域名：!isLowValueDomain(domain)

条件（任一满足）→ index = true：
  A. domain ∈ HOT_DOMAINS          // 热门域名白名单（curated 常量 ~20 条）
  B. investigationCount ≥ 3        // 已被主动分析 ≥3 次（真实字段）
  C. domain ∈ MANUALLY_APPROVED    // 人工审核名单（curated 常量，初始为空）
否则 → noindex
```

- **A/C 名单落代码常量**（如 `src/lib/seo/index-gate.ts` 内导出）：数据库不允许变更（禁止 migrate/push），code constant 是零 Schema 成本的实现，注释说明可维护方式（后续可迁移 DB 表）。初始 HOT_DOMAINS ≈ 20 条主流平台（google/facebook/instagram/youtube/wikipedia/baidu/taobao/tencent/zhihu/weibo/github/openai 等，实施时定稿）。
- **页面侧输入**：从已装配的 `report` 提取（`report.latest?.status`、`report.target.investigationCount`、`report.infrastructure.dns/ips`、`report.technology`）→ 零新增查询；同时删除 generateMetadata 中对 `assembleSearchReport` 的调用（旧 gate 需要，V1 不需要）。
- **sitemap 侧输入**（批量，目标 ~282 条）：一次 `investigation.findMany`（按 startedAt desc，JS 取每 target 首条 = 最新状态）+ `investigation.groupBy count` + `fact.findMany`（factType ∈ dns_records/resolved_ips/technologies，JS 判非空）→ 同一 `classifyWebsiteIndexability` 过滤。原快照/事件/洞察计数查询删除。
- **admin/seo 页**：表格改展示 Gate V1 的输入字段 + 决策原因（替代五维 breakdown）。

### 修改风险

- 中（影响索引集合）：Gate V1 与五维门集合不同 —— 部分现 index 页面（数据丰富但分析 <3 次且非白名单）会转 noindex；热门大站（facebook/taobao 等）转 index（规划目标行为）。sitemap 成员随之变化，Google 需重新爬取（短期波动，属设计预期）。
- 降级路径：A/B/C 名单与门槛均为常量/纯函数，回滚 = 改常量或切回旧实现（旧文件删除前保留 diff 记录）。
- `investigationStatus` 取**最新一次**调查：最近一次失败但历史有数据 → noindex（严格口径，规划文档记录该权衡）。

### 不修改

- **禁止 Prisma migrate / db push / 生产数据库**；`registry.evaluateQualityGate` 五维实现保留（IP/ASN/证书/组织/技术实体页仍在用，不在本阶段范围）；
- 分析管线、facts 引擎、事件/洞察引擎、快照逻辑、采集器。

### 验证

- 新增单测：`index-gate.test.ts`（门槛矩阵：成功/失败、数据 1/2/3 项、白名单命中、次数 2/3、低价值域名拦截、null 状态）+ `low-value-domain.test.ts`；
- 现有 vitest 全量回归（121+ 例）不得破坏；
- typecheck / lint / build 全绿；
- 部署批准后线上验证：sitemap.xml 与页面 robots 一致性脚本（取 sitemap website 条目 → 逐页抓 robots → 断言无 noindex 条目混入；`indexable=false` 条目不在 sitemap）。

---

## P0-4：Canonical 指向首页错误

### 根因（已定位）

- `src/app/layout.tsx:32-34`：根布局 `generateMetadata` 设置了全局 `alternates: { canonical: SITE_URL }` —— Next App Router 中**任何未自行声明 canonical 的路由都会继承它**。
- 全仓 canonical 自持盘点（grep 实证）：
  - **已自持**：首页 `page.tsx:227` ✓、SEO 落地页 `(seo)/[...slug]/page.tsx:55` ✓、实体页 asn:180 / cert:168 / ip:218 / org:169 / tech:136 ✓、website:42 ✓
  - **未自持（继承 layout）**：`/bulk`（bulk/page.tsx:7-10）、`/docs/api`（docs/api/page.tsx:4-7）、`/report`（report/page.tsx:7-14）、`/search`、`/dashboard`、`/login`、`/register`、`/analyze/[id]`
  - `/report/[domain]`（report/[domain]/page.tsx）是纯 308 permanentRedirect 页（无 HTML 输出），canonical 不落 HTML，无需处理。
- 审计实测 `/bulk`、`/docs/api`、`/report` canonical = 首页 —— 三者均为「有内容但未自持 canonical」的页面，问题精确命中。

### 涉及文件

| 文件 | 动作 |
|---|---|
| `src/app/layout.tsx` | 修改：**移除** `alternates.canonical`（32-34 行） |
| `src/app/bulk/page.tsx` | 修改：加 canonical `${SITE_URL}/bulk`（+ P0-5 description） |
| `src/app/docs/api/page.tsx` | 修改：加 canonical `${SITE_URL}/docs/api`（+ P0-5 description/robots） |
| `src/app/report/page.tsx` | 修改：加 canonical `${SITE_URL}/report` |

### 修改方案

1. `layout.tsx` 删除全局 canonical —— 未自持页面将**不再输出 canonical 标签**（正确语义：noindex/私有页不该有 canonical）。
2. 受影响页面清单中，**indexable 且有 SEO 价值的 3 个**（bulk/docs/api/report）全部补自持 canonical（/report 已有自持 description 与 noindex robots，仅补 canonical；/bulk 保持 noindex；/docs/api 保持 index）。
3. `/search`（登录私有、noindex,follow）、`/dashboard`、`/login`、`/register`、`/analyze/[id]`：**不补**（私有页，无 canonical 即正确）。

### 修改风险

- 低。已盘点全部 generateMetadata 出口，确认无页面依赖 layout canonical 提供正确值；首页自持 canonical 不受影响。
- 唯一注意：`not-found` 边界（全局 404）此前带首页 canonical —— 移除后 404 页无 canonical（更正确）。

### 不修改

- 首页 page.tsx:221-227（已自持）、SEO 落地页、实体页的 canonical 逻辑；
- `(seo)/[...slug]`、entity.ts、technology.ts。

### 验证

- 静态验证：grep 断言 `layout.tsx` 无 alternates；新增/修改的 3 个页面均有 `alternates.canonical` 且值正确；
- 部署批准后线上验证：curl `/bulk` `/docs/api` `/report` 的 canonical 各指自身；随机抽 `/login` `/dashboard` 断言无 canonical 标签。

---

## P0-5：Metadata 重复

### 根因（已定位）

- `/docs/api`（docs/api/page.tsx:4-7）：generateMetadata **仅 title**，description 与 canonical 全部继承 layout 首页默认值（`dict.metadata.defaultDesc`）→ 线上实测 description 与首页相同。
- `/bulk`（bulk/page.tsx:7-10）：仅 title + noindex，description 继承首页。
- `/report`（report/page.tsx:7-14）：**已有**自持 title（reportsTitle）+ description（reportsDesc）+ robots —— 仅缺 canonical（P0-4 处理）。
- 其余静态核心页盘点：SEO registry 落地页（含 /how-it-works、/relationship-intelligence、/search-intelligence、/tools/* 等）在 `(seo)/[...slug]/page.tsx:37-55` 自持 title/description/canonical（registry + dict.seoPages 驱动）✓；实体页自持 ✓；首页自持 ✓。

### 涉及文件

| 文件 | 动作 |
|---|---|
| `src/app/docs/api/page.tsx` | 修改：+ 独立 description + robots(index,follow 显式) + canonical（P0-4 同改） |
| `src/app/bulk/page.tsx` | 修改：+ 独立 description + canonical（noindex 保持） |
| `src/lib/i18n.ts` | 修改：新增 `dict.bulk.metaDescription`、`dict.apiDocs.metaDescription`（zh/en 两套 + 类型定义） |

### 修改方案

- description 描述页面真实功能（禁止关键词堆砌）：/docs/api → 「SiteIntel 开放 API v1 文档：报告查询、分析触发、SSE 进度流、限流与 API Key 说明」；/bulk → 「批量分析工具：一次提交最多 20 个域名，登录后使用」类文案，zh/en 各一。
- 新增 i18n 键强制走类型定义（TS strict）→ 漏配任何一套字典直接 typecheck 失败（安全网）。

### 修改风险

- 低。仅 2 页 metadata 增益；i18n 键为纯增量。

### 不修改

- `(seo)/[...slug]`、实体页、首页、/search、dashboard/login/register 的 metadata；
- `dict.seoPages` 结构。

### 验证

- typecheck 通过 = 两套字典键齐全；单测可选（metadata 为页面层，代码级人工审查 diff）；
- 部署批准后线上验证：/docs/api、/bulk 的 `<title>`/`<meta name=description>`/canonical 各自独立且不等于首页。

---

## P0-6：Sitemap 数据污染

### 根因（已定位）

- `src/app/sitemap.ts:17-21`：website 条目来源 = `prisma.target.findMany` 全量（take 500），仅过「快照+事件/洞察」计数过滤 —— **example.com、ecn.t1-3.tiles.virtualearth.net（Bing CDN 瓦片子域）等无用户价值目标进入了 sitemap**（审计实测在 sitemap.xml 中）。
- 来源链：这些域名经发现引擎（SSL SAN/CNAME/跳转/共享 IP 提取）或测试分析进入 Target 表，属真实数据但非「网站」价值实体。

### 涉及文件

| 文件 | 动作 |
|---|---|
| `src/lib/seo/low-value-domain.ts` | **新增**（P0-3 共用同一模块） |
| `src/app/sitemap.ts` | 修改：website 条目过滤 = `classifyWebsiteIndexability`（内含低价值域名判断）+ 域名级显式过滤 |

### 修改方案（可维护的 EXCLUDED 规则，不硬编码几个域名）

```ts
// src/lib/seo/low-value-domain.ts（新）
isLowValueDomain(domain): boolean —— 任一命中即低价值：
  1. RFC 2606/6761 保留后缀：.example / .test / .invalid / .localhost（含 example.com/net/org）
  2. 基础设施/CDN 默认域名后缀（curated 常量数组，注释标明来源，可增删）：
     .tiles.virtualearth.net 等 Bing 瓦片域、.akamaiedge.net、.azureedge.net、
     .cloudfront.net、.fastly.net、.edgekey.net、.trafficmanager.net 等
  3. 已知停放/测试/示例域名（curated 常量数组，初始为空或少量）
```

- **保守原则**：规则 2 仅收录「明显是 CDN/基础设施而非网站」的后缀，实施时以 sitemap 实际出现的污染域为准 + 少量成熟已知项；避免误伤真实站点（如用户业务本身部署在云前端）。
- 该模块同时供 P0-3 Gate 门槛 3 使用（单一事实来源）。
- sitemap 中 website 条目最终 = 目标 ∈ Target && `classifyWebsiteIndexability(...).indexable`（含低价值拦截）。

### 修改风险

- 中低。误伤风险由「保守 curated 列表 + 单测覆盖真实/非真实样本」控制；只影响 sitemap 条目与 website 页索引，不删除任何数据、不影响分析。

### 不修改

- Target 表数据（不删除 example.com 等行——它们是真实观测）；
- 发现引擎候选提取逻辑、实体图谱。

### 验证

- 单测：`low-value-domain.test.ts`（example.com ✓低、ecn.t1.tiles.virtualearth.net ✓低、azureedge ✓低、openai.com ✗、baidu.com ✗、真实 CDN 部署站点样本不误伤）；
- 部署批准后线上验证：sitemap.xml 无 example.com / *.tiles.virtualearth.net 条目。

---

## 实施顺序（严格按用户给定 A-E）

| 阶段 | 内容 | 产出 |
|---|---|---|
| 1-A | P0-1 五态 + 路由级 404 + i18n | 页面状态机、not-found.tsx |
| 1-B | P0-2+P0-3 index-gate.ts + low-value-domain.ts + sitemap + admin/seo + 删 quality-gate.ts | 统一 Gate V1 |
| 1-C | P0-4 layout canonical 移除 + 3 页补 canonical | canonical 自持化 |
| 1-D | P0-5 2 页 description + i18n 键 | metadata 独立化 |
| 1-E | P0-6（实现已随 1-B 落地，此处= sitemap 过滤接线确认 + 单测补齐） | 污染过滤 |

实际代码修改在 `seo/phase-1-p0` 分支一次完成（1-A→1-E 同批次），git commit 可拆 2-3 个逻辑提交（状态机 / Gate 统一 / metadata）。

## Git 工作流

1. `cd C:\Users\deepo\siteintel-github && git checkout main && git pull origin main`（确认 b52ec84）
2. `git checkout -b seo/phase-1-p0`
3. 全部修改在该分支；完成后 `git diff` 逐项审查：
   - 无 `.env*`（仅 .env.example 占位）、无 Token/Password/API Key 字面量、无 `154.204.176.66`/`:3003`/`www/wwwroot` 等服务器信息（grep 全量复查）
4. `pnpm install`（如 prisma generate 需 DATABASE_URL，仅命令行内联占位值，**不落盘**）
5. `pnpm lint && pnpm typecheck && pnpm test`（含新增单测）`&& pnpm build`（NEXT_PUBLIC_SITE_URL 内联）
6. 生成修改报告（SITEINTEL_SEO_PHASE1_P0_REPORT.md）→ **停止，等部署批准**
7. 部署（另行批准后）：合并到 main → push → SSH 直传服务器 → 服务器 git commit → build → restart（服务器工作流不动历史）

## 汇总：修改文件清单（预计 12 个文件，8 改 3 新 1 删）

- 改：website/[domain]/page.tsx、sitemap.ts、layout.tsx、bulk/page.tsx、docs/api/page.tsx、report/page.tsx、admin/seo/page.tsx、i18n.ts
- 新：seo/index-gate.ts、seo/low-value-domain.ts、website/[domain]/not-found.tsx、单测 ×2
- 删：seo/quality-gate.ts
- 不动：prisma schema/migrations、robots.ts、registry.ts、entity.ts、technology.ts、report-page.tsx、not-found.tsx（全局）、分析管线/事实引擎/采集器、数据库

## 不修改内容（重申）

- Prisma Schema / migrations / 生产数据库（零 migrate、零 push、零数据操作）
- 分析管线、Provider、facts/events/insights 引擎、发现引擎、监控调度
- 产品定位、首页/报告页 UI 内容、五维实体页 Gate（IP/ASN/证书/组织/技术）
- 线上环境、服务器 git 历史、robots.txt 策略

## 验证矩阵（汇总）

| 项 | 本地（代码级） | 部署批准后（线上） |
|---|---|---|
| P0-1 | 状态机单测 + typecheck/build | /website/不存在域 → 404；处理中/失败态实测 |
| P0-2/3 | index-gate 单测矩阵 + 全量回归 | sitemap vs 页面 robots 一致性脚本 |
| P0-4 | grep 断言 layout 无 canonical、3 页自持 | curl 3 页 canonical 各自指向 |
| P0-5 | typecheck（i18n 键完整性） | /docs/api /bulk 独立 title/desc |
| P0-6 | low-value-domain 单测 | sitemap 无 example/瓦片域 |

## 结论

本计划基于 1bdfcb4/b52ec84 代码逐行审计（非猜测），全部 6 个 P0 根因已定位到文件行级，方案全部为最小代码修改（8 改 3 新 1 删），零 Schema/零数据库/零生产触碰，Indexability 统一为单一函数事实来源。

**等待用户确认「批准执行 SEO Phase 1 P0」后开始实际代码修改。**
