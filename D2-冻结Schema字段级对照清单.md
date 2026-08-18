# D2 字段级机械对照清单 — Prisma 实现 vs Schema V1.0 冻结版

> 对照日期:2026-08-17
> 对照方法:冻结版《网址发现平台-数据库Schema与数据同步契约-V1.0-冻结版.md》§2.1-§2.24 逐表逐字段(字段/类型/NULL/默认/唯一/索引/关系/枚举 CHECK)→ `prisma/schema.prisma` 逐项核对,**非摘要对照**
> 对照结论:**24 表 / 221 字段全部一致;无新增字段、无删减字段、无类型/语义/默认/唯一/索引偏差**(差异明细见 §4-§8,均为"Prisma 无法表达的 DB 约束"登记项)

---

## 1. 表清单(24/24)

| # | 冻结表名 | Prisma Model | `@@map` | 字段数 | 对照 |
|---|---|---|---|---|---|
| 1 | websites | Websites | websites | 19 | ✅ |
| 2 | categories | Categories | categories | 8 | ✅ |
| 3 | tags | Tags | tags | 7 | ✅ |
| 4 | website_tags | WebsiteTags | website_tags | 6 | ✅ |
| 5 | tasks | Tasks | tasks | 11 | ✅ |
| 6 | task_aliases | TaskAliases | task_aliases | 8 | ✅ |
| 7 | task_related_tags | TaskRelatedTags | task_related_tags | 5 | ✅ |
| 8 | website_tasks | WebsiteTasks | website_tasks | 10 | ✅ |
| 9 | website_relations | WebsiteRelations | website_relations | 10 | ✅ |
| 10 | website_snapshots | WebsiteSnapshots | website_snapshots | 6 | ✅ |
| 11 | website_status | WebsiteStatus | website_status | 19 | ✅ |
| 12 | website_metrics | WebsiteMetrics | website_metrics | 7 | ✅ |
| 13 | website_changes | WebsiteChanges | website_changes | 7 | ✅ |
| 14 | recommendations | Recommendations | recommendations | 10 | ✅ |
| 15 | recommendation_evidence | RecommendationEvidence | recommendation_evidence | 8 | ✅ |
| 16 | curated_picks | CuratedPicks | curated_picks | 9 | ✅ |
| 17 | submissions | Submissions | submissions | 11 | ✅ |
| 18 | reports | Reports | reports | 9 | ✅ |
| 19 | user_events | UserEvents | user_events | 7 | ✅ |
| 20 | search_logs | SearchLogs | search_logs | 9 | ✅ |
| 21 | sync_queue | SyncQueue | sync_queue | 17 | ✅ |
| 22 | sync_logs | SyncLogs | sync_logs | 10 | ✅ |
| 23 | settings | Settings | settings | 3 | ✅ |
| 24 | admin_users | AdminUsers | admin_users | 5 | ✅ |
| **合计** | 24 | 24 | — | **221** | 24 ✅ |

> 模型名 PascalCase 映射表名(snake_case)经 `@@map` 与冻结版一致;全部主键 `id text CUID`(冻结 §0.1);时间戳 `timestamptz` → Prisma `DateTime`,`@default(now())` + `@updatedAt`(冻结 §0.2)。

---

## 2. 逐表字段对照(§2.1-§2.24)

> 符号:✅ 一致;`D3` = 语义一致但该约束需 D3 Migration 原生 SQL 实现;`APP` = 应用层实现(冻结版自身定义在应用层)。

### 2.1 websites(19 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | `String @id @default(cuid())` | ✅ |
| domain | text | 否 | — | `String`(注释:非实体唯一键) | ✅ |
| normalized_domain | text | 否 | — | `String @unique` | ✅ |
| slug | text | 否 | — | `String @unique` | ✅ |
| display_name | text | 否 | — | `displayName String` | ✅(同步不覆盖=应用层语义,D4 落实) |
| homepage_url | text | 否 | 派生 `https://{normalized_domain}` | `homepageUrl String` 无 DB 默认 | ✅ 语义;派生在应用层(D4/D7 创建时),非 DB 默认 |
| title | text? | 是 | — | `title String?` | ✅ |
| description | text | 否 | — | `description String` | ✅ |
| summary_text | text? | 是 | — | `summaryText String?` | ✅ |
| category_id | text? | 是 | — | `categoryId String?` + FK | ✅ |
| status | text | 否 | `draft` | `@default("draft")` | ✅ + D3 CHECK |
| source | text | 否 | `manual` | `@default("manual")` | ✅ + D3 CHECK |
| seo_indexable | boolean | 否 | `false` | `@default(false)` | ✅ |
| data_quality_score | int | 否 | `0` | `@default(0)` | ✅ |
| first_seen_at | timestamptz? | 是 | — | `firstSeenAt DateTime?` | ✅ |
| last_seen_at | timestamptz? | 是 | — | `lastSeenAt DateTime?` | ✅ |
| published_at | timestamptz? | 是 | — | `publishedAt DateTime?` | ✅ |
| created_at / updated_at | timestamptz | 否 | now() | `@default(now())` / `@updatedAt` | ✅ |
| 索引 | — | — | — | `UNIQUE(normalized_domain)` `UNIQUE(slug)` `INDEX(category_id)` `INDEX(status, seo_indexable)` `INDEX(data_quality_score)` | ✅(§8 全量) |

