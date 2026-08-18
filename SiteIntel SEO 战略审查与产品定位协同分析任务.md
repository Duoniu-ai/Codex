# SiteIntel SEO 战略审查与产品定位协同分析任务

## 一、你的角色

你现在不是普通的 SEO 顾问，也不是单纯执行 SEO 修改任务。

请你以以下多个角色同时参与本次分析：

- 高级 SEO 战略顾问
- 搜索引擎技术 SEO 专家
- 网站产品经理
- 数据产品架构师
- 程序化 SEO 专家
- Next.js / SSR SEO 架构顾问
- 独立审查员

你的任务不是迎合我提供的现有方案，而是进行一次**独立、严格、批判性的审查**。

如果现有 SEO 方案存在错误、过度设计、产品定位偏移、SEO 风险或技术逻辑问题，请明确指出。

不要为了“优化 SEO”而牺牲 SiteIntel 的长期产品定位。

---

# 二、项目背景：请先完整理解 SiteIntel

## 1. 网站

SiteIntel

域名：

`https://siteintel.cc`

当前核心定位：

> Website Data Intelligence Platform  
> 网站数据分析与洞察平台

SiteIntel 的核心逻辑不是传统意义上的“站长工具”。

用户输入：

```text
一个网站
一个域名
一个 URL
```

系统对目标进行分析，获取和组织多个维度的数据，例如：

```text
Domain
DNS
IP
ASN
SSL
Technology
Hosting
Infrastructure
Website Metadata
Historical Changes
Relationships
```

最终形成一个网站的数据分析结果。

---

# 三、非常重要：SiteIntel 不是什么

请特别注意以下产品边界。

SiteIntel **不希望被做成**：

```text
WHOIS 查询工具
+
IP 查询工具
+
DNS 查询工具
+
SSL 查询工具
+
网站测速工具
+
备案查询工具
```

这种传统“站长工具集合”。

这些能力即使存在，也应该只是：

> 数据采集能力  
> 数据分析能力  
> 产品底层数据来源

而不是 SiteIntel 的最终产品定位。

SiteIntel 的核心价值应该是：

```text
输入一个网站
        ↓
自动获取多个维度的数据
        ↓
进行结构化分析
        ↓
发现网站之间的数据关联
        ↓
识别技术和基础设施
        ↓
记录历史变化
        ↓
形成可继续探索的数据关系
```

因此：

> SEO 策略必须服务于 SiteIntel 的数据产品，而不能反过来为了 SEO 流量，把 SiteIntel 降级成一个普通站长工具站。

这是本次审查的第一原则。

---

# 四、SiteIntel 当前已经具备的重要基础

根据目前的项目情况：

- 网站使用 Next.js
- 动态页面具备 SSR 能力
- `/website/:domain` 可以生成独立的 Title 和 Description
- 网站可以分析一个具体网站
- 已经存在网站详情页
- 已经存在技术分析相关页面
- 已经存在基础设施分析相关页面
- 已经存在搜索相关页面
- 已经存在工具类页面
- 已经存在案例和指南类页面

当前 SiteIntel 的一个重要页面结构方向是：

```text
Website
    ↓
IP
    ↓
DNS
    ↓
Technology
    ↓
Infrastructure
    ↓
Relationships
    ↓
Historical Changes
```

未来希望形成真正的数据关系网络，而不是孤立的查询结果页。

例如：

```text
website/openai.com
        ↓
IP 页面
        ↓
同 IP 网站
        ↓
ASN
        ↓
Hosting / Infrastructure
        ↓
Technology
        ↓
使用相同技术的网站
```

用户应该能够不断向下探索。

---

# 五、当前 SEO 审计文件

我会同时提供一份文件：

`siteintel_seo_audit.md`

这份文件是针对 SiteIntel 当前全站进行的一次 SEO 诊断。

其中提出了大量修改建议，包括：

