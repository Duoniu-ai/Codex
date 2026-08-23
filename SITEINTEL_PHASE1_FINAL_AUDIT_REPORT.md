# SiteIntel Phase 1 全量验收 + 2.0 Master Architecture 对齐审计报告

> 审计性质：**严格只读**（未修改代码 / 数据库 / Schema / 配置 / 生产服务器 / 部署 / Git 历史）
> 日期：2026-08-23 ｜ 执行工具：Codex（唯一执行工具）
> 依据：Git 只读查询、GitHub API、生产 HTTPS 冒烟、Phase 1 Step 1-8 完成报告、2.0 架构冻结文档（MASTER / DATA / PRODUCT / ADR / REVIEW）、Step 8 生产实测记录

---

## 1. 执行摘要

Phase 1（Step 1-8）**全部验收通过**，代码、Schema、Git tag、测试均与实际部署一致，未发现方案-实现漂移；发现 3 项需人工处理的 P1 级问题与若干 P2 记录项。

关键结论：

1. siteintel 远程 main = `9eec3de`（Step 8 生产 tag 对应 commit），Phase 1 全部 8 个闭环均已进入 main 并打 tag。
2. **本地 master 漂移**：`C:\Users\deepo\siteintel` 本地 master 为 `ad8ed30`，比 main **领先 5 个未推送 commit**（2026-08-19 Technology Intelligence Phase 2.1-2.5，a08d81e→ad8ed30）。这些提交未出现在任何远程分支，需人工决策是否推送（不在本轮授权内）。
3. **生产 SSH 主机密钥变更**（2026-08-23 发现）：本地 known_hosts 记录的 ed25519 指纹 `SHA256:n+VJW...` 与服务器新提供 `SHA256:gKGZla...` 不一致，本轮按安全纪律**未接受**新密钥，SSH 级生产复核被阻断；生产 HTTPS 冒烟通过（首页/站点地图/API 文档均 200）。可能原因：服务器重建 / SSH key 轮换 / 中间人，需人工确认。
4. 数据底座健康：无 orphan（Fact/Snapshot/Raw 全 0）；Job/JobRun 无爆炸；Candidate pending 1032；Raw/Snapshot 无保留策略（P1）。
5. 2.0 Master Architecture 对齐：数据底座 12 层中 7 层 EXISTING、4 层 PLANNED（Finding/Recommendation/Action/Verification）、Event≈Signal 已收敛、Insight hypothesis 字段语义越界（CONFLICT 保留项）。

---

## 2. Phase 1 Step 1-8 逐项验收

### 2.1 Step 1 工程基线统一

| 检查项 | 结果 |
|---|---|
| 方案 | ✅ `SITEINTEL_PHASE1_P0_SECURITY_PLAN.md`（P0-2 节） |
| 实际代码 | ✅ main 已含 multi-source `06f2486` 内容 + `deploy.sh` |
| commit / tag | `f89ad89` / `production-phase1-baseline-2026-08-18` |
| Schema | ✅ 无变化 |
| 生产部署 | ✅ 已发布（Step 1 完成报告记录） |
| 测试 | ✅ 完整测试通过 |
| 已知限制 | 本地 master 本轮审计发现 5 个后续未推送 commit（见 §3.2） |
| 漂移 | ❌ 无 |

### 2.2 Step 2 Safe Fetch / SSRF

| 检查项 | 结果 |
|---|---|
| 方案 | ✅ `SITEINTEL_PHASE1_P0_SECURITY_PLAN.md`（P0-1 节） |
| 实际代码 | ✅ probe 统一走 `safeFetch`（resolveSafeAddresses + IP 钉扎 + allowedPorts 80/443 + timeout + body 上限 + UNSAFE_REDIRECT/BLOCKED_PORT 分类） |
| commit / tag | `de8b6da` / `production-phase1-step2-safefetch-2026-08-18` |
| Schema | ✅ 无变化 |
| 生产部署 | ✅ 已发布 |
| 测试 | ✅ DNS rebinding mock、redirect bypass、timeout、oversize、blocked reason 全覆盖 |
| 已知限制 | 固定可信 URL（Telegram/AI/Radar）未走 safeFetch（P2，记录在案） |
| 漂移 | ❌ 无 |

