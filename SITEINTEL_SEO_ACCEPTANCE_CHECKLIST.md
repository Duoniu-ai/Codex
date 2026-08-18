# SITEINTEL SEO ACCEPTANCE CHECKLIST

# SiteIntel SEO 修改验收清单

版本：V1.0

---

# 一、验收原则

禁止仅根据：

- 代码已修改
- 编译成功
- 页面能打开

判断完成。

必须同时进行：

- 代码检查
- 实际页面访问
- HTTP 状态验证
- HTML 源码检查
- Metadata 检查
- 搜索引擎规则检查

---

# 二、P0 验收

## Metadata

- [ ] 首页 Title 正确
- [ ] 首页 Description 正确
- [ ] `/search` Title 正确
- [ ] `/bulk` Title 正确
- [ ] `/report` Title 正确
- [ ] `/docs/api` Title 正确
- [ ] `/search-intelligence` H1 正确
- [ ] 工具页拥有独立 Metadata
- [ ] 没有错误复用首页 Description

---

# 三、Canonical

随机抽查至少 10 个页面：

- [ ] Canonical 指向自身
- [ ] `/bulk` 不指向首页
- [ ] 动态 Website 页面 Canonical 正确
- [ ] 不存在 Canonical 循环

---

# 四、HTTP 状态

测试正常页面：

`/`

- [ ] HTTP 200

测试正常 Website：

`/website/openai.com`

- [ ] HTTP 200

测试不存在 Website：

`/website/nonexistent-test-domain-123456789.com`

- [ ] HTTP 404

禁止：

- HTTP 200 + 空白页面
- HTTP 200 + 默认页面

---

# 五、INDEX Gate

至少测试四类：

## 高质量热门网站

- [ ] INDEX
- [ ] Sitemap 包含

## 普通低热度网站

- [ ] 根据规则正确 INDEX / NOINDEX

## 停放 / 测试域名

- [ ] NOINDEX
- [ ] 不进入 Sitemap

## 分析失败

- [ ] NOINDEX 或 404
- [ ] 不进入 Sitemap

---

# 六、Sitemap

检查：

- [ ] Sitemap 可访问
- [ ] XML 正确
- [ ] 没有 404 URL
- [ ] 没有 NOINDEX URL
- [ ] 没有测试 URL
- [ ] Static URL 正确
- [ ] Indexable Website URL 正确

---

# 七、Robots

检查：

- [ ] robots.txt 可访问
- [ ] Sitemap 地址正确
- [ ] 没有错误禁止核心页面
- [ ] NOINDEX 页面 Metadata 正确
- [ ] INDEX 页面允许抓取

---

# 八、JSON-LD

检查：

- [ ] 首页 Organization
- [ ] 首页 WebSite
- [ ] 工具页结构化数据正确
- [ ] BreadcrumbList 正确
- [ ] URL 与页面一致

禁止：

- [ ] 虚构评分
- [ ] 虚构评论
- [ ] 虚构 FAQ

---

# 九、页面内容

检查：

- [ ] 每个页面 H1 明确
- [ ] H1 与 Title 语义一致
- [ ] Description 不重复
- [ ] 页面存在真实主体内容
- [ ] 页面不是纯模板数据
- [ ] 没有关键词堆砌

---

# 十、内部链接

Website 页面检查：

- [ ] 可以进入真实 IP 数据
- [ ] 可以查看 Technology
- [ ] 可以继续探索 DNS / Infrastructure
- [ ] 大量关联结果已分页
- [ ] 没有链接农场

---

# 十一、性能

检查：

- [ ] SSR HTML 包含核心内容
- [ ] Metadata 不依赖浏览器 JavaScript
- [ ] 主要内容不完全依赖客户端渲染
- [ ] 不存在明显 SEO 内容闪烁

---

# 十二、最终验收

只有以下全部满足：

P0 Metadata  
+ Canonical  
+ HTTP  
+ 404  
+ INDEX Gate  
+ Robots  
+ Sitemap

全部通过后：

`SEO Phase 1 = PASS`

如果任何 P0 项失败：

`SEO Phase 1 = FAIL`

禁止使用：

- 大部分完成
- 基本没问题
- 理论上已经实现

作为验收结论。

必须根据线上真实访问结果判断。

---

# 十三、最终验收报告格式

Claude Code 完成后必须输出：

`SITEINTEL_SEO_IMPLEMENTATION_REPORT.md`

报告必须包含：

## 1. 修改文件

记录：

- 文件路径
- 修改内容
- 修改原因

## 2. 实际测试 URL

至少包括：

- 首页
- 搜索页
- Bulk
- API
- 10 个 Website 页面
- 3 个不存在 URL

## 3. HTTP 状态

逐个记录：

- URL
- Status
- Expected
- Result

## 4. Metadata

抽查：

- Title
- Description
- Canonical
- Robots

## 5. Sitemap

记录：

- URL 数量
- INDEX 页面数量
- NOINDEX 页面数量
- 异常数量

## 6. 最终结论

只能使用：

`PASS`

或：

`FAIL`

如果 FAIL，必须列出：

- 未完成项目
- 原因
- 影响
- 下一步修复方案

禁止模糊结论。
