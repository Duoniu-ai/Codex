# SiteIntel — SEED_DAILY_BATCH 10 → 50 配置调整报告

> 日期：2026-08-17 ｜ 生产：154.204.176.66:3003（systemd `siteintel`）
> 任务范围：**仅调整每日 Seed 批次上限 10 → 50**。零代码修改、零功能开发、未触碰评分/Probe/Investigation/去重/预算/重试/自动繁衍/DB 结构。

---

## 一、修改内容

| 项 | 值 |
|---|---|
| 修改文件 | `/www/wwwroot/siteintel.cc/.env`（唯一生效位置，审计确认） |
| 修改行 | `SEED_DAILY_BATCH=10` → `SEED_DAILY_BATCH=50` |
| 修改方式 | `sed -i 's/^SEED_DAILY_BATCH=10$/SEED_DAILY_BATCH=50/'`（行首行尾锚定，无其他行受影响） |
| 修改前备份 | `.env.bak.seed-batch-10-20260817`（449B，含 10 版配置） |
| 代码修改 | **无**（未部署任何代码，未重复部署历史代码） |
| Schema / Migration | **无** |

**审计确认**：`SEED_DAILY_BATCH` 全服务器仅存在于 `.env` 一处（grep 键清单 10 个键，无重复定义、无其他覆盖点）；代码侧从环境加载（启动日志 `collector armed` 实锤）。

## 二、修改前基线（2026-08-17 08:32 UTC，修改前快照）

| 项 | 值 |
|---|---|
| systemd | active (running)，07:34:27 起（57min），PID 695313，Memory 110.9M |
| CPU / Load | 0.01 / 0.03 / 0.04（4 核） |
| 内存 | 3.2Gi / 7.8Gi（available 4.2Gi） |
| 磁盘 | 50G / 97G（48G 空闲，52%） |
| 数据库 | 41 MB |
| 队列 | pending 856 / analyzed 113 |
| Seed 现状 | `source='cloudflare_radar'` 共 7 行（rank 16/19/21/22/23/24/27） |
| Seed 预算 | `7/10 today`（10 预算已耗 7） |
| 分析预算 | 今日已耗 60（测试期临时 cap 60 遗留），cap 已恢复 50 |

## 三、执行与生效

1. 备份 `.env` → `.env.bak.seed-batch-10-20260817`
2. `sed` 精确替换 → `grep` 验证文件值 = `SEED_DAILY_BATCH=50`
3. `systemctl restart siteintel`（唯一一次正常重启，08:32:19 生效，新 PID 700473）

## 四、配置验证（SEED_DAILY_BATCH=50 实际生效）

**① 文件层**：`.env` 中 `SEED_DAILY_BATCH=50`（grep 实证）

**② 代码加载层**（重启后启动日志）：
```
Aug 17 08:32:20 node[700473]: [discovery] collector armed: tick 15000ms, daily cap 50, seed batch 50/day, retry backoff 1h→168h (max 7 failures)
```
→ `seed batch 50/day` 实锤代码已加载新值（对比修改前 07:34:28 日志 `seed batch 10/day`）。

**③ 行为层**（重启后自动窗口 08:32:36）：
```
[discovery] seed cloudflare_radar: fetched 43, +24 new (31/50 today)
```
→ 预算记账按"入池新增"计数：31 = 7（修改前已入池）+ 24（本次新增），与 50 预算完全对账，**未突破预算**（剩余 19 今日可用）。

## 五、回归验证（未绕过去重 / 未重复导入 / 预算受控）

