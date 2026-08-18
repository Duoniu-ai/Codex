# SiteIntel 全站分析报告审计 —— 白盒逐条核验

**审计对象**：外部黑盒分析《siteintel-full-site-analysis.md》（2026-08-15，线上抓取 + sitemap 枚举，24 节、§21 未实现清单 8 项、§22 缺陷清单 15 项）
**审计方法**：白盒三路对照 —— ① 服务器 git HEAD 全量源码（2026-08-16 快照）；② 线上页面实测（经 127.0.0.1:3003 直连，绕过 Cloudflare 缓存）；③ 生产 PostgreSQL 只读 SELECT（用户已授权）
**审计日期**：2026-08-16

---

## 0. 执行摘要

外部报告整体质量高：对模板体系、管线架构、SSE 异步模式的推断全部准确。15 项缺陷中 **12 项属实**（其中 2 项为 P0/P1 级活跃缺陷，含 1 项外部分析漏掉的高危 SSRF），**1 项为解析器伪影**（不可复现），**2 项为数据误读**。§21 的 8 项「未实现」中 **3 项已过时**（Baidu 连接器、搜索关联、监控告警均已上线，报告抓取时间早于或恰逢部署窗口）。

**需立即修（P0）**：
1. **provider_changed 轮询 DNS 假阳性**——生产库实测仍在每 30 分钟产生一条（wordpress.org 58/69 条全库事件）
2. **HTTP/metadata 采集器 SSRF**——外部报告未发现的新问题，公共分析端点可被恶意 DNS 指向内网

**审计结论汇总（§21 + §22 共 23 项）**：

| 结论 | 数量 | 项目 |
|---|---|---|
| 属实·已确认根因 | 12 | §22-1,2,3,4,5,6,7,8,10,11,12,13,14,15（14 项，其中 8/14 为低危确认） |
| 属实·已过时（已实现） | 3 | §21-Baidu 连接器、搜索关联、Webhook/Telegram |
| 属实·属设计决策 | 2 | §21-性能评分（§26 不凭空评分）、SSE 无鉴权（CUID 缓解，泄露面仅进度元数据） |
| 误报/不可复现 | 2 | §22-9 技术页标题异常（JSON-LD 解析伪影）、§23 的 127.0.0.1「测试数据」推断（真实 DNS 数据） |
| 已实现·无需处理 | 3 | §21-/report 报表索引、/bulk、AI 行动计划（休眠待 AI_API_KEY） |

---

## 1. §21 未实现功能清单 —— 逐项核验

| # | 外部报告断言 | 白盒核验结论 | 证据 |
|---|---|---|---|
| 1 | GSC 集成未实现（Phase 3 占位） | ✅ 属实 | 线上 `/search/google` 仍标 Phase 3，CTA 链到 `/`；代码中 google-search-console 无任何 provider 实现（lib/search/ 仅 baidu-official.ts） |
| 2 | Bing 集成未实现（Phase 2 占位） | ✅ 属实 | 同上，`/search/bing` 占位文案；无 bing 实现 |
| 3 | Baidu 站长平台未实现 | ❌ **已过时** | 2026-08-14 上线：SearchConnection 六表、lib/search/baidu-official.ts 链接提交 API（用户真实 token 实测 success:1 remain:8）、sync.ts 6h 调度器。线上 `/search/baidu` 文案已改为「连接百度站长数据」。**能力边界诚实标注**：索引量/关键词无公开 API，仪表盘明确显示「不可用」而非伪装 |
| 4 | 搜索-基础设施事件关联未实现 | ❌ **已过时** | lib/search/insights.ts `runSearchInsightRules` 已实现（search_regression / search_correlation，72h 先导关联，严禁因果表述）；报告页 03 区块已接真实搜索数据（report-page.tsx:285 `searchReport ?` 条件渲染 indexedPages 等） |
| 5 | 性能评分缺失 | ✅ 属实·**属设计决策** | audit.ts:366-374 performance 维度 status=unknown，明确「§26 原则：不凭空评分」。外部报告自己的分类（deliberately deferred, not a bug）正确 |
| 6 | Webhook/Telegram 告警未在公共 API 文档 | ✅ 半属实·**功能已上线** | lib/notify.ts Telegram + 签名 Webhook（HMAC-SHA256）双通道、SSRF 防护、lastNotifiedAt 去重、仪表盘设置+测试按钮，端到端已验证。仅不通过公共 API 暴露——属设计选择，非「文案先行」 |
| 7 | AI 解读行动计划不可验证 | ✅ **已实现（休眠）** | ai.ts:102 explainStrategy + /api/strategy/[domain]/explain + strategy-explain-button.tsx。无 AI_API_KEY 时 503 休眠且前端隐藏按钮——与外部报告「点击测试」推测一致，实现存在 |
| 8 | /report 行为不明 | ✅ **正常页面** | report/page.tsx 为分页报表索引（50/页、?page=N、noindex），列出全部已分析域名及最新调查状态 |
| 9 | /bulk 未验证 | ✅ **正常功能** | bulk API 登录门控（401）+ 每 IP 限流 + ≤20 域名 + 轮询状态；nav 直接链到 /bulk |

