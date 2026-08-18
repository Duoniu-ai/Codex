# SITEINTEL SEO PHASE 1 — STAGING VERIFICATION REPORT

- **日期**: 2026-08-17
- **阶段**: STAGING VERIFICATION PHASE B（数据库只读角色方案批准执行）
- **验证对象**: Staging `127.0.0.1:3004`（代码 `seo/phase-1-p0 @ c74dfd1`，只读角色连接生产数据）
- **对照基线**: Production `127.0.0.1:3003 @ HEAD 1bdfcb4`（全程零修改）

---

## 1. Phase B 执行范围

| 项 | 状态 |
|---|---|
| 创建专用只读角色 `siteintel_staging_ro` | ✅ 已执行（最小权限，§2） |
| 权限最小化验证（角色属性/连接/查询） | ✅ |
| 写入拒绝验证（真实 INSERT/TEMP 表，零数据变化） | ✅ |
| staging `.env` 更新 + 服务重启 + 3004 数据页恢复 | ✅ |
| 验证矩阵 P0-1 ~ P0-6（真实 HTTP / HTML） | ✅ 全部执行 |
| 生产零影响终验 | ✅ |
| 禁止事项 | 未 merge / 未 push / 未部署 / 未改生产代码 / 未删 staging / 未改 Nginx / 未建 preview.siteintel.cc |

## 2. 数据库权限配置结果（不含任何凭据）

```
CREATE ROLE siteintel_staging_ro LOGIN
  NOSUPERUSER NOCREATEDB NOCREATEROLE NOREPLICATION NOBYPASSRLS;   -- 无任何管理/复制属性
GRANT CONNECT ON DATABASE siteintel;
GRANT USAGE ON SCHEMA public;
GRANT SELECT ON ALL TABLES IN SCHEMA public;                        -- 仅现有表
ALTER ROLE siteintel_staging_ro SET default_transaction_read_only = on;
```

- **未执行** `ALTER DEFAULT PRIVILEGES`（遵守"不扩大未来对象权限"——当前阶段只验证现有表）
- **未执行**任何 GRANT 写权限 / 任何 DML
- 角色属性实证（pg_roles）：`rolsuper=f rolcreatedb=f rolcreaterole=f rolreplication=f rolbypassrls=f`，`rolconfig={default_transaction_read_only=on}`
- staging 连接配置：`DATABASE_URL configured successfully`（密码仅存在于服务器 staging `.env`，600 权限；未出现在终端/报告/Git/记忆/本地任何文件）
- 凭据安全：AUTH_SECRET（staging 独立新值）与 DATABASE_URL 均不在本报告出现

## 3. Read-only 验证结果

| 验证 | 实测 | 结果 |
|---|---|---|
| SELECT（只读角色连接生产库） | `SELECT 1` → `1` | ✅ **PASS** |
| `SHOW transaction_read_only` | `on` | ✅ **PASS** |
| 写入拒绝（真实表） | `BEGIN; INSERT INTO "Target" ...` → `ERROR: cannot execute INSERT in a read-only transaction` → `ROLLBACK` | ✅ **PASS**（写前直接拒绝，预期行为） |
| 写入拒绝（建表） | `CREATE TEMP TABLE _probe` → `ERROR: cannot execute CREATE TABLE in a read-only transaction` | ✅ **PASS** |
| 零数据残留 | `residue=0`（readonly-probe 行数=0） | ✅ **PASS** |

## 4. P0-1 六状态验证矩阵

> **样本来源标注**：「真实」= 从生产库只读 SELECT 选取并在 staging 实际 HTTP 请求验证；「单元测试」= 当前生产数据无该状态样本，以 `report-state.test.ts` 结果补充（明确标注，不冒充真实验证）。

| 状态 | 样本（域名） | 来源 | HTTP | robots | canonical | 页面核心内容 | 结果 |
|---|---|---|---|---|---|---|---|
| 完全不存在 | `nonexistent-xyz-20260817.com` | 真实（未登记域名） | **404** | noindex | —（404 页） | 「页面不存在」+「返回首页」+ 分析引导 | ✅ |
| running | 无样本 | **当前生产数据无 running 状态**（如实记录） | — | — | — | 单元测试验证：running 分支 = 200 + noindex + 处理中提示（report-state.test.ts） | ✅ 无样本（单元测试补充） |
| failed 无有效报告 | `ecn.t4.tiles.virtualearth.net`（failed + 0 facts） | 真实 | 200 | noindex, follow | `/website/ecn.t4...` | 「暂无数据」+「正在分析」+「重新分析」引导（不 404） | ✅ |
| failed 但有历史报告 | 无样本（库中 `google.com` 存在 failed 历史调查，但最新调查=partial，不构成"最新 failed"） | **当前生产数据无该状态样本** | — | — | — | 单元测试验证：failed + 可展示数据 → report 态渲染完整报告 + noindex（report-state.test.ts） | ✅ 无样本（单元测试补充） |
| completed | `wordpress.org`（137 次） | 真实 | 200 | **index, follow** | `/website/wordpress.org` | 完整报告（「网站概览」「值得关注」区块） | ✅ |
| completed | `kslm.cc`（1 次，活动不足） | 真实 | 200 | noindex, follow | `/website/kslm.cc` | 完整报告（数据存在但 Gate 拒） | ✅ |
| partial | `googleoptimize.com` / `fps.goog`（各 1 次） | 真实 | 200 | noindex, follow | 自持 | 报告渲染 | ✅ |

