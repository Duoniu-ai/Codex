# IPCECE 全面升级实施规划

**项目名称：** IPCECE Network Intelligence Platform Upgrade  
**适用对象：** Claude Code + DeepSeek V4 Pro  
**文档性质：** 项目总纲 + 技术实施规范 + 开发任务清单  
**原则：** 基于现有线上项目进行渐进式升级，不盲目推倒重写；先审计现有代码，再实施改造。

---

# 1. 项目最终定位

IPCECE 不再定位为“IP 查询工具集合”。

升级后的定位：

> **网络身份、IP 情报与网络环境诊断平台**

英文：

> **Network Identity & IP Intelligence Platform**

核心用户问题统一为：

> **我的网络身份、IP 出口和浏览器环境现在到底是什么状态？**

系统必须把现有功能统一纳入以下六大 Intelligence Domain：

1. IP Intelligence
2. Network Identity
3. Privacy & Leak Detection
4. Network Connectivity
5. Service Accessibility
6. Network Reports

---

# 2. 第一原则：禁止无计划重写

在任何开发之前，Claude Code / DeepSeek 必须先：

1. 完整扫描当前项目目录。
2. 识别前端框架、后端框架、数据库、ORM、部署方式。
3. 输出当前路由列表。
4. 输出当前 API 列表。
5. 输出所有页面及功能。
6. 输出重复组件。
7. 输出重复请求逻辑。
8. 输出数据源与第三方 API。
9. 输出环境变量。
10. 输出潜在安全问题。

## 强制要求

禁止：

- 直接删除现有页面。
- 直接替换整个项目。
- 修改后导致原有 URL 失效。
- 修改 API 后破坏旧前端。
- 将所有逻辑集中到一个巨大文件。
- 为了“重构”而重构。

必须采用：

> Audit → Architecture → Incremental Refactor → Test → Deploy

---

# 3. 最终产品架构

```text
IPCECE
│
├── Home
│   └── Network Health Check
│
├── IP Intelligence
│   ├── IP Lookup
│   ├── My IP
│   ├── ASN Lookup
│   ├── IP Risk Score
│   └── Batch IP Lookup
│
├── Privacy & Leak Detection
│   ├── DNS Leak
│   ├── WebRTC Leak
│   ├── IPv6 Leak
│   ├── Browser Fingerprint
│   └── Environment Consistency
│
├── Network Identity
│   ├── Proxy Detection
│   ├── VPN Detection
│   ├── Exit Consistency
│   └── Bot Detection
│
├── Network Test
│   ├── Ping
│   ├── DNS Lookup
│   ├── Connectivity
│   ├── Global Latency
│   └── Speed Test
│
├── Service Accessibility
│   ├── AI Services
│   ├── Streaming Services
│   └── Other Global Services
│
├── Network Tools
│   ├── CDN Lookup
│   ├── Cloudflare Tools
│   └── Utility Tools
│
└── Reports
    ├── Network Report
    ├── Share Report
    └── Export
```

---

# 4. 首页必须升级为 Network Health Check

首页不能继续只是工具导航页。

首页首屏核心必须是：

# 自动网络环境体检

用户访问后执行轻量级基础检测：

```text
Detecting Network Environment

✓ Public IP
✓ IPv4 / IPv6
✓ ASN / ISP
✓ Proxy Status
✓ DNS Status
✓ WebRTC
✓ Browser Environment
✓ IP Risk
```

## 分层检测

### Fast Scan

目标：1~3 秒内返回。

包含：

- Public IP
- Country / Region
- ASN
- ISP
- IPv4 / IPv6
- 基础代理判断

### Deep Scan

异步执行：

- DNS Leak
- WebRTC Leak
- IPv6 Leak
- Browser Consistency
- Risk Intelligence
- Multi Exit
- Service Accessibility

前端必须逐项显示检测状态，而不是长时间白屏等待。

---

# 5. 核心新功能：Network Health Score

必须建立统一评分系统。

输出：

```text
Network Health

82 / 100

Status: Good
```

评分维度：

```text
IP Identity          20
Privacy              20
Leak Protection      20
Network Consistency  15
IP Reputation        15
Service Accessibility 10
```

总分 100。

## 重要要求

评分不能是随机数字。

必须：

```text
score = weighted rules
```