**修正后的真实未实现清单**：仅 GSC、Bing 两个连接器（各 1 个占位页），以及性能评分（有意为之）。外部报告对「搜索情报整个功能是死胡同」的定性在 08-14 之后已不成立。

---

## 2. §22.1 数据质量缺陷 —— 逐项核验

### 2.1 【P0·活跃】provider_changed 轮询 DNS 假阳性 —— 属实，根因已定位，仍在产生噪音

**生产库实测**（2026-08-16）：全库 69 条 provider_changed，**wordpress.org 独占 58 条**，时间戳每 30 分钟一条（监控 tick 节奏），最近一条 01:49。事件内容铁证：

```json
previous: {"dnsProvider": "ns3.wordpress.org", "emailProvider": "smtp1-dca.wordpress.org"}
current:  {"dnsProvider": "ns3.wordpress.org", "emailProvider": "smtp2-dca.wordpress.org"}
```

仅 nameserver/MX 在解析应答中的**顺序轮换**即触发「高」severity 事件（snapshots.ts:195-196）。

**根因**（外部报告推断完全正确，此处给出精确代码位置）：
- `lib/analyzers/infrastructure.ts:181-191`：`dnsProvider.name = known?.name ?? nameservers[0]` —— 取**未排序**的 NS 数组第一个元素；wordpress.org 的 ns1-4.wordpress.org 不匹配任何已知规则 → name 直接是裸主机名 → 轮询翻转
- `lib/analyzers/infrastructure.ts:196-204`：`emailProvider` 同理取 `mx[0]`（未按 MX priority 排序），smtp1-dca ↔ smtp2-dca 轮换
- 讽刺的是 183-184 行注释写着「sorted: resolver return order varies between runs」——**证据字符串排序了，name 没排**，半个修复
- Phase 0.5 的 hash 稳定性修复只解决了快照去重层（set 语义），没解决 analyzer 派生层
- 放大链：provider_changed(high) → provider_change 洞察(≥70%置信) + frequent_changes(30 天≥3 条) + correlation 引擎 infrastructure_migration 复合事件

**修复方案**（改动面小）：
1. `analyzeInfrastructure`：NS 先 `.sort()` 再取 [0]；MX 按 priority 升序排序后取最小者（同时修正「取错 MX」问题——MX 应答顺序与优先级无关）
2. 可选加固：provider_changed 事件生成时，若 previous/current 的槽位值集合相同（仅顺序不同）则抑制——但方案 1 已使 name 稳定，此条非必需
3. 历史噪音清理需用户确认（DELETE 58 条事件 + resolve 相关洞察，属生产数据变更，另行授权）

### 2.2 两个近似重复的洞察类型 —— 属实（设计层面）

`rules.ts` 两条独立规则：
- Rule 6（`infrastructure_migration`）：7 天窗口内 ≥2 种基础设施事件类型聚类
- Rule 10（`infrastructure_migration_event`）：correlation 引擎（Phase 5）同轮 ≥2 事实变化复合事件，3+ 事实 corroboration +6 分

线上 wordpress.org 报告两者并存（各 ×9/×2 提及）。设计意图可辨（窗口聚类 vs 同轮关联），但标题仅差一个词（"Possible ... Detected" vs "Detected"），证据高度重叠，无跨规则去重——外部报告「用户分不清到底发生了几件事」的判断成立。**建议**：合并展示层或明确差异化文案（如「短期多事件聚类」vs「同轮复合信号」），P2。

### 2.3 技术页重复计数 —— 属实，根因是 www 实体分裂

