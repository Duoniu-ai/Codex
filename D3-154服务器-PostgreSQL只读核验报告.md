# D3-154服务器-PostgreSQL只读核验报告

> 核验日期:2026-08-17
> 执行依据:《D3 154 Linux PostgreSQL 真实实例只读核验指令》(用户下发,六阶段/十五节)
> 执行方式:全部只读(SELECT、系统信息查询、等价只读诊断);未执行任何 CREATE/ALTER/DROP/GRANT/REVOKE/migrate/seed,未修改 `.env`、schema.prisma、migration、冻结文档、业务代码;无临时脚本残留,工作区 clean
> 结论:**BLOCKED**

---

## A. 当前服务器身份

| 项 | 值 |
|---|---|
| 本地环境 | Windows 11(MINGW64,hostname `Pc`)— **非 154** |
| 核验方式 | SSH 进入 154:`ssh root@PRIVATE_SERVER -p REDACTED_PORT -i ~/.ssh/PRIVATE_KEY`,全部检查命令在 154 服务器上执行 |
| 服务器 hostname | `PRIVATE_HOSTNAME` |
| 服务器 IP | PRIVATE_NETWORK(内网;公网 SSH 入口 PRIVATE_SERVER) |
| 内核 | Linux 5.15.0-187-generic #197-Ubuntu SMP 2026-07-17 x86_64 GNU/Linux |

## B. PostgreSQL 实例信息

| 项 | 值 |
|---|---|
| 主机 / 端口 | PRIVATE_SERVER / 5432 |
| 服务状态 | `postgresql.service` **active**(enabled),Active since 2026-08-12 15:01:22 UTC |
| 集群 | `14/main` online,端口 5432,数据目录 /var/lib/postgresql/14/main |
| 监听 | `0.0.0.0:5432` + `[::]:5432`(进程 postgres pid=1038) |
| 版本 | PostgreSQL **14.23**(Ubuntu 14.23-0ubuntu0.22.04.1,x86_64-pc-linux-gnu,gcc 11.4.0) |

## C. nav_disc 数据库状态

- 是否存在:**否**。`pg_database` 全库列表:`backup_check`(owner postgres)、`ip_query_db`(owner PRIVATE_DB_ROLE)、`postgres`、`siteintel`(owner siteintel)、`template0`、`template1` — **无 nav_disc**
- 连接实测:`psql -d nav_disc` → `FATAL: database "nav_disc" does not exist`(证据确凿)
- owner / encoding / collation:不可评估(库不存在;实例数据库均为 UTF8 / C.UTF-8)

## D. PRIVATE_DATABASE_USER 状态

- 是否存在:**否**。`pg_roles` 查询 `rolname='PRIVATE_DATABASE_USER'` 无任何输出
- LOGIN / SUPERUSER / CREATEDB / CREATEROLE:不可评估(角色不存在,非推测,为查询返回空)
- 实例现有角色:postgres(超管)、siteintel、PRIVATE_DB_ROLE、PRIVATE_DB_ROLE

## E. 数据库权限

- `has_database_privilege('PRIVATE_DATABASE_USER','nav_disc',CONNECT/CREATE/TEMP)`:查询返回 **NULL/空**(角色与库均不存在,无权限对象可查)——权限**无法实际确认**,这是对象缺失的必然结果,非推测
- CREATE DATABASE 能力:PRIVATE_DATABASE_USER 不存在无从谈起;postgres 超管具备,siteintel 独立库先例可循

## F. public schema 权限

- 各现有库 public schema owner 均为 `postgres`(backup_check / ip_query_db / postgres / siteintel)
- PRIVATE_DATABASE_USER 对 nav_disc.public 的 USAGE/CREATE:无法查询(库/角色不存在)
- 先例:siteintel 库 53 表 owner 全部 = `siteintel`(库 owner = 角色 owner),独立库独立用户模式已在生产验证

## G. 当前表状态

| 库 | public 表数 | 说明 |
|---|---|---|
| backup_check | 4 | — |
| ip_query_db | 11 | ip_asn/ip_geo/ip_query_history/ip_risk/page_visit_ips/unified_ip_info/webrtc_leak_logs(owner PRIVATE_DB_ROLE)+ admin_users/permissions/role_permissions/roles(owner postgres) |
| postgres | 0 | 空 |
| siteintel | 53 | 全部 owner = siteintel |

## H. Prisma Migration 状态

