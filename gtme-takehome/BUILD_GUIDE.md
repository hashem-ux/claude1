# GTME Take-Home: Build Guide and Scoring Model

This is a working plan for building the take-home in the Clay web app. Clay's CLI and API
can't create workbooks, tables or columns, so every step below happens in the app. Treat the
weights and thresholds as a starting point. Change anything you can't defend in the
walkthrough.

---

## 0. Data cleanup (done before import)

`Accounts_clean.csv` is the file to import. Changes from the original:

| Issue in the raw file | What I did | Why it matters |
|---|---|---|
| No header row | Added `Company Name, Domain` | Clay maps columns from headers |
| Duplicates: ControlUp ×3, Aerospike ×2, Samara ×2 | Kept one row each: **50 rows → 46 accounts** | Duplicates double-spend credits and create double outreach |
| 11 rows with no domain (ActivTrak, Anomali, Apptegy, Assembly Software, EverDriven, Evolv Technology, Givebacks, Ketch, Medrio, Seekr, TalenTrust) | Filled from a Clay company match. The `Domain Source` column marks these | Domain is the join key for every enrichment. Without it, matches are unreliable |

Worth mentioning in the walkthrough. Calling these out shows you looked at the data:
- **Quartz Network**: quartznetwork.com resolves to `qrtz.link` in Clay's data.
- **Propeller** is headquartered in Australia, not the US. That matters because contacts must be US-based.
- **Samara** is listed as "Construction" with 11–50 employees but $25M–75M revenue. It sells backyard homes to consumers, so it's a weak fit despite the revenue.
- **Evolv Technology** is the only public company (NASDAQ: EVLV), so it's the only one where the brief's "AI adoption in the 10-K" example literally applies. That's why I chose a different AI data point (see step 2D).

---

## 1. Workspace setup

1. Create a workbook named **GTME Take-Home – Hashem Shouman**.
2. Import `Accounts_clean.csv` as a new table named **Accounts**.

---

## 2. Account enrichment

Run every enrichment on 5 rows first and check the results before running all 46.
The workspace has **200 credits**, so watch Clay's cost estimate on each column.

### A. Firmographics: revenue + industry
- Add Clay's company enrichment ("Enrich Company") keyed on **Domain**.
- Map out: `Annual Revenue` (range), `Industry`, `Employee Count`, `Country`.
- What to expect: all 46 matched in Clay's data during my check. Revenue comes back as ranges
  (`1M-5M` … `200M-500M`), which is why the scoring uses bands, not exact figures.

### B. Sales team headcount
Clay's company record doesn't include a headcount by department. Options, cheapest first:
1. **Find People at Company** filtered to the sales job function (titles like Account
   Executive, SDR, BDR, Sales). Use the **result count** as the sales headcount, and don't
   enrich those people. Check the app's cost estimate first.
2. A headcount-by-department enrichment from a provider in Clay's marketplace, if your plan
   has one.

Mention the limitation: the count is based on LinkedIn profiles, so it undercounts companies
with thin LinkedIn presence. It's a directional signal, not an exact number.

### C. Industry fit (formula, no credits)
LinkedIn's industry labels are noisy. "Software Development" covers both a B2B SaaS company and
a consumer app. Map them into fit buckets with a formula column:

| Bucket | Industries in this list | Points (of 25) |
|---|---|---|
| **Core**: B2B software, security, data | Software Development; Computer and Network Security; Desktop Computing Software Products; Data Infrastructure and Analytics; Technology, Information and Internet; IT Services | 25 |
| **Adjacent**: tech-enabled B2B, GTM-heavy | Advertising Services; Marketing Services; Business Consulting; Financial Services (B2B fintech); Education Administration Programs; Security and Investigations | 16 |
| **Weak**: services, healthcare, real estate | Hospitals and Health Care; Mental Health Care; Legal Services; Staffing and Recruiting; Real Estate; Philanthropic Fundraising | 8 |
| **Poor**: physical goods, consumer | Wholesale; Construction; Pharmaceutical Manufacturing; Machinery Maintenance | 3 |

The AI column in 2D can correct a bucket. For example, a "Real Estate" company that sells
software to brokers should count as Core.

### D. AI data point: "GTM Motion and Intent" (one AI research column)

**Why this one instead of "AI adoption in the 10-K":** 45 of 46 accounts are private, so a
10-K signal would be empty for almost every row. The question that actually predicts a Clay
deal is *"Does this company run an outbound or data-driven sales motion, and is it investing
in one right now?"*

