# Shopify Analytics Platform — Phase 1 Product Plan

> **Mission:** Help Shopify merchants ($500K–$3M GMV) see their true profit and best customers in under 5 minutes. No pixel. No complexity. Just clarity.

---

## Table of Contents

1. [App Overview](#app-overview)
2. [Tech Stack](#tech-stack)
3. [Data Architecture](#data-architecture)
4. [Pages & Features](#pages--features)
   - [Onboarding Flow](#1-onboarding-flow)
   - [Summary / Home](#2-summary--home-page)
   - [Sales Analytics](#3-sales-analytics-page)
   - [Customer Analytics](#4-customer-analytics-page)
   - [Funnel Analytics](#5-funnel-analytics-page)
   - [Settings](#6-settings-page)
5. [Weekly Email Report](#weekly-email-report)
6. [Metrics Reference](#metrics-reference)
7. [Build Timeline](#build-timeline)
8. [Pricing Model](#pricing-model)
9. [Shopify Marketplace Requirements](#shopify-marketplace-requirements)
10. [Post-Launch Targets](#post-launch-targets)
11. [Phase 2 Roadmap](#phase-2-roadmap)

---

## App Overview

### What We're Building
A merchant-facing profit and customer analytics platform for Shopify stores. The core value proposition is showing merchants their **true net profit** (after COGS, ad spend, shipping, and fees) and **who their best customers are** — data they cannot get from Shopify natively.

### Target Merchant
- Shopify stores doing $500K–$3M GMV annually
- Solo founders or small teams (1–5 people)
- Running paid ads on Meta (primary) and Google
- Frustrated with Shopify's shallow analytics
- Priced out of or overwhelmed by Triple Whale ($300–$500/mo)

### Competitive Positioning

| | Triple Whale | Polar Analytics | **Our App** |
|---|---|---|---|
| Target | $5M+ DTC brands | $1M–$10M brands | $500K–$3M brands |
| Core value | Ad attribution | Multi-channel | P&L + Retention |
| Price | $300–$500/mo | $200–$400/mo | $79–$149/mo |
| First-party pixel | Required | Required | Not required (Phase 1) |
| Setup time | 30–60 min | 20–40 min | **Under 5 min** |

### What We Explicitly Don't Build in Phase 1
- First-party tracking pixel
- Meta / Google Ads API integration (manual input only)
- Multi-touch attribution
- Custom report builder
- Mobile app
- Multi-user / team access
- AI-generated insights
- CSV export

---

## Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| Backend | Node.js (Express) | Best Shopify SDK support |
| Database | PostgreSQL | Reliable, scales well |
| Frontend | Next.js + Tremor | Pre-built analytics components |
| Job Queue | BullMQ + Redis | Handle sync jobs reliably |
| Email | Resend | Simplest transactional email |
| Hosting | Railway | One-click Postgres + Redis + app |
| Billing | Shopify Billing API | Mandatory for marketplace |

---

## Data Architecture

### Shopify Data Sources

| Data | Method | Frequency |
|---|---|---|
| Orders (historical) | REST API backfill | On install (12 months) |
| New orders | Webhook: `orders/create` | Real-time |
| Order updates / refunds | Webhook: `orders/updated`, `refunds/create` | Real-time |
| Abandoned checkouts | REST API polling | Every hour |
| Customers | REST API + webhook | On install + real-time |

### Database Schema (MVP)

```sql
-- Merchants
merchants (
  id, shopify_domain, access_token,
  cogs_percentage, avg_shipping_cost,
  processing_fee_percentage,
  meta_daily_budget, google_daily_budget,
  tiktok_daily_budget, other_daily_budget,
  installed_at, trial_ends_at, plan
)

-- Raw Data
orders (
  id, merchant_id, shopify_order_id,
  total_price, subtotal, discount_amount,
  refund_amount, created_at, customer_id,
  is_new_customer, landing_site, source_name
)

order_line_items (
  id, order_id, product_id, product_title,
  quantity, price, sku, variant_id
)

customers (
  id, merchant_id, shopify_customer_id,
  first_order_at, last_order_at,
  total_spent, order_count
)

checkouts (
  id, merchant_id, shopify_checkout_id,
  total_price, created_at, completed_at,
  abandoned_at, email, device_type
)

-- Pre-Aggregated (all dashboard queries hit this, never raw tables)
daily_summaries (
  id, merchant_id, date,
  gross_revenue, net_revenue,
  order_count, new_customers, returning_customers,
  total_discounts, total_refunds,
  cogs, shipping_cost, fees,
  ad_spend, net_profit, profit_margin
)

product_daily_stats (
  id, merchant_id, product_id,
  product_title, date,
  units_sold, gross_revenue, net_revenue
)
```

### Aggregation Strategy
- Pre-compute `daily_summaries` on every webhook event
- Dashboard queries **never** touch raw `orders` table at runtime
- Recompute last 7 days on every background sync to catch any webhook gaps

---

## Pages & Features

---

### 1. Onboarding Flow

**Goal:** Merchant sees their profit number in under 5 minutes.

**Flow:**
```
Install (Shopify OAuth) → COGS Setup → Ad Budget → Loading → Dashboard
```

---

#### Screen 1: Welcome + COGS Setup

**Purpose:** Collect the minimum inputs needed to compute true profit.

**Fields:**
- COGS % — "What % of your revenue goes to product cost?" (e.g. 30%)
- Average shipping cost per order — pre-fill $4.50
- Payment processing fee — radio select:
  - 2.9% + 30¢ (Shopify Payments / Stripe) — default
  - 2.2% + 30¢ (Shopify Payments Advanced)
  - Custom %

**UX Rules:**
- Background data sync starts the instant OAuth completes — never block merchant
- Helper text explains COGS % with a concrete example
- Pre-fill common values to reduce friction
- Single CTA: "Continue →"

---

#### Screen 2: Ad Budget (Optional)

**Purpose:** Capture daily ad spend per channel for P&L accuracy.

**Fields (all optional, zero-filled by default):**
- Meta Ads daily budget ($/day)
- Google Ads daily budget ($/day)
- TikTok Ads daily budget ($/day)
- Other daily budget ($/day)

**UX Rules:**
- Clearly marked as optional — "Skip for now" link always visible
- Teaser: "Auto-sync with Meta Ads — coming soon"
- Backend computes weekly/monthly spend by multiplying daily × period days

---

#### Screen 3: Data Loading

**Purpose:** Show sync progress, prevent bounce during wait.

**Elements:**
- Progress bar with real percentage
- Live count: "Imported 1,847 orders", "Synced 3,204 customers"
- Rotating tip cards at the bottom (one every 5 seconds)
- If sync > 2 minutes: offer to email when ready, let them close tab

**Tip Card Examples:**
- "The average Shopify merchant loses 12% of revenue to untracked discounts"
- "Returning customers cost 5x less to convert than new ones"
- "Most stores don't know their true profit margin — you're about to"

---

#### Screen 4: First Dashboard View (Aha Moment)

**Purpose:** Deliver the payoff — their profit number, immediately.

**Elements:**
- Large profit number front and center (last 30 days)
- Full P&L breakdown: Revenue → Ad Spend → COGS → Shipping → Fees → **Net Profit**
- Single CTA: "Go to Dashboard →"
- Tagline: "Most Shopify merchants don't know this number. Now you do."

---

#### Post-Onboarding Email Sequence

| Day | Email | Content |
|---|---|---|
| 0 | Welcome | Profit summary + 3 things to check first |
| 2 | Tip | "Check your refund rate — here's what's normal" |
| 7 | First weekly report | Auto-generated Monday report |
| 10 | Re-engage (if no login) | "Here's what changed this week in your store" |
| 14 | Trial ending | "Your trial ends in 2 days — here's what you'd lose" |

---

### 2. Summary / Home Page

**Purpose:** The merchant's daily dashboard. One screen with everything that matters.

**This is the most important page in the app — design it to be opened every morning.**

---

#### Period Switcher
Toggle between: Today | This Week | This Month | Last 30 Days | Last 90 Days

All metrics update instantly on toggle. Show the equivalent prior period for every metric.

---

#### P&L Block (Hero Section)

| Metric | Description | Format |
|---|---|---|
| Gross Revenue | Total revenue before any deductions | $XX,XXX ▲/▼ X% vs prior |
| Discounts | Total discount value given away | −$X,XXX |
| Refunds | Total refund amount | −$X,XXX |
| Net Revenue | Gross − Discounts − Refunds | $XX,XXX |
| Ad Spend | Daily budget × days in period (manual) | −$X,XXX |
| COGS | Net Revenue × COGS % | −$X,XXX |
| Shipping | Avg shipping cost × order count | −$X,XXX |
| Processing Fees | Calculated from fee % | −$X,XXX |
| **Net Profit** | **The number they care about most** | **$XX,XXX ▲/▼ X%** |
| Profit Margin | Net Profit / Gross Revenue | XX.X% |

**Design note:** Net Profit gets 2x font size of every other metric. It's the star.

---

#### KPI Cards Row

| Metric | Description | Period Comparison |
|---|---|---|
| Total Orders | Order count for period | ▲/▼ X% vs prior |
| Average Order Value (AOV) | Net Revenue / Orders | ▲/▼ X% vs prior |
| New Customers | First-time buyers | ▲/▼ X% vs prior |
| Returning Customers | Repeat buyers | ▲/▼ X% vs prior |
| Repeat Purchase Rate | % customers who bought again (90-day) | ▲/▼ X pts |
| Refund Rate | Refunded orders / Total orders | ▲/▼ X pts |

---

#### Revenue Trend Chart

- Bar chart — daily revenue for selected period
- Two series: Gross Revenue (light) + Net Profit (dark)
- Hover tooltip shows: Date, Revenue, Ad Spend, Net Profit
- Period-over-period overlay toggle (show last period as line)

---

#### Quick Insights Strip

3 auto-generated callouts beneath the chart:

- "🔴 Refund rate is 6.2% — above the 3–4% healthy range"
- "🟢 Returning customer revenue up 22% vs last month"
- "🟡 Discounts account for 14% of gross revenue this month"

Rules: Show max 3. Always show one positive, one negative, one neutral if possible.

---

### 3. Sales Analytics Page

**Purpose:** Help merchants understand where their revenue comes from and where it leaks.

---

#### Revenue by Product (Table + Bar Chart)

Columns:

| Column | Description |
|---|---|
| Product Name | Product title |
| Units Sold | Quantity sold in period |
| Gross Revenue | Total revenue from product |
| Discounts | Discount amount applied to this product |
| Refunds | Refund amount for this product |
| Net Revenue | Gross − Discounts − Refunds |
| Est. Profit | Net Revenue − (COGS % × Net Revenue) |
| % of Total | This product's share of net revenue |

- Default sort: Net Revenue descending
- Show top 20 products
- Bar chart view toggle — horizontal bars by net revenue

---

#### Revenue by Day Chart

- 30-day or 90-day bar chart
- Toggle between: Gross Revenue / Net Revenue / Net Profit
- Click a bar to see that day's breakdown in a side panel

---

#### Discount Analysis Block

| Metric | Value |
|---|---|
| Total Discounts Given | $X,XXX |
| Discounts as % of Revenue | X.X% |
| Orders With Discounts | XXX (XX% of orders) |
| Top Discount Codes | Code — Used X times — $X,XXX given |
| Avg Discount per Order | $XX |

Callout: Flag if discounts exceed 10% of revenue.

---

#### Refund Analysis Block

| Metric | Value |
|---|---|
| Total Refunds | $X,XXX |
| Refund Rate (by orders) | X.X% |
| Refund Rate (by revenue) | X.X% |
| Top Refunded Products | Product — X refunds — $X,XXX |
| Avg Days to Refund | XX days |

Callout: Flag if refund rate exceeds 4%.

---

#### Revenue Breakdown (Donut Charts)

Two side-by-side donuts:
1. Revenue by source: Direct / Organic / Paid / Email / Unknown (parsed from `landing_site` UTM)
2. Revenue by device: Mobile / Desktop / Tablet

---

### 4. Customer Analytics Page

**Purpose:** Help merchants understand who their customers are, which ones are most valuable, and whether they're retaining them.

---

#### Customer Overview Cards

| Metric | Description | Period Comparison |
|---|---|---|
| Total Customers | Unique customers in period | ▲/▼ X% |
| New Customers | First purchase in period | ▲/▼ X% |
| Returning Customers | 2nd+ purchase in period | ▲/▼ X% |
| New vs Returning Revenue Split | % of revenue from each | vs prior period |
| Avg Customer LTV (12-month) | Avg total spend per customer over 12 months | — |
| Avg Orders per Customer | Total orders / unique customers | — |

---

#### New vs Returning Revenue (Stacked Bar Chart)

- 12-week stacked bar chart
- New customer revenue (one color) + Returning customer revenue (another color)
- Shows trend: is the store becoming more reliant on returning customers? (good sign)

---

#### Repeat Purchase Rate Block

| Metric | Value |
|---|---|
| 30-Day Repeat Rate | % of customers who bought again within 30 days |
| 60-Day Repeat Rate | % within 60 days |
| 90-Day Repeat Rate | % within 90 days |
| Avg Days Between 1st and 2nd Purchase | XX days |
| Avg Days Between 2nd and 3rd Purchase | XX days |

**Why avg days between purchases matters:** Tells the merchant when to trigger a win-back email.

Callout: "Your customers typically buy again within 34 days. Consider a re-engagement email at day 30."

---

#### Top Customers by LTV (Table)

Show top 25 customers.

| Column | Description |
|---|---|
| Customer | First name + last initial (privacy) |
| Total Spent | Lifetime spend |
| Orders | Total order count |
| AOV | Average order value |
| First Order | Date of first purchase |
| Last Order | Date of most recent purchase |
| Days Since Last Order | Recency indicator |

- Highlight in red: customers with high LTV but 90+ days since last order (win-back opportunity)
- Export button: CSV of top customers (Phase 1 nice-to-have)

---

#### Customer Acquisition Trend (Line Chart)

- 12-week line chart
- Two lines: New customers per week + Returning customers per week
- Shows whether the business is growing or relying on existing base

---

### 5. Funnel Analytics Page

**Purpose:** Show where customers drop off between browsing and buying.

**Data source:** Shopify `checkouts` API (abandoned checkouts) + `orders` (completed purchases).

**Note:** Session-level funnel (add-to-cart rate) requires a tracking pixel and is Phase 2. Phase 1 funnel starts at checkout initiation.

---

#### Funnel Visualization

```
Checkouts Initiated      XXXX        100%
       ↓
Checkouts Completed      XXX         XX%     ← Checkout Completion Rate
       ↓
Orders Placed            XXX         XX%     ← Overall Conversion
```

Show for: Today / This Week / This Month / Last 30 Days

---

#### Funnel Metrics Cards

| Metric | Description |
|---|---|
| Checkout Initiation Count | Total checkouts started in period |
| Checkout Completion Rate | Completed checkouts / Total initiated |
| Cart Abandonment Rate | 1 − Completion Rate |
| Abandoned Checkout Value | $ value of uncompleted checkouts |
| Avg Time to Complete Checkout | From initiation to order (completed only) |

---

#### Abandoned Checkout Breakdown

| Metric | Value |
|---|---|
| Total Abandoned Checkouts | XXX |
| Total Abandoned Value | $X,XXX |
| Abandoned by Device | Mobile XX% / Desktop XX% / Tablet XX% |
| Abandoned Checkouts (with email) | XXX — "These can be recovered" |
| Avg Abandoned Cart Value | $XX |

Callout: "You have $X,XXX in abandoned checkouts this month. Consider setting up Shopify's abandoned cart emails."

---

#### Checkout Conversion Trend (Line Chart)

- 12-week line chart
- Checkout completion rate over time
- Spot improvements after store changes, promotions, or checkout flow edits

---

#### Top Abandoned Products

Products most frequently left in abandoned checkouts (from checkout line items).

| Column | Description |
|---|---|
| Product | Product title |
| Times Abandoned | Count of abandoned checkouts containing this product |
| Abandoned Value | Total $ value abandoned |
| Completion Rate | % of checkouts with this product that completed |

---

### 6. Settings Page

**Purpose:** Let merchants update their cost inputs and manage their account.

---

#### Store Settings

| Setting | Description |
|---|---|
| COGS % | Update product cost percentage — recalculates all historical profit |
| Average Shipping Cost | Per-order shipping cost |
| Processing Fee | Payment processor fee % |

"Save Changes" recalculates all `daily_summaries` in a background job.

---

#### Ad Budget Settings

| Setting | Description |
|---|---|
| Meta Ads Daily Budget | $/day |
| Google Ads Daily Budget | $/day |
| TikTok Ads Daily Budget | $/day |
| Other Daily Budget | $/day |

Show last updated date. Prompt if not updated in 30 days.

---

#### Notification Settings

| Setting | Default |
|---|---|
| Weekly email report (every Monday) | On |
| Refund rate alert (if > 5%) | On |
| Revenue drop alert (if > 20% vs prior week) | On |
| Trial expiry reminder | On |

---

#### Account & Billing

- Current plan name + price
- Trial end date (if applicable)
- "Manage Billing" → Shopify Billing portal
- "Disconnect Store" → Triggers GDPR cleanup + app uninstall

---

#### Data Sync Status

| Item | Status |
|---|---|
| Last sync | "2 minutes ago" |
| Orders synced | X,XXX |
| Customers synced | X,XXX |
| Sync coverage | "Last 12 months" |
| "Force re-sync" | Button — re-runs full backfill |

---

## Weekly Email Report

Sent every Monday at 9AM (merchant's local time).

**Subject:** `Your store made $X,XXX last week — here's the breakdown`

**Content:**

```
Hi [First Name],

Here's your weekly profit report for [Store Name].

LAST WEEK vs WEEK BEFORE
─────────────────────────────────────────
Net Profit        $4,200    ▲ 18%
Gross Revenue     $9,800    ▲ 12%
Ad Spend          $2,100    ▼  5%
Orders              186     ▲  8%
New Customers        94     ▲ 14%
Returning            92     ▲ 22%
─────────────────────────────────────────

TOP PRODUCT THIS WEEK
Product Name — $X,XXX revenue — XX units

THIS WEEK'S INSIGHT
Your returning customers drove 58% of revenue
— up from 41% last month. Your retention is
improving.

[View Full Dashboard →]
```

**Rules:**
- Always include one specific insight (auto-generated)
- Deep link directly to the relevant dashboard page
- Plain text + minimal HTML — high deliverability
- Unsubscribe link required

---

## Metrics Reference

Complete list of every metric shown in Phase 1, with definitions.

### Revenue Metrics

| Metric | Formula | Where Shown |
|---|---|---|
| Gross Revenue | Sum of order totals | Summary, Sales |
| Net Revenue | Gross − Discounts − Refunds | Summary, Sales |
| Net Profit | Net Revenue − Ad Spend − COGS − Shipping − Fees | Summary (hero) |
| Profit Margin | Net Profit / Gross Revenue × 100 | Summary |
| AOV | Net Revenue / Order Count | Summary, Sales |
| Discount Rate | Total Discounts / Gross Revenue × 100 | Sales |
| Refund Rate | Refunded Orders / Total Orders × 100 | Summary, Sales |

### Cost Metrics

| Metric | Formula | Where Shown |
|---|---|---|
| COGS | Net Revenue × COGS % | Summary, P&L |
| Ad Spend | Daily Budget × Days in Period | Summary, P&L |
| Shipping Cost | Avg Shipping Cost × Order Count | Summary, P&L |
| Processing Fees | Net Revenue × Fee % + (Order Count × $0.30) | Summary, P&L |

### Customer Metrics

| Metric | Formula | Where Shown |
|---|---|---|
| New Customers | Customers with 1 order ever | Summary, Customers |
| Returning Customers | Customers with 2+ orders ever | Summary, Customers |
| Customer LTV (12-mo) | Total spend / Unique customers (12 months) | Customers |
| Repeat Purchase Rate (30d) | Customers who bought again within 30d / Total customers | Customers |
| Avg Days to 2nd Purchase | Avg(2nd order date − 1st order date) | Customers |

### Funnel Metrics

| Metric | Formula | Where Shown |
|---|---|---|
| Checkout Initiation Count | Count of checkouts created | Funnel |
| Checkout Completion Rate | Completed checkouts / Total checkouts | Funnel |
| Cart Abandonment Rate | 1 − Checkout Completion Rate | Funnel |
| Abandoned Checkout Value | Sum of uncompleted checkout totals | Funnel |

---

## Build Timeline

### 8-Week Plan (Solo Developer)

| Week | Focus | Deliverables |
|---|---|---|
| 1 | Foundation | Shopify OAuth, app setup, orders + customers sync |
| 2 | Data Pipeline | Webhooks, 12-month backfill, `daily_summaries` aggregation |
| 3 | Summary Page | P&L block, KPI cards, revenue trend chart |
| 4 | Sales Page | Product breakdown, discount analysis, refund analysis |
| 5 | Customer Page | LTV table, new vs returning, repeat purchase rate |
| 6 | Funnel Page | Abandonment rate, funnel visualization, abandoned products |
| 7 | Billing + Email | Shopify Billing API, weekly email report, onboarding emails |
| 8 | Launch Prep | GDPR webhooks, privacy policy, app listing, submit for review |

### Shopify App Review Checklist (Week 8)
- [ ] GDPR mandatory webhooks: `app/uninstalled`, `shop/redact`, `customers/redact`, `customers/data_request`
- [ ] Privacy policy page live
- [ ] Data handling disclosure
- [ ] App listing copy + screenshots
- [ ] 14-day free trial configured in Shopify Billing API
- [ ] Test on dev store with real data flow

---

## Pricing Model

| Plan | Price | Order Limit | Target Merchant |
|---|---|---|---|
| Starter | $19/mo | 500 orders/month | Just getting started |
| Growth | $49/mo | 5,000 orders/month | Scaling stores |
| Scale | $99/mo | Unlimited | Established brands |

**Rules:**
- 14-day free trial on all plans — no credit card required (Shopify handles billing)
- No freemium — kills support bandwidth for a solo builder
- Price in Shopify Billing API (USD, charged through Shopify)

---

## Shopify Marketplace Requirements

### Mandatory Webhooks (Day 1)

| Webhook | Purpose |
|---|---|
| `app/uninstalled` | Clean up merchant data, cancel billing |
| `shop/redact` | Delete all shop data (GDPR) |
| `customers/redact` | Delete specific customer data (GDPR) |
| `customers/data_request` | Return customer data on request (GDPR) |

### App Listing Requirements
- App icon (1200×1200px)
- 3–5 app screenshots (1600×900px)
- Short description (100 chars)
- Long description (detailing features)
- Privacy policy URL
- Support email

### Review Timeline
- Shopify App Review: 2–4 weeks
- Submit in Week 8, expect approval 2–4 weeks later

---

## Post-Launch Targets

| Milestone | Target Timeline |
|---|---|
| Live on Shopify App Store | End of Week 8 |
| First 10 installs | Week 10–12 |
| First paying merchant | Month 3 |
| $1,000 MRR | Month 4 |
| First merchant referral | Month 4–5 |
| 40% Week-2 retention | Month 3 (measure from first installs) |
| Validate Phase 2 features | Month 5 (interview top 10 merchants) |

### Key Metrics to Track (Internal)

| Metric | Target | Why |
|---|---|---|
| Time to first profit view | < 3 min | Onboarding quality |
| Week-2 retention | > 40% | Core value validation |
| Trial-to-paid conversion | > 20% | Pricing + value fit |
| Weekly active users | > 60% of paid | Engagement / stickiness |
| Churn rate (monthly) | < 8% | Long-term viability |

---

## Phase 2 Roadmap

**Trigger: 10+ paying merchants, $1K MRR, validated demand**

### Phase 2 — Retain (Month 4–6)
- Meta Ads API integration (auto ad spend sync)
- Blended ROAS per campaign
- Google Ads API
- Cohort retention table
- CSV export

### Phase 3 — Expand (Month 7–12, after $2K MRR)
- TikTok Ads API
- First-party tracking pixel
- Multi-touch attribution
- Creative analytics (which ad creatives drive best ROAS)
- AI-generated weekly insights
- Multi-user / team access

---

*Last updated: Phase 1 Plan — Pre-launch*
*Built for solo development. Scope locked to 8-week delivery.*
