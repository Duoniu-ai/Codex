# SiteIntel Phase 1 — P0 安全收口方案（分析稿）

> 文档性质：**只读分析 + 实施计划（未执行任何修改）**
> 日期：2026-08-18 ｜ 依据：Gap Analysis Report（`SITEINTEL_2.0_GAP_ANALYSIS_REPORT.md`）P0 清单
> 范围：P0-1 Safe Fetch / P0-2 Baseline / P0-3 Report API / P0-4 SSE
> 原则：先分析后执行；一项一闭环；等授权再实施；不顺手重构；不加未来表；不引入新架构

---

# 0. 总览与执行顺序结论（先读这里）

| 项 | 结论 | 依赖 | 建议顺序 |
|---|---|---|---|
| P0-2 Baseline | 生产代码**无法从 main 重建**（multi-source 未合入）；但分支与 main 可快进合并 | 无 | **第 1**（先行） |
| P0-1 Safe Fetch | probe 是唯一“未验证主机名出站”路径；改造需在统一后的 main 上做 | P0-2 | 第 2（与 P0-2 同一发布批次最稳） |
| P0-3 Report API | 前端导出按钮**依赖匿名访问**；需限流 + 数据裁剪，不能直接关闭 | 无 | 第 3 |
| P0-4 SSE | 匿名调查 id 公开可读；需流 token + 前端联动 | 无 | 第 3 |

**为什么 P0-2 先于 P0-1**：probe.ts 只存在于 `multi-source/phase0-cfradar` 分支（main 和本地 master 均没有）。若先改 probe，改动落在未合入分支上，基线更加分裂。正确做法：先把 multi-source 评审并入 main（FF，3 commits，16 文件），在统一基线上修 probe，一次发布同时消除“安全缺口”和“基线分裂”。

---

# 1. P0-1：Discovery probe 绕过 safe-fetch

## 1.1 真实调用链（生产 multi-source 分支）

```
DiscoveryCandidate(status=pending)
   ↓
src/lib/discovery/collector.ts — tick() 每 15s 单飞
   ├── ① pickNextToAnalyze（probeStatus=ok 且退避到期）→ investigate() → startInvestigation（走 safe-fetch ✅）
   └── ② pickNextToProbe（未探/到期重探）→ probe(candidate)   ← 本项
          ↓
       probeDomain(candidate.domain)   （src/lib/discovery/probe.ts）
          ├── lookup(domain)            ← node:dns/promises，3s 超时，地址【不校验】
          ├── fetch("https://domain/")  ← Node 原生 fetch，8s AbortSignal.timeout
          │       redirect: "follow"（Node 默认 ≤20 跳）| 无 body 上限 | 无 IP 钉扎
          │       TLS/连接失败 → 回退 fetch("http://domain/")
          └── ProbeResult{reachable,statusCode,latencyMs,reason}
          ↓
       DiscoveryCandidate 写回：probeStatus / lastStatusCode / probeLatencyMs /
       probeFailureCount / lastFailureReason / nextRetryAt（指数退避 1h→168h，7 次后 skipped）
          ↓
       rescore() → computeScore()（accessibilityFactor 按 reason 降权）
```

## 1.2 当前使用的 fetch 实现（全量出站调用清单）

