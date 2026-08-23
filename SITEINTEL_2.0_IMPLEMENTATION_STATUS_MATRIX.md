# SiteIntel 2.0 Implementation Status Matrix

> 审计配套文档 ｜ 日期：2026-08-23 ｜ 只读 ｜ 状态标记：EXISTING / PARTIAL / PLANNED / RECOMMENDED / CONFLICT / DEPRECATED
> 依据：真实代码（siteintel main `9eec3de`）+ Schema + 生产数据 + 2.0 架构冻结文档

---

## 1. 核心闭环

| 环节 | 状态 | 说明 |
|---|---|---|
| Discover | EXISTING | 多源（Cloudflare Radar + seed）DiscoveryCandidate 池 |
| Collect | EXISTING | 7 providers（dns/rdap/ip/ssl/http/metadata/technology） |
| Normalize | EXISTING | Pipeline → Fact（zod 校验 + schemaVersion） |
| Understand | EXISTING | Entity / Relationship 提取 |
| Relate | EXISTING | 9 类 EntityRelationship（8633 行） |
| Observe | EXISTING | Snapshot（4227，aspect 级 hash 去重） |
| Detect | EXISTING | diff（Step 6 收敛）+ 12 类 Event |
| Explain | EXISTING | Insight 规则引擎（13 条规则 + contradiction） |
| Recommend | PARTIAL | 22 项 audit → 计算式推荐，不落库 |
| Act | PLANNED | Action Center 未建 |
| Verify | PLANNED | 修复验证未建 |
| Monitor | EXISTING | daily/weekly/monthly 重扫 + Telegram/Webhook |
| Rediscover | PARTIAL | 无 analyzedValue 价值回填 |

---

## 2. 十二层数据底座

| 层 | 状态 | 生产证据 | 说明 |
|---|---|---|---|
| Raw | EXISTING | RawCollection 4760 | 不可变证据根；无保留策略（PLANNED） |
| Fact | EXISTING | 3237（schema0=3202 / schema1=35） | 报告唯一读源；schemaVersion + zod 门 |
| Entity | EXISTING | 4016（subdomain 1390） | type+normalized 唯一；SAN 噪声待治理 |
| Relationship | EXISTING | 8633 | observed/probable/inferred；status 字段 PLANNED |
| Observation/Snapshot | EXISTING | Snapshot 4227 | Snapshot = Observation 存储实现（无重复表） |
| Event | EXISTING | 80 | **唯一对外时间线**（Step 6 收敛） |
| Finding | PLANNED | — | SecurityFinding/SEOFinding 未建（Phase 4/5） |
| Insight | EXISTING（PARTIAL） | 规则引擎 + hypothesis 字段 | hypothesis 语义越界（CONFLICT，保留收敛） |
| Recommendation | PARTIAL | 计算式不落库 | 独立表 PLANNED（Phase 7） |
| Action | PLANNED | — | 未建 |
| Verification | PLANNED | — | 未建 |
| Signal（内部） | EXISTING | 3481 | 已内部化；历史孤儿 125 保留 |

---

## 3. 六大产品领域

| 领域 | 状态 | 关键现状 |
|---|---|---|
| Website Intelligence | EXISTING（核心） | Target/Fact/Entity/Relation/Snapshot/Event/Insight + 报告 + 实体反查页；Page 模型 PLANNED |
| Technology Intelligence | PARTIAL | 指纹库 57+、分类、部分版本、反查页；依赖清单/TechnologyVersion PLANNED |
| Security Intelligence | PARTIAL（数据层） | TLS/安全头存在性/技术版本（被动）；无 Finding/Vulnerability/Remediation |
| SEO Intelligence | PARTIAL（Technical） | title/desc/H1/robots/sitemap/JSON-LD/内链 + 8 项 audit；无 Keyword/Ranking/Opportunity |
| Change Intelligence | EXISTING | 12 类事件 + 13 条洞察 + 信号相关性 + 矛盾检测 + 统一时间线 |
| Growth Intelligence | PARTIAL | 22 项 audit → P0/P1/P2 推荐 + SWOT-lite + AI 解释；无持久化生命周期 |

---

## 4. 架构决策 / 基础设施

