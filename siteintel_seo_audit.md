# SiteIntel (siteintel.cc) 全站 SEO 诊断与修改规划

> 针对国内搜索环境（百度、必应中国、360、搜狗）的全面分析与优化建议

---

## 一、当前全站 SEO 现状总览

| 页面 | Title | Description | Robots | 主要问题 |
|------|-------|-------------|--------|----------|
| `/` 首页 | `看懂任何一个网站 \| SiteIntel` | 有 | `index,follow` | 标题过于文艺，缺核心商业词 |
| `/how-it-works` | `SiteIntel 如何分析网站数据...` | 有 | `index,follow` | 较好 |
| `/website-intelligence` | `网站数据分析与网站情报...` | 有 | `index,follow` | 较好 |
| `/relationship-intelligence` | `网站关联分析与实体关系情报...` | 有 | `index,follow` | 较好 |
| `/search-intelligence` | `SEO搜索数据分析与搜索情报...` | 有 | `index,follow` | **H1 为英文** |
| `/infrastructure-intelligence` | `网站基础设施分析 - IP...` | 有 | `index,follow` | 较好 |
| `/cases/...` | `基础设施迁移检测案例...` | 有 | `index,follow` | 较好 |
| `/guides/...` | `网站迁移检测指南...` | 有 | `index,follow` | 较好 |
| `/tools/dns` | `DNS查询与解析分析工具...` | 有 | `index,follow` | 较好 |
| `/tools/website-analysis` | `网站分析工具...` | 有 | `index,follow` | 较好 |
| `/website/wordpress.org` | `wordpress.org 网站分析...` | 有 | `index,follow` | 较好 |
| `/website/openai.com` | `openai.com 网站分析...` | 有 | **`noindex`** | **异常屏蔽** |
| `/bulk` | `批量分析 \| SiteIntel` | **默认描述** | **`noindex,nofollow`** | 缺优化、被屏蔽 |
| `/report` | `近期报告 \| SiteIntel` | 有 | `noindex,follow` | 被屏蔽 |
| `/search` | `登录 SiteIntel \| SiteIntel` | **默认描述** | `noindex,nofollow` | **标题错配** |
| `/login` | `登录 SiteIntel \| SiteIntel` | **默认描述** | `noindex,nofollow` | 被屏蔽 |
| `/docs/api` | `SiteIntel 开放 API \| SiteIntel` | **默认描述** | `index,follow` | 缺描述 |

### 关键发现

- 站点采用 Next.js，**动态路由（`/website/:domain`）已实现 SSR**，能够返回独立 Title/Description，这是优势。
- 但 **4 个页面使用了完全相同的默认 Description**（首页 slogan），且 **2 个页面 Title 错配或过于单薄**。
- `/website/openai.com` 被设置了 `noindex`，而 `/website/wordpress.org` 却是 `index`，**规则不一致**。
- `/bulk` 被 `noindex,nofollow`，浪费了"批量网站分析"这一高意图关键词的着陆页价值。

---

## 二、不符合国内用户的 6 大类问题

### 1. 标题（Title）策略偏离国内搜索意图

**问题表现：**

- 首页标题"看懂任何一个网站"是**品牌向的 slogan**，而非**搜索意图向的关键词**。国内用户不会搜索"看懂网站"，而是搜索"网站分析工具"、"域名信息查询"、"在线网站检测"。
- `/bulk` 仅使用"批量分析 \| SiteIntel"，未覆盖"批量网站分析工具"、"批量域名查询"等词。
- `/search` 页面 Title 显示为"**登录 SiteIntel**"，存在明显的**标题与内容错配**（该页面实际为搜索功能页，却被标记为登录页）。

**国内用户习惯：**

- 百度用户偏好**明确的功能词 + 修饰词**（如"免费"、"在线"、"查询"、"工具"）。
- 对比：国外用户可能接受"Understand any website"，但国内等效搜索行为是"查网站信息"、"网站技术栈查询"。

### 2. Meta Description 大量缺省，未做差异化

**问题表现：**

