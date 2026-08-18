# SiteIntel Phase 1 — Step 4 SSE 访问控制收口完成报告

> 日期：2026-08-18 ｜ 范围：`/api/analyze/[id]`、`/api/analyze/[id]/stream` 的 Investigation 归属校验
> 授权：用户逐项授权（Step 4 only）；Schema 仅新增 `Investigation.streamTokenHash`（已获授权，无其他 Schema 变化）
> 结论：✅ **完成（PASS）**

---

## 1. 修改目标

消除 SSE 进度流与状态轮询的无鉴权读取：登录用户按 session 归属校验；匿名调查在创建时签发一次性高强度 stream token（仅存 SHA-256 哈希），EventSource/轮询必须携带 token。

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
- nginx：`/api/analyze/` location 增加 `access_log off`（配置备份 `siteintel.cc.conf.bak-step4-20260818`，`nginx -t` + reload 已生效）——EventSource 的 `?token=` 永不进入服务器访问日志。

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
