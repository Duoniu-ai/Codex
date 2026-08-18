# SiteIntel 2.0 Product Architecture（产品架构冻结）

> 日期：2026-08-18 ｜ 只读冻结 ｜ 状态标记：EXISTING / PARTIAL / PLANNED / RECOMMENDED

---

# 1. 产品定义

SiteIntel 不是工具集合。用户输入一个网站，得到的是一个**持续演化的数据智能画像**，并能回答六个问题：

1. What：这个网站是什么？
2. How：它怎么运行？
3. Connected：它和谁/什么基础设施/什么技术相关？
4. What Changed：它发生了什么变化？
5. What Is Wrong：它有什么问题与风险？
6. What Should I Do：我下一步怎么做（修复/优化/增长）？

核心闭环：Understand → Detect → Explain → Recommend → Act → Verify → Monitor。

---

# 2. 用户旅程（冻结）

```text
输入网站（首页/工具/分享链接）
  → 分析进度（SSE）
  → Website Profile（第一屏判断）
  → 钻取（Security/SEO/Technology/Infrastructure/History/Relationships）
  → 证据（Evidence Chain）
  → 问题（Findings）
  → 建议（Recommendations/Action Plan）
  → 执行后重分析（Verification）
  → 监控（Monitor/Alert）
  → 再次访问（Timeline/变化）
```

现状：输入→Profile→钻取→证据 EXISTING；问题=Insight+Audit（PARTIAL）；建议=计算式推荐（PARTIAL）；执行/验证 PLANNED；监控 EXISTING；Timeline EXISTING。

---

# 3. 信息架构（当前 vs 目标）

## 当前（EXISTING）

```text
/
├── 首页（用户化 + 真实示例）
├── 产品页（website/search/infrastructure/technology/relationship/monitoring intelligence）
├── 工具页（website-analysis/dns/ip/ssl/technology）
├── 搜索引擎入口页（baidu/bing/google）
├── 指南/案例
├── /website/[domain]（质量门索引）
├── 实体页：/ip /asn /certificate /organization /technology
├── /report（探索索引）
└── 私有区：dashboard/bulk/search 控制台/admin/analyze/claim
```

## 目标（PLANNED 增量，不推翻现有）

```text
Website Profile（第一屏 Health + 六大维度入口）
  ├── Security 视图
  ├── SEO 视图（Technical → Keywords/Rankings/Opportunities）
  ├── Technology 视图
  ├── Infrastructure 视图
  ├── History/Timeline 视图
  ├── Relationships/Explore 视图
  ├── Recommendations/Action Center 视图
  └── Monitoring 视图（Alert 历史/趋势）
```

---

# 4. Website Profile 八层（冻结）

| 层 | 问题 | 现状 |
|---|---|---|
| 1 | 这个网站怎么样？ | EXISTING（首屏 4 指标 + 一句话结论） |
| 2 | 为什么？ | PARTIAL（Audit 维度分 + 结论；缺维度解释文案） |
| 3 | 证据是什么？ | EXISTING（证据链展开） |
| 4 | 发生过什么？ | EXISTING（History/Events） |
| 5 | 存在什么问题？ | PARTIAL（Insight/Audit/Contradiction；缺 Finding 状态机） |
| 6 | 怎么解决？ | PARTIAL（Recommendation 计算式） |
| 7 | 下一步做什么？ | PLANNED（Action Center） |
| 8 | 修复后改善了吗？ | PLANNED（Re-scan/Verification） |

---

# 5. 六大领域产品视图

## 5.1 Website Intelligence（EXISTING，深化）
- 当前：Identity/Domain/Infrastructure/Technology/证书/注册/关联站点。
- 目标：加 Page 级视图、地理、URL 规范化；数据互跳（IP→网站、技术→网站、证书→网站）EXISTING。

## 5.2 Technology Intelligence（PARTIAL）
- 当前：技术列表（分类/版本/置信度/首次/最近观测）+ 技术反查页。
- 目标：版本分布、依赖清单、共现技术、历史变化聚合。

## 5.3 Security Intelligence（PARTIAL → Phase 4）
- 当前：TLS/安全头/到期洞察（被动）。
- 目标：Security 视图 = Findings 列表（severity/confidence/evidence）+ 修复计划（P0/P1/P2）+ 验证回环。
- 界面承诺纪律：在 Finding 模型落地前，UI 明确标注“被动配置检查”，不宣传漏洞扫描。