- `/docs/api`、`/search`、`/login`、`/bulk` 等页面复用了首页的 slogan 描述："输入一个网站，了解它是什么..."。
- 描述中**缺少行动号召（CTA）**和**差异化卖点**，在百度 SERP 中点击率（CTR）会受损。
- 描述长度未针对国内搜索引擎优化（百度对描述的截断逻辑与 Google 不同，通常更短）。

### 3. 核心关键词布局缺失国内高频词

**问题表现：**

- Meta Keywords 和页面内容中**缺少国内站长生态的高频词**：
  - **站长工具**、**站长查询**（国内极强的品类词）
  - **ICP 备案查询**、**Whois 查询**、**域名备案**（国内独有的强需求）
  - **服务器 IP 查询**、**同 IP 网站查询**（国内 SEO 从业者常用）
  - **网站测速**、**网站安全检测**（相关需求词）
- 当前关键词列表包含大量英文词汇（`website analysis, domain intelligence, tech stack detection`），**国内用户几乎不用英文搜索这些需求**。

### 4. 页面层级与索引策略矛盾

**问题表现：**

- `/report`（近期分析报告列表）设置为 `noindex,follow`——**浪费了聚合页价值**。报告列表页天然适合捕获"最新网站分析"、"xxx 网站最新数据"等长尾流量。
- `/website/openai.com` 被 `noindex`，而同类型的 `/website/wordpress.org` 却是 `index`。如果这是**按分析次数或网站热度动态决定**的，规则不透明，会导致大量潜在高价值页面被误屏蔽。
- `/bulk` 被 `noindex,nofollow`，但"批量分析"是 B 端用户的高意图词，应开放索引。

### 5. 内容结构与语义标记问题

**问题表现：**

- `/search-intelligence` 的 **H1 为英文"Search Intelligence"**，而页面 Title 和正文均为中文。国内搜索引擎对 H1 的权重分配较高，英文 H1 会削弱"搜索情报"、"SEO 数据分析"等中文词的排名信号。
- `/bulk` 页面**零个 H2 标签**，内容结构单薄，百度难以识别页面主题的层级关系。
- 缺少**面包屑导航（BreadcrumbList）**的结构化数据，不利于百度在搜索结果中展示层级路径。

### 6. 国内生态适配不足

**问题表现：**

- **无 ICP 备案展示**：国内用户对未备案的 `.cc` 域名存在天然信任门槛，页面应在 footer 明确标注运营主体与备案号（如有）。
- **无国内社交媒体 Open Graph 适配**：当前 OG 标签针对 Facebook/Twitter，未配置**微信分享卡片**（国内无 og 平台，但需确保分享时描述可读）。
- **Sitemap 未拆分**：单一大 sitemap 未区分优先级，且未包含高价值的 `/website/:domain` 详情页 URL（这些页面是长尾流量的核心）。

---

## 三、分页面修改规划

### 1. 首页 `/`

| 元素 | 当前状态 | 修改建议 |
|------|----------|----------|
| **Title** | `看懂任何一个网站 \| SiteIntel` | `SiteIntel - 免费网站分析工具 \| 域名查询·技术栈检测·IP DNS 分析` |
| **Meta Description** | `输入一个网站，了解它是什么...` | `免费在线分析任意网站：查询域名 IP、DNS、SSL 证书、技术栈、服务器及基础设施变化。SiteIntel 提供带证据的网站情报与关联分析。` |
| **Meta Keywords** | 含大量英文词 | `网站分析工具, 域名查询, 站长工具, 网站技术栈检测, IP 查询, DNS 分析, SSL 证书查询, 免费网站检测, 基础设施分析` |
| **H1** | `看懂任何一个网站` | **保持不变**（可作为品牌 slogan，但建议在 H1 下方增加带关键词的副标题） |
| **H2 补充** | 现有 5 个 | 增加 `为什么使用 SiteIntel 进行网站分析`、`SiteIntel 与传统站长工具的区别` 等带关键词的 H2 |

**修改理由：** 首页需要同时承载品牌词与品类词。将核心功能词前置，"免费"和"在线"是百度高点击修饰词。

---

### 2. 批量分析 `/bulk`