### 2.2 categories(8 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| name | text | 否 | — | `String` | ✅ |
| slug | text | 否 | — | `@unique` | ✅ |
| parent_id | text? | 是 | — | `parentId String?` FK onDelete **SetNull** | ✅ |
| description | text | 否 | '' | `@default("")` | ✅ |
| sort_order | int | 否 | 0 | `@default(0)` | ✅ |
| status | text | 否 | `active` | `@default("active")` | ✅ + D3 CHECK |
| seo_indexable | boolean | 否 | `false` | `@default(false)` | ✅ |
| 索引 | — | — | — | `INDEX(parent_id)` `INDEX(status, sort_order)` | ✅ |

### 2.3 tags(7 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id / name / slug | text | 否 | — | ✅;slug `@unique` | |
| tag_type | text | 否 | — | `String` | ✅ + D3 CHECK(**仅四枚举,禁止第五种**) |
| group | text? | 是 | — | `group String?`(注释:仅管理分组) | ✅ |
| description | text | 否 | '' | `@default("")` | ✅ |
| status | text | 否 | `active` | `@default("active")` | ✅ + D3 CHECK |
| 索引 | — | — | — | `INDEX(tag_type, status)` | ✅ |

### 2.4 website_tags(6 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| website_id | text | 否 | — | FK onDelete **Cascade**(draft 硬删) | ✅ |
| tag_id | text | 否 | — | FK onDelete **Restrict** | ✅ |
| source | text | 否 | `manual` | `@default("manual")` | ✅ + D3 CHECK |
| confidence | int | 否 | 100 | `@default(100)` | ✅(0-100 范围 APP) |
| created_at | timestamptz | 否 | now() | ✅ | |
| 约束/索引 | — | — | — | `UNIQUE(website_id, tag_id)` `INDEX(tag_id)` | ✅ |

### 2.5 tasks(11 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id / name / slug | text | 否 | — | ✅;slug `@unique` | |
| description | text | 否 | '' | `@default("")` | ✅ |
| status | text | 否 | `draft` | `@default("draft")` | ✅ + D3 CHECK |
| seo_indexable | boolean | 否 | `false` | `@default(false)` | ✅ |
| candidate_threshold | int | 否 | 5 | `@default(5)` | ✅ |
| min_evidence_count | int | 否 | 3 | `@default(3)` | ✅ |
| candidate_count | int | 否 | 0 | `@default(0)` | ✅ |
| created_at / updated_at | timestamptz | 否 | now() | ✅ | |
| 索引 | — | — | — | `INDEX(status, seo_indexable)` | ✅ |

### 2.6 task_aliases(8 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| task_id | text | 否 | — | FK onDelete **Cascade** | ✅ |
| alias | text | 否 | — | `alias String` | ✅ |
| alias_normalized | text | 否 | — | `@unique` 全局唯一 | ✅ |
| weight | int | 否 | 50 | `@default(50)` | ✅ |
| source | text | 否 | `manual` | `@default("manual")` | ✅ + D3 CHECK |
| status | text | 否 | `active` | `@default("active")` | ✅ + D3 CHECK |
| created_at | timestamptz | 否 | now() | ✅ | |
| 索引 | — | — | — | `INDEX(task_id)` | ✅ |

### 2.7 task_related_tags(5 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| task_id | text | 否 | — | FK onDelete **Cascade** | ✅ |
| tag_id | text | 否 | — | FK onDelete **Restrict** | ✅ |
| source | text | 否 | `manual` | `@default("manual")` | ✅ + D3 CHECK |
| created_at | timestamptz | 否 | now() | ✅ | |
| 约束/索引 | — | — | — | `UNIQUE(task_id, tag_id)` `INDEX(tag_id)` | ✅ |