每一个扣分必须返回原因。

例如：

```json
{
  "score": 82,
  "level": "good",
  "deductions": [
    {
      "category": "privacy",
      "score": -8,
      "reason": "IPv6 address exposure detected"
    },
    {
      "category": "risk",
      "score": -10,
      "reason": "Datacenter network classification"
    }
  ]
}
```

---

# 6. 建立统一 Detection Engine

禁止每个页面独立请求多个 API。

建立：

```text
Detection Engine
│
├── IP Intelligence Provider
├── Geo Provider
├── ASN Provider
├── Proxy Provider
├── Risk Provider
├── Leak Detector
├── Browser Detector
├── Connectivity Tester
└── Service Access Provider
```

建议目录：

```text
src/
├── modules/
│   ├── ip-intelligence/
│   ├── risk-intelligence/
│   ├── privacy/
│   ├── browser/
│   ├── connectivity/
│   └── service-access/
│
├── providers/
│   ├── geo/
│   ├── asn/
│   ├── proxy/
│   └── reputation/
│
├── engine/
│   ├── detection-engine
│   ├── scoring-engine
│   └── report-engine
│
├── schemas/
└── shared/
```

实际目录必须根据现有项目技术栈调整，不允许为了套目录而强行改变框架。

---

# 7. 建立统一 API Schema

所有接口统一数据结构。

推荐：

```json
{
  "success": true,
  "data": {},
  "meta": {
    "request_id": "",
    "cached": false,
    "generated_at": ""
  },
  "error": null
}
```

IP Intelligence 标准模型：

```json
{
  "ip": {
    "address": "",
    "version": "IPv4"
  },
  "network": {
    "asn": "",
    "isp": "",
    "organization": ""
  },
  "location": {
    "country": "",
    "country_code": "",
    "region": "",
    "city": "",
    "timezone": ""
  },
  "classification": {
    "type": "",
    "hosting": false,
    "proxy": false,
    "vpn": false,
    "tor": false
  },
  "risk": {
    "score": 0,
    "level": "",
    "reasons": []
  },
  "confidence": {
    "score": 0,
    "sources": 0,
    "conflicts": []
  }
}
```

## 兼容性要求

现有 API 如果已经被前端调用：

- 优先保持旧接口可用。
- 新接口使用 `/v2/` 或等价版本机制。
- 禁止无版本直接破坏性修改。

---

# 8. Data Fusion Engine

如果系统使用多个数据源，不能简单“谁返回就显示谁”。

建立：

```text
Sources
   ↓
Normalizer
   ↓
Field Validation
   ↓
Conflict Detection
   ↓
Weighted Confidence
   ↓
Final Result
```

每个 Provider 返回内部统一格式：

```json
{
  "source": "provider_name",
  "data": {},
  "confidence": 0.9,
  "updated_at": ""
}
```

## 冲突规则

例如：

```text
Source A → Singapore
Source B → Singapore
Source C → Hong Kong
```

系统：

1. 计算权重。
2. 判断数据源可信度。
3. 判断更新时间。
4. 输出最终结果。
5. 保存冲突信息。

最终：

```json
{
  "country": "Singapore",
  "confidence": 87,
  "conflicts": [
    {
      "field": "country",
      "values": ["Singapore", "Hong Kong"]
    }
  ]
}
```

---

# 9. IP Risk Engine

风险评分必须透明。

推荐：

```text
IP Risk Score

68 / 100

Level: Medium
```

风险因子：

- Proxy
- VPN
- Tor
- Datacenter
- Hosting Network
- Abuse / Reputation
- Blacklist
- ASN Classification
- Location Conflict
- DNS / Network Anomaly

返回：

```json
{
  "score": 68,
  "level": "medium",
  "factors": [
    {
      "name": "Datacenter Network",
      "impact": 15
    }
  ],
  "positive_signals": []
}
```

禁止出现无法解释的“神秘评分”。

---

# 10. 浏览器端检测优先

以下能力优先在浏览器执行：

- WebRTC
- Canvas
- WebGL
- Audio Context
- Screen
- Timezone
- Language
- User Agent
- Hardware Information
- Browser Environment Consistency

后端只接收真正需要服务端分析的数据。

原则：

> Local Detection First.