### 2.3 Step 3 /api/report Rate Limit

| 检查项 | 结果 |
|---|---|
| 方案 | ✅ `SITEINTEL_PHASE1_P0_SECURITY_PLAN.md`（P0-3 节） |
| 实际代码 | ✅ 匿名 30 req/hr 滑动窗口 + 429 + Retry-After；nginx `set_real_ip_from` 22 条 Cloudflare 段 + `real_ip_header CF-Connecting-IP` |
| commit / tag | `27bbf93` / `production-phase1-step3-report-ratelimit-2026-08-18` |
| Schema | ✅ 无变化 |
| 生产部署 | ✅ 已发布；生产实证直连伪造 X-Forwarded-For 无效 |
| 测试 | ✅ 200/429/Retry-After/多 IP 独立/伪造头/其他 API 不受影响 |
| 已知限制 | 源站 80 端口仍可直连（绕过 Cloudflare 头信任链），P1 记录 |
| 漂移 | ❌ 无 |

### 2.4 Step 4 SSE Access Control

| 检查项 | 结果 |
|---|---|
| 方案 | ✅ `SITEINTEL_PHASE1_P0_SECURITY_PLAN.md`（P0-4 节） |
| 实际代码 | ✅ `/api/analyze/[id]` SSE：登录用户 session→userId 归属校验；匿名一次性高强度 token→`streamTokenHash`(sha256) 校验；nginx 对 `/api/analyze/` `access_log off`（防 query token 落日志） |
| commit / tag | `c4d79e1` / `production-phase1-step4-sse-auth-2026-08-18` |
| Schema | ✅ 仅 Investigation 增 `streamTokenHash`（migration `20260818_stream_token_hash` 已部署） |
| 生产部署 | ✅ 已发布 |
| 测试 | ✅ 合法登录 200 / 合法 token 正常 SSE / 错误 token 401 / 缺 token 拒绝 / 跨调查 401 / 跨用户 403 / token 不明文落库 / 不落日志 |
| 已知限制 | 旧匿名调查（`streamTokenHash IS NULL`）读取返回 401，需重新分析（P2 记录） |
| 漂移 | ❌ 无 |

### 2.5 Step 5 Fact Validation

| 检查项 | 结果 |
|---|---|
| 方案 | ✅ `SITEINTEL_PHASE1_STEP5_FACT_VALIDATION_PLAN.md` |
| 实际代码 | ✅ 9 类 factType zod schema + `schemaVersion` 列（0=legacy，1=canonical v1）；写路径在 Evidence.create 前校验，失败跳过；读路径 report 跳过无效 v1、legacy 宽容 |
| commit / tag | `61e75bf` / `production-phase1-step5-fact-validation-2026-08-18` |
| Schema | ✅ Fact 增 `schemaVersion`（已部署） |
| 生产部署 | ✅ 已发布 |
| 测试 | ✅ 每类 schema 正/反例 + 读路径门 + 写路径门 |
| 已知限制 | legacy 188 条无效 Fact（140 metadata + 48 infrastructure）保留未迁移；sitemap/insights 未单独接入读取 gate（各自防御性解析），P2 记录 |
| 漂移 | ❌ 无 |

### 2.6 Step 6 Event / Signal 收敛

| 检查项 | 结果 |
|---|---|
| 方案 | ✅ `SITEINTEL_PHASE1_STEP6_EVENT_SIGNAL_CONVERGENCE_PLAN.md`（方案 A） |
| 实际代码 | ✅ `syncOneFact` diff 前置：空 diff → 不写 fact_changed Signal、不写 Event；非空 diff → 1 Signal → 1..N Event；fact_created/fact_contradicted 保持 Signal-only |
| commit / tag | `559134c` / `production-phase1-step6-event-signal-2026-08-18` |
| Schema | ✅ 无变化 |
| 生产部署 | ✅ 已发布；生产实证：同值重分析 0 Signal / 0 Event；真变化（ntp/icloud）→ 1 fact_changed ↔ 1 ssl_changed |
| 测试 | ✅ 空 diff 无 Signal/无 Event、真变化单 Signal+正确 Event、不重复、不同时间可分别产生、Report/Timeline/Discovery 回归 |
| 已知限制 | 历史 125 个孤儿 Signal 保留（零迁移，Plan 已声明） |
| 漂移 | ❌ 无 |

