# SiteIntel 自动繁殖停止——生产环境只读故障诊断指令

## 0. 任务性质

这是一次 **生产环境只读故障诊断（Read-Only Incident Diagnosis）**。

项目：

- 网站：https://www.siteintel.cc/
- 项目：SiteIntel
- 当前生产版本必须以当前 Git HEAD / production deployment 为准
- 近期已完成生产对齐验收，GitHub main 当前已知版本为 `c74dfd1`

### 绝对禁止

本轮任务：

- ❌ 不修改任何业务代码
- ❌ 不修改数据库结构
- ❌ 不执行数据删除
- ❌ 不执行数据修复
- ❌ 不修改 Cron / Scheduler
- ❌ 不修改 Worker
- ❌ 不修改 Queue
- ❌ 不修改环境变量
- ❌ 不修改 Cloudflare 配置
- ❌ 不重新部署
- ❌ 不重启生产服务
- ❌ 不“顺手优化”
- ❌ 不根据猜测直接修复

只允许：

- 阅读代码
- 阅读配置
- 查询数据库
- 查询任务状态
- 查询日志
- 查询队列
- 查询生产 API
- 查询 Git 历史
- 对比当前线上行为
- 建立完整证据链

最终只输出诊断报告。

---

# 1. 当前异常

当前线上 SiteIntel 首页数据显示大致为：

- 已分析域名：58
- 调查次数：84
- 实体：511
- 关系：891
- 快照：726
- 洞察：0

当前明显怀疑：

> SiteIntel 的“自动繁殖”链路已经停止、卡死、没有触发，或者产生的新候选目标没有重新进入调查队列。

需要找出真正原因。

不要预设故障位置。

---

# 2. 先确认当前生产真实状态

首先检查：

1. 当前 Git branch
2. 当前 Git HEAD
3. GitHub main HEAD
4. 当前 production deployment 对应 commit
5. 工作区是否存在未提交修改
6. 当前数据库连接目标
7. 当前生产环境配置来源
8. 当前 Worker / Cron / Scheduler / Queue 架构

输出：

```text
Git HEAD:
GitHub main:
Production commit:
Working tree:
Database:
Scheduler:
Queue:
Worker:
Deployment:
```

如果当前生产版本不是 `c74dfd1`，必须明确指出。

---

# 3. 建立“自动繁殖完整链路”

不要只找一个名为 `auto-propagation` 或 `auto-breed` 的函数。

必须从代码中反向追踪整个业务闭环。

建立：

```text
Seed Domain
    ↓
Investigation Created
    ↓
Investigation Queued
    ↓
Worker Picks Task
    ↓
Collectors Execute
    ↓
Entities Extracted
    ↓
Relationships Created
    ↓
Candidate Domains / URLs Discovered
    ↓
Candidate Deduplication
    ↓
Candidate Qualification
    ↓
New Investigation Created
    ↓
Queue
    ↓
Worker
    ↓
Next Generation
```

对每一个节点找到：

- 文件
- 函数
- 数据表
- API
- Job
- Queue
- Worker
- 状态字段
- 触发条件

形成完整调用链。

---

# 4. 查找自动繁殖入口

全项目搜索以下概念及其实际实现：

```text
auto
automatic
propagation
breed
spawn
discover
candidate
seed
recursive
recursive investigation
next generation
related domain
related website
enqueue
queue
worker
scheduler
cron
job
investigation
investigate
```

同时搜索项目实际使用的中文/业务命名。

不要只依赖关键词。

目标：

> 找到“调查结果如何产生下一批调查对象”的真实实现。

---

# 5. 检查自动繁殖触发器

重点检查：

### A. Cron

确认：

- 是否存在
- 调度频率
- 最近执行时间
- 是否成功
- 是否失败
- 是否被禁用
- 是否存在时区问题
- 是否执行到了错误环境

### B. Scheduler

确认：

- Scheduler 是否运行
- 是否创建任务
- 是否有锁
- 是否被锁死
- 是否有重复执行保护
- 是否存在过期锁

### C. Queue

确认：

