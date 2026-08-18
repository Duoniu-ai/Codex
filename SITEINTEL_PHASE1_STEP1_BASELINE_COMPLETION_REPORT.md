# SiteIntel Phase 1 — Step 1 工程基线统一完成报告

> 日期：2026-08-18 ｜ 范围：P0-2（multi-source 合入 main + 测试 + staging + 生产发布 + 本地同步 + deploy.sh + tag）
> 授权：用户逐项授权（Step 1 only）。**未执行**：probe/safe-fetch/api/report/SSE/schema/Job System/Event-Signal/Fact 等任何后续项。
> 结论：✅ **完成（PASS）**

---

## 1. 合并前 main commit

`c74dfd17dc559b4ba18812aef24a98ea3d72c3b8`（SEO Phase 1 P0，2026-08-17）

## 2. multi-source commit

`06f2486264e6943f2f75f9c2af57ffe4188c99a0`（多源引擎 Phase 0 3 commits：`e32bd93` → `15de2d9` → `06f2486`）

## 3. 合并后 main commit

- FF 合并后：main = `06f2486264e6943f2f75f9c2af57ffe4188c99a0`
- 最终 main（含 deploy.sh 正式 commit）：`f89ad892c5544252f927e5f886f4ecf46fe3dfd9`

合并前复验（GitHub API + 本地克隆）：ahead 3 / behind 0；16 个文件与审计清单逐字一致（+1322/−65）；无未审计新冲突。

## 4. 生产发布 commit

- 生产构建产物（.next，BUILD_ID `C8M1pHtHv8m-QD4m4Xghh`）由 **06f2486** 代码构建。
- 生产 git HEAD/index 最终对齐 **f89ad89**（= main = tag；该 commit 相对 06f2486 仅新增非运行时的 deploy.sh）。
- 部署前保护 tag：`pre-deploy-step1-20260818`（→ c74dfd1）。

## 5. staging 验证结果（PASS）

| 项 | 结果 |
|---|---|
| 代码 | 06f2486 归档解包至 `/www/wwwroot/siteintel.cc-staging`，probe/score/sources/migration 齐备 |
| 构建 | `pnpm install --frozen-lockfile` + `pnpm prisma generate` + `pnpm build` 成功 |
| 服务 | `siteintel-staging` active（127.0.0.1:3004） |
| HTTP | `/` `/robots.txt` `/sitemap.xml` `/website/wordpress.org` `/website/example.com` 全 200 |
| 新代码生效 | 启动日志：`[discovery] collector armed: tick 15000ms, daily cap 20, seed batch 50/day, retry backoff 1h→168h (max 7 failures)`；`.next/server` 含 cloudflare_radar/probeStatus 编译产物 |
| Seed 源 | `[discovery] seed cloudflare_radar import failed ... 400`（staging 无 token，按设计降级） |
| Analyze 端点 | POST 500 = 只读角色硬拦截（`cannot execute UPDATE in a read-only transaction`）——隔离正确，staging 零污染 |
| 调度器 | monitor / search-sync / discovery 全部武装（写路径被只读 DB 拒绝，无生产影响） |

## 6. production 验证结果（PASS）

| 项 | 结果 |
|---|---|
| 服务 | `siteintel` active，NRestarts=0，新 PID 777669 |
| 本地 HTTP | `/` `/robots.txt` `/sitemap.xml` `/website/wordpress.org` `/website/example.com` `/api/report/example.com` `/api/tools/dns` 全 200 |
| 公网 HTTP | `https://siteintel.cc/` `/robots.txt` `/sitemap.xml` `/website/wordpress.org` `/api/report/example.com` 全 200；`/api/v1/report` 无 Key → 401（v1 鉴权正常） |
| 网站分析 | POST `/api/analyze` example.com → 新调查 `cmsy5ha680001s51x3m4zq242`，**6s 内 completed**；报告页 200 |
| Discovery | 重启后 `collector armed: tick 15000ms, daily cap 50, seed batch 50/day`；seed 窗口 `fetched 43, +0 new (7/50 today)`（去重正常）；预算记账 50/50 |
| Worker/Scheduler | monitor（30m）/ search-sync（6h）/ discovery（15s）三调度器全部武装，0 fatal |
| 数据库 | 零 schema 变更（migration 已于 08-17 部署）；新列可用性以运行证据确认（probe/score/seed 查询全部成功） |
| Git | 生产 HEAD = master = `f89ad89`；`git diff HEAD -- src/ prisma/ deploy.sh` 零差异；服务不受 HEAD 对齐影响 |

## 7. 完整测试结果

