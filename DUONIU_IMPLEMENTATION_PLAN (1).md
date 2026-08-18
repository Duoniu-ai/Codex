# DUONIU.cc 全站升级实施计划

> 生成时间：2026-08-15
> 依据：`CURRENT_ARCHITECTURE.md`（架构审计）+ `DUONIU_UPGRADE_RULES.md`（九章规则）
> 性质：**实施方案**——不修改代码、不删除功能、不直接重构；经确认后按 Phase 执行
> 目标：将「网络工具集合站」升级为「DuoNiu Network Intelligence Platform」

---

# 1. 当前问题总结

## 1.1 前端问题

| # | 问题 | 证据 |
|---|---|---|
| F1 | 首页 1229 行堆叠式长页，一次性渲染几十字段，无七 Tab 结构 | index.vue（规则八违反） |
| F2 | 5 个页面浏览器直连 Provider API，前端写死 Provider 返回结构 | index/env/purity/history 直连 ipquery.duoniu.cc；useIpLine 直连 ipf.duoniu.cc（规则六违反） |
| F3 | 查询逻辑三处重复（queryIpLine 在 index/route/asn 重复），无 features 层 | 规则九违反 |
| F4 | SimpleTool.vue 555 行六合一组件，组件内直接 $fetch | 规则九「组件直接请求API」违反 |
| F5 | 无 Universal Search：搜索框只认 IP，输入 AS13335/域名/CIDR 无类型识别 | 规则四违反 |
| F6 | /cidr/[q] 动态路由缺失，产生 Vue Router No match 警告 | 线上日志 |
| F7 | 19 个 .bak 文件混在 app/ 源码目录 | 扫描统计 |

## 1.2 后端问题

| # | 问题 | 证据 |
|---|---|---|
| B1 | admin-api 反代 127.0.0.1:3001 串线到 ipcesu 测试站，duoniu 后台无独立后端 | /proc/271126/cwd = /www/wwwroot/ipcesu.com |
| B2 | DB_PASS 未配置 → webrtc_peers 落库失败，peer 采集数据静默丢失 | nohup.out 6 条 `[peer] store failed` |
| B3 | 数据服务在旧树 /www/wwwroot/duoniu/（PHP :8899 + python :8010/:8898），与 Nuxt 双树割裂 | 扫描 |
| B4 | global.duoniu.cc 在 Nuxt 代码中零引用，职责悬空 | grep 无结果 |
| B5 | CF Worker duoniu-api 的 KV/D1 为模板占位配置，真实部署状态未知 | wrangler.toml YOUR_* |
| B6 | 2 个 2026-08-13 挂死的 nuxt build 孤儿进程未清理 | PID 55535/56476 |

## 1.3 API 问题

| # | 问题 | 证据 |
|---|---|---|
| A1 | 12 条 server/api 散路由，无统一错误信封，错误透出英文 `fetch failed`/`HTTP 4xx` | 各路由源码（规则七违反） |
| A2 | 容灾缺位：仅 dns.ts 双源，其余单源无 fallback | 规则七违反 |
| A3 | /api/ping 接受任意 http(s) URL 服务端探测，存在 SSRF 风险 | ping.ts 无目标校验 |
| A4 | 无 Provider Manager 抽象：上游直写在路由里，切换/降级/限流都无抓手 | 规则五/六违反 |
| A5 | 无 API 鉴权/限流（peer.post 除外无任何保护） | 扫描 |

## 1.4 数据架构问题

| # | 问题 | 证据 |
|---|---|---|
| D1 | 无实体模型：IP/ASN/网段/域名各自为政，无统一 Entity 结构与关系层 | 规则二违反 |
| D2 | 历史数据仅 location-history.json（22 个流行 IP、月度、文件存储），无 asn/network 历史 | 规则七「Database Snapshot」缺失 |
| D3 | 前端把 Provider 结构当契约（IpRecord 对齐 ipapi.is），换数据源即破 | useIpQuery.ts |
| D4 | webrtc_peers 表已建但写入失败；无任何历史表 | 见 B2 |

## 1.5 SEO 问题

| # | 问题 | 证据 |
|---|---|---|
| S1 | 首页正文几乎无关键词化内容（hero 静态文案 + 客户端查询渲染），SSR 输出对爬虫价值低 | 首页 HTML |
| S2 | 无结构化数据（JSON-LD WebSite/SearchAction/FAQPage/Breadcrumb 均缺） | 扫描（仅 sitemap 模块） |
| S3 | 动态路由（asn/history/cidr 落地页）无独立 SEO 元信息生成策略 | useSeo.ts 只有固定页面字典 |
| S4 | /ip/xx.html 旧 URL 301 到 /?ip=$1，canonical 目标分散（?ip= 与 /history/ 并存） | nginx conf |
| S5 | 无 llms.txt（AI 爬虫已放行但无 AI 友好清单，可选补齐） | 扫描 |

## 1.6 性能问题

| # | 问题 | 证据 |
|---|---|---|
| P1 | 首页一次性并行拉取 ipquery+ipf+rdomains+history+latency 五路数据后全量渲染 | index.vue |
| P2 | 全站 JS 整包混淆后 40 文件共 1.5MB，无 chunk 级优化 | build 日志 |
| P3 | 无缓存分层（仅 3 处内存缓存），同 IP 重复查询每次全量打上游 | asn-detail/rdomains/location-history |
| P4 | 阻塞型探测（latency 18 节点、tracepath 20s）与页面渲染耦合 | latency.ts/trace.ts |

