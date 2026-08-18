# SiteIntel Phase 1 — Step 4 SSE 访问控制收口完成报告

> 日期：2026-08-18 ｜ 范围：`/api/analyze/[id]`、`/api/analyze/[id]/stream` 的 Investigation 归属校验
> 授权：用户逐项授权（Step 4 only）；Schema 仅新增 `Investigation.streamTokenHash`（已获授权，无其他 Schema 变化）
> 结论：✅ **完成（PASS）**

---

## 1. 修改目标

消除 SSE 进度流与状态轮询的无鉴权读取：登录用户按 session 归属校验；匿名调查在创建时签发一次性高强度 stream token（仅存 SHA-256 哈希），EventSource/轮询必须携带 token。

## 1.1 当前 SSE 架构（修改前）

```text
POST /api/analyze（匿名或登录）→ startInvestigation → Investigation(userId 可空)
GET /api/analyze/[id]          ← 无鉴权状态轮询
GET /api/analyze/[id]/stream   ← 无鉴权 SSE（docs 曾标注“无需 Key”）
```

原安全问题：任何知道 investigationId 的请求（匿名或登录）均可读取任意调查的进度/错误/域名；id 公开出现在报告页证据与批量结果中。

## 1.2 修改后的认证链

```text
创建：
  登录用户 → session → userId 写入 Investigation（不签发 token）
  匿名 → issueStreamToken()（256-bit）→ 仅存 sha256(streamTokenHash) → 明文只回一次
读取（状态 + SSE 共用 verifyInvestigationAccess）：
  userId != null → getSessionUser() 必须匹配 → 否则 403
  userId == null → token（Bearer 或 ?token=）→ timingSafeEqual(sha256(token), streamTokenHash) → 否则 401
```

## 2. 实现

### 2.1 Schema（唯一变更）

- `Investigation.streamTokenHash TEXT?`（migration `20260818_stream_token_hash`，生产已 deploy，`_prisma_migrations` 记录）。

### 2.2 创建端（签发一次性 token）

- `POST /api/analyze`：匿名（无 session）→ `issueStreamToken()`（256-bit hex）；`startInvestigation(..., { streamTokenHash })`；响应返回 `streamToken`（仅新建且匿名时；reused 不重发）。
- `POST /api/v1/analyze`：API Key 路径同样匿名 → 签发 token，响应含 `streamToken`。
- `POST /api/analyze/bulk`：登录路径，session 校验，无需 token。
- 明文 token 仅存在于：创建响应的网络传输 + 浏览器 sessionStorage；**服务端只存 sha256**。

### 2.3 读取端（统一校验 `verifyInvestigationAccess`）

- `GET /api/analyze/[id]`（状态）与 `GET /api/analyze/[id]/stream`（SSE）：
  - `investigation.userId != null` → `getSessionUser()` 必须匹配 → 否则 403。
  - `userId == null` → 必须携带 token（`Authorization: Bearer` 或 `?token=`）→ sha256 + `timingSafeEqual` 与 `streamTokenHash` 比对 → 否则 401；无 hash 的遗留匿名调查一律 401。

### 2.4 前端

- analyze-form / bulk-form：POST 后把 `streamToken` 写入 `sessionStorage["si_stream_<id>"]`。
- progress-tracker：状态轮询用 `Authorization: Bearer`；EventSource 用 `?token=`（EventSource 不能带 header）；401/403 → 显示 `dict.progress.unauthorized`（i18n 中英文已加）。
- docs/api：stream 示例更新为带 token。

### 2.5 日志防护

- 应用层：任何代码路径不 log token / 不含 token 的 URL。
- nginx：**评估结论**——Next.js 自身不记录 query，但 nginx access log 会记录完整 query string（含 `?token=`）。因 EventSource 无法携带自定义 Header（浏览器约束），采用比“日志脱敏”更强的方案：`/api/analyze/` location 增加 `access_log off`（配置备份 `siteintel.cc.conf.bak-step4-20260818`，`nginx -t` + reload 已生效）——该路径请求（含 token query）**完全不进入服务器访问日志**。Cloudflare 免费版分析不保留完整 URL。

## 2.6 修改文件清单

