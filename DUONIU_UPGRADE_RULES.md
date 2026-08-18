# DUONIU.cc 全站升级改版规则文件

## 项目名称

DuoNiu Network Intelligence Platform

## 项目定位

将 duoniu.cc 从传统网络工具集合站升级为：

> 全球网络 IP、ASN、BGP、路由、域名及网络环境分析平台。

中文定位：

> 多牛网络情报平台

---

# 一、当前项目技术环境

## 前端

- Nuxt 3
- Vue 3
- TypeScript
- SSR

## 服务端

- Nginx
- Node.js v20
- systemd 管理服务

服务：

```
duoniu-preview
```

端口：

```
3901
```

---

# 二、升级核心目标

禁止继续发展为：

```
IP查询工具集合
+
大量独立小工具
+
页面之间无关联
```

升级为：

```
Network Intelligence Engine
```

核心理念：

```
用户输入任何网络对象

↓

自动识别类型

↓

调用统一数据分析引擎

↓

返回结构化情报

↓

建立实体关系
```

支持对象：

```
IPv4

IPv6

Domain

ASN

CIDR

Prefix

Network
```

---

# 三、产品信息架构

```
DuoNiu
│
├── IP Intelligence
│
│   ├── IP 查询
│   ├── IP 风险检测
│   ├── IP 纯净度
│   ├── IP Leak
│   └── IP History
│
├── Network Intelligence
│
│   ├── ASN
│   ├── BGP
│   ├── CIDR
│   ├── Route
│   └── Whois
│
├── Network Diagnostics
│
│   ├── DNS
│   ├── Ping
│   ├── Trace
│   └── Environment
│
├── Developer
│
│   └── API
│
└── Information
    │
    ├── About
    └── FAQ
```

---

# 四、统一搜索系统

首页升级为：

```
Universal Network Search
```

用户输入：

```
1.1.1.1
```

自动识别：

```
IPv4
```

输入：

```
AS13335
```

识别：

```
ASN
```

输入：

```
1.1.1.0/24
```

识别：

```
CIDR
```

输入：

```
cloudflare.com
```

识别：

```
Domain
```

---

# 五、统一查询架构

禁止每个页面独立开发查询逻辑。

统一：

```
Search Input

↓

Query Parser

↓

Query Router

↓

Intelligence Service

↓

Data Provider Layer

↓

Database / Cache

↓

Frontend Display
```

---

# 六、数据层原则

前端禁止直接调用：

```
第三方 API

Provider API

外部数据源
```

必须：

```
Frontend

↓

DuoNiu API

↓

Data Service

↓

Provider Manager

↓

External Sources
```

---

# 七、数据容灾规则

所有重要数据：

必须支持：

```
Primary Provider

↓

Fallback Provider

↓

Redis Cache

↓

Database Snapshot
```

禁止直接显示：

```
Failed to fetch

Network Error

Provider Error
```

统一转换：

```
数据暂时不可用

正在显示最近缓存数据
```

---

# 八、首页改版规则

首页第一屏必须只展示：

```
IP

Country

ISP

ASN

IP Type

Risk Score
```

目标：

用户 3 秒知道：

```
这个 IP 是什么。
```

---

首页结果采用：

```
Overview

Network

Location

Risk

History

Routing

Raw Data
```

Tab 模式。

禁止：

一次性展示几十个字段。

---

# 九、代码架构规则

推荐：

```
app

├── pages

├── components

├── features

│   ├── ip

│   ├── asn

│   ├── bgp

│   ├── dns

│   ├── cidr

│   ├── history

│   └── diagnostics

├── composables

├── services

├── utils

└── types
```

禁止：

```
单页面超过1000行

组件直接请求API

页面重复开发逻辑
```