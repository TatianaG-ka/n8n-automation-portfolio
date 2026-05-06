# Test cases — Email Triage v6.1

Test data: [`../../test_emails.json`](../../test_emails.json) (24 emails)
Workflow: `Email Triage - v6.1 PRODUCTION`

This document is the **executable test specification** — it lists every test case, expected category, expected urgency, expected routing, and what to verify in each downstream system. Pair it with [`test_results_2026-04-08.md`](test_results_2026-04-08.md) for the most recent execution baseline.

> **Note on test data:** Test cases use placeholder names matching [`docs/google-sheets-template.xlsx`](../google-sheets-template.xlsx) (Piotr Nowak, Marta Wiśniewska, Jan Zieliński, Karolina Dąbrowska, Anna Kowalska as crisis manager). To run these tests against your own deployment, your `Zespół` Sheet should map categories to your actual team — expected routing in this document will then point to your team members instead.

## How to test

1. Open the workflow in n8n
2. Click **Test workflow** (manual run)
3. In Gmail Trigger node, use **Test step** with pinned data — paste an object from `test_emails.json`
4. Verify each node's output against the expectations below

---

## Standard tests (10 categories)

| # | ID | Subject | Expected category | Expected urgency | Assigned person | Verification |
|---|---|---|---|---|---|---|
| 1 | TEST001 | Inquiry about consulting services for company ABC | `zapytanie_ofertowe` | normalne | Piotr Nowak | Gmail label, draft with enthusiastic tone |
| 2 | TEST002 | Re: Optimization project — work status | `pytanie_klienta` | normalne | Marta Wiśniewska | Helpful, concrete draft |
| 3 | TEST004 | Invoice FV/2026/03/045 — correction request | `sprawy_fakturowe` | normalne | Jan Zieliński | Formal, precise draft |
| 4 | TEST005 | Cooperation proposal — consortium | `wspolpraca_partnerstwo` | normalne | Piotr Nowak | Open, professional draft |
| 5 | TEST006 | Job application — Strategy Consultant | `rekrutacja_cv` | normalne | Karolina Dąbrowska | Draft with next steps info |
| 6 | TEST007 | Invitation: Conference 2026 | `zaproszenie_wydarzenie` | normalne | Karolina Dąbrowska | Polite draft with thank-you |
| 7 | TEST008 | NDA for signature — project Delta | `sprawy_prawne_umowy` | normalne | Jan Zieliński | Formal, cautious draft |
| 8 | TEST009 | Automated confirmation #12345 | `wiadomosc_systemowa` | normalne | Karolina Dąbrowska | Draft: suggest no reply |
| 9 | TEST010 | MEGA DEAL — CRM system for 1 zł! | `spam` | normalne | Karolina Dąbrowska | Draft: "no reply (spam)" |
| 10 | TEST020 | Late report delivery | `reklamacja_eskalacja` | normalne | Marta Wiśniewska | Empathetic, apologetic draft |

---

## Urgent flag tests

| # | ID | Subject | Expected category | Why urgent | Verification |
|---|---|---|---|---|---|
| 11 | TEST003 | URGENT — Dissatisfaction with report | `reklamacja_eskalacja` | Legal threat, "by tomorrow" deadline | Slack `#pilne`, Gmail star, DM to Marta + crisis DM to Anna Kowalska |
| 12 | TEST011 | OUTAGE — Reporting system not working | `reklamacja_eskalacja` | Service outage, "immediate intervention" | Slack `#pilne`, crisis DM, draft with apology |
| 13 | TEST012 | URGENT — Missing payment, final notice | `sprawy_fakturowe` | Debt collection, deadline | Slack `#pilne`, DM to Jan Zieliński |
| 14 | TEST016 | Pre-litigation notice — contract breach | `sprawy_prawne_umowy` | Legal threat, "immediately" | Slack `#pilne`, crisis DM, cautious draft |

---

## Edge cases

