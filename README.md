# Marketing & Sales Automation (n8n + Lark + Claude AI) — v2.0

A production, multi-channel **lead-capture + AI chatbot + CRM + ads-reporting** system built on
**n8n**, **Lark/Feishu Bitable**, and **Claude AI**. It captures leads from Facebook Lead Ads,
Facebook Messenger, Google Lead Forms, website and generic webhook channels; an AI bot qualifies
them for two brands (lithium batteries vs. forklift rental) and a recruitment track; then it
writes the results into a Lark CRM and alerts the right Sales/HR groups — and reports Meta + Google
ad spend every morning.

> **This repository is a sanitized TEMPLATE.** Every real credential and every real infrastructure
> ID (Lark base/table IDs, Facebook page/app IDs, ad-account IDs, chat IDs, domains, contacts) has
> been replaced with `<PLACEHOLDER>` tokens. Nothing here connects to anyone's live account until
> **you** fill in your own values. See **[PLACEHOLDERS.md](PLACEHOLDERS.md)** and **[SETUP.md](SETUP.md)**.

## What's new in v2.0

- **Bidirectional lead de-duplication** — a single ad submission can hit the system as *both* a Lead-Ads webhook *and* a Messenger "form echo". v2.0 keeps a phone-keyed marker table (`mh_lead_dedup`) and gates **both** branches (fail-open) so the same customer is never written twice — regardless of which event arrives first.
- **De-dup housekeeping** — a dedicated `MH-Dedup-Prune` workflow (cron) clears expired markers.
- **Honest session flags** — a deduplicated lead now records `lead_created=false` instead of silently looking "created".
- **13 workflows total**, MH-03 core now hardened end-to-end (HMAC-signed webhook tests pass).

See **[docs/MH_CHANGELOG_2026-06-09.md](docs/MH_CHANGELOG_2026-06-09.md)** for the full diff and test evidence.

---

## What it does

```
SOURCES                         PROCESSING (n8n + Claude AI)          DESTINATIONS (Lark CRM + alerts)
───────                         ───────────────────────────          ─────────────────────────────
FB comment ─(Meta auto DM)─┐
FB Messenger ──────────────┤
FB Lead Form ──────┐       ├─►  MH-03  ── brand routing ──►  Lark Leads / Deals / CSKH (staging)
Google Lead Form ──┼─► MH-04│        (BATTERY ↔ FORKLIFT ↔ Recruitment)  + marketing table
Zalo / hotline ────┴─► MH-01│        + 2-way lead de-duplication        + Sales groups (2 brands)
Website ───────────► MH-WEBSITE                                          + HR group
                            │
                            ▼
                  MH-Dashboard (daily 8AM / weekly / monthly)  → Lark report + Meta & Google ad spend
                  MH-Dedup-Prune (cron)                        → expire stale de-dup markers
                  MH-DLQ (error trigger)                       → centralized failure alerts
```

## Key features

- **One webhook, three event types** — Facebook leadgen / message / comment routed from a single endpoint.
- **Brand-aware AI bot** — auto-detects battery (Q1–Q14) vs. forklift rental (R1–R5) and a recruitment flow; never quotes prices; extracts phone via regex on the raw message (not AI-guessed).
- **2-way lead de-duplication** — phone-keyed marker (`mh_lead_dedup`); leadgen marks key `core`, Messenger form-echo marks key `core:msg`; each branch checks the other before writing, so a dual-event submission yields one lead. Fail-open: any error falls back to creating the lead, never silently dropping it.
- **Debounce-and-merge** — coalesces rapid multi-message bursts into one AI call.
- **Per-customer sessions in an n8n Data Table** — no cross-customer clobber under concurrency.
- **Re-engagement** — returning customers with a new need get a fresh lead + a "returning customer" alert.
- **Auto follow-up** — silent customers are nudged at 10 min / 4 h / 24 h.
- **CRM hub linking** — on completion the bot creates a Customer record and links Lead/Deal/CSKH to it.
- **Ads dashboard** — pulls Meta (Graph API) + Google Ads (GAQL) spend/clicks/CTR into a morning/weekly/monthly Lark report.
- **Resilience** — every external call has retry (3×) + `onError: continueRegularOutput`; all errors funnel into a Dead-Letter-Queue workflow that alerts Lark.

## Repository layout

```
workflows/all-wf.json   # all 13 n8n workflows (sanitized export — import this into n8n)
docs/                   # full technical + operational documentation (Vietnamese)
PLACEHOLDERS.md         # every <PLACEHOLDER> token and what to replace it with
SETUP.md                # step-by-step deployment guide
.env.example            # the placeholders in env-var form
```

### Documentation map (`docs/`)
| File | What it covers |
|---|---|
| `MH_SYSTEM_ARCHITECTURE.md` | Per-workflow technical architecture (nodes, connections, table mapping) |
| `MH_PROJECT_HANDOVER.md` | Handover quick reference |
| `BAO_CAO_DU_AN_MH.md` | Management/project report |
| `MH_USER_GUIDE.md` | Role-based operating guide (Sales / Marketing / HR / Manager) |
| `MH_DANH_GIA_PRODUCTION_2026-06-08.md` | Production readiness assessment + packaging checklist |
| `MH_SESSION_DATATABLE_PLAN.md` | Session storage design (Data Table) |
| `MH_ADS_DASHBOARD_PLAN.md` | Meta + Google ads dashboard design |
| `MH_BOTMESSENGER_UPGRADE_PLAN.md`, `MH_BOT_EVALUATION_2026-05-30.md` | Bot upgrade plan & evaluation |
| `MH_CHANGELOG_2026-06-09.md` | v2.0 — bidirectional de-duplication (latest) |
| `MH_CHANGELOG_2026-06-02.md` | Earlier change history |
| `GoogleAds_API_Design_Document.md`, `botmrketlark.md` | Google Ads API design, marketing notes |

> Docs are in **Vietnamese** (the system's operating language). The workflow logic is language-agnostic.

## Quick start

1. Install **n8n** (Docker recommended) — see [SETUP.md](SETUP.md).
2. Import `workflows/all-wf.json`.
3. Replace every `<PLACEHOLDER>` (search the imported workflows) with your own IDs/credentials — see [PLACEHOLDERS.md](PLACEHOLDERS.md).
4. Create your Lark base + tables, Facebook app, Google Ads OAuth, and a public webhook (e.g. Cloudflare Tunnel).
5. Activate the workflows and test with a signed HMAC webhook before going live.

## Tech stack

n8n · Lark/Feishu Bitable · Claude AI (Haiku) · Facebook Graph API v21 · Google Ads API · Cloudflare Tunnel.

## Security note

This is a sanitized template. **Never commit real tokens.** Keep credentials in n8n's credential
store or environment variables, not hardcoded in nodes. The included `.gitignore` blocks common
secret files. If you fork and customize, run a secret scan before every push.

## License

[MIT](LICENSE) — provided as-is, without warranty. Brand names, product names and recruitment
content in the docs are illustrative of the original deployment and are not endorsements.
