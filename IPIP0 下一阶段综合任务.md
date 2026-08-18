# IPIP0 下一阶段综合任务
## 首页结果页第二轮 UX 改造 + 阶段四 P1 数据质量与 Gate D 持续推进

你现在开始执行 IPIP0 下一阶段综合任务。

本任务由两条并行工作线组成：

**Track A：首页结果页第二轮 UX 改造**

**Track B：阶段四 P1 数据质量 / 数据覆盖度复查 + Gate D 持续积累**

两条工作线必须相互隔离：

- Track A 只改前端展示、交互、页面结构和已有 API 数据消费方式，不重构数据库。
- Track B 只负责真实数据质量、持续 reprobe、健康检查和 Gate D，不开发新的公开实体页面。
- 不允许为了 UX 改造而修改已经 PASS 的 Collector、Snapshot、query_log 生命周期逻辑。
- 不允许为了 P1 数据增长而新增公开页面。
- 不允许使用 Mock Data。
- 不允许修改 `D:\phpstudy_pro\WWW\ipquery`。

---

# 一、当前真实项目状态

当前已经完成并通过：

### 阶段一
基础页面、核心查询、P1 修复。

### 阶段二
`ipquery` 后端基础设施迁移到 IPIP0。

已经具备：

- 独立 `ipip0` 数据库
- IpStoreDb
- RDAP
- ASN Entity
- 风险数据
- 查询日志
- Collector
- Redis / Cache / Queue 等基础能力（以当前真实代码为准）

### 阶段三
数据模型与实体关系：

- IP Entity
- ASN Entity
- Risk Profile
- Risk Evidence
- Risk Snapshot
- Query Event

并已完成：

- `/ip/{ip}/risk`
- `/asn/{asn}`

两个真实数据驱动详情页。

### 阶段四 P0
已经通过：

- Collector pacing
- retry / exponential backoff
- 429 cooldown
- concurrency lock
- checkpoint / resume
- Risk Snapshot 去重
- query_log 90 天保留
- query_log_daily 聚合
- ASN 30 天刷新
- failure persistence

### 阶段四 P1
目前真实数据约为：

- IP Entity：103
- ASN Entity：29
- Risk Evidence：37
- Risk Snapshot：116
- query_log：281

Gate：

- A：IP ≥100 ✅
- B：ASN ≥20 ✅
- C：主要 ASN ≥3 samples ✅
- D：至少 10 IP × 至少 3 个不同自然日快照 ⏳
- E：采集健康 ✅

Gate D 尚未完成，这是正常的跨自然日积累问题。

---

# 二、Track A：首页结果页第二轮 UX 改造

## 总目标

不是重新设计 IPIP0。

不是重新做一套视觉系统。

不是增加大量字段。

而是：

> **把当前已经存在的真实数据，更清晰地组织起来，并把已有的 ASN / Risk 实体详情能力真正连接到首页。**

必须基于当前线上实际页面和当前真实代码执行。

---

# 三、先做 UX 改造前审计

正式修改前，先只读检查：

1. 当前首页实际 HTML / CSS / JS
2. 当前结果卡片 DOM 结构
3. 当前设计系统已有：
   - Color Token
   - Badge
   - Tag
   - Status
   - Button
   - Card
   - Typography
4. 当前 API 返回字段
5. 当前 ASN 链接实现
6. 当前 Risk 详情链接实现
7. 当前多来源位置数据结构
8. 当前全球延迟数据结构
9. 中英文页面是否共用组件
10. 当前移动端结构

先生成：

`IPIP0-首页第二轮UX改造前审计.md`

只读审计完成后再开始修改。

---

# 四、强制 UI 规则

这是本次任务的重要限制。

## 1. 不允许自行发明新视觉体系

尤其禁止因为参考其他网站而新增：

```html
<span class="ent-badge">ISP</span>
```

或类似：

- ISP 绿色 Badge
- Hosting 黄色 Badge
- Risk 红色 Badge
- 新颜色分类体系
- 新的 Entity Badge 体系

除非当前 IPIP0 项目代码中已经存在对应组件和设计 Token。

必须优先：

> 复用当前 IPIP0 已有颜色、标签、按钮、卡片、状态组件。

如果当前项目没有合适组件：

1. 先检查现有组件是否可以复用。
2. 如果确实没有，报告缺口。
3. 采用与现有系统一致的最小扩展。
4. 不得自行建立一套新的颜色语义系统。

## 2. 不改变已有品牌视觉基础

不要无理由修改：

- 页面主背景
- 核心品牌颜色
- Logo
- Header
- Footer
- 全局字体体系

这次是信息架构和交互优化，不是视觉重做。

---

# 五、首页结果卡片改造

当前页面主要存在的问题：

结果信息还是较明显的“字段平铺”。

要改为更清晰的信息层级。

建议按照真实现有字段，调整为：

## ① 基础信息

默认展示：

- IP
- Location
- ASN
- Organization / Company
- ISP / Operator（依据当前真实字段语义）

注意：

“公司”和“运营商”不得在没有数据依据时强行做语义推断。