| 路径 | 网络实现 | 主机名是否用户可控 | 是否走安全校验 |
|---|---|---|---|
| `discovery/probe.ts` | `dns.lookup` + 原生 `fetch` | **是（候选域名）** | **❌ 无任何校验** |
| `providers/http.ts` | `safe-fetch` | 是 | ✅ |
| `providers/metadata.ts` | `safe-fetch` | 是 | ✅ |
| `providers/ip.ts` | `safe-fetch`（固定 ipwho.is/uapis.cn/rdap.org） | 固定 | ✅ |
| `providers/rdap.ts` | `safe-fetch`（固定 rdap.org/verisign） | 固定 | ✅ |
| `providers/ssl.ts` | `tls.connect` + `resolveSafeAddresses`（IP 钉扎 + SNI） | 是 | ✅ |
| `search/baidu-official.ts` | `safe-fetch`（固定 data.zz.baidu.com） | 固定 | ✅ |
| `ownership.ts` | `safe-fetch`（验证文件/meta，域名用户可控） | 是 | ✅ |
| `notify.ts` Telegram | `fetchWithTimeout`（固定 api.telegram.org） | 固定 | ⚠️ 无校验但固定可信 |
| `notify.ts` Webhook | `fetchWithTimeout` + 每跳 `isSafeWebhookUrl` | 用户配置 | ✅（逐跳复检） |
| `ai.ts` | `fetchWithTimeout`（固定 open.bigmodel.cn） | 固定 | ⚠️ 无校验但固定可信 |
| `discovery/sources/cloudflareRadar.ts` | 原生 `fetch`（固定 api.cloudflare.com） | 固定 | ⚠️ 无校验但固定可信 |

**结论**：全项目唯一“用户可控主机名 + 无校验”的出站路径 = `probe.ts`。其余未走 safe-fetch 的都是固定可信 URL（Telegram/AI/Cloudflare Radar），风险低，仅“统一网络层”原则层面欠账。

## 1.3 safe-fetch 真实实现（`src/lib/security/safe-fetch.ts` + `ssrf-guard.ts`）

1. `new URL(rawUrl)` → 仅 http/https。
2. `resolveSafeAddresses(hostname)`：IP 字面量直接分类；主机名 `dns.lookup(all, verbatim)` 全量解析；**任一地址不安全即整组拒绝**（防混合 DNS 答案）。
3. `isBlockedAddress`：IPv4（0/8、10/8、127/8、100.64/10、169.254/16、172.16/12、192.168/16、192.0.0/24、TEST-NET、198.18/15、224+）；IPv6（::1、::、fe80、fec0、fc/fd、ff、2001:db8）+ **内嵌 IPv4 解包**（::ffff:x、NAT64 64:ff9b、6to4 2002::）。
4. **IP 钉扎**：`http(s).request({host: ip, servername: hostname, headers.host})`——socket 层不再二次解析（防 DNS rebinding）。
5. **逐跳重定向复验**：每个 redirect 目标在新循环中重新 resolve+validate；跳数上限 6。
6. **资源上限**：timeout 12s（默认）、maxBytes 1.5MB、编码解码（gzip/deflate/br）。
7. 未覆盖：**端口白名单**（任意端口放行，仅校验地址公网性）。

## 1.4 为什么形成第二套 Fetch

- 历史：probe 是 2026-08-17 多源 Phase 0 新增（`e32bd93`/`15de2d9`/`06f2486`），当时作为“轻量预检”独立实现，未复用 safe-fetch；safe-fetch 的 pin/redirect 语义需要 caller 配合（resolver 注入），probe 作者选择了更简单的原生 fetch。
- 结果：违反规划 §49“禁止业务层自行实现第二套 Fetch”，且引入真实 SSRF 攻击面。

## 1.5 当前实际 SSRF 风险（攻击路径）

**可达路径**：攻击者（匿名）POST /api/analyze 一个受控域名 → 该域名 DNS/CNAME/redirect/SAN 指向内网主机名或私有 IP → 候选入池 → collector 15s 内 probe 该候选 → `fetch` 直接连接私有地址。

实际影响分级：
- 内网探测：可访问生产服务器同网段内网服务（154.204.176.x 宿主内网）——**高**。
- 云元数据：当前服务器为物理 VPS（154.204.176.66），非云 metadata 场景；若未来迁云则 169.254.169.254 直读——**潜在高**。
- DNS rebinding：`lookup` 与 `fetch` 各解析一次，攻击者可做“公网解析→内网解析”切换——**中高**。
- redirect bypass：`fetch redirect:follow` 直接跟随到内网地址——**高**。
- 非标准端口：fetch 可访问公网主机任意端口——**中**。
- 超大响应：无 body 上限（probe 丢弃 body，但响应仍被 fetch 完整缓冲）——**低-中**（内存）。