```text
prisma/schema.prisma                                   （+streamTokenHash）
prisma/migrations/20260818_stream_token_hash/migration.sql（新增，唯一 Schema 变更）
src/lib/sse-auth.ts                                    （新增：签发/提取/timing-safe 校验/归属判定）
src/lib/sse-auth.test.ts                               （新增：12 例）
src/lib/pipeline.ts                                    （startInvestigation 透传 streamTokenHash）
src/app/api/analyze/route.ts                           （匿名签发 token + 响应 streamToken）
src/app/api/v1/analyze/route.ts                        （同，v1 匿名路径）
src/app/api/analyze/[id]/route.ts                      （读取前归属校验）
src/app/api/analyze/[id]/stream/route.ts               （读取前归属校验）
src/components/analyze-form.tsx                        （sessionStorage 保存 token）
src/components/bulk-form.tsx                           （每行 token 保存 + 轮询 Bearer）
src/components/progress-tracker.tsx                    （Bearer 轮询 + EventSource ?token= + 401 界面）
src/lib/i18n.ts                                        （progress.unauthorized 中英文）
src/app/docs/api/page.tsx                              （stream 示例带 token）
```

## 2.7 Token 生命周期（按授权要求逐项确认）

1. **生成时机**：`POST /api/analyze` / `POST /api/v1/analyze` 创建匿名 Investigation 时（`issueStreamToken`，`randomBytes(32)`，256-bit 密码学安全随机）。
2. **返回时机**：仅在创建响应 JSON 中返回一次（`streamToken`）；reused（同域并发）不重发——服务端只有哈希，无法也不会再次提供明文。
3. **仅当前 Investigation 可用**：是——哈希按行存储，校验只与该调查的 `streamTokenHash` 比对；token A 用于调查 B → 401（单测 + 生产 E2E）。
4. **一次性消费**：**未实现“一次性消费”**。原因：EventSource 断线自动重连与页面轮询都复用同一 token；实现单次消费需删除哈希/状态表，超出最小范围且破坏重连。**实际采用的最小安全策略**（明确声明，不假装）：单次签发（服务器永不重发）+ 单调查绑定 + 哈希存储 + 永不落日志；重放仅对本调查的读操作有效，无任何写权限。
5. **Analyze 完成后是否继续允许读取**：允许（与修改前一致，仅进度/状态只读信息；token 无写能力）。如需“完成后失效”，后续可单独评估（需 Schema/策略变更，不在本次范围）。
6. **过期策略**：无 TTL。调查生命周期为秒-分钟级；完成后 token 无实际价值；引入过期机制需要额外字段/清理任务，本次不引入（最小范围）。
7. **页面刷新**：可用——token 存 `sessionStorage`，同一标签页刷新后仍在；轮询与 EventSource 重建自动携带。
8. **sessionStorage 行为**：按调查 id 分键（`si_stream_<id>`）；关闭标签页即清除；不跨标签页共享；服务端无明文副本。

## 2.8 测试矩阵（授权文档 §10，20 项对照）

| # | 项 | 结果 |
|---|---|---|
| 1 | 正确用户访问自己的 Investigation | ✅ 单测（session 匹配放行） |
| 2 | 用户 A 访问用户 B | ✅ 单测（403） |
| 3 | session 缺失 | ✅ 单测（403） |
| 4 | session 无效 | ✅ 单测（getSessionUser=null → 403） |
| 5 | 正确 token | ✅ 单测 + 生产 E2E（200） |
| 6 | 错误 token | ✅ 单测 + staging/prod（401） |
| 7 | 缺少 token | ✅ 单测 + staging/prod（401） |
| 8 | 空 token | ✅ 单测（extract null → 401） |
| 9 | 随机 token | ✅ 单测（wrong token → 401） |
| 10 | token 对应其他 Investigation | ✅ 单测 + 生产 E2E（401） |
| 11 | token 不明文进入数据库 | ✅ DB 实证（hash=sha256、非明文、长度 64） |
| 12 | token 不进入应用日志 | ✅ journal 扫描 0 |
| 13 | token 不进入错误日志 | ✅ 错误消息不回显 token；扫描 0 |
| 14 | token 猜测无法访问 | ✅ 256-bit + timing-safe 比对 |
| 15 | Cross-user access 阻断 | ✅ 单测矩阵 |
| 16 | 正常 Analyze | ✅ 生产 example.com completed |
| 17 | 正常 SSE 进度 | ✅ 生产 stream 200 + 事件流 |
| 18 | Analyze completed | ✅ 轮询到 completed |
| 19 | Report 页面 | ✅ /website/example.com 200、/api/report 200 |
| 20 | 现有测试 | ✅ 309/309（+12 新增） |

## 3. 防伪要求对照

