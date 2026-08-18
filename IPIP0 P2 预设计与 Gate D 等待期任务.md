# IPIP0 P2 预设计与 Gate D 等待期任务

当前阶段五最终收口审计已完成。

当前状态：

- COMPLETE：14
- PARTIAL：1
- 暂缓：1
- COMING SOON：2
- P0/P1：0
- Gate A：PASS
- Gate B：PASS
- Gate C：PASS
- Gate D：未通过，0/10
- Gate E：PASS

当前唯一 P2 阻塞项为 Gate D。

本任务暂不开发 P2，不修改代码，只做预设计和持续数据观察。

---

## 一、Gate D 继续自然积累

继续每日：

```bash
php collector\run.php scan --file=data\seeds\reprobe_existing.txt --job=reprobe_daily_日期
php collector\run.php cleanup-logs
```

要求：

- 不扩大扫描范围
- 不修改时间字段
- 不手工插入历史 Snapshot
- 不人为制造跨日数据
- 不修改 Gate D 标准

每日仅检查：

- IP Entity
- ASN Entity
- Risk Snapshot
- 不同自然日 Snapshot 数
- Gate A/B/C/D/E

---

# 二、P2 预设计：Latency 状态机

只读审计当前实现。

目标状态至少规划：

```text
未开始
↓
排队
↓
测试中
↓
部分完成
↓
完成
```

异常：

```text
超时
无响应
失败
服务不可用
```

必须确认：

- 当前真实任务模型
- 当前真实返回结构
- 当前有哪些状态已经存在
- timeout 为什么目前不可达
- queueing 为什么目前缺失

不要修改代码。

输出：

`IPIP0-P2-Latency状态机设计.md`

---

# 三、P2 预设计：Organization Entity

基于当前真实数据：

目前约：

36 ASN
34 个独立 Organization 字符串

先设计：

`organization_entity`

要求明确：

1. Organization Entity 主键
2. 原始名称
3. 规范化名称
4. 归一化规则
5. ASN ↔ Organization 关系
6. IP ↔ Organization 间接关系
7. 数据来源
8. 更新时间
9. 冲突处理
10. 人工审核机制

特别注意：

**禁止自动把字符串相同直接视为同一 Organization。**

必须设计：

```text
原始名称
↓
Normalization
↓
Candidate Match
↓
确认 / 拒绝
↓
Entity
```

如果无法可靠判断：

保留为不同实体或待审核。

不要猜。

输出：

`IPIP0-P2-Organization-Entity设计.md`

---

# 四、P2 预设计：Network Prefix Entity

当前真实情况：

RDAP 有范围文本，
但尚未结构化为 CIDR。

先设计：

`prefix_entity`

必须明确：

1. Prefix 主键
2. CIDR 标准格式
3. 原始 RDAP range
4. 起始 IP
5. 结束 IP
6. IPv4 / IPv6
7. ASN 关系
8. Organization 关系
9. RIR
10. 计算规则
11. 范围重叠处理
12. 数据源更新策略

特别注意：

不要为了页面方便直接解析字符串写死。

必须设计可靠的：

```text
RDAP Range
↓
IP Start / IP End
↓
CIDR 分解
↓
Canonical Prefix
```

输出：

`IPIP0-P2-Prefix-Entity设计.md`

---

# 五、P2 预设计：Country / RIR 聚合

只分析是否有必要建立聚合视图。

基于当前真实：

- IP
- ASN
- RIR
- Country

判断：

- 哪些指标可以真实计算
- 哪些指标样本太少
- 哪些统计会产生误导

不要创建页面。

输出建议：

`IPIP0-P2-Country-RIR聚合设计.md`

---

# 六、P2 预设计：字体本地化

只做技术评估。

检查：

- 当前字体来源
- Google Fonts 依赖
- 国内 fallback
- 是否影响中文渲染
- 是否值得本地化

不要修改。

---

# 七、最终输出

生成：

`IPIP0-P2-预设计与GateD等待期报告.md`

报告必须包含：

1. Gate D 当前状态
2. P2 各模块设计状态
3. Organization Entity 归一化方案
4. Prefix Entity CIDR 方案
5. Latency 状态机方案
6. Country/RIR 聚合方案
7. 字体本地化方案
8. 当前数据规模对各模块的支撑程度
9. Gate D 达标后推荐实施顺序

禁止进入实际开发。

禁止修改任何代码、数据库、Collector 或数据。