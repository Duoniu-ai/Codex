# PROJECT STATE — Claude Code → Codex 历史上下文迁移

> 生成时间：2026-08-18（Asia/Singapore）
> 生成方式：只读迁移。读取 Claude Code memory / 状态文件 / Git 只读信息后整理，**未修改任何源代码、数据库、部署或 Git 状态**。
> 主要依据：`C:\Users\deepo\.claude\projects\C--Users-deepo\memory\*.md`（最后更新 2026-08-17）、`C:\www\wwwroot\websites\PROJECT_STATUS.md`、`PROJECT_BOUNDARY.md`、项目 Git 只读状态。
> 详细文件索引见 [CODEX_CLAUDE_HISTORY.md](CODEX_CLAUDE_HISTORY.md)。

---

## 1. 项目列表

1. **SiteIntel.cc** — 网站数据情报平台（生产）
2. **网址发现平台**（SiteIntel 导航站 / 网站发现平台，独立产品，非 SiteIntel 子项目）
3. **IPcesu.com** — IP 测速网
4. **IPcece.com** — IP 工具站
5. **duoniu.cc** — 多牛网（IP 纯净度/线路/历史/ASN/BGP/风控查询）
6. **IPIP0.net** — IP 查询识别站

---

## 2. 项目路径

| 项目 | 生产/服务器路径（154.204.176.66） | 本地路径 |
|---|---|---|
| SiteIntel | `/www/wwwroot/siteintel.cc`（systemd `siteintel` :3003） | `C:\Users\deepo\siteintel`（镜像） |
| 网址发现平台 | `/www/wwwroot/websites`（systemd `navigation` :3010，计划） | `C:\www\wwwroot\websites` |
| IPcesu | `/www/wwwroot/ipcesu.com`（:3002）+ test.ipcesu.com（:3001） | —（无本地仓库，SSH 开发） |
| IPcece | `/www/wwwroot/ipcece.com`（纯静态 + CF Worker） | `D:\phpstudy_pro\WWW\IPcece.com`（源码+git） |
| duoniu | `/www/wwwroot/duoniu.cc/duoniu-new-vue`（systemd `duoniu-preview` :3901） | — |
| IPIP0 | 线上 Laravel + Cloudflare（代码不在本地） | `D:\phpstudy_pro\WWW\ipip0`（本地复刻） |

---

## 3. 最近工作时间（2026-08-14 → 2026-08-17）

- **2026-08-17**：SiteIntel SEO Phase 1 P0 全链路（实施→staging→生产部署→Git 发布）、多源自动发现 Phase 0 + CF Radar 上线验收、monitor 死循环修复、Seed 灰度测试 + 每日批次 10→50、Git 历史统一 + 生产 HEAD 对齐；网址发现平台 D3（Migration+Seed）COMPLETE；IPIP0 GitHub 基线建立。
- **2026-08-16**：SiteIntel 2.0 Phase 3A/3B/3B1/3B2/3B3/3C 上线、白盒审计 P0–P2 修复与数据纠错收口；duoniu Step 0–8 全部上线；IPIP0 阶段一~五（数据模型/实体页/采集可靠性/首页二轮 UX）。
- **2026-08-15**：SiteIntel V2 Phase 0/0.5/1 上线与数据治理；IPcece P0–P5 + Phase 1–5 + 全站 153 页体检修复；duoniu Phase 0–3 上线。

---

## 4. 已完成工作（✅ 已收口，默认不重复执行）

### SiteIntel
- 上线（08-14）+ 双语 + 监控/告警/账号/批量/开放 API/AI 解释/Search Intelligence/SEO Engine/数据增长引擎。
- V2 Phase 0 审计、Phase 0.5 九项修复、Phase 1 四表事实引擎上线。
- 白盒审计 P0–P2 全部修复并部署（safe-fetch 统一网络层、NS/MX 稳定排序、www 剥除、OrganizationAlias、AS0 哨兵等）。
- safe-fetch path 回归事故数据纠错（15 条关系/1 事件/2 洞察/6 目标重分析，0 残留）。
- 修复后全站回归审计 PASS（0 回归，121 测试）；P3 收尾（3 个孤立 domain 删除）；Domain 创建守卫上线；Discovery 盘点 + Priority Aging 上线。
- www 92 实体合并已执行；provider_changed 清理已执行；Organization 治理 **Phase A** 已实施。
- 2.0：UX 基线审计 → IA 设计 → Phase 3A 首页 / 3B / 3B.1 / 3B.2 / 3B.3 / 3C 全部上线。
- SEO Phase 1 P0：实施→验收 PASS WITH MODIFICATIONS→staging 验证 PASS→生产部署 PASS→Git FF merge + tag 发布。
- 自动繁殖停摆只读诊断（结论：设计性饱和，非故障）；多源发现白盒审计；Phase 0 + CF Radar 上线验收 PASS；Seed 灰度测试 PASS；SEED_DAILY_BATCH 10→50 生效。
- Git 历史统一（legacy-prod-history 66 commits）+ 生产 HEAD 对齐 c74dfd1。

