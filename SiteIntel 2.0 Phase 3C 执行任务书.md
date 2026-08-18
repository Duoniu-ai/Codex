# SiteIntel 2.0 Phase 3C 执行任务书
## 导航与公共组件用户化统一

请以以下文件及当前线上实际版本作为本阶段核心依据：

1. `SITEINTEL-2.0-PRODUCT-UPGRADE-BLUEPRINT.md`
2. `SITEINTEL-USER-EXPERIENCE-BASELINE-AUDIT.md`
3. `SITEINTEL-2.0-INFORMATION-ARCHITECTURE.md`
4. `SITEINTEL-2.0-PHASE-3A-HOMEPAGE-REPORT.md`
5. `SITEINTEL-2.0-PHASE-3B-REPORT.md`
6. `SITEINTEL-2.0-PHASE-3B2-REPORT.md`
7. `SITEINTEL-2.0-PHASE-3B3-REPORT.md`

同时必须以当前线上已经完成的：

- Phase 3A 首页
- Phase 3B 旗舰分析页
- Phase 3B.1
- Phase 3B.2
- Phase 3B.3

作为实际视觉和用户语言基准。

---

# 一、本阶段目标

正式执行：

> **SiteIntel 2.0 Phase 3C：导航与公共组件用户化统一**

核心目标：

让 SiteIntel 全站的 Header、Navigation、Footer、Breadcrumb、公共 CTA、公共 Card、状态标签、空状态和错误状态，都与已经完成的首页和旗舰分析页保持一致。

当前首页已经告诉用户：

> **看懂任何一个网站**

当前旗舰分析页已经形成：

> 网站概览 → 它如何运行 → 最近发生了什么 → 值得关注 → 关联探索 → 深入数据

因此 3C 要解决的问题是：

> **全站其他页面不能继续使用旧的“管理员 / 系统 / 数据平台”语言。**

---

# 二、严格执行范围

## 允许修改

仅允许修改：

- Header
- Navigation
- Footer
- Breadcrumb
- 公共 CTA / Button
- 公共 Card
- 公共状态标签
- 公共空状态组件
- 公共错误状态组件
- 这些公共组件对应的 i18n
- 必要的公共样式
- 导航层级与链接组织
- 开发者 / 专业用户入口的层级

## 禁止修改

禁止：

- 修改数据库
- migration
- 修改生产数据
- 修改 Discovery
- 修改 Collector
- 修改分析 Pipeline
- 修改 safe-fetch
- 修改 Event Rule
- 修改 Insight Rule
- 修改 Relationship Engine
- 修改 API 合约
- 修改首页核心结构
- 修改 `/website/[domain]` 核心内容结构
- 修改 Entity 页面主体内容
- 修改工具页面主体内容
- 修改搜索页面主体内容
- 进入 Phase 3D

如果某个页面自身存在问题：

> 只记录，不在本阶段顺手改页面主体。

---

# 三、先审查当前公共层

修改前先检查当前：

- `site-header`
- header navigation
- mobile navigation
- footer
- breadcrumb
- button / CTA
- card
- status badge
- empty state
- error state

并确认它们目前被哪些页面复用。

请先输出：

```text
公共组件
→ 使用页面
→ 当前用户可见表达
→ 问题
→ 目标表达
```

然后实施。

---

# 四、Navigation 重新设计

Navigation 必须按照：

> **用户任务**

组织，而不是：

> **系统模块**

重点检查当前一级导航。

推荐方向：

```text
首页
网站分析
探索
工具
```

如果当前真实页面结构更适合其他命名，可以根据现有页面给出最终方案。

但必须解决以下问题：

## 1. API 文档

API 文档属于：

> 开发者 / 专业用户入口

不要让它继续与普通用户核心入口拥有同等权重。

推荐：

- Footer「开发者」
- 或用户菜单中的二级入口
- 或其他明显但低优先级的位置

不要删除。

---

## 2. 批量分析

`/bulk` 属于高级能力。

不要让普通用户把它理解成核心入口。

推荐：

- 登录后功能入口
- 用户中心
- 二级菜单

不要删除。

---

## 3. Report

`/report` 不应该继续表现成一个“系统报告数据库”。

应转译成：

> 报告库 / 探索

并放到适当的二级位置。

---

## 4. Technology / Infrastructure

不要继续让导航变成：

