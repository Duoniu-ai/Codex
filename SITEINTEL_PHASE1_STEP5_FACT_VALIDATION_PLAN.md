# SiteIntel Phase 1 — Step 5 Fact 数据校验与形状治理（实施前分析）

> 日期：2026-08-18 ｜ 性质：**只分析，不实施**（未修改代码/数据库/生产）
> 依据：真实代码（facts.ts/pipeline.ts/report.ts/sitemap.ts/intelligence）+ 生产只读统计（3237 条 Fact）
> 结论：当前可以最小变更实施；**schemaVersion 列与 Zod 依赖两项需另行授权**；历史数据不迁移、不自动清洗

---

# 1. 当前 Fact 写入路径

```text
pipeline.executePipeline
  → buildSnapshotData(bundle)                     （snapshots.ts，10 aspect）
  → storeSnapshots                                 （历史存储）
  → syncFacts(targetId, investigationId, snapshotData, rawIds)   （facts.ts）
      → buildFactDrafts(snapshotData)              （9 类 factType，从 aspect 取 payload）
      → syncOneFact(draft)                         （逐条）
          ① Evidence.create                        （先建证据）
          ② Fact.upsert                            （value=toJson(draft.value)，无校验）
          ③ 值变化 → Signal.create + diffFact → Event
  → runSignalCorrelation / runContradictionDetector / runInsightEngine（读 Fact）
```

**现状：Fact.value 以 `toJson(draft.value)` 原样落库，写入路径无任何运行时校验**；类型正确性只依赖 TypeScript 编译期类型与 snapshots.ts 的构造代码。

# 2. 当前 Fact.value 形状（9 类，生产实测）

生产统计（2026-08-18 只读）：**总计 3237 条**；dns_records 404 / http_status 379 / infrastructure 406 / metadata 347 / registrar 241 / resolved_ips 393 / summary 406 / technologies 280 / tls_certificate 381。

实测形状（首行样本，截断）：

| factType | 形状 | 要点 |
|---|---|---|
| resolved_ips | `[ [ip, asn, org], ... ]` | 3 元组数组；asn/org 可为 null |
| dns_records | `{ records: [type,name,value][], nameservers: string[] }` | 3 元组；样本 MX value 为 `"0 "`（priority 空）——形状脆弱实证 |
| tls_certificate | `{ fingerprint, serial, subject, issuer, issuerOrg, validFrom, validTo, daysRemaining, signatureAlgorithm, san[] }` | 字符串/数字/null 混合 |
| http_status | `{ status, finalHost, securityHeaders{5×bool}, redirects:[status,host][] }` | redirects 为 2 元组 |
| technologies | `{ tech:[slug,name,category,version,confidence][], trackingIds:[type,id][] }` | 5 元组 + 2 元组 |
| metadata | `{ title, description, language, generator, faviconUrl, hasRobots, hasSitemap, wordCount, h1Count, h1Text, internalLinks, externalLinks, structuredDataCount }` | 大量 nullable |
| registrar | `{ registrar, createdAt, updatedAt, expiresAt, status[], nameservers[] }` | 日期字符串；status 数组 |
| infrastructure | `{ cdn, hosting, dnsProvider, emailProvider }` 各为 `{name, confidence, evidence[]}` 或 null；hosting 含 `hidden` | evidence 数组含 cf-ray 等易变字段（已 canonicalize 于 hash） |
| summary | `{ en: WebsiteSummary, zh: WebsiteSummary }` | 生产 406 条全部为本地化格式；**legacy 扁平格式 0 条** |

# 3. 不同 Fact type 的真实 payload 差异

