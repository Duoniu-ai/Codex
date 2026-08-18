# DISCOVERY-CANDIDATE-FILTER-DESIGN.md

SiteIntel Discovery Candidate Quality Filter v1 —— 设计文档（READ ONLY / DRY-RUN ONLY）
设计时间：2026-08-16 · **未修改代码、未修改数据、未改 priority、未消费候选、未改额度、未部署**

---

## 1. 当前问题

588/天候选供给 vs 20/天消费 = **29:1 结构性失衡**；pending 875 且日净增 +570。质量审计已定位根源：**SAN 洪峰来自约 15 个跨国大平台的证书 SAN 列表**（Amazon/eBay/Pinterest/PayPal/Airbnb 的各国变体、Netflix/Yahoo/Bloomberg/GitLab/Snapchat/BBC/Wikimedia/Meta/Pixar 等），每次分析其中任一成员 → 其证书 SAN 再次列出同一批平台的全部国别端点 → **自我繁殖闭环**。

## 2. 875 pending 分类结果（v1 规则 dry-run 实测）

| 分类 | 数量 | 占比 | 主要来源 |
|---|---:|---:|---|
| HIGH | 12 | 1.4% | shared_infra×6、redirect×3（threads.com/open.spotify.com/about.gitlab.com）、cname×1、san×2（alibabacloud.com、78500.cn） |
| MEDIUM | 470 | 53.7% | SAN 2 段注册域为主（含 alicloud/antgroup/qiyi/sina 等真实企业域） |
| LOW | 296 | 33.8% | 3 段非平台子域 + m./wap. 移动变体 |
| FILTER | 91 | 10.4% | 基础设施模式（tiles/cdn/api/login/mail）+ 平台家族子域 |
| NEVER_AUTO_ANALYZE | 6 | 0.7% | 分析/遥测端点（app-analytics-services*、fps.goog、googleoptimize.com） |

**过滤 FILTER+NEVER 后剩余 778（仅滤掉 11.1%）**——模式过滤对总量几乎无效。

## 3. Candidate Quality Score 公式（v1，纯函数，不写库）

```
candidateQualityScore = clamp(0,100,
    sourceScore
  + apexBonus
  + patternPenalty
  + platformFamilyPenalty
  + existingBonus
)
sourceScore:   shared_infra/shared_ip 60 · redirect 55 · cname 50 · san 45
apexBonus:     2 段注册域 +25 · 3 段 +8 · 更深 +2
patternPenalty:  NEVER 模式 −50 · FILTER 模式 −40
platformFamilyPenalty: 3 段及以上且家族 ∈ 平台后缀清单 −30
existingBonus:  已存在 Domain Entity/Target +15
```
分级：≥75 HIGH · 55-74 MEDIUM · 40-54 LOW · 其余按确定性规则归 FILTER/NEVER。

## 4. Source + Domain Pattern 联合规则（实测验证后）

| 规则 | 内容 | 实测依据 |
|---|---|---|
| NEVER | 分析/遥测/追踪模式（analytics/tracking/telemetry/beacon/collector…）、app-analytics-services*、fps.goog、googleoptimize.com | 6 条命中，全部为 AT&T/Bing 分析端点 |
| FILTER | 基础设施模式（tiles/cdn/edge/static/img/api/login/auth/mail/smtp）+ **大平台家族子域**（≥3 段且家族 ∈ 平台后缀清单） | 91 条命中 |
| LOW | m./wap./www. 移动变体 + 3 段普通子域 | 296 条 |
| MEDIUM | SAN 2 段注册域（非平台）、cname 非基础设施 | 470 条 |
| HIGH | shared_infra / shared_ip / 高质量 redirect / 用户相关 | 12 条 |

## 5. SAN 分层策略

- **apex（2 段）默认保持候选价值**：误杀验证实测 **2 段域保留率 99.2%**（478 条中仅 4 条分析端点被 NEVER 命中）✓
- **大平台属性域（平台后缀清单）**：子域 → FILTER；2 段 apex → LOW（如 amazon.com.au 本身可低优先级分析，但绝不让其子域繁殖）
- **移动变体（m./wap.）** → LOW（主域通常已被覆盖）
- **staging/dev/beta/qa 前缀** → FILTER（实测 staging.claude.ai、amb-staging.chatgpt.com、develop-stage.netflix.com）

## 6. shared_infra / shared_ip 策略

**KEEP + HIGH PRIORITY**。实测：100% Target 转化、10 关系/候选、质量分 87 全场最高、当前 6 条已等待 47.5h。**建议**（本阶段不修改）：将 shared_infra/shared_ip 的默认 priority 从 40/75 提升至与 redirect 同级（55+）或更高——与 aging 机制协同，消除其被 SAN 插队导致的 5 天等待。

## 7. 当前策略模拟（20/天）

| 指标 | 值 |
|---|---|
| 存量 pending | 875 |
| 24h 供给 / 消费 | 588 / 18 |
| 净变化 | +570/天 |
| 存量清空时间 | 44 天（且持续被淹没，实际永不清空） |

## 8. 新策略模拟（三层过滤，20/天）

| 过滤层 | 24h 供给变化 | 剩余供给 | 净变化/天 |
|---|---:|---:|---:|
| ① NEVER+FILTER 模式（消费侧） | 588 → 564 | 564 | +544 |
| ② 平台后缀清单（消费侧，实测 50.7% SAN 命中） | 564 → **285** | 285 | +265 |
| ③ 平台 apex 降 LOW + 移动变体降 LOW + staging 过滤 | 285 → ≈170 | ≈170 | ≈+150 |
| **存量清空时间（② 后）** | — | 582 条 | **29 天** |

