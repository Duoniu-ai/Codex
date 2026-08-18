# SITEINTEL SEO PHASE 1 — STAGING PREPARATION (PHASE A) REPORT

- **日期**: 2026-08-17
- **批准依据**: STAGING VERIFICATION PLAN 审阅通过 + Phase A 执行授权（STAGING PREPARATION — PHASE A）
- **范围**: 本地 commit + 独立 staging 目录 + 独立 systemd + 独立 build + 服务启动 + 3004 验证 + 数据库只读审计
- **边界遵守**: 未创建 PG 角色 / 未 GRANT-REVOKE / 未改生产库 / 未写库 / 未 pg_dump / 未创建 staging 库 / 未建 preview.siteintel.cc / 未改 nginx / 未 push / 未 merge / 未部署生产

---

## 1. 本地 Commit

| 项 | 值 |
|---|---|
| Hash | **`c74dfd17dc559b4ba18812aef24a98ea3d72c3b8`**（`c74dfd1`） |
| Message | `feat(seo): implement SEO Phase 1 P0 fixes` |
| 分支 | `seo/phase-1-p0`（基于 `main @ b52ec84`；**未 merge、未 push**） |
| 文件 | 14 个（7 修改 + 7 新增），+831/−33 |
| 变更文件 | `src/app/{bulk,docs/api,layout,report,sitemap}/page.tsx|sitemap.ts|layout.tsx`、`src/app/website/[domain]/{page.tsx,not-found.tsx}`、`src/lib/i18n.ts`、`src/lib/seo/{index-gate,report-state,low-value-domain}.ts` + 对应 3 个测试文件 |
| Commit 前检查 | `git status` 确认仅 P0 文件（无杂散）；`git diff` + 新增文件全文 grep 敏感信息（154.204.176.66/password/api_key/token/secret/bearer/wwwroot/私钥）→ **零匹配** |
| Commit 后 `git status` | **工作树干净**（无残留） |

**Secret 保证**: 未提交 .env、密钥、Token、密码、服务器私有信息（grep 实证零匹配；`.env` 由 .gitignore 排除，git archive 不包含）。

## 2. Staging 目录

```
/www/wwwroot/siteintel.cc-staging/
├── src/  prisma/  scripts/  config（git archive HEAD=c74dfd1 传输，2.1 MB，14 顶层项）
├── .env          （独立构造，600 权限，见 §3）
├── .next/        （staging 独立 build 产物）
├── node_modules/ （pnpm install 全新安装）
└── staging.out   （systemd 日志）
```

- 传输方式: `git archive --format=tar.gz HEAD`（仅 tracked 文件 = GitHub FILESET 同构，不含 .env/.git/node_modules/.well-known）→ scp → 解压
- 与生产 `/www/wwwroot/siteintel.cc` 完全独立（生产目录零触碰）

## 3. 环境变量（独立构造，非复制生产 .env）

| 键 | staging 值 | 说明 |
|---|---|---|
| `DATABASE_URL` | `postgresql://staging_placeholder:...@127.0.0.1:5432/staging_placeholder` | **占位凭据**（未配置任何真实凭据；连接时被 PG 认证拒绝——预期） |
| `NEXT_PUBLIC_SITE_URL` | `http://127.0.0.1:3004` | canonical/sitemap 基址 |
| `AUTH_SECRET` | 新生成随机值（`openssl rand -base64 32`） | 独立会话签名 |
| `ADMIN_EMAILS` | `gaoye2020@gmail.com` | admin 访问控制 |
| `AUTO_DISCOVER` | `0` | 禁发现引擎 |
| `DISCOVER_DAILY_CAP` | `0` | 兜底 |

- **TELEGRAM_BOT_TOKEN: 已移除**（staging 不告警）
- 未复制生产 .env；生产密码未出现在终端输出、报告或任何本地文件（服务器端以环境变量方式传递，不回显）
- 权限 `-rw-------`（600）

## 4. Systemd Service 状态

```
/etc/systemd/system/siteintel-staging.service
├── WorkingDirectory=/www/wwwroot/siteintel.cc-staging
├── EnvironmentFile=/www/wwwroot/siteintel.cc-staging/.env
├── ExecStart=/root/.nvm/versions/node/v20.20.0/bin/node .../next start --hostname 127.0.0.1 --port 3004
├── Restart=on-failure, RestartSec=5
├── StandardOutput/Error=append:/www/wwwroot/siteintel.cc-staging/staging.out
└── 状态: ● active (running), PID 672915, next-server v16.3.1
```

