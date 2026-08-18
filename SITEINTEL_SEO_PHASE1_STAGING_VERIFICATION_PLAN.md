# SITEINTEL SEO PHASE 1 — STAGING VERIFICATION PLAN

- **日期**: 2026-08-17
- **上游**: Production Baseline（1bdfcb4 / GitHub main b52ec84）→ SEO Phase 1 P0 实施（分支 `seo/phase-1-p0`）→ 验收 PASS WITH MODIFICATIONS（3 个 P3 记录项，确认不混入本次）
- **本阶段性质**: 测试环境部署方案（审计完成，**未创建/未执行/未连接数据库/未修改服务器**）
- **总原则**: 禁止覆盖 siteintel.cc 生产、禁止修改生产数据库、禁止 migrate/db push、禁止写库、不自动 merge；staging 完全独立

---

## 1. 当前生产部署结构（2026-08-17 服务器只读审计实证）

| 项 | 实证值 |
|---|---|
| 服务 | `siteintel.service`（systemd，`After=network.target postgresql.service`） |
| 工作目录 | `/www/wwwroot/siteintel.cc`（git 仓库，HEAD 1bdfcb4，无 remote） |
| 启动命令 | `node /www/wwwroot/siteintel.cc/node_modules/next/dist/bin/next start --hostname 127.0.0.1 --port 3003` |
| 运行时 | Node **v20.20.0**（`/root/.nvm/versions/node/v20.20.0/bin/node`，SSH 非登录 shell 为 Node 18 的坑已有记录） |
| 监听 | **127.0.0.1:3003 仅本机**（`--hostname 127.0.0.1`，不暴露公网） |
| Nginx | `/www/server/panel/vhost/nginx/siteintel.cc.conf`：80+443 SSL（Cloudflare 代理回源），server_name `siteintel.cc www.siteintel.cc`，反代 `127.0.0.1:3003`；`location = /api/analyze` 精确匹配（宝塔 WAF 加斜杠 301 的既有坑，**必须保留**）；`/www/server/panel/vhost/nginx/well-known/siteintel.cc.conf` 证书验证；`.well-known` 静态直出 |
| 证书 | `siteintel.cc/fullchain.pem`，SAN = **仅 siteintel.cc / www.siteintel.cc（非通配符）** |
| 数据库 | PostgreSQL 14，监听 `0.0.0.0:5432`；应用连接串在 `/www/wwwroot/siteintel.cc/.env`（键名实证：`DATABASE_URL / NEXT_PUBLIC_SITE_URL / AUTH_SECRET / TELEGRAM_BOT_TOKEN / ADMIN_EMAILS / AUTO_DISCOVER / DISCOVER_DAILY_CAP`，值未读取） |
| 构建 | `.next/BUILD_ID` 存在（08-16 构建） |
| 部署方式 | SSH 直传 → pnpm build → `systemctl restart siteintel`（无 CI、无 remote、无 GitHub 拉取链路） |
| 既有 staging 先例 | **duoniu-staging**：systemd 单元（`Environment=PORT=3902` + EnvironmentFile + 独立工作目录 + 独立日志 `staging.out`）+ `staging.duoniu.cc` 子域名 nginx conf（**爬虫 UA 403 白名单过滤** + 敏感文件 403）——同服务器已有成熟 staging 模式可仿 |
| 资源 | 端口 3004/3005/3903 **空闲**；磁盘 `/dev/vda1` 97G 总量 48G 可用（51%）；无 Docker |
| 同服务器服务 | siteintel(3003) / ipcesu(3002) / test.ipcesu(3001) / duoniu-preview(3901) / duoniu-staging(3902) |

## 2. 推荐测试环境架构

**两阶段：阶段一（本方案主体）= 本机环回 staging，零公网暴露；阶段二（可选）= preview 子域名供浏览器可视化验收。**

### 阶段一 — 环回 staging（推荐，零风险）

```
浏览器/curl ──SSH 隧道──▶ 127.0.0.1:3004（本机端口，不暴露公网）
                              ▲
                              │
                   siteintel-staging.service（systemd，新单元）
                              │
                   /www/wwwroot/siteintel.cc-staging（新目录，独立代码副本）
                              │ 只读角色连接
                              ▼
                  PostgreSQL 14（生产实例，只读角色 siteintel_staging_ro）
```

