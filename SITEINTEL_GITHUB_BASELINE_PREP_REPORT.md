# SITEINTEL GITHUB BASELINE PREP REPORT

GitHub 基线准备审计报告 —— 生产基线 → 公开仓库推送前逐项审计

审计时间：2026-08-17（只读审计，未执行任何删除/修改/推送）
审计对象：`154.204.176.66:/www/wwwroot/siteintel.cc`（git 仓库，无 remote）
目标：`https://github.com/Duoniu-ai/siteintel`（**尚未创建**，gh CLI 已登录 Duoniu-ai）

**本阶段执行原则**：只审计、只建议、不执行。所有变更命令列于 §10，等待用户明确指令后执行。

---

## 1. 当前 Production Baseline

| 项 | 值 |
|---|---|
| 代码基线 | 服务器 git **HEAD `1bdfcb4`**（08-16 10:27 UTC，66 commits） |
| 运行构建 | `.next/BUILD_ID cQ-ZftIn6HAbu-vHzVSYg`（08-16 09:16:36）＝ HEAD src（v2 报告 4 条独立证据） |
| 线上版本 | siteintel.cc 当前线上 = 该构建（PRODUCTION BASELINE CONFIRMED，见 SITEINTEL_PRODUCTION_BASELINE_REPORT.md v2） |
| 分支 | `master` 与 `f1-candidate-filter`（二者同点 1bdfcb4，后者为发现候选过滤的预留分支） |
| 仓库规模 | .git 7.6 MB / 1160 loose objects / 240 tracked 文件 / 无 stash / 无 unreachable 对象 |

---

## 2. Git 状态

```
HEAD   1bdfcb4  docs: discovery candidate quality filter v1 design (READ ONLY dry-run)
工作树  M DISCOVERY-CANDIDATE-FILTER-DESIGN.md          （文档修改未提交）
       D baidu_verify_codeva-r7wFVf611L.html            （大写版已从磁盘删除，删除未提交）
       ?? .well-known/  + 14 个未跟踪文档（见 §7）
remote 无
```

无 remote、无 stash、无 tag；`git fsck --no-reflogs --unreachable` 为空（无游离敏感对象）。

---

## 3. 文件卫生问题（汇总）

| # | 问题 | 影响 | 处置 |
|---|---|---|---|
| H1 | `.user.ini` 被 tracked | 服务器环境文件进入公开仓库 | §4：untrack + gitignore |
| H2 | `baidu_verify` 双份 tracked（内容相同） | 冗余 + 大写版为死文件 | §5：保留小写，删除大写跟踪 |
| H3 | `.well-known/` untracked（含验证令牌） | 运行期数据可能被误提交 | §6：gitignore（文件留服务器） |
| H4 | 40+ 份内部审计/规划文档（tracked 24 + untracked 14） | 公开仓库暴露内部报告 | §7：文档公开策略 |
| H5 | `DISCOVERY-CANDIDATE-FILTER-DESIGN.md` 工作树修改未提交 | 推送时需决定去留 | 随 §7 一并处理 |

无代码卫生问题（lint/typecheck/build 绿、无 .next/node_modules 入库、migrations 齐全）。

---

## 4. .user.ini 审计

**内容**：`open_basedir=/www/wwwroot/siteintel.cc/:/tmp/`（单行）
**用途**：宝塔面板（bt panel）自动生成的 PHP 运行目录限制文件。
**是否影响生产运行**：**否**。SiteIntel 为 Next.js（Node 服务，systemd 端口 3003），不经过 PHP-FPM；且 nginx 第 27 行 `location ~ ^/(\.user.ini|...)` 明确拒绝访问，文件从未被服务。
**git 历史**：于 `ab6eb8f`（SiteIntel X Search Foundation 提交）被顺带卷入，非有意添加。
**建议**：**不应版本控制** → `git rm --cached .user.ini` + `.gitignore` 追加 `.user.ini`。
**影响说明**：仅从 git 索引移除，磁盘文件保留（宝塔需要它继续存在），nginx 行为不变、Next.js 行为不变、线上零影响。

---

