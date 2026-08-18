# SiteIntel Phase 1 — Step 2 Safe Fetch / Discovery Probe 安全收口完成报告

> 日期：2026-08-18 ｜ 范围：P0-1（Discovery probe 统一纳入 safe-fetch 安全边界）
> 授权：用户逐项授权（Step 2 only）。**未执行**：/api/report、SSE、Event/Signal、Fact、Job System、Discovery 数量/候选策略、SEO、新业务功能、数据库 Schema 变更。
> 结论：✅ **完成（PASS）**

---

## 1. 修改目标

消除全项目唯一已确认的“用户可控主机名 + 无完整安全校验 + 原生 fetch”出站路径：`src/lib/discovery/probe.ts`。统一为 `resolveSafeAddresses()` → `safeFetch()`（地址验证 + IP 钉扎 + 逐跳重定向复验 + 端口白名单 + 超时/响应体控制 + 明确错误分类），不另建第二套网络模块。

## 2. 修改前攻击路径

```
匿名 POST /api/analyze 受控域名（DNS/CNAME/redirect/SAN 指向内网）
   → 候选入池（DiscoveryCandidate）
   → collector tick（15s）pickNextToProbe
   → probeDomain:
       dns.lookup(domain)      ← 地址【不校验】
       fetch("https://domain/") ← Node 原生 fetch
         redirect: follow（≤20 跳，无逐跳复验）｜无 body 上限｜无 IP 钉扎
   → 可直连 127.0.0.1 / RFC1918 / IPv6 私网 / 169.254.169.254 / 任意端口
```

## 3. 修改后请求路径

```
probeDomain(domain)
   → resolveSafeAddresses(domain, resolver)    ← 全部地址逐项校验，任一不安全整组拒绝
   → safeFetch(url, {
       timeoutMs: 8000, maxBytes: 64_000, maxRedirects: 6,
       allowedPorts: [80, 443],
       resolver: (h) => h === domain ? 已验证地址 : resolveSafeAddresses(h)
     })
   → 初始主机复用已验证地址（IP 钉扎，不二次解析）；每跳 redirect 重新校验
   → 失败分类：dns_fail / unsafe_address / blocked_port / unsafe_redirect /
                refused / tls_fail / timeout / http_ok / http_error
```

## 4. 修改文件

| 文件 | 改动 |
|---|---|
| `src/lib/security/safe-fetch.ts` | `allowedPorts`（默认 `[80,443]`）、`SafeFetchError.code`、每跳端口校验、解析/超时/跳数错误编码 |
| `src/lib/discovery/probe.ts` | 网络层整体重写：`resolveSafeAddresses` + `safeFetch` + resolver 钉扎 + 分类 |
| `src/lib/discovery/score.ts` | 安全拦截 reason 降权（unsafe_address/blocked_port/unsafe_redirect → 0.05）与中文标签 |
| `src/lib/discovery/collector.ts` | `probeReasonFromFailure` 返回类型收窄为 `ProbeReason`（兼容 `\w+` 提取，无需改逻辑） |
| `src/lib/security/safe-fetch.test.ts` | 端口策略、错误码、redirect 复验、公网 IPv6、超大响应截断、超时分类 |
| `src/lib/discovery/probe.test.ts` | 新增（25 例）：地址分类矩阵、DNS rebinding 钉扎、分类/回退 |
| `src/lib/discovery/score.test.ts` | 新增安全拦截评分用例 |

## 5. 删除/替换的旧调用

- 删除：`import { lookup } from "node:dns/promises"`；`fetch(url, { redirect: "follow", signal: AbortSignal.timeout(...) })`；`PROBE_UA`。
- 替换：`lookup` → `resolveSafeAddresses`（带 3s DNS 超时）；原生 `fetch` → `safeFetch`。
- 项目其余出站路径复核：http/metadata/ip/rdap/baidu/ownership 均已走 safeFetch；ssl 走 `resolveSafeAddresses`+tls 钉扎；notify/ai/Radar 为固定可信 URL（本轮范围外，记录在案）。

