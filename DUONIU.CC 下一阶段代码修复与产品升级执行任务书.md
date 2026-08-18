# DUONIU.CC 下一阶段代码修复与产品升级执行任务书

## 一、任务背景

本任务以：

> 《DUONIU-AUDIT-20260816.md》

作为本次工作的**唯一核心审计依据**。

审计已经确认：

- duoniu.cc 技术底座总体达标；
- Nuxt 3 SSR、Provider 适配器、统一数据结构、实体落库、SEO 基建均已完成；
- Phase 0–6 基础能力已经上线；
- staging 环境此前发现的 500 问题已经修复；
- 当前主要问题不再是底层技术架构，而是：
  - 产品入口不完整；
  - 旗舰功能未被突出；
  - 部分功能处于半成品状态；
  - 缺少统一 Verdict 等级体系；
  - 缺少结果解释与行动建议；
  - 历史数据尚未形成完整产品能力；
  - 部分已有功能没有形成统一产品闭环。

因此，本阶段工作的核心原则是：

# 不推翻现有架构，不重新设计整个网站，而是在现有基础上完成产品层升级。

---

# 二、总执行原则

Claude Code 在执行过程中必须遵守以下规则。

## 1. 以现有代码和审计报告为准

禁止：

- 推翻 Nuxt 3 现有架构；
- 重写已稳定运行的 Provider；
- 重复实现已有数据能力；
- 因页面改版重新建设新的数据链路；
- 创建与现有评分系统重复的新评分系统；
- 为视觉效果进行无必要的大规模重构。

优先：

```text
复用已有数据
↓
复用已有 Provider
↓
复用已有 Service
↓
复用已有实体数据
↓
升级产品展示与分析层
```

---

## 2. 禁止再次进行首页大规模视觉重构

当前首页继续保持现有堆叠布局。

明确禁止：

- 恢复此前被否决的七 Tab 首页；
- 未经确认重新设计首页整体结构；
- 大规模替换首页 UI。

允许：

- 在现有结构中增加 Verdict；
- 增加结果解释；
- 增加行动建议；
- 优化模块顺序；
- 增加必要的产品入口；
- 增强已有模块之间的关联。

所有涉及首页整体视觉结构的大改：

```text
必须：
开发 → staging → 预览 → 用户确认
```

未经确认不得直接 promote 到生产。

---

# 三、P0：生产版本同步

## 目标

将已经修复并通过 staging smoke-test 的代码安全同步至生产环境。

当前审计确认：

- staging 最新 HEAD：`ce0e18a`
- staging 14/14 smoke-test 已通过；
- staging 500 已修复；
- 生产仍然运行旧构建；
- 生产缺少最新语言切换删除及相关修复。

相关修复：

```text
b3c8187
3fcce29
ce0e18a
```

---

## 执行要求

按照项目现有部署流程执行。

必须：

1. 确认 staging 当前 HEAD；
2. 再次运行 staging smoke-test；
3. 确认所有关键页面正常；
4. 执行生产 promote；
5. 生产环境重新启动后运行 smoke-test；
6. 检查首页；
7. 检查 `/purity`；
8. 检查 `/history`；
9. 检查导航；
10. 检查 SSR 无报错。

禁止直接修改生产代码。

必须通过：

```text
Git
↓
Build
↓
Staging
↓
Smoke Test
↓
Promote
↓
Production Smoke Test
```

---

# 四、P1：重构全站导航信息架构

## 当前问题

审计确认：

- `/purity` 旗舰页面不在导航；
- `/history` 不在导航；
- 工具菜单只暴露部分工具；
- 已开发功能存在入口缺失。

---

## 目标导航

基于现有页面和现有路由重新整理导航。

建议结构：

```text
首页

IP 环境检测

IP 纯净度检测

IP 历史记录

网络工具
├── DNS 查询
├── Whois 查询
├── Ping 测试
├── Trace 路由
├── BGP 查询
├── CIDR 查询
├── ASN 查询
└── Route 查询

开发者
├── API

帮助
├── 常见问题
└── 关于我们
```

---

## 执行要求

修改：

```text
app/layouts/default.vue
```

或当前项目实际导航实现文件。

要求：

