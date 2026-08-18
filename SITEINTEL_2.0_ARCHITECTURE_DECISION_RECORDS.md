# SiteIntel 2.0 Architecture Decision Records（冻结）

> 日期：2026-08-18 ｜ 只读冻结 ｜ 每个 ADR：Context / Decision / Consequences / Alternatives

---

## ADR-001 Repository Separation

- **Context**：siteintel 仓库长期混入文档与代码工作副本，Codex 仓库曾混入其他项目文件；生产来源需唯一可追溯。
- **Decision**：`Duoniu-ai/siteintel`（private）= SiteIntel 唯一正式产品代码/数据库/部署/测试仓库；`Duoniu-ai/Codex` = SiteIntel 文档/报告/状态/架构知识库。代码不进 Codex；报告不进 siteintel tracked（README/CURRENT_ARCHITECTURE 除外）。
- **Consequences**：生产可重建性明确（main + deploy.sh）；文档发布链路明确（本地工作树 → Codex main）；需维护双仓库纪律。
- **Alternatives**：单仓库全放（文档污染代码历史）；每项目独立知识库（过度拆分）。

---

## ADR-002 PostgreSQL First

- **Context**：未来可能引入 GraphDB/搜索/OLAP；当前数据规模（约 400 Target/3k Entity/6k 关系/41MB）远未到瓶颈。
- **Decision**：长期以 PostgreSQL 单体为唯一数据底座；JSONB 仅用于载荷与原文；业务状态一律结构化表；分区/专用引擎按阈值触发（见 Data Architecture §19）。
- **Consequences**：开发简单、事务一致、备份单一；规模到后需渐进演进而非一步到位。
- **Alternatives**：Neo4j（关系查询早期优化但运维复杂）；专用搜索集群（当前无压力）。

---

## ADR-003 Relationship First

- **Context**：实体图谱是最大数据资产（5.9k+ 关系），反查能力是差异化。
- **Decision**：Relationship 是一等资产：统一 EntityRelationship（type/kind/confidence/evidence/first/last seen），禁止为新关系随意建独立表；关系变化纳入时间模型。
- **Consequences**：Explore/反查可复用同一查询层；关系噪声（SAN 子域）需治理。
- **Alternatives**：独立关系表（每类一张）→ 查询与治理碎片化。

---

## ADR-004 Evidence First

- **Context**：报告曾依赖快照 JSONB 的脆形状；洞察证据链曾断裂。
- **Decision**：结论必须可追溯 Source→Raw→Fact→Evidence→Entity/Relationship→Observation→Finding/Insight；Fact 为报告唯一读源；关键载荷写入前 zod 校验；Evidence 渐进补 source_url/raw_reference。
- **Consequences**：开发多一层校验成本；换源/重放/审计能力显著提升。
- **Alternatives**：继续 JSONB 直接投影（形状漂移风险，已出现 tuple 假设脆弱问题）。

---

## ADR-005 Safe Fetch Boundary

- **Context**：Step 2 已修复 Discovery probe 绕过 safe-fetch 的 SSRF 缺口；出站通道曾分裂。
- **Decision**：safe-fetch 是唯一“用户可控主机名”出站通道（DNS 校验+IP 钉扎+逐跳 redirect 复验+端口白名单 80/443+timeout+body 上限+明确错误码）；固定可信 URL 通道（Telegram/AI/Radar）后续统一；禁止业务层自建第二套 Fetch。
- **Consequences**：攻击面收敛；新采集源必须走统一层。
- **Alternatives**：各模块自行 fetch（历史教训：probe SSRF）。

---

## ADR-006 Passive Security Only

- **Context**：产品定位是数据智能平台，不是攻击平台。
- **Decision**：安全检测只做 Passive / Safe / Non-destructive；禁止 Exploit、Brute-force、端口扫描、入侵、持久化；Finding/Remediation/Verification 只基于可观察数据。
- **Consequences**：法律与伦理风险低；检测深度受限（无主动验证漏洞利用）。
- **Alternatives**：主动扫描（定位漂移 + 合规风险）。

---

## ADR-007 AI Explains / Data Proves

- **Context**：AI 幻觉会污染情报可信度。
- **Decision**：AI 不产生基础事实；只做解释/摘要/策略建议，输入限定为结构化证据；基础事实必须来自数据；UI 在无配置时不展示 AI 能力（现状已遵守）。
- **Consequences**：结论可信；AI 功能保守。
- **Alternatives**：AI 直接生成结论（不可追溯，违反 Evidence First）。

---

## ADR-008 No Premature Microservices

- **Context**：单机单进程远未饱和；调度器/API/Worker 均在 Next.js 进程内。
- **Decision**：保持模块化单体 + 单 worker；后台任务先统一为进程内 Job 消费器；仅在多实例/资源隔离成为明确瓶颈时拆服务。
- **Consequences**：部署简单（deploy.sh + systemd）；扩展需先做 Job 与 Redis 准备。
- **Alternatives**：立即拆 worker 服务（运维与部署复杂度上升，收益为零）。

---

## ADR-009 No Premature GraphDB

- **Context**：Explore/反查依赖关系查询；Neo4j 常被当作“先进”选择。
- **Decision**：PostgreSQL relationship model 优先；GraphDB 仅当关系规模（>500 万）或查询模式证明 PG 不足时再评估。
- **Consequences**：单库一致、备份简单；图查询在超大关系下可能需迁移。
- **Alternatives**：立即引入 Neo4j（双写/一致性/运维成本，当前规模无收益）。

---

## 决策记录维护规则

- 新 ADR 必须：说明 Context/Decision/Consequences/Alternatives，标注日期与状态（Proposed/Accepted/Superseded）。
- 与现状冲突的决策必须显式标注 CONFLICT 并给出收敛计划。
