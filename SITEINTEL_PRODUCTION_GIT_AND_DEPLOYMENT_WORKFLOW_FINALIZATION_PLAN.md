# SITEINTEL PRODUCTION GIT & DEPLOYMENT WORKFLOW FINALIZATION PLAN

- **日期**: 2026-08-17
- **性质**: 只读审计 + 未来唯一正式工作流设计（**未执行任何修改**）
- **前置成果**: Git History Unification（方案 B）已完成——生产 66 commits 真实历史已归档至 GitHub `legacy-prod-history` + archive tag + 服务器 tag + bundle（四重冗余）
- **本阶段状态快照**: 与全部历史报告**零不一致**（服务 active / NRestarts=0 / 3003 200 / 工作区 sitemap.ts hash 0c4594a6 = c74dfd1 CRLF 版未变 / GitHub 6 refs 逐字正确）

---

## A. 当前状态（2026-08-17 快照）

| 层面 | 值 | 状态 |
|---|---|---|
| GitHub main | `c74dfd1` | ✅ 正式开发主线 |
| GitHub seo/phase-1-p0 | `c74dfd1` | ✅ 已 merge，保留 |
| GitHub legacy-prod-history | `1bdfcb4`（66 commits） | ✅ 只读归档 |
| GitHub 3 tags | baseline→b52ec84 / seo-p0→c74dfd1 / archive→1bdfcb4 | ✅ 全部在位 |
| 生产 git master | `1bdfcb4` | ⚠️ 与工作区内容分层 |
| 生产工作区 | 30 项未提交 = 14 P0（c74dfd1 内容 CRLF）+ 16 卫生 | ⚠️ 分层（预期） |
| 生产运行代码 | .next 产物 = c74dfd1 构建 | ✅ 与线上行为一致 |
| 生产服务 | active / NRestarts=0 / 3003 robots+home 200 | ✅ |
| f1 worktree | 1bdfcb4 + 4 项未提交 | ✅ 独立未动 |
| staging + siteintel_staging_ro | 保留 | ✅ |

## B. 已完成历史归档（方案 B 成果，不再改动）

生产 66 commits（2026-08-14~16）以原对象原 hash 四重保留：GitHub `legacy-prod-history` 分支 → GitHub `production-history-archive-20260817` tag → 服务器 `production-legacy-1bdfcb4` tag → `/www/backup/siteintel-prod-history-20260817.bundle`（824KB，verify OK）。**该历史永久只读，不再作为开发来源。**

## C. 当前生产漂移（精确定义）

```
逻辑主线:  GitHub main = c74dfd1
生产指针:  master = 1bdfcb4（本地独立历史 tip，已归档）
生产内容:  工作区 src = c74dfd1 内容（CRLF，hash 实证 3/3 + 本次 0c4594a6 复证）
运行产物:  .next = c74dfd1 构建
```

漂移本质 = **指针层与内容层分离**（1bdfcb4 vs c74dfd1），根因 = 上次部署采用文件直传覆盖（git 零写入）+ 历史分叉（基线孤儿根）。**内容层与实际运行已完全一致**（线上验收 PASS），**漂移仅在 git 指针层**。

后果（若不修复）：
1. `git status` 永久显示 14 个 P0 文件为"未提交修改"——**无法区分真修改与部署层差异**，任何 git 审计失真
2. 生产无法用 `git diff <tag>` 验证部署内容（index 基于 1bdfcb4）
3. 未来一切部署/回滚（bundle 通道）的"HEAD 验证门禁"失去意义（HEAD 永远是错的）

## D. 是否需要修复 Git HEAD — 裁定：**是，执行 `update-ref master → c74dfd1`**

### D.1 论证

| 选项 | 后果 | 裁定 |
|---|---|---|
| 不修复 | git 指针层永久失真，门禁失效，未来每次部署都带着一个"已知错误" | ✗ 否决 |
| 修复（update-ref） | 指针 = 内容 = 主线，git 成为可信验证工具 | **✓ 执行**（待批准） |

### D.2 执行后影响分析（逐项）