- 监听: `127.0.0.1:3004`（`ss -tlnp` 实证，**仅本机**，零公网暴露）
- 未 enable（不随开机自启——staging 为临时验证环境，符合预期）
- 生产 `siteintel.service` 单元未修改

## 5. Build 结果

```
pnpm install --frozen-lockfile  → PASS（postinstall 自动 prisma generate，v6.19.3）
npx prisma generate            → PASS
pnpm build                     → PASS
```

- 全路由编译：`/website/[domain]`、`/sitemap.xml`、`/bulk`、`/docs/api`、`/report` 全部 **ƒ (Dynamic)**（force-dynamic，构建期不连库——占位 DATABASE_URL 不阻塞 build，与验收复验一致）
- 依赖: node v20.20.0（SSH PATH 显式导出）/ pnpm 10.29.2；pnpm-workspace.yaml 放行 Prisma 构建脚本（esbuild 被 pnpm 忽略——已手动执行 install.js 兜底）

## 6. 127.0.0.1:3004 验证结果

| 探测 | 结果 | 结论 |
|---|---|---|
| 端口监听 | `127.0.0.1:3004`（next-server pid 672915） | ✅ 服务启动成功 |
| `GET /robots.txt` | **HTTP 200** | ✅ 服务可响应（静态规则，不依赖 DB） |
| `GET /`（首页，读 DB） | **HTTP 500** | ⚠️ 预期：`PrismaClientInitializationError` — `staging_placeholder` 凭据被 PG 拒绝（占位 URL 按设计不提供真实凭据） |

**按 Phase A 预案**: staging 因缺少数据库无法验证数据页面 → **停止数据验证，转数据库只读审计**（§8）。HTTP 层/进程层验证通过：服务启动成功、端口仅本机、静态资源可响应。

## 7. 生产服务状态确认

| 项 | 结果 |
|---|---|
| `systemctl is-active siteintel` | **active**（未被停止/重启） |
| 生产 3003 `/robots.txt` | HTTP 200 ✅ |
| 生产 HEAD | `1bdfcb4`（未动） |
| 生产 git status | 与基线期完全一致（.user.ini staged 删除 / .gitignore M / 大写 baidu D / 若干 untracked 文档——均为 08-17 基线建立前的已知项；`DISCOVERY-CANDIDATE-FILTER-DESIGN.md` M 经 mtime 实证为 **08-16 10:46** 的 Discovery F1 残留，与本次无关） |
| 生产 nginx | 未修改 |

## 8. 数据库依赖情况

**现状**: staging 数据页（首页/website/sitemap 等）依赖 PostgreSQL；当前占位凭据 → 认证拒绝 → 数据页 500。需要 Phase B 提供真实只读访问后方可执行验证矩阵。

**只读审计实证（2026-08-17，全部为 SELECT/SHOW 级只读查询，零写入）**:

| 项 | 实证值 | 对方案的影响 |
|---|---|---|
| PG 版本 | **14.23**（Ubuntu 22.04，pg_dump 14.23 同版本可用） | 支持 `default_transaction_read_only`（9.5+）；pg_dump/restore 版本匹配 |
| siteintel 库大小 | **37 MB**（public schema 32 MB） | 数据规模极小——dump/恢复均在秒级 |
| 最大表行数 | EntityRelationship 5,831 / Evidence 3,948 / RawCollection 3,661 / Entity 3,216 / Snapshot 2,996 / Signal 2,385 / Fact 2,294 / DiscoveryCandidate 924 / Investigation 518 / Target 281 | 全库轻量，任何方案都零压力 |
| 可登录角色 | `postgres`（superuser）、`siteintel`（**全部表 owner，完全读写**）、`ip_query_user`（属 ip_query_db 库，与 siteintel 库无关） | **当前无任何只读角色**；siteintel 凭据不可直接用于 staging（写端点会真实写生产库） |
| pg_hba | `127.0.0.1/32 → scram-sha-256`（staging 连接路径）；`0.0.0.0/0 → md5`（既有外部面，不在本次范围） | 新建角色 + 密码可经 127.0.0.1 scram 认证 |
| 磁盘 | 48G 可用（51% 使用） | 两方案均零压力 |
| 同实例其他库 | ip_query_db 11 GB / backup_check 2,958 MB（他项目，不涉及） | 快照方案需注意目标库名唯一 |

