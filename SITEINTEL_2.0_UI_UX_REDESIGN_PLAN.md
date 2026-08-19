# SiteIntel 2.0 UI/UX 产品重构规划

> 版本：v1.0
> 日期：2026-08-19
> 范围：**仅 SiteIntel**
> 当前基线：Phase 1 / Step 8 已完成并通过验证
> 性质：产品/UI/UX 规划，不是立即执行的代码任务

## 1. 为什么现在需要 UI 重构

SiteIntel 后端和数据能力已经明显复杂化，继续按后台数据模块堆叠页面，会让用户看到“工具集合/数据控制台”，而不是 Website Intelligence 产品。

BuiltWith 的启发是信息覆盖全面；Wappalyzer 的启发是 Technology Catalog 和双向探索清晰。SiteIntel 应吸收“全面”，但用更强的信息层级解决“复杂”。

## 2. 新的 UX 目标

用户进入一个网站分析页后，默认先看到：

1. 它是谁
2. 它如何运行
3. 使用什么技术
4. 最近发生什么
5. 有什么值得关注
6. 与什么存在公开关联
7. 过去发生过什么
8. 证据是什么

不是先看到 Entity / Fact / Relationship / Raw Collection。

## 3. Website Profile 信息架构

```text
Website Profile
├── Identity / Summary
├── Overview
├── Infrastructure
│   ├── IP
│   ├── ASN
│   ├── Provider / Hosting / CDN
│   └── Location
├── Domain / Network
│   ├── DNS
│   ├── Redirects
│   └── Certificate
├── Technology
│   ├── Category
│   ├── Technology
│   ├── Confidence
│   └── Evidence
├── Recent Changes
├── Worth Watching / Insights
├── Relationships / Explore
├── History Timeline
├── Evidence
└── Advanced / Raw Data
```

## 4. 首屏设计原则

首屏不追求显示所有参数，而是形成“情报摘要”。

示例结构：

```text
example.com
网站状态正常

使用 Cloudflare 提供网络服务，检测到 WordPress。
最近 30 天发现 2 项基础设施变化。

[查看变化] [完整详情]

基础设施     技术       最近变化      风险/关注
IP / ASN     12项       2项           1项
```

所有结论必须可下钻到证据。

## 5. Technology UI

技术必须支持：
- 分类展示
- 搜索/过滤
- 技术实体链接
- First Seen / Last Seen
- Added/Removed/Changed
- Confidence
- Detection Evidence
- Alternatives
- Related Technologies

避免首屏几十个技术 chip；采用类别摘要 + 展开全部。

## 6. Technology Profile

长期独立页面：

```text
Technology
├── Overview
├── What it is
├── Detection
├── Websites using it
├── Market Share
├── Countries
├── Languages
├── Top Websites
├── Alternatives
├── Commonly paired technologies
├── History
├── Trends
└── Evidence
```

## 7. History UI

历史必须用时间线表达，不只是一张数据库表。

```text
2024 ─── historical evidence
2025 ─── reconstructed state
2026 ─── SiteIntel observed
                 │
                 ├─ IP changed
                 ├─ Certificate changed
                 └─ Technology added
```

Observed / Reconstructed / Inferred 必须有明显但不过度干扰的标识。

## 8. Change UI

用户语言优先：
- IP 地址发生变化
- 网络服务商发生变化
- SSL 证书已更新
- 检测到新的技术
- 某项技术不再出现

高级层再显示 event type、source、confidence、raw diff。

## 9. Relationship UI

推荐“关联探索”而非“Relationship Graph”。

默认展示：
- 共享 IP
- 共享 ASN
- 共享 Certificate
- 共享 Provider
- 共享 Technology

必须显示关系性质与证据，避免误导为主体归属。

## 10. Evidence UI

每个重要结论支持：

```text
结论
↓
为什么
↓
证据数量
↓
来源
↓
观察时间
↓
查看原始数据
```

## 11. Global Explore

长期形成：
- Websites
- Technologies
- Infrastructure
- Relationships
- Changes

支持 Website ↔ Technology ↔ Infrastructure 双向探索。

## 12. 导航

优先：

`首页 / 网站分析 / 探索 / 网站变化（成熟后） / 工具`

内部数据模型不得成为一级用户导航。

## 13. Responsive / Mobile

必须从设计阶段支持：
- 手机首屏摘要
- 卡片纵向排列
- 横向表格可滚动
- 时间线适配
- 图谱提供列表替代模式
- 触控目标尺寸
- 不依赖 hover

## 14. SEO / Performance

UI 改版不能破坏既有 SEO P0/P1 和 Step 8 Query/Read Performance。

必须：
- 保留语义 HTML
- 保证可抓取核心内容
- 避免大量 JS 才能看到核心结论
- 避免 soft 404
- 保留正确 noindex/index 规则
- 不因动画/图谱造成首屏性能退化

## 15. 设计系统

先建立组件语言，再大规模改页面：
- Typography
- Spacing
- Card
- Badge
- Status
- Evidence
- Timeline
- Table
- Tabs
- Disclosure
- Empty/Error/Loading
- Chart
- Relationship preview

不要每个页面单独设计一套 UI。

## 16. UI 实施阶段

### UI-A：Audit
只读审计现有页面、路由、组件、数据绑定、SEO、移动端和性能。

### UI-B：Information Architecture
确定 Website Profile / Technology Profile / Explore 的最终信息架构。