Use one Claygent column (Clay's AI research agent) that returns structured JSON. One column
with three fields costs less than three separate columns:

> Research the company at {{Domain}} ({{Company Name}}). Use their website, careers page and
> recent news. Return JSON:
> - `sells_to`: "B2B", "B2C" or "Both". Who pays them?
> - `gtm_hiring`: true/false. Are they currently hiring for SDR/BDR, Account Executive,
>   Revenue Operations, Marketing Operations, Sales Operations or GTM Engineer roles? Include
>   the job title and URL as evidence.
> - `ai_in_gtm`: 0–3. 0 = no evidence, 1 = mentions AI generally, 2 = uses AI or automation
>   tools in sales/marketing, 3 = publicly invests in AI-led GTM (for example GTM engineering
>   roles, AI SDRs, case studies).
> - `evidence`: one sentence with a source URL.
> If you can't verify something, say "unknown". Don't guess.

Then pull `sells_to`, `gtm_hiring` and `ai_in_gtm` into their own columns with formulas.

Why it's defensible: hiring RevOps or SDRs is a *timing* signal. It means they're building
the motion Clay powers right now. B2C detection catches false positives like Samara and
Octave that look good on revenue alone.

---

## 3. Scoring model (0–100) → tier

| Input | Weight | Bands |
|---|---|---|
| **Revenue** | 30 | 75M+ = 30 · 25–75M = 22 · 10–25M = 14 · 5–10M = 7 · <5M = 3 |
| **Industry fit** | 25 | Core 25 · Adjacent 16 · Weak 8 · Poor 3 (from 2C) |
| **Sales headcount** | 25 | 50+ = 25 · 20–49 = 20 · 10–19 = 14 · 5–9 = 8 · 1–4 = 3 · 0/unknown = 0 |
| **GTM Motion and Intent (AI)** | 20 | `gtm_hiring` true = 10 · `ai_in_gtm` × 3.33 (max 10) |
| **Adjustments** | — | `sells_to` = B2C → −15 · HQ outside the US → −5 |

**Tiers:** **Gold ≥ 70** · **Silver 45–69** · **Bronze < 45**

Why these weights:
- **Revenue (30)** carries the most weight because it sets budget and contract size. Clay's
  paid plans make the most sense for companies with real GTM spend.
- **Industry (25) and sales headcount (25)** measure how much a company sells the way Clay's
  customers sell. Many sellers doing outbound means more seats and workflows to automate.
- **AI signal (20)** is weighted lowest because it's the noisiest input, but it's the only one
  that captures *timing*.
- The B2C penalty is a deliberate override. A $50M consumer company is still a bad fit.

Build it as one formula column, or one column per input plus a sum. The per-input version is
easier to explain on a screen share.

---

## 4. Contacts table

1. From the Accounts table, use **Find People at Companies** (Domain as input) and write
   results to a new table named **Contacts**.
2. Filters:
   - **Job title contains:** `Revenue Operations`, `RevOps`, `Marketing Operations`,
     `Marketing Ops` (optional: `GTM Operations`). Exclude `Intern`.
   - **Location:** United States.
   - **Current employer only.**
   - **Limit: 2 per company.**
3. Rank by seniority so the 2 slots go to the most senior people (VP/Head → Director →
   Manager → specialist).
4. **Work email:** add a work-email waterfall (several providers tried in order, stopping at
   the first hit) on name + company domain, then an email-validation step. Keep only
   **valid** emails.
5. Pull **Account Tier** and **Score** from Accounts with a lookup, so reps can sort contacts
   by tier.

Credit-saving order: if credits run low, find emails for **Gold and Silver accounts first**.
That's a decision you can explain in the walkthrough, not a corner cut.

Expect gaps. Many companies with 50–200 employees have no dedicated RevOps or MOps person. That's
a finding in itself: no ops owner often means a *bigger* need for Clay. The fallback,
contacting Sales Ops or a CRO, is outside the brief, so mention it rather than doing it.

---

## 5. For the working session

- **Criteria:** the four inputs and their weights, plus the B2C override.
- **Data you focused on:** revenue and industry give the fit, sales headcount gives the scale,
  and the AI column gives the timing.
- **Output you'd act on:** Gold accounts with a hiring signal and a verified RevOps/MOps
  email. Those go to a rep this week, with the evidence from the AI column as the opening
  line.
- **How you'd know it worked:** compare reply and meeting rates by tier. Gold should beat
  Silver, and Silver should beat Bronze. If it doesn't, re-weight. Also track email bounce
  rate (data quality) and the share of Gold accounts that turn into opportunities within 60 days.
