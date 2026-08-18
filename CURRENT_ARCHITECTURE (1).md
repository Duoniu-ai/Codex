# DUONIU.cc 当前架构审计报告

> 生成时间：2026-08-15
> 依据：全量只读扫描（未修改任何代码）
> 对照文件：`DUONIU_UPGRADE_RULES.md`（服务器 /www/wwwroot/duoniu.cc/DUONIU_UPGRADE_RULES.md，与本地副本 MD5 一致）

---

# 1. 当前技术架构

## 1.1 前端（生产）

| 项 | 值 |
|---|---|
| 框架 | Nuxt 3.17.5 + Vue 3 + TypeScript，SSR（ssr: true） |
| 目录 | `/www/wwwroot/duoniu.cc/duoniu-new-vue`（**无 git 仓库**） |
| 运行时 | systemd `duoniu-preview`，端口 3901，`node .output/server/index.mjs`（nvm node v20.20.0） |
| 构建 | `npm run build`（nuxt build + postbuild `scripts/obfuscate-client.mjs` 整包混淆，40 个 JS 文件） |
| 模块 | @nuxtjs/sitemap（hostname https://www.duoniu.cc）；无数据库 ORM、无状态管理库、无测试框架 |
| 统计 | 百度统计 hm.js `f135edf092b4b018ff7f6cb4264c54fb`（2026-08-15 接入，nuxt.config.ts app.head.script 内联） |

## 1.2 反向代理 / 站点（nginx 宝塔 conf）

主域 `duoniu.cc.conf`（/www/server/panel/vhost/nginx/）：

- `root /www/wwwroot/duoniu.cc/public`，`location /` → Nuxt 3901（proxy_cache off、no-cache 头）
- `/_nuxt/` → 3901（缓存 1h）
- `/vps/` → Django（gunicorn 127.0.0.1:5000，sqlite，VPS 推广/采集子系统）
- `/admin/` 静态 alias → public/admin/（纯静态页）；`/admin-api/` → `127.0.0.1:3001/api/`（⚠ 见问题 #1）
- `/api/ip/stats`、`/api/asn/stats`、`/api/country/stats`、`/api/ip/detail` → 公网 HTTPS 反代 `api.duoniu.cc`（Cloudflare Worker）
- UA 黑名单拦采集工具（curl/scrapy/masscan 等仍拦截；AI 爬虫已于 2026-08-15 放行，见备份 bak-20260815-aibots）
- 多条旧 URL 301 兼容（/ip/xxx.html、/latency、/batch 等）；OSM 瓦片反代

## 1.3 后端数据服务（旧树 `/www/wwwroot/duoniu/`，与 Nuxt 分属两套目录）

| 服务 | 域名 | 端口/方式 | 实现 | 管理 |
|---|---|---|---|---|
| 43 主库（纯净度/归属地/ASN/滥用/score/track） | ipquery.duoniu.cc | nginx :8899 → PHP 8.3 | /www/wwwroot/duoniu/ipquery/public（PHP 应用） | 宝塔 PHP 站 |
| IP 线路/BGP/IXP（PeeringDB+RIPEstat） | ipf.duoniu.cc | python :8010 | /www/wwwroot/duoniu/ipf/server.py（sqlite 缓存） | systemd `ipf.service` |
| 全球 IP 归属（GeoLite2 mmdb） | global.duoniu.cc | python :8898 | /www/wwwroot/duoniu/global/server.py | systemd `ipquery-global.service` |
| 统计/工具 API | api.duoniu.cc | Cloudflare Worker `duoniu-api` | /www/wwwroot/duoniu.cc/cloudflare-workers/（/status 在线 200） | wrangler |

## 1.4 数据存储

