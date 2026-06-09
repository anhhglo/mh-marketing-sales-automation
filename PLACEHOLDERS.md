# Placeholders — what to replace before running

This template ships with `<PLACEHOLDER>` tokens instead of real values. Search the imported
workflows (and the docs, if you reuse them) for each token and substitute your own value.

> Tip: in n8n you can export the workflow JSON, find/replace placeholders in a text editor, then
> re-import — or edit each node by hand. Prefer storing real **secrets** in n8n credentials or env
> vars rather than pasting them back into node parameters.

## 1. Secrets (NEVER commit the real values)

| Placeholder | What it is | Where to get it |
|---|---|---|
| `<FB_APP_SECRET>` | Facebook App secret (HMAC verify of webhooks) | Meta App → Settings → Basic |
| `<FB_ACCESS_TOKEN>` | Facebook Page access token (messaging + leadgen) | Meta — prefer a **System User** token (never-expires data access) |
| `<FB_VERIFY_TOKEN>` | Webhook verify token (must match Meta config) | You choose it |
| `<LARK_APP_ID>` / `<LARK_APP_SECRET>` | Lark/Feishu custom app credentials | Lark Developer Console → Credentials |
| `<CLAUDE_PROXY_KEY>` | API key for your Claude endpoint/proxy | Your Anthropic key or proxy |
| `<GOOGLE_CLIENT_ID>` / `<GOOGLE_CLIENT_SECRET>` / `<GOOGLE_REFRESH_TOKEN>` | Google OAuth for Ads API | Google Cloud Console + OAuth playground |
| `<GOOGLE_ADS_DEV_TOKEN>` | Google Ads developer token | Google Ads API Center (needs Basic Access) |
| `<MH04_WEBHOOK_KEY>` | Shared key on the Google webhook URL | You choose it |
| `<CF_API_TOKEN>` / `<CF_ACCOUNT_TAG>` / `<CF_ZONE_ID>` / `<CF_TUNNEL_SECRET>` / `<CF_TUNNEL_ID>` | Cloudflare Tunnel (public webhook) | Cloudflare dashboard |

## 2. Infrastructure IDs

| Placeholder | What it is |
|---|---|
| `<LARK_BASE_MHPOWER>` / `<LARK_BASE_MHRENTAL>` | The two Lark Bitable base (app) tokens |
| `<TBL_PW_*>` / `<TBL_RT_*>` | Lark table IDs inside each base (see mapping below) |
| `<N8N_DT_SESSIONS>` / `<N8N_DT_BUFFERS>` | n8n Data Table IDs for bot sessions/buffers |
| `<FB_PAGE_ID>` / `<FB_APP_ID>` | Facebook Page & App IDs |
| `<META_AD_ACCOUNT_MAIN>` / `<META_AD_ACCOUNT_ALT>` | Meta ad-account IDs (`act_…`) for the dashboard |
| `<GOOGLE_ADS_CUSTOMER_ID>` / `<GOOGLE_ADS_LOGIN_CUSTOMER_ID>` | Google Ads customer & MCC/login customer IDs |
| `<LARK_CHAT_SALES_POWER>` / `<LARK_CHAT_SALES_RENTAL>` / `<LARK_CHAT_HR>` | Lark group chat IDs (`oc_…`) that receive alerts |
| `<LARK_BOT_WEBHOOK_ID>` | Lark custom-bot incoming-webhook hook id (reports + DLQ) |
| `<PUBLIC_WEBHOOK_DOMAIN>` | Your public webhook domain (Cloudflare Tunnel hostname) |
| `<N8N_LOCAL_IP>` | n8n local address (docs only) |

### Lark table placeholder map
`PW` = MHpower (battery) base, `RT` = MHrental (forklift) base. `_COPY` = staging tables the bot
writes to; `_ORIG` = the real operational CRM tables (the bot does **not** write to these by design);
`DB1–DB5` = dashboard report tables; `_LEGACY` = retired pipeline tables.

| Placeholder | Original role (Vietnamese name) |
|---|---|
| `<TBL_PW_LEADS_COPY>` | 3.1.0 Leads Copy |
| `<TBL_PW_DEALS_COPY>` | 3.1.0 Deals Copy (NKT, Q1–Q14) |
| `<TBL_PW_CSKH_COPY>` | Lịch sử CSKH Copy |
| `<TBL_PW_MARKETING>` | leads marketings (power) |
| `<TBL_PW_CUSTOMER_COPY>` | Khách hàng Copy (hub) |
| `<TBL_PW_LEADS_ORIG>` / `<TBL_PW_DEALS_ORIG>` / `<TBL_PW_CUSTOMER_ORIG>` | Real Leads / Deals / Customer |
| `<TBL_RT_LEADS_COPY>` | 1. Leads liên hệ Copy |
| `<TBL_RT_CSKH_COPY>` | 2. Lịch sử CSKH Copy |
| `<TBL_RT_MARKETING>` | leads marketings (rental) |
| `<TBL_RT_CUSTOMER_COPY>` | Khách hàng Copy (hub) |
| `<TBL_RT_CONTRACT_COPY>` | Hợp đồng Copy |
| `<TBL_RT_DB1_MARKETING>` … `<TBL_RT_DB5_PRODUCTION>` | Dashboard tables DB1–DB5 |
| `<TBL_RT_*_LEGACY>` | Retired rental-pipeline tables (no longer used) |

## 3. Business content / PII (replace with your own)

| Placeholder | What it is |
|---|---|
| `<RECRUIT_CONTACT_PHONE>` / `<RECRUIT_CONTACT_EMAIL>` | Recruitment contact the bot gives applicants |
| `<EMAIL_REDACTED>` | Internal emails referenced in docs |
| `<EXAMPLE_PHONE>` / `<EXAMPLE_PSID>` | Example customer phone / Messenger PSID from test logs |

---

After substituting, **search the repo again** for any leftover `<…>` token to make sure nothing was missed.
