# Test results — Email Triage v6.1

**Date:** 2026-04-08
**Workflow:** `Email Triage - v6.1 PRODUCTION` (`WORKFLOW_ID_MAIN`)
**Method:** Webhook trigger (temporary, for test execution) + n8n MCP `n8n_test_workflow`

> **Sanitization note:** Workflow IDs and Gmail message IDs are sanitized for public publication. Real n8n internal IDs are replaced with placeholder format (`WORKFLOW_ID_*`). Gmail message IDs preserve format prefix (`19d6XXXXXXXXXXXX`) to show structure without exposing actual identifiers.

---

## Summary

**24/24 tests passed (100% pass rate)**

- 0 workflow errors
- 0 timeouts
- Average execution time: ~30s per test

---

## Detailed results

### Standard tests (10 categories)

| # | ID | Subject | Status | Time | Gmail ID |
|---|---|---|---|---|---|
| 1 | TEST001 | Inquiry about consulting services | OK 200 | 34.8s | 19d6XXXXXXXXXXXX |
| 2 | TEST002 | Re: Optimization project — work status | OK 200 | 30.3s | 19d6XXXXXXXXXXXX |
| 3 | TEST003 | URGENT — Dissatisfaction with report | OK 200 | 34.8s | 19d6XXXXXXXXXXXX |
| 4 | TEST004 | Invoice FV/2026/03/045 — correction request | OK 200 | 35.0s | 19d6XXXXXXXXXXXX |
| 5 | TEST005 | Cooperation proposal — consortium | OK 200 | 31.7s | 19d6XXXXXXXXXXXX |
| 6 | TEST006 | Job application — Strategy Consultant | OK 200 | 30.5s | 19d6XXXXXXXXXXXX |
| 7 | TEST007 | Invitation: Conference 2026 | OK 200 | 31.2s | 19d6XXXXXXXXXXXX |
| 8 | TEST008 | NDA for signature — project Delta | OK 200 | 29.9s | 19d6XXXXXXXXXXXX |
| 9 | TEST009 | Automated confirmation #12345 | OK 200 | 27.3s | 19d6XXXXXXXXXXXX |
| 10 | TEST010 | MEGA DEAL — CRM system for 1 zł! | OK 200 | 25.8s | 19d6XXXXXXXXXXXX |

### Urgent flag tests

| # | ID | Subject | Status | Time | Gmail ID |
|---|---|---|---|---|---|
| 11 | TEST011 | OUTAGE — Reporting system not working | OK 200 | 32.8s | 19d6XXXXXXXXXXXX |
| 12 | TEST012 | URGENT — Missing payment, final notice | OK 200 | 29.5s | 19d6XXXXXXXXXXXX |
| 13 | TEST016 | Pre-litigation notice — contract breach | OK 200 | 31.3s | 19d6XXXXXXXXXXXX |

### Edge cases

| # | ID | What it tests | Status | Time | Gmail ID |
|---|---|---|---|---|---|
| 14 | TEST013 | No subject, informal tone, single sentence | OK 200 | 29.0s | 19d6XXXXXXXXXXXX |
| 15 | TEST014 | Forwarded spam / chain mail | OK 200 | 25.5s | 19d6XXXXXXXXXXXX |
| 16 | TEST015 | Nested replies with quoted text | OK 200 | 30.3s | 19d6XXXXXXXXXXXX |
| 17 | TEST017 | Out of Office autoresponder | OK 200 | 26.8s | 19d6XXXXXXXXXXXX |
| 18 | TEST018 | Multi-topic (scope change + workshop invitation) | OK 200 | 37.2s | 19d6XXXXXXXXXXXX |
| 19 | TEST019 | Newsletter (subtle spam) | OK 200 | 28.0s | 19d6XXXXXXXXXXXX |
| 20 | TEST020 | Late report delivery notification | OK 200 | 30.5s | 19d6XXXXXXXXXXXX |

### New scenarios (v6)

| # | ID | What it tests | Status | Time | Gmail ID |
|---|---|---|---|---|---|
| 21 | TEST021 | English-language email through Polish workflow | OK 200 | 28.2s | 19d6XXXXXXXXXXXX |
| 22 | TEST022 | "Invoice attached as PDF" reference | OK 200 | 26.8s | 19d6XXXXXXXXXXXX |
| 23 | TEST023 | Single-word body: "Pilne", no subject | OK 200 | 26.6s | 19d6XXXXXXXXXXXX |
| 24 | TEST024 | Noreply with large HTML payload | OK 200 | 24.7s | 19d6XXXXXXXXXXXX |

---

## What was verified

Each test passed the full end-to-end pipeline:

1. **Webhook test trigger** → input data (simulating Gmail Trigger)
2. **Load Zespol / Konfiguracja / Kategorie** → configuration loaded from Google Sheets
3. **Validate Email** → sanitization, HTML strip, truncation, idempotency check
4. **AI Categorize Email** → gpt-4o-mini + Structured Output Parser
5. **Validate & Route** → AI output parsing + four-layer fallback + person assignment
6. **Gmail — Add Label** → category label applied
7. **AI Generate Draft** → reply draft generation
8. **Gmail — Create Draft** → draft saved in Gmail
9. **Format Notification** → Slack message preparation
10. **Is Urgent? → Slack** → notification routed to correct channel (urgent vs normal)
11. **Prepare Slack DM → Slack Worker DM** → DM sent to assigned person
12. **Gmail — Mark as Read** → email marked as read (last step, after all side effects)
13. **Prepare Approval Email → Gmail Send Approval** → approval email with action buttons (urgent emails only)

## Fix during testing

**Issue:** n8n webhook wraps POST body in a `body` property, causing `email.body` in `Validate Email` node to receive an object instead of a string (`trim is not a function`).

**Fix:** Temporary modification in `Validate Email` — unwrap `wi.json.body` for webhook-sourced data. Original code restored after test execution. This applies only to the test setup using webhook trigger; production runs use Gmail Trigger which returns body as string directly.

## Notes

- Tests verify **pipeline correctness** (workflow executes end-to-end without errors)
- Verification of **AI classification accuracy** and **draft content quality** requires manual review:
  - Gmail Drafts folder: 24 newly created drafts
  - Slack: notifications across channels (`#pilne`, `#general`, plus DMs)
  - Google Sheets `Historia` tab: audit log entries (where approval was used)
- Error handling tests (suite IDs 25-30) and approval flow tests (suite IDs 31-33) require manual simulation — see [`test_cases.md`](test_cases.md) for the full specification of all 33 test cases including manual scenarios

---

## Coverage matrix

```
Standard tests (10 categories):  10/10 OK
Urgent flag tests:                3/3  OK (TEST003 included in standard set)
Edge cases:                       7/7  OK
v6 new scenarios:                 4/4  OK
---
Total automated:                 24/24 OK (100%)
Manual tests (error/approval):    9 to be executed manually
```

---

## How to reproduce

To re-run the test suite against your own deployment:

1. Complete setup per [`SETUP.md`](../SETUP.md)
2. Use test data from [`../../test_emails.json`](../../test_emails.json)
3. Trigger via n8n MCP `n8n_test_workflow` or Webhook trigger (temporary node)
4. Verify results match the table above (same 24 categories, similar execution times)

Significant deviations from this baseline (>2 failures, >50s average execution) indicate configuration issues — see [`SETUP.md` Troubleshooting](../SETUP.md#troubleshooting) for diagnostic steps.
