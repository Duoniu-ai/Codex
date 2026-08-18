# SITEINTEL PRODUCTION BASELINE REPORT

SiteIntel 生产基线确认报告（v2 —— 2026-08-17 复验版）

**验证方式**：线上真实页面逐页抓取 + 生产服务器实际部署代码只读审计 + Git 提交与部署时间线交叉审计。
**原则**：线上生产环境 = 唯一真实产品验证环境；不建立本地运行环境、不连接生产数据库、不修改业务代码、不重新部署。

---

## 0. 验证方法与本报告定位

| 项 | 说明 |
|---|---|
| 验证对象 | 线上 `https://siteintel.cc`（Cloudflare 回源）真实页面 + 服务器 `154.204.176.66:/www/wwwroot/siteintel.cc` 实际部署代码（git 仓库） |
| 本报告目标 | 建立 **SITEINTEL_PRODUCTION_BASELINE**：线上实际运行版本 = 当前生产服务器实际代码 = 后续 GitHub（Duoniu-ai/siteintel）代码基线 |
| 明确排除 | 本地 `C:\Users\deepo\siteintel` 不作为验证基准（仅作为可选的编辑工作区，一切以服务器代码为准） |
| 禁止项遵守 | 未启动本地项目 / 未装本地 PostgreSQL / 未建隧道 / 未连生产库 / 未写 .env / 未改代码 / 未重部署 ✓ |

---

## 1. 当前线上真实产品状态（2026-08-17 逐页实测）

### 1.1 HTTP 状态总览

| URL | HTTP | 说明 |
|---|---|---|
| `/` | 200 | 首页（Phase 3A 用户化版） |
| `/search` | 307 → `/login` | Search Intelligence 控制台，未登录重定向 |
| `/bulk` | 200 | 批量分析页 |
| `/report` | 200 | 近期报告索引（分页 50/页） |
| `/report/[domain]` | 308 → `/website/[domain]` | permanentRedirect 迁移（A 级页面规范 URL） |
| `/docs/api` | 200 | API 文档页 |
| `/claim` | **404** | 无根路由（仅 `/claim/[domain]`），符合代码 |
| `/claim/[domain]` | 307 → `/login` | 未登录跳登录；登录后为所有权认领流程 |
| `/website/openai.com` 等 26 页 | 200 | 情报页（INDEX Gate 控制索引） |
| `/website/不存在域名` | **200 + noindex** | 「还没有 X 的分析结果」空态引导页（**非 404**——P0-1 口径待定版，见 SEO 审计） |
| `/technology/nginx` 等 33 页 | 200 | 技术情报页（≥3 站点门槛，可索引） |
| `/robots.txt` | 200 | 应用规则 + Cloudflare 注入 AI 爬虫块 |
| `/sitemap.xml` | 200 | 179 URL |

### 1.2 关键页面元数据实测

| 页面 | Title | Canonical | Robots | Description |
|---|---|---|---|---|
| `/` | 看懂任何一个网站 \| SiteIntel | `https://siteintel.cc`（自身） | index, follow | 首页专属 heroSub |
| `/bulk` | 批量分析 \| SiteIntel | **→ 首页**（P0-4） | noindex, nofollow | **复用首页**（P0-5） |
| `/report` | 近期报告 \| SiteIntel | **→ 首页**（P0-4） | noindex, follow | 「SiteIntel 最近分析的域名及其调查状态。」 |
| `/docs/api` | SiteIntel 开放 API \| SiteIntel | **→ 首页**（P0-4） | index, follow | **复用首页**（P0-5） |
| `/website/openai.com` | openai.com 网站分析 - 技术、基础设施与变化 | `/website/openai.com`（自身） | **noindex, follow** | 域专属 reportDesc |
| `/website/wordpress.org` | 同格式 | 自身 | **index, follow** | 域专属 |
| `/technology/nginx` | Nginx网站技术分析与使用情况 - SiteIntel | `/technology/nginx`（自身） | index, follow | 技术页专属 |
| `/claim/[domain]`（未登录 307 响应体） | Claim Website \| SiteIntel | — | noindex, nofollow | — |

### 1.3 Sitemap 结构（179 URL，lastmod 今日动态生成）