**生产库实测**：sitevance.tw 存在**两个 domain 实体**：
```
www.sitevance.tw  | firstSeen 2026-08-14 06:57   (uses→wordpress)
sitevance.tw      | firstSeen 2026-08-15 02:51   (uses→wordpress)
```
全库 **90 个 www.* 前缀 domain 实体**残留（2026-08-14 之前实体层未做 www 归并的遗留数据）。技术页展示时 `.replace(/^www\./, "")` 去 www → 两行同显示为 sitevance.tw（technology.ts:53），计数用 `domains.length` 而非去重后数量 → 页面头「3 个网站」与列表 4 个 chip 不一致。外部报告观察精确，机制为「www 实体分裂」而非「调查次数计数」——两者表象相同，根因不同。

**修复**：① technology.ts 组装时按 normalized 去重；② 一次性迁移合并 90 个 www 实体（关系重指向 apex）；③ entities.ts `normalizeValue` 对 domain 类型剥 www 防复发。P1。

### 2.4 组织实体碎片化 —— 属实，生产数据佐证

normalizeValue（entities.ts:419-433）对 organization 仅 lowercase+trim，无公司名归一化。生产库实际碎片：

| 同一真实主体 | 实体数 | 示例 |
|---|---|---|
| 阿里巴巴 | **8** | "Alibaba Cloud - HK" / "- US" / "Alibaba Cloud Computing (Beijing) Co., Ltd." / "Alibaba Cloud Computing Ltd. d/b/a HiChina" / "Alibaba Cloud LLC" / "Alibaba Cloud Singapore Private Limited" / "Aliyun Computing Co., LTD" / "Aliyun Computing Co.LTD"（注意最后两个仅差一个空格） |
| 中国电信 CHINANET | **4** | Beijing / Guangdong / Hubei / Fujian 各省网络分别成实体 |
| Akamai | 2 | "Akamai International, BV" vs "Akamai Technologies, Inc." |
| 同一地址（Collyer Quay 大厦） | 3 | "16 COLLYER QUAY" / "16 COLLYER QUAY 18-29 INCOME AT RAFFLES" / "6 COLLYER QUAY" |

外部报告判断成立。属 OSINT 数据固有问题的放大版——上游（ipwho.is/rdap.org）注册名不统一。**建议**：P2 建组织别名归一化表（手动 curated alias map 起步，如 Alibaba Cloud* → Alibaba Cloud），影响关系图谱聚合质量。

### 2.5 AS0 实体 —— 属实，是上游哨兵值无防护，非「未知桶」设计

**生产库实测**：asn 实体 normalized="0"，2026-08-15 05:28 创建，关系：
```
belongs_to | ip 2408:8706:0:4997::11 | asn 0   （×3 个 IPv6，China Unicom 段）
belongs_to | asn 0 | organization "China Unicom Network"
```
根因：上游对无 ASN 映射的 IPv6 返回裸 "0"，`extractEntities`（entities.ts:138-145）未做哨兵值过滤即建实体；`normalizeValue` 剥 AS 前缀后 "AS0"→"0"。外部报告的两个假设中第二个成立（解析边界未处理，非有意设计）。`/asn/AS0` 已在 sitemap（≥3 IP 关联过门）。**修复**：P1 在 entities.ts 加 asn==="0"/"AS0" 哨兵守卫，sitemap 层同步排除。

---

## 3. §22.2 SEO/元数据缺陷 —— 逐项核验

### 3.1 【P1】sitemap 首页重复条目 —— 属实，根因定位

线上实测：
```xml
<loc>https://siteintel.cc</loc>   <changefreq>daily</changefreq>   <priority>1</priority>
<loc>https://siteintel.cc/</loc>  <changefreq>weekly</changefreq>  <priority>1</priority>
```
与外部报告观察完全一致（daily vs weekly）。根因：sitemap.ts:142 `.filter((page) => page.url !== SITE_URL)` 精确字符串比较——registry 中 route="/" 的首页条目生成 `${SITE_URL}/`（带尾斜杠），与 `SITE_URL`（无尾斜杠）不等 → 漏过滤。**修复**：比较前统一去尾斜杠，或对 "/" route 特判。P1，一行改动。

### 3.2 【P1】实体页 OG 元数据全站默认值 —— 属实，代码级确认