## 1.7 部署问题

| # | 问题 | 证据 |
|---|---|---|
| G1 | 无 git 仓库，回滚靠 19 个 .bak + 20+ tar.gz 人工挑选 | 扫描 |
| G2 | 无测试环境：preview.duoniu.cc 旧域配置与主域分叉（AI 拦截未同步） | conf 对比 |
| G3 | 宝塔保存站点配置会覆盖手工 conf（历史多次） | 记忆 |
| G4 | 无测试框架、无 lint 门禁，改动靠手工冒烟 | 扫描 |
| G5 | postbuild 混淆关闭了 Source Map，线上排障困难 | 构建管线 |

---

# 2. 改造总体架构

## 2.1 分层设计

```
┌─────────────────────────────────────────────────┐
│  Frontend（Nuxt3 pages + features + composables）│  只调用 /api/*，禁止直连外部
└──────────────────────┬──────────────────────────┘
                       │ 统一错误信封 / 鉴权 / 限流
┌──────────────────────▼──────────────────────────┐
│  DuoNiu API Layer（server/api，面向页面语义）      │  按 Entity 聚合，不暴露 Provider 细节
└──────────────────────┬──────────────────────────┘
                       │ 实体识别 → 编排 → 组装
┌──────────────────────▼──────────────────────────┐
│  Intelligence Service（server/services）          │  QueryParser / EntityRecognizer /
│                                                  │  EntityAssembler / RelationshipBuilder
└──────────────────────┬──────────────────────────┘
                       │ 统一适配器接口 + 优先级 + 降级
┌──────────────────────▼──────────────────────────┐
│  Provider Adapter Layer（server/providers）       │  每个上游一个适配器，输出统一片段
└──────────────────────┬──────────────────────────┘
                       │ 网络调用（不改动上游）
┌──────────────────────▼──────────────────────────┐
│  Existing Data Sources（现状不动）                 │  ipquery PHP:8899 / ipf py:8010 /
│                                                  │  global py:8898 / CF Worker / RIPEstat /
│                                                  │  rdap.org / DoH / hackertarget / 本机探测
└─────────────────────────────────────────────────┘
```

## 2.2 每层职责

| 层 | 职责 | 不做什么 |
|---|---|---|
| Frontend | 页面/组件渲染、交互、首屏 6 要素 + 七 Tab；取数统一走 `useApi()` | 不拼 URL 调外部域名；不解析 Provider 原始结构 |
| DuoNiu API | 页面语义端点（/api/intel/ip、/api/intel/asn、/api/search/parse…）；参数校验、SSRF 防护、限流、统一错误信封 `{ok,data,error,degraded,stale,source}` | 不直接实现业务；不暴露上游原始 JSON |
| Intelligence Service | 输入识别（IPv4/IPv6/ASN/CIDR/Domain）、多源编排（并行 allSettled）、实体组装（IpEntity/AsnEntity/NetworkEntity/DomainEntity）、冲突检测（多源不一致标记）、历史快照对比 | 不做网络请求本身（经 Adapter） |
| Provider Adapter | 每个上游一个文件：超时/重试/配额、原始响应 → EntityFragment 归一化、声明优先级与能力 | 不面向页面做聚合；不改上游行为 |
| Data Sources | 维持现状运行（PHP/python/Worker/公网 API 全部不动） | — |

## 2.3 为什么这样设计

1. **对应规则五**：Search Input → Query Parser（Intelligence Service）→ Query Router（API 路由）→ Intelligence Service → Provider（Adapter）→ Cache/DB —— 分层即规则五的落地。
2. **对应规则六**：前端禁止直连 Provider 的唯一解法是 BFF 式 API 层；Adapter 层让"换数据源"变成改一个文件而非改页面。
3. **对应规则七**：容灾（Primary→Fallback→Cache→Snapshot）只能在 Adapter 层统一实现——每个 Adapter 声明 priority，Service 按序尝试并打 degraded/stale 标记，页面只读信封字段即可渲染「数据暂时不可用，正在显示最近缓存数据」。
4. **对应规则九**：features/ 目录承载每类对象的组件；services/ 承载可复用逻辑；types/ 承载实体契约。

## 2.4 如何兼容当前系统