- 不创建重复导航；
- 保持现有响应式逻辑；
- 保持移动端导航；
- 保持当前主题切换；
- 保持面包屑逻辑；
- 检查所有导航链接；
- 不产生 404；
- 不删除 `/as/[asn]` 兼容跳转。

---

# 五、P1：完成 `/purity` 旗舰功能

## 当前状态

审计确认：

当前 purity 页面已经具备：

- 0–100 数值评分；
- good/warn/bad 状态；
- 风险 flags。

但仍存在：

- 无统一字母等级；
- 黑名单检测卡待建；
- AI 出口检测卡待建；
- 无历史对比；
- 无重新检测；
- 无导出；
- 无完整结果解释；
- 无行动建议。

---

# 六、建立统一 Verdict 等级体系

## 核心原则

禁止创建第二套评分算法。

现有：

```text
Risk / Purity Score
0 - 100
```

继续作为唯一底层评分。

新增：

```text
Verdict Mapping Layer
```

结构：

```text
原始检测数据
        ↓
风险因子
        ↓
现有 0-100 Score
        ↓
统一 Verdict Mapping
        ↓
S+ / S / A / B / C
```

---

## 要求

创建统一映射逻辑。

禁止：

- 首页一套等级；
- purity 一套等级；
- API 一套等级；
- history 一套等级。

必须保证：

```text
同一个 Score
=
同一个 Verdict
```

建议建立独立的共享逻辑，例如：

```text
app/utils/verdict.ts
```

实际路径根据现有项目架构确定。

---

## Verdict 必须包含

每个 Verdict 输出统一数据结构：

```text
score
grade
status
label
summary
```

例如：

```text
score: 82
grade: "A"
status: "good"
label: "可信度良好"
summary: "未发现明显高风险信号，但仍存在部分网络环境风险因素。"
```

具体等级区间由 Claude Code 根据现有评分算法设计合理映射。

禁止随意改变底层评分含义。

---

# 七、完成 Purity 页面两个待建功能

## 1. 黑名单检测

必须优先复用现有首页已经使用的数据链路。

禁止：

```text
首页查询一套
Purity 再查询一套
```

必须：

```text
统一 Provider
↓
统一数据结果
↓
首页 / Purity / API 复用
```

---

## 2. AI 出口检测

必须优先检查现有：

- ChatGPT 检测；
- Claude 检测；
- Gemini 检测；
- DeepSeek 检测。

已有逻辑。

Purity 页面应直接复用现有检测能力。

展示：

```text
平台
状态
检测结果
必要说明
```

不要重新建立独立检测系统。

---

# 八、P2：建立“结果解释 + 行动建议”层

这是本阶段重要任务。

当前网站大量页面存在：

```text
检测
↓
显示数据
↓
结束
```

需要升级为：

```text
检测
↓
数据
↓
风险解释
↓
Verdict
↓
Action
```

---

## 必须新增

### 1. 为什么得到这个结果

例如：

```text
✓ 未发现公开黑名单记录

⚠ 当前 ASN 属于数据中心网络

⚠ 非住宅网络

✓ 未发现 WebRTC 泄漏

✓ AI 平台部分可访问
```

要求：

- 结果必须来源于真实检测数据；
- 不允许 AI 编造风险原因；
- 每条结论必须能够追溯到具体检测字段。

---

## 2. 行动建议

根据检测结果生成：

```text
What should I do?
```

例如：

```text
当前 IP 更适合：

✓ 普通服务器部署
✓ 开发测试

需要谨慎：

⚠ AI 平台长期使用
⚠ 社交平台账号环境

不建议：

✕ 高风险业务
```

要求：

- 建议必须由规则系统生成；
- 禁止使用纯随机文案；
- 禁止脱离检测数据；
- 保持规则可维护。

建议：

```text
Detection Result
↓
Rule Engine
↓
Recommendation
```

建立独立规则映射层。

---

# 九、P2：建设 `/history` IP 历史时间线

## 当前状态

审计确认：

- Phase 6 已经开始数据积累；
- 页面已有当前快照；
- 时间线仍然是占位。

---

## 第一阶段目标

不要直接开发复杂大数据系统。

先实现：