- 端口 **3004**（127.0.0.1），与生产 3003 完全隔离
- **无 DNS、无证书、无 nginx 修改**——验证全程经服务器本地 curl 或 `ssh -L 3004:127.0.0.1:3004` 隧道
- 爬虫/搜索引擎零接触（不监听公网）→ staging 不可能被收录，SEO 零污染

### 阶段二 — preview.siteintel.cc（可选，供用户浏览器验收）

仿 `staging.duoniu.cc.conf` 模式：新证书（SAN 非通配，需宝塔签发 + CF DNS 记录）+ nginx conf（**必须含爬虫 UA 白名单 403 + noindex 头**，防 staging 被搜索引擎收录）+ `preview.siteintel.cc` 指向 127.0.0.1:3004。**仅在阶段一验证通过后另行批准执行。**

## 3. 测试环境 URL / 访问方式建议

| 阶段 | 访问方式 | URL |
|---|---|---|
| 一（本次） | 服务器本地：`curl -H "Host: siteintel.cc" http://127.0.0.1:3004/...` | `http://127.0.0.1:3004` |
| 一（本地浏览器/脚本） | SSH 隧道：`ssh -L 3004:127.0.0.1:3004 root@154.204.176.66` 后本地访问 | `http://localhost:3004` |
| 二（可选） | 公网 | `https://preview.siteintel.cc` |

注意：Next.js 页面渲染依赖 `NEXT_PUBLIC_SITE_URL` 生成 absolute URL（canonical/sitemap/OG）——阶段一设为 `http://127.0.0.1:3004`；**验证 canonical 时以 `alternates.canonical` 输出的相对路径语义为准**（见 §9 验证方法），如需验证绝对 URL 形态可临时设 `NEXT_PUBLIC_SITE_URL=http://localhost:3004`（隧道模式下一致）。

## 4. 代码来源

- 代码 = 本地 `C:\Users\deepo\siteintel-github` 工作树（分支 `seo/phase-1-p0`，当前 7 modified + 7 untracked，**尚未 commit**）
- 传输方式：**本地先 `git commit` 到 `seo/phase-1-p0`**（仅本地分支，不进 main、不进 remote）→ `git archive` 或 rsync 直传服务器 `/www/wwwroot/siteintel.cc-staging/`（排除 `node_modules/ .next/ .env .git`）
- 服务器 staging 目录独立于生产目录——生产 `/www/wwwroot/siteintel.cc` 零触碰
- 服务器 staging 内执行 `pnpm install`（或复用 node_modules 副本，以 pnpm-lock 为准）→ `pnpm prisma generate`（schema 与本分支一致，零迁移）→ build

## 5. 环境变量策略

staging `.env` = 生产 `.env` 副本，**逐键策略**：

| 键 | staging 值 | 说明 |
|---|---|---|
| `DATABASE_URL` | **staging 只读角色**（§6） | 硬性只读，写端点被 PG 拒绝 |
| `NEXT_PUBLIC_SITE_URL` | `http://127.0.0.1:3004`（阶段一） | canonical/sitemap/OG 基址 |
| `AUTH_SECRET` | **新生成随机值** | 与会话 cookie 独立，杜绝跨环境会话串用 |
| `TELEGRAM_BOT_TOKEN` | **移除** | staging 绝不向真实用户告警 |
| `ADMIN_EMAILS` | 保留 | admin 页访问控制 |
| `AUTO_DISCOVER` | `0` | 禁发现引擎 |
| `DISCOVER_DAILY_CAP` | `0` | 兜底 |

**调度器防护（关键）**：`instrumentation.ts` 启动时注册监控 tick + discovery 节拍。staging 下即使调度器触发，其写路径（建 Investigation/Fact 等）会被只读角色拒绝——**DB 级硬兜底**，最多产生错误日志（方案验证项 9.7 需确认无重试风暴）。阶段一仅读页面验证，调度器影响可控；若验证中出现写错误刷屏，作为发现记录上报。

## 6. 数据库访问策略（不执行，方案待批准）

**主方案 — 只读角色直连生产实例**：

```
CREATE ROLE siteintel_staging_ro LOGIN PASSWORD '<随机生成>';
GRANT USAGE ON SCHEMA public TO siteintel_staging_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO siteintel_staging_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO siteintel_staging_ro;
ALTER ROLE siteintel_staging_ro SET default_transaction_read_only = on;
```