**结论：必须修复。**

## 1.6 safe-fetch 已覆盖 vs probe 需要的最小兼容能力

| 能力 | safe-fetch 现状 | probe 最小需求 | 需改动 |
|---|---|---|---|
| localhost/127.0.0.1/private IPv4 | ✅ | ✅ | 无 |
| IPv6 回环/私网/内嵌 IPv4/NAT64/6to4 | ✅ | ✅ | 无 |
| link-local / cloud metadata | ✅ | ✅ | 无 |
| DNS rebinding | ✅（解析一次 + IP 钉扎） | ✅ | 无（probe 传入已解析地址即可） |
| redirect bypass | ✅（逐跳复验） | ✅ | 无 |
| 非标准端口 | ⚠️ 任意端口放行 | 建议拒绝 | **safe-fetch 加 `allowedPorts`（默认 80/443）** |
| timeout | 12s 默认（可配） | 8s（保持现状） | 传参即可 |
| response size | 1.5MB 默认（可配） | 64KB 足够（probe 只读状态码） | 传参即可 |
| redirect count | 6 默认（可配） | 6 | 无 |
| DNS 失败分类 | 抛 SafeFetchError | 需区分 dns_fail/blocked | **probe 层分类** |
| TLS/连接失败分类 | 抛 SafeFetchError | 需区分 refused/tls_fail/timeout | **probe 层分类** |
| 初始地址复用（防二次解析） | resolver 参数支持 | ✅ | probe 注入 hostname→已校验地址 |

## 1.7 最小修改方案（实施时才执行，当前不修改）

**改动文件（预计 4-5 个）**：

1. `src/lib/security/safe-fetch.ts`
   - 新增 `allowedPorts?: number[]`（默认 `[80, 443]`）；`requestOnce` 与每跳 URL 校验端口。
   - 保持 API 向后兼容（默认即限制标准端口；现有全部调用方均为 80/443，已验证）。
2. `src/lib/discovery/probe.ts`（重写内部网络层）
   - `resolveSafeAddresses(domain)` 一次性解析+校验（错误分类见下）。
   - `safeFetch(url, { timeoutMs: 8000, maxBytes: 64_000, maxRedirects: 6, allowedPorts: [80, 443], resolver: async (hostname) => hostname === domain ? validated : resolveSafeAddresses(hostname) })`。
   - HTTPS 失败 → HTTP 回退一次（保持现状语义；回退路径同样走 safeFetch）。
   - 错误分类映射：
     - `ENOTFOUND / EAI_AGAIN / host does not resolve` → `dns_fail`
     - `private address / blocked` → 新 reason `blocked`
     - 我方超时 / TimeoutError → `timeout`
     - `ECONNREFUSED` → `refused`
     - 证书/握手错误 → `tls_fail`
     - 跳数超限 / 其余 → `http_error`（statusCode null）
3. `src/lib/discovery/score.ts`
   - `accessibilityFactor` 增加 `blocked` 分支（factor 0.05，标签“地址被安全策略拦截”），避免落入 TLS 兜底误标。
4. `src/lib/discovery/collector.ts`
   - `probeReasonFromFailure` 同步支持 `blocked`（正则已按 `^probe: (\w+)` 提取，自动兼容）。
5. 测试（新增 `probe.test.ts` + 扩展 `safe-fetch.test.ts`）。

**不修改范围**：不改候选提取/评分权重/退避序列/预算/调度节拍；不改 collectAll 与 7 个 provider；不改 notify/ai（固定可信 URL，另行记录）；不改数据库。

## 1.8 测试方案（新增清单）

