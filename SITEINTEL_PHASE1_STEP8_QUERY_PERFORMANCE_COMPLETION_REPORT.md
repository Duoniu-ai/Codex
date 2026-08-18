# SiteIntel Phase 1 — Step 8 Query & Read Performance Foundation 完成报告

> 日期：2026-08-18 ｜ 授权：按 Step 8 方案实施（P0 + 批准的 P1）｜ 结论：✅ **完成（PASS）**

---

## 1. 实际修改文件

```text
src/lib/read-cache.ts                    （新增：ttlMemo 有界短 TTL 缓存 + React cache 包装）
src/lib/read-cache.test.ts               （新增：命中/过期/容量驱逐 3 例）
src/app/website/[domain]/page.tsx        （报告/搜索装配走缓存 → 消除 SSR 双次装配）
src/app/page.tsx                         （首页数据 60s TTL 缓存，key 含 locale）
src/app/sitemap.ts                       （sitemap 5min TTL 缓存）
src/app/api/report/[domain]/route.ts     （/api/report 30s TTL 缓存，限流之后）
```

## 2-3. 每项优化前后查询数量与 TTFB

**查询数量（静态计数）**

| 页面 | 优化前 | 优化后 |
|---|---:|---:|
| /website/[domain]（zh 默认） | ~32-38 SQL（报告×2 + 搜索×1） | ~16-19 SQL（报告×1 + 搜索×1；命中缓存后 0-1 SQL） |
| 首页 | ~15 SQL/请求 | 命中缓存 0 SQL；TTL 过期后 ~15 SQL |
| sitemap | ~12 SQL/请求 | 命中缓存 0 SQL |
| /api/report | ~12-14 SQL/请求 | 命中缓存 0 SQL |

**TTFB（源站 127.0.0.1:3003，前后各 2-3 次采样）**

| 页面 | 优化前 cold→warm | 优化后 cold→warm/hit |
|---|---:|---:|
| / | 156→21-29ms | 195→25ms（命中） |
| /website/example.com | 173→77ms | 122→42ms（命中） |
| /website/wordpress.org | 79→60ms | 77→41ms（命中） |
| /sitemap.xml | 158→63ms | 181→**5.5ms**（命中） |
| /api/report/example.com | 72→27ms | 78→**6ms**（命中） |

## 4. cache hit/miss

- 命中实证：sitemap 与 /api/report 第二次请求 TTFB 5-6ms（对比冷 78-181ms）；报告页第二次请求 41-42ms。
- 未命中路径：TTL 过期后回源计算（单测覆盖过期重算）；React cache 保证同请求内元数据+页面共享（即使 TTL 刚过期）。

## 5. EXPLAIN 前后

- 未新增/修改查询（缓存只减少调用次数，不改变执行计划）；Step 8 方案 EXPLAIN 基线（全部 Index Scan，0.03-0.12ms）保持不变。

## 6. staging

- 部署成功（build/active/页面 200）；只读库不影响读缓存验证（命中逻辑由单测+生产覆盖）。

## 7. production

- BUILD_ID `4iMBt9TwwpKSNBxk1fvGh`；服务 active、NRestarts=0、**0 fatal**；HEAD 对齐 `9eec3de`，tracked 零差异。
- 回归：首页/报告页/wordpress/sitemap/api-report 全部 200，响应大小与优化前一致（内容未变）。

## 8. 测试数量

全量 **354/354 通过（25 文件）**（新增 read-cache 3 例）；typecheck 0；build 通过。

## 9. typecheck / build

`tsc --noEmit` 0 错误；`next build` 成功。

## 10. Git commit

```text
b263f45 perf(read): add bounded short-TTL read cache helper (ttlMemo + React cache wrappers)
9413c30 perf(read): dedupe report/search assembly per SSR request via read cache
fa4718e perf(read): homepage 60s short-TTL cache
ec5a8a0 perf(read): sitemap 5-minute TTL cache
9eec3de perf(read): api/report 30s TTL cache (after rate limit)
```

siteintel main = 生产 HEAD = `9eec3de`。

## 11. Git tag

`production-phase1-step8-query-performance-2026-08-18` → 9eec3de（annotated，已推送）。

## 12. production HEAD

`9eec3de4c00e8c2520af2739b24d3832093ad76b`（= local master = GitHub main）。

## 13. rollback

- 单 commit 回滚：`git revert <commit>`（5 个独立 commit 可分别回滚）；缓存为应用层、无 Schema/数据变更。
- 全部回滚：恢复 tag `production-phase1-step7-job-system-2026-08-18`=75bd5ef → build/restart。

## 14. 未完成项目（有意跳过/暂缓，记录原因）

1. **P1-2 实体页查询并行化/合并**：**本轮未做**。原因：当前实体页 warm 46-81ms、无 N+1、顺序查询有界（≤15），重构 4 页 Promise.all 与签名改动属中风险/低即时收益；按原则 15 记录，待 10 万 Target 门禁（顺序延迟放大）再评估。
2. **P2 全部**（门禁索引、JSONB 下推、响应裁剪、保留执行）：方案明确为 10 万规模门禁触发；当前 ~400 Target 未达门禁，**未执行**（不提前堆索引/裁剪）。
3. 未启用运行时 SQL 计数仪表（避免生产配置变更）；查询数以静态计数+TTFB 为证。

## 15. 已知限制

1. 内存缓存为单实例语义（与 Job System 一致）；多实例前不引入 Redis。
2. 缓存 TTL：首页/报告 60s、sitemap 5min、api/report 30s——最新数据最多延迟相应 TTL（报告页 live 状态仍显示 checkedAt）。
3. 未缓存：SSE 进度、Monitor/安全状态、tools 实时查询、管理后台、用户态数据。
4. TTFB 为源站口径（Cloudflare 回源另加网络开销）；未做压力测试。

---

## GitHub Publication

- Repository: Duoniu-ai/Codex
- Branch: main
- Commit: 见提交结果（推送后 local HEAD = origin/main）
- Tag: production-phase1-step8-query-performance-2026-08-18（siteintel 仓库）
- File: SITEINTEL_PHASE1_STEP8_QUERY_PERFORMANCE_COMPLETION_REPORT.md
- Remote verification: PASS（推送后验证）

---

*完成，停止。未开始 Step 9；未引入 Redis/读副本/搜索集群等任何被禁基础设施。*