- 新建独立角色（零改动生产角色）；只读 + 事务级只读双重约束
- 影响：页面渲染（读）全正常；`/api/analyze`、`/api/v1/analyze`、claim、dashboard 等一切写端点被 PG 拒绝 → staging 不可能写入任何生产数据
- **性质**：PG 配置变更（新建角色 + 授权），零数据变更；生产应用连接不受影响
- **备选 — 快照库**：`pg_dump 生产 | pg_restore 到新库 siteintel_staging`（同样配只读角色）。零生产接触（除 dump 读），但数据静态、需刷新，且运行中状态（running/failed）样本以 dump 时刻为准
- **推荐主方案**：验证数据最新最真实（六状态样本以生产实时为准）；只读角色硬约束比应用层屏蔽可靠
- ⚠️ 两项均**待批准后执行**；执行阶段仅需服务器 root + .env 内既有凭据，不涉及任何写数据操作

## 7. 生产零影响保障

1. **文件系统**：新目录 `/www/wwwroot/siteintel.cc-staging`，生产目录零触碰
2. **进程**：新 systemd 单元 `siteintel-staging.service`（端口 3004），生产服务不动
3. **网络**：3004 仅监听 127.0.0.1，无公网入口 → 无爬虫/无 SEO 污染/无攻击面
4. **数据库**：只读角色——staging 一切写操作被 PG 拒绝；生产角色/数据/表零变更
5. **Nginx**：阶段一零修改；阶段二仅追加新 server 块（仿 duoniu 先例），不动 siteintel.cc.conf
6. **证书/DNS**：阶段一不需要；阶段二需批准后签发（不影响现有证书）
7. **调度器**：写失败被 DB 拒（见 §5）；发现引擎/监控 tick 无法污染生产
8. **会话安全**：独立 AUTH_SECRET，staging cookie 无法在生产域生效

## 8. 部署步骤（方案，未执行）

```
[1] 本地：git add + commit 到 seo/phase-1-p0（仅本地分支；不进 main/remote）        ← 需批准
[2] 本地：git archive --format=tar 或 rsync → scp 至服务器 /www/wwwroot/siteintel.cc-staging/
[3] 服务器：cp /www/wwwroot/siteintel.cc/.env .env-staging 并按 §5 修改 → 生成新 AUTH_SECRET
[4] 服务器：CREATE ROLE siteintel_staging_ro + GRANT（§6 主方案）                   ← 需批准（DB 配置变更）
[5] 服务器：cd staging && pnpm install && pnpm prisma generate
[6] 服务器：NEXT_PUBLIC_SITE_URL=http://127.0.0.1:3004 pnpm build
[7] 服务器：写 /etc/systemd/system/siteintel-staging.service（仿 duoniu-staging 单元：
         WorkingDirectory、Environment=PORT 由 ExecStart --port 3004 指定、
         ExecStart=node next start --hostname 127.0.0.1 --port 3004、
         独立日志 staging.out）→ systemctl daemon-reload → systemctl start siteintel-staging
[8] 冒烟：curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:3004/ → 200
[9] 执行 §9 验证矩阵 → 生成 STAGING VERIFICATION REPORT → 验收
[10] （可选，批准后）阶段二：preview.siteintel.cc 证书 + nginx conf + CF DNS
```

## 9. 回滚方式

```
systemctl stop siteintel-staging
rm /etc/systemd/system/siteintel-staging.service && systemctl daemon-reload
rm -rf /www/wwwroot/siteintel.cc-staging
（如需）DROP ROLE siteintel_staging_ro;   ← 不影响生产角色/数据
（阶段二）移除 nginx server 块 + DNS 记录
```

生产目录、生产服务、生产 .env、生产数据全程零触碰——回滚 = 删除新增物，生产无痕。

## 10. 验证项目

### 10.0 验证前置（执行阶段，需批准只读连接）

用生产 `.env` 内 DATABASE_URL（或只读角色）执行**只读 SELECT**，为六状态各找一个真实示例域名（记录域名+状态，不导出敏感数据）：

