# SiteIntel Phase 1 — Step 7 Minimal Job System（实施前方案）

> 日期：2026-08-18 ｜ 性质：**只分析，不实施**（未改代码/数据库/生产/历史数据）
> 依据：真实代码（instrumentation/monitoring/search-sync/discovery）+ 生产只读检查（无 cron/systemd timer）
> 原则：不重写调度器；只建立 Job/JobRun 两实体；无 Redis/队列/分布式锁/微服务

---

# 1. 当前 scheduler 全貌

```text
instrumentation.ts register()
  ├── 启动恢复：>1h 的 running Investigation → failed
  ├── initMonitoring()       setInterval 30min（单飞 running 标志）
  ├── initSearchSync()       setInterval 6h（单飞 running 标志）
  └── initDiscoveryCollector() setInterval 15s（单飞 running 标志）
```

- 生产实证：**无 cron、无 systemd timer**（`crontab -l` 与 `systemctl list-timers` 均为空）——全部后台任务为进程内 setInterval。
- 启动延迟：monitor 45s / search 60s / discovery 5min（bootTimer，unref）。
- 失败恢复：仅 Investigation 的启动恢复（>1h running → failed）；调度器自身状态（RUNNING map、内存限流、progress 内存 registry）重启即失。

# 2. Task Matrix

| Task Name | Type | Trigger | Frequency | Owner | Concurrency | Retry | Failure Handling | Observability | Data Mutation | Risk |
|---|---|---|---|---|---|---|---|---|---|---|
| monitor | MONITOR | setInterval 30min | 30min | 进程内 | 单飞（running flag） | 自然重试（下次 tick） | per-monitor try/catch；失败不推进 lastRunAt | console.log | Investigation/Fact/Event/Insight/Monitor.lastRunAt/lastNotifiedAt | 长 tick 跨 tick 边界（历史死循环已修复） |
| search-sync | SEARCH_SYNC | setInterval 6h | 6h | 进程内 | 单飞 | nextSyncAt 6h 后重试 | per-property try/catch + quota_blocked | console.log + SearchSyncJob 表 | Search 行表/Snapshot/Event/Insight/Alert | 依赖第三方凭据；无凭据静默 |
| discovery | DISCOVERY | setInterval 15s | 连续消费 | 进程内 | 单飞 | per-candidate 指数退避 1h→168h，7 次 skipped | per-candidate try/catch；失败写 lastFailureReason | console.log + DiscoveryCandidate 状态 | Candidate/Investigation/Target/Entity | 候选池大时每 tick 全表拉取；重启丢进度 registry |
| seed（discovery 子任务） | DISCOVERY_SEED | 6h 窗口（discovery tick 内） | 6h | 进程内 | 单飞（随 discovery） | 下次窗口重试 | per-source try/catch（degraded） | console.log | DiscoveryCandidate（只入池） | 预算记账依赖 DB 计数 |
| data-quality-scan（未来） | DATA_QUALITY | 手动脚本（esbuild+node） | 按需 | 手动 | 1 | 无 | 手动 | 无（JSON 输出） | 无（只读） | 无 |

# 3. 当前风险

1. **调度器状态全部在内存**：重启丢失 RUNNING map、progress registry、限流窗口；无法回答“上次 tick 是否成功”。
2. **无统一可观测性**：只有 console.log/journald；任务级成功/失败/耗时/错误分类不可查询。
3. **防重复仅靠单进程单飞**：长任务（monitor tick 可达 10-30min）跨 tick 边界时靠 running flag 兜底；若未来多实例或 tick 重叠，无 DB 级约束。
4. **崩溃恢复只覆盖 Investigation**：JobRun 级别无恢复（本方案补齐）。
5. **历史 monitor 死循环**（wordpress.org 141 次空转）已修复（lastRunAt 语义），但无任务级审计记录可回溯。

# 4. Job 定义

```text
Job = 任务定义（唯一 name）
  id / name(unique) / type / description / enabled /
  maxAttempts / config(Json?) / createdAt / updatedAt
```

初始注册（seed）：monitor / search_sync / discovery / data_quality_scan。

# 5. JobRun 定义