- Queue 名称
- Producer
- Consumer
- Worker
- pending 数量
- processing 数量
- completed 数量
- failed 数量
- dead-letter 数量
- retry 数量

### D. Worker

确认：

- Worker 是否运行
- 最近一次消费时间
- 最近一次成功时间
- 最近一次失败时间
- 是否存在异常
- 是否存在超时
- 是否存在权限错误
- 是否存在环境变量缺失

---

# 6. 检查数据库中的真实任务状态

不要只看代码。

直接查询生产数据库。

重点寻找：

- investigation
- investigation_jobs
- jobs
- tasks
- queue
- candidates
- discovered_domains
- domains
- entities
- relationships
- events
- insights
- snapshots

实际表名以项目为准。

至少回答：

### 调查任务

```text
总调查任务：
成功：
失败：
pending：
running：
queued：
cancelled：
```

### 最近 24 小时

```text
创建任务数：
成功任务数：
失败任务数：
新发现域名：
新发现候选：
新入队任务：
```

### 最近 7 天

同样统计。

---

# 7. 检查“自动繁殖是否实际上根本没有产生候选域名”

这是本次诊断最重要的部分之一。

必须区分：

### 情况 A

调查器没有发现候选目标。

```text
Investigation
    ↓
No Candidates
```

### 情况 B

调查器发现了候选目标，但候选目标被过滤。

```text
Investigation
    ↓
Candidates
    ↓
Dedup / Filter
    ↓
0 accepted
```

### 情况 C

候选目标已经通过过滤，但没有创建新的 Investigation。

```text
Candidate
    ↓
Accepted
    ↓
NO Investigation
```

### 情况 D

新 Investigation 已创建，但没有进入 Queue。

```text
Investigation Created
    ↓
Queue Missing
```

### 情况 E

已经进入 Queue，但 Worker 没有消费。

```text
Queue
    ↓
Worker STOPPED
```

### 情况 F

Worker 消费了，但执行失败。

```text
Worker
    ↓
Investigation
    ↓
ERROR
```

必须明确属于哪一种。

---

# 8. 检查去重逻辑

重点检查是否存在“自动繁殖被过度去重”的情况。

检查：

```text
domain dedup
URL dedup
IP dedup
entity dedup
relationship dedup
investigation dedup
candidate dedup
```

重点回答：

> 是否因为某个去重条件改变，导致所有新发现目标都被认为“已经调查过”？

例如：

```text
candidate domain
      ↓
already exists
      ↓
skip
```

但实际上：

```text
domain exists
≠
domain investigated
```

需要确认项目是否正确区分：

- 已发现
- 已入库
- 已调查
- 正在调查
- 调查失败
- 可以重新调查

---

# 9. 检查“繁殖上限”

重点搜索是否存在：

```text
maxDepth
maxGeneration
maxCandidates
maxDomains
maxInvestigations
dailyLimit
perRunLimit
batchSize
MAX_*
limit
threshold
cooldown
```

确认：

- 是否设置了最大繁殖深度
- 是否达到上限
- 是否因为数据库中的历史数据被认为已经达到上限
- 是否存在错误的全局计数
- 是否存在单域名计数错误

特别检查：

```text
generation
depth
parent_id
root_id
seed_id
```

---

# 10. 检查“自动繁殖是否只对特定调查类型生效”

确认：

- 手动调查是否会触发繁殖
- 自动调查是否会触发繁殖
- API 调查是否会触发繁殖
- 历史调查是否会触发繁殖
- 首次调查是否会触发繁殖
- 二次调查是否会触发繁殖

明确触发条件。

例如：

```text
if source === "auto"
```

或者：

```text
if investigation.type === "scheduled"
```

或者：

```text
if depth < MAX_DEPTH
```

必须找出真实条件。

---

# 11. 检查新域名发现机制

分析所有可能产生新网站/域名的来源：

- DNS
- CNAME
- NS
- MX
- SSL Certificate
- IP
- ASN
- CDN
- Technology
- Third-party services
- Links
- Redirects
- Same IP
- Same certificate
- Same infrastructure
- WHOIS / RDAP
- 页面引用
- Sitemap
- robots.txt

不要假设所有数据都应该产生新域名。