### 2.8 website_tasks(10 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| website_id | text | 否 | — | FK onDelete **Cascade** | ✅ |
| task_id | text | 否 | — | FK onDelete **Restrict** | ✅ |
| match_basis | text | 否 | '' | `@default("")` | ✅ |
| source | text | 否 | `manual` | `@default("manual")` | ✅ + D3 CHECK |
| sort_weight | int | 否 | 50 | `@default(50)` | ✅(0-100 APP) |
| override_score | int? | 是 | — | `overrideScore Int?` | ✅ |
| status | text | 否 | `active` | `@default("active")` | ✅ + D3 CHECK |
| created_at / updated_at | timestamptz | 否 | now() | ✅ | |
| 约束/索引 | — | — | — | `UNIQUE(website_id, task_id)` `INDEX(task_id)` `INDEX(website_id)` | ✅ |

### 2.9 website_relations(10 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| website_id_from / website_id_to | text | 否 | — | FK ×2 onDelete **Restrict** | ✅ |
| relation_type | text | 否 | — | `String` | ✅ + D3 CHECK;P0 allowlist 门控 = APP(仅 similar/alternative,related 禁止创建) |
| basis | text | 否 | '' | `@default("")` | ✅ |
| source | text | 否 | `manual` | `@default("manual")` | ✅ + D3 CHECK |
| confidence | int | 否 | 100 | `@default(100)` | ✅(0-100 APP) |
| status | text | 否 | `active` | `@default("active")` | ✅ + D3 CHECK |
| created_at / updated_at | timestamptz | 否 | now() | ✅ | |
| 约束/索引 | — | — | — | `UNIQUE(from, to, relation_type)` `INDEX(from)` `INDEX(to)` | ✅ |

### 2.10 website_snapshots(6 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| website_id | text | 否 | — | FK onDelete **Restrict** | ✅ |
| type | text | 否 | `sync` | `@default("sync")` | ✅ + D3 CHECK(P0 仅 sync) |
| data | jsonb | 否 | — | `Json` | ✅ |
| summary_hash | text | 否 | — | `summaryHash String` | ✅ |
| created_at | timestamptz | 否 | now() | ✅ | |
| 索引 | — | — | — | `INDEX(website_id, created_at DESC)` `INDEX(website_id, type, created_at)` | ✅(Prisma `sort: Desc`) |

### 2.11 website_status(19 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| website_id | text | 否 | — | FK onDelete **Restrict** + `@unique` | ✅ |
| online / http_status | boolean?/int? | 是 | — | `Boolean?` / `Int?` | ✅ |
| checked_at | timestamptz? | 是 | — | `DateTime?`(注释:不参与 hash) | ✅ |
| ip / asn / asn_org / country | text? | 是 | — | 4 × `String?` | ✅ |
| technologies | jsonb | 否 | `[]` | `Json @default("[]")` | ✅ |
| ssl_expires_at / ssl_days_remaining | timestamptz?/int? | 是 | — | `DateTime?` / `Int?` | ✅ |
| cdn / hosting | text? | 是 | — | `String?` | ✅ |
| seo_summary / health_summary | jsonb | 否 | `{}` | `Json @default("{}")` ×2 | ✅ |
| summary_hash | text | 否 | '' | `@default("")` | ✅(§8 明确不建索引) |
| last_sync_at | timestamptz? | 是 | — | `DateTime?` | ✅ |
| updated_at | timestamptz | 否 | now() | `@updatedAt` | ✅ |
| 索引 | — | — | — | `UNIQUE(website_id)` | ✅(仅此,无 summary_hash 索引) |

### 2.12 website_metrics(7 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| website_id | text | 否 | — | FK onDelete **Restrict** + `@unique` | ✅ |
| sync_count / change_count / task_count / month_clicks | int | 否 | 0 | `@default(0)` ×4 | ✅ |
| updated_at | timestamptz | 否 | now() | `@updatedAt` | ✅ |
| 索引 | — | — | — | `UNIQUE(website_id)` | ✅ |

### 2.13 website_changes(7 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| website_id | text | 否 | — | FK onDelete **Restrict** | ✅ |
| change_type | text | 否 | `siteintel_sync` | `@default("siteintel_sync")` | ✅ + D3 CHECK |
| delta | jsonb | 否 | — | `Json` | ✅ |
| hash_prev / hash_next | text | 否 | — | `hashPrev String` / `hashNext String` | ✅ |
| created_at | timestamptz | 否 | now() | ✅ | |
| 索引 | — | — | — | `INDEX(website_id, created_at DESC)` | ✅ |

