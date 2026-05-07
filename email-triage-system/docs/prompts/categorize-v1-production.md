# Categorize Email — v1 (production)

**Status:** Active in production
**Workflow:** `01-email-triage-main.json`
**Node:** AI Categorize Email (LangChain Chain)
**Model:** `gpt-4o-mini`, temperature 0.1, maxTokens 500
**Output parser:** Structured Output Parser (JSON schema)
**Tested:** 24/24 end-to-end pass (test_results_2026-04-08.md)
**Token cost:** ~100 input tokens per email + ~150 output tokens

## Rationale

This is the prompt that has been running in production. Three design decisions are worth highlighting:

**Categories injected dynamically from Sheets.** The list of available categories is not hardcoded — it's pulled from the `Kategorie` Sheet at every workflow run. Operator adds a new category in Sheets → the AI adapts on the next email, no code change required. This is the AI side of the config-as-data architecture (see [ADR-002](https://github.com/TatianaG-ka/n8n-automation-portfolio/blob/main/email-triage-system/docs/design_decisions_ADR.md#adr-002)).

**Structured Output Parser does the heavy lifting.** The schema (5 fields, `urgency` as enum) is enforced by LangChain's parser, not by prose instructions. The model is constrained to valid output structurally, not just asked nicely. This is why the prompt itself can be relatively short — the parser catches what prose would miss.

**Polish-language prompt for Polish business context.** The model performs better with prompts in the same language as the input data. All instructions, urgency rules, and category descriptions are in Polish.

## Known limitations

These are the reasons v2 candidate exists. Documented here for transparency:

- Zero-shot — no examples of correct classifications shown to the model
- Eight urgency criteria listed flat, no grouping into mental categories
- No explicit tie-breaking rule for ambiguous cases
- No counter-examples (emails that look urgent but aren't, e.g. marketing using "PILNE")
- Persona ("ekspert od kategoryzacji w 5-osobowym zespole") doesn't change model behavior

## System message

```
Jesteś ekspertem od kategoryzacji maili w firmie doradczej
({{ zespol_rozmiar }}-osobowy zespół). Analizujesz maile przychodzące
na skrzynkę kontaktową i klasyfikujesz je.

KATEGORIE:
{{ Load Kategorie → numbered list: index. klucz — opis_dla_ai }}

ZASADY PILNOŚCI:
- PILNE: eskalacja od kluczowego klienta, awaria usługi, deadline w ciągu
  24h, groźba prawna, żądanie natychmiastowej interwencji, utrata danych,
  incydent bezpieczeństwa, windykacja
- NORMALNE: wszystko inne

Zawsze zwracaj WYŁĄCZNIE poprawny JSON. Nie dodawaj żadnego tekstu
przed ani po JSON.

Sub-nodes podpięte (ai_languageModel + ai_outputParser):
- OpenAI Chat Model — gpt-4o-mini, temperature 0.1, maxTokens 500
- Structured Output Parser — schema example:
{
  "category": "zapytanie_ofertowe",
  "urgency": "normalne",
  "summary": "Klient pyta o wycenę usług doradczych",
  "key_entities": ["firma X", "usługi doradcze"],
  "urgency_reason": ""
}
```

## User message

```
Przeanalizuj poniższy email i zwróć kategoryzację w formacie JSON.

Temat: {{ subject }}
Od: {{ fromName }} <{{ from }}>
Data: {{ date }}

Treść:
{{ body || '(brak treści - oceń kategorię tylko na podstawie tematu i nadawcy)' }}

---
Dostępne kategorie:
{{ Load Kategorie → list: klucz — opis_dla_ai }}

Zwróć WYŁĄCZNIE poprawny obiekt JSON z polami:
- category: jedna z powyższych kategorii
- urgency: "pilne" lub "normalne"
- summary: krótkie podsumowanie maila po polsku (1-2 zdania)
- key_entities: tablica kluczowych podmiotów/tematów
- urgency_reason: jeśli pilne - uzasadnienie, inaczej pusty string
```

## Output schema (Structured Output Parser)

```json
{
  "type": "object",
  "properties": {
    "category":       { "type": "string" },
    "urgency":        { "type": "string", "enum": ["pilne", "normalne"] },
    "summary":        { "type": "string" },
    "key_entities":   { "type": "array", "items": { "type": "string" } },
    "urgency_reason": { "type": "string" }
  },
  "required": ["category", "urgency", "summary"]
}
```

## Production behavior

The downstream `Validate and Route` Code node implements four-layer fallback for cases where this prompt fails (see [main README](../../README.md#four-layer-ai-parsing-fallback) for full breakdown). Layer 3 (semantic fallback for invented categories) and Layer 4 (keyword fallback for structurally broken output) are the production safety net for this prompt.

In 24 end-to-end tests + production monitoring, fallback Layer 3 has never fired (model has not invented categories). Fallback Layer 4 has never fired either (model has always returned parseable structure). This is the empirical baseline that v2 needs to beat.
