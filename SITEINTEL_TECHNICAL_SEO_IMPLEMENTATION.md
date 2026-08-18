# SITEINTEL TECHNICAL SEO IMPLEMENTATION

# Next.js 技术 SEO 实施规范

版本：V1.0

---

# 一、目标

本文件规定 SiteIntel 在 Next.js 中实现：

- Metadata
- Dynamic Metadata
- Canonical
- Robots
- Sitemap
- 404
- JSON-LD
- Dynamic Route SEO

---

# 二、Metadata

所有页面必须使用 Next.js Metadata API。

静态页面使用独立 Metadata。

动态页面：

`app/website/[domain]/page.tsx`

必须使用：

`generateMetadata()`

动态生成：

- Title
- Description
- Canonical
- OpenGraph
- Robots

---

# 三、动态页面规则

禁止所有动态页面共享 Metadata。

每个 Website 页面必须基于真实数据生成。

示例：

Title：

`openai.com 网站分析 | IP、技术栈与基础设施 - SiteIntel`

Description：

`查看 openai.com 的网站基础信息、服务器 IP、DNS、技术栈及基础设施分析。`

如果数据不存在：

- 返回 404，或
- 根据真实业务状态返回 200 + noindex

不得生成虚假 SEO 内容。

---

# 四、404 规则

动态路由必须明确区分：

- 网站不存在 → `notFound()`
- 分析记录不存在 → `notFound()`
- 参数非法 → `notFound()`

必须确保真实 HTTP 状态为：

`404`

禁止：

`200 + 默认页面`

---

# 五、Soft 404 检测

禁止：

任意域名  
→ `/website/xxxxx`  
→ HTTP 200  
→ 默认页面

开发完成后必须测试：

`/website/nonexistent-test-domain-123456.com`

预期：

`404`

---

# 六、Canonical

每个页面 Canonical 必须指向自身标准 URL。

示例：

`https://siteintel.cc/website/openai.com`

禁止所有页面 Canonical 指向首页。

动态页面必须统一：

- domain normalize
- lowercase
- remove protocol
- remove invalid trailing slash
- generate canonical

---

# 七、Robots

动态 Website 页面：

```text
IF isIndexable = true
→ index,follow

ELSE
→ noindex,follow
```

NOINDEX 页面通常仍允许 follow，用于允许搜索引擎发现正常内部链接。

---

# 八、Sitemap

当前推荐使用：

`app/sitemap.ts`

当前阶段生成：

- Static Pages
- Indexable Website Pages

未来 URL 增加后拆分：

- sitemap-static
- sitemap-website
- sitemap-entity

Sitemap 只允许包含可 INDEX URL。

---

# 九、ISR

对于：

`/website/:domain`

如果数据更新频率不是实时秒级，可以使用 ISR。

示例：

`revalidate = 3600`

或：

`revalidate = 86400`

具体值必须根据：

- 数据更新频率
- 分析任务更新频率
- 用户实时性需求

决定。

不得机械固定。

---

# 十、JSON-LD

首页：

- Organization
- WebSite

工具页：

- WebApplication

文章：

- Article

详情页：

- WebPage
- BreadcrumbList

禁止：

- 虚构 Review
- 虚构 Rating
- 虚构 FAQ

---

# 十一、Breadcrumb

路径示例：

首页  
↓  
网站分析  
↓  
openai.com

JSON-LD 与可见 Breadcrumb 必须一致。

---

# 十二、分页

未来如果“相关网站”数量较大，必须分页。

禁止单页输出 500+ 个相关链接。

建议：

每页 20 个。

---

# 十三、链接策略

不使用机械规则：

“每页最多 50 个链接”

正确方式：

- 高价值实体：正常内部链接
- 大量相关网站：分页
- 低价值或功能性链接：按业务需要处理

核心目标：

> 用户可以继续探索，但页面不能变成链接农场。

---

# 十四、Meta Keywords

不作为 SEO 开发重点。

优先级：

P0：

- HTTP
- Canonical
- Robots
- Index
- Title
- 404

P1：

- Description
- H1
- Content
- Internal Links

P2：

- Structured Data
- Sitemap
- Breadcrumb

P3：

- Meta Keywords

---

# 十五、推荐目录结构

根据现有项目实际目录调整：

```text
app/
├── page.tsx
├── robots.ts
├── sitemap.ts
├── website/
│   └── [domain]/
│       ├── page.tsx
│       └── not-found.tsx
├── tools/
├── guide/
└── api/
```

建议集中 SEO 逻辑：

```text
lib/seo/
├── metadata.ts
├── canonical.ts
├── indexability.ts
├── structured-data.ts
└── sitemap.ts
```

---

# 十六、禁止事项

禁止：

- 所有页面使用首页 Description
- 不存在页面返回 HTTP 200
- NOINDEX URL 加入 Sitemap
- 虚假结构化数据
- 动态页面未判断数据质量直接 INDEX
