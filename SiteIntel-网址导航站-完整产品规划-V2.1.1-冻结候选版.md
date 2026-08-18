# SiteIntel 网站发现平台 — 完整产品规划 V2.1.1（冻结候选版）

> 本版以 V2.1 为基础，根据产品基线审计完成一致性修订。V2.1 的产品方向不变；本版只负责消除阻断性矛盾、补齐架构前必须明确的决策，并作为技术架构设计的唯一候选基线。

---

# 0. 冻结前结论

产品核心链路保持不变：

```text
用户需求
↓
Query 解析
↓
Task 识别
↓
Constraint / Tag 提取
↓
候选网站
↓
可解释推荐
↓
相似 / 替代比较
↓
点击
↓
收藏 / 再次发现
```

核心原则：

1. Task ≠ Query。
2. Constraint ≠ Task。
3. SEO 实体优先，Query Landing Page 选择性建设。
4. 不把技术变化或可访问性称为“网站活跃度”。
5. P0 只证明核心闭环，不追求功能完整。
6. SiteIntel 提供网站事实与变化数据；导航站拥有 Task、Relation、Recommendation、User Behavior 等独立资产。

---

# 1. 产品定位

对外可以使用“网址导航”降低理解成本；产品内部定位统一为：

> **网站发现平台（Website Discovery Platform）**

传统导航：

```text
分类 → 网站 → 点击
```

本产品：

```text
需求 → 任务 → 发现 → 推荐 → 比较 → 选择
```

---

# 2. 核心数据模型

## 2.1 Category

回答：

> 这个网站属于什么类型？

例如：

- AI 工具
- 设计工具
- 在线工具

## 2.2 Task

回答：

> 用户想完成什么事情？

例如：

- 制作 PPT
- 压缩 PDF
- 生成 Logo
- 翻译文档

Task 必须稳定、可复用，并由编辑治理。

## 2.3 Query

用户实际输入。

例如：

> 免费做 PPT 的 AI 网站

Query 不直接等于 Task。

## 2.4 Constraint

Query 中的筛选条件。

例如：

- 免费
- AI
- 在线
- 中文
- 无需注册

## 2.5 Alias

Task 的常见表达。

例如：

```text
Task：制作 PPT
Aliases：做PPT / 制作演示文稿 / 在线做幻灯片
```

---

# 3. Task 治理规则

Task 数据结构：

```text
Task
├── name
├── slug
├── description
├── aliases
├── status
└── websites (N:N)
```

治理规则：

- 不能因为某个关键词有搜索量就建立新 Task。
- Task 必须具有稳定需求。
- Task 页开放索引前：
  - 候选网站 ≥5；
  - 至少 3 个网站具备完整、可解释的推荐依据。
- 低于阈值可以保留内部数据，但不得生成开放索引页面。
- Task 命名使用用户语言。

---

# 4. 搜索逻辑

```text
Query
↓
Query Parsing
↓
Task
+
Constraint
↓
Candidate Websites
↓
Ranking
↓
Results
```

示例：

```text
Query：免费做PPT的AI网站

Task：制作PPT
Constraint：
- 免费
- AI

结果：
先找到“制作PPT”的候选集合，
再根据 Constraint 过滤和排序。
```

P0 使用：

- 规则；
- Task aliases；
- Tag/Constraint 映射。

P0 不引入 LLM 或复杂语义检索。

---

# 5. 推荐与关系

## P0

### 相似

可由以下结构化因素计算：

- Task 重合；
- Category 接近；
- Tag/Feature 重合。

### 替代

优先采用人工精选。

### 推荐理由

必须可以解释，例如：

- 与“制作PPT”任务匹配；
- 支持免费在线使用；
- 核心功能已验证；
- 最近检测正常访问。

不得生成空泛模板化“强烈推荐”。

## P1

- 相关关系；
- 对比页规模化。

## P2

复杂竞争、生态、上下游关系仅在真实需求与数据充分时评估。

同 CDN、同 ASN、同技术栈等不能直接生成用户可见的“相关网站”。

---

# 6. 数据质量评分【补齐】

数据质量评分是内部质量控制指标，不等于“网站好坏”。

建议由以下维度构成：

| 维度 | P0 |
|---|---|
| 基础信息完整度 | 必须 |
| Category / Task 完整度 | 必须 |
| 推荐依据完整度 | 必须 |
| 当前可访问状态 | 必须 |
| 数据新鲜度 | 必须 |
| 人工核验状态 | 推荐 |

P0 可使用规则评分。