### 2.7 Step 7 Minimal Job System

| 检查项 | 结果 |
|---|---|
| 方案 | ✅ `SITEINTEL_PHASE1_STEP7_JOB_SYSTEM_PLAN.md` |
| 实际代码 | ✅ Job(4：monitor/search_sync/discovery/data_quality_scan) + JobRun + 原子 claim（INSERT…ON CONFLICT DO NOTHING）+ heartbeat + boot crash recovery（running→failed）+ `pruneJobRuns(30d)` helper |
| commit / tag | `75bd5ef` / `production-phase1-step7-job-system-2026-08-18` |
| Schema | ✅ Job/JobRun 两表 + `JobRun_one_running_per_job` partial unique（已部署） |
| 生产部署 | ✅ 已发布 |
| 测试 | ✅ claim 并发唯一、heartbeat、crash recovery、retry、data_quality_scan 只读 |
| 已知限制 | 单实例恢复语义；`pruneJobRuns` 未建自动 CLEANUP Job；seed 随 discovery 会话、无独立 Job；无 JobRun 观测页面（P2） |
| 漂移 | ❌ 无 |

### 2.8 Step 8 Query & Read Performance

| 检查项 | 结果 |
|---|---|
| 方案 | ✅ `SITEINTEL_PHASE1_STEP8_QUERY_PERFORMANCE_PLAN.md` |
| 实际代码 | ✅ `read-cache.ts`（ttlMemo 有界 1000 条 + React cache）+ website 页装配去重 + 首页 60s + sitemap 5min + api/report 30s（限流后） |
| commit / tag | `9eec3de` / `production-phase1-step8-query-performance-2026-08-18` |
| Schema | ✅ 无变化 |
| 生产部署 | ✅ 已发布（BUILD_ID `4iMBt9TwwpKSNBxk1fvGh`）；实测 sitemap 63→5.5ms、api/report 27→6ms、报告页 77→42ms |
| 测试 | ✅ 352→354 用例；typecheck 0；build 通过 |
| 已知限制 | 实体页并行化未做（warm 46-81ms 无 N+1，低收益，记录）；P2 门禁索引/JSONB 下推/响应裁剪未做；慢查询仪表未启用 |
| 漂移 | ❌ 无 |

---

## 3. 生产基线

### 3.1 基线核对

| 项目 | 值 | 状态 |
|---|---|---|
| siteintel/main（GitHub） | `9eec3de4c00e8c2520af2739b24d3832093ad76b` | ✅ 与 Step 8 tag 一致 |
| local master（`C:\Users\deepo\siteintel`） | `ad8ed30` | ⚠️ **领先 main 5 个未推送 commit** |
| production HEAD | `9eec3de`（Step 8 部署记录） | ⚠️ SSH 复核被主机密钥变更阻断 |
| production BUILD_ID | `4iMBt9TwwpKSNBxk1fvGh`（Step 8 记录） | 同上 |
| migration 状态 | 10 个 migration 全部 applied（Step 7/8 部署记录） | 同上 |
| 服务状态 | systemd `siteintel` active（Step 8 记录），HTTPS 冒烟 2026-08-23 通过 | ✅ |
| NRestarts / fatal | 0（Step 8 记录） | ⚠️ 本轮未 SSH 复核 |
| 数据库连接 | PostgreSQL 14 / 5432，正常（Step 8 记录） | ⚠️ 本轮未 SSH 复核 |

### 3.2 本地 master 漂移（P1 审计发现）

