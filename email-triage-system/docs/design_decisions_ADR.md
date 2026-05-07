# Architecture Decision Records — Email Triage System

This document captures the key architectural decisions made during the development of the Email Triage workflow system. Each decision follows the **ADR (Architecture Decision Record)** format: context, decision, consequences, alternatives considered.

ADRs are immutable historical records. If a decision is later reversed, a new ADR supersedes the old one rather than editing it.

---

## Index

| # | Title | Status |
|---|---|---|
| [ADR-001](#adr-001) | Use Structured Output Parser with 4-layer fallback for AI classification | Accepted  |
| [ADR-002](#adr-002) | Store all configuration as data in Google Sheets, not in workflow code | Accepted |
| [ADR-003](#adr-003) | Idempotency via `$getWorkflowStaticData` cache of last 500 messageIds | Accepted  |
| [ADR-004](#adr-004) | Two-stage send: draft default, send-on-approval opt-in | Accepted |
| [ADR-005](#adr-005) | Mark email as read at end of flow, not at start | Accepted |
| [ADR-006](#adr-006) | Dedicated error workflow with dual-channel alerting (Slack + Gmail) | Accepted |
| [ADR-007](#adr-007) | Differentiated retry strategy per service | Accepted |
| [ADR-008](#adr-008) | Approval flow integrated in main workflow, not separate workflow | Accepted |
| [ADR-009](#adr-009) | Serial Sheets load instead of parallel | Accepted  |
| [ADR-010](#adr-010) | Single `Validate and Route` Code node instead of two separate nodes | Accepted |

---

## ADR-001
### Use Structured Output Parser with 4-layer fallback for AI classification

**Status:** Accepted   
**Date:** 2026-03 (v6 design), revised 2026-04 (v6.1 — semantic graceful degradation added as separate layer)

#### Context

The workflow needs to classify each incoming email into one of 10 predefined categories with reasonable accuracy. Two failure modes are unacceptable:

1. **Hard fail** — workflow stops, mail goes unprocessed, user notices nothing
2. **Silent corruption** — AI returns garbage, workflow proceeds with bad data, mail routed to wrong person

LLMs are notoriously unpredictable: they sometimes ignore explicit instructions to "return only JSON" and add markdown backticks, conversational text, or invent new categories. A naive `JSON.parse()` on raw output will fail unpredictably.

Critically, **AI failures are not all the same kind**. The output can be:
- **Structurally malformed** — invalid JSON, markdown fences, conversational prose
- **Semantically wrong** — valid JSON, but with a category outside the allowed enum (LLM hallucinates a category that wasn't in the prompt)

These two failure modes need different recovery strategies.

#### Decision

Use **four layers of resilience** for AI output parsing, each handling a distinct type of failure:

```
Layer 1 (LLM-level, structural success): LangChain Structured Output Parser
  → Enforces JSON schema with explicit field types at LLM call boundary
  → Output: structured object with category/urgency/summary/key_entities/urgency_reason
  → Catches: most malformed outputs at the source

Layer 2 (Code-level, structural recovery): JS parser handles 4 output formats
  → aiOutput.output as object (Structured Parser worked perfectly)
  → aiOutput.output as string (parser returned string with markdown fences)
  → aiOutput.text as string (alternate output property when parser is bypassed)
  → aiOutput.category exists directly (raw response without wrapper)
  → Strips ```json...``` markdown fences, calls JSON.parse with cleanup

Layer 3 (Code-level, semantic graceful degradation): Category enum validation
  → Check `validCategories.includes(aiResult.category)` after parsing
  → If category is parsed but invented (LLM hallucinated): replace with config.default_category
  → Append " [AUTO-FALLBACK]" to summary (visible to operator)
  → Set _parseError = true → flag in Slack notification
  → Different from Layer 4: AI returned valid JSON, just hallucinated a category

Layer 4 (Code-level, catastrophic fallback): Keyword classifier
  → Triggered ONLY when try/catch fires (Layers 1-3 all failed)
  → Regex patterns on subject + body
  → Maps: "faktur" → sprawy_fakturowe, "reklamac" → reklamacja_eskalacja, etc.
  → Detects urgency from Polish keywords ("pilne", "natychmiast", "asap")
  → Sets _parseError = true → propagates to Slack notification as warning
  → Different from Layer 3: AI returned no usable output at all
```

#### Consequences

**Positive:**
- Workflow never crashes on AI output. Always returns a valid category.
- Operator gets warned (`_parseError: true`) in Slack when classification quality is degraded — informed, not surprised.
- LLM upgrades (gpt-4o-mini → gpt-4o → next model) require zero code changes — schema is declarative.
- Can switch AI providers (OpenAI → Anthropic → local model) by swapping the LLM node, parser logic stays.
- **Distinction between semantic and structural failures** allows operator to diagnose root cause from `_parseError` flag and `errorReason` field.

**Negative:**
- 4 layers of code to maintain. Each has different failure modes.
- Keyword fallback (Layer 4) is less accurate than AI — doesn't understand context. A mail saying "we received your invoice and reject it" gets `sprawy_fakturowe` instead of `reklamacja_eskalacja`.
- Adds ~125 lines of JS to `Validate and Route` Code node (vs ~50 lines for naive single-layer approach).

**Mitigations:**
- Comprehensive test suite (24 cases) catches regressions in any layer.
- `_parseError` flag in output enables monitoring: count of fallback activations per day = AI quality KPI.


#### Alternatives considered

1. **Single layer: rely only on Structured Output Parser**
   Rejected — Structured Parser doesn't always survive token-truncation or rate-limit errors. Need belt-and-suspenders.

2. **Function calling instead of structured output**
   Considered — gpt-4o-mini supports function calling. Decided against because LangChain Structured Output Parser is more portable across LLM providers (Anthropic, local models).

3. **Always retry on parse fail (no fallback)**
   Rejected — could turn a 1-second classification into 30-second loop on persistent issue. Latency unacceptable for real-time inbox triage.

4. **Send mail to "uncategorized" inbox if AI fails**
   Rejected — defeats purpose of automation. Operator would have to manually triage anyway.

5. **Treat semantic and structural failures as one layer (original design)**
   Rejected during v6.1 — obscured the actual logic flow and made it harder to diagnose AI quality issues. Splitting them improved both code clarity and operational observability.

---

## ADR-002
### Store all configuration as data in Google Sheets, not in workflow code

**Status:** Accepted
**Date:** 2026-02 (v4 onwards)

#### Context

The system has many parameters that change without warning:
- Team composition (who handles what category)
- Slack user IDs (people change accounts, leave team)
- AI prompt tuning (categories evolve, edge cases discovered)
- Tone of voice per category (entuzjastyczny / formalny / empatyczny)
- Limits and thresholds (body truncation, retry counts, etc.)

Hardcoding these in n8n Code nodes means every change requires:
1. Edit workflow JSON
2. Re-import / save in n8n UI
3. Test
4. Risk regression

**Operator (non-technical client)** cannot do steps 1-4. Result: every config change becomes a developer ticket.

#### Decision

Move all configuration to **Google Sheets** with 4 tabs, loaded at the start of every execution:

| Tab | Purpose | Schema |
|---|---|---|
| `Zespół` | Team routing | `kategoria` → `osoba` → `rola` → `slack_channel` → `slack_user_id` → `gmail_label_id` → `emoji` |
| `Konfiguracja` | System parameters | `klucz` → `wartosc` → `opis` (key/value pairs) |
| `Kategorie` | AI prompt context | `klucz` → `nazwa_pl` → `emoji` → `opis_dla_ai` → `ton_odpowiedzi` |
| `Historia` | Audit log (auto-append) | `data` → `email_nadawcy` → `kategoria` → `temat` → `decyzja` → `przypisano` |

> **Note on naming convention:** The Sheet tab name uses Polish character `ę` (`Zespół`), matching the tab name as displayed in Google Sheets UI. The corresponding n8n node that loads this tab is named `Load Zespol` without the `ę` — n8n node names use ASCII to avoid issues with expression syntax and automatic references. This intentional split (Polish for user-facing tab, ASCII for code identifier) is consistent across all workflow JSON.

The classifier prompt is built dynamically from `Kategorie`:

```javascript
KATEGORIE:
{{ $('Load Kategorie').all()
   .map(r => r.json.klucz + ' — ' + r.json.opis_dla_ai)
   .join('\n') }}
```

The draft generator picks tone of voice per category from `Kategorie.ton_odpowiedzi`.

Routing picks team member from `Zespół[category]` (loaded via `Load Zespol` node — see naming note above).

#### Consequences

**Positive:**
- **Operator self-service:** adding a new category is 1 row in `Kategorie` + 1 row in `Zespół`. No deploy.
- **Audit trail visible to operator:** `Historia` is human-readable, can be filtered/sorted in Google UI.
- **Decoupled from workflow:** can A/B test prompt variations by editing one cell.
- **Disaster recovery:** workflow JSON can be re-imported from scratch; config persists in Sheets.

**Negative:**
- **3 Sheets API calls per execution** = ~2-3 seconds of latency overhead. Acceptable for 1-minute polling, would be a problem for sub-second triggers.
- **No transactions:** if operator edits Sheets mid-execution, partial config can be loaded. Mitigated by serial load (atomic enough at this scale).
- **No schema enforcement:** operator can break workflow by typoing column header. Mitigated by clear instructions in `Konfiguracja.opis` column.
- **Sheets quota:** Google Sheets API has 60 req/min/user. At 1-minute polling × 3 calls = 3 req/min. Plenty of headroom unless email volume spikes 20×.

**Cost:**
- Google Sheets is free at this scale. No additional spend.

#### Alternatives considered

1. **Environment variables + hardcoded categories**
   Rejected — defeats operator self-service goal.

2. **Database (Postgres / Supabase)**
   Rejected — over-engineering for 10 categories × 5 team members. Operator cannot edit DB rows safely.

3. **n8n credentials store for secrets, hardcoded for non-secrets**
   Rejected — even non-secrets change frequently (Slack User IDs, prompt text).

4. **Notion database**
   Considered — operator already uses Notion. Rejected because n8n Notion integration is less mature than Google Sheets, and Sheets has better concurrent-edit semantics.

---

## ADR-003
### Idempotency via `$getWorkflowStaticData` cache of last 500 messageIds

**Status:** Accepted (with v7 follow-up planned)
**Date:** 2026-04 (v6 hardening)

#### Context

The workflow uses Gmail Trigger with `unread` filter and 1-minute polling. Standard pattern: trigger fires on new mail → workflow processes → marks as read.

Failure modes that cause duplicate processing:
1. **Workflow fails between AI Categorize and Mark as Read** → mail stays unread → next poll (60s later) re-triggers it.
2. **Manual re-execute from n8n UI** → same mail processed twice.
3. **Webhook test mode** during development → same payload sent multiple times.

Without dedupe, this means: 2× Slack DMs, 2× draft creation in Gmail, 2× audit log entry, 2× approval emails to assignee. Embarrassing for the user, confusing for the team.

n8n Cloud (and most self-hosted setups) **doesn't have a built-in database** for cross-execution state.

#### Decision

Use **`$getWorkflowStaticData('global')`** to maintain a cache of the last 500 processed `messageId` values. Skip duplicates **before any side-effects**:

```javascript
const staticData = $getWorkflowStaticData('global');
if (!Array.isArray(staticData.processedIds)) staticData.processedIds = [];

for (const item of items) {
  const messageId = email.id || email.messageId || '';

  // Skip duplicates BEFORE any processing
  if (messageId && staticData.processedIds.includes(messageId)) {
    skippedDuplicates++;
    continue;
  }

  // ... validation, AI, routing ...

  // Mark as processed AFTER validation passes (clean state on failure)
  if (messageId) {
    staticData.processedIds.push(messageId);
    if (staticData.processedIds.length > 500) {
      staticData.processedIds = staticData.processedIds.slice(-500);
    }
  }
}
```

#### Consequences

**Positive:**
- Zero side-effects on duplicate. Mail is processed exactly once even with retries.
- **Cleanup is automatic** — sliding window of 500 keeps memory bounded.
- **Skip happens early** (before AI calls) → no wasted OpenAI tokens on duplicates.
- **Marked AFTER validation** → if validation throws, messageId is NOT cached, so retry can re-process.

**Negative:**
- **Static data is in-memory** — lost on n8n instance restart. After restart, the first email batch could double-process if Gmail Trigger fires before mark-as-read completes for in-flight execution.
- **Static data is per-workflow, not shared** — Extended workflow has its own cache (which is OK; webhook IDs are different).
- **No persistence guarantee** — n8n docs warn that `getWorkflowStaticData` may be cleaned up under memory pressure.

**Real-world impact at scale:**
- 500 messageIds × ~50 bytes each = ~25 KB memory. Negligible.
- Restart frequency on Hostinger n8n: ~weekly (auto-updates). Worst case: 10 minutes of duplicate-vulnerable window per week.

#### v7 follow-up

This ADR is **partially accepted** — for current scale (20 mails/day), in-memory cache is sufficient. For higher scale or stricter guarantees, v7 will migrate to Sheets lookup:

```javascript
// v7 plan:
// 1. Append messageId + timestamp to "Processed" sheet tab
// 2. Before processing, query last 500 rows of "Processed"
// 3. Skip if found
// Trade-off: +1 Sheets API call per execution, but persists across restarts
```

#### Alternatives considered

1. **No idempotency, rely on Gmail unread filter**
   Rejected — fails on workflow crash mid-flow. Mark-as-read may not have run.

2. **Mark as read at the very start (defensive)**
   Rejected — see ADR-005. Loses mails on failure.

3. **External database (Redis / Postgres)**
   Rejected — over-engineering. Adds infrastructure dependency.

4. **Sheets-based dedup from day 1**
   Rejected — extra latency for ~0% benefit at current scale. Premature optimization.

5. **Use Gmail's `historyId` for incremental sync**
   Considered — would eliminate need for messageId cache. Rejected because n8n Gmail Trigger doesn't expose `historyId` API.

---

## ADR-004
### Two-stage send: draft default, send-on-approval opt-in

**Status:** Accepted
**Date:** 2026-03 (v6)

#### Context

The system serves a consultancy firm. Outgoing emails carry legal weight:
- NDAs and contract negotiations
- Pricing commitments
- Partnership proposals
- Crisis communication during incidents

A single AI-generated email sent without human review could:
- Quote wrong prices → financial loss
- Make false commitments → legal liability
- Leak confidential information → NDA breach
- Damage client relationship → revenue impact

GPT-4o-mini hallucination rate is non-zero. Even with low rates (~1-3%), at scale that's catastrophic.

#### Decision

**Workflow never auto-sends.** Two stages:

**Stage 1 — Draft (always runs):**
- AI generates draft response
- Saves as Gmail draft in original thread (`resource: "draft"`)
- Notifies assignee via Slack DM
- **No outbound communication to the original sender**

**Stage 2 — Approval (configurable, opt-in):**
- For categories flagged as approval-required, additionally:
  - HTML email sent to assignee with 3 buttons (✅ ✏️ ❌)
  - Each button = link to `$execution.resumeUrl?action=...`
  - `Wait` node listens on webhook with `limitWaitTime: 2 days`
  - On `?action=zatwierdz`: Gmail `reply` actually sends draft to original sender
  - On `?action=popraw`: Slack DM to assignee, no send
  - On `?action=odrzuc`: Slack notify, draft stays in Gmail
  - On timeout: Slack alert to #general

**Default:** Stage 1 only. Approval is opt-in per category in `Zespół` config (future v7 will add explicit `approval_required: true/false` column).

#### Consequences

**Positive:**
- **Zero risk of unauthorized auto-send.** AI cannot reach customer without human action.
- **80/20 split:** AI does classify + draft + routing (80% of work). Human does click (20% of work).
- **Audit trail** in `Historia` sheet shows every approval/rejection with timestamp.
- **Approval UX is async:** assignee can approve from phone, in meeting, etc. — just clicks button.

**Negative:**
- **Latency:** customer doesn't get instant response. Even fast approval = ~30 minutes typical.
- **Approval inbox spam:** if all categories use approval flow, assignee gets duplicate emails (one notification, one approval). Mitigation: only enable approval for high-stakes categories.
- **Wait node holds execution open for up to 2 days** — at 20 mails/day × 2 days × 50% approval rate = ~20 concurrent open executions. n8n handles this fine but it's not free.
- **HTML email button rendering varies** — Outlook/Gmail/Apple Mail render slightly differently. Tested in 3 clients, all functional.

#### Alternatives considered

1. **Auto-send for all categories**
   Rejected — see context. Unacceptable risk for regulated industry.

2. **Auto-send for "low-risk" categories (spam, system messages)**
   Considered — would reduce assignee workload. Deferred to v7 (need explicit category flag).

3. **Approval via Slack interactive message instead of email**
   Considered — better UX (no separate email). Rejected because Slack interactive messages require complex setup (Slack App, signing secrets, OAuth scopes) and the assignee is already getting a Slack DM. Email approval is portable across orgs.

4. **No approval flow, draft only**
   Considered — simpler. Rejected because client explicitly requested ability to "approve and send with one click" for time-sensitive responses.

---

## ADR-005
### Mark email as read at end of flow, not at start

**Status:** Accepted
**Date:** 2026-04 (v5 → v6 refactor)

#### Context

In v5, `Gmail - Mark as Read` was placed immediately after `Validate Email`. The reasoning: prevent duplicate processing if the same Gmail Trigger event fires twice.

Problem discovered in production: when AI or Slack failed mid-flow, the email was already marked as read. Next Gmail Trigger poll (`unread` filter) would skip it. **Mail effectively lost** — user never saw it, system never alerted.

Real incident: OpenAI rate-limited during a busy hour. 7 emails marked as read, 7 emails lost. Customer complained when 2 reservations went unanswered.

#### Decision

Move `Gmail - Mark as Read` to **end of notification path**, after Slack DM succeeds.

```
... Validate → AI → Route → Add Label → Generate Draft → Create Draft → Format Notification
                                                                          ↓
... → Is Urgent? → Slack channel → Prepare DM → Slack Worker DM → ⭐ MARK AS READ
```

Failure anywhere upstream of Mark as Read = mail stays unread = re-processed on next poll.

#### Consequences

**Positive:**
- **No mail is lost** on transient failures. Workflow self-heals on retry.
- Operator gets organic alerts via Error Handler when something goes wrong, can intervene.

**Negative:**
- **Possible double processing** when failure occurs *after* Mark as Read but somewhere downstream (e.g., approval email send fails after Mark as Read succeeded). Covered by ADR-003 (idempotency cache).
- The approval flow path **never marks as read** in current implementation — the mail stays unread until manual cleanup. Acceptable trade-off, but listed in v7 backlog.

#### Alternatives considered

1. **Mark as read at start, retry on failure**
   Rejected — would need explicit retry queue, complex. Just don't mark until success.

2. **Mark as read in a separate "cleanup" workflow that runs after success**
   Considered — cleaner separation. Rejected because adds operational complexity for marginal benefit.

3. **Don't mark as read at all, rely on `processedIds` cache**
   Rejected — Gmail inbox would visually fill up with "unread" indicators forever. Bad UX for human operator.

---

## ADR-006
### Dedicated error workflow with dual-channel alerting (Slack + Gmail)

**Status:** Accepted
**Date:** 2026-03 (v6)

#### Context

n8n offers two error-handling patterns:
1. **Inline `try/catch`** in each Code node — local handling, custom logic
2. **`errorWorkflow` setting** — global workflow that runs on any unhandled error

Both have value:
- Inline `try/catch` for known failure modes (AI parse fail → fallback)
- `errorWorkflow` for unknown/unexpected failures (network glitch, n8n itself failing)

The error workflow needs to **alert someone immediately** and **survive partial outages**. If we alert via Slack only and Slack OAuth has expired or workspace is down, we're blind.

#### Decision

**Two layers of error handling:**

**Layer 1 — Inline try/catch in every Code node** with enriched error message:
```javascript
catch (error) {
  const enriched = new Error(`[NodeName] ${error.message}`);
  enriched.description = `Node: NodeName\nOriginal: ${error.message}`;
  throw enriched;
}
```
This makes execution log immediately point to failing node + original error.

**Layer 2 — Dedicated `errorWorkflow`** bound via settings (`errorWorkflow: "HmSyczIa6tXTEhFe"`) on both main and Extended workflows.

Error Handler workflow has 5 elements (4 functional nodes + 1 sticky note for inline documentation):
1. **Error Trigger** — receives error context
2. **Format Error Alert** (Code) — classifies severity + builds message:
   - HIGH 🔴 → `ECONNREFUSED|timeout|rate limit|503` (service outage)
   - LOW 🟠 → `parse|validation|FALLBACK` (handled gracefully but worth noting)
   - MEDIUM 🟡 → everything else
3. **Slack #general** (primary, retry 3×) — formatted alert with workflow/node/timestamp/severity
4. **Gmail backup** (parallel, `onError: continueRegularOutput`) — same content as HTML email to admin
5. **Sticky note** — inline documentation of dual-channel rationale (visible to anyone editing the workflow)

**Both Slack and Gmail are triggered in parallel.** If Slack fails, Gmail still arrives. If Gmail fails, Slack alert is still seen.

#### Consequences

**Positive:**
- **No single point of failure** for alerting. Two independent channels.
- **Severity classification** lets operator triage: HIGH = wake up at night, LOW = fix during work hours.
- **Workflow-level binding** means adding error handling to a new workflow is one line in settings, not refactoring.
- **Enriched errors** in execution log mean "what failed" is obvious without reading raw stack traces.

**Negative:**
- **Two notification points** — operator might get confused which is canonical. Mitigated by identical content + clear "primary: Slack" labeling.
- **Slack and Gmail rate limits** are separate — can both be hit during incident. Rare in practice.
- **Severity classification is keyword-based**, not semantic — false negatives possible (a real outage that doesn't say "503").

#### Alternatives considered

1. **Slack-only alerting**
   Rejected — see context. Single point of failure.

2. **PagerDuty / Opsgenie integration**
   Rejected — over-engineering for 1-person operations. Costs money. Free Slack + Gmail covers needs.

3. **Email-only alerting**
   Rejected — email arrival latency is unpredictable. Slack DM is faster for active monitoring.

4. **No errorWorkflow, just inline try/catch everywhere**
   Rejected — n8n itself can fail (instance crash, OOM). Need external visibility.

---

## ADR-007
### Differentiated retry strategy per service

**Status:** Accepted (revised 2026-04)
**Date:** 2026-03 (v6), revised 2026-04 (v6.1 — clarified AI retry asymmetry between MAIN and EXTENDED workflows)

> **Revision note (2026-04):** Original ADR stated "OpenAI calls: 3×/3s/throw" as a uniform policy. In review, observed that this only applies to AI Generate Tasks in EXTENDED workflow. MAIN workflow's AI Categorize Email and AI Generate Draft have no node-level retry — they rely on the 4-layer fallback chain (ADR-001) for resilience. ADR updated to honestly document this asymmetry with rationale.

#### Context

Different external services have different failure characteristics:

| Service | Common failures | Recovery time |
|---|---|---|
| Google Sheets | Transient timeouts, quota errors | Seconds |
| Slack | Rate limits, OAuth refresh | Seconds-minutes |
| Gmail | OAuth refresh, label-not-found | Seconds |
| OpenAI | 5xx during high load, rate limits | Seconds-minutes |

A blanket "retry 3× wait 5s" everywhere is wasteful (Sheets recovers in <1s) or insufficient (OpenAI 5xx cluster needs longer backoff).

Some calls are **safe to retry** (idempotent reads), others **may cause duplication** (Gmail draft creation).

#### Decision

Per-node retry strategy:

| Service / Operation | Retry | Wait | `onError` | Rationale |
|---|---|---|---|---|
| Google Sheets reads (`Load Zespol`, `Load Config`, `Load Kategorie`) | 3× | 2s | throw | Idempotent, fast recovery |
| Slack messages in MAIN (URGENT Channel, Normal Channel, Worker DM) | 3× | 3s | continueRegularOutput | Rate limits common, msg duplication tolerable |
| Slack alert in Error Handler (Slack - Error Alert) | 3× | 5s | continueRegularOutput | Higher backoff because called during incident |
| Gmail mutations (label, draft, mark as read, send) | 0× | — | continueRegularOutput | Risk of duplication; non-critical |
| OpenAI calls in EXTENDED (AI Generate Tasks) | 3× | 3s | throw | 5xx recoverable, but throw if persists |
| OpenAI calls in MAIN (AI Categorize Email, AI Generate Draft) | 0× | — | (relies on Layer 4 keyword fallback) | See note below |
| Trello card creation | 0× | — | continueRegularOutput | Risk of duplicate cards |
| Drive folder creation | 0× | — | continueRegularOutput | Risk of duplicate folders (Sheets cache covers) |

> **Note on AI retry asymmetry:** MAIN workflow's AI calls (categorization + draft generation) do not have node-level retry — instead, they rely on the 4-layer fallback chain documented in [ADR-001](#adr-001) for resilience. When OpenAI returns 5xx in MAIN, the workflow proceeds via Layer 4 (keyword fallback) with `_parseError: true` flag rather than retrying. EXTENDED's AI Generate Tasks does have retry (3×/3s) because it's called via HTTP node (not Agent node) and has no equivalent fallback chain.
>
> This asymmetry is intentional but worth flagging: at 20 mails/day a 5xx burst is rare, and Layer 4 fallback is acceptable degradation. At higher scale, adding 3× retry to MAIN AI nodes would reduce keyword-fallback rate at the cost of latency. Tracked in v7 backlog.

**Critical path** (validation, AI, routing) — fail fast and let `errorWorkflow` handle catastrophic failures. For AI specifically, the 4-layer fallback is the recovery mechanism (see ADR-001), not retry.

**Non-critical path** (Slack notifications, Gmail labels) — `onError: continueRegularOutput` allows flow to continue even if one notification fails.

#### Consequences

**Positive:**
- **Fast recovery** for transient errors without hammering services.
- **No accidental duplication** of side-effects (Gmail drafts, Trello cards, Drive folders).
- **Critical path stays fast** — failures surface immediately to operator.

**Negative:**
- **Retry config is in node parameters**, not centralized — easy to forget when adding new nodes.
- **Different teams may need different strategies** — current values are tuned for n8n + Hostinger + ~20 mails/day. At higher scale, may need adjustment.

#### Alternatives considered

1. **Uniform retry strategy (3× × 5s for everything)**
   Rejected — wastes time on fast-recovering services, insufficient for slow ones.

2. **Aggressive retry (10× with exponential backoff)**
   Rejected — single failure can monopolize execution slot, blocking other mails.

3. **No retries, fail fast on everything**
   Rejected — transient errors are common in cloud APIs, manual retry would be needed too often.

4. **Move retry logic to a wrapper function in Code nodes**
   Considered — would centralize. Rejected because n8n native retry is already declarative and works reliably.

---

## ADR-008
### Approval flow integrated in main workflow, not separate workflow

**Status:** Accepted
**Date:** 2026-04 (v5 → v6 refactor)

#### Context

In v5, the approval flow was implemented in a separate workflow ("Extended"), triggered by webhook from the main flow.

Problem: **the webhook was never triggered.** Every email went through main flow → notification, but the approval webhook was dead code. Discovered when reviewing audit logs (zero approval events).

Approval is **core requirement** ("draft must be reviewable by human"), not a bonus feature. Splitting it into a separate workflow created a hidden dependency and silent failure mode.

#### Decision

Move approval flow into **main workflow** as a parallel branch from `Format Notification`:

```
Format Notification ──┬── BRANCH 1: Slack notification → Mark as Read
                      │
                      └── BRANCH 2: Approval flow → Wait → Switch → ...
```

Both branches always run. Approval is part of the natural flow.

#### Consequences

**Positive:**
- **No silent failure** — approval is part of the flow, not an isolated optional path.
- **Single workflow to debug** — execution log shows everything in one place.
- **Wait node is lightweight** — n8n suspends execution without consuming resources during the 2-day timeout.

**Negative:**
- **Main workflow is bigger** — 33 nodes vs original 27. More surface area to maintain.
- **All emails go through approval flow currently** — even spam. Should be conditional per category in v7.
- **Open executions accumulate** — at 20 mails/day × 2 days × 50% approval rate = ~20 concurrent suspended. n8n handles fine, but visible in executions list.

#### Alternatives considered

1. **Keep in separate Extended workflow, fix the webhook**
   Considered — would preserve separation. Rejected because the original split was premature optimization. Extended is for *bonus features* (Trello, Drive), approval is *core*.

2. **Approval as separate workflow but triggered automatically (no webhook)**
   n8n doesn't support workflow-to-workflow direct invocation natively (only via webhook or `Execute Workflow` node). Considered using `Execute Workflow` node — rejected because it makes debugging harder (two execution logs to correlate).

3. **No approval, just Slack notification**
   Rejected — see ADR-004. Approval is needed for regulated industry.

---

## ADR-009
### Serial Sheets load instead of parallel

**Status:** Accepted (n8n bug workaround)
**Date:** 2026-03 (v6)

#### Context

Workflow loads 3 Google Sheets at start: `Zespół`, `Konfiguracja`, `Kategorie`. Initially loaded in parallel using `Merge` node with `numberInputs: 3` and `chooseBranch` mode.

**Discovered n8n bug:** `Merge` node with 3 inputs in `chooseBranch` mode does not generate the 3rd input slot in UI. Workflow runs but silently drops the 3rd input.

Tried multiple workarounds:
- Different Merge mode (combine, append) — semantics wrong for our use
- Merge node v2 vs v3 — same bug
- Reported to n8n GitHub (issue exists for this specific pattern)

Loading times (20 mails/day):
- Parallel: ~1.5s
- Serial: ~3s

#### Decision

Use **serial chain**: `Load Zespol → Load Config → Load Kategorie`. Each waits for the previous to complete.

#### Consequences

**Positive:**
- **Reliable** — no n8n bugs in serial flow.
- **Easier to debug** — execution log shows each load distinctly.

**Negative:**
- **+1.5s latency per execution** vs parallel.

**Cost-benefit:**
- 1.5s × ~600 executions/month = 15 minutes/month total latency added.
- For 1-minute polling with 20 mails/day, this is invisible to user.

If volume grows 10×, parallel loading worth revisiting.

#### Alternatives considered

1. **Use SplitInBatches + Merge with workaround**
   Considered — rejected as more complex than serial.

2. **Combine 3 sheets into 1 with multi-tab structure** (single API call)
   Considered — Sheets API supports `range` parameter for fetching multiple ranges in one call. Rejected because it would couple workflow to specific Sheets layout. Current setup is cleaner.

3. **Cache Sheets in n8n static data, reload only on schema change**
   Considered — would eliminate latency entirely. Rejected because cache invalidation is hard and operator expects edits to take effect immediately.

---

## ADR-010
### Single `Validate and Route` Code node instead of two separate nodes

**Status:** Accepted (refactor from v5)
**Date:** 2026-04 (v6 refactor)

#### Context

In v5, two separate nodes existed: `Validate AI Response` and `Parse AI & Route`. Both contained nearly identical JSON parsing logic for the AI output.

Problems:
- **Duplication** — bug fixes had to be applied in both places. Already had instances of inconsistency between them.
- **Synchronization risk** — easy to update one and forget the other.
- **Harder to test** — need to test both in sync.

#### Decision

**Merge into single `Validate and Route` Code node** that handles:
1. AI output parsing (3 formats: object / string / raw)
2. Validation against allowed categories
3. Keyword fallback if parsing fails
4. Routing assignment from `Zespół` config
5. Crisis manager assignment if `isUrgent`

#### Consequences

**Positive:**
- **Single source of truth** — one place to fix bugs, one place to add features.
- **Atomic operation** — if validation fails, routing isn't attempted (avoids inconsistent state).
- **Easier to test** — single node, single set of test cases.

**Negative:**
- **Larger Code node** — ~125 lines of JS in one node. Harder to scan visually.
- **No granular execution log** — if an issue happens in routing logic, the whole node shows as failed (not a specific sub-step).

#### Alternatives considered

1. **Keep two nodes, extract shared logic to a third "common" node**
   Rejected — n8n doesn't support shared functions across Code nodes natively. Would need to duplicate or use complex `$workflow` references.

2. **Move to TypeScript / external module**
   Rejected — over-engineering. n8n Code nodes are fine for this scale.



