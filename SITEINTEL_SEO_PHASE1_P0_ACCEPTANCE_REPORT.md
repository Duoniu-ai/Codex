# SITEINTEL SEO PHASE 1 — P0 POST-IMPLEMENTATION ACCEPTANCE REPORT

- **日期**: 2026-08-17
- **审计性质**: 只读验收（零代码修改、零 commit、零 push、零 deploy、零 DB 操作）
- **审计对象**: 分支 `seo/phase-1-p0`（基于 `main` @ `b52ec84`）的 P0 修改（7 modified + 7 added，+143/−33）
- **对照基线**: 服务器生产 HEAD `1bdfcb4`（本地同步副本 `C:\Users\deepo\siteintel`）+ GitHub Baseline `main @ b52ec84`

---

## 1. Git Branch / Commit

| 项 | 值 |
|---|---|
| 工作分支 | `seo/phase-1-p0`（未 merge、未 push） |
| Base | `main @ b52ec84`（GitHub Baseline，tag production-baseline-2026-08-17） |
| 服务器基线 | `1bdfcb4`（production HEAD，未动） |
| 工作区 | 7 modified + 7 untracked（本次新增），+143/−33 行 |
| 本审计 | 未创建/修改任何文件（报告除外）；未执行 git 写操作 |

## 2. 代码基线验证（服务器 1bdfcb4 vs GitHub b52ec84）

**方法**: 本地基线副本 `C:\Users\deepo\siteintel`（HEAD 1bdfcb4 实证）与 `git show main:<path>`（b52ec84）逐文件字节级 + `--strip-trailing-cr` 内容级对比。

**结果**:

| 文件 | 结果 |
|---|---|
| 本次修改的 7 个文件（bulk/docs/api/layout/report/sitemap/website[domain]/i18n） | **内容一致**（4 个文件仅 CRLF 行尾符差异——基线 Windows 同步引入，不影响 TS 编译；3 个完全一致） |
| 抽查：report.ts / schema.prisma / quality-gate.ts / registry.ts | 内容一致 |
| 本次新增的 7 个文件 | 基线侧不存在（预期） |

**结论**: 服务器基线 1bdfcb4 与 GitHub main b52ec84 **不存在业务代码漂移**。本次 P0 修改基于真实生产基线。✅

## 3. P0-1 状态机验收

**Prisma 真实关系（schema.prisma 实证）**:
- `Target` 1:N → `Investigation`（onDelete Cascade）；`Investigation.status: String @default("running")`（running|completed|partial|failed，含 `error` 字段）
- `Target` 1:N → `Snapshot`（type: dns|ip|ssl|http|technology|metadata|infrastructure|full）
- `Target` **无** count 列 —— `investigationCount` 来自 `assembleReport` 的 `prisma.investigation.count()` 实时查询（report.ts:25）
- `assembleReport` 返回 `null` **仅当 `prisma.target.findUnique` 无结果**（report.ts:22）—— 与状态机语义严格一致

**状态机逐项验证（代码 + 37 个新增测试）**:

| 状态 | 代码路径 | HTTP | Robots | 验收 |
|---|---|---|---|---|
| Target 不存在 | `classifyWebsiteReportState(null) === null` → `notFound()` | **404** | noindex（not-found 页继承默认） | ✅ |
| running 无数据 | `"processing"` 分支 | 200 | noindex | ✅ |
| running 有历史数据 | `hasDisplayableData` true → `"report"` | 200 | noindex（stale-or-processing） | ✅ |
| failed 无数据 | `"no-data"` 分支 | 200 | noindex | ✅ |
| failed 有历史数据 | `"report"` + ReportPageContent | 200 | noindex（stale-or-processing） | ✅ |
| completed/partial 无数据 | `"no-data"` 分支 | 200 | noindex（no-data） | ✅ |
| completed/partial 有数据 | `"report"` + Gate V1 | 200 | Gate 决定 | ✅ |

