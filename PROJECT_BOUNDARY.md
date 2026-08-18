# PROJECT BOUNDARY — 网址发现平台（网址导航站）

> 独立产品项目边界声明。违反本文件的任何操作均为越界。
> 更新日期:2026-08-17

## 边界声明

1. **本项目是独立产品**（网址发现平台 / 网址导航站）,不是 SiteIntel 子项目;
2. **SiteIntel 是上游数据来源**,不是本项目模块;
3. 本项目只允许通过**稳定 API / 同步契约**访问 SiteIntel（`GET /api/v1/report/{domain}?lang=zh` + navigation-sync 专用 Key）;
4. **禁止直接访问 SiteIntel PostgreSQL**;
5. **禁止共享 Prisma Schema**（本项目独立 schema）;
6. **禁止复制 SiteIntel 业务代码**;
7. 本项目拥有独立:
   - repository;
   - PostgreSQL database（nav_disc,独立库独立用户）;
   - Prisma schema;
   - deployment（systemd `navigation`,端口 3010）;
   - environment variables（Key 仅存服务器 .env,前端永不接触）;
8. 单向同步:SiteIntel → 稳定 API → 同步服务 → 本项目独立库;禁止反向耦合。

## 当前产品阶段

**API 契约设计与接口冻结（候选版）** — 2026-08-17

已完成:
- 产品规划 V2.1.2 冻结
- SiteIntel 上游数据能力审计（PASS WITH 2 GAPS）
- 技术架构 V1.0-rev2（与 Schema 冻结版完成基线同步,候选版）
- 数据库 Schema 与数据同步契约 V1.0 冻结（24 表,验收 PASS）
- 基线文档整理入库（docs/product、docs/architecture、docs/audit）

**尚未允许:**
- 创建 schema.prisma;
- 执行 Prisma migrate;
- 创建 API 代码（route.ts）;
- 创建业务页面 / 前端业务代码;
- 创建 Worker 实现;
- 创建 PostgreSQL 数据库。

## 红线（永久）

- 不修改 Schema V1.0 冻结版（变更走:变更提案 → 影响分析 → 新版本号 → 审计 → 新冻结版）;
- 不修改产品 V2.1.2 冻结版;
- 不修改 SiteIntel;
- 不引入 Redis / Elasticsearch / BullMQ / Docker / 微服务 / 复杂 RBAC / 账号体系;
- API 设计发现 Schema 不满足需求 → 登记 Schema Change Request,不直接改 Schema。