| 对象 | 影响 | 说明 |
|---|---|---|
| 30 项未提交 | → 14 P0 项消失（由 checkout 提供）；**剩 16 卫生项原样** | `git checkout -- src/` 只覆盖 src 子树 |
| 16 项卫生文件 | **零影响** | 不在 checkout 范围（.gitignore/文档/.user.ini 删除等） |
| 14 项 P0 文件 | 被覆盖为 **LF 版**（内容不变，行尾归一） | 已 hash 实证内容 = c74dfd1；JS/JSX 行尾语义中性；`.next` 产物不变 → **运行零影响** |
| f1 worktree | **零影响** | 独立分支/独立 worktree（1bdfcb4 仍被 f1 引用，对象永久可达） |
| legacy 历史 | **零影响** | GitHub 分支 + tag + bundle 已独立保存 |
| 现有 tags | **零影响** | update-ref 仅移动 `refs/heads/master` 一个 ref |
| 生产服务 | **零影响** | 不 restart、不 touch .next/node_modules/.env |

### D.3 最安全方法（唯一推荐路径）

```
[前置] 服务 active、3003 200、历史保护在位（production-legacy-1bdfcb4 tag + bundle，均已完成）
[1] 本地: git bundle create /tmp/siteintel-align.bundle main        ← 含 c74dfd1 全对象
[2] 本地: git bundle verify /tmp/siteintel-align.bundle              ← 必须 ok
[3] scp → 服务器 /tmp/（字节数比对）
[4] 服务器: git fetch /tmp/siteintel-align.bundle 'refs/heads/*:refs/remotes/bundle/*'
[5] 服务器: git cat-file -t c74dfd1 → commit                        ← 对象入库验证
[6] 服务器: git update-ref refs/heads/master c74dfd1                ← 精确指针（非 reset）
[7] 服务器: git checkout -- src/                                    ← 工作区对齐（LF 化）
[8] 验证门禁（§D.4）
```

### D.4 验证方式

```
git rev-parse master = c74dfd1
git rev-parse HEAD   = c74dfd1
git status --porcelain = 仅 16 项卫生（与 §1.3 白名单逐一比对，无新增）
git diff --exit-code -- src/ → rc=0
git rev-parse f1-candidate-filter = 1bdfcb4（未动）
git rev-list -n1 production-legacy-1bdfcb4 = 1bdfcb4（tag 未动）
systemctl is-active siteintel = active（从未 restart）
curl 3003 robots/home = 200
线上 https://www.siteintel.cc /sitemap.xml /website/wordpress.org = 200
```

### D.5 完整回滚方案

```
[1] git update-ref refs/heads/master 1bdfcb4        ← 对称一步（指针回原位）
[2] 工作区：接受 LF（内容与 1bdfcb4 仅行尾差异，git diff -w 干净、JS 语义等价）；
    或如执行前做了 tar 备份 src/ 则恢复 CRLF 原样
[3] 验证：rev-parse master = 1bdfcb4；status 恢复 30 项原清单
[4] 服务：从未 restart，无需恢复
```

**回滚零风险**：git 层对称可逆、服务零触碰、历史四重冗余不受影响。

---

## E. 未来标准开发流程（唯一正式 Git 工作流）

```
feature branch（从 main 创建，命名 <type>/<feature>）
   ↓ 本地开发（force-dynamic 页面、vitest 单测、typecheck）
   ↓ 本地验证: pnpm test && pnpm typecheck && pnpm build
   ↓ code review（PR；含 secret scan——.env 模式零命中为必须）
   ↓ acceptance（P0/P1 gate 逐项验收，QA 清单）
   ↓ merge main（FF 优先；禁止 force push；禁止改写历史）
   ↓ 创建 release tag（annotated: production-<feature>-<YYYYMMDD>）
   ↓ 部署流程（§G）
   ↓ production Git HEAD verification（§G 部署后门禁）
```

**铁律**：
- 唯一分支起点 = GitHub `main`（legacy-prod-history 永久只读）
- 唯一合并终点 = GitHub `main`（squash 或 FF，提交信息含功能+验证摘要）
- 每个正式发布必须带 release tag（annotated，指向 main HEAD）
- 禁止：直接向 main 提交、force push、改写已推送历史

## F. 未来标准 staging 流程（既有资产复用）

```
本地/CI: pnpm build && pnpm test
部署至 staging（bundle 通道）: bundle → scp → /www/wwwroot/siteintel.cc-staging fetch/checkout → pnpm build → systemctl restart siteintel-staging
验证: 127.0.0.1:3004 六状态矩阵（404/processing/no-data/report/Gate）+ canonical/metadata/sitemap 污染检查
DB: 只读角色 siteintel_staging_ro（default_transaction_read_only=on 实证写拒）——staging 读生产数据，零写风险
通过 → 进入 code review / acceptance
```

