# SITEINTEL-2.0-PHASE-3B-REPORT.md

SiteIntel 2.0 Phase 3B — 旗舰网站分析结果页用户化重构实施报告
执行时间：2026-08-16 · 核心页面：/website/[domain] · 依据：BLUEPRINT + PHASE-1-AUDIT + PHASE-2-IA + PHASE-3A 实际状态

---

## 1. 修改文件

| 文件 | 修改内容 |
|---|---|
| `src/components/report/report-page.tsx` | 页面重排为六层用户理解路径 + 一句话结论接入 + DNS 记录折叠 + 搜索数据下沉 + 导出下沉深入数据 + 事件/关联文案人化 |
| `src/components/report/insight-card.tsx` | 移除机器 slug 展示；置信度下沉到「为什么这样判断」展开区 |
| `src/lib/intelligence/event-text.ts`（新） | 前台事件语言转换器：22 类真实 Event 类型 → 用户标题 + 「为什么值得看」一行（中英双语，未知类型兜底） |
| `src/lib/report-verdict.ts`（新） | 一句话结论组装器（纯函数）：只拼接真实数据，缺数据省略分句，无数据返回 null |
| `src/lib/i18n.ts` | zh eventTypes 补齐 4 类缺失 + 全量改写；**en eventTypes 从原始 slug 全量翻译**（22 类）；区块名用户化（网站概览/它如何运行/最近发生了什么/值得关注/它与什么有关联/深入数据）；relCols 去「实体」；reportTitle/reportDesc 用户化；新增 whyJudged 键 |
| `src/lib/intelligence/event-text.test.ts`（新） | 22 类全量映射 + 双语 + 未知类型兜底测试 |
| `src/lib/report-verdict.test.ts`（新） | 完整/降级/离线/无数据四种结论生成测试 |

## 2. 页面修改前后结构

| 前 | 后 |
|---|---|
| 头部 → 01 执行摘要 → 02 关键洞察（slug 可见）→ 03 搜索情报（空态占第 3 位）→ 04 这是什么网站 → 05 基础设施 → 06 技术 → 07 健康度 → 08 行动路线 → 10 历史（raw slug）→ 09 关系 → 11 相关网站 → 12 原始数据与证据 | 头部 → **一句话结论** → 01 网站概览（执行摘要+这是什么网站合并）→ 02 它如何运行（基础设施+技术，DNS 折叠）→ 03 最近发生了什么（人化事件+原因）→ 04 值得关注（无 slug 洞察）→ 05 它与什么有关联（相关网站+关联图谱）→ 06 深入数据（健康度+行动路线+搜索+原始证据+导出） |

## 3. 一句话结论实现方式

`lib/report-verdict.ts` `buildVerdict(input, locale)`——纯函数按条件拼接：技术身份（有技术才生成）→ 网络服务（cdn/hosting/dnsProvider 任一真实存在）→ 在线状态（有探测结果才说）→ 30 天变化数（>0 才说）→ 重点项数（critical/high 洞察 >0 才说）。**生产实测**：wordpress.org 生成完整结论；downcc.com（无技术数据）自动降级为无结论段落——绝不编造「正常/安全/可靠」。

## 4. Event 用户语言映射

新 `event-text.ts` 全量覆盖 12 类基础设施 + 10 类搜索事件：`provider_changed`→「网站服务商发生变化」+ 原因行；`ip_changed`→「检测到新的 IP 地址」；`infrastructure_migration`→「检测到基础设施变化」……每个事件在页面上回答三问：**发生了什么（人化标题）/ 什么时候（日期）/ 为什么值得看（why 行）**；before→after 值保留为展开行。i18n `eventTypes` 同步补全与改写（en 侧此前是原始 slug，已全量翻译）。

## 5. Insight 用户语言映射

卡片结构改为：重要度标签 → **人化标题** → 一句话说明 → 首次发现时间 → 「▸ 查看依据 (N)」；展开后首行「**为什么这样判断**」+ 置信度 → 证据列表（含事件人化标签与时间）。机器 slug（`insight.type`）从默认视图**移除**；置信度不再出现在卡片顶部。

## 6. Relationship 用户语言映射

关系类型标签沿用已有 `relTypeLabels` 人化映射（解析到/托管于/使用/共享/归属/跳转到/关联）；关系表列头「源实体/目标实体/关系」→「关联对象/关联对象/关联」；区块标题「关系」→「它与什么有关联」；「N 条关系」→「N 条关联」；相关网站表（真实共享证据）置于图谱之前作为首层关联表达。

