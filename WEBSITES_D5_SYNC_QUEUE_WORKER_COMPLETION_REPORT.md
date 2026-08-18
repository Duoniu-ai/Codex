# SITEINTEL_D5_SYNC_QUEUE_WORKER_COMPLETION_REPORT

> 项目:**Websites**(网址发现平台/网址导航站;独立产品,SiteIntel 仅为上游数据源)
> 报告日期:2026-08-18(+0800)
> Git 基线:D4 Commit = `20fd42e78b5114db152558f1ceb2e01402b67cbe`
> D5 实施范围:`src/lib/queue/**`、`src/lib/worker/**`、`src/instrumentation.ts`(冻结任务拆解 D5)
> 数据库:nav_disc @ 154(本轮**零写入/零结构变更**,仅只读基线)

---

## 1. D5 Summary

**D5 = PASS(30/30 验收矩阵)**

- 实现 Sync Queue + Worker 全链路:状态机(5.1)/ CAS(5.2)/ 超时恢复(5.3)/ 退避(5.4)/ 调度(5.5)/ 幂等(5.6)/ 日志(5.7)
- D5 测试 **72 项**全绿;全量 `npm test` **159/159 PASS**;`tsc --noEmit` PASS;`eslint` PASS;`git diff --check` PASS
- 复用 D4 全部六模块(client/normalize/mapping/summary-hash/persist/throttle),未复制、未新建第二套实现
- D4 Production E2E = BLOCKED(HTTP 401),单独待处理,未触碰

---

## 2. Queue State Machine(5.1)

- `src/lib/queue/state-machine.ts`:6 枚举(pending/processing/completed/retrying/failed/skipped,无第七状态)+ §9.2 全转换表 + 非法转换拒绝 + 终态判定
- 测试:合法转换全表 / 非法转换(completed/skipped 不可重执行、pending→completed 拒绝)/ 事件不匹配拒绝

---

## 3. CAS(5.2)

- `src/lib/queue/queue-db.ts` `claimNext`:单条条件更新(§9.3 SQL 逐字),`结果数组 length==1` 才算成功;禁止先 SELECT 再 UPDATE;ORDER BY priority / next_retry_at / id
- 测试:单 worker 领取 / count=0 拒绝 / **双 worker 竞争只一个成功 / 三 worker 竞争 1 acquired 2 rejected** / 锁定任务排除 / 排序与到期

---

## 4. Timeout Recovery(5.3)

- `recoverTimeout`:步骤 1(attempts+1 < max → retrying + 退避)/ 步骤 2(attempts+1 >= max → failed);锁释放;超时计入 attempts
- 测试:<10min 不恢复 / ==10min 边界不恢复(严格小于)/ 超 1ms 恢复 / attempts 边界分派

---

## 5. Retry / Backoff(5.4)

- `src/lib/queue/backoff.ts`:`min(2^attempts × 60s, 4h)`(attempts 为更新后值)
- `process.ts applyRetry`:先 +1 再判定;**第 5 次失败即 failed,无第 6 次**
- 测试:attempts=1..5 退避序列 / 4h 封顶(10/100/1024)/ attempts=4+1=5 → failed 无 retry / attempts=3+1=4 → retry 且退避基于 4

---

## 6. Scheduler(5.5)

- `src/lib/worker/tick.ts` `runTick`:超时恢复(最先)→ 节流门控 → CAS 领取 → 执行
- `src/lib/worker/scheduler.ts`:setInterval 单实例 tick(默认 15min)+ 防重复注册(Symbol 守卫)+ unref
- 测试:tick 周期触发 / stop / 防重复注册 / 异常捕获不中断调度 / 两 tick 并发只一个执行

---

## 7. Worker(5.5-5.6 执行链)

- `src/lib/worker/process.ts`:fetchReport → normalize → mapping → summary-hash → persist → complete/retry/fail → sync_logs
- 错误分类(§16):可重试(network/timeout/5xx/429/404→analyze);不可重试(401/malformed/missing-key/invalid-domain/contract)
- 404 → analyze 分支(受 30/h 令牌桶);节流耗尽 → retrying 不触发 analyze
- 任何 CAS 失败的 worker 不进入同步链路(tick 保证)

---

## 8. Idempotency(5.6)

- 入队按 `UNIQUE(normalized_domain, job_type)` 幂等(重复入队零新增)
- 执行幂等闭环 = CAS + D4 summary_hash + D4 persist 零写(复用,未重写)
- 测试:重复入队同实体 / 重复 tick 单执行 / 重复同步零写(沿用 D4 管道)

---

## 9. Logging / Instrumentation(5.7)

- `src/lib/queue/log.ts`:sync_logs 落库(action/status/error/http_status/duration_ms)+ `sanitizeForLog`(si_ 令牌 / postgresql URL / 长令牌 → REDACTED)
- `src/instrumentation.ts` `register()`:`NEXT_PHASE=phase-production-build` 守卫 + NODE_ENV=development 跳过 + 生产启动调度
- 测试:build/dev 不启动 / 生产启动防重复 / 日志脱敏 / process 错误不泄漏 Key

---

## 10. Tests