| 现状 | 兼容策略 |
|---|---|
| 前端直连 ipquery/ipf | 先建 Adapter + API 端点（双轨），页面逐 Phase 切换到 /api/intel/*；切换完成前 Provider 仍可直连（灰度期共存） |
| PHP/python 上游 | **完全不改**——Adapter 只是把 HTTP 调用搬进 Nuxt 服务端，请求参数/结构照旧 |
| 现有 12 条 /api 路由 | 保留别名：新路由上线后旧路由内部改为转发新 Service，逐步只读化，**不删除** |
| IpRecord 等前端类型 | 原样迁移到 server/types/entities.ts 作为 Entity 基础，新增字段可选，保持向后兼容 |
| webrtc_peers / location-history.json | 维持现状；DB 修复见 §6；JSON 迁移入库后保留文件为备份 |
| nginx / systemd / 宝塔 | duoniu-preview 服务名与 3901 端口不变；新增 staging 单元与 preview 域指向（见 §7） |
| 混淆构建管线 | 保留；开发/测试构建加环境变量开关跳过混淆，生产照常 |

---

# 3. 分阶段实施计划

> 每个 Phase 交付可上线的增量，且**不删除任何现有功能**。Phase 之间可独立回滚。

## Phase 0 — 项目安全准备（0.5 天）

**目标：让后续所有改动可回滚、可追溯、可测试。**

### 修改文件
- 无（不动业务代码）

### 新增文件
| 路径 | 内容 |
|---|---|
| duoniu-new-vue/.gitignore | node_modules/.output/.nuxt/.cache/data/*.log |
| duoniu-new-vue/.env.example | DB_HOST/DB_PORT/DB_NAME/DB_USER/DB_PASS 占位（真实值用户填入服务器 .env，不入库） |
| duoniu-new-vue/README.md | 构建/部署/回滚命令速查 |
| /www/backup/duoniu-phase0-<日期>/ | 备份归档目录 |

### 执行步骤
1. **Git 初始化**（duoniu-new-vue）：
   ```
   cd /www/wwwroot/duoniu.cc/duoniu-new-vue
   git init && git add -A && git commit -m "baseline: pre-upgrade snapshot 2026-08-15"
   git tag baseline-20260815
   ```
   旧树决策：/www/wwwroot/duoniu/（PHP+python）**不入本仓库**，单独 `tar czf /www/backup/duoniu-legacy-20260815.tgz` 归档，后续 Adapter 只读不写。
2. **当前版本备份**：
   - 构建产物：`tar czf /www/backup/duoniu-phase0-20260815/.output-prod-20260815.tgz .output`
   - nginx：复制 duoniu.cc.conf / preview.duoniu.cc.conf / ipquery.conf / ipf.duoniu.cc.conf / global.duoniu.cc.conf
   - 数据：location-history.json、.cache/、MySQL（需用户提供凭据后 `mysqldump ipquery`）
   - 清理前置：把 19 个 .bak 移到 `legacy/bak-20260815/`（保留不删），20+ tar.gz 移到 /www/backup/duoniu-targz/
3. **测试环境**（§7 详述）：systemd `duoniu-staging`（端口 3902，独立 .output-staging），nginx 把 preview.duoniu.cc 指向 3902 并同步主域规则（AI 放行、proxy_cache off）。
4. **回滚方案落地**（§7 详述）：写 `scripts/rollback.sh`（恢复 .output 备份 + restart）与 `scripts/deploy.sh`（build→smoke→promote）。
5. **授权清理**（需用户确认）：kill 2 个孤儿 build 进程；配置 .env 真实 DB_PASS 并 `systemctl daemon-reload + restart duoniu-preview`（修 B2）。

### 风险
- git add -A 误纳入 node_modules（.gitignore 先行，双查 `git status`）
- staging 与生产共享上游/DB，staging 写 webrtc_peers 会污染统计 → staging 单元环境变量设 `DB_NAME=ipquery_staging`（允许建测试库）或 peer 上报禁用开关
- 杀进程误伤（只按 PID 杀，先 ps 复核命令行为 `nuxt build`）

### 测试方案
- `git status` 干净、`git log` 有 baseline tag
- staging 访问 preview.duoniu.cc 返回 200 且含 hm.js
- 生产 curl 首页 200、百度统计仍在（回归基线）
- rollback.sh 在 staging 上演练一次成功

---

## Phase 1 — 基础架构重构（2-3 天）

**目标：搭出分层骨架（services/providers/types/features + 统一 API 层），存量功能零变化。**

### 新增文件
| 路径 | 职责 |
|---|---|
| server/types/entities.ts | IpEntity / AsnEntity / NetworkEntity / DomainEntity / EntityLink / ProviderFragment 类型（由 useIpQuery.ts、useIpLine.ts 类型迁移而来，字段向后兼容） |
| server/utils/api.ts | 统一响应信封 `ok(data)` / `fail(code,msg)` / `degraded(data,note)`；错误文案 zh/en 字典（含规则七降级文案） |
| server/utils/cache.ts | 内存 LRU（TTL 可配）+ 磁盘缓存助手（沿用 .cache/ 模式） |
| server/providers/ipquery.ts | 适配 PHP :8899（/、/score、/track） |
| server/providers/ipf.ts | 适配 python :8010（?ip / ?asn / history=1） |
| server/providers/global.ts | 适配 python :8898（geo 兜底） |
| server/providers/ripestat.ts | RIPEstat overview/announced-prefixes |
| server/providers/rdap.ts | rdap.org ip/domain |
| server/providers/doh.ts | 阿里 DoH→Google DoH 链路 |
| server/providers/hackertarget.ts | 反查域名 |
| server/providers/selfprobe.ts | 本机 ping/latency/trace 探测 |
| server/services/intelligence/recognizer.ts | 实体识别（Phase 2 使用，本阶段先落地接口） |
| server/services/intelligence/assembler.ts | 多源编排（allSettled、优先级、降级标记） |
| app/composables/useApi.ts | 前端唯一取数入口（$fetch 封装 + 错误信封解析） |
| app/features/（空目录占位） | ip/ asn/ bgp/ dns/ cidr/ history/ diagnostics/ 子目录 |

### 修改文件
| 文件 | 改动 |
|---|---|
| server/utils/db.ts | 连接配置加 `enableKeepAlive` 已有；增加启动日志与失败告警日志（不静默） |
| nuxt.config.ts | runtimeConfig 增加 providerBaseUrl/providerTimeouts 配置段 |
| app/pages/*（5 个直连页） | **本阶段不改**（双轨期） |
| 12 条 server/api 路由 | 内部实现改为调用对应 provider 适配器（行为一致），保留路由路径不变 |

### 风险
- **API 兼容风险**：把直连搬进服务端后，前端拿到的结构若字段名有出入会回归 → 适配器归一化时做「逐字段 diff 测试」（录制当前线上响应快照对比）
- **数据错误风险**：超时/重试参数与上游不一致导致 5xx 增多 → provider 超时设 8-15s 且总兜底 `degraded`
- **SSRF**：新 Service 层统一出口校验（ping/任何 URL 参数：禁私网/回环/链路本地，参考 siteintel ssrf-guard 做法）

### 测试方案
- `npm run build` 通过（nvm v20）
- 录制-回放：对 8.8.8.8、AS13335、cloudflare.com 三对象，对比「旧直连响应」与「新 /api 响应」逐字段 diff 为 0
- 上游宕机演练：临时改 ipf 端口验证返回 degraded 而非 500
- 页面回归：curl 首页/env/purity/asn 均 200 且渲染关键字段

---

## Phase 2 — Universal Search（2 天）

**目标：任何入口输入任何网络对象，自动识别类型并路由（规则四）。**

### 新增文件
| 路径 | 职责 |
|---|---|
| server/api/search/parse.get.ts | 服务端识别端点：`GET /api/search/parse?q=` → `{type, normalized, redirect, confidence}` |
| server/services/intelligence/recognizer.ts 补全 | 识别规则（下表） |
| server/services/intelligence/router.ts | type → 页面路由映射 |
| app/components/search/UniversalSearch.vue | 通用搜索框（含识别 loading、错误提示、历史建议） |
| app/composables/useSearch.ts | 调 parse + router.push 封装 |
| app/pages/domain/[q].vue | 域名情报落地页（骨架，Phase 3 填充） |

### 识别规则（Query Parser）

| 输入示例 | 识别为 | 归一化 | 路由 |
|---|---|---|---|
| `8.8.8.8` / `2400:cb00::` | IPv4 / IPv6 | 标准 IP 文本 | /ip/[ip]（新统一页，先落地为首页 ?ip= 兼容） |
| `AS13335` / `as 13335` / `13335`（上下文 ASN） | ASN | 去 AS 前缀纯数字 | /asn/[asn] |
| `1.1.1.0/24` / `2001:db8::/32` | CIDR | 前缀文本 | /cidr/[q]（新增动态路由，修 F6） |
| `cloudflare.com` / 子域名 | Domain | 小写去尾点 | /domain/[q] |
| 歧义输入（纯数字、带 / 非合法前缀） | unknown | — | 搜索框提示修正 |

### 修改文件
| 文件 | 改动 |
|---|---|
| app/pages/index.vue | 搜索框换成 UniversalSearch（保留原 IP 查询行为为 ipv4 分支） |
| app/pages/history/index.vue、asn/index.vue、tools.vue 等带搜索框页面 | 统一替换为 UniversalSearch |
| app/pages/cidr.vue | 新增 `/cidr/[q].vue` 动态路由，/cidr 保留为工具页 |

### 风险
- 误识别（如 `1.1.1` 域名形态）→ 识别规则带 confidence，<0.9 时前端回退提示「按 IP/域名分别查询」
- 路由变化影响老书签 → 旧路径全部保留，新路径叠加
- SSR 下 parse 需要与客户端一致 → 识别逻辑纯函数放 server/utils，两端共用

### 测试方案
- parse 单测表：≥20 个用例（5 类型 × 正常/边界/恶意输入）
- 页面测试：首页输入 AS13335 → 跳 /asn/13335；输入 1.1.1.0/24 → /cidr/1.1.1.0%2F24
- SSR：curl 新落地页 200 且 meta 正确

---

## Phase 3 — 核心页面改造（5-7 天）

**目标：按序把 12 个模块迁到 features/ + 统一 API，首页完成七 Tab 化。每页改造遵循同一模板（修改文件/风险/测试见各页表）。**

### 通用模板（每页适用）
- 页面薄壳化：`app/pages/xx.vue` → `<FeatureX />`；逻辑迁 `app/features/xx/`
- 取数改 `useApi()`，Provider 直连删除
- SEO 元信息在页面用 `useSeoMeta`（siteSeo 字典已有项沿用）
- 保留全部现有展示字段与功能，只做结构迁移

### 3.1 首页（最高优先级）

- **新增**：app/features/home/HomeSummary.vue（第一屏 6 要素条：IP/Country/ISP/ASN/Type/Risk）、HomeTabs.vue（七 Tab 容器）、tabs/OverviewTab.vue、NetworkTab.vue、LocationTab.vue、RiskTab.vue、HistoryTab.vue、RoutingTab.vue、RawDataTab.vue
- **修改**：index.vue 1229 行拆分——现有卡片内容**原样搬入对应 Tab**（不删字段），hero 换 UniversalSearch + HomeSummary
- **风险**：大文件拆分回归面最大 → 每拆一个 Tab 构建冒烟一次；Tab 懒加载避免首屏劣化
- **测试**：七 Tab 切换、?ip= 直达仍渲染 Overview、与改造前截图对比字段齐全

### 3.2-3.12 各页改造表

| 序 | 模块 | 新 features 目录 | 统一 API | 风险 | 测试要点 |
|---|---|---|---|---|---|
| 2 | IP Intelligence（env/ip-leak） | features/ip/env、features/ip/leak | /api/intel/ip（聚合 ipquery+ipf+global） | 本地探测逻辑（useEnvCheck 纯前端）必须保留原样 | WebRTC/DNS/IPv6 探测结果与现状一致 |
| 3 | Purity | features/ip/purity | /api/intel/ip?view=purity | 评分算法 computePurity 迁移时数值必须一致 | 固定样本评分 diff=0 |
| 4 | ASN | features/asn/ | /api/intel/asn（聚合 ripestat+ipf+ip-api） | 上游两套（RIPEstat 直连 vs ipf）结果需合并去重 | AS13335 前缀数/邻居数对比现状 |
| 5 | BGP | features/bgp/ | /api/intel/bgp | 现有 /api/bgp→ipf 链路保持 | 查询结果与现状一致 |
| 6 | CIDR | features/cidr/ | 前端计算（无服务端依赖）+ /cidr/[q] 动态路由 | 修路由 warn 时勿破坏 /cidr 工具页 | 输入 /24 /16 IPv6 边界 |
| 7 | DNS | features/dns/ | /api/intel/dns（doh 适配器，保留阿里→Google 容灾） | PTR 自动反转逻辑保留 | A/AAAA/PTR/MX 各类型 |
| 8 | Whois | features/whois/ | /api/intel/whois（rdap 适配器） | rdap.org 限流，加缓存 | 域名/IP 两形态 |
| 9 | Ping | features/ping/ | /api/intel/ping（selfprobe；**加 SSRF 校验**） | 校验规则误伤合法域名（含 IDN/短链） | 公网/私网/元数据地址拒绝矩阵 |
| 10 | Trace | features/trace/ | /api/intel/trace | tracepath 依赖系统二进制，容器/权限差异 | 超时保留部分跳点逻辑不变 |
| 11 | Route | features/route/ | /api/intel/route（ipf 适配器） | 与 BGP 数据同源但视图不同，勿合并接口 | /route/[ip] 与 /bgp/[q] 输出对比 |
| 12 | History | features/history/ | /api/intel/history（location-history 存储 + 新增 asn_history 见 §6） | 月快照与按需补采的并发去重（pending 集合）逻辑保留 | 当月无数据时异步补采不阻塞响应 |

### Phase 3 完成标准
- `grep -rn "ipquery.duoniu.cc\|ipf.duoniu.cc" app/` 除 plugins/track 与注释外**零命中**（track 插件走 /api 化在 Phase 4 一并处理）
- 全部旧路由可访问、功能与改造前一致（回归清单 12 项全过）

---

## Phase 4 — SEO 升级（2 天）

### 修改文件
| 文件 | 改动 |
|---|---|
| app/composables/useSeo.ts | 新增 domain/ip/cidr 落地页动态元信息生成函数（模板 + 实体字段插值） |
| app/pages/*.vue | 统一 useSeoMeta + useCanonical；动态页 og:title/og:description |
| app/layouts/default.vue | 注入全局 JSON-LD：WebSite + SearchAction（https://www.duoniu.cc/?ip={query}） |
| app/pages/faq.vue | 注入 FAQPage Schema（问题/答案对） |
| app/pages/about.vue | Organization Schema |
| app/features/*/页面 | BreadcrumbList Schema（面包屑组件联动） |
| nuxt.config.ts | sitemap 模块补动态源：/asn/[asn]、/cidr/[q]、/history/[ip]、/domain/[q]（从 entities 表取热门实体）；exclude /admin/**、/api/** |
| app/plugins/track.client.ts | 上报改走 /api/track（Nuxt 服务端转发 ipquery），消除前端直连遗留 |
| public/robots.txt | 保持搜索引擎+AI 放行现状（已上线）；加 sitemap 行确认 |
| public/llms.txt（新增） | AI 友好站点说明 + 关键页面清单（可选但建议，AI 爬虫已放行） |

### 风险
- Schema 语法错误会被 Google 忽略——用 Rich Results Test 逐类型验证
- 动态 sitemap 引入构建期数据依赖（entities 表），数据为空时降级为静态路由清单
- 百度验证文件（baidu_verify_*）与统计代码不能被新构建覆盖——继续走 .output/public 同步与 nginx 直出约定（参考 siteintel 的 `location ^~ /baidu_verify_` 方案，如 duoniu 后续需要）

### 测试方案
- build 后 curl 抽查 6 类页面：title/description/canonical/JSON-LD 齐全
- sitemap.xml 含新动态路由且无 /admin
- robots.txt 与线上一致（GPTBot Allow）
- 百度统计脚本仍出现在所有页面（回归）

---

## Phase 5 — 性能优化（2-3 天）

### 修改内容
| 项 | 方案 |
|---|---|
| SSR 优化 | SEO 关键页（首页/实体落地页）保持 SSR；探测类（latency/ping/trace）仅在客户端触发；useAsyncData 加 getCachedData 复用 |
| Cache 分层 | providers 全部接入 server/utils/cache.ts（LRU TTL：ipquery 5min / ipf 10min / ripestat 1h / rdap 1h / hackertarget 10min）；实体组装结果缓存 5min；nginx 维持 proxy_cache off（HTML） |
| Lazy Loading | 七 Tab 面板 `defineAsyncComponent` 按 Tab 加载；TopologyMap/DetectPanel 懒加载；SimpleTool 六合一拆分后各 feature 自动按路由分包 |
| Virtual List / 分批 | rdomains（≤500）、asn 前缀列表（≤500）、历史记录列表：超 100 条用「加载更多」+ 分批渲染；数据达万级再引入虚拟滚动（当前规模不必） |
| Bundle | 混淆范围收窄：业务核心 chunk 混淆、vendor 不混淆（体积与排障平衡，需用户确认） |
| 探测异步化 | /api/intel/latency 允许 `?mode=stream` 分段返回；trace 保持超时机制 |

### 测试方案
- 构建产物各路由 chunk 体积报告（目标：实体落地页 gzip < 250KB）
- 同 IP 第二次查询命中缓存（服务端日志加 cache hit 标记验证）
- Lighthouse 抽测首页：FCP < 1.8s / LCP < 2.5s（移动 4G 模拟）
- 线上压测：ab 或 wrk 500 并发首页 200 率 > 99.9%

---

# 4. 数据架构设计（Provider Adapter 重点）

## 4.1 适配器接口（统一契约）

```ts
// server/providers/types.ts
export interface QueryRequest {
  entityType: 'ip' | 'asn' | 'network' | 'domain'
  value: string                      // 归一化后的对象值
  views?: string[]                   // 需要的片段: overview|routing|history|risk...
}

export interface ProviderFragment {
  provider: string                   // 适配器名
  entityType: EntityType
  view: string
  data: Partial<IpEntity | AsnEntity | NetworkEntity | DomainEntity>
  fetchedAt: string
  stale?: boolean                    // 来自缓存
}

export interface ProviderAdapter {
  name: string
  priority: number                   // 越小越优先（Primary=1, Fallback=2…）
  capabilities: { entityType: EntityType; views: string[] }[]
  fetch(req: QueryRequest, ctx: FetchCtx): Promise<ProviderFragment>
}
```

## 4.2 Provider 矩阵（现状 → 适配器）

| Adapter | 优先级 | 能力 | 上游 | 现状调用方 |
|---|---|---|---|---|
| ipquery | 1（IP 主源） | ip: overview/risk/purity/score | PHP :8899 | 前端直连 5 页 + track 插件 |
| ipf | 1（路由主源） | ip: routing/history；asn: overview/topology/ix | python :8010 | useIpLine 三函数 + /api/bgp |
| global | 2（位置兜底） | ip: location | python :8898（mmdb） | 当前零引用 → 激活为 fallback |
| ripestat | 2（ASN 兜底） | asn: overview/prefixes | stat.ripe.net | /api/asn-detail、/api/asn-list |
| rdap | 1（注册信息） | ip/domain: registration/whois | rdap.org | /api/whois |
| doh | 1/2（阿里→Google） | domain: dns | 阿里 DoH / Google DoH | /api/dns |
| hackertarget | 1 | ip: rdomains | hackertarget | /api/rdomains |
| selfprobe | 1 | diagnostics: ping/trace/latency | 本机 | /api/ping、/api/trace、/api/latency |

## 4.3 统一实体输出（Entity）

```ts
// server/types/entities.ts（要点）
interface IpEntity {
  ip: string; version: 4 | 6
  risk: { is_bogon; is_proxy; is_vpn; is_tor; is_datacenter; is_abuser; is_crawler; is_mobile; is_satellite }
  purity: { score: number; category: 'good'|'warn'|'bad'; flags: Record<string, boolean> }
  company?: { name; domain; type; network }
  asn?: { asn; org; country; route; descr }
  location?: { country; countryCode; state; city; lat; lon; timezone }
  routing?: { origin; visibility; firstSeen; lastSeen }
  rdomains?: string[]
  history?: { month; country; isp; asn }[]
  sources: string[]                    // 实际参与组装的 provider 列表
  degraded?: string[]                  // 降级说明（规则七文案来源）
}

interface AsnEntity {
  asn: number; name; cc; rir
  announced: boolean
  prefixes: { v4: string[]; v6: string[]; total: number }
  topology?: { upstreams; downstreams; peers }
  ix?: { name; speed; status }[]
  org?: { name; domain; abuseEmail }
}

interface NetworkEntity {
  prefix: string; version: 4|6
  netmask: string; wildcard: string
  firstUsable; lastUsable; broadcast?: string
  hostCount: number
  members?: string[]                   // 归属该网段的已知实体
}

interface DomainEntity {
  name: string; apex: string
  resolved?: { ip: string; type: string }[]
  dns?: Record<string, string[]>
  registration?: { registrar; created; expires; nameservers }
  relationship?: { sharedIpEntities: string[] }
}

interface EntityLink {
  from: { type: EntityType; value: string }
  to: { type: EntityType; value: string }
  relation: 'announces' | 'resolves_to' | 'owned_by' | 'contained_in' | 'shares_ip_with'
  source: string
}
```

## 4.4 组装流程（Intelligence Service）

```
GET /api/intel/ip?value=8.8.8.8
  1. recognizer → { type: ip, value: 8.8.8.8 }
  2. capabilities 匹配 → [ipquery(P1), ipf(P1 routing), global(P2 location), hackertarget(P1 rdomains)]
  3. Promise.allSettled 并行调用（各 8-15s 超时）
  4. merge：P1 优先，P2 补位；同字段冲突 → 冲突标记 conflictingSources
  5. cache 写入（5min）+ history 对比（entity_snapshots 见 §6）
  6. 返回信封 { ok, data: IpEntity, degraded?, stale?, sources }
```

---

# 5. 数据库建议（兼容现有，不重建）

> 原则：**不 DROP/ALTER 任何现有表**；新增表为增量；location-history.json 迁移后保留原文件为备份。

## 5.1 现状
- MySQL 127.0.0.1:3306，库 `ipquery`，表 `webrtc_peers`（写入失败待修）
- 文件存储：data/location-history.json（月度快照）、.cache/asn-list.json

## 5.2 兼容步骤
1. 修复凭据：.env 配真实 DB_PASS（用户提供）→ systemd EnvironmentFile 或直接环境变量 → 重启验证 `[peer]` 写入成功
2. 迁移脚本 `scripts/migrate-history.mjs`（幂等）：把 location-history.json 的每条记录 upsert 进 `ip_history`，字段对齐后 JSON 文件保留为只读备份

## 5.3 新增表（均为 CREATE TABLE IF NOT EXISTS，纯增量）

```sql
-- 实体主表（IP/ASN/网段/域名统一登记）
CREATE TABLE IF NOT EXISTS entities (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  type ENUM('ip','asn','network','domain') NOT NULL,
  value VARCHAR(255) NOT NULL,                 -- 归一化值
  first_seen DATETIME DEFAULT CURRENT_TIMESTAMP,
  last_seen DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  meta JSON NULL,
  UNIQUE KEY uk_type_value (type, value)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 实体关系（规则二「建立实体关系」）
CREATE TABLE IF NOT EXISTS entity_relations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  from_id BIGINT UNSIGNED NOT NULL,
  to_id BIGINT UNSIGNED NOT NULL,
  relation ENUM('announces','resolves_to','owned_by','contained_in','shares_ip_with') NOT NULL,
  source VARCHAR(32) NOT NULL,
  first_seen DATETIME DEFAULT CURRENT_TIMESTAMP,
  last_seen DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_rel (from_id, to_id, relation, source),
  KEY idx_to (to_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- IP 历史（承接 location-history.json）
CREATE TABLE IF NOT EXISTS ip_history (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  ip VARCHAR(45) NOT NULL,
  month CHAR(7) NOT NULL,                      -- YYYY-MM
  country VARCHAR(64), region VARCHAR(64), city VARCHAR(64),
  isp VARCHAR(255), asn INT,
  lat DECIMAL(9,6), lon DECIMAL(9,6),
  source VARCHAR(32) DEFAULT 'ipquery',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_ip_month (ip, month)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ASN 历史（新增维度）
CREATE TABLE IF NOT EXISTS asn_history (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  asn INT NOT NULL,
  month CHAR(7) NOT NULL,
  name VARCHAR(255), cc CHAR(2),
  prefixes4 INT, prefixes6 INT,
  source VARCHAR(32) DEFAULT 'ripestat',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_asn_month (asn, month)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 通用实体快照（规则七「Database Snapshot」+ 变化检测）
CREATE TABLE IF NOT EXISTS entity_snapshots (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  entity_id BIGINT UNSIGNED NOT NULL,
  snapshot_at DATETIME NOT NULL,
  content_hash CHAR(32) NOT NULL,
  data JSON NOT NULL,
  UNIQUE KEY uk_entity_hash (entity_id, content_hash),
  KEY idx_entity_time (entity_id, snapshot_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

> 表名 `network_entity` 的需求由 `entities`（type='network'）+ `entity_snapshots` 覆盖，不单独建表，避免与 entities 分裂。

## 5.4 缓存层（规则七 Redis Cache）
- 短期：沿用内存 LRU + .cache 磁盘（现状模式，够用）
- 中期（数据量大后）：宝塔 Redis + ioredis，缓存 key `intel:{type}:{value}:{view}`，TTL 分层
- 落库策略：组装成功且非 stale 时写 entity_snapshots（hash 变化才插行）

---

# 6. 部署方案

## 6.1 环境矩阵

| 环境 | 域名 | 端口 | systemd | 说明 |
|---|---|---|---|---|
| 开发 | 无（SSH 本机） | 3903（`npm run dev`，bind 127.0.0.1） | 无 | 热更新；混淆关闭；走真实上游 |
| 测试 | preview.duoniu.cc | 3902 | duoniu-staging | 独立构建产物 .output-staging；同步主域 nginx 规则 |
| 生产 | www.duoniu.cc | 3901 | duoniu-preview（**名不换**，避免 conf/历史依赖断裂） | 现状即生产 |

- 测试环境数据策略：LOC_HISTORY_STORE 独立文件；MySQL 只读（peer 上报禁用开关 `DISABLE_PEER_STORE=1`），避免污染生产统计

## 6.2 部署流程（scripts/deploy.sh，每个 Phase 结束执行）

```
1. git tag vX.Y-phaseN && git push（如有远端）
2. 构建：export PATH=/root/.nvm/versions/node/v20.20.0/bin:$PATH
   npm run build（生产含混淆；staging 用 npm run build:staging 跳过混淆）
3. 冒烟：node scripts/smoke-test.mjs https://preview.duoniu.cc（首页/关键页 200 + hm.js + 关键字段断言）
4. 晋升：rsync .output → .output-prod（先 .output-prod.prev 轮换）
5. systemctl restart duoniu-preview && sleep 4 && is-active 校验
6. 线上验证：curl 首页/实体页/百度统计/AI bot UA 200
7. 记录：git tag + 更新 /www/backup/duoniu-deploy-<date>.md
```

## 6.3 回滚流程（scripts/rollback.sh）

```
1. systemctl stop duoniu-preview
2. mv .output-prod .output-prod.bad-$(date)；mv .output-prod.prev .output-prod
3. systemctl start duoniu-preview
4. 线上验证 200
（nginx conf 回滚：cp /www/backup/<conf>.bak-* → /www/server/panel/vhost/nginx/ && nginx -s reload）
（数据回滚：迁移类脚本均为幂等 INSERT，回滚不动数据；如确需，用 mysqldump 备份恢复）
```

## 6.4 红线
- 宝塔面板保存站点配置会覆盖手工 conf → 每次改 conf 后复查，改动前备份
- 构建前必查 `NEXT_PUBLIC_*`/runtimeConfig 指向生产域名（ipcesu 事故教训同理适用）
- HTML 不允许被 nginx/CF 缓存（proxy_cache off 与 no-cache 头维持现状）

---

# 7. 最终验收标准

## 7.1 功能验收

- [ ] 现有 12 类页面全部可访问、功能与改造前一致（回归清单逐项通过，**零功能删除**）
- [ ] Universal Search：IPv4/IPv6/Domain/ASN/CIDR 五类型输入均正确识别与路由（20 用例全过）
- [ ] 首页：第一屏 6 要素（IP/Country/ISP/ASN/Type/Risk）；七 Tab 完整切换
- [ ] 前端零 Provider 直连：`grep -rn "ipquery.duoniu.cc\|ipf.duoniu.cc" app/` 零命中（track 插件已 /api 化）
- [ ] 所有错误展示统一降级文案「数据暂时不可用，正在显示最近缓存数据」，无裸 `Failed to fetch`
- [ ] webrtc_peers 写入恢复（DB 凭据修复后 INSERT 成功）
- [ ] 管理后台路径修复：/admin-api/ 不再指向 3001 错误目标（或按用户决策下线该反代）

## 7.2 性能验收

- [ ] 首页 LCP < 2.5s、FCP < 1.8s（移动 4G 模拟，Lighthouse）
- [ ] 实体落地页 gzip 后 < 250KB / 路由
- [ ] 缓存命中：同实体二次查询服务端不再回源（hit 日志验证）
- [ ] 500 并发压测首页 200 率 > 99.9%，p95 响应 < 800ms
- [ ] SSR TTFB < 500ms（本地回源）

## 7.3 SEO 验收

- [ ] 全部页面 title/description/canonical 唯一且带品牌词
- [ ] JSON-LD：WebSite+SearchAction、FAQPage、Organization、BreadcrumbList 通过 Rich Results Test
- [ ] sitemap.xml 覆盖新动态路由、排除 /admin /api
- [ ] robots.txt 搜索引擎+AI 爬虫放行（现状保持）；采集工具拦截保持
- [ ] 百度统计 hm.js 全站存在；百度验证文件与统计代码不被构建覆盖
- [ ] llms.txt（如实施）可被 GPTBot/ClaudeBot 抓取（实测 200）

## 7.4 安全验收

- [ ] /api/intel/ping 等含 URL 参数端点：私网/回环/链路本地/云元数据地址全被拒（拒绝矩阵测试）
- [ ] 所有 /api/* 有输入校验（长度/格式/枚举），无原型污染与注入面
- [ ] 无密钥泄漏：前端构建产物 grep 无 AK/SK/TOKEN；.env 不入 git
- [ ] API 限流：单 IP 高频请求返回 429（peer/track/实体查询均覆盖）
- [ ] UA 拦截、CSP 相关头不劣化；AI 爬虫放行不回退

---

# 附：执行顺序总览与依赖

```
Phase 0（安全准备） ──→ 一切的前提，先做
Phase 1（基础架构） ──→ Phase 2/3 依赖其 services/providers/types
Phase 2（Universal Search） ──→ 依赖 Phase 1；可与 Phase 3.1 首页改造合并进行
Phase 3（页面改造） ──→ 依赖 Phase 2 的搜索组件；内部顺序 1→12
Phase 4（SEO） ──→ 依赖 Phase 3 页面形态稳定（否则 meta 改两次）
Phase 5（性能） ──→ 依赖 Phase 3 结构（缓存/Tab 懒加载建立在 features 上）
数据迁移（§5） ──→ Phase 1 修凭据、Phase 3.12 前后做 history 迁移
部署/回滚脚本 ──→ Phase 0 落地，此后每 Phase 复用
```

**风险总提示**：每个 Phase 单独 tag、单独冒烟、可独立回滚；不跨 Phase 合并上线。数据层（entities/relations/snapshots）从 Phase 3.12 开始写入，早期 Phase 只读现有源，保证「不删除任何现有功能、不破坏现有数据」两条底线。