必须建立：

```text
Data Source
    ↓
Entity
    ↓
Relationship
    ↓
Candidate
```

实际映射。

---

# 12. 检查候选域名质量门槛

检查是否存在：

```text
isValidDomain
isPublicDomain
isIndexable
isAllowed
isEligible
isSafe
isInteresting
isNew
```

确认是否因为某个条件变得过于严格，导致：

```text
candidate_count > 0
accepted_count = 0
```

如果存在，给出实际统计。

---

# 13. 检查洞察为什么为 0

“自动繁殖停止”和“洞察 0”可能有关，也可能是两个独立问题。

必须单独诊断。

检查：

```text
events
signals
insights
hypotheses
snapshots
rules
detectors
```

回答：

### A

是否真的产生过 Event？

### B

Event → Insight 是否存在？

### C

是否因为规则没有命中导致 Insight = 0？

### D

是否 Insight Worker 没有运行？

### E

是否 Insight 写入失败？

### F

数据库是否有 Insight 表但没有数据？

### G

前端是否错误显示 0？

不要把：

```text
Insight = 0
```

直接解释成：

```text
自动繁殖停止
```

必须分别判断。

---

# 14. 检查最近一次成功繁殖

这是本次诊断的关键证据。

找到：

> 最近一次真正产生“新调查对象”的记录。

输出：

```text
Parent Investigation:
Parent Domain:
Discovered Candidate:
Candidate:
Candidate Created At:
Candidate Accepted:
New Investigation ID:
Queued:
Worker Started:
Worker Completed:
Result:
```

然后向前追溯。

如果找不到任何历史记录，也必须明确：

> 当前数据库中无法证明自动繁殖曾经成功运行。

---

# 15. 检查时间线

建立：

## 最近 30 天自动繁殖时间线

至少输出：

```text
日期
自动任务执行次数
成功次数
失败次数
发现候选数量
接受候选数量
新调查数量
Worker 消费数量
```

示例：

```text
2026-08-10 | 5 | 5 | 0 | 32 | 12 | 12 | 12
2026-08-11 | 5 | 5 | 0 | 28 | 10 | 10 | 10
...
2026-08-16 | 0 | 0 | 0 | 0 | 0 | 0 | 0
2026-08-17 | 0 | 0 | 0 | 0 | 0 | 0 | 0
```

如果出现明显断点，找出：

> **最后一次正常运行时间**

以及：

> **第一次异常时间**

---

# 16. Git 回归分析

对比：

```text
当前 HEAD
vs
上一稳定版本
```

重点检查最近涉及：

- investigation
- crawler
- discovery
- candidate
- queue
- worker
- scheduler
- propagation
- database
- dedup
- insight

的 commit。

目标不是修改。

目标是判断：

> 是否某个最近提交导致自动繁殖停止。

如果存在高度相关 commit：

```text
Commit:
Date:
Author:
Changed files:
Relevant change:
Potential impact:
Evidence:
```

---

# 17. 生产日志证据

检查生产日志中：

```text
ERROR
WARN
timeout
queue
worker
cron
scheduler
investigation
candidate
database
connection
permission
rate limit
429
500
503
```

重点关注自动繁殖相关日志。

输出真实错误，不要只总结。

如果没有日志，也必须明确：

> 当前系统缺乏足够的自动繁殖可观测性。

---

# 18. 检查环境变量

只检查名称和值是否存在。

禁止泄露 Secret。

检查：

```text
CRON_*
QUEUE_*
WORKER_*
DATABASE_*
REDIS_*
SCHEDULER_*
CLOUDflare_*
API_*
```

对于 Secret：

```text
EXISTS
MISSING
```

即可。

绝对不要输出：

- API Key
- Token
- Password
- Secret
- Cookie
- 私钥

---

# 19. 最终建立故障树

必须最终输出：