线上 `/ip/154.204.176.66` 实测：
```html
<meta property="og:title" content="SiteIntel — 网站分析_域名情报_技术栈检测_基础设施查询">
<meta property="og:url" content="https://siteintel.cc">
<meta property="og:image" content="https://siteintel.cc/opengraph-image?21b98d5ed89ede37">
```
即根 layout（layout.tsx:35-42）的站级默认 OG 块。影响面比外部报告所述更大：**ip/asn/certificate/organization/technology 五类实体页**的 generateMetadata 均无 openGraph 块（grep 全无命中），全部继承首页 OG——社交分享全部显示首页品牌信息。网站报告页（/website/[domain]）有独立 opengraph-image.tsx 不受影响。**修复**：seo/entity.ts 装配数据后各实体页 generateMetadata 补 openGraph（title/description/url 至少；图片可复用首页 OG 图或按类型生成）。P1。

### 3.3 静态页 lastmod 全站一致 —— 属实（低危）

sitemap.ts 对全部静态页 `lastModified: new Date()`，且 `dynamic = "force-dynamic"` 每请求重建 → 所有 A–E 模板页 lastmod 同毫秒（实测 01:53:18.717Z 相同）。外部报告判断准确。影响有限（sitemap lastmod 仅是提示信号），与 Next.js 常规做法一致。**建议**：P3 可维护每页内容更新时间戳；不改也不构成实质风险。

### 3.4 技术页标题渲染异常 —— 不可复现（解析器伪影），另发现一个真实小缺陷

线上实测 `<head>`：
```html
<title>WordPress网站技术分析与使用情况 - SiteIntel</title>
```
标题渲染完全正确。外部报告所述「frontmatter title 为空 + 标题串出现在 body 末尾」几乎可以确定是抓取工具把页内 `<script type="application/ld+json">` 的 `"name"` 字段当作文本提取（JsonLd 组件在组件树末尾，正文最后一行）。当日页面已是正确实现（Phase 2 部署于报告抓取前）。

**但发现真实缺陷**：technology/[slug]/page.tsx:128 `generateMetadata` 内 `const isZh = true` 硬编码 → EN locale 用户拿到中文 title/description。P2 修复为读 locale。

---

## 4. §22.3 产品/UX 缺陷 —— 逐项核验

### 4.1 无法律/合规链接 —— 属实

site-footer.tsx 全站页脚仅标语两行，无 ToS/隐私政策/ Cookie 声明；仓库中无 /privacy、/terms 路由（grep 全仓仅 claim 页一处「隐私说明」）。外部报告判断成立，且其点出的产品性质（对外发布任意域名的基础设施画像与历史）确实需要数据来源/移除政策。P2。

### 4.2 无移除/退出流程 —— 属实

ownership.ts + /api/ownership 仅正向认领（claim/verify/status/revoke 自己的认领）；无「从 SiteIntel 移除我的域名」/ takedown 流程。与 4.1 合并处理（同一法律/产品决策）。P2。

### 4.3 SSE 无鉴权 —— 属实（低危，外部报告严重度判断准确）

stream/route.ts 无任何鉴权/限流；investigationId 为 Prisma CUID（schema @default(cuid())）不可枚举；泄露面仅进度步骤元数据（不含最终报告内容）。可选加固：对 userId 非空的调查要求同会话。P3。

### 4.4 重新分析按钮无限流提示 —— 属实

report-actions.tsx:18-28：非 2xx 响应仅 `setBusy(false)` 静默复位，429/retryAfterSec 不展示。/api/analyze 匿名端点限流 20/h/IP（route.ts:17）。外部报告判断成立。**修复**：展示错误文案（i18n 加 rateLimited 键）+ 可选倒计时。P2。

### 4.5 图谱超节点噪声 —— 属实（机制确认，设计风险）

report.ts `fetchRelationships` = 直接关系（≤100）+ 一跳（≤60），按 confidence 排序，**无相关性强弱过滤**。共享 GA/GTM tracking_id（百万级站点共享）与 nameserver 一跳即可引入 40+ 无关第三方域名。现有缓解：60 节点上限、类型筛选图例、≤30 边标签、degree≥2 才显示标签。related.ts 的 STRENGTH 强度分级（ssl 90 / tracking 85 / provider 35 / technology 10）**只用于「相关网站」列表，不用于图谱**。外部报告的规模风险判断成立。**建议**：P3 图谱接入强度阈值或默认隐藏 ubiquitous 技术/追踪 ID 边（可切换显示）。

### 4.6 搜索情报死胡同 —— 属实且有新发现

线上实测三个搜索引擎页 CTA 均链到 `/`：
- `/search/google`：3 处 href="/"（占位，合理）
- `/search/baidu`：**功能已上线，CTA 仍链到 `/`（bug）**