### 2.14 recommendations(10 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| website_id | text | 否 | — | FK onDelete **Restrict** | ✅ |
| task_id | text? | 是 | — | `taskId String?` FK **Restrict** | ✅ |
| context | text | 否 | — | `String` | ✅ + D3 CHECK |
| score | float | 否 | — | `Float` | ✅ |
| confidence | text | 否 | `medium` | `@default("medium")` | ✅ + D3 CHECK |
| source | text | 否 | — | `String`(**无默认**,与冻结版一致) | ✅ + D3 CHECK |
| status | text | 否 | `active` | `@default("active")` | ✅ + D3 CHECK |
| created_at / updated_at | timestamptz | 否 | now() | ✅ | |
| 索引 | — | — | — | `INDEX(website_id, task_id, context)` `INDEX(context, status)` `INDEX(task_id)` | ✅ |

### 2.15 recommendation_evidence(8 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| recommendation_id | text | 否 | — | FK onDelete **Cascade** | ✅ |
| evidence_type | text | 否 | — | `String` | ✅ + D3 CHECK(**仅五枚举**) |
| factor_name | text | 否 | — | `factorName String` | ✅ |
| score | float | 否 | — | `Float` | ✅ |
| basis | jsonb | 否 | — | `Json` | ✅ |
| generated_by | text | 否 | `rule` | `@default("rule")` | ✅ + D3 CHECK |
| created_at | timestamptz | 否 | now() | ✅ | |
| 索引 | — | — | — | `INDEX(recommendation_id)` | ✅ |

### 2.16 curated_picks(9 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| website_id | text | 否 | — | FK onDelete **Restrict** | ✅ |
| task_id | text? | 是 | — | `taskId String?` FK **Restrict** | ✅ |
| reason | text | 否 | '' | `@default("")` | ✅ |
| sort_order | int | 否 | 0 | `@default(0)` | ✅ |
| status | text | 否 | `draft` | `@default("draft")` | ✅ + D3 CHECK |
| published_at | timestamptz? | 是 | — | `DateTime?` | ✅ |
| created_at / updated_at | timestamptz | 否 | now() | ✅ | |
| 索引 | — | — | — | `INDEX(status, sort_order)` | ✅ |

### 2.17 submissions(11 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| domain | text | 否 | — | `domain String` | ✅ |
| normalized_domain | text | 否 | — | `@unique` | ✅ |
| note | text | 否 | '' | `@default("")` | ✅ |
| ip / user_agent | text | 否 | '' | `@default("")` ×2 | ✅ |
| status | text | 否 | `pending` | `@default("pending")` | ✅ + D3 CHECK |
| reviewed_by / reviewed_at / review_note | text?/timestamptz?/text? | 是 | — | `String?` / `DateTime?` / `String?` | ✅ |
| created_at | timestamptz | 否 | now() | ✅ | |
| 约束 | — | — | — | `UNIQUE(normalized_domain)` | ✅(无 FK,§14) |

### 2.18 reports(9 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| website_id | text | 否 | — | FK onDelete **Restrict** | ✅ |
| field | text | 否 | — | `String` | ✅(白名单 10 项 = APP,冻结版自身定义) |
| current_value / expected_value | jsonb | 否 | — | `Json` ×2 | ✅ |
| note | text | 否 | '' | `@default("")` | ✅ |
| status | text | 否 | `pending` | `@default("pending")` | ✅ + D3 CHECK |
| resolved_value | jsonb? | 是 | — | `Json?` | ✅ |
| created_at | timestamptz | 否 | now() | ✅ | |
| 索引 | — | — | — | `INDEX(status, created_at)` | ✅ |

### 2.19 user_events(7 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| visitor_id | text | 否 | — | `visitorId String` | ✅ |
| event_type | text | 否 | — | `String` | ✅ + D3 CHECK(无 favorite) |
| website_id / task_id | text? | 是 | — | FK ×2 onDelete **SetNull** | ✅ |
| payload | jsonb? | 是 | — | `Json?` | ✅ |
| created_at | timestamptz | 否 | now() | ✅ | |
| 索引 | — | — | — | `INDEX(visitor_id, event_type)` `INDEX(event_type, created_at)` | ✅ |

### 2.20 search_logs(9 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| query / query_normalized | text | 否 | — | `query String` / `queryNormalized String` | ✅ |
| task_id | text? | 是 | — | FK onDelete **SetNull** | ✅ |
| constraints | jsonb | 否 | `[]` | `Json @default("[]")` | ✅ |
| result_count | int | 否 | 0 | `@default(0)` | ✅ |
| clicked_website_id | text? | 是 | — | FK onDelete **SetNull** | ✅ |
| visitor_id | text? | 是 | — | `String?` | ✅ |
| created_at | timestamptz | 否 | now() | ✅ | |
| 索引 | — | — | — | `INDEX(query_normalized, created_at)` `INDEX(task_id, created_at)` | ✅ |