**专项确认**:
1. ✅ **failed 不会错误进入 404**：`classifyWebsiteReportState` 仅 `null` 触发 `notFound()`；failed → no-data 或 report，均非 null（测试覆盖 failed×2 路径）
2. ✅ **历史报告不因最新失败丢失**：failed + 有可展示数据 → `"report"` 态渲染完整报告（hasDisplayableData 是宽松存在性判定，与 Gate 严格 2/3 门槛分层）
3. ✅ **not-found.tsx 不误渲染其他错误**：App Router 中路由级 not-found 仅由 `notFound()` 调用触发；全仓无 `error.tsx`（默认 error 边界处理运行时异常，与 404 边界分离）；not-found.tsx 内部 `getDictionary` 抛错有 catch 兜底（cookies 不可用），绝无二次抛出
4. ✅ **真 HTTP 404**：`notFound()` + 路由级 not-found.tsx 是 Next.js App Router 的标准 404 机制（HTTP 状态码 404 + 自定义页面），非视觉 404
5. ✅ **旧行为对比**：基线中不存在域名 = 200+noindex 空态（soft 404），本次已修复为真 404

## 4. Index Gate V1 调用链

**唯一事实来源确认**（全仓 grep 实证）:
- `classifyWebsiteIndexability` 生产调用点 **仅 2 处**：`sitemap.ts:91`（批量）+ `report-state.ts:75`（页面 robots，经 `websiteRobotsPolicy`）
- `/website/[domain]` 页面 metadata robots（`website/[domain]/page.tsx:30-31`）与 sitemap **同一函数、同构输入** → 同一 Target 必然同判

**旧 quality-gate 隔离确认**:
- `evaluateWebsitePageGate`（quality-gate.ts 页级门）唯一调用点 = `admin/seo/page.tsx:46`（内部审核页）—— **完全退出 public 链路** ✅
- `evaluateQualityGate`（registry 五维门）调用点 = `entity.ts`×4（ip/asn/cert/org 页）+ `technology.ts`×1 + quality-gate.ts 自身（admin 内部）—— **不再参与任何 website 页 robots/sitemap** ✅
- `robots.ts` 零 diff；admin 页自身 noindex,follow ✅

**字段真实性验证（fact 形状与 Prisma 类型对照）**:

| 数据 | report.ts 解析（页面侧） | sitemap.ts 解析（批量侧） | 写入端（facts.ts/provider） | 一致 |
|---|---|---|---|---|
| DNS | `dnsAspect["records"]` 数组 + `["nameservers"]` | `obj["records"]` 数组非空 | `{records, nameservers, hasA, hasCname}`（providers/dns.ts） | ✅ |
| IP | resolved_ips **tuple 数组**（factList） | `Array.isArray && length>0` | 同 tuple 数组（pipeline 采集器） | ✅ |
| Technology | `value["tech"]` 数组 | `obj["tech"]` 数组非空 | `{tech: [...]}`（fingerprint 引擎） | ✅ |

无错误字段路径、无 null 判断错误、无 array/object 类型误判。partial 状态仅作前置门槛（`!== "completed" && !== "partial"` 即拒），**绝不自动通过**（测试覆盖）。

## 5. Sitemap 一致性

**唯一条件**: `classifyWebsiteIndexability(input).indexable === true`（sitemap.ts:91 过滤后生成 URL）—— 无第二条路径。

**PASS CASES（进入 sitemap）**:
- github.com（completed + 数据 ≥2 + 白名单 A，测试实证）
- acme-corp.com（completed + DNS+IP 2 项 + 调查 ≥3 次，条件 B，测试实证）
- 任意 completed/partial + 数据 ≥2 + 白名单/≥3 次/人工审核（代码推演）

**BLOCKED CASES（不进 sitemap）**:
- example.com（低价值门槛 3，即使全数据+99 次，测试实证）
- r1.tiles.virtualearth.net / tiles.virtualearth.net（CDN 后缀，测试实证）
- running/failed/null 状态（门槛 1，测试实证）
- completed 但数据 <2 项（门槛 2，测试实证）
- completed + 2 项数据 + 调查 1 次 + 非白名单（below-activity-threshold，测试实证）
- 私网/回环 IP 页（isBlockedAddress，既有逻辑）；AS0（sentinel，既有逻辑）