禁止为了方便把所有浏览器信息全部上传并长期存储。

---

# 11. Network Report

新增核心页面：

```text
/reports/network
```

或根据现有路由规范实现。

报告包含：

```text
Network Identity
IP Intelligence
Privacy
Leak Detection
Browser Environment
Risk Score
Connectivity
Service Accessibility
Final Health Score
```

生成唯一：

```text
Report ID
```

支持：

- 分享链接
- JSON Export
- Markdown Export
- Copy Summary

## 隐私要求

分享报告时必须：

- 默认隐藏完整 IP 或提供脱敏选项。
- 默认不包含完整浏览器指纹。
- 不包含用户 Cookie。
- 不包含不必要的请求头。
- 报告必须有过期机制。

---

# 12. 异步任务系统

Deep Scan 不能全部同步等待。

架构：

```text
Create Scan
    ↓
Task ID
    ↓
Fast Result
    ↓
Async Workers
    ↓
SSE Stream
    ↓
Final Report
```

优先使用 SSE。

仅当现有项目确实需要双向实时通信时才使用 WebSocket。

SSE 事件示例：

```text
event: progress
data: {"module":"dns","status":"running","progress":45}

event: result
data: {"module":"dns","status":"completed","data":{}}

event: complete
data: {"report_id":"..."}
```

必须支持：

- Timeout
- Cancel
- Retry
- Partial Result
- Provider Failure Isolation

一个 Provider 失败不能导致整个报告失败。

---

# 13. 缓存架构

目标：

减少第三方 API 成本，提高响应速度。

架构：

```text
Browser Cache
      ↓
Edge Cache
      ↓
KV / Redis
      ↓
PostgreSQL Cache
      ↓
External Provider
```

TTL 必须按数据类型设置，而不是统一缓存时间。

建议：

| 数据 | 建议 |
|---|---|
| ASN | 7~30 天 |
| ISP / Organization | 1~7 天 |
| Geo | 1~7 天 |
| Proxy / VPN | 15~60 分钟 |
| Risk | 15~60 分钟 |
| Service Status | 1~10 分钟 |
| 用户实时 IP | 极短缓存或不缓存 |

实际 TTL 根据数据源更新频率调整。

---

# 14. PostgreSQL 数据设计

保留：

```text
ip_intelligence_cache
provider_cache
risk_cache
asn_cache
geo_cache
network_nodes
service_status
report
report_item
system_metrics
```

不要长期保存：

- 原始完整浏览器指纹
- 用户私密 Cookie
- 无必要的完整 Headers
- 可用于长期追踪用户的唯一标识

报告数据必须设置：

```text
expires_at
```

自动清理。

---

# 15. 安全：SSRF 防护

所有需要服务器主动访问用户输入目标的功能必须经过统一校验。

建立：

```text
Target Validation Layer
```

禁止访问：

```text
localhost
127.0.0.0/8
::1
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
169.254.0.0/16
metadata endpoints
```

同时必须：

- DNS Resolution 后再次验证 IP。
- 防 DNS Rebinding。
- 设置连接超时。
- 设置响应大小限制。
- 禁止非 HTTP/HTTPS 协议。
- 限制重定向。
- Rate Limit。
- Concurrency Limit。

---

# 16. Rate Limit 与 Anti-Abuse

工具类网站必须防止 API 被刷。

实现：

```text
IP based limit
+
Endpoint based limit
+
Expensive operation quota
```

例如：

```text
Normal Lookup:
60 / minute

Deep Scan:
10 / hour

Batch Lookup:
根据用户等级
```

不能只使用前端限制。

---

# 17. 前端组件系统

必须抽取通用组件。

建议：

```text
components/
├── ui/
│   ├── card
│   ├── badge
│   ├── progress
│   └── skeleton
│
├── detection/
│   ├── scan-progress
│   ├── result-card
│   ├── score-card
│   └── status-indicator
│
├── intelligence/
│   ├── ip-card
│   ├── geo-card
│   ├── asn-card
│   └── risk-card
│
└── report/
```

要求：

- Loading 状态统一。
- Error 状态统一。
- Empty 状态统一。
- Retry 逻辑统一。
- Copy 行为统一。
- Mobile Responsive。

---

# 18. 页面 UX 改造

每个工具页面必须遵循：

