# SiteIntel Phase 1 — Step 3 /api/report 最小限流完成报告

> 日期：2026-08-18 ｜ 范围：P0-3（/api/report 匿名限流 + 真实客户端 IP 信任边界）
> 授权：用户逐项授权（Step 3 only）。**未执行**：SSE、数据库 Schema/Migration/新表、Discovery、Safe Fetch、SEO/Security/Keyword/Ranking、v1 重构、报告数据结构重构、Redis/Upstash/新基础设施。
> 结论：✅ **完成（PASS）**

---

## 1. 当前调用关系

- `/api/report/[domain]`（GET，`src/app/api/report/[domain]/route.ts`）唯一调用方 = 报告页“导出 JSON”按钮（`report-page.tsx:1162` → `ExportJsonButton` 客户端 fetch）。
- 报告页本身**不依赖**该端点（服务端 `assembleReport` 直读数据库）。
- 无内部服务调用、无重复请求、无合法 crawler（robots.txt 已 disallow `/api/`）。
- 与 v1 关系：`/api/v1/report` 需 X-API-Key + 每 key 限流/配额；`/api/report` 此前匿名无限流，可绕过 v1 配额批量抓取。

## 2. 当前风险

- **无限流**：机器可匿名批量拉取完整报告 JSON（含 evidence rawData）。
- **IP 可伪造（更严重）**：源站 80 端口可直连（实测 `http://154.204.176.66` + Host 头 → 200）；nginx 未配置 real_ip，透传客户端可控的 `CF-Connecting-IP`/`X-Forwarded-For`；旧 `clientIp()` 信任 XFF 首项（Cloudflare 追加而非替换）→ `X-Forwarded-For: victim-ip` 可绕过限流并污染受害者配额。

## 3. 最终 rate limit

匿名按真实客户端 IP：**30 requests / hour**（滑动窗口，`checkRateLimit("report:" + ip, 30, 3_600_000)`）。

## 4. 为什么采用这个值

- 唯一消费场景是“导出 JSON”按钮，正常用户单次导出 = 1 请求；30/hr 覆盖重度使用，远高于正常导出频率。
- 与现有端点口径一致（analyze/tools 均为 20/hr，login 10/15min），30/hr 处于同一量级，不误伤正常用户。
- 实测 30 次内全部 200，第 31 次 429——边界清晰。

## 5. IP 获取方式

- 应用层 `clientIp()` 信任顺序改为：**`X-Real-IP`（nginx 写入，权威）→ `CF-Connecting-IP`（无 nginx 环境兜底，如 staging 直连）→ `unknown`**。
- **彻底移除 `X-Forwarded-For`**（其首项由客户端控制，Cloudflare 追加而非替换，永不信任）。

## 6. Cloudflare / proxy 信任边界

- nginx `siteintel.cc.conf` 新增（生产已生效）：
  ```
  set_real_ip_from <Cloudflare 官方 IPv4 15 段>
  set_real_ip_from <Cloudflare 官方 IPv6 7 段>
  real_ip_header CF-Connecting-IP;
  ```
- 效果：仅当直连对端 ∈ Cloudflare 段时，`$remote_addr` 还原为 CF-Connecting-IP（真实客户端）；非 CF 直连保持对端真实 IP。nginx 总是用 `$remote_addr` 覆盖 `X-Real-IP`，客户端无法注入。
- 实测验证（access log）：
  - 经 CF 请求 → `$remote_addr` = 本地真实公网 IPv6（240e:390:...），非 CF 边缘 IP；
  - 直连 + 伪造 `CF-Connecting-IP: 9.9.9.9` / `XFF: 8.8.8.8` → `$remote_addr` = 真实对端 IPv4（183.150.63.208），伪造无效。
- 配置备份：`siteintel.cc.conf.bak-step3-20260818`；`nginx -t` 通过；`nginx -s reload` 热生效（该机 nginx 为 BT 直启，非 systemd 管理，已确认）。

## 7. 实现方式

- `src/app/api/report/[domain]/route.ts`：normalize 后、assembleReport 前调用 `checkRateLimit`，复用现有内存滑动窗口（单实例，无新依赖）。
- `src/lib/ip.ts`：信任链修正（X-Real-IP → CF-Connecting-IP → unknown，删除 XFF）。
- `src/lib/ip.test.ts`：新增 5 例信任边界测试。

## 8. 是否使用共享存储

**否**。复用现有进程内存限流器（`src/lib/rate-limit.ts`），未引入 Redis/Upstash/新服务。已知限制：重启清零（现有代码注释已声明；生产验证后已重启清空测试桶）。