- **数组形态**：4 类使用 tuple 数组（resolved_ips / dns_records.records / technologies.tech+trackingIds / http_status.redirects）；其余为对象。
- **Nullable 语义**：metadata 多数字段 null；resolved_ips asn/org null；infrastructure 四个 slot 可 null。
- **易变字段**：infrastructure evidence 含请求级数据（cf-ray），已通过 canonicalForHash 从 hash 剔除，但 value 本身保留。
- **日期/状态**：registrar 使用 ISO 字符串与状态数组；tls_certificate 的 validTo 为 `"Oct 27 ... GMT"` 格式（非 ISO）——**schema 必须按实际格式约束，不能假设 ISO**。
- **历史差异**：summary legacy 扁平（无 en/zh）在生产为 0，但代码仍保留兼容分支；旧行未迁移。

# 4. 当前 tuple 假设具体位置

| 文件/函数 | 假设 |
|---|---|
| report.ts `ipFromTuple`（L309-319） | `[0]=ip, [1]=asn, [2]=org` |
| report.ts `dnsRecordsFrom`（L351-353） | records `[0]=name, [1]=type, [2]=value` |
| report.ts `httpFrom`（L379） | redirects `[0]=status, [1]=host` |
| report.ts `emailFrom`（L389-391） | MX records `[1]=type, [2]=value` |
| report.ts `technologyFrom`（L411-417） | tech `[0]=slug,[1]=name,[2]=category,[3]=version,[4]=confidence` |
| report.ts `trackingFrom`（L455-457） | trackingIds `[0]=type,[1]=id` |
| facts.ts `diffFact`（resolved_ips/dns/technologies） | 与上述 tuple 相同的索引假设 |
| sitemap.ts `hasDnsFromFactValue/hasIpsFromFactValue/hasTechnologyFromFactValue` | 同上（值存在性判断） |
| intelligence/rules.ts `tls_expiry/domain_expiry` | tls_certificate/registrar 为对象形态 |
| contradictions.ts 4 规则 | dns_records/registrar/http_status/infrastructure 对象形态 |

结论：tuple 假设集中在 **report.ts 与 sitemap.ts（读取端）** 及 facts.ts（diff 端）；写入端是唯一“无校验生成点”。

# 5. 最小 Schema 设计（Zod）

**依赖**：新增 `zod`（当前 package.json 无任何运行时校验库；“现有验证体系”= TS 类型 + 手工 shape helpers，无运行时校验）。Zod 体积小、与 TS 集成，符合“优先使用 Zod”。

**新文件**：`src/lib/fact-schemas.ts`——`FACT_SCHEMAS: Record<FactType, ZodType>`，9 个 schema 严格按 §2 实测形状定义：

```text
resolved_ips:  z.array(z.tuple([z.string(), z.string().nullable(), z.string().nullable()]))
dns_records:   z.object({ records: z.array(z.tuple([z.string(), z.string(), z.string()])),
                          nameservers: z.array(z.string()) })
tls_certificate: z.object({ fingerprint: z.string(), serial: z.string(), subject: z.string(),
                          issuer: z.string(), issuerOrg: z.string().nullable(),
                          validFrom: z.string(), validTo: z.string(), daysRemaining: z.number(),
                          signatureAlgorithm: z.string(), san: z.array(z.string()) })
http_status:   z.object({ status: z.number().int(), finalHost: z.string(),
                          securityHeaders: z.object({ hsts: z.boolean(), xFrameOptions: z.boolean(),
                            contentSecurityPolicy: z.boolean(), xContentTypeOptions: z.boolean(),
                            referrerPolicy: z.boolean() }),
                          redirects: z.array(z.tuple([z.number().int(), z.string()])) })
technologies:  z.object({ tech: z.array(z.tuple([z.string(), z.string(), z.string(), z.string(), z.number()])),
                          trackingIds: z.array(z.tuple([z.string(), z.string()])) })
metadata:      z.object({ 14 字段，string/boolean 严格，数值与可选字段 nullable })
registrar:     z.object({ 4 个日期/注册商字符串 nullable + status[] + nameservers[] })
infrastructure: z.object({ cdn/hosting/dnsProvider/emailProvider 四 slot：
                          { name: string, confidence: number, evidence: z.array(z.unknown()) }
                          + hosting.hidden: boolean；全部 nullable })
summary:       z.object({ en: summaryShape, zh: summaryShape })
summaryShape:  z.object({ description: string,
                          inferences: z.array(z.object({ text: string, confidence: number,
                            evidence: z.array(z.unknown()) })),
                          categories: z.array(z.string()) })
```

