# SiteIntel 上游数据能力审计报告 — 网址发现平台架构阶段 1

> 依据《网址发现平台-完整产品规划-V2.1.2-最终冻结版》§23 路线第 1 步执行。
> 审计对象:SiteIntel 生产代码基线(本地 `C:\Users\deepo\siteintel` = 线上 @ 1bdfcb4,2026-08-17 确认)。
> 审计目标:确认 SiteIntel 现有 API 与数据结构能否支撑导航站的单向同步需求,识别缺口,给出同步契约草案。

---

## 一、审计结论(先说结论)

**PASS WITH 2 GAPS。**

1. SiteIntel 现有开放 API v1(report 端点)已覆盖导航站 §11 需求的 **90% 以上**——域名基础信息、DNS/IP/ASN、SSL/CDN、技术栈、可访问状态、快照摘要全部可经 `GET /api/v1/report/{domain}` 单端点获取,无需新增采集能力。
2. **缺口 1(阻断冷启动节奏)**:批量触发分析被 `v1/analyze` 的 30/h 硬顶限制(代码 `Math.min(key.rateLimitPerHour, 30)`)。冷启动 300~500 域名首灌需 10~17 小时。
3. **缺口 2(影响 P1 更新提醒)**:变化信号无独立增量端点。`v1/report` 只带最近 50 条事件历史,无法"自某时间点以来"查询;"收藏更新提醒"(P1)需要它。
4. 非缺口优化项:report 返回 `evidence.rawData` 全量原始数据,对"导航站只存摘要"原则过重——导航站侧裁剪即可,无需 SiteIntel 改动。
5. 架构硬规则(冻结版 §11"不耦合 SiteIntel 内部数据库结构,只经稳定 API/事件/同步契约")**可行且已被现有 v1 API 支持**;`/api/report/[domain]` 存在无鉴权公开 JSON 端点,导航站应使用 v1+Key 而非该端点。

---

## 二、SiteIntel 现有 API 能力盘点

### 2.1 开放 API v1(消费方:导航站)

| 端点 | 鉴权 | 限流 | 说明 |
|---|---|---|---|
| `POST /api/v1/analyze` | X-API-Key | 30/h 硬顶(min(key.rateLimitPerHour,30)) | 触发全管线分析(7 provider),返回 investigationId/reportUrl |
| `GET /api/v1/report/{domain}?lang=zh\|en` | X-API-Key | key.rateLimitPerHour(默认 100/h) | 返回完整 IntelligenceReport JSON;未分析 404;lang 默认 en |
| `POST /api/v1/bulk` | 登录 session(非 Key) | — | 批量分析 ≤20 域名,**API Key 不可用** |

### 2.2 其他相关端点(导航站不使用)

- `GET /api/report/{domain}` — 与 v1 同数据但**无鉴权公开**。存在理由:调试/未来公开 API。**不建议**作为同步通道(无 Key 无法审计、无配额)。
- `GET /api/tools/dns|ip|ssl` — 匿名 20/h 工具端点,单点查询,不适合批量。
- 内部路由(monitor/ownership/search/keys 等)全部需要登录 session。

### 2.3 鉴权与配额机制(已具备,需配置)

- Key 格式 `si_<48 hex>`,服务端只存 sha256 哈希 + 8 字符前缀;明文仅在创建时展示一次。
- `ApiKey.rateLimitPerHour`(默认 100,可配置)、`quotaMonthly`(null=不限)——**给导航站建专用 Key 时直接把 rateLimitPerHour 调高即可,无需代码改动**。

---

## 三、数据结构盘点(与导航站需求映射)

`IntelligenceReport`(types.ts:288)顶层结构,逐项对照冻结版 §11"SiteIntel 负责"清单:

