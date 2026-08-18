# SITEINTEL STATE — Claude Code → Codex 上下文深度恢复（SiteIntel.cc）

> 生成时间：2026-08-18（只读整理，未修改任何代码/数据库/部署/Git）
> 依据：`C:\Users\deepo\siteintel\` 下报告 + `.claude\projects\C--Users-deepo\memory\siteintel-project.md` + 本地 Git 只读 + GitHub API 只读复核（2026-08-18）。
> 状态标记：`COMPLETED` / `ANALYZED / NOT EXECUTED` / `DRY-RUN` / `WAITING FOR AUTHORIZATION` / `UNKNOWN`。

---

## 1. 最近已完成的生产修复（COMPLETED — 不得重复执行）

| 项 | 内容 | 验证 |
|---|---|---|
| P0-1 采集管道 SSRF | `lib/security/safe-fetch.ts` 统一安全网络层（解析→逐地址校验→IP 钉扎→逐跳重定向再校验），7 provider + ownership + baidu 接入 | 生产 e2e：回环域名被拦 `private address 127.0.0.1` |
| P0-2 provider_changed 假阳性 | NS 稳定排序 + MX 按 priority 取值；历史噪音清理已执行（64 条删 + 4 条洞察 resolved + 备份 + verify） | wordpress.org 重分析 0 新假阳性 |
| P1-1 www 规范化（代码） | domain 实体剥 www（subdomain 保留）、技术页按规范域去重 | sitevance.tw 重复 chip 2→1 |
| P1-1 历史 www 实体合并 | **已执行**（见 §4） | --verify 6/6，线上 PASS |
| P1-2 AS0 哨兵 | 实体创建守卫 + sitemap 排除 + 页面 404；历史僵尸行按决策保留 | `/asn/AS0` 线上 404 |
| P1-3 sitemap 首页重复 | 尾斜杠归一比较 | 135 URL 零异常 |
| P1-4 实体页 OG | ip/asn/certificate/organization/technology 五类实体化 OG+twitter | 线上 og:title/url 实体专属 |
| P1-5 搜索页 CTA | ctaHref 显式配置 → /search 中心 | 三引擎页 CTA 正确 |
| P2-1 组织别名机制 | OrganizationAlias 表 + migration + 解析层 + seed 19 行（授权执行）；**历史实体合并未执行** | 110 测试通过 |
| P2-2/3/4/5 | 技术页 locale / 429 提示 / 回环私网 IP 标注+noindex / 洞察展示差异化 | 线上实测通过 |
| safe-fetch path 回归 | 修复 `path` 漏传 + 数据纠错（15 关系删 + 1 事件删 + 2 洞察 resolve + 6 目标重分析） | verify 6/6，0 残留 |
| 回归审计 | SITEINTEL-POST-REMEDIATION-REGRESSION-AUDIT | 0 回归，121 测试 PASS |
| P3 收尾 | 3 个孤立 domain 删除（逐域备份）；downcc 保留 partial | verify 全过 |
| Domain 创建守卫 | 无存在性证据不建 domain 实体 | 126 测试 |
| Discovery 盘点 + Aging | 7 类候选源 + 72 分钟节拍 + aging 公式 | 138 测试 |
| 2.0 Phase 3A/3B/3B1/3B2/3B3/3C | 首页/旗舰报告页/视觉层级/变化证据链/阅读路径/导航统一上线 | 189 测试，术语 0 |
| monitor 死循环修复 | 无通知通道也推进 lastRunAt | 实测 07:30 零活动 |
| 多源 Phase 0 + CF Radar | 候选池扩展 + SeedSource + 15s 连续消费 + 10 因子评分 + 指数退避 + 预算 50/天 | 生产验收 PASS（251 测试） |
| Seed 灰度测试 | 预算 10→7 新增；2 个评分透明度缺陷修复（06f2486） | 258 测试全绿 |
| SEED_DAILY_BATCH 10→50 | 仅改服务器 .env + 一次重启，零代码修改 | 31/50 记账对账，dup=0 |
| SEO Phase 1 P0 全链路 | 实施→验收（PASS WITH MODIFICATIONS）→staging 验证 PASS→生产部署 PASS→Git FF merge + tag | 226 测试；线上 PASS |
| Git 历史统一 + HEAD 对齐 | legacy-prod-history 66 commits + 生产 master 对齐 c74dfd1 | 10/10 + 16/16 PASS |

---

## 2. Git commit / 生产版本 / 部署状态

### 远端（2026-08-18 只读 API 复核通过）
- 仓库：`Duoniu-ai/siteintel`（**PRIVATE**，default branch `main`，最近 push 2026-08-17T07:33Z）
- `main` = **`c74dfd1`**（= 生产基线 + SEO Phase 1 P0 14 文件 +831/−33）
- tags：`production-seo-phase1-p0-2026-08-17`→c74dfd1、`production-history-archive-20260817`→1bdfcb4、`production-baseline-2026-08-17`→b52ec84（全部与记录一致）
- 分支 `multi-source/phase0-cfradar`（e32bd93，已 push，**未 merge 进 main**）

### 生产（按 2026-08-17 记录；实时线上状态 UNKNOWN）
- 生产 git `master` HEAD = **c74dfd1**（08-17 对齐，16/16 PASS；回滚=update-ref 回 1bdfcb4 + read-tree）
- 生产运行代码 = c74dfd1 内容 + 多源 Phase 0（e32bd93）+ Seed 评分修复（06f2486）（多源文件在生产 git 的落位未实时核验）
- 服务：systemd `siteintel` active（:3003，0 fatal）；staging `siteintel-staging`（:3004）仍 active，**待删（WAITING FOR AUTHORIZATION）**
- 回滚资产在位：服务器 tags（production-legacy-1bdfcb4、pre-deploy-seo-p0-20260817、production-pre-head-alignment-20260817）+ bundle 备份 /www/backup/siteintel-prod-history-20260817.bundle

### 本地镜像 `C:\Users\deepo\siteintel`（只读确认）
- master @ **1bdfcb4**（生产 legacy 66 commits 末端），无本地分支/tag
- 工作树含预期未提交项（D baidu 大写、?? .well-known/、?? 12+ 报告 md、M DISCOVERY 等）——卫生/排除文件，**不自行清理**

---

## 3. 数据库结构 / 数据状态 / 最近数据处理

### 结构（记录口径）
- PostgreSQL 14，库 `siteintel`（连接 127.0.0.1:5432，凭据仅服务器 .env）；早期只读核查 19 表与 schema 一致
- 后续增量：OrganizationAlias（+kind/status/note，migration 20260816_organization_alias_phase_a）、OrganizationDenyLog、DiscoveryCandidate +18 列/3 索引 + Target.isApex（migration 20260817_multi_source_candidate_pool 已部署）

### 数据规模（两份快照，时间不同）
| 指标 | 08-16 快照 | 08-17 诊断 |
|---|---:|---:|
| Target | 279 | 289 |
| Investigation | 480 | 537 |
| Entity 总数 | 3196 | 3230 |
| Relationship | 5659 | 5888 |
| Snapshot | 3047 | 3147 |
| Event | 62 | 65 |
| Insight（active） | 55（49 active） | 58 |
| DiscoveryCandidate | 926（pending 875） | 926+（868 pending） |
| Fact | 2246 | — |
| RawCollection | 3346 | — |
| Organization 实体 | 160 | — |
| OrganizationAlias | 19（approved 18 / pending 1） | — |
| AS0 | 1 僵尸行（永久保留） | — |

### 数据质量（08-16 快照，九项全绿）
www 残留 0、孤立 Entity/Relationship 0、自环 0、重复 Event 0、无证据 Event 0、active Insight 证据链 0 断裂/0 跨引用、无存在性证据 Domain 0；Relationship 重复/自环/孤立 0。

### 最近数据处理结果
- provider_changed 历史噪音清理：64 删 + 4 洞察 resolved（备份 EventBackup/InsightBackup_20260816_provider_changed）→ **COMPLETED**
- www 合并：294→210 domain 实体（详见 §4）→ **COMPLETED**
- safe-fetch 污染纠错：15 关系 + 1 事件删除、2 洞察 resolve、6 目标重分析 → **COMPLETED**
- Seed：SiteIntel seed 50/天生效；dns_fail 评分缺陷修复已部署生效 → **COMPLETED**

---

## 4. www Entity 合并（COMPLETED — 无需再次执行）

- **做过什么**：08-16 先出 dry-run 影响报告（92 实体/191 关系/17 冲突/82 自环），用户授权后执行（commit `3382909`）。
- **结果**：92 个 www.* 实体 → **0**；84 repoint + 8 rename；191 关系中 12 重指向、17 冲突零损失合并（evidence 并集 + firstSeen 取早 + lastSeen 取晚）、82 条 apex↔www 自环删除；domain 实体 294→210；孤立关系 0、重复 normalized 0、剩余自环 0。
- **备份/审计**：`EntityBackup_20260816_www_merge`（92 行）、`RelationshipBackup_20260816_www_merge`（191 行）、`scripts/audit/merge-www-entities-2026-08-16.json`；`--verify` 6/6。
- **线上验证**：/technology sitevance chip=1、报告页 200、sitemap 133 URL 零异常、provider_changed 12 条无新增。
- **当前状态**：COMPLETED；数据快照 www 残留 0。**不需要再次执行。**

---

## 5. Organization 历史实体合并

### 已完成（COMPLETED）
- 建模报告（08-16）：137→160 org 实体盘点；21 个历史待处理实体分 6 组（同 ASN 拼写变体/区域命名/知名主体/注册商角色/上游可疑 Kuaishou/垃圾值）；推荐方案 A。
- Phase A（commit `b031a5d` + 报告 `0d64a71`）：OrganizationAlias 加 kind/status/note；OrganizationDenyLog 表；deny 规则；asn→org 补 metadata.kind；alias 解析仅 approved 生效；Kuaishou 置 pending + MANUAL REVIEW 备注。migration 20260816_organization_alias_phase_a 已部署，110/110 测试。

### 未执行（ANALYZED / NOT EXECUTED / DRY-RUN / WAITING FOR AUTHORIZATION）
- **21 个历史实体合并（Phase C 迁移计划 → Phase D 人工确认 → Phase E 正式迁移）未执行**。脚本 `scripts/merge-organization-aliases.mjs` 存在且默认 dry-run；生产数据零 UPDATE/DELETE/合并。
- 组 4 注册商实体：合并目标已固化（两条 HiChina 拼写 → "HiChina (Alibaba Cloud)"，与 Alibaba Cloud 主实体分离），但**执行时机待确认**。
- Kuaishou / ASN 23724：`status=pending`，人工核实真实 ASN 前**不合并、不删除**（WAITING FOR MANUAL REVIEW）。
- China Unicom Network / AS0：未处理，等 AS0 治理统一处置（WAITING）。

### 约束（硬性）
> **未获得用户新的明确授权前，Codex 不得执行 Organization 历史实体正式合并。** 保持 dry-run 状态。

---

## 6. 数据差异 / 异常 / 暂缓事项

### 已解决（COMPLETED）
- seed 别名勘误 94→90（用户裁决，以实际脚本为准）
- safe-fetch path 污染（错误 ip→asn 400619 关系等）→ 已纠错，0 残留
- provider_changed 历史假阳性事件 → 已清理
- 3 个 dns_fail 候选评分标签 → 06f2486 修复后刷新（130→115 实证）
- wordpress.org 监视器死循环（P0，多源 Phase 0 修复；修复前 ~48 次/天 → 修复后正常）
- 候选繁殖停摆 → 判定为**设计性饱和（非故障）**，机制按设计工作
- 3 个孤立 domain 实体 → 已删（wwww.baidu.com.cn / xinngkongrj.com / laosijtv.com）

### 保留/有意不改（记录在案）
- AS0 僵尸行：永久保留（页面 404/不入 sitemap/不再新增）
- downcc.com：保留为 `real_domain_collection_failed`（DNS 失败但 RDAP 成功）
- example.com：completed + 数据仍 **noindex**（低价值防线实证，非 bug）
- Kuaishou/23724：pending，不动
- f1 worktree（/root/siteintel-f1 @ 1bdfcb4，4 项未提交）：**绝不触碰**
- 49 个基线排除文件（生产对齐后 ?? 33 项，含目录折叠）：磁盘保留，处置待决策

### 暂缓（WAITING FOR AUTHORIZATION / DECISION）
- Organization 历史合并（§5）
- 多源 Phase 2（Majestic）——暂停等确认（回滚：DB /tmp/siteintel_pre_phase0.sql + git checkout master + .env.bak.phase0）
- SEO Phase 2 / 阶段 II 卫生收口 / deploy.sh / 删 staging / 删 `siteintel_staging_ro`
- tsconfig.json 归一化、12 文档 + DISCOVERY 入库、README/License、baidu_verify 大小写双文件

### 记录缺样本（不伪造）
- running / failed-有历史 两种状态线上无样本 → 报告标注 `NO PRODUCTION SAMPLE AVAILABLE`，待生产自然出现后补验

---

## 7. 审计 / 修复 / 验收 / 数据库 / 部署报告清单

报告均在 `C:\Users\deepo\siteintel\`（服务器对应副本在 /www/wwwroot/siteintel.cc 或 /www/backup）：

- 修复：`SITEINTEL-REMEDIATION-REPORT.md`、`SAFE-FETCH-DATA-CORRECTION-REPORT.md`、`SITEINTEL-POST-REMEDIATION-REGRESSION-AUDIT.md`
- 数据/实体：`WWW-ENTITY-MERGE-IMPACT-REPORT.md`（含执行记录）、`ORGANIZATION-ENTITY-MODELING-REPORT.md`、`ORGANIZATION-ENTITY-PHASE-A-REPORT.md`、`DOMAIN-ENTITY-GARBAGE-CORRECTION-REPORT.md`、`DOMAIN-ENTITY-CREATION-GUARD-REPORT.md`、`SITEINTEL-CURRENT-DATA-SNAPSHOT.md`、`DISCOVERY-PRIORITY-AGING-REPORT.md`
- 2.0：`SITEINTEL-2.0-PRODUCT-UPGRADE-BLUEPRINT.md`、`SITEINTEL-USER-EXPERIENCE-BASELINE-AUDIT.md`、`SITEINTEL-2.0-INFORMATION-ARCHITECTURE.md`、`SITEINTEL-2.0-PHASE-3A/3B/3B1/3B2/3B3/3C-REPORT.md`
- SEO：`SITEINTEL_SEO_PLAN_AUDIT_REPORT.md`、`SITEINTEL_SEO_PHASE1_P0_IMPLEMENTATION/ACCEPTANCE/STAGING_*/PRODUCTION_*/GIT_RELEASE_REPORT.md`
- Git/部署：`SITEINTEL_PRODUCTION_BASELINE_REPORT.md`、`SITEINTEL_GITHUB_BASELINE_PREP/ESTABLISHED.md`、`SITEINTEL_PRODUCTION_GIT_BASELINE_UNIFICATION_PLAN.md`、`SITEINTEL_GIT_HISTORY_UNIFICATION_ARCHITECTURE/REPORT.md`、`SITEINTEL_PRODUCTION_GIT_AND_DEPLOYMENT_WORKFLOW_FINALIZATION_PLAN.md`、`SITEINTEL_PRODUCTION_GIT_HEAD_ALIGNMENT_REPORT.md`
- 多源/诊断：`SITEINTEL_AUTO_PROPAGATION_PRODUCTION_DIAGNOSIS.md`、`SITEINTEL_MULTI_SOURCE_DISCOVERY_AUDIT.md`、`SITEINTEL_MULTI_SOURCE_PHASE0_CFRADAR_ACCEPTANCE_REPORT.md`、`SITEINTEL_MULTI_SOURCE_SEED_TEST_REPORT.md`、`SITEINTEL_SEED_DAILY_BATCH_50_REPORT.md`

---

## 8. 当前真正剩余的 TODO

### WAITING FOR AUTHORIZATION
1. Organization 历史实体合并（Phase C→D→E；21 实体，脚本 dry-run 就绪）
2. 多源发现第二阶段（Majestic；等确认）
3. SEO Phase 2（entity SEO 等）
4. 删除 staging（siteintel-staging :3004）与只读角色 `siteintel_staging_ro`
5. deploy.sh 部署脚本落库（未来部署通道=Git bundle）
6. 阶段 II 卫生收口（16 项 B 类 + tsconfig 归一化 + 49 排除文件处置 + 12 文档/DISCOVERY 入库）
7. f1-candidate-filter 分支 4 项未提交处置决策

### WAITING FOR DECISION
- README 首页模块描述更新 / LICENSE 添加（GitHub）
- baidu_verify 大小写双文件最终处置
- Kuaishou ASN 23724 真实归属人工核实（保留 vs 删边）

### P3 / 记录项（不在已冻结任务书内，未授权不实施）
- entity 页 robots/sitemap 同源化
- SSE 流端点鉴权；图谱超节点降噪（GA/Fonts 类普适技术）；sitemap lastmod 内容时间
- 法律页（Privacy/ToS）+ 移除/退出流程
- 全仓 12 个历史 lint 错误清理

### 数据层观察（记录不擅动）
- DiscoveryCandidate pending 积压（875+，20/天额度 vs 588/天供给；SAN 源 94%）
- shared_infra@40 候选持续饥饿；aged 60 封顶 100 的权衡可调（已记录）
- 监控/洞察噪声型增长（wordpress.org 曾 34 条洞察/24h；修复后已收敛）

---

## 9. UNKNOWN（未猜测）
- 今天生产实时状态：服务 HTTP、git 工作树细目、数据库实时计数、多源 Phase 0 文件在生产 git 的落位——本地无法直读，需授权后只读核验
- IPIP0/duoniu 等其他项目与本项目无关，未混入

---

## 10. Codex 接手约束

1. 已完成工作一律不重复执行（§1/§4/§6）。
2. 未授权的高风险操作（生产 DB、部署、Git push、Organization 合并、Majestic、SEO Phase 2）保持等待状态。
3. 执行任何新任务前先读本文件 + `PROJECT_STATE.md` + 相关报告；改动遵循 `AGENTS.md`（低 Token 规范）。
4. 发现与记录不符时：停止 → 报告 → 等用户确认，不猜测。
