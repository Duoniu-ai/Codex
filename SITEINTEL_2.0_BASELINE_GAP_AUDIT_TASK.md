# SiteIntel 2.0 Baseline & Gap Audit 执行任务

> 版本：v1.0
> 日期：2026-08-19
> 类型：**只读审计任务**
> 项目：SiteIntel
> 当前基线：Phase 1 / Step 8（Query & Read Performance Foundation）已完成并通过验证

## 0. 执行纪律

本任务只分析，不修改代码、数据库、迁移、环境变量、生产配置或部署。

不得重复执行已经验收的 Step 8。

不得把 Websites 纳入本次执行范围。GitHub 中 Websites 资料只能作为背景，不得作为 SiteIntel 当前能力。

发现架构冲突、数据缺口或不确定项时，记录后停止，不自行修复。

## 1. 审计资料来源

优先读取：
1. 当前 SiteIntel 仓库/工作目录
2. 已完成 Step 8 的验收报告及状态文件
3. SiteIntel 2.0 Product Upgrade Blueprint
4. SiteIntel 2.0 Master Architecture
5. SiteIntel Data Architecture
6. SiteIntel Product Architecture
7. SiteIntel Gap Analysis / ADR
8. SiteIntel 2.0 后期产品与技术总规划 v1.1
9. SiteIntel 2.0 UI/UX 产品重构规划

## 2. 必须核实的当前基线

输出：
- 当前 Git HEAD
- 工作区状态
- 部署/生产基线（如果现有资料允许确认）
- 数据库 schema 版本/冻结状态
- Step 8 commit / report / verification evidence
- 当前主要前后端架构

不得根据旧聊天内容猜测；能验证则验证，不能验证则标记 UNKNOWN。

## 3. 产品能力 Gap Matrix

逐项分类：

| 能力 | 当前状态 | 数据层 | 后端 | 前端 | API | 证据 | 优先级 |
|---|---|---|---|---|---|---|---|
| Website Intelligence | | | | | | | |
| Technology Intelligence | | | | | | | |
| Technology Taxonomy | | | | | | | |
| Technology History | | | | | | | |
| Technology Churn | | | | | | | |
| Historical Backfill | | | | | | | |
| Historical Reconstruction | | | | | | | |
| Observed/Reconstructed/Inferred | | | | | | | |
| Change Engine | | | | | | | |
| Evidence Chain | | | | | | | |
| Confidence | | | | | | | |
| IP Intelligence | | | | | | | |
| ASN Intelligence | | | | | | | |
| DNS History | | | | | | | |
| Certificate History | | | | | | | |
| Subdomain Intelligence | | | | | | | |
| Relationship Intelligence | | | | | | | |
| Technology Market Intelligence | | | | | | | |
| Trends | | | | | | | |
| Risk | | | | | | | |
| AI Intelligence | | | | | | | |
| Monitoring | | | | | | | |
| API/Dataset | | | | | | | |

状态只能使用：
- DONE：真实存在且有验证证据
- PARTIAL：部分存在
- DATA-READY：底层已有但产品未完成
- UI-GAP：数据已有但前端未表达
- MISSING：没有
- LATER：明确属于后期
- UNKNOWN：无法验证

## 4. BuiltWith / Wappalyzer 映射

分别列出：
- 已吸收
- 已有但不完整
- 仅规划
- 不适合 SiteIntel
- 建议不做

禁止把竞品页面字段直接等同于 SiteIntel 数据字段。

## 5. UI/UX Gap Audit

审计至少：
- 首页
- 网站分析页
- 技术展示
- DNS/IP/ASN/Certificate 展示
- Event/History
- Relationship/Explore
- Evidence
- Loading/Empty/Error
- Mobile
- Navigation
- SEO
- Performance

每项标记：
- 当前页面
- 当前问题
- 用户影响
- 数据是否已有
- 是否需要后端支持
- 是否需要新组件
- 优先级

重点判断：
> 当前 UI 是否已经把已有 SiteIntel 数据能力完整、清晰、用户化地表达出来？

## 6. 历史冷启动专项审计

必须明确：
- 当前有哪些真实历史
- 哪些只有当前快照
- 是否已有 first_seen/last_seen
- 是否已有 snapshot/observation
- 是否已有 event/diff
- 能否接入合法公开历史证据
- 哪些历史只能重建
- 如何标记 Observed / Reconstructed / Inferred

不得在没有证据的情况下制造历史记录。

## 7. Step 8 防回归检查

必须确认本次审计结论不会要求重复实现 Step 8。

后续任何新能力若涉及 Query/Read：
- 必须说明是在既有性能基础上新增查询
- 必须避免全表扫描和无界查询
- 必须有定向性能验证

## 8. 最终输出

生成：

### A. SITEINTEL_2.0_BASELINE_REPORT.md
当前真实状态。

### B. SITEINTEL_2.0_GAP_MATRIX.md
产品/数据/后端/前端/API 的差距矩阵。

### C. SITEINTEL_2.0_UI_UX_GAP_REPORT.md
UI/UX 专项差距报告。

### D. SITEINTEL_2.0_NEXT_PHASE_RECOMMENDATION.md
只推荐一个下一阶段小闭环，并说明：
- 为什么现在做
- 前置条件
- 不做什么
- 代码范围
- 数据库范围
- 验证范围
- 停止条件

## 9. 停止条件

出现以下任一情况必须停止并报告：
- Step 8 状态无法确认
- 生产基线无法确认且结论依赖生产状态
- schema 状态与现有文档冲突
- 发现需要迁移但没有授权
- 发现大范围架构重构需求
- 需要修改现有冻结契约
- 发现 Websites 被误混入 SiteIntel 范围

## 10. 验收标准

本任务只有在：
- 完成只读审计
- 有证据支持的 DONE/PARTIAL/MISSING 分类
- UI/UX 有独立 Gap Report
- BuiltWith/Wappalyzer 映射完成
- 历史冷启动问题有明确结论
- Step 8 未被重复规划
- 下一阶段只有一个明确的小闭环

后才算完成。
