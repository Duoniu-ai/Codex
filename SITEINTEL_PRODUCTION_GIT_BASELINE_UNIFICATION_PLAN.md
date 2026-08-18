# SITEINTEL 生产 Git 基线统一 — 审计报告与修复方案设计

- **日期**: 2026-08-17
- **性质**: 只读审计 + 方案设计（**未执行任何修复/修改**）
- **审计对象**: 生产服务器 `154.204.176.66:613` → `/www/wwwroot/siteintel.cc`（siteintel.service, 3003）
- **目标**: 设计最安全方案，使未来达到 `GitHub main = Production Git HEAD = Production working tree = Production running code`

---

## 0. 审计执行记录（全部只读命令）

| # | 命令（服务器或本地） | 目的 |
|---|---|---|
| 1 | `git rev-parse HEAD` / `git branch --show-current` / `git branch -a` / `git tag --list` | 基本拓扑 |
| 2 | `git status --short` / `git status --porcelain \| wc -l` | 未提交清单（30 项） |
| 3 | `git worktree list` | 发现 **/root/siteintel-f1** 第二 worktree |
| 4 | `git remote -v` | 确认**无 remote** |
| 5 | `git ls-files \| grep -E "\.env|\.next"` / `git check-ignore -v .env .next` | `.env`/`.next` 未跟踪且已 ignore |
| 6 | `git diff --stat` / `git diff --cached --stat` | 未暂存/已暂存 diff 统计 |
| 7 | `file` 三连（工作区 / index / HEAD） | 行尾实测 |
| 8 | `git hash-object`（服务器工作区） vs 本地 blob 模拟 CRLF | **内容一致性 3/3 匹配** |
| 9 | `git cat-file -t c74dfd1`（服务器） | 对象库无 c74dfd1 |
| 10 | `git log --oneline`（双方） / `git rev-parse b52ec84^`（本地） | **b52ec84 = 孤儿 root commit** |
| 11 | `ls ~/.ssh/`（服务器） | 无 GitHub 部署密钥 |
| 12 | `git status --ignored --porcelain` | 运行时文件 ignore 完整性 |
| 13 | `git config --get user.name/email` | 生产 identity 已配置 |
| 14 | `git rev-parse f1-candidate-filter` / f1 worktree status | f1 分支 = 1bdfcb4，4 项未提交 |

**禁止事项零违反**: 未执行 reset --hard / clean -fd / 任何写操作 / .env 修改 / DB 操作 / nginx / staging 触碰 / restart。

---

## 1. 当前生产 Git 状态完整解释

### 1.1 拓扑

```
生产服务器 154.204.176.66
├── 主仓库 /www/wwwroot/siteintel.cc   [master @ 1bdfcb4]   ← 运行代码目录
│     ├── 无 remote（git remote -v 为空）
│     ├── 对象库：无 c74dfd1（git cat-file 证实）
│     ├── 无 GitHub 凭据（~/.ssh 仅 id_ed25519 服务器自身密钥）
│     └── 30 项未提交（§1.3 分类）
└── worktree /root/siteintel-f1        [f1-candidate-filter @ 1bdfcb4]
      └── 4 项未提交（collector.ts M / domain.ts M / candidate-filter.ts ?? / f1-dry-run.test.ts ??）
          = 08-16 Discovery F1 进行中的工作现场
```

### 1.2 三层分离的本质

| 层面 | 当前值 | 说明 |
|---|---|---|
| GitHub main | `c74dfd1` | 基线孤儿快照 b52ec84 → SEO P0 c74dfd1（tag production-seo-phase1-p0-2026-08-17） |
| 生产 git 历史（master） | `1bdfcb4` | **与 GitHub 历史无血缘**：b52ec84 是 root commit（基线重新打包），生产本地链（9bdf6b2→…→1bdfcb4）从未进入 GitHub |
| 生产文件内容（工作区） | = c74dfd1 内容（CRLF） | 部署为文件覆盖（archive/直传），git 层零写入 |

### 1.3 30 项未提交文件分类（审计实证）

**A 类 — SEO Phase 1 P0 部署文件（14 项）→ 应纳入 Git（对齐后由 c74dfd1 提供）**