## 9. 429 行为

超限返回 `HTTP 429`，body 与项目现有错误格式一致：`{ "error": "Rate limit exceeded", "retryAfterSec": N }`（与 /api/analyze 同构）。

## 10. Retry-After

响应头 `Retry-After: <秒>`（实测 staging `3599`、production `3593`/`3582`）。

## 11. 测试结果

- 新增 `ip.test.ts` 5 例：X-Real-IP 优先、XFF 伪造被忽略、CF-Connecting-IP 伪造被忽略、无 nginx 时 CF 兜底、无头返回 unknown。
- 全量：**297/297 通过（19 文件）**；typecheck 0 错误；本地 build 通过。
- staging 真实 HTTP 矩阵（127.0.0.1:3004）7/7：
  1. 正常 200 ✅ 2. 30 次内全 200 ✅ 3. 第 31 次 429 + Retry-After ✅ 4. 不同 IP 独立桶 ✅ 5. 固定 IP + 轮换 XFF → 30 次后 429（XFF 无效）✅ 6. 仅 XFF（无信任头）→ unknown 桶 30 次后 429 ✅ 7. tools/dns 200 不受影响 ✅
- production 真实矩阵：
  - 经 CF：30×200 → 31st 429 + Retry-After ✅；限流后 5 次轮换 XFF 仍 429 ✅
  - 直连伪造轮换：30 次轮换 CF-Connecting-IP/XFF 全 200（同一真实 IP 桶）→ 31st 429 ✅
  - 其他 API：tools/dns 200、报告页 200、首页 200 ✅
  - 验证后重启清桶，`/api/report` 恢复 200 ✅

## 12. staging

代码部署 `/www/wwwroot/siteintel.cc-staging`（27bbf93），构建成功，服务 active（:3004），上述矩阵全部 PASS。

## 13. production

- nginx real_ip 信任边界：已配置并热重载（备份在位），access log 双向验证通过。
- 应用部署：3 文件（ip.ts / ip.test.ts / report route），`pnpm install + prisma generate + build` 成功，`systemctl restart siteintel`，BUILD_ID `ApSUYoilzyvSZ4_cbwNpb`，NRestarts=0。
- 生产 git HEAD/master 对齐 `27bbf93`，tracked 树零差异；服务 active。

## 14. Git commit

`27bbf93e5951cf95f9e5d163cb9cb51080778557` — `fix(api): rate-limit /api/report per real client IP (30/hr) with hardened IP trust`（独立 commit，仅 3 个文件）。

## 15. Git tag

`production-phase1-step3-report-ratelimit-2026-08-18`（annotated）→ `27bbf93`，已推送 GitHub。

## 16. 回滚方式

- 代码：`git revert 27bbf93` 或恢复 tag `production-phase1-step2-safefetch-2026-08-18`=de8b6da → build/restart。
- nginx：`cp siteintel.cc.conf.bak-step3-20260818 siteintel.cc.conf && nginx -t && nginx -s reload`。
- 数据：零 Schema/零结构性变更；无数据写入。
- 注意：回滚代码但保留 nginx real_ip 无害（IP 还原仍正确）；回滚 nginx 但保留新代码会使 X-Real-IP 变为 CF 边缘 IP（多用户共享桶，过度限流）——**必须成对回滚或成对保留**。

## 17. 已知限制

1. 内存限流重启清零（单实例可接受，多实例前不引入 Redis）。
2. Cloudflare IP 段需随官方更新维护（22 段已配置；更新周期建议季度或官方变更通知时）。
3. 源站 80 端口仍可直接访问（绕过 Cloudflare 安全特性，如 WAF/缓存）；本步通过 real_ip 修复了 IP 伪造，但未关闭直连入口（影响多 vhost，需单独决策）。
4. 导出按钮在用户同一 IP 下 30/hr 后短暂 429（验证后已重启清零；正常使用远低于该值）。

## 18. 后续建议

- 源站直连治理（防火墙/nginx allow 仅 Cloudflare 或独立评估）——独立闭环，需另行授权。
- Cloudflare IP 段自动化同步（cron 拉取 + nginx reload）——属运维自动化，另行决策。
- v1 与 /api/report 契约收敛（导出改服务端、匿名端点收口）——原方案 B，独立闭环。
- 多实例部署前将限流迁至共享存储（Redis）——当前单实例不需要。

---

**完成，停止。** 未开始 Step 4（SSE）；等待验收。