## 7. 删除/下沉的管理员语言

| 项 | 处置 |
|---|---|
| 洞察卡机器 slug（`{insight.type}`） | 删除 |
| 历史行 raw slug | 替换为人化标题；before→after 值保留 |
| 「调查次数/最新运行/COMPLETED」头部行 | 保留但弱化（专业信息，字号极小） |
| 「执行摘要」 | →「网站概览」 |
| 「历史」 | →「最近发生了什么」 |
| 「关系」 | →「它与什么有关联」 |
| 「原始数据与证据」 | →「查看完整数据」+ 移入深入数据层 |
| DNS 全表首屏直出 | 折叠为 details（点击展开） |
| 搜索情报空态占据第 3 位 | 下沉深入数据层（连接状态不打断用户主线） |
| 导出 JSON 按钮（头部） | 下沉深入数据层底部 |

## 8. 数据来源复用情况

全部复用现有后端数据，零新查询：summary/overview/infrastructure/technologies facts、events、insights、relationships、related、audit、recommendations、searchReport、rawCollections。新增仅两个**纯展示层函数**（event-text / report-verdict），不触碰数据模型。

## 9. 是否修改 API

**否。** 页面仍消费 assembleReport 同一份 JSON（含 /api/report/[domain] 导出）。

## 10. 是否修改数据库

**否。** 无 schema/migration/数据变更。

## 11. SEO 修改

reportTitle：`{domain} 网站分析 - 技术、基础设施与变化`（zh）/ `{domain} Website Analysis — Technology, Infrastructure & Changes`（en）；reportDesc 同步用户化（中英对称）。canonical/OG/robots 结构不变，实体 SEO 能力保持。生产实测 title 正确。

## 12. 测试结果

| 项 | 结果 |
|---|---|
| typecheck | ✅ |
| lint | ✅（1 处 react-hooks/purity 对服务器组件 Date.now() 的告警已加说明性行级豁免） |
| test | ✅ **166/166**（新增 28 例：event-text 22 类映射+兜底、report-verdict 4 场景） |
| build | ✅ |

## 13. 生产验证（4 个样本）

| # | 检查 | wordpress.org | duoniu.cc（普通站） | downcc.com（数据少） | baidu.com（变化多） |
|---|---|---|---|---|---|
| 1 | 页面正常 | 200 ✓ | 200 ✓ | 200 ✓ | 200 ✓ |
| 2 | 一句话结论不编造 | 完整结论 ✓ | 完整结论 ✓ | **无结论（自动降级）✓** | 生成 ✓ |
| 3 | Event 文案人化 | 「网站服务商发生变化」+ 原因行 ✓ | — | — | ✓ |
| 4 | Insight 文案人化 | 「为什么这样判断」+ 查看依据 ✓ | ✓ | ✓ | ✓ |
| 5 | Relationship 文案人化 | 关联对象列头 ✓ | ✓ | ✓ | ✓ |
| 6 | 机器术语可见正文 | **0** ✓ | **0** ✓ | **0** ✓ | **0** ✓ |
| 7 | 原始详情可访问 | DNS 折叠可展开 ✓ | ✓ | ✓ | ✓ |
| 8 | 无数据缺失 | 六层全渲染 ✓ | ✓ | ✓ | ✓ |
| 9 | SEO 正常 | title 新格式 ✓ | — | — | — |
| 10 | 首页→分析页路径 | 首页示例卡 → 200 ✓ | — | — | — |
| 附加 | 英文版 | 六层 EN 标题 + EN 结论 ✓ | — | — | — |

## 14. 回滚方式

`git revert` 本 commit，或 `git checkout <parent> -- <7 个文件>` + 重建重启；纯前端无数据依赖。

## 15. 已知问题

1. 首页/报告页内嵌字典序列化载荷仍含内部术语（客户端组件共享全量字典的既有现象，不可见、无 SEO 影响——3C 字典拆分候选，超出本阶段范围）。
2. `react-hooks/purity` 规则对服务器组件 `Date.now()` 的既有告警以行级豁免处理（附说明注释），未全局放宽规则。
3. 部分站点 hosting 槽位为组织名（如「WordPress.org」）时，结论句为「通过 WordPress.org 提供网络服务」——数据真实但读感略生硬，属数据层现象，未在前台强行改写。

## 16. 未处理内容

导航（3C）、页脚、首页后续微调、实体页/工具页/搜索页（3D）、内嵌字典拆分、关系图谱超节点精选策略、/report 索引页。

---

*Phase 3B 完成。未进入 3C/3D。*
