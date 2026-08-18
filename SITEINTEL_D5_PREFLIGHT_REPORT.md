# SITEINTEL_D5_PREFLIGHT_REPORT

> 项目:**Websites**(网址发现平台/网址导航站;独立产品,SiteIntel 仅为上游数据源)
> 预检日期:2026-08-18(+0800)
> Git HEAD:20fd42e78b5114db152558f1ceb2e01402b67cbe(D4,worktree clean)
> 本轮性质:**只读预检**;未修改代码/数据库/`.env`,未提交 Git
> 说明:文件名沿用历史规划文档命名,项目正式身份为 **Websites**,不做更名

---

## 1. D5 Definition

| 项 | 内容 |
|---|---|
| D5 名称 | **Sync Queue 与 Worker**(P0 任务拆解 V1.0 冻结版 D5) |
| D5 目标 | 实现 sync_queue 状态机、CAS 领取、锁与超时恢复、指数退避、Worker 调度与断点续跑 |
| D5 输入 | Schema §2.21(sync_queue)/§2.22(sync_logs)/§9(状态机与领取语义);API 契约 §9(Worker 内部契约);架构 §4.2 |
| D5 输出 | `src/lib/queue`、`src/lib/worker`、`src/instrumentation.ts` 实现 + 对应单测 |
| D5 依赖 | D3(与 D4 并行;实际已并行完成,D5 直接叠加 D4) |
| D5 验收标准 | 状态机全转换单测(含第 5 次失败→failed、无第 6 次);CAS 并发领取测试(两 worker 只一个成功);超时恢复两步骤测试 |
| D5 DoD | 队列基础设施独立可验证 + 测试全绿;人工确认状态机边界测试结果 |

**允许修改目录(冻结)**:`src/lib/worker`、`src/lib/queue`、`src/instrumentation.ts`

---

## 2. Current State

### 数据库(2026-08-18 实时只读复核)

| 对象 | 状态 |
|---|---|
| sync_queue | 17 字段,0 行;CHECK `chk_sync_queue_job_type`/`chk_sync_queue_status`;UNIQUE `(normalized_domain, job_type)`;索引 priority / status_locked_at / status_next_retry_at — 与冻结 §2.21/§8 一致 |
| sync_logs | 10 字段,0 行;CHECK `chk_sync_logs_action`/`chk_sync_logs_status` — 与冻结 §2.22 一致 |
| settings | `sync_queue.max_attempts = {"value": 5}`(默认 5,可调) |

### 代码

| 对象 | 状态 |
|---|---|
| `src/lib/queue` | 目录存在,**0 文件**(空壳,待 D5 实现) |
| `src/lib/worker` | 目录存在,**0 文件**(空壳,待 D5 实现) |
| `src/instrumentation.ts` | **不存在**(待 D5 新建;Next 16 自动加载) |
| `src/lib/sync`(D4) | 6 模块 + index + 7 测试文件,全部已提交(`20fd42e`) |
| `next.config.ts` | 最小配置,**无 NEXT_PHASE 构建守卫**(D5.5 需补) |

### 前置状态

- D4 Production E2E = **BLOCKED(HTTP 401)**,单独记录,本轮不处理(见 §12)
- GitHub Push = 未完成(网络问题),**不阻塞本地 D5 开发**

---

## 3. D4 Dependency

| D4 Module | D5 Dependency | Required |
|---|---|---|
| client | Worker `process` 调用 `fetchReport` / `triggerAnalyze`;按 `SyncClientError.kind` 决策 complete/retry/fail | **YES** |
| normalize | 入队校验(域名非法 → pending→failed);mapping 内部复用 | **YES** |
| mapping | `process` 中 `mapReportToCanonical` 裁剪 | **YES** |
| summary-hash | `process`/persist 中 `buildSummaryHash` 零写判定 | **YES** |
| persist | `persistSyncResult` 落库(§11.3 顺序) | **YES** |
| throttle | `SyncThrottle.canAnalyze/canFetchReport` 门控出站;D5 **不得重复实现**令牌桶 | **YES** |

> D4 全部模块只读复用,不修改。

---

## 4. Database Assessment

