# SiteIntel 多源自动发现引擎 — Phase 0 + Cloudflare Radar 阶段验收报告

> 日期：2026-08-17 ｜ 分支：`multi-source/phase0-cfradar` @ e32bd93（已推送 GitHub 私有仓库）
> 生产：154.204.176.66:3003（systemd `siteintel`，重启 07:00:12 UTC）
> 依据：用户 2026-08-17 批准 13 条 + 指令 §21-§25。**本阶段完成后暂停，不自动进入第二阶段。**

---

## 一、执行摘要

本轮交付 = 用户批准范围内的最小闭环：

```
Phase 0（monitor 死循环修复）＋ Schema 扩展 ＋ SeedSource 抽象（Cloudflare Radar 首发）
＋ 轻量 Probe ＋ 透明评分 ＋ 持续消费调度（15s 节拍）＋ 指数退避 ＋ 种子日预算 50
```

**生产验证全部按用户 §12 清单逐项通过（见第三节）**：候选增加 ✓、Probe 正常 ✓、可访问率可算 ✓、Target 真实增加 ✓、调查真实完成 ✓、Entity/Relationship 真实增加 ✓、DB 增长合理 ✓、资源正常 ✓。

**一个待办（不阻塞验收）**：Cloudflare Radar API 实测必须鉴权（`9106 Missing X-Auth-Key/X-Auth-Email or Authorization`）——`CLOUDFLARE_API_TOKEN` 按用户选择由用户稍后自行添加（dash.cloudflare.com → My Profile → API Tokens → Create Token → Cloudflare Radar - Read），写入 `.env` 后重启即激活种子导入；代码已按该路径优雅降级（日志明确、调度不中断）。

---

## 二、部署清单（与回滚点）