| 项 | 状态 | 说明 |
|---|---|---|
| ADR-001 Repo Separation | EXISTING（生效） | siteintel=代码；Codex=文档 |
| ADR-002 PostgreSQL First | EXISTING（生效） | PG 14 单体；JSONB 仅载荷 |
| ADR-003 Relationship First | EXISTING | 8633 关系；统一表 |
| ADR-004 Evidence First | EXISTING | Fact→Evidence 链 + zod；Evidence source_url 补全 PLANNED |
| ADR-005 Safe Fetch Boundary | EXISTING | 唯一用户可控主机名出站；固定 URL 通道 P2 记录 |
| ADR-006 Passive Security Only | EXISTING | 禁止主动扫描 |
| ADR-007 AI Explains / Data Proves | EXISTING | 严格契约 |
| ADR-008 No Premature Microservices | EXISTING | 单进程 + 最小 Job 消费器 |
| ADR-009 No Premature GraphDB | EXISTING | PG 关系模型优先 |
| Job System | EXISTING（最小版） | Job(4) + JobRun + partial unique + crash recovery；JobAttempt/JobLog PLANNED |
| SourceRegistry | PLANNED | 新源接入仍要改 5-7 个核心文件 |
| 缓存 | EXISTING（短 TTL） | read-cache + 首页 60s + sitemap 5min + api/report 30s；Redis 门禁未触发 |
| Search | PLANNED | PG FTS 路线；搜索集群门禁（Target>20 万）未触发 |
| Explore / Relationship Graph | PLANNED | 实体页互跳 EXISTING；全局 Explore 未建 |
| API 产品 | PARTIAL | v1 analyze/report EXISTING；实体/关系/历史等 PLANNED |
| Rate Limit | EXISTING | analyze 20/hr、tools 20/hr、report 30/hr、login 10/15min（内存滑动窗口） |
| SSE Auth | EXISTING | streamTokenHash + session 归属 + nginx 日志关闭 |
| Data Freshness | PARTIAL | 重扫 TTL 策略部分落地；Change-Driven Refresh PLANNED |
| Data Quality | PARTIAL | 矛盾检测 + admin/data-quality 页 + data_quality_scan Job（只读）；Data Quality Score PLANNED |
| 保留策略 | PLANNED | Raw 90 天滚动 / Snapshot aspect 最近 N 条（未实施） |
| 分区 | PLANNED（门禁） | Target/Entity 50 万或表 5GB 时评估 |

---

## 5. 阶段路线状态

| 阶段 | 状态 |
|---|---|
| Phase 0 生产稳定 | ✅ 完成（probe/限流/SSE 收口） |
| Phase 1 Data Foundation | ✅ Step 1-8 完成（SourceRegistry/保留策略为 Step 9 候选） |
| Phase 2 Discovery Intelligence | 待授权（方向已定：Data Flywheel） |
| Phase 3 Website Intelligence（Page） | PLANNED |
| Phase 4 Security Intelligence | PLANNED（被动） |
| Phase 5 SEO Intelligence | PLANNED（Technical 先行） |
| Phase 6 Search/Ranking | PLANNED |
| Phase 7 Growth Intelligence | PLANNED |
| Phase 8 Monitoring/Explore | PLANNED |
| Phase 9 API/Platform | PLANNED |
| Phase 10 Commercialization | PLANNED |

---

## 6. 已知语义冲突（CONFLICT）与收敛状态

| 冲突项 | 状态 | 收敛状态 |
|---|---|---|
| Event ≈ Signal | CONFLICT（已收敛） | Step 6：Signal 内部化，Event 唯一时间线 ✅ |
| Insight hypothesis 语义越界 | CONFLICT（保留） | 仅搜索类规则使用；未来一等公民再拆表 |
| Fact vs Observation | 清晰（无重复表） | Snapshot=Observation 存储 |
| Investigation vs Job | 清晰 | JobRun 包裹 Investigation（规划，尚未包裹） |

---

## GitHub Publication

- Repository: Duoniu-ai/Codex
- Branch: main
- Commit: 8c8d4c78f904d163644a1abb9b9bef5166db24c0
- Tag: 无
- File: SITEINTEL_2.0_IMPLEMENTATION_STATUS_MATRIX.md
- Remote verification: PASS