`C:\Users\deepo\siteintel` 本地 master 含 5 个**未推送、未部署**的 commit（Technology Intelligence Phase 2.1-2.5，2026-08-19）：

| commit | 内容 |
|---|---|
| `a08d81e` | feat(technology): establish technology catalog and taxonomy |
| `99466ee` | feat(technology): add technology profile backend |
| `4f9122b` | feat(technology): add technology profile frontend |
| `2755bd3` | feat(technology): add technology historical intelligence |
| `ad8ed30` | feat(technology): add technology distribution intelligence |

`git ls-remote origin` 确认这些 commit **未出现在任何远程 ref**。影响：Phase 2.1-2.5 的本地代码无法从 GitHub main 重建；不属于本轮授权范围，**未做任何处理**，建议人工决策（推送或归档）。

### 3.3 生产 HTTP 冒烟（2026-08-23，只读）

| 端点 | HTTP | TTFB | 说明 |
|---|---|---|---|
| `https://siteintel.cc/` | 200 | 1.29s | 首页（冷缓存，Body 105KB） |
| `https://siteintel.cc/sitemap.xml` | 200 | 0.23s | 站点地图（缓存生效） |
| `https://siteintel.cc/docs/api` | 200 | 0.23s | API 文档 |

---

## 4. 2.0 Master Architecture 对齐（摘要）

完整状态矩阵见 `SITEINTEL_2.0_IMPLEMENTATION_STATUS_MATRIX.md`。要点：

| 维度 | 状态 | 证据 |
|---|---|---|
| 核心闭环 Discover→Observe→Explain | EXISTING | 多源 Discovery + Pipeline + Fact/Entity/Relation + Event/Insight |
| Recommend（计算式不落库） | PARTIAL | audit 22 项 → P0/P1/P2 推荐 |
| Act / Verify | PLANNED | Action Center 未建 |
| Rediscover（价值回填） | PARTIAL | Candidate 有 score/probe/退避，无 analyzedValue 回填 |
| 十二层数据底座（Raw→Verification） | 7 EXISTING / 4 PLANNED / 1 PARTIAL | Finding/Recommendation/Action/Verification 未建；Insight PARTIAL |
| 安全架构 | EXISTING（收口） | SafeFetch 唯一出站 + 限流 + SSE 鉴权 |
| Job 架构 | PARTIAL→EXISTING（最小版） | Job/JobRun 已建；三调度器已登记 |
| Search / GraphDB / Microservices | 门禁未触发 | PG 单体保持 |
| ADR-001..009 | 全部成立 | 无违反 |

**严禁把 PLANNED 写成已实现**：Finding、Keyword、Ranking、Recommendation、Action、Verification、Page 模型、SourceRegistry、全局 Explore、Redis、搜索集群均处于 PLANNED/RECOMMENDED。

---

## 5. 数据架构审计

| 层 | 现状 | 用途 | 写入路径 | 读取路径 | 与 Master 一致 | 风险 |
|---|---|---|---|---|---|---|
| Raw | EXISTING（4760） | 证据根 | collectAll providers | Evidence 溯源 | ✅ | 无保留策略（P1） |
| Fact | EXISTING（3237：schema0=3202 / schema1=35） | 报告唯一读源 | pipeline→facts（zod 门） | report | ✅ | legacy 188 无效未迁移（P2） |
| Entity | EXISTING（4016，subdomain 1390=34.6%） | 图谱节点 | entity upsert | 实体页/图谱 | ✅ | SAN 子域噪声（P1，Phase 2 治理） |
| Relationship | EXISTING（8633） | 关系资产 | relationship upsert | 实体页/图谱 | ✅ | 无 status 字段（PLANNED） |
| Observation/Snapshot | EXISTING（4227） | 时间点观测 | snapshot hash 去重 | 历史/diff | ✅（Snapshot=Observation 存储） | 无保留策略（P1） |
| Event | EXISTING（80） | **唯一对外时间线** | syncOneFact diff 门 | Timeline/Report | ✅（Step 6 收敛） | — |
| Signal | EXISTING（3481） | 内部迁移日志 | syncOneFact | 仅内部 | ✅ 已内部化 | 历史孤儿保留（P2） |
| Finding | PLANNED | 状态化问题 | — | — | 未实现 | — |
| Insight | EXISTING（PARTIAL） | 规则解释 | 规则引擎 | Report | ⚠️ hypothesis 字段语义越界（CONFLICT 保留） | — |
| Recommendation | PARTIAL | 建议 | audit 计算不落库 | Report | ⚠️ 无持久化 | — |
| Action / Verification | PLANNED | 跟踪任务 | — | — | 未实现 | — |