```text
New table   = NO
New fields  = NO
FK/UNIQUE/INDEX/CHECK = NO(冻结基线已含全部所需结构)
Migration   = NO
Data migration = NO
```

D5 仅运行时读写 `sync_queue` / `sync_logs` 与读 `settings`,全部使用 D3 已落地结构;本轮零写入。

---

## 5. Code Assessment

| 对象 | 现状 | D5 处置 |
|---|---|---|
| `src/lib/queue` | 空目录 | 新建:状态机/CAS/超时恢复/退避/日志 |
| `src/lib/worker` | 空目录 | 新建:tick 调度与 process 执行 |
| `src/instrumentation.ts` | 不存在 | 新建:register() + NEXT_PHASE 守卫 + setInterval tick |
| `next.config.ts` | 无守卫 | D5.5 需在 instrumentation 侧实现构建守卫(最小改动) |
| `src/lib/sync/*`(D4) | 已提交 | 只读复用,不修改 |

---

## 6. Queue / Worker Assessment

- **状态机(§9.1/§9.2)**:6 枚举 `pending/processing/completed/retrying/failed/skipped`(无第七状态);非法转换拒绝(5.1)。
- **CAS 原子领取(§9.3)**:单条条件 `updateMany(status='pending' AND (next_retry_at IS NULL OR next_retry_at<=now()))`,`count==1` 才算成功;**禁止先 SELECT 再 UPDATE**(5.2)。
- **超时恢复(§9.4)**:每 tick 最先执行;步骤 1(attempts+1<max → retrying+next_retry_at)/步骤 2(attempts+1>=max → failed);超时计入 attempts(5.3)。
- **退避(§9)**:`next_retry_at = now() + min(2^attempts × 60s, 4h)`(attempts 为更新后值)(5.4)。
- **调度(5.5)**:`instrumentation.ts` `setInterval` tick(架构 §4.2 每 15 分钟)+ NEXT_PHASE 构建守卫(避免 build 期跑 tick)。
- **幂等/断点续跑(5.6)**:UNIQUE(normalized_domain, job_type) + D4 摘要 upsert + summary_hash 零写;systemd 重启后 processing 超期自动恢复。
- **状态上报(5.7)**:sync_logs 落库(action/status/error/http_status/duration_ms/payload_size)。

---

## 7. Retry / Concurrency Assessment

| 风险面 | 冻结防线 | 评估 |
|---|---|---|
| Worker 重复执行 | CAS 领取 count==1;processing 不再满足领取条件;locked_by=PID | 低(按 §9.3 实现 + 双 worker 单测) |
| 丢任务 | 领取后崩溃 → 10min 超时恢复放回队列 | 低 |
| 重复任务 | UNIQUE(normalized_domain, job_type) + 摘要零写幂等 | 低 |
| 无限重试 | attempts+1 后用更新值判定;>=max(默认 5)→ failed;**第 5 次即 failed,无第 6 次** | 低 |
| poison task | 域名非法→入队即 failed;failed 留待 Admin 人工处置(5.11) | 低 |
| 并发竞态 | 单实例 tick + 原子领取;禁 SELECT-then-UPDATE;多实例复用 locked_by 无需架构变更 | 低(测试强制) |
| 节流重复 | D5 只调用 D4 `SyncThrottle`,不重复实现 | 低 |
| Observability | instrumentation = 调度层(setInterval + 守卫);业务日志进 sync_logs | 中(需新建,注意不吞异常) |

---

## 8. D5 Difference Matrix

