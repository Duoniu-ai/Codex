# SiteIntel Phase 1 — Step 5 Fact 数据校验与形状治理完成报告

> 日期：2026-08-18 ｜ 授权：用户正式授权（zod、Fact schema、写入校验、schemaVersion migration、读取兼容、只读旧数据扫描）
> 结论：✅ **完成（PASS）**

---

## 1. 修改文件

```text
package.json / pnpm-lock.yaml                    （+zod ^4.4.3）
prisma/schema.prisma                              （Fact.schemaVersion Int @default(0)）
prisma/migrations/20260818_fact_schema_version/migration.sql（新增，additive）
src/lib/fact-schemas.ts                           （新增：9 类 canonical v1 zod schemas + validateFactValue/factValidationIssues）
src/lib/fact-schemas.test.ts                      （新增：schema 正反例矩阵）
src/lib/facts.ts                                  （写入校验接入 + schemaVersion=1）
src/lib/facts-validation.test.ts                  （新增：写入路径 invalid/valid 行为）
src/lib/report.ts                                 （读取端 v1 无效跳过 + legacy 宽容）
scripts/audit/fact-legacy-quality-scan.ts         （新增：只读旧数据扫描工具）
```

## 2. Schema change / migration

- `Fact.schemaVersion INTEGER NOT NULL DEFAULT 0`（additive，无删除/修改/UPDATE）。
- Migration `20260818_fact_schema_version` 已应用：`_prisma_migrations` 记录；生产列实测 `schemaVersion:integer:0`；**全部 3237 条历史行保持 0**。
- Migration 前生产备份：`/tmp/siteintel_pre_step5_20260818.dump`。

## 3. Fact schema（canonical v1）

- 9 类 schema 严格依据**生产真实分布 + 代码构造逻辑**（代表性抽样：空串/空数组/null/旧变体/非 ISO 日期/MX "0 " 等边界均已覆盖）。
- 关键字段严格验证；`.passthrough()` 允许未知扩展字段（不因未来非关键字段拒绝整条 Fact）。
- 已知 legacy 变体：metadata 旧字段集（缺 6 字段）、infrastructure 旧 analyzer profile（含 ips/ssl/http、缺部分 slot）——扫描已量化，未迁移。

## 4. schemaVersion semantics（固定）

- `0` = legacy / pre-runtime-validation（历史行，不迁移）。
- `1` = canonical Fact schema v1（新写入）。
- 非更新次数/抓取次数/生命周期版本。
- 旧行在自然重分析时按新写入升级为 1（非批量 UPDATE，属正常 upsert 路径）。

## 5. validation behavior

- 位置：`syncOneFact`，**Evidence.create 之前**（流程：draft → schema validation → 失败记录+skip → 成功 → Evidence → Fact → Signal/Event）。
- 失败：不创建 Evidence/Fact/Signal/Event；pipeline 不崩溃（per-draft 隔离）。
- 诊断日志：factType / targetId / investigationId / schemaVersion=1 / error issues(path+code) / 时间；**不记录完整 value**。
- 写入：Fact create/update 均置 schemaVersion=1。

## 6. read compatibility

- schemaVersion=0：保留现有宽容读取（不变）。
- schemaVersion=1：`report.ts fetchFacts` 对无效 v1 行安全跳过（不崩溃、不输出垃圾、记录诊断）；其余读取方（facts diff/contradictions/insight/sitemap）沿用各自防御性解析。

## 7. legacy scan

- 结果详见 [SITEINTEL_PHASE1_STEP5_FACT_LEGACY_QUALITY_REPORT.md](SITEINTEL_PHASE1_STEP5_FACT_LEGACY_QUALITY_REPORT.md)：3237 条，valid 3049（94.2%），invalid 188（5.8%，全部为两类 legacy 形状变体）；零写入。

## 8. tests

- 新增 20 例：schema 正反例（每类真实样本通过、错类型/缺字段/tuple 长度/未知类型拒绝、nullable/空数组边界、passthrough 扩展）；写入路径（invalid draft 不建 Evidence/Fact/Signal、pipeline 继续；valid draft schemaVersion=1）。
- 全量：**329/329 通过（22 文件）**；typecheck 0；本地 build 通过。

## 9. typecheck / build

- `tsc --noEmit` 0 错误；`next build` 成功（生产 BUILD_ID `sv4InVKQeOgJAVqv0CnZX`）。

## 10. staging

- 代码部署成功（schema/facts 含新逻辑），build 通过，服务 active，home 200。
- 说明：staging 只读库无法新建 Fact；写入路径由单测覆盖，生产完成真实写入验证。

## 11. production

- Migration 应用并验证（3237 行 schema0 保持不变）。
- 真实分析 example.com → completed；该 Target 9 条 Fact **全部 schemaVersion=1**；全局 schema1=9。
- Report 页面 200；monitor/search-sync/discovery 调度器正常；seed 导入正常；0 fatal；NRestarts=0。
- 生产 HEAD 对齐 `61e75bf`，tracked 树零差异。

## 12. Git commit

- 代码 commit：`61e75bfa347fda109999ef2e7767d81212368c81`（feat(data): canonical Fact schema validation with schemaVersion）
- siteintel main = 生产 HEAD = `61e75bf`

## 13. GitHub commit / tag

- Tag：`production-phase1-step5-fact-validation-2026-08-18` → 61e75bf（annotated，已推送）
- 本报告推送至 Duoniu-ai/Codex main（见 GitHub Publication）。

## 14. rollback

- 代码：`git revert 61e75bf` 或恢复 tag `production-phase1-step4-sse-auth-2026-08-18`=c4d79e1 → build/restart。
- Schema：`ALTER TABLE "Fact" DROP COLUMN "schemaVersion";`（additive 可逆；备份在位）。
- 历史数据：零变更，无回滚需求；扫描未写库。
- 校验失败影响：仅跳过无效 draft（等价 partial），回滚代码即恢复。

## 15. known limitations

1. 188 条 legacy 变体（metadata 140 / infrastructure 48）保持 schemaVersion=0 与宽容读取；如需升级需重分析（自然生成 v1）或另行授权。
2. 读取端治理目前集中报告路径（fetchFacts）；sitemap/insight/contradiction 仍为各自防御性解析（无崩溃风险，未加统一 gate，避免扩大范围）。
3. `zod@^4.4.3` 为新增依赖；esbuild 构建脚本在服务器需 `pnpm dlx esbuild` 或本地打包（esbuild postinstall 被 pnpm 忽略所致，不影响应用运行）。
4. v1 schema 对日期仅约束为 string（未做日期语义校验），与生产真实格式（混合 ISO 与 "Oct 27 ... GMT"）一致。

---

## GitHub Publication

- Repository: Duoniu-ai/Codex
- Branch: main
- Commit: 见提交结果（推送后 local HEAD = origin/main）
- Tag: production-phase1-step5-fact-validation-2026-08-18（siteintel 仓库）
- File: SITEINTEL_PHASE1_STEP5_FACT_VALIDATION_COMPLETION_REPORT.md
- Remote verification: PASS（推送后验证）

---

*完成，停止。未开始 Step 6；未自动修复旧 Fact。*
