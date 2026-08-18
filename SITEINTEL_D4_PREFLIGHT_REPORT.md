# SITEINTEL_D4_PREFLIGHT_REPORT

> 项目:网址发现平台(网址导航站)— 独立产品,SiteIntel 仅为上游数据源
> 预检日期:2026-08-18(+0800)
> Git HEAD:038492d2672b25f085fa3ba90db3c3b1f269907e(D3 提交后,worktree clean)
> 本轮性质:**只读预检**,未修改任何代码/数据库/`.env`,未执行 migration/seed,未提交 Git

---

# A. D4 定义

## D4 名称

**SiteIntel 数据同步实现**(P0 开发任务拆解 V1.0 冻结版,D4,与 D5 并行)

## D4 目标

实现 SiteIntel → 导航站**单向同步业务逻辑**:消费契约 / normalize / 字段映射裁剪 / summary_hash / 持久化顺序 / 节流。

## D4 输入(冻结文档)

- Schema 冻结版 §5(normalize 规则)、§11(同步契约+映射表)、§12(summary_hash)、§13(变化信号)
- API 契约 §8(SiteIntel 消费契约)、§9(内部契约函数签名)
- 技术架构 §4(同步架构)

## D4 输出

- `src/lib/sync` 下纯函数模块:SiteIntel 客户端、normalize、映射裁剪、summary_hash、持久化顺序、令牌桶节流
- 对应单元测试;D4 DoD:同步链路可跑通 + 边界测试通过 + 禁止持久化清单冒烟通过

## D4 涉及数据库对象(读/写)

`websites` / `website_snapshots` / `website_status` / `website_metrics` / `website_changes` / `sync_queue`(读入队状态)

## D4 涉及代码模块

仅 `src/lib/sync`(纯函数优先)。禁止触碰 `src/lib/worker`、`src/lib/queue`、`src/instrumentation.ts`(D5 范围)。

## D4 涉及 API / service

- 出站:SiteIntel `GET /api/v1/report/{domain}?lang=zh` + `X-API-Key: si_<navigation-sync Key>`;404 → 出站 `POST /api/v1/analyze`(30/h 硬顶)
- 无 HTTP 入站端点(为 D6 供数);内部契约按 API 契约 §9 签名语义

## D4 验收标准(冻结)

折叠/hash/映射单测全绿;单域名端到端同步(手动入队 → 拉取 → 裁剪 → 落库 → hash 对比 → 零写验证);同步后库中无禁止持久化字段。

## D4 与 D3 的依赖关系

D4 前置依赖 D3(数据库结构/seed 就绪)。D4 不依赖 D5;与 D5 并行,通过 API 契约 §9 签名对接。唯一执行闸门:**navigation-sync 专用 Key 未创建则 D4 挂起等待**(冻结任务书 D4.2)。

---

# B. 当前状态

## 数据库(2026-08-18 实时只读复核,154 服务器 nav_disc)

| 对象 | 状态 |
|---|---|
| D4 涉及表 | websites=150 行;website_snapshots / website_status / website_metrics / website_changes / sync_queue / sync_logs = 0 行(待同步建立) |
| 表结构 | 24 表 / 221 字段 / snake_case,camelCase 列 0 |
| 约束 | PK 24 / FK 25 / CHECK 30 / UNIQUE 16 / INDEX 31,与冻结版一致 |
| sync_queue | 17 字段结构正确,含 UNIQUE(normalized_domain, job_type) |
| settings | 7 键齐全,含 `sync_queue.max_attempts` |
| migration | `20260817_init` 成功记录 applied,checksum 与本地 migration.sql 逐字节一致;无 drift |

## 代码

| 对象 | 状态 |
|---|---|
| `src/lib/sync` | 目录存在,0 文件(D4 落点,空待实现) |
| `src/lib/queue` / `src/lib/worker` | 目录存在,0 文件(D5 落点,未实现) |
| `src/instrumentation.ts` | 不存在(D5 落点) |
| 现有 lib | `api`(errors/pagination)、`auth`、`rate-limit`(sliding-window)已实现,供复用 |
| Prisma client | 已生成(`node_modules/.prisma/client`) |
| 配置 | `.env` 中 `NAVIGATION_SYNC_KEY=` **空占位**(Key 未创建,仅服务器注入) |

