# SiteIntel 2.0 — Phase 2.6 Technology Ecosystem & Alternatives 验收报告

> 日期：2026-08-23 ｜ 范围：Phase 2.6（最小范围：Related / Alternatives / Parent Exposure）
> 授权：用户明确指令（Gap Assessment 确认的最小范围，未提供独立执行书）
> 结论：✅ 完成（派生聚合 + Profile 展示；无 Schema 变更；未部署生产；未 push code）

---

# 1. Scope（范围）

严格限定为 Gap Assessment 确认的三项：

1. Related Technologies（co-use：同站共现）
2. Alternatives（same-category overlap：同分类网站重叠）
3. Parent Exposure（Catalog.parent 暴露）

全部基于现有数据与关系派生，未新增 Schema / Prisma 模型 / Migration。

---

# 2. Data Sources（数据来源）

- `TechnologyCatalog`：category / parent（身份与同分类候选来源）。
- `EntityRelationship(uses)`：website → technology（共现与重叠的唯一关系源）。
- `Entity`：technology 实体 value/normalized（关联展示名与 catalog slug 解析）。
- 无新采集、无 crawler 修改、无 fake records。

---

# 3. Aggregation Logic（聚合逻辑）

- `aggregateCoUse`（纯函数）：一次 `uses` 查询 → 按 tech 分组，website 桶内去重（normalized domain），排序 count DESC → name ASC，输出 `{ slug, name, sharedWebsites }`。
- `sameCategoryCatalogSlugs`：从冻结 Catalog 取同 category slug 集合（排除自身），作为 alternatives 过滤条件。
- `catalogSlugForTechnologyName`：canonicalName + curated aliases → catalog slug；未命中回退 URL slugify。
- Related 与 Alternatives **共享同一条查询**（零额外 DB 调用）。
- Parent：`resolveCatalogParent(catalog.parent)` → `{ name, slug } | null`，slug 经 catalog 解析。

---

# 4. Deduplication（去重）

- 同一 website 在同一 related/alternative 桶内只计一次（www legacy 经 normalized domain 去重，复用既有 P1-1 口径）。
- 同一 tech 只出现一次（按 targetId 分组）。

---

# 5. Coverage Semantics（覆盖语义）

- Related / Alternatives 均为 **observed co-occurrence**（真实 uses 关系），非全球生态或市场数据。
- 无重叠数据时返回空数组（不猜测、不伪造）。
- UI 文案保持 observed 口径；不展示 market share / 完整生态。

---

# 6. Backend（后端）

统一 payload 新增（`GET /api/v1/technology/[slug]`，无新端点）：

```jsonc
technology.parent: { name, slug } | null
related:  { total, items: [{ slug, name, sharedWebsites }] }   // top 8
alternatives: { total, items: [{ slug, name, sharedWebsites }] } // top 8
```

- 保持 X-API-Key / rate limit / quota / 400 / 404 / 429 / ttlMemo。
- 未缓存请求新增 1 条批量 co-use 查询（与现有 evidence/distribution 共用一次 website 遍历数据），无 N+1。

---

# 7. Frontend（前端）

- Identity 面板新增 Parent 行（链接 /technology/[slug]）。
- Explore 区块新增两组 chips：Related technologies（co-used）、Alternatives (same category)，显示 `×共享网站数`，链接到对应技术页。
- 空数据不渲染区块；无新 URL；不改变 SEO gate。

---

# 8. SEO（SEO）

- 未新增 URL；canonical / title / description / robots / index partition 不变。
- Related / Alternatives 为 /technology/[slug] 内部内容，内链指向既有技术页。

---

# 9. Performance（性能）

```text
Cached:     0 database queries（ttlMemo）
Uncached:   bounded indexed queries（+1 条 co-use 批量查询；related/alternatives 共享）
No N+1:     ✅
```

---

# 10. Tests（测试）

```text
existing tests: 412
new tests:      6（backend 4 + view 2）
total:          418 / 418 PASS（28 files）
```

新增覆盖：

- co-use 聚合 + 桶内去重 + 确定性排序
- alternatives 同分类过滤（Tailwind CSS javascript 入选；Next.js framework 排除）
- catalog slug / parent 解析（含未知回退、null）
- assembly 集成（真实查询形状 → related/alternatives/parent）
- view 映射 + 空态

---

# 11. Typecheck（类型检查）

```text
tsc --noEmit: 0 错误（PASS）
```

---

# 12. Build（构建）

```text
next build（Turbopack，Next 16.3.1）: PASS
路由：ƒ /technology/[slug]（dynamic）· ƒ /api/v1/technology/[slug]
唯一 warning：jobs.ts crypto/Edge 历史警告（与本阶段无关）
```

---

# 13. Database（数据库）

**NO。** 无 Schema / Prisma / migration / seed / 数据修改。

---

# 14. Production Verification（生产验证）

⚠️ 未执行：SSH host key 变更（154.204.176.66:613 ED25519 fingerprint 与 known_hosts 不一致），按安全规则未绕过验证、未修改 known_hosts。生产只读验证待用户确认后补做；查询形状已由单测 fixture（与 Phase 2.5 生产实测同构）覆盖。

---

# 15. Git Commit（提交）

```text
Repository: Duoniu-ai/siteintel（本地 master，未 push）
Message:    feat(technology): add technology ecosystem and alternatives
Files:      src/lib/technology/profile.ts · profile.test.ts · profile-view.ts ·
            profile-view.test.ts · src/lib/i18n.ts · src/app/technology/[slug]/page.tsx
```

Commit: 647393e（feat(technology): add technology ecosystem and alternatives，6 files，+346/−16）

---

# 16. Known Limitations（已知限制）

1. Related / Alternatives 仅基于 SiteIntel 观察到的共现/重叠，不代表完整生态或替代关系。
2. Alternatives 仅含与目标技术有真实网站重叠的同分类技术；无重叠的“理论替代”不展示。
3. 列表上限 top 8（后端常量），无分页（后续如需可扩展）。
4. 生产只读验证因 SSH host key 变更暂缓（安全起见）。

---

*Phase 2.6 完成，立即停止。未进入 Phase 2.7；未部署生产；未 push。*
