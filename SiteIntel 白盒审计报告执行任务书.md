# SiteIntel 白盒审计报告执行任务书

请以我提供的 `SITEINTEL-FULL-SITE-ANALYSIS-AUDIT.md` 作为本次工作的唯一核心依据，对当前 SiteIntel 项目进行代码审查、修复和升级。

这不是一次重新设计项目，也不是要求你自由扩展新功能。

本次任务目标是：

> **根据白盒审计报告中已经确认的问题，按照 P0 → P1 → P2 的优先级，修复当前 SiteIntel 的安全漏洞、数据质量问题、实体治理问题、SEO/元数据问题和已确认的产品缺陷。**

---

# 一、最高执行原则

## 1. 先审查，再修改

不要直接根据报告机械修改代码。

你必须先：

1. 阅读完整审计报告。
2. 阅读当前项目完整代码结构。
3. 核对报告中提到的文件、函数和逻辑。
4. 确认当前代码与报告中的 git HEAD 状态是否一致。
5. 如果当前代码已经发生变化，必须以当前代码为准，并说明：
   - 哪些问题仍然存在。
   - 哪些已经被修复。
   - 哪些报告中的代码位置已经失效。
6. 不允许因为修复一个问题破坏现有功能。

---

# 二、本次任务严格按照以下优先级执行

# P0：必须立即处理

## P0-1：修复分析管道 SSRF

审计报告确认：

- `providers/http.ts`
- `providers/metadata.ts`
- `providers/ssl.ts`

存在用户提交域名后，直接进行网络访问的问题。

当前风险包括：

- DNS 指向 `127.0.0.1`
- IPv6 回环地址
- RFC1918 私网
- Link-local
- `169.254.169.254`
- 云 Metadata 服务
- 保留地址
- 恶意重定向到内网
- Redirect 每一跳未重新验证

### 执行要求

建立统一的安全网络访问层，例如：

```text
lib/security/safe-fetch.ts
```

或者其他你认为更合理的位置。

要求：

1. 所有 HTTP 网络访问统一经过安全层。
2. DNS 解析后必须验证实际 IP 是否允许访问。
3. 禁止访问：
   - localhost
   - 127.0.0.0/8
   - ::1
   - RFC1918 私网
   - link-local
   - loopback
   - unspecified
   - multicast
   - metadata 地址
   - 保留地址
4. IPv4 和 IPv6 都必须覆盖。
5. 每次 HTTP redirect 必须重新进行目标地址验证。
6. 不允许仅验证初始域名。
7. SSL/TLS 采集同样需要经过安全目标验证。
8. 尽量复用现有 `ssrf-guard.ts` 的 IP 范围判断逻辑，避免出现两套安全规则。
9. 不允许只在 `/api/analyze` 入口进行字符串过滤。
10. 必须防止 DNS Rebinding。

### 必须新增测试

至少覆盖：

```text
127.0.0.1
localhost
::1
192.168.x.x
10.x.x.x
172.16.x.x - 172.31.x.x
169.254.169.254
redirect -> 127.0.0.1
IPv6 private / loopback
```

---

## P0-2：修复 provider_changed DNS 假阳性

审计确认：

```text
wordpress.org
```

由于 NS 和 MX 返回顺序变化，持续产生：

```text
provider_changed
```

并进一步污染：

```text
provider_change insight
frequent_changes
infrastructure_migration
```

### 根因

当前逻辑可能直接使用：

```text
nameservers[0]
mx[0]
```

而 DNS Resolver 返回顺序并不稳定。

### 修复要求

#### Nameserver

必须：

```text
nameservers
→ normalize
→ stable sort
→ 再选择 canonical provider
```

不能直接使用原始返回数组的第一个元素。

#### MX

必须按照 MX Priority 排序。

不能依赖 DNS 返回顺序。

例如：

```text
mx
→ sort by priority ASC
→ secondary stable sort
→ select canonical value
```

### 额外要求

检查快照差异逻辑。

如果：

```text
previous set == current set
```

只是数组顺序不同，则不得产生：

```text
provider_changed
```

### 历史数据

不要自动删除生产数据库中的历史事件。

请单独生成：

```text
scripts/cleanup-provider-false-positives.ts
```