### 网址发现平台
- 产品 V2.1.2 / Schema V1.0 / API 契约 V1.0 / 信息架构 V1.0 / P0 任务拆解 V1.0 五基线冻结。
- GAP-01 裁定关闭（首页两个只读直查）。
- D1 初始化、D2 冻结 Schema 实现（24 表/221 字段逐字对照 PASS）、D3 Migration+数据库验收+Seed（30 任务×150 网站 + 分类/标签/别名/settings/curated_picks，幂等）。

### IPcesu
- 新版上线（08-13）、SEO 迁移、监控真实化、品牌名统一；两次事故修复（canonical 指向测试域名、proxy_cache 缓存旧 HTML）。

### IPcece
- P0/P1/P1.5 + Phase 1–5（网络健康检查、报告分享、融合端点、JSON-LD）全部上线；153 页 hydration 修复 0 失败；Worker 安全加固 + 5 端点补齐。

### duoniu
- Phase 0（git 基线/备份/staging/deploy.sh）、Phase 1–3、Step 0–8（verdict 规则层、purity 完整化、history 时间线、DNS 5 解析器对比、Trace 双视图、复制导出）全部上线。

### IPIP0
- 本地复刻两项功能完善、二次白盒审计、阶段一~五（SEO/API 文档/风险证据链/多语言/数据模型/实体页/采集可靠性/首页二轮 UX/结果卡片修复）完成。

---

## 5. 当前进行中

- **SiteIntel**：多源自动发现引擎**已暂停**，等用户确认后进第二阶段（Majestic）；2.0 Phase 3D（功能页）已拆好待批准。
- **网址发现平台**：D3 COMPLETE，进入 D4∥D5（同步实现）但**唯一前置未决 = SiteIntel navigation-sync 专用 API Key**。
- **IPIP0**：Gate D 跨日快照积累中（每日 re-probe，需 2–3 自然日；本机无 Scheduler，任务已 CLI 化）。

---

## 6. 未完成任务 / 待用户决策

### SiteIntel
- 多源发现：**第二阶段 Majestic 暂停等确认**（回滚基线已备：DB /tmp/siteintel_pre_phase0.sql + git checkout master + .env.bak.phase0）。
- 2.0：Phase 3D（功能页）待批准。
- SEO：Phase 2 未执行；entity 页 robots/sitemap 同源化（P3 记录项）；running/failed+历史样本待生产自然出现后补验。
- Git/部署治理：deploy.sh 待批；阶段 II 卫生收口待决策（tsconfig.json 归一化、49 排除文件保持 ?? 或补 ignore、12 文档 + DISCOVERY 入库）；README 更新/License 待决策；`baidu_verify` 大小写双文件遗留待决策。
- 清理类（待批）：删除 staging（siteintel-staging :3004）、删除只读角色 `siteintel_staging_ro`。
- 生产 f1 worktree（`/root/siteintel-f1`，f1-candidate-filter @ 1bdfcb4，4 项未提交）：**统一方案绝不触碰**，保持原样。
- 运维遗留（任务书外 P3）：全仓 12 个历史 lint 错误、法律页/opt-out、SSE 鉴权、图谱超节点。

### 网址发现平台
- D4∥D5 依赖 SiteIntel navigation-sync Key；D6∥D7（API）→ D8∥D9（前端）→ D10（测试/验收/部署）。
- 正式域名（D10.5）未决。

### IPIP0
- **Gate D 达标前锁定 P2（Org/Prefix 实体）与任何新页面**。
- 阶段五 P0 两项：指纹/端口状态文案区分、移动端 nav（<768px 无汉堡菜单）。
- P1：Native/Anycast 宣称落地、About 类完整内容、ASN 页 org/network 闭环（等 Gate D→P2）。
- P2 清单：PHP_BIN 硬编码 + start /B（Linux 必改）、延迟结果缓存/去重、_expires 泄漏、timeout 状态、ip-api.com 明文 HTTP。

