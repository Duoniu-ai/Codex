# SiteIntel Phase 2 方向决策

> 审计配套文档 ｜ 日期：2026-08-23 ｜ 只读（未实施任何 Phase 2 工作）
> 依据：Phase 1 全量验收审计 + 2.0 Master Architecture 冻结文档 + 生产数据健康审计

---

## 1. 决策摘要（唯一推荐顺序）

基于“数据基础成熟度 > 产品价值 > 复杂度”的依赖排序，**唯一推荐 Phase 2 执行顺序**：

```text
1. D. Discovery / Data Flywheel（首个闭环）
2. F. 基础设施收尾（SourceRegistry + Fact 读取 gate 统一 + 保留策略 + 实体页并行化）
3. A. Website Security Intelligence（被动）
4. B. Website SEO Intelligence（Technical 先行）
5. C. Search / Ranking Intelligence
6. E. Intelligence / Recommendation（Growth Intelligence）
```

理由（一句话）：**候选队列已有 1032 条 pending 且 SAN 噪声主导，把“量”变成“资产”是当前唯一当务之急；Security/SEO/Ranking 全部依赖更成熟的页面与观测数据，应在其后。**

---

## 2. 候选方向比较

| 维度 | A. Security | B. SEO | C. Ranking | D. Discovery | E. Intelligence/Recommendation | F. 基础设施 |
|---|---|---|---|---|---|---|
| 产品价值 | 高（风险/合规） | 高（获客） | 中（差异化） | 高（数据资产复利） | 高（闭环收口） | 中（底座） |
| 数据基础 | TLS/安全头/技术（已有被动） | Technical SEO 8 项 audit（已有） | 无搜索观测数据 | Candidate 1032 pending / score / probe / 退避（已有） | 依赖 Security/SEO 数据 | 大部分已有 |
| 当前已有能力 | tls_expiry/domain_expiry/安全头存在性 | title/desc/H1/robots/sitemap/JSON-LD/内链 + 8 项 audit | 无 | 多源入池、评分、探活、退避、classifyDomain tier | 22 项 audit → 计算式推荐 + AI 解释 | read-cache/Job/JobRun/SafeFetch |
| 缺失数据 | 依赖清单、版本覆盖、头值、CVE 库 | canonical/noindex 采集、Page 级模型、搜索观测 | 引擎观测（仅 Google 可验证） | analyzedValue 回填、根域限额、SAN 降噪入池 | Finding 状态机、Action/Verification 表 | SourceRegistry 表、读取 gate、保留策略 |
| 后端工作 | 中（新采集 + 映射） | 中（采集扩展 + Page） | 高（GSC 接入 + 观测表） | 低-中（回填列 + 队列查询 SQL 化） | 中（状态机 + 落库） | 低 |
| 数据库工作 | SecurityFinding（Phase 4） | Keyword/SearchEngine（Phase 5） | KeywordObservation/Ranking（Phase 6） | Candidate 加 2-3 列（无新表） | Recommendation/Action（Phase 7） | SourceRegistry 1 表 + 索引 |
| 前端工作 | Security 视图 | SEO 视图 | Ranking 视图 | Admin 候选队列页 | Action Center | 无 |
| 风险 | 定位漂移（主动扫描） | 引擎能力虚构 | 数据不可得 | 扩量失控（有禁令） | 语义越界 | 低 |
| 复杂度 | 中 | 中-高 | 高 | 低-中 | 中 | 低 |
| 对核心定位贡献 | 回答“哪里存在风险” | 回答“为什么搜索表现不足” | 回答“排名如何变化” | 数据资产自我增强 | 回答“下一步做什么” | 让以上都可持续 |
| 为什么现在做/不做 | 依赖 Page/依赖数据，Phase 4 更稳 | 依赖 Page 模型，Phase 5 | 无观测数据且引擎受限，Phase 6 | **现在：数据最成熟、风险最低、闭环最短** | 依赖前序数据，Phase 7 | 现在/紧随 D：低风险高杠杆 |

---

## 3. 推荐理由展开：为什么 D 是唯一首选

1. **数据基础最成熟**：DiscoveryCandidate 已有 18 列（source/priority/score/probeStatus/backoff/classifyDomain tier），缺的只是 `analyzedValue` 价值回填与队列观测，属于“补列”而非“新建领域”。
2. **质量 > 数量 的落地**：当前 pending 1032、analyzed 163、SAN 源主导。候选价值回填 + 根域限额 + SAN 降噪入池直接兑现“不扩大 Discovery 数量”的原则。
3. **任务层已就绪**：Step 7 的 Job/JobRun 可包 discovery 会话；Step 8 的缓存基建可支撑队列页。
4. **不触碰新业务表**：本闭环不加 Security/SEO/Keyword 表，符合 ADR-002/冻结纪律。
5. **复利效应**：候选→Target 关联（sourceCandidateId）+ 价值反馈让下一轮发现由数据库驱动，是“数据 flywheel”的起点，也是 Security/SEO/Ranking 的原料来源。

---

## 4. Website Security Intelligence 特别分析

### 4.1 目标与边界

分析完成后回答：明显安全风险、风险层面（JS/Frontend/Backend/Server/Configuration/Dependency）、证据、严重度、影响范围、修复建议、修复后验证。

