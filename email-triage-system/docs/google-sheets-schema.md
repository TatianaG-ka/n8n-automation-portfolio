# Google Sheets schema

Configuration and audit log for the Email Triage System lives in a single Google Sheets spreadsheet with **four tabs**. This document describes each tab, its columns, and the values that need to be in place before workflows are activated.

> **Quick start:** Download [`google-sheets-template.xlsx`](google-sheets-template.xlsx), open in Google Sheets via File → Import, then customize the example values in `Zespół`, `Konfiguracja`, and `Kategorie` to your team and configuration. The `Historia` tab populates automatically.

## Why Sheets, not config files

The four tabs are read by the workflow on every email — so any change in Sheets takes effect on the next email, no n8n redeploy needed. Operators (people without developer access) can:

- Add a new category in `Kategorie` + assign person in `Zespół`
- Reassign team members in `Zespół`
- Tune AI classification by editing `opis_dla_ai` in `Kategorie`
- Change tone of voice by editing `ton_odpowiedzi` in `Kategorie`
- Adjust system limits in `Konfiguracja`

For the architectural rationale: see [ADR-002](https://github.com/TatianaG-ka/n8n-automation-portfolio/blob/main/email-triage-system/docs/design_decisions_ADR.md#adr-002).

---

## Tab 1: `Zespół`

Maps each email category to the person responsible, their Slack identifiers, and the Gmail label to apply.

**Headers (row 1):**

| Column | Field | Required | Purpose |
|---|---|---|---|
| A | `kategoria` | Yes | Category key (must match `Kategorie.klucz`) |
| B | `osoba` | Yes | Person's name (used in Slack DMs) |
| C | `rola` | Yes | Role label (used in notification templates) |
| D | `slack_channel` | Yes | Channel name (without `#`) |
| E | `slack_user_id` | Yes | Slack user ID (`U_XXXXXXX` format, NOT username) |
| F | `gmail_label_id` | Yes | Gmail label ID (`Label_NNNNNN` format, see SETUP Phase 2.2) |
| G | `emoji` | Yes | Emoji prefix in notifications |

**Example rows in template:** 10 rows, one per category. The `kategoria` values must match exactly the keys in `Kategorie.klucz` (case-sensitive).

**For demo / single-user testing:** you can use your own Slack user ID for all 10 rows — DMs will all come to you.

---

## Tab 2: `Konfiguracja`

System-wide parameters: company branding, channel routing for crisis handling, AI input limits.

**Headers (row 1):**

| Column | Field | Required | Purpose |
|---|---|---|---|
| A | `klucz` | Yes | Configuration key |
| B | `wartosc` | Yes | Value |
| C | `opis` | No | Description (helps operators understand each setting) |

**Required keys:**

| Key | Example value | Purpose |
|---|---|---|
| `crisis_manager_name` | Anna Kowalska | Person handling urgent escalations |
| `crisis_manager_role` | Dyrektor / Zarządzanie kryzysowe | Their role label |
| `crisis_slack_id` | U_XXXXXX | Slack user ID for crisis manager |
| `crisis_channel` | pilne | Slack channel for urgent emails |
| `normal_channel` | general | Slack channel for normal notifications |
| `firma_nazwa` | ConsultPro Sp. z o.o. | Company name in draft replies |
| `firma_podpis` | Zespół ConsultPro Sp. z o.o. | Signature in draft replies |
| `body_limit` | 3000 | Max characters of email body sent to AI (token cost control) |
| `subject_limit` | 200 | Max characters of subject sent to AI |
| `draft_max_words` | 150 | Soft cap on draft length |
| `error_notify_channel` | general | Slack channel for error alerts |
| `default_category` | pytanie_klienta | Fallback category when AI returns unknown value (see [four-layer fallback](../README.md#four-layer-ai-parsing-fallback)) |

---

## Tab 3: `Kategorie`

The AI's category list — descriptions used in the classification prompt and tone of voice for draft generation. **This is the single most important tab for AI behavior.**

**Headers (row 1):**

| Column | Field | Required | Purpose |
|---|---|---|---|
| A | `klucz` | Yes | Category key (matches `Zespół.kategoria`) |
| B | `nazwa_pl` | Yes | Display name (used in Slack notifications) |
| C | `emoji` | Yes | Emoji prefix |
| D | `opis_dla_ai` | Yes | **Description used IN the AI classification prompt** — descriptive triggers per category |
| E | `ton_odpowiedzi` | Yes | Tone of voice for draft generation per category |

**Why this matters for AI behavior:**

- **`opis_dla_ai` is literally part of the AI prompt.** The categorize chain injects all 10 descriptions at runtime — operator changes the description, AI behavior changes on the next email. No prompt redeploy.
- **`ton_odpowiedzi` flows into the draft generation prompt.** Tones supported: `formalny`, `empatyczny`, `entuzjastyczny`, `pomocny`, `profesjonalny` (and combinations). v2 candidate prompt maps tones to specific linguistic moves — see [`prompts/generate-draft-v2-candidate.md`](prompts/generate-draft-v2-candidate.md#tone-odpowiedzi).

**Example rows in template:** 10 rows covering the full classifier vocabulary. Each row pairs a Polish business email category with both an AI-trigger description and a default response tone.

**Tuning guidance:**

- If AI misclassifies a category → make `opis_dla_ai` more specific (add distinctive keywords, narrow scope)
- If draft tone feels off → adjust `ton_odpowiedzi` (e.g., `formalny, ostrożny` instead of just `formalny`)
- After tuning, monitor `_parseError` rate in execution logs

---

## Tab 4: `Historia`

Audit log of every email classification + every approval decision. Auto-populated by the workflow — operators do not edit this tab.

**Headers (row 1):**

| Column | Field | Populated by | Purpose |
|---|---|---|---|
| A | `data` | Workflow on each email | ISO timestamp |
| B | `email_nadawcy` | Workflow | Sender email address |
| C | `kategoria` | Workflow | Assigned category (post-fallback if AI failed) |
| D | `temat` | Workflow | Email subject |
| E | `decyzja` | Workflow on approval | One of: `Zatwierdzono`, `Poprawiono`, `Odrzucono`, `Timeout` |
| F | `przypisano` | Workflow | Person from `Zespół.osoba` who handled it |

**Why this audit trail matters:**

In a regulated business context (NDAs, corporate clients, legal matters), being able to answer "who saw this email, who decided what, when" is non-negotiable. The `Historia` tab is the answer — every decision has a row, every row has a timestamp and accountable person.

For analysis, the tab can be queried with Sheets formulas:

```
// How many emails per category last quarter
=COUNTIFS(C:C, "reklamacja_eskalacja", A:A, ">2026-01-01", A:A, "<2026-04-01")

// What % of approvals were modified before sending
=COUNTIF(E:E, "Poprawiono") / COUNTA(E:E)
```

For higher-volume installations, consider exporting `Historia` to BigQuery or PostgreSQL on a schedule — see [v7 roadmap](../README.md#roadmap-v7).

---

## Schema validation

Before activating workflows, verify:

- [ ] All 4 tabs exist with exact names: `Zespół`, `Konfiguracja`, `Kategorie`, `Historia` (case-sensitive, with Polish character `ę` in `Zespół`)
- [ ] Headers in row 1 match exactly (column letters and field names per tables above)
- [ ] All 10 categories in `Kategorie.klucz` have corresponding rows in `Zespół.kategoria`
- [ ] All `slack_user_id` values use `U_XXXXX` format (Slack user ID), NOT usernames
- [ ] All `gmail_label_id` values use `Label_NNNNNN` format from Gmail Labels API
- [ ] `Konfiguracja.crisis_slack_id` and `default_category` are set
- [ ] Spreadsheet is shared with the Google account authenticated in n8n

If any of the 24 end-to-end tests fail with `Cannot find Sheet` or `Unknown category`, this validation is the first thing to re-check.

---

## Files

- [`google-sheets-template.xlsx`](google-sheets-template.xlsx) — Pre-filled template with example team and full category list. Import to Google Sheets via File → Import.

## Related documentation

- [Setup guide](SETUP.md) — Phase 1 walks through creating the Sheet from scratch
- [ADR-002](design_decisions_ADR.md#adr-002) — Architectural rationale for config-as-data
- [`prompts/categorize-v1-production.md`](prompts/categorize-v1-production.md) — How `Kategorie.opis_dla_ai` is injected into the AI prompt
- [`prompts/generate-draft-v1-production.md`](prompts/generate-draft-v1-production.md) — How `Kategorie.ton_odpowiedzi` flows into draft generation
