# SiteIntel 2.0 — Phase 2.5 Technology Distribution 验收报告

> 日期：2026-08-19 ｜ 范围：Phase 2.5（Technology Distribution only）
> 授权：按《SITEINTEL 2.0 — Phase 2.5 Technology Distribution 执行指令》执行
> 结论：✅ 完成（Distribution 后端聚合 + 前端区块 + Coverage 语义；无 Schema 变更；未部署生产；未 push）

---

# 1. Scope（范围）

在 Technology Profile（/technology/[slug] 与 GET /api/v1/technology/[slug]）中增加 Observed Distribution：

```text
Observed websites
Country distribution（真实 IP geo）
Language distribution（真实 metadata.language）
Infrastructure / ASN distribution（真实 belongs_to 关系 + Organization）
Distribution coverage（已知数 / 百分比 / 分母明确）
```

所有数据均为 SiteIntel 观察口径，**不是全球市场份额**。

---

# 2. Data Sources（数据来源）

| 维度 | 数据源 | 说明 |
|---|---|---|
| Country | Website → `uses` tech → domain → `resolves_to` IP entity → `metadata.country` | 真实 IP geo；空值不参与已知 |
| Language | Website → Target → `Fact(metadata).language` | 真实页面语言；按 ISO 639-1 基础码归一化 |
| ASN | Website → IP → `belongs_to` ASN entity → `belongs_to` Organization | 真实关系，不从 IP 字符串猜 ASN |
| Websites | `uses` 关系 distinct 域名（既有口径） | 观察单位 |

未新增采集、未修改 crawler、未创建任何 fake records。

---

# 3. Aggregation Logic（聚合逻辑）

- 纯函数 `aggregateTechnologyDistribution`：输入查询行，输出分布（可单测）。
- Country：每站每国去重（同一网站多个 IP 同国只计 1）；多国网站（CDN/anycast）如实出现在每个桶。
- Language：`zh-CN`/`zh`/`zh-cmn-Hans` → `Chinese`，`en-US`/`en` → `English`；仅 ISO 639-1 已知码参与已知语言（`tc` 等非标码按 unknown 排除）。
- ASN：展示归一化为站点惯例 `AS37963`；组织来自 ASN → Organization 关系（无则 null）。
- 排序：count DESC，再 name/ASN ASC（deterministic）。

---

# 4. Deduplication（去重）

- Website 为观察单位：同一 website 在同一桶内只计一次（按 normalized domain 去重）。
- 多 IP 同国 / 多 IP 同 ASN：桶内去重（生产实测 React：多 IP 同国计 1）。
- 多国网站：出现在其真实关联的每个国家桶（覆盖计数仍只计一次）。

---

# 5. Coverage Semantics（覆盖语义）

- 分母 = 观察网站总数；已知数 = 具有该维度数据的 distinct websites。
- 百分比 = round(known / websites × 100)；websites=0 → null（前端显示 Not available）。
- 示例：websites=16，countriesKnown=16 → Country data 100%；如 known=14 → 88%。
- 每项均可在 UI 看到明确文案："Based on SiteIntel observed data; a website may appear in multiple country / ASN buckets."

---

# 6. Backend（后端）

- 扩展现有统一 payload（无第二套 API、无独立历史查询体系）：
```jsonc
distribution: {
  websites, countries: [{name, websites}], languages: [{name, websites}],
  asns: [{asn, organization, websites}],
  coverage: { countriesKnown, languagesKnown, asnsKnown,
              countriesPercent, languagesPercent, asnsPercent },
  scope: "observed"
}
```
- 保持 X-API-Key / rate limit / quota / 400 / 404 / 429 / ttlMemo（cache key 不变，slug+page+pageSize）。
- 查询为批量聚合（country / asn / org / language 各 1 条），与 evidence 共享一次 website 遍历，无 N+1、无重复查询。

---

# 7. Frontend（前端）

- /technology/[slug] 新增 section 03 Distribution（后续区块顺延 04–08）：
  - 三张 compact card：Countries / Languages / Infrastructure (ASN)，ranked list + 数量。
  - 排名列表每类最多 10 项 + "+N more"。
  - Coverage chips：Country data / Language data / ASN data（百分比或 Not available）。
  - 明确 observed 口径 note；无地图、无饼图、无 dashboard 化。
- 移动端：grid 自动堆叠、truncate、无宽表。

---

# 8. SEO（SEO）

- 未新增 URL；canonical / title / description / robots / index partition 完全不变。
- Distribution 属于既有 /technology/[slug] 内容，不改变 SEO gate（Phase 2.3 校准的 INDEX 分区不受影响）。

---

# 9. Performance（性能）

```text
Cached:     0 database queries（ttlMemo）
Uncached:   bounded indexed queries（既有 ≤10 + 新增 country/asn/org/language 4 条批量查询 ≈14，
            全部走索引；无每国家/语言/ASN 单查）
No N+1:     ✅（一次 uses 遍历共享给 evidence + distribution）
```

---

# 10. Tests（测试）

```text
existing tests: 401
new tests:      11
total:          412 / 412 PASS（28 files）
```

新增覆盖：

- Country 聚合 + 桶内去重（多 IP 同国计 1）
- 多国网站：进入多个桶，覆盖计数只计一次
- Language 归一化（zh-CN/zh→Chinese；en-us→English；"" 与 "tc" 排除）
- ASN 聚合 + Organization + 展示归一化（37963→AS37963）
- Coverage 百分比（3 站已知 2 → 67%）
- Deterministic 排序（count desc / name asc）
- 空分布（0 站 → 空数组 + null 百分比）
- Assembly 集成（真实查询形状 → distribution 填充）
- View：分布区块映射、覆盖率标签、10 项上限 + "+N more"、缺失数据 Not available

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

**NO。** 无 Schema / Prisma / migration / seed / 数据修改。仅生产**只读**验证（临时脚本已清理）。

---

# 14. Production Verification（生产验证，只读）

React 实测：

```text
websites: 16
countries: China 12 · Hong Kong 3 · Singapore 2
asns:      AS37963 7 · AS55990 2 · AS45102 2 · AS132203 1 · AS4134 1
           （org 样例：Tencent Cloud / Amazon Data Services Singapore / China UNICOM）
languages: zh 7 · en 3（tc 等非标码按 unknown 排除）
```

查询链（uses → resolves_to → belongs_to → metadata fact）在生产数据上全部命中。

---

# 15. Git Commit（提交）

```text
Repository: Duoniu-ai/siteintel（本地 master，未 push）
Message:    feat(technology): add technology distribution intelligence
Files:      src/lib/technology/profile.ts · profile.test.ts · profile-view.ts ·
            profile-view.test.ts · src/lib/i18n.ts · src/app/technology/[slug]/page.tsx
```

Commit: ad8ed30（feat(technology): add technology distribution intelligence，6 files，+750/−31）

---

# 16. Known Limitations（已知限制）

1. Country/ASN 为 IP 级 geo/关系聚合：同一网站多 IP（CDN/anycast）会出现在多个桶，百分比分母为“已知网站数”，桶内求和可 >100%——UI 已明确说明。
2. Language 仅覆盖有 metadata.language 的网站；非标码（如 "tc"）按 unknown 排除（保守，不冒充已知）。
3. Organization 仅来自 ASN→Organization 关系，缺失时显示 null。
4. 观察口径不等于全球分布；不提供 market share / trend / forecast。
5. 生产视觉验证待部署后进行。

---

*Distribution = SiteIntel observed data，非全球市场份额。Phase 2.5 完成，立即停止。*