**严格区分（冻结）**：

| 类别 | 允许 | 举例 |
|---|---|---|
| Passive / Safe / Non-invasive | ✅ | TLS 配置、安全头、暴露服务指纹、过期证书、技术版本（公开响应）、robots/开放目录（页面元数据） |
| Active scanning / Exploitation | ❌ 禁止 | 漏洞利用、暴力破解、端口扫描、注入、持久化 |

### 4.2 数据承载评估（复用优先）

| 安全事实 | 现有承载 | 结论 |
|---|---|---|
| TLS 配置/证书 | Fact(tls_certificate) + Entity(cert) | ✅ 可承载 |
| Security Headers | Fact(http_status) 仅存在性 | ⚠️ 需扩展头值字段 |
| 技术/版本 | Fact(technologies) | ⚠️ 版本覆盖有限 |
| 依赖清单/依赖风险 | 无 | ❌ 需新采集（Phase 4 前置） |
| 已知漏洞 | 无 | ❌ 需 Vulnerability 实体（Phase 4） |
| Finding 状态机 | 无 | ❌ 需 SecurityFinding（Phase 4） |
| Remediation / Verification | 无 | ❌ 需新表（Phase 4） |

结论：Phase 4 前**不建 SecurityFinding/Vulnerability 表**；先以“被动配置检查”形式映射到现有 Fact/Insight/Event。

### 4.3 为什么不在 Phase 2 首位

依赖成熟的技术版本与依赖数据（Phase 3 的 Page/画像深化），且 UI 承诺纪律要求“被动配置检查”标注，避免定位漂移；价值高但前置成本高。

---

## 5. Website SEO Intelligence 特别分析

### 5.1 目标

回答：SEO 状态、页面结构问题、Technical SEO、Indexability、Canonical、robots、sitemap、content signals、keyword opportunities、ranking opportunities、search visibility、页面级问题、内容方向、增长建议。

### 5.2 搜索引擎能力矩阵（冻结，禁止虚构）

| 引擎 | Index | Keyword | Ranking | CTR/Traffic | API | 能力 |
|---|---|---|---|---|---|---|
| Google | ✅ GSC 官方 | ✅ 查询级 | ✅ position | ✅ | ✅ Search Console API | STRONG（用户验证后） |
| Bing | ⚠️ Webmaster 部分 | ⚠️ 需验证 | ⚠️ 需验证 | ⚠️ 需验证 | ✅ IndexNow/Webmaster | PARTIAL（待接入验证） |
| Baidu | ⚠️ 网页端有 | ❌ 无公开 API | ❌ | ❌ | ✅ 链接提交（已验证） | PARTIAL（submission only） |
| Shenma | ❌ 无可靠 API | ❌ | ❌ | ❌ | ❌ | UNSUPPORTED（仅估计） |
| Sogou | ❌ 无可靠 API | ❌ | ❌ | ❌ | ❌ | UNSUPPORTED（仅估计） |

原则：只记录实际可获取数据；无真实数据时 UI 显示“暂无数据”，禁止估计冒充事实。

### 5.3 为什么不在 Phase 2 首位

Keyword/Ranking/Opportunity 依赖 Page 级模型与搜索观测（Phase 3/5 前置）；当前 Technical SEO 基础可先在现有采集上深化，但价值密度低于 Discovery 治理。

---

## 6. Phase 2 首个闭环范围（供后续授权参考，本轮未实施）

### 6.1 D-1 Discovery 价值回填

- Candidate 增加 `analyzedValue` / `rootYield` 回填列（需单独 Schema 授权）。
- 候选→Target 关联 `sourceCandidateId`。
- 每根域候选上限 + 低价值标记（复用 classifyDomain tier）。

### 6.2 D-2 SAN 降噪

- 源入池时子域/基建降权（`isInfra` 入池逻辑，Step 7 方案已预留）。
- 消费端按根域限额（rootDomainId soft link 已存在）。

### 6.3 D-3 Admin 队列观测

- 候选队列页（pending/analyzed/failed/score/source 分布）。
- 基于 Step 8 缓存基建，避免全表扫描（当前每 15s 全量拉 pending 为已知风险点）。

### 6.4 明确不做

- 不建 Security/SEO/Keyword/Ranking 表；不扩量；不引入 Redis/搜索/图数据库。

---

## 7. 决策记录

| 项 | 决议 |
|---|---|
| Phase 2 首选 | D. Discovery / Data Flywheel |
| 次选 | F. 基础设施收尾（SourceRegistry + 读取 gate + 保留策略） |
| 第三 | A. Security（被动，Phase 4 建表） |
| 第四 | B. SEO（Technical 先行，Phase 5） |
| 第五 | C. Ranking（Phase 6，依赖观测） |
| 第六 | E. Intelligence/Recommendation（Phase 7） |
| 状态 | 建议（待人工审核授权） |

---

## GitHub Publication

- Repository: Duoniu-ai/Codex
- Branch: main
- Commit: （提交后回填）
- Tag: 无
- File: SITEINTEL_PHASE2_DIRECTION_DECISION.md
- Remote verification: PASS（提交推送后回填）
