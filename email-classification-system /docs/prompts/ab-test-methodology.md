# A/B test methodology

This document defines how prompt candidate versions are validated against production versions before deployment.

## Why A/B testing for prompts

Prompt engineering literature is full of best practices, but "best" is contextual. Few-shot examples generally improve classification, but the *specific examples* you choose can hurt accuracy if they don't match your data distribution. Anti-hallucination phrase substitutions help, but if the substitutions are awkward in Polish business correspondence, operators will modify every draft.

The only valid answer to "is v2 better than v1?" is empirical: run both on the same inputs, compare metrics defined in advance, decide based on results.

## Test set design

### Categorize Email — test set

A 50-email evaluation set covering all 10 production categories plus edge cases:

- **30 emails** — 3 per production category (zapytanie_ofertowe, sprawy_fakturowe, reklamacja_eskalacja, sprawy_prawne_umowy, wspolpraca_partnerstwo, rekrutacja_cv, zaproszenie_wydarzenie, pytanie_klienta, spam, wiadomosc_systemowa)
- **10 emails** — urgent flag detection (mix of true urgent and false-positive baits like marketing using "PILNE")
- **10 emails** — edge cases from existing `test_emails.json`: empty subject, forwarded spam, nested replies, out-of-office, multi-topic, English email through Polish flow, single-word subject, large HTML payload, subtle marketing spam

Each email has a **ground truth label** (correct category, correct urgency) assigned by a human reviewer before A/B testing begins. Ground truth labels are committed to `docs/testy/categorize-eval-set.json`.

### Generate Draft — test set

A 30-email evaluation set covering 4 representative tones × edge cases:

- **20 emails** — 5 per tone (formalny, empatyczny, entuzjastyczny, profesjonalny), each tone applied to 5 different email contexts
- **5 emails** — spam category (verify early-exit behavior)
- **5 emails** — borderline cases (multi-topic, ambiguous tone fit, very long original email, very short original email, non-Polish input)

Generate Draft does not have ground truth — drafts are open prose. Evaluation uses operator review (see metrics below).

## Metrics — Categorize Email

Run v1 and v2 on the same 50-email test set. Record the following:

| Metric | Definition | Target for v2 to win |
|---|---|---|
| **Category accuracy** | % of emails classified correctly per ground truth | v2 ≥ v1 (no regression on simple cases) |
| **Urgency accuracy** | % of emails with correct urgent/normal flag per ground truth | v2 ≥ v1 + 5 pp on edge cases |
| **Edge case accuracy** | Same as urgency accuracy but only on the 10 edge cases | v2 ≥ v1 + 10 pp |
| **Parse error rate** | % of calls that triggered Layer 2/3/4 fallback in `Validate and Route` | v2 ≤ v1 |
| **Hallucinated category rate** | % of calls where AI returned a category outside the allowed enum | v2 ≤ v1 (already 0% in v1) |
| **`urgency_reason` quality** | % of urgent classifications where reason explicitly references criterion 1/2/3 | v2 ≥ 90% |
| **Token cost** | Average input + output tokens per call | v2 within 3× of v1 (with caching), 5× without |

### Decision criteria — Categorize

v2 deploys to production if **all** of:
- v2 wins on at least 4 of 7 metrics
- v2 doesn't regress more than 5 percentage points on any accuracy metric
- v2 hallucinated category rate stays at 0%
- Cost increase is acceptable (v2 ≤ 15 zł/month total at 20 emails/day)

If any condition fails, v2 stays as candidate. Iterate on prompt, re-test.

## Metrics — Generate Draft

Run v1 and v2 on the same 30-email test set. Two evaluation modes:

### Automated metrics

| Metric | Definition | Target for v2 to win |
|---|---|---|
| **Length adherence** | % of drafts within 4-6 sentence target | v2 ≥ 80% (v1 baseline likely ~50%) |
| **Structural compliance** | % of drafts with all 4 elements (greeting, ack, next step, closing) | v2 ≥ 90% |
| **Mandatory tag presence** | % of drafts ending with `[DRAFT - DO WERYFIKACJI...]` | Both should be 100% |
| **Spam early-exit correctness** | % of spam emails handled with single-line output | v2 = 100% |

### Human review metrics

The 30 drafts from v1 and 30 drafts from v2 are randomized and presented to a human reviewer (operator from the client's team, or a stand-in) without label. Reviewer marks each draft as:

- **Approve** — would send as-is
- **Modify** — requires editing before sending
- **Reject** — start over

| Metric | Definition | Target for v2 to win |
|---|---|---|
| **Approve rate** | % of drafts marked Approve | v2 ≥ v1 + 10 pp |
| **Hallucination instances** | Count of drafts with specific commitments (date, price, name) requiring verification | v2 ≤ v1 / 2 |
| **Tone match** | Subjective 1-5 score: does the draft tone match the configured tone for the category? | v2 average ≥ v1 + 0.5 |

### Decision criteria — Generate Draft

v2 deploys to production if **all** of:
- v2 wins on Approve rate by at least 10 percentage points
- v2 hallucination instances are at most half of v1's count
- v2 doesn't regress on tone match (v2 average ≥ v1 average)
- Cost increase is acceptable

If Approve rate is similar but hallucination drops significantly, that's still a win — fewer manual edits = lower operator workload, even if total approval count is unchanged.

## Test execution

### Pre-test setup

1. Lock both v1 and v2 prompt versions in this directory (no edits during test)
2. Commit ground truth labels for Categorize test set
3. Set up two parallel n8n workflows (clone of main, one with v1 prompt, one with v2)
4. Verify both clones produce expected output on a single test email before batch run

### Batch execution

Run all test emails through both workflows. Save outputs to:
- `docs/testy/ab-test-categorize-v1-{date}.json`
- `docs/testy/ab-test-categorize-v2-{date}.json`
- `docs/testy/ab-test-generate-v1-{date}.json`
- `docs/testy/ab-test-generate-v2-{date}.json`

### Analysis

Compute automated metrics from JSON outputs (Python script in `docs/testy/analyze.py`). Run human review for Generate Draft (presentation order randomized to avoid bias).

Commit results as `decisions/{node}-v2-decision-{date}.md` with:
- Final metrics table (v1 vs v2)
- Decision (deploy / iterate / abandon)
- Reviewer notes
- Sample drafts highlighting strengths/weaknesses

## Sample size discussion

50 emails for Categorize and 30 for Generate Draft are minimum-viable sample sizes. They detect large effects (10+ percentage points) reliably. They miss subtle improvements (2-3 percentage points).

For consulting-firm scale (20 emails/day, ~600/month), this is acceptable — the cost of a false-negative ("v2 was actually better, but our test missed it") is small. For higher-stakes deployments, sample sizes should scale with daily volume.

## What this methodology does NOT cover

- **Long-term prompt drift.** A prompt that wins A/B test today might degrade as language patterns or business context shift. Quarterly re-evaluation recommended.
- **Adversarial inputs.** Test set is representative, not adversarial. Prompt injection attacks, encoding attacks, and similar are out of scope here (covered in security review).
- **Operator preference variability.** One human reviewer may differ from another. For production decisions, use the operator who will actually approve emails in the real workflow.

## Why document this at all

A senior reviewer asks "how do you know v2 is better?" The strongest answer is "I ran an A/B test on this test set, with these metrics, and the results are committed in `decisions/`." Without this document, the answer is "best practices suggest..." — which is weaker.

Documenting the methodology is itself the deliverable. Even before any test runs, this file demonstrates that prompts are treated as production artifacts requiring empirical validation, not creative work shipped on intuition.
