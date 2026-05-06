# SETUP — Email Triage System

Step-by-step guide for importing and running the Email Triage workflows from scratch.

---

## Prerequisites

You'll need accounts and access to:

- [ ] **n8n instance** — self-hosted (Hostinger, Railway, Docker, fly.io) or n8n Cloud. Tested on n8n version `1.x` (any recent version works).
- [ ] **Gmail account** with Inbox you want to triage. Must be able to grant OAuth2 access.
- [ ] **Google account** with Sheets access (can be same as Gmail).
- [ ] **Google Drive access** — for Extended workflow attachment archiving.
- [ ] **OpenAI API key** with billing enabled. Estimated cost: ~$1.50/month for 20 emails/day on `gpt-4o-mini`.
- [ ] **Slack workspace** with admin access (to install custom Slack app).
- [ ] **Trello account** with API access — for Extended workflow.
- [ ] (Optional) GitHub account if you want to fork/clone this repo.

---

## Phase 1 — Google Sheets setup

The system reads its configuration from a single Google Spreadsheet with 4 tabs. You can either use the pre-filled template or create the tabs manually. Schema reference: [`google-sheets-schema.md`](google-sheets-schema.md).

### 1.1 Create the spreadsheet

1. Go to [sheets.google.com](https://sheets.google.com)
2. Create new spreadsheet, name it **`Baza_danych`** (or anything you prefer — you'll reference it by ID)
3. Note the **Spreadsheet ID** from URL: `https://docs.google.com/spreadsheets/d/{SPREADSHEET_ID}/edit`

### 1.2 Create 4 tabs

**Recommended:** download [`google-sheets-template.xlsx`](google-sheets-template.xlsx), then in Google Sheets click **File → Import → Upload**, select the template. All 4 tabs and 10 example category rows are pre-filled — you only customize values to your team.

**Manual alternative** — minimum schema for each tab:

- **`Zespół`** (note the `ę`) — 7 columns: `kategoria | osoba | rola | slack_channel | slack_user_id | gmail_label_id | emoji`. One row per category (10 rows total)
- **`Konfiguracja`** — 3 columns: `klucz | wartosc | opis`. ~12 required keys (see schema doc)
- **`Kategorie`** — 5 columns: `klucz | nazwa_pl | emoji | opis_dla_ai | ton_odpowiedzi`. One row per category (10 rows). Critical: `Kategorie.klucz` values must match `Zespół.kategoria` exactly
- **`Historia`** — 6 columns: `data | email_nadawcy | kategoria | temat | decyzja | przypisano`. Leave rows 2+ empty — workflow auto-populates

> **Tip — for demo / testing:** you can use your own Slack user ID for all rows in `Zespół` — DMs will all come to you.

> **Tip — `opis_dla_ai` is literally part of the AI classification prompt.** Edit it to tune classification behavior without touching the workflow. Same for `ton_odpowiedzi` and draft generation.

### 1.3 Share the spreadsheet

Make sure your Google account that will authenticate n8n has **edit access** to this spreadsheet.

---

## Phase 2 — Gmail setup

### 2.1 Create 10 Gmail labels

In Gmail web UI:

1. Settings (⚙️) → See all settings → Labels tab
2. Click "Create new label"
3. Create one label per category (matching `Zespół.kategoria`):
   - `Zapytanie_ofertowe`
   - `Pytanie_klienta`
   - `Reklamacja_eskalacja`
   - `Sprawy_fakturowe`
   - `Wspolpraca_partnerstwo`
   - `Rekrutacja_cv`
   - `Zaproszenie_wydarzenie`
   - `Sprawy_prawne_umowy`
   - `Wiadomosc_systemowa`
   - `Spam_email`

### 2.2 Get Gmail Label IDs

Gmail labels have IDs like `Label_3381030360629276002`. To get them:

**Option A (n8n UI):**
1. In n8n, create a temporary Gmail node with operation "Get All Labels"
2. Run it, copy the IDs from output

**Option B (Gmail API directly):**
1. Go to [Google API Explorer for Gmail](https://developers.google.com/gmail/api/reference/rest/v1/users.labels/list)
2. Authenticate with your Gmail account
3. Set `userId: me`
4. Click "Execute"
5. Copy IDs from response

Paste IDs into `Zespół.gmail_label_id` column.

---

## Phase 3 — Slack setup

### 3.1 Create Slack app

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → "Create New App" → "From scratch"
2. Name it (e.g., "Email Triage Bot"), pick your workspace
3. Under "OAuth & Permissions", add these **Bot Token Scopes**:
   - `chat:write` (send messages to channels)
   - `chat:write.public` (post to channels without joining)
   - `im:write` (send DMs)
   - `users:read` (resolve user IDs)
   - `channels:read` (resolve channel IDs)
4. Click "Install to Workspace" → authorize

### 3.2 Get Bot User OAuth Token

After installation, "OAuth & Permissions" page shows **Bot User OAuth Token** starting with `xoxb-`. Copy it — you'll paste into n8n.

### 3.3 Create channels

In your Slack workspace, create:
- `#pilne` (private or public, your choice)
- `#general` (probably already exists)

### 3.4 Get channel IDs

In Slack desktop app: right-click channel → "Open channel details" → bottom shows `Channel ID` like `C0ARMH3Q85A`. Paste into `Konfiguracja.crisis_channel` and `normal_channel`.

### 3.5 Get user IDs for team

For each team member: click their profile → "View full profile" → `...` menu → "Copy member ID". Paste into `Zespół.slack_user_id`.

---

## Phase 4 — OpenAI setup

1. Go to [platform.openai.com/api-keys](https://platform.openai.com/api-keys)
2. Create new secret key, name it (e.g., "n8n-email-triage")
3. Copy the key (starts with `sk-`) — you'll only see it once
4. Make sure billing is enabled and you have credits

> **Cost estimate:** for 20 emails/day with `gpt-4o-mini`, ~$1.50/month. Set a hard limit if worried.

---

## Phase 5 — Trello setup (Extended workflow only)

Skip this if you only want the main workflow.

1. Go to [trello.com/app-key](https://trello.com/app-key) → copy your **API Key**
2. On same page, click "Generate a Token" → authorize → copy **Token**
3. In your Trello board, create a list called "Do zrobienia" (or any name)
4. Get the list ID:
   - Open board in browser, append `.json` to URL: `https://trello.com/b/BOARDID.json`
   - Search for your list name in the response, copy `id`
   - Paste into workflow JSON: `Trello - Create Card` node → `listId` parameter

---

## Phase 6 — n8n credentials

In n8n UI, go to **Credentials** (left sidebar) and create:

### 6.1 Gmail OAuth2

1. Create credential → "Gmail OAuth2 API"
2. Click "Connect my account" → authorize Gmail
3. Save. n8n stores tokens automatically.

### 6.2 Google Sheets OAuth2

1. Create credential → "Google Sheets OAuth2 API"
2. Connect, authorize.

### 6.3 Google Drive OAuth2 (for Extended)

1. Create credential → "Google Drive OAuth2 API"
2. Connect, authorize.

> **Tip:** Gmail/Sheets/Drive can all use the same Google account — credentials are separate per API in n8n.

### 6.4 OpenAI API

1. Create credential → "OpenAI API"
2. Paste your `sk-...` key
3. Save.

### 6.5 Slack OAuth2

1. Create credential → "Slack OAuth2 API"
2. Paste **Bot User OAuth Token** (`xoxb-...`)
3. Save.

### 6.6 Trello API (for Extended)

1. Create credential → "Trello API"
2. Paste API Key + Token
3. Save.

---

## Phase 7 — Import workflows

> **Note:** JSON files in [`../workflows/`](../workflows/) are sanitized — they use placeholder values like `your-admin@example.com`, `YOUR_GOOGLE_SHEET_ID`, `WORKFLOW_ID_ERROR_HANDLER`. See [`workflows/README.md`](../workflows/README.md) for the full list of placeholders to replace during this phase.

### 7.1 Import Error Handler first

This must exist before you can bind it as `errorWorkflow` to the others.

1. In n8n, click "Workflows" → "+ Add" → "Import from File"
2. Upload `03-email-triage-error-handler.json`
3. Open imported workflow, **note the workflow ID** from URL: `https://your-n8n.com/workflow/{WORKFLOW_ID}`
4. Click "Save"
5. Update credentials in nodes: Slack alert + Gmail backup

### 7.2 Import Main workflow

1. Import `01-email-triage-main.json`
2. **Update Sheets references:**
   - In all 4 Google Sheets nodes (`Load Zespol`, `Load Config`, `Load Kategorie`, `GSheets - Log Decision`), update `documentId.value` to **YOUR Spreadsheet ID** from Phase 1
3. **Update credentials:**
   - Gmail Trigger → your Gmail OAuth
   - All Gmail nodes → your Gmail OAuth
   - All Sheets nodes → your Sheets OAuth
   - OpenAI nodes → your OpenAI credential
   - All Slack nodes → your Slack OAuth
4. **Update channel IDs:**
   - `Slack - URGENT Channel` → channelId.value to your `#pilne` channel ID (e.g., `C0ARMH3Q85A`)
   - `Slack - Normal Channel` → uses `_config.normal_channel` from Sheets, no manual update needed
5. **Update Gmail send target:**
   - `Gmail - Send Approval` → `sendTo` parameter → use `={{ $json.assignedEmail || 'your-admin@example.com' }}`
6. **Bind error workflow:**
   - Workflow Settings (gear icon) → "Error Workflow" → select the Error Handler from 7.1
7. Save.

### 7.3 Import Extended workflow

1. Import `02-email-triage-extended.json`
2. Update credentials (same as above + Trello + Drive)
3. **Update Sheets references** in all GSheets nodes — your Spreadsheet ID
4. **Update Trello list ID:**
   - `Trello - Create Card` node → `listId` → your list ID from Phase 5
5. **Update Drive folder strategy:**
   - `GDrive: Utwórz folder` → set parent folder for client folders
6. **Bind error workflow** (same Error Handler)
7. **Note webhook URL** of `Webhook - From Main Flow` node — you'll need it
8. Save.

### 7.4 Connect main → extended (optional)

If you want main to trigger extended:

1. In Main workflow, after `GSheets - Log Decision`, add HTTP Request node
2. Method: POST, URL: webhook URL from 7.3 step 7
3. Body: send relevant fields from main flow context

---

## Phase 8 — First test

### 8.1 Pin test data

1. Open Main workflow
2. Right-click `Gmail Trigger` node → "Edit pinned data"
3. Paste this test mail JSON:

```json
[{
  "id": "TEST001",
  "threadId": "thread_test001",
  "subject": "Zapytanie o usługi doradcze",
  "body": "Dzień dobry, chciałbym zapytać o zakres oferowanych przez Państwa usług doradczych oraz orientacyjne ceny. Pozdrawiam, Jan Testowy",
  "from": "jan.testowy@example.com",
  "fromName": "Jan Testowy",
  "date": "2026-05-01T10:00:00Z"
}]
```

### 8.2 Execute

1. Click "Execute Workflow" (top right)
2. Watch the execution unfold node-by-node
3. Verify each step:
   - Sheets load successfully (3× green)
   - Validate Email passes
   - AI classifies (check output: should be `zapytanie_ofertowe`)
   - Route assigns to correct person from Zespół
   - Gmail label added (check Gmail UI)
   - Draft created (check Gmail Drafts)
   - Slack notification arrived (check #pilne or #general)
   - DM arrived (check yourself)
   - Mark as read OK

### 8.3 Test idempotency

Run the same execution again. Console should show:
```
[Validate Email] Skipped 1 duplicate messageId(s) via idempotency cache
```
The mail is NOT re-processed — second run produces no Slack DM, no draft, no anything.

### 8.4 Test approval flow

Send another test email. When approval email arrives in your inbox:

1. Click ✅ Zatwierdź button
2. Check that workflow resumes (n8n executions)
3. Verify Gmail Reply was sent (check Sent folder)
4. Verify Sheets Historia got new row

### 8.5 Test timeout

Optional but recommended:

1. Temporarily change `Wait - Decision` `resumeAmount` to `5` and `resumeUnit` to `minutes`
2. Send test email, do NOT click any button
3. Wait 5 minutes
4. Verify Slack Timeout Alert fires in #general
5. Restore `resumeAmount: 2, resumeUnit: days`

### 8.6 Test error handling

1. Temporarily change OpenAI credential to invalid key
2. Send test email
3. Verify Error Handler fires:
   - Slack alert in #general with severity 🔴 HIGH
   - Email backup arrives in your inbox
4. Restore OpenAI key

---

## Phase 9 — Activate

Once all tests pass:

1. Open Main workflow → top-right toggle → **Active**
2. Open Extended workflow → toggle → **Active**
3. Error Handler workflow does NOT need to be active — it's triggered on demand via `errorWorkflow` setting.

### Monitoring first 24h

- Check n8n Executions tab every few hours
- Verify successful executions (green) outnumber failures (red)
- Check Slack #general for any unexpected error alerts
- Check Gmail Drafts folder — drafts should be appearing for each new email

---

## Troubleshooting

### Workflow doesn't trigger on new emails

- Check Gmail Trigger is **Active**
- Check OAuth still valid (re-authenticate if expired)
- Check pollTimes interval (default 1 minute)
- Check Gmail filters — make sure mail isn't being auto-archived before trigger sees it

### AI returns weird categories

- Check `Kategorie.opis_dla_ai` text — is it descriptive enough?
- Check `Konfiguracja.default_category` — what's your fallback?
- Look at execution log for `_parseError: true` flag — indicates AI fallback fired

### Slack messages not sending

- Verify Bot User OAuth Token is `xoxb-...` not `xoxp-...` (user token won't work for bot operations)
- Check bot is invited to channels (`/invite @YourBot` in #pilne and #general)
- Check `Zespół.slack_user_id` values are correct format (`U0XXXXX`, not username)

### "Cannot find Sheet" errors

- Verify `documentId.value` in all Sheets nodes matches your Spreadsheet ID
- Verify sheet tab names match exactly (case-sensitive with Polish character: `Zespół` not `Zespol` or `zespol`)
- Verify Google account authenticated in n8n has access to spreadsheet

### Approval buttons don't work

- Test webhook URL by visiting it manually with `?action=test` — should return 200
- Check `$execution.resumeUrl` is not blocked by your reverse proxy
- Verify `Wait - Decision` is set to `resume: webhook`

### Idempotency seems to skip all emails

- `$getWorkflowStaticData('global').processedIds` may have grown corrupted
- Reset: in n8n shell or temporarily add Code node `$getWorkflowStaticData('global').processedIds = []` and run once

---

## Cost monitoring

Track these monthly:

| Item | Estimate at 20 mails/day | How to check |
|---|---|---|
| OpenAI API | ~$1.50/month | platform.openai.com → Usage |
| Google Sheets API | $0 (free tier) | console.cloud.google.com |
| Gmail API | $0 (free tier) | console.cloud.google.com |
| Slack | $0 (free tier OK) | n/a |
| Trello | $0 (free tier OK) | n/a |
| n8n hosting (Hostinger) | ~$10/month | Hostinger billing |

**Total: ~$12/month** for full system.

---

## Maintenance

**Weekly:**
- Check Sheets `Historia` for any unusual decision patterns
- Check n8n Executions for spikes in failures

**Monthly:**
- Review keyword fallback hit rate (count of `_parseError: true` in execution logs) — if >5% of mails, time to tune AI prompt
- Review approval timeout rate — if many timing out, maybe extend or reduce categories using approval

**As needed:**
- Add/edit categories — just add row to `Kategorie` + `Zespół`
- Change team — edit `Zespół`, no deploy
- Update prompts — edit `Kategorie.opis_dla_ai`, no deploy
- Update tone of voice — edit `Kategorie.ton_odpowiedzi`, no deploy

---

## Going further

After basic setup works, consider:

- Set up monitoring dashboard (Grafana + n8n metrics endpoint)
- Add Sentry integration for production error tracking
- Implement v7 features from roadmap (see [README.md](../README.md#roadmap-v7))
- Run [test_emails.json](../test_emails.json) suite weekly as health check

---

## Help

If something doesn't work after following this guide:

1. Check [Real n8n gotchas](../README.md#real-n8n-gotchas-with-workarounds) in main README for known issues
2. Check [design-decisions.md](design-decisions.md) for "why" context
3. Open an issue on GitHub
4. Contact the maintainer (see [main README](../README.md#contact) for current contact details)