# IP 历史时间线

展示历史快照。

每次快照至少比较：

```text
时间
IP
ASN
ISP
国家/地区
IP 类型
风险评分
Verdict
黑名单状态
AI 平台状态
```

---

## 必须支持变化识别

自动识别：

```text
ASN Changed

ISP Changed

Country Changed

IP Type Changed

Risk Score Increased

Risk Score Decreased

Blacklist Status Changed

AI Availability Changed
```

---

## 数据规则

禁止重新设计新的快照数据库。

必须优先检查：

- Phase 6 实体数据；
- 已有 Snapshot；
- 已有数据库 Schema。

在现有数据模型上扩展。

如果现有数据不足：

允许增加字段。

但必须：

- migration；
- backward compatibility；
- 不破坏旧数据；
- 明确数据版本。

---

# 十、P3：DNS 多解析器对比

新增：

# Multi Resolver Comparison

至少支持根据现有能力评估接入：

```text
Cloudflare DNS
Google DNS
AliDNS
DNSPod
Quad9
```

最终展示：

| Resolver | A | AAAA | Response Time | TTL | Status |
|---|---|---|---|---|---|

重点功能：

- 多 DNS 结果是否一致；
- 是否存在解析差异；
- 响应时间差异；
- 异常结果提示。

禁止只展示静态数据。

---

# 十一、P3：Trace / Route 可视化

项目首页已经存在 Leaflet 地图能力。

优先复用。

实现：

```text
Source
↓
Hop
↓
Hop
↓
Transit ASN
↓
Destination
```

展示：

- Hop；
- IP；
- ASN；
- ISP；
- Location；
- RTT；
- 网络变化。

要求：

地图不是唯一展示方式。

必须同时保留：

```text
Table View
+
Map View
```

避免部分 IP 无法定位时页面失效。

---

# 十二、报告与结果导出

暂不进行复杂商业化。

但需要为后续功能预留：

```text
Copy Result
Export JSON
```

第一阶段至少实现：

### Copy Analysis

复制：

- IP；
- Score；
- Verdict；
- 风险因素；
- 关键检测结果；
- 检测时间。

格式必须适合：

- ChatGPT；
- Claude；
- DeepSeek；
- 普通文本分享。

---

# 十三、SEO 继续增强

现有 SEO 基础不允许破坏。

保持：

- SSR；
- sitemap；
- robots；
- llms.txt；
- JSON-LD；
- Canonical；
- 页面 Meta。

重点增强：

## `/purity`

增加：

- 评分方法说明；
- 检测维度；
- Verdict 等级解释；
- FAQ；
- 方法论内容。

目的：

让 `/purity` 不只是工具页面，而成为：

> IP 纯净度检测方法论与实际检测页面。

---

# 十四、废弃组件处理

审计确认：

```text
app/features/home/
```

存在约 9 个无引用组件。

包括：

- HomeSummary；
- HomeTabs；
- 旧 7 Tab 相关组件。

---

## 执行规则

第一步：

```text
全项目 grep
```

确认：

- 无 import；
- 无动态引用；
- 无路由引用；
- 无测试依赖。

第二步：

```text
git tag
```

或建立删除前安全 commit。

第三步：

删除无引用代码。

第四步：

执行：

```text
pnpm build
pnpm test
smoke-test
```

禁止未经确认直接删除可能存在隐藏依赖的代码。

---

# 十五、统一代码规范

本阶段新增功能必须：

## 数据层

```text
Provider
Service
Entity
API
```

保持现有架构。

禁止在 Vue 页面中直接：

- 请求第三方 API；
- 编写复杂业务规则；
- 写重复数据转换逻辑。

---

## UI 层

组件负责：

```text
Display
Interaction
State
```

业务逻辑必须下沉。

---

## 规则系统

以下内容必须集中管理：

- Verdict Mapping；
- Risk Explanation；
- Action Recommendation；
- History Change Detection。

禁止散落在多个 `.vue` 文件中。

---

# 十六、禁止事项

本阶段明确禁止：

### 1.

禁止推翻 Nuxt 3 架构。

### 2.

禁止重新建设已有 Provider。

### 3.

禁止为首页重新设计七 Tab。

