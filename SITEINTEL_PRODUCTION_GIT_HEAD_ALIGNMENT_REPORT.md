# SITEINTEL PRODUCTION GIT HEAD ALIGNMENT — 执行与验收报告

- **日期**: 2026-08-17
- **批准**: 用户批准 Workflow Finalization Plan ① D 项（master `1bdfcb4` → `c74dfd1`）；首次工作区对齐失败停止后，用户批准恢复选项 1（`git checkout c74dfd1 -- src/` + 全 index 同步 + 全量验证）
- **执行范围**: 仅生产 Git 指针/index/src 工作区与 `c74dfd1` 对齐。**未执行**: deploy.sh 启用 / 阶段 II 卫生收口 / SEO Phase 2 / staging 删除 / siteintel_staging_ro 删除 / restart / build / DB / nginx / .env

---

## 1. 执行前只读基线（11 项，全部 PASS）

| # | 检查 | 实测 | 判定 |
|---|---|---|---|
| 1 | 服务 active | `active` | ✅ |
| 2 | 3003 HTTP | robots=200 / home=200 | ✅ |
| 3 | master = 1bdfcb4（对齐前） | `1bdfcb4d7178f21b6ae435e4fbb4c5cf606d6bae` | ✅ |
| 4 | c74dfd1 对象可验证 | fetch bundle 后 `cat-file -t` = commit | ✅ |
| 5 | GitHub main = c74dfd1 | 前置阶段 ls-remote 逐字验证 | ✅ |
| 6 | production-baseline-2026-08-17 不变 | → b52ec84 | ✅ |
| 7 | production-seo-phase1-p0-2026-08-17 不变 | → c74dfd1 | ✅ |
| 8 | production-history-archive-20260817 不变 | → 1bdfcb4（本地+GitHub） | ✅ |
| 9 | legacy-prod-history 不变 | → 1bdfcb4，66 commits | ✅ |
| 10 | f1 worktree 完好 | HEAD=1bdfcb4，4 项未提交 | ✅ |
| 11 | 16 项卫生文件逐一记录 | §1.3 白名单（§6 对比表） | ✅ |

## 2. 保护锚点 tag（update-ref 前建立）

```
git tag -a production-pre-head-alignment-20260817 -m "..." 1bdfcb4
→ 终验: git rev-list -n1 production-pre-head-alignment-20260817 = 1bdfcb4d71… ✅
```

## 3. bundle / 对象验证（Git bundle 通道，零凭据落服务器）

| 步骤 | 结果 |
|---|---|
| 本地 `git bundle create /tmp/siteintel-align.bundle main` | ✅ |
| `git bundle verify` | ✅ ok |
| scp → 服务器 /tmp/ | ✅ 字节一致（425721 B 本地 = 服务器） |
| 服务器 `git fetch /tmp/siteintel-align.bundle 'refs/heads/*:refs/remotes/bundle/*'` | ✅ |
| `git cat-file -t c74dfd1` | ✅ commit（对象入库） |

## 4. update-ref 记录（指针层）

```
git update-ref refs/heads/master c74dfd1        ← 精确指针移动（非 reset），exit 0
```
**注意**: update-ref 只移动分支指针，**不更新 index**（index 仍为 1bdfcb4 树）。这是第 5 节事故的根因。

## 5. ⚠️ 首次工作区对齐失败 — 停止记录（用户批准恢复前的状态）

### 5.1 现象
- 按方案执行 `git checkout -- src/`（无 commit 参数）→ **从陈旧 index 检出**
- 结果: 7 个 P0 修改文件被覆盖为 **1bdfcb4 旧版**（实证: worktree sitemap.ts = `30dac3ee…` = 1bdfcb4 blob ≠ 预期 `ec337b41…` = c74dfd1）
- `git status --porcelain` = 109 行（49 `A` 排除文件 + ~34 `M` + 6 `D` + `MM`/`AM`），非预期 16 行
- `git diff --exit-code -- src/` rc=0 为**假象**（默认比较 worktree vs 陈旧 index，非 vs HEAD）

### 5.2 根因
`git update-ref` 不更新 index；`git checkout -- <path>` 从 index 检出而非从 commit 检出。正确形式应为 `git checkout c74dfd1 -- src/`（commit tree → index+worktree）。

### 5.3 处置
按用户停止条件（"立即停止，不自行修复，不执行 reset，不执行 force"）**停止并报告**，未做任何修复。服务/运行/保护全部完好（详见 §5.4）。

### 5.4 停止期间保护状态（全部实证）
服务 active / NRestarts=0 / 3003 200；.next 产物未触碰；3 个保护 tag 在位；f1 worktree 4 项原样；49 排除文件与 16 卫生项磁盘原样。

