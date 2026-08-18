# duoniu.cc — Website Repositioning & Product Planning

**Prepared:** August 2026
**Scope:** Repositioning, information architecture, layout, functional card-module design, competitive analysis, UX recommendations, and implementation roadmap.

> **Methodology note (read first):** duoniu.cc blocks automated crawling (`robots.txt` disallow), so this document is built from (1) the site's disclosed sitemap URL structure, and (2) research into direct competitors in the same category. Where a recommendation is inferred rather than confirmed from the live page, it is labeled **[Inferred]**. Treat layout/copy suggestions as a design proposal to validate against the real site, not a description of what currently exists.

---

## 1. What duoniu.cc Is, Based on the Sitemap

The disclosed URL list maps cleanly onto a single product category:

| Path | Likely Function |
|---|---|
| `/` | Homepage — IP summary / entry point |
| `/whois` | WHOIS domain & IP registrant lookup |
| `/asn` (+ `/asn/174`) | ASN (Autonomous System) lookup, with individual ASN detail pages |
| `/bgp` | BGP routing table / announcement lookup |
| `/cidr` | CIDR block / subnet calculator or lookup |
| `/dns` | DNS record lookup (A/AAAA/MX/TXT/NS, resolver test) |
| `/route` | Route/traceroute-adjacent network path tool |
| `/trace` | Traceroute tool |
| `/ping` | Ping / latency test |
| `/ip-leak` | IP/DNS/WebRTC leak test (VPN/proxy verification) |
| `/purity` | **IP purity/risk scoring** — the likely flagship/monetizable feature |
| `/env` | Browser/network "environment" consistency check (timezone, language, WebRTC, fingerprint) |
| `/tools` | Tool index/directory hub |
| `/history` | Query history (user's past lookups) |
| `/about` | About/company page |
| `/faq` | FAQ |

**Category verdict:** This is a **network diagnostics / IP intelligence toolkit**, positioned in the same space as ping0.cc, IP透视 (iptoushi.com), ip86.net, IPPure, IPIPseek, and Net.Coffee. The presence of `/purity`, `/ip-leak`, and `/env` (not just basic whois/ping) signals the site is oriented toward **proxy/VPN quality verification** — i.e., its core user is not a general network engineer but someone who needs to verify an IP is "clean" before using it.

---

## 2. Target User & Use Cases [Inferred from category]

| Segment | Core Need |
|---|---|
| Cross-border e-commerce sellers (Amazon, TikTok Shop, Shopee) | Verify proxy/residential IP won't trigger account association or bans before logging into a platform account |
| Social media / ads operators (TikTok, Meta, Google Ads matrix accounts) | Same as above — avoid IP-based account linking penalties |
| VPN / proxy resellers and power users | Confirm IP purity before selling or before switching nodes |
| AI tool users (ChatGPT, Claude, Gemini access via proxy) | Confirm IP isn't flagged as datacenter/blocked region |
| Network/sysadmins, streaming unlock testers | BGP/ASN/route diagnostics, streaming region unlock checks |

The unifying job-to-be-done: **"Before I use this IP/network exit for something important, tell me if it's safe — and if not, tell me exactly why and what to do about it."**

---

## 3. Competitive Landscape

| Competitor | Core Angle | Differentiators | Weaknesses |
|---|---|---|---|
| **ping0.cc** | Simple, fast IP risk lookup | Extremely lightweight, long-trusted brand in China proxy community, has a lightweight server-side agent tool | Dated visual design, limited explanation of *why* a score is given |
| **iptoushi.com (IP透视)** | Purity + camouflage scoring (S+ to C grade) | Clear grading system, environment-consistency checks, "copy result → paste into ChatGPT/Claude for AI diagnosis" gimmick, before/after node-switch comparison | Newer, smaller brand trust |
| **ip86.net** | IP quality + platform-fit advice | Explicit "fits ChatGPT / TikTok / Amazon" recommendations, blacklist + shared-exit detection | Less real-time network diagnostics (ping/trace) |
| **ippure.com (IPPure)** | Positions explicitly for cross-border e-commerce, streaming, LLMs, and scrapers | Clear multi-audience positioning statement | — |
| **IPIPseek** | Aggregator — shows the *same* IP's results across multiple third-party checkers side by side | Cross-validation is a strong trust mechanic | Doesn't do its own proprietary scoring |
| **Net.Coffee** | IP query + proxy/routing-rule verification, 35+ site exit checks (Cloudflare, Discord, ChatGPT, etc.) | Strong "which sites can I actually reach" angle | More power-user/technical framing |
| **ipin.io** | API-first, developer-facing IP intelligence | Strong docs, badge/API output | Less consumer-facing |

**Gap analysis — what's underserved in this market:**
1. **No dominant player owns "explain it to me in plain language."** Most tools output raw fields (ASN, usage_type, is_vpn). iptoushi's "copy for AI diagnosis" trick hints at demand for *plain-language interpretation* — duoniu.cc could own this outright with a built-in explainer rather than outsourcing to ChatGPT.
2. **Full-stack diagnostics + purity score in one flow is rare.** Most sites are either "network engineer toolbox" (whois/bgp/asn/trace) *or* "purity score for e-commerce sellers" — few do both well in one coherent journey. duoniu.cc's sitemap suggests it already has both; the opportunity is presenting them as **one guided workflow**, not a disconnected tool list.
3. **No one owns "before/after" node-switching comparison as a first-class feature** (only iptoushi does this loosely). Given the audience frequently rotates proxies, a persistent comparison/history view is high-value — and duoniu.cc already has `/history`, which is a strong foundation.
4. **Trust/methodology transparency is weak across the category.** Sites give scores without showing their reasoning. Publishing a clear, auditable scoring methodology is a credibility differentiator.

---

## 4. Repositioning Recommendation

### Current likely positioning [Inferred]
A general-purpose "network tools" site — a grab-bag of whois/ping/trace/dns utilities with a purity checker bolted on.

### Recommended repositioning
> **"Know your IP before it costs you an account."**
> duoniu.cc is a **pre-flight check for anyone about to use a proxy, VPN, or new IP for something that matters** — cross-border selling, ad accounts, streaming, or AI tools — combining a plain-language purity/risk verdict with the full technical diagnostics (WHOIS/ASN/BGP/route/DNS/trace) to back it up.

**Positioning pillars:**
1. **Verdict-first, evidence-second.** Lead every page with a clear pass/fail-style judgment (like a credit score), then let technical users drill into raw data.
2. **One workflow, not eleven disconnected tools.** Reframe the site from "a toolbox" to "a diagnostic flow": Check → Understand → Fix → Re-check → Track history.
3. **Explain, don't just report.** Every risk flag should carry a one-line "what this means" and "what to do" — this is the single biggest differentiation opportunity in the category.
4. **Built for repeat use.** Cross-border operators check IPs constantly (new proxy batches, node rotation). `/history` and comparison views should be core to retention, not an afterthought.

---

## 5. Site-Wide Layout & Information Architecture

### 5.1 Global Navigation (recommended)

```
[Logo: duoniu.cc]   IP 检测   工具箱 ▾   历史记录   关于/FAQ        [当前IP卡片] [语言] [登录/收藏]
                     (Purity)  (whois/asn/bgp/cidr/dns/route/trace/ping/env/ip-leak)
```

- Primary nav collapses the 11 diagnostic pages into **2 top-level items**: the flagship **Purity Check** and a **Tools** dropdown/mega-menu for the rest — this fixes the "disconnected tool list" problem noted above.
- Persistent mini IP-status chip in the header (current IP + quick grade) so users always see their status while navigating, similar to iptoushi.com's pattern.

### 5.2 Homepage Layout (top to bottom)

1. **Hero / Instant Verdict Card** — auto-detects visitor's IP on load, shows a big grade (S+/S/A/B/C) with a one-sentence verdict, no click required. This is the single highest-value UX pattern to borrow from iptoushi.com.
2. **Score Breakdown Row** — 4–6 sub-score cards (Proxy/VPN risk, Datacenter vs. Residential, Blacklist status, DNS/WebRTC leak, Environment consistency, ASN reputation).
3. **"What should I do?" action panel** — contextual recommendations based on the score (e.g., "Datacenter IP detected — not recommended for TikTok/Amazon logins").
4. **Deep-dive Tools grid** — card grid linking to whois / asn / bgp / cidr / dns / route / trace / ping / env / ip-leak.
5. **Platform-fit strip** — icons for ChatGPT / TikTok / Amazon / Netflix / Discord with a green/yellow/red compatibility indicator for the current IP (borrowed from ip86.net's positioning, and Net.Coffee's multi-site check).
6. **History/Compare teaser** — "Track your last 10 checks" card driving toward account creation or local storage-based history.
7. **Trust/methodology footer block** — brief, transparent explanation of how the score is calculated (differentiator vs. competitors).
8. **FAQ + About + footer nav**

---

## 6. Homepage Functional Card Modules (detailed)

| # | Card Module | Content | Purpose |
|---|---|---|---|
| 1 | **IP Verdict Card** | Current IP, country/city flag, ISP, grade badge (S+–C), one-line verdict | Immediate value, zero-click |
| 2 | **Proxy/VPN Risk Card** | is_vpn / is_tor / is_datacenter flags, confidence % | Core purity signal |
| 3 | **Blacklist Card** | Number of blacklists hit (Spamhaus-style), list names | Trust/authority signal |
| 4 | **Leak Test Card** | DNS leak / WebRTC leak pass-fail | Links to `/ip-leak` |
| 5 | **Environment Consistency Card** | Timezone vs. IP country match, language, browser fingerprint flags | Links to `/env` |
| 6 | **ASN / Network Card** | ASN number, org name, network type (residential/mobile/hosting) | Links to `/asn` |
| 7 | **Platform Compatibility Card** | Grid of platform logos with green/yellow/red status | New card type — key differentiator |
| 8 | **Tools Directory Card (grid)** | Icon + 1-line description for each of the 8 remaining tools | Discoverability of full toolbox |
| 9 | **History/Compare Card** | Mini sparkline of past scores for saved IPs/nodes | Retention driver |
| 10 | **CTA / Share Card** | "Copy shareable report" / "Export for AI diagnosis" button | Growth loop (mirrors iptoushi's AI-copy trick, done natively instead of outsourced) |

---

## 7. Inner-Page Function-Card Modules

### 7.1 `/purity` (flagship page)
- Full-width verdict header (grade + score out of 100)
- Expandable sub-score cards with **"why this matters" microcopy** on each
- "Compare with previous check" toggle (pulls from `/history`)
- "Re-check after switching node" button
- Export/share module

### 7.2 `/whois`
- Search bar (domain or IP)
- Registrant/registrar summary card
- Raw WHOIS output (collapsible, monospace)
- Related ASN link-out card

### 7.3 `/asn` and `/asn/{id}`
- ASN summary card (org, country, allocation date)
- Announced prefixes table
- "IPs in this ASN reputation" mini card
- Peer/upstream visualization **[Inferred — nice-to-have]**

### 7.4 `/bgp`
- Prefix/AS path lookup input
- Route table results (table view)
- "Is this route currently visible globally?" status card

### 7.5 `/cidr`
- CIDR calculator input (subnet mask ↔ range converter)
- Result card: network address, broadcast, usable range, host count

### 7.6 `/dns`
- Record-type selector (A/AAAA/MX/TXT/NS/CNAME)
- Multi-resolver comparison table (e.g., resolved via Google/Cloudflare/local ISP) — strong trust-building feature
- DNS propagation status card

### 7.7 `/route` and `/trace`
- Target input field
- Hop-by-hop table with latency per hop
- Visual path map **[Inferred — high-value addition, common in category]**
- "Export as text" button (sysadmin-friendly)

### 7.8 `/ping`
- Target input, live latency chart
- Packet loss %, jitter stat cards
- Multi-location ping option (if infra supports) **[Inferred]**

### 7.9 `/ip-leak`
- Big pass/fail leak indicator
- DNS server list detected
- WebRTC local/public IP exposure card
- "How to fix" guidance card (VPN config tips)

### 7.10 `/env`
- Timezone vs. IP-country match card
- Browser language vs. IP-country match card
- Fingerprint/consistency score
- "Location permission compare" (optional geo-distance check, opt-in) — mirrors iptoushi's pattern

### 7.11 `/history`
- Table/list of past checks with timestamp, IP, grade
- Trend sparkline
- Compare-two-entries feature

### 7.12 `/tools`
- Full card grid of all diagnostic tools (index/directory page)

### 7.13 `/about`, `/faq`
- Standard content pages; FAQ should include a "how is the score calculated" section reinforcing the trust/transparency differentiator

---

## 8. UX Recommendations

1. **Reduce time-to-first-insight to zero.** Auto-run the purity check on page load — never make the user click "check" for their own IP.
2. **Unify visual language for "grade."** Use one consistent badge system (color + letter grade) across purity, platform-fit, and leak results so users pattern-match instantly.
3. **Always answer "so what do I do now."** Every risk flag needs a plain-language explanation and a suggested action — this is the core differentiation vs. competitors who just dump raw data.
4. **Make the toolbox feel like one product, not 11 microsites.** Shared header, shared IP-context chip, consistent card design system across all inner pages.
5. **Design for repeat, high-frequency use.** Cross-border operators check dozens of IPs per week — prioritize speed, keyboard-friendly input, and low-friction history/comparison.
6. **Mobile-first for the verdict card.** Many users check IPs on mobile after switching a VPN app — the hero verdict card must be fully legible and functional on small screens.
7. **Build the "AI-ready export" natively.** Since users already copy results into ChatGPT/Claude/DeepSeek for interpretation (per competitor iptoushi.com), offer a "Copy structured report" button — turns a workaround into an owned feature and a differentiator.
8. **Progressive disclosure for technical tools.** whois/bgp/cidr/dns should default to a simple summary card, with "view raw data" as an expandable option — serves both non-technical sellers and technical sysadmins from the same page.
9. **Trust and transparency as UI, not just copy.** A visible "Methodology" link/tooltip next to every score builds credibility in a category where users are inherently skeptical of unverifiable scores.

---

## 9. Visual/Brand Direction [Inferred, proposal]

- **Tone:** clinical-but-friendly — more "diagnostic dashboard" than "hacker toolbox." Avoid overly technical/dark "terminal" aesthetics that alienate non-technical e-commerce sellers, while keeping enough data density to satisfy power users.
- **Color system:** traffic-light semantics (green/amber/red) for all risk indicators, consistent across every card and page.
- **Typography:** clear numeral-forward type for scores/latency figures; monospace only for raw technical output (WHOIS, traceroute).
- **Iconography:** a distinct icon per tool (whois, dns, trace, etc.) used consistently in nav, homepage grid, and page headers for recognizability.

---

## 10. SEO & Content Strategy Notes

- Each tool page (`/whois`, `/dns`, `/trace`, etc.) is a natural long-tail SEO target (e.g., "在线whois查询", "IP纯净度检测"). Ensure each has unique, indexable copy beyond just the tool widget — a short explainer of what the tool does and when to use it.
- `/purity` is the flagship page and primary conversion/SEO target — competitors like iptoushi.com and ippure.com are actively targeting this exact keyword cluster ("IP纯净度检测"); a strong differentiated meta description and on-page methodology content will help compete.
- `/faq` should target long-tail informational queries ("为什么IP会被判定为机房IP", "如何提高IP纯净度") to capture top-of-funnel traffic.

---

## 11. Growth & Monetization Angles [Inferred]

1. **Freemium API** for developers/agencies who need to batch-check IP purity (mirrors ipin.io's positioning) — a natural expansion given the toolset already exists.
2. **Browser extension** for one-click purity checks without visiting the site — strong retention/distribution play matching the "check before every login" use case.
3. **Proxy/VPN affiliate partnerships**: when a score is poor, recommend (as affiliate links) reputable residential-proxy providers as the "fix."
4. **Team/agency accounts**: cross-border agencies managing many accounts need multi-IP history and shared dashboards — a plausible paid tier.

---

## 12. Implementation Priority Roadmap

### Phase 1 — Foundation (highest impact, lowest effort)
- [ ] Auto-run purity check on homepage load (zero-click verdict)
- [ ] Unify navigation: collapse 11 pages into "Purity Check" + "Tools" menu
- [ ] Standardize the grade/badge visual system across all pages
- [ ] Add plain-language "what this means / what to do" microcopy to every risk flag

### Phase 2 — Differentiation
- [ ] Build Platform Compatibility card (ChatGPT/TikTok/Amazon/Netflix/Discord status)
- [ ] Native "Copy structured report for AI diagnosis" export feature
- [ ] Publish transparent scoring methodology page/tooltip
- [ ] Strengthen `/history` into a comparison tool (before/after node switch)

### Phase 3 — Depth & Retention
- [ ] Multi-resolver DNS comparison on `/dns`
- [ ] Hop-by-hop visual path map on `/trace` and `/route`
- [ ] Mobile-optimized verdict card
- [ ] SEO content buildout per tool page + FAQ long-tail targeting

### Phase 4 — Growth
- [ ] Public API / developer tier
- [ ] Browser extension
- [ ] Team/agency multi-IP dashboard (paid tier)
- [ ] Affiliate integration for proxy/VPN recommendations

---

## 13. Summary

duoniu.cc already has the right underlying toolset (WHOIS/ASN/BGP/CIDR/DNS/route/trace/ping/purity/leak/env) to compete in the IP-diagnostics-for-cross-border-operators category — the opportunity is **repositioning from "a bundle of network tools" to "a guided pre-flight check with a trustworthy verdict,"** matching the pattern that's winning in this space (iptoushi.com's grading, ip86.net's platform-fit framing, IPIPseek's cross-validation) while adding a genuine differentiator: **native plain-language explanation and actionable guidance**, rather than raw data dumps.

---

*This document is a planning proposal based on sitemap structure and category research, not a direct audit of the live site's current copy or visuals. Recommend validating each section against actual page screenshots/content before implementation.*