### 4.

禁止创建第二套评分算法。

### 5.

禁止为了增加功能而制造重复数据请求。

### 6.

禁止直接修改生产环境代码。

### 7.

禁止未经 staging 测试直接上线。

### 8.

禁止删除 `/as/[asn]` 兼容跳转。

### 9.

禁止破坏：

- SEO；
- Sitemap；
- JSON-LD；
- robots；
- SSR；
- 百度统计。

---

# 十七、执行顺序

严格按照以下顺序。

## Step 0

生产版本同步。

---

## Step 1

导航重构。

完成：

```text
/purity
/history
全部工具入口
```

---

## Step 2

Purity 完整化。

完成：

```text
Blacklist
AI Detection
Verdict
Risk Explanation
Action Recommendation
```

---

## Step 3

建立统一 Verdict 系统。

要求：

首页、Purity、History、API 可复用。

---

## Step 4

History 时间线。

完成：

```text
Snapshot
Timeline
Change Detection
```

---

## Step 5

DNS 多解析器对比。

---

## Step 6

Trace / Route 可视化。

---

## Step 7

Copy Analysis / JSON Export。

---

## Step 8

清理废弃 Home Tabs 代码。

必须确认无引用后删除。

---

# 十八、每完成一个阶段必须输出

Claude Code 不要只输出：

> 已完成。

每个阶段必须生成：

```text
CHANGELOG
```

内容包括：

### 修改文件

```text
file path
```

### 新增文件

```text
file path
```

### 删除文件

```text
file path
```

### 数据库变更

```text
migration
schema
```

### API 变化

```text
endpoint
request
response
```

### 验证结果

```text
Build
SSR
Smoke Test
Production Test
```

---

# 十九、最终验收标准

完成后必须满足：

## 产品

- [ ] `/purity` 成为明确旗舰功能；
- [ ] `/history` 可从导航进入；
- [ ] 所有现有工具有合理入口；
- [ ] 用户可以获得统一 Verdict；
- [ ] 用户知道为什么得到这个 Verdict；
- [ ] 用户可以获得下一步行动建议；
- [ ] IP 历史数据可以形成时间线；
- [ ] 关键变化可以自动识别。

## 技术

- [ ] Nuxt Build 通过；
- [ ] TypeScript 无新增错误；
- [ ] SSR 正常；
- [ ] staging 正常；
- [ ] 14 页 smoke-test 通过；
- [ ] 生产环境正常；
- [ ] 无新增未使用代码；
- [ ] 无重复 Provider；
- [ ] 无重复评分逻辑。

## SEO

- [ ] Sitemap 正常；
- [ ] robots 正常；
- [ ] JSON-LD 正常；
- [ ] Meta 正常；
- [ ] Canonical 正常；
- [ ] `/purity` SEO 内容增强。

---

# 二十、最终目标

本阶段不是增加更多零散工具。

最终目标是把 duoniu.cc 从：

```text
IP 查询工具集合
```

升级为：

# IP Environment & Trust Intelligence Platform

完整用户流程：

```text
输入 IP
    ↓
自动识别网络环境
    ↓
基础信息分析
    ↓
风险检测
    ↓
平台可用性检测
    ↓
统一 Verdict
    ↓
为什么
    ↓
应该怎么办
    ↓
历史变化
```

---

# 最终执行原则

Claude Code 在开始工作前必须：

1. 读取本执行任务书；
2. 读取当前项目架构；
3. 检查当前 Git HEAD；
4. 检查 staging 与 production 状态；
5. 检查现有 Provider、Service、Entity；
6. 识别哪些能力已经存在；
7. 优先复用；
8. 再进行修改。

**禁止根据任务书想象项目结构。**

必须以：

> 当前真实代码 + 当前数据库 Schema + 当前部署状态

为实际执行依据。

如果任务书中的建议与当前代码存在冲突：

```text
先报告差异
↓
说明影响
↓
给出最小风险方案
↓
不要直接推翻现有系统
```

# 本任务的核心不是“重新开发 duoniu.cc”。

# 而是：

> **在已经完成的技术底座上，把已有数据能力整合成真正清晰、可理解、可解释、可行动的产品。**