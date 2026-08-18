# SITEINTEL SEO PHASE 1 — P0 REPAIR 实现报告

- **日期**: 2026-08-17
- **分支**: `seo/phase-1-p0`（基于 `main` @ `b52ec84`，未 merge / 未 push / 未部署）
- **依据**: `SITEINTEL_SEO_PHASE1_P0_IMPLEMENTATION_PLAN.md` + 三个强制修正约束（P0-1 状态语义 / P0-2·3 数据门槛 / quality-gate 依赖审计）
- **约束遵守**: 零 Schema 变更 · 不修改生产 DB · 不 migrate · 不 db push · 不直接改 main · 最小修改

---

## 1. 实际修改文件

**修改（7 个，+143/−33 行）**

| 文件 | 改动 |
|---|---|
| `src/app/website/[domain]/page.tsx` | P0-1 五状态机 + Gate V1 robots（替换旧五维门） |
| `src/app/sitemap.ts` | P0-2/3 批量侧 Gate V1 接线（同构输入） |
| `src/app/layout.tsx` | P0-4 移除全局 canonical |
| `src/app/bulk/page.tsx` | P0-4/5 metaDescription + 自持 canonical |
| `src/app/docs/api/page.tsx` | P0-4/5 metaDescription + 自持 canonical + 显式 index |
| `src/app/report/page.tsx` | P0-4 自持 canonical |
| `src/lib/i18n.ts` | P0-1 processing 文案 + P0-5 bulk/apiDocs metaDescription（zh/en） |

**新增（7 个）**

| 文件 | 用途 |
|---|---|
| `src/lib/seo/low-value-domain.ts` | P0-6 低价值域名判定（sitemap 与 Gate 共用） |
| `src/lib/seo/index-gate.ts` | P0-2/3 Gate V1 唯一判定来源 + 页面侧输入提取 |
| `src/lib/seo/report-state.ts` | P0-1 资源状态机 + robots 顶层决策 |
| `src/app/website/[domain]/not-found.tsx` | P0-1 路由级 404（真实资源不存在态 + 分析引导） |
| `src/lib/seo/low-value-domain.test.ts` | 测试（14 例） |
| `src/lib/seo/index-gate.test.ts` | 测试（17 例） |
| `src/lib/seo/report-state.test.ts` | 测试（13 例） |

**保留未动**: `src/lib/seo/quality-gate.ts`（约束三：`evaluateWebsitePageGate` 仍被 `admin/seo` 使用，保留内部审核用途；`evaluateQualityGate` 仍被 entity/technology 页使用）、`src/app/robots.ts`（disallow 规则零改动）、`src/app/admin/seo/page.tsx`（零改动）。

---

## 2. 各 P0 修改内容

### P0-1 — Soft 404 修复（资源状态机）

- 新增 `report-state.ts`：`classifyWebsiteReportState`（渲染状态）+ `websiteRobotsPolicy`（索引决策）+ `hasDisplayableData`（宽松存在性，区别于 Gate 严格门槛）。
- 页面 `generateMetadata`：**默认 `noindex, follow`**（旧代码默认 `index`，非法域名会错误可索引）；normalized 后按 `websiteRobotsPolicy` 决定。
- 页面 body 五状态机：`null → notFound()`；`processing → 处理中提示 + 刷新链接`；`no-data → 引导重新分析`；`report → ReportPageContent`。
- 新增路由级 `not-found.tsx`：真实 404 + 分析引导（AnalyzeForm），`getDictionary` 防御性兜底（cookies 在 not-found 边界可能不可用），绝不落入 Next 裸错误页。
- `searchReport` 仅当 report 存在时装配（减少无效查询）。

### P0-2 — NOINDEX ≠ sitemap 冲突（统一判定）

- 单一判定函数 `classifyWebsiteIndexability`：页面 robots（经 `websiteRobotsPolicy`）与 sitemap.xml **共用同一函数、同构输入**。
- sitemap 由「全部 Target 进 sitemap」改为「仅 Gate 通过者进 sitemap」；输入从批量查询构造：最新调查状态（`investigation.findMany` orderBy startedAt desc 取首条）、调查次数（`groupBy _count`）、DNS/IP/Tech 实际数据存在性（`fact.findMany` + 与 `report.ts` 同源的三形状解析器）。

### P0-3 — Simple Index Gate V1

- 门槛（全部满足）：① 非低价值域名 ② 最新调查状态 ∈ {completed, partial} ③ 实际数据完整度 ≥ 2/3（DNS/IP/Technology，基于真实 fact 数据，绝不由 status 推断）。
- 条件（任一）：A `HOT_DOMAINS` 白名单（22 条 curated）/ B `investigationCount ≥ 3`（真实 count）/ C `MANUALLY_APPROVED_DOMAINS`（curated，初始空）。
- 零 Schema 变更：复用 Investigation.status、investigationCount（report 实时 count）、Snapshot/Fact 实际数据。
- 门槛 3 先于门槛 1/2 判定；门槛 2 先于条件层（热门域名数据不足也一票否决——测试覆盖）。

### P0-4 — canonical 自持