- 首页 Title 修改
- Meta Description 修改
- Meta Keywords
- `/bulk` 开放索引
- `/search` 修复 Title
- `/search-intelligence` 修改 H1
- `/website/:domain` 详情页开放索引
- Sitemap 拆分
- BreadcrumbList
- 新增 ICP 查询页面
- Whois 查询
- IP 查询
- 内容 SEO
- 国内搜索引擎适配

这份文件**不是最终方案**。

它现在需要被你进行一次独立审查。

---

# 六、当前存在的重要争议

以下问题是目前我们最需要解决的。

请不要简单回答“可以”或“建议”。

请逐项深入分析。

---

## 争议 1：SiteIntel 是否应该大量使用「站长工具」关键词？

现有方案建议首页和页面中增加：

```text
站长工具
站长查询
域名查询
IP 查询
网站测速
ICP 查询
Whois 查询
```

问题是：

这些关键词确实可能存在搜索流量。

但是：

如果大量使用这些词，会不会导致 SiteIntel 的搜索定位逐渐变成：

> 一个普通站长工具网站？

请分析：

1. SiteIntel 是否应该使用“站长工具”作为核心 SEO 品类词？
2. 是否应该只在部分工具页使用，而不是首页核心定位？
3. 首页最合理的核心关键词体系应该是什么？
4. 如何同时兼顾：
   - SEO 搜索需求
   - 产品定位
   - 用户理解
   - 品牌长期发展

请给出明确结论。

---

## 争议 2：`/website/:domain` 是否应该全部开放搜索引擎索引？

现有方案建议：

> 只要网站分析完成且内容完整，就设置 `index,follow`。

但这里存在巨大风险。

未来 SiteIntel 可能拥有：

```text
10 万
100 万
甚至更多
```

网站分析页面。

其中可能包含：

- 空网站
- 停放域名
- 失效网站
- 分析失败的网站
- 数据很少的网站
- 内容高度重复的网站
- 自动生成但没有独特价值的页面
- CDN 默认域名
- 子域名
- 低价值长尾域名

如果全部 Index，可能产生：

```text
Thin Content
Duplicate Content
Near Duplicate Content
Scaled Content
低质量程序化页面
搜索引擎抓取预算浪费
```

因此，请你重点判断：

### SiteIntel 是否应该建立「动态页面索引准入机制」？

例如：

```text
数据质量评分
        ↓
内容完整度评分
        ↓
页面独特性评分
        ↓
用户价值评分
        ↓
SEO 价值评分
        ↓
INDEX / PARTIAL INDEX / NOINDEX
```

请你设计一个真正可落地的机制。

要求：

不是空泛概念。

请明确：

### A. 什么情况下允许 INDEX？

### B. 什么情况下 NOINDEX？

### C. 是否存在第三种状态？

例如：

```text
INDEX
NOINDEX
CANDIDATE
```

或者其他合理机制。

### D. 是否需要热门网站优先索引？

### E. 是否需要设置最少数据字段要求？

### F. 如何避免几十万个自动页面被搜索引擎判定为低质量内容？

请给出可执行规则。

---

# 七、程序化 SEO：这是本次最重要的部分

SiteIntel 的最大潜力可能不是传统内容 SEO。

而是：

> 数据驱动的程序化 SEO。

未来可能存在如下实体：

```text
Website
Domain
IP
ASN
DNS
Nameserver
Technology
SSL
Hosting
Infrastructure
Organization
Country
Category
Relationship
Historical Change
```

这些实体之间存在关系。

例如：

```text
Website
   ↓
IP
   ↓
Other Websites
```

或者：

```text
Website
   ↓
Technology
   ↓
Other Websites Using Same Technology
```

或者：

```text
Website
   ↓
ASN
   ↓
Infrastructure Provider
```

因此请你重点设计：

# SiteIntel 的「实体型 SEO 架构」

请回答：

### 1. 哪些实体应该成为独立 SEO 页面？

例如：

```text
/website/:domain
/ip/:ip
/asn/:asn
/technology/:slug
/nameserver/:domain
```

哪些值得建立？

哪些不值得建立？

---

### 2. 每种页面的 SEO 搜索价值是什么？