具体权重不写死在产品文档中，应在后台可配置。

索引门槛：

```text
候选网站 ≥5
+
至少 3 个具备完整推荐依据
+
数据质量达到最低阈值
+
页面具备独立信息价值
```

---

# 7. 网站状态与变化信号

统一使用：

- 当前网站状态；
- 最近检测时间；
- 数据更新时间；
- 近期变化；
- 连续可用性。

禁止直接宣称：

- 网站内容活跃；
- 网站运营活跃；
- 网站业务增长；
- 网站质量更高。

统一数据语义：

```text
website_snapshots
= 原始检测快照

website_status
= 当前聚合状态

website_metrics
= 长期统计
```

不再使用含义模糊的 `Activity` 作为核心数据实体。

---

# 8. SEO 架构

优先级：

1. Website
2. Category
3. Task
4. 精选对比
5. 精选专题

Query Landing Page 属于第二阶段能力。

默认：

```text
/task/ppt-maker                  可索引
/search?q=免费做PPT              默认 noindex
/task/ppt-maker?price=free       默认 noindex
/free-ppt-maker                  仅验证价值后建设
```

## Category 与 Task 页面防重复

Category 回答：

> 这一类网站是什么？如何分类和选择？

Task 回答：

> 我想完成这件事，有哪些网站可以完成？为什么推荐？

规则：

- 不互相 canonical；
- H1 不同；
- 正文结构不同；
- 选择指南不同；
- 排序逻辑不同；
- 没有独立信息价值的页面宁可 noindex。

需要持续检查：

- 文本相似度；
- 网站集合重叠度；
- 搜索词重叠度。

---

# 9. 冷启动与种子 Task

第一阶段目标：

> 30–50 个高质量 Task × 每个 5–10 个候选网站。

不追求：

> 10,000 个网站 + 大量空分类。

Task 选型来源：

- 人工需求判断；
- 百度/Bing 搜索结果观察；
- 搜索建议与相关搜索；
- 合法可用的第三方趋势/关键词工具；
- SiteIntel 已有网站数据；
- 后续站内搜索日志。

“高搜索量”不是唯一标准。

优先：

```text
需求稳定
+
候选网站充足
+
用户价值明确
+
能够形成高质量页面
```

---

# 10. 编辑内容生产 Pipeline

每个种子 Task：

1. 定义 Task；
2. 撰写用户需求说明；
3. 收集 5–10 个候选网站；
4. 核验基础信息；
5. 标注 Category / Tag / Feature；
6. 建立推荐依据；
7. 建立相似或人工替代关系；
8. 撰写 Task 页内容；
9. 审核后发布。

Phase 0 先完整跑通少量 Task，测算单 Task 人工成本，再扩展。

---

# 11. SiteIntel 与导航站边界

## SiteIntel

负责：

- 域名基础信息；
- 技术栈；
- SSL；
- CDN；
- 网站检测；
- 快照；
- 可访问状态；
- 页面/技术/DNS 变化信号。

## 导航站

独立拥有：

- Category；
- Task；
- Tag；
- Website ↔ Task；
- Website Relations；
- Recommendation；
- Curated Picks；
- User Behavior；
- Search Logs。

默认：

> SiteIntel → Navigation 单向同步。

导航站不能直接暴露 SiteIntel 内部数据库结构。

---

# 12. P0 MVP

P0 必须全部在 Phase 1 完成。

## 用户端

- 首页；
- Category；
- Task 页面；
- Website 详情；
- 搜索；
- Query → Task + Constraint 基础解析；
- 相似网站；
- 人工精选替代网站；
- 可解释推荐理由；
- 网站状态与变化信号；
- 本地收藏；
- 用户提交网站；
- 网站数据纠错入口；
- 编辑精选 / 今日推荐。

## SEO

- Sitemap；
- Schema；
- Canonical；
- noindex；
- Task 索引阈值；
- 自动降级机制。

P0 闭环：

```text
需求
↓
Task + Constraint
↓
候选网站
↓
推荐理由
↓
相似 / 替代
↓
点击
↓
收藏 / 再发现
```

---

# 13. 收藏与用户身份

P0：

- 不要求注册；
- 收藏保存在本地；
- 可使用匿名 visitor_id 记录非敏感行为；
- 不承诺跨设备同步。

P1：

- 账号；
- 云端同步；
- 本地收藏迁移。

---

# 14. Tag 数据模型

统一：

```text
tags
- id
- name
- slug
- tag_type
- description
```

`tag_type`：

