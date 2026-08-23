# SITEINTEL-2.0-PHASE-3A-HOMEPAGE-REPORT.md

SiteIntel 2.0 Phase 3A — 首页用户化重构实施报告
执行时间：2026-08-16 · 依据：BLUEPRINT + PHASE-1-AUDIT + PHASE-2-IA 三份文件

---

## 1. 本阶段目标

把首页从「系统/数据平台展示页」改造为「普通用户能立即理解并开始使用的网站分析入口」——用户结果优先、系统能力下沉、内部术语不进入核心视觉区。

## 2. 修改文件清单

| 文件 | 修改内容 |
|---|---|
| `src/app/page.tsx` | 首页区块全量重组（五层新结构）+ 新增 `generateMetadata`（首页专属 SEO）+ 四张价值卡动态数据查询 |
| `src/lib/i18n.ts` | home 区块 zh/en 全量重写（移除 stats/modules/pipeline/audience 等 13 个键组，新增 valueCards/howSteps/scenes/trustLine 等 9 个键组）；metadata.defaultTitle/defaultDesc zh/en 更新为新产品定位 |
| `src/components/analyze-form.tsx` | 新增可选 `ctaLabel` 属性（首页覆盖为「开始分析」，不影响其他页面表单） |

**未修改**：任何后端/数据/API/安全/Discovery 逻辑；页脚（footer 仍用 heroTitle1/2/3 旧值，属 3C 范围）；导航（3C）；旗舰报告页与实体页（3B/3D）。

## 3. 修改前后首页结构对比

| 顺序 | 修改前 | 修改后 |
|---|---|---|
| 1 | HERO（H1「网站数据分析与情报洞察平台」+ 三句 tagline + 副标题 + 分析表单） | HERO（H1「看懂任何一个网站」+ 用户副标题 + 输入框 + **开始分析** + 输入提示） |
| 2 | LIVE STATS 六项数据库计数 | （删除） |
| 3 | 适合哪些用户（6 个角色 chips）+ 真实 Intelligence 示例 | **四张用户价值卡**（网站如何运行 / 最近发生了什么 / 有什么值得关注 / 它与什么有关联——各绑定一个真实示例域的实时计数） |
| 4 | SiteIntel 分析什么（5 模块卡）+ 「每个结论都…」说明卡 | **如何使用**（三步：输入网站→自动分析→查看发现）+「了解 SiteIntel 如何工作 →」链接 |
| 5 | 情报管线（7 阶段架构图 + 管线说明） | （删除） |
| 6 | 收尾 CTA「你的目标基础设施里藏着什么？」 | **什么时候用得着**（4 个真实能力场景） |
| 7 | — | **真实分析示例**（4 卡）+ 一行可信度「已分析 {n} 个网站 · 分析基于网站公开可访问的信息」 |
| 8 | — | 收尾 CTA「输入网站，开始看懂它」 |

## 4. 修改前后用户语言对比

| 位置 | 修改前 | 修改后 |
|---|---|---|
| H1 | 网站数据分析与情报洞察平台 | **看懂任何一个网站** / Understand any website |
| 副标题 | 输入一个域名，获得完整的情报报告——基础设施、技术栈、实体关系… | 输入一个网站，了解它是什么、如何运行、最近发生了什么，以及公开信息中有哪些值得关注的变化 |
| 主 CTA | 分析 | **开始分析** / Start analysis |
| 示例区标题 | 真实 Intelligence 示例 | **真实分析示例** / Real analysis examples |
| 示例卡动作 | 查看报告 → | **查看完整分析 →** |
| 收尾 CTA | 你的目标基础设施里藏着什么？ | **输入网站，开始看懂它** |
| 示例卡描述 | N 次调查 · N 个事件 · N 条洞察 | N 次分析 · N 个变化 · N 条发现 |

## 5. 删除/下沉的后台化内容

| 内容 | 处置 |
|---|---|
| 六项统计条（实体数/关系数/洞察数/快照数/调查次数/已分析域名） | **删除主视觉**；仅保留一行「已分析 {n} 个网站」作为可信度背书（真实数据，无语义夸张） |
| 情报管线 7 阶段架构图 + 管线说明文字 | **删除**；技术说明由「了解 SiteIntel 如何工作 →」引导至 /how-it-works（3D 再人化） |
| 「SiteIntel 分析什么」5 模块卡 | **替换**为四张用户价值卡 |
| 「每个结论都标注…0-100 置信度…原始采集数据」说明卡 | 删除（置信度/证据链说明属于报告页内表达） |
| 「适合哪些用户」角色 chips | 替换为「什么时候用得着」场景（用户任务视角） |
| 「调查/事件/洞察/目标」词汇 | 全部替换（见 §4 表） |

## 6. 新首页信息架构

```
第一屏  看懂任何一个网站（H1 + 副标题 + 输入框 + 开始分析 + 输入提示）
第二层  输入一个网站，你能得到什么（4 张价值卡，各绑定真实示例域的实时数据）
第三层  如何使用（三步）+ 工作原理入口
第四层  什么时候用得着（4 个真实能力场景）
第五层  真实分析示例（4 卡）+ 已分析 N 个网站 + 数据来源说明
收尾    输入网站，开始看懂它
```

## 7. 首页真实能力与表达对应关系

