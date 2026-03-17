# 🤖 n8n Automation & AI Integration Portfolio

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

🇵🇱 [Wersja polska / Polish version](README_PL.md)

---

## 👋 About Me

**Tatiana Golińska** — Automation & Integration Engineer.

I'm transitioning from test automation and manual testing (5+ years with Playwright, Selenium, REST Assured) into business process automation. Currently building n8n-based automation solutions for e-commerce clients, working with REST APIs, OpenAI, and vector databases.

📧 tatiana.golinska@gmail.com · [LinkedIn](https://www.linkedin.com/in/tetiana-golinska/) · [GitHub](https://github.com/TatianaG-ka/)

---

## 🎬 Demo: Multilingual AI Chatbot 

[![Raymon AI Assistant Demo](https://img.shields.io/badge/▶_Watch_Demo-FF0000?style=for-the-badge&logo=youtube&logoColor=white)](https://youtu.be/watch?v=K9fpfK8-U-0)

A multilingual AI customer support chatbot deployed on a demo website for **Raymon Bicycles** (German e-bike brand). The bot **automatically detects the customer's language** and responds in the same language — Polish, German, English, or any other.

| Feature | Details |
|---------|---------|
| **Auto language detection** | Customer writes in any language → bot responds in the same language |
| **Knowledge base** | Product specs, pricing, warranty (Raymon CarePlus), dealer locations |
| **Voice support** | Text chat + voice call via ElevenLabs |
| **Backend** | n8n (workflow logic & API orchestration) |
| **AI** | OpenAI GPT + ElevenLabs (voice & conversational AI) |
| **Hosting** | Netlify |


> 💡 This project demonstrates end-to-end chatbot delivery: from n8n workflow design through AI integration to live deployment on a real product website.

---

## 🏢 Client Projects (in testing, production deployment planned Q1 2026)

I'm currently building automation solutions for **2 e-commerce clients**. Details are anonymized per client agreement.

| Automation | What It Does | Tech Stack |
|-----------|-------------|------------|
| **AI Customer Support Chatbot** | Answers customer questions from FAQ and product docs using RAG | n8n, OpenAI API, Pinecone, Webhooks |
| **Email Classification System** | Auto-sorts incoming customer emails into categories: complaints, questions, finance, other — routes to correct department | n8n, Gmail API, OpenAI API |
| **Invoice Processing** | Extracts data from incoming invoices via email, forwards to accounting | n8n, Gmail API, REST API |
| **Customer Reactivation Follow-ups** | Automated email sequences for inactive customers | n8n, Gmail API, OpenAI API |
| **Allegro Product Descriptions** | AI-generated product descriptions for Allegro marketplace | n8n, OpenAI API, REST API |
| **Content Automation** | Auto-generates and schedules social media posts and blog content | n8n, OpenAI API, Webhooks |

> ⚠️ Workflow JSON files for client projects are not shared publicly due to confidentiality.

---

## 📚 Learning Projects

These workflows were built while completing n8n and AI automation courses. They demonstrate my understanding of key integration patterns and are fully functional — importable into any n8n instance.

| # | Project | What It Demonstrates | Integrations |
|---|---------|---------------------|--------------|
| 01 | [**AI Marketing Team**](01-ai-marketing-team/) | Multi-tool AI agent with voice input | OpenAI GPT-4, DALL·E, Whisper, Telegram |
| 02 | [**RAG E-commerce Chatbot**](02-RAG%20E-commerce%20Chatbot/) | RAG architecture: auto-ingestion + vector search | OpenAI, Pinecone, Google Drive |
| 03 | [**Sales Automation Pipeline**](03-sales-automation-pipeline/) | Complete sales cycle with AI follow-ups | Gmail, Google Sheets, Calendly, OpenAI |
| 04 | [**Multilingual AI Chatbot**] | [![Raymon AI Assistant Demo](https://img.shields.io/badge/▶_Watch_Demo-FF0000?style=for-the-badge&logo=youtube&logoColor=white)](https://youtu.be/watch?v=K9fpfK8-U-0) | OpenAI GPT + ElevenLabs (voice & conversational AI)I |

---

**What I can build:**
- RAG chatbots with vector knowledge bases (Pinecone, Supabase)
- Multilingual AI assistants with auto language detection
- Email classification and routing automation
- Multi-step follow-up sequences with AI-generated content
- Document ingestion and processing (PDF → embeddings → vector store)
- Multi-channel communication (WhatsApp, Telegram, Email)
- End-to-end chatbot deployment (n8n → AI → live website)

---

## 🛠️ Tech Stack

| Category | Technologies |
|----------|-------------|
| **Automation** | n8n, REST API, Webhooks, JSON |
| **AI / LLM** | OpenAI, Calude, Whisper, RAG |
| **Voice AI** | ElevenLabs (conversational AI, TTS, STT) |
| **Vector DBs** | Pinecone, Supabase Vector |
| **Databases** | PostgreSQL, Google Sheets, Airtable |
| **Communication** | WhatsApp Business API, Telegram Bot API, Gmail |
| **Test Automation** | Playwright, Selenium, REST Assured, Postman |
| **DevOps** | Git, GitHub, GitHub Actions |

---

## 📈 What's Next

- 🔄 Production deployment of client projects (Q1 2026)
- 📝 Publishing workflow templates to n8n community library
- 🎓 Completing: Python, LangChain, FastAPI courses
- 🔧 More projects coming as I expand my client base

---

*Built with ❤️ and n8n by [Tatiana Golińska](https://github.com/TatianaG-ka/)*
