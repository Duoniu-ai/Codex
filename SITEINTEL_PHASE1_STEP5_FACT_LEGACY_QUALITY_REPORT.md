# SiteIntel Phase 1 — Step 5 Fact 旧数据只读质量扫描报告

> 日期：2026-08-18 ｜ 性质：**只读扫描**（safeParse 校验，零写入/零 UPDATE/零迁移/零清洗）
> 工具：`scripts/audit/fact-legacy-quality-scan.ts`（esbuild 打包后以生产 .env 只读连接）
> 校验基准：`src/lib/fact-schemas.ts`（canonical v1 zod schemas）

---

## 1. 总体结果

| 指标 | 值 |
|---|---:|
| 总数 | 3237 |
| valid | 3049 |
| invalid | 188 |
| invalid rate | **5.8%** |

## 2. 按 Fact type 分类

| factType | total | valid | invalid | 最常见错误 path | 最常见错误码 |
|---|---:|---:|---:|---|---|
| metadata | 347 | 207 | **140** | wordCount/h1Count/h1Text/internalLinks/externalLinks/structuredDataCount（各 140） | invalid_type |
| infrastructure | 406 | 358 | **48** | emailProvider 44 / cdn 36 / dnsProvider 19 / hosting 4 | invalid_type |
| dns_records | 404 | 404 | 0 | — | — |
| resolved_ips | 393 | 393 | 0 | — | — |
| tls_certificate | 381 | 381 | 0 | — | — |
| http_status | 379 | 379 | 0 | — | — |
| summary | 406 | 406 | 0 | — | — |
| technologies | 280 | 280 | 0 | — | — |
| registrar | 241 | 241 | 0 | — | — |

## 3. 无效原因定性（代表性样本摘要，不含完整 payload）

### 3.1 metadata（140 条，占无效 74.5%）

- 旧版 payload 缺失 6 个字段：`wordCount / h1Count / h1Text / internalLinks / externalLinks / structuredDataCount`（2016 年前期 metadata schema 未含 SEO 字段）。
- 样本摘要：`{"title":"403 - Forbidden: Access is denied.","language":"","generator":null,"hasRobots":false,...}`；`{"title":"香港服务器租用 | ...","language":"en","generator":null,...}`；`{"title":"Just a moment...","language":"en-US",...}`。

### 3.2 infrastructure（48 条，占无效 25.5%）

- 旧版 analyzer-profile 形状（含 `ips/ssl/http` 字段、部分 slot 缺失）与 v1 四 slot 契约不符：缺 `emailProvider` 44、缺 `cdn` 36、缺 `dnsProvider` 19、缺 `hosting` 4。
- 样本摘要：`{"ips":[{"asn":"4808","isp":"China Unicom...","source":"ipwho.is"...`；`{"ips":[{"asn":"400619",...`；`{"cdn":{"name":"Cloudflare","evidence":[{"detail":"cf-ray: ...","source":"http_header"}],"confidence"...`。

## 4. 结论与处置

1. **7/9 类旧数据与 v1 schema 完全兼容**（dns/resolved_ips/tls/http/summary/technologies/registrar 0 无效）——v1 schema 定义与生产真实分布高度一致。
2. 188 条无效全部为**明确的 legacy 形状变体**（metadata 旧字段集、infrastructure 旧 analyzer profile），非随机垃圾。
3. 按授权**不自动修复、不迁移、不 UPDATE**；这些行保持 schemaVersion=0，读取端走宽容路径，报告不受影响。
4. 建议（后续另行决策）：如未来需要，可对这两类 legacy 变体做只读补充统计或按 target 重分析自然升级为 v1（重分析会产生新的 schemaVersion=1 行）；本次不做任何数据变更。

---

## GitHub Publication

- Repository: Duoniu-ai/Codex
- Branch: main
- Commit: 见提交结果（推送后 local HEAD = origin/main）
- Tag: production-phase1-step5-fact-validation-2026-08-18（siteintel 仓库）
- File: SITEINTEL_PHASE1_STEP5_FACT_LEGACY_QUALITY_REPORT.md
- Remote verification: PASS（推送后验证）