## 5. Baidu Verify 文件审计

| 文件 | tracked | 磁盘 | 线上实测 | 结论 |
|---|---|---|---|---|
| `baidu_verify_codeva-r7wFVf611L.html`（大写） | 是 | **无**（工作树 D） | **404**（死文件） | 移除跟踪 |
| `baidu_verify_codeva-r7wfvf611l.html`（小写） | 是 | 有（32 B，www） | **200**（在用） | **保留** |

- 两文件内容**完全相同**（git diff HEAD 两版本 = IDENTICAL）。
- nginx `location ^~ /baidu_verify_` 静态直出磁盘文件；Linux 大小写敏感 → 线上仅小写可访问。
- 百度站长验证早已完成（08-15），用户挂的链接为小写——**线上真正需要的 = 小写文件**。
- **建议**：GitHub 基线保留小写文件；大写版删除跟踪（磁盘已删，仅需提交该删除）。两者皆公开文件，进仓库无安全风险。

---

## 6. .well-known 文件审计（逐文件）

| 文件 | 用途 | 线上是否使用 | 是否进入 Git | 原因 |
|---|---|---|---|---|
| `.well-known/siteintel-verification.txt` | 域名所有权验证（Claim 功能，V2 §36） | **是**（nginx `location ~ \.well-known { allow all; root ... }` 静态直出，实测 200） | **否** | 内容为运行期生成的 32 位十六进制验证令牌（公开服务，非密钥）；属服务器运行数据而非项目代码；GitHub 中的副本必然是过期数据，且存在随公开仓库传播的误用风险 |

- 目录内无 security.txt / apple-app-site-association / assetlinks / ACME 等其他文件。
- **建议**：`.gitignore` 追加 `.well-known/`。**影响**：磁盘文件保留、nginx 静态服务不受 git 影响、Claim 功能继续可用。

---

## 7. 文档公开策略

### 7.1 分类标准（按用户给定）

- **A 应公开**：README / Architecture / Public documentation / Development documentation
- **B 不建议公开**：内部审计 / 生产环境分析 / SEO 内部策略 / 临时执行报告 / 含内部路径或基础设施信息的报告

### 7.2 分类结果

**A 应公开（2 份）**：

| 文件 | 说明 |
|---|---|
| `README.md` | 产品介绍 + 架构说明（无服务器路径），可公开 |
| `CURRENT_ARCHITECTURE.md` | 架构审计文档，不含基础设施信息（未命中 IP/路径扫描），可公开 |

**B 不建议公开（tracked 24 份）**：

- 审计/执行报告（15）：DISCOVERY-QUALITY-AUDIT、DISCOVERY-PRIORITY-AGING-REPORT、DOMAIN-ENTITY-CREATION-GUARD-REPORT、DOMAIN-ENTITY-GARBAGE-CORRECTION-REPORT、ORGANIZATION-ENTITY-MODELING-REPORT、ORGANIZATION-ENTITY-PHASE-A-REPORT、SAFE-FETCH-DATA-CORRECTION-REPORT、SITEINTEL-DISCOVERY-SYSTEM-AUDIT、SITEINTEL-FRONTEND-JS-EXPOSURE-AUDIT、SITEINTEL-POST-REMEDIATION-REGRESSION-AUDIT、SITEINTEL-REMEDIATION-REPORT、SITEINTEL-USER-EXPERIENCE-BASELINE-AUDIT、WWW-ENTITY-MERGE-IMPACT-REPORT、DISCOVERY-CANDIDATE-FILTER-DESIGN、SITEINTEL-CURRENT-DATA-SNAPSHOT（**生产数据快照**）
- 产品/版本状态（13）：SITEINTEL-2.0-INFORMATION-ARCHITECTURE、2.0-PHASE-3A/3B/3B1/3B2/3B3/3C-REPORT、docs/SITEINTEL-V2-PHASE0-AUDIT、docs/V2-PHASE1~11-13-STATUS ×12、docs/SITEINTEL-COMPLETION-STATUS
- 内部产品规划（6）：SiteIntel_V2_产品与技术架构总纲_最新版（含产品路线图，**待定 A/B**，建议 B）、docs/SiteIntel-X-Architecture、docs/SiteIntel-X-SEO-Architecture、docs/SiteIntel-X-Search-Intelligence-Final、docs/SiteIntel-X-Search-Intelligence-Plan、docs/SiteIntel-Website-Upgrade-SEO-Master-Plan