## 9. 两种数据库方案比较

| 维度 | 方案 1：专用只读角色 + `default_transaction_read_only=on` | 方案 2：pg_dump 快照 + 独立 staging 库 |
|---|---|---|
| 操作 | `CREATE ROLE siteintel_staging_ro` + `GRANT SELECT ON ALL TABLES` + `ALTER ROLE ... SET default_transaction_read_only=on` | `pg_dump 生产 \| pg_restore 到新库 siteintel_staging`（+ 同样需建角色供 staging 连接） |
| 执行耗时 | **~2 分钟**（纯配置语句） | ~10-15 分钟（dump+restore+建库+角色） |
| 对生产影响 | 零锁、零数据变更（仅元数据授权）；生产连接不受影响 | 一致性快照读（repeatable read）；37 MB 秒级完成，几乎无感 |
| 数据实时性 | **实时**（staging 与生产同实例） | 静态快照（验证样本随生产变化而过期；running/failed 状态以 dump 时刻为准） |
| 写防护 | 双重硬约束（仅 SELECT + 事务级只读）——**一切写端点被 PG 拒绝** | 需单独配置（若新库只读角色则同方案 1；若给写权限则快照库可被污染——不推荐） |
| 磁盘 | 零 | ~100 MB（dump 文件 + 新库） |
| 风险 | 新增一个带密码可登录角色（随机密码、600 权限 .env、仅 127.0.0.1 scram） | dump 文件短暂存于磁盘（可即时清理）；快照过期需刷新 |
| 适用场景 | 短期真实数据验证（本场景） | 长期隔离环境 / 需要自由写操作的压力测试 / 大数据量避免共享实例负载 |

## 10. 推荐下一步方案

**推荐：方案 1 — 专用只读角色 `siteintel_staging_ro` + `default_transaction_read_only=on`**

依据:
1. **数据规模决定一切**：siteintel 库仅 37 MB、最大表 5.8k 行——方案 2 的隔离优势（大数据量 dump 耗时/共享实例负载规避）在本规模下完全不存在，dump+restore 只带来"数据过期"缺陷（验证矩阵需要真实的 running/failed 状态样本，静态快照不满足）
2. **侵入最小**：方案 1 是配置级操作（零锁、零数据变更）；方案 2 复制数据（虽只读但多一步数据搬运）
3. **写防护最强**：双重硬约束（GRANT SELECT + 事务级只读）下，staging 的 `/api/analyze`、claim、dashboard 等一切写端点被 PG 直接拒绝——比应用层屏蔽可靠；方案 2 若达到同等防护仍需要同样建一个只读角色
4. **实时性**：验证时 running/failed/completed 状态以生产实时为准，无刷新成本
5. 认证可行性已实证：127.0.0.1/32 scram-sha-256，新建角色即插即用

**Phase B 待批准操作（单次，~2 分钟）**:
```sql
CREATE ROLE siteintel_staging_ro LOGIN PASSWORD '<随机生成>';
GRANT USAGE ON SCHEMA public TO siteintel_staging_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO siteintel_staging_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO siteintel_staging_ro;
ALTER ROLE siteintel_staging_ro SET default_transaction_read_only = on;
```
+ staging `.env` 的 `DATABASE_URL` 替换为该角色（密码仅写入 staging .env 600 权限文件，不回显）→ `systemctl restart siteintel-staging` → 执行验证矩阵。

---

## 总结

- Phase A 全部目标达成：commit `c74dfd1` ✅ / staging 目录 ✅ / systemd 服务 active ✅ / build PASS ✅ / 127.0.0.1:3004 进程与 HTTP 层可达 ✅
- 生产零影响实证：siteintel active、3003 正常、HEAD 1bdfcb4 未动、生产目录无新增修改
- 数据库依赖：按预案停止数据验证，只读审计完成（37 MB / 无只读角色 / scram 认证可行 / pg_dump 可用 / 磁盘充足）
- **A 判定：当前无安全的只读访问方式**（siteintel 是表 owner 具完全写权限，不可直接用于 staging）
- **推荐：方案 1**（只读角色 + 事务级只读），需 Phase B 批准后执行

**完成，停止。等待 Phase B 批准。**
