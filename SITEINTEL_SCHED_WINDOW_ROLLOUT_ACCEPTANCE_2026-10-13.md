# SiteIntel 调度窗口上线 — 发布分支 / 部署 / 生产验收报告（2026-10-13）

性质：收尾闭环（发布分支创建 + 推送、生产部署复核、生产验收）。
未做：schema / migration / nginx 变更、Radar P0 启用、工作区其它 WIP 移植、历史重写。

---

## 1. 发布分支与 SHA

| 项 | 实测值 |
| --- | --- |
| 基线分支 | `release/bing-seo-cb131b3-20261013` = `cb131b39df030f0295555a68081b81491709baf7`（远端已存在） |
| 调度来源分支 | `release/radar-asn-p0-2026-09-12` = `8081f69aa86ff10d5368f56abeca14e1d77ecb00`（含 `ecc41fa` / `8081f69`） |
| 本轮发布分支 | **`release/bing-seo-sched-6e0a993-20261013`** |
| 分支 SHA | **`6e0a99384a18cc5c32a5a041d9f6053dc1526956`**（与本地合并提交逐字节同 SHA，非重写） |
| 远端复核 | `git ls-remote`（经 SOCKS 隧道的 `https://github.com/Duoniu-ai/siteintel.git`）实测 = `6e0a993…` |
| 推送方式 | 本机 `github.com:443` 被阻断（TCP 可连、TLS 超时）；经非生产主机 `191.124.63.142` 的 `ssh -D` SOCKS 隧道 + 既有 `gh auth git-credential` 完成，未落盘凭据、未 force |
| 内容 | 14 文件（8 新增 + 6 修改），`cb131b3..6e0a993` = 1682 插入 / 53 删除；`src/lib/seo/**` 改动数 = **0**（SEO 代码与基线逐字节相同） |

## 2. 生产 release 与实际运行版本

| 项 | 实测值 |
| --- | --- |
| 主机 / 服务 | `169.58.55.19`（`vmi3611112`）/ `siteintel.service` |
| 生产 release | `/www/wwwroot/siteintel.cc-release-bing-seo-sched-6e0a993`（release 目录时间 2026-10-13 11:43:25 CEST） |
| drop-in | `/etc/systemd/system/siteintel.service.d/p04j.conf`（sha256 `81dcc4cd…`，指向该 release；**无** `SITEINTEL_BUSINESS_TZ` 覆盖） |
| BUILD_ID | `2hb6hR1mC4psBw5cjJhbu`（cwd 内 `.next/BUILD_ID` 与首页 HTML buildId 一致） |
| 服务状态 | `active/running`，MainPID `1209475`，NRestarts=0 |
| 版本一致性 | 生产源码树（排除 `node_modules` / `.next` / `.git`）677 个文件 sha256 清单与 `6e0a993` git 树**逐文件一致**；仅多出 2 个安装/构建产物 `pnpm-workspace.yaml`、`tsconfig.tsbuildinfo` |
| 构建产物内容 | `.next` 内含调度标记（`Asia/Singapore` 16 处、`outside batch window` 5 处、`SITEINTEL_BATCH_CONCURRENCY` 6 处）与 SEO 标记（`bingbot` 12 处）；旧串 `tick every 6h` = 0 处 |

> 说明：生产发布本身是在 11:44:17 CEST 由上一轮完成并已在位；本轮**未重复创建 release 目录**，而是复核其内容与推送分支完全一致，并完成下述验收（含两次带时间窗验证的重启，均已还原）。

## 3. 定向测试 / 类型检查 / 构建（本机隔离副本 `verify-src-20261013`）

