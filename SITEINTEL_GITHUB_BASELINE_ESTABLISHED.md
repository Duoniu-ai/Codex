# SITEINTEL GITHUB BASELINE ESTABLISHED

GitHub 生产基线建立报告 —— 2026-08-17 执行记录

## 结论

# GITHUB PRODUCTION BASELINE ESTABLISHED

---

## 1. 仓库

| 项 | 值 |
|---|---|
| 仓库 | https://github.com/Duoniu-ai/siteintel |
| 可见性 | **PRIVATE**（2026-08-17 创建时为 PUBLIC，随后按用户指令转为 Private；公开期间 0 fork / 0 star，未产生任何公开衍生） |
| 默认分支 | `main` |

## 2. 基线 Commit 与 Tag

| 项 | 值 |
|---|---|
| 初始 Commit Message | `chore: establish SiteIntel production baseline` |
| Commit Hash | `b52ec84`（本地 = 远端 `b52ec84c0341` 一致） |
| Tag | `production-baseline-2026-08-17`（**annotated**，`301bf34`，本地 = 远端一致，指向新基线 commit b52ec84） |
| 提交数 | 1（全新独立初始提交，不含服务器 66 个历史 commit） |

## 3. PUBLIC GITHUB FILESET（共 191 文件，与 PREP 报告 §8 一致）

- `src/`（应用源码，含 ssrf 防护、INDEX Gate、claim/report/website 路由）
- `prisma/`（schema + 7 migrations）
- `scripts/`（回填/运维脚本，--env-file 模式，无内嵌凭据）
- `package.json` / `pnpm-lock.yaml` / `pnpm-workspace.yaml` / 构建质量配置（next.config.ts / tsconfig.json / eslint.config.mjs / postcss.config.mjs / vitest.config.ts）
- `README.md`、`CURRENT_ARCHITECTURE.md`（A 类公开文档）
- `baidu_verify_codeva-r7wfvf611l.html`（小写，线上在用验证文件）
- `.env.example`（CHANGE_ME 占位模板）

## 4. 被排除的文件类型（不进 GitHub）

| 类型 | 说明 |
|---|---|
| `.user.ini` | 服务器运行环境文件（服务器端已 untrack，磁盘保留） |
| `.well-known/` | 运行期验证令牌数据（.gitignore，磁盘保留） |
| `baidu_verify_codeva-r7wFVf611L.html`（大写） | 死文件（线上 404），仅保留小写 |
| `docs/`（内部目录） + 23 份 root B 类 md | 内部审计/执行/SEO 报告 |
| `SiteIntel_V2_产品与技术架构总纲_最新版.md` | 内部产品规划 |
| `scripts/audit/*.json` | 生产 DB 实体 ID/时间窗口审计数据 |
| `scripts/safe-fetch-data-correction.mjs` | 单次事故修正脚本（含事故审计注释） |
| 14 项 untracked 内部文档 | 白盒审计、SEO 规划、导航站规划等 |

## 5. 敏感信息检查结果（推送前全量 + 推送后远端复核）

| 检查项 | 结果 |
|---|---|
| `.env`（生产，7 键）进仓库 | **否**（远端仅 `.env.example` 占位，CHANGE_ME）✓ |
| DATABASE_URL / AUTH_SECRET / TELEGRAM_BOT_TOKEN | 仅 `process.env.*` 变量引用，**无任何真实值** ✓ |
| API Key / Token / Password / Private Key / SSH Key | **无**（API keys 命中为产品 API-Key 管理功能代码 `src/app/api/keys/`）✓ |
| 服务器 IP `PRIVATE_SERVER` | **无**（staging 已净化，见 §6）✓ |
| 内部路径 `:PRIVATE_PORT` / `/www/PRIVATE_PATH` | **无** ✓ |
| `.user.ini` / `.well-known` / 大写 baidu | 远端**无** ✓ |
| 服务器原 git 历史 | 未推送、未改写、未触碰（见 §7）✓ |
| git fsck（预推审计） | 无游离对象 ✓ |

