# SITEINTEL SEO EXECUTION ROADMAP

# SiteIntel SEO 分阶段执行路线图

版本：V1.0

---

# 一、当前阶段

当前已分析网站：

> 约 282

当前阶段不追求：

- 程序化 SEO 规模
- 实体页面规模
- 100 万 URL
- 复杂算法

当前目标：

> 修复真实问题，建立正确基础。

---

# P0：立即执行

## 任务 1：全站 Metadata 审计

检查核心页面：

- `/`
- `/search`
- `/bulk`
- `/search-intelligence`
- `/report`
- `/docs/api`
- `/tools/*`
- `/website/:domain`

确保：

- Title 正确
- Description 唯一
- Canonical 正确
- Robots 正确
- OpenGraph 正确

禁止：

- 多个页面使用首页 Metadata
- 页面 Title 与实际功能不一致
- Canonical 指向错误页面

验收：

> 每个核心页面 Metadata 独立。

---

## 任务 2：修复 HTTP 状态码

必须建立：

正常页面 → 200  
真实不存在 → 404  
永久移动 → 301  
临时移动 → 302 / 307  
低价值但真实存在 → 200 + noindex

禁止：

不存在的动态路由 → HTTP 200 → 默认页面

---

## 任务 3：建立 INDEX Gate V1

实现：

`isIndexable(domain)`

逻辑：

```text
IF analysisSuccess = true
AND coreDataCount >= 2
AND isLowQuality = false
AND (
  isPopular = true
  OR analysisCount >= 3
  OR isManuallyApproved = true
)
THEN INDEX
ELSE NOINDEX
```

---

## 任务 4：重建 Sitemap

当前 Sitemap 只允许包含：

- 首页
- 核心功能页
- 高质量工具页
- INDEX 的 Website 页面

禁止：

- NOINDEX 页面进入 Sitemap
- 404 页面进入 Sitemap
- 测试 URL 进入 Sitemap
- 失败分析页面进入 Sitemap

---

## 任务 5：Robots 统一

检查：

- `robots.txt`
- `robots.ts`
- 动态 robots meta

保证规则统一。

---

# P1：第一阶段建设

## 任务 6：建立 Breadcrumb

页面：

- `/website/:domain`
- `/tools/*`
- `/guide/*`

使用：

- 可见 Breadcrumb
- `BreadcrumbList` JSON-LD

---

## 任务 7：建立真实内容 SEO

优先创建：

- 如何分析一个网站
- 如何查看网站技术栈
- 如何查询网站服务器
- 如何查看网站 IP
- DNS 解析是什么意思
- 如何判断网站使用什么技术

内容必须：

- 原创
- 与真实产品能力关联
- 能自然链接到 SiteIntel 功能

---

## 任务 8：建立内部链接基础

Website 页面：

Website  
↓  
IP / DNS / Technology / ASN / 相关分析

当前只建立模块级导航。

不要大规模生成数百个 Related Links。

---

# P2：达到 1,000 网站后

触发条件：

`analyzedWebsites >= 1,000`

执行：

- Technology 页面试点
- ASN 页面试点
- 实体页面数据门槛
- Sitemap 分类
- 内部链接实验

---

# P3：达到 10,000 网站后

执行：

- 实体页面
- 关系页面
- 程序化 SEO
- 质量评分 V2

---

# P4：达到 100,000 网站后

执行：

- Sitemap 分片
- 自动 noindex
- 页面淘汰
- 抓取预算控制
- 实体热度
- 页面质量评分

---

# 本周必做 5 件事

1. 修复所有错误 Metadata。
2. 修复动态页面 404 / Soft 404。
3. 上线 INDEX Gate V1。
4. 重建 Sitemap。
5. 建立 SEO 自动验收脚本。