| 状态 | 文件 | 内容验证 |
|---|---|---|
| M ×7 | `src/app/bulk/page.tsx` `src/app/docs/api/page.tsx` `src/app/layout.tsx` `src/app/report/page.tsx` `src/app/sitemap.ts` `src/app/website/[domain]/page.tsx` `src/lib/i18n.ts` | hash 模拟 CRLF 后 = c74dfd1 blob（3/3 匹配实证） |
| ?? ×7 | `src/app/website/[domain]/not-found.tsx` `src/lib/seo/index-gate.ts` `index-gate.test.ts` `low-value-domain.ts` `low-value-domain.test.ts` `report-state.ts` `report-state.test.ts` | 同上 |

**B 类 — 原有生产卫生文件（16 项）→ 决策项（§10 决策表）**

| 状态 | 文件 | 性质 |
|---|---|---|
| M | `.gitignore`（+4 行：`.user.ini`、`.well-known/`） | 基线期 ignore 补充，有价值 |
| D（staged） | `.user.ini` | 从版本控制移除服务器配置文件（配套 ignore） |
| D（unstaged） | `baidu_verify_codeva-r7wFVf611L.html`（大写） | 百度验证文件工作区已删；**小写版仍在 index**（大小写双文件遗留） |
| M | `DISCOVERY-CANDIDATE-FILTER-DESIGN.md`（+41 行） | 08-16 F1 会话指引追加 |
| ?? ×12 | `SITEINTEL-2.0-PRODUCT-UPGRADE-BLUEPRINT.md`、`SITEINTEL_SEO_ACCEPTANCE_CHECKLIST.md`、`SITEINTEL_SEO_EXECUTION_ROADMAP.md`、`SITEINTEL_SEO_MASTER_PLAN_V2.md`、`SITEINTEL_TECHNICAL_SEO_IMPLEMENTATION.md`、`SiteIntel 2.0 Phase 3C 执行任务书.md`、`docs/SITEINTEL-FULL-SITE-ANALYSIS-AUDIT.md`、`docs/SiteIntel 白盒审计报告执行任务书.md`、`docs/SiteIntel-Claude-Code-执行指南（互补关系）.md`、`docs/navigation/`、`docs/siteintel-full-site-analysis-audit.md`、`siteintel-full-site-analysis.md` | 规划/审计文档（与运行无关） |

**C 类 — 运行时生成文件 → 已全部 ignore，保持在 Git 工作区外 ✓**

`git status --ignored` 实证：`.env`、`.next/`、`.user.ini`、`.well-known/`、`next-env.d.ts`、`node_modules/`、`tsconfig.tsbuildinfo` —— 全部被 `.gitignore` 覆盖，**不会进入 git，也不受任何 git 操作影响**。

**D 类 — 不应覆盖的文件**

| 文件 | 保护状态 |
|---|---|
| `.env`（生产凭据：DATABASE_URL/AUTH_SECRET/TELEGRAM_BOT_TOKEN） | `.gitignore:21` 已 ignore，**未被 git 跟踪**（`git ls-files` 仅 `.env.example`）✓ |
| `/root/siteintel-f1` worktree（F1 工作现场） | 独立 worktree + 独立分支，任何主仓库操作不触碰 ✓ |

**E 类 — 可能影响生产的配置文件**

审计结果：**无 E 类问题**。`next.config.*`、`package.json`、`prisma/` 等配置文件全部 clean（不在未提交清单中）；`.env` 由 ignore 保护（见 D 类）。唯一被 git 跟踪的"服务器相关"文件为 `.env.example`（模板，无真实凭据）与 `.user.ini`（B 类已计划移除）。

### 1.4 行尾事实（关键）

| 层面 | 行尾 | 实证 |
|---|---|---|
| GitHub blob（c74dfd1） | **LF** | 本地 `git show … \| file -` = UTF-8 无 CRLF 标记 |
| 服务器工作区（运行文件） | **CRLF** | `file src/app/bulk/page.tsx` = "with CRLF line terminators" |
| 服务器 index / HEAD（1bdfcb4） | **LF** | `git show :…` / `git show 1bdfcb4:…` = ASCII |

