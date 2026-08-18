# SITEINTEL Git 历史桥接与统一架构 — 只读审计报告

- **日期**: 2026-08-17
- **性质**: 只读审计 + 架构方案设计（**未执行任何修改**）
- **背景**: 用户已暂停基线统一计划阶段 I，要求先解决"生产真实历史与 GitHub 正式历史完全断裂"的架构问题
- **审计对象**: 生产 `154.204.176.66:613` `/www/wwwroot/siteintel.cc`（66 commits，master @ 1bdfcb4） vs GitHub `Duoniu-ai/siteintel`（main @ c74dfd1，孤儿根 b52ec84）

---

## 0. 审计执行记录（全部只读）

| # | 检查 | 结果 |
|---|---|---|
| 1 | `git rev-list --count master` / `--all` | 66 / 66（master 与 f1 同链） |
| 2 | `git log --all --oneline` | 完整 66 条清单已取得（§2） |
| 3 | 首末 commit 时间 | 2026-08-14 04:46 UTC → 2026-08-16 10:27 UTC（2.5 天集中开发） |
| 4 | `.env` 全历史树扫描（66 commits × ls-tree） | **0 命中**（.env 从未进入任何 commit） |
| 5 | commit message 敏感模式扫描 | 0 真实密钥（命中均为设计说明文本） |
| 6 | `.git` 体积 | 仅 **7.6M** |
| 7 | `1bdfcb4` 树清单（240 文件）vs `b52ec84` 树清单（191 文件） | 差异 = **49 个被排除文件**（§2.3）；**反向差异 = 0**（基线 = 生产树纯子集） |
| 8 | P0 文件内容同源性（6/6 hash 匹配） | b52ec84(LF) CRLF 化 == 1bdfcb4（sitemap/i18n/report/website 4 文件）；layout/docs-api 直接相同（LF 版） |
| 9 | 生产历史 blob 行尾 | **混合**：sitemap/i18n/report/website = CRLF；layout/docs-api/bulk = LF（历史部署来源不同所致） |
| 10 | c74dfd1 vs b52ec84 diff | 14 文件 +831/−33（与部署记录一致，仅 P0 改动） |

**禁止事项零违反**：无 update-ref / reset / checkout / clean / rebase / filter-repo / merge / commit / push / tag / 生产文件 / .env / DB / restart / nginx / staging / siteintel_staging_ro。

---

## 1. 术语框架（先明确定义，避免混淆）

| 术语 | 定义 | 本次是否涉及 |
|---|---|---|
| **保留历史** | 历史对象（commit/blob/tree）不被删除或改写，继续存在于仓库中，可达性可能转移 | ✓ 各方案均含 |
| **恢复历史** | 将已归档/已分离的历史重新挂回某分支使其成为主线的一部分（如 reset 回旧点） | ✗ 不采用（会丢弃 GitHub 正式历史） |
| **桥接历史** | 通过新 commit/merge/tag 在两个无血缘历史之间建立明确的、可记录的关系，不改写任何既有对象 | ✓ 方案 B 核心（tag + 文档 + 分支共存） |
| **合并历史** | 通过 merge（含 --allow-unrelated-histories）将两历史合成一条线 | △ 方案 C 的一种形式（冲突面大，不推荐） |
| **替换生产 Git 指针** | 修改生产仓库分支指针（update-ref / reset / branch -f）使其指向另一 commit | ✓ 阶段 I（已有方案，本报告不重复设计） |

**核心原则（本报告全部方案遵守）**：任何方案不得改写既有 commit 对象（hash 不可变）；"保留历史"与"改写历史"互斥。

---

## 2. 双历史现状模型（实测）

```
时间轴
─────────────────────────────────────────────────────────────►
2026-08-14 ──────► 2026-08-16                    2026-08-17
生产真实历史（66 commits，本地独立）                         ──┐
71d3512 MVP ──► … ──► 9bdf6b2 Phase 3C ──► 1bdfcb4 (master)   │ 断裂
  feat×23 / fix×18 / docs×11 / test×1 / 其他×13               │（无共同祖先）
GitHub 正式历史（2 commits，孤儿根）                           │
b52ec84（root，191 文件快照）─► c74dfd1（SEO P0，14 文件）      ──┘
tag: production-baseline-2026-08-17 → b52ec84
tag: production-seo-phase1-p0-2026-08-17 → c74dfd1
生产部署代码 = c74dfd1 内容（CRLF 覆盖，已实证）
```