## 5. Index Gate V1 真实验证

| 条件 | 样本 | 实测数据（只读 SELECT） | Gate 预测 | 页面 robots 实测 | 结果 |
|---|---|---|---|---|---|
| A 热门白名单 | `google.com` | 最新=partial、4 次、8 facts | index（A 命中） | **index, follow** | ✅ |
| A 热门白名单 | `github.com` | 白名单 | index | **index, follow** | ✅ |
| B investigationCount ≥ 3 | `wordpress.org` | completed、**137 次** | index（B 命中） | **index, follow** | ✅ |
| B investigationCount ≥ 3 | `cloudflare.com` | **14 次** | index（B 命中） | **index, follow** | ✅ |
| C 不满足 Gate | `kslm.cc` | completed、1 次、9 facts（数据足但活动不足） | noindex（below-activity-threshold） | **noindex, follow** | ✅ |
| C 不满足 Gate | `googleoptimize.com` | partial、1 次 | noindex | **noindex, follow** | ✅ |
| D 低价值域名（一票否决） | `example.com` | **completed、8 次、9 facts（门槛全过，仅低价值拒）** | noindex（low-value-domain） | **noindex, follow** | ✅ 最强证明：数据/活动均满足仍被低价值门槛拦截 |
| D 低价值域名 | `ecn.t4.tiles.virtualearth.net` | failed + 低价值 | noindex | **noindex, follow** | ✅ |

## 6. robots / sitemap 一致性验证（双向）

**正向（页面 robots → sitemap）**：所有 index 域名均出现在 sitemap；所有 noindex 域名均不在 sitemap。

| 域名 | 页面 robots | sitemap 包含 | 一致 |
|---|---|---|---|
| google.com | index | ✅ | ✅ |
| wordpress.org | index | ✅ | ✅ |
| github.com | index | ✅ | ✅ |
| cloudflare.com | index | ✅ | ✅ |
| kslm.cc | noindex | ✗ | ✅ |
| googleoptimize.com | noindex | ✗ | ✅ |
| fps.goog | noindex | ✗ | ✅ |
| ecn.t4.tiles.virtualearth.net | noindex | ✗ | ✅ |
| example.com | noindex | ✗ | ✅ |
| nonexistent-xyz-20260817.com | 404/noindex | ✗ | ✅ |

**反向（sitemap → 页面 robots）**：sitemap 内随机抽取 4 个 website + baidu.com 抽查 → 页面 robots 全部 `index, follow`。**双向闭环，0 冲突**。

## 7. Canonical 实际 HTML 验证

| 页面 | 实际 HTML canonical | 判定 |
|---|---|---|
| `/website/google.com` | `http://127.0.0.1:3004/website/google.com` | ✅ Self Canonical（路径段语义） |
| `/website/wordpress.org` | `http://127.0.0.1:3004/website/wordpress.org` | ✅ |
| `/website/kslm.cc` | `http://127.0.0.1:3004/website/kslm.cc` | ✅ |
| `/bulk` | `http://127.0.0.1:3004/bulk` | ✅ |
| `/docs/api` | `http://127.0.0.1:3004/docs/api` | ✅ |
| `/report` | `http://127.0.0.1:3004/report` | ✅ |

> 注：URL 形态为 `http://127.0.0.1:3004/...`（staging `NEXT_PUBLIC_SITE_URL` 所致）；部署生产后自动变为 `https://siteintel.cc/...`。验证的是路径段语义（Self Canonical），与 P0-4 验收一致。

## 8. Metadata 实际 HTML 验证