或者 SQL migration / maintenance script。

默认：

```text
dry-run
```

只有人工明确确认后才能执行真实删除。

---

# 三、P1：数据质量和实体系统

## P1-1：www 域名实体重复问题

审计确认目前存在：

```text
www.example.com
example.com
```

被当成两个独立 Domain Entity 的情况。

并且历史数据库存在大量 `www.*` 实体。

### 要求

建立统一 Domain Canonicalization。

例如：

```text
Input
↓
lowercase
↓
remove protocol
↓
remove path
↓
normalize trailing dot
↓
canonical domain
```

对于 `www`：

不要简单粗暴地假设所有情况下都必须删除。

需要根据当前项目 Domain Entity 的语义确定：

- Entity 是否代表 hostname。
- Entity 是否代表 registrable domain。
- `www` 是否应该作为 alias。
- 如何避免破坏真实 hostname 数据。

最终要求：

1. 防止新的重复实体产生。
2. 技术统计页面必须去重。
3. 历史数据必须提供 migration。
4. migration 默认 dry-run。
5. 输出迁移前后统计。
6. 关系必须安全重指向 canonical entity。
7. 不允许产生孤立 relationship。

---

## P1-2：AS0 哨兵实体

审计确认：

```text
AS0
0
```

属于上游无 ASN 映射时产生的哨兵值。

要求：

1. Entity 创建前过滤。
2. 不允许新的 `AS0` 实体进入数据库。
3. sitemap 排除无效 ASN。
4. 页面访问已有 AS0 时进行合理处理。
5. 历史数据迁移必须可审计。
6. 不自动删除生产数据。

---

# 四、P1：SEO 和页面修复

## 1. sitemap 首页重复

当前存在：

```text
https://siteintel.cc
https://siteintel.cc/
```

要求：

统一 URL Canonicalization。

确保 sitemap 中首页只存在一个。

---

## 2. Entity 页面 OG Metadata

以下实体页面目前继承首页默认 OG：

- IP
- ASN
- Certificate
- Organization
- Technology

要求：

每个实体页面生成自己的：

```text
title
description
canonical URL
openGraph.title
openGraph.description
openGraph.url
```

不要继续让实体页面分享首页：

```text
SiteIntel — 网站分析_域名情报...
```

如果暂时没有实体专属 OG Image，可以先复用统一图片。

但 metadata 内容必须实体化。

---

## 3. Search 页面 CTA

修复：

```text
/search/google
/search/bing
/search/baidu
```

CTA 路由配置错误。

必须检查：

```text
PAGE_SECTIONS
sectionHref
/search
/search-intelligence
```

确保实际跳转逻辑正确。

同时检查：

```text
/search/baidu
```

残留的：

```text
Phase 3
```

文案。

已经实现的功能不能继续显示为未实现。

---

# 五、P2：实体治理架构

这是本次升级的重要部分。

不要只做几个字符串 replace。

请评估并建立一个最小可行的：

# Entity Resolution / Canonicalization Layer

建议数据流程：

```text
Raw Data
↓
Normalization
↓
Validation
↓
Canonicalization
↓
Alias Resolution
↓
Canonical Entity
```

重点解决：

## Organization

例如：

```text
Alibaba Cloud - HK
Alibaba Cloud - US
Alibaba Cloud Computing Ltd.
Aliyun Computing
...
```

不能无限产生碎片 Entity。

第一阶段不需要实现复杂 AI Entity Resolution。

建议：

```text
organization_alias
```

或类似机制：

```text
alias
canonicalName
entityId
source
confidence
createdAt
```

先支持：

- curated alias
- exact normalized match
- 基础规则匹配

不要直接使用模糊匹配自动合并所有公司。

自动合并必须非常谨慎。

---

# 六、P2：产品与 UX 修复

## 1. Technology metadata locale

修复：

```text
const isZh = true
```

不能硬编码。

metadata 必须根据 locale 正确生成。

---

## 2. 重新分析 429 提示

当前重新分析触发 Rate Limit 后前端静默恢复。

要求：

- 显示明确错误。
- 告知用户限制原因。
- 如果 API 返回 retryAfterSec，则显示等待时间。
- 不允许静默失败。

---

## 3. 127.0.0.1 等回环地址处理