### 2.21 sync_queue(17 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| domain | text | 否 | — | `domain String` | ✅ |
| normalized_domain | text | 否 | — | `normalizedDomain String` | ✅ |
| job_type | text | 否 | `sync` | `@default("sync")` | ✅ + D3 CHECK |
| payload | jsonb | 否 | `{}` | `Json @default("{}")` | ✅ |
| status | text | 否 | `pending` | `@default("pending")` | ✅ + D3 CHECK(状态机语义 = APP/D4) |
| priority | int | 否 | 0 | `@default(0)` | ✅ |
| locked_at / locked_by | timestamptz?/text? | 是 | — | `DateTime?` / `String?` | ✅ |
| attempts | int | 否 | 0 | `@default(0)` | ✅(先 +1 再判定 = APP,D4) |
| max_attempts | int | 否 | 5 | `@default(5)` | ✅(≥1 = APP;settings 可调) |
| next_retry_at | timestamptz? | 是 | — | `DateTime?` | ✅ |
| last_error / last_http_status / summary_hash | text?/int?/text? | 是 | — | `String?` / `Int?` / `String?` | ✅ |
| created_at / updated_at | timestamptz | 否 | now() | ✅ | |
| 约束/索引 | — | — | — | `UNIQUE(normalized_domain, job_type)` `INDEX(status, next_retry_at)` `INDEX(status, locked_at)` `INDEX(priority)` | ✅(无 FK,§14) |

### 2.22 sync_logs(10 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| domain | text | 否 | — | `domain String` | ✅ |
| normalized_domain | text? | 是 | — | `String?`(关联查询,非去重键) | ✅ |
| action | text | 否 | — | `String`(**无默认**) | ✅ + D3 CHECK |
| status | text | 否 | — | `String`(**无默认**) | ✅ + D3 CHECK |
| http_status / error | int?/text? | 是 | — | `Int?` / `String?` | ✅ |
| duration_ms / payload_size | int | 否 | 0 | `@default(0)` ×2 | ✅ |
| created_at | timestamptz | 否 | now() | ✅ | |
| 索引 | — | — | — | `INDEX(normalized_domain, created_at)` | ✅(无 FK,§14) |

### 2.23 settings(3 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| key | text | 否 | — | `@id`(**UNIQUE 主键**) | ✅ |
| value | jsonb | 否 | — | `Json` | ✅ |
| updated_at | timestamptz | 否 | now() | `@updatedAt` | ✅ |
| 约束 | — | — | — | 白名单 7 key = APP(冻结版自身定义) | ✅ |

### 2.24 admin_users(5 字段)

| 冻结字段 | 类型 | NULL | 默认 | Prisma | 备注 |
|---|---|---|---|---|---|
| id | text | 否 | CUID | ✅ | |
| email | text | 否 | — | `@unique` | ✅ |
| password_hash | text | 否 | — | `passwordHash String`(scrypt) | ✅ |
| name | text? | 是 | — | `String?` | ✅ |
| created_at | timestamptz | 否 | now() | ✅ | |
| 约束 | — | — | — | `UNIQUE(email)`;与 ADMIN_EMAILS 交集 = APP | ✅ |

---

## 3. Unique 约束对照(全量)

| # | 冻结 UNIQUE | Prisma | 对照 |
|---|---|---|---|
| 1 | websites.UNIQUE(normalized_domain) | `normalizedDomain String @unique` | ✅ |
| 2 | websites.UNIQUE(slug) | `slug String @unique` | ✅ |
| 3 | categories.UNIQUE(slug) | ✅ | ✅ |
| 4 | tags.UNIQUE(slug) | ✅ | ✅ |
| 5 | website_tags.UNIQUE(website_id, tag_id) | `@@unique([websiteId, tagId])` | ✅ |
| 6 | tasks.UNIQUE(slug) | ✅ | ✅ |
| 7 | task_aliases.UNIQUE(alias_normalized)(全局唯一) | `aliasNormalized String @unique` | ✅ |
| 8 | task_related_tags.UNIQUE(task_id, tag_id) | `@@unique([taskId, tagId])` | ✅ |
| 9 | website_tasks.UNIQUE(website_id, task_id) | `@@unique([websiteId, taskId])` | ✅ |
| 10 | website_relations.UNIQUE(from, to, relation_type) | `@@unique([websiteIdFrom, websiteIdTo, relationType])` | ✅ |
| 11 | website_status.UNIQUE(website_id) | `websiteId String @unique` | ✅ |
| 12 | website_metrics.UNIQUE(website_id) | `websiteId String @unique` | ✅ |
| 13 | submissions.UNIQUE(normalized_domain) | `normalizedDomain String @unique` | ✅ |
| 14 | sync_queue.UNIQUE(normalized_domain, job_type) | `@@unique([normalizedDomain, jobType])` | ✅ |
| 15 | settings.key UNIQUE(主键) | `@id` | ✅ |
| 16 | admin_users.UNIQUE(email) | `email String @unique` | ✅ |