| 表达 | 真实能力（数据来源） | 虚构？ |
|---|---|---|
| 网站如何运行 | 技术指纹 + 基础设施画像（technologies/infrastructure facts） | 否 |
| 最近发生了什么 | Event 系统（真实事件计数） | 否 |
| 有什么值得关注 | Insight 规则引擎（真实洞察计数） | 否 |
| 它与什么有关联 | 实体关系图（真实关系计数，groupBy 查询） | 否 |
| 已分析 N 个网站 | Target 表实时 count（线上实测 216） | 否 |
| 场景四则 | 分析/监控/技术/关联探索能力（均为已上线功能） | 否（未承诺 SEO 排名/流量/全网覆盖等） |

## 8. SEO 修改

| 项 | 修改前 | 修改后 |
|---|---|---|
| title | 网站数据分析与情报洞察平台（layout 默认） | **看懂任何一个网站 \| SiteIntel**（首页专属 absolute） |
| description | 平台式描述 | heroSub 用户式描述 |
| og:title / twitter:title | layout 默认（平台式） | 同上（首页专属 OG 块） |
| og:description / twitter:description | 平台式 | heroSub |
| canonical | SITE_URL（layout） | SITE_URL（首页显式声明） |
| defaultTitle/defaultDesc（全站回退） | 「SiteIntel — 网站分析_域名情报…」 | 「SiteIntel — 看懂任何一个网站」+ 用户式描述（zh/en 同步） |
| JSON-LD | WebSite + Organization（description 走新 defaultDesc） | 保留结构，文案随新定位 |
| robots / sitemap | index + priority 1 | 不变 |

## 9. 中英文 i18n 修改

- home 区块 zh/en 键结构**完全对称**：heroEyebrow/heroH1/heroSub/inputHint/startAnalysis/valueTitle/valueCards(4×3 字段)/howTitle/howSteps(3×2)/howMore/scenesTitle/scenes(4×2)/examplesTitle/examplesNote/viewExample/trustLine/trustNote/ctaEyebrow/ctaTitle
- heroTitle1/2/3 保留（页脚依赖，3C 处理）
- metadata.defaultTitle/defaultDesc zh/en 同步更新
- 生产验证：zh/en 双语首页均正确渲染全部新键

## 10. 测试结果

| 项 | 结果 |
|---|---|
| typecheck | ✅ |
| lint | ✅（3 个改动文件 0 问题） |
| test | ✅ **138/138**（无测试引用被移除键） |
| build | ✅ 0 error |

## 11. 生产验证结果

| # | 检查 | 结果 |
|---|---|---|
| 1 | 中文首页 | ✅ H1「看懂任何一个网站」、四卡/三步/场景/示例/信任行全部渲染 |
| 2 | 英文首页 | ✅ H1「Understand any website」、全部区块英文化（si_lang=en cookie 实测） |
| 3 | 网站输入 | ✅ 输入框 + 示例 chips 渲染正常 |
| 4 | 分析 CTA | ✅ 按钮「开始分析」×2 处（hero + 收尾）；POST 端点活性确认（无效载荷 400，未产生数据） |
| 5 | 首页所有链接 | ✅ /how-it-works 200、/tools/website-analysis 200、4 个示例报告链接全部 200 |
| 6 | 移动端基础布局 | ✅ 全部使用现有响应式类（sm:grid-cols-2/3/4、max-w 容器），无新增固定宽度 |
| 7 | SEO metadata | ✅ title/og 均为「看懂任何一个网站 \| SiteIntel」 |
| 8 | 后台术语零暴露 | ✅ **可见正文 0 命中**（实体数/关系数/洞察数/快照数/调查次数/情报管线/目标基础设施/Intelligence 示例全无）；唯一 1 处命中位于内嵌字典序列化载荷（客户端共享全量字典所致，非可见文案——与早前「Phase 3」残留同类现象，3C 可处理） |
| 9 | 数据库统计退出核心视觉 | ✅ 六项统计条移除，仅一行「已分析 216 个网站」 |
| 10 | 现有分析功能无回归 | ✅ 138 测试全过、示例报告页 200、API 端点正常 |

## 12. 发现的问题

1. **内嵌字典序列化泄漏**（非本轮引入）：客户端组件共享全量 i18n 字典，导致「采集 DNS 记录」「调查次数」等内部文案出现在首页 HTML 的脚本载荷中（不可见，无 SEO 影响——Google 不执行/不索引脚本载荷内容）。根治方案=按页面拆分字典（3C 候选，涉及客户端组件传参重构，不在 3A 范围）。
2. 页脚仍为旧 tagline（「数据成为信息…」）——3C 范围。
3. 导航「分析什么」仍指向 `/#modules` 锚点——新页面保留该 id 于价值卡区，链接不失效，文案 3C 更新。

## 13. 未修改的内容

导航、页脚、旗舰报告页、实体页、工具页、搜索页、how-it-works 内容、后端/数据/API/安全/Discovery/调度/额度、登录与账号体系、SEO 注册表与质量门。

## 14. 是否存在后端改动

**无。** 三个文件均为纯前端（页面组件 + i18n 文案 + 组件 prop）；新增的关联计数查询使用既有 Prisma 关系表（groupBy 只读），无 schema/migration/API 变更。

## 15. 回滚方式

Git 回滚至 `git revert` 或 `git checkout <parent> -- src/app/page.tsx src/lib/i18n.ts src/components/analyze-form.tsx` + 重建重启；无数据依赖，回滚零副作用。

---

*Phase 3A 完成。未进入 3B/3C/3D。*