| 元素 | 当前状态 | 修改建议 |
|------|----------|----------|
| **Robots** | `noindex,nofollow` | 改为 `index,follow` |
| **Title** | `批量分析 \| SiteIntel` | `批量网站分析工具 - 批量域名查询与检测 \| SiteIntel` |
| **Meta Description** | 默认 slogan | `支持批量输入域名，一键分析多个网站的 IP、DNS、SSL、技术栈与基础设施。适合站长、安全研究员与营销团队批量排查。` |
| **Canonical** | 错误指向首页 `/` | 修正为 `https://siteintel.cc/bulk` |
| **H2 补充** | 0 个 | 增加 `批量分析能查到什么`、`适用场景`、`如何批量查询网站信息` |

**修改理由：** 该页面是工具型着陆页，"批量"是强修饰词，不应屏蔽。

---

### 3. 搜索页 `/search`

| 元素 | 当前状态 | 修改建议 |
|------|----------|----------|
| **Title** | `登录 SiteIntel \| SiteIntel`（**错配！**） | `搜索网站情报 - 域名与网站数据分析查询 \| SiteIntel` |
| **Meta Description** | 默认 slogan | `在 SiteIntel 数据库中搜索已分析的网站，查看域名情报、技术变更与基础设施关联。` |
| **Robots** | `noindex,nofollow` | 评估业务逻辑后，若页面有实际内容，改为 `index,follow`；若必须登录，保持 `noindex` 但修正 Title |

**修改理由：** 当前 Title 与页面内容完全不符，属于严重的 SEO 技术 bug。

---

### 4. 搜索情报 `/search-intelligence`

| 元素 | 当前状态 | 修改建议 |
|------|----------|----------|
| **H1** | `Search Intelligence` | `搜索情报与 SEO 数据分析` |
| **H2 补充** | 现有 7 个 | 增加 `百度收录查询与分析`、`Google 排名追踪`、`关键词排名变化监控` |

**修改理由：** H1 必须使用中文，且需包含"SEO"、"百度"等国内高频词。

---

### 5. 网站详情页 `/website/:domain`

| 元素 | 当前状态 | 修改建议 |
|------|----------|----------|
| **索引规则** | 部分 `noindex`（如 openai.com） | **统一规则**：只要分析完成且内容完整，全部设为 `index,follow` |
| **Title 模板** | `{domain} 网站分析 - 技术、基础设施与变化` | `{domain} 网站分析 - IP·DNS·SSL·技术栈检测与历史变化 \| SiteIntel` |
| **Description 模板** | `{domain} 的网站分析：它是什么...` | `查看 {domain} 的网站技术栈、服务器 IP、DNS 记录、SSL 证书及基础设施变化。SiteIntel 提供带证据的免费网站分析报告。` |
| **Meta Keywords** | 当前有 | 增加 `站长查询, 域名信息, 同 IP 网站查询` |
| **结构化数据** | 当前 1 个 JSON-LD | 增加 `WebPage` + `BreadcrumbList` 类型，帮助百度理解页面层级 |

**修改理由：** 这类页面是**长尾流量的核心**（如"openai.com 网站分析"、"wordpress.org 用的什么技术"）。每个被分析的知名网站都是自然搜索的入口，不应被无差别屏蔽。

---

### 6. API 文档 `/docs/api`

| 元素 | 当前状态 | 修改建议 |
|------|----------|----------|
| **Title** | `SiteIntel 开放 API \| SiteIntel` | `SiteIntel API 文档 - 网站数据接口与域名查询 API` |
| **Meta Description** | 默认 slogan | `SiteIntel 开放 API 文档：通过接口批量查询网站技术栈、IP、DNS、SSL 与基础设施数据。支持开发者集成与自动化分析。` |

---

### 7. 新增/优化页面建议（内容扩展）

为了捕获国内用户的**信息型搜索**（这类搜索量极大），建议新增或优化以下页面：

| 建议页面/板块 | 目标关键词 | 内容形式 |
|---------------|------------|----------|
| `/guides/what-is-website-analysis` | `什么是网站分析`、`网站分析工具怎么用` | 科普长文 |
| `/guides/website-tech-stack` | `网站技术栈查询`、`如何查看网站用什么搭建` | 教程 |
| `/guides/icp-lookup` | `ICP 备案查询`、`网站备案信息查询` | 功能页或说明页（即使暂不提供 ICP 数据，也可用内容截流） |
| `/compare` | `站长工具推荐`、`网站分析工具对比` | 对比表格（SiteIntel vs 站长之家 vs 爱站） |
| `/tools/whois` | `Whois 查询`、`域名注册信息查询` | 独立工具页（若已有数据，单独成页） |
| `/tools/ip-lookup` | `IP 地址查询`、`服务器 IP 查询` | 独立工具页 |

