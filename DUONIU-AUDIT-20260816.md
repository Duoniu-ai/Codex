# duoniu.cc 项目全面审计报告

**审计日期：** 2026-08-16
**审计范围：** /www/wwwroot/duoniu.cc/duoniu-new-vue/ 代码库、生产（3901）与 staging（3902）部署、线上实测
**审计依据：** 对照《duoniu-cc-website-analysis-and-repositioning.md》（重定位与产品规划提案，用户 2026-08-16 提供）逐节验证
**审计方式：** SSH 实盘只读检查（git 历史/状态、路由清单、源码 grep、双环境本地 curl、smoke-test 14 页）+ 线上域名实测

---

## 1. 总览

| 维度 | 结论 |
|---|---|
| 技术底座 | ✅ 达标。Nuxt3 SSR + 9 Provider 适配器 + 统一信封 + 实体落库（Phase 0-6 全部上线），SEO（sitemap/JSON-LD/llms.txt/robots）齐备 |
| 部署健康 | ⚠️ 审计中发现 staging 500（已修复，见 §2）；生产构建落后 3 个 commit，待 promote |
| 与文档对比 | 文档 16 路径全覆盖 + 实际多 3 路径；但「旗舰页导航缺失」「等级系统未建」「历史时间线未建」「变现全无」四大差距 |
| 产品形态 | 文档定位「网络诊断/代理验证工具箱」成立；现有功能已超越文档认知（风控值行、IP 黑名单行、大模型检测行、data-tip 微文案） |

---

## 2. 部署与版本状态（审计时点实测）

| 环境 | 端口 | 构建时间 | 状态 | 包含 commit |
|---|---|---|---|---|
| 生产 | 3901 | 2026-08-15 08:10 | ✅ 200（www.duoniu.cc 实测） | 到 `173fb61`，**不含** `b3c8187` |
| staging | 3902 | 2026-08-16（修复后重建） | ✅ 200（staging.duoniu.cc 实测） | 到 `ce0e18a`（最新 HEAD） |

### staging 500 根因（审计中发现并已修复）

1. **直接原因**：`b3c8187`（删语言切换）删除 `useLanguage.ts` 及全部导入，但 `app/components/UniversalSearch.vue:45` 残留 `t(...)` 调用 → SSR `ReferenceError: t is not defined` → 凡渲染 UniversalSearch 的页面（`/`、`/history`）500，其余 17 条路由正常。
2. **连带发现**：`b3c8187` 的按钮级删除**不完整**——`app/layouts/default.vue` 遗留 lang-toggle 按钮本体（引用已删除的 `toggleLang`）、`onMounted(() => initLang())`（客户端挂载即抛错）、`.lang-toggle` 样式 6 行。
3. **修复 commits**：
   - `3fcce29` fix: UniversalSearch.vue 一行（`t(defaultPlaceholder, ...)` → `defaultPlaceholder`）
   - `ce0e18a` fix: default.vue 8 行删除（按钮 + initLang 调用 + 样式），全 app grep 零残留
   - `47549f1` docs: 分析文档入库
   - tag：`staging-500-fix-20260816` → `ce0e18a`
4. **验证**：staging 重建后 smoke-test **14/14 全过**；首页 HTML `lang-toggle`=0、`Switch to English`=0、`placeholder="输入IP / 域名 / ASN"` 生效；外网 staging.duoniu.cc 200。
5. **遗留**：生产仍运行 08:10 旧构建（仍显示 lang-toggle 按钮 + 旧占位文案），**promote 待用户确认**（执行规则②：生产操作需人工确认）。

---

## 3. 文档前提与方法论核验

| 文档声明 | 核验结果 |
|---|---|
| 「duoniu.cc blocks automated crawling (robots.txt disallow)」 | ❌ **已过时**。2026-08-15 已放行搜索引擎 + 14 个 AI bot（robots.txt 显式 Allow + nginx UA 白名单两层），sitemap.xml 线上 200、llms.txt 存在。文档基于「无法直接访问」的推断前提不再成立，可改为直接对站验证 |
| 「based on sitemap URL structure」 | sitemap 覆盖实际路径属实（见 §4） |

---

## 4. 文档逐节对照（假设 vs 实际）

### §1 Sitemap 路径覆盖

文档列出的 16 个路径**全部存在**：

| 路径 | 存在 | 备注 |
|---|---|---|
| `/` `/whois` `/asn` `/asn/{id}` `/bgp` `/cidr` `/dns` `/route` `/trace` `/ping` `/ip-leak` `/purity` `/env` `/tools` `/history` `/about` `/faq` | ✅ 全部 | 16/16 |

文档**未提及**的实际路径：`/api`（API 文档页）、`/domain/[q]`（域名页）、`/as/[asn]`（301 → `/asn/[asn]`，旧链接兼容，有意保留）。

### §2-3 目标用户与竞品分析