- **MySQL** 127.0.0.1:3306，库 `ipquery`，仅一张自建表 `webrtc_peers`（server/utils/db.ts 的 ensurePeerTable 创建）。**DB_PASS 未配置**（无 .env、systemd 单元无 Environment）→ 写入失败（见问题 #2）
- **JSON 文件**：`data/location-history.json`（IP 位置月度快照，cron 每日 03:15 采集 22 个流行 IP）；`.cache/asn-list.json`（ASN 列表磁盘缓存 1h）
- **Cloudflare Worker**：wrangler.toml 声明 KV（STATS_CACHE）+ D1（duoniu-db），但 ID 为模板占位 `YOUR_KV_NAMESPACE_ID`（部署状态存疑，见问题 #8）

## 1.5 部署方式

```
本地改代码（服务器上直接改，无 git）
  → export PATH=/root/.nvm/versions/node/v20.20.0/bin:$PATH && npm run build（含 postbuild 混淆）
  → systemctl restart duoniu-preview
  → 宝塔保存站点配置会覆盖手工改的 nginx conf（历史教训）
```

- 回滚依赖手动备份：nuxt.config.ts.bak-*、conf.bak-*（/www/backup/ 与 vhost 目录各有一批）
- 项目根堆积 20+ 个 dn-*.tar.gz 打包快照；app/ 内有 19 个 *.bak* 文件混在源码目录
- 2 个 2026-08-13 遗留的挂死 `nuxt build` 进程（PID 55535/56476）尚未清理
- 双栈并存：老 Vite Vue3 版 `duoniu-vue/` 保留未删；`preview.duoniu.cc` 旧预览域仍在

---

# 2. 页面对应文件

## 2.1 pages/（27 个活跃文件，另有 19 个 .bak 干扰文件）