- staging 目录/systemd/只读角色**永久保留**为标准测试环境
- staging 永远落后于 main（部署明确版本），不接受随机代码

## G. 未来标准 Production Release 流程

### G.1 部署通道比较（唯一推荐裁定）

| 方案 | 机制 | 凭据落服务器 | 安全面 | 可行性（现状） | 裁定 |
|---|---|---|---|---|---|
| **A. Git bundle** | 本地打包 → scp → 服务器 fetch | **无** | 不变 | ✅ 已实证（本轮 fetch 66 commits + bundle verify） | **✓ 唯一推荐** |
| B. GitHub SSH deploy key | 私钥放服务器 → 直接 fetch | 有（私钥文件） | 扩大（服务器被攻陷=仓库写权限） | 需新配置 | ✗ 暂缓（可作远期） |
| C. fine-grained token | HTTPS + token 存服务器 env/文件 | 有（token 落盘） | 扩大 + 泄漏风险 | 需新配置 | ✗ 不推荐 |
| D. 其他（本地 push 生产 / CI runner） | ssh path remote / CI 产物推送 | 有 / 无 | 需改生产配置 denyCurrentBranch 或引入 CI | 改动大 | ✗ 不推荐 |

**推荐 A 的理由**：零新增凭据（服务器安全面不变）、零配置变更、与既有操作习惯一致（本轮历史导入已实证 bundle 全链路）、bundle 可自校验（verify）+ 字节比对双重完整性。**将流程脚本化**（`deploy.sh`：bundle create → verify → scp → 远程 fetch/checkout/build/restart/三查验证），单命令完成、输出留档。

### G.2 部署前门禁（全部必须 PASS，任一 FAIL 停止）

| # | 检查 | 命令/标准 |
|---|---|---|
| 1 | GitHub main commit | `git ls-remote origin refs/heads/main` = 目标 release commit |
| 2 | release tag | `git ls-remote origin refs/tags/production-<feature>-<date>` 存在且 `rev-list -n1` = main |
| 3 | production target commit | 明确 = tag 指向 commit（不是 "HEAD 上的什么"） |
| 4 | working tree（生产） | `git status --porcelain` = 仅 16 项卫生白名单（卫生项收口后 = 0） |
| 5 | diff（部署源） | 本地 `git diff --stat <prev-tag> <release-tag>` 人工审阅范围 |
| 6 | build | `pnpm build` exit 0（本地或生产构建；FAIL → 不 restart 不继续） |
| 7 | DB migration | `git diff <prev-tag> <release-tag> -- prisma/`：零 diff = 无需迁移；有 migration = 先备份 + 明确批准 |
| 8 | service status | `systemctl is-active siteintel` = active（基线） |

### G.3 部署后验证（全部必须 PASS）

| # | 检查 | 命令/标准 |
|---|---|---|
| 1 | production Git HEAD | `git rev-parse HEAD` = release commit |
| 2 | deployed file hash/content | `git hash-object <部署文件>` == `git show <tag>:<file> \| git hash-object --stdin`（LF 统一后直接相等；抽查 ≥3） |
| 3 | service active | `systemctl is-active siteintel` + `systemctl show -p NRestarts` = 0 |
| 4 | restart count | 部署仅允许 1 次 restart，NRestarts 不增长（无 crash loop） |
| 5 | HTTP health | 3003 robots/home 200 + 线上 https://www.siteintel.cc 200 |
| 6 | sitemap | 目标页面集合正确（含/不含按 Gate 判定），无污染（example/virtualearth/CDN 0 命中） |
| 7 | robots | 抽查目标页 robots 与 release 预期一致（index/noindex 对照） |
| 8 | release tag 对应关系 | `git rev-parse HEAD` == `git rev-list -n1 production-<feature>-<date>` |

## H. 正式回滚机制（与部署同通道，对称）

