# SITEINTEL SEO PLAN AUDIT REPORT

SiteIntel SEO 四份规划文件审计报告 —— 规划 vs 当前真实代码 vs 线上真实行为

审计时间：2026-08-17
审计方式：只读代码审计 + 线上实测（siteintel.cc 全 URL 实测）+ 数据库结构审计（prisma/schema.prisma + 2026-08-16 生产数据快照）
审计范围文件（实际位置：`C:\Users\deepo\Downloads\`，并非项目根目录）：

1. `SITEINTEL_SEO_MASTER_PLAN_V2.md`（V2.0，392 行）
2. `SITEINTEL_SEO_EXECUTION_ROADMAP.md`（V1.0，228 行）
3. `SITEINTEL_TECHNICAL_SEO_IMPLEMENTATION.md`（V1.0，340 行）
4. `SITEINTEL_SEO_ACCEPTANCE_CHECKLIST.md`（V1.0，292 行）

审计项目：`C:\Users\deepo\siteintel`（本地代码快照，**非 git 仓库**）+ 线上 siteintel.cc

> ⚠️ 重大前置发现：**本地代码落后于线上部署版本**。实测发现 4 处差异（见 §4.10），线上行为以实测为准，本地代码仅供参考。本报告所有"线上实测"结论均为 2026-08-17 实际请求结果。

---

# 1. 四份文件总体审计结论

## 规划文件本身：PASS WITH MODIFICATIONS

四份文件的战略方向正确、与约 282 个网站的数据规模匹配、与 Next.js 16 App Router 技术栈匹配，绝大部分技术要求（generateMetadata、notFound()、app/sitemap.ts、robots.ts、JSON-LD、Breadcrumb）在当前项目中**都具备落地条件且大部分已经实现**。

但存在以下问题，必须修正规划或修正实现后才能继续执行：

1. **一个核心内部冲突**：MASTER §6/§7 规定"Simple Index Gate V1、当前不建立复杂评分模型、只使用 INDEX/NOINDEX"，但项目已实现的是一套五维 100 分评分模型（`src/lib/seo/quality-gate.ts` + `registry.ts`，≥80 INDEX / 60-79 复核 / <60 NOINDEX）。规划的"三关"（热门/分析次数≥3/人工审核）在代码中完全不存在。
2. **文件间口径不一致**：对"分析记录不存在的页面"该返回 404 还是 200+noindex，四份文件自身说法矛盾（TECH §三允许 200+noindex，TECH §四/§五和 CHECKLIST §四要求 404）。
3. **实体页面策略被超前执行**：33 个 `/technology/:slug` 页面已上线并 INDEX（门槛为 ≥3 个关联网站），违反 MASTER §8 的延迟策略（<1,000 网站不生成实体独立页面）和 ROADMAP P2 的触发条件（≥1,000 网站）。
4. **规划引用的页面清单与真实路由不一致**：MASTER 第一层品牌页列出 `/about`、`/docs`、`/guide`，线上三者全部 404。

## 线上现状按 ACCEPTANCE CHECKLIST 严格口径验收：FAIL

按 CHECKLIST §十二的规则（"任何 P0 项失败 = FAIL"）对线上实测：

- 不存在域名 `/website/nonexistent-test-domain-123456789.com` → **HTTP 200**（预期 404）→ P0 失败
- Sitemap 中 26 个 Website URL 有 **10 个实测为 noindex**（预期 0 个）→ P0 失败
- `/bulk`、`/docs/api`、`/report` 的 Canonical 全部指向首页（预期指向自身）→ P0 失败
- `/docs/api` 复用首页 Description（预期独立）→ P0 失败

**结论：规划文件 = PASS WITH MODIFICATIONS；线上现状按清单口径 = FAIL（多项 P0 未过，需先修复再验收）。**

---

# 2. 四份文件之间的问题

## 2.1 冲突

| # | 冲突 | 涉及文件 | 详情 |
|---|------|---------|------|
| C1 | **INDEX Gate 模型冲突**：MASTER §6/§7 要求 Simple Index Gate V1 三关、禁止五维评分；代码注释引用的"SEO 文档 §7"实现的是五维 100 分评分 | MASTER §6/§7 vs 实现 | 执行时依据的文档版本与当前 4 份文件不一致，或执行偏离了文件。ROADMAP 任务 3 的 `isIndexable(domain)` 伪代码（analysisSuccess/coreDataCount/isLowQuality/isPopular/analysisCount/isManuallyApproved）与代码实现（评分模型）完全不符 |
| C2 | **404 口径冲突**：TECH §三说"数据不存在：返回 404，**或**根据真实业务状态返回 200+noindex"；TECH §四/§五与 CHECKLIST §四明确要求不存在域名 → `notFound()` → 真实 404；ROADMAP 任务 2 禁止"200+默认页面" | TECH §三 vs TECH §四/五、CHECKLIST §四、ROADMAP P0-任务2 | 文件自身给了两条互斥的路，未定版。实现走的是"200+noindex+暂无报告页"路线 |
| C3 | **实体页面阶段冲突**：MASTER §8 规定 <1,000 网站"不大规模生成实体独立页面"、成长阶段 Technology 门槛 50；ROADMAP P2 规定 ≥1,000 才试点；实现已上线 33 个 Technology 页（门槛 3）且 INDEX | MASTER §8、ROADMAP P2 vs 实现 | 实现超前于规划两个阶段 |
| C4 | **品牌页清单冲突**：MASTER §五 第一层列 `/`、`/about`、`/docs`、`/guide` 为核心页面；ROADMAP/CHECKLIST 的核心页面清单是 `/`、`/search`、`/bulk`、`/report`、`/docs/api` 等。两套清单互相不覆盖 | MASTER §五 vs ROADMAP P0-任务1、CHECKLIST §二 | 线上 `/about`、`/docs`、`/guide` 全部 404；`/search-intelligence` 只出现在 ROADMAP 清单 |

## 2.2 重复

| # | 重复内容 | 出现位置 | 评价 |
|---|---------|---------|------|
| D1 | "NOINDEX URL 禁止进入 Sitemap" | ROADMAP 任务 4、TECH §八、CHECKLIST §六 | 三处口径一致，重复无害 |
| D2 | "禁止复用首页 Metadata/Description" | ROADMAP 任务 1、TECH §十六、CHECKLIST §二 | 三处一致，重复无害 |
| D3 | 404/Soft 404 规则 | ROADMAP 任务 2、TECH §四、§五、CHECKLIST §四 | **口径不完全一致**（见 C2），重复且互相矛盾 |
| D4 | INDEX Gate 规则 | MASTER §6、ROADMAP 任务 3（伪代码）、TECH §七（robots 映射） | MASTER 与 ROADMAP 一致，TECH 只是引用，无冲突 |
| D5 | 禁止虚构结构化数据（Review/Rating/FAQ） | TECH §十、CHECKLIST §八 | 一致，重复无害 |

## 2.3 缺失

| # | 缺失 | 影响 |
|---|------|------|
| M1 | 未定义"分析失败页"（Investigation failed/partial）的处理：404 还是 200+noindex。CHECKLIST §五写"NOINDEX **或** 404"二选一未定版。当前实现：目标存在即渲染 200 页面（含失败目标），按 gate 决定 robots | 失败/空数据页面策略无标准，验收无依据 |
| M2 | 未定义"预置热门网站"的维护机制：存在哪（DB 字段 / 代码列表）、谁维护、何时更新 | Gate 3 的 A 条件无法落地（当前确实没落地） |
| M3 | 未定义"人工审核标记"的操作入口与存储 | Gate 3 的 C 条件无法落地（当前确实没落地） |
| M4 | 未定义 Sitemap 与页面 robots 不一致时的冲突解决规则。实现代码注释写"页面 robots 是权威"，但四份文件都禁止 NOINDEX 进 Sitemap——二者矛盾无解决路径 | 现状就是该矛盾的具体体现（10/26 不一致） |
| M5 | TECH §九推荐 ISR（revalidate 3600/86400），但未说明与当前全站 `force-dynamic` 渲染模型的冲突如何裁决 | ISR 无法直接叠加在 force-dynamic 上，需要改渲染模型，规划未给出取舍 |
| M6 | 四份文件均未涉及本地代码库与线上部署的版本同步/基线要求 | 本次实测已发现本地落后线上，无同步规范则后续执行会基于过期代码 |
| M7 | 站点实际是 zh/en 双语，但四份文件完全没有 hreflang/多语言 SEO 策略 | 双语内容（字典驱动）无对应 SEO 规范 |
| M8 | ROADMAP"本周必做 5 件事"第 5 项"SEO 自动验收脚本"在 TECH 与 CHECKLIST 中没有任何对应规范（CHECKLIST 是人工验收） | 该任务无验收标准 |

## 2.4 优先级问题

| # | 问题 | 详情 |
|---|------|------|
| PR1 | TECH §十四把 Sitemap 列为 P2、Meta Keywords 列为 P3，但 ROADMAP 把"重建 Sitemap"列为 P0 任务 4。同一份执行体系内 Sitemap 优先级不一致 | TECH §十四 vs ROADMAP P0 |
| PR2 | ROADMAP"本周必做"5 件事假设 P0 未动工；实际项目已完成了 P1/P2 级别的工作（Breadcrumb、内容页、实体页），而 P0 仍有 4 项未过。执行顺序已倒挂，ROADMAP 未反映真实进度 | 整体 |

## 2.5 阶段性问题

| # | 问题 | 详情 |
|---|------|------|
| ST1 | 实体页面（33 个 Technology 页）在 <1,000 阶段上线并 INDEX，违反 MASTER §8 与 ROADMAP P2 触发条件 | 超前两个阶段 |
| ST2 | 五维评分模型在 MASTER 明确"当前不建立复杂评分模型"的情况下被实现 | 超前于规划禁令 |
| ST3 | MASTER §10 Phase 1 要求"先修复技术 SEO"，但 Phase 1 的 P0 项（404、Sitemap 一致性、Canonical）至今未全部达标，而 Phase 2 内容已部分上线 | 阶段倒挂 |

---

# 3. 规划要求 vs 当前项目实际情况

状态取值：✅已真实实现 / 🟡部分实现 / ❌未实现 / ⛔当前架构不支持 / 🔧需要调整方案

## 3.1 SITEINTEL_SEO_MASTER_PLAN_V2.md

| 文件 | 要求 | 当前实际实现 | 状态 | 问题 | 是否需要修改 |
|------|------|-------------|------|------|-------------|
| MASTER §2 | 产品定位"Website Data Intelligence Platform"，非单点查询站 | 首页 H1"看懂任何一个网站"，7 层数据管线（pipeline.ts）真实存在 | ✅ | 无 | 否 |
| MASTER §五 第一层 | 品牌页 `/`、`/about`、`/docs`、`/guide` | `/` 存在；`/about`、`/docs`、`/guide` 实测全部 **404** | 🟡 | 规划列出的品牌页 3/4 不存在；实际品牌内容页是 `/website-intelligence`、`/how-it-works` 等 | 是（改规划清单或补页面） |
| MASTER §五 第二层 | 功能页 `/tools/*` 5 个 | 5 个 `/tools/*` 落地页存在（catch-all 渲染），实测 200 + index | ✅ | 内容由字典驱动、单页结构，非独立功能工具 | 否 |
| MASTER §五 第三层 | 内容 SEO（指南/方法文章） | `/guides/*` 3 页 + `/cases/*` 2 页已上线，实测 200 + index | ✅ | 内容来自字典模板，每个页面 5-7 个 H2 段落，非深度文章 | 否（内容深度后续加强） |
| MASTER §五 第四层 | `/website/:domain` 动态页 | 存在，SSR + generateMetadata + canonical + 308 www→apex | ✅ | 无 | 否 |
| MASTER §6 | Simple Index Gate V1 三关（分析成功+2 核心维度 / 排除低价值 / 热门或 ≥3 次或人工审核） | 实现的是**五维 100 分评分**（数据覆盖 25+证据 20+历史 15+情报 20+内容 20，≥80 INDEX）。三关中的"热门/≥3 次/人工审核"**完全未实现** | 🔧 | 实现与规划模型完全不同；规划明确禁止复杂评分 | 是（二选一，见 §7 R1） |
| MASTER §7 | 只用 INDEX/NOINDEX 两种状态 | 实现存在 `INDEX_AFTER_QUALITY_GATE` 中间态（60-79 分，映射为 noindex,follow） | 🔧 | 与"只用两种状态"冲突（中间态只影响内部语义，robots 输出仍是二值） | 是（并入 R1） |
| MASTER §8 | <1,000 网站不生成实体独立页面；Technology 门槛 50（成长阶段） | **33 个 `/technology/:slug` 页面已上线并 INDEX**，门槛为 ≥3 个关联网站 | 🔧 | 超前执行、门槛远低于规划（3 vs 50） | 是（见 §7 R2） |
| MASTER §9 | 关键词策略（首页品牌词、工具页功能词） | 首页 title"看懂任何一个网站"、工具页独立关键词，与规划方向一致 | ✅ | 无 | 否 |
| MASTER §10 Phase 1 | 技术 SEO 修复、TDK、Canonical、404、Robots、Sitemap | 部分完成：Robots ✓、动态 Metadata ✓；404/Soft404 ✗、Canonical（静态页）✗、Sitemap 一致性 ✗ | 🟡 | 见 P0 清单 | 否（继续执行） |
| MASTER §10 | Phase 1 不建五维评分、不建大规模实体页 | 两者都已上线 | 🔧 | 与规划禁令直接冲突 | 是（并入 R1/R2） |

## 3.2 SITEINTEL_SEO_EXECUTION_ROADMAP.md

| 文件 | 要求 | 当前实际实现 | 状态 | 问题 | 是否需要修改 |
|------|------|-------------|------|------|-------------|
| ROADMAP P0-任务1 | 全站 Metadata 审计（Title/Description/Canonical/Robots/OG 独立） | 动态页 ✓；`/bulk`、`/docs/api`、`/report` Canonical 实测全部指向首页；`/docs/api` Description 实测 = 首页 Description | 🟡 | 3 个静态页违反"Canonical 指向自身"与"禁止复用首页 Description" | 否（修复代码） |
| ROADMAP P0-任务2 | HTTP 状态码：不存在 → 404；禁止 200+默认页 | 不存在域名实测 **200 + noindex + "暂无报告"默认页**；非法参数（如 `invalid_domain!!`）正确 404 | ❌ | 与规划直接冲突（线上证据见 §4.6） | 否（修复代码） |
| ROADMAP P0-任务3 | `isIndexable(domain)`：analysisSuccess + coreDataCount≥2 + isLowQuality + isPopular/analysisCount≥3/isManuallyApproved | 实现为五维评分；isPopular、isManuallyApproved 无字段无逻辑；isLowQuality 无检测逻辑；analysisCount 可推导（investigationCount）但未被 gate 使用 | 🔧 | 规划伪代码与实现完全不符 | 是（见 §5） |
| ROADMAP P0-任务4 | Sitemap 只含 INDEX URL | 实测 sitemap 共 179 URL，其中 26 个 Website URL 里 **10 个页面 robots 为 noindex**（threads.net、whatsapp.com、instagram.com、facebook.com、ddooo.com、taobao.com、tencent.com、example.com、huawei.com、kugou.com） | ❌ | Sitemap 过滤条件（full 快照+events/insights）与页面 gate（≥80 分）不同源，系统性不一致 | 否（修复代码） |
| ROADMAP P0-任务5 | Robots 统一（robots.txt/robots.ts/robots meta） | robots.ts 存在且规则合理；线上 robots.txt = 应用规则 + Cloudflare 注入的 AI 爬虫规则；无规则冲突 | ✅ | 线上含本地没有的 `/claim/` disallow（本地落后线上的证据之一） | 否 |
| ROADMAP P1-任务6 | Breadcrumb（可见 + BreadcrumbList JSON-LD） | `/website/:domain` 详情页与全部 SEO 落地页均有可见 Breadcrumb + BreadcrumbList JSON-LD；`/technology/:slug` 也有 | ✅ | 无 | 否 |
| ROADMAP P1-任务7 | 真实内容 SEO | 3 guides + 2 cases 已上线 | ✅ | 内容为字典模板生成 | 否 |
| ROADMAP P1-任务8 | 模块级内部链接 | 详情页有 IP/DNS/Tech/关系/历史模块导航；related 网站上限 10；tech 页互链 | ✅ | 未发现 500+ 链接问题 | 否 |
| ROADMAP P2 | ≥1,000 网站后 Technology/ASN 试点 | **已提前上线**（33 个 tech 页） | 🔧 | 触发条件被跳过 | 是（并入 R2） |
| ROADMAP"本周必做" | 5 件事 | 1 ✗ 2 ✗ 3 🔧 4 ✗ 5 ✗（无自动验收脚本，仅有 admin 人工校验页） | ❌ | 与真实进度脱节 | 否（按新执行顺序 §8 重排） |

## 3.3 SITEINTEL_TECHNICAL_SEO_IMPLEMENTATION.md

| 文件 | 要求 | 当前实际实现 | 状态 | 问题 | 是否需要修改 |
|------|------|-------------|------|------|-------------|
| TECH §二 | 全部使用 Next.js Metadata API；动态页 generateMetadata | 所有页面均用 Metadata API；`/website/[domain]` 有 generateMetadata（title/description/canonical/OG/robots 动态生成） | ✅ | 无 | 否 |
| TECH §三 | 禁止动态页共享 Metadata；基于真实数据 | 每域名独立 title/description/canonical；实测 ✓ | ✅ | 无 | 否 |
| TECH §三 | 数据不存在 → 404 或 200+noindex | 实现 200+noindex+"暂无报告"页（本地 page.tsx:75-88） | 🟡 | 与 §四/§五 的 404 要求冲突（见 C2） | 是（定版口径，R4） |
| TECH §四 | 网站不存在/分析记录不存在/参数非法 → notFound() 真实 404 | 参数非法 → 404 ✓；域名合法但 Target 不存在 → **200** ✗ | 🟡 | 核心违规 | 否（修复代码） |
| TECH §五 | 测试 `/website/nonexistent-test-domain-123456.com` 预期 404 | 实测同型 URL → **HTTP 200**（size 81KB 默认页） | ❌ | 规划的软 404 检测直接失败 | 否（修复代码） |
| TECH §六 | Canonical 指向自身；禁止指向首页 | 动态页 ✓（`/website/openai.com` canonical 自身）；**`/bulk`、`/docs/api`、`/report` 实测 canonical = 首页** | 🟡 | 根因：layout.tsx 设置了 `alternates.canonical = SITE_URL`，未覆盖的页面全部继承首页 canonical | 否（修复代码） |
| TECH §六 | domain normalize（lowercase/去协议/去尾斜杠） | normalizeDomain + stripWww + www→apex 308 已实现 | ✅ | permanentRedirect 实测为 **308**（非规划写的 301；308 对 SEO 等价，属细节差异） | 否 |
| TECH §七 | robots：INDEX→index,follow；否则 noindex,follow | 动态页按 gate 映射实现，实测 ✓（openai.com=noindex,follow） | ✅ | 无（模型本身冲突见 R1） | 否 |
| TECH §八 | app/sitemap.ts；只含可 INDEX URL | 存在，动态生成 179 URL；**含 noindex URL（10/26 website 页实测）** | 🟡 | 与页面 gate 不同源 | 否（修复代码） |
| TECH §九 | ISR revalidate 3600/86400（数据非实时可用） | 全站 `force-dynamic`；实测响应头 `Cache-Control: private, no-cache, no-store`，**零缓存、无 ISR** | ❌ | 规划推荐 ISR，架构现状是每次请求全量查库组装。279 目标规模下性能无碍，但两者不匹配；ISR 需改变渲染模型 | 是（裁决保留 SSR 或改 ISR，R6） |
| TECH §十 | JSON-LD：首页 Organization+WebSite；工具页 WebApplication；文章 Article；详情 WebPage+BreadcrumbList | 首页实测含 Organization+WebSite；工具落地页 SoftwareApplication；guides/cases Article；详情页 WebPage；BreadcrumbList 全覆盖 | ✅ | 无 | 否 |
| TECH §十一 | JSON-LD 与可见 Breadcrumb 一致 | 两者同源渲染 | ✅ | 无 | 否 |
| TECH §十二 | 相关网站分页，禁 500+ 链接 | related 上限 10（lib/related.ts 默认 limit=10），无分页需求 | ✅ | 无 | 否 |
| TECH §十三 | 链接策略 | 符合 | ✅ | 无 | 否 |
| TECH §十四 | 优先级 P0 HTTP/Canonical/Robots/Index/Title/404 | 与 ROADMAP 的 P0 基本一致（Sitemap 优先级两处不同，见 PR1） | ✅ | 无 | 否 |
| TECH §十五 | 目录结构 app/sitemap.ts、app/robots.ts、website/[domain]/not-found.tsx；lib/seo/{metadata,canonical,indexability,structured-data,sitemap}.ts | app 层结构 ✓（`not-found.tsx` 在 `src/app/` 根级，不在 `website/[domain]/` 下，效果等价）；lib/seo 实际只有 3 个文件（registry/quality-gate/technology），canonical/structured-data/sitemap 逻辑内联在页面与 registry 中 | 🟡 | 建议结构未完全落地，但不影响功能 | 否（可后续整理） |
| TECH §十六 | 禁止：复用首页 Description / 不存在页 200 / NOINDEX 进 Sitemap / 虚假结构化数据 / 未判断质量直接 INDEX | **违反 2 项**：不存在页 200（实测）；NOINDEX 进 Sitemap（10/26 实测）。其余 3 项合规 | 🟡 | 同上 | 否（修复代码） |

## 3.4 SITEINTEL_SEO_ACCEPTANCE_CHECKLIST.md

| 文件 | 要求 | 当前实际实现 | 状态 | 问题 | 是否需要修改 |
|------|------|-------------|------|------|-------------|
| CHECKLIST §二 | 首页/各核心页 Title、Description 正确、独立 | 首页 ✓；`/bulk`、`/report`、`/docs/api` Title ✓；**`/docs/api` Description = 首页 Description**（实测） | 🟡 | `/docs/api` 违反"没有错误复用首页 Description" | 否（修复代码） |
| CHECKLIST §三 | Canonical 指向自身；`/bulk` 不指向首页；抽查 10 页 | **`/bulk`、`/docs/api`、`/report` 实测全部指向首页**；动态页 ✓；未发现循环 | ❌ | `/bulk` 直接违反清单明文要求 | 否（修复代码） |
| CHECKLIST §四 | `/` 200 ✓；`/website/openai.com` 200 ✓；不存在域名 404 | 前两项 ✓；不存在域名 **200** ✗ | ❌ | 清单验收会直接失败 | 否（修复代码） |
| CHECKLIST §五 | 四类测试：热门站 INDEX；普通站按规则；停放/测试域名 NOINDEX 且不入 Sitemap；失败 NOINDEX/404 且不入 Sitemap | 热门站：openai.com/facebook.com/instagram.com 实测 **noindex** ✗；停放/测试域名：**example.com 在 Sitemap** ✗；失败：无独立策略 ✗ | ❌ | 四类中三类不达标 | 否（修复代码 + 定版规则） |
| CHECKLIST §六 | Sitemap 无 NOINDEX URL | **10/26 website URL 实测 noindex** | ❌ | 系统性不一致 | 否（修复代码） |
| CHECKLIST §七 | robots.txt 可访问、Sitemap 地址正确、未错误禁止核心页 | 全部 ✓（robots.txt 200；Sitemap 行正确；核心页未禁） | ✅ | 无 | 否 |
| CHECKLIST §八 | 首页 Organization/WebSite；工具页结构化数据；BreadcrumbList | 实测 ✓ | ✅ | 无 | 否 |
| CHECKLIST §九 | H1 明确、与 Title 语义一致、页面有真实内容 | 动态页 H1=域名，与 title 一致 ✓；落地页 H1 与 title 一致 ✓ | ✅ | 无 | 否 |
| CHECKLIST §十 | Website 页可达 IP/Technology/DNS 模块 | 实测报告页含完整模块（openai.com 页含关系图、nameserver 等） | ✅ | 无 | 否 |
| CHECKLIST §十一 | SSR 含核心内容、不依赖浏览器 JS | 实测 SSR HTML 完整包含内容（报告 321KB HTML） | ✅ | 无 | 否 |
| CHECKLIST §十二 | 最终验收：任一 P0 失败 → FAIL | **多项 P0 失败 → FAIL** | ❌ | 不能出 PASS 结论 | 否（先修复） |
| CHECKLIST §十三 | 报告输出 `SITEINTEL_SEO_IMPLEMENTATION_REPORT.md` | 尚未产出（本文件是审计报告，非实施报告） | ❌ | 待修复完成后执行 | 否 |

---

# 4. 当前项目真实技术结构

以下全部为实测/实读结果，无猜测。

## 4.1 框架与版本

| 项 | 实际值 |
|---|---|
| Next.js | **16.3.0**（node_modules 实测版本，App Router） |
| React | 19.2.0 |
| TypeScript | 5.9.0 |
| 样式 | Tailwind CSS 4.3.0 |
| ORM | Prisma 6.16.0 + **PostgreSQL**（datasource provider=postgresql；生产为服务器本地 127.0.0.1:5432） |
| 包管理 | pnpm |
| 本地代码管理 | **无 git 仓库**（本地目录是代码快照） |
| 部署 | 154 服务器 systemd `siteintel`，端口 3003，域名 siteintel.cc，前面有 Cloudflare（robots.txt 实测含 Cloudflare Managed Content 注入块） |

## 4.2 路由结构（src/app/，App Router）

| 路由 | 文件 | 说明 |
|---|---|---|
| `/` | `page.tsx` | 首页，force-dynamic，实时统计 + 示例卡 |
| `/website/[domain]` | `website/[domain]/page.tsx` | **核心动态路由**，force-dynamic |
| `/report/[domain]` | `report/[domain]/page.tsx` | 旧路由，**308 永久跳转**到 `/website/[domain]`（实测） |
| `/technology/[slug]` | `technology/[slug]/page.tsx` | 技术实体页，force-dynamic，带独立 gate |
| `/analyze/[id]` | `analyze/[id]/page.tsx` | 进度页，noindex,nofollow |
| `/bulk` `/search` `/report` `/docs/api` `/dashboard` `/login` `/register` `/admin/seo` | 各独立目录 | `/search` 未登录 307 → /login（实测）；`/admin/seo` 是管理员 SEO 校验页（ADMIN_EMAILS） |
| `/website-intelligence`、`/tools/*`、`/search/*`、`/guides/*`、`/cases/*` 等 21 个 SEO 落地页 | `(seo)/[...slug]/page.tsx` | catch-all + SEO Page Registry（`src/lib/seo/registry.ts`）+ 字典内容（i18n.ts seoPages） |
| `/api/*` | `api/` 下约 19 个 route | API 路由 |

## 4.3 Metadata 实现方式

- 根布局 `layout.tsx`：`generateMetadata` 输出默认 title 模板（`%s | SiteIntel`）、默认 description、**`alternates.canonical = 首页 URL`**、robots index,follow。
- 动态页 `/website/[domain]`：`generateMetadata` 按域名生成绝对 title、独立 description、自身 canonical、按 gate 输出 `index, follow` 或 `noindex, follow`、OG/Twitter。
- SEO 落地页：catch-all 的 `generateMetadata` 从 Registry + 字典取 title/description/keywords/canonical；未注册 slug 返回 `robots: {index:false}` 且页面 notFound()（实测未知路径 404 + 自动 noindex meta）。
- 静态页：各自 `generateMetadata`。
- **未覆盖 canonical 的页面（/bulk、/docs/api、/report、/analyze/[id] 等）全部继承 layout 的首页 canonical** —— 这是 3 个页面 canonical 指向首页的根因。

## 4.4 数据获取方式

- **全站 `force-dynamic` SSR**：每次请求直接 Prisma 查询组装（实测响应头 `Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate`）。
- **无 ISR、无静态生成、无客户端数据填充**（核心情报全在 SSR HTML 里，实测 32 万字节 HTML 含完整内容）。
- 报告组装链路：`assembleReport(domain, locale)` → Target（含最新 Investigation）→ 并行查 Insight/关系/related/Event/Snapshot/RawCollection → 组装 `IntelligenceReport` JSON 契约 → `ReportPageContent` 渲染。

## 4.5 数据库结构（prisma/schema.prisma 实际字段）

核心模型：`Target`（domain 唯一键）→ `Investigation`（status: running/completed/partial/failed）→ `RawCollection`（provider 原始采集）/ `Snapshot`（type: dns/ip/ssl/http/technology/metadata/infrastructure/summary/full，JSONB 数据）/ `Event` / `Insight` / `Entity` + `EntityRelationship`（type: resolves_to/hosted_by/uses/protected_by/shares/...）/ `Monitor` / `User` / `ApiKey` / Search* 系列 / `DiscoveryCandidate`。

**Target 表只有 5 个字段**：id、domain、firstSeenAt、lastSeenAt + 关系。**没有任何 SEO 标志字段**（见 §5）。

## 4.6 动态路由 `/website/:domain` 真实行为（实测）

| 输入 | 实测结果 |
|---|---|
| `/website/openai.com` | 200，robots=noindex,follow，canonical 自身，title"openai.com 网站分析 - 技术、基础设施与变化" |
| `/website/wordpress.org` | 200，robots=index,follow |
| `/website/nonexistent-test-domain-123456789.com` | **200**，robots=noindex,follow，渲染"暂无报告"+分析表单默认页（81KB）—— **规划的 Soft 404 检测目标，实测失败** |
| `/website/invalid_domain!!` | 404 ✓（normalizeDomain 拒绝） |
| `/website/OPENAI.com` | 200（lowercase 归一，canonical 收敛）✓ |
| `/website/www.openai.com` | 308 → `/website/openai.com` ✓ |
| `/report/openai.com` | 308 → `/website/openai.com` ✓ |

结论：**"域名合法但从未分析"的页面 = HTTP 200 + noindex + 默认"暂无报告"页。规划（TECH §四/§五、CHECKLIST §四）要求的 404 未实现。** 注：本地代码 page.tsx:75-88 的"noReportYet"渲染逻辑与线上行为一致。

## 4.7 Sitemap 实现（实测）

- `src/app/sitemap.ts` + `force-dynamic`，Prisma 实时查询生成。
- 实测 2026-08-17：**179 个 URL** = 首页 1 + 静态落地页 119 + `/technology/*` 33 + `/website/*` 26。
- Website 页入选条件（代码）：最近 500 个 Target 中 `full 快照 ≥1 且 (events ≥1 或 insights ≥1)`。
- Technology 页入选条件：`uses` 关系 ≥3 个网站。
- **26 个 Website URL 中 10 个实测页面 robots=noindex**（threads.net、whatsapp.com、instagram.com、facebook.com、ddooo.com、taobao.com、tencent.com、example.com、huawei.com、kugou.com）——违反"NOINDEX 不入 Sitemap"。
- 含测试域名 `example.com`、Bing CDN 瓦片子域 `ecn.t1/t2/t3.tiles.virtualearth.net`（实测 index,follow + 在 Sitemap）——规划 Gate 2 的低价值排除未实现。

## 4.8 Robots 实现（实测）

- `src/app/robots.ts`：`Allow: /` + disallow `/analyze/ /api/ /admin/ /dashboard/ /login/ /register/` + Sitemap 行。
- 线上 robots.txt 实测：应用规则 **+ `/claim/` disallow**（本地代码没有——本地落后线上证据）+ Cloudflare 注入的 AI 爬虫块（GPTBot/ClaudeBot/CCBot/Bytespider/Google-Extended 等 Disallow，Content-Signal 声明）。
- 规则无冲突；robots.txt 未禁止 `/bulk`、`/search`、`/report`（三者由 meta robots 控制：/bulk noindex,nofollow、/search noindex,nofollow、/report noindex,follow）。

## 4.9 Website 分析数据结构（真实存在）

| 规划概念 | 实际存在形式 |
|---|---|
| IP 数据 | ✅ `Snapshot.type="ip"` 或 `full.ip`（JSONB 元组 [addr, asn, org]）；Entity type="ip" |
| DNS 数据 | ✅ `Snapshot.type="dns"`（records/nameservers） |
| Technology 数据 | ✅ `Snapshot.type="technology"`（tech/trackingIds）+ Entity type="technology" + EntityRelationship type="uses" |
| SSL/HTTP/RDAP/Metadata | ✅ 对应 Snapshot 类型 |
| analysisSuccess | ❌ 无字段。可推导：`Investigation.status`（completed/partial/failed） |
| analysisCount | ❌ 无字段。可推导：Investigation 计数（`assembleReport` 已计算 investigationCount） |
| 热门网站标记 | ❌ 无字段、无逻辑 |
| 人工审核标记 | ❌ 无字段、无逻辑（仅 `/admin/seo` 只读校验页 + ADMIN_EMAILS） |
| 停放/测试域名标记 | ❌ 无字段、无检测逻辑 |
| 数据规模（2026-08-16 生产快照） | Target **279**、Investigation 480（completed 287 / partial 191 / failed 2）、Insight 55（active 49）、Event 62、Entity 3196（technology 57）、DiscoveryCandidate 926 |

## 4.10 本地代码 vs 线上部署差异（实测 4 处）

| # | 差异点 | 本地代码 | 线上实测 |
|---|---|---|---|
| 1 | robots.txt disallow | 无 `/claim/` | 含 `/claim/` |
| 2 | 字典含 claim 认领功能（claimCta/ownerBadge/verifyHint） | 无 | 有（HTML 内嵌字典可见） |
| 3 | 首页模块卡片（"网站如何运行"链接 service.urchin.com 等） | 无此渲染 | 有 |
| 4 | reportTitle 文案 | "{domain} 网站情报报告_基础设施_技术栈_变化洞察" | 线上实际 title 为"{domain} 网站分析 - 技术、基础设施与变化" |

**含义：线上是比本地更新的构建。后续任何修改必须先以线上为准同步本地基线，否则会基于过期代码做修改。**

---

# 5. INDEX Gate V1 可行性审计

逐项对照规划 Gate 字段与真实数据模型。

| 规划字段/条件 | 当前是否存在 | 是否必须新增 | 可用现有数据替代 | 最小改动方案 |
|---|---|---|---|---|
| **analysisSuccess**（第一关） | ❌ 无字段 | **否** | `Investigation.status`：completed=成功；partial=部分成功（现有 191 条 partial，占 40%）；failed=失败（仅 2 条）。且可结合"是否存在 full 快照或 ≥1 个维度快照" | 不新增字段。定义：`analysisSuccess = latest.status !== "failed"`（partial 视为成功但降级）。当前 `assembleReport` 已取最新 Investigation |
| **IP 数据** | ✅ 有 | 否 | `Snapshot type="ip"` / `full.ip` | 无需改动。gate 已在用 `report.infrastructure.ips.length` |
| **DNS 数据** | ✅ 有 | 否 | `Snapshot type="dns"` | 无需改动。gate 已在用 dns.records/nameservers |
| **Technology 数据** | ✅ 有 | 否 | `Snapshot type="technology"` + EntityRelationship "uses" | 无需改动。gate 已在用 `report.technology.length` |
| **analysisCount**（第三关 B） | ❌ 无字段 | **否** | `Investigation` 行数（`assembleReport` 已算 `investigationCount`，sitemap 也可 groupBy） | 不新增字段。在 gate 输入中加 `analysisCount` 维度即可。**当前 gate 完全没有使用该值** |
| **热门网站标记**（第三关 A） | ❌ 无字段、无逻辑 | **是（二选一）** | 无现有数据可替代。可选择：① 代码内维护 curated 列表（如 `lib/seo/popular.ts` 常量数组，首批放 20-50 个知名站点）；② Target 加 Boolean 字段 + admin 维护入口 | 推荐最小改动：**代码常量列表**（零数据库迁移、零管理界面）。若需动态维护再升级为字段 |
| **人工审核标记**（第三关 C） | ❌ 无字段、无逻辑 | **是（二选一）** | 无现有数据可替代。已有基础设施：`/admin/seo` 页 + ADMIN_EMAILS 鉴权。可：① Target 加 `manualStatus String?`（migration，2 分钟）；② 用代码列表与热门列表合并 | 推荐：加 1 个字段 `Target.manualIndex String?`（值为 "APPROVED"/"DENIED"）——数据规模 279 行，`prisma db push` 即可，不破坏现有结构；配合 admin 页面两个按钮（或暂用 SQL 手工更新） |
| **停放/测试域名排除**（第二关） | ❌ 无字段、无检测逻辑 | **否（可用启发式）** | 可从 `report` 推断：标题为空 + technology 为空 + 无 IP/DNS → 极低值；或维护黑名单正则（localhost、test-、park、sedo 停放特征）。现有 `overview.title/description/inferences` 来自 website-summary 分析器 | 最小改动：在 gate 中加 `isLowValue(report)` 启发式函数（空标题、无技术栈、无摘要）+ 代码内小型黑名单。**不要**为此做自动检测服务 |
| Gate 模型本身 | 已实现五维评分 | — | — | 见 R1：建议以 MASTER V1 三关为准重写 `isIndexable`，评分保留为内部参考或删除 |

**结论：INDEX Gate V1 的落地不需要"凭空新增"大量字段。仅 1 个可选字段（人工审核标记），其余全部可由现有数据推导或代码常量解决。当前实现的问题不是缺数据，而是 gate 模型与规划不符。**

当前五维评分模型在真实数据上的反直觉结果（实测）：
- facebook.com、instagram.com、whatsapp.com、taobao.com、tencent.com、openai.com —— 全部 noindex（评分 <80）。原因：评分奖励"变化历史与洞察数量"，热门大站基础设施稳定 → 历史/洞察得分低 → 永远上不了 80 分。**该模型与"热门网站应 INDEX"的规划意图直接相反。**
- 而 CDN 瓦片子域 `ecn.t*.tiles.virtualearth.net` 反而 index（数据覆盖满）。

---

# 6. P0 问题清单（只列必须修复）

| ID | 问题 | 实际证据 | 影响 | 修改位置 | 修复建议 |
|----|------|---------|------|---------|---------|
| P0-1 | 不存在域名返回 HTTP 200 + 默认页（Soft 404） | 实测 `GET /website/nonexistent-test-domain-123456789.com` → **200**、81KB"暂无报告"默认页、noindex。规划 TECH §四/五、CHECKLIST §四要求 404 | 搜索引擎按 Soft 404 处理；违反规划 P0 项，验收必 FAIL | `src/app/website/[domain]/page.tsx`（本地 :75-88）的 noReportYet 分支 | **定版后修改**：Target 不存在 → `notFound()`（真 404）；Target 存在但数据为空/分析失败 → 200+noindex 保留（需先定版口径，见 R4） |
| P0-2 | Sitemap 含 NOINDEX URL（10/26） | 实测 26 个 sitemap Website URL 中 threads.net、whatsapp.com、instagram.com、facebook.com、ddooo.com、taobao.com、tencent.com、example.com、huawei.com、kugou.com 页面 robots 均为 noindex | 违反 ROADMAP 任务 4 / TECH §八/§十六 / CHECKLIST §六；搜索引擎对 sitemap 与 robots 冲突的处置不可控 | `src/app/sitemap.ts` 的 gatedTargets 过滤 | sitemap 与页面共用同一个 gate 判定（调用 `evaluateWebsitePageGate`，只收 policy=INDEX），消除双标准 |
| P0-3 | 热门大站被 noindex（Gate 3 缺失） | 实测 openai.com、facebook.com、instagram.com、whatsapp.com、taobao.com、tencent.com、huawei.com、kugou.com 全部 noindex,follow | 规划"预置热门网站 → INDEX"完全未落地；最该被收录的页面全部不可收录 | `src/lib/seo/quality-gate.ts` + `registry.ts` | 实现规划第三关：热门列表（代码常量）+ `analysisCount >= 3` + 人工审核。见 §5 最小改动方案 |
| P0-4 | `/bulk`、`/docs/api`、`/report` Canonical 指向首页 | 实测三页均输出 `<link rel="canonical" href="https://siteintel.cc"/>`；根因是 `layout.tsx` 的 `alternates.canonical` 被无 canonical 的页面继承 | 违反 TECH §六、CHECKLIST §三（明文"`/bulk` 不指向首页"）；/docs/api 是 index,follow 页，canonical 错指会浪费其索引资格 | `src/app/layout.tsx` + 各页面 generateMetadata | 从 layout 移除 `alternates.canonical`（首页自身在 page 级声明），并为 /bulk、/docs/api、/report、/analyze/[id] 等补齐自身 canonical（或明确 noindex 页可不设） |
| P0-5 | `/docs/api` 复用首页 Description（且 index,follow） | 实测 `/docs/api` 的 meta description = 首页默认 description"输入一个网站，了解它是什么…"，robots=index,follow | 违反"禁止所有页面使用首页 Description"；与 canonical 错指叠加 | `src/app/docs/api/page.tsx` generateMetadata | 补独立 description（字典已有 apiDocs 文本可用） |
| P0-6 | 测试域名/CDN 子域可索引且进 Sitemap（Gate 2 缺失） | 实测 `example.com` 在 Sitemap（页面 noindex，矛盾）；`ecn.t1/t2/t3.tiles.virtualearth.net` index,follow 且在 Sitemap | 规划第二关（停放/测试/CDN 默认域名 → NOINDEX）未实现 | `src/lib/seo/quality-gate.ts`（加 isLowValue）| 启发式低价值判定 + 代码黑名单（见 §5），先覆盖已知 4 例 |
| P0-7 | 本地代码落后于线上部署 | 实测 4 处差异（robots `/claim/`、claim 功能字典、首页模块卡、reportTitle 文案） | 后续任何基于本地代码的修改都可能与线上冲突或回退线上功能 | 项目部署流程 | 以线上为准重建本地基线（从 154 服务器拉取当前部署源码或确认差异清单后再动代码） |

（robots.txt、robots.ts 未发现规则冲突；`/report/[domain]` 308 与规划"301"的差异属等价实现，均不列为 P0。）

---

# 7. 需要修正的规划文件内容

| 文件 | 原规则 | 问题 | 修正建议 |
|------|--------|------|----------|
| MASTER §6/§7 | Simple Index Gate V1 三关；禁止复杂评分；只用 INDEX/NOINDEX | 实现的是五维 100 分评分，与文件相反；且评分模型对"热门但稳定的站点"系统性判 noindex，违背 Gate 3 意图 | **二选一**：①（推荐）执行端回退为 V1 三关（热门列表 + analysisCount≥3 + 人工审核 + 低价值排除），删除或降级评分模型；② 若保留评分，必须把评分模型正式写入 MASTER（含维度、阈值 80/60、与三关的关系），并承认热门站会被 noindex 的现实并给出补救规则 |
| MASTER §8 + ROADMAP P2 | <1,000 网站不生成实体页；Technology 门槛 50；≥1,000 才试点 | 33 个 technology 页已上线 INDEX，门槛 3 | 修正规划为现状：<1,000 阶段允许 Technology 实体页试点，门槛定为 ≥3（防薄页，当前实现值合理）；其他实体（ASN/Hosting/Nameserver）仍按 20-50 门槛延迟。或执行端将 33 页降为 noindex（不推荐，页面内容真实非薄页） |
| MASTER §五 第一层 | 品牌核心页 `/`、`/about`、`/docs`、`/guide` | 3/4 页面不存在（404） | 把清单改为真实存在的页面：`/`、`/website-intelligence`、`/how-it-works`、`/docs/api`；或补建 /about 等页面后再保留原清单 |
| TECH §三/§四/§五 + ROADMAP 任务 2 + CHECKLIST §四 | 不存在域名 → 404（§四/五）；但 §三允许"200+noindex" | 文件内部口径矛盾，执行端无法同时满足 | **定版**：① 域名合法但从未被分析 → `notFound()` 404（推荐，符合"禁止 200+默认页"的多数条款）；② 域名存在但最新分析失败/数据为空 → 200+noindex 允许保留（页面提供"重新分析"入口）。并把该口径写进 TECH §四 与 CHECKLIST §四，删除"或"字选项 |
| ROADMAP P0-任务3 伪代码 | `isIndexable(domain)` 六条件 | 与实现（评分）不符，且条件字段一半不存在 | 伪代码改为落地版（基于现有数据的推导式），并标注哪些是推导、哪些需新增（热门列表/人工审核标记） |
| TECH §九 | ISR revalidate 3600/86400 | 全站 force-dynamic + no-store，ISR 与现状冲突 | 增补裁决条款：当前阶段保留 force-dynamic SSR（279 目标，数据实时性优先，性能无压力）；当 Target 数达到阈值（如 ≥2,000）再评估 ISR。避免执行端"机械固定 revalidate" |
| TECH §十五 | lib/seo 五文件结构 | 实际 3 文件（registry/quality-gate/technology） | 按实际结构更新（canonical/metadata/structured-data 已内联），或作为整理项而非必须项 |
| ROADMAP"本周必做"第 5 项 | SEO 自动验收脚本 | TECH/CHECKLIST 无对应规范 | 在 CHECKLIST §十三 增补自动验收脚本的输出格式与验收项（HTTP 状态/robots meta/sitemap 一致性抽查，可一键运行）；并明确脚本与人工验收的分工 |
| CHECKLIST §四 | 测试 URL 只验收 HTTP 状态 | 未验收 robots meta，导致"200+noindex"被误判为通过 | 每个测试 URL 增加"预期 robots"列（如 `/website/nonexistent...` → 预期 404 + 无页面；`/website/openai.com` → 预期 200 + 按 gate 的 robots） |
| CHECKLIST §五 | "热门网站 → INDEX + Sitemap 包含" | 没有定义热门网站名单，验收无法执行 | 附上初始热门名单（建议 20-50 个域名）作为验收 fixture，与 Gate 实现共用同一列表 |
| 全部 4 份文件 | 无版本互引、无"文档 vs 代码"同步机制 | 代码注释引用"SEO 文档 §7"，但现行 MASTER §7 内容不同；执行已偏离文档 | 各文件加版本号与引用锚点（如"MASTER §6 为 Index Gate 唯一权威"）；规定任何实现偏离必须回写文档，防止再次漂移 |
| 全部 4 份文件 | 未提及本地/线上代码基线 | 本次实测发现本地落后线上 | 增补一条工作规则：执行前必须先与 154 服务器当前部署源码对齐（或使用 git 化管理），否则禁止修改 |

---

# 8. 最终执行顺序

基于真实项目状态重新生成：

## Phase 0：必须先确认的架构问题（不动代码）

1. **同步代码基线**：以线上为准重建本地副本（解决 §4.10 的 4 处差异），否则后续修改不可信。
2. **定版 Index Gate 模型**：按 §7 R1 裁决 —— 推荐回退到 Simple Index Gate V1 三关（评分模型移除或降为内部诊断）。
3. **定版"无数据页"策略**：按 §7 R4 —— 推荐"从未分析 → 404；分析失败/空数据 → 200+noindex"。
4. **定版实体页策略**：按 §7 R2 —— 推荐承认 33 个 technology 页现状（门槛 ≥3 防薄页）。
5. **定版初始热门网站名单**（20-50 个域名），供 Gate 与验收共用。

## Phase 1：P0 技术 SEO 修复

1. 修复 Soft 404（P0-1）：`website/[domain]/page.tsx` 中 Target 不存在 → `notFound()`。
2. 修复 Canonical 继承（P0-4）+ `/docs/api` Description（P0-5）。
3. 修复 Sitemap 与页面 robots 同源（P0-2）：sitemap 改用 `evaluateWebsitePageGate` 的 policy。
4. Gate 2 低价值排除（P0-6）：启发式 + 黑名单，覆盖 example.com / virtualearth 瓦片域等。
5. 全站复测（按 CHECKLIST 修正后的口径）→ 出 `SITEINTEL_SEO_IMPLEMENTATION_REPORT.md`。

## Phase 2：INDEX Gate V1（在 Phase 1 定版后）

1. 落地热门列表（代码常量 `lib/seo/popular.ts`）。
2. 接入 `analysisCount >= 3`（用 investigationCount，无需新增字段）。
3. 人工审核标记：`Target.manualIndex` 字段（唯一建议新增的字段）+ `/admin/seo` 页两个操作按钮（或首期 SQL 手工维护）。
4. 组装 `isIndexable(domain)` 与 ROADMAP 伪代码对齐，并用 admin SEO 页回归 4 类测试（CHECKLIST §五）。

## Phase 3：Sitemap / Robots 收口

1. Sitemap 全量一致性抽查（自动化脚本：对 sitemap 中全部 Website URL 逐页核对 robots meta，不一致即报错——对应 ROADMAP"自动验收脚本"）。
2. robots.ts 与线上 robots.txt 逐行比对（含 Cloudflare 注入块说明）。
3. Sitemap 分类拆分（sitemap-static / sitemap-website / sitemap-entity）——当前 179 URL 尚不需要，仅做技术预留。

## Phase 4：内容与内部链接

1. 按修正后的品牌页清单补齐（/about 或调整规划）。
2. 指南内容从字典模板逐步升级为独立深度文章（3 guides + 2 cases 已有骨架）。
3. 详情页模块级内部链接强化（已有基础，按 TECH §十三 微调）。

## Phase 5：未来数据规模达到条件后的实体 SEO

1. `Target >= 1,000`：Technology 实体页扩面、ASN/Hosting/Nameserver 试点（门槛 20-50）。
2. `>= 10,000`：实体页全量、Sitemap 分片、质量评分 V2。

---

# 附：本次审计实测 URL 汇总（2026-08-17）

| URL | 实测状态 |
|---|---|
| `/` | 200 · index,follow · canonical 自身 |
| `/website/openai.com` | 200 · **noindex,follow** · canonical 自身 |
| `/website/wordpress.org` | 200 · index,follow |
| `/website/nonexistent-test-domain-123456789.com` | **200** · noindex,follow（规划预期 404） |
| `/website/invalid_domain!!` | 404 ✓ |
| `/report/openai.com` | 308 → /website/openai.com |
| `/search` | 307 → /login |
| `/bulk` | 200 · noindex,nofollow · **canonical=首页** |
| `/docs/api` | 200 · index,follow · **canonical=首页** · **description=首页** |
| `/report` | 200 · noindex,follow · **canonical=首页** |
| `/sitemap.xml` | 200 · 179 URL（1 首页 + 119 静态 + 33 technology + 26 website；其中 10 个 website 页实测 noindex） |
| `/robots.txt` | 200 · 应用规则 + `/claim/`（本地代码无）+ Cloudflare AI 爬虫块 |
| `/search-intelligence` `/tools/dns` `/guides/website-migration` `/cases/infrastructure-migration` `/how-it-works` `/website-monitoring` `/tools/technology` | 全部 200 · index,follow · canonical 自身 |
| `/technology/cloudflare` | 200 · index,follow · canonical 自身 |
| `/about` `/docs` `/guide` | 全部 404（MASTER 品牌页清单中 3/4 不存在） |
| `/guides/nonexistent-guide` `/xyzzy-nonexistent-page` | 404 ✓（404 页自动 noindex） |

---

审计完成，未修改任何现有代码或配置。
