# SiteIntel 数据资产战略

> 目标：从 2.x 开始，尽可能沉淀可被 SiteIntel 主站及未来子产品复用的公开互联网数据。

## A. 网站实体

- Domain
- Root domain
- URL
- Website
- Subdomain
- 页面标题
- Meta description
- Canonical
- robots / sitemap
- favicon
- Open Graph
- 网站截图
- 页面结构特征
- 网站状态
- 首次发现时间
- 最近观察时间

## B. 网络与基础设施

- IP
- IPv4 / IPv6
- ASN
- ISP / Organization
- Hosting provider
- CDN
- DNS records
- Nameserver
- MX
- TLS / certificate observations
- HTTP headers
- Redirect chain
- Geolocation observation
- Infrastructure history

## C. 技术栈

- Technology
- Technology category
- Framework
- CMS
- JavaScript libraries
- Analytics
- Advertising / marketing technology
- CDN technology
- Hosting technology
- Security technology
- Payment technology
- Fonts
- UI / design technology
- Technology version（仅在来源可靠时）
- Technology first seen / last seen

## D. 企业公开信息

- Organization / Company
- Brand
- Product
- Public company description
- Industry
- Publicly stated founding information
- Public contact channels
- Public social profiles
- Public careers / recruitment signals
- Public product descriptions
- Public geographic footprint
- Public multilingual / internationalization signals

不把私人联系方式、非公开个人资料或不必要的个人敏感信息作为产品数据资产。

## E. 品牌资产

- Logo
- Logo variants
- Logo historical versions
- Favicon
- Brand colors
- Typography / font signals
- Slogan
- Brand name variants
- Brand / website relationship
- Brand / company relationship

## F. 产品资产

- Product name
- Product URL
- Product category
- Product description
- Product logo
- Product website
- Product → company
- Product → brand
- Product → technology
- Product history

## G. 媒体资源

- Website screenshot
- Page screenshot
- Mobile screenshot（后续）
- Open Graph image
- Public hero images
- Public media metadata
- Public icon resources

对媒体文件必须记录来源、抓取时间、使用权 / 版权状态和是否允许再分发；“能够抓取”不等于“可以自由商业再利用”。

## H. 实体关系

建议关系成为独立数据层，而不是把所有信息塞进 Website 表：

- Company ↔ Website
- Company ↔ Brand
- Company ↔ Product
- Brand ↔ Website
- Brand ↔ Product
- Website ↔ Technology
- Website ↔ IP
- IP ↔ ASN
- Website ↔ CDN
- Website ↔ Infrastructure
- Company ↔ Company（仅在有公开证据时）

## I. 历史数据

所有高价值实体都应逐渐支持时间维度：

- first_seen
- last_seen
- observed_at
- valid_from / valid_to（适合事实有效期）
- source
- confidence
- evidence

重点保存变化，而不是无限保存重复快照。

## J. 情报信号

可从基础数据派生：

- 新站上线
- 子域名增长
- 技术栈升级
- 基础设施迁移
- CDN 变化
- 国家 / 地区扩张信号
- 多语言增加
- 招聘扩张信号
- 产品数量变化
- 品牌变化
- 网站群关联
- 企业数字化升级信号

## K. 数据元信息

这是未来数据资产非常重要的一层：

- Source
- Source URL
- Collector
- Collected at
- Updated at
- Confidence
- Evidence
- License / rights status
- Public / private classification
- Retention policy
- Data version
- Observation version

## L. 数据复用原则

任何新增采集字段都优先回答：

1. SiteIntel 主站是否使用？
2. 企业情报是否使用？
3. 未来 Logo / Icon / Technology / Product 子站是否使用？
4. API 是否可以复用？
5. 是否能形成历史变化？
6. 是否能形成实体关系？

如果一个数据字段只能服务一个页面，而不能进入长期数据模型，应谨慎增加。