合并树（06f2486）本地完整运行：**17 个测试文件，258/258 通过**（含新增 probe/score/cloudflareRadar/collector 测试，domain 18 例、ssrf-guard 63 例、safe-fetch 17 例）。

## 8. deploy.sh 内容及用途

新增 [deploy.sh](C:/Users/deepo/siteintel/deploy.sh)（1.7KB，已入库），服务器端最小部署脚本：

- 输入：目标 commit（默认 `origin/main`；支持 bundle 预取）。
- 步骤（仅真实部署所需）：`git checkout <commit> -- .`（应用 tracked 文件，保留 untracked 的 .env/node_modules/.next/docs）→ `pnpm install --frozen-lockfile` → `pnpm prisma generate` → `pnpm build` → `systemctl restart siteintel` → 3003 robots 冒烟。
- 明确不包含：DB migration（schema 变更另行授权执行）、CI/CD、部署平台、Docker/systemd/nginx 重写。

用途：**让 main 可明确、可重复地构建/部署生产版本**（此前无任何部署脚本，生产无法从 main 重建）。

## 9. 本地工作副本最终 HEAD

- `C:\Users\deepo\siteintel`：master = **f89ad89**（upstream = origin/main），工作树含 06f2486 全量多源代码 + deploy.sh。
- 旧 master（1bdfcb4）已重命名为 **`legacy-prod-history`** 分支保留，未删除。
- 未提交文档（33+ 项）原样保留；两份此前被跟踪的本地修改文档（DISCOVERY-CANDIDATE-FILTER-DESIGN.md、SiteIntel_V2 总纲）已备份-切换-还原为 untracked，内容未动。

## 10. multi-source 分支状态

**保留**。GitHub `multi-source/phase0-cfradar` 仍存在，HEAD = `06f2486`，未删除（已合入 main，分支留作历史）。

## 11. Git tag

- `production-phase1-baseline-2026-08-18`（annotated）→ **f89ad89**，已推送 GitHub。
- 命名沿用项目既有规范（`production-<feature>-<date>`，对照 `production-seo-phase1-p0-2026-08-17`）。
- 服务器保护 tag `pre-deploy-step1-20260818` → c74dfd1（回滚锚点，保留）。

## 12. 是否能够从 main 重建生产版本

**是（CONFIRMED）**。main（f89ad89）包含生产运行的 06f2486 全量代码 + deploy.sh；生产 git HEAD = main = tag；执行 `bash deploy.sh f89ad89`（或 origin/main）即可重建当前生产版本。此前“生产运行代码无法从 main 重建”的问题已消除。

## 13. 发现的问题

1. staging `.env` 的 `AUTO_DISCOVER` 实际为开启态（启动日志 collector armed；seed 因无 token 400 降级；probe 写被只读角色拒绝）——staging 调度器在跑但零污染；属 staging 配置卫生，已记录，不在本步范围。
2. 服务器 pnpm 10 对生命周期脚本有提示，`prisma generate` 需显式执行——本步已显式执行；deploy.sh 已固化该步骤。
3. 直接以命令行提取生产 DB 密码做 schema 查询未成功（URL 格式差异）——未继续尝试，避免凭据暴露；以运行证据确认 schema 新列可用。
4. sitemap 从 255 → 256（+1，正常数据增长，非回归）。

## 14. 未解决的问题

1. `tsconfig.json` M（既有卫生项，diff -w 干净）——阶段 II 收口决策。
2. 49 项未跟踪卫生文件（docs/.env.bak/scripts 等）处置待决策。
3. staging 调度器配置收口（AUTO_DISCOVER 显式 false）+ staging 删除/只读角色清理（原等待授权项，未触碰）。
4. P0-1（probe）、P0-3（report API）、P0-4（SSE）——未开始，等待逐项授权。

## 15. 回滚方式

- **GitHub main**：`git push origin c74dfd17dc559b4ba18812aef24a98ea3d72c3b8:main`（或恢复既有 tag `production-seo-phase1-p0-2026-08-17`）；multi-source 分支仍保留。
- **生产代码**：服务器 `git checkout c74dfd1 -- <16 文件>` + `pnpm build` + `systemctl restart siteintel`；或 `git checkout pre-deploy-step1-20260818 -- .` 还原部署前状态；完整备份 `/www/backup/siteintel-prod-history-20260817.bundle` 在位。
- **生产 HEAD**：`git update-ref refs/heads/master c74dfd1` + `git read-tree --reset c74dfd1`。
- **数据**：本步零 schema / 零结构性变更；唯一业务写入为验证用的一次 example.com 分析（正常产品行为，无需回滚）。
- **本地镜像**：`git switch legacy-prod-history` 回到旧基线（未删除）。

---

**完成，停止。** 未自动开始 P0-1；等待下一步授权。
