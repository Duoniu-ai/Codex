# SiteIntel Phase 1 — Step 7 Minimal Job System 完成报告

> 日期：2026-08-18 ｜ 授权：按 Step 7 Plan 实施 ｜ 结论：✅ **完成（PASS）**

---

## 1. 修改文件

```text
prisma/schema.prisma                          （Job + JobRun 模型）
prisma/migrations/20260818_job_system/migration.sql（新增：两表 + 4 索引 + partial unique）
src/lib/jobs.ts                               （ensureJob/claimJobRun/complete/fail/heartbeat/recover/prune/分类）
src/lib/jobs-data-quality.ts                  （data_quality_scan 执行器 + 只读统计）
src/lib/jobs.test.ts                          （新增 12 例）
src/lib/monitoring.ts                         （tick 包裹 JobRun + heartbeat）
src/lib/search/sync.ts                        （tick 包裹 JobRun + heartbeat）
src/lib/discovery/collector.ts                （进程级会话 JobRun + 60s 心跳）
src/instrumentation.ts                        （启动：stale JobRun 恢复 + ensureJobs）
scripts/run-data-quality-job.ts               （data-quality 手动触发脚本）
```

## 2. Migration

`20260818_job_system`（additive）：`Job`/`JobRun` 两表 + `JobRun_jobId_status` / `JobRun_status_startedAt` / `JobRun_jobId_startedAt` / `JobRun_heartbeatAt` 索引 + **partial unique** `JobRun_one_running_per_job`。生产已 deploy，`_prisma_migrations` 记录；migration 前备份 `/tmp/siteintel_pre_step7_20260818.dump`。

## 3. Job schema

`id/name(unique)/type/description/enabled/maxAttempts/config/createdAt/updatedAt`；4 个种子（monitor、search_sync、discovery、data_quality_scan），ensureJobs 幂等注册（生产实测 4 行）。

## 4. JobRun schema

`id/jobId(FK)/status/attempt/startedAt/finishedAt/heartbeatAt/errorCategory/errorSummary/meta/createdAt`。

## 5. Index

见 §2；`running_per_job_max` 实测恒为 1。

## 6. Claim mechanism

原子 `INSERT ... ON CONFLICT ("jobId") WHERE status='running' DO NOTHING RETURNING`（无 SELECT→INSERT 竞态）；失败方安全跳过本 tick（生产/单测实证）。

## 7. Concurrency

partial unique index 硬约束：同一 Job 最多 1 个 running；monitor/search 每 tick 一个 JobRun，discovery 进程级会话。

## 8. Recovery

启动时**将所有 running JobRun 标记 failed(crash)**（单实例语义：boot 时 running 必属已死进程，杜绝快速重启遗留僵尸行阻塞 claim）。生产实证：`[boot] marked 1 stale JobRun(s) as failed` + 新会话成功启动。

## 9. Heartbeat

monitor 每处理 5 个监控点、search-sync 每属性、discovery 每 4 tick（≈60s）更新 heartbeatAt——低频写入。

## 10. Retry

scheduler 任务 maxAttempts=1（自然重试=下一 tick）；data_quality_scan maxAttempts=3（同 JobRun 内有限重试，无指数退避；测试覆盖失败一次后成功）。

## 11. Discovery 处理

**进程级会话 JobRun**（每个进程生命周期 1 条 running，60s 心跳）；禁止每 tick 生成历史行；重启后旧会话被恢复为 failed，新会话重新 claim。

## 12. Data Quality Scan

只读（`runDataQualityScan`：total/valid/invalid/invalidRate/schemaVersion 分布/byType/topPaths，写入 JobRun.meta，**不含完整 Fact.value**）。生产运行成功：meta 实测 `{total:3237, valid:3049, byType:{metadata:{total:347,valid:207,invalid:140,...}}}`；未触碰任何 Fact。

## 13. 测试结果

新增 12 例（Job 幂等/claim race/状态机/heartbeat/recovery/retention/截断/质量统计/端到端重试）；全量 **351/351 通过（24 文件）**；typecheck 0；build 通过。

## 14. staging

迁移后表可读；部署成功（build/active/home 200）；只读库无法写 JobRun，执行链路由单测+生产验证覆盖。

## 15. production

| 验证 | 结果 |
|---|---|
| 4 Job 种子 | ✅ |
| monitor/search_sync/discovery JobRun | ✅ 各 1 条 |
| 状态分布 | 3 success + 1 failed(crash 旧会话) + 1 running(新会话) |
| 并发上限 | running_per_job_max=1 ✅ |
| data_quality_scan | ✅ success + meta 统计 |
| crash recovery | ✅ 重启后旧会话 failed + 新会话启动 |
| 服务 | active、NRestarts=0、0 fatal；HEAD=75bd5ef、tracked 零差异 |

## 16. backup

`/tmp/siteintel_pre_step7_20260818.dump`（5.6MB）在位。

## 17. rollback

- 代码：`git revert 81513e1 75bd5ef` 或恢复 tag `production-phase1-step6-event-signal-2026-08-18`=559134c。
- Schema：`DROP TABLE "JobRun"; DROP TABLE "Job";`（新表，无业务数据关联）；不影响 Fact/Evidence/Event/Signal/Investigation/Target。

## 18. 已知限制

1. 单实例语义（恢复策略假定 boot 时 running 必属已死进程）；多实例需另行设计（当前无此需求）。
2. JobRun 保留策略（30 天）仅提供 `pruneJobRuns` helper，未创建 CLEANUP Job（按授权）。
3. seed 未独立成 Job（随 discovery 会话执行，符合方案判断）。
4. 观测页面未新增（JobRun 可通过 DB/admin 数据质量页读取；后续可视化另行授权）。

## 19. 与 Step 7 Plan 逐项对应

Plan §1-3（现状/矩阵/风险）→ 报告 §1-2 与方案记录；§4-5（Job/JobRun）→ §3-4；§6-7（状态/Retry）→ §10；§8（并发）→ §7；§9-10（Schema/Index）→ §2/§5；§11（Retention）→ §18-2；§12-13（迁移/DQ Job）→ §11-12；§14-16（恢复/防重）→ §8/§6-7；§17-20（测试/staging/prod/回滚）→ §13-17。无偏差。

---

## GitHub Publication

- Repository: Duoniu-ai/Codex
- Branch: main
- Commit: 见提交结果（推送后 local HEAD = origin/main）
- Tag: production-phase1-step7-job-system-2026-08-18（siteintel 仓库）
- File: SITEINTEL_PHASE1_STEP7_JOB_SYSTEM_COMPLETION_REPORT.md
- Remote verification: PASS（推送后验证）

---

*完成，停止。未开始 Step 8。*