| 导航站需求(冻结版 §11) | IntelligenceReport 字段 | 现状 |
|---|---|---|
| 域名基础信息 | `target.{domain,firstSeenAt,lastSeenAt,factCoverage}` | ✅ |
| DNS | `infrastructure.dns.{records,nameservers,provider}` | ✅ |
| IP / ASN | `infrastructure.ips[]`(address/version/asn/org/country/city/isp) | ✅ |
| SSL | `infrastructure.ssl`(subject/issuer/san/validTo/daysRemaining…) | ✅ |
| CDN | `infrastructure.cdn.{name,confidence}` | ✅(推断,带置信度) |
| 托管 | `infrastructure.hosting.{name,hidden,confidence}` | ✅(含"藏在 CDN 后"诚实标记) |
| 技术栈 | `technology[]`(name/category/confidence/version/firstSeenAt/lastSeenAt) | ✅ |
| 网站检测 / 可访问状态 | `overview.websiteStatus.{online,status,checkedAt}` | ✅ |
| 快照摘要(导航站只存摘要) | `overview.{title,description,summaryText,categories,inferences}` | ✅(summary 双语已生成) |
| 网站状态与变化信号 | `history[]`(最近 50 条 Event:type/severity/detectedAt/previous/current/evidence) | ⚠️ 有数据无增量端点 |
| SEO 爬取信号(可选增强) | `overview.{hasRobots,hasSitemap,wordCount,h1Count,internalLinks,…}` | ✅ 现成,详情页决策辅助层可用 |
| 健康度/审计(可选增强) | `audit`(六维评分)/`recommendations`/`contradictions` | ✅ 现成,注意口径(站内自评,非质量排名依据) |

**注意口径边界(对应冻结版 §23.3)**:
- `overview.websiteStatus.online` 是"最近一次检测可访问",不是"持续在线"——导航站 UI 用"最近检测正常访问"表述。
- `audit` 是 SiteIntel 自身的规则审计(安全头/SEO 信号),**不能**解读为"网站质量评分"。
- `insights` 是变化洞察(active 状态),可作为详情页"值得关注"素材,需透传证据链。

---

## 四、缺口与建议方案

### 缺口 1:v1/analyze 批量触发 30/h 硬顶(阻断冷启动节奏)

- 证据:`src/app/api/v1/analyze/route.ts:23` — `checkRateLimit(..., Math.min(key.rateLimitPerHour, 30), ...)`,硬编码 30/h。
- 影响:冷启动 30~50 任务 × 5~10 站 ≈ 300~500 域名,首次触发分析需 10~17 小时,拖慢"首灌→建页"节奏。
- 建议(SiteIntel 侧,3 选 1):
  - A(推荐,最小改动):`v1/analyze` 的硬顶从 `Math.min(key.rateLimitPerHour, 30)` 放开为按 key 配置(或提高到 200/h);导航站 Key 单独配高配额。
  - B:新增 `POST /api/v1/analyze/bulk`(≤20 域名,复用 bulk 逻辑,走 Key 鉴权)——与 web bulk 重复,开发量大,不优先。
  - C:接受 30/h,冷启动期分批(≈10 小时),后续维护量小(每批 ≤30),可作临时方案。
- 验收:导航站专用 Key 连续触发分析不被 429(500 个域名窗口)。

### 缺口 2:变化信号无增量端点(影响 P1 收藏更新提醒)

- 现状:`v1/report` 的 `history` 固定取最近 50 条;事件表(Event/Fact/Signal)有完整数据但 v1 未暴露查询。
- 影响:P0 的"网站状态与变化信号"可靠"定期重拉 report 对比本地摘要"实现(P0 不阻断);但 P1"收藏更新提醒"需要"我收藏的网站最近有什么变化",增量查询是正解,重拉全报告成本高(每个网站一次完整 JSON)。
- 建议(SiteIntel 侧新增 1 个端点):
  - `GET /api/v1/events?domain=X&since=<ISO>&category=&limit=50` → 返回 `[{type, category, severity, detectedAt, previous, current}]`(裁剪 evidence 原始指针,不含敏感字段)。
  - 与现有 `v1/report` 同鉴权/限流/配额路径,成本低(直接查 Event 表,复用 report 的聚合逻辑)。