| 分组 | 数量 | 备注 |
|---|---|---|
| 首页 | 1 | — |
| SEO 落地页 | 6 | website-intelligence / how-it-works / relationship-intelligence / search-intelligence / infrastructure-intelligence / technology-intelligence |
| 工具/搜索/指南/案例 | 5+3+3+2 | /tools/* ×5、/search/* ×3、/guides/* ×3、/cases/* ×2 |
| /website/* | 26 | **其中 10 页页面级 noindex**（threads/whatsapp/instagram/facebook/ddooo/taobao/tencent/example/huawei/kugou）——sitemap 门槛与页面 Gate 不同源（P0-2）；**含 ecn.t1~t3.tiles.virtualearth.net 瓦片子域与 example.com**（P0-6 低价值项） |
| /technology/* | 33 | ≥3 站点门槛（规划要求 <1000 延迟 + 门槛 50，实际门槛 3——已上线即可索引） |
| /asn/* | 40 | 实体页 |
| /ip/* | 29 | 实体页 |
| /organization/* | 16 | 实体页 |
| /certificate/* | 14 | 实体页 |

### 1.4 首页功能实测（Phase 3A/3C）

- H1「看懂任何一个网站」；导航 4 任务：首页 / 网站分析(/tools/website-analysis) / 探索(/report) / 工具(/tools/dns) + 登录。
- `#modules` 四张用户价值卡，实时 DB 数据驱动（本日实测：wordpress.org · 已分析 134 次 / fps.goog · 记录到 0 个变化 / zngg.net · 0 条值得关注 / service.urchin.com · 30 条公开关联）——0 值为真实数据诚实降级，无编造。
- 信任行「已分析 285 个网站」= 实时 target 计数。
- JSON-LD：WebSite + Organization；robots.txt：应用 disallow `/analyze/ /api/ /admin/ /dashboard/ /login/ /register/ /claim/` + CF 注入 AI 爬虫块（GPTBot/ClaudeBot/CCBot/Bytespider/Google-Extended 等）+ Content-Signal。

---

## 2. 当前生产服务器代码状态

| 项 | 结论 |
|---|---|
| 路径 | `/www/wwwroot/siteintel.cc`（git 仓库，master，无 remote） |
| HEAD | `1bdfcb4`（docs: discovery candidate quality filter v1 design，2026-08-16 10:27 UTC） |
| 提交数 | 66 commits |
| 构建 | `.next/BUILD_ID` = `cQ-ZftIn6HAbu-vHzVSYg`，写于 2026-08-16 09:16:36 UTC |
| 服务 | systemd `siteintel` **active since 08-16 09:16:36 UTC**（构建后立即重启，此后未再重启）；端口 3003 仅本机可连（`--hostname 127.0.0.1`），nginx 反代 + Cloudflare |
| 运行环境 | Node v20.20.0（nvm）；PostgreSQL 14 本机 127.0.0.1:5432，库/用户 siteintel |
| .env 键名 | `ADMIN_EMAILS AUTH_SECRET AUTO_DISCOVER DATABASE_URL DISCOVER_DAILY_CAP NEXT_PUBLIC_SITE_URL TELEGRAM_BOT_TOKEN`（7 键）；可选 AI 键（AI_API_KEY/AI_API_URL/AI_MODEL）与 NEXT_PUBLIC_BAIDU_TONGJI_ID 未配置（AI 休眠 503、百度统计用代码默认值） |
| 工作树 | `M DISCOVERY-CANDIDATE-FILTER-DESIGN.md`、`D baidu_verify_codeva-r7wFVf611L.html`、untracked（.well-known/、SEO 规划 4 份、docs/navigation/、多份审计报告）——均不影响构建产物 |

### 2.1 构建后 4 个 commit 的改动范围（逐 commit 核实）

| commit | 内容 | src 改动 |
|---|---|---|
| 386ea06 | docs: frontend JS exposure audit | 0（仅 1 个 md） |
| b977b82 | docs: production data snapshot | 0（仅 1 个 md） |
| a0a6bb7 | docs: discovery candidate quality audit | 0（仅 1 个 md） |
| 1bdfcb4 | docs: discovery candidate quality filter v1 design | 0（仅 1 个 md） |

**运行构建 = HEAD 的 src**，四条独立证据收敛：
1. `find src -newer .next/BUILD_ID` 为空（无任何 src 文件新于构建产物）；
2. 线上导航（首页/网站分析/探索/工具+登录）与 HEAD `src/components/site-header.tsx` 逐项一致（Phase 3C 特征）；
3. 线上 robots.txt disallow 列表与 HEAD `src/app/robots.ts` 逐字一致；
4. 线上 /claim/[domain] 307 响应体 title「Claim Website」与 HEAD `src/app/claim/[domain]/page.tsx` 的 `metadata.title` 一致。

---

## 3. 线上与生产代码一致性验证（交叉审计）

| # | 线上观测 | 服务器代码依据 | 一致 |
|---|---|---|---|
| 1 | 导航 4 任务 + 登录 | site-header.tsx（nav.home/analyze/explore/tools + login） | ✓ |
| 2 | robots.txt disallow `/claim/ /register/ /analyze/ /api/ /admin/ /dashboard/ /login/` | app/robots.ts disallow 数组逐字相同 | ✓ |
| 3 | /search 307 → /login | app/search/page.tsx `if (!user) redirect("/login")` | ✓ |
| 4 | /claim/[domain] 307 → /login，title "Claim Website" | app/claim/[domain]/page.tsx（redirect + metadata.title 硬编码） | ✓ |
| 5 | /website/X title "{domain} 网站分析 - 技术、基础设施与变化" | app/website/[domain]/page.tsx `dict.metadata.reportTitle.replace("{domain}")` | ✓ |
| 6 | /website robots = INDEX Gate 结果（16 index / 10 noindex） | generateMetadata → `evaluateWebsitePageGate`（≥80 INDEX） | ✓ |
| 7 | 不存在域名 → 200 + 「还没有 X 的分析结果」+ noindex | website 页 `if (!report)` 空态分支 | ✓ |
| 8 | /report/[domain] 308 → /website/[domain] | app/report/[domain]/page.tsx permanentRedirect | ✓ |
| 9 | 首页 #modules 四卡 + 实时指标（含 0 值） | app/page.tsx cardDemos（investigations/events/insights/relationships 映射） | ✓ |
| 10 | /bulk /docs/api /report canonical → 首页、/docs/api /bulk description 复用首页 | layout.tsx `alternates.canonical: SITE_URL` 继承 + 无页级 generateMetadata 覆盖 | ✓ |
| 11 | sitemap 26 website 门槛（full≥1 且 events/insights≥1）+ 10 页 noindex 仍收录 | app/sitemap.ts gatedTargets 与页面级评分 Gate 不同源 | ✓（问题为设计遗留 P0-2） |
| 12 | /technology/nginx index,follow；33 页进 sitemap | sitemap.ts ≥3 站点门槛 + technology 页 | ✓ |
| 13 | 首页 title「看懂任何一个网站 | SiteIntel」（absolute） | app/page.tsx generateMetadata `{absolute: heroH1}` | ✓ |

**结论：线上每一处可观测行为均能在服务器 HEAD 代码中找到对应实现，无任何"线上有而代码无"的观测。**

---

## 4. 当前 Git / 部署时间线

### 4.1 近期提交（15 条，服务器 git log 实测，HEAD=本地 HEAD=1bdfcb4）

```
1bdfcb4 08-16 10:27 docs: discovery candidate quality filter v1 design (READ ONLY dry-run)
a0a6bb7 08-16 10:15 docs: discovery candidate quality audit (READ ONLY)
b977b82 08-16 10:06 docs: production data snapshot (READ ONLY)
386ea06 08-16 09:57 docs: frontend JS exposure audit (READ ONLY)
9bdf6b2 08-16 09:17 feat(nav): Phase 3C user-task navigation & public component unification
3379d51 08-16 08:54 fix(report): long unbreakable values overflow cards on mobile
24a53b7 08-16 08:49 feat(report): Phase 3B.3 reading path & first-screen value refinement
1668a98 08-16 08:32 feat(report): Phase 3B.2 concrete changes + evidence chain
3730215 08-16 08:13 feat(report): Phase 3B.1 visual hierarchy refinement
513d53e 08-16 08:01 feat(report): SiteIntel 2.0 Phase 3B - flagship report page user-ification
605f8ab 08-16 07:42 feat(homepage): SiteIntel 2.0 Phase 3A - user-centric homepage
be54ed2 08-16 07:24 docs(ia): SiteIntel 2.0 Phase 2 information architecture design (READ ONLY)
cfba048 08-16 07:18 docs(ux): SiteIntel 2.0 Phase 1 user-experience baseline audit (READ ONLY)
467c736 08-16 07:06 feat(discovery): priority aging anti-starvation for candidate queue
7cf476b 08-16 06:58 docs: discovery system read-only white-box audit
```

### 4.2 部署时间线

- 最近一次完整部署：**2026-08-16 09:16 UTC**（`pnpm build` 完成 → `systemctl restart siteintel`，服务 active since 09:16:36）。
- 部署后 4 个 commit（09:57–10:27）全部为单文件 docs，不影响运行产物；运行构建保持 HEAD 代码。
- 部署工作流（既定）：SSH 直传 → `git commit` → `pnpm build`（PATH 需 nvm node v20.20.0）→ `systemctl restart siteintel`；无 CI/CD、无 remote、无 tag。
- 仓库规模：.git 7.6 MB，1160 loose objects（未 repack），66 commits。

---

## 5. 4 处差异复验结果（线上功能 vs 生产服务器代码）

此前发现的 4 处差异，本次按「线上实际功能是否能在当前生产服务器代码中找到对应实现」复验：

| # | 差异项 | 复验结论 | 分类 |
|---|---|---|---|
| ① | robots `/claim/` | 服务器 HEAD `robots.ts` disallow 数组**含 `/claim/`**，线上 robots.txt 与之逐字一致（含 /analyze/ /api/ /admin/ /dashboard/ /login/ /register/） | **生产代码已实现** |
| ② | Claim 认领功能字典 | `dict.ownership.*` zh/en 完整存在（claim="生成验证令牌"、claiming、claimFailed、claimCta="认领这个网站"、ownerBadge、verified 等）且 claim 页正文 + OwnershipPanel 实际使用；**注意：claim 页 `metadata.title` 硬编码英文 "Claim Website"**（未走字典，zh 下 title 仍为英文——代码现状，线上一致） | **生产代码已实现**（含 1 处 P3 级打磨项：title 应改用字典） |
| ③ | 首页模块卡 | `app/page.tsx` #modules 四张价值卡（dict.home.valueCards + 实时 DB cardDemos 映射：investigations/events/insights/relationships）与线上渲染逐位一致（含 0 值诚实降级） | **生产代码已实现** |
| ④ | reportTitle 文案 | `dict.metadata.reportTitle` zh="{domain} 网站分析 - 技术、基础设施与变化" / en 同构，website 页 generateMetadata 与 report JSON-LD `name` 双处使用；线上标题逐字一致 | **生产代码已实现** |

**总结：4/4 均为「生产代码已实现」，无「线上存在但服务器代码未找到」、无「需进一步定位部署来源」、无「确认存在代码分叉」。**
4 处差异的历史根因 = 旧本地副本过时（旧本地缺 /claim/ 路由与 robots 条目、旧 reportTitle 文案、旧首页卡、缺 claim 字典键），与 2026-08-16 同步归档的 `siteintel.local-old-20260817.tar.gz` 差异表 #4/#5/#6 完全对应；自 08-16 基线同步后已全部消除。

---

## 6. 当前生产代码是否可以作为 GitHub 基线

**可以作为。** 判定依据：

| 检查项 | 结果 |
|---|---|
| 内容 = 线上功能 | ✓ §3 交叉审计 13/13 一致，生产代码即线上真实产品 |
| git 仓库完整 | ✓ 66 commits、HEAD 1bdfcb4、git 用户已配置（SiteIntel Deploy / deploy@siteintel.cc） |
| 敏感信息 | ✓ tracked 240 文件中**无真实凭据**（秘密扫描仅 pnpm-lock `js-tokens` 误报与 migration 列名 passwordHash/ApiKey）；`.env` 被 .gitignore 覆盖（`git ls-files` 确认不含）；`.env.example` 为 CHANGE_ME 占位模板 |
| 代码卫生 | ✓ .next / node_modules / *.tsbuildinfo / *.pem / .env* 均忽略；migrations 已入库（7 个） |
| 规模 | ✓ .git 7.6 MB，可直接推送 |

**流程（推荐）**：生产服务器实际代码 → 线上功能对应确认（本报告 §3）→ 安全导出/同步（tar over ssh，排除 node_modules/.next/.env）→ git 整理（§7 清单）→ 敏感信息复查 → 推送 GitHub Duoniu-ai/siteintel（**仓库当前不存在**，需创建；gh CLI 已登录 Duoniu-ai）→ 建立 Production Baseline Tag。

---

## 7. 推送到 Duoniu-ai/siteintel 前需要处理的问题

| # | 问题 | 建议处置 |
|---|---|---|
| 1 | **`.user.ini` 被 tracked**（宝塔面板服务器专用文件，非项目内容） | `git rm --cached .user.ini` + 加入 .gitignore，不进仓库 |
| 2 | **baidu_verify 验证文件双份 tracked**（大写已删待提交 `D` + 小写仍 tracked） | 决定删除或保留；建议 `git rm` 移除（验证早已完成，百度侧不需要仓库内文件） |
| 3 | **`.well-known/` untracked**（含 siteintel-verification.txt 所有权验证标记） | 加入 .gitignore，防止后续误提交 |
| 4 | **根目录大量文档**（tracked 的有 20+ 份审计/规划报告，含生产数据快照 `SITEINTEL-CURRENT-DATA-SNAPSHOT.md`；untracked 的有 SEO 规划 4 份、2.0 蓝图、3C 任务书等） | **用户决策**：审计报告公开策略（建议：已提交的历史保持不动；未跟踪文档评估后纳入或删除）。若倾向公开技术仓库，可保留全部——报告均无密钥 |
| 5 | **git 历史 66 commits 含内部文档**（README-ONLY 审计、生产数据观察、白盒审计报告） | 确认可公开；如不愿公开内部报告，需新建仓库从 1bdfcb4 起重写历史或选择性地仅推送 src（不建议，破坏基线可追溯性） |
| 6 | loose objects 未 repack（1160 个） | 推送前 `git gc` 可选（7.6 MB 无必要） |
| 7 | 无 remote / 无 tag | push 后建立 Production Baseline Tag（如 `v2.0.0-prod-1bdfcb4`） |
| 8 | `.env` 本身 | 已忽略 ✓ 无需操作（推送前可 `git check-ignore .env` 复核） |

---

## 8. 后续 SEO Phase 1 应基于哪个生产代码版本

- **基准 = 服务器 HEAD `1bdfcb4`（= 当前运行构建的 src = 线上功能）。**
- 修复方式：基于 1bdfcb4 开分支/编辑（本地 `C:\Users\deepo\siteintel` @ 1bdfcb4 可作为编辑工作区，但**验证与上线永远以服务器为准**）→ 同步服务器工作树 → `pnpm build` → `systemctl restart siteintel`；schema 变更必须走 Prisma migration，禁止 db push。
- P0 待办来源 = `SITEINTEL_SEO_PLAN_AUDIT_REPORT.md`（P0-1 404 口径待定版、P0-2 sitemap 同源、P0-3 Gate 三关、P0-4/5 canonical/desc、P0-6 低价值排除），本次实测再次确认其现场：/bulk /docs/api /report canonical→首页、/docs/api /bulk description 复用首页、sitemap 26 条含 10 noindex + 瓦片子域 + example.com、不存在域 200 空态。
- **禁止**以任何旧本地副本（siteintel.local-old-20260817.tar.gz 及更早）作为代码依据。

---

## 历史附录（v1 报告 2026-08-17 上午，供追溯）

- 旧本地副本落后生产的 14 项差异（缺 claim/ip/asn/certificate/organization 5 组路由、7 个 migrations、12 个 vitest、lib/seo/entity.ts；119 vs 161 src 文件）已全部消除：旧本地已归档 `siteintel.local-old-20260817.tar.gz`，**永不反向覆盖**。
- 线上 → 本地同步（v1）已执行并验证：git HEAD / 工作树 / md5 / 路由 / migrations / typecheck 全一致；服务器 .env **未拉取到本地**（本地仅 .env.example）——本 v2 报告不依赖本地环境，进一步强化该隔离。
- `siteintel.local-old-20260817.partial-copy-DELETABLE` 为中断残留，可删。

---

## 最终结论

# PRODUCTION BASELINE CONFIRMED

- 线上真实页面 14 项逐页实测，每项行为均在服务器 HEAD 代码（`1bdfcb4`，运行构建 08-16 09:16 = HEAD src，四条独立证据）中找到对应实现，**无代码分叉**。
- 4 处差异复验：**4/4 生产代码已实现**（差异根因 = 旧本地副本过时，已归档消除）。
- Git/部署时间线完整（66 commits、无 remote、最近部署 08-16 09:16 UTC、部署后 4 commit 纯文档）。
- 生产代码**可作为 GitHub 基线**（Duoniu-ai/siteintel，仓库需创建）；推送前 8 项处理清单（其中 .user.ini / baidu_verify / .well-known 为必办卫生项，文档公开策略需用户决策）。
- 后续 SEO Phase 1 一律基于服务器 HEAD `1bdfcb4`（= 线上），禁止任何旧副本介入。

**线上实际运行版本 = 当前生产服务器实际代码 = 后续 GitHub 代码基线（Duoniu-ai/siteintel）。**