| 路由 | 文件 | 行数 | 数据来源 |
|---|---|---|---|
| `/` 首页 | app/pages/index.vue | **1229** | 浏览器**直连** ipquery.duoniu.cc（IP 主记录/score）、**直连** ipf.duoniu.cc（线路/历史/ASN，via useIpLine）；同域 /api/rdomains、/api/location-history、/api/latency |
| `/env` IP 环境检测 | app/pages/env.vue | 677 | 前端本地探测（useEnvCheck）+ **直连** ipquery + /api/dns(PTR) + /api/env-headers |
| `/ip-leak` IP 泄漏 | app/pages/ip-leak.vue | 411 | 前端本地（WebRTC/DNS/IPv6 探测，useEnvCheck） |
| `/purity` 纯净度 | app/pages/purity.vue | 146 | **直连** ipquery.duoniu.cc |
| `/history` 历史入口 | app/pages/history/index.vue | 51 | 跳转 /history/[ip] |
| `/history/[ip]` | app/pages/history/[ip].vue | 102 | **直连** ipquery（当前值）+ /api/location-history |
| `/asn` ASN 列表 | app/pages/asn/index.vue | 179 | /api/asn-list（Nuxt 代理 RIPEstat+ip-api）+ **直连** ipf（queryAsnLine） |
| `/asn/[asn]` | app/pages/asn/[asn].vue | 253 | **直连** ipf（useAsyncData SSR）+ /api/asn-detail（RIPEstat 前缀） |
| `/route` / `/route/[ip]` 线路 | app/pages/route.vue / route/[ip].vue | 102/101 | **直连** ipf（queryIpLine） |
| `/bgp`、`/bgp/[q]` | bgp/index.vue、bgp/[q].vue | 6/8 | 薄壳 → SimpleTool(type=bgp)，内部走 /api/bgp → ipf |
| `/dns`、`/dns/[q]` | dns/index.vue、dns/[q].vue | 6/8 | 薄壳 → SimpleTool(type=dns) → /api/dns（阿里 DoH→Google DoH） |
| `/ping`、`/ping/[q]` | ping/*.vue | 6/8 | 薄壳 → SimpleTool(type=ping) → /api/ping |
| `/trace`、`/trace/[q]` | trace/*.vue | 6/8 | 薄壳 → SimpleTool(type=trace) → /api/trace（tracepath） |
| `/whois`、`/whois/[q]` | whois/*.vue | 6/8 | 薄壳 → SimpleTool(type=whois) → /api/whois（rdap.org） |
| `/cidr` | app/pages/cidr.vue | 6 | 薄壳 → SimpleTool(type=cidr)（前端本地计算） |
| `/tools` 工具合集 | app/pages/tools.vue | 59 | 静态链接页 |
| `/api` API 文档 | app/pages/api.vue | 73 | 静态 |
| `/about` `/faq` | about.vue / faq.vue | 121/107 | 静态 |

## 2.2 组件 / 组合式 / 插件

| 文件 | 行数 | 职责 |
|---|---|---|
| components/SimpleTool.vue | 555 | 巨型多态组件：bgp/dns/ping/trace/whois/cidr 六合一（type 分支），**组件内直接 $fetch** |
| components/AppBreadcrumb.vue | 188 | 面包屑 |
| components/TopologyMap.vue | 111 | BGP 拓扑 SVG |
| components/DetectPanel.vue | 75 | 环境检测面板 |
| components/ScoreGauge.vue | 61 | 评分仪表盘 |
| composables/useEnvCheck.ts | 451 | 环境/泄漏探测（WebRTC/DNS/字体/canvas/音频/WebGL/隐身检测）纯前端 |
| composables/useSeo.ts | 150 | 各页 SEO 元信息字典 + useCanonical |
| composables/useIpLine.ts | 126 | **直连 ipf.duoniu.cc** 的查询函数（queryIpLine/queryAsnLine/queryIpHistory） |
| composables/useIpQuery.ts | 99 | IpRecord 类型 + 纯净度计算（本地） |
| composables/useLanguage.ts | ~42 | 中/EN 切换（localStorage） |
| plugins/track.client.ts | 26 | 进站上报 ipquery.duoniu.cc/track（会话一次） |
| layouts/default.vue | 381 | 顶栏/抽屉导航、语言与主题切换 |

---

# 3. API 接口列表

## 3.1 Nuxt 内部 API（server/api/，同域 /api/*）

| 路由 | 上游 | 缓存 | 容灾 |
|---|---|---|---|
| GET /api/asn-detail?asn= | stat.ripe.net（overview+announced-prefixes） | 内存 1h | 无（失败返回 error） |
| GET /api/asn-list | stat.ripe.net + ip-api.com/batch（48 个热门 ASN） | 内存+磁盘 .cache 1h | 部分（国家查询失败保留 fallback） |
| GET /api/bgp?asn=&ip= | **ipf.duoniu.cc**（RIPEstat+PeeringDB） | 上游自管 | 无 |
| GET /api/dns?name=&type= | 阿里 DoH → Google DoH | 无 | ✅ 双源轮询 |
| GET /api/env-headers | 本机（回显请求头） | — | — |
| GET /api/latency | 18 全球节点 + 4 CDN（HEAD→GET 降级） | 无 | 单点失败只标记该点 |
| GET /api/location-history/[ip] | 本地 JSON 月度快照；当月缺失异步触发采集（ipquery） | 文件即缓存 | 采集失败静默，下次再试 |
| POST /api/peer | WebRTC peer 上报 → MySQL webrtc_peers | — | **写入失败（DB 无密码），静默丢数据** |
| GET /api/ping?url= | 任意 http(s) URL HEAD 探测 | 无 | 无 |
| GET /api/rdomains/[ip] | api.hackertarget.com 反查 | 内存 10min | 无 |
| GET /api/trace?target= | 本机 tracepath（timeout 20s） | 无 | 超时保留部分跳点 |
| GET /api/whois?q= | rdap.org（ip/domain 自动分流） | 无 | 无 |

## 3.2 外部服务（前端或 Nuxt 直连）

| 服务 | 关键端点 | 调用方 |
|---|---|---|
| ipquery.duoniu.cc（PHP :8899） | `/?ip=`（主记录，对齐 ipapi.is 结构）、`/score?ip=`、`/track`（POST 进站上报） | 浏览器直连（index/env/purity/history/track 插件） |
| ipf.duoniu.cc（python :8010） | `/?ip=`、`/?asn=`、`/?ip=&history=1` | 浏览器直连（useIpLine）+ /api/bgp 服务端转发 |
| global.duoniu.cc（python :8898） | geo 查询 | Nuxt 代码中**未见引用**（疑似 PHP 43 主库侧使用） |
| api.duoniu.cc（CF Worker） | /myip /cfinfo /ipdetail /geo /ping /dns /bulk /links /status /aicheck /asn /whois | nginx 反代 4 条统计路由（/api/ip/stats 等） |
| 其他 | ipify.org、dns.google、cloudflare-dns、doh.pub、hackertarget、rdap.org、RIPEstat、ip-api | 前端探测或 Nuxt 服务端 |

---

# 4. 数据流

## 4.1 IP 主查询（首页/纯净度/历史，核心链路）

```
浏览器（index.vue 等）
  ├─ $fetch https://ipquery.duoniu.cc/?ip=xxx        ← 直连 Provider（违反规则六）
  ├─ queryIpLine/queryAsnLine/queryIpHistory
  │     → https://ipf.duoniu.cc/…                    ← 直连 Provider
  ├─ $fetch /api/rdomains/{ip}  → Nuxt → hackertarget
  ├─ $fetch /api/location-history/{ip} → Nuxt → 本地 JSON（当月缺失异步补采 ipquery）
  └─ $fetch /api/latency → Nuxt → 18 节点探测
```

## 4.2 工具链路（SimpleTool 六合一）

```
浏览器 → /api/{bgp|dns|ping|trace|whois}（Nuxt 服务端）
  bgp  → ipf.duoniu.cc（RIPEstat+PeeringDB）
  dns  → 阿里 DoH → Google DoH
  ping → 本机 HEAD 探测
  trace→ 本机 tracepath
  whois→ rdap.org
```

## 4.3 采集链路（ping0 式 peer 采集，**当前已断**）

```
浏览器 peer.min.js（混淆版）→ POST /api/peer → MySQL webrtc_peers
                                       ↑ DB_PASS 缺失 → ER_ACCESS_DENIED → ok:false 静默丢弃
浏览器 track.client.ts 插件 → POST ipquery.duoniu.cc/track（PHP 侧，独立于 MySQL）
```

## 4.4 历史快照链路

```
cron 03:15 scripts/location-history.mjs（22 个流行 IP）
  → ipquery.duoniu.cc 拉当月归属 → data/location-history.json（按月 upsert）
```

---

# 5. 存在问题

1. **admin-api 串线（高危）**：duoniu.conf `location /admin-api/ → 127.0.0.1:3001/api/`，而 3001 端口当前跑的是 **ipcesu 测试站 Next.js**（/proc/271126/cwd = /www/wwwroot/ipcesu.com，2026-08-14 启动）。duoniu 管理后台没有独立后端进程（全站 systemd 仅 duoniu-preview），后台 API 已断或打到别的项目的 API。
2. **webrtc_peers 落库失败**：无 .env、systemd 单元未配置 DB_PASS（nuxt.config 默认空密码）→ nohup.out 已出现 6 条 `[peer] store failed`，peer 采集数据全部静默丢失。
3. **前端直连 Provider（违反规则六）**：index/env/purity/history 浏览器直连 ipquery.duoniu.cc；useIpLine 直连 ipf.duoniu.cc。依赖 PHP 8899 的 CORS 放行，前端把 Provider 结构类型写死（IpRecord 对齐 ipapi.is）。
4. **首页 1229 行堆叠式长页（违反规则八/九）**：hero + 基本信息/位置/ASN/企业/纯净度/风控/线路/历史/延迟/拓扑/反查全部一次性堆叠渲染，无 Overview/Network/Location/Risk/History/Routing/Raw Data 七 Tab。
5. **无统一搜索（违反规则四/五）**：搜索框仅按 IP 查询；输入 AS13335、1.1.1.0/24、域名不会做类型识别与路由。
6. **无 git 仓库**：duoniu-new-vue 无 .git，19 个 .bak 文件 + 20+ tar.gz 是唯一"版本管理"，事故回滚靠人工找备份。
7. **/cidr/[q] 动态路由缺失**：线上日志出现 `Vue Router warn: No match found for "/cidr/60.28.64.0/21"`，存在生成错误路径的内链或外部入口。
8. **CF Worker duoniu-api 配置为模板占位**：wrangler.toml 的 KV/D1 是 `YOUR_KV_NAMESPACE_ID` 占位（/status 200 只能证明 Worker 在线，D1/KV 绑定状态未知）；且与 PHP 8899 职责重叠（都有 ip 统计）。
9. **容灾缺位（违反规则七）**：12 个内部 API 仅 dns.ts 有双源；asn-detail/rdomains/ping/whois 单源无 fallback；错误直接透出英文 `fetch failed`/`HTTP 4xx`，无"数据暂时不可用/正在显示缓存"统一文案。
10. **两套后端树并存**：Nuxt 在 duoniu.cc/，数据服务在 duoniu/（PHP 43 主库 + 2 个 python），global.duoniu.cc 在 Nuxt 代码中零引用，职责边界模糊。
11. **preview.duoniu.cc 旧预览域**：conf 里仍有 AI bot 拦截（2026-08-15 只改了主域），且与主域规则分叉。
12. **无测试/无 lint 门禁**：无任何测试框架；代码改动靠手工冒烟。
13. **构建产物混淆**：postbuild 整包混淆使前端线上排障困难（Source Map 关闭）。
14. **孤儿进程**：2 个 2026-08-13 挂死的 nuxt build 进程待清理（需授权）。