`src/lib/discovery/probe.test.ts`（新）：
- 127.0.0.1 / localhost 主机名 → `blocked`，不发连接。
- ::1、fe80:: 链路本地、169.254.169.254 → `blocked`。
- DNS 返回“公网+私网”混合 → `blocked`。
- redirect 到私网地址 → 第二跳 `blocked`（probe 结果不可达）。
- 非标准端口（:8080）→ 拒绝（allowedPorts）。
- ENOTFOUND → `dns_fail`；ECONNREFUSED → `refused`；超时 → `timeout`；TLS 失败 → `tls_fail`；>6 跳 → `http_error`。
- 大响应（>64KB）→ 仍返回状态码、内存受控。
- DNS rebinding 模拟：resolver 第二次返回私网地址 → 连接仍用首次校验地址（IP 钉扎）。

`src/lib/security/safe-fetch.test.ts`（扩展）：
- allowedPorts 默认拒绝 8080/8443，允许 80/443。

`src/lib/discovery/score.test.ts`（扩展）：
- `blocked` reason → factor 0.05 且标签正确。

## 1.9 staging / production 验证

- staging：完整候选链路实测 10-20 条（seed + 内源），确认 probeStatus/score/退避正常、日志无 blocked 误报；对照 baseline 数据抽查。
- production：灰度预算（DISCOVER_DAILY_CAP 保持 50）观察 24h；确认 0 异常错误、0 私网连接（可由日志 reason=blocked 计数佐证）。

## 1.10 rollback

- 代码：`git revert`（或恢复上次发布 tag）重建；probe 改动不涉及 DB，无数据回滚。
- 行为验证：回滚后确认候选链路仍走旧 probe（不可达降权语义一致）。

---

# 2. P0-2：生产 / main / 本地代码基线分裂

## 2.1 Commit 事实表（2026-08-18 只读复核）

| 位置 | HEAD | 说明 |
|---|---|---|
| GitHub main | `c74dfd1` | = 生产基线 b52ec84 + SEO Phase 1 P0（14 文件） |
| GitHub multi-source 分支 | `06f2486` | 在 main 之上 3 commits：`e32bd93`（多源引擎 Phase 0）→ `15de2d9`（Radar 源预过滤）→ `06f2486`（评分透明性修复）；`ahead 3 / behind 0`，**可快进合并** |
| 生产运行代码 | `c74dfd1` + multi-source 文件（≈ `06f2486` 状态） | 据 08-17 记录：生产 git HEAD=c74dfd1，但 .next 构建产物含多源文件；**未实时复核服务器文件哈希** |
| 本地镜像 | `1bdfcb4`（master） | legacy-prod-history 末端；**与 main 无共同祖先**（历史为另一条 root）；无 remote、无 tag |
| 部署脚本 | 无 `deploy.sh` | scripts/ 下仅审计/数据脚本 |

## 2.2 明确回答

1. **哪些代码已经生产运行**：main（c74dfd1 全量）+ multi-source 分支 3 commits 的文件（probe/score/sources/collector v2/candidates v2/domain v2/监控修复/migration）。
2. **哪些代码没有进入 main**：multi-source 的 16 个文件（`prisma/schema.prisma` 增量、migration `20260817_multi_source_candidate_pool`、discovery 层、domain.ts、monitoring.ts、测试）。
3. **哪些代码本地缺失**：本地 master 缺 main 的 SEO P0 14 文件、缺 multi-source 全部、缺生产 tag；且本地 33+ 未跟踪文档。
4. **deploy.sh 是否能复现生产**：**不能**——不存在 deploy.sh；且即使手写部署脚本，从 main 构建也会**缺 multi-source 文件**（probe/评分/种子源），生产行为无法重建。
5. **是否存在“生产运行代码无法从 main 重建”**：**是（CONFIRMED）**。这是本次最需要先解决的问题。

## 2.3 基线统一方案（只给方案，不执行）

