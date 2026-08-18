# SiteIntel 双仓库关系审计报告

> 日期：2026-08-18 ｜ 性质：**只读审计**（未修改代码/文件/Git/数据库/生产/服务器；未 commit/push/reset/rebase/force push）
> 范围：`Duoniu-ai/siteintel` 与 `Duoniu-ai/Codex`
> 佐证：GitHub API 只读、本地 git 只读、生产服务器只读（`git rev-parse`/`BUILD_ID`/`deploy.sh`）

---

## 1. 执行摘要

两个仓库职责**已经事实分离且无代码冲突**：

- **`Duoniu-ai/siteintel`（private）= SiteIntel 唯一正式产品代码仓库**，也是生产运行代码的唯一来源（生产 git HEAD = `27bbf93` = siteintel main，生产服务器无 remote、Codex commit 对象不存在于生产 git）。
- **`Duoniu-ai/Codex`（public）= SiteIntel 文档/报告/状态知识库**，历史与当前树均只含 `.md`/`.pdf`/`.tgz`，**从未包含任何 SiteIntel 源代码**；两仓库无共同祖先、无相同 commit、无 fork/clone 关系。
- 上轮已授权清理已生效：Codex 当前工作树 44 个文件（40 份 SiteIntel 文档 + 4 份 Ambiguous），23 个非 SiteIntel 文件已删除（历史保留）。
- 遗留事项：清理任务的完成报告（SITEINTEL_REPOSITORY_CLEANUP_COMPLETION_REPORT.md）因本审计任务开始而**未生成/未推送**，等待用户决定。

## 2. siteintel 仓库现状