注意：

审计报告已经确认：

```text
127.0.0.1
```

并不一定是测试数据。

可能是目标域名真实 DNS 返回。

因此不要直接删除所有相关数据。

要求：

1. 保留真实 DNS Observation。
2. 但禁止 SSRF 实际访问。
3. 回环 / 私网 IP Entity 不进入 sitemap。
4. Entity 页面明确标注：
   - Loopback Address
   - Private / Non-public Address
5. 避免搜索引擎把这些页面当成正常公共 IP 页面。

---

## 4. Insight 重复问题

检查：

```text
infrastructure_migration
infrastructure_migration_event
```

不要简单删除其中一个。

先分析两者：

- 触发条件
- 时间窗口
- 数据来源
- Evidence
- 用户展示场景

如果底层语义不同，则：

明确差异化展示。

例如：

```text
短期基础设施变化聚类
```

和：

```text
同轮多事实复合变化
```

如果最终发现用户层面没有必要区分，则在 Presentation Layer 合并。

---

# 七、本次不要处理的事项

除非在执行过程中发现它们阻塞 P0-P2，否则不要擅自进行：

- 大规模 UI 重设计
- 新增大量页面
- 新增无关 API
- 重写整个数据库
- 删除历史数据
- 引入大型 ORM 替换
- 重写整个采集管道
- 引入复杂 AI Agent
- 大规模修改现有产品定位

本次目标是：

> **修复 + 稳定 + 数据治理基础建设**

不是重新开发 SiteIntel。

---

# 八、生产数据安全规则

当前项目涉及生产 PostgreSQL。

因此：

## 默认允许

```text
SELECT
```

## 禁止自动执行

```text
DELETE
UPDATE
DROP
TRUNCATE
```

涉及以下内容时必须：

1. 先生成 migration plan。
2. 输出影响记录数量。
3. 提供 dry-run。
4. 等待人工确认。

尤其包括：

- 58 条 wordpress.org 假阳性事件。
- 90 个 www Domain Entity。
- AS0 Entity。
- Organization Entity 合并。

---

# 九、每完成一个阶段必须验证

每个 P0/P1/P2 修复完成后：

## 1. Type Check

执行项目现有：

```text
typecheck
```

或对应命令。

## 2. Lint

执行：

```text
lint
```

## 3. Build

执行：

```text
build
```

## 4. Test

执行已有测试。

并为以下新增：

- SSRF tests
- DNS sorting tests
- MX priority tests
- Entity canonicalization tests
- AS0 validation tests
- Metadata tests（如项目结构支持）

---

# 十、最终输出要求

完成全部工作后，不要只告诉我：

> 已修复。

必须生成：

```text
SITEINTEL-REMEDIATION-REPORT.md
```

报告必须包括：

## A. 执行摘要

```text
P0 完成情况
P1 完成情况
P2 完成情况
```

## B. 修改文件

例如：

```text
lib/security/safe-fetch.ts
providers/http.ts
providers/metadata.ts
...
```

说明每个文件为什么修改。

## C. 每个问题状态

使用：

```text
FIXED
PARTIALLY FIXED
ALREADY FIXED
NOT FIXED
DEFERRED
```

并说明原因。

## D. 安全验证

特别说明：

```text
SSRF 防护覆盖范围
redirect 防护
DNS validation
IPv4
IPv6
metadata endpoint
```

## E. 数据迁移

分别列出：

```text
www entity migration
AS0 cleanup
provider_changed cleanup
organization alias
```

并说明：

- dry-run 结果。
- 预计影响数量。
- 是否执行。
- 是否需要人工确认。

## F. 测试结果

输出：

```text
typecheck
lint
test
build
```

的最终结果。

## G. 遗留问题

列出本次没有处理的 P3 或需要后续处理的事项。

---

# 最终目标

本次工作完成后，SiteIntel 不应该只是“增加更多功能”。

而应该达到：

```text
更安全
更稳定
数据更可信
实体更统一
事件噪音更少
SEO 更正确
后续 Intelligence 分析有可靠的数据基础
```

请严格按照：

```text
P0 → P1 → P2
```

顺序执行。

不要跳过验证。

不要未经确认修改生产数据。

先分析当前代码，再开始修改。