### IPcesu
- AI 工具密钥（ZHIPU_API_KEY 等）、IPv6 工具验证、支付真实渠道。

### IPcece / duoniu
- IPcece 运维性待办：KV 缓存分层、CF WAF 规则（用户侧面板步骤已文档化）、混淆策略定期评估、ipapi.is 配额监控。
- duoniu：无明确待办（任务书 Step 0–8 已闭环）；route 页无逐跳坐标未做地图（差异已向用户说明）。

---

## 7. 已知问题

- SiteIntel：历史 lint 12 项（与近期改动无关）；本地 lint 7 个 error 属 eslint 环境漂移（非 blocker，无 CI）；`tsconfig.json` 工作树差异=行尾归一化差异（零修改产生）。
- 网址发现平台：无已知技术遗留；唯一阻塞为 navigation-sync Key。
- duoniu：生产/部署通道受 Claude 分类器影响（deploy.sh 需人工分步），迁移到 Codex 后按规范执行。
- IPIP0：collector 无 Scheduler（本机），自动任务必须按文档化方式调度。

---

## 8. 数据库状态

| 项目 | 数据库 | 状态（截至 2026-08-17 记录） |
|---|---|---|
| SiteIntel | PostgreSQL 14，库 `siteintel` | 37MB；无 _prisma_migrations 期初已核查，后走 migration 工作流；真实数据（08-17 诊断）：289 Target / 537 调查 / 3230 实体 / 5888 关系 / 3147 快照 / 65 事件 / 58 洞察；候选池 926+（868 pending）；seed 50/天 active；只读角色 `siteintel_staging_ro` 待删 |
| 网址发现平台 | PostgreSQL，独立库 `nav_disc`（owner `nav_disc_user`） | Migration `20260817_init` 已应用（24 表/221 字段/30 CHECK）；seed 已执行且幂等 |
| IPIP0 | 本地 MySQL 库 `ipip0`（8 业务表 + asn_entity/risk_tag/risk_snapshot） | 113 IP / 30 ASN / 61 evidence / 163 snapshot（08-16 终态）；线上 Laravel 库未接 |
| duoniu | 复用 PHP 主库（ipquery 20 表） | 无新表；webrtc_peers 落库正常 |

红线：网址发现平台**禁止直连 SiteIntel PostgreSQL**，同步只走冻结消费契约；任何生产库修改须用户明确授权。

---

## 9. 部署状态（截至 2026-08-17 记录，实时线上状态未再核验）

| 项目 | 状态 |
|---|---|
| SiteIntel | 生产运行 **c74dfd1**（systemd active :3003，0 fatal）；staging `siteintel-staging` :3004 仍 active（待删）；多源 Phase 0 已上线（e32bd93）；seed 50/天已生效 |
| 网址发现平台 | **尚未部署业务代码**（D1–D3 完成，D4 起阻塞）；部署目标 systemd `navigation` :3010 |
| IPcesu | 生产 :3002 active；test :3001 保留（vhost 曾丢失，需注意） |
| IPcece | 纯静态（nginx）+ CF Worker（wrangler 已登录本机） |
| duoniu | `duoniu-preview` :3901 active（Step 0–8 已上线） |
| IPIP0 | 线上 Laravel 未动；本地复刻仅开发/演示 |

---

## 10. Git 状态（只读确认，2026-08-18）

- **SiteIntel GitHub（PRIVATE, Duoniu-ai/siteintel）**：`main` = c74dfd1；`legacy-prod-history` = 1bdfcb4（66 commits）；tags：`production-baseline-2026-08-17`(b52ec84)、`production-seo-phase1-p0-2026-08-17`(c74dfd1)、`production-history-archive-20260817`(1bdfcb4)；服务器 tags 另有 `production-legacy-1bdfcb4`、`pre-deploy-seo-p0-20260817`、`production-pre-head-alignment-20260817`。
- **SiteIntel 本地镜像 `C:\Users\deepo\siteintel`**：master @ 1bdfcb4；工作树含预期未提交项（D baidu 大写、?? .well-known/、?? 12+ 报告 md、M DISCOVERY 等卫生/排除文件）——与记录一致，属预期状态，**不自行清理**。
- **网址发现平台本地 `C:\www\wwwroot\websites`**：master @ `de6e954`（chore: establish project and frozen schema baseline）；D3 工作未提交：`M PROJECT_STATUS.md`、`M prisma/schema.prisma`、`?? prisma/migrations/`、`?? docs/audit/D3-*.md`×3。
- **IPIP0**：GitHub PRIVATE `Duoniu-ai/ipip0` @ f8d592d（86 文件）。
- **duoniu**：服务器仓库（tags phase-x-complete 系列）。
- 禁止自主 commit / push / 部署；重要修改前先 `git status / branch / log -1`。

