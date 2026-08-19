# SiteIntel 2.0 - 后期产品与技术总规划

> 版本：v1.1
> 日期：2026-08-19
> 状态：长期规划基线
> 适用范围：**仅 SiteIntel**
> 当前工程基线：**Phase 1 / Step 8（Query & Read Performance Foundation）已完成并通过验证**

## 1. 文档定位

本文件是 SiteIntel 2.0 的长期产品与技术总纲，不是一次性开发任务书。它定义产品最终形态、数据能力、情报能力、用户体验、技术路线和后期产品化方向。

执行原则：先分析、不直接改代码；小闭环；定向验证；验收后固化；禁止无边界重构。

**Websites 不属于本规划。** GitHub 中的 Websites 资料仅可作为项目关系背景，不得作为 SiteIntel 当前能力、上线状态或开发范围。

## 2. 当前基线

已完成：
- Phase 1 / Step 8：Query & Read Performance Foundation
- 现有 SiteIntel 2.0 产品蓝图及既有数据/架构原则
- Raw → Fact → Entity → Relationship → Snapshot/Observation → Event → Finding/Insight → Recommendation/Action/Verification
- PostgreSQL First / Relationship First / Evidence First / History Matters / Data First / Quality > Quantity / Safe by Default / AI Explains, Data Proves

Step 8 不得作为未来待办重复规划。后续能力必须建立在其上。

## 3. 最终产品定位

SiteIntel = **Website / Internet Entity Intelligence Platform**。

用户输入网站只是入口，真正产品对象是一个持续变化、具备实体、关系、历史、证据和事件的互联网实体。

最终需要回答：
1. 它是什么？
2. 它如何运行？
3. 使用什么技术？
4. 位于什么网络/基础设施？
5. 与哪些公开实体存在关联？
6. 过去是什么样？
7. 发生过什么变化？
8. 为什么得出这些结论？
9. 哪些变化值得关注？
10. 下一步可以做什么？

## 4. 竞品吸收原则

### BuiltWith
吸收：Website Profile、Technology Entity、Technology Taxonomy、First/Last Detected、Historical Technology、Technology Change/Churn、Subdomains、Relationships、Trends、Risk、Notifications、API/Dataset、AI Website Intelligence。

### Wappalyzer
吸收：Technology Catalog、技术实体页、Category、Market Share、Top Websites、Countries、Languages、Alternatives、Technology Distribution、持续重新验证、Browser Extension/分布式观察、Technology → Website 双向导航、技术筛选。

### 不复制竞品边界
暂不把 Sales Intelligence、CRM、Lead Generation、Future Customers、Revenue/Employee enrichment 等作为当前核心；后期可作为数据产品研究方向。

## 5. SiteIntel 四大产品入口

### 5.1 Website Intelligence
网站档案、基础设施、技术、DNS、证书、子域、变化、历史、关联、证据。

### 5.2 Technology Intelligence
Technology Catalog、分类、检测证据、技术历史、市场份额、国家/语言分布、Top Websites、Alternatives、技术生态、趋势。

### 5.3 Infrastructure Intelligence
IP、IPv4/IPv6、ASN、Provider、Hosting、CDN、DNS、Certificate、Datacenter、基础设施关系与历史。

### 5.4 Historical Intelligence
Observed、Historical Backfill、Reconstructed、Inferred、Unified Timeline、Change/Event、Evidence、Confidence。

## 6. 核心数据模型长期方向

### Technology Entity
- technology_id
- name/vendor
- parent/category/subcategory
- description/official_url
- detection rules/signals
- first_seen/last_seen
- status
- confidence
- historical_usage

### Observation / Snapshot
每次有效观测都必须具备时间、来源、采集方式、实体/关系上下文和可追溯证据。

### History Classification
- **Observed**：SiteIntel 自己观测
- **Reconstructed**：由历史证据拼接出的历史状态
- **Inferred**：模型推断，不得伪装成事实

三者必须可区分，并记录 source、timestamp、method、confidence、evidence。

### Change/Event
统一覆盖：Technology Added/Removed/Changed、IP Changed、ASN Changed、DNS Changed、Certificate Changed、Hosting/CDN Changed、Subdomain Added/Removed、Relationship Added/Removed、状态变化。

### Relationship
关系必须具备 type、first_seen、last_seen、confidence、evidence、source；禁止把“共享 IP”等弱关联直接表达为主体归属。

## 7. Historical Backfill 战略

SiteIntel 从 2026-08-19 开始持续观测时，不能把此前时间全部视为“无历史”。