```text
Technology Intelligence
Infrastructure Intelligence
Entity Intelligence
```

应该统一为用户任务语言。

例如：

- 看看网站用了什么技术
- 看看网站运行在哪里
- 探索网站关联

具体名称必须以当前真实页面能力为准。

---

# 五、Header 用户化

Header 必须与首页完全一致。

原则：

## 左侧

品牌：

SiteIntel

保持简洁。

## 中间

只放最重要的用户任务。

不超过必要数量。

## 右侧

保留：

- 登录 / 用户
- 高级功能入口

但不要让：

API
Batch
Docs

抢占普通用户主要视觉位置。

---

# 六、Mobile Navigation

移动端必须重新检查：

- 菜单层级
- 一级入口数量
- 二级入口折叠
- 登录
- 开发者入口
- 高级功能

目标：

普通用户打开菜单后，不应看到一大堆内部产品结构。

---

# 七、Footer 重新组织

Footer 不应该继续像：

> 产品内部模块目录。

建议分组：

## 产品

- 网站分析
- 探索
- 工具

## 资源

- 使用指南
- 案例
- 报告库

## 开发者

- API
- 开发文档

## 关于

- 关于 SiteIntel
- 隐私政策
- 服务条款
- 数据移除 / 认领相关入口

必须基于当前真实页面。

不存在的页面不能编造链接。

---

# 八、公共 CTA 统一

目前不同页面很可能存在：

- Analyze
- Explore
- View
- Open
- Intelligence
- Details

等不同表达。

建立统一规范。

## 主 CTA

优先使用：

- 开始分析
- 查看分析
- 查看变化
- 查看关联
- 查看完整数据

## 次级 CTA

- 查看详情
- 展开全部
- 查看证据
- 深入了解

## 高级操作

- 重新分析
- 监控
- 认领
- 导出

特别检查：

### “认领”

普通用户未必理解“认领”是什么意思。

请评估是否需要改成更明确的：

> 认领网站

或：

> 认领这个网站

并配合简短解释。

不要修改认领后端逻辑。

---

# 九、公共 Card 统一

Card 不再默认采用：

> 模块标题 + 大量数据 + 小按钮

统一向“用户结果型卡片”靠拢。

卡片优先使用：

```text
用户问题 / 用户结果
↓
一句解释
↓
关键值
↓
下一步操作
```

例如：

错误：

> Technology Intelligence  
> 37 technology entities

推荐：

> 网站使用了什么技术？  
> 检测到 WordPress、Cloudflare 和 Google Analytics  
> [查看全部技术]

---

# 十、公共状态标签统一

统一：

- 在线
- 部分可用
- 数据有限
- 暂无数据
- 最近发生变化
- 已分析
- 正在分析

避免：

- Target
- Investigation
- Pending
- Failed
- Unknown Entity

这些内部状态只能存在于调试或高级层。

---

# 十一、公共空状态

全站统一用户语言。

例如：

## 未分析网站

> 还没有这个网站的分析结果  
> 输入网站并开始分析，我们会整理公开可获取的信息。

CTA：

> 开始分析

---

## 数据有限

> 目前可获取的公开信息有限  
> 你仍然可以查看本次检测结果，或稍后重新分析。

---

## 暂无变化

> 暂未发现值得关注的变化  
> 最近的检测没有发现明显的网站配置或技术变化。

---

## 无关联

> 暂未发现可确认的公开关联  
> 这不代表不存在其他关联，只是当前公开数据不足以建立可靠关联。

注意：

这些只是表达规范。

如果当前项目已有更准确的状态逻辑，必须保留真实语义。

---

# 十二、公共错误状态

不要泄漏：

- Target not found
- Investigation failed
- Entity missing
- Provider error

转换成用户语言。

例如：

> 暂时无法获取完整的网站信息  
> 部分检测没有成功完成，但已有结果仍然可以查看。

配：

> 重新分析

或：

> 返回网站分析

---

# 十三、Breadcrumb

Breadcrumb 不应该暴露内部数据库结构。

例如避免：

```text
Website
Target
Entity
Technology Entity
```

推荐：

```text
首页
>
网站分析
>
wordpress.org
```

或者：

```text
首页
>
探索
>
WordPress
```

根据页面真实上下文确定。

---

# 十四、语言统一

中文与英文必须同步。

重点清查：