例如：

| 页面 | 用户搜索意图 | SEO 价值 | 是否 Index |
|---|---|---|---|
| Website | 查询具体网站 | 高 | 条件 Index |
| IP | 查询服务器 | 高 | 条件 Index |
| ASN | 网络基础设施研究 | 中 | Index |
| Technology | 技术介绍 | 高 | Index |
| Nameserver | 长尾查询 | 需要判断 | 条件 Index |

请你重新设计完整体系。

---

### 3. SiteIntel 如何建立内部链接网络？

例如：

```text
网站详情页
↓
IP 页面
↓
同 IP 网站
↓
ASN 页面
↓
相关基础设施
↓
使用相同技术的网站
```

请设计一个：

> SEO 内部链接图谱

要求避免：

- 无限循环链接
- 大规模重复链接
- 页面权重稀释
- 自动化垃圾页面

---

# 八、请审查现有 SEO 文件中的所有建议

请读取我提供的：

`siteintel_seo_audit.md`

然后不要直接接受其中结论。

请对文件中的建议逐条进行分类。

请使用下面格式：

| 原建议 | 你的判断 | 是否执行 | 修改方式 | 原因 |
|---|---|---|---|---|

分类必须包括：

```text
A. 可以直接执行
B. 需要修改后执行
C. 不建议执行
D. 当前阶段不需要
```

重点审查：

- 首页 Title
- 首页 Description
- Meta Keywords
- `/bulk`
- `/search`
- `/search-intelligence`
- `/website/:domain`
- `/report`
- `/docs/api`
- Sitemap
- Canonical
- Robots
- Breadcrumb
- JSON-LD
- ICP 查询
- Whois
- IP 查询
- 对比页面
- 指南内容
- 国内搜索引擎适配

不要遗漏。

---

# 九、关于 ICP 查询的问题

现有方案建议建立：

```text
/guide/icp-lookup
```

甚至提出：

> 即使暂时没有 ICP 数据，也可以通过内容截流。

对此需要你明确判断。

请区分：

### 情况 A

真正的：

> ICP 备案查询工具

用户输入域名，返回备案数据。

### 情况 B

内容型：

> ICP 备案查询指南

告诉用户如何查询。

### 情况 C

没有 ICP 查询能力，却创建：

> ICP 备案查询页面

请判断：

这三种页面在：

- 用户体验
- 搜索意图
- SEO
- 长期品牌信任

方面的区别。

并明确 SiteIntel 当前应该采用哪一种。

---

# 十、不要过度关注 Meta Keywords

请分析：

Meta Keywords 是否应该成为 SiteIntel SEO 改造的重点？

请明确 SEO 优先级。

例如：

```text
P0
索引策略
Canonical
Robots
Title
H1
重复内容
动态页面质量

P1
Description
内部链接
结构化数据
Sitemap

P2
内容扩展
实体页面
程序化 SEO

P3
Meta Keywords
```

如果你认为这个排序不合理，请重新排序。

---

# 十一、国内搜索引擎策略

SiteIntel 的主要目标用户是中文用户。

需要考虑：

- 百度
- Bing
- 360
- 搜狗

但同时不要为了国内 SEO 做低质量关键词堆砌。

请设计：

## SiteIntel 中文 SEO 关键词体系

至少分成：

### 第一层：核心品类词

例如：

```text
网站分析
网站数据分析
网站检测
```

### 第二层：功能需求词

例如：

```text
网站技术栈查询
服务器 IP 查询
DNS 分析
```

### 第三层：实体长尾词

例如：

```text
openai.com 网站分析
xxx 用什么技术
xxx IP 地址
```

### 第四层：知识型搜索

例如：

```text
如何查看网站技术栈
如何查询网站服务器
```

请你重新设计。

注意：

不要简单堆砌“站长工具”关键词。

关键词必须和 SiteIntel 的真实能力对应。

---

# 十二、最终请输出一份「独立审查报告」

你的最终回答必须包含以下部分：

---

## 第一部分

