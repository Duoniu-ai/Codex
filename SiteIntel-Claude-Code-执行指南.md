# SiteIntel 审计修复 — Claude Code 执行指南

> 基于 `SITEINTEL-FULL-SITE-ANALYSIS-AUDIT.md` 的白盒审计修复任务
> 执行顺序：P0 → P1 → P2

---

## 执行前准备（第 0 步）

先让 Claude Code 做代码审查，确认当前状态：

```
请阅读项目根目录下的 SITEINTEL-FULL-SITE-ANALYSIS-AUDIT.md（白盒审计报告），然后：

1. 扫描项目完整代码结构，列出关键目录和文件
2. 核对审计报告中提到的以下文件是否存在及当前状态：
   - lib/analyzers/infrastructure.ts
   - lib/entities.ts
   - providers/http.ts
   - providers/metadata.ts
   - providers/ssl.ts
   - lib/ssrf-guard.ts
   - sitemap.ts
   - (seo)/[...slug]/page.tsx
   - technology/[slug]/page.tsx
   - report-actions.tsx
   - site-footer.tsx
   - rules.ts
   - insight-text.ts
   - 以及任何相关测试文件

3. 对审计报告中的每个问题，确认：
   - 问题是否仍然存在
   - 代码位置是否变化
   - 是否已被部分修复

4. 输出一份「当前代码状态核对表」，格式：
   | 审计项 | 文件 | 状态 | 备注 |

确认完毕后再开始 P0 修复。
```

---

## P0-1：SSRF 安全修复

### Prompt 1 — 创建安全 fetch 层

```
根据审计报告，我需要建立一个统一的 SSRF 防护网络访问层。

当前问题：
- providers/http.ts 直接 fetch 用户提交的域名，跟随重定向，无 IP 校验
- providers/metadata.ts 同样模式（首页、robots.txt、sitemap.xml、favicon）
- providers/ssl.ts TLS 握手也无目标 IP 校验
- lib/ssrf-guard.ts 仅用于 webhook，未覆盖采集管道

要求：
1. 在 lib/security/safe-fetch.ts 创建安全 fetch 包装器
2. 复用 lib/ssrf-guard.ts 的 IP 范围判断逻辑（如果适用），避免两套规则
3. 流程：
   - DNS 解析目标域名（resolve4/resolve6）
   - 逐地址验证：禁止 127.0.0.0/8、::1、10.0.0.0/8、172.16.0.0/12、192.168.0.0/16、169.254.0.0/16、link-local、loopback、multicast、unspecified、保留地址
   - IPv4 和 IPv6 都必须覆盖
   - 按解析后的公网 IP 发起连接，带 Host/SNI
   - 每次 HTTP redirect 的 Location 必须重新解析并校验
4. 不要只在 /api/analyze 入口做字符串过滤，必须在网络层防护
5. 防止 DNS Rebinding

请实现 safe-fetch.ts，然后修改 providers/http.ts、providers/metadata.ts、providers/ssl.ts 统一接入。

完成后运行 typecheck 和 lint。
```

### Prompt 2 — SSRF 测试

```
为 lib/security/safe-fetch.ts 编写测试，覆盖以下场景：

必须测试的阻止案例：
- 127.0.0.1
- localhost
- ::1
- 192.168.1.1
- 10.0.0.1
- 172.16.0.1 ~ 172.31.255.255 中至少一个
- 169.254.169.254
- 0.0.0.0
- 255.255.255.255
- fe80::1
- fc00::1
- redirect chain 中跳转到 127.0.0.1

必须测试的允许案例：
- 正常公网域名（如 example.com 解析后的 IP）
- 正常公网 IPv4
- 正常公网 IPv6

请使用项目现有测试框架编写。如果项目没有测试框架，请说明并建议方案。
运行测试并确保全部通过。
```

---

## P0-2：provider_changed DNS 假阳性

### Prompt 3 — 修复 NS/MX 排序

```
修复 lib/analyzers/infrastructure.ts 中 provider_changed 假阳性问题。

根因：
- nameservers[0] 直接取未排序数组第一个元素
- mx[0] 直接取未按 priority 排序的第一个元素
- wordpress.org 的 NS/MX 轮询导致每 30 分钟产生 provider_changed 事件

修复要求：
1. nameservers 数组在取 [0] 之前先 .sort() 稳定顺序
2. mx 数组先按 priority 升序排序，再取最小 priority 的条目
3. 检查快照差异逻辑：如果 previous set == current set（只是数组顺序不同），不得产生 provider_changed
4. 注释中提到的「sorted: resolver return order varies between runs」需要真正落实

修改后运行 typecheck 和 lint。
```

### Prompt 4 — 历史数据清理脚本