- `layout.tsx` 移除全局 `alternates: { canonical: SITE_URL }`（修复 /bulk、/docs/api、/report 错误继承首页 canonical 的问题）。
- 每个页面自行声明：`/bulk → /bulk`、`/docs/api → /docs/api`、`/report → /report`；无声明页面（login/dashboard/search 等）正确地**不输出** canonical。
- 验证：全仓 grep，layout 0 alternates；公开页面全部自持（见 §6）。

### P0-5 — metadata 独立

- `bulk`：新增 `metaDescription`（zh/en）+ canonical（保持 noindex, follow——登录功能页）。
- `apiDocs`：新增 `metaDescription`（zh/en）+ canonical + **显式 `index, follow`**（公开文档）。
- `report`：新增 canonical（保持 noindex, follow——动态分页且含未过 Gate 目标）。
- 无关键词堆砌；description 均为一句话业务描述。

### P0-6 — sitemap 低价值域名过滤

- `low-value-domain.ts`：① RFC 2606/6761 保留段（`example`/`invalid`/`localhost` 段级匹配，不误伤 `notexample.com`）② `.test` TLD ③ CDN 基础设施后缀（`.tiles.virtualearth.net` 等 8 条，裸域与子域双匹配）④ curated 名单（初始空）。
- 规则保守可维护：任何新增条目必须注释来源；真实站点零误伤（测试断言）。

---

## 3. P0-1 最终资源状态矩阵

| Target | 最新 Investigation | 可展示数据 | 页面 HTTP | robots | sitemap |
|---|---|---|---|---|---|
| **不存在** | — | — | **404**（路由级 not-found + 引导） | noindex | — |
| 存在 | running | 无 | 200 · processing 提示 | noindex | — |
| 存在 | running | 有（历史数据） | 200 · 报告 | noindex（stale-or-processing） | — |
| 存在 | failed | 无 | 200 · 引导重分析 | noindex（no-data） | — |
| 存在 | failed | 有（历史报告） | 200 · 报告 | noindex（stale-or-processing） | — |
| 存在 | completed/partial | 无 | 200 · 引导重分析 | noindex（no-data） | — |
| 存在 | completed/partial | 有且 Gate V1 过 | 200 · 报告 | **index, follow** | **✓** |
| 存在 | completed/partial | 有但 Gate V1 拒 | 200 · 报告 | noindex | — |

核心：**404 = 资源不存在**；分析失败/数据不足/低质量一律保留资源语义（200 noindex + 对应引导）。

## 4. Index Gate V1 最终逻辑

```
classifyWebsiteIndexability(input)  // input: {domain, investigationStatus, investigationCount, hasDns, hasIps, hasTechnology}
  ├─ 门槛3: isLowValueDomain(domain)                    → ✗ low-value-domain
  ├─ 门槛1: status ∉ {completed, partial}               → ✗ not-analyzed
  ├─ 门槛2: dataAspects = dns+ips+tech 实际非空项数 < 2  → ✗ insufficient-data
  └─ 条件层（任一）:
       A: domain ∈ HOT_DOMAINS                           → ✓ indexed
       B: investigationCount ≥ 3                         → ✓ indexed
       C: domain ∈ MANUALLY_APPROVED_DOMAINS             → ✓ indexed
       （全不满足）                                      → ✗ below-activity-threshold
```

页面侧输入：`gateInputFromReport(report)`（已装配 report 提取，零新增查询；DNS = records 或 nameservers 非空，IP = resolved 非空，Tech = 顶层 technology 非空）。
Sitemap 侧输入：批量查询构造同构字段（§2 P0-2）。

## 5. sitemap 与 robots 单一事实来源验证

- `classifyWebsiteIndexability` 生产调用点仅 2 处（grep 实证）：`src/app/sitemap.ts:91`（批量）与 `src/app/report-state.ts:75`（页面 robots，经 `websiteRobotsPolicy`）。两处输入同构。
- 页面 robots 与 sitemap 对同一 Target 必然同判 → **NOINDEX 绝不进 sitemap** 成立。
- `robots.ts` 零 diff（disallow 规则不变）。
- `quality-gate.ts` 完全退出 public robots/sitemap 链路（仅 admin/seo + entity 内部页使用）。

## 6. canonical 修复验证（grep 实证）

- `layout.tsx`：`alternates` 出现次数 = 0。
- 公开页面 canonical 矩阵：`/`（SITE_URL）、`/(seo)/[...slug]`、`/asn/[asn]`、`/bulk`、`/certificate/[fingerprint]`、`/docs/api`、`/ip/[ip]`、`/organization/[slug]`、`/report`、`/technology/[slug]`、`/website/[domain]` — 全部自持。
- 无 canonical 页面（`/login`、`/register`、`/dashboard`、`/search`、`/admin/*`）正确地不输出标签（无继承）。
- 修复目标验证：`/bulk` 不再输出 `https://siteintel.cc/` canonical；`/docs/api`、`/report` 同理。

## 7. metadata 修复验证