### 你对 SiteIntel 产品定位的理解

请用你自己的话重新定义：

> SiteIntel 到底是什么？

如果你的理解只是：

> 网站分析工具

那么说明理解不够。

请给出更准确的产品定义。

---

## 第二部分

### 对现有 `siteintel_seo_audit.md` 的总体评价

请给出：

```text
总体评分：X / 10
```

并分析：

- 正确部分
- 有风险部分
- 错误部分
- 缺失部分

---

## 第三部分

### 原 SEO 文件逐项审查表

必须逐条给出：

```text
原建议
你的判断
执行级别
修改建议
原因
```

---

## 第四部分

### SiteIntel 正确的 SEO 战略

请明确：

```text
品牌 SEO
+
核心功能 SEO
+
内容 SEO
+
实体页面 SEO
+
程序化 SEO
+
数据关系 SEO
```

这几个部分如何组合。

---

## 第五部分

### 动态页面 INDEX / NOINDEX 决策机制

这是重点。

请设计：

```text
数据评分机制
页面质量机制
索引准入规则
Sitemap 准入规则
热门页面优先机制
自动下架机制
```

要求能够真正交给开发团队实现。

---

## 第六部分

### SiteIntel 实体 SEO 页面架构

请输出：

```text
Website
IP
ASN
Technology
DNS
Nameserver
Infrastructure
Relationship
History
```

哪些应该成为独立页面。

给出：

- URL 结构
- 页面核心内容
- 搜索意图
- 是否 Index
- 与其他页面如何关联

---

## 第七部分

### SEO 内部链接网络

请设计 SiteIntel 的：

> 页面 → 数据 → 实体 → 关系 → 继续探索

结构。

同时防止：

- 无限链接
- 重复页面
- 自动垃圾页面
- 权重稀释

---

## 第八部分

### 最终执行优先级

请给出：

```text
P0
立即修复

P1
第一阶段

P2
第二阶段

P3
长期建设
```

并且每个任务必须说明：

```text
为什么做
解决什么问题
开发复杂度
预期收益
SEO 风险
```

---

# 十三、重要限制

在分析过程中，请严格遵守：

### 不要：

- 为了 SEO 强行增加 SiteIntel 没有能力提供的工具
- 建立大量只有模板不同的数据页面
- 让所有 `/website/:domain` 自动 Index
- 把 SiteIntel 首页改造成“万能站长工具”
- 依赖 Meta Keywords 作为核心 SEO 策略
- 使用关键词堆砌
- 创建搜索意图与实际功能不一致的页面

---

# 十四、最后的核心问题

请你最后必须回答：

> **如果 SiteIntel 未来拥有 100 万个网站分析结果，应该如何让这 100 万个页面成为资产，而不是成为 SEO 风险？**

这是本次审查最重要的问题。

请从：

```text
数据质量
页面独特性
实体关系
程序化 SEO
索引控制
内部链接
用户搜索需求
长期增长
```

八个角度回答。

---

# 十五、输出要求

请不要简单给建议。

请输出一份完整的：

# 《SiteIntel SEO 战略独立审查报告》

要求：

1. 必须独立判断，不要迎合现有方案。
2. 必须明确指出错误。
3. 必须给出可以执行的规则。
4. 尽可能站在未来拥有几十万到上百万动态数据页面的角度设计。
5. SEO 必须服务产品，而不是让产品为了 SEO 变形。
6. 如果你发现当前 `siteintel_seo_audit.md` 中还有我没有指出的问题，也请主动补充。
7. 先分析、再判断、最后给出最终架构。
8. 不要只讲概念，必须考虑 Next.js SSR、动态路由、Canonical、Robots、Sitemap、JSON-LD、程序化页面和实际开发成本。

现在，请先完整阅读：

1. 本提示词
2. 我提供的 `siteintel_seo_audit.md`

然后开始进行独立审查。

**不要直接修改代码。**

当前阶段只需要：

> 产品定位校准  
> SEO 战略审查  
> 风险识别  
> 架构设计  
> 最终执行方案建议