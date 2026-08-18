# IPIP0 阶段五 P1 产品完整性补全

阶段五 P0 已全部 PASS。

现在只进入 P1，不做 P2。

## P1-1：完成 About / Privacy / Terms

先读取当前真实页面和项目已有文案。

目标：

- 去掉占位页/简化页
- 保留当前站点定位
- 中英文完整对应
- 当前实际数据处理方式与页面内容一致
- 不写不存在的功能
- 不写不存在的数据采集行为

Privacy / Terms 必须与当前真实产品保持一致，尤其不要声明当前不存在的：

- 查询历史功能
- 指纹扫描
- 端口扫描
- 其他尚未上线能力

About 页面明确说明 IPIP0 当前真实能力和产品定位。

完成后验证：

- `/about`
- `/privacy`
- `/terms`
- `/en/about`
- `/en/privacy`
- `/en/terms`

---

## P1-2：完善域名查询结果

当前产品支持域名查询，但结果仍主要采用 IP 视角。

先审计当前域名查询后端到底返回什么真实数据。

在不破坏现有 IP 查询的前提下，增加域名结果结构：

```text
Domain
├── A
├── AAAA
├── CNAME
└── Resolved IPs
      └── IP Intelligence
```

要求：

- 只展示真实 DNS 数据
- 不新增 Mock
- 如果某记录不存在，显示真实空态
- 每个解析 IP 可以继续进入现有 IP 详情/查询结果
- 保持现有 UI 设计系统
- 中英文同步

不要在本阶段扩展 DNS 历史、WHOIS 历史等未实现能力。

---

## P1-3：Native / Anycast

先进行真实能力审计。

检查：

- 当前后端是否已有相关字段
- 当前数据源是否支持
- 当前是否已有判断逻辑
- 是否能可靠得到 Native / Anycast 结果

如果当前没有可靠数据源：

不要硬做页面结果。

先建立：

`IPIP0-Native-Anycast能力评估.md`

明确：

- 数据来源
- 判断条件
- 数据结构
- API 需要什么
- 成本 / 限制
- 是否值得实现

只有确认数据源和判断逻辑可靠后，才开发。

---

## 约束

本阶段：

- 不修改 Collector
- 不修改 Risk Snapshot
- 不修改 query_log
- 不修改 ASN Entity 模型
- 不开发 Organization 页面
- 不开发 Network Prefix 页面
- 不开发新的 SEO 程序化页面
- 不修改 `D:\phpstudy_pro\WWW\ipquery`
- 不使用 Mock Data

优先复用现有组件和已有 API。

---

## 验收

完成后验证：

1. About / Privacy / Terms 中英文
2. 域名查询
3. 域名 A / AAAA / CNAME
4. 域名解析 IP 跳转
5. IPv4 / IPv6
6. 空数据
7. 无效域名
8. Native / Anycast 能力结论
9. 375 / 390 / 768 / Desktop
10. 全站 API 回归

生成：

`IPIP0-阶段五-P1产品完整性补全与验收报告.md`

报告必须明确：

- 哪些已经 COMPLETE
- 哪些仍是 PARTIAL
- 哪些由于缺少真实数据能力暂缓
- 是否还有 P0 遗留

完成 P1 后停止，不自动进入 P2。