- D5 新增 6 个测试文件、**72 项**通过;全量 `npm test` **159/159 PASS**(D4 与既有测试保持全绿)
- 竞争验证:双 worker / 三 worker 仅一个获得执行权(内存 CAS 语义,未制造真实生产竞争)
- 静态检查:`tsc --noEmit` PASS、`eslint` PASS、`git diff --check` PASS

---

## 11. 30 CHECK Matrix

| # | D5 Check | Result | Evidence |
|---|---|---|---|
| 01 | state machine | PASS | `state-machine.ts` 6 枚举 + 转换表;单测 29 项 |
| 02 | valid transitions | PASS | §9.2 全表合法转换断言 |
| 03 | invalid transitions | PASS | completed/skipped 不可重执行等非法转换拒绝 |
| 04 | CAS acquisition | PASS | claimNext 单 worker pending→processing(locked_by/at) |
| 05 | CAS count=0 rejection | PASS | 无可领取/已被抢 → 空结果,不执行同步 |
| 06 | two-worker race | PASS | Promise.all 2 workers → 1 acquired |
| 07 | three-worker race | PASS | 3 workers → 1 acquired / 2 rejected |
| 08 | processing timeout | PASS | locked_at < 10min 不恢复 |
| 09 | timeout boundary | PASS | ==10min 不恢复(严格小于);超 1ms 恢复 |
| 10 | safe recovery | PASS | 恢复释放锁;attempts+1 分派 retrying/failed |
| 11 | retry classification | PASS | classifySyncError 可重试/不可重试全覆盖 |
| 12 | retry attempt 1 | PASS | attempts=0 +1=1 → retrying,退避 2min |
| 13 | retry attempt 2 | PASS | attempts=1 +1=2 → retrying,退避 4min |
| 14 | retry attempt 3 | PASS | attempts=2 +1=3 → retrying,退避 8min |
| 15 | retry attempt 4 | PASS | attempts=3 +1=4 → retrying,退避 16min |
| 16 | retry attempt 5 | PASS | attempts=4 +1=5 → failed |
| 17 | no attempt 6 | PASS | 第 5 次即 failed,retry=0 |
| 18 | backoff | PASS | retryIntervalMs(1..5)=2/4/8/16/32min |
| 19 | 4h backoff cap | PASS | attempts=10/100/1024 → 4h 封顶 |
| 20 | scheduler tick | PASS | scheduler 周期触发 + stop + 防重复注册 |
| 21 | eligible task selection | PASS | priority / next_retry_at 到期 / id 排序 |
| 22 | locked task exclusion | PASS | processing 行不可再领取 |
| 23 | D4 throttle reuse | PASS | tick/process 仅调用 `SyncThrottle`,无第二套 |
| 24 | D4 client reuse | PASS | process 直接 import `SyncClient` |
| 25 | D4 hash/persist reuse | PASS | process 直接 import `buildSummaryHash`/`persistSyncResult` |
| 26 | idempotency | PASS | UNIQUE 入队幂等 + 重复 tick 单执行 + 零写闭环 |
| 27 | sync_logs | PASS | writeSyncLog 字段正确 + 脱敏 |
| 28 | secret safety | PASS | sanitizeForLog + 错误/队列字段不泄漏 Key |
| 29 | scope / diff check | PASS | git 仅 D5 文件;diff --check 0 |
| 30 | final D5 acceptance | PASS | 以上 29 项 + 159/159 + tsc/eslint 全过 |

**结果:30 / 30 PASS(0 FAIL / 0 BLOCKED / 0 NOT_RUN)**

---

## 12. Database

```text
Schema Change = NO
Migration = NO
Data Migration = NO
```

- 仅运行时读写 sync_queue / sync_logs 与读 settings.sync_queue.max_attempts(=5)
- CAS 领取与超时恢复按冻结 §9.3/§9.4 SQL 语义实现(未改表结构)

---

## 13. Security

```text
Secrets = CLEAN
NAVIGATION_SYNC_KEY = REDACTED
```

- 测试统一使用 mock/REDACTED_TEST_KEY;真实 Key 未出现于测试、报告、日志、Git
- 错误信息与队列 last_error 经 `sanitizeForLog` 脱敏

---

## 14. Scope

```text
D5 = PASS
D6+ = NOT EXECUTED
D10 = NOT EXECUTED

D4 = UNCHANGED(未修改 src/lib/sync)
D4 Production E2E = BLOCKED(HTTP 401,单独待处理,未触碰)
```

---

## 15. Git

实际修改/新增文件:

```text
src/lib/queue/state-machine.ts
src/lib/queue/backoff.ts
src/lib/queue/queue-db.ts
src/lib/queue/log.ts
src/lib/queue/state-machine.test.ts
src/lib/queue/backoff.test.ts
src/lib/queue/queue-db.test.ts
src/lib/worker/process.ts
src/lib/worker/tick.ts
src/lib/worker/scheduler.ts
src/lib/worker/process.test.ts
src/lib/worker/tick.test.ts
src/lib/worker/scheduler.test.ts
src/instrumentation.ts
src/instrumentation.test.ts
SITEINTEL_D5_PREFLIGHT_REPORT.md
SITEINTEL_D5_SYNC_QUEUE_WORKER_COMPLETION_REPORT.md
```

D5 Commit:`feat(websites): implement sync queue and worker`(提交后 HEAD 见 git log)