如果当前字段映射仍存在历史遗留问题，优先保持真实数据，同时在报告中指出。

---

## ② 网络属性

展示：

- IP Type
- IP Tags
- Network Prefix（只有真实数据存在时展示）

ASN 必须成为可点击入口：

`ASxxxxx →`

跳转：

`/asn/{asn}`

不要新增假的 ASN 信息。

---

## ③ 风险与信誉

首页只展示摘要，不展示全部证据。

建议结构：

```text
风险评分
30%
中风险

风险标签
XXX / XXX / XXX

查看风险详情 →
```

点击：

`/ip/{ip}/risk`

首页不重复展开完整：

- evidence_summary
- source
- confidence
- detected_at
- expires_at

详情页负责完整解释。

---

## ④ 技术信息

整合当前已有：

- Service Fingerprint
- Port Exposure

保持现有信息，不凭空增加服务或端口。

如果真实数据不足：

使用当前项目已有空态。

禁止 Mock。

---

# 六、多来源位置对比升级

这是本轮重要改造之一。

当前逻辑已经存在多个数据源。

目前如果页面只是：

```text
IPinfo
DB-IP
MaxMind
IP2Location
...
```

继续保留来源明细。

但是在来源列表之前增加一个真实计算的“共识摘要”。

例如：

```text
位置共识
莱索托 · 马塞卢

一致性：高
5 / 6 数据源一致
```

具体数值必须来自真实 API / SQL 计算。

禁止硬编码。

## 计算要求

使用当前真实数据源结果：

- 国家一致性
- 城市一致性
- 数据源数量
- 一致数量
- 分歧数量

至少输出：

- Consensus
- Consistency Level
- Agreement Count
- Total Sources

如果现有数据无法可靠计算：

不要模拟。

显示：

`暂无法形成可靠共识`

并在报告中说明原因。

---

# 七、全球延迟测试升级

保持现有国家 / 节点网格。

在网格上方增加真实摘要：

```text
全球网络表现

最佳节点：英国
最低延迟：206 ms

最快区域：欧洲
```

但必须基于实际返回数据动态计算。

禁止硬编码。

如果没有足够节点：

使用真实空态。

不允许为了视觉效果制造延迟数字。

---

# 八、首页信息层级建议

最终首页结果区域应形成：

```text
IP 查询结果
│
├── 基础信息
│   ├── IP
│   ├── 位置
│   ├── ASN → ASN详情
│   ├── 公司 / 组织
│   └── 运营商
│
├── 网络属性
│   ├── IP 类型
│   ├── IP 标签
│   └── 网络段
│
├── 风险与信誉
│   ├── 风险评分
│   ├── 风险标签
│   └── 查看风险详情
│
├── 技术信息
│   ├── 服务指纹
│   └── 端口暴露
│
├── 多来源位置
│   ├── 共识结论
│   └── 来源明细
│
└── 全球延迟
    ├── 总结
    └── 节点明细
```

不要机械创建 6 个大卡片。

具体视觉结构必须根据当前页面已有组件实现。

---

# 九、首页行为要求

查询前：

保持当前查询入口。

查询后：

首页标题和结果区域应明显体现：

> 当前正在查看一个 IP 情报结果

但不要删除查询框。

用户应该能够：

```text
查询
↓
查看结果
↓
点击 ASN
↓
ASN 详情
↓
返回 IP

或者：

查询
↓
风险评分
↓
风险详情
↓
返回 IP
```

必须保证原有查询流程不被破坏。

---

# 十、移动端

必须至少测试：

- 375px
- 390px
- 768px

重点检查：

1. 风险入口是否清晰
2. ASN 点击范围
3. 多来源位置是否横向溢出
4. 全球延迟节点是否溢出
5. 标签是否换行异常
6. 卡片是否过长
7. Header 是否拥挤
8. Button / Link 触控区域

不得为了移动端重新建立独立组件体系。

---

# 十一、Track B：P1 数据质量与 Gate D

第二条线继续进行，但严格受控。

## 1. 每日任务继续

当前建议任务：

```bash
php collector\run.php scan --file=data/seeds/reprobe_existing.txt --job=reprobe_daily
php collector\run.php cleanup-logs
```

只允许按照已经批准的 P1 策略运行。

不允许自动扩大扫描量。

---

# 十二、对当前 103 IP 做数据质量复查

不要简单继续增加 200 个 IP。

先分析现有真实数据。

必须统计：

### IP 覆盖

- 总 IP
- IPv4 / IPv6
- 新增 IP / 日
- 重复率

### ASN 分布

- ASN 总数
- 每个 ASN IP 数
- Top ASN
- ASN 集中度
- 每 ASN 最少样本数

### 风险质量

- 有风险评分 IP 数
- Risk Evidence 覆盖率
- Risk Snapshot 覆盖率
- 不同 Risk Level 分布
- 风险标签分布

### 时间数据

- 有 Snapshot 的 IP 数
- ≥2 个日期 Snapshot 的 IP
- ≥3 个日期 Snapshot 的 IP
- 当前 Gate D 实际完成数量