```text
自动繁殖
│
├── Trigger
│   └── PASS / FAIL
│
├── Scheduler
│   └── PASS / FAIL
│
├── Queue Producer
│   └── PASS / FAIL
│
├── Queue
│   └── PASS / FAIL
│
├── Worker
│   └── PASS / FAIL
│
├── Investigation
│   └── PASS / FAIL
│
├── Entity Extraction
│   └── PASS / FAIL
│
├── Candidate Discovery
│   └── PASS / FAIL
│
├── Deduplication
│   └── PASS / FAIL
│
├── New Investigation Creation
│   └── PASS / FAIL
│
└── Next Generation
    └── PASS / FAIL
```

---

# 20. 最终结论必须分类

只能从以下分类中选择：

### A. Trigger 故障

自动繁殖根本没有被触发。

### B. Scheduler 故障

调度器没有创建任务。

### C. Queue 故障

任务创建但无法进入队列。

### D. Worker 故障

队列有任务但 Worker 不消费。

### E. Investigation 故障

Worker 执行但调查失败。

### F. Discovery 故障

调查正常但没有产生候选目标。

### G. Dedup / Filter 故障

产生候选但全部被过滤。

### H. Recursive Creation 故障

候选通过但没有创建下一代调查。

### I. Generation Limit

达到了设计上的繁殖限制。

### J. Insight 独立故障

自动繁殖正常，但 Insight Pipeline 单独失效。

### K. Multiple Failures

存在多个独立故障。

---

# 21. 必须给出“证据等级”

每一个结论标记：

```text
CONFIRMED
HIGH CONFIDENCE
MEDIUM CONFIDENCE
LOW CONFIDENCE
UNKNOWN
```

禁止把猜测写成事实。

---

# 22. 最终报告结构

生成：

```text
SITEINTEL_AUTO_PROPAGATION_PRODUCTION_DIAGNOSIS.md
```

报告必须包含：

# SiteIntel 自动繁殖生产故障诊断报告

## 1. 执行摘要

## 2. 当前生产版本

## 3. 当前线上数据状态

## 4. 自动繁殖架构

## 5. 完整调用链

## 6. Scheduler / Cron 检查

## 7. Queue 检查

## 8. Worker 检查

## 9. Investigation 检查

## 10. Entity / Relationship 检查

## 11. Candidate Discovery 检查

## 12. Dedup / Filter 检查

## 13. Recursive Investigation 检查

## 14. Generation / Depth Limit 检查

## 15. 最近 30 天运行时间线

## 16. 最近一次成功繁殖证据

## 17. 最后一次正常运行时间

## 18. 第一次异常时间

## 19. Git 回归分析

## 20. Production Log 分析

## 21. Insight = 0 独立分析

## 22. 故障树

## 23. 根因判断

## 24. 证据等级

## 25. 修复建议

---

# 23. 修复建议要求

本轮虽然禁止修改代码，但报告最后可以提出修复建议。

修复建议必须分级：

### P0

导致自动繁殖完全停止的问题。

### P1

导致繁殖效率严重下降的问题。

### P2

可观测性、统计、日志、后台管理等问题。

不要提出与本次故障无关的大规模重构。

---

# 24. 最重要的验收标准

最终报告必须回答下面 12 个问题：

1. 自动繁殖现在到底有没有运行？
2. 最后一次运行是什么时候？
3. 最后一次成功是什么时候？
4. 最近一次失败是什么时候？
5. Scheduler 是否正常？
6. Queue 是否正常？
7. Worker 是否正常？
8. Investigation 是否正常？
9. Candidate Discovery 是否正常？
10. Candidate 是否成功变成下一代 Investigation？
11. 哪一个具体环节断了？
12. 为什么现在 58 个域名 / 84 次调查之后没有继续增长？

如果无法回答某一项，必须明确：

```text
UNKNOWN
```

并说明缺少什么证据。

---

# 25. 最终禁止事项

再次强调：

本轮任务结束前：

**绝对不要修改任何文件。**

不要：

```text
git commit
git push
git reset
git checkout
npm install
数据库迁移
修改配置
修改环境变量
修改 Cron
修改 Worker
重新部署
```

如果发现问题：

> 只记录问题，不修复。

最终只交付：

```text
SITEINTEL_AUTO_PROPAGATION_PRODUCTION_DIAGNOSIS.md
```

以及终端中的关键结论摘要。

# END