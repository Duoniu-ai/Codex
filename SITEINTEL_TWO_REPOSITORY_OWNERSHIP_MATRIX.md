# SiteIntel 双仓库文件归属矩阵

> 日期：2026-08-18 ｜ 性质：只读审计（未修改/未删除/未提交/未推送）
> 范围：`Duoniu-ai/siteintel`（代码仓库）与 `Duoniu-ai/Codex`（文档仓库）

## 审计时点状态

| 仓库 | main HEAD | 可见性 | 说明 |
|---|---|---|---|
| Duoniu-ai/siteintel | `27bbf93` | private | 正式产品代码仓库；生产来源 |
| Duoniu-ai/Codex | `9c14d01` | **public** | 文档仓库；刚刚完成非 SiteIntel 文件清理（947d85b + 9c14d01） |

## 归属矩阵

### A. SiteIntel 正式源代码（仅 siteintel 仓库）

| 文件/目录 | siteintel | Codex | 重复 | 应保留位置 | 原因 |
|---|---|---|---|---|---|
| `src/`（179 文件：app/components/lib） | ✅ tracked | ❌ 无 | 无 | siteintel | 生产源代码（前端/后端/API/Worker/Scheduler） |
| `prisma/`（schema + 7 migrations） | ✅ tracked | ❌ 无 | 无 | siteintel | 数据库 Schema / ORM |
| `scripts/`（7 文件） | ✅ tracked | ❌ 无 | 无 | siteintel | 数据治理/审计脚本 |
| `deploy.sh` | ✅ tracked | ❌ 无 | 无 | siteintel | 部署文件（生产可重建） |
| `package.json` / `pnpm-lock.yaml` / `pnpm-workspace.yaml` | ✅ tracked | ❌ 无 | 无 | siteintel | 构建配置 |
| `next.config.ts` / `tsconfig.json` / `eslint.config.mjs` / `postcss.config.mjs` / `vitest.config.ts` | ✅ tracked | ❌ 无 | 无 | siteintel | 构建/测试配置 |
| `.env.example` / `.gitignore` | ✅ tracked | ❌ 无 | 无 | siteintel | 配置模板（不含密钥） |

### B. SiteIntel 项目文档（Codex 仓库为发布地，siteintel 工作树为本地副本）

| 文件 | siteintel 工作树 | Codex 仓库 | 重复 | 应保留位置 | 原因 |
|---|---|---|---|---|---|
| `README.md` | ✅ tracked | ❌ | 无 | siteintel | 代码仓库 README |
| `CURRENT_ARCHITECTURE.md` | ✅ tracked | ❌（旧版为 DUONIU 同名文件，已删） | 无 | siteintel | 架构文档随代码维护 |
| SiteIntel-X-Architecture.md / X-SEO-Architecture.md / X-Search-Intelligence-*.md / SiteIntel_V2_总纲 / 网址发现平台-*（5）/ 网址导航站-*（4）/ 上游数据能力审计 / SiteIntel-Website-Upgrade-SEO-Master-Plan / SiteIntel.cc Master Prompt ×2 / 多源发现引擎指令 / 白盒审计任务书 / 自动繁殖诊断指令 / SEO 规划 ×4 / siteintel-full-site-analysis / siteintel_seo_audit / SITEINTEL-FULL-SITE-ANALYSIS-AUDIT / SITEINTEL-2.0-PRODUCT-UPGRADE-BLUEPRINT / SITEINTEL SEO PDF | ✅ 本地未跟踪副本 | ✅ tracked（36 份） | 本地副本 vs Codex 发布版（内容一致，行尾差异） | **Codex 仓库（发布）+ siteintel 工作树（工作副本）** | 正式项目文档；代码仓库不 track 文档 |

### C. SiteIntel 状态文件