**校验位置（最小改动）**：`syncOneFact` 在 `Evidence.create` **之前**执行 `FACT_SCHEMAS[factType].safeParse(draft.value)`：
- 通过 → 按现状继续（evidence → fact upsert → signal/event）。
- 失败 → `console.error("[facts] validation failed", factType, targetId, firstIssue)`（**不回显 value**），跳过该 draft；**pipeline 不中断**（与现有“单 fact 失败隔离”一致）。

**schemaVersion**：`Fact.schemaVersion Int @default(0)`（migration：`ALTER TABLE "Fact" ADD COLUMN "schemaVersion" INTEGER NOT NULL DEFAULT 0;`）；新写入置 1。**该项属生产 Schema 变更，需单独授权后实施**（本轮不执行）。

# 6. 新旧数据兼容策略

- **不迁移、不清洗历史数据**（原则：不大规模迁移；旧数据只读统计）。
- 写入端：新数据一律校验 + schemaVersion=1。
- 读取端：`schemaVersion` 0/缺省 = legacy → 沿用现有宽容 helpers；1 = 严格 → 仍使用宽容解析兜底（safeParse 失败则字段省略/置空，**不崩溃、不显示垃圾**）。
- 可选只读质量扫描（实施阶段、只读）：用 FACT_SCHEMAS 对现有 3237 条逐条 safeParse，输出“无效旧数据统计报告”，**不自动修复**（供后续治理决策）。

# 7. 测试方案

1. `fact-schemas.test.ts`：每类 1 个真实样本通过 + 破坏性变异（缺字段/错类型/错 tuple 长度/null 语义）拒绝。
2. `facts.test.ts` 扩展：mock prisma——无效 draft → 不创建 Evidence/Fact、记日志、pipeline 继续；有效 draft → 正常。
3. `report.ts` 读取兜底：legacy 形状 + schemaVersion=0 正常渲染；schemaVersion=1 有效数据正常；无效值被省略不崩溃。
4. 全量测试 + typecheck + build。
5. staging：真实 Analyze 一次（新 Fact schemaVersion=1、校验通过）。
6. production：真实 Analyze 回归 + 只读无效旧数据统计报告。

# 8. 回滚方案

- 代码：`git revert <commit>`（无数据变更）。
- schemaVersion 列：`ALTER TABLE "Fact" DROP COLUMN "schemaVersion";`（若已授权执行；旧数据零影响）。
- 历史数据：全程未改，无回滚需求。
- 校验失败影响：仅跳过无效 draft（等同于该 provider 数据缺失的 partial 语义），可回滚代码恢复原状。

# 9. 需要授权的事项（本次不执行）

1. 新增 `zod` 依赖。
2. `Fact.schemaVersion` 列 migration（生产 Schema 变更）。
3. （可选）只读质量扫描执行授权。

# 10. 实施范围（获批后）

```text
package.json（+zod）
src/lib/fact-schemas.ts（新增）
src/lib/fact-schemas.test.ts（新增）
src/lib/facts.ts（校验接入 + schemaVersion=1 + 失败跳过）
src/lib/report.ts（读取兜底接入 schemaVersion，最小改动）
prisma/schema.prisma + migration（schemaVersion，单独授权）
```

禁止：Event/Signal 收敛、Job、Discovery、SEO/Security/Keyword、新业务表、历史数据迁移/清洗。

---

## GitHub Publication

- Repository: Duoniu-ai/Codex
- Branch: main
- Commit: 见提交结果（本计划推送后 local HEAD = origin/main）
- Tag: 无
- File: SITEINTEL_PHASE1_STEP5_FACT_VALIDATION_PLAN.md
- Remote verification: PASS（推送后验证）