根因：seo-landing.tsx:114-116 三元 `sectionHref === "/search" ? "/search" : "/"`，但 PAGE_SECTIONS（(seo)/[...slug]/page.tsx:25-27）配置的 href 是 `"/search-intelligence"` 而非 `"/search"` → 三元永远走 `/`。属配置值失配，一处改动即可让三个搜索页 CTA 正确指向搜索中心（登录门控）。P1。另：/search/baidu 页文案残留 1 处 "Phase 3" 字样（i18n 未清干净），P2。

---

## 5. 新发现（外部报告未覆盖）

### 5.1 【P0·安全】分析管道 SSRF —— HTTP/metadata 采集器无解析后 IP 校验

**外部黑盒分析完全遗漏**（其 §22 无此项），但它的 §23 观察（127.0.0.1 进 sitemap）正是该向量的可见症状。

- `providers/http.ts`：对用户提交域名直接 `fetchWithTimeout("https://{domain}/")`，**逐跳跟随重定向**（Location 任意目标，包括内网 URL），无 DNS 解析校验、无重定向目标校验
- `providers/metadata.ts`：同样模式 ×3（首页、/robots.txt、/sitemap.xml）+ favicon
- `providers/ssl.ts`：TLS 握手同域（同类别）
- 入口是**公共匿名** POST /api/analyze（20/h/IP 限流可被 IP 轮换绕过）
- ssrf-guard.ts（DNS 解析后封私网/回环/元数据地址）**仅应用于 webhook**（notify.ts + settings 路由），未覆盖采集管道
- 影响：恶意方控制自己域名的 DNS → 指向 127.0.0.1/内网/169.254.169.254 → 服务器发起 GET（仅 80/443 端口）→ 响应状态/头部/重定向链存入 RawCollection 并在**公开报告页证据区展示**（§9.14 的透明设计反成泄露通道）；IMDSv1 场景下可泄露云元数据
- **生产库已有自然证据**：ddooo.com、360kan.com、findlaw.cn 三个真实域名解析到 127.0.0.1（这些站点自己发布了回环 A 记录，疑似防爬策略），探测已打到 SiteIntel 服务器本机并入库——不是攻击，但证明了向量现实存在且已被有机数据触发

**修复方案**（P0）：
1. 公共采集统一走安全 fetch 包装：先 `dns.resolve4/resolve6` 解析目标，逐地址断言公网（复用 ssrf-guard 的范围判定），再按 IP 连接并带 Host/SNI（或 undici Agent + 自定义 lookup 拒绝私网——一次性覆盖重定向所有跳）；重定向每一跳都做同样的解析校验
2. TLS 采集器同样按解析后 IP 校验
3. 附带收益：ddooo.com 类站点的探测将明确失败而非打到本机，127.0.0.1 实体自然不再增长

### 5.2 127.0.0.1 实体页 —— 真实数据，非测试种子（外部报告推断错误）

上述三个域名的 A 记录真实解析到 127.0.0.1（上游 DNS 数据如实入库，采集器行为正确）。外部报告「几乎肯定是测试/种子数据或采集器 bug」的推断**不成立**。但下游处理仍应改善：回环/私网 IP 实体页应排除出 sitemap 并在页面标注「回环地址」——否则如本报告所示，任何外部审查者都会把它当 bug。P2。

### 5.3 监控调度器放大假阳性

wordpress.org 的 58 条 provider_changed 全部由 30 分钟监控 tick 产生（时间戳 00:19/00:49/01:19 节奏铁证）。调度器本身按设计工作，但意味着：**一个被监控的「unknown NS」域名 = 每天 48 条高 severity 假事件 + 洞察持续活跃**。2.1 修复后此放大自动消失；修复前建议对 wordpress.org 监控暂停或容忍。

### 5.4 技术页 metadata 硬编码中文（见 3.4，P2）

---

## 6. 外部报告事实性勘误

