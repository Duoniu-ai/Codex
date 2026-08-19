# SiteIntel 2.0 — Phase 2.2 Technology Profile Backend 验收报告

> 日期：2026-08-19 ｜ 范围：Phase 2.2（Technology Profile Backend only）
> 授权：按《SITEINTEL 2.0 — Phase 2.2 Technology Profile Backend 执行指令》执行
> 结论：✅ 完成（本地代码 + 查询层 + API + 测试 + Build；无 Schema 变更；未部署生产；未 push）

---

# 1. Scope（范围）

本阶段将 Phase 2.1 建立的 Technology Catalog 与现有数据底座连接，形成稳定 Technology Profile Backend：

```text
TechnologyCatalog → Technology Entity → Fingerprint → Website Relationships
→ Events → Evidence → Technology Profile API
```

覆盖：Identity / Detection / Adoption / Relationships（分页）/ Events / Evidence / Coverage / Caching / Error Handling / Backward Compatibility。

---

# 2. Before State（改造前）

| 项 | 状态 |
|---|---|
| Technology Catalog | 104 行（Phase 2.1，生产已迁移+seed） |
| 实体链接 | 64/65（`Iconfont (Alibaba)` 未链接，按设计） |
| `uses` 关系（tech target） | 897（distinct 域名 320；含 8 条 www legacy 源） |
| Events | technology_added=5 · technology_removed=1 |
| Fact.technologies | 328 条（全部带 evidenceIds，1–20 条/条） |
| Evidence（factType=technologies） | 543 条（source=technology, collector=fingerprint_engine） |
| Technology API | 不存在（v1 仅 analyze/report） |
| read-cache | Step 8 ttlMemo 已就绪（60s） |
| 测试基线 | 365/365 PASS；typecheck PASS；build PASS |

---

# 3. API Architecture（API 架构）

- 新增统一入口：`GET /api/v1/technology/[slug]`（`src/app/api/v1/technology/[slug]/route.ts`）。
- **不引入 section 参数**：统一响应在当前数据规模下体积小、分页内联，拆分端点只增加成本无收益（Phase 2 Plan §11 决策落地）。
- 遵循既有 v1 模式：X-API-Key 鉴权（401）→ 每 key 速率限制 + 月度配额（429）→ 成功后 trackUsage；错误语义与 /api/v1/report 一致。
- 分页参数：`page`（默认 1）、`pageSize`（默认 20，最大 50），非法输入返回 400。

---

# 4. Response Contract（响应契约）

```jsonc
{
  "api": "siteintel-v1",
  "technology": {           // Identity
    "canonicalName", "displayName", "slug", "vendor", "category",
    "subcategory", "description", "officialUrl", "aliases",
    "entityValue",          // 观测到的实体名（无则 null）
    "catalogAvailable"      // Catalog 行是否可用
  },
  "detection": {
    "fingerprintCount", "fingerprints": [{ "slug", "name", "category", "confidence", "methods" }]
  },
  "adoption": {
    "websiteCount",         // 真实 uses 关系 distinct 域名数（observed 口径）
    "firstSeen", "lastSeen", // 关系 min/max 时间戳（非 Catalog createdAt）
    "scope": "observed"
  },
  "relationships": {        // Website ↔ Technology，分页
    "total", "page", "pageSize", "items": [{ "website", "relationship": { "type", "kind", "confidence" }, "firstSeen", "lastSeen" }]
  },
  "events": {               // technology_added / technology_removed
    "total", "items": [{ "type", "domain", "detectedAt", "previous", "current" }]
  },
  "evidence": {
    "total",                // 链接到使用该技术网站 Fact 的 Evidence 行数（真实计数）
    "items": [{ "id", "source", "collector", "observedAt", "confidence" }]  // 最近 20 条样例
  },
  "coverage": {
    "observedWebsiteCount",
    "hasHistory", "historicalCoverage": "partial | insufficient",
    "geographic": "unavailable",
    "hasDistributionData": false,
    "hasEvidence"
  }
}
```

无伪造字段：所有计数与时间戳来自真实实体图 / 关系 / Event / Fact / Evidence。

---

# 5. Identity（身份）

- 来源：`TechnologyCatalog`（canonicalName/displayName/vendor/category/subcategory/description/officialUrl/aliases）。
- Catalog 缺失（表未迁移）→ 优雅降级为 Entity-only：`catalogAvailable=false`，身份取实体 value，其余元数据 null，不 500。
- 实体解析：优先 `entityId`；未链接时按 normalized 候选（canonicalName、displayName、aliases、slug 变体）只读匹配，不模糊合并——`Iconfont (Alibaba)` 通过 alias `iconfont (alibaba)` 可被解析（Phase 2.1 遗留案例的查询层兜底，不改数据）。

---

# 6. Detection（检测）