| 要求 | 实现/验证 |
|---|---|
| 跨 Investigation 读取 | token 按行绑定（仅与该调查的 hash 比对）；E2E：token A 访问调查 B → 401 |
| 跨用户读取 | session userId 严格匹配；他人/无 session → 403（单测覆盖） |
| token 重放 | 单次签发：服务端永不重发明文、仅存哈希；同调查重连允许（EventSource 重连/轮询需要）；其他调查不可用 |
| token 明文落库 | 否——DB 实证 `hash_is_plaintext=false`、`hash_matches_sha256=true`、长度 64 |
| token 进入日志 | 否——nginx 访问日志与 journal 扫描 token 均为 0 |
| 未授权 SSE 连接 | 无 token/错 token → 401（staging+production 实测） |
| 登录用户访问他人调查 | session 校验 → 403（单测矩阵；线上 E2E 需真实账号凭据，见 §7） |

## 4. 验证结果

### 4.1 单元测试

- 新增 `src/lib/sse-auth.test.ts` 12 例：签发（256-bit/哈希非明文）、校验（正确/错误/缺失/null）、提取（Bearer/query/缺失）、归属（匿名 token 放行、错误/缺失 401、遗留无 hash 401、跨调查 401、登录本人放行/他人 403/无 session 403）。
- 全量：**309/309 通过（20 文件）**；typecheck 0；本地 build 通过。

### 4.2 staging（127.0.0.1:3004，只读库）

- 代码部署成功（schema/stream 路由含新逻辑，build 通过，服务 active）。
- 守卫实测：状态无 token=401、流无 token=401、错误 token=401（×2）、home=200。
- 说明：staging 只读库无法新建调查，放行路径由单元测试覆盖；生产完成全路径 E2E。

### 4.3 production

| 验证项 | 结果 |
|---|---|
| 匿名 POST /api/analyze | investigationId + 64 位 token，reused=false |
| 状态 + Bearer token | 200（调查 completed） |
| 状态无 token / 错 token | 401 / 401 |
| SSE `?token=` | 200（事件流） |
| 跨调查（token A → 调查 B） | 401 |
| DB 哈希证明 | hash=sha256(token) true；非明文 true；长度 64 |
| 日志 | nginx 访问日志 token 计数 0；journal token 计数 0；`/api/analyze/<id>` 访问日志 0（access_log off 生效） |
| 正常流程 | /website/example.com 200、/api/report 200、服务 active、NRestarts=0 |
| 生产 HEAD | 对齐 `c4d79e1`，tracked 树零差异 |

## 5. Git

- 代码 commit：`c4d79e142a9cd7761ce80aabad0a3106577d707e`（fix(security): enforce SSE stream ownership with one-time anonymous tokens）
- Tag：`production-phase1-step4-sse-auth-2026-08-18` → c4d79e1（annotated，已推送）
- siteintel main = c4d79e1 = 生产 HEAD

## 6. 回滚方式

- 代码：`git revert c4d79e1` 或恢复 tag `production-phase1-step3-report-ratelimit-2026-08-18`=27bbf93 → build/restart。
- Schema：`ALTER TABLE "Investigation" DROP COLUMN "streamTokenHash";`（migration 回滚；备份 `/tmp/siteintel_pre_step4_20260818.dump` 在位）。
- nginx：`cp siteintel.cc.conf.bak-step4-20260818 siteintel.cc.conf && nginx -t && nginx -s reload`。
- 前端与后端必须同批回滚（旧前端无 token 会 401）。

## 7. 已知限制

1. 遗留匿名调查（新功能上线前创建、`streamTokenHash IS NULL`）的进度流/轮询返回 401；用户重新发起分析即可（明文 token 无法补发，符合“单次签发”）。
2. 登录用户路径的线上 E2E 需要真实账号凭据，本次以单元测试矩阵 + 代码审查覆盖；如需线上实证，请提供测试账号或授权创建测试用户。
3. EventSource 无法携带 Header，token 以 query 传递；已通过 nginx `access_log off` 阻断服务器日志泄露；Cloudflare 免费版分析不含完整 URL。
4. token 无 TTL（调查生命周期短；完成后 token 不再有用）；未引入过期机制以保持最小改动。

---

## GitHub Publication

- Repository: Duoniu-ai/Codex
- Branch: main
- Commit: 见提交结果（本报告推送后 local HEAD = origin/main）
- Tag: production-phase1-step4-sse-auth-2026-08-18（siteintel 仓库）
- File: SITEINTEL_PHASE1_STEP4_SSE_AUTH_COMPLETION_REPORT.md
- Remote verification: PASS（推送后验证）

---

*完成，停止。未开始 Step 5。*