---

## 11. 重要报告与状态文件位置

- Claude Code memory（核心历史）：`C:\Users\deepo\.claude\projects\C--Users-deepo\memory\`（siteintel-project.md、siteintel-nav-project.md、ipcesu/ipcece/duoniu/ipip0-project.md、MEMORY.md）。
- **SiteIntel 深度状态**：[SITEINTEL_STATE.md](SITEINTEL_STATE.md)（修复/Git/数据库/www与Organization合并/TODO 状态矩阵）。
- SiteIntel 报告：`C:\Users\deepo\siteintel\*.md`（约 40 份，完整清单见 CODEX_CLAUDE_HISTORY.md）。
- 网址发现平台：`C:\www\wwwroot\websites\PROJECT_STATUS.md`、`PROJECT_BOUNDARY.md`、`docs/`（product/architecture/audit/development/api 五类，冻结版+验收记录）。
- 规划/任务书等输入文档：`C:\Users\deepo\Downloads\`（SiteIntel*、IPCECE* 系列）。

---

## 12. 禁止重复执行的任务（已收口）

- SiteIntel：P0–P2 白盒审计修复、safe-fetch 数据纠错、P3 收尾、Domain 创建守卫、Discovery+Aging、www 92 实体合并、provider_changed 清理、Organization Phase A、2.0 Phase 3A–3C、SEO Phase 1 P0 全链路、多源 Phase 0+CF Radar、monitor 死循环修复、seed 灰度+50/天、Git 历史统一、生产 HEAD 对齐。
- 网址发现平台：D1、D2、D3（含 seed）。
- IPcece：P0–P5 + Phase 1–5 + 全站体检修复。
- duoniu：Phase 0–3 + Step 0–8。
- IPIP0：两项功能完善、阶段一~五、各 P0/P1 修复。

---

## 13. 待确认 / 不得自动执行

- SiteIntel **Organization 历史合并冻结在 Phase C**（此前保持 dry-run）——迁移上下文**不得因此自动执行**。
- SiteIntel 多源 Phase 2（Majestic）暂停等确认；SEO Phase 2 未启动；staging 与只读角色删除待批；阶段 II 卫生收口待决策。
- 网址发现平台 D4∥D5：等 navigation-sync Key，到货前不开发同步实现。
- IPIP0：Gate D 达标前不进入 P2/P3。
- 所有生产数据库操作、部署、Git commit/push 一律先经用户确认。

---

## 14. 信息无法确认（UNKNOWN，未猜测）

- SiteIntel / 各站点**实时线上状态**（服务、版本、HTTP）本次未实际访问，仅按 2026-08-17 记录。
- 生产服务器（154.204.176.66）当前 Git 工作树细目、`/www/wwwroot/siteintel.cc` 实时内容：本地无法直读（未建立 SSH 访问，本次也未授权远程操作）。
- IPIP0 线上 Laravel 代码与数据状态。
- duoniu 生产当前运行构建是否含最后一批修复（按记录 Step 0–8 已上线）。
- SiteIntel 数据库实时计数（候选池/seed 当日记账）需在生产只读查询确认。

---

## 15. Claude Code → Codex 衔接说明

- Codex 已继承上述历史；后续任务**先读本文件与对应 memory/报告，再定位代码**，不重新扫描、不重复执行已收口任务。
- 所有项目遵守 `AGENTS.md`（低 Token 开发规范）：分析→方案→确认→最小修改→定向验证→报告。
- 与 Claude Code 的差异：Claude 会话分类器曾拦截部署类命令（duoniu deploy.sh）；Codex 按规范同样默认不自动部署，需用户逐次授权。
- 下一步候选（等用户指令）：SiteIntel 多源 Phase 2 / SEO Phase 2 / 2.0 Phase 3D；网址发现平台 D4∥D5（Key 到货后）；IPIP0 Gate D 推进。