```text
JobRun = 一次实际执行
  id / jobId(FK→Job) / status / attempt / startedAt / finishedAt /
  heartbeatAt / errorCategory / errorSummary(截断) / meta(Json?，非敏感计数) / createdAt
```

# 6. 状态模型

```text
queued → running → success
                ↘ failed（attempt < maxAttempts 时下次触发产生新 JobRun）
```

- 最小状态集：queued / running / success / failed（不引入 retrying/cancelled——attempt 字段表达重试语义，scheduler 的自然重试 = 新 JobRun）。
- 状态由唯一写入方更新（claim/complete/fail），禁止业务代码直接改状态。

# 7. Retry 模型

- **scheduler 驱动任务（monitor/search/discovery）**：maxAttempts=1；重试 = 下一个 tick 创建新 JobRun（自然重试，不额外退避——现有语义保持不变）。
- **一次性任务（data_quality_scan）**：maxAttempts=3；失败后由执行器按 attempt+1 重跑（有限重试，不做复杂指数退避）。
- JobRun.attempt 记录实际第几次；超过 maxAttempts → status=failed。

# 8. Concurrency 模型

**明确策略：同一个 Job 同时只允许 1 个 running JobRun**。

- 实现：partial unique index：`CREATE UNIQUE INDEX ... ON "JobRun"("jobId") WHERE status = 'running'`（Prisma 不支持 partial index，通过 migration 原生 SQL 创建）。
- 执行器 claim 流程：`INSERT ... ON CONFLICT DO NOTHING`（或 select-for-update + 唯一索引兜底）→ 冲突则跳过本 tick（等价现有单飞）。
- discovery 的连续消费：采用**会话 JobRun**（进程启动 claim running，每次 tick 心跳，停机/恢复时 complete/fail），避免 15s 级 JobRun 海量行。

# 9. PostgreSQL Schema（migration 提案）

```sql
CREATE TABLE "Job" (
  "id" TEXT NOT NULL,
  "name" TEXT NOT NULL,
  "type" TEXT NOT NULL,
  "description" TEXT,
  "enabled" BOOLEAN NOT NULL DEFAULT true,
  "maxAttempts" INTEGER NOT NULL DEFAULT 1,
  "config" JSONB,
  "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  "updatedAt" TIMESTAMP(3) NOT NULL,
  CONSTRAINT "Job_pkey" PRIMARY KEY ("id")
);
CREATE UNIQUE INDEX "Job_name_key" ON "Job"("name");

CREATE TABLE "JobRun" (
  "id" TEXT NOT NULL,
  "jobId" TEXT NOT NULL,
  "status" TEXT NOT NULL DEFAULT 'queued',
  "attempt" INTEGER NOT NULL DEFAULT 1,
  "startedAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  "finishedAt" TIMESTAMP(3),
  "heartbeatAt" TIMESTAMP(3),
  "errorCategory" TEXT,
  "errorSummary" TEXT,
  "meta" JSONB,
  "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT "JobRun_pkey" PRIMARY KEY ("id"),
  CONSTRAINT "JobRun_jobId_fkey" FOREIGN KEY ("jobId") REFERENCES "Job"("id") ON DELETE CASCADE ON UPDATE CASCADE
);
```

# 10. Index

```sql
CREATE INDEX "JobRun_jobId_status_idx" ON "JobRun"("jobId", "status");
CREATE INDEX "JobRun_status_startedAt_idx" ON "JobRun"("status", "startedAt");
CREATE INDEX "JobRun_jobId_startedAt_idx" ON "JobRun"("jobId", "startedAt" DESC);
CREATE UNIQUE INDEX "JobRun_one_running_per_job" ON "JobRun"("jobId") WHERE status = 'running';
```

# 11. Retention

- JobRun 保留 30 天（或每 Job 最近 10,000 行，先到先删），由未来 CLEANUP Job 执行（本阶段只定义策略，不实现自动清理）。
- Job 定义行长期保留。

# 12. Scheduler 迁移方式（逐步，不重写）

```text
setInterval tick（保留）
  → ensureJob(jobDef)（幂等注册）
  → claimJobRun(jobId)（partial unique index 防重；冲突=跳过本 tick）
  → execute 现有逻辑（零改动）
  → completeJobRun(success|failed, meta/errorSummary)
```