```
创建 scripts/cleanup-provider-false-positives.ts

功能：
- 识别 wordpress.org 的 provider_changed 假阳性事件（58 条）
- 默认 dry-run 模式，只输出将要删除的记录数和详情
- 需要明确的环境变量或 CLI 参数（如 --execute）才能执行真实 DELETE
- 输出迁移前后统计
- 考虑相关 insight 的级联处理（但不要自动删除，先输出影响分析）

要求：
- 使用项目现有数据库连接方式
- 禁止自动执行 DELETE
- 提供清晰的 dry-run 报告
```

---

## P1-1：www 域名实体重复

### Prompt 5 — Domain Canonicalization

```
建立 Domain Entity 的 canonicalization 机制。

当前问题：
- www.sitevance.tw 和 sitevance.tw 是两个独立 domain 实体
- 全库 90 个 www.* 前缀残留
- technology.ts 展示时去 www 导致重复计数

要求：
1. 在 entities.ts normalizeValue 中对 domain 类型剥除 www 前缀（或建立 canonical domain 规则）
2. 技术统计页面（technology.ts）组装时按 normalized domain 去重
3. 防止新的重复实体产生
4. 不要简单粗暴删除所有 www——需要评估：
   - Entity 是否代表 hostname
   - Entity 是否代表 registrable domain
   - www 是否应该作为 alias

先输出你的设计方案，确认后再实现。
```

### Prompt 6 — www 实体迁移脚本

```
创建 scripts/merge-www-entities.ts

功能：
- 找出所有 www.* 前缀的 domain 实体及其对应的 apex domain 实体
- 默认 dry-run，输出：
  - 影响的实体数量
  - 关系重指向计划
  - 是否有孤立关系风险
- 需要 --execute 参数才执行真实迁移
- 关系必须安全重指向 canonical entity
- 不允许产生孤立 relationship

输出迁移前后统计。
```

---

## P1-2：AS0 哨兵实体

### Prompt 7 — AS0 防护

```
修复 AS0 哨兵实体问题。

当前问题：
- 上游对无 ASN 映射的 IPv6 返回裸 "0"
- extractEntities 未做哨兵值过滤即建实体
- normalizeValue 剥 AS 前缀后 "AS0"→"0"
- /asn/AS0 已进入 sitemap

修复：
1. entities.ts 中对 asn === "0" 或 "AS0" 做哨兵守卫，拒绝创建 AS0 实体
2. sitemap 层排除无效 ASN（AS0）
3. 已有 AS0 实体页面访问时进行合理处理（如返回 404 或提示「无效 ASN」）
4. 不要自动删除生产数据

运行 typecheck 和 lint。
```

---

## P1-3：SEO 修复

### Prompt 8 — Sitemap 首页重复

```
修复 sitemap.ts 中首页重复条目问题。

当前：
- https://siteintel.cc 和 https://siteintel.cc/ 同时存在于 sitemap
- 一个 daily，一个 weekly

修复：
- sitemap.ts:142 的过滤逻辑统一去除尾斜杠后再比较
- 确保首页只出现一次

运行 typecheck、lint、build。
```

### Prompt 9 — Entity 页面 OG Metadata

```
为以下实体页面补充独立的 openGraph metadata：
- /ip/[ip]
- /asn/[asn]
- /certificate/[id]
- /organization/[id]
- /technology/[slug]

当前它们继承首页默认 OG：
<meta property="og:title" content="SiteIntel — 网站分析_域名情报...">

要求：
- 每个实体页面的 generateMetadata 补充 openGraph 块
- 包含：title, description, url
- 图片可复用首页 OG 图或按类型生成
- 不要继续分享首页品牌信息

检查现有 generateMetadata 实现方式，保持风格一致。
运行 typecheck、lint、build。
```

### Prompt 10 — Search 页面 CTA

```
修复搜索页面 CTA 路由配置错误。

当前问题：
- /search/google、/search/bing、/search/baidu 的 CTA 都链到 /
- (seo)/[...slug]/page.tsx 中 PAGE_SECTIONS 的 href 是 "/search-intelligence"
- seo-landing.tsx 中三元判断用 "/search" 做比较

修复：
1. 统一 href 配置值，确保 CTA 正确指向搜索中心
2. 检查 /search/baidu 页面是否还有残留的 "Phase 3" 文案，已上线的功能不能显示为未实现
3. 确保三个搜索页 CTA 行为一致

运行 typecheck、lint、build。
```

---

## P2-1：Entity Resolution 基础

### Prompt 11 — Organization Alias 机制

```
建立最小可行的 Entity Resolution / Canonicalization Layer。

重点解决 Organization 碎片化问题：
- "Alibaba Cloud - HK" / "Alibaba Cloud - US" / "Alibaba Cloud Computing Ltd." / "Aliyun Computing Co., LTD"
- "Akamai International, BV" vs "Akamai Technologies, Inc."
- "CHINANET" 各省分支

要求：
1. 设计 organization_alias 表结构（或等效机制）
2. 支持：curated alias、exact normalized match、基础规则匹配
3. 第一阶段不需要 AI 模糊匹配——自动合并必须非常谨慎
4. alias → canonicalName → canonical entityId
5. 不要破坏现有 entity 和 relationship 结构

先输出表结构设计和方案，确认后再实现。
```