### 2.1 生产历史资产清单（决定"历史价值"的实物）

| 资产 | 数量 | 说明 |
|---|---|---|
| commit | 66 | 全量开发历史（MVP → Phase 0-13 → 2.0 Phase 1-3C → F1 文档） |
| 树文件 | 240 | = 191（GitHub 基线集）+ 49 被排除文件 |
| 被排除文件（49） | 27 报告 md + docs/ 15 + scripts/audit/*.json 4 + safe-fetch-correction.mjs + .user.ini + 大写 baidu_verify + V2 总纲 | 基线建立时的有意排除（内部文档/运维材料） |
| 体积 | .git 7.6M | 导入成本可忽略 |
| 敏感状态 | .env 0 入库、message 0 密钥 | 历史本身无凭据风险 |

### 2.2 历史同源性结论（决定桥接可行性的关键）

- **b52ec84 是 1bdfcb4 树的纯子集**（反向差异 = 0）
- 内容 hash 验证 **6/6**：b52ec84 blob（LF）与 1bdfcb4 blob（混合行尾）内容同源
- **生产历史 blob 行尾混合**（部分 CRLF 入库）→ 任何"以生产历史为主历史"的方案都会把混合行尾带入 GitHub 主线（除非改写历史，违背完整保留）

---

## 3. 方案 A：保留 GitHub 孤儿历史，生产旧历史仅本地归档

**定义**：GitHub main 保持现状（b52ec84 → c74dfd1）；生产旧历史在**服务器本地**打 legacy tag + bundle 备份，不进入 GitHub。

**执行路径（预设计）**：
```
[1] 服务器: git tag -a production-legacy-1bdfcb4 -m "..." 1bdfcb4   ← 历史可达性锚点
[2] 服务器: git bundle create /www/backup/siteintel-prod-history-20260817.bundle --all
[3] 本地: 同步 bundle 副本（跨机冗余）
[4] 阶段 I 继续（update-ref master → c74dfd1 + checkout src/）
```

**10 项评估**：

| 维度 | 判定 |
|---|---|
| 改变 commit hash | **否**（全部原对象） |
| force push | **否** |
| 影响 GitHub main | **否**（GitHub 零改动） |
| 影响 tag（两个 production tag） | **否** |
| 保留生产原始历史 | **是**，但仅服务器 + bundle（单点冗余为主） |
| 影响 f1 worktree | **否**（f1 分支自然引用 1bdfcb4） |
| 影响运行代码 | **否**（纯 git 层，不 restart） |
| 单一历史 | **是**（main 唯一开发主线；旧历史为只读档案） |
| 未来分支起点 | GitHub `main` |
| 防再次漂移 | 靠部署机制规范（§7），与方案选择无关 |

**优劣**：最安全（GitHub 零改动、零风险）；缺点 = 生产历史跨机冗余弱（bundle 放本地可缓解）、GitHub 上不可检索生产历史。

---

## 4. 方案 B：生产历史导入 GitHub 为 legacy 分支（桥接，不改写任何现有 ref）

**定义**：生产 66 commits 以**原对象、原 hash** 导入 GitHub 成为只读 `legacy-prod-history` 分支；main 保持不动；通过 annotated tag + README 记录两者关系（孤儿关系明确声明）；服务器本地再保留 tag + bundle 双保险。

**执行路径（预设计）**：
```
[1] 本地: git fetch root@154.204.176.66:/www/wwwroot/siteintel.cc master:refs/heads/legacy-prod-history
        （ssh 路径 fetch，66 commits 原对象导入本地；7.6M；服务器侧只读）
[2] 本地: git tag -a production-history-archive-20260817 -m "production real history (2026-08-14~16)" 1bdfcb4
[3] 本地: git push origin legacy-prod-history                  ← 新分支创建，非改写，无 force
[4] 本地: git push origin production-history-archive-20260817  ← 新 tag
[5] GitHub: legacy 分支 description + README「历史关系」章节（声明孤儿关系与追溯方式）
[6] 服务器: tag production-legacy-1bdfcb4 + bundle（与 A 相同，双保险）
[7] 阶段 I 继续（生产指针对齐 c74dfd1）
```

**10 项评估**：

| 维度 | 判定 |
|---|---|
| 改变 commit hash | **否**（原对象导入，1bdfcb4 链 hash 不变） |
| force push | **否**（新增 ref，非改写任何既有 ref） |
| 影响 GitHub main | **否**（main 不动） |
| 影响 tag | **否**（两个 production tag 不动；新增 archive tag 为纯增量） |
| 保留生产原始历史 | **是，完整且跨机**（GitHub legacy 分支 + 服务器 tag + f1 分支 + bundle 四重） |
| 影响 f1 worktree | **否**（f1 分支仍在服务器原样；不推送 f1 分支避免混淆） |
| 影响运行代码 | **否**（纯 git 层，不 restart） |
| 单一历史 | **是**（main 唯一开发主线；legacy 永久只读档案） |
| 未来分支起点 | GitHub `main` |
| 防再次漂移 | 部署机制规范（§7） |

**优劣**：安全性 = 方案 A（零改写）；历史完整性 = A 的超集（GitHub 跨机冗余 + 可 log/diff/检索）。**唯一增量代价**：GitHub Private 仓库新增 66 commits 内容（含 49 个基线排除文件——内部报告与运维材料）。敏感扫描已净（.env 0 入库 / message 0 密钥），Private 仓库下可接受，**但需用户明示确认**（§8 决策项 ①）。若用户拒绝 → 退回方案 A。

---

## 5. 方案 C：以生产历史为主历史，GitHub 正式历史降级为 release snapshot

**定义**：将生产 66 commits 导入并成为 GitHub main 的主线，GitHub 现有 b52ec84→c74dfd1 作为"正式版本快照"桥接（legacy 化或并入）。

**两种实现形式**：
- **C1（重写 main）**：新 main = 生产历史 + 桥接 commit（树 = c74dfd1 树 ± 49 文件）→ **force push 改写 GitHub main**
- **C2（孤儿 merge）**：`git merge --allow-unrelated-histories` 将 c74dfd1（连同 b52ec84）并入生产历史 → merge commit M 使两历史同时可达（tag 不悬空），仍需 force push main

**10 项评估**：

| 维度 | 判定 |
|---|---|
| 改变 commit hash | 生产 66 不变；**b52ec84/c74dfd1 从 main 历史消失**（C1）或降级为第二父（C2） |
| force push | **必须**（main 历史拓扑改写）——违反常规 Git 礼仪与用户既有规则，Private 仓库无 fork/star 缓解但不可逆 |
| 影响 GitHub main | **改写**（历史拓扑完全变化） |
| 影响 tag | **高风险**：production-baseline-2026-08-17（→b52ec84）与 production-seo-phase1-p0-2026-08-17（→c74dfd1）对象变孤儿 → **GitHub GC 悬空风险**（C1 必须显式保活；C2 借第二父可达） |
| 保留生产原始历史 | **是**（成为主线主体） |
| 影响 f1 worktree | 否 |
| 影响运行代码 | 否 |
| 单一历史 | 是（但主线 = 旧历史） |
| 未来分支起点 | 新 main |
| 防再次漂移 | 依赖部署机制（§7） |

**结构性硬伤**（三条）：
1. **行尾混入**：生产历史 blob 行尾混合（CRLF/LF 实证）→ 以生产历史为主线的 GitHub 仓库 checkout/diff 长期混乱（服务器 core.autocrlf 未设 → checkout 得 CRLF 文件；Windows clone 双转换）——除非改写历史归一（违背完整保留，方案自相矛盾）
2. **文件集决策推翻**：基线建立时有意排除的 49 个内部文件（docs/报告/审计 json）将全部进入 GitHub main 主线历史——需要推翻基线时的明确决策
3. **tag 语义漂移 + force push 不可逆**：正式版本 tag 不再指向"main 历史可达语义"的稳定点（C1）

**优劣**：唯一优点是"主线历史真实性最高"；代价是 force push、tag 悬空风险、行尾混乱、文件集决策推翻、复杂度最高。

---

## 6. 方案 D：其他候选

| 候选 | 定义 | 判定 |
|---|---|---|
| **D1（已并入 B）** | B + 服务器 tag + bundle 双保险 | 采用（= 方案 B 完整形态） |
| D2 | `git replace` / grafts 建立假血缘 | **不推荐**：replace refs 不随 clone 传播、GitHub 不展示、需每端显式配置、易碎隐晦 |
| D3 | subtree / 子仓库挂载 | **不适用**：生产历史是整仓演进史，无目录边界 |
| D4 | 彻底放弃旧历史（仅 bundle，连 tag 也不建） | 弱于 A（冗余更低），无增益 |

---

## 7. 方案对比总表

| 维度 | A 归档本地 | **B legacy 入 GitHub（推荐）** | C 生产历史为主 | D2/D4 |
|---|---|---|---|---|
| **安全性** | ★★★★★ | ★★★★★（零改写） | ★★（force push 不可逆） | ★★★ |
| **历史完整性** | ★★★（单机+bundle） | ★★★★★（GitHub+服务器+tag+bundle 四重） | ★★★★★（成为主线） | ★★ |
| **生产风险** | 零 | 零 | 零（运行无关） | 零 |
| **GitHub 风险** | 零 | 极低（新增 ref 与 tag） | **高**（main 改写 + tag 悬空风险） | 中 |
| **需要 force push** | 否 | **否** | **是** | 否 |
| **需要修改 commit** | 否 | **否** | 否（但 main 拓扑改写） | 否 |
| **影响既有 tag** | 否 | **否**（纯增量） | **高风险（悬空）** | 否 |
| **影响 F1 worktree** | 否 | **否** | 否 | 否 |
| **长期维护复杂度** | 低 | **低**（一次性导入，永久只读） | 高（行尾/文件集/拓扑持续纠缠） | 低但隐晦 |
| **推荐等级** | ★★★★（备选） | **★★★★★ 唯一推荐** | ★（不建议） | ★★ |

---

## 8. 唯一推荐方案：B（生产历史导入 GitHub legacy 分支 + 桥接归档）

**推荐理由（四合一）**：
1. **零改写、零 force push**：GitHub main、两个 production tag、全部 commit hash 原样不动——满足"最安全"的第一原则
2. **历史完整性最大化**：生产 66 commits 以原对象进入 GitHub（跨机冗余、可浏览、可 diff、可检索），叠加服务器 tag + f1 分支 + bundle = 四重保留，任何单点故障不丢历史
3. **单一开发主线**：main（LF 归一、文件集干净）是唯一开发起点，legacy-prod-history 为永久只读档案——未来所有 feature 分支从 main 创建，杜绝双主线
4. **桥接关系显式化**：孤儿血缘不假装、不改写，用 annotated tag（production-history-archive-20260817）+ README + branch description 明确记录"生产真实历史终结于 1bdfcb4（2026-08-16），GitHub 正式历史始自 b52ec84（2026-08-17 基线快照）"

**方案 B 的完整执行路径（预设计，待批准后细化执行）**：

| 阶段 | 动作 | 验证 |
|---|---|---|
| 0 | 只读预备确认（服务 active / 3003 200 / HEAD 1bdfcb4 / 工作区 30 项清单不变） | 全 PASS 才继续 |
| 1 | 本地：`git fetch root@154.204.176.66:/www/wwwroot/siteintel.cc master:refs/heads/legacy-prod-history` | `git rev-parse legacy-prod-history` = 1bdfcb4；`git rev-list --count legacy-prod-history` = 66 |
| 2 | 本地：`git tag -a production-history-archive-20260817 -m "…" 1bdfcb4` | `git rev-list -n1 production-history-archive-20260817` = 1bdfcb4 |
| 3 | 本地：`git push origin legacy-prod-history` | `git ls-remote origin refs/heads/legacy-prod-history` = 1bdfcb4 |
| 4 | 本地：`git push origin production-history-archive-20260817` | ls-remote tag = 对应 annotated 对象 |
| 5 | GitHub：分支 description + README「历史关系」章节 | 人工复核 |
| 6 | 服务器：`git tag -a production-legacy-1bdfcb4` + `git bundle create` 备份 | tag rev-list + bundle verify |
| 7 | 阶段 I 对齐（update-ref master → c74dfd1 + checkout src/）——已有独立方案文档 | 按 SITEINTEL_PRODUCTION_GIT_BASELINE_UNIFICATION_PLAN.md §7 |

**失败停止条件**：① 步骤 1 fetch 失败或 rev-parse ≠ 1bdfcb4 → 停止（不推送）② 步骤 3 push 被拒（仓库策略）→ 停止，检查 ③ 步骤 6 前不执行阶段 I ④ 任何一步出现非预期对象 → 停止报告。
**回滚**：legacy 分支/tag 可删除（纯增量，`git push origin :refs/heads/legacy-prod-history` 删除分支、`git push origin :refs/tags/...` 删除 tag）——**零残余**；阶段 I 回滚按既有方案 §9。

---

## 9. 未来部署一致性机制（防 Git HEAD ≠ 运行文件再次发生）

> 本轮审计实证了漂移的三个根源：① 文件直传部署（git 零写入）② 部署文件行尾与仓库不一致 ③ 无强制验证门禁。以下机制系统性消除：

**原则：部署的唯一代码源 = GitHub main（或正式 release tag）；生产仓库永远通过 git 操作获得文件。**

| # | 机制 | 内容 |
|---|---|---|
| 1 | 部署通道标准化 | 本地 `git bundle create`（目标 tag/commit）→ scp → 生产 `git fetch <bundle>` → `git checkout <commit> -- <files>` → build → restart（部署后 bundle 即删）。**禁止**任何"本地工作区文件直传覆盖 src/"（历史 CRLF 混入的根源） |
| 2 | 部署后强制三查 | `git rev-parse HEAD` = 部署 commit；`git diff --exit-code <commit> -- src/` 必须干净（阶段 I 对齐 LF 后恒成立）；`systemctl is-active` = active |
| 3 | 行尾基线统一 | 阶段 I 完成 src/ LF 化后，生产工作区 = GitHub blob（LF）恒一致；未来部署文件全部来自 blob → 行尾永不再次分裂 |
| 4 | 卫生项收口 | 阶段 II 将 16 项 B 类入库 GitHub main → 生产工作区 100% clean → `git status` 成为有效部署门禁（脏则部署失败） |
| 5 | 门禁自动化（可选未来） | 部署脚本脚本化三查；CI 侧 main push 自动 bundle 发布物 |

---

## 10. 用户决策项（方案 B 的最终确认点）

| # | 决策 | 默认建议 |
|---|---|---|
| ① | 生产历史（含 49 个内部文件：27 报告 md + docs/ 15 + audit json 4 等）进入 GitHub Private 仓库 | **接受**（敏感扫描已净：.env 0 入库 / message 0 密钥；仓库 Private；历史为只读镜像不改写）。若拒绝 → 退回方案 A（仅本地归档） |
| ② | f1-candidate-filter 分支是否同步推 GitHub | **不推**（工作分支，4 项未提交工作区留在服务器；legacy 分支已覆盖其历史） |
| ③ | 未来部署通道：bundle 为主 + 服务器 GitHub deploy key 可选加装 | bundle 先行；deploy key 留作可选后续（涉及向服务器引入凭据，需单独批准） |

---

## 11. 结论

- **断裂本质**：生产 66 commits 真实历史（2026-08-14~16）与 GitHub 孤儿正式历史（b52ec84 基线快照 → c74dfd1）无共同祖先；基线 = 生产树的纯子集 + LF 归一 + 49 文件排除（6/6 hash 同源实证）。
- **唯一推荐**：**方案 B**——生产历史以原对象导入 GitHub `legacy-prod-history` 只读分支 + archive tag 桥接（零改写、零 force push、四重冗余），main 保持唯一开发主线；叠加既有阶段 I（生产指针对齐）+ 阶段 II（卫生项入库）+ §9 部署一致性机制，实现长期单一 Git 历史与部署零漂移。
- **明确否决**：方案 C（force push + tag 悬空风险 + 行尾混入 + 文件集决策推翻）；D2/D3/D4（不适用或弱化版）。
- **本阶段仅审计 + 方案设计，未执行任何修改。** 等待批准（含 §10 三项决策）后执行。

---
**完成，停止。等待批准。** 未执行：任何 git 写操作（update-ref/reset/checkout/clean/rebase/filter-repo/merge/commit/push/tag）/ 生产文件 / .env / DB / restart / nginx / staging / siteintel_staging_ro / SEO Phase 2。