## 6. DNS rebinding 防护实现

- `resolveSafeAddresses` 一次性解析+校验 → 把已验证地址集通过 `resolver` 注入 `safeFetch`：初始主机名再次出现时直接返回钉扎地址，**不再触发 DNS 查询**（probe.test.ts 用计数证明：resolver 仅调用 1 次，第二次 DNS 答案（127.0.0.1）从未被使用）。
- `safeFetch` 在 socket 层用 `host: 已验证IP` + `Host/SNI` 连接，连接期间永不二次解析。
- redirect 到不同主机名时走 `resolveSafeAddresses` 新解析并同样校验（混合公网/私网答案整组拒绝）。

## 7. redirect 防护

- `safeFetch` 每跳循环重新 `resolveSafeAddresses` + 端口校验后才连接（初始 URL 安全不代表 redirect 安全）。
- 测试：初始公网 → 302 到 127.0.0.1 → `UNSAFE_REDIRECT` 且仅 1 次连接（第二跳拒绝在连接前）；302 到 `:8443` → `BLOCKED_PORT`。

## 8. allowedPorts 实现

- `SafeFetchOptions.allowedPorts` 默认 `[80, 443]`；每跳在连接前校验 `url.port`（空端口按协议默认 80/443）。
- 未开放 8080/8443/3000/8000/8001/9000 等；`BLOCKED_PORT` 错误码。
- 兼容性：现有全部调用方均为 80/443（ipwho.is 443 / uapis.cn 80 / rdap.org 443 / baidu 80 / 页面抓取默认端口），零行为回归（全套测试通过佐证）。

## 9. timeout

- probe：DNS 3s（withTimeout）+ HTTP 8s（`timeoutMs: 8000`）；超时分类 `timeout`，**不触发 http 回退**。
- safeFetch 默认 12s 保持；传输超时带 `TIMEOUT` 错误码。

## 10. response body 限制

- probe 传 `maxBytes: 64_000`（只读状态码）；safeFetch 在 socket 层截断（超限即停读并 destroy），永不完整缓冲超大响应。
- 测试：200 字节 body + maxBytes 50 → `text()` ≤ 50 字节。

## 11. blocked reason

| 场景 | reason（ProbeResult / 入库 `probe:` 前缀） | safeFetch 错误码 |
|---|---|---|
| 私网/回环/链路本地/metadata/IPv6 私网 | `unsafe_address` | UNSAFE_ADDRESS |
| redirect 到不安全地址 | `unsafe_redirect` | UNSAFE_REDIRECT |
| 非允许端口 | `blocked_port` | BLOCKED_PORT |
| DNS 不可解析 | `dns_fail` | UNRESOLVABLE / ENOTFOUND |
| 连接拒绝 | `refused` | CONNECTION_REFUSED |
| 超时 | `timeout` | TIMEOUT |
| 跳数超限 | `http_error` | TOO_MANY_REDIRECTS |
| 证书/握手 | `tls_fail` | — |

不再出现模糊的单一 “fetch failed”（错误详情保留在 `lastFailureReason`）。

## 12. 新增测试

- `probe.test.ts`：25 例（地址分类矩阵 11 组、DNS rebinding 钉扎 2 例、分类/回退 9 例、错误信息可读性）。
- `safe-fetch.test.ts`：+8 例（默认端口拒绝、显式放行端口、错误码、redirect 复验×2、公网 IPv6、超大响应、TIMEOUT 分类）。
- `score.test.ts`：+1 例（三种安全拦截 reason 降权）。

## 13. 测试结果

**292/292 通过（18 个测试文件）**；typecheck（tsc --noEmit）0 错误。

## 14. build 结果