**16/16 全一致**。domain 刻意不唯一(冻结 §5)✅;snapshots/status 的 summary_hash **不建索引**(冻结 §8 不建清单)✅。

---

## 4. Index 对照汇总(§8 全量,含排序)

| 表 | 冻结索引 | Prisma | 对照 |
|---|---|---|---|
| websites | INDEX(category_id) | `@@index([categoryId])` | ✅ |
| websites | INDEX(status, seo_indexable) | `@@index([status, seoIndexable])` | ✅ |
| websites | INDEX(data_quality_score) | `@@index([dataQualityScore])` | ✅ |
| categories | INDEX(parent_id) | `@@index([parentId])` | ✅ |
| categories | INDEX(status, sort_order) | `@@index([status, sortOrder])` | ✅ |
| tags | INDEX(tag_type, status) | `@@index([tagType, status])` | ✅ |
| website_tags | INDEX(tag_id) | `@@index([tagId])` | ✅ |
| tasks | INDEX(status, seo_indexable) | `@@index([status, seoIndexable])` | ✅ |
| task_aliases | INDEX(task_id) | `@@index([taskId])` | ✅ |
| task_related_tags | INDEX(tag_id) | `@@index([tagId])` | ✅ |
| website_tasks | INDEX(task_id) | `@@index([taskId])` | ✅ |
| website_tasks | INDEX(website_id) | `@@index([websiteId])` | ✅ |
| website_relations | INDEX(website_id_from) | `@@index([websiteIdFrom])` | ✅ |
| website_relations | INDEX(website_id_to) | `@@index([websiteIdTo])` | ✅ |
| website_snapshots | INDEX(website_id, created_at DESC) | `@@index([websiteId, createdAt(sort: Desc)])` | ✅ |
| website_snapshots | INDEX(website_id, type, created_at) | `@@index([websiteId, type, createdAt])` | ✅ |
| website_changes | INDEX(website_id, created_at DESC) | `@@index([websiteId, createdAt(sort: Desc)])` | ✅ |
| recommendations | INDEX(website_id, task_id, context) | `@@index([websiteId, taskId, context])` | ✅ |
| recommendations | INDEX(context, status) | `@@index([context, status])` | ✅ |
| recommendations | INDEX(task_id) | `@@index([taskId])` | ✅ |
| recommendation_evidence | INDEX(recommendation_id) | `@@index([recommendationId])` | ✅ |
| curated_picks | INDEX(status, sort_order) | `@@index([status, sortOrder])` | ✅ |
| reports | INDEX(status, created_at) | `@@index([status, createdAt])` | ✅ |
| user_events | INDEX(visitor_id, event_type) | `@@index([visitorId, eventType])` | ✅ |
| user_events | INDEX(event_type, created_at) | `@@index([eventType, createdAt])` | ✅ |
| search_logs | INDEX(query_normalized, created_at) | `@@index([queryNormalized, createdAt])` | ✅ |
| search_logs | INDEX(task_id, created_at) | `@@index([taskId, createdAt])` | ✅ |
| sync_queue | INDEX(status, next_retry_at) | `@@index([status, nextRetryAt])` | ✅ |
| sync_queue | INDEX(status, locked_at) | `@@index([status, lockedAt])` | ✅ |
| sync_queue | INDEX(priority) | `@@index([priority])` | ✅ |
| sync_logs | INDEX(normalized_domain, created_at) | `@@index([normalizedDomain, createdAt])` | ✅ |

**31/31 全一致**(外加 16 个 UNIQUE 的隐式索引)。无额外索引、无遗漏。

---

## 5. Relation(外键)对照汇总(§10 外键总则)