**目标态**：main = 生产可重建基线；本地镜像跟踪 main；deploy.sh 可复现；tag 记录发布。

步骤（每步等授权）：
1. **评审**：对 16 个文件做 diff 评审（已做过 read 复核：全部集中在 discovery 层 + domain/monitoring 小改 + schema 增量 + 测试）。
2. **合并**：在 main 上 FF 合并 `multi-source/phase0-cfradar` → main=`06f2486`（`git merge --ff-only`；ahead 3 behind 0，无冲突预期）。保留原分支作为历史。
3. **重建验证**：从新 main `pnpm install && prisma generate && pnpm test`（预期 258 通过）+ `pnpm build`。
4. **staging 验证**：从新 main 部署 staging，跑多源验收清单（probe/评分/seed 预算/退避）。
5. **生产发布**：git bundle 通道部署，`systemctl restart siteintel`；验证 HTTP 200 + 日志 `collector armed` + DB schema 与 migration 一致（应已一致）。
6. **本地镜像同步**：加 remote → fetch → 建 `master` 指向 main（保留未提交文档，不清理）；legacy 历史保留为本地存档分支。
7. **deploy.sh 落库**：版本化部署脚本（fetch/bundle → install → build → smoke → restart → verify），使“main 可重建生产”可操作。
8. **tag**：`production-phase1-baseline-2026-08-18`。
9. **收尾**：更新 SITEINTEL_STATE.md；staging/只读角色删除列入后续授权项（不属于本 P0 批次）。

**风险**：① 服务器 .next 与 06f2486 若有未记录漂移，重建后行为可能微变——以 staging 验收覆盖；② FF 后 main 增加 3 commits，回滚用既有 tag（production-baseline-2026-08-17=b52ec84 / SEO tag=c74dfd1）；③ 本地镜像同步需保留 33+ 未跟踪文件（只建分支，不 reset）。

**rollback**：main `git reset --hard c74dfd1`（保留原 tag）；生产回滚 = 恢复 bundle 备份 + 旧 .next。

---

# 3. P0-3：/api/report 匿名无限流

## 3.1 当前 endpoint 事实（只读）

| 维度 | 现状 |
|---|---|
| Endpoint | `GET /api/report/[domain]`（`src/app/api/report/[domain]/route.ts`） |
| 谁能访问 | **任何人（匿名）**；无 session、无 API Key 判断 |
| Rate limit | **无**（对比：/api/analyze 20/hr/IP；v1 按 key 限流） |
| 返回内容 | `assembleReport(domain, locale)` 完整 IntelligenceReport JSON |
| 是否含 raw data | **是**：`evidence[]` 每项含 `rawData`（provider 原文：DNS 记录、HTTP 头、RDAP 注册数据、TLS 证书等） |
| 前端是否依赖 | **是**：报告页“导出 JSON”按钮（`report-page.tsx:1162` → `ExportJsonButton` → 客户端 fetch 该 URL） |
| 页面级等价暴露 | /website/[domain] 公开页面本身已服务端渲染 EvidenceSection（rawData JSON 展开，60KB 截断）——**rawData 本来就是公开产品数据** |
| v1 付费通道 | `GET /api/v1/report/[domain]`：必须 X-API-Key，按 key 限流 + 月度配额 + `?lang=`，同一 assembleReport |
| 文档 | docs/api 只写 v1 与 stream；未将 /api/report 列为正式端点 |

## 3.2 风险定性

- 增量风险不是“泄密”（公开页面已含 rawData），而是：**无鉴权 + 无限流 + 机器可读 JSON 批量抓取**，可绕开 v1 配额体系，也是未来 API 商业化的直接漏洞。
- 次要风险：与 v1 契约并存导致维护双份行为。

## 3.3 最小兼容方案（推荐 A + 可选 B）