## 测试

- 现有测试:`src/lib/api/errors.test.ts`、`pagination.test.ts`、`auth.test.ts`、`rate-limit/sliding-window.test.ts`(D1 基线 22/22 PASS)
- D4 相关测试:无(sync 模块尚未实现)
- 测试运行器:`node --test "src/lib/**/*.test.ts"`(新增测试自动纳入)

---

# C. D4 差异矩阵

| # | D4 Requirement(冻结) | Current State | Target State | Difference | Risk |
|---|---|---|---|---|---|
| 1 | `src/lib/sync` 模块落地 | 目录存在、0 文件 | 4.1-4.6 纯函数模块 + 导出 | 需新建 | LOW |
| 2 | 4.1 SiteIntel 客户端(GET report + X-API-Key + 超时/重试 + 404→analyze) | 不存在 | `client.ts` | 需新建 | LOW |
| 3 | 4.2 normalize 折叠(§5:小写/去协议/去 www/去尾斜杠/去端口/punycode,子域保留) | 不存在 | `normalize.ts` + 边界单测 | 需新建 | LOW |
| 4 | 4.3 字段映射与裁剪(§11.2 白名单;IP 单值;SSL 摘要;tech 仅名称;audit→health_summary;history[] 忽略;evidence.rawData 禁止) | 不存在 | `mapping.ts` | 需新建 | MEDIUM(禁止持久化清单,靠单测 + DoD 冒烟) |
| 5 | 4.4 summary_hash(§12:12 参与 / 7 排除 / canonicalize / SHA-256 hex64) | 不存在 | `summary-hash.ts` | 需新建 | MEDIUM(确定性/误判,靠单测) |
| 6 | 4.5 持久化顺序(§11.3:实体确认→裁剪→hash→diff→snapshots 追加+status upsert+changes+metrics;相同→零写;事务) | 不存在 | `persist.ts` | 需新建 | MEDIUM(零写/事务边界,靠集成验证) |
| 7 | 4.6 节流(内存令牌桶:analyze 30/h 硬顶 + report 配额,耗尽不阻塞) | 现有 rate-limit 为 API 滑动窗口,非令牌桶 | `throttle.ts`(sync 内) | 需新建 | LOW |
| 8 | D4 单测 + e2e 验收 | 无 sync 测试 | 边界单测 + 单域名 e2e + 禁止字段冒烟 | 需新建 | LOW |
| 9 | navigation-sync Key | `NAVIGATION_SYNC_KEY` 空占位,Key 未创建 | 服务器 `.env` 注入 `si_<Key>` | **外部前置未就绪** | 执行闸门(未创建则 D4 挂起) |
| 10 | 数据库对象 | 7 张 D4 表结构正确、约束齐、sync 表空 | 维持冻结结构 | 无差异 | LOW |
| 11 | migration | 20260817_init applied,无 drift | 无新 migration | 无差异 | LOW |
| 12 | 无关代码 | worktree clean @ 038492d,无 D5+ 实现 | 仅新增 sync 模块 | 无冲突 | LOW |

---

# D. D4 文件变更范围

| File | Current Role | Expected Change | D4 Required? |
|---|---|---|---|
| `src/lib/sync/client.ts` | 不存在 | 新建:SiteIntel 客户端(§8.1) | YES |
| `src/lib/sync/normalize.ts` | 不存在 | 新建:§5 折叠 | YES |
| `src/lib/sync/mapping.ts` | 不存在 | 新建:§11.2 映射裁剪 | YES |
| `src/lib/sync/summary-hash.ts` | 不存在 | 新建:§12 hash | YES |
| `src/lib/sync/persist.ts` | 不存在 | 新建:§11.3 持久化顺序 | YES |
| `src/lib/sync/throttle.ts` | 不存在 | 新建:令牌桶(4.6) | YES |
| `src/lib/sync/*.test.ts` | 不存在 | 新建:边界单测 | YES |
| `src/lib/api/errors.ts` | 已实现 | 复用,不修改 | NO(只读复用) |
| `prisma/schema.prisma` | 冻结 | 无改动 | NO |
| `prisma/migrations/` | 冻结基线 | 无改动 | NO |
| `src/lib/worker` / `src/lib/queue` / `src/instrumentation.ts` | 空/不存在 | D5 范围,本轮不处理 | NO(记录) |
| `.env` / `.env.example` | 占位 | Key 仅服务器注入,无需改动 | NO |