---

## P2-2：产品与 UX

### Prompt 12 — Technology metadata locale

```
修复 technology/[slug]/page.tsx:128 的硬编码中文问题。

当前：
- const isZh = true
- EN locale 用户拿到中文 title/description

修复：
- 读取当前 locale
- metadata 根据 locale 正确生成

运行 typecheck、lint、build。
```

### Prompt 13 — 重新分析 429 提示

```
修复 report-actions.tsx 的限流提示问题。

当前：
- 非 2xx 响应仅 setBusy(false) 静默复位
- 429/retryAfterSec 不展示给用户

修复：
1. 当收到 429 时，读取 retryAfterSec
2. 展示明确错误文案（如「请求过于频繁，请 X 秒后重试」）
3. 需要加 i18n 键（如 rateLimited）
4. 不允许静默失败

运行 typecheck、lint、build。
```

### Prompt 14 — 回环地址页面处理

```
处理 127.0.0.1 等回环/私网 IP Entity 页面。

注意：
- 这些不一定是测试数据（ddooo.com 等真实域名解析到 127.0.0.1）
- 不要直接删除数据

要求：
1. 回环/私网 IP Entity 不进入 sitemap
2. Entity 页面明确标注：
   - "Loopback Address" / "Private / Non-public Address"
3. 保留真实 DNS Observation 数据
4. 避免搜索引擎把这些当成正常公共 IP 页面

运行 typecheck、lint、build。
```

### Prompt 15 — Insight 重复差异化

```
分析并处理 infrastructure_migration 和 infrastructure_migration_event 的重复问题。

当前：
- Rule 6：7 天窗口内 ≥2 种基础设施事件类型聚类
- Rule 10：correlation 引擎同轮 ≥2 事实变化复合事件
- 标题仅差一个词，证据高度重叠

要求：
1. 先分析两者的触发条件、时间窗口、数据来源、Evidence
2. 如果底层语义不同，在展示层明确差异化：
   - 例如「短期基础设施变化聚类」vs「同轮多事实复合变化」
3. 如果用户层面没有必要区分，在 Presentation Layer 合并展示
4. 不要简单删除规则

先输出分析结论，确认后再修改文案/展示逻辑。
```

---

## 最终步骤：生成修复报告

### Prompt 16 — Remediation Report

```
请基于本次所有修改，生成 SITEINTEL-REMEDIATION-REPORT.md，放在项目根目录。

报告必须包含：

## A. 执行摘要
- P0 完成情况（2 项）
- P1 完成情况（5 项）
- P2 完成情况（5 项）

## B. 修改文件清单
列出所有修改的文件，说明每个文件的修改原因。

## C. 问题状态表
| 审计项 | 状态 | 说明 |
状态使用：FIXED / PARTIALLY FIXED / ALREADY FIXED / NOT FIXED / DEFERRED

## D. 安全验证
- SSRF 防护覆盖范围
- Redirect 防护
- DNS validation
- IPv4/IPv6 覆盖
- Metadata endpoint 保护

## E. 数据迁移
- www entity migration（dry-run 结果、影响数量、是否执行）
- AS0 cleanup
- provider_changed cleanup
- organization alias（如已实现）

## F. 测试结果
- typecheck 结果
- lint 结果
- test 结果
- build 结果

## G. 遗留问题
- 本次未处理的 P3 项
- 需要后续处理的事项
- 需要人工确认的数据操作

确保报告准确反映实际完成的工作。
```

---

## 执行策略建议

### 不要一次性全丢
Claude Code 上下文有限，建议每 2-3 个 prompt 一批：

| 批次 | 内容 | 预计时间 |
|------|------|---------|
| Batch 1 | Prompt 0（代码审查） | 5-10 min |
| Batch 2 | Prompt 1-2（SSRF + 测试） | 20-30 min |
| Batch 3 | Prompt 3-4（provider_changed + 清理脚本） | 15-20 min |
| Batch 4 | Prompt 5-6（www canonicalization + 迁移） | 20-30 min |
| Batch 5 | Prompt 7（AS0）+ Prompt 8-10（SEO） | 20-30 min |
| Batch 6 | Prompt 11-15（P2 各项） | 30-40 min |
| Batch 7 | Prompt 16（报告） | 10 min |

### 每批结束后
1. 让 Claude Code 运行 `typecheck`、`lint`、`build`
2. 确认无错误再进入下一批
3. 如果有测试，确保测试通过

### 数据迁移原则
- 所有迁移脚本默认 dry-run
- 需要 `--execute` 或环境变量才执行真实操作
- 不要自动 DELETE/UPDATE/DROP 生产数据