**内容一致性证明**：服务器工作区 hash（CRLF）== 本地 c74dfd1 blob 经 `perl -pe 's/\n/\r\n/g'` 模拟后的 hash，抽查 3 文件 **3/3 匹配** → **服务器工作区 = c74dfd1 内容，唯一差异为行尾**。CRLF 来源：部署文件来自本地 Windows 工作区（`core.autocrlf=true` checkout 为 CRLF），覆盖服务器后保留。

---

## 2. 为什么 HEAD 与实际文件不一致

因果链（4 步）：

1. **GitHub 基线是孤儿快照**：b52ec84（root commit）是将生产代码打包为基线，生产本地完整历史（1bdfcb4 链）从未推入 GitHub → 两个历史体系并行。
2. **部署采用文件覆盖**：SEO P0 部署走 archive/文件直传 + cmp 校验（方案 B），全程无 git 写操作 → 生产 git 指针停留在 1bdfcb4。
3. **行尾二次差异**：部署文件为本地 CRLF 工作区产物，与仓库 LF blob 行尾不同 → 即使指针对齐，`git diff` 也会因行尾显示全量差异（`git diff -w` 才能看到真实内容 = 对齐后恰好干净）。
4. **无同步通道**：服务器无 remote、无 GitHub 凭据、对象库无 c74dfd1 → 指针无法本地移动（update-ref 需要对象存在）。

**结论：不一致是"部署模型 + 历史分叉 + 通道缺失"三重原因，非意外改动。**

---

## 3. 哪些文件可以安全纳入 Git

| 类别 | 文件 | 纳入方式 | 安全性 |
|---|---|---|---|
| A（14） | P0 部署文件 | 对齐 c74dfd1 后由 checkout 提供（**不单独提交**） | 内容已 hash 实证 = c74dfd1 |
| B 推荐纳入 | `.gitignore` M | 阶段 II commit（§10） | 纯仓库配置，+4 行 ignore |
| B 推荐纳入 | `.user.ini` staged D | 阶段 II commit | 服务器配置出仓，配套 ignore |
| B 推荐纳入 | `DISCOVERY-CANDIDATE-FILTER-DESIGN.md` M | 阶段 II commit | 设计文档 |
| B 推荐纳入 | 12 个 untracked 文档 | 阶段 II commit（入库前人工确认无密钥/敏感凭据——预期无） | 规划文档 |
| B 用户决策 | `baidu_verify_codeva-r7wFVf611L.html` D | 决策：①纳入删除（弃用百度验证）②恢复文件 ③与大小写副本合并 | 仅影响百度站长验证文件存在性 |

**阶段 I（本次统一）不提交任何文件**——只对齐指针与工作区（14 A 类 clean），16 项 B 类保持原位不动。

---

## 4. 哪些文件必须保留在 Git 工作区外

| 文件 | 原因 | 保护机制 |
|---|---|---|
| `.env` | 生产全部凭据 | `.gitignore:21` + 未跟踪 ✓（永不入库） |
| `.next/` `node_modules/` `tsconfig.tsbuildinfo` `next-env.d.ts` | 运行时/依赖产物 | `.gitignore:5` 等 ✓ |
| `.user.ini` | 服务器配置文件（计划移除出仓） | ignore + staged delete ✓ |
| `.well-known/` | 域名验证/证书材料 | `.gitignore`（本次审计发现新增行，与 .gitignore M 配套）✓ |
| `/root/siteintel-f1` worktree | F1 进行中工作现场 | 独立 worktree/分支，git 操作隔离 ✓ |

---

## 5. 如何安全将生产 Git HEAD 从 1bdfcb4 对齐到 c74dfd1

### 5.1 通道选择（审计实证对比）

| 方案 | 可行性 | 结论 |
|---|---|---|
| A. 服务器直接 fetch GitHub（需 remote + 凭据） | 服务器无 GitHub 密钥、无 remote | ✗ 需先向服务器引入凭据（不推荐，扩大攻击面） |
| **B. git bundle 离线引入（推荐）** | 本地打包 → scp → 服务器 fetch bundle | ✅ **零凭据、零 remote 修改、零网络依赖** |
| C. 本地向服务器 push（ssh path remote） | 需配置 receive.denyCurrentBranch 等 | △ 需要修改生产仓库配置（denyCurrentBranch 是本地配置修改，且非 bare push 有风险） |