- 全部来自 `FINGERPRINTS`（104 规则）+ `FINGERPRINT_CATALOG` 映射，静态数据、零 DB 查询。
- `fingerprints`：规则 slug/name/category/confidence/methods（http_header / cookie / html_pattern / meta_generator / asset_url，按规则字段推导）。
- 事件匹配值集合 `eventValuesForCatalogSlug`：catalog slug ∪ 对应 fingerprint slug（如 `alibaba-cloud-cdn` ↔ `aliyun-cdn`），确保事件按指纹 slug 也能命中。

---

# 7. Adoption（采用）

- `websiteCount`：`COUNT(DISTINCT s.normalized)`（`EntityRelationship(targetId, type='uses') JOIN Entity(domain)`）——P1-1 去重口径（www legacy 分裂只计一次），scope 明确为 `observed`。
- `firstSeen/lastSeen`：同查询 `MIN/MAX` 关系时间戳，**不是** Catalog.createdAt。
- 生产实测：React → websiteCount=16，firstSeen=2026-08-15T05:26:46Z，lastSeen=2026-08-19T00:05:30Z。

---

# 8. Relationships（关系）

- 查询：`EntityRelationship(targetId=技术实体, type='uses', source.type='domain')`，`orderBy [lastSeenAt desc, id asc]`（确定性排序），`skip/take` 分页。
- 页内去重 www legacy（按 source.normalized），`total` 与 adoption 一致。
- 无 N+1：单页 1 条查询 + include 关联实体。

---

# 9. Events（事件）

- 仅消费既有 `technology_added` / `technology_removed`，不创建新事件类型。
- 查询：`current/previous = eventValues`（JSON equals，按 fingerprint slug），`orderBy [detectedAt desc, id asc]`，上限 200（当前全量仅 6 条）。
- 生产实测：`fastly` removed 事件按 `previous: { equals: "fastly" }` 命中；React 无事件 → 返回空数组（正确）。
- 输出带 website（target.domain）、前后值、时间。

---

# 10. Evidence（证据）

- 只消费既有 Evidence：Fact(technologies).evidenceIds → Evidence 行，批量 id 查询（无 N+1）。
- `total` = 去重后的 evidenceIds 总数（真实计数）；`items` = 最近 20 条样例（id/source/collector/observedAt/confidence）。
- 当前 Evidence 的 per-technology 链接（entityId、rawValue 明细）仍为占位/通用——如实返回可用部分，`coverage.hasEvidence` 按真实计数给出；完整 Evidence 契约强化属 Phase 2.7。
- 生产实测：React → 16 facts → 18 evidence ids。

---

# 11. Coverage（覆盖）

```text
observedWebsiteCount : 真实 distinct 域名数
hasHistory           : 是否存在 related events
historicalCoverage   : partial（有观测事件）| insufficient（无）
geographic           : unavailable（Phase 2.2 不计算分布）
hasDistributionData  : false
hasEvidence          : 真实 evidence 计数 > 0
```

无新 Coverage 表，全部派生。

---

# 12. Caching（缓存）

- 复用 Step 8 `ttlMemo`（`src/lib/read-cache.ts` 新增 `cachedTechnologyProfile`）。
- Cache key：`technology-profile:{slug}:{page}:{pageSize}`（含分页，避免分页相互污染）。
- TTL：60s（与 report/search 一致）；失效：TTL 驱动（低写公共数据，无显式 invalidation）。
- Query fingerprint：slug + 分页参数即指纹；缓存命中时 0 次 DB 查询。

---

# 13. Performance（性能）

- 未缓存请求 ≤ 10 条 SQL，全部走索引/主键：
  1. Catalog slug（unique） 2. Entity id（PK）/normalized（index/unique） 3. adoption `$queryRaw`（targetId index + entity PK） 4. websites 分页（targetId index） 5. events（JSONB equals，量小） 6–10. evidence 批量（Entity id in / Target.domain unique / Fact unique(targetId,factType) / Evidence PK + factType index）。
- 无 1+N：websites 单查询、evidence 批量。
- Index 评估：无需新增索引；`Event(type, current/previous)` 专项索引留待 Phase 2.4（计划 S3，按查询依据再评估）。
- 生产实测查询语义全部通过（React 16 站 / 18 evidence / fastly 事件命中）。

---

# 14. Error Handling（错误处理）

| 场景 | 行为 |
|---|---|
| 无/无效 API Key | 401（与 v1 一致） |
| 速率限制 / 配额超限 | 429（retryAfterSec） |
| 非法 slug（格式/长度） | 400 |
| 非法分页参数 | 400 |
| 技术不存在（Catalog 与 Entity 均无） | 404 `{ error: "Technology not found" }` |
| Catalog 缺失（表未迁移） | Entity-only 降级，不 500 |
| 元数据部分为 null | 部分响应，不失败 |
| 无网站/无事件/无证据 | 空数组 + 真实 total=0 |

---

# 15. Backward Compatibility（向后兼容）