| # | FK | 冻结 onDelete | Prisma | 对照 |
|---|---|---|---|---|
| 1 | websites.category_id → categories.id | (主分类,无明确级联) | `onDelete: Restrict` | ✅ 历史保护(§10 总则 Restrict 类) |
| 2 | categories.parent_id → categories.id | **SetNull** | `onDelete: SetNull` | ✅ |
| 3 | website_tags.website_id → websites.id | **Cascade**(draft 硬删) | `onDelete: Cascade` | ✅ |
| 4 | website_tags.tag_id → tags.id | **Restrict** | `onDelete: Restrict` | ✅ |
| 5 | task_aliases.task_id → tasks.id | **Cascade** | `onDelete: Cascade` | ✅ |
| 6 | task_related_tags.task_id → tasks.id | **Cascade** | `onDelete: Cascade` | ✅ |
| 7 | task_related_tags.tag_id → tags.id | **Restrict** | `onDelete: Restrict` | ✅ |
| 8 | website_tasks.website_id → websites.id | **Cascade** | `onDelete: Cascade` | ✅ |
| 9 | website_tasks.task_id → tasks.id | **Restrict** | `onDelete: Restrict` | ✅ |
| 10 | website_relations.website_id_from → websites.id | **Restrict** | `onDelete: Restrict` | ✅ |
| 11 | website_relations.website_id_to → websites.id | **Restrict** | `onDelete: Restrict` | ✅ |
| 12 | website_snapshots.website_id → websites.id | **Restrict** | `onDelete: Restrict` | ✅ |
| 13 | website_status.website_id → websites.id | **Restrict** | `onDelete: Restrict` | ✅ |
| 14 | website_metrics.website_id → websites.id | **Restrict** | `onDelete: Restrict` | ✅ |
| 15 | website_changes.website_id → websites.id | **Restrict** | `onDelete: Restrict` | ✅ |
| 16 | recommendations.website_id → websites.id | **Restrict** | `onDelete: Restrict` | ✅ |
| 17 | recommendations.task_id → tasks.id | **Restrict** | `onDelete: Restrict` | ✅ |
| 18 | recommendation_evidence.recommendation_id → recommendations.id | **Cascade** | `onDelete: Cascade` | ✅ |
| 19 | curated_picks.website_id → websites.id | **Restrict** | `onDelete: Restrict` | ✅ |
| 20 | curated_picks.task_id → tasks.id | **Restrict** | `onDelete: Restrict` | ✅ |
| 21 | reports.website_id → websites.id | **Restrict** | `onDelete: Restrict` | ✅ |
| 22 | user_events.website_id → websites.id | **SetNull** | `onDelete: SetNull` | ✅ |
| 23 | user_events.task_id → tasks.id | **SetNull** | `onDelete: SetNull` | ✅ |
| 24 | search_logs.task_id → tasks.id | **SetNull** | `onDelete: SetNull` | ✅ |
| 25 | search_logs.clicked_website_id → websites.id | **SetNull** | `onDelete: SetNull` | ✅ |

**25 个 FK / 25 一致**。刻意**不建 FK**(冻结 §14 注):sync_queue / sync_logs 与 websites 仅 normalized_domain 逻辑关联、submissions 与 websites 审核联动 ✅。外键实现级别:DB 层 FK(Prisma 默认 relationMode,冻结附录 A.7 决策)。

---

## 6. Enum-CHECK 对照汇总(30 项,text + CHECK)

| # | 表.字段 | 冻结枚举值 | Prisma 表达 | 状态 |
|---|---|---|---|---|
| 1 | websites.status | draft/pending/active/archived | String + 注释 | **D3** |
| 2 | websites.source | manual/submission | String + 注释 | **D3** |
| 3 | categories.status | active/hidden | String + 注释 | **D3** |
| 4 | tags.tag_type | attribute/feature/constraint/scenario(仅四,禁第五种) | String + 注释 | **D3** |
| 5 | tags.status | active/hidden | String + 注释 | **D3** |
| 6 | website_tags.source | manual/rule | String + 注释 | **D3** |
| 7 | tasks.status | draft/active/pending_review/archived | String + 注释 | **D3** |
| 8 | task_aliases.source | manual/rule | String + 注释 | **D3** |
| 9 | task_aliases.status | active/hidden | String + 注释 | **D3** |
| 10 | task_related_tags.source | manual/rule | String + 注释 | **D3** |
| 11 | website_tasks.source | manual/rule | String + 注释 | **D3** |
| 12 | website_tasks.status | active/archived | String + 注释 | **D3** |
| 13 | website_relations.relation_type | similar/alternative/related(P0 allowlist 仅前二) | String + 注释 | **D3**(门控 APP) |
| 14 | website_relations.source | rule/manual | String + 注释 | **D3** |
| 15 | website_relations.status | active/superseded | String + 注释 | **D3** |
| 16 | website_snapshots.type | sync(P0 仅 sync) | String + 注释 | **D3** |
| 17 | recommendations.context | task_page/search/detail/home | String + 注释 | **D3** |
| 18 | recommendations.confidence | high/medium/low | String + 注释 | **D3** |
| 19 | recommendations.source | rule/manual | String + 注释 | **D3** |
| 20 | recommendations.status | active/superseded | String + 注释 | **D3** |
| 21 | recommendation_evidence.evidence_type | task_match/constraint_match/quality/status_signal/editorial(仅五) | String + 注释 | **D3** |
| 22 | recommendation_evidence.generated_by | rule/manual | String + 注释 | **D3** |
| 23 | curated_picks.status | draft/published/archived | String + 注释 | **D3** |
| 24 | submissions.status | pending/approved/rejected/duplicate | String + 注释 | **D3** |
| 25 | reports.status | pending/accepted/rejected/resolved | String + 注释 | **D3** |
| 26 | user_events.event_type | visit/reason_expand/submit/search_click(无 favorite) | String + 注释 | **D3** |
| 27 | sync_queue.job_type | sync(P0 唯一) | String + 注释 | **D3** |
| 28 | sync_queue.status | pending/processing/completed/retrying/failed/skipped | String + 注释 | **D3** |
| 29 | sync_logs.action | sync/analyze/report/diff | String + 注释 | **D3** |
| 30 | sync_logs.status | success/failed/retry | String + 注释 | **D3** |