**推荐 B（bundle）**：`git bundle` 在本地（Windows）打包 `main`（含 c74dfd1 及全部祖先对象），scp 至服务器 `/tmp/`，服务器 `git fetch <bundle>` 将对象引入对象库（fetch 是 git 层安全写入，不触碰工作区/配置/服务）。bundle 自校验（`git bundle verify`）保证传输完整性。

### 5.2 对齐三步（指针 / 工作区 / 保护）

```
[1] 历史保护：git tag production-legacy-1bdfcb4 1bdfcb4      ← 生产全部本地历史可达性保险
[2] 指针移动：git update-ref refs/heads/master c74dfd1       ← 精确分支指针（非 reset，不触碰任何工作区文件）
[3] 工作区对齐：git checkout -- src/                          ← 用 index（=c74dfd1）覆盖 src 工作区（LF 化）
```

- **为什么不用 `git reset`**：reset 家族会隐式携带混合工作区/index 语义，且用户已明确禁止 `reset --hard`；`update-ref` 是纯指针操作，语义最小、可精确回滚。
- **为什么 src 会 LF 化**：c74dfd1 blob 为 LF，checkout 覆盖后工作区 = LF。**对运行代码零影响**（`.next` 产物不变、不 restart；JS/JSX 行尾语义中性——下次 rebuild 产物与现产物语义等价，§8 验证项）。
- **16 项 B 类卫生文件不受影响**：不在 checkout 范围内（`git checkout -- src/` 只覆盖 src 子树），原位保留。

### 5.3 对齐后状态预期

```
GitHub main        = c74dfd1
Production HEAD    = c74dfd1   ← 本次达成
Production src 工作区 = c74dfd1（LF，diff 干净） ← 本次达成（14 A 类 clean）
Production 16 卫生项  = 保持 dirty（白名单）       ← 阶段 II 处理（§10）
Production running  = c74dfd1 内容（已部署验证）   ← 已达成
```

---

## 6. 是否需要 stash / patch / 临时备份

| 机制 | 需要？ | 说明 |
|---|---|---|
| stash | **不需要** | 无暂存/恢复需求；16 项卫生文件原地保留，14 A 类由 checkout 直接覆盖（内容已 hash 一致，无冲突风险） |
| patch | 不需要 | 14 文件内容已在 c74dfd1 中，无需补丁 |
| **临时备份（推荐，双保险）** | 可选 ① | `tar czf /www/backup/siteintel-src-pre-unify-20260817.tgz src/`（仅 src，~KBs）——行尾回滚用；**不做**则回滚后工作区为 LF（内容等价，git 层反而更干净，§9） |
| **历史保护 tag（必须）** | 必做 ② | `production-legacy-1bdfcb4`（与 f1 分支、reflog 构成三重保留） |

---

## 7. 精确执行顺序 + 每步验证命令

> 阶段 I 共 8 步。**任何一步验证 FAIL → 立即停止（§8），不继续**。执行前需用户批准。