---

# 6. 与升级规则的差距

| 规则章节 | 要求 | 现状 | 差距评估 |
|---|---|---|---|
| §1 技术环境 | Nuxt3+Vue3+TS+SSR / nginx+node20+systemd | ✅ 完全符合（duoniu-preview/3901） | 无 |
| §2 核心目标 | 禁止"工具集合+独立小工具"；统一 Network Intelligence Engine；对象自动识别→统一引擎→结构化情报→实体关系 | ❌ 现状正是"IP 查询工具集合"：各页独立查询、无实体关系层、无结构化情报概念 | 大 |
| §3 信息架构 | 5 大栏目（IP/Network/Diagnostics/Developer/Information） | 页面覆盖 ~70%（大部分工具页存在）；但导航是平铺工具菜单，无栏目层级；Developer 仅文档页 | 中 |
| §4 统一搜索 | Universal Search + 类型识别（IPv4/IPv6/ASN/CIDR/Domain） | ❌ 首页搜索仅 IP；无 Parser | 大 |
| §5 统一查询架构 | Search Input→Query Parser→Query Router→Intelligence Service→Data Provider→DB/Cache→Display | ❌ 全链缺失：页面直接各自 $fetch；SimpleTool 六合一但仍是"每页一份查询逻辑"变体 | 大 |
| §6 数据层 | 前端禁止直连第三方/Provider；必须 Frontend→DuoNiu API→Data Service→Provider Manager | ❌ 5 个页面直连 Provider；Nuxt server/api 只有 12 条散路由，无 Provider Manager 抽象 | 大 |
| §7 容灾 | Primary→Fallback→Redis Cache→DB Snapshot；统一降级文案 | ⚠ 仅 dns 双源；无 Redis、无快照层；错误文案不统一 | 中 |
| §8 首页 | 第一屏 6 要素（IP/Country/ISP/ASN/Type/Risk）+ 七 Tab | ❌ 第一屏是静态 hero 文案；结果是堆叠长列表（一次几十字段） | 大 |
| §9 代码架构 | features/{ip,asn,bgp,dns,cidr,history,diagnostics} + services + types；单页≤1000 行；组件不直接请求 API；页面不重复逻辑 | ❌ 无 features/、无 services/、无 types/；index.vue 1229 行；SimpleTool 组件内直接 $fetch；queryIpLine 在 index/route/asn 三处重复 | 大 |

