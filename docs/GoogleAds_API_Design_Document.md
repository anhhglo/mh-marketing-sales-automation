# Google Ads API — Tool Design Document
**Company:** MH (MHrental — Forklift Rental / MHpower — Lithium Batteries for Forklifts)
**Developer token MCC:** <GOOGLE_ADS_LOGIN_CUSTOMER_ID> · **API contact:** <EMAIL_REDACTED>
**Date:** 2026-06-04 · **Access requested:** Basic Access

---

## 1. Overview
We operate an **internal marketing & sales automation system** for our own company. The system captures B2B leads from multiple channels and stores them in our CRM. We request Google Ads API access for **two internal, read-oriented purposes only**:
1. **Campaign performance reporting** — read cost, clicks, impressions, conversions and keyword metrics from **our own** Google Ads accounts to build internal dashboards.
2. **Lead processing support** — correlate Google Lead Form submissions with the originating campaign.

The tool accesses **only our own advertising accounts** managed under MCC **<GOOGLE_ADS_LOGIN_CUSTOMER_ID>**. It does **not** manage third-party accounts, does **not** resell data, and is used **only by our internal employees**.

## 2. Business model
- **MHrental:** rents and sells forklifts (load, personnel/boom/scissor lifts) to construction & industrial clients in Vietnam.
- **MHpower:** supplies SuperV lithium batteries & industrial batteries for electric forklifts.
- We run **Google Ads Search, Shopping, and Display (remarketing)** campaigns to generate B2B leads and sales for these two brands, on **our own** Google Ads accounts (e.g. account <GOOGLE_ADS_CUSTOMER_ID> "Xe nâng người 1 - MH + LGMG"). Some Search campaigns also use lead forms.

## 3. System architecture
```
[ Google Lead Form (Ads) ] --webhook push--> [ n8n automation server ] --> [ Lark CRM (Bitable) ]
[ Google Ads accounts ] <--read-only API-->  [ Reporting service ]     --> [ Internal dashboard ]
                                              (uses Google Ads API token)
```
- **n8n** (self-hosted) receives lead-form submissions (webhook) and writes them to **Lark Bitable** CRM. *(This part does NOT use the Google Ads API token.)*
- A separate **internal reporting service** (our own Node.js script) calls the **Google Ads API** with the developer token to pull campaign metrics for internal dashboards.
- Hosting: on-premise server (Docker), accessed only by internal staff over the company network.

## 4. Google Ads API usage (read-only)
| API service / method | Purpose | Read/Write |
|----------------------|---------|------------|
| `GoogleAdsService.SearchStream` (GAQL) | Query `campaign`, `ad_group`, `metrics` (cost_micros, clicks, impressions, conversions) | **Read** |
| `GoogleAdsService.Search` | List campaigns / ad groups of our own accounts | **Read** |
| `KeywordPlanIdeaService` (optional) | Keyword search-volume metrics for planning | **Read** |

- **No write/mutate operations** are performed (no creating/editing/pausing campaigns).
- Queries are scoped to **our own customer IDs** only, via `login-customer-id = <GOOGLE_ADS_LOGIN_CUSTOMER_ID>`.
- Polling frequency: a few times per day for dashboard refresh (well within rate limits).

## 5. Required Minimum Functionality (RMF)
The tool provides meaningful reporting functionality beyond a single API call:
- Aggregates campaign metrics across date ranges and brands (MHrental vs MHpower).
- Computes derived KPIs (cost per lead, leads by campaign, conversion rate) for internal review.
- Presents results in an internal dashboard for the sales/marketing team.
- Correlates ad spend with leads captured in the CRM to evaluate channel effectiveness.

## 6. Data flow & storage
- Metrics retrieved from Google Ads API are stored in our internal CRM (Lark Bitable) / dashboard, on the company's own infrastructure.
- No advertising data is shared externally or resold.
- Access restricted to authenticated internal employees only.

## 7. Users & access control
- **Internal users only** (employees of MH). No external/public/client access.
- OAuth credentials and developer token are stored securely on the server, not exposed to end users.

## 8. Compliance statement
- The tool accesses **only the company's own Google Ads accounts** under MCC <GOOGLE_ADS_LOGIN_CUSTOMER_ID>.
- **Read-only** for reporting; **no** automated campaign mutation.
- **No** resale, sublicensing, or third-party account management.
- We will keep the API contact email monitored and respond to Google API team notices.
- The tool complies with Google Ads API Terms & Conditions and Required Minimum Functionality.

---
*Contact: <EMAIL_REDACTED> — Internal marketing/sales automation, MH (MHrental / MHpower), Vietnam.*