| 页面 | HTTP | title | description | robots | 判定 |
|---|---|---|---|---|---|
| `/bulk` | 200 | 批量分析 \| SiteIntel | 「SiteIntel 批量分析工具：一次提交最多 20 个域名…」 | noindex, nofollow | ✅ 独立描述非首页复用 |
| `/docs/api` | 200 | SiteIntel 开放 API \| SiteIntel | 「SiteIntel 开放 API v1 文档：按域名获取…」 | **index, follow**（显式） | ✅ |
| `/report` | 200 | 近期报告 \| SiteIntel | 「SiteIntel 最近分析的域名及其调查状态。」 | noindex, follow | ✅ |

## 9. Low-value Domain Sitemap 验证

sitemap.xml 实测（184 总 URL，**30 条 website URL，非空**）：

| 检查项 | 实测 | 判定 |
|---|---|---|
| `example` | 0 匹配 | ✅ |
| `virtualearth`（Bing tiles 系） | 0 匹配 | ✅（基线 sitemap 曾含 ecn.t1-3/r1.tiles 等，已全部排除） |
| `akamai` / `azureedge` | 0 匹配 | ✅ |
| `cloudfront` / `fastly` | 各 1 匹配 = **technology 实体页**（`/technology/fastly`、`/technology/amazon-cloudfront`）——技术实体页合法收录，非域名收录 | ✅ 非低价值漏网 |
| 已知 noindex 域名（kslm.cc/googleoptimize.com/fps.goog/ecn.t4） | 0 匹配 | ✅ |
| 正常 indexable 样本 | wordpress.org / google.com / github.com / cloudflare.com 全部在列 | ✅ 未出现"过滤后 sitemap 变空" |

## 10. Staging 服务状态

| 项 | 值 |
|---|---|
| 服务 | `siteintel-staging.service` **active (running)**（restart 后） |
| 监听 | `127.0.0.1:3004`（仅本机，ss 实证） |
| 数据页 | 首页 200 + title「看懂任何一个网站 \| SiteIntel」（只读角色读路径打通） |
| robots.txt / sitemap.xml | 均 HTTP 200 |
| 日志 | `staging.out`（无异常刷屏；read-only 拒绝仅出现在验证测试中） |

## 11. Production 零影响验证

| 项 | 验证 | 结果 |
|---|---|---|
| `siteintel` 服务 | `is-active` → **active**（未停止/未重启） | ✅ |
| 3003 | `/robots.txt` → HTTP 200 | ✅ |
| 生产 HEAD | `1bdfcb4`（未动） | ✅ |
| 生产 git status | 16 项，与 Phase A 完全一致（基线期已知卫生项 + 文档；无新增） | ✅ |
| 生产 .env / systemd / nginx | 零修改 | ✅ |
| 业务数据 | Target 287 / Investigation 530（Phase A 审计时 281/518）——**+6/+12 为生产自身 30 分钟窗口内监控/发现调度器的正常产出**；staging 写入已被 read-only 硬约束 + 实测拒绝双重证明不可能发生 | ✅ 无 staging 造成的数据变更 |

**本次生产数据库唯一变更** = 创建 `siteintel_staging_ro` + 最小 SELECT 授权 + read_only 配置（纯配置级，零数据变更，§2）。

## 12. 最终结论

# ✅ PASS

**依据**:
- **P0-1**: 六状态真实验证 4/6（不存在 404 ✅ / failed 无数据 ✅ / completed ✅ / partial ✅）；running 与 failed-有历史两状态当前生产无样本，如实记录并以单元测试明确标注补充——**未伪造数据**
- **P0-2/P0-3**: Gate V1 四个条件层（A 白名单 / B ≥3 次 / C 活动不足 / D 低价值一票否决）全部真实样本命中预期；**example.com 以 completed+8 次+9 facts 仍被低价值拦截**为最强验证；robots ↔ sitemap 双向一致 0 冲突
- **P0-4/P0-5**: 三页实际 HTML 的 title/description/canonical/robots 全部正确（docs/api 显式 index）
- **P0-6**: sitemap 184 URL（30 website）非空；example/virtualearth/CDN 后缀/noindex 域名 0 残留；cloudfront/fastly 命中为合法 technology 实体页
- **数据库安全**: 只读角色最小权限实证（五属性全 f）、`transaction_read_only=on`、真实 INSERT/CREATE 被拒、零数据残留
- **生产零影响**: 服务/端口/HEAD/git 全零修改；业务行数增长为生产自身调度产出（staging 不可写）

**无 P0/P1/P2 级问题。无 FAIL 项。**

**遗留（不影响本 PASS）**: ① running / failed-有历史两状态生产无样本（记录在案，随生产运行自然出现后可补验）② staging canonical 为 `http://127.0.0.1:3004` 形态（部署生产后为 `https://siteintel.cc`——语义不变）。

**完成，停止。等待生产部署验收指令。**