**专项确认**:
1. ✅ noindex 页面绝不进 sitemap（同函数同输入，结构保证）
2. ✅ sitemap 不绕过 Gate（过滤在 URL 生成之前）
3. ✅ example.com / Bing tiles CDN 排除（测试实证）
4. ✅ CDN 子域不会因白名单误入（低价值判定**先于**白名单执行；白名单 22 条均为主流站点，无基础设施域名）
5. ✅ 正常域名不误伤（测试实证 openai/baidu/github 通过；`notexample.com`、`examplehosting.net` 段级匹配不命中）

## 6. Canonical 全局影响检查

- `layout.tsx`：`alternates` 出现次数 = **0**（grep 实证）→ 全局 canonical 继承已根除
- 修复目标：`/bulk → /bulk`、`/docs/api → /docs/api`、`/report → /report`（代码实证）
- **全局生成metadata扫描**：所有 indexable 页面 canonical 自持矩阵：

| 页面 | canonical | robots | 验收 |
|---|---|---|---|
| /（首页） | SITE_URL（既有） | index | ✅ |
| 19 条 registry 静态页（/website-intelligence 等） | `/页面`（(seo)/[...slug]:55） | index（继承 layout） | ✅ |
| /website/[domain] | `/website/{domain}` | Gate V1 | ✅ |
| /ip//asn//certificate//organization//technology | `url`（各自页） | registry 门 | ✅ |
| /docs/api | `/docs/api`（本次） | index（本次显式） | ✅ |
| /bulk、/report | `/bulk`、`/report`（本次） | noindex | ✅ |

- **无 canonical 页面**（login/register/dashboard/search/claim/analyze/admin×2）= 全部 noindex 页 —— 正确地不输出 canonical，无意外失去 canonical 的 indexable 页面 ✅

## 7. Metadata 验收

| 页面 | title | description | canonical | robots | 验收 |
|---|---|---|---|---|---|
| /bulk | dict.bulk.title（既有） | **dict.bulk.metaDescription（本次，zh/en）** | /bulk（本次） | noindex,follow | ✅ |
| /docs/api | dict.apiDocs.title（既有） | **dict.apiDocs.metaDescription（本次，zh/en）** | /docs/api（本次） | **index,follow 显式**（本次；基线无声明依赖 layout 继承） | ✅ |
| /report | dict.metadata.reportsTitle | dict.metadata.reportsDesc（既有） | /report（本次） | noindex,follow | ✅ |

- 双语完整性：4 个 metaDescription 值 + 2 处类型声明全部存在（i18n.ts 实证），无 undefined、无 fallback 空串、**无重复首页 description**（基线 /docs/api、/bulk 复用首页 description 的缺陷已消除）
- locale 机制无改动（getDictionary 既有 normalizeLocale，字典值全字符串）—— 无 locale 错误
- 无关键词堆砌（均为一句业务描述）

## 8. Low-value Filter 验收

- 逻辑：① RFC 2606/6761 保留段（`parts.includes` **段级精确匹配**，非 substring）② `.test` TLD ③ CDN 基础设施后缀（8 条，带前导点，裸域 `suffix.slice(1)` 精确匹配）④ curated 名单（空）
- **过度过滤检查**（测试实证）：openai.com ✅ / baidu.com ✅ / github.com ✅ 均不被过滤；`exampletron.com`、`notexample.com`、`examplehosting.net` 不误伤（"example" 仅作为独立段拦截）
- 误伤风险分析：CDN 后缀均带前导点（`endWith(".cloudfront.net")` 不命中 `notcloudfront.net`）；curated 名单为空；白名单 22 条均无低价值域名
- 无模糊 substring 匹配 —— 无商业域名误杀风险 ✅

## 9. Test / Typecheck / Build（本次审计复验）

