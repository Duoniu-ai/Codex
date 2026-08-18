DUONIU_EXECUTION_RULES.md
# DUONIU.cc AI Engineering Execution Rules


Version: 1.0


Project:


DuoNiu Network Intelligence Platform


Applicable AI Agents:


- Claude Code
- DeepSeek V4 Pro
- Other Coding Agents




---


# 0. 核心目标


本文件用于约束 AI Coding Agent 对 DUONIU.cc 项目的修改行为。




项目目标：


从：


Network Tools Website




升级为：


DuoNiu Network Intelligence Platform




升级方向：


- Universal Search
- Network Entity Intelligence
- Provider Aggregation
- Network Relationship
- Historical Data Analysis
- SEO Intelligence Pages




但是：


稳定性优先于创新。


线上服务优先于代码优化。




---


# 1. 最高执行原则




## Rule 1.1


禁止一次性重构。




任何大型改造必须：


拆分 Phase。




正确：



Phase 0
安全准备

↓

Phase 1
基础架构

↓

Phase 2
搜索系统

↓

Phase 3
页面升级

↓

Phase 4
SEO

↓

Phase 5
性能





错误：



重新设计整个项目
一次提交全部代码





---


# 2. 修改前必须分析




任何代码修改前必须输出：



TASK ANALYSIS

目标：

影响范围：

涉及文件：

修改方案：

风险：

回滚方案：

测试方案：





未经分析：


禁止修改。




---


# 3. 禁止破坏规则




AI 禁止：


## 删除功能


禁止删除：



pages
components
server/api
database table
existing endpoint





除非：


用户明确批准。




---


## 删除代码


发现：


- 重复代码
- 老组件
- 无引用文件




处理：



标记
记录
等待确认





禁止：


直接 rm。




---


# 4. 线上环境保护规则




DUONIU.cc 是生产项目。




禁止自动执行：





systemctl restart
nginx reload
kill process
docker restart
database migration
DNS change
Cloudflare change





必须：


等待人工确认。




---


# 5. Git规则




所有修改必须进入 Git。




修改前：



git status





创建：



git commit





格式：



phase-x:
description





例如：



phase-1:
add provider adapter layer





每个阶段：


必须：



git tag phase-x-complete





---


# 6. Backup规则




涉及：


- nginx
- systemd
- database
- config




必须先备份。




格式：





filename.backup.YYYYMMDD





例如：



duoniu.cc.conf.backup.20260815





---


# 7. 架构规则




目标架构：



Frontend

↓

API Layer

↓

Service Layer

↓

Provider Adapter

↓

Data Source





---


# 8. Frontend规则




## 禁止


页面直接访问：



http://external-api





禁止：



$fetch(provider-url)





必须：



useApi()

↓

server/api





---


## 页面职责




Page：


负责：


- layout
- SEO
- 用户交互




禁止：


包含：


- 数据处理
- Provider逻辑
- 复杂算法




---


# 9. Backend规则




Server API：


职责：


- 参数验证
- 权限控制
- 错误处理




业务逻辑：


必须进入：



server/services





数据源：


必须进入：



server/providers





---


# 10. Provider规则




任何数据来源：


必须 Adapter 化。




禁止：


业务代码直接：



fetch(ipquery)
fetch(ipf)
fetch(rdap)





必须：



Provider Adapter

统一输出 Entity





---


# 11. API兼容规则




旧API：


不能删除。




例如：


旧：



/api/ip





新增：



/api/intel/ip





迁移：


内部转发。




等待稳定后：


再评估废弃。




---


# 12. 数据库规则




禁止：



DROP TABLE

DELETE DATA

TRUNCATE





数据库升级：


只能：



CREATE TABLE IF NOT EXISTS

ADD COLUMN





任何迁移：


必须：


backup。




---


# 13. 数据模型规则




未来统一：


Entity Model。




包括：



IP Entity

ASN Entity

Network Entity

Domain Entity





禁止：


每个页面创建自己的数据结构。




---


# 14. SEO规则




任何页面升级：


必须检查：



title

description

canonical

schema

sitemap

robots





禁止：


删除旧URL。




URL变化：


必须：


301。




---


# 15. 性能优化规则




优化顺序：


第一：


缓存




第二：


懒加载




第三：


代码拆分




第四：


数据库优化




禁止：


通过删除功能提升性能。




---


# 16. 安全规则




所有输入：


必须验证。




重点：



URL
IP
Domain
CIDR
ASN





禁止：


SSR攻击风险：



127.0.0.1

localhost

169.254.x.x

private network





---


# 17. 测试规则




每次修改必须：




## Build



npm run build





## 页面测试




至少：



/
env
purity
asn
bgp
dns
whois





## API测试




检查：



status

response

error format





---


# 18. AI自主行为限制




AI 可以：


✅ 分析代码


✅ 提出方案


✅ 创建文件


✅ 修改明确范围代码




AI 不可以：


❌ 自己扩大需求


❌ 自己重构其它模块


❌ 自己修改生产配置


❌ 自己删除文件


❌ 自己迁移数据库




---


# 19. 每阶段交付格式




完成后必须输出：





PHASE REPORT

Completed

文件:

修改:

Testing

Build:

Pages:

API:

Risk

发现问题:

Next Step

下一阶段:





---


# 20. 最终验收目标




完成后：


DUONIU.cc 应具备：





Universal Search

↓

Entity Intelligence

↓

Network Relationship

↓

Historical Observation

↓

SEO Landing Pages

↓

Intelligence Platform





最终原则：



不破坏

不重写

不冒进

渐进升级

每步可回滚





END

这版比上一版更适合直接放入项目根目录。

最终目录建议：

duoniu-new-vue/


├── CURRENT_ARCHITECTURE.md
├── DUONIU_UPGRADE_RULES.md
├── DUONIU_IMPLEMENTATION_PLAN.md
└── DUONIU_EXECUTION_RULES.md   ← 新版

然后给 Claude Code：

开始工作前必须阅读：


CURRENT_ARCHITECTURE.md
DUONIU_UPGRADE_RULES.md
DUONIU_IMPLEMENTATION_PLAN.md
DUONIU_EXECUTION_RULES.md




不要修改代码。


首先执行 Phase 0 分析。


输出执行计划后等待确认。