长期采用：
1. SiteIntel 自有观察
2. 合法公开历史来源
3. 历史快照/缓存/公开档案
4. 多证据重建
5. 推断层

所有回溯结果必须标明证据等级和来源，不得伪造历史快照。

## 8. Evidence & Confidence

目标链：

`Source → Raw → Fact → Evidence → Entity/Relationship → Observation/Event → Insight`

任何重要结论都应回答：
- 为什么？
- 证据是什么？
- 何时发现？
- 来自哪里？
- 置信度如何？

默认 UI：结论 → 解释 → 证据 → 原始数据。

## 9. Technology Intelligence 长期能力

Technology 不再只是字符串字段，而是独立情报实体。

目标字段/能力：
- ID / Parent / Category / Subcategory
- Vendor / Description / Official URL
- Detection method / signals
- First Seen / Last Seen
- Added / Removed / Reappeared / Changed
- Confidence / Evidence Count
- Websites using it
- Market Share
- Country / Language distribution
- Top Websites
- Alternatives
- Frequently paired technologies
- Growth / Decline trend

## 10. Infrastructure Intelligence

IP、ASN、DNS、Certificate、Hosting/CDN 都应逐步实体化并历史化。

例如 Certificate 应具备 issuer、subject、SAN、serial、validity、algorithm、fingerprint、关联网站和历史。

## 11. Relationship Intelligence

形成 Website ↔ Website、Website ↔ IP、Website ↔ ASN、Website ↔ Certificate、Website ↔ Technology、Website ↔ DNS、Website ↔ Provider 等关系网络。

长期目标不是“图形展示”，而是可查询、可解释、可追溯的关系数据资产。

## 12. Change Intelligence

建立统一 Change Engine，把所有重要实体变化转成 Event，并形成：

`Snapshot → Diff → Event → Evidence → Insight → Notification`

## 13. Technology Market Intelligence

后期形成 Technology → Websites 的双向入口，并支持市场份额、国家、语言、流量层级、Top Websites、Alternatives、技术组合和趋势。

## 14. Risk Intelligence

后期建设可解释风险：IP、ASN、Hosting、Certificate、Technology、Infrastructure、Relationship、Historical、Exposure。

风险分数必须有证据与原因，不能只输出一个黑盒数字。

## 15. Trend Intelligence

长期支持：Technology Growth/Decline、Website Growth、IP/ASN/Infrastructure Growth、Regional Adoption、Technology Adoption/Decline、市场结构变化。

## 16. AI Intelligence

不直接复制 BuiltWith 的 AI Index。SiteIntel 后期形成自己的：
- AI Framework
- AI SDK/API
- LLM/AI service
- MCP/OpenAPI
- llms.txt
- AI crawler
- Agent Interface
- AI Discoverability
- AI Readiness

原则：AI 负责解释、总结、推荐；数据和证据负责证明。

## 17. Monitoring / Notification

成熟后支持 IP/DNS/Certificate/Technology/Subdomain/Relationship/Risk 变化提醒，并允许用户订阅网站或实体。

## 18. API / Dataset / Data Product

后期开放 Website、Technology、Infrastructure、History、Change、Relationship、Evidence 等稳定接口；再考虑 Dataset/Data Feed/MCP/Enterprise。

## 19. Browser Extension / Distributed Observation

长期研究 Browser Extension、轻量 Probe、边缘观察节点等分布式观察方式。必须遵循隐私、安全、合法公开数据和最小采集原则。它属于后期架构研究，不是当前开发任务。

## 20. 2.0 UI/UX 产品重构

SiteIntel 当前 UI 应进入产品化重构阶段。目标不是复制 BuiltWith，而是吸收其信息完整度并改善信息层级。

### 20.1 核心原则
- 结论优先
- 用户问题优先于内部模块
- 渐进式信息披露
- 高信息密度但不信息噪音化
- 专业数据可深入，普通用户无需理解内部数据模型
- Desktop / Mobile 一致

### 20.2 Website Profile
推荐顺序：
1. 网站身份/一句话结论
2. Overview
3. How it runs / Infrastructure
4. Technology
5. Recent Changes
6. Worth Watching / Insights
7. Relationships / Explore
8. History Timeline
9. Evidence
10. Raw/Advanced Data

### 20.3 Technology Profile
独立 Technology 页面应逐步提供：描述、分类、检测方式、使用网站、市场份额、国家、语言、Top Websites、Alternatives、技术组合、历史、趋势、证据。

### 20.4 全局导航
建议围绕用户任务：
- 首页
- 网站分析
- 探索
- 网站变化（成熟后）
- 工具