## 6. 批准恢复执行（选项 1）+ 30 项前后对比

```
cd /www/wwwroot/siteintel.cc
git checkout c74dfd1 -- src/          ← commit tree 检出 → src index+worktree = c74dfd1 (LF)，exit 0
git read-tree --reset c74dfd1         ← 全 index 同步至 c74dfd1 树（不触碰工作区），exit 0
```

### 6.1 30 项未提交文件 前后对比

| 类 | 项 | 对齐前 | 对齐后 | 说明 |
|---|---|---|---|---|
| **A. P0 部署差异（14）** | 7 修改文件（bulk/docs-api/layout/report/sitemap/website-page/i18n） | M | **已消除** ✅ | index+worktree = c74dfd1 LF，hash 抽查 6/6 匹配 |
| | 7 新增文件（not-found/index-gate/low-value-domain/report-state 等） | ?? | **已消除** ✅ | 由 checkout 纳入 index+worktree，clean |
| **B. 卫生项（16）** | `.gitignore`（+4 行） | M | **已消除** ✅ | c74dfd1 基线本身包含卫生 ignore 行，worktree=HEAD |
| | `.user.ini` | D（staged） | **已消除** ✅ | 不在 c74dfd1；磁盘文件被 .gitignore 忽略（`!!`） |
| | `baidu_verify_codeva-r7wFVf611L.html`（大写） | D（unstaged） | **已消除** ✅ | 不在 c74dfd1 且磁盘已无；小写规范版被跟踪且 clean |
| | `DISCOVERY-CANDIDATE-FILTER-DESIGN.md` | M（+41 行） | ??（内容未动） | 属 49 排除文件之一，跟踪状态 M→??，磁盘字节不变（待阶段 II 决策） |
| | 12 个 untracked 文档 | ?? | ??（原样） | 规划/审计文档，磁盘未动（待阶段 II 决策） |
| **C. runtime ignore** | .env/.next/node_modules/tsconfig.tsbuildinfo/next-env.d.ts/.well-known | 全部 ignore | 全部 ignore ✅ | `git status --ignored` 实证 |
| **D. 保护** | .env / f1 worktree | 未跟踪/独立 | 未跟踪/独立 ✅ | 全程零触碰 |
| **E. 其他** | 49 个基线排除文件 | 跟踪且 clean（不可见） | ?? 33 项（含目录折叠） | **设计后果**（基线 191 文件树排除 49 文件）：对齐后如实显示为 untracked，磁盘原样保留（部署功能文件） |

### 6.2 新显现项（如实报告）: ` M tsconfig.json`

| 项 | 取证 |
|---|---|
| worktree hash | `db8663e590…` == **1bdfcb4 blob 逐字节一致** → 文件未被本执行触碰（checkout 仅 src/，read-tree 仅 index） |
| c74dfd1 blob | `1530f6a262…`（基线打包时末行换行规整） |
| 内容差异 | `git diff -w c74dfd1 -- tsconfig.json` rc=0 → **仅空白/末行换行差异，内容等价** |
| 定性 | 对齐前因 HEAD=1bdfcb4（worktree 与 index 一致）不可见；对齐后如实显现。**非本次修改产生，零运行影响**（tsconfig 为构建期配置，.next 产物未动、服务未重启） |
| 处置 | 不自行修复；列入阶段 II 卫生收口决策项（归一化入库或保留原样） |

## 7. 最终验证（全量）

| # | 检查 | 命令/标准 | 实测 | 判定 |
|---|---|---|---|---|
| 1 | HEAD | `git rev-parse HEAD` = c74dfd1 | c74dfd1… | ✅ |
| 2 | master | `git rev-parse master` = c74dfd1 | c74dfd1…（branch=master） | ✅ |
| 3 | index == HEAD | `git diff --cached --quiet` rc=0 | **rc=0** | ✅ |
| 4 | src 工作区 == HEAD | `git diff --exit-code HEAD -- src/` rc=0 | **rc=0** | ✅ |
| 5 | 全部跟踪文件 diff | `git diff --exit-code HEAD` | 仅 tsconfig.json（1 行，§6.2） | ✅（已定性） |
| 6 | 文件数 | index == c74dfd1 树 | **198 == 198** | ✅ |
| 7 | hash 抽查 ×6 | worktree vs `c74dfd1:<file>` | sitemap/index-gate/layout/not-found/i18n/bulk **6/6 逐字匹配** | ✅ |
| 8 | P0 文件已跟踪 | `git ls-files` 含 index-gate/low-value-domain/report-state/not-found | 8 匹配，clean | ✅ |
| 9 | src/ 无 untracked | `??` 中 src/ 计数 | **0** | ✅ |
| 10 | log | `git log --oneline -2` | c74dfd1 / b52ec84 | ✅ |
| 11 | 服务 | is-active / NRestarts | **active / 0**（从未 restart） | ✅ |
| 12 | HTTP | 3003 robots/home/sitemap + 线上 www | **200 / 200 / 200 / 200** | ✅ |
| 13 | f1 worktree | HEAD + 未提交 | 1bdfcb4 + 4 项（collector/domain/candidate-filter/f1-dry-run） | ✅ |
| 14 | 服务器 3 tags | 逐一 rev-list | 全部 → 1bdfcb4（含保护锚点） | ✅ |
| 15 | f1 分支 | rev-parse | 1bdfcb4 | ✅ |
| 16 | 历史备份 bundle | /www/backup/ | 824169 B 在位 | ✅ |