**方案 A（推荐，零前端改动，1 个文件 + 测试）**
1. 复用 `checkRateLimit("report:" + clientIp, 30, 3_600_000)`（与 analyze 同模式）。
2. 保留匿名读取（导出按钮继续工作）；文档注明该端点为“调试/导出”用途。
3. 可选加固：匿名响应裁剪 `evidence[].rawData`（仅保留 provider/dataType/status/collectedAt），v1 返回全量——但**导出 JSON 的价值包含 rawData**，裁剪会削弱导出功能；故默认不裁剪，仅限流。

**方案 B（彻底，需改前端，2-3 个文件）**
1. 导出改为服务端：新增 `GET /api/report/[domain]/export`（或由页面 route handler 生成文件流），ExportJsonButton 指向新端点。
2. `/api/report/[domain]` 改为与 v1 同权（要求 Key）或返回 410。

**推荐**：先做方案 A（限流）保证边界；方案 B 作为后续“API 契约收敛”独立闭环，不在本 P0 批次内。

## 3.4 影响面

- 受影响页面：仅报告页导出按钮（若走方案 B）；方案 A 无页面影响。
- 受影响调用方：无其他 src 内引用（rg 确认仅 report-page.tsx 一处）。
- 不修改范围：v1、assembleReport 内部逻辑、报告页渲染。

## 3.5 测试 / staging / production / rollback

- 测试：路由级测试（限流 30/hr/IP 返回 429；正常 200）。
- staging：匿名 curl 连续请求验证 429。
- production：上线后抽样验证导出按钮正常 + 限流生效。
- rollback：撤销该文件改动（无 DB 变更）。

---

# 4. P0-4：SSE progress endpoint 无鉴权

## 4.1 事实（只读）

| 维度 | 现状 |
|---|---|
| Endpoint | `GET /api/analyze/[id]`（状态 JSON）+ `GET /api/analyze/[id]/stream`（SSE） |
| 创建方式 | ① 匿名 `POST /api/analyze`（20/hr/IP，investigation.userId=null）；② `POST /api/v1/analyze`（API Key）；③ `POST /api/analyze/bulk`（登录，userId=用户） |
| Token / owner 关系 | **无 token**；只有 investigation.userId（可空） |
| 前端如何连接 | `progress-tracker.tsx`：先 `fetch(/api/analyze/{id})` 再 `new EventSource(/api/analyze/{id}/stream)`——EventSource **不能带自定义 Header**，当前无任何凭证 |
| 跨用户读取 | **可能**：investigationId 公开出现在报告页（history.evidence.investigationId）、bulk 结果行、进度 URL；任何知 id 者可读进度/错误/域名 |
| 泄露内容敏感度 | 低-中：进度步骤 + 错误文本（可能含 URL/头信息）+ 域名；不含报告正文 |
| Token 泄露风险 | 当前无 token 可泄；方案引入后需防 URL 残留日志 |
| docs | docs/api 明确写“无需 Key”——需同步更新 |

## 4.2 最小鉴权方案

**规则**：
- `investigation.userId != null`（登录/批量创建）→ 校验 session cookie 与 userId 匹配即放行（**前端零改动**，bulk-form 已带 cookie）。
- `investigation.userId == null`（匿名创建，主路径）→ 需要**流 token**。

**token 设计（推荐：持久化哈希列，不加新表）**：
1. 创建时生成 32-hex 随机 `streamToken`；`Investigation` 新增列 `streamTokenHash`（sha256），仅存哈希（与 API Key 同策略）。
2. `POST /api/analyze` / `POST /api/v1/analyze` 响应增加 `streamToken`（仅返回一次）。
3. `GET /api/analyze/[id]` 与 `GET /api/analyze/[id]/stream` 校验：session 匹配 **或** `?token=`（或 `Authorization: Bearer`）哈希匹配。
4. 前端：`analyze-form.tsx` / `bulk-form.tsx` 在 POST 后把 token 写入 `sessionStorage["si_stream_<id>"]`；`progress-tracker.tsx` 读取并拼 `?token=` 到两个请求。
5. 可选加固：token TTL（如 24h）与一次性使用；不做也足够（哈希存储 + 低敏感度）。