---

# E. Database Change Plan

**D4 不产生任何数据库结构变更。**

- Migration:无(维持 `20260817_init`)
- 新表 / 新字段 / FK / UNIQUE / INDEX / CHECK:无
- 数据迁移:无
- 仅运行时写入(通过 Prisma client,遵守 §11.3 顺序与事务):websites / website_snapshots / website_status / website_metrics / website_changes;读取 sync_queue 入队状态
- 禁止:ALTER / DROP / DELETE / TRUNCATE / INSERT(手工)/ seed / migration 执行(本轮)

---

# F. Data Compatibility

- 种子数据(150 网站等)与现有 CHECK/FK 兼容;D4 写入路径只需遵守冻结映射与 hash 规则,无历史数据兼容问题
- 上游字段超出枚举/类型边界时,由映射层按冻结表裁剪转换;`website_status` 无 CHECK 枚举字段,风险低
- 零写原则依赖 summary_hash 正确实现;变化信号仅产生于 §12.1 集合内字段

---

# G. Risk Assessment

| 风险项 | 评估 |
|---|---|
| D3 回归 | 低(schema/migration 零改动,仅新增代码) |
| destructive migration | 无 |
| 历史数据兼容 | 低(sync 表为空,seed 数据兼容) |
| schema drift | 无(30/30 CHECK + migration checksum 复核) |
| 未提交修改 | 无(worktree clean @ 038492d) |
| secrets | 无(`.env` 未跟踪;`NAVIGATION_SYNC_KEY` 空占位) |
| 与 D4 无关修改 | 无 |
| 禁止持久化清单泄漏 | 中,靠映射单测 + DoD 冒烟覆盖(泄漏即缺陷) |
| 执行闸门 | navigation-sync Key 未创建 → D4 实施挂起等待(冻结任务书 D4.2) |
| 范围越界 | D4 仅允许 `src/lib/sync`;worker/queue/instrumentation 属 D5,记录不处理 |

**总体 D4 Risk:LOW**(不含外部 Key 依赖;Key 属执行前置而非分析风险)

---

# H. Test Plan

## 单元测试(新增,`node --test` 自动收集)

- normalize:www 折叠 / 子域名不折叠 / 端口去除 / punycode / 尾斜杠 / 协议去除
- mapping:白名单映射逐项;禁止持久化字段(evidence.rawData / IP 明细 / 证书链 / history[] / insights / audit 原样)绝不落库
- summary_hash:12 参与 / 7 排除 / canonicalize 确定性(键序/数组排序/时间格式/数值归一化)/ 输出固定 64 位 hex
- throttle:analyze 30/h 硬顶;令牌耗尽不阻塞其他功能

## 集成 / 验收(D4 DoD,需人工确认)

- 单域名端到端同步:手动入队 → 拉取 → 裁剪 → 落库 → hash 对比 → 零写验证
- 禁止持久化清单冒烟:同步后库中无任何禁止字段
- 事务回滚:同步失败不落脏数据
- 回归:D1 基线 22/22 测试保持全绿

---

# I. Scope Boundary

```text
D4 = 分析完成,尚未执行
D5 = 未执行
D6 = 未执行
D7 = 未执行
D8 = 未执行
D9 = 未执行
D10 = 未执行
```

---

# J. 预检结论

```text
D4 Definition: FOUND(冻结任务拆解 D4)
Current State: READY(执行闸门:navigation-sync Key 未创建)
Database Change Required: NO
Code Change Required: YES(仅 src/lib/sync,新建模块)
Data Migration Required: NO
D4 Risk: LOW
Execution: NOT STARTED
```

> 本报告为预检交付物,未提交 Git(按指令保持只读)。