| 文件 | siteintel | Codex | 重复 | 应保留位置 | 原因 |
|---|---|---|---|---|---|
| `SITEINTEL_STATE.md` | ✅ 本地未跟踪（`Documents\Codex\2026-08-18\you\` 亦有副本，内容有差异） | ✅ tracked | 是（本地版本差异） | **Codex 仓库** | SiteIntel 状态文件；建议以 Codex 仓库版为发布基线，本地副本按需同步 |
| `PROJECT_STATE.md` | ❌ | ✅ tracked（Ambiguous） | 无 | **Codex 仓库（待人工确认）** | 通用上下文迁移状态，非 SiteIntel 专属 |

### D. SiteIntel Phase / Step 报告（Codex 仓库为发布地）

| 文件 | siteintel 工作树 | Codex 仓库 | 重复 | 应保留位置 | 原因 |
|---|---|---|---|---|---|
| SITEINTEL_2.0_GAP_ANALYSIS_REPORT.md | ✅ 未跟踪 | ✅ tracked（947d85b） | 本地=发布（行尾差异） | **Codex 仓库** | 正式审计报告 |
| SITEINTEL_PHASE1_P0_SECURITY_PLAN.md | ✅ 未跟踪 | ✅ tracked（947d85b） | 同上 | **Codex 仓库** | 正式方案 |
| SITEINTEL_PHASE1_STEP1_BASELINE_COMPLETION_REPORT.md | ✅ 未跟踪 | ✅ tracked | 同上 | **Codex 仓库** | 完成报告 |
| SITEINTEL_PHASE1_STEP2_SAFE_FETCH_COMPLETION_REPORT.md | ✅ 未跟踪 | ✅ tracked | 同上 | **Codex 仓库** | 完成报告 |
| SITEINTEL_PHASE1_STEP3_REPORT_RATELIMIT_COMPLETION_REPORT.md | ✅ 未跟踪 | ✅ tracked（947d85b） | 同上 | **Codex 仓库** | 完成报告 |
| SITEINTEL_REPOSITORY_CLEANUP_PLAN.md | ✅ 未跟踪 | ✅ tracked（947d85b） | 同上 | **Codex 仓库** | 仓库治理计划 |
| SITEINTEL_REPOSITORY_CLEANUP_COMPLETION_REPORT.md | ❌ 未生成 | ❌ | — | **Codex 仓库（待补）** | 上轮清理任务被本审计中断，完成报告待用户决定后补 |

### E. Codex 工作资料（Ambiguous，已保留在 Codex 仓库，等待人工判断）

| 文件 | Codex 仓库 | 建议 |
|---|---|---|
| `Codex + DeepSeek V4 Flash 低 Token 开发规范.md` | ✅ tracked | 保留观察；通用规范，非 SiteIntel 专属 |
| `Claude Code 历史上下文迁移任务.md` | ✅ tracked | 保留观察；通用迁移任务 |
| `D3 数据库连接阻塞后的服务器 PostgreSQL 核验指令.md` | ✅ tracked | 保留观察；D3 项目指令 |

### F. 其他项目文件（已从 Codex 当前工作树删除，历史保留）

| 文件 | 原属 | 当前状态 |
|---|---|---|
| CURRENT_ARCHITECTURE.md / (1).md | DUONIU.cc | Codex 工作树已删（9c14d01）；历史保留 |
| DUONIU-*.md（6）| Duoniu | 同上 |
| IPCECE_*.md（1）| IPCECE | 同上 |
| IPIP0-*.md（8）| IPIP0 | 同上 |
| PROJECT-FULL-AUDIT.md / (1).md | ipcesu.com | 同上 |
| duoniu-cc-website-analysis-and-repositioning.md | Duoniu | 同上 |
| ipcesu-final-plan.md / ipcesu_deploy.tgz / ipcesu_src.tgz | ipcesu | 同上（tgz 仅存于历史） |

### G. 临时/工作区文件（不属于 GitHub 仓库）

| 位置 | 内容 | 说明 |
|---|---|---|
| `C:\Users\deepo\Documents\Codex\`（本地目录，非 git） | ipcesu.com 源码/node_modules、ipcece/ipip0 工作产物、SSH 工作日志等 | 桌面 Codex 工作区；与 GitHub Codex 仓库**无关**，不在仓库管理范围 |

### H. 不确定文件（等待人工判断）

1. `PROJECT_STATE.md`（Codex 仓库，已保留）
2. `Claude Code 历史上下文迁移任务.md`（Codex 仓库，已保留）
3. `Codex + DeepSeek V4 Flash 低 Token 开发规范.md`（Codex 仓库，已保留）
4. `D3 数据库连接阻塞后的服务器 PostgreSQL 核验指令.md`（Codex 仓库，已保留）

## 结论

- **不存在 SiteIntel 源代码重复**：Codex 仓库历史与当前树均只有 `.md`/`.pdf`/`.tgz`，从未包含任何 `.ts`/`.prisma`/`.json` 源码。
- 文档“重复”仅为本地工作副本 vs Codex 发布版（内容一致，仅行尾差异），非分叉。
- 推荐：siteintel = 唯一代码/部署仓库；Codex = 唯一文档/报告/状态发布仓库。