| 项 | 值 |
|---|---|
| 代码提交 | `e32bd93`（分支 multi-source/phase0-cfradar，15 文件 +1216/−65，已推送 GitHub） |
| Migration | `20260817_multi_source_candidate_pool`（ALTER Target + DiscoveryCandidate 加 18 列/3 索引，`prisma migrate deploy` 成功） |
| .env 变更 | `DISCOVER_DAILY_CAP=20→50`；新增 `DISCOVER_TICK_MS=15000`、`SEED_DAILY_BATCH=50`（旧 .env 备份为 `.env.bak.phase0`） |
| 数据库备份 | `/tmp/siteintel_pre_phase0.sql`（26.6MB，migration 前 pg_dump） |
| 服务 | `systemctl restart siteintel` ✓（Next.js 16.3.1，启动日志确认新调度器 armed） |
| 生产代码变更文件 | monitoring.ts / domain.ts / discovery/{candidates,collector,probe,score}.ts / discovery/sources/* / schema.prisma / migrations/* |

**回滚（完全对称）**：
```bash
systemctl stop siteintel
git -C /www/wwwroot/siteintel.cc checkout -- src/lib/monitoring.ts src/lib/domain.ts \
  src/lib/discovery/candidates.ts src/lib/discovery/collector.ts prisma/schema.prisma
rm -f /www/wwwroot/siteintel.cc/src/lib/discovery/probe.ts /www/wwwroot/siteintel.cc/src/lib/discovery/score.ts
rm -rf /www/wwwroot/siteintel.cc/src/lib/discovery/sources /www/wwwroot/siteintel.cc/prisma/migrations/20260817_multi_source_candidate_pool
cp .env.bak.phase0 .env
sudo -u postgres psql -d siteintel -f /tmp/siteintel_pre_phase0.sql   # 数据回滚（仅需要时）
export PATH=/root/.nvm/versions/node/v20.20.0/bin:$PATH && npm run build && systemctl restart siteintel
```
> 代码回滚无需数据回滚（加列不破坏旧代码）；数据回滚仅在新数据不可接受时执行。

---

## 三、生产数据验证（对照用户 §12 验收清单，全部实测）

| # | 验证项 | 结果（实测证据） | PASS |
|---|---|---|---|
| 1 | Candidate 是否增加 | 30 分钟新增 7 候选（6 cname + 1 redirect）；候选池 926→933+；内部闭环再产出（api.t.sina.com.cn +1、api.weibo.com +2、company.verified.weibo.com +1 逐条日志） | ✅ |
| 2 | Probe 是否正常 | 27 ok / 2 fail / 914 待探；ok 全部带状态码（200/404/405/403）；fail 带 8s 超时与退避（`nextRetryAt` 已设、`probeFailureCount=1`） | ✅ |
| 3 | 可访问率 | 27/(27+2) = 93% 已探样本可访问（含 403/404 降权样本；不可达=超时 2 条，按设计重探） | ✅ |
| 4 | Target 是否真实增加 | 45 分钟新增 28 Target（weibo.cn、krcom.cn、api.* 等全部真实入库） | ✅ |
| 5 | Investigation 是否真实完成 | 连续消费 7/50 → 33/50（07:00-07:14）；日志逐条 `analyzed — N/50 today`；今日调查全 completed/partial | ✅ |
| 6 | Entity/Relationship 是否真实增加 | 45 分钟新增 Entity 72、Relationship 278（EntityRelationship 45m=278） | ✅ |
| 7 | DB 增长是否合理 | 数据库 39MB（26.6MB 备份基线，含新列/新索引）；磁盘 48G 空闲 | ✅ |
| 8 | API/服务器资源是否正常 | load 0.02-0.09（4 核）；内存 3.2/7.9GB；首页 200/139ms；零运行时错误日志 | ✅ |

## 四、核心行为验证

### 4.1 Phase 0 — monitor 死循环修复（用户批准第 1 条）
- 修复：投递失败分支（monitoring.ts:174-192）不再"不动 lastRunAt"，改为 `lastRunAt` 推进 + `lastNotifiedAt` 保留（警报语义不丢，下次到期重试投递）；事件/洞察逻辑零改动。
- 实测：07:01:13 运行后日志 `alert delivery failed on all channels — alert retained, schedule advanced`；**07:30 节拍零 monitor 活动**；wordpress.org 今日调查数 = **1**（修复前 ~48 次/天）。✅

### 4.2 Probe 预检（§7）+ 降权不删（§8）
- 实测日志：`probed weibo.cn → http_ok 200 229ms`、`probed admin.event.weibo.com → timeout 8000ms`、`probed iapps.gslb.sinaedge.com → http_error 403 5ms`。
- 403/404 可达→降权可分析；超时→fail+退避 1h 重探；无任何候选被删除。✅

### 4.3 透明可解释评分（用户批准第 10 条）
- 实测：`weibo.cn score=195, 6 条原因，首条 "HTTP 200 正常 (+100)"`；`api.weibo.com score=149, 8 条原因`（子域降权+重复惩罚生效）；`company.admin.weibo.com`（超时）评分含降权原因。任何候选可回答"为什么排这里"。✅

### 4.4 持续消费（用户批准第 4/13 条——保守起步，不冲理论极限）
- 15s 节拍 + 日预算 50（未放大）；实测 3.5 分钟 12 次分析，负载 0.03；预算耗尽时静默（5 分钟一次日志）。
- 用户输入路径零改动（不排队，天然最高优先）。✅

### 4.5 主域优先 + 独立网站口径（用户批准第 8/11 条）
- `classifyDomain` 生效：新候选全部带 domainType（7 个全为 subdomain——当前消化的是 weibo 家族子域；apex 在种子导入后进入）。
- **Target.isApex 回填实测**：weibo.cn→`t`、krcom.cn→`t`；api.chaohua.weibo.cn 等 11 个子域→`f`。KPI-1"独立网站"口径字段已就绪。✅

### 4.6 失败自愈（用户批准第 9 条）
- 分析失败：`failureCount`+`lastFailureReason`+`nextRetryAt`（1h→4h→12h→24h→72h→168h，7 次后 skipped 保留记录）；预检失败独立计数。✅

### 4.7 外部源约束（用户批准第 2/7 条）
- 种子只入 Candidate Pool（`addExternalCandidate` 三重去重，绝不建 Target）；日预算 `SEED_DAILY_BATCH=50` 记账实测无（待 token）。✅（代码路径已部署，行为验证待 token）

### 4.8 兼容性（§19）
- 现有 7 步 pipeline、Entity/Relationship/Snapshot/Event/Insight 零改动；admin/seo 页兼容；251/251 测试 + typecheck + build 全绿（本地）；生产 build 成功。✅

---

## 五、未决事项与风险观察

| # | 事项 | 状态/建议 |
|---|---|---|
| 1 | **CF API Token** | 用户稍后自行添加 `CLOUDFLARE_API_TOKEN` 到 .env + restart 即激活种子导入；激活后需补验：种子候选入库、来源多样性计数、预算记账 |
| 2 | 918 存量候选待探 | 以 ~1 探/30s 消化（≈2880/天）；全部探完后再评估节拍调优 |
| 3 | npm install 的 postinstall 报错（`Cannot read properties of null (reading 'matches')`） | 无实际影响（显式 `prisma generate` 成功、build 通过）；建议下阶段排查（疑似 npm 10 与无 lockfile 的交互） |
| 4 | 服务器 git 无 remote | 部署走 scp；建议下阶段为服务器仓库补 remote 以便直接 pull 回滚 |
| 5 | `radar api returned 400` vs 直接 curl `9106` | 同为鉴权缺失的不同表示；token 后统一消除 |

---

## 六、结论与下一阶段（等待确认）

- **本阶段验收结论：PASS**（§12 全部实测项通过；唯一待办 = CF Token，用户已明确稍后自添）。
- 按用户指令**暂停**，不自动进入第二阶段（Majestic Million）。
- 下一步候选（用户确认后执行）：① 用户添加 token 后补验种子导入；② 观察 24-72h 稳定运行（预算消耗曲线、退避重探、资源）后再决定放大 `DISCOVER_DAILY_CAP`；③ 第二阶段 Majestic Million SeedSource。
