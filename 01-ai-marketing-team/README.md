# 🎨 AI Marketing Team Agent

> 📚 **Learning Project** — Built during n8n automation course. Demonstrates working knowledge of the integration patterns used in my [client projects](../README.md#-client-projects-in-testing-production-deployment-planned-q1-2026).
>
> 📚 **Projekt edukacyjny** — Zbudowany podczas kursu automatyzacji n8n. Demonstruje praktyczną znajomość wzorców integracyjnych używanych w moich [projektach klienckich](../README_PL.md#-projekty-klienckie-w-fazie-testów-wdrożenie-produkcyjne-planowane-q1-2026).


![n8n](https://img.shields.io/badge/n8n-EA4B71?style=flat-square&logo=n8n&logoColor=white)
![OpenAI](https://img.shields.io/badge/GPT--4o-412991?style=flat-square&logo=openai&logoColor=white)
![Telegram](https://img.shields.io/badge/Telegram-26A5E4?style=flat-square&logo=telegram&logoColor=white)

**A multi-tool AI marketing assistant accessible via Telegram that creates images, writes blog posts, generates LinkedIn content, and processes voice commands.**


![Workflow Screenshot](01-ai-marketing-team/screenshots/workflow-canvas.png)


---

## ✨ Features

- **Image generation** — Create images from text descriptions
- **Image editing** — Modify existing images with AI-powered edits
- **Blog post writing** — Generate full blog articles on any topic
- **LinkedIn post creation** — Write professional LinkedIn content
- **Video script generation** — Create video content scripts
- **Voice input** — Send voice messages on Telegram; auto-transcribed
- **Conversation memory** — Remembers context across messages

## 🔧 How It Works

```mermaid

    A[Telegram Message] --> B{Text or Voice?}
    B -->|Voice| C[Download Audio]
    C --> D[Transcribe recording OpenAI]
    D --> E[AI Marketing Agent]
    B -->|Text| E
    E --> F{Which Tool?}
    F -->|Create Image| G[Image Creator]
    F -->|Edit Image| H[Image Editor]
    F -->|Blog Post| I[Blog Writer]
    F -->|LinkedIn| J[LinkedIn Writer]
    F -->|Video| K[Video Script]
    F -->|Think| L[Research & Plan]
    G --> M[Telegram Response]
  
```

# 🇵🇱 Wersja polska

## 🎨 Agent AI — Zespół Marketingowy

**Wielonarzędziowy asystent marketingowy AI dostępny przez Telegram — tworzy obrazy, pisze posty na bloga, generuje treści na LinkedIn i przetwarza komendy głosowe.**

### Funkcje
- **Generowanie obrazów** — Tworzenie grafik z opisów tekstowych
- **Edycja obrazów** — Modyfikacja istniejących grafik z użyciem AI
- **Pisanie postów na blog** — Generowanie pełnych artykułów na dowolny temat
- **Posty na LinkedIn** — Profesjonalne treści na LinkedIn
- **Skrypty wideo** — Tworzenie scenariuszy do materiałów wideo
- **Komendy głosowe** — Wiadomości głosowe na Telegramie, automatyczna transkrypcja
- **Pamięć rozmów** — Zapamiętuje kontekst między wiadomościami