| # | ID | What it tests | Expected outcome | What to watch for |
|---|---|---|---|---|
| 15 | TEST013 | No subject, informal tone, single sentence | `pytanie_klienta` / normalne | `Validate Email`: subject = `(brak tematu)`. AI handles short body |
| 16 | TEST014 | Forwarded spam / chain mail | `spam` / normalne | AI recognizes spam despite "Fwd: Fwd:" in subject |
| 17 | TEST015 | Nested replies with quoted text | `zapytanie_ofertowe` / normalne | AI extracts content from "> >" quoted blocks |
| 18 | TEST017 | Out of Office autoresponder | `wiadomosc_systemowa` / normalne | Draft: suggest no reply needed |
| 19 | TEST018 | Multi-topic (scope change + workshop invitation) | `pytanie_klienta` / normalne | AI identifies primary topic. Draft addresses both |
| 20 | TEST019 | Newsletter (subtle spam) | `spam` / normalne | AI distinguishes newsletter from valuable email. "unsubscribe" in body |

---

## New scenarios (v6)

| # | ID | What it tests | Expected outcome | What to watch for |
|---|---|---|---|---|
| 21 | TEST021 | English-language email | `zapytanie_ofertowe` / normalne | AI categorizes correctly despite foreign language. Draft must be **in Polish** |
| 22 | TEST022 | "Invoice attached as PDF" reference | `sprawy_fakturowe` / normalne | Category recognition from body text without attachment analysis |
| 23 | TEST023 | Single-word body: "Pilne", no subject | `pytanie_klienta` / **pilne** | Minimal input. Keyword fallback should detect "pilne". subject = `(brak tematu)` |
| 24 | TEST024 | Noreply with large HTML payload | `wiadomosc_systemowa` / normalne | HTML stripping in `Validate Email`. Stripped body: "Account update..." |

---

## Error handling tests (manual simulation required)

These tests require **deliberately triggering errors** to verify Error Handler workflow behavior:

| # | Scenario | How to simulate | Expected outcome |
|---|---|---|---|
| 25 | OpenAI doesn't respond | Set API key to invalid value | Keyword fallback activates. Slack warning "AI fallback". Error Handler alert |
| 26 | Gmail label doesn't exist | Set non-existent label ID in Sheets | Flow continues (`onError: continueRegularOutput`). Warning in Format Notification |
| 27 | Slack credentials missing | Don't configure Slack OAuth | `retryOnFail` 3× then `onError`. Draft already in Gmail |
| 28 | Email duplicate | Run workflow 2× with same email | Second run: `Validate Email` returns `[]` (skipped via idempotency cache) |
| 29 | Google Sheets unavailable | Set Sheet ID to invalid value | `retryOnFail` 3×. Error Handler: Slack + Email backup alert |
| 30 | Approval timeout | Send approval email, don't click for 2 days | Slack timeout alert after `Wait` node fires |

---

## Approval flow tests

| # | Scenario | How to test | Expected outcome |
|---|---|---|---|
| 31 | Draft approved | Click **✅ Zatwierdź i wyślij** in approval email | Gmail reply sent to client. Slack: "Draft approved". `Historia` log: `decyzja=zatwierdz` |
| 32 | Draft needs revision | Click **✏️ Do poprawki** | Slack DM to assignee: "edit draft in Gmail". `Historia` log: `decyzja=popraw` |
| 33 | Draft rejected | Click **❌ Odrzuć** | Slack `#general`: "Draft rejected". `Historia` log: `decyzja=odrzuc`. No reply to client |

---

## Coverage matrix

```
Categories:        10/10 (100%)
Urgency split:     pilne: 5 tests, normalne: 19 tests
Edge cases:        12 scenarios
Error handling:    6 simulations
Approval flow:     3 scenarios + timeout
---
Total test cases:  33
```

## Most recent execution baseline

24 automated tests (categories + urgency + edge cases + v6 new scenarios) executed on 2026-04-08 — see [`test_results_2026-04-08.md`](test_results_2026-04-08.md).

The remaining 9 tests (error handling 25-30, approval flow 31-33) require manual simulation per [`SETUP.md` Phase 8](../SETUP.md#phase-8--first-test).