Fact 校验：`schemaVersion=1` 35 条走 canonical 校验；`schemaVersion=0` 3202 条 legacy 宽容读取；生产旧 188 条无效 Fact 仍安全（读取端已隔离，未迁移）。

---

## 6. 安全审计（只读复核）

| 边界 | 状态 | 复核结果 |
|---|---|---|
| Safe Fetch 唯一出站 | ✅ | probe 已统一；固定可信 URL 通道 P2 记录 |
| SSRF / DNS rebinding / redirect | ✅ | 已测（Step 2）；IP 钉扎 + 逐跳复验 |
| allowed ports | ✅ | 仅 80/443 |
| timeout / body limit | ✅ | 已实现并测试 |
| /api/report rate limit | ✅ | 30/hr + 429 + Retry-After；CF 信任链 |
| IP trust chain | ✅ | nginx 22 条 CF 段 + CF-Connecting-IP；直连伪造头无效 |
| SSE token / session | ✅ | hash 落库、一次性、nginx 关闭 /api/analyze/ access log |
| 跨调查/跨用户 | ✅ | 401/403 已测 |
| 源站 80 直连 | ⚠️ P1 | 生产 80 端口仍可直连（绕过 CF），建议防火墙收口（本轮未改） |
| SSH 主机密钥变更 | ⚠️ P1 | 见 §3.1；需人工确认服务器身份 |

本轮未发现新的 P0。P1 两项：源站 80 直连、SSH 主机密钥变更待人工确认。

---

## 7. Job System 审计

| 检查项 | 结果 |
|---|---|
| Job 定义 | ✅ 4 个（monitor / search_sync / discovery / data_quality_scan） |
| JobRun 状态 | 10 行：running=1 / failed=2 / success=7 |
| 单 Job 单 running | ✅ `running_per_job_max=1`（partial unique 生效） |
| 重复运行 | ✅ 无 |
| stale running | ✅ crash recovery 在 boot 时回收 |
| 失败任务 | 2 个 failed（错误摘要可查，无敏感 payload） |
| JobRun 爆炸增长 | ✅ 10 行，无爆炸 |
| 未覆盖任务 | ⚠️ seed 随 discovery 会话；无独立 CLEANUP Job（P2） |
| 隐藏 setInterval | ✅ 三个调度器均已登记 JobRun（Step 7 完成报告） |

---

## 8. 性能审计（Step 8 后）

| 路径 | 查询形态 | 优化后 | 状态 |
|---|---|---|---|
| 首页 | 聚合装配 | React cache + 60s TTL | ✅ |
| website 报告页 | 装配去重 | 去重 + ttlMemo | ✅ 77→42ms |
| /api/report | 限流后缓存 | 30s TTL | ✅ 27→6ms |
| sitemap | 全量查询 | 5min TTL | ✅ 63→5.5ms |
| 实体页 | 顺序查询（无 N+1） | warm 46-81ms，未并行化（低收益） | ✅ 记录 |
| 慢查询 | — | 未启用仪表 | P2 |

判断：当前性能问题为 **P2**（1 万 Target 以内无压力）；1-2 万 Target 是下一个触发点（报告查询形态已先行治理）。

---

## 9. 数据库健康审计（只读）