---

## 四、全站技术 SEO 修复清单

### 1. 统一默认 Meta 模板

为所有未设置自定义 Description 的页面设置**分层 fallback 规则**：

- 工具页：`{工具名} - {功能描述} | SiteIntel`
- 指南页：`{指南标题} - {核心问题解答} | SiteIntel`
- 当前直接 fallback 到首页 slogan 会导致大量重复描述（Duplicate Meta Descriptions）。

### 2. 修正 Canonical 标签

- `/bulk` 的 Canonical 错误指向首页，需修正为自身 URL。

### 3. 面包屑导航 + 结构化数据

- 在 `/website/:domain`、`/cases/:slug`、`/guides/:slug` 等页面增加面包屑导航，并补充 `BreadcrumbList` JSON-LD。

### 4. Sitemap 策略升级

当前 sitemap 只包含静态页面，应增加：

- **动态 sitemap 索引**：`/sitemap-websites.xml`（包含已分析的知名网站列表，如前 1000 个被分析的网站）。
- **静态 sitemap**：`/sitemap-static.xml`（首页、指南、案例、工具页）。

百度对 sitemap 的抓取频率依赖文件更新时间和 URL 质量，建议将高权重页面 `priority` 设为 `1.0`，工具页设为 `0.8`。

### 5. Robots 规则审计

- 移除 `/bulk` 的 `noindex`。
- 统一 `/website/:domain` 的索引规则：内容完整 → `index`；内容缺失/分析失败 → `noindex`。
- `/report` 可改为 `index,follow`（作为最新分析聚合页有 SEO 价值），或至少改为 `noindex,follow`（当前已是，但建议评估是否开放）。

### 6. 国内搜索专项适配

- **百度站长平台**（百度搜索资源平台）验证与提交。
- **必应站长工具**（Bing Webmaster）提交（必应中国市场份额不可忽视）。
- 页面增加**百度统计**或**51.la**代码（国内搜索引擎对自有生态产品有一定友好度）。
- 在页面底部增加**备案号**（如有）和**公安网备**号，提升信任度与转化率。

---

## 五、执行优先级建议

| 优先级 | 事项 | 预期效果 |
|--------|------|----------|
| **P0（本周）** | 修复 `/search` 标题错配；修正 `/bulk` 的 `noindex` 和 Canonical；统一 `/website/:domain` 索引规则 | 立即止损，释放批量分析与详情页流量 |
| **P1（本月）** | 重写首页、批量分析、API 页的 Title/Description；将搜索情报 H1 改为中文 | 提升核心页面在百度的关键词排名与 CTR |
| **P2（本季度）** | 新增指南页（什么是网站分析、技术栈查询等）；拆分 sitemap；增加面包屑结构化数据 | 捕获信息型长尾流量，提升站内权重流通 |
| **P3（长期）** | 增加 Whois、IP 查询等独立工具页；对比评测内容；国内社交媒体分享卡片优化 | 建立"站长工具"品类心智，与站长之家/爱站形成差异化 |

---

## 六、总结

SiteIntel 的技术架构（Next.js SSR）已经具备了良好的 SEO 基础，动态详情页能够输出独立 TDK 是很大的优势。**当前主要问题集中在：**

1. **策略层**：首页标题过于品牌向，未植入国内用户实际搜索的品类词（站长工具、域名查询、免费网站分析）。
2. **技术层**：部分页面存在 `noindex` 误伤、Title 错配、Canonical 错误、重复描述。
3. **内容层**：缺少针对国内搜索习惯的科普型、对比型内容，且 H1/H2 的语义标记在个别页面使用了英文。

按照上述规划修改后，预计在国内搜索引擎（尤其是百度和必应）上的**品牌词+品类词曝光量**会有显著提升，特别是 `/website/:domain` 详情页的长尾流量和工具型页面的搜索点击率。