**结论**：技术底座（§1）达标；产品形态与代码架构（§2/§4/§5/§6/§8/§9）与规则差距大——现状是"页面齐全但互不关联的工具集合"，与"Network Intelligence Engine"目标属于架构级重构，而非局部修补。

---

# 7. 改造风险

| # | 风险 | 等级 | 说明与建议 |
|---|---|---|---|
| 1 | **Provider 契约迁移** | 高 | 前端直连改走 DuoNiu API 需新建 BFF/Provider 层并重写 5 个页面的取数逻辑；ipquery 返回结构（ipapi.is 对齐）被前端类型写死，迁移期需兼容 |
| 2 | **PHP 43 主库是黑盒** | 高 | /www/wwwroot/duoniu/ipquery 无版本管理、接口（含 /score /track）无文档；重构数据层前必须先盘点其全部端点与调用方（含老站 duoniu-vue 与 vps 子系统是否也依赖它） |
| 3 | **无 git、无测试** | 高 | 大改前必须先 `git init` 建基线 + 快照当前 .output；否则重构中途无法回滚、无法 diff |
| 4 | **混淆构建管线** | 中 | postbuild 混淆依赖 javascript-obfuscator 与固定脚本；改造期间混淆会放大排障成本，建议开发期关闭、上线前开启 |
| 5 | **MySQL 凭据治理** | 中 | DB_PASS 缺失导致 peer 采集已丢数据；恢复需要用户在宝塔确认 ipquery 库口令并配置 systemd Environment 或 .env |
| 6 | **admin 端口冲突** | 中 | 恢复 duoniu 管理后台需另起端口/进程（3001 已被 ipcesu test 占用），改 conf 后需复查宝塔覆盖问题 |
| 7 | **双后端树割裂** | 中 | ipf/global 在旧树 /www/wwwroot/duoniu/ 下（systemd 托管，无 Restart 声明需核实）；迁移统一引擎时是否接管/复用需决策 |
| 8 | **多域规则分叉** | 低 | preview.duoniu.cc / shop.duoniu.cc.bak 等残留 conf 与主域规则不一致（如 AI 拦截），改版时需一并处理或下线 |
| 9 | **宝塔 conf 覆盖** | 低 | 手工改 nginx 会被宝塔保存操作覆盖（历史多次），改版涉及 conf 变更时需备份并复查 |
| 10 | **SSR 与阻塞探测** | 低 | ping/trace/latency 是耗时阻塞型工具（最长 30s），统一到服务端后需异步化/队列化，避免拖垮 SSR 首屏 |
| 11 | **数据积累断层** | 低 | location-history 只从 22 个流行 IP 起步且依赖 ipquery；实体关系/历史库要从零积累，改版初期"情报"页面内容稀薄 |

## 附：改造前建议的基线动作（未执行，仅建议）

1. `git init` 建仓库并提交当前源码基线；确认 duoniu/ipquery、duoniu/ipf、duoniu/global 是否纳入
2. 备份并盘点 MySQL（需用户提供/授权凭据）、location-history.json、.cache
3. 决策 PHP 43 主库的保留/接管/替换（影响 Provider 层设计）
4. 清理：19 个 .bak、20+ tar.gz、2 个孤儿 build 进程、preview 域
5. 确认 CF Worker duoniu-api 的 KV/D1 真实部署状态（wrangler 实际部署的 toml）
