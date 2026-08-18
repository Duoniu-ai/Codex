# SITEINTEL SEO PHASE 1 P0 — PRODUCTION DEPLOYMENT PLAN

- **日期**: 2026-08-17
- **当前生产**: `1bdfcb4`（/www/wwwroot/siteintel.cc，siteintel.service，127.0.0.1:3003）
- **目标代码**: `c74dfd1`（seo/phase-1-p0，GitHub Private 远端已验证一致；staging 3004 全量验证 PASS）
- **本计划性质**: 方案文档（当前阶段仅部署前准备，**未执行任何部署操作**）

---

## 1. 生产修改文件清单（c74dfd1 vs 1bdfcb4）

**修改（7 个）**

| 文件 | 修改目的 | 影响运行逻辑 | 影响数据库 |
|---|---|---|---|
| `src/app/website/[domain]/page.tsx` | P0-1 五状态机（null→404 / processing / no-data / report）+ robots 改由 `websiteRobotsPolicy`（默认 noindex） | **是**（/website 页渲染与索引判定） | 否 |
| `src/app/sitemap.ts` | P0-2/3 sitemap 批量侧接入 Gate V1（同构输入，仅 indexable 收录） | **是**（sitemap.xml 内容） | 否 |
| `src/app/layout.tsx` | P0-4 移除全局 `alternates.canonical` | **是**（各页 canonical 不再错误继承首页） | 否 |
| `src/app/bulk/page.tsx` | P0-4/5 自持 canonical + metaDescription | 是（SEO 元数据） | 否 |
| `src/app/docs/api/page.tsx` | P0-4/5 自持 canonical + metaDescription + 显式 index,follow | 是（SEO 元数据） | 否 |
| `src/app/report/page.tsx` | P0-4 自持 canonical | 是（SEO 元数据） | 否 |
| `src/lib/i18n.ts` | P0-1 processing 文案 + P0-5 bulk/apiDocs metaDescription（zh/en） | 是（文案/元数据） | 否 |

**新增（7 个）**

| 文件 | 修改目的 | 影响运行逻辑 | 影响数据库 |
|---|---|---|---|
| `src/lib/seo/report-state.ts` | P0-1 资源状态机 + robots 顶层决策（Gate V1 页面侧唯一出口） | **是** | 否 |
| `src/lib/seo/index-gate.ts` | P0-2/3 Simple Index Gate V1（唯一事实来源，页面+sitemap 共用） | **是** | 否 |
| `src/lib/seo/low-value-domain.ts` | P0-6 低价值域名判定（sitemap 与 Gate 共用） | **是** | 否 |
| `src/app/website/[domain]/not-found.tsx` | P0-1 路由级 404 页（真 404 + 分析引导） | **是** | 否 |
| `src/lib/seo/index-gate.test.ts` | 测试（17 例） | 否 | 否 |
| `src/lib/seo/report-state.test.ts` | 测试（13 例） | 否 | 否 |
| `src/lib/seo/low-value-domain.test.ts` | 测试（14 例） | 否 | 否 |

**部署范围结论**: 仅 `src/` 下 14 个文件；不含 package.json、next.config、prisma/、config/、scripts/、.env、node_modules。

## 2. 数据库影响

- **无 Prisma migration**（c74dfd1 相对基线 `prisma/` 零 diff，实证）
- **无 schema 修改**
- **无数据库数据写入**（只读角色验证阶段实证 staging 写被拒；部署代码不含任何 DML）
- **不需要 `prisma migrate`**（生产 migrations 目录 7 个既有项，无新增）
- **不需要 `prisma db push`**
- Prisma Client 已由既有 node_modules 提供（schema 未变，无需重新 generate）

## 3. 部署步骤（可回滚，待批准后执行）

> 前提：生产仓库当前有 16 项**既有未提交项**（基线期卫生项：.gitignore M / .user.ini D staged / 大写 baidu D / 12 项 untracked 文档）——**不得清空**；以下步骤全部避开这些路径。

**方案 A（推荐，服务器 git fetch 应用）**：
```
[1] cd /www/wwwroot/siteintel.cc
[2] git tag pre-deploy-seo-p0-20260817            ← 部署前 Tag（1bdfcb4 当前位置）
[3] git remote add origin https://github.com/Duoniu-ai/siteintel.git   （如无 remote）
    git fetch origin seo/phase-1-p0
    git fetch origin c74dfd17dc559b4ba18812aef24a98ea3d72c3b8
[4] git checkout c74dfd17dc559b4ba18812aef24a98ea3d72c3b8 -- <14 个 src 文件>   ← 仅应用变更文件
[5] export PATH=/root/.nvm/versions/node/v20.20.0/bin:$PATH
    cd /www/wwwroot/siteintel.cc && pnpm build
    ※ build 失败 → 立即停止，**不得重启生产**，执行 §4 回滚
[6] systemctl restart siteintel
[7] HTTP 验证：
    - http://127.0.0.1:3003/robots.txt → 200
    - http://127.0.0.1:3003/  → 200 + title「看懂任何一个网站 | SiteIntel」
    - http://127.0.0.1:3003/website/nonexistent-xyz-20260817.com → **404**
    - http://127.0.0.1:3003/sitemap.xml → 无 example.com / virtualearth / 低价值 CDN
    - http://127.0.0.1:3003/website/github.com → 200 + robots=index,follow
    - http://127.0.0.1:3003/website/example.com → 200 + robots=noindex
```

**方案 B（备选，既有 SSH 直传流程）**：
```
[1] 本地: git archive --format=tar.gz c74dfd1 -o seo-p0.tar.gz  → scp 至服务器 /tmp/
[2] 服务器: cd /www/wwwroot/siteintel.cc && git tag pre-deploy-seo-p0-20260817
[3] tar xzf /tmp/seo-p0.tar.gz --strip-components=0 覆盖 src/（archive 仅含 tracked 文件，不触碰 .env/node_modules/.gitignore）
[4] 同方案 A [5][6][7]
```

## 4. 回滚方案（恢复到 1bdfcb4）

**回滚触发条件**：build 失败 / 重启后 3003 异常 / 任一验证项 FAIL。

```
[1] cd /www/wwwroot/siteintel.cc
[2] git checkout 1bdfcb4 -- <14 个 src 文件>        ← 文件级回退（保留所有未提交卫生项）
[3] export PATH=/root/.nvm/versions/node/v20.20.0/bin:$PATH && pnpm build
[4] systemctl restart siteintel
[5] 验证恢复：3003 robots 200 / 首页 200 / /website/nonexistent-xyz-20260817.com 恢复为基线行为（200 空态）
[6] （如需彻底恢复）备份目录 /www/backup/siteintel.cc-pre-seo-p0-20260817/ 整目录还原
```

- 回滚全程不动 .env / node_modules / 数据库（零 schema、零数据，部署无任何 DB 变更——回滚无需 DB 操作）
- 部署前 Tag `pre-deploy-seo-p0-20260817` 在回滚后保留（可删），**不影响** production-baseline-2026-08-17

## 5. 部署后观察项（建议，非本阶段）

- 重新提交 sitemap 至 Google Search Console（Gate 收缩后收录量预期下降）
- 观察 24-48h：不存在域名 404 无异常误伤（连续 4xx 日志）、sitemap 无新增低价值条目
- entity 页 robots/sitemap 同源化（验收 P3 记录项）后续任务书处理

---

**本阶段（PRE-PRODUCTION RELEASE PREPARATION）已完成：Git 状态确认 ✅ / Push GitHub ✅ / 只读检查 6 项 ✅ / 部署计划生成 ✅。未执行任何部署操作。**