| 状态 | 样本来源（只读查询） |
|---|---|
| 不存在域名 | 任意未登记域名（如 `nonexistent-xyz-20260817.com`，无需查库） |
| running | `SELECT domain FROM target t JOIN investigation i ... WHERE i.status='running' ORDER BY i.startedAt DESC LIMIT 1` |
| failed（无数据） | 同上 `status='failed'` + 无 fact 目标 |
| failed（有历史报告） | 同上 `status='failed'` + 存在 fact/历史 |
| completed | `status='completed'` |
| partial | `status='partial'` |

如某状态无样本（如 running 恰好为空），记录为「无样本，以单元测试+代码推演补充」。

### 10.1 P0-1 — 六状态 HTTP / Robots / HTML / 页面内容

对每个样本：`curl -s -w "\n%{http_code}" http://127.0.0.1:3004/website/{domain}`，断言：

| 状态 | 期望 HTTP | 期望 robots meta | 页面内容断言 |
|---|---|---|---|
| 不存在 | **404** | noindex（或默认） | 404 文案 + 分析引导 |
| running（无数据） | 200 | noindex | 「正在分析中」处理中文案 + 刷新链接 |
| failed（无数据） | 200 | noindex | 引导重新分析（不 404） |
| failed（有历史报告） | 200 | noindex | 完整报告渲染（历史不丢） |
| completed | 200 | 依 Gate | 报告渲染 |
| partial | 200 | 依 Gate | 报告渲染 |

### 10.2 P0-2/P0-3 — Gate 一致性（robots 与 sitemap 同判）

样本：热门白名单域名（如 github.com）、`investigationCount ≥3` 域名、不满足 Gate 域名、低价值 CDN/测试域名（如 `example.com`）。

断言矩阵（对每个样本）：
- `/website/{domain}` 抓 HTML → `robots` meta 判定
- `/sitemap.xml` → 该域名**是否在列**
- **结论必须一致**：robots index ⇔ 在 sitemap；noindex ⇔ 不在 sitemap

### 10.3 P0-4 — canonical

```
curl -s http://127.0.0.1:3004/bulk | grep -o 'rel="canonical" href="[^"]*"'
curl -s http://127.0.0.1:3004/docs/api | grep -o 'rel="canonical" href="[^"]*"'
curl -s http://127.0.0.1:3004/report | grep -o 'rel="canonical" href="[^"]*"'
```

断言：`/bulk` → `.../bulk`；`/docs/api` → `.../docs/api`；`/report` → `.../report`（Self Canonical，URL 形态随 NEXT_PUBLIC_SITE_URL 为 `http://127.0.0.1:3004/...`；语义验证路径段）。

### 10.4 P0-5 — metadata

对 `/bulk` `/docs/api` `/report` 抓 HTML，断言四要素存在且非空、非首页复用：
- `<title>`：对应页面标题（非首页默认）
- `<meta name="description">`：对应页面 metaDescription（zh 默认语言下为中文；`?lang` 或 cookie 切换 en 复验英文）
- `<meta name="robots">`：bulk=noindex / docs/api=**index** / report=noindex
- canonical：见 10.3

### 10.5 P0-6 — Sitemap 污染检查

```
curl -s http://127.0.0.1:3004/sitemap.xml
```

断言**不存在**：`example.com`、任何 `*.tiles.virtualearth.net`、任何 CDN 基础设施后缀（`.akamaiedge.net`/`.cloudfront.net`/`.azureedge.net`/`.fastly.net` 等）、任何 10.2 中 robots=noindex 的 website URL。

### 10.6 辅助验证

- `/robots.txt` 与生产一致（disallow 规则未变）
- `curl http://127.0.0.1:3004/` 首页 200 + title 正常
- staging 日志无异常刷屏（调度器写拒绝噪音记录后上报）

---

## 待批准事项汇总

| # | 事项 | 类型 |
|---|---|---|
| 1 | 本地 `git commit` 到 `seo/phase-1-p0`（不进 remote/main） | 代码操作 |
| 2 | 代码直传服务器 + 目录/systemd/构建（§8 [2][3][5][6][7][8]） | 服务器配置（新增物，零生产触碰） |
| 3 | 创建只读角色 `siteintel_staging_ro` + GRANT（§6 主方案） | **DB 配置变更（零数据变更）** |
| 4 | 执行验证矩阵（§10，含只读 SELECT 取六状态样本） | 验证 |
| 5 | （可选）阶段二 preview.siteintel.cc 子域名 | 独立批准 |

**本方案未执行任何创建/修改/连接操作。等待批准后进入部署。**