```text
1. 用户输入 / 自动识别
2. Start Detection
3. Progress
4. Core Result
5. Detailed Result
6. Explanation
7. Recommended Next Tool
```

例如：

IP 查询完成后：

```text
Next Actions

→ Check IP Risk
→ Check Proxy Status
→ Check DNS Leak
→ Generate Full Network Report
```

禁止每个工具变成信息孤岛。

---

# 19. Service Accessibility 重构

现有单独的 Claude / 流媒体检测应统一为：

```text
Service Accessibility
```

分类：

```text
AI Services
Streaming
Communication
Developer Services
```

系统只报告：

- DNS
- Network Reachability
- Region / Network Signal
- Public Availability Signal

必须避免把“某个服务的官方账号资格判断”伪装成绝对结论。

结果应该明确：

```text
Network Accessibility Signal
```

而不是：

```text
Guaranteed Available
```

---

# 20. SEO 架构

SEO 采用：

```text
Tool Page
    ↓
Guide
    ↓
FAQ
    ↓
Related Tool
```

示例：

```text
/ip-lookup
    ├── /guides/what-is-ip
    ├── /guides/ipv4-vs-ipv6
    ├── /guides/how-to-find-asn
    └── /tools/ip-risk
```

要求：

- 每个工具独立 Title。
- 独立 Meta Description。
- Canonical。
- Sitemap。
- Robots。
- FAQ Schema。
- Breadcrumb Schema。
- Tool / WebApplication Schema（符合实际内容）。
- 不生成大量重复薄内容。
- 不伪造发布日期。

---

# 21. 性能目标

定义 Performance Budget。

目标：

```text
LCP < 2.5s
INP < 200ms
CLS < 0.1
```

首屏：

- 不加载所有工具代码。
- 路由级懒加载。
- 非关键组件延迟加载。
- API 并发控制。
- 避免重复请求同一 IP 数据。

---

# 22. 可观测性

必须增加：

```text
request_id
provider_latency
provider_error_rate
cache_hit_rate
task_duration
endpoint_latency
```

日志禁止记录：

- 完整敏感 Header
- Cookie
- Authorization
- 用户隐私内容

错误追踪必须能够定位：

```text
哪个模块
哪个 Provider
哪个请求 ID
耗时多少
缓存是否命中
```

---

# 23. API 文档

所有 API 自动或半自动生成文档。

至少包括：

```text
GET /api/v2/ip/{ip}
GET /api/v2/my-ip
POST /api/v2/network-scan
GET /api/v2/network-scan/{id}
GET /api/v2/network-scan/{id}/stream
```

具体路由必须根据现有项目调整。

文档包括：

- 参数
- Response
- Error Code
- Rate Limit
- Cache Policy
- Example

---

# 24. 实施阶段

## Phase 0：项目全面审计

必须先完成：

- 代码结构审计。
- 页面清单。
- API 清单。
- 数据库 Schema。
- 数据 Provider。
- 环境变量。
- 重复逻辑。
- 性能问题。
- 安全问题。

输出：

```text
docs/audit/
```

---

## Phase 1：建立基础架构

完成：

- Unified API Response。
- Schema。
- Provider Interface。
- Detection Engine。
- Error Handling。
- Logging。
- Request ID。

禁止先修改大量 UI。

---

## Phase 2：首页 Network Health Check

完成：

- Fast Scan。
- Deep Scan。
- Progress UI。
- Health Score。
- Explanation。

这是第一个面向用户发布的重大升级。

---

## Phase 3：Network Report

完成：

- Report Engine。
- Task System。
- SSE。
- Share。
- Export。
- Expiration。

---

## Phase 4：迁移旧工具

按模块迁移：

1. IP
2. Proxy
3. Leak
4. Browser
5. Risk
6. Connectivity
7. Service Access

每迁移一个模块：

- 保持旧 URL。
- 完成测试。
- 检查性能。
- 再迁移下一个。

---

## Phase 5：SEO 与内容结构

完成：

- Metadata。
- Structured Data。
- Sitemap。
- Topic Cluster。
- Internal Linking。
- Related Tools。

---

# 25. 测试要求

必须建立：

```text
Unit Tests
Integration Tests
API Tests
E2E Tests
```

重点测试：