| # | D5 Requirement(冻结) | Current State | Target State | Difference | Risk |
|---|---|---|---|---|---|
| 1 | 5.1 状态机(6 枚举+转换表+非法拒绝) | 无实现 | `src/lib/queue/state-machine.ts` | 需新建 | LOW |
| 2 | 5.2 CAS 原子领取(updateMany count==1) | 无实现 | `src/lib/queue/claim.ts` | 需新建 | MEDIUM(竞态,测试强制) |
| 3 | 5.3 超时恢复两步骤 | 无实现 | `src/lib/queue/recover.ts` | 需新建 | MEDIUM(边界 off-by-one) |
| 4 | 5.4 指数退避 min(2^n×60s,4h) | 无实现 | 并入 state-machine/backoff | 需新建 | LOW |
| 5 | 5.5 Worker 调度 + NEXT_PHASE 守卫 | instrumentation 不存在、next.config 无守卫 | `src/instrumentation.ts` | 需新建 | MEDIUM(build 期误跑) |
| 6 | 5.6 幂等与断点续跑 | D3 UNIQUE + D4 零写已就绪 | worker/process 复用 | 组装 | LOW |
| 7 | 5.7 sync_logs 状态上报 | 无实现 | queue/log.ts | 需新建 | LOW |
| 8 | process 同步执行(client→D4 管道→complete/retry/fail) | 无实现 | `src/lib/worker/process.ts` | 需新建 | MEDIUM(错误→状态映射) |
| 9 | 数据库对象 | sync_queue/sync_logs 结构齐全、空 | 无需变更 | 无差异 | LOW |
| 10 | D4 依赖 | D4 已提交可复用 | 只读复用 | 无冲突 | LOW |

---

## 9. Expected File Scope

| File | Purpose | Change Needed |
|---|---|---|
| `src/lib/queue/state-machine.ts` | 6 枚举状态机 + 转换表 + 非法转换拒绝 + 退避公式 | 新建 |
| `src/lib/queue/claim.ts` | CAS 原子领取(updateMany count==1) | 新建 |
| `src/lib/queue/recover.ts` | 超时恢复两步骤(attempts 边界) | 新建 |
| `src/lib/queue/log.ts` | sync_logs 落库(action/status/error) | 新建 |
| `src/lib/worker/process.ts` | 执行同步:调用 D4 client→mapping→hash→persist;按错误分类 complete/retry/fail;404→analyze 分支 | 新建 |
| `src/lib/worker/tick.ts` | 单次 tick:recover → claim → process(节流门控) | 新建 |
| `src/instrumentation.ts` | register():NEXT_PHASE 守卫 + setInterval tick | 新建 |
| 对应 `*.test.ts` | 状态机全转换/双 worker CAS/超时恢复/退避/日志/process 错误映射 | 新建 |
| `next.config.ts` | 若需构建守卫(与 instrumentation 配合) | 最小改动(仅守卫,不涉业务) |

**不触碰**:D4 `src/lib/sync`、schema/migrations、Admin API(D7)、前端(D8/D9)、D10。

---

## 10. Risks

- **高优先级**:CAS 竞态与超时恢复 off-by-one(冻结 §9 已消除歧义,单测强制);build 期 instrumentation 误跑(NEXT_PHASE 守卫)。
- **中优先级**:process 阶段错误→状态机映射(SyncClientError.kind → retry/fail);analyze 分支受 30/h 硬顶。
- **低优先级**:单实例部署下无跨进程竞态;多实例语义已由 locked_by + 原子领取覆盖。
- **外部依赖(记录不处理)**:D4 Production E2E = BLOCKED(HTTP 401)——真实同步链路未验证,不阻塞 D5 本地开发与单测。

---

## 11. Test Plan

- 状态机:全转换表单测(含非法转换拒绝;第 5 次失败→failed、无第 6 次)
- CAS:双 worker 并发领取,只有一个成功(count==1)
- 超时恢复:步骤 1/步骤 2 边界(attempts+1 后 <max / >=max)
- 退避:`min(2^attempts×60s, 4h)` 边界(attempts=0..,4h 封顶)
- process:成功→completed+sync_logs(success);429/5xx/网络→retrying(记 last_http_status);404→analyze 分支(节流门控);attempts 超限→failed;SyncClientError.kind 全覆盖
- tick:recover→claim→process 顺序;无任务时零副作用
- 回归:全量 `npm test`(D4 68 项保持全绿)+ tsc/eslint

---

## 12. Scope Boundary

```text
D5 = PREFLIGHT COMPLETE
D5 Implementation = NOT STARTED
D6+ = NOT EXECUTED
D10 = NOT EXECUTED

D4 Production E2E = BLOCKED
Reason = HTTP 401(单独待处理,不擅自修复)

GitHub Push = NOT DONE(网络问题,不阻塞 D5)
```

> 本报告为预检交付物,按指令**未提交 Git**。