- 用户画像推断与站点现有功能吻合（风控值、原生IP、大模型检测均面向代理/VPN 验证人群）。
- 竞品 gap 分析合理。文档核心差异化「plain-language explanation」**已有雏形**：首页 14 行信息中 5 项（IP 类型/原生IP/风控值/黑名单/大模型检测）带 data-tip 弹窗解释「是什么+怎么算」，FAQ 有「风控值怎么算」问答——但未系统化（无独立方法论页）。

### §4-5 重定位与导航 —— **文档第一大差距**

文档建议「IP 检测 + 工具箱」两顶级项。**实际导航**（`app/layouts/default.vue` menus 数组）：

```
首页 / IP环境检测 / IP泄漏检测 / ASN列表 / 查询工具▾(仅DNS/Whois/Ping/CIDR 4项) / 常见问题▾(API/关于)
```

| 问题 | 严重度 |
|---|---|
| **旗舰页 `/purity` 不在导航**（只能从首页知识库/搜索进入） | 🔴 高——文档定位的旗舰功能在站内不可见 |
| `/history` 不在导航（文档视其为留存核心） | 🔴 高 |
| 工具下拉仅 4/8（缺 bgp/trace/route/cidr） | 🟡 中 |

### §6 首页（文档提议 8 模块 vs 实际）

用户 2026-08-15 拍板恢复旧堆叠布局（七 Tab 方案曾被否决）。实际首页：

**已有（文档认知之外）**：
- SSR 自动加载默认数据 + 访客 IP 自动填充搜索框（「零点击洞察」部分达成）
- 基本信息 14 行，含 **风控值**、**IP 黑名单（AbuseIPDB 实时）**、**大模型检测**（ChatGPT/Claude/Gemini/DeepSeek 可达性——文档 Phase 2 平台兼容卡的 AI 雏形）
- 8 张卡：WebRTC 泄漏检测、IP 位置历史、注册地历史（真实 RDAP）、IP 反查域名、BGP 拓扑、地图定位（Leaflet）、延迟测试
- data-tip 弹窗微文案（5 处）

**缺失（对照文档 §5.2/§6 十模块）**：大 verdict 等级卡（S+–C）、评分分解卡（Proxy/VPN/机房/黑名单/泄漏/环境子分）、「What should I do」行动面板、平台兼容条（非 AI 平台）、历史/对比 teaser、方法论 footer 块。

⚠️ **约束**：七 Tab 大改已被用户否决一次——首页任何视觉重构必须先 staging 预览经用户确认（规则与偏好冲突时用户优先）。

### §7 内页模块

| 页面 | 文档要求 | 实际 | 判定 |
|---|---|---|---|
| purity（旗舰） | 等级徽章 + 子分卡微文案 + 前后对比 + 重检 + 导出 | 0-100 数值分 + good/warn/bad 徽章 + 风险 flags 网格；**无 S+/S/A/B/C 字母等级**；页面挂两个**「待建」卡**（黑名单查询、AI 出口检测）；无对比/重检/导出 | ⚠️ 半成品 |
| whois | 摘要卡 + 原始输出折叠 + ASN 关联 | 已 features 化，有 JSON-LD（WebApplication+FAQPage） | ✅ |
| asn/[id] | 摘要 + 前缀表 + 信誉卡 | 详情页存在（holder 正常） | ✅ |
| bgp/cidr/dns/ping/trace | 各自结果模块 | 已 features 化 + SEO 区块 + JSON-LD | ✅（dns 缺多解析器对比） |
| dns | 多解析器对比表（§7.6） | **无** | ❌ |
| route/trace | 可视路径地图（§7.7） | **无**（首页 Leaflet 可复用） | ❌ |
| history | 趋势 sparkline + 两两对比 | 仅「当前快照 + 时间线建设中」占位（Phase 6 落库后数据开始积累） | ❌ 未建 |
| tools | 全工具卡网格 | 存在（SimpleTool 已拆 6 组件） | ✅ |
| about/faq | 含「评分如何计算」章节 | FAQ 有 2 条方法论问答（家宽 vs 机房、风控值怎么算） | 🟡 半满足 |

### §8-10 UX / SEO

- ✅ 时间到第一洞察（首页零点击出数据）、一致设计系统（Phase 5 提取 main.css、明暗主题）
- ❌ 等级体系统一（无数值→字母等级映射）、行动建议、「AI-ready 导出」
- 🟡 信任透明：FAQ 2 条 + purity 页说明（自认「基础版」，机房/代理本地库完善中）
- ✅ SEO：8 个工具/leak/env 组件全部 JSON-LD + SEO 区块；sitemap 动态源（线上 200，含 /asn/13335）；llms.txt；robots 放行；百度统计
- ⚠️ purity 旗舰页仅基础 meta（useSeoMeta title/desc/keywords），无方法论长文内容——「IP纯净度检测」关键词竞争需内容支撑

### §11-12 变现与路线图

- 变现：**全无**（api.vue 仅 3 个免费端点 + 限流说明；无 API 商业化/浏览器扩展/affiliate/团队版）
- Phase 1：自动检测部分 ✓ / **导航统一 ❌** / 等级系统 ❌ / 微文案部分 ✓
- Phase 2：平台兼容卡仅 AI 雏形 / 复制报告 ❌ / 方法论页 ❌ / 历史对比 ❌
- Phase 3：SEO 内容 ✅（已完成大半）/ DNS 对比 ❌ / 路径图 ❌ / 移动 verdict ❌
- Phase 4：全 ❌