- 未删除/修改任何既有 endpoint（technology reverse lookup、website report、/technology/[slug] 页面、v1 analyze/report 均不变）。
- `seo/technology.ts` 与页面组装未改动；公共查询逻辑未复制——Profile 组装独立成 `src/lib/technology/profile.ts`，页面层 Phase 2.3 可复用。
- read-cache 仅追加导出，不影响既有 report/search 缓存。

---

# 16. Tests（测试）

```text
existing tests: 365
new tests:      19（src/lib/technology/profile.test.ts）
total:          384 / 384 PASS（27 files）
typecheck:      tsc --noEmit 0 错误
```

新增覆盖：

- Identity：有效技术 / 未知技术（404）/ Catalog 表缺失降级 / 部分元数据 null
- Adoption：websiteCount / firstSeen / lastSeen / 未观测技术零值
- Relationships：分页 skip/take / 确定性排序 / www legacy 去重 / 空页
- Events：added+removed / 空结果
- Evidence：存在（批量 + 精确 total）/ 不存在
- Error：404 / 部分 Catalog
- Cache：命中 / 失效重算 / TTL 过期 / 分页独立 key
- 纯函数：fingerprint slug→catalog slug 映射、检测方法推导、alias 解析、实体候选（Iconfont 案例）

---

# 17. Build（构建）

```text
next build（Turbopack，Next 16.3.1）: PASS
路由注册：ƒ /api/v1/technology/[slug]（dynamic）
既有 crypto/Edge warning：jobs.ts 历史警告，与本阶段无关
```

---

# 18. Schema Changes（Schema 变更）

**NO。**

- 无新表、无新索引、无 migration、无 Prisma schema 修改。
- 未创建 History / Distribution / Ecosystem / Alternatives 表（与计划一致）。
- 本阶段唯一 DB 相关动作：针对生产库的**只读**验证查询（见 §19）。

---

# 19. Production Status（生产状态）

| 项 | 状态 |
|---|---|
| 生产代码部署 | NO（服务器仍为 9eec3de + Phase 2.1 migration/seed） |
| 生产 migration / seed | NO |
| 服务重启 | NO（MainPID 未变，零重启） |
| 生产 schema 修改 | NO |
| 生产只读验证 | ✅（React 16 站 / 18 evidence；fastly 事件命中；临时脚本已清理） |
| GitHub Push | NO |

---

# 20. Files Changed（文件变更）

```text
src/lib/technology/profile.ts                    （新增：Profile 组装 + 类型 + 纯函数）
src/lib/technology/profile.test.ts               （新增：19 例）
src/app/api/v1/technology/[slug]/route.ts        （新增：v1 API）
src/lib/read-cache.ts                            （修改：+cachedTechnologyProfile，TTL 60s）
```

未包含：pnpm-workspace.yaml（用户本地修改）、.next、.env、文档、临时文件。

---

# 21. Risks（风险）

| 风险 | 缓解 |
|---|---|
| Event 查询依赖 JSON equals + 映射值集合 | 纯函数测试 + 生产实测（fastly 命中）；值集合覆盖 catalog slug ∪ fingerprint slug |
| Evidence 为通用占位（无 per-technology 链接） | 如实返回可追溯部分（facts→evidenceIds），`hasEvidence` 真实；Phase 2.7 强化 |
| www legacy 去重影响 total 语义 | 与既有 P1-1 口径一致，文档明确 observed 口径 |
| 内存缓存 TTL 60s 带来短暂陈旧 | 低写公共数据，与 report/search 一致 |
| Catalog 表缺失降级路径 | try/catch + Entity-only 降级 + 单测覆盖 |
| 未建 Event(type,current) 索引 | 当前 6 条事件无性能风险；Phase 2.4 按查询依据再评估（计划 S3） |

---

# 22. Non-Goals（明确不做）

1. ❌ Technology Profile 完整 UI（Phase 2.3）
2. ❌ Technology History 完整功能（Phase 2.4）
3. ❌ Distribution / Market Share / Country / Language 页面（Phase 2.5）
4. ❌ Ecosystem / Alternatives 完整功能（Phase 2.6）
5. ❌ Evidence 系统重构（Phase 2.7）
6. ❌ Security Intelligence / Global Explore / 全站 Historical Intelligence
7. ❌ 新搜索引擎、大规模 schema 重构
8. ❌ 生产部署、migration、seed、服务重启
9. ❌ GitHub push

---

# 23. Commit（提交）

```text
Repository: Duoniu-ai/siteintel（本地 master，未 push）
Message:    feat(technology): add technology profile backend
Files:      src/lib/technology/profile.ts · profile.test.ts · api/v1/technology/[slug]/route.ts · read-cache.ts
```

Commit: `99466ee`（feat(technology): add technology profile backend，4 files，+1018）

---

*Phase 2.2 完成，立即停止。未进入 Phase 2.3 及以后阶段。*