| 验证项 | 结果 | 证据 |
|---|---|---|
| 不立即重复导入已处理 Seed | ✅ | `fetched 43` 中 19 条被去重/过滤挡掉；`cf_radar` 行 7 → 31，只 +24 new 无重复；池内重复域计数 **0** |
| 已在池候选 → 信号累计（不新建行） | ✅ | apple-dns.net `sourceCount` 2→3、akadns.net/aaplimg.com 1→2（8:32 窗口再命中，按设计累计信号） |
| 去重逻辑未被绕过 | ✅ | `dup_domains=0`；Target 去重路径正常（19 条命中未入池） |
| 预算控制未突破 | ✅ | `31/50 today`（=7 旧 +24 新），fetched 43 中仅 24 记账，50 上限未被突破 |
| 基建预过滤仍生效 | ✅ | 报告 §3.2 名单 21 个服务商域（google.com/cloudflare.com/microsoft.com/…/sentry.io）实测在池中 = **0** |
| 未立即人工导入 50 | ✅ | 仅重启触发的自动窗口（按 6h 间隔 + 预算 31/50 记账），无人工导入 |
| 新增错误日志 | ✅ 零 | `\] error|ERROR|ERR_` 计数 0；唯一 "error" 字面命中为 probe 业务状态 `http_error 404`（非日志错误） |

## 六、健康检查（修改后 08:36 UTC）

| 项 | 值 | 判定 |
|---|---|---|
| `systemctl status siteintel` | active (running)，08:32:19 起，NRestarts 正常，Memory 74.1M | ✅ |
| 最近错误日志 | 0（重启后零 error/fatal/uncaught） | ✅ |
| CPU / Load | 0.13 / 0.08 / 0.06（4 核） | ✅ |
| 内存 | 3.1Gi / 7.8Gi（available 4.3Gi） | ✅ |
| 磁盘 | 50G / 97G（48G 空闲） | ✅ |
| 数据库 | 41 MB | ✅ |
| Seed 数量 / 状态 | 31 行（7 analyzed + 24 pending，rank 16-71 连续消费中：ui.com http_ok 200、tiktokcdn/cloudfront dns_fail 降权保留等） | ✅ |
| 生效配置 | `SEED_DAILY_BATCH=50`（三层层级实证，见第四节） | ✅ |

**服务健康状态：PASS**

## 七、Git 状态

- **本次零代码修改**：唯一变更为服务器 `.env`（属于 runtime 配置，`.gitignore` 忽略项，历史上从未入库，服务器 git 无 remote 亦不涉及）。
- **无需提交**，无需要 merge / push 的内容，GitHub 基线（main @ c74dfd1）不受影响。
- 服务器 git 工作树状态与修改前一致（.env 为 ignore 项，不产生 git 变更）。

## 八、观察项（如实记录，不扩大任务范围）

1. **报告 §3.2 名单措辞偏差**：Seed 测试报告将 tiktokv.com / tiktokcdn.com / cloudfront.net 列为"被预过滤"，实测本次预算 50 窗口下这 3 个域**进入池中**（rank 29/30/34）——实际原因：7:24 灰度窗口预算 10 只取到 rank 27 即耗尽，rank≥29 从未被取到，被误记为"被过滤"。预过滤规则本身工作正常（21 名单域 0 在池），此为其既存行为，非本次修改引入。
2. **分析预算 `60/50`**：今日已耗 60（测试期临时 cap 60 遗留）> 当前 cap 50，预算耗尽静默（5 分钟一条日志），明日 UTC 日重置自动回归 50/天。与 seed 预算相互独立，不阻塞 Seed 50/天。
3. **部分新候选 dns_fail**：cloudfront.net / msftncsi.com 等从生产服务器解析失败（既有现象，与 apple-dns.net 同因），按"降权不删"保留 + 指数退避。

## 九、结论

- **SEED_DAILY_BATCH 10 → 50 已完成并生效**，三层层级实证（文件 / 代码加载日志 / 行为记账）。
- 服务健康 PASS，零错误日志，去重与预算控制未被绕过，无重复导入，零代码修改。
- **按要求保持 50/天**，不再自动提升；进一步扩容（100/500/1000）等待后续单独确认。
- 未执行：《SiteIntel 自动繁衍规则修复任务》、人工导入、任何数据删除、10 个 Seed 灰度测试重放。

## 十、恢复 / 回滚（如需）

```bash
# 回到 10/天（备份版本）
cp /www/wwwroot/siteintel.cc/.env.bak.seed-batch-10-20260817 /www/wwwroot/siteintel.cc/.env
systemctl restart siteintel
```