- attribute
- feature
- constraint
- scenario

不要建立四套 Tag 表。

---

# 15. 数据库核心实体

P0 核心：

```text
websites
categories
tags
website_tags

tasks
website_tasks

website_relations

website_snapshots
website_status
website_metrics

recommendations / recommendation_evidence
curated_picks

submissions
reports

user_events
search_logs
```

根据实际 Schema 可以合并辅助表，但不能改变产品语义。

---

# 16. Version / Priority / Phase 映射

| 概念 | 含义 |
|---|---|
| Version | 文档版本 |
| P0/P1/P2/P3 | 功能优先级 |
| Phase | 实际开发顺序 |

硬规则：

> P0 必须在 Phase 1 完成。

| 优先级 | 开发阶段 |
|---|---|
| P0 | Phase 1 |
| P1 | Phase 2–3 |
| P2 | 后续 |
| P3 | 长期探索 |

---

# 17. 开发路线

## Phase 0：数据与工程准备

- SiteIntel API 能力审计；
- 数据同步契约；
- 核心 Schema；
- Task/Tag/Relation 数据模型；
- 数据质量评分；
- 状态信号模型；
- 编辑 Pipeline；
- 30–50 种子 Task；
- SEO 自动降级。

## Phase 1：P0 MVP

完成第 12 节全部 P0。

## Phase 2：P1

- 账号；
- 云端收藏；
- 更新提醒；
- 站长认领；
- 相关关系；
- 对比页；
- 搜索日志分析；
- 编辑工作台增强；
- 商业精选位。

## Phase 3：增长与推荐演进

- 行为数据推荐；
- Task 词典扩展；
- 经验证的 Query Landing Page；
- A/B 测试；
- SEO 差异化治理。

## Phase 4：长期能力

- 个性化；
- 更复杂的搜索；
- 复杂关系；
- 数据飞轮；
- 数据/API 服务。

---

# 18. 推荐系统升级条件

不以“网站数量达到多少”作为唯一触发条件。

升级需要：

- 行为事件达到可分析规模；
- 数据质量稳定；
- 有基线指标；
- 可以进行离线验证或 A/B 测试；
- 新模型能够证明优于规则系统。

网站数量只是覆盖度，不代表推荐系统可以升级。

---

# 19. Task 识别验收

建立固定测试集：

- 100 条人工标注 Query；
- 每条标注：
  - 是否存在可识别 Task；
  - 正确 Task；
  - Constraint。

只统计：

> 当前种子 Task 覆盖范围内、理论上应该能够识别的 Query。

验收：

```text
Task Top-1 命中率 ≥85%
```

Constraint 单独统计，不混入 Task 指标。

---

# 20. 第一阶段指标

重点：

- 任务成功率；
- 搜索 → 点击率；
- 推荐理由展开率；
- 收藏率；
- Task 页面自然流量；
- 数据错误率；
- 用户纠错率；
- 数据新鲜度。

“用户纠错率”对应 P0 的数据纠错入口。

---

# 21. P0 最终验收

必须验证：

### 产品

- 用户能否理解输入需求；
- Task 是否正确；
- Constraint 是否正确；
- 是否能找到候选网站；
- 推荐理由是否可解释；
- 相似/替代是否合理；
- 收藏是否可用。

### 数据

- Task/Category/Tag 不混淆；
- 无伪造状态数据；
- 无低质量开放索引 Task 页；
- 数据质量评分可计算；
- SiteIntel 同步可追踪。

### SEO

- Task/Category 不重复；
- 参数页默认 noindex；
- Sitemap 仅包含合格页面；
- 低于阈值页面自动降级。

---

# 22. 最终产品定义

> **SiteIntel 网站发现平台：根据用户想完成的事情，帮助用户发现、理解、比较并选择合适的网站。**

最终边界：

```text
SiteIntel
= 观察网站，生产网站事实与变化数据

导航站
= 理解需求，组织网站知识，帮助用户发现与选择
```

---

# 23. 冻结规则

本 V2.1.1 是技术架构设计候选基线。

冻结前只允许继续检查：

1. 是否仍有 Task / Query 混用；
2. P0 与 Phase 是否同步；
3. “活跃度”术语是否被错误使用；
4. SEO 页面是否违反实体优先；
5. P0 数据模型是否覆盖 P0 功能。

通过最终一致性检查后，改名：

`SiteIntel-网址导航站-完整产品规划-V2.1.1-冻结版.md`

之后停止继续扩张产品规划，进入技术架构。