```
release tag（当前）→ 目标 = 上一个 release tag（或 pre-deploy 锚点 tag）
[1] 本地: bundle create <prev-tag> → verify → scp
[2] 服务器: fetch bundle → git checkout <prev-tag> -- <变更文件>（或整树）
[3] pnpm build（FAIL → 停止，不 restart）
[4] systemctl restart siteintel
[5] 验证（§G.3 同款：HEAD/服务/HTTP/sitemap/robots）
[6] 回滚记录：日期/原因/前后 tag/验证结果 → 报告
```

**触发条件**：部署后验证任一 FAIL / 线上异常（Soft 404 复现、robots↔sitemap 冲突、大面积 500、服务 crash loop）。**回滚是正式流程而非应急 hack**——每次发布前确认上一 release tag 在位（`git ls-remote` 双端）。

## I. 永久禁止的操作（明确清单）

| # | 禁止项 | 原因 |
|---|---|---|
| 1 | **archive/scp 直接覆盖生产 src 而不更新 Git HEAD** | 本次漂移的根源；导致指针-内容分层永久化 |
| 2 | force push / 改写 GitHub main 历史 | 不可逆；破坏 tag/发布语义 |
| 3 | 删除/移动 production-baseline / production-seo-phase1-p0 / production-history-archive 任一 tag | 发布与归档锚点 |
| 4 | 在 legacy-prod-history 上开发/merge | 只读归档 |
| 5 | 生产仓库手工 git commit（除受控卫生项收口） | 绕过来源管理 |
| 6 | 生产直接编辑 src/ 文件 | 未经版本控制变更 |
| 7 | reset --hard / clean -fd / rebase / filter-repo（生产） | 数据销毁风险 |
| 8 | 部署时修改 .env / 数据库 / nginx（未经独立批准） | 安全边界 |
| 9 | 删除 staging / siteintel_staging_ro | 标准测试环境资产 |
| 10 | 未经 §G.2 门禁发布 | 流程完整性 |

## J. 推荐最终架构（完整图景）

```
                        ┌──────────────────────────────────────────────┐
                        │  GitHub Duoniu-ai/siteintel（Private）         │
  开发主线               │  main = c74dfd1（唯一开发起点/合并终点）        │
  feature/<type>/<name> │  ├── production-baseline-2026-08-17 → b52ec84 │
      │ PR + review     │  ├── production-seo-phase1-p0-2026-08-17      │
      ↓                 │  └── 未来: production-<feature>-<date> tags    │
  merge main（FF/squash）│  只读归档: legacy-prod-history（66 commits）+  │
      ↓                 │  production-history-archive-20260817          │
  release tag（annotated）└──────────────────────────────────────────────┘
      ↓                                     │
  本地 deploy.sh（bundle 通道）◄─────────────┘
      ↓                                     │
  生产服务器 154.204.176.66                  │ bundle verify + scp 字节比对
  ├── master（对齐后 = release commit）       │
  ├── src/ 工作区 = tag 内容（LF，diff 干净）  │
  ├── .next 产物 = build 验证通过后 restart   │
  ├── siteintel.service（active, NRestarts=0）│
  ├── f1 worktree（独立，永久不动）           │
  └── staging 3004 + siteintel_staging_ro    │
       （标准测试环境，永久保留）              │
  ┌───────────────────────────────────────────┘
  门禁: §G.2 部署前 8 项 + §G.3 部署后 8 项 + §H 回滚通道
```

**一致性承诺（唯一状态方程）**：
```
GitHub main == release tag target == Production git HEAD
           == Production working tree == Production running code
```
漂移消除机制 = 部署唯一源（GitHub）+ 部署唯一通道（bundle）+ 部署唯一流程（§G 门禁）+ 回滚对称（§H）。

---

## 结论与待批准项

1. **本阶段仅审计与设计，零修改执行** ✓（快照与历史报告零不一致）
2. **D 项裁定：需要执行 `update-ref master → c74dfd1`**（方法 §D.3、验证 §D.4、回滚 §D.5；前置条件已全部满足）
3. **部署通道唯一推荐 = Git bundle（方案 A）**，脚本化 deploy.sh
4. 待批准：① D 项指针对齐执行 ② deploy.sh 编写与启用 ③ 卫生项收口（阶段 II，可另行排期）

---
**完成，停止。等待批准。** 未执行：update-ref / reset / checkout 覆盖 / restart / deploy / push / merge / DB / nginx / .env / staging 删除 / siteintel_staging_ro 删除 / SEO Phase 2。