| 项 | 实测 |
|---|---|
| main HEAD | `27bbf93e5951cf95f9e5d163cb9cb51080778557`（fix(api): rate-limit /api/report…） |
| 默认分支 | `main` |
| 可见性/大小 | private / ~995 KB |
| 分支（4） | main、legacy-prod-history（1bdfcb4）、multi-source/phase0-cfradar（06f2486）、seo/phase-1-p0 |
| Tags（6） | production-baseline-2026-08-17→b52ec84；production-history-archive-20260817→1bdfcb4；production-seo-phase1-p0-2026-08-17→c74dfd1；production-phase1-baseline-2026-08-18→f89ad89；production-phase1-step2-safefetch-2026-08-18→de8b6da；production-phase1-step3-report-ratelimit-2026-08-18→27bbf93 |
| 最近 20 commits | 仅 8 个（main 根 = b52ec84 基线）：b52ec84 → c74fd1 → e32bd93 → 15de2d9 → 06f2486 → f89ad89 → de8b6da → 27bbf93 |
| 目录结构 | 210 tracked 文件：`src/` 179、`prisma/` 10（schema + 7 migrations）、`scripts/` 7、根配置（package.json、pnpm-lock、next.config、tsconfig、eslint、postcss、vitest、.env.example、.gitignore、deploy.sh、README.md、CURRENT_ARCHITECTURE.md、baidu_verify） |
| package.json | Next.js 16.3 / React 19 / Prisma 6 / jose / vitest（依赖极简） |
| 数据库 | prisma/schema.prisma + migrations（含 20260817_multi_source_candidate_pool） |
| 前端 | src/app（App Router 页面 + API routes）+ src/components |
| 后端 | src/lib（providers/pipeline/discovery/intelligence/search/security/seo…） |
| API | src/app/api/* 与 api/v1/* |
| Worker/Scheduler | src/instrumentation.ts + monitoring/search-sync/discovery |
| tests | src 内 19 个测试文件（292 用例，本会话实测） |
| deployment | deploy.sh（最小可重建脚本）+ systemd/nginx 文档化流程 |
| README/状态 | README.md、CURRENT_ARCHITECTURE.md tracked；其余报告在本地工作树未跟踪（45 项） |
| 是否生产实际源代码 | **是** |
| 生产 commit 是否来自本仓库 | **是**（见 §7） |
| 是否存在 SiteIntel 核心代码 | **是**（完整） |

## 3. Codex 仓库现状

| 项 | 实测 |
|---|---|
| main HEAD | `9c14d01`（chore(repo): remove non-SiteIntel files） |
| 可见性/大小 | **public** / ~1712 KB（历史含 2 个 tgz 与 1 个 pdf 所致） |
| 分支 | 仅 main |
| Tags | 无 |
| 最近 commits（6） | f4d9dd4 / 71056a8 / 70d931e / 0a8eebe（“Add files via upload”×4，网页上传）→ 947d85b（docs: publish Phase 1 reports + cleanup plan）→ 9c14d01（清理） |
| 当前目录结构 | 44 个文件，全部根目录扁平：43 `.md` + 1 `.pdf` |
| SiteIntel 代码 | **无**（历史与当前均无 .ts/.prisma/.json/.js 源码） |
| SiteIntel 文档 | 40 份 tracked（规划/架构/审计/SEO/指令/Phase 1 报告） |
| PROJECT_STATE.md | ✅ tracked（Ambiguous，保留） |
| SITEINTEL_STATE.md | ✅ tracked |
| SITEINTEL_* 文件 | 16 份 tracked（含 5 份 Phase 1 正式文档） |
| Phase 1 报告 | STEP1/2/3 + GAP + P0 计划 + 清理计划 全部 tracked |
| 其他项目文件（当前工作树） | **0**（清理后）；历史中曾含 DUONIU/IPIP0/ipcesu/IPCECE 共 23 个文件 + 2 个 ipcesu tgz |
| 本地 `C:\Users\deepo\Documents\Codex\` 目录 | 桌面 Codex 工作区（含 ipcesu 源码、其他项目产物）——**与 GitHub Codex 仓库无关**，不属仓库管理范围 |

## 4. 两仓库职责分析

| 维度 | siteintel | Codex |
|---|---|---|
| 实际职责 | 产品代码 + 部署 + 数据库迁移 + 测试 | 文档 + 报告 + 状态 + 规划知识库 |
| 生产方式 | Git（commit/branch/tag/FF merge） | GitHub 网页上传 + 少量 git 提交 |
| 可见性 | private | public |
| 代码唯一性 | 是（唯一） | 否（无代码） |
| 与生产关系 | 生产来源（唯一） | 无 |
| 与本地关系 | C:\Users\deepo\siteintel（master=main） | 发布镜像（本地工作树文档为权威工作副本） |

## 5. 代码重复分析

- **不存在 SiteIntel 代码重复**。Codex 仓库全历史文件扩展名仅 `.md` / `.pdf` / `.tgz`（tgz 为 ipcesu 源码包，已从工作树删除、仅存历史）。
- 文档“重复”仅 2 类：
  1. Codex 发布版 vs siteintel 本地工作树副本（STEP1/2 逐字节比对：**仅 CRLF/LF 行尾差异，内容完全一致**）；
  2. `SITEINTEL_STATE.md` Codex 版 vs `Documents\Codex\2026-08-18\you\` 本地版（哈希不同，内容版本差异，待定）。
- 无分叉：不存在两个仓库对同一文件的竞争编辑。

## 6. Git 历史关系

- **无共同祖先**：siteintel 的历史根为 b52ec84（2026-08-17 生产基线，另有一条独立根 legacy-prod-history=1bdfcb4）；Codex 的历史根为网页上传提交（2026-08-18）。
- **非 fork / 非 clone**：Codex 由 GitHub 网页 “Add files via upload” 创建，无 clone 关系。
- **无相同 commit**：两仓库 commit SHA 全集零交集。
- **存在文档复制**：本地生成的 SiteIntel 文档被上传/提交至 Codex（内容一致，行尾差异）；siteintel 仓库从未 track 这些文档（除 README/CURRENT_ARCHITECTURE）。
- **无代码迁移**：siteintel 代码从未进入 Codex；Codex 内容从未进入 siteintel 代码树。

## 7. Production 来源确认（实际部署证据，非猜测）

| 项 | 实测证据 |
|---|---|
| production repository | **Duoniu-ai/siteintel**（唯一） |
| production commit | `27bbf93e5951cf95f9e5d163cb9cb51080778557` |
| GitHub repository main | siteintel main = `27bbf93`（一致） |
| local repository | `C:\Users\deepo\siteintel` master = `27bbf93`（一致） |
| 生产服务器 git | `/www/wwwroot/siteintel.cc` HEAD=`27bbf93`；`git log` 与 siteintel main 相同；**无 remote**（代码经 git bundle 通道交付） |
| 构建产物 | 生产 `.next/BUILD_ID` = `ApSUYoilzyvSZ4_cbwNpb`（Step 3 部署构建） |
| deploy.sh | 生产存在（1751B），tracked 于 siteintel 仓库 |
| Codex 关联 | 生产 git 中 `9c14d01` 对象**不存在** → 生产不可能来自 Codex |
| 结论 | **生产运行代码 100% 来自 Duoniu-ai/siteintel** |

## 8. 当前 Phase 1 文件归属

| 文件 | 当前 tracked 位置 | 建议保留位置 |
|---|---|---|
| SITEINTEL_PHASE1_STEP1_BASELINE_COMPLETION_REPORT.md | Codex | Codex（发布）+ siteintel 本地工作树（工作副本） |
| SITEINTEL_PHASE1_STEP2_SAFE_FETCH_COMPLETION_REPORT.md | Codex | 同上 |
| SITEINTEL_PHASE1_STEP3_REPORT_RATELIMIT_COMPLETION_REPORT.md | Codex | 同上 |
| SITEINTEL_2.0_GAP_ANALYSIS_REPORT.md | Codex | 同上 |
| SITEINTEL_PHASE1_P0_SECURITY_PLAN.md | Codex | 同上 |
| PROJECT_STATE.md | Codex | Codex（Ambiguous，待人工确认归属） |
| SITEINTEL_STATE.md | Codex | Codex（发布基线）；本地副本按需同步 |

原则：**siteintel 仓库不 track 报告文档**（避免代码仓库被文档污染）；Codex 是文档唯一发布仓库；本地 siteintel 工作树保留权威工作副本。

## 9. 非 SiteIntel 文件清单（Codex 当前工作树）

**0 个**（清理已完成）。历史中曾存在 23 个（DUONIU ×8、IPCECE ×1、IPIP0 ×8、ipcesu 相关 ×4、CURRENT_ARCHITECTURE(DUONIU) ×2），均保留在 git 历史中可追溯。

## 10. 不确定文件清单（4 个，均保留在 Codex）

1. `PROJECT_STATE.md` — 通用 Claude→Codex 上下文迁移状态
2. `Claude Code 历史上下文迁移任务.md` — 通用迁移任务
3. `Codex + DeepSeek V4 Flash 低 Token 开发规范.md` — 通用开发规范
4. `D3 数据库连接阻塞后的服务器 PostgreSQL 核验指令.md` — D3 项目指令

## 11. 推荐长期仓库结构

```
Repository A：Duoniu-ai/siteintel（private）
  角色：SiteIntel 唯一正式产品代码/部署/数据库迁移/测试仓库
  内容：src/ prisma/ scripts/ deploy.sh 构建配置 README CURRENT_ARCHITECTURE
  规则：不 track 报告/计划/状态文档；文档只在本地工作树存在或由 Codex 发布

Repository B：Duoniu-ai/Codex（建议评估转为 private）
  角色：SiteIntel 文档/报告/状态/规划唯一发布仓库
  内容：SITEINTEL_* 与 SiteIntel* 文档、Phase 报告、状态文件
  规则：只放 SiteIntel 项目文件；禁止其他项目文件进入
```

## 12. 推荐清理方案（待用户审核，本次未执行）

1. **Codex 仓库**：
   - 已完成的清理（9c14d01）保持现状；
   - 待补 `SITEINTEL_REPOSITORY_CLEANUP_COMPLETION_REPORT.md` 并推送；
   - 4 个 Ambiguous 文件：等待人工决定保留/迁移/删除（建议至少保留 `Codex + DeepSeek V4 Flash 低 Token 开发规范.md` 作为团队规范，其余视归属）；
   - 评估可见性：仓库当前 public，含生产服务器公网 IP/部署路径等运维细节——建议转为 private（需用户操作或授权）。
2. **siteintel 仓库**：维持现状（代码仓库不 track 文档）；可选：`README.md`/`CURRENT_ARCHITECTURE.md` 继续随代码维护。
3. **本地工作区**：`C:\Users\deepo\Documents\Codex\` 中的其他项目产物（ipcesu 等）与 GitHub Codex 仓库无关，不需要处理；建议单独建立各项目自己的仓库，避免再次混入。

## 13. 清理风险

- Codex 工作树删除不影响任何运行系统（文档仓库，无代码依赖）；git 历史完整，可随时恢复。
- 文档“重复”仅为工作副本 vs 发布版；若未来编辑不同步，可能出现版本漂移——建议以“本地 siteintel 工作树 = 编辑源，Codex = 发布镜像”为唯一流程。
- Codex public 可见性：运维细节（服务器 IP、部署路径、安全方案）公开可读，属信息暴露风险；转为 private 需用户决定。
- 4 个 Ambiguous 文件的删除/保留若判断错误，影响仅为文档丢失（git 历史可恢复），风险低。

## 14. 下一步建议

1. 用户审核本报告与归属矩阵。
2. 决定 4 个 Ambiguous 文件去留。
3. 决定 Codex 仓库可见性（private/public）。
4. 决定是否补推 `SITEINTEL_REPOSITORY_CLEANUP_COMPLETION_REPORT.md`。
5. 确立长期规则：**代码只进 siteintel；文档只进 Codex；本地 siteintel 工作树为文档编辑源**。

---

*本报告为只读审计产物，未对任何仓库/生产/服务器做修改。*