### RDAP

- ASN Entity 数
- RDAP freshness
- 最老 enriched_at
- 缺失字段比例
- Organization / Country / Prefix 数据完整度

### 数据源健康

- Success
- Retry
- Failure
- 429
- Timeout
- Parse Error

### 数据分布异常

特别检查：

- 是否过度集中于少数 ASN
- 是否存在大量重复网络
- 是否存在明显测试 IP 残留
- 是否存在异常数据
- 是否存在来源单一导致的偏差

---

# 十三、Gate D 不允许伪造完成

继续保持：

```text
Gate D：
≥10 IP × ≥3 不同自然日 Snapshot
```

必须来自真实自然日期。

禁止：

- 修改时间字段
- 手动制造历史数据
- SQL 回填虚假 Snapshot
- 多次同日刷新冒充不同日期

当前继续：

```text
A ✅
B ✅
C ✅
D ⏳
E ✅
```

只有真实满足条件才变为 PASS。

---

# 十四、增加“数据健康观察”结果

本次任务完成后输出当前真实：

```text
IP Entity
ASN Entity
Risk Evidence
Risk Snapshot
Query Log

24h 新增
7d 新增

Collector Success
Retry
Failure
429

RDAP freshness

Gate A/B/C/D/E
```

只读输出。

暂时不建立公开管理页面。

---

# 十五、P2 / P3 继续锁定

在 Gate D 之前：

禁止：

- Organization 公开详情页
- Network Prefix 公开详情页
- Country 详情页
- 大量程序化 SEO 页面
- 新公开实体页面
- 全网 IPv4 扫描
- 大规模 ASN 扫描

当前只允许：

**数据质量复查 + 已有页面 UX 优化 + 日常受控 reprobe。**

---

# 十六、代码修改边界

Track A：

允许修改：

- 首页模板
- 首页 CSS
- 首页 JS
- 现有 API 调用消费逻辑
- 现有组件
- 多来源位置摘要计算的前端展示 / 已有后端计算接口

如果需要新增后端接口：

必须先检查现有 API 是否可以直接复用。

禁止为了一个展示字段重复创建新的 API。

Track B：

允许修改：

- CLI Task
- Collector 调度相关代码
- 数据健康统计
- 日常任务

但是：

禁止修改已经 PASS 的：

- Snapshot 去重规则
- query_log 生命周期
- ASN 30 天刷新策略
- failure persistence
- concurrency lock

除非发现真实回归问题。

---

# 十七、必须生成的报告

执行前：

`IPIP0-首页第二轮UX改造前审计.md`

执行后生成两份报告：

### 报告一

`IPIP0-首页第二轮UX改造与验收报告.md`

必须包含：

- 修改了哪些页面
- 修改了哪些组件
- 复用了哪些现有设计组件
- 新增了哪些交互
- ASN 链接验证
- Risk 链接验证
- Location Consensus 验证
- Latency Summary 验证
- 移动端测试
- 中英文测试
- 空态
- 错误态
- Zero Mock 验证

### 报告二

`IPIP0-阶段四-P1-数据质量与Gate复查报告.md`

必须包含：

- 103+ IP 当前真实数据
- ASN 分布
- Risk 覆盖
- Snapshot 跨日情况
- RDAP freshness
- Collector health
- 当前 Gate A/B/C/D/E
- 当前数据质量问题
- 当前数据覆盖问题
- 下一步建议

---

# 十八、最终验收标准

本任务只有在以下条件全部满足时才算 PASS：

## UX

- [ ] 首页原有查询功能正常
- [ ] ASN 可进入 `/asn/{asn}`
- [ ] Risk 可进入 `/ip/{ip}/risk`
- [ ] 多来源位置显示真实共识
- [ ] 全球延迟显示真实摘要
- [ ] 不出现 Mock Data
- [ ] 不新增未经现有设计系统定义的颜色 / Badge
- [ ] 中英文正常
- [ ] 375px 正常
- [ ] 390px 正常
- [ ] 768px 正常
- [ ] 全站 API 回归通过

## Data

- [ ] 每日 reprobe 正常
- [ ] Collector pacing 正常
- [ ] query_log 正常
- [ ] Snapshot 去重正常
- [ ] ASN refresh 正常
- [ ] Failure queue 正常
- [ ] 数据健康指标正常
- [ ] Gate 状态真实反映
- [ ] Gate D 未满足前不得报告 PASS

---

# 十九、执行原则

最重要的原则：

**不要为了完成任务而增加数据。**

**不要为了页面效果制造数据。**

**不要为了视觉效果创造新的颜色语义。**

**不要为了 SEO 创建没有真实数据支撑的页面。**

**不要破坏已经 PASS 的后端数据生命周期。**

**不要修改 `D:\phpstudy_pro\WWW\ipquery`。**

先审计，再改造；先真实数据，再展示；先验证，再报告。

最终目标不是“页面看起来更丰富”，而是：

> **让 IPIP0 已经建立起来的真实 IP Intelligence 数据能力，真正被首页用户看见、理解并继续深入探索。**