**B 不建议公开（untracked 14 项）**：SITEINTEL-2.0-PRODUCT-UPGRADE-BLUEPRINT、SEO 规划 ×4（MASTER_PLAN_V2 / EXECUTION_ROADMAP / TECHNICAL_SEO_IMPLEMENTATION / ACCEPTANCE_CHECKLIST）、3C 执行任务书、docs/SITEINTEL-FULL-SITE-ANALYSIS-AUDIT、docs/白盒审计报告执行任务书、docs/SiteIntel-Claude-Code-执行指南、docs/siteintel-full-site-analysis-audit、docs/navigation/（导航站产品文档）、siteintel-full-site-analysis.md（外部白盒分析）

**基础设施信息暴露面**：tracked 8 份 + untracked 10 份文档含 `154.204.176.66` / `/www/wwwroot` / `:3003` / nginx conf 路径（§7.2 列表与 §4-6 之外）。**公开即暴露服务器拓扑**——B 类文档不应进入公开仓库。

### 7.3 历史暴露问题（必须由用户决策）

66 个 commit 的历史**已包含全部 tracked B 类文档**。Git 推送 = 历史全量公开，工作树的文件去留**不影响历史可见性**。三种路径（本阶段均不执行）：

| 路径 | 做法 | 代价 |
|---|---|---|
| ① 全量推送（含 B 文档历史） | 保留 66 commit 完整历史，仓库即「完整工程档案」 | 内部报告/服务器拓扑公开；无密钥（已验证），属技术性披露 |
| ② 全新基线初始提交 | 以 PUBLIC GITHUB FILESET（§8）做 1 个初始 commit，Tag 指向新根提交 | 丢失 66 commit 历史（**未改写任何旧提交**，原仓库保留在服务器） |
| ③ 折中 | 保留 src/migrations 相关历史子集（按路径重写，filter-repo 类） | **违反本阶段禁止项**（history rewrite），仅作备忘，不推荐 |

推荐 **①**（若接受文档公开）或 **②**（若严格保密）——取决于用户对「内部报告公开」的态度；若选 ②，需在 GitHub 建立 Tag（如 `v2.0.0-prod-1bdfcb4`）锚定基线身份。

---

## 8. PUBLIC GITHUB FILESET（推荐公开文件清单）

| 文件 / 类型 | 当前状态 | 是否推送 GitHub | 原因 |
|---|---|---|---|
| `src/`（161 个 .ts/.tsx） | tracked | **是** | 核心代码 = 线上功能 |
| `prisma/`（schema + 7 migrations） | tracked | **是** | 数据库结构基线 |
| `scripts/` | tracked | **是** | 回填/运维脚本（--env-file 模式，无内嵌凭据） |
| `package.json` / `pnpm-lock.yaml` / `pnpm-workspace.yaml` | tracked | **是** | 依赖基线 |
| `tsconfig.json` / `next.config.ts` / `postcss.config.mjs` / `eslint.config.mjs` / `vitest.config.ts` | tracked | **是** | 构建/质量配置（next.config 无密钥） |
| `README.md` | tracked | **是** | 公开产品文档 |
| `CURRENT_ARCHITECTURE.md` | tracked | **是** | 公开架构文档 |
| `baidu_verify_codeva-r7wfvf611l.html`（小写） | tracked | **是** | 线上在用验证文件（§5） |
| `.env.example` | tracked | **是** | 占位模板（CHANGE_ME） |
| `.user.ini` | tracked | **否** → untrack | 服务器环境文件（§4） |
| `baidu_verify_codeva-r7wFVf611L.html`（大写） | tracked+已删 | **否** → 提交删除 | 死文件（§5） |
| `.well-known/` | untracked | **否** → gitignore | 运行期验证数据（§6） |
| 其余 24 份 tracked B 类文档 | tracked | **否**（若走路径②）| 内部审计/规划（§7） |
| 14 项 untracked 文档 | untracked | **否** | 内部文档（§7） |
| `.env` / `.env.local` / `.next` / `node_modules` / `*.tsbuildinfo` | 忽略 | **否** | gitignore 已覆盖（§9） |