## 6. 两处有意调整（仅 staging 副本，生产零触碰）

1. **`src/lib/ssrf-guard.test.ts:48`**：测试夹具公网 IP `PRIVATE_SERVER` → `9.9.9.9`（Quad9 公网 DNS）。
   该 IP 为白名单测试数据（验证守卫不误伤公网 IP）；为满足「无服务器 IP」验证标准，对暂存副本做单行替换。
   **测试语义不变，服务器 src 未修改，1bdfcb4 未变。**
2. **排除 `scripts/safe-fetch-data-correction.mjs`**：单次事故修正脚本，注释含事故审计描述（调用方 IP 说明），归类 B 类内部执行产物，不进公开仓库。功能脚本（backfill/verify/seed 等）保留。

另：Windows 大小写不敏感文件系统导致两个仅大小写不同的 baidu 文件在暂存盘塌缩为一份（大写删除连带小写）；已通过 archive 单文件提取恢复小写版，远端确认仅小写存在。

## 7. 服务器原 Git 历史完整性

| 项 | 状态 |
|---|---|
| HEAD | 保持 `1bdfcb4` 未动 ✓ |
| 66 个 commit 历史 | 完整保留于服务器，未改写/未重置/未 force push ✓ |
| 服务器卫生处理（未提交，仅工作区） | `git rm --cached .user.ini`（staged）+ `.gitignore` 追加 `.user.ini`、`.well-known/`；大写 baidu 删除（磁盘已删） |
| 磁盘运行文件 | `.user.ini`、`.well-known/siteintel-verification.txt` 完好 ✓ |
| remote | 服务器无 remote（未配置、未添加）✓ |

## 8. 推送后 10 项验证（全部 PASS）

| # | 项 | 实测 |
|---|---|---|
| 1 | 仓库可访问 | PRIVATE ✓（创建时 PUBLIC，后转私有，见 §1） |
| 2 | 默认分支 | `main` ✓ |
| 3 | HEAD commit | `b52ec84`（本地=远端）✓ |
| 4 | Tag 存在且指向基线 commit | `production-baseline-2026-08-17` ✓ |
| 5 | 无 `.env` | 仅 `.env.example` ✓ |
| 6 | 无生产密码 | 全量扫描 CLEAN ✓ |
| 7 | 无内部审计报告 | 远端树无 SITEINTEL* 报告 / docs/ ✓ |
| 8 | 无服务器 IP / 内部路径泄露 | `PRIVATE_SERVER`、`:PRIVATE_PORT`、`www/PRIVATE_PATH` 全 CLEAN；ssrf-guard.test.ts 远端实测含 `9.9.9.9` ✓ |
| 9 | 小写 baidu 保留（大写不误入） | 远端仅 `baidu_verify_codeva-r7wfvf611l.html` ✓ |
| 10 | `.well-known` 未误删 / 线上功能未受影响 | 仓库无此目录；线上实测 `/.well-known/siteintel-verification.txt` **200**、小写 baidu **200**、`siteintel.cc` **200** ✓ |

远端 blob 数 = 191，与本地暂存一致（逐文件一致）。

## 9. 后续状态

- GitHub 基线已建立：`Duoniu-ai/siteintel` @ `b52ec84`（tag `production-baseline-2026-08-17`）
- 生产基线锚点不变：服务器 HEAD `1bdfcb4` = 线上构建 = GitHub `b52ec84` 的代码源
- 未执行：SEO Phase 1（等待下一步指令）
- 待用户决策（非本次范围）：README 首页模块描述更新、LICENSE 添加

---
*执行原则遵守：未修改服务器 git 历史、未 filter-repo、未 rebase、未 force push、未修改业务代码/SEO/数据库/生产环境、未重新部署。*