| 库 | `_prisma_migrations` | 迁移记录 |
|---|---|---|
| backup_check | 否 | — |
| ip_query_db | 否 | — |
| postgres | 否 | — |
| siteintel | **是** | **7 条,全部 finished_at 非空、rolled_back_at 全 NULL**:0001_init → 20260815_monitor_last_run_at → 20260815_phase1_evidence_facts_signals_contradictions → 20260815_phase10_domain_ownership → 20260815_phase12_api_usage_quota → 20260816_organization_alias → 20260816_organization_alias_phase_a(2026-08-15 ~ 08-16) |

> siteintel 为 Prisma 管理的独立库,与本项目 nav_disc 无关联。

## I. 24 表冲突检查(以冻结版表名 snake_case 为准)

- **nav_disc 库内冲突:不适用**(库不存在,空库创建无任何冲突对象)
- **跨库同名表 1 张**:`ip_query_db.admin_users`(owner=postgres,11 列:integer id / username / email / password_hash / role_id / status / 时间戳 ×3 / last_login_ip)——为 IPIP0 查询站(Laravel)业务表,与目标 `admin_users`(5 字段:text CUID id)同名但**不同库、不同结构**;**不构成 D3 首迁冲突**(不同数据库对象互不干扰)
- 其余 23 张目标表在全部现有库中**零重名**
- **实现偏差(重要发现,需用户裁决)**:D2 对照清单声明"模型名 PascalCase 映射表名经 `@@map` 与冻结版一致",但当前 `prisma/schema.prisma` **实际没有任何 `@@map` 指令**(全文件仅注释提及)。若 D3 直接首迁,Prisma 将按模型名创建 `"Websites"` 等 **PascalCase 表名**,与冻结版 snake_case 表名**不一致**。属 D2 实现偏差,本次核验仅登记报告,未自行修改任何文件。

## J. D3 执行条件(对照 READY_FOR_D3 13 条件)

| # | 条件 | 状态 |
|---|---|---|
| 1 | 当前环境确认是 154 Linux | ✅ SSH 核验 |
| 2 | PostgreSQL 服务正常 | ✅ active/enabled |
| 3 | 实际监听正常 | ✅ 0.0.0.0:5432 + [::]:5432 |
| 4 | `nav_disc` 存在 | ❌ 不存在 |
| 5 | `PRIVATE_DATABASE_USER` 存在 | ❌ 不存在 |
| 6 | `PRIVATE_DATABASE_USER` 可连接 `nav_disc` | ❌ 库不存在 |
| 7 | database CONNECT 已确认 | ❌ 不可查(对象缺失) |
| 8 | public schema CREATE 已确认 | ❌ 不可查(对象缺失) |
| 9 | D3 所需数据库权限已确认 | ❌ 不可查(对象缺失) |
| 10 | 当前 public schema 表结构已确认 | ✅ 逐库列出 |
| 11 | `_prisma_migrations` 状态已确认 | ✅ 逐库确认 |
| 12 | 24 表冲突检查完成 | ✅ 零冲突(跨库同名记录在案) |
| 13 | 无数据库层面 D3 阻塞 | ❌ 库/角色缺失 |

## K. 最终结论

**BLOCKED**

- 154 服务器 PostgreSQL 14.23 实例**完全就绪**(服务/监听/版本/先例/零冲突);
- 但 `nav_disc` 数据库与 `PRIVATE_DATABASE_USER` 角色**均未创建** → 条件 4-9 不满足,不满足 READY_FOR_D3 全部条件;
- 附加裁决项(schema.prisma 无 `@@map` 实现偏差)不影响本轮 BLOCKED 结论,但应在 D3 首迁前裁决。

## L. 唯一阻塞项

> **nav_disc 数据库与 PRIVATE_DATABASE_USER 角色未创建**(需用户授权后由 D3 正式阶段创建)

等待用户提供:
1. 创建方式:授权 D3 阶段通过 `sudo -u postgres` 执行 `CREATE DATABASE nav_disc OWNER PRIVATE_DATABASE_USER` + `CREATE ROLE PRIVATE_DATABASE_USER LOGIN PASSWORD '...'`(与 siteintel 先例一致),或由用户自行创建;
2. 密码策略:我生成强随机密码注入服务器 `.env`,或用户提供;
3. schema.prisma `@@map` 偏差处置:D3 首迁前修复(补 24 行 `@@map`)或维持现状(需用户批准修改实现,属 D2 范围外变更登记)。

**本轮仅只读核验,未执行 D3。**