- **monitor tick**：每个 tick 1 个 JobRun；meta 记录 {monitored, alertsSent}。
- **search-sync tick**：每个 tick 1 个 JobRun；保留 SearchSyncJob 作为 per-property 子记录（不删）。
- **discovery**：进程启动 1 个会话 JobRun（heartbeat 每次 tick 更新）；停机 complete；崩溃由恢复逻辑 fail。
- 禁止：删除/重写现有 setInterval 逻辑（本阶段仅包裹记录状态）。

# 13. Data Quality Job（第一个候选）

- Job：`data_quality_scan`（type=DATA_QUALITY，maxAttempts=3，enabled=true，不自动调度或按周调度——待定）。
- 执行器：调用只读 Fact 质量扫描（复用 Step 5 的 scan 逻辑，收敛为 lib 函数），输出统计到 JobRun.meta（total/valid/invalid/rate/byType），不写业务数据。
- 验证价值：只读、零风险、成功/失败易观察——作为 Job/JobRun 首个端到端验证。

# 14. 失败恢复

- 业务异常 → fail(errorCategory=business, errorSummary=截断消息, finishedAt)。
- 超时（JobRun 最长执行时间按 Job 配置，如 monitor 45min）→ fail(errorCategory=timeout)。
- 心跳超时（discovery 会话 5min 无心跳）→ fail(errorCategory=stale)。

# 15. Crash Recovery

- 启动恢复（instrumentation.register 扩展）：将 `status='running'` 且（finishedAt 为空 且 heartbeatAt/startedAt 早于阈值，如 30min）的 JobRun → `failed(errorCategory='crash', errorSummary='interrupted by server restart')`。
- 与现有 Investigation 启动恢复并行，互不干扰。

# 16. 防重复

1. **partial unique index**：单 Job 单 running（DB 级硬约束，防多实例/重叠 tick）。
2. **claim 冲突跳过**：并发触发时后到者跳过本 tick（等价现有单飞）。
3. **幂等执行体**：现有任务逻辑本身幂等（factValuesEqual 门、候选退避、SearchSyncJob upsert）——JobRun 只记录不改变执行语义。
4. **心跳 + 启动恢复**：崩溃后不残留 running 僵尸行。

# 17. 测试方案

1. claim：同一 Job 第二次 claim → 冲突跳过。
2. 状态机：queued→running→success；queued→running→failed。
3. attempt：失败 attempt+1；超 maxAttempts → failed。
4. crash recovery：伪造旧 running 行 → 启动恢复标记 failed(crash)。
5. heartbeat：discovery 会话心跳更新；无心跳超时 → stale fail。
6. 现有三调度器回归：monitor/search/discovery 行为不变（包裹后全量测试 + staging）。
7. data_quality_scan JobRun：成功写入 meta 统计；只读零业务写。
8. 全量测试 + typecheck + build。

# 18. staging

- 迁移应用到生产 DB 后 staging 读取正常（staging 只读角色，仅读 Job/JobRun）。
- staging 无法写 JobRun（只读库）→ 执行链路由单测 + production 验证覆盖（与既往步骤一致）。

# 19. production

- migration 前备份；`prisma migrate deploy`；seed 4 个 Job。
- 观察 24h：monitor/search/discovery 每个 tick 产生 JobRun 且 success；discovery 会话心跳正常；无 running 堆积；无 fatal。
- data_quality_scan 手动触发一次 → success + meta 统计。

# 20. 回滚

- 代码：git revert（执行器包裹层），调度器逻辑未动，回滚无行为回归。
- Schema：DROP TABLE "JobRun"; DROP TABLE "Job";（新表，无历史数据风险）；备份在位。
- 若实施阶段发现需要改动现有调度逻辑：先停止并报告，不扩大范围。

---

## GitHub Publication

- Repository: Duoniu-ai/Codex
- Branch: main
- Commit: 见提交结果（推送后 local HEAD = origin/main）
- Tag: 无
- File: SITEINTEL_PHASE1_STEP7_JOB_SYSTEM_PLAN.md
- Remote verification: PASS（推送后验证）