| 检查 | 结果 |
|---|---|
| `vitest run` 全量 | **226/226 PASS**（15 文件；含本次新增 37 例 SEO 测试） |
| `tsc --noEmit` | **0 错误** |
| `next build`（production，DATABASE_URL/NEXT_PUBLIC_SITE_URL 内联） | **PASS**（Compiled successfully / exit 0；/website/[domain]、/sitemap.xml、/bulk、/docs/api、/report 全部编译） |

## 10. Lint / CI 风险

| 项 | 结论 |
|---|---|
| 7 个 lint error 文件 | admin/data-quality、api-keys-manager、notify-settings、progress-tracker、relationship-graph×2、discovery/candidates —— **git status 实证全部未修改**（与本次分支无关） |
| 7 个 error 性质 | react-hooks 7.1.1 新 React Compiler 规则（setState-in-effect ×4、impure-in-render ×1、cannot-modify ×1）+ 基线 prefer-const ×1 —— 本次全新 npm install 解析到新版本规则所致（服务器既有 node_modules 不受影响） |
| 本分支文件 lint | 0 error（1 个 warning 已随实现阶段修复后清零） |
| 是否有 CI | **无 `.github/workflows`** —— 无 CI pipeline 会执行 lint，无 CI 失败风险 |
| next build 是否执行 lint | build 已实测 PASS（无论 lint 静态报告如何，生产 build 不失败；服务器部署流程 = SSH 直传 → pnpm build → systemctl restart，不经 lint 门禁） |
| **BLOCKER 判定** | **非 BLOCKER**（无 CI 失败路径、build 不失败、文件与分支无关、服务器依赖不受影响） |

## 11. 发现的问题（只记录，未修改）

| # | 级别 | 描述 | 归属 |
|---|---|---|---|
| 1 | P3 | **entity 类页面（ip/asn/cert/org/tech）的 robots（registry 五维门 ≥80）与 sitemap（count≥3/2）非同一判定**。窗口：count 达标但五维 <80 的实体（如托管 3 站但无证据/历史/关联的 IP → 约 45 分）→ 页面 noindex + sitemap 收录。基线既有设计，本次 P0 未引入未加剧（P0-2 范围=website 页） | 基线遗留，建议纳入后续任务书（entity 页同源化） |
| 2 | P3 | `/docs/api` 本次显式 index,follow 但未登记 registry/sitemap（基线未登记）→ 可索引但 sitemap 无条目。非冲突（sitemap 是建议性收录），可选后续登记 | 本次引入的决策，建议后续定夺 |
| 3 | P3 | `resolved_ips` 畸形 tuple（tuple[0] 非 string）时 sitemap `hasIps=true`（数组非空）而页面 `ips=[]`（tuple 过滤）—— 理论边角，写入端受采集器形状约束，当前生产无此形态 | 理论边界，无需处理 |

**未发现**: 无 P0/P1/P2 级问题；本分支修改零缺陷。

## 12. 最终结论

# ✅ PASS WITH MODIFICATIONS

**依据**:
- **PASS 部分**: 本分支全部 P0 修改（P0-1 状态机 / P0-2·3 Gate V1 / P0-4 canonical / P0-5 metadata / P0-6 低价值过滤）逐项验收通过；基线与生产零漂移；226/226 测试、typecheck、production build 全绿；lint 7 error 与本次无关且非 BLOCKER。
- **MODIFICATIONS 部分**（均为既有或决策类记录项，**不阻塞本分支合入**）: ① entity 类页面 robots/sitemap 非同一判定（基线遗留，与 P0-2 同型，建议后续任务书覆盖）② /docs/api 建议纳入 sitemap/registry 的决策 ③ resolved_ips 理论边界。
- **合入建议**: 本分支可按既有流程 merge → 服务器拉取 → 测试站验证 → 生产部署（部署后重新提交 sitemap 并观察收录）。entity 页同源化作为独立后续任务处理。

**验收完成，等待下一步指令。**