| 步 | 动作 | 验证命令（必须通过） |
|---|---|---|
| 0 | **预备确认**：服务 active；确认无其他人在生产操作 | `systemctl is-active siteintel` → `active`；`curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:3003/robots.txt` → `200` |
| 1 | 本地打包 bundle：`cd /c/Users/deepo/siteintel-github && git bundle create /tmp/siteintel-unify.bundle main` | `git bundle verify /tmp/siteintel-unify.bundle` → `ok`；`git bundle list-heads` 含 `main` → `c74dfd1` |
| 2 | 传输：`scp -P 613 /tmp/siteintel-unify.bundle root@154.204.176.66:/tmp/` | 本地 `ls -l` 与服务器 `ls -l` 字节数一致 |
| 3 | 服务器引入对象：`cd /www/wwwroot/siteintel.cc && git fetch /tmp/siteintel-unify.bundle 'refs/heads/*:refs/remotes/bundle/*'` | `git cat-file -t c74dfd1` → `commit`（对象入库） |
| 4 | **历史保护 tag**：`git tag -a production-legacy-1bdfcb4 -m "production local history preserve (pre-unification)" 1bdfcb4` | `git rev-list -n1 production-legacy-1bdfcb4` → `1bdfcb4…` |
| 5 | **指针移动**：`git update-ref refs/heads/master c74dfd1` | `git rev-parse master` → `c74dfd1…`；`git rev-parse HEAD` → `c74dfd1…` |
| 6 | **工作区对齐**：`git checkout -- src/` | `git status --short` → 14 个 src 项全部消失；`git diff --exit-code -- src/` → rc=0 |
| 7 | **终验**： | 见 §7.1 |
| 8 | **服务确认**（只读）： | `systemctl is-active siteintel` → `active`；`curl http://127.0.0.1:3003/robots.txt` → 200；线上 `https://www.siteintel.cc/`、`/sitemap.xml`、`/website/wordpress.org` → 200 + title/robots 正确 |

### 7.1 终验清单（步骤 7）

```
git rev-parse HEAD                                  → c74dfd1
git branch --show-current                           → master
git status --porcelain                              → 仅剩 16 项 B 类卫生（§1.3 清单逐一比对，无新增/无缺失）
git diff --stat                                     → 0 变化（B 类文档类 diff 除外——它们保持原 dirty）
git log --oneline -2                                → c74dfd1 / b52ec84
git cat-file -t production-legacy-1bdfcb4           → tag
git rev-parse f1-candidate-filter                   → 1bdfcb4（f1 分支未动）
git status --porcelain（/root/siteintel-f1）         → 4 项未提交（F1 现场未动）
```

---

## 8. 失败停止条件

| # | 条件 | 动作 |
|---|---|---|
| 1 | 步骤 0 服务非 active 或 3003 异常 | **停止**，不执行任何 git 操作，报告 |
| 2 | bundle verify 失败 / 传输字节数不一致 | 停止，重新打包/重传，不执行 fetch |
| 3 | 步骤 3 后 `cat-file -t c74dfd1` 非 commit | 停止，bundle 未完整引入，检查后重试 |
| 4 | 步骤 4 tag 创建失败 | 停止（历史保护缺失，不得移动指针） |
| 5 | 步骤 5 后 rev-parse 非 c74dfd1 | 停止（指针异常，立即按 §9 回滚） |
| 6 | 步骤 6 后 src 仍有 diff 或 status 出现**新文件/未知项** | 停止，不得重启服务；比对 §1.3 清单定位 |
| 7 | 步骤 7/8 任一验证 FAIL | 停止；git 层已对齐但服务异常 → 按 §9 回滚 git 层后报告 |
| 8 | 任何一步出现服务 crash / journal 新 fatal | 立即停止 |

> 注：步骤 1-7 全程**不重启服务、不触碰 .env/node_modules/.next/数据库/nginx/staging**——git 层操作与运行进程完全隔离，服务无感知。

---

## 9. 完整回滚方案（阶段 I）

> 回滚场景：步骤 5-8 任一失败，或对齐后发现问题。

```
[1] 指针回退：git update-ref refs/heads/master 1bdfcb4        ← 与 [5] 对称
[2] 工作区：若执行了可选 tar 备份 → 解压恢复 src（CRLF 原样）
           若未备份 → 接受 LF（内容与 1bdfcb4 仅行尾差异；git diff -w 干净；JS 语义等价）
[3] 清理：git remote 不存在（无清理项）；refs/remotes/bundle/* 可删（git update-ref -d）
[4] 验证：git rev-parse master → 1bdfcb4；git status 恢复为 30 项原清单
[5] 服务：systemctl is-active siteintel → active（从未重启，无需恢复）
[6] 历史 tag production-legacy-1bdfcb4 保留（无害，指向 1bdfcb4）
```

**回滚对运行零影响**（服务全程未 restart、.next 未触碰）。生产历史（1bdfcb4 链）由 `production-legacy-1bdfcb4` tag + `f1-candidate-filter` 分支 + reflog 三重保留，**任何操作不导致历史丢失**。

---