## 8. 三方一致性（验收方程）

```
GitHub main                          = c74dfd1     ✅（前置阶段 ls-remote 逐字验证；
Production Git HEAD (master)         = c74dfd1     ✅   本执行零 push/pull，状态不可能变化）
Local mirror main                    = c74dfd1     ✅（clean，status=0）
Production src 工作区                = c74dfd1     ✅（index==HEAD rc=0 + diff rc=0 + hash 6/6）
Production running code（.next）      = c74dfd1     ✅（构建产物未触碰、服务未重启、线上 200）
```
**注**: 验收时本机直连 GitHub 的 `ls-remote` 复查被本地网络阻断（Connection was reset），以本会话零远程写操作 + 前置阶段逐字验证为据。可在网络恢复后复核：`git ls-remote origin refs/heads/main`。

## 9. 服务与运行确认

| 项 | 实测 |
|---|---|
| systemctl is-active | active |
| NRestarts | 0（本次全程零 restart） |
| 3003 robots / home / sitemap | 200 / 200 / 200 |
| 线上 https://www.siteintel.cc/ | 200 |
| .env / .next / node_modules / tsconfig.tsbuildinfo | 未触碰（`git status --ignored` 全部 `!!`） |

## 10. 完整回滚命令（如需恢复对齐前状态）

```
[1] git update-ref refs/heads/master 1bdfcb4          ← 指针对称回退
[2] git read-tree --reset 1bdfcb4                     ← index 恢复 1bdfcb4 树（工作区不触碰）
[3] 工作区 src = c74dfd1 LF 内容（内容与 1bdfcb4 仅行尾差异，diff -w 干净；与对齐前 CRLF 版内容等价——对齐前审计已 hash 实证）
[4] 验证: rev-parse master = 1bdfcb4; git status 恢复 30 项原清单
[5] 服务: 从未 restart，无需恢复
[6] 保护 tag production-pre-head-alignment-20260817 保留（指向 1bdfcb4，无害）
```
回滚零风险：git 层完全对称可逆；历史四重冗余（GitHub legacy 分支 + 归档 tag + 服务器 tag + bundle）不受影响。

## 11. 结论

```
✅ Production Git HEAD Alignment 完成:
    master/HEAD: 1bdfcb4 → c74dfd1（update-ref 精确指针）
    index:       1bdfcb4 树 → c74dfd1 树（read-tree 全同步）
    src 工作区:  = c74dfd1 内容（LF，diff 干净，hash 6/6 实证）
    服务/运行:   零触碰（active / NRestarts=0 / 线上 200）
    f1/legacy/历史/tags/备份: 全部原样
    30 项未提交:  14 项 P0 全部消除；16 项卫生中 4 项自然解决、12 项文档原样、1 项状态 M→??（内容未动）
    新显现项:    tsconfig.json M（基线打包行尾规整产物，diff -w 干净，非修改产生，零运行影响）
```

**未执行（遵守范围与禁令）**: deploy.sh 启用 / 阶段 II 卫生收口 / SEO Phase 2 / staging 删除 / siteintel_staging_ro 删除 / restart / build / DB / nginx / .env。

**待决策（阶段 II 卫生收口排期）**:
1. `tsconfig.json` M — 归一化入库（`git checkout c74dfd1 -- tsconfig.json`）或保留原样
2. 49 个基线排除文件（33 个 ?? 项含目录折叠）— 保持 untracked（现状，设计使然）或补入 .gitignore 清单
3. 12 个 untracked 规划文档 + DISCOVERY-CANDIDATE-FILTER-DESIGN.md — 阶段 II 入库（commit 前人工确认无敏感凭据）

---
**完成，停止。等待下一步验收。**
