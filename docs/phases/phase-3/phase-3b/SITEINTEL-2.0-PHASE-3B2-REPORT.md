# SITEINTEL-2.0-PHASE-3B2-REPORT.md

SiteIntel 2.0 Phase 3B.2 — 旗舰报告页「变化与证据链」精修
执行时间：2026-08-16 · 范围：仅 /website/[domain] 前台展示 · **禁止事项全部遵守**（无数据库 Schema / Event Rule / Insight Rule / Discovery / Pipeline 改动）

---

## 1. 修改文件

| 文件 | 修改 |
|---|---|
| `src/lib/types.ts` | ReportHistoryEntry 增加 `investigationId`/`evidence`（组装层透传既有字段）；ips 增加 `country/city/isp` 可选字段 |
| `src/lib/report.ts` | ① history 映射透传 investigationId + evidence；② 从 IP 实体 metadata 组装位置信息（**复用实体层既有数据，非新采集源**） |
| `src/lib/intelligence/event-diff.ts`（新） | 变化具体化（describeEventDiff 按 22 类真实 Event payload 提取 before/after）、同轮网络类事件前台聚合（aggregateEventItems）、证据元数据描述（describeEventEvidence） |
| `src/lib/intelligence/event-diff.test.ts`（新） | 15 例：IP/DNS/metadata/provider diff、无 diff 诚实降级、复合事件不伪造、聚合三场景、证据描述 |
| `src/components/report/report-page.tsx` | 第 9 张「网络位置」卡；变化卡/变化行接入具体 diff + 聚合 + 「查看证据 ↓」锚点；值得关注排序（重要度→证据丰富度→置信度） |
| `src/lib/i18n.ts` | 新增 6 键（netLocation/multiNetLocations/noNetLocation/evidenceJump/evidenceAnchor 等，zh/en 对称） |

## 2. 网络与基础设施第 9 张卡：网络位置

- **数据来源**：IP 实体 metadata（country/city/isp——IP 富集时已入库的既有数据），report 组装层按 resolved IP 关联，**零新数据源**
- **表达**：单一国家 → 「国家 · 城市」；多国家 → 「检测到多个网络位置」+ 每 IP 一行（地址 · 国家 城市，最多 4 行 + 总数）；无数据 → 自然空态「暂无网络位置数据」（downcc.com 实测命中）
- **不伪造**：位置全部来自真实实体元数据

## 3. 变化具体化（真实 before/after）

| Event 类型 | 具体化方式（生产实测） |
|---|---|
| ip_changed | `IP 地址：2620:109:b00a::4206:2afc → 2620:109:b00a::4206:2afc、66.6.42.252` |
| dns_changed | `DNS 记录：新增 1 条、移除 2 条` + 展开「移除：A 1.2.3.4、NS ns1… / 新增：A 5.6.7.8」（集合差异） |
| metadata_changed | `网站信息：生成器 发生变化` + 字段级 before/after（标题/描述/语言/robots/sitemap，布尔转「有/无」） |
| provider_changed | `N 项服务商变化` + 槽位级（CDN/托管/DNS 服务商/邮箱服务商） |
| nameserver/ssl/rdap/technology/status | 各自结构提取（NS 列表、签发机构/指纹、注册商/到期时间、技术名、HTTP 状态码） |
| **无可靠 diff** | **headline=null 诚实降级**为泛化标题 + 证据入口（不伪造） |
| 复合事件（infrastructure_migration/website_overhaul） | 无 before/after 不编造——展示「包含变化：dns_records、resolved_ips」+ 证据 |

## 4. 同轮事件前台聚合

同一 investigationId 内 ≥2 个网络类事件（ip/dns/nameserver）→ 展示层聚合为「**检测到网络配置变化**」卡，展开逐条列出各成员变化及具体 diff。数据库 Event 零改动，纯展示聚合（wordpress.org 实测 ×2 组合生效；不同调查/无调查 ID 不聚合）。

## 5. 证据链

每个变化卡/行提供「**查看证据 ↓**」：证据元数据一行（detail · 检测维度 factType · 快照哈希前 8 位）→ 锚点跳转「原始检测记录」（06 深入数据，含 Snapshot/RawCollection 原始数据）。零新数据库模型。

## 6. 「网站信息发生变化」专项

按 Event payload 具体化：有字段级数据 → 「网站信息：标题/描述/生成器… 发生变化」+ 逐字段 before/after；无字段级数据 → 保持诚实泛化标题 + 证据详情入口。绝不编造字段。

## 7. 值得关注排序

前台排序：重要度（critical/high 优先）→ **证据丰富度（evidence 数降序）** → 置信度——有具体证据支撑的发现优先，泛化低信息量事件不再占据突出位置。

## 8. 验证结果

| 项 | 结果 |
|---|---|
| typecheck / lint / test / build | ✅ / ✅（改动文件 0 问题）/ ✅ **178/178**（新增 15 例）/ ✅ |
| wordpress.org | 第 9 张卡 ✓；IP/DNS/metadata 具体 diff ✓；聚合 ×2 ✓；证据跳转 ✓；术语 0 ✓ |
| duoniu.cc | 第 9 张卡 ✓；术语 0 ✓ |
| downcc.com | 第 9 张卡自然空态 ✓；术语 0 ✓ |
| baidu.com | 第 9 张卡 ✓；术语 0 ✓ |
| 无证据事件不伪造 | ✅ 复合事件/无 diff 事件实测降级 |
| Event 与 Insight 不重复堆叠 | ✅ 变化区只渲染事件（人化+diff）；值得关注区只渲染洞察（排序优化） |
| DNS+IP 同轮聚合 | ✅ wordpress.org 实测「检测到网络配置变化」 |
| 原始数据可访问 | ✅ 证据锚点 → 深入数据折叠区可展开 |
| 中英文一致 | ✅ EN 实测 Network location / View evidence / IP address diff |
| SEO 未破坏 | ✅ title 不变 |

## 9. 已知问题

- 事件证据跳转使用页面内锚点（#evidence），单页滚动定位；若证据区默认折叠，用户到达后需再点一次展开（可接受）。
- 聚合仅覆盖网络类（ip/dns/nameserver）；其他类型同轮事件仍单条展示（无需求扩大）。

## 10. 回滚方式

`git revert` 或 checkout 6 个文件至父 commit + 重建重启；纯前端/组装层，零数据依赖。

---

*Phase 3B.2 完成。未进入 3C。*
