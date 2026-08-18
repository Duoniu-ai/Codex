# SiteIntel 仓库清理计划 — Duoniu-ai/Codex

> 日期：2026-08-18 ｜ 性质：只读盘点 + 清理计划（执行前）
> 目标：`Duoniu-ai/Codex`（main）当前工作树只保留 SiteIntel 项目相关文件；Git 历史保留，不 force push、不重写历史。

---

## 1. 当前仓库文件数量

**63 个 tracked 文件**（全部位于仓库根目录，无子目录结构；无 .gitignore；无 untracked 文件；无 ignored 文件）。

- 分支：`main`
- 清理前 HEAD：`0a8eebe225d2e55190fc88c05a79941b19225fc8`（= origin/main）
- remote：`https://github.com/Duoniu-ai/Codex.git`

## 2. SiteIntel 文件数量

当前树中明确属于 SiteIntel 的文件：**36 个**（SITEINTEL_* / SiteIntel* / siteintel* / 网址发现平台-* 家族）。

另按既定文档发布规则补充 3 份正式 SiteIntel 文档（当前不在仓库，需新增）：

- `SITEINTEL_2.0_GAP_ANALYSIS_REPORT.md`
- `SITEINTEL_PHASE1_P0_SECURITY_PLAN.md`
- `SITEINTEL_PHASE1_STEP3_REPORT_RATELIMIT_COMPLETION_REPORT.md`

（STEP1/STEP2 完成报告已在仓库中。）

## 3. 非 SiteIntel 文件数量

**23 个**（DUONIU / IPCECE / IPIP0 / CURRENT_ARCHITECTURE(DUONIU) / PROJECT-FULL-AUDIT(ipcesu) / ipcesu 源码与部署包 / duoniu-cc 分析）。

## 4. Ambiguous 文件数量

**4 个**（无法确认归属，按规则不删除，等待人工判断）：

1. `PROJECT_STATE.md`（Claude Code → Codex 历史上下文迁移状态，非 SiteIntel 专属）
2. `Claude Code 历史上下文迁移任务.md`（通用上下文迁移任务）
3. `Codex + DeepSeek V4 Flash 低 Token 开发规范.md`（通用开发规范）
4. `D3 数据库连接阻塞后的服务器 PostgreSQL 核验指令.md`（D3 项目数据库核验指令）

## 5. 准备删除的文件完整列表（23 个）

```text
CURRENT_ARCHITECTURE (1).md
CURRENT_ARCHITECTURE.md
DUONIU-AUDIT-20260816.md
DUONIU.CC 下一阶段代码修复与产品升级执行任务书.md
DUONIU_EXECUTION_RULES.md
DUONIU_IMPLEMENTATION_PLAN (1).md
DUONIU_IMPLEMENTATION_PLAN.md
DUONIU_UPGRADE_RULES.md
IPCECE_全面升级实施规划_ClaudeCode_DeepSeek.md
IPIP0 P2 预设计与 Gate D 等待期任务.md
IPIP0 下一阶段综合任务.md
IPIP0 阶段五 P0 产品完整性修复.md
IPIP0 阶段五 P1 产品完整性补全.md
IPIP0 阶段五：全站产品功能完整性审计.md
IPIP0 首页结果卡片专项修复任务.md
IPIP0-两项功能完善分析报告.md
IPIP0-全站分析与升级改版规划.md
PROJECT-FULL-AUDIT (1).md
PROJECT-FULL-AUDIT.md
duoniu-cc-website-analysis-and-repositioning.md
ipcesu-final-plan.md
ipcesu_deploy.tgz
ipcesu_src.tgz
```

## 6. 准备保留的关键目录 / 文件

- 仓库为扁平结构，无子目录；保留范围为根目录下全部 SiteIntel 文件（36 个现有 + 3 个新增）。
- 保留的 4 个 Ambiguous 文件（待人工判断）。
- 保留的 SiteIntel 文件清单见本计划第 2 节归类；关键类型：产品规划（V2_0/V2_1/V2.1.1/V2.1.2）、架构（SiteIntel-X-Architecture、SiteIntel_V2 总纲）、审计（SITEINTEL-FULL-SITE-ANALYSIS-AUDIT、siteintel-full-site-analysis、siteintel_seo_audit）、状态（SITEINTEL_STATE）、Phase 1 完成报告（STEP1/2/3）、上游数据能力审计、多源发现引擎指令、SEO 规划、白盒审计任务书、自动繁殖诊断指令、Claude Code 执行指南、Master Prompt、PDF 战略审查报告。

## 7. 判断依据

- **删除**：文件名/内容明确指向 Duoniu.cc、IPCECE、IPIP0、ipcesu.com 等项目（内容抽查实证：CURRENT_ARCHITECTURE*.md 标题为「DUONIU.cc 当前架构审计报告」，PROJECT-FULL-AUDIT*.md 标题为「IP测速网（ipcesu.com）项目全面审计任务」）。
- **保留**：文件名/内容明确含 SiteIntel / siteintel / 网址发现平台 / 网址导航站（SiteIntel 产品家族）。
- **Ambiguous**：通用上下文/规范/状态类，无法确认归属 → 按规则不删除。
- **敏感信息**：待新增 3 份报告与计划/完成报告不含凭据（无 API Key/Token/Password/DB URL/SSH Key）；服务器公网 IP 与部署路径属于既有公开文档既定内容（STEP1/2 已在仓库），与本次规则一致。

---

## 执行摘要（本计划批准后）

1. 新增 3 份 SiteIntel 正式文档 + 本计划 → commit。
2. `git rm` 上述 23 个非 SiteIntel 文件 → 独立 commit `chore(repo): remove non-SiteIntel files`。
3. `git status` / `git diff --stat` / `git diff --name-status` 复查，确认无 SiteIntel 误删。
4. push origin main（禁 force push / rebase / 改历史）。
5. 验证 origin/main = local HEAD；GitHub main 树确认 SiteIntel 文件存在、非 SiteIntel 文件移除。
6. 生成并推送完成报告。
