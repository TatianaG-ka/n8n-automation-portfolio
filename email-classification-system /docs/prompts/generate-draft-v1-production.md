# Generate Draft — v1 (production)

**Status:** Active in production
**Workflow:** `01-email-triage-main.json`
**Node:** AI Generate Draft (LangChain Chain)
**Model:** `gpt-4o-mini`, temperature 0.4, maxTokens 500
**Tested:** 24/24 end-to-end pass (test_results_2026-04-08.md)
**Token cost:** ~250 input tokens per email + ~200 output tokens

## Rationale

Generation is harder than classification because the output is open prose, not constrained schema. v1 prioritizes three things:

**Tone configurable from Sheets, not hardcoded.** The `ton_odpowiedzi` field is fetched per-category from `Kategorie` Sheet (`profesjonalny` / `formalny` / `empatyczny` / `entuzjastyczny`). Operator changes tone for any category by editing one cell — no prompt redeploy.

**Hard rules against hallucination.** Three explicit "NIE" rules: no specific prices, no specific timelines, no commitments. The model is reminded that the output is a draft, not a sent email.

**Spam handling as exception clause.** Last rule of the prompt — if category is spam, output a single fixed line. This works in production but is positioned weakly (end of rule list, easy for the model to skip).

## Known limitations

These are reasons v2 candidate exists. Documented for transparency:

- Zero-shot — no examples of correct drafts shown to the model
- No structural template — model decides email structure (greeting, acknowledgment, next step, closing) on its own, leading to inconsistent format across drafts
- "NIE podawaj cen" is a negative constraint without alternative — model sometimes still hallucinates "skontaktujemy się w 24h"
- "Maksymalnie 150 słów" is a soft constraint regularly exceeded
- Tone is a single word ("empatyczny") interpreted by the model — no behavioral mapping
- Spam exception buried at end of prompt — easy to miss for borderline spam-like emails
- `[DRAFT - DO WERYFIKACJI PRZED WYSYŁKĄ]` tag is mechanical — could be more functional (e.g. include checklist for operator)

## System message

```
Jesteś asystentem {{ firma_nazwa || 'firmy doradczej' }}. Przygotowujesz
profesjonalne szkice odpowiedzi na maile.

ZASADY:
- Odpowiedź ZAWSZE po polsku, profesjonalna i uprzejma
- Ton: {{ ton_odpowiedzi z Kategorie[category] || 'profesjonalny' }}
- NIE podawaj konkretnych cen, terminów ani zobowiązań
- NIE składaj obietnic — to szkic do weryfikacji przez człowieka
- Podpisz jako "{{ firma_podpis || 'Zespół firmy doradczej' }}"
- Na końcu ZAWSZE dodaj: "\n\n[DRAFT - DO WERYFIKACJI PRZED WYSYŁKĄ]"
- Maksymalnie {{ draft_max_words || '150' }} słów
- Jeśli to spam: napisz tylko "Sugerowana akcja: brak odpowiedzi (spam).
  [DRAFT - DO WERYFIKACJI PRZED WYSYŁKĄ]"
```

## User message

```
Przygotuj profesjonalny szkic odpowiedzi na poniższy email.

Kategoria: {{ category }}
Pilność: {{ urgency }}
Temat: {{ subject }}
Od: {{ fromName }} <{{ from }}>
Podsumowanie: {{ summary }}

Treść oryginalna:
{{ body }}
```

## No structured output parser

Unlike `Categorize Email`, this node does NOT use Structured Output Parser — the output is free-form prose (the email draft). The downstream `Gmail - Create Draft` node takes the text directly and creates a Gmail draft.

## Production behavior

In 24 end-to-end tests + production monitoring, this prompt produces drafts that:
- Always end with `[DRAFT - DO WERYFIKACJI PRZED WYSYŁKĄ]` (compliance with mandatory tag: 100%)
- Stay roughly within length limit (drafts averaging 100-200 words, occasional 220-word outliers)
- Handle spam correctly when category is `spam` (no full draft generated)
- Match tone roughly — "empatyczny" drafts are noticeably softer than "formalny" ones

What's harder to measure without operator approval data:
- Hallucination rate (model occasionally still suggests "skontaktujemy się w 24h" despite the rule)
- Operator approval rate vs modify rate vs reject rate
- Sentence-level adherence to anti-commitment rule

This is exactly what A/B testing v2 against v1 should reveal.