**零迁移替代方案（备选）**：token 存内存 Map（progress.ts），无 schema 变更；缺点：重启即失效（进度页重连失败，需重新分析）。**推荐主方案（持久列）**，理由：可审计、重启不失效、与 API Key 哈希策略一致；新增一列不属于“新增未来暂时不用的表”。

## 4.3 改动清单（实施时）

代码：`api/analyze/route.ts`、`api/v1/analyze/route.ts`、`api/analyze/bulk/route.ts`（发 token）；`api/analyze/[id]/route.ts` + `stream/route.ts`（校验）；`prisma/schema.prisma`（Investigation.streamTokenHash）+ migration；`analyze-form.tsx`、`bulk-form.tsx`、`progress-tracker.tsx`；`docs/api/page.tsx`（更新说明）。

不修改范围：pipeline、监控调度、报告页面、v1/report。

## 4.4 测试 / staging / production / rollback

- 测试：匿名无 token → 401/403；错误 token → 401；正确 token → 200/SSE；登录用户 session 可读自有 investigation；他人 session 不可读。
- staging：匿名分析 → 页面进度正常（token 流程）；登出后携带 token 仍可读；bulk 流程正常。
- production：抽样匿名 + 登录分析各若干次；确认 0 误伤（进度页、bulk、v1 streamUrl）。
- rollback：撤销代码 + 保留列（无害）或一并回滚 migration；旧前端无 token 会 401——**回滚必须代码+前端同批**。

---

# 5. 推荐执行顺序（单一发布批次）

```
批 1（本 P0 批次）：
   Step 1  P0-2 基线统一：multi-source FF 合入 main（评审 → 测试 258 → staging → 生产 → 本地同步 → tag）
   Step 2  P0-1 probe 改造（在统一 main 上）：safe-fetch allowedPorts + probe 重写 + blocked 分类 + 测试
   Step 3  P0-3 /api/report 限流（方案 A）
   Step 4  P0-4 SSE token（含 migration + 前端联动 + docs）
   Step 5  全量回归：pnpm test + build + staging 验收 + 生产验收 + SITEINTEL_STATE.md 更新
```

依赖说明：Step 2 依赖 Step 1（probe 只在分支存在）；Step 3/4 相互独立，可并行开发、同批验证。

---

# 6. 每项风险与回滚汇总

| 项 | 主要风险 | 回滚 |
|---|---|---|
| P0-2 | 服务器文件与分支漂移导致重建行为微变 | git reset + bundle 恢复 |
| P0-1 | blocked 误分类把可达站点降权 | 代码 revert（无 DB 变更）；用日志核对 blocked 计数 |
| P0-3 | 限流误伤导出按钮 | 提高阈值或 revert |
| P0-4 | 前端 token 流程遗漏导致进度页 401 | 代码+前端+列同批回滚 |

---

# 7. 待授权清单（下一步需要你确认的点）

1. P0-2 是否授权执行“multi-source 合并 + 本地同步 + deploy.sh 落库”？（涉及 Git 写入，权限在你）
2. P0-1 是否按“safe-fetch allowedPorts 默认 80/443 + probe 重写 + blocked 分类”执行？
3. P0-3 选方案 A（限流，零前端改动）还是 A+B（再收掉导出 JSON 的 rawData）？
4. P0-4 选持久列（推荐）还是内存 token（零迁移）？
5. 本批次是否一次授权全部 4 项，还是逐项授权、逐项验收？

> 本轮仅为分析与方案稿，未执行任何修改。等你逐项授权后，从 Step 1（P0-2 基线统一）开始，每项按“分析 → 报告 → 授权 → 实施 → 测试 → staging → 生产 → 记录”小闭环推进。