> 30 项枚举全部按冻结 §0.3 用 **text + CHECK** 表达;Prisma 不支持 CHECK → 模型用 String + 注释标注枚举值集合,**语义与冻结版完全一致**,DB CHECK 统一在 **D3 Migration 层以原生 SQL 追加**(见下)。

---

## 7. D3 Migration 层需实现清单(非 SCR)

| 类别 | 内容 | 数量 |
|---|---|---|
| CHECK 约束 | §6 全部 30 项枚举 CHECK | 30 |
| 索引(Prisma 可表达,已在模型) | 无(全部由 Prisma 生成) | 0 |
| 其他 DB 约束 | 无(唯一/主键/外键全部 Prisma 原生) | 0 |

> 结论:Prisma 无法表达的约束**仅 CHECK 30 项**,全部登记 D3 Migration 层实现;模型语义与冻结版一致,不构成 SCR。D3 另需处理的非约束事项:数据库建库(nav_disc)、Prisma migrate 首迁、枚举 CHECK SQL、保留策略清理任务(应用层,非 Migration)。

---

## 8. 应用层约束登记(冻结版自身定义为应用层,非 DB)

| # | 约束 | 冻结依据 | 落地包 |
|---|---|---|---|
| 1 | reports.field 白名单 10 项(display_name/description/category_id/homepage_url/online/http_status/technologies/ssl_expires_at/cdn/hosting) | §2.18 | D4/D7(API 层,FIELD_NOT_ALLOWED) |
| 2 | settings.key 白名单 7 项(ranking.weights/quality.weights/task.index_thresholds/constraint.dictionary/website_changes.retention/search_logs.retention_days/sync_queue.max_attempts) | §2.23 | D7(API 层) |
| 3 | homepage_url 默认派生 `https://{normalized_domain}` | §2.1/§5 | D4/D7(创建时) |
| 4 | relation_type P0 allowlist 仅 similar/alternative | §2.9/§15-14 | D4/D5(API 门控,RELATION_TYPE_FORBIDDEN) |
| 5 | sync_queue 状态机/attempts 先 +1 再判定/退避/CAS 领取/10min 超时 | §9 | D6(worker) |
| 6 | recommendations 实时搜索永不写入 / superseded 不硬删 | §4 | D5/D8(规则) |
| 7 | 收藏本地化(localStorage,不进 DB/events) | §3 | 前端(D8) |
| 8 | 0-100 数值范围(confidence/sort_weight/data_quality_score 等) | 各字段说明 | 各包 API 校验 |
| 9 | sync_queue.max_attempts ≥ 1(settings 覆盖) | §2.23 | D7(设置校验) |
| 10 | 域名格式校验(pending → failed) | §9.1 | D4/D6 |
| 11 | ADMIN_EMAILS 交集校验 | §2.24 | D7.1(已有 admins.ts) |

---

## 9. 对照结论

1. **24 表 / 221 字段 / 16 UNIQUE / 31 索引 / 25 FK / 30 CHECK 枚举全部逐项一致**,无新增、无删减、无语义偏差;
2. Prisma 无法表达的 DB 约束仅 CHECK 30 项 → 统一登记 **D3 Migration 层实现**(非 SCR,冻结语义未改);
3. 冻结版自身定义在应用层的约束(白名单/门控/派生/状态机)全部登记 §8,不改变 DB 模型;
4. 实现过程修正 2 处 Prisma 模型错误(非冻结版问题):`Websites.status` 关系字段与标量重名 → 改名 `websiteStatus`;`Tasks.reports` 误加关系(冻结版 Reports 无 task_id)→ 删除。均为实现修正,冻结版无变化。