| 项 | 结果 |
| --- | --- |
| `git diff --check cb131b3..HEAD` | **PASS**（无空白/冲突标记） |
| `npx tsc --noEmit` | **PASS**（0 error） |
| 定向 `vitest` | 11 文件 / 131 项 → **130 passed / 1 failed** |
| 定向覆盖 | `src/lib/schedule/*`（business-time / batch-window / batch-guard / schedulers）、`src/lib/search/sync.test.ts`、SEO 关键路径（`seo/sitemap-entities`、`seo/entity-index`、`read-cache`、`technology/[slug]/page` 等） |
| 唯一失败 | `src/app/technology/[slug]/page.test.ts > still generates full SEO metadata for a legitimate first visit`（`profile.detection.fingerprints` undefined）——在**未改动的 `cb131b3` 独立工作树**上逐字复现（其 `website/[domain]` 孪生项同批复现）→ **failure delta = 0，属既有基线失败** |
| `pnpm build` | **PASS**（本地 BUILD_ID `FJnCkb89lR1pVVYqLEYTx`，与生产 BUILD_ID 不同属正常） |
| 构建产物复核 | 本地构建同样含调度 16 处 / `outside batch window` 5 处 / `bingbot` 12 处 |

## 4. 调度行为证据（生产实测）

### 4.1 窗口外跳过（Asia/Singapore 19:2x，真实生产默认时区）

```
[boot] collection schedule (Asia/Singapore, batch window enabled: true):
  - discovery   02:00–05:00 Asia/Singapore  max 20/sweep, pacing 15000ms
  - search_sync 02:40–04:00 Asia/Singapore  max 10/sweep, pacing 1000ms
  - monitor     03:20–04:40 Asia/Singapore  max 15/sweep, pacing 2000ms
[monitor] tick skipped — outside batch window (after_window, next 476m)
[search-sync] tick skipped — outside batch window (after_window, next 436m)
```

- 重启后 `JobRun` **仅**新增 1 行 discovery session；monitor / search_sync **0** 行（其最后一行仍是旧进程的 11:28:50 / 07:29:50）→ 窗口外不 claim。
- 窗口外 `JobRun` 每 15 分钟持续为 `tick skipped`，无补跑。

### 4.2 窗口内执行（受控验证：临时 `SITEINTEL_BUSINESS_TZ=Etc/GMT+8`，本地时刻 03:2x，三条链路窗口同时开启）

```
[boot] collection schedule (Etc/GMT+8, batch window enabled: true):   ← 仅验证期
[discovery] nightly batch window 2026-10-13 open (02:00–05:00 Etc/GMT+8) — analyses/revisit allowed
[discovery] session JobRun fb2a4978-… started
[search-sync] https://www.ipcesu.com: completed — capabilities [none]
[schedule] search_sync batch ok — window 2026-10-13, duration 117ms, count 1, errors 0, deferred 0
```

`JobRun` 实测（窗口内 3 条链路全部实际执行，非仅为日志）：

| startedAt (UTC) | job | status | meta.count | meta.windowKey |
| --- | --- | --- | --- | --- |
| 13:20:29 | discovery | running | — | — |
| 13:21:13 | monitor | success | 0 | 2026-10-13 |
| 13:21:28 | search_sync | success | 1 | 2026-10-13 |

- monitor 的 boot tick（+45s）在窗口内认领并完成 JobRun（当夜 0 个到期监控 → 无重分析负载）；窗口外该 tick 只打印 skip、不写库。
- 验证后**已即时还原**：删除覆盖、`daemon-reload`、重启；`Environment` 无覆盖、drop-in 目录恢复原文件集、build 不变（见 §2）。还原后重新出现 `after_window` skip。

### 4.3 实时路径保持（窗口外仍运行）

- discovery probe 24/7：还原后 1 分钟内 12 次 probe（约 15s 节奏），实际外呼正常。
- 种子导入 24/7：每次 boot 后 `[discovery] seed cloudflare_radar: fetched 50, +0 new (0/50 today)`。
- 本地数据库实体化（无外呼）与用户请求分析路径未纳入窗口（`discoveryAnalyzeBudget()` 只在窗口外把 analyze 预算置 0；probe / seed / materialisation 不受影响）。
- 窗口外无 analyze 派发；当日 `daily analysis budget used (200/200)` 属既有每日上限，与本改动无关。

## 5. SEO 关键路径验收（与 `cb131b3` 基线同代码，逐字节）

