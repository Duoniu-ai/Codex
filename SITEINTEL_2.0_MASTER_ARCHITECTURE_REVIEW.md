# SiteIntel 2.0 Master Architecture Review（冻结评审总结）

> 日期：2026-08-18 ｜ 只读 ｜ 配套：MASTER_ARCHITECTURE / DATA_ARCHITECTURE / PRODUCT_ARCHITECTURE / ADRs

---

## 1. 当前架构

Next.js 16 App Router 单体 + Prisma 6 + PostgreSQL 14（单机 systemd :3003 + nginx + Cloudflare）。数据流 Raw→Fact→Entity→Relationship→Snapshot→Event→Insight 完整（EXISTING）；安全网络层统一（Step 2）；/api/report 限流（Step 3）；生产 = siteintel main 27bbf93（deploy.sh 可重建）。三大短板：无 Job 系统（进程内 setInterval）、Event/Signal 双轨、报告查询 N+1 + 无缓存。

## 2. 目标架构

PostgreSQL 单体 + 统一证据链（Raw→…→Verification）+ 六大领域共享模型 + 最小 Job/JobRun 消费器 + Source Registry + 类型化 Fact + 关系资产化 + 可解释健康度 + 统一 Action 链。不引入 GraphDB/微服务/搜索集群。

## 3. 主要差距

| 差距 | 级别 |
|---|---|
| Job System 缺失（三调度器 in-process） | P0 |
| Event/Signal 双轨 + Insight hypothesis 语义越界 | P0（数据一致性） |
| Fact.value 无校验/schemaVersion；Evidence rawValue 占位 | P1 |
| 报告/实体页 N+1 + 零缓存 | P1（性能） |
| Discovery 无价值回填 + SAN 94% 自我繁殖 | P1 |
| SSE 无鉴权（Step 4 待授权） | P1 |
| Security/SEO 业务模型（Finding/Keyword）未建 | P2（按阶段） |
| Source Registry 缺失（新源接入要改 5-7 个核心文件） | P1 |

## 4. 最大风险

1. 规模崩溃顺序：报告查询形态 + 单进程调度器先崩（1-2 万 Target 即明显），其次候选全表扫描与存储增长。
2. 数据语义漂移：JSONB 无约束 + 双轨记录持续累积技术债。
3. Codex 仓库 public：运维信息公开（服务器 IP/路径/端口），建议转 private。

## 5. 最大数据资产机会

**实体关系图谱**（3k+ 实体 / 5.9k+ 关系）——共享 IP/证书/技术/基础设施的反向关联是 Explore 与安全/竞争情报的原材料；下一步把 Discovery 候选价值回填后，图谱会成为自我增强资产。

## 6. 最大产品机会

**从“报告”到“下一步做什么”**：把现有 Audit/Recommendation 升级为统一 Action Center（安全修复 + SEO 机会 + 技术优化），并接入验证回环——这是其他查询工具没有的闭环。

## 7. 当前最值得做的 5 件事

1. **SSE 鉴权（Phase 1 Step 4）**——唯一剩余 P0/P1 边界。
2. **数据一致性收口**：Event/Signal 收敛 + Fact zod/schemaVersion + Evidence 指针。
3. **最小 Job System（Job+JobRun）**：三调度器登记 + DATA_QUALITY 巡检。
4. **报告/实体页查询并行化 + 短缓存**：解决第一个规模瓶颈。
5. **Discovery 价值回填 + SAN 降噪 + admin 队列页**：把“量”变成“资产”。

## 8. 当前绝对不要做的 10 件事

1. GraphDB 2. 微服务/K8s 3. 搜索集群 4. 复杂 AI Agent 5. 漏洞扫描/攻击平台 6. Discovery 扩量 7. SecurityFinding/Keyword/Recommendation 提前建表 8. 大规模重写 9. 商业化/付费 10. Codex 仓库继续混入其他项目文件。

## 9. 第一阶段实施建议

Phase 1 剩余闭环（按授权逐项）：Step 4 SSE 鉴权 → Fact 校验/schemaVersion → Event/Signal 收敛 → Job/JobRun 最小版 + 三调度器登记 → Source Registry 轻量表 → 报告查询并行化/短缓存 → Discovery 价值回填列。每项：分析→授权→实施→测试→staging→生产→报告→推送 Codex。

## 10. 下一阶段具体授权建议

建议按以下顺序逐项授权：

1. Phase 1 Step 4（SSE 鉴权，P1）。
2. Phase 1 Step 5（Fact 数据校验 + schemaVersion + Evidence 指针加固）。
3. Phase 1 Step 6（Event/Signal 统一 + Insight 语义收敛）。
4. Phase 1 Step 7（最小 Job System + DATA_QUALITY 巡检 Job）。
5. Phase 1 Step 8（报告/实体页查询优化 + 短缓存）。
6. Phase 2 Step 1（Discovery 价值回填 + SAN 降噪 + admin 队列页）。

每项完成后按既有格式（完成报告 + GitHub Publication）交付并停止等待验收。

---

*本评审为只读架构冻结产物；未修改任何代码/数据库/生产/服务器/Git 工作树。*
