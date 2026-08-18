# SITEINTEL SEO MASTER PLAN V2

# SiteIntel SEO 总体战略与产品定位总纲

版本：V2.0  
状态：正式执行战略文件  
适用项目：siteintel.cc

---

# 一、文件目的

本文件用于统一 SiteIntel 的：

- 产品定位
- SEO 战略
- 页面索引策略
- 动态页面策略
- 实体页面策略
- 内容 SEO
- 程序化 SEO
- 数据规模增长路线

本文件是 SiteIntel SEO 工作的最高级战略文件。

后续任何页面开发、SEO 修改、新工具建设、动态页面生成、Sitemap 修改、Robots 修改、内容扩展、实体页面建设，都不得违背本文件定义的核心原则。

---

# 二、SiteIntel 产品定位

## 2.1 产品定义

SiteIntel 是：

> Website Data Intelligence Platform  
> 网站数据分析与智能洞察平台

SiteIntel 的核心不是提供单点查询，而是：

Website / Domain / URL  
→ Data Collection  
→ Data Normalization  
→ Website Analysis  
→ Entity Recognition  
→ Relationship Discovery  
→ Structured Intelligence  
→ Continuous Exploration

用户输入一个网站后，不应该只得到一个 IP、一条 WHOIS、一组 DNS 或一个技术栈，而应该得到关于该网站的结构化数据、技术特征、基础设施信息、实体关系以及可继续探索的数据路径。

---

# 三、SiteIntel 不应被定义为什么

SiteIntel 不应被整体定位为：

- 万能站长工具
- WHOIS 查询站
- IP 查询站
- DNS 查询站
- SSL 查询站
- 网站测速站
- ICP 备案查询站

这些功能可以存在，但属于数据能力和功能能力，是 SiteIntel 数据智能产品的一部分，而不是核心品牌定义。

---

# 四、SEO 核心原则

## 原则 1：SEO 服务产品

错误模式：

寻找关键词 → 制造页面 → 产品被 SEO 改造成工具站

正确模式：

真实产品能力 → 真实用户需求 → 真实搜索意图 → 高质量页面 → SEO 增长

## 原则 2：不为了流量制造不存在的能力

禁止：

- 没有 ICP 数据却建立 ICP 查询工具
- 没有测速能力却建立网站测速工具
- 没有安全检测能力却建立网站安全检测

如果没有真实能力，只能创建指南、教程、方法论或知识内容，不能伪装成功能工具。

## 原则 3：程序化页面必须先有价值，再开放索引

禁止：

分析一个域名 → 自动生成页面 → 全部 INDEX

正确逻辑：

数据生成 → 基础质量检查 → 判断页面价值 → INDEX / NOINDEX → 满足条件后加入 Sitemap

---

# 五、SEO 五层体系

## 第一层：品牌 SEO

目标：

> SiteIntel = 网站数据分析与智能洞察

核心页面：

- `/`
- `/about`
- `/docs`
- `/guide`

首页不以“免费站长工具”“万能站长工具”“域名查询工具大全”作为核心定位。

## 第二层：功能 SEO

针对真实产品能力建立搜索入口，例如：

- `/tools/website-analysis`
- `/tools/dns`
- `/tools/technology`
- `/tools/ip`

这些页面允许围绕真实搜索需求进行优化：

- 网站分析
- DNS 查询
- IP 查询
- 技术栈检测
- 网站检测

允许在功能和内容场景中适度使用：

- 站长工具
- 站长常用工具
- 网站分析工具

但不得改变首页和品牌定位。

## 第三层：内容 SEO

建立真实知识内容，例如：

- 如何查询网站技术栈
- 如何查看网站服务器 IP
- DNS 如何解析
- 如何判断网站是否使用 Cloudflare
- 如何分析一个网站
- 网站技术架构分析方法

内容必须服务于搜索需求、产品理解和 SiteIntel 内部功能。

## 第四层：动态 Website SEO