- nav
- CTA
- button
- footer
- empty state
- error state
- status
- breadcrumb

禁止：

中文已经用户化，英文仍然是旧系统术语。

也禁止：

中文自然，但英文直接逐字机器翻译导致生硬。

---

# 十五、管理员 / 开发者 / 普通用户分层

这是 3C 很重要的一点。

SiteIntel 现在有三类用户：

## 普通用户

想：

> 分析一个网站

看到：

- 网站分析
- 探索
- 工具

## 专业用户

想：

> 深入数据、批量分析、导出、监控

可以进入：

- 报告库
- 批量分析
- 监控
- 导出

## 开发者

想：

> 调 API

进入：

- API 文档
- 开发者资源

不要让三种用户在同一层级竞争。

---

# 十六、全站术语扫描

完成公共层修改后，对用户可见页面进行扫描。

以下内部词汇不应出现在：

Header
Navigation
Footer
Breadcrumb
公共 CTA
公共 Card
公共状态
公共空状态

包括：

```text
Entity
Target
Relationship
Fact
Event
Insight
Investigation
Pipeline
Collection
DiscoveryCandidate
provider_changed
infrastructure_migration
metadata_changed
ip_changed
dns_changed
ssl_changed
```

注意：

高级数据详情、API 文档、原始证据区域不属于本规则范围。

---

# 十七、不要误伤已完成的首页和旗舰报告页

3A、3B、3B.1、3B.2、3B.3 已经验收。

本阶段只允许：

> 让公共组件与它们保持一致。

不得因为 3C 而重新设计：

- 首页 Hero
- 首页价值卡
- 报告页六层结构
- 报告页变化与证据链
- 报告页首屏

如果共享组件修改导致这两个页面出现视觉回归：

立即修正。

---

# 十八、测试与验证

完成后必须执行：

- typecheck
- lint
- test
- build

并做线上验证。

至少验证：

### 首页

Header / Footer 正常。

### `/website/wordpress.org`

Header / Footer 与新产品语言一致。

### Technology 页面

导航与 CTA 统一。

### Tool 页面

导航与公共组件统一。

### Search 页面

统一状态和 CTA。

### `/report`

确认已作为次级入口，不再承担主导航核心位置。

### `/bulk`

确认入口层级合理，不影响真实功能。

### API Docs

确认仍然可以访问，但不占普通用户主导航。

### 中英文

Header / Footer / CTA / 状态全部一致。

### Mobile

至少验证：

- Header
- Menu
- Footer
- CTA
- 长标题
- 长域名
- 长主机名

不会产生横向溢出。

---

# 十九、最终验收标准

3C 完成后，用户应该感觉：

> **SiteIntel 到处都是同一个产品。**

而不是：

首页是一套语言
→ 报告页是一套语言
→ 工具页又是一套语言
→ 搜索页又像后台
→ API 又抢主导航

最终统一为：

```text
看懂任何一个网站
↓
开始分析
↓
查看网站
↓
了解它如何运行
↓
了解最近发生了什么
↓
了解值得关注的信息
↓
探索关联
↓
需要时深入数据
```

---

# 二十、最终交付

完成后生成：

`SITEINTEL-2.0-PHASE-3C-REPORT.md`

报告必须包含：

1. Header 修改
2. Navigation 修改
3. Mobile Navigation 修改
4. Footer 修改
5. Breadcrumb 修改
6. CTA 统一
7. Card 统一
8. Status 统一
9. Empty State 统一
10. Error State 统一
11. 中英文 i18n 修改
12. 页面入口层级变化
13. API / Bulk / Report 的入口处理
14. 用户/专业用户/开发者分层
15. 全站内部术语扫描结果
16. 首页 / 报告页回归验证
17. 测试结果
18. 生产验证结果
19. Git commit
20. 回滚方式

---

# 二十一、最终执行规则

本阶段完成后：

**立即停止。**

不要进入 Phase 3D。

不要重新修改：

- 首页主体
- 旗舰报告页主体
- 数据库
- API
- Discovery
- Event
- Insight
- Relationship

只完成公共导航和公共组件统一。

最终回复我：

1. 是否完成
2. 修改文件
3. 导航前后变化
4. 公共组件前后变化
5. API / Bulk / Report 如何重新分层
6. 中英文验证
7. 移动端验证
8. 测试结果
9. Git commit
10. 报告文件位置