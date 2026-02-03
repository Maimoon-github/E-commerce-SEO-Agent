# Aurum Pickleball — Stakeholder Map (Approvals + KPI Ownership)

This stakeholder map defines **who approves what** and **who owns each KPI** for the `AurumSEO Operator` program.

---

## 1) Core roles (recommended)

> Use real names in implementation; below are role titles with clean accountability.

- **Business Owner (Accountable Exec)**: Founder / GM / Head of Ecommerce  
- **SEO Lead (Program Owner)**: SEO Strategist / SEO Manager (owns SEO roadmap + prioritization)  
- **Engineering Lead (Tech Owner)**: Web/Platform Lead (owns codebase + releases)  
- **Ecommerce Merch Lead (Catalog Owner)**: Product/Merchandising Manager (owns PDP data quality)  
- **Content Lead (Editorial Owner)**: Managing Editor / Content Manager  
- **Brand Lead (Voice & Claims Owner)**: Brand/Creative Director  
- **Analytics Lead (Measurement Owner)**: Analytics Engineer / Data Analyst  
- **Partnerships/PR Lead (Authority Owner)**: Partnerships Manager  
- **Customer Support Lead (VOC Owner)**: Support Manager (voice-of-customer signal owner)  
- **Security/IT Admin (Credentials Owner)**: DevOps/Security

---

## 2) Approval responsibility matrix (ChangeSet approvals)

### 2.1 Approval gates (risk-based)
- **Risk 0–2**: Auto-execute *only if* action is not approval-gated and not blocked
- **Risk 3**: Requires SEO Lead approval (and Brand Lead if copy/claims)
- **Risk ≥4**: Requires **SEO Lead + Engineering Lead** (and Security/IT if credentials/infrastructure)

---

### 2.2 ChangeSet approval map

| ChangeSet / Action Category | Primary Approver (A) | Responsible Implementer (R) | Consulted (C) | Informed (I) |
|---|---|---|---|---|
| **Meta updates** (title/meta) on specific URLs | SEO Lead | SEO Ops / Content Lead (CMS edits) | Brand Lead | Business Owner, Analytics Lead |
| **On-page copy modules** (category/PDP copy blocks) | Content Lead | Content team | SEO Lead, Brand Lead | Business Owner |
| **Internal linking blocks** (module/anchors) | SEO Lead | Content Lead / Web Dev | Engineering Lead | Analytics Lead |
| **Schema JSON-LD updates** (Product/Offer/Review/Breadcrumb) | Engineering Lead | Web Dev | SEO Lead | Business Owner |
| **Canonical changes** (variants/duplicates) | Engineering Lead | Web Dev | SEO Lead, Merch Lead | Analytics Lead |
| **Robots.txt / meta-robots policy / sitemap generation** | Engineering Lead | Web Dev / DevOps | SEO Lead, Security/IT | Business Owner |
| **Redirect rules** (301/302), fixing chains/404 | Engineering Lead | Web Dev | SEO Lead | Analytics Lead |
| **Faceted navigation / parameter control** | Engineering Lead | Web Dev | SEO Lead | Business Owner |
| **CWV/performance changes** (images, bundles, caching) | Engineering Lead | Web Dev | SEO Lead | Business Owner |
| **Publish new content** (blog/hub page) | Content Lead | Content team | SEO Lead, Brand Lead | Analytics Lead |
| **Refresh existing content** (update posts) | Content Lead | Content team | SEO Lead | Analytics Lead |
| **Product page data changes** (specs, naming, variant consolidation) | Merch Lead | Merch team | SEO Lead, Brand Lead | Engineering Lead |
| **Outreach plan creation** (targets, angles) | Partnerships/PR Lead | Partnerships | SEO Lead | Business Owner |
| **Outreach sending** (emails/DMs) | Partnerships/PR Lead | Partnerships | Brand Lead, Legal (if needed) | SEO Lead |
| **Tooling/API credentials** (grant/rotate access) | Security/IT Admin | Security/IT Admin | Engineering Lead | SEO Lead |
| **Tracking changes** (GA4 events, GTM tagging) | Analytics Lead | Analytics Engineer | Engineering Lead, SEO Lead | Business Owner |