- IPv4。
- IPv6。
- Invalid IP。
- Private IP。
- Reserved IP。
- Provider Timeout。
- Provider Conflict。
- Partial Failure。
- Cache Hit。
- Rate Limit。
- SSRF Attempt。
- DNS Rebinding Attempt。

---

# 26. Definition of Done

任何 Phase 完成必须满足：

```text
[ ] Type Check Passed
[ ] Build Passed
[ ] Lint Passed
[ ] Tests Passed
[ ] Existing Routes Verified
[ ] API Compatibility Verified
[ ] Mobile Verified
[ ] Error States Verified
[ ] Loading States Verified
[ ] Security Review Completed
```

---

# 27. Claude Code + DeepSeek V4 Pro 执行方式

建议不要一次让模型“全部重写”。

按以下顺序执行。

## Prompt 1：全面审计

任务：

> 阅读整个 IPCECE 项目。不要修改任何代码。生成完整项目审计报告，包括技术栈、目录结构、路由、页面、API、数据库、Provider、环境变量、重复逻辑、性能问题、安全问题、SEO 问题。将报告保存到 docs/audit/PROJECT_AUDIT.md。

---

## Prompt 2：制定迁移计划

任务：

> 根据 docs/audit/PROJECT_AUDIT.md 和本项目升级规划，设计最小破坏性的升级方案。不要开始大规模编码。输出：
>
> 1. CURRENT_ARCHITECTURE.md
> 2. TARGET_ARCHITECTURE.md
> 3. MIGRATION_PLAN.md
> 4. BREAKING_CHANGE_RISKS.md
>
> 要求保留现有 URL 和现有功能。

---

## Prompt 3：基础架构

任务：

> 开始实施 Unified Detection Architecture。
>
> 优先创建 Schema、Provider Interface、Detection Engine、统一 Error Handling 和 Request ID。
>
> 不修改现有业务行为，采用兼容层或 v2 API。
>
> 完成后执行完整测试，并输出 IMPLEMENTATION_REPORT.md。

---

## Prompt 4：首页升级

任务：

> 在不破坏现有首页核心功能和 SEO 的前提下，将首页升级为 Network Health Check。
>
> 实现 Fast Scan、Deep Scan、实时进度、Network Health Score 和结果解释。
>
> 优先复用现有 API 和组件，避免重复逻辑。
>
> 完成移动端和桌面端测试。

---

## Prompt 5：Network Report

任务：

> 实现统一 Network Report。
>
> Deep Scan 必须支持 Partial Result、Timeout、Provider Failure Isolation。
>
> 优先使用 SSE 返回进度。
>
> 支持分享、JSON Export、Markdown Export。
>
> 默认执行隐私脱敏，并实现报告过期机制。

---

# 28. 最终核心原则

IPCECE 下一阶段不是：

> 再增加 20 个工具。

而是：

> **把已经拥有的能力统一成一个 Network Intelligence System。**

最终用户路径：

```text
Visit IPCECE
      ↓
Automatic Fast Scan
      ↓
See Current Network Status
      ↓
Run Deep Scan
      ↓
Understand IP / Privacy / Risk / Connectivity
      ↓
Generate Network Report
      ↓
Continue to Related Tools
```

最终产品形态：

> **IPCECE = 用户的网络身份诊断入口。**

---

# 29. 给执行模型的总指令

在执行任何修改前，必须遵守：

1. 先读代码，再修改。
2. 不假设技术栈。
3. 不推倒重写。
4. 保留现有 URL。
5. 保持 API 向后兼容。
6. 每次修改后测试。
7. 每个 Phase 独立提交。
8. Provider 失败必须可降级。
9. 所有外部请求必须设置 Timeout。
10. 所有可能 SSRF 的功能必须统一校验。
11. 不记录不必要的用户隐私数据。
12. 所有评分必须可解释。
13. 所有检测必须显示“不确定性”和“数据限制”，不能输出无法证明的绝对结论。
14. 优先复用现有组件。
15. 完成每个 Phase 后更新 CHANGELOG 和 IMPLEMENTATION_REPORT。

**最终目标不是代码量更多，而是：**

> 更统一、更准确、更稳定、更安全、更快、更容易维护，并让 IPCECE 从工具集合升级为真正的 Network Intelligence Platform。