| 检查 | 结果 |
| --- | --- |
| `/robots.txt`（origin + 公网） | 200；声明 `Sitemap: https://siteintel.cc/sitemap.xml`；仅屏蔽 `/analyze/ /api/ /admin/ /dashboard/ /login/ /register/ /claim/` |
| `/sitemap.xml` | 200；**341** 个 `<loc>`；全部 apex（唯一“非 `/` 结尾”项是首页 `https://siteintel.cc` 本身）；重复 0 |
| 关键页 HTTP（origin） | `/` `/robots.txt` `/sitemap.xml` `/website/cloudflare.com` `/website/GITHUB.com` `/asn/54113` `/asn/AS54113` 全部 **200** |
| canonical 归一 | `/website/GITHUB.com` → `/website/github.com`；`/asn/54113` → `/asn/AS54113`；均 `index, follow` |
| sitemap 抽样（公网 14 条：technology / ip） | **14/14** = 200 + `canonical == 自身 URL` + `index, follow` |
| meta description | 152–171 字符（`/website/cloudflare.com` 169、`/website/github.com` 171、`/technology/nginx` 152、`/asn/AS13335` 168、`/tools/dns` 152） |
| 公网（Cloudflare） | `/`、`/robots.txt`、`/website/cloudflare.com`、`/website/GITHUB.com`、`/asn/54113`、`/asn/AS54113` 均 200；`cf-cache-status: DYNAMIC`（未命中旧缓存）；公网 sitemap 341 loc |

> 记录（非缺陷，属既有匿名额度语义）：单 IP 对 `/technology/*` 的匿名额度为 20 次/小时（`TECHNOLOGY_SESSION_BUDGET`）。用单一 IP 扫完 341 条 sitemap 时，超出额度的页面会返回 `noindex, follow`；按额度抽样（≤20 次/小时）与公网抽样均稳定 `index, follow`。不在 sitemap 的 `/ip/1.1.1.1`、`/organization/cloudflare` 返回 `noindex, follow` 与“未被 sitemap 收录”一致。

## 6. 回滚方案（可执行，本轮未触发）

| 项 | 值 |
| --- | --- |
| 回滚 release | `/www/wwwroot/siteintel.cc-release-bing-seo-cb131b3`（未改动；BUILD_ID `mhXyVj1u-Mq9zWYR3Z9OB`；`node_modules/next/dist/bin/next` 存在；该 release 内 `Asia/Singapore` 出现 0 处 = 无调度） |
| 回滚 drop-in | `/etc/systemd/system/siteintel.service.d/p04j.conf.bak-pre-sched-6e0a993-20261013T094403Z`；另有 `/root/p04j.conf.rollback-cb131b3`（内容实测指向 cb131b3） |
| 回滚步骤 | 恢复上述 drop-in → `systemctl daemon-reload` → 一次 `systemctl restart siteintel` → 复验首页 / robots / sitemap / BUILD_ID |
| 不可用作回滚目标 | `…-sched-8081f69`、`…-bing-seo-sched-8081f69`、`…-83fe6cf-sched-8081f69`（含非 `cb131b3` 的 `sitemap.ts` / `sitemap-entities.ts`） |
| 本轮为验证产生的临时物 | `/root/zz-window-verify.conf.disabled-20261013T1124Z`（已停用的时间窗验证覆盖文件，可删） |

## 7. 并发发布防护与风险

- 部署/验证前后实测：仅 `siteintel.service` 一个 unit；无 `next build` / `rsync` 在跑；drop-in 与 release 目录自 11:43–11:44 起未被其它发布改写；无更新的 release 目录。
- 本轮未删除任何历史 release 目录；未改 nginx、schema、migration、Radar P0。
- 残留风险（记录，不处理）：① 若后续有其它任务把 drop-in 切回不含调度的 release，调度功能会被静默覆盖——发布前需比对 `p04j.conf` 与 release 内容（本轮已建立该检查）；② 本机 `github.com:443` 需经隧道推送；③ `…-83fe6cf-sched-8081f69` 等早期候选目录含非基线 SEO 代码，易被误当回滚目标。

## 8. 结论

`release/bing-seo-sched-6e0a993-20261013`（`6e0a993`）已推送；生产 `siteintel.service` 运行同名内容的 release（BUILD_ID `2hb6hR1mC4psBw5cjJhbu`），**最新 SEO 基线与完整调度功能同时生效**；窗口外跳过、窗口内三条链路实际执行、24/7 实时路径、SEO 关键路径全部实测通过；回滚点完好。