**Hard stop (Blocked actions):**
- Paid links, PBNs, manipulative outreach  
- Automated outreach sending without human control  
- Checkout/pricing changes by the SEO agent  
- Any action that exposes/stores secrets in prompts/state/logs

---

## 3) KPI ownership map (who owns each KPI and who supports it)

### 3.1 Business KPIs (North Star)

| KPI | Owner (Accountable) | Responsible (Day-to-day) | Supporting Roles |
|---|---|---|---|
| **Organic Revenue** | Business Owner | Analytics Lead | SEO Lead, Merch Lead |
| **Organic Conversion Rate (CVR)** | Business Owner | Analytics Lead | Merch Lead, Content Lead, Engineering Lead |
| **Average Order Value (AOV) from Organic** | Business Owner | Merch Lead | Analytics Lead, SEO Lead |
| **Revenue per Organic Session** | Business Owner | Analytics Lead | SEO Lead, Engineering Lead |

---

### 3.2 SEO Performance KPIs

| KPI | Owner (Accountable) | Responsible | Supporting Roles |
|---|---|---|---|
| **Non-brand clicks & impressions** | SEO Lead | SEO Lead | Analytics Lead |
| **CTR on priority pages/queries** | SEO Lead | SEO Ops | Content Lead, Brand Lead |
| **Rank distribution (Top 3/10/20)** | SEO Lead | SEO Ops | Content Lead |
| **Share of Voice (priority clusters)** | SEO Lead | SEO Ops | Partnerships/PR Lead (authority signals), Content Lead |
| **Indexed eligible PDP ratio** | SEO Lead | Engineering Lead | Merch Lead |

---

### 3.3 Technical KPIs

| KPI | Owner (Accountable) | Responsible | Supporting Roles |
|---|---|---|---|
| **Core Web Vitals pass rate (mobile)** | Engineering Lead | Web Dev | SEO Lead |
| **Crawl waste reduction / parameter containment** | Engineering Lead | Web Dev | SEO Lead |
| **Redirect/404/soft-404 count downtrend** | Engineering Lead | Web Dev | SEO Lead |
| **Valid structured data coverage** | Engineering Lead | Web Dev | SEO Lead |
| **Sitemap freshness + coverage** | Engineering Lead | Web Dev | SEO Lead |

---

### 3.4 Authority KPIs (Ethical)

| KPI | Owner (Accountable) | Responsible | Supporting Roles |
|---|---|---|---|
| **New relevant referring domains** | Partnerships/PR Lead | Partnerships | SEO Lead |
| **Anchor text naturalness** | SEO Lead | SEO Ops | Partnerships/PR Lead |
| **Link velocity stability (no suspicious spikes)** | SEO Lead | SEO Ops | Partnerships/PR Lead |

---

## 4) KPI reporting cadence (so nobody argues later)

| Reporting Artifact | Frequency | Owner | Audience |
|---|---|---|---|
| SEO Watchtower (indexation, CTR drops, revenue landers) | Daily | SEO Lead | Engineering Lead, Business Owner (optional) |
| Weekly SEO Ops Report (wins, regressions, next sprint) | Weekly | SEO Lead | Business Owner, Engineering Lead, Content Lead |
| Monthly Growth Review (SOV, revenue attribution, roadmap) | Monthly | Business Owner | All leads |
| Quarterly Technical Audit + Content Refresh plan | Quarterly | Engineering Lead + Content Lead | Business Owner, SEO Lead |

---

## 5) Approval escalation path (when things are urgent)

- **P0 (revenue-impacting regression or indexation block):**
  - First call: **SEO Lead + Engineering Lead**
  - If unresolved in 24–48h: escalate to **Business Owner**
- **High-risk technical changes (robots/sitemaps/canonicals/sitewide templates):**
  - Must be approved by **Engineering Lead + SEO Lead**
  - Security/IT involved if credentials or infra touched
- **Brand/claims conflicts (medical/legal/product claims):**
  - **Brand Lead** has veto power
  - Legal review if required by your org

---

## 6) Practical implementation tip (tight and clean)

Treat approvals as a **single structured object** per ChangeSet:
- `approval_required`
- `approvers[]` (role-based)
- `decision`
- `timestamp`
- `comment`

This makes audit logs defensible and prevents “who approved this?” drama later.
