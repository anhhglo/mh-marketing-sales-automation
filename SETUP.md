# Setup guide

This deploys the MH automation on your own infrastructure. Plan ~half a day end-to-end.

## Prerequisites

- **n8n** v2.5+ (Docker recommended). The workflows use Code, HTTP Request, Webhook, Data Table,
  Schedule, IF and Wait nodes — all core nodes, no paid features required.
- A **public HTTPS endpoint** for webhooks (Meta/Google must reach n8n). The original uses a free
  **Cloudflare Tunnel**; ngrok or any reverse proxy works too.
- A **Lark/Feishu** workspace + a custom app with Bitable read/write scopes.
- A **Facebook** Page + App (for Lead Ads + Messenger), and optionally **Google Ads** API access.
- A **Claude** API key (direct Anthropic, or a compatible proxy endpoint).

## 1. Run n8n

```bash
docker run -d --restart unless-stopped --name n8n \
  -p 5678:5678 \
  -e GENERIC_TIMEZONE="Asia/Ho_Chi_Minh" \
  -e TZ="Asia/Ho_Chi_Minh" \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n:latest
```

## 2. Import workflows

n8n → **Workflows → Import from File →** `workflows/all-wf.json`. This imports all 12 workflows
(MH-01, MH-03, MH-03a, MH-04, MH-WEBSITE, MH-Dashboard Daily/Weekly/Monthly, MH-DLQ, MH-Session-Prune,
and two disabled legacy bots).

## 3. Create your Lark base + tables

Create two Bitable bases (battery + rental) with the tables listed in
[PLACEHOLDERS.md](PLACEHOLDERS.md) §2. The bot writes Vietnamese **column names** verbatim — keep
the field names from `docs/MH_SYSTEM_ARCHITECTURE.md` §10, or adjust both the table and the node code.

## 4. Create the n8n Data Tables

Create two Data Tables: `mh_sessions` (columns: `psid`, `session_json`, `is_complete`,
`last_is_bot`, `updated_at`, `created_at`) and `mh_buffers`. Note their IDs for the placeholders.

## 5. Fill in placeholders

Replace every `<PLACEHOLDER>` (see [PLACEHOLDERS.md](PLACEHOLDERS.md)). For **secrets**, prefer n8n
credentials / env vars over hardcoding in node parameters.

## 6. Wire up the channels

- **Facebook webhook:** point your Page webhook (fields `leadgen, messages, messaging_postbacks, feed`)
  at `https://<your-domain>/webhook/mh-facebook-lead`. The verify token must equal `<FB_VERIFY_TOKEN>`.
  **Turn OFF** Meta Instant Reply / AI Inbox so the n8n bot receives messages.
- **Google Lead Form:** `https://<your-domain>/webhook/mh-google-lead?key=<MH04_WEBHOOK_KEY>`.
- **Generic intake (Zalo/hotline/API):** `https://<your-domain>/webhook/mh-lead-intake` (header auth).
- **Ads dashboard:** set the Meta token + Google OAuth/dev-token. The Google Ads REST version is
  pinned in the URL (e.g. `/v21/`); Google sunsets versions yearly — bump it when you get a 404.

## 7. Test before go-live

Send a signed HMAC webhook (test PSIDs prefixed `6200000…`) to exercise the bot without real
traffic; verify a Lead/Deal/CSKH record appears and the Sales alert fires; then delete the test
records by `record_id`. See `docs/MH_PROJECT_HANDOVER.md` and the changelog for the exact procedure.

## 8. Activate

Enable the active workflows. Assign the DLQ (`MH-DLQ`) as the **error workflow** in each workflow's
settings so failures are centralized.

---

### Notes
- Every external call already has retry + `onError: continueRegularOutput`; a flaky dependency won't
  break the chain, and the alert still fires.
- The bot writes to **staging ("Copy") tables** by default so it never touches a live CRM. Re-point
  to your real tables only once you trust the flow.