**诚实结论**：即使三层消费侧过滤全部落地，供给仍是消费的 ~8.5 倍——**消费侧过滤无法使 pending 收敛**，只能缓解。

## 9. 每日供给预测

- 当前：588/天（SAN 578）
- 消费侧三层过滤后：≈170/天
- **根治方向（源侧，需单独批准）**：`extractCandidatesFromBundle` 对 SAN 提取做平台后缀跳过 + 每家族每轮最多保留 N 条（如 3 条）——洪水源头是 ~15 个封闭家族，源侧一刀可砍掉 80%+ 供给；叠加消费侧过滤后供给有望降至 30-60/天，与 20/天消费进入同数量级

## 10. pending 曲线预测

| 策略 | 曲线 |
|---|---|
| 当前 | 单调上升（+570/天），无收敛 |
| 消费侧三层过滤 | 上升放缓（+150/天），29 天可清一次存量但持续再积压 |
| 源侧平台跳过 + 消费侧过滤 | **接近收敛**（供给 30-60/天 vs 消费 20/天；配合适度提额可转为净下降） |

## 11. 误杀风险

- 2 段注册域保留率实测 **99.2%** ✓（唯一被滤的 2 段域是分析端点，正确）
- 风险点 ①：平台后缀清单必须**精确到平台**（如 `*.ebay.*`），不能按 ccTLD 家族整族过滤——com.cn 家族里混着 hanjushe.com.cn、meiju-tv.com.cn 等真实独立站（实测样本）
- 风险点 ②：清单需可增删（curated 表或代码常量 + 审计日志），新增平台时先观察再入单
- 风险点 ③：MEDIUM 池含金矿（alicloud/antgroup/alipay-eco 等），**绝不可把 SAN 整体降级**

## 12. 推荐规则（最终）

```
第一道 确定性禁止（NEVER）：
  分析/遥测/追踪模式 · app-analytics-services* · 已知纯端点

第二道 公共基础设施过滤（FILTER）：
  tiles/cdn/edge/static/img/api/login/auth/mail 模式
  + 大平台家族子域（平台后缀清单 × ≥3 段）
  + staging/dev/beta/qa 前缀

第三道 候选质量评分：
  candidateQualityScore（§3 公式）→ HIGH/MEDIUM/LOW

第四道 Priority Aging（已有）：
  effectivePriority = priority + min(等待天数×5, 40)

第五道 人工/用户提交优先：
  用户提交（/api/analyze）与监控目标永远优先于 Discovery 消费

平台后缀清单（v1 种子，从真实数据归纳）：
  amazon*（含 edgeflow/frontier.amazon）、ebay.*、pinterest.*、
  paypal.*、airbnb.*、netflix.com、yahoo.com、bloomberg.*、
  gitlab.com、snapchat.com、snap.com 家族、bbc.co.uk、
  wikimedia 家族（wikibooks/wikidata/wikinews/…）、pixarinabox.*、
  analytics-services*、instagram/facebook/fbcdn/messenger.com
```

## 13. 实施顺序（待批准）

1. **Phase F1（消费侧，最小改动）**：collector 消费查询前加过滤谓词（NEVER+FILTER 模式 + 平台后缀 + staging），不改库、不改 priority——dry-run 验证后一次小改动上线
2. **Phase F2（源侧，根治）**：`extractCandidatesFromBundle` SAN 提取加平台后缀跳过 + 每家族/轮上限——这是「修改 Discovery Source」，需按治理规则单独批准
3. **Phase F3（可选）**：shared_infra/shared_ip 默认 priority 提升至 55+

## 14. 回滚方案

全部过滤逻辑实现为纯函数/常量（如 `lib/discovery/candidate-filter.ts`）+ 特征开关（环境变量 `DISCOVERY_FILTER_V1`），回滚 = 关开关或 revert commit；无数据迁移、无库变更。

## 15. 是否建议进入实施阶段

**建议：批准 Phase F1 + F2 设计后进入实施**——但必须明确：

1. **消费侧过滤单独实施只是缓解**（29:1 → 8.5:1），不能解决根本问题；
2. **根治必须动源侧**（SAN 提取的平台跳过 + 家族上限），否则任何消费侧优化都会被 578/天的供给重新淹没；
3. 在 F2 落地前，**不建议提高 20/天额度**——提额只会加速自我繁殖闭环（更多平台成员被分析 → 更多同族候选）；
4. 20/天额度在 F1+F2 之后重新评估（若供给降至 30-60/天，提额至 40-50/天即可转为净下降）。

---

*设计完。本次全部为只读 dry-run（临时脚本已清理），未修改任何代码、数据与配置。*
当前会话只处理 Discovery Candidate Filter F1。

请先阅读：
DISCOVERY-CANDIDATE-FILTER-DESIGN.md

当前目标：
完成 F1 消费侧过滤，不做 F2。

现有 F1 代码可能已经部分修改，请先检查 git status / diff，确认当前状态后继续。

严格执行：
1. F1 dry-run
2. 保护样本验证
3. candidate-filter tests
4. 全量 test
5. typecheck
6. lint
7. build
8. 部署 F1
9. 不消费 pending Candidate
10. 不修改 priority
11. 不修改每日 20/day
12. 不进入 F2

dry-run 只输出汇总，不要逐条打印 875 个 Candidate，避免上下文过大。

重点输出：
- pending 总量
- FILTER
- NEVER_AUTO_ANALYZE
- KEEP
- 各 reason 数量
- 各 source × verdict
- 保护样本结果
- 是否误杀
- F1 对当前供给的模拟影响

完成后生成：
DISCOVERY-CANDIDATE-F1-REPORT.md

然后停止。
