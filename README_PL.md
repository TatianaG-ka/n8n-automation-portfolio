# 🤖 Portfolio: Automatyzacja n8n & Integracja AI

![n8n](https://img.shields.io/badge/n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)
![OpenAI](https://img.shields.io/badge/OpenAI-412991?style=for-the-badge&logo=openai&logoColor=white)
![Claude](https://img.shields.io/badge/Claude_Code-555?style=for-the-badge&logo=claude)
![REST API](https://img.shields.io/badge/REST_API-005571?style=for-the-badge&logo=fastapi&logoColor=white)
![Pinecone](https://img.shields.io/badge/Pinecone-000?style=for-the-badge&logo=pinecone&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-3ECF8E?style=for-the-badge&logo=supabase&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![Telegram](https://img.shields.io/badge/Telegram-26A5E4?style=for-the-badge&logo=telegram&logoColor=white)
![WhatsApp](https://img.shields.io/badge/WhatsApp-25D366?style=for-the-badge&logo=whatsapp&logoColor=white)
![ElevenLabs](https://img.shields.io/badge/ElevenLabs-000?style=for-the-badge&logo=elevenlabs&logoColor=white)

🇬🇧 [English version](README.md)

---

## 👋 O mnie

**Tatiana Golińska** — Automation & Integration Engineer.

Przechodzę z automatyzacji i manualnego testowania (5+ lat z Playwright, Selenium, REST Assured) do automatyzacji procesów biznesowych. Aktualnie buduję rozwiązania automatyzacji w n8n dla klientów e-commerce, pracując z REST API, OpenAI, Claude i bazami wektorowymi.

📧 tatiana.golinska@gmail.com · [LinkedIn](https://www.linkedin.com/in/tetiana-golinska/) · [GitHub](https://github.com/TatianaG-ka/)

---

## 🎬 Demo: Wielojęzyczny Chatbot AI

[![Raymon AI Assistant Demo](https://img.shields.io/badge/▶_Obejrzyj_Demo-FF0000?style=for-the-badge&logo=youtube&logoColor=white)](https://www.youtube.com/watch?v=K9fpfK8-U-0)

Wielojęzyczny chatbot AI do obsługi klienta wdrożony na demo stronie **Raymon Bicycles** (niemiecka marka e-bike'ów). Bot **automatycznie rozpoznaje język klienta** i odpowiada w tym samym języku — po polsku, niemiecku, angielsku i w każdym innym.

| Funkcja | Szczegóły |
|---------|-----------|
| **Auto-detekcja języka** | Klient pisze w dowolnym języku → bot odpowiada w tym samym |
| **Baza wiedzy** | Specyfikacje produktów, ceny, gwarancja (Raymon CarePlus), lokalizacje dealerów |
| **Obsługa głosu** | Czat tekstowy + rozmowa głosowa przez ElevenLabs |
| **Backend** | n8n (logika workflow i orkiestracja API) |
| **AI** | OpenAI GPT + ElevenLabs (głos i konwersacyjne AI) |
| **Hosting** | Netlify |

<!-- 📸 DODAJ SCREENSHOT: n8n workflow canvas tego chatbota
![Raymon Chatbot Workflow](05-raymon-multilingual-chatbot/screenshots/workflow-canvas.png)
-->

> 💡 Ten projekt pokazuje kompleksową dostawę chatbota: od projektu workflow w n8n, przez integrację AI, po wdrożenie na żywej stronie produktowej.

---

## 🏢 Projekty klienckie (w fazie testów, wdrożenie produkcyjne planowane Q1 2026)

Aktualnie buduję rozwiązania automatyzacji dla **2 klientów z branży e-commerce**. Szczegóły zanonimizowane.

| Automatyzacja | Co robi | Stack technologiczny |
|--------------|---------|---------------------|
| **Chatbot AI do obsługi klienta** | Odpowiada na pytania klientów z bazy FAQ i dokumentów produktowych (RAG) | n8n, OpenAI API, Pinecone, Webhooks |
| **System klasyfikacji emaili** | Automatyczne sortowanie emailów od klientów: reklamacje, pytania, finanse, inne — routing do odpowiednich działów | n8n, Gmail API, OpenAI API |
| **Obsługa faktur przychodzących** | Wyodrębnianie danych z faktur na emailu, przekierowanie do księgowości | n8n, Gmail API, REST API |
| **Follow-up nieaktywnych klientów** | Automatyczne sekwencje emaili reaktywacyjnych | n8n, Gmail API, OpenAI API |
| **Opisy produktów na Allegro** | Generowanie opisów produktów z AI na marketplace Allegro | n8n, OpenAI API, REST API |


> ⚠️ Pliki JSON workflow klienckich nie są udostępniane publicznie ze względu na poufność.

---

## 📚 Projekty edukacyjne

Workflow zbudowane podczas kursów n8n i automatyzacji AI. Pokazują moje zrozumienie kluczowych wzorców integracyjnych. Są w pełni funkcjonalne — można je zaimportować do dowolnej instancji n8n.

| # | Projekt | Co demonstruje | Integracje |
|---|---------|---------------|------------|
| 01 | [**AI Marketing Team**](01-ai-marketing-team/) | Wielonarzędziowy agent AI z wejściem głosowym | OpenAI GPT-4, DALL·E, Whisper, Telegram |
| 02 | [**RAG Chatbot E-commerce**](02-RAG-E-commerce-Chatbot/) | Architektura RAG: auto-ingestion + wyszukiwanie wektorowe | OpenAI, Pinecone, Google Drive |
| 03 | [**Pipeline Sprzedaży**](03-sales-automation-pipeline/) | Pełen cykl sprzedaży z AI follow-upami | Gmail, Google Sheets, Calendly, OpenAI |

---


**Co potrafię zbudować:**
- Chatboty RAG z wektorowymi bazami wiedzy (Pinecone, Supabase)
- Wielojęzycznych asystentów AI z automatyczną detekcją języka
- Automatyczną klasyfikację i routing emaili
- Wieloetapowe sekwencje follow-up z treściami generowanymi przez AI
- Ingestię i przetwarzanie dokumentów (PDF → embeddingi → vector store)
- Komunikację wielokanałową (WhatsApp, Telegram, Email)

- Kompleksowe wdrożenie chatbota (n8n → AI → żywa strona)

---

## 🛠️ Stack technologiczny

| Kategoria | Technologie |
|-----------|-------------|
| **Automatyzacja** | n8n, REST API, Webhooks, JSON |
| **AI / LLM** | OpenAI, Claude (itp), Whisper, RAG |
| **Voice AI** | ElevenLabs (konwersacyjne AI, TTS, STT) |
| **Bazy wektorowe** | Pinecone, Supabase Vector |
| **Bazy danych** | PostgreSQL, Google Sheets, Airtable |
| **Komunikacja** | WhatsApp Business API, Telegram Bot API, Gmail |
| **Automatyzacja testów** | Playwright, Selenium, REST Assured, Postman |
| **DevOps** | Git, GitHub, GitHub Actions |

---

## 📈 Co dalej

- 🔄 Wdrożenie produkcyjne projektów klienckich (Q1 2026)
- 📝 Publikacja szablonów workflow w bibliotece społeczności n8n
- 🎓 W trakcie: Python, LangChain, FastAPI
- 🔧 Kolejne projekty w miarę rozszerzania bazy klientów

---

*Zbudowane z ❤️ i n8n przez [Tatiana Golińska](https://github.com/TatianaG-ka/)*