## 5.4 SEO Intelligence（PARTIAL → Phase 5）
- 当前：Technical SEO 8 项 audit。
- 目标：Technical 详情 + Indexability + Keywords/Rankings + Opportunities/Content Gap。
- 界面承诺纪律：无真实搜索数据时显示“暂无数据”，禁止虚构。

## 5.5 Change Intelligence（EXISTING）
- 当前：Timeline（事件/洞察）。
- 目标：按维度筛选（infrastructure/security/seo/ranking）+ Finding 状态变化并入时间线。

## 5.6 Growth Intelligence（PARTIAL → Phase 7）
- 目标：Action Center = “你现在最值得做的 10 件事”，按 Impact×Effort×Urgency 排序，每条可标记执行并重扫验证。

---

# 6. Explore / Relationship Graph

- 现状：报告内图谱（自研 SVG 力导向）+ 实体页。
- 目标（PLANNED，Phase 8）：全局 Explore（从任何实体开始，IP→ASN→Org→Domains、Technology→Websites），仍基于 PostgreSQL。

---

# 7. Timeline

- 现状：Event 时间线 + 洞察。
- 目标：统一时间线（安全/SEO/技术/基础设施/排名变化），支持按维度/严重度筛选，事件可跳转证据。

---

# 8. Website Health（可解释评分）

- 现状：Audit 维度分（technical/security/infrastructure/seo/content/performance），unknown 不计分。
- 目标：Health 卡 = 总分 + 各维度分 + “为什么”（命中的 check、缺失数据说明）；禁止简单平均。

---

# 9. Action Center（PLANNED）

```text
P0 Critical（安全/数据）→ P1 High（核心）→ P2 Opportunity（增长）
每条：问题/证据/影响/修复方式/预期成本/验证方式
```

安全与 SEO 共用同一入口（统一 Finding→Recommendation→Action 链）。

---

# 10. Monitoring

- 现状：daily/weekly/monthly 重扫 + Telegram/Webhook 告警 + dashboard 列表。
- 目标（PLANNED）：告警历史（AlertDelivery）、趋势视图、按维度监控（security/seo/ranking）、邮件通道。

---

# 11. Search（PLANNED，Phase 8）

- 产品形态：实体检索（域名/IP/ASN/Org/Technology/Keyword），支持组合条件；先 PG 后专用搜索。

---

# 12. API 产品（PLANNED 按阶段）

- 现有：v1 analyze/report（API Key + 配额）。
- 目标：Website/Entity/Relationship/Search/History/Events/Security/SEO/Keywords/Rankings/Recommendations/Monitoring 按数据成熟度开放；API 文档持续更新。

---

# 13. 当前前端差距

| 差距 | 现状 | 计划 |
|---|---|---|
| 报告页单体组件（64KB） | report-page.tsx | 按区块拆分（Phase 2） |
| 无 Security/SEO 独立视图 | 报告内嵌 | Phase 4/5 |
| 无 Action Center | 报告内推荐 | Phase 7 |
| 无全局 Explore | 实体页互跳 | Phase 8 |
| 无 Compare | — | Phase 8 可选 |
| Admin 无候选队列/治理队列 | data-quality/seo 页 | Phase 2 |
| 监控视图弱 | dashboard 列表 | Phase 8 |

---

# 14. 产品优先级（冻结，按依赖）

1. 报告查询性能 + 区块拆分（Phase 1，产品体验基础）。
2. Discovery 队列可见性（Phase 2，数据资产）。
3. Website 画像深化（Page 级）（Phase 3）。
4. Security 视图（Phase 4）。
5. SEO 视图（Phase 5）。
6. Keyword/Ranking 视图（Phase 6）。
7. Action Center（Phase 7）。
8. Monitoring/Explore（Phase 8）。
9. API 产品化（Phase 9）。

---

# 15. 结论

产品架构冻结为：**一个入口（网站）、一套画像（Profile）、一条闭环（发现→判断→建议→行动→验证→监控）、一个探索面（关系图谱）**；所有新页面都是同一数据底座上的投影，禁止新增孤立工具页。