| 指标 | 值 | 判断 |
|---|---|---|
| Target / Investigation | 407 / 682 | 正常 |
| Fact（schema1） | 35 | 新写入已校验 |
| 孤儿（Fact/Snapshot/Raw） | 0 / 0 / 0 | ✅ |
| Candidate pending / analyzed | 1032 / 163 | ⚠️ 积压（P1，Phase 2 治理） |
| Candidate probe：ok/fail | 874 / 158 | 正常退避 |
| SAN 子域占比（Entity） | 1390/4016=34.6% | ⚠️ 此前“94%”指 Candidate source=san 占比，勿混淆 |
| Raw / Snapshot 保留策略 | 无 | P1（90 天滚动 / aspect 最近 N 条，PLANNED） |
| JobRun 增长 | 10 行 | ✅ 无爆炸 |
| 重复 / 脏数据 | 未见明显 | ✅ |

**禁止自动清理**：以上均为记录项。

---

## 10. 产品一致性审计

当前 SiteIntel 已不是纯“WHOIS+DNS+IP+Technology+SEO 工具集合”，而是**Website Intelligence Platform 底座**：

- 完整数据链 Website→Collection→Evidence→Fact→Entity→Relationship→Snapshot→Event→Insight（EXISTING）。
- 反查/图谱（IP/ASN/Certificate/Organization/Technology）EXISTING。
- 缺失：Finding→Recommendation→Action→Verification 闭环（PLANNED）、Page 级模型、全局 Explore。
- 结论：产品底座与 2.0 定位一致，处于“数据平台 + 局部 Intelligence”阶段。

---

## 11. Phase 2 入口判断

唯一推荐顺序与逐方向比较见 `SITEINTEL_PHASE2_DIRECTION_DECISION.md`。摘要：

1. **D. Discovery / Data Flywheel**（Phase 2 首个闭环）：候选价值回填 + SAN 降噪 + admin 队列页。数据基础最成熟（1032 pending / score/probe/退避已有），符合“质量>数量”与数据复利。
2. **F. 基础设施收尾**（Phase 2.5 候选）：SourceRegistry + Fact 读取 gate 统一 + 实体页并行化 + 保留策略。
3. **A. Security Intelligence**（Phase 4 前不建表）：被动安全数据可先映射到现有 Fact/Entity，依赖 Page 与依赖数据。
4. **B. SEO Intelligence**（Phase 5）：依赖页面模型与观测；搜索引擎能力受限（仅 Google 强、Bing 待验证、百度仅链接提交、神马/搜狗无 API）。
5. **C. Search / Ranking**（Phase 6）：依赖 SEO 观测。
6. **E. Intelligence / Recommendation**（Phase 7）：依赖 Security/SEO 数据成熟。

---

## 12. 发现的问题汇总

| 级别 | 问题 | 状态 |
|---|---|---|
| P1 | SSH 主机密钥变更，生产 SSH 复核被阻断 | 待人工确认（本轮未接受新密钥） |
| P1 | 本地 master 5 个 Technology Intelligence commit 未推送 | 待人工决策（不在授权内） |
| P1 | 源站 80 端口可直连（绕过 CF 信任链） | 记录，待授权收口 |
| P1 | Raw/Snapshot 无保留策略 | 记录（PLANNED） |
| P1 | Candidate pending 1032 积压 + SAN 噪声 | Phase 2 治理 |
| P2 | legacy 188 无效 Fact 未迁移；sitemap/insights 读取 gate | 记录 |
| P2 | 旧匿名调查（streamTokenHash NULL）401；JobRun 无观测页 | 记录 |
| P2 | 固定可信 URL 未走 safeFetch；Insight hypothesis 语义越界 | 记录 |

---

## 13. 未解决的问题

1. SSH 主机密钥变更的根因（服务器重建？key 轮换？）未确认。
2. 本地 master 5 个未推送 commit 的去向未定。
3. 源站 80 直连收口方案未实施（需授权）。
4. Phase 2 首个闭环的实施授权未下发。

---

## GitHub Publication

- Repository: Duoniu-ai/Codex
- Branch: main
- Commit: 8c8d4c78f904d163644a1abb9b9bef5166db24c0
- Tag: 无
- File: SITEINTEL_PHASE1_FINAL_AUDIT_REPORT.md
- Remote verification: PASS
