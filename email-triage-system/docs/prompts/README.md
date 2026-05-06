# Prompts

This directory contains all production prompts used in the Email Triage workflow, versioned separately from the workflow JSON files.

## Why separate from workflow JSON

Prompts are production artifacts that evolve faster than the workflow structure around them. Embedding them in 50KB JSON exports makes them invisible to git diff, hard to review, and impossible to A/B test cleanly.

Treating them as code (separate files, version history, peer review) makes prompt changes:
- **Auditable** — every change has a commit, an author, and a reason
- **Reversible** — rolling back is one git revert away, not a re-export of n8n
- **Reviewable** — someone can comment on a specific line of a prompt in a PR
- **Testable** — A/B comparisons run against named versions, not against "whatever was in JSON last week"

This is the same pattern used in production ML systems for model prompts: prompts as first-class artifacts in the repo.

## Current state

| Node | Production | Candidate | Status |
|---|---|---|---|
| AI Categorize Email | [v1](categorize-v1-production.md) | [v2](categorize-v2-candidate.md) | v1 in production. v2 ready for A/B test. |
| AI Generate Draft | [v1](generate-draft-v1-production.md) | [v2](generate-draft-v2-candidate.md) | v1 in production. v2 ready for A/B test. |

## Versioning policy

A new prompt version goes to production only after:

1. Generated outputs on a test set of at least 20 representative emails per category
2. Comparison against current production on the metrics defined in [`ab-test-methodology.md`](ab-test-methodology.md)
3. Decision document with results committed to this folder
4. Rollback plan: previous version stays in repo, can be redeployed in under 5 minutes by reverting the workflow JSON change

This policy applies to non-trivial prompt changes — typo fixes, category list updates from Sheets, and minor wording tweaks don't need full A/B treatment.

## Why v1 still in production

v2 candidates apply 2026-era prompt engineering best practices: few-shot examples, explicit decision order, anti-hallucination phrase substitutions, structured early-exit for spam, behavioral mapping of tone descriptors. Theoretical improvements on each axis are well-documented in the prompt engineering literature.

But improvements in *this specific data distribution* must be measured, not assumed. v1 has 24/24 test pass and stable production behavior. Moving to v2 without empirical validation would be premature — confidence in a prompt should come from data, not from intuition that newer techniques are better.

## Cost trade-offs

v2 candidates use roughly 5× more input tokens than v1 (v1 ~100 input tokens, v2 ~500 input tokens per call) due to few-shot examples. At 20 emails/day on `gpt-4o-mini`:

- v1 cost: ~$1.50/month (~5 zł)
- v2 estimated cost: ~$2.50/month (~10 zł) without prompt caching, lower with caching

OpenAI's prompt caching (available since late 2024) caches the static portion of the system message across calls. Since v2 system messages are stable across all 20 emails/day, caching should bring v2 input cost back close to v1 levels. This is a hypothesis to validate during A/B testing, not a confirmed saving.

## Adding a new prompt version

When iterating on a prompt:

1. Don't edit the production file. Create a new candidate file: `<node>-vN-candidate.md`
2. Document the rationale at the top: what's changing, why, expected impact
3. Run A/B test per [`ab-test-methodology.md`](ab-test-methodology.md)
4. Commit results as `decisions/<node>-vN-decision.md` (whether deployed or not)
5. If deployed: rename candidate to `<node>-vN-production.md`, archive previous as `<node>-vN-1-archived.md`

This pattern keeps the full history visible and recoverable.

## Files

- [`categorize-v1-production.md`](categorize-v1-production.md) — Active classification prompt
- [`categorize-v2-candidate.md`](categorize-v2-candidate.md) — Candidate with few-shot examples
- [`generate-draft-v1-production.md`](generate-draft-v1-production.md) — Active draft generation prompt
- [`generate-draft-v2-candidate.md`](generate-draft-v2-candidate.md) — Candidate with 4-element structure
- [`ab-test-methodology.md`](ab-test-methodology.md) — Test set design, metrics, decision criteria