核心页面：

`/website/:domain`

不是所有网站详情页都允许索引，必须采用阶段性索引策略。

---

# 六、当前阶段动态页面 INDEX 规则

当前 SiteIntel 数据规模约为：

> 282 个已分析网站

因此当前不建立复杂评分模型，采用：

# Simple Index Gate V1

## 第一关：分析是否成功

至少满足：

- 分析任务成功
- IP、DNS、Technology 至少两个核心维度存在有效数据

否则：`NOINDEX`

## 第二关：排除低价值目标

以下直接 `NOINDEX`：

- 停放域名
- 测试域名
- localhost
- 明显测试子域名
- CDN 默认域名
- 无实际内容页面
- 分析错误页面
- 数据异常页面

## 第三关：索引资格

满足以下任意条件：

A. 预置热门网站  
B. 用户主动分析次数 ≥ 3  
C. 人工标记为高价值  
D. 后续具备明确真实搜索需求

则：`INDEX`

否则：`NOINDEX`

---

# 七、当前 INDEX 状态模型

当前只使用：

- `INDEX`
- `NOINDEX`

暂不建立复杂五维评分。

---

# 八、实体页面建设策略

未来 SiteIntel 可能拥有：

- Technology
- IP
- ASN
- Hosting
- Nameserver
- DNS
- SSL
- Infrastructure

等实体页面。

但当前数据量不足，因此采用实体页面延迟开放策略。

## 当前阶段：< 1,000 网站

原则：

- Website 页面为主
- 实体信息作为详情模块
- 不大规模生成实体独立页面

## 成长阶段：1,000–10,000 网站

允许试点，但必须达到最低数据门槛：

| 实体 | 最低关联网站数量 |
|---|---:|
| Technology | 50 |
| ASN | 20 |
| Hosting | 20 |
| Nameserver | 20 |

未达到门槛时：

- 不生成公开实体页，或
- 页面保持 `NOINDEX`

---

# 九、关键词策略

## 首页

核心方向：

- 网站分析
- 网站数据分析
- 网站检测
- 网站技术分析
- 网站基础设施分析

避免首页变成：

- 免费站长工具
- 域名查询工具
- WHOIS 查询
- IP 查询

## 工具页

允许围绕真实能力：

- IP 查询
- DNS 查询
- 网站分析工具
- 技术栈检测

进行 SEO。

## 内容页

允许覆盖：

- 站长常用网站分析方法
- 网站分析工具推荐
- 如何检测网站技术

关键词必须与真实内容和真实能力一致。

---

# 十、SiteIntel SEO 发展阶段

## Phase 1：282 → 1,000

重点：

- 技术 SEO 修复
- 基础 TDK
- Canonical
- 动态 Metadata
- INDEX / NOINDEX
- 404 / Soft 404
- Robots
- Sitemap

不建立：

- 五维评分
- 大规模实体页面
- 大规模程序化 SEO

## Phase 2：1,000 → 10,000

重点：

- 实体页面试点
- 内部链接试点
- Sitemap 分类
- 搜索表现验证
- 简单质量评分

## Phase 3：10,000 → 100,000

重点：

- 程序化 SEO
- 实体 SEO
- 数据关系页面
- 质量评分

## Phase 4：100,000 → 1,000,000

重点：

- 抓取预算
- Sitemap 分片
- 自动质量淘汰
- 页面生命周期
- 热门实体优先
- 关系网络
- 高级评分机制

---

# 十一、最终原则

SiteIntel SEO 的发展顺序必须是：

先修复  
↓  
先建立高质量页面  
↓  
验证搜索需求  
↓  
验证实体页面  
↓  
扩大数据规模  
↓  
程序化 SEO  
↓  
形成数据资产

而不是：

先制造 100 万页面  
↓  
再想办法让搜索引擎收录

最终目标：

> 让每一个被 INDEX 的页面，都成为 SiteIntel 的长期数据资产。