本地 `next build` 通过；服务器 staging 与 production `pnpm install --frozen-lockfile + pnpm prisma generate + pnpm build` 均通过（生产 BUILD_ID `J8w-mcbJ2yp0RlugwaAVf`）。

## 15. staging 验证

- 06f2486→de8b6da 归档部署 `/www/wwwroot/siteintel.cc-staging`，构建成功，服务 active（:3004）。
- `/` `/robots.txt` `/sitemap.xml` `/website/wordpress.org` 全 200。
- `.next/server` 编译产物含 `allowedPorts` / `UNSAFE_REDIRECT` / `unsafe_address`（新代码确认编译生效）。
- 三调度器武装日志正常；写路径被只读角色硬拦截（staging 零污染）。
- 池内无可探候选（生产已全部 probe 过），probe 未自然触发——以编译产物 + 单元测试覆盖该路径。

## 16. production 验证

| 项 | 结果 |
|---|---|
| 服务 | active，NRestarts=0，新 PID 780717 |
| 公网 HTTP | 首页/robots/sitemap/website/example.com/api/report/api/tools/dns 全 200 |
| 正常网站分析 | example.com → completed（新调查 6s 内） |
| 被阻止目标无法访问 | localtest.me（解析 127.0.0.1/::1）→ partial；分步证据：`ssl [failed] private address ::1`、`http [failed] Could not connect` —— 安全边界端到端生效 |
| Discovery 正常运行 | collector armed（15s/50/50）、seed 窗口 fetched 43 去重 0 新增、预算记账 50/50 |
| 新 probe 代码生产实证 | 05:16:09 新进程 probe `gigjam.com → tls_fail 18333ms`（到期重探触发；错误详情含 redirect 目标 www.microsoft.com——逐跳复验路径证据） |
| Scheduler | monitor/search-sync/discovery 全武装，0 uncaught/fatal |
| Candidate/Investigation 异常 | 无：pending 1031→1031、analyzed 163→163、skipped 0；targets 405→406、investigations 669→671（+2 均为验证分析本身） |
| Git | 生产 HEAD = master = `de8b6da`；tracked 树与 HEAD 零差异 |

## 17. Git commit

`de8b6dafda9f9cf097640dd51369dd925f5789b4` — `fix(security): route discovery probe through safe-fetch with port/pinning guards`（独立 commit，仅 7 个相关文件，无混入）。

## 18. Git tag

`production-phase1-step2-safefetch-2026-08-18`（annotated）→ `de8b6da`，已推送 GitHub。

## 19. 回滚方式

- 代码：`git revert de8b6da`（或恢复 tag `production-phase1-baseline-2026-08-18`=f89ad89）→ 重新 build/restart。
- 生产文件级：`git checkout f89ad89 -- <7 文件>` + `pnpm build` + `systemctl restart siteintel`。
- 数据：本步**零 Schema / 零结构性变更**；唯一新增为验证分析的正常业务写入。
- 行为回归点：回滚后 probe 恢复原生 fetch（旧漏洞复现）——仅用于极端回滚场景。

## 20. 已知限制

1. probe 在 production 的自然触发依赖候选到期重探/新种子（本轮记录 1 次实证：gigjam.com）；更完整的生产观察将在未来 24h 随调度自然累积（含可能出现的 unsafe_address 候选，正是本修复的价值）。
2. notify（Telegram/AI）与 Cloudflare Radar 仍使用固定可信 URL 的非 safeFetch 通道（无用户可控主机名，风险低）；统一到 safeFetch 属后续工程项，不在本步范围。
3. `response_too_large` 目前以 socket 截断方式受控（不报错、不完整缓冲）；未引入“超限即失败”的严格模式（会改变既有 metadata 抓取语义，未授权不做）。
4. staging 的 AUTO_DISCOVER 配置卫生项（此前记录）仍未收口（不属于本步）。

---

**完成，停止。** 未自动进入 P0-3（/api/report）与 P0-4（SSE）；等待下一步授权。
