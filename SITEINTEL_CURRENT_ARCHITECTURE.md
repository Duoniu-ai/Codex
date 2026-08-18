# SiteIntel — CURRENT ARCHITECTURE

> 审计日期：2026-08-14 · 对应 Master Plan §25 Step 1

## 1. 技术栈

Next.js 16.3 (App Router, SSR) · TypeScript strict (noUncheckedIndexedAccess) · Tailwind CSS 4 ·
Prisma 6 · PostgreSQL 14 · 部署：systemd 服务（端口 3003）+ nginx 反代 + Cloudflare 代理 ·
双语 UI（zh 默认 / en，cookie 记忆）· 零外部前端依赖（图谱自研 SVG 力导向）

## 2. 数据模型（Intelligence Core）

```
Target ──< Investigation ──< RawCollection（原始证据）
   │
   ├──< Snapshot（哈希快照：dns/ip/ssl/http/technology/metadata/rdap/infrastructure/summary/search/full）
   ├──< Event（8 类网站事件 + 10 类搜索事件，带 category 维度）
   ├──< Insight（规则引擎产物：certainty/正反证据/建议下一步）
   └──< Monitor（每日监控 + lastNotifiedAt 告警去重）

Entity ──< EntityRelationship（observed/probable/inferred + confidence + evidence）

SearchConnection（凭证 AES-256-GCM 加密）─< SearchProperty
  ├──< SearchDailyMetric / SearchQueryDaily / SearchPageDaily（行数据，不进快照 JSONB）
  ├──< SearchSubmission（百度链接提交历史）
  └──< SearchSyncJob（同步作业）

DiscoveryCandidate（数据增长引擎：source + priority + status）

User（scrypt + JWT 会话）─< ApiKey（哈希存储）/ Monitor / SearchConnection
```

七层数据流：`Raw → Entities → Relationships → Snapshots → Events → Insights → Report`。
快照 hash 去重、事件带前后值、洞察带证据链——所有页面是同一份数据的**投影**，无平行数据模型。

## 3. 核心模块

| 模块 | 位置 | 说明 |
|---|---|---|
| Provider 采集 | src/lib/providers/ | 7 个无 Key 数据源，失败隔离（partial success） |
| 调查管线 | src/lib/pipeline.ts | 异步执行 + SSE 进度 + RUNNING 去重 + 候选提取钩子 |
| 实体/关系 | src/lib/entities.ts | upsert 幂等，图随分析累积 |
| 快照/事件 | src/lib/snapshots.ts, events.ts | 哈希对比 → 类型化事件 |
| Insight 引擎 | src/lib/intelligence/ | 7 条 MVP 规则 + 2 条搜索关联规则，置信度可解释 |
| 搜索情报 | src/lib/search/ | Provider 适配器 + 同步调度器 + 事件 diff |
| 数据增长 | src/lib/discovery/ | 候选提取 + 优先评分 + 20/天自动采集 |
| 监控/告警 | src/lib/monitoring.ts, notify.ts | 每日重跑 + Telegram/Webhook（SSRF 防护） |
| SEO Engine | src/lib/seo/ | Registry + 质量门 + 技术页投影 |

## 4. 路由地图

```
公开情报层（INDEX）：
  /                          首页（关键词 H1 + 真实示例）
  /how-it-works  /website-intelligence  /search-intelligence
  /infrastructure-intelligence  /technology-intelligence
  /relationship-intelligence  /website-monitoring
  /tools/website-analysis|dns|ip|ssl|technology
  /search/baidu|bing|google
  /guides/website-migration|baidu-seo|ssl
  /cases/infrastructure-migration|search-traffic-drop
  /website/[domain]          （质量门控制索引；/report/[domain] 308 迁入）
  /technology/[slug]         （≥3 真实使用站点才可索引）

私有区（NOINDEX / robots 禁）：
  /dashboard /bulk /search（控制台）/login /register /admin/seo /analyze/[id] /api/*

API：
  /api/analyze(+bulk) /api/report/[domain] /api/monitor/[domain]
  /api/search/* /api/notify/* /api/auth/* /api/keys/*
  /api/v1/report /api/v1/analyze（X-API-Key）
```

## 5. 调度器（instrumentation.ts 注册）

- `[monitor]` 每 30 分钟：23h 未更新的监控目标重跑管线 + 告警去重推送
- `[search-sync]` 每 6 小时：到期属性同步搜索数据 → 事件 → 关联洞察
- `[discovery]` 每 72 分钟：候选队列自动采集（AUTO_DISCOVER / DISCOVER_DAILY_CAP）
- 启动时：>1h 的 running 调查标记 failed

## 6. 安全

凭证加密（AES-256-GCM）、scrypt+timingSafe、JWT(jose)、API Key 仅存哈希、
Webhook SSRF 守卫（DNS 解析 + 私网拦截）、限流（x-real-ip 防伪造）、
生产无 source maps、SameSite=Lax 会话。

## 7. 已知限制（诚实清单）

- 百度官方：链接提交 API 已验证可用；索引量/关键词无公开 API（待真实站点验证）
- 服务器无 IPv6；相似度/聚类等 O(N²) 能力待数据规模 + Worker 架构
- 监控告警仅站内 + Telegram/Webhook，无邮件通道