## 10. 卫生项处理决策（阶段 II — 实现 100% 四者一致，待批准后执行）

> 阶段 I 达成后，唯一残留 = 16 项 B 类 dirty。**要达成"working tree = HEAD = GitHub main"完全一致，卫生项必须入库**（否则永远 dirty 或需 ignore 式隐藏——不推荐，反模式）。

**关键约束**：卫生项 commit 会使 HEAD 成为 c74dfd1 的后代（`c74dfd1 + hygiene commit`）。此时：
- `production-seo-phase1-p0-2026-08-17` tag 仍指向 c74dfd1（正式版本语义不变）✓
- GitHub main 更新为卫生 commit → 生产 fetch 对齐 → **四者 100% 一致** ✓

| 决策项 | 建议 | 理由 |
|---|---|---|
| `.gitignore` M（+4 行） | **纳入** | 正确的仓库维护（.user.ini/.well-known ignore） |
| `.user.ini` staged D | **纳入** | 服务器配置文件出仓（与 ignore 配套落定） |
| `baidu_verify_codeva-r7wFVf611L.html`（大写）D | **纳入删除**（若弃用百度验证）；或恢复并统一为小写 | 大小写双文件遗留；需要用户决策 |
| `DISCOVERY-CANDIDATE-FILTER-DESIGN.md` M | **纳入** | F1 设计文档，入库有长期价值 |
| 12 个 untracked 规划文档 | **建议纳入**（commit 前人工确认无敏感凭据——预期无，仓库为 Private） | 规划沉淀；不纳入则永远 dirty 或需 ignore |
| f1 worktree（4 项未提交） | **保持原位**（不入本次 commit；未来 F1 合并时随 f1-candidate-filter 分支处理） | 进行中工作现场，不提前污染 |

**阶段 II 执行顺序（预设计，批准后细化）**：
1. 生产：`git add` 卫生项（白名单，逐项确认）→ `git commit -m "chore: production hygiene files"`（identity 已配置：SiteIntel Deploy）
2. 生产：`git bundle create hygiene.bundle master` → scp 回本地
3. 本地：fetch bundle → `git merge --ff-only` → `git push origin main`（GitHub main 更新）
4. 生产：`git fetch`（同步）→ 终验四者一致（working tree 100% clean）
5. 生成阶段 II 报告

---

## 11. 风险清单

| 风险 | 等级 | 缓解 |
|---|---|---|
| 行尾 LF 化（src 14 文件 CRLF→LF） | 低 | `.next` 产物不变、不 restart → 运行代码零影响；下次 rebuild 语义等价（JSX/字符串行尾中性）；§8 验证项覆盖 |
| 生产本地历史可达性 | 低 | tag + f1 分支 + reflog 三重保留；git gc 默认 90 天不清理 reflog |
| bundle 传输损坏 | 低 | `git bundle verify` + 字节数比对 + cat-file 三重校验 |
| 16 项 B 类在统一期间被误操作 | 低 | checkout 范围仅 `src/`；执行顺序中 status 清单比对（§7.1） |
| 服务受影响 | 极低 | 全程零 restart、零触碰 .next/node_modules/.env；git 层与进程隔离 |
| 阶段 II commit 引入敏感信息 | 低 | commit 前人工确认清单；仓库 Private；不含 .env（ignore 实证） |

---

## 12. 结论

- **审计完成**：生产 git 拓扑、30 项未提交（A 14 / B 16）、C/D/E 类安全边界、行尾事实、历史分叉、通道缺失全部实证。
- **方案确定**：阶段 I 用 **bundle 离线引入 + update-ref 指针移动 + src checkout 对齐**，零凭据、零服务影响、可精确回滚，将 HEAD 从 1bdfcb4 对齐至 c74dfd1；阶段 II（待批准）将 16 项卫生项入库推回 GitHub，实现 **GitHub main = Production Git HEAD = Production working tree = Production running code 完全一致**。
- **本次仅审计 + 方案设计，未执行任何修改**。等待批准后按 §7 执行阶段 I。

---
**完成，停止。等待批准。** 未执行：任何 git 写操作 / .env / DB / nginx / staging / siteintel_staging_ro / restart / SEO Phase 2。
