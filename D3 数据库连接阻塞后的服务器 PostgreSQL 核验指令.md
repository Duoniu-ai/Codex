# D3 数据库连接阻塞后的服务器 PostgreSQL 核验指令

当前 D3 前置核验已经确认：

- 本地 `localhost:5432` 无 PostgreSQL 实例
- `DATABASE_URL` 当前只是本地占位连接串
- D3 暂不允许执行
- 禁止修改 Prisma Schema
- 禁止执行 migration
- 禁止 `prisma db push`
- 禁止 `CREATE / ALTER / DROP`
- 禁止 INSERT / UPDATE / DELETE
- 禁止修改 `.env`
- 禁止修改冻结文档

现在不要继续 D3。

## 第一阶段：只确认真实数据库服务器位置

根据项目现有配置、部署配置、文档以及服务器信息，确认项目实际 PostgreSQL 应该部署在哪里。

当前已知：

- 项目数据库名称：`nav_disc`
- 数据库用户：`nav_disc_user`
- 本地 `localhost:5432` 不存在 PostgreSQL
- 冻结版附录 A.5 已注明真实实例应在 154 服务器（或指定目标主机）

如果 154 服务器就是生产/目标服务器，请以 154 服务器为目标进行后续核验。

## 第二阶段：只读检查 154 服务器 PostgreSQL

连接服务器后，只允许进行只读检查。

检查：

1. PostgreSQL 服务是否运行
2. PostgreSQL 实际监听地址
3. PostgreSQL 实际监听端口
4. PostgreSQL 版本
5. 是否存在 `nav_disc` 数据库
6. 是否存在 `nav_disc_user`
7. `nav_disc_user` 是否能够连接 `nav_disc`
8. `nav_disc_user` 的数据库级权限
9. `public` schema 权限
10. 当前数据库的 schema
11. 当前已有业务表
12. 每张表 owner
13. 是否存在 `prisma_migrations`
14. migration 历史
15. 是否存在与目标 24 表同名的现有表
16. 当前数据库是否为空数据库

## 第三阶段：权限核验

必须实际通过 PostgreSQL 查询确认以下权限，而不是推测：

- CREATE DATABASE（如果该权限属于实际 D3 建库流程）
- CONNECT
- public schema CREATE
- CREATE TABLE
- CREATE INDEX
- INSERT
- UPDATE
- DELETE

同时确认：

- `nav_disc_user` 是否能够成为目标表 owner
- 是否存在其他角色/owner 导致 Prisma migration 无法正常执行

## 第四阶段：严格禁止修改

本次核验期间：

禁止：

- `CREATE DATABASE`
- `CREATE TABLE`
- `CREATE INDEX`
- `ALTER`
- `DROP`
- `TRUNCATE`
- `INSERT`
- `UPDATE`
- `DELETE`
- `GRANT`
- `REVOKE`
- `prisma migrate`
- `prisma migrate deploy`
- `prisma migrate dev`
- `prisma db push`
- `prisma db execute`
- seed
- 任何结构修改

只允许 SELECT、系统信息查询以及等价的只读诊断命令。

## 第五阶段：如果 154 服务器 PostgreSQL 已存在

不要立即创建数据库，也不要修改数据库。

先输出完整核验结果：

### A. PostgreSQL 实例

- 主机：
- 端口：
- PostgreSQL 版本：
- 服务状态：
- 监听状态：

### B. 数据库

- `nav_disc` 是否存在：
- 数据库 owner：
- encoding：
- collation：
- 当前 schema：

### C. 用户

- `nav_disc_user` 是否存在：
- 是否允许 LOGIN：
- 是否可以 CONNECT：
- 是否为 superuser：
- 是否为 CREATEDB：
- 是否为 CREATEROLE：

### D. 权限

逐项输出：

- database CONNECT
- schema CREATE
- table CREATE
- index CREATE
- INSERT
- UPDATE
- DELETE
- owner 能力

### E. 当前结构

输出：

- public schema 中全部表
- 每张表 owner
- `prisma_migrations` 是否存在
- migration 数量
- migration 名称/状态

### F. 24 表冲突

将目标 D3 的 24 张表逐一检查：

| 目标表 | 是否已存在 | Owner | 是否存在潜在冲突 |
|---|---|---|---|

不要修改任何对象。

## 第六阶段：如果 154 服务器没有 PostgreSQL

也不要自行安装、创建数据库或修改服务器。

只报告：

1. PostgreSQL 是否完全不存在
2. 是否存在其他数据库服务
3. 项目原本预期的 PostgreSQL 部署方式
4. 当前需要由我提供/确认什么信息
5. D3 需要的最低数据库环境

## 最终输出

最后只给出：

# D3 数据库环境核验报告

并明确判断：

- `BLOCKED`
- 或 `READY_FOR_D3`

只有在以下全部满足时才允许标记 `READY_FOR_D3`：

1. 真实 PostgreSQL 实例可连接
2. `nav_disc` 已确认
3. `nav_disc_user` 已确认
4. 权限已实际查询确认
5. 当前数据库结构已确认
6. migration 状态已确认
7. 24 表冲突检查已完成
8. 没有发现会阻塞 D3 的数据库层问题

在此之前，**绝对不要执行 D3。**

另外，整个过程保持工作区 clean，不修改 `.env`、Prisma Schema、migration、冻结文档或任何业务代码。