---

## 9. 敏感信息检查（只查存在性，未打印任何值）

| 检查项 | 结果 |
|---|---|
| `.env` 是否在 git 历史中 | **从未提交**（`git log --all -- .env` 为空）✓ |
| 生产 DB 密码（DATABASE_URL 值）在历史中 | **不存在**（`git log -S <已知密码>` 为空）✓ |
| 私钥（RSA/EC/OPENSSH/DSA）在 tracked 文件 | **无** ✓ |
| API Key 模式（sk-/ghp_/AIza…）在 tracked 文件 | **无**（排除 pnpm-lock 误报）✓ |
| 测试凭据（test@siteintel.cc / testpass123）在代码文件 | **无**（仅可能出现在 md 文档，属公开测试账号）✓ |
| 游离 git 对象（fsck） | **空**（无孤儿 blob 含敏感数据）✓ |
| `.env` 磁盘权限 | `-rw-------`（600，www 属主）✓ |
| `.git/config` | 无 remote、无凭据 ✓ |
| 其他 env/conf/log/bak/pem 类 tracked | 仅 `.env.example`（占位）+ `.user.ini`（§4）✓ |

**结论：不存在真实敏感信息进入 Git 的路径。** `.env`（7 键：ADMIN_EMAILS/AUTH_SECRET/AUTO_DISCOVER/DATABASE_URL/DISCOVER_DAILY_CAP/NEXT_PUBLIC_SITE_URL/TELEGRAM_BOT_TOKEN）仅存于服务器磁盘 600 权限，gitignore 覆盖。

---

## 10. 结论与待执行清单（等待指令）

# READY WITH MANUAL ACTION

安全上**可创建 Duoniu-ai/siteintel**（无任何敏感信息泄露路径），但推送前需完成以下人工决策/操作（**本报告未执行任何一项**）：

**待执行操作（获批后）**：
1. `git rm --cached .user.ini` + `.gitignore` 追加 `.user.ini`（§4，零影响）
2. 提交 `baidu_verify_codeva-r7wFVf611L.html` 的删除（§5，磁盘已删，仅补提交）
3. `.gitignore` 追加 `.well-known/`（§6，零影响）
4. **用户决策**：文档公开策略三选一（§7.3 路径①/②/③）——决定 B 类 24 tracked + 14 untracked 文档与完整 66-commit 历史的公开范围
5. 决定 `DISCOVERY-CANDIDATE-FILTER-DESIGN.md` 工作树修改去留
6. （可选）公开仓库建议补充 LICENSE（当前无 LICENSE 文件，默认保留所有权利）与 README 首页模块描述更新（README 仍写「Hero + 五模块 + 管线」，落后于 Phase 3A 实际——仅文档更新，非代码）

**随后（另行指令）**：创建 `Duoniu-ai/siteintel`（gh CLI 已登录）→ 按 §8 FILESET 推送 → 建立 Production Baseline Tag → 回到 SEO Phase 1。

---

## 附：审计证据清单

- `.user.ini` 内容与 nginx 第 27 行拒绝规则（§4）
- `git diff HEAD:baidu_verify_大写 HEAD:baidu_verify_小写` = IDENTICAL（§5）
- 线上实测：`/baidu_verify_codeva-r7wFVf611L.html`→404、`/baidu_verify_codeva-r7wfvf611l.html`→200、`/.well-known/siteintel-verification.txt`→200（§5/§6）
- nginx：`location ~ \.well-known { allow all; root ... }`、`location ^~ /baidu_verify_ { root ... }`、`location ~ ^/(\.user.ini|...) { return 404; }`（§4-6）
- `git fsck --no-reflogs --unreachable` 空输出（§9）
- `git log --all -S <生产DB密码>` 空输出（§9）