---

## 5. 关键发现清单

| # | 发现 | 严重度 | 处理建议 |
|---|---|---|---|
| 1 | staging 500（UniversalSearch 残留 t()） | 🔴 | ✅ 已修复（3fcce29/ce0e18a），生产 promote 待确认 |
| 2 | `/purity` 旗舰页不在导航 | 🔴 | 导航加「纯净度检测」一级项（工作量小） |
| 3 | `/history` 不在导航 | 🔴 | 同上 |
| 4 | 工具下拉 4/8 | 🟡 | 补 bgp/trace/route/cidr |
| 5 | purity 页两个「待建」卡（黑名单/AI 出口检测） | 🟡 | 黑名单数据源首页已在用（AbuseIPDB），可复用；AI 出口检测与大模型检测行逻辑相近 |
| 6 | 无 S+/S/A/B/C 字母等级体系 | 🟡 | 文档 Phase 1 #3；涉及全站视觉，需 staging 预览 |
| 7 | history 时间线/趋势图未建 | 🟡 | 等 Phase 6 快照数据积累 |
| 8 | 生产构建落后（b3c8187 + 2 修复未上线） | 🟡 | 待用户确认 promote |
| 9 | 文档「robots disallow」前提过时 | 🟢 | 文档勘误（本报告 §3） |
| 10 | `app/features/home/` 9 个组件（HomeSummary/HomeTabs/7 Tab）无引用 | 🟢 | 按规则⑤待用户确认删除 |
| 11 | 变现/API 商业化全无 | 🟢 | 文档 Phase 4，可长期规划 |
| 12 | `/as/[asn]` 301 跳转 | 🟢 | 有意保留（旧链接兼容），非垃圾 |

---

## 6. 差距优先级与工作量评估

| 优先级 | 事项 | 工作量 | 备注 |
|---|---|---|---|
| P0 | 生产 promote（b3c8187 + 2 修复） | 5 分钟 | 待用户确认；rsync + restart duoniu-preview + 生产冒烟 |
| P1 | 导航补 /purity（旗舰）+ /history + 工具下拉补齐 8 项 | 小 | 改 default.vue menus 数组，无风险 |
| P1 | purity 页两个待建卡落地 | 中 | 数据源可复用首页现成链路 |
| P2 | 首页 verdict/等级系统 | 中大 | **必须 staging 预览经用户确认**（七 Tab 教训） |
| P2 | 历史时间线/趋势图 | 中 | 依赖数据积累 |
| P3 | DNS 多解析器对比、路径地图、复制报告导出 | 中 | 差异化功能 |
| 维护 | features/home 9 组件删除确认；文档勘误 | 小 | 按规则⑤待确认 |

---

## 7. 结论

duoniu.cc 技术底座与 SEO 基建（Phase 0-6）已全面达标，文档所述「工具集齐备、缺重定位」的判断基本成立，且站点已有文档未认知的差异化资产（风控值、AbuseIPDB 黑名单行、大模型检测行、data-tip 微文案、实体关系图谱）。与文档提案的核心差距集中在**产品呈现层**：旗舰页不可见（导航缺失）、无等级 verdict 体系、历史对比未建、变现为零。文档路线图 Phase 1-3 的建议与现状缺口吻合，可作为下一阶段实施依据；涉及首页视觉的改动须坚持「staging 预览 → 用户确认」流程。

---

## 8. 附录

### 8.1 相关 commits（审计与修复）

```
47549f1 docs: add website analysis and repositioning proposal (user-provided)
3fcce29 fix: staging 500 - leftover t() call in UniversalSearch after language toggle removal
ce0e18a fix: remove leftover lang-toggle button, initLang call and styles (incomplete b3c8187 removal)
tag: staging-500-fix-20260816 → ce0e18a
```

### 8.2 staging 冒烟结果（修复后）

```
14/14 passed（/ /env /purity /asn /bgp /dns /whois /ping /trace /cidr /history /tools /about /faq 全 200 + baidu=true）
```

### 8.3 关键文件索引

| 文件 | 内容 |
|---|---|
| app/layouts/default.vue | 全局导航（menus 数组 §4 所述）、主题切换、面包屑 |
| app/pages/index.vue | 首页（旧堆叠布局，8 卡 + 14 行信息） |
| app/features/ip/purity/PurityPage.vue | 纯净度页（评分卡 + 2 待建卡） |
| app/features/history/IpHistoryPage.vue | 历史页（快照 + 时间线占位） |
| app/components/UniversalSearch.vue | 统一搜索框（本次修复文件） |
| app/features/home/ | 9 个无引用组件（待删） |
| scripts/deploy.sh | 部署流程（build → staging 冒烟 → --promote） |
| scripts/smoke-test.mjs | 14 页冒烟（staging/prod 双跑） |
