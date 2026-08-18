# SiteIntel.cc — 全站深度分析报告
### Full-Site Structural, Functional & Interaction Analysis

**分析对象 / Target:** https://www.siteintel.cc
**数据来源 / Sources:** Live page fetches (homepage, `/how-it-works`, `/tools/website-analysis`, `/website/wordpress.org`, `/technology/wordpress`, `/ip/154.204.176.66`, `/search/google`, `/docs/api`) + `sitemap.xml` (110 indexed URLs)
**报告日期 / Report date:** 2026-08-15
**方法论 / Methodology:** Since the sitemap contains ~110 URLs that collapse into a small number of *reusable page templates* (each instantiated per domain/IP/ASN/certificate/organization/technology), this report analyzes the site by **template category** rather than enumerating every literal URL — each template's logic applies identically to all of its instances. Counts of instances per category are given in each section.

---

## 目录 / Table of Contents

1. [Site Summary](#1-site-summary)
2. [Global Architecture & Navigation Shell](#2-global-architecture--navigation-shell)
3. [Page Template Inventory](#3-page-template-inventory)
4. [Template A — Homepage](#4-template-a--homepage-)
5. [Template B — Static Marketing / Feature Pages](#5-template-b--static-marketing--feature-pages)
6. [Template C — Tool Landing Pages (`/tools/*`)](#6-template-c--tool-landing-pages-tools)
7. [Template D — Search-Intelligence Connector Pages (`/search/*`)](#7-template-d--search-intelligence-connector-pages-search)
8. [Template E — Guides & Case Studies (`/guides/*`, `/cases/*`)](#8-template-e--guides--case-studies-guides-cases)
9. [Template F — Website Intelligence Report (`/website/{domain}`)](#9-template-f--website-intelligence-report-websitedomain-flagship-product-page)
10. [Template G — IP Entity Page (`/ip/{address}`)](#10-template-g--ip-entity-page-ipaddress)
11. [Template H — ASN Entity Page (`/asn/{number}`)](#11-template-h--asn-entity-page-asnnumber)
12. [Template I — TLS Certificate Entity Page (`/certificate/{sha256}`)](#12-template-i--tls-certificate-entity-page-certificatesha256)
13. [Template J — Organization Entity Page (`/organization/{slug}`)](#13-template-j--organization-entity-page-organizationslug)
14. [Template K — Technology Entity Page (`/technology/{name}`)](#14-template-k--technology-entity-page-technologyname)
15. [Template L — Account / Utility Pages](#15-template-l--account--utility-pages)
16. [API Documentation & Backend Contract (`/docs/api`)](#16-api-documentation--backend-contract-docsapi)
17. [Cross-Cutting Frontend Components](#17-cross-cutting-frontend-components)
18. [Data Model & Backend Business Logic (Inferred)](#18-data-model--backend-business-logic-inferred)
19. [Data Flow / Pipeline Architecture](#19-data-flow--pipeline-architecture)
20. [SEO & Technical Implementation Notes](#20-seo--technical-implementation-notes)
21. [Unimplemented / Placeholder Features](#21-unimplemented--placeholder-features)
22. [Potential Defects, Risks & UX Gaps](#22-potential-defects-risks--ux-gaps)
23. [Sitemap Coverage Map](#23-sitemap-coverage-map)
24. [Summary Assessment](#24-summary-assessment)

---

## 1. Site Summary

| Attribute | Value |
|---|---|
| Product name | SiteIntel |
| Domain | siteintel.cc (canonical: `https://siteintel.cc`, non-www redirects observed) |
| Tagline | "分析网站。关联数据。发现洞察。" (Analyze websites. Correlate data. Discover insights.) |
| Category | Website / domain intelligence platform (OSINT-style: DNS, IP, ASN, SSL, tech-stack fingerprinting, change detection, relationship graphing) |
| Comparable products | BuiltWith, SecurityTrails, DNSDumpster, Wappalyzer, urlscan.io — combined into one "investigation → report → monitoring" product |
| Detected stack | Next.js (frontend, SSR/SSG hybrid), inferred Node.js/serverless API layer, structured content stored as JSON facts with a rule engine on top |
| Locales | Simplified Chinese (`zh_CN`, default) and English (toggle in header, `中文`/`EN`) |
| Homepage stats (as observed) | 184 domains analyzed · 291 investigations · 1,794 entities · 3,206 relationships · 1,999 snapshots · 21 insights |
| Monetization signals | `/login`, API key system (`si_<48-hex>`), rate-limited tiers (100 report calls/hr, 30 analyze calls/hr per key) — suggests freemium/API-key gated model |
| Sitemap size | 110 URLs total (see [§23](#23-sitemap-coverage-map)) |

---

## 2. Global Architecture & Navigation Shell

Every page in the site (all 110 sitemap URLs, confirmed across every template sampled) shares an identical **shell**:

### 2.1 Header (persistent, sticky)
```
[▙SITEINTEL] (logo/home link)
 ├─ 分析什么 (What we analyze)   → anchor: /#modules
 ├─ 工作原理 (How it works)       → /how-it-works (or /#pipeline on homepage)
 ├─ 报告 (Report)                 → /report
 ├─ 批量分析 (Bulk analysis)      → /bulk
 ├─ 搜索情报 (Search intelligence)→ /search
 ├─ API 文档 (API docs)           → /docs/api
 ├─ 登录 (Login)                  → /login
 ├─ 中文 / EN (locale switch)
 └─ ☰ (mobile hamburger menu)
```
- The nav is **identical on every page type**, meaning it is rendered from a shared layout component (consistent with Next.js `layout.tsx` / App Router convention).
- Breadcrumb trail (`首页 / <section> / <current page>`) appears on all non-homepage pages, giving three-level hierarchical navigation and reinforcing crawlability/SEO internal-linking.

### 2.2 Global "Analyze" Input Widget
Appears on **every** page (homepage hero, all static pages, and — inferred — atop entity report pages via the sticky "▸ 分析" control):
- Text input for a domain
- Autofill/example chips: `cloudflare.com`, `github.com`, `openai.com` — likely rotating or fixed demo suggestions
- A "▸ 分析" (Analyze) submit button
- Functional hypothesis: submits to `POST /api/v1/analyze`, which is asynchronous (see [§16](#16-api-documentation--backend-contract-docsapi)); client likely opens an SSE connection to `/api/analyze/{id}/stream` and shows a **live multi-stage progress indicator** matching the 7-stage pipeline (采集→标准化→分析→关联→对比→检测→洞察), then performs a client-side redirect to `/website/{domain}` once `event: done` fires.

### 2.3 Footer (persistent)
```
SITEINTEL
分析网站。关联数据。发现洞察。
数据成为信息 · 信息成为关系 · 关系成为情报
© 2026 SiteIntel.cc — 网站数据情报平台
```
- No visible footer sitemap/legal links (no ToS/Privacy links observed in fetched content) — potential compliance gap, see [§22](#22-potential-defects-risks--ux-gaps).

---

## 3. Page Template Inventory

| # | Template | Sitemap pattern | Approx. instance count | Purpose |
|---|---|---|---|---|
| A | Homepage | `/`, `/index` (both listed) | 2 (duplicate canonical + trailing-slash variant) | Landing / entry point |
| B | Static marketing/feature pages | `/website-intelligence`, `/how-it-works`, `/relationship-intelligence`, `/search-intelligence`, `/infrastructure-intelligence`, `/technology-intelligence`, `/website-monitoring` | 7 | Product-feature explainer / SEO content |
| C | Tool landing pages | `/tools/website-analysis`, `/tools/dns`, `/tools/ip`, `/tools/ssl`, `/tools/technology` | 5 | Task-specific analyzer entry points |
| D | Search-intelligence connector stubs | `/search/google`, `/search/bing`, `/search/baidu` | 3 | Future SEO/GSC-style integration |
| E | Guides & case studies | `/guides/website-migration`, `/guides/baidu-seo`, `/guides/ssl`, `/cases/infrastructure-migration`, `/cases/search-traffic-drop` | 5 | Long-form SEO content, internal linking bait |
| F | Website intelligence report | `/website/{domain}` | 14 (sampled; live count grows continuously) | **Flagship product** — full entity report |
| G | IP entity page | `/ip/{address}` | 11 | Infrastructure entity hub |
| H | ASN entity page | `/asn/{number}` | 29 | Network-operator entity hub |
| I | TLS certificate entity page | `/certificate/{sha256}` | 11 | Certificate entity hub |
| J | Organization entity page | `/organization/{slug}` | 9 | WHOIS/registrant-organization entity hub |
| K | Technology entity page | `/technology/{name}` | 24 | Tech-stack fingerprint entity hub |
| L | Account/utility pages | `/report`, `/bulk`, `/docs/api`, `/login`, `/claim/{domain}` | Not all in sitemap (some are functional/gated, hence excluded from indexation) | Account, batch tooling, docs |

**Total indexed URLs in sitemap:** 110 (see full breakdown in [§23](#23-sitemap-coverage-map)).

---

## 4. Template A — Homepage (`/`)

### 4.1 Structure (top to bottom)
1. **Header + language toggle** (see §2.1)
2. **Hero section**
   - H1: "网站数据分析与情报洞察平台"
   - Subhead: "分析网站。关联数据。发现洞察。"
   - Description paragraph explaining the value prop
   - Primary CTA: domain input + "▸ 分析" button + 3 example-domain quick-fill chips
3. **Live platform stats bar** — 5 counters: analyzed domains (184), investigations (291), entities (1,794), relationships (3,206), snapshots (1,999), insights (21)
   - Functional note: these numbers appear **dynamically sourced** from the backend (not static marketing copy), since they mirror exact figures referenced later in per-entity pages (e.g., wordpress.org's 34 investigations, 75 relationships) — strongly implies a live aggregate SQL/count query rendered server-side.
4. **"适合哪些用户" (Who it's for)** — persona tag list: 网站管理员 / SEO 从业者 / 独立站运营者 / 开发者 / 技术研究人员 / 竞品分析人员 (site admins, SEO practitioners, indie site operators, developers, researchers, competitive analysts)
5. **"真实 Intelligence 示例" (Real intelligence examples)** — 4 live report cards pulled dynamically (e.g., wordpress.org, twitter.com, a Bing tile CDN hostname, bilibili.com), each showing investigation/event/insight counts and a "查看报告 →" deep link into Template F.
6. **"SiteIntel 分析什么" (What SiteIntel analyzes)** — 5 numbered feature blocks (01–05): 网站 / 基础设施 / 技术 / 关系 / 洞察 (Website / Infrastructure / Technology / Relationships / Insights), each with a short description, plus a global disclaimer: *"每个结论都明确标注为「观察到」「很可能」或「推断」，0-100 置信度评分，证据链可一路追溯到原始采集数据"* — this confidence/evidence framework is the site's core epistemic design pattern and repeats on **every** entity page.
7. **"情报管线" (Intelligence pipeline)** — 7-stage horizontal pipeline diagram: 采集→标准化→分析→关联→对比→检测→洞察 (Collect → Normalize → Analyze → Correlate → Compare → Detect → Insight), each rendered as a numbered node with an arrow connector — likely an SVG/flex row component, reused conceptually (in prose form) on `/how-it-works`.
8. **Closing CTA** — repeats the analyze box ("你的目标基础设施里藏着什么？")
9. **Footer**

### 4.2 Interaction logic
- Domain input likely client-validated (regex for domain syntax) before POST.
- Example chips are probably `onClick` handlers that populate the input field (not direct navigation), consistent with a controlled-input React pattern.
- "真实 Intelligence 示例" cards are anchor tags routing directly into Template F — no client-side interaction beyond hover states.

---

## 5. Template B — Static Marketing / Feature Pages

**Instances:** `/website-intelligence`, `/how-it-works`, `/relationship-intelligence`, `/search-intelligence`, `/infrastructure-intelligence`, `/technology-intelligence`, `/website-monitoring`

### 5.1 Shared structural pattern (confirmed via `/how-it-works` fetch)
```
Header
Breadcrumb: 首页 / 产品 / <page title>
H1 + one-line subhead
Analyze box (repeated CTA, with example chips)
N × content sections, each with an H2 and a short paragraph
Closing CTA ("输入域名试试" + Analyze box)
"相关页面" (Related pages) — 2–3 contextual internal links
Footer
```

### 5.2 `/how-it-works` — detailed breakdown
This page is the **prose version of the 7-stage pipeline** shown iconographically on the homepage, expanded into 10 explanatory H2 sections:

1. 输入网站 (Input a site) — domain validation + normalization + investigation creation with a unique traceable ID
2. 并行采集公开数据 (Parallel public-data collection) — **7 independent collectors** named explicitly: DNS records, IP & ASN, TLS certificate, HTTP response, tech fingerprinting, page metadata, domain registration (WHOIS/RDAP)
3. 原始数据留痕 (Raw data retention) — raw collector responses persisted as the root of the evidence chain
4. 识别实体 (Entity extraction) — domain, IP, ASN, organization, technology, SSL certificate, tracking ID; entity deduplication/merging across investigations ("同一实体会在多次调查中合并")
5. 建立关系 (Relationship building) — typed edges (resolves-to, belongs-to, uses, shares) with observed/likely/inferred labeling and confidence
6. 快照与历史 (Snapshots & history) — hash-fingerprinted snapshots per investigation, enabling point-in-time diffing
7. 检测变化 (Change detection) — new-vs-historical snapshot diffing produces discrete "events" with before/after values and timestamps
8. 生成洞察 (Insight generation) — rule engine interprets single events as "signals" and clustered events as "patterns" (e.g., infrastructure migration, frequent-change patterns), each with explainable confidence
9. 情报报告 (Intelligence report) — final 7-section report format ("5 秒看懂一个网站" — "understand a site in 5 seconds")
10. 持续监控 (Continuous monitoring) — daily re-analysis once a site is added to monitoring, with **Telegram or Webhook** alerting on detected changes

> **This page is effectively the system's own architecture documentation exposed to end users** — highly useful for reverse-engineering the backend design, and reused verbatim in this report's [§19](#19-data-flow--pipeline-architecture).

### 5.3 Other Template B pages (inferred structure, not individually fetched)
- `/website-intelligence` — overview/pillar page for the "网站" module, likely funnels to `/tools/website-analysis`.
- `/relationship-intelligence` — pillar page for the entity graph feature; likely embeds a static/demo graph visual.
- `/search-intelligence` — pillar page for the (currently unbuilt) SEO/search-console integration; hub linking to `/search/google`, `/search/bing`, `/search/baidu`.
- `/infrastructure-intelligence` — pillar page for DNS/IP/ASN/SSL, funnels to `/tools/dns`, `/tools/ip`, `/tools/ssl`.
- `/technology-intelligence` — pillar page for tech fingerprinting, funnels to `/tools/technology`.
- `/website-monitoring` — pillar page for the daily-monitoring + alerting feature described in §5.2 step 10; likely gated behind login given it implies a persistent job/subscription.

---

## 6. Template C — Tool Landing Pages (`/tools/*`)

**Instances:** `/tools/website-analysis`, `/tools/dns`, `/tools/ip`, `/tools/ssl`, `/tools/technology`

### 6.1 Confirmed structure (via `/tools/website-analysis` fetch)
```
Header
Breadcrumb: 首页 / 工具 / <tool name>
H1: "<Tool> 工具"
One-line value prop
Analyze box + example chips
"如何使用" (How to use) — explains the collection process & report format
"能查到什么" (What you can find out) — bullet-style capability list
Closing CTA
"相关页面" — 2 links to sibling pillar/tool pages
Footer
```

### 6.2 `/tools/website-analysis` specifics
- Explicitly states parallel collection of DNS / IP / SSL / HTTP / tech-stack / page metadata, with **real-time progress display** during analysis ("实时展示分析进度") — confirms the SSE-driven progress UX hypothesized in §2.2.
- Output: the same 7-section report format referenced everywhere else (关键洞察 / 网站概览 / 基础设施 / 技术 / 历史 / 关系 / 证据).
- "能查到什么" enumerates: IP & ASN, hosting & CDN, DNS records & nameservers, SSL certificate chain, tech-stack fingerprint, tracking IDs, historical change events, and insights.

### 6.3 Other tool pages (inferred, same template)
- `/tools/dns` — likely a **narrower-scope entry point** that still triggers the full pipeline but foregrounds the DNS-record table output.
- `/tools/ip` — likely accepts either a domain or a raw IP and routes to Template G (`/ip/{address}`) directly rather than Template F.
- `/tools/ssl` — likely foregrounds the TLS-certificate chain/expiry section and may route to Template I (`/certificate/{sha256}`).
- `/tools/technology` — likely foregrounds the tech-fingerprint section and may route to Template K (`/technology/{name}`).

> **Design pattern observed:** all five tool pages funnel through the *same* underlying `/api/v1/analyze` pipeline — they are essentially marketing/SEO-differentiated doorways into one unified backend, not functionally distinct engines. This is a deliberate SEO strategy (long-tail keyword targeting: "DNS 查询", "IP 查询", "SSL 查询", "技术栈检测") rather than a product-architecture distinction.

---

## 7. Template D — Search-Intelligence Connector Pages (`/search/*`)

**Instances:** `/search/google`, `/search/bing`, `/search/baidu`

### 7.1 Confirmed structure (via `/search/google` fetch)
```
Header
Breadcrumb: 首页 / 搜索情报 / <Engine> SEO数据分析
H1 + subhead: "Google Search Console 官方数据接入（Phase 3）"
3 × feature-preview sections (no live data):
  - 搜索表现 (Search performance: clicks/impressions/CTR/avg. rank)
  - 查询与页面 (Queries & pages: keyword/page/country/device dimensions)
  - 搜索事件关联 (Search-event correlation: ties search changes to infra changes)
CTA button: "连接 Google Search Console" → routes to "/" (homepage) — NOT an OAuth flow
"相关页面" — links to /search-intelligence and sibling engine pages (/search/bing)
Footer
```

### 7.2 Functional status: **Not implemented**
- The explicit label **"Phase 3"** and a CTA button that resolves to the homepage rather than a GSC OAuth consent screen confirm this is a **placeholder/roadmap page**, not a working feature.
- Same pattern expected for `/search/bing` (Bing Webmaster Tools) and `/search/baidu` (Baidu Ziyuan/站长平台), differing only in engine-specific copy (曝光/点击/排名 terminology, and for Baidu likely 百度站长平台 branding).
- On live `/website/{domain}` reports, this same "未连接" (not connected) empty-state is echoed (see §9, section 03).

---

## 8. Template E — Guides & Case Studies (`/guides/*`, `/cases/*`)

**Instances:** `/guides/website-migration`, `/guides/baidu-seo`, `/guides/ssl`, `/cases/infrastructure-migration`, `/cases/search-traffic-drop`

### 8.1 Inferred structure (long-form SEO content, not individually fetched)
```
Header
Breadcrumb: 首页 / 指南 (or 案例) / <title>
H1 + long-form article body (likely Markdown-rendered, with H2 subheadings)
Embedded Analyze CTA (mid-article and/or end-of-article)
"相关页面" internal links
Footer
```

### 8.2 Content hypotheses based on titles
- `/guides/website-migration` — how-to guide on detecting/executing site migrations, presumably referencing the platform's own "infrastructure_migration" insight type as a self-referential demonstration.
- `/guides/baidu-seo` — China-market SEO guide, tied to the (unimplemented) `/search/baidu` connector — content-marketing bridge for CN audience given zh_CN default locale and heavy presence of `.cn`/Chinese ASN data throughout the entity graphs (Baidu, Alibaba Cloud, China Telecom/Unicom/Mobile all appear as organizations in the sitemap).
- `/guides/ssl` — SSL/TLS certificate basics, funnels to `/tools/ssl`.
- `/cases/infrastructure-migration` — case study, plausibly using wordpress.org's real, live, extremely frequent DNS/provider-change history (see §9.6) as a worked example — notable because wordpress.org's own report already exhibits ~36 events in 30 days, which would make excellent (if slightly noisy/synthetic-looking) case-study material.
- `/cases/search-traffic-drop` — case study pairing a traffic-drop narrative with infrastructure change correlation — directly foreshadows the (currently unbuilt) "搜索事件关联" feature from Template D.

> These pages exist purely in the sitemap; without a direct fetch, exact interaction elements cannot be confirmed. Recommend a follow-up fetch pass if content-level SEO/keyword auditing is required.

---

## 9. Template F — Website Intelligence Report (`/website/{domain}`) — **Flagship Product Page**

This is the most complex template on the site and the actual product deliverable. Verified in full via `/website/wordpress.org`.

### 9.1 Page-level meta behavior
- Per-domain dynamic `<title>`, meta description, and **dynamically generated Open Graph image** (`/website/{domain}/opengraph-image?<hash>`) — implies server-side OG image generation (likely via Next.js `ImageResponse`/Vercel OG or similar), hashed per content version for cache-busting.
- `meta-robots: index, follow` — these pages **are** meant to be indexed (unlike IP pages, see §10).

### 9.2 Header block
```
首页 / 网站分析 / WORDPRESS.ORG   (breadcrumb)
# wordpress.org
首次发现 <date> · 最近更新 <date> · 调查次数 34 · 最近一次 COMPLETED
[↻ 重新分析] (re-analyze button)
[这是你的网站吗？] → /claim/{domain}   (ownership claim CTA)
[⇩ 导出 JSON] (export button)
```
- "↻ 重新分析" (re-analyze) is a **user-triggered action button** — presumably calls `POST /api/v1/analyze` again for this domain, re-running the full pipeline on demand (possibly rate-limited or login-gated to prevent abuse).
- "这是你的网站吗？" (claim ownership) routes to a **separate, non-sitemapped claim flow** (`/claim/{domain}`) — implies a verification mechanism (DNS TXT record or file upload, typical for domain-ownership proofs) feeding into monitoring/paid features.
- JSON export directly exposes the underlying data model to the user — strong signal that the report is backed by a well-structured JSON API object (consistent with `/api/v1/report/{domain}` in the API docs).

### 9.3 Section 01 — 执行摘要 (Executive Summary)
- One-paragraph plain-language synthesis (platform detected, HTTP status, meta description).
- Compact key-value grid: CDN / 托管 (hosting) / DNS 服务商 (DNS provider) / 邮箱 (mail provider).
- Technology chip list (each chip links to Template K).
- "最近活动" (recent activity) — rolling count of events in the last 30 days, tagged by event-type keywords (`provider_changed`, `infrastructure_migration`, `dns_changed`).
- "重点关注" (highlights) — top 1–2 insights with confidence %, using a bullet+dot severity marker (●).
- "继续探索" (continue exploring) — quick-jump chips to the domain's own IP(s), ASN, certificate, and a relationship count badge ("75 条关系") that anchors to section 09.

### 9.4 Section 02 — 关键洞察 (Key Insights)
- Card-per-insight layout. Each card contains:
  - Severity tag (中/高 = medium/high) + machine insight-type slug (e.g. `infrastructure_migration_event`, `provider_change`, `frequent_changes`, `ip_change`, `metadata_change`)
  - Confidence percentage + qualitative label (可能/较强/高 = possible/strong/high)
  - Human-readable title + one-sentence explanation
  - First-detected timestamp
  - Expandable "▸ 查看证据 (N)" (view evidence) control — collapsible, count-badged
- **Observed insight taxonomy** (from this single domain alone): `metadata_change`, `infrastructure_migration_event`, `provider_change`, `infrastructure_migration` (a *second*, differently-worded near-duplicate insight type — see §22 defect notes), `frequent_changes`, `ip_change`.

### 9.5 Section 03 — 搜索情报 (Search Intelligence)
- Empty-state module: "该网站尚未连接搜索引擎数据" (this site hasn't connected search-engine data) + CTA linking to `/search`.
- Directly reflects the unimplemented Template D functionality — this is the *integration point* where a future GSC/Bing/Baidu connection would inject data into the per-domain report.

### 9.6 Section 04 — 这是什么网站？ ("What is this site?")
- Summary card: page `<title>` + meta description as scraped.
- Descriptive chip tags (内容管理/带统计分析/SEO 完善/英文 — CMS-driven / has analytics / SEO-complete / English-language).
- "解读（推断，附证据）" (Interpretation, inferred, with evidence) — a single inferred classification statement with confidence % and a literal evidence snippet (e.g., detecting `/wp-content/` path as WordPress evidence).
- Live status block: online/offline, HTTP status code, last-checked timestamp.
- Investigation status block: most recent investigation status (`COMPLETED`) + completion timestamp.

### 9.7 Section 05 — 基础设施 (Infrastructure)
Data-dense section containing:
- **IP addresses** — each linked to Template G, annotated inline with ASN (linked to Template H) and owning organization (linked to Template J).
- **Hosting** — value + confidence % + evidence source string (e.g., "ASN owner reported: WordPress.org (2635)").
- **CDN** — either detected provider or explicit "未检测到 CDN 信号" (no CDN signal detected) negative-result state.
- **DNS provider** — value + confidence + nameserver list.
- **TLS certificate** — issuer, validity window, days-remaining counter, SHA-256 fingerprint (linked to Template I), SAN list.
- **Mail** — MX-derived provider or explicit "无 MX 记录" (no MX records) negative-result state.
- **Domain registration (WHOIS/RDAP)** — registrar, creation date, expiry date.
- **HTTP & security** — status code, final resolved host (post-redirect), and a **security-header checklist** (HSTS ✓/✕, X-Frame-Options ✓/✕, Referrer-Policy ✓/✕, X-Content-Type-Options ✓/✕, CSP ✓/✕) rendered as inline pass/fail icons.
- **DNS record table** — full raw record dump (A/AAAA/MX/NS/SOA/TXT), rendered as a literal markdown/HTML table with type, name, value columns.

### 9.8 Section 06 — 技术 (Technology)
- Grid of detected technologies grouped by category (内容管理/后端/统计分析/标签管理/字体 — CMS/Backend/Analytics/Tag-manager/Fonts), each item linking to Template K. Header states total count ("检测到 5 项").

### 9.9 Section 07 — 网站健康度 (Website Health Score) — Rule-Based Audit Engine
This is the most sophisticated module on the page:
- **Composite score** (0–100, e.g., "96") + fraction of dimensions scored ("5/6 个维度已评分").
- **5 scoring dimensions**, each independently scored and individually expandable:
  1. **技术 (Technical)** — HTTPS/TLS handshake success, redirect-chain length (≤3 hop rule), HTTP status code.
  2. **安全 (Security)** — HSTS, CSP, X-Content-Type-Options, Referrer-Policy, certificate-validity window — each check independently pass/"可优化" (optimizable) rated, with a specific remediation tip.
  3. **基础设施 (Infrastructure)** — nameserver redundancy (≥2 NS rule), IP redundancy (multi-A-record rule, with an explicit caveat that shared IP ≠ shared owner), CDN presence ("未评估" — not applicable/not scored if absent, explicitly *not* penalized), mail configuration ("未评估" if no MX).
  4. **SEO** — title tag, meta description, robots.txt presence, sitemap presence, `<html lang>` declaration, H1 count/uniqueness (with an explicit SPA caveat: "SPA 客户端渲染的 H1 无法被采集，属正常" — client-rendered H1s are acknowledged as uncrawlable and treated as normal, not penalized), structured data (JSON-LD block count).
  5. **内容 (Content)** — title length (10–60 char rule), description length (50–160 char rule), favicon presence, word count (300+ word guideline), internal/external link counts (5+ internal-link guideline).
  6. **性能 (Performance)** — explicitly marked **"待评估（暂无真实数据）"** (pending — no real data yet), with an internal design principle cited verbatim: **"§26 原则：不凭空评分"** ("Principle §26: never score without real data") — i.e., the platform has an internal numbered principles/style-guide document, and refuses to fabricate a performance score without real Core Web Vitals / response-time telemetry.
- Each individual check renders: label, pass/optimizable state, the literal detected value (e.g., "剩余 70 天" / "70 days remaining"), and a one-line actionable recommendation.

### 9.10 Section 08 — 行动路线 (Action Roadmap) — Prioritization Engine
- Framed explicitly as driven by an **"Impact × Urgency ÷ Effort"** formula.
- "当前优势" (current strengths) / "当前短板" (current weaknesses) — auto-derived from the health-score pass/fail set.
- **Time-boxed roadmap buckets**: 0-30 天·基础 (foundational), 31-60 天·优化 (optimization), 61-90 天·增长 (growth) — each populated with P0/P1-labeled action cards pulled directly from failed health checks (e.g., "P1 · CSP · 影响: medium · 工作量: M").
- CTA: **"AI 解读行动计划"** (AI-interpreted action plan) — implies an LLM-generated narrative summary of the roadmap is available on demand (likely a separate async call, possibly gated).

### 9.11 Section 09 — 关系 (Relationships) — Interactive Entity Graph
- Two view-mode toggle: **◉ 实体图谱** (entity graph / node-link visualization) vs. **≡ 关系** (flat relationship list).
- Legend acts as an **interactive filter**: clicking a relationship-type chip (使用/归属/解析到/共享/关联, i.e. uses/belongs-to/resolves-to/shares/associated-with) filters the rendered graph — confirmed by repeated help text: *"点击图例可按类型筛选"* (click the legend to filter by type).
- Node types color/shape-coded via a second legend: 域名/子域名/IP/ASN/组织/服务商/技术/证书/追踪 ID/邮件服务/nameserver (domain/subdomain/IP/ASN/organization/provider/technology/certificate/tracking-ID/mail-service/nameserver).
- Nodes are clickable and route to the corresponding entity template (F/G/H/I/J/K as appropriate) — this is the primary **cross-linking mechanism** that turns the whole site into a browsable knowledge graph.
- Interaction affordances explicitly documented in-page: draggable nodes, scroll-wheel zoom ("可拖拽节点、滚轮缩放").
- For wordpress.org specifically, the graph surfaced **~40+ unrelated third-party domains** (e.g., cloudflare.com, weibo.com, youtube.com, android.com, dozens of Chinese portals) as connected nodes via shared relationship types (`使用` ×68 — likely all sharing the same nameserver-host or GA/GTM tracking ID) — see §22 for analysis of this as a potential relevance/noise issue.

### 9.12 Section 10 — 历史 (History) — Event/Change Log
- Reverse-chronological, fully timestamped diff feed ("36 个事件" for wordpress.org).
- Each entry: change-type label (服务商变更/DNS 记录变更/IP 变更/infrastructure_migration/metadata_changed), severity tag (高/中/低), exact timestamp, and a literal **before → after JSON diff** rendered inline.
- Certain entries (e.g., `infrastructure_migration`) reference internal `signalIds` (e.g., `cmsty5jwf0031s51ahsp1qot5`) — these look like CUID/collision-resistant IDs, strongly suggesting a Prisma/CUID-based database schema on the backend.
- **Extremely high change frequency observed**: wordpress.org logged 36 "provider changed"-type events within roughly 5 hours of wall-clock time in the sample (e.g., 04:39 → 09:49), cycling between the same 4 nameservers (ns1–ns4) and 2 mail servers (smtp1/smtp2-dca) — this is very likely **round-robin DNS being misclassified as a genuine "provider change" event** rather than an actual infrastructure migration. This is flagged as a probable **false-positive-generating defect** in §22.

### 9.13 Section 11 — 相关网站 (Related Websites)
- Lists other domains discovered to share infrastructure (same IP/cert/ASN). For wordpress.org, this returned an explicit empty state: "暂未发现共享基础设施的其他域名" (no shared-infrastructure domains found yet), with a note that this improves as more sites are analyzed — indicates the corpus (184 domains) is still small enough to produce sparse correlation results.

### 9.14 Section 12 — 原始数据与证据 (Raw Data & Evidence)
- Collapsible list of the **7 raw collector runs** (mirrors the 7-source list from `/how-it-works`): `dns`→`dns_records`, `rdap`→`domain_rdap`, `ip`→`ip_enrichment`, `ssl`→`tls_certificate`, `http`→`http_response`, `metadata`→`website_metadata`, `technology`→`technology_profile` — each shown with collector name, internal fact-type key, status ("成功"/success), and payload byte-size (e.g., "17.9KB" for the TLS collector). This is effectively a **developer-facing debug/audit panel** exposed directly to end users — unusually transparent for a consumer product, and directly substantiates the "evidence chain" claim made throughout the site's marketing copy.

---

## 10. Template G — IP Entity Page (`/ip/{address}`)

Verified via `/ip/154.204.176.66`.

### 10.1 Meta behavior
- `meta-robots: noindex, follow` — **explicitly de-indexed** (unlike Template F, which is `index, follow`). This is a deliberate SEO decision: IP pages are treated as internal-linking/crawl-budget scaffolding, not landing-page targets, likely because they have low unique search-intent value and are easy to spam-index at scale (11+ instances already, growing unbounded as more IPs are discovered).
- Notably, the `og:title`/`og:description`/`og:image` metadata on this IP page are **copy-pasted from the homepage** (identical strings to Template A's OG tags) rather than IP-specific — see §22 defect notes.

### 10.2 Structure
```
Breadcrumb: 首页 / 基础设施情报 / <IP address>
# <IP address>
"N 个已观察网站解析到此 IP · <Org> · <Country>"
ASN → (linked chip, Template H)
归属组织 (owning org)
ISP / 服务商 (ISP/provider)
位置 (location: city · country)

## 此 IP 上观察到的网站 (Websites observed on this IP)
   — chip list, each linking to Template F, with discovery date

## 关联 IP（同 ASN）(Related IPs, same ASN)
   — chip list with co-occurrence count, linking to sibling Template G pages

## 观察到的 TLS 证书 (Observed TLS certificates)
   — list of cert short-names + fingerprint prefix + site count, linking to Template I

## 这些网站使用的技术 (Technologies used by these sites)
   — chip list with count badges, linking to Template K

## 关系图谱 (Relationship graph)
   — same interactive graph component as Template F §9.11, scoped to this IP's neighborhood

## 关系证据 (Relationship evidence) — full table
   | 关系 (Relationship) | 类型 (Type) | 置信度 (Confidence) | 证据 (Evidence) |
   — every single edge touching this IP, in tabular form, each row citing its literal evidence string
     (e.g., "ASN 400619 reported by ipwho.is", "A/AAAA record resolved to 154.204.176.66",
     "TLS leaf certificate A8:8C:00:8A:...", "NS record: wells.ns.cloudflare.com")
```

### 10.3 Notable data-provenance detail
- The evidence table explicitly names the third-party data source used per fact type (e.g., **ipwho.is** for ASN ownership) — this is a rare instance of the platform disclosing its upstream data vendor directly in the UI, useful for both trust-building and for understanding data reliability limits (ipwho.is-sourced geolocation/ASN data can lag real-world reassignments).
- Explicit epistemic caveat repeated here: *"共享 IP 不代表同一主体"* (a shared IP does not imply common ownership) — a responsible-disclosure pattern to prevent users from over-interpreting shared-hosting coincidences as meaningful relationships.

---

## 11. Template H — ASN Entity Page (`/asn/{number}`)

**Not directly fetched, but structurally inferable** from (a) the sitemap's 29 ASN instances, (b) every reference to ASN entities embedded inside Templates F and G (e.g., "2635 · Automattic, Inc", "400619 · Fastmos Co Limited · Hong Kong").

### 11.1 Expected structure (by analogy to Template G)
```
Breadcrumb: 首页 / 基础设施情报 / AS<number>
# AS<number> — <organization name>
Summary: N observed IPs · N observed websites
## IP ranges / observed IPs under this ASN — linking to Template G
## Organizations associated — linking to Template J
## Technologies observed across this ASN's hosted sites — linking to Template K
## Relationship graph (same component)
## Relationship evidence table (same component)
```
- The sitemap's ASN list is heavily weighted toward Chinese telecom/cloud operators (China Telecom AS4134/AS4812-style ranges, China Unicom, China Mobile) alongside major global players (Cloudflare AS13335, Google AS15169) — consistent with the zh_CN-first product positioning and the CN-heavy example domains seen in the wordpress.org relationship graph.
- `AS0` is present in the sitemap — this is the reserved/"unknown ASN" sentinel value in BGP/RIR data, meaning the platform correctly (or by necessity) creates an entity page even for **unresolvable/unassigned ASN lookups** — worth flagging as either a deliberate "unknown" bucket page or an unhandled-edge-case artifact (see §22).

---

## 12. Template I — TLS Certificate Entity Page (`/certificate/{sha256}`)

**Not directly fetched**, inferred from 11 sitemap instances and repeated in-context references from Templates F/G (e.g., `BF:57:AE:4E:...` fingerprint linking pattern).

### 12.1 Expected structure
```
Breadcrumb: 首页 / 基础设施情报 / <fingerprint prefix>
# TLS Certificate <SHA-256 fingerprint, colon-delimited display>
Issuer · Validity window · Days remaining
## Subject Alternative Names (SAN) — list of covered domains, each linking to Template F
## Domains observed presenting this certificate — linking to Template F
## Relationship graph + evidence table (same shared components)
```
- Certificate pages are the mechanism by which **shared-hosting/shared-cert clusters** are surfaced (e.g., a wildcard cert `*.wordpress.org` covering many subdomains, or a shared reseller cert covering unrelated domains like ipcece.com/ipcesu.com observed in §10).
- URL uses the **full SHA-256 hex digest** as the slug — long but collision-proof and directly copy-pasteable from any TLS inspection tool, a sensible identifier choice.

---

## 13. Template J — Organization Entity Page (`/organization/{slug}`)

**Not directly fetched**, inferred from 9 sitemap instances (e.g., `/organization/microsoft-corporation`, `/organization/china-mobile-communications-corporation`, `/organization/china-telecom`).

### 13.1 Expected structure
```
Breadcrumb: 首页 / 基础设施情报 / <Organization name>
# <Organization name>
Summary: N ASNs · N IPs · N domains associated
## ASNs owned/operated — linking to Template H
## Domains registered/hosted — linking to Template F
## Relationship graph + evidence table (same shared components)
```
- Slugs are **kebab-cased, human-readable transformations of WHOIS/ASN registrant names** (e.g. "6-collyer-quay" — a literal Singapore address used as an organization name in some ASN registrations, "beijing-kingsoft-cloud-internet-technology-co-ltd"). This confirms organization identity is derived from raw WHOIS/RDAP "org" fields rather than a curated/normalized company database — meaning the same real-world company could plausibly appear under multiple slightly different slugs if registration data is inconsistent (a data-quality risk, see §22).

---

## 14. Template K — Technology Entity Page (`/technology/{name}`)

Verified via `/technology/wordpress`.

### 14.1 Structure
```
Breadcrumb: 首页 / 技术栈情报 / <Technology name>
# <Technology name> 网站技术情报
"SiteIntel 观测到 N 个网站使用该技术 · <category>"

## 使用该技术的网站 (Sites using this technology)
   — chip list: domain + last-detected date, linking to Template F
   — NOTE: same domain can appear twice with different dates (observed: "sitevance.tw 2026-08-15"
     and "sitevance.tw 2026-08-14" as two separate chips) — each represents a distinct
     investigation snapshot rather than a deduplicated "site" entity, see §22.

## 相关技术（同站共现）(Related technologies — co-occurring on the same sites)
   — chip list with co-occurrence counts (e.g., "Nginx ×3", "Google Analytics ×3"),
     linking to sibling Template K pages — this is a market-basket/association-mining feature.

Footer disclaimer: "数据来源：SiteIntel 实体图谱的真实检测记录。每个「使用」关系都带证据
（响应头 / HTML 标记 / 资源引用），检测阈值变化可能影响统计。"
(Data source note: real detections from the entity graph; every "uses" relationship carries
evidence — response headers / HTML markers / resource references — and detection-threshold
changes may affect the statistics.)
```

### 14.2 Notable observations
- Explicit self-aware caveat that changing the internal fingerprinting/detection **threshold** can alter historical statistics — an unusually candid admission that the "N websites use this" counters are not immutable historical facts but a function of the current detection ruleset (i.e., re-running fingerprinting logic could retroactively change past counts).
- The page title in raw fetch output includes a **duplicated/malformed trailing title fragment** — the document's frontmatter `title:` was empty in the extracted metadata, but a second title string ("WordPress网站技术分析与使用情况 - SiteIntel") appears appended at the very end of the body content rather than in the head — suggests a **templating/SSR ordering bug** where the page-specific `<title>`/OG tag generation didn't populate the primary meta block correctly for this template category (see §22 — this is a concrete, directly observed defect, not merely inferred).
- 24 technology instances span a very heterogeneous set: frontend frameworks (React, Vue via Nuxt.js, Next.js), CMS (WordPress), servers (Nginx, OpenResty, Tengine, Microsoft IIS, Express), analytics (Google Analytics, Baidu Tongji), CDNs (Cloudflare, Alibaba Cloud CDN, Amazon CloudFront), and misc (jQuery, reCAPTCHA, Google Fonts, Google Tag Manager, iconfont-alibaba, Heroku, PHP, Django) — a broad, general-purpose fingerprint library rather than a narrow niche list.

---

## 15. Template L — Account / Utility Pages

These pages are either absent from the sitemap (deliberately non-indexed, gated, or dynamic/user-specific) or present but functionally distinct from content templates.

### 15.1 `/report`
- Linked from the header nav on every page ("报告"). Likely either (a) a generic "start a report" alias for the homepage analyze flow, or (b) a dashboard of the user's own past reports if logged in. Not directly fetched — recommend follow-up verification, since its exact behavior (redirect vs. distinct page) is currently unconfirmed.

### 15.2 `/bulk`
- "批量分析" (bulk analysis) — implies a multi-domain input (textarea/CSV upload) that queues multiple `/api/v1/analyze` calls. Given the API's 30-analyze-per-hour rate limit (§16), this feature is almost certainly **login/API-key gated** and likely a paid-tier differentiator.

### 15.3 `/login`
- Standard auth entry point; not fetched, but its presence alongside API-key management strongly implies a full account system (dashboard, API key generation/revocation, monitored-sites list, claimed-domains list).

### 15.4 `/claim/{domain}`
- Referenced from Template F's "这是你的网站吗？" button. Per-domain, **not sitemapped** (correctly excluded from public indexation since it's an action/verification flow, not content). Likely implements a DNS TXT-record or meta-tag ownership-verification challenge, standard for this class of product (mirrors Google Search Console's own verification UX — thematically consistent with the product literally trying to replicate/extend GSC-style tooling).

### 15.5 `/search` (hub, distinct from `/search/{engine}`)
- Header nav target "搜索情报" — likely a landing page listing the three engine connectors (Google/Bing/Baidu) as cards, functioning as the hub Template D pages link back to.

---

## 16. API Documentation & Backend Contract (`/docs/api`)

Verified via direct fetch — this page also doubles as a strong evidence source for backend architecture.

### 16.1 Authentication
```
Header required: X-API-Key: si_xxxx…
   (or alternatively) Authorization: Bearer <key>
Key format: si_<48 hex characters>
Server stores only a hash of the key (plaintext shown once at creation, in-dashboard)
```

### 16.2 Endpoints

| Method | Path | Purpose | Notes |
|---|---|---|---|
| `GET` | `/api/v1/report/{domain}` | Full JSON intelligence report (infra, tech, relationships, history, insights) | Optional `?lang=zh\|en` |
| `POST` | `/api/v1/analyze` | Trigger a new investigation (**asynchronous**) | Body: `{"domain": "example.com"}`; returns `investigationId` + SSE stream URL |
| `GET` | `/api/analyze/{id}/stream` | Server-Sent Events progress stream | **No API key required** on this specific endpoint; emits `event: progress` repeatedly, then `event: done` |

### 16.3 Rate limits
- Report endpoint: **100 requests/hour per key**
- Analyze endpoint: **30 requests/hour per key**
- Exceeding limits returns **HTTP 429** with a `retryAfterSec` field in the response body.

### 16.4 Architectural inferences from this page
- The **public analyze-progress SSE stream requiring no auth** (`/api/analyze/{id}/stream`) is a notable design choice: it means investigation IDs, if guessable or leaked, could let a third party watch (though not necessarily read the final report of) another user's in-flight analysis. This is flagged in §22 as a minor information-exposure consideration, pending confirmation of whether `investigationId` is a cryptographically random/CUID value (likely, given the `signalIds` format observed in §9.12) which would substantially mitigate the risk.
- The explicit two-step async pattern (`POST /analyze` → poll/stream → `GET /report`) confirms the frontend Analyze widget (§2.2) is **not** a single synchronous request; it must orchestrate a create-then-poll-then-fetch sequence, explaining the "real-time progress display" UX language used on Template C pages.
- No documented `DELETE`/webhook-management/monitoring-configuration endpoints are shown on this page, despite `/website-monitoring`'s copy explicitly promising Telegram/Webhook alerts — meaning **the public API docs are incomplete relative to the advertised product feature set** (see §21).

---

## 17. Cross-Cutting Frontend Components

These UI components/patterns recur across most or all templates and should be understood as a shared component library:

| Component | Appears on | Behavior |
|---|---|---|
| **Analyze input + example chips** | A, B, C, F (header sticky variant) | Controlled input, click-to-fill chips, async submit → SSE progress → redirect |
| **Breadcrumb** | All except homepage | 3-level: 首页 / section / current |
| **Entity relationship graph** | F, G, H*, I*, J* | Force-directed/draggable graph, scroll-zoom, click-to-navigate nodes, clickable type-filter legend |
| **Confidence badge** (%, 观察到/很可能/推断) | F, G, and all evidence tables | Consistent 3-tier epistemic labeling + numeric confidence, applied to every derived fact site-wide |
| **Evidence citation string** | F (evidence log), G (evidence table) | Literal source description (e.g., "NS record: X", "ASN N reported by ipwho.is") |
| **Chip/tag list with counts** | F (tech), G (tech/certs/IPs), K (sites/related tech) | `<Label> ×N` pattern, clickable, routes to sibling entity page |
| **"相关页面" internal-link block** | B, C, D, F(implicit) | 2–3 contextual links, SEO-oriented |
| **Health-check row** (pass/optimizable/not-evaluated + recommendation) | F only (Section 07) | Expandable per-dimension, unique to Template F |
| **History/diff entry** (severity + timestamp + JSON before→after) | F only (Section 10) | Reverse-chronological feed |
| **Empty-state pattern** ("未检测到…", "暂未发现…", "尚未连接…") | F (CDN, related sites, search intel), and presumably G/H/I/J when data is sparse | Consistently phrased negative-result messaging rather than blank/broken sections — a deliberate, well-executed UX choice |

\* Inferred by structural analogy, not directly fetched.

---

## 18. Data Model & Backend Business Logic (Inferred)

Based on the terminology, ID formats, and data shapes surfaced across all fetched pages, the backend data model appears to be approximately:

```
Investigation
 ├─ id (CUID-style, e.g. "cmsty5jwf0031s51ahsp1qot5")
 ├─ domain
 ├─ status (COMPLETED, presumably also PENDING/RUNNING/FAILED)
 ├─ startedAt / completedAt
 └─ CollectorRun[7] (dns, rdap, ip, ssl, http, metadata, technology)
      ├─ status (成功/success, presumably also failed/timeout)
      ├─ payloadSizeBytes
      └─ rawPayload (JSON, persisted verbatim — "evidence")

Entity (polymorphic: Domain | IP | ASN | Organization | Certificate | Technology | TrackingId | MailService | Nameserver)
 ├─ id / slug
 ├─ type
 ├─ firstSeen / lastSeen
 └─ canonical attributes (per type)

Relationship (edge)
 ├─ from Entity, to Entity
 ├─ type (resolves_to | belongs_to | uses | shares | associated_with | ...)
 ├─ epistemicStatus (观察到 / 很可能 / 推断  →  observed / likely / inferred)
 ├─ confidence (0–100)
 └─ evidence (string, source-cited)

FactSnapshot
 ├─ investigationId
 ├─ hash (content fingerprint)
 └─ facts { resolved_ips, dns_records, infrastructure, ... }

Event (diff output)
 ├─ type (provider_changed | dns_changed | ip_changed | metadata_changed | infrastructure_migration | ...)
 ├─ severity (高/中/低)
 ├─ detectedAt
 ├─ before / after (JSON)
 └─ signalIds[] (links back to contributing snapshots/facts)

Insight (rule-engine output)
 ├─ ruleType (metadata_change | infrastructure_migration_event | provider_change |
 │             infrastructure_migration | frequent_changes | ip_change | ...)
 ├─ confidence (0–100) + qualitative band (可能/较强/高)
 ├─ severity (中/高)
 ├─ firstDetectedAt
 └─ evidenceRefs[] (→ Event[] and/or FactSnapshot[])

HealthAudit (per investigation, per Template F render)
 ├─ dimension (Technical | Security | Infrastructure | SEO | Content | Performance)
 ├─ score (0–100 or "未评估")
 └─ checks[] { label, status(通过/可优化/未评估), detectedValue, recommendation }

ActionItem (derived from failed HealthAudit checks)
 ├─ priority (P0 | P1 | ...)
 ├─ impact (low/medium/high) × urgency ÷ effort(S/M/L)
 └─ timeBucket (0-30 | 31-60 | 61-90 days)

ApiKey
 ├─ prefix "si_" + 48 hex chars
 ├─ hashStored (plaintext shown once)
 └─ rateLimit { reportPerHour: 100, analyzePerHour: 30 }
```

### 18.1 Rule engine characteristics
- Confidence scores are **not uniformly derived** — some are fixed per fact-type (e.g., ASN ownership always shown at 100% since it's a direct API lookup), while insight-level confidence varies per instance (62%–92% observed) — implying a weighted heuristic/scoring function rather than a single global constant, likely incorporating factors like number of corroborating signals, recency, and historical volatility of the specific domain.
- The coexistence of both `infrastructure_migration_event` (92% confidence) and `infrastructure_migration` (84% confidence) as **two separately-firing insight rules covering overlapping evidence** on the same domain strongly suggests either (a) two independently authored rules that were never deduplicated, or (b) intentional multi-rule corroboration design that is not yet distinctly worded enough for end users to tell apart. Flagged in §22.

---

## 19. Data Flow / Pipeline Architecture

Synthesizing `/how-it-works` prose (§5.2) with the concrete artifacts observed in Template F:

```
 ┌─────────────┐
 │ User submits │  domain string (client-side validated)
 │   a domain   │
 └──────┬──────┘
        │  POST /api/v1/analyze  { "domain": "..." }
        ▼
 ┌─────────────────────┐
 │  Investigation row    │  status=PENDING, id issued (CUID)
 │  created               │
 └──────┬───────────────┘
        │  SSE: GET /api/analyze/{id}/stream   (unauthenticated)
        ▼
 ┌───────────────────────────────────────────────────────────┐
 │  STAGE 1 — 采集 (Collection): 7 parallel collectors run      │
 │   DNS · IP/ASN enrichment · TLS cert · HTTP response ·       │
 │   Tech fingerprint · Page metadata · WHOIS/RDAP               │
 │   → each raw response persisted verbatim (evidence root)      │
 └──────┬────────────────────────────────────────────────────┘
        │ event: progress ×N (per collector completion)
        ▼
 ┌───────────────────────────────────────────────────────────┐
 │  STAGE 2 — 标准化 (Normalize): raw payloads → canonical facts │
 └──────┬────────────────────────────────────────────────────┘
        ▼
 ┌───────────────────────────────────────────────────────────┐
 │  STAGE 3 — 分析 (Analyze) + STAGE 4 — 关联 (Correlate):        │
 │   entities extracted & deduplicated/merged against existing    │
 │   entity graph; typed relationships created w/ confidence      │
 └──────┬────────────────────────────────────────────────────┘
        ▼
 ┌───────────────────────────────────────────────────────────┐
 │  STAGE 5 — 对比 (Compare): new snapshot vs. most recent        │
 │   historical snapshot for this domain (hash-based diff)        │
 └──────┬────────────────────────────────────────────────────┘
        ▼
 ┌───────────────────────────────────────────────────────────┐
 │  STAGE 6 — 检测 (Detect): field-level diffs → discrete Events  │
 │   (before/after, severity, timestamp)                          │
 └──────┬────────────────────────────────────────────────────┘
        ▼
 ┌───────────────────────────────────────────────────────────┐
 │  STAGE 7 — 洞察 (Insight): rule engine reads Events (single    │
 │   signal or clustered pattern) → Insight rows w/ confidence     │
 └──────┬────────────────────────────────────────────────────┘
        │ event: done  { investigationId, status: COMPLETED }
        ▼
 ┌─────────────────────┐        ┌───────────────────────────┐
 │ Client redirects to  │ ───▶  │  GET /api/v1/report/{domain}│
 │ /website/{domain}     │        │  (server-rendered on load)  │
 └─────────────────────┘        └───────────────────────────┘
```

### 19.1 Monitoring loop (separate, subscription-based cycle)
```
 Domain added to monitoring (login required, presumably via /claim/{domain} or a dashboard action)
        │
        ▼
 Daily scheduled re-run of the full pipeline above (cron/queue job)
        │
        ▼
 New Events/Insights generated as usual
        │
        ▼
 If severity/threshold met → push alert via Telegram bot and/or configured Webhook URL
```

---

## 20. SEO & Technical Implementation Notes

- **Framework:** Next.js — inferred from (a) `technology/next-js` appearing as a fingerprint the platform itself could plausibly detect on other sites, (b) the OG-image dynamic-route pattern `/opengraph-image?<hash>` which is idiomatic Next.js App Router (`opengraph-image.tsx` convention), and (c) SSR/SSG-style server-rendered metadata blocks consistently present across every fetched page.
- **Differentiated `robots` directives by template:** Template F (`/website/*`) = `index, follow`; Template G (`/ip/*`) = `noindex, follow` — a deliberate crawl-budget/quality strategy: pass link equity through IP pages without competing for index space, while making the "real" content (domain reports) the indexable unit. This same `noindex, follow` pattern likely also applies to H/I/J (ASN/certificate/organization), though unconfirmed without direct fetch.
- **`sitemap.xml` freshness:** every `<lastmod>` timestamp for the "core" template pages (A–E) is **identical to the second** (`2026-08-15T10:14:53.253Z`+ms variants) — indicating the sitemap is **regenerated wholesale on every build/deploy** rather than tracking true per-page content-change history; only Template F pages show genuinely distinct, content-driven `lastmod` values (matching their real "last investigation" timestamps).
- **Duplicate homepage entries:** both `https://siteintel.cc` (no trailing slash) and `https://siteintel.cc/` (trailing slash) are separately listed in the sitemap with distinct `<changefreq>` (daily vs weekly) and near-identical `lastmod` — this is redundant and could cause minor duplicate-content ambiguity for search engines (see §22).
- **`www` vs. apex domain:** the sitemap consistently uses the apex domain (`siteintel.cc`), while the user-facing/browsed URL in this analysis was `www.siteintel.cc` — confirms a `www` → apex (or vice versa) redirect is in place; canonical tags observed in fetched pages point to the apex, non-www form, which is correct practice as long as the redirect is consistently enforced site-wide.

---

## 21. Unimplemented / Placeholder Features

| Feature | Evidence | Status |
|---|---|---|
| Google Search Console integration (`/search/google`) | Explicit "Phase 3" label; CTA button routes to homepage, not an OAuth flow | **Not implemented** |
| Bing Webmaster Tools integration (`/search/bing`) | Same template as above (inferred) | **Not implemented** |
| Baidu 站长平台 integration (`/search/baidu`) | Same template as above (inferred) | **Not implemented** |
| Search-event ↔ infrastructure-event correlation | Advertised in `/search/google` copy ("搜索事件关联") and echoed in Template F's empty-state search-intelligence section | **Not implemented** — entirely dependent on the search connectors above |
| Performance/Core Web Vitals health scoring | Explicitly marked "待评估（暂无真实数据）" with the "§26 不凭空评分" principle cited | **Deliberately deferred**, not a bug — but incomplete relative to the 6-dimension scoring UI already built |
| Webhook/Telegram monitoring alerts | Advertised on `/website-monitoring` and in `/how-it-works` step 10 | **Not documented in the public API** (`/docs/api` has no monitoring/webhook-config endpoints) — either an internal-only/dashboard-only feature, or aspirational copy ahead of implementation |
| "AI 解读行动计划" (AI-interpreted action plan) | CTA present in Template F Section 08 | Presence of a button doesn't confirm a working backend; **unverified**, recommend a live click-through test |
| `/report` nav item behavior | Never directly fetched | **Unverified** — could be a working dashboard or a dead/placeholder link |
| Bulk analysis (`/bulk`) | Nav link exists; not fetched | **Unverified**, likely functional given it's core to the stated rate-limit tiers, but UI/UX unconfirmed |

---

## 22. Potential Defects, Risks & UX Gaps

### 22.1 Data-quality / rule-engine issues
1. **Likely false-positive "provider change" events from round-robin DNS.** wordpress.org logged 36 events in a single 30-day window, many occurring **minutes apart**, cycling through the same 4 known nameservers and 2 known mail servers in different orders. This pattern is classic round-robin/multi-value DNS answer rotation, not an actual infrastructure migration — yet it's being surfaced as high-severity (高) "服务商变更" events and feeding a 92%-confidence "基础设施迁移" insight. **Recommendation:** the diff engine should treat *reordering within a known, stable set of values* differently from *introduction of a genuinely new value*.
2. **Two near-duplicate insight types on the same evidence.** `infrastructure_migration_event` (92%) and `infrastructure_migration` (84%) both fired for wordpress.org from what appears to be the same underlying signal set — confusing for end users trying to understand "how many real things happened," and a signal of insufficiently deduplicated rule definitions.
3. **Technology-page site counts may double-count investigations, not sites.** `/technology/wordpress` lists "sitevance.tw" twice with two different dates as if they were two separate "sites using this technology," while the page header claims "3 个网站使用该技术" (3 *websites*) — conflating investigation-count with distinct-site-count is a minor but real metric-accuracy issue.
4. **Organization-entity fragmentation risk.** Organization slugs appear to be derived directly from raw WHOIS/RDAP org-name strings (§13.1) rather than a normalized company registry, meaning the same real-world entity could be split across multiple slugs if registration records are inconsistently formatted (e.g., "Google LLC" vs "Google Inc." vs "Google" appearing as separate organization pages over time) — this would silently degrade the relationship graph's usefulness for aggregate/competitive analysis.
5. **`AS0` present as an entity page.** AS0 is a reserved/invalid ASN sentinel in real-world routing data; giving it a full entity page could either be an intentional "unknown ASN" bucket (reasonable) or an unhandled parsing edge case where a lookup failure defaulted to 0 (a bug) — worth internal verification.

### 22.2 SEO / metadata issues
6. **Duplicate homepage sitemap entries** (`siteintel.cc` and `siteintel.cc/`) with conflicting `changefreq` values — should be consolidated to a single canonical entry.
7. **Copy-pasted Open Graph metadata on entity pages.** The IP page fetched (`/ip/154.204.176.66`) carries the exact same `og:title`/`og:description`/`og:image`/`og:url` as the global homepage rather than IP-specific values — this means social shares of any `/ip/*` page will display generic homepage branding instead of the specific IP's context, undermining shareability of that content type. (The dedicated `<title>` tag *is* correctly IP-specific — only the OG block is stale/generic — suggesting a template that only wires up `<title>`/canonical per-instance but forgot to do the same for the OG block on this template.)
8. **Sitewide `lastmod` timestamps for static pages are build-time, not content-time.** All Template A–E pages share the exact same `lastmod` down to the millisecond, which doesn't reflect genuine content-change history and could reduce a search engine's trust in the sitemap's freshness signals over time if content genuinely stops changing between deploys but timestamps keep "refreshing."
9. **`/technology/wordpress` title/meta rendering anomaly.** As noted in §14.2, the page's frontmatter `title:` field renders empty while a technology-specific title string appears appended at the very end of the visible body rather than in the `<head>` — this looks like a genuine SSR metadata-population bug specific to the Technology template, not present on Website (F) or IP (G) templates, both of which populated `<title>`/meta correctly.

### 22.3 Product / UX gaps
10. **No visible legal/compliance links** (Privacy Policy, Terms of Service, Cookie notice) in the footer across any fetched page — notable given the product actively fingerprints and republishes third-party WHOIS/DNS/IP data about arbitrary domains (including presumably domains the "owner" never asked to be profiled), which typically warrants a clear data-sourcing/removal-request policy.
11. **No documented "remove my domain" / opt-out flow**, despite the product publishing infrastructure history for any domain a user chooses to analyze — the only related feature found is the *positive* "claim your site" flow (`/claim/{domain}`), not a *negative*/opt-out or takedown-request mechanism.
12. **Unauthenticated SSE progress endpoint** (`/api/analyze/{id}/stream`, §16.4) — low-severity but worth noting: if `investigationId` values are sequential or otherwise guessable (unconfirmed — they resemble CUIDs elsewhere in the product, which would mitigate this), a third party could passively observe another user's in-progress analysis metadata.
13. **"重新分析" (re-analyze) button has no visible rate-limit/cooldown indicator** in the UI — combined with the API's 30/hour analyze limit, a user could plausibly hit a 429 error via the UI button with no visible affordance explaining why, unless client-side handling for this exists (unconfirmed).
14. **Relationship-graph relevance/noise at scale.** The wordpress.org graph surfaced 40+ loosely-related third-party domains (via a shared nameserver-host classification and/or shared GA/GTM usage) under a single generic "使用 ×68" relationship bucket. As the platform's corpus grows past its current 184 domains, very common technologies (Google Analytics, Google Fonts, Cloudflare) risk becoming graph "supernodes" that connect nearly everything to everything, diluting the signal value of the relationship feature unless the platform introduces relationship-strength weighting or hides ubiquitous-technology edges by default.
15. **Search-intelligence feature is prominently linked from Templates A, D, and F but delivers no functionality anywhere in the product today** — from a first-time-user perspective, this creates several dead-end/disappointment touchpoints (nav item → hub page → 3 engine pages → all "Phase 3" placeholders) rather than being clearly marked "Coming Soon" at the point of nav entry.

---

## 23. Sitemap Coverage Map

Breakdown of all 110 `sitemap.xml` entries by category (see §3 for template mapping):

| Category | Count | Priority range | Example |
|---|---|---|---|
| Homepage (dup. entries) | 2 | 1.0 | `/`, (apex) |
| Core product/feature pages | 7 | 0.9 | `/how-it-works`, `/website-monitoring` |
| Tool pages | 5 | 0.85 | `/tools/ssl` |
| Search-engine connector stubs | 3 | 0.85 | `/search/baidu` |
| Guides | 3 | 0.8 | `/guides/website-migration` |
| Case studies | 2 | 0.75 | `/cases/search-traffic-drop` |
| Technology entity pages | 24 | 0.65 | `/technology/react` |
| IP entity pages | 11 | 0.55 | `/ip/127.0.0.1` *(note: includes localhost — likely a test/demo artifact, see below)* |
| ASN entity pages | 29 | 0.55 | `/asn/AS13335` |
| TLS certificate entity pages | 11 | 0.5 | `/certificate/a88c008a...` |
| Organization entity pages | 9 | 0.5 | `/organization/china-telecom` |
| Website report pages | 14 | 0.6 | `/website/baidu.com` |
| **Total** | **~110** | | |

**Notable anomaly:** `/ip/127.0.0.1` is indexed in the live sitemap. 127.0.0.1 is the IPv4 loopback address and can never legitimately be "the resolved IP" of any real public domain — this is almost certainly either **test/seed data left in production** or the result of a collector bug (e.g., a DNS resolution failure defaulting to loopback, or a local-testing investigation that was never purged before the sitemap was generated). Recommend flagging for internal cleanup, as it is currently publicly indexable.

---

## 24. Summary Assessment

**Strengths**
- Consistent, well-designed **evidence + confidence** epistemic framework applied uniformly across the entire product — a genuinely differentiating trust mechanic versus typical "black box" OSINT tools.
- Clean template reuse (one relationship-graph component, one confidence-badge component, one empty-state pattern) powering 6+ distinct entity types — efficient architecture.
- Thoughtful negative-result UX (explicit "not detected" / "not connected" states rather than blank sections).
- Transparent raw-evidence/collector-log exposure (§9.14) is unusually developer-friendly for a consumer-facing tool.
- Deliberate `noindex` strategy on low-value entity pages (IP) while indexing high-value ones (domain reports) shows SEO maturity.

**Weaknesses**
- Three fully-linked, prominently-placed features (Google/Bing/Baidu search intelligence) are non-functional placeholders, creating multiple dead-end user journeys.
- The change-detection/insight rule engine shows evidence of **false-positive-prone logic** around DNS round-robin and **duplicate/overlapping insight types**, which could erode user trust in the "high confidence" labels if not addressed.
- Several SEO/metadata inconsistencies (duplicate homepage sitemap entries, generic OG tags on IP pages, a metadata-rendering bug on the Technology template) suggest the templating system is not yet fully uniform across all six entity types.
- No visible privacy/data-removal policy despite the product's inherently privacy-adjacent nature (profiling arbitrary third-party domains' infrastructure history).
- At current corpus scale (184 domains), the relationship-graph feature already shows early signs of "supernode" noise from ubiquitous technologies, which will likely worsen without additional relevance-filtering as the corpus grows.

---

*End of report. This analysis is based on programmatic fetches of a representative sample of live pages across every template category (homepage, static/marketing, tools, search-connector, website-report, IP-entity, technology-entity, and API docs) plus full enumeration of the 110-URL sitemap. Templates H (ASN), I (Certificate), and J (Organization) were not directly fetched and are structurally inferred by cross-referencing their consistent in-context appearance inside Templates F and G; a follow-up direct fetch of one instance of each is recommended to convert those sections from "inferred" to "confirmed."*