- `/bulk`：title（既有）+ description（新增 zh/en）+ canonical `/bulk` + robots noindex,follow ✓
- `/docs/api`：title（既有）+ description（新增 zh/en）+ canonical `/docs/api` + robots **index,follow 显式** ✓（此前无 robots 声明依赖 layout 继承）
- `/report`：title/description（既有）+ canonical `/report` + robots noindex,follow ✓
- i18n 类型定义与 zh/en 值三处同步（bulk.metaDescription / apiDocs.metaDescription / report.processing*），无关键词堆砌。

## 8. 低价值域名过滤测试（P0-6）

`low-value-domain.test.ts` 14 例，覆盖：
- 保留段段级匹配（`example.com`、`sub.example.org`、`foo.localhost`）✓
- `.test` TLD（`foo.test`、裸 `test`）✓
- **不误伤**：`exampletron.com`、`notexample.com`、`examplehosting.net` ✓
- CDN 后缀：`r1.tiles.virtualearth.net`（2026-08-16 生产污染源）+ 裸域 `tiles.virtualearth.net`（测试抓出裸域漏拦 bug 并修复）+ 其余 7 条后缀 ✓
- 真实站点通过：`siteintel.cc`、`baidu.com`、`github.com`、`www.microsoft.com` ✓
- 空输入 = 低价值 ✓

## 9. Build / Test 结果

| 检查 | 结果 |
|---|---|
| `vitest run`（全量 15 文件 226 例） | **226/226 PASS**（含新增 37 例 SEO 测试） |
| `tsc --noEmit` | **PASS**（0 错误） |
| `next build`（production） | **PASS** — 全路由编译，/website/[domain]、/sitemap.xml、/bulk、/docs/api、/report 均 ƒ dynamic ✓ |
| `eslint .` | 本分支文件 **0 error**；7 个 error 全部位于未改动文件（admin/data-quality、api-keys-manager、notify-settings、progress-tracker、relationship-graph） |

**测试抓出并修复的真实 bug**：
1. `report-state.ts` 曾读 `infrastructure.technology`（真实类型是 report 顶层 `technology`）→ 运行时会抛 `undefined.length`，一旦部署即打挂 /website 页 robots → 已修复。
2. `isLowValueDomain` 对裸域 `tiles.virtualearth.net` 漏拦（带前导点后缀的 `endsWith` 匹配不到裸域本身）→ 已修复（`suffix.slice(1)` 精确匹配裸域）。

**lint 7 错误说明（环境漂移，非本分支回归）**：基线验证时服务器安装的 eslint-plugin-react-hooks 为旧版本；本次全新 `npm install` 解析到 `eslint-config-next ^16.3.0` 依赖的 react-hooks 7.1.1，新增 React Compiler 规则（"Calling setState synchronously within an effect" / "Cannot call impure function during render"）触发既有代码报错。证据：7 个错误文件在 `git status` 中均为未修改（基线代码），本分支改动文件 lint 干净。不修改 package.json（最小修改原则），上线服务器用既有 node_modules 不受影响。

## 10. Git diff 摘要

- 7 修改 + 7 新增，+143/−33 行；无删除文件、无 renames。
- 新增文件：3 个 lib 模块 + 3 个测试 + 1 个路由级 not-found。
- 修改文件 diff 逐文件审查完成；关键行为变化：website 页默认 robots 从 `index` → `noindex`（防御方向），sitemap 从全量 Target → Gate 过滤。
- **Secret scan**：`git diff` 全文 grep `154.204.176.66 | :3003 | :3001 | wwwroot | password | api_key | token | secret` → **零匹配**。无 .env、无密钥、无私服信息。
- `package-lock.json`（npm install 副产物）已删除，仓库保持基线无 lockfile 状态。`node_modules/` 不入库（gitignore）。

## 11. 风险与未解决问题

1. **Gate V1 上线后 sitemap 收录量预计大幅收缩**：282 个网站中仅满足「completed/partial + 数据 ≥2 项 + 白名单/≥3 次/人工」者进入。这是约束二要求的保守方向；观察期后可调整 `MIN_INVESTIGATIONS`/`MIN_DATA_ASPECTS` 常量（两处均已导出）。
2. **HOT_DOMAINS 是 curated 常量**：22 条白名单之外的知名站点（如各政府/教育域名）在低活动度下暂不可索引——需要时经人工审核（MANUALLY_APPROVED_DOMAINS）或提高活动度进入。
3. **i18n fallback**：not-found 边界 cookies 不可用时中文默认文案兜底（代码注释已说明）。
4. **lint 环境漂移**（见 §9）：上线服务器的既有依赖不受影响；CI/新环境需锁 react-hooks 版本或更新既有代码（不在本阶段范围）。
5. **未部署、未 merge、未 push**：等待验收指令。建议验收流程：Code Review → 服务器分支拉取 → 测试站（test.ipcesu.com 同服务器端口 3001 为 SiteIntel 测试实例，或 3003 为生产——按既有运维流程处理）→ 生产部署 + sitemap 提交。
6. **Sitemap 提交 Google Search Console**：Gate 收缩后新 sitemap 需重新提交并观察索引行为（P0-2 冲突的最终验证）。