- 降级方案(若 SiteIntel 侧暂不开发):导航站 P0 用"摘要级 diff"(本地存上次拉取的摘要 hash,重拉对比);P1 更新提醒延后到该端点上线。

### 优化项(非缺口):report 载荷裁剪

- `report.evidence[]` 带每个 provider 的 `rawData` 全量原始数据,单域名 JSON 可达数百 KB;导航站"只存摘要"不需要它。
- 方案:导航站同步层消费时忽略 `evidence` 字段(客户端裁剪,零 SiteIntel 改动);如未来同步网站数上规模,再请求 SiteIntel 加 `?summary=1` 裁剪参数。

---

## 五、同步契约草案(导航站侧设计,供阶段 2 技术架构细化)

### 5.1 同步方式

- 服务端 API Key:在 SiteIntel 建专用 Key(命名 `navigation-sync`,rateLimitPerHour 按缺口 1 方案配置)。
- 拉取:`GET https://siteintel.cc/api/v1/report/{domain}?lang=zh`,带 `X-API-Key`。
- 周期:每日 tick(对齐 SiteIntel 监控节奏);变化信号走缺口 2 端点(上线后)。
- 幂等:同一域名重复拉取无害(本地 upsert,摘要 hash 变化才更新)。

### 5.2 导航站本地存储(摘要层,对应冻结版 §15)

```text
website_status 表(新增)
- domain            ← target.domain
- online            ← overview.websiteStatus.online
- http_status       ← websiteStatus.status
- checked_at        ← websiteStatus.checkedAt(可验证表述)
- title / summary   ← overview.title / summaryText
- categories[]      ← overview.categories
- technologies[]    ← technology[].name
- cdn / hosting     ← infrastructure.cdn.name / hosting.name
- ssl_expires_at    ← infrastructure.ssl.validTo
- last_sync_at      ← 本地
- summary_hash      ← 本地摘要序列化 hash(变化检测用)
```

- 存储原则:只存摘要,不存全量 report、不存 evidence.rawData、不存 DNS 记录明细(§11 单向同步语义)。

### 5.3 变化信号口径(冻结版 §23.3 硬规则落实)

- 导航站 UI 只可表述:"最近检测正常访问 / 数据更新时间 / 近期检测到页面或技术变化(透传 Event type)/ SSL 将于 X 过期"。
- 禁止表述:"网站活跃 / 运营活跃 / 业务增长 / 质量更高"——即使 report.audit 或 history 存在也不使用这些词。

### 5.4 错误处理

| 场景 | 行为 |
|---|---|
| 404(未分析) | 导航站入队 → `POST v1/analyze` 触发 → 回填后重拉 |
| 429 | 指数退避(60s/120s/240s),记入同步日志 |
| SiteIntel 不可达 | 本地缓存兜底,状态标"数据更新时间:X",不阻塞页面 |
| Key 失效 | 同步日志告警,人工换 Key |

### 5.5 冷启动节奏(缺口 1 未修时的保守排期)

- 300~500 域名 ÷ 30/h ≈ 10~17 小时首灌 → 建议缺口 1 采用方案 A 后再评估;若不做,首灌可与"编辑撰写任务页内容"并行,不阻塞 Phase 1 主线。

---

## 六、结论与下一步

1. SiteIntel 上游数据能力**满足导航站 P0 全部需求**(经 v1/report + 摘要裁剪)。
2. 需 SiteIntel 侧小改动 2 项(均可独立小票):① v1/analyze 硬顶放开;② 新增 v1/events 增量端点。建议排入 SiteIntel 下一迭代,导航站 Phase 1 开发不阻塞(有降级方案)。
3. 同步契约草案(§5)作为阶段 2"技术架构"的输入之一;数据库 Schema 阶段与 website_status 表设计合并细化。

下一步:**网址发现平台技术架构(阶段 2)**——整体架构选型(Next.js 单体 vs 前后端分离、与 SiteIntel 同栈复用 vs 独立)、路由与页面结构、任务/搜索/推荐模块设计。