| 位置 | 外部报告说法 | 实际情况 |
|---|---|---|
| §7/§21 | 百度连接器未实现，三引擎同模板 | 百度已实现并真实验证（08-14）；Google/Bing 仍占位 |
| §21 | 搜索-基础设施关联「完全依赖连接器、未实现」 | 关联规则引擎已实现；缺的只是 Google/Bing 数据源 |
| §21 | 监控告警「要么内部功能要么文案先行」 | 内部功能，双通道实测通过 |
| §23 | /ip/127.0.0.1「几乎肯定是测试/种子数据或采集器 bug」 | 真实 DNS 数据（ddooo.com/360kan.com/findlaw.cn 自发布回环 A 记录） |
| §14.2/§22.9 | 技术页 title「SSR 元数据渲染 bug」 | 不可复现；JSON-LD name 字段被提取器误当正文的伪影 |
| §9.12 | 「36 个事件」 | 全库该类型 69 条、wordpress.org 58 条（节奏=监控 tick 30 分钟），核心判断正确 |
| §16.4 | 「公共 API 文档相对产品功能不完整」 | 监控/告警是登录后仪表盘功能，docs/api 有 webhookNote 说明——设计选择，表述成立但非缺陷 |
| §23 | sitemap 110 URL | 审计时已 131（语料库增长正常） |
| §2.2 | UI 提交到 POST /api/v1/analyze | UI 实际走 /api/analyze（匿名 20/h）；v1 需 X-API-Key——推断小错 |
| §1/§4.3 | 首页统计数动态 | ✅ 正确（page.tsx 实时 COUNT 查询） |

---

## 7. 修复优先级汇总

| 优先级 | 项 | 位置 | 工作量 |
|---|---|---|---|
| **P0** | 采集管道 SSRF（5.1） | providers/http.ts、metadata.ts、ssl.ts + 安全 fetch 包装 | 中（半天级） |
| **P0** | provider_changed 轮询假阳性（2.1） | analyzers/infrastructure.ts:180-205（NS 排序 + MX 按 priority 排序） | 小（分钟级）+ 历史数据清理需授权 |
| P1 | 搜索页 CTA 路由失配（4.6） | (seo)/[...slug]/page.tsx:25-27 href 改 "/search" | 一行 |
| P1 | sitemap 首页重复（3.1） | sitemap.ts:142 尾斜杠归一 | 一行 |
| P1 | 实体页 OG 块（3.2） | 五类实体页 generateMetadata 补 openGraph | 小 |
| P1 | AS0 哨兵守卫（2.5） | entities.ts:138-145 | 小 |
| P1 | 技术页 www 实体去重 + 90 实体迁移（2.3） | technology.ts + 迁移脚本 + normalizeValue 剥 www | 中（迁移需确认） |
| P2 | 技术页 metadata isZh 硬编码（3.4/5.4） | technology/[slug]/page.tsx:128 | 一行 |
| P2 | 重新分析 429 提示（4.4） | report-actions.tsx + i18n | 小 |
| P2 | 法律页 + 移除流程（4.1/4.2） | 新增 /privacy /terms + 页脚链接 | 中 |
| P2 | 组织别名归一化（2.4） | 新增 alias 表/映射 | 中 |
| P2 | 127.0.0.1 类回环页处置（5.2） | sitemap + 页面标注 | 小 |
| P2 | 洞察文案差异化/合并（2.2） | rules.ts + insight-text.ts | 小 |
| P3 | 图谱超节点过滤（4.5） | relationship-graph/report.ts 强度阈值 | 中 |
| P3 | SSE 可选鉴权（4.3） | stream/route.ts | 小 |
| P3 | 静态页 lastmod 内容时间（3.3） | sitemap.ts | 小 |

---

## 8. 附：生产库证据快照（2026-08-16，只读）

- Event 总量 95：provider_changed 69（wordpress.org 58）/ infrastructure_migration 7 / technology_added 5 / ip_changed 4 / dns_changed 4 / website_overhaul 2 / metadata_changed 2 / ssl_changed 1 / rdap_changed 1
- www.* 前缀 domain 实体：90 个（最早 2026-08-14 04:32）
- AS0 实体关系：3 条 IPv6 belongs_to（2408:8706:0:4997::11/12/13）+ 1 条 → China Unicom Network
- 127.0.0.1 实体：3 条 resolves_to（ddooo.com / 360kan.com / findlaw.cn），证据串 "A/AAAA record resolved to 127.0.0.1"
- wordpress.org provider_changed 最新 6 条时间戳：2026-08-16 01:49、00:49、00:19、08-15 23:49、23:19、22:49（30 分钟节奏）
- 组织实体 160+，其中 Alibaba 系 ≥8 变体、CHINANET 4 省、Collyer Quay 3 变体

*审计报告完。本报告基于 git HEAD（commit 5109a91 起全量）+ 线上实测 + 生产库只读查询；修复项未动任何生产代码与数据。*
