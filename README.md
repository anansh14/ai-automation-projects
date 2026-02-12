# n8n + LLM Automation Lab

This repository is a small collection of “real” automation systems I’ve built with **n8n**, **LLMs**, and a few common SaaS tools (Gmail, Notion, etc.).

The goal wasn’t to make toy demos, but to explore what a solo engineer can automate end‑to‑end:

- turning messy HTML into structured datasets
- triaging real emails with AI
- generating full books from a single prompt
- running technical SEO and career‑intelligence pipelines
- automating job search and application workflows

Each project lives in its own folder with:

- a short `README.md` that explains what it does  
- one or more exported `workflow*.json` files from n8n (with secrets removed)

If you use n8n yourself, you can import the workflows and adapt them to your own stack.

---

## Repository structure

```text
.
├─ ai-email-triage/
│  ├─ README.md
│  └─ workflow.json
|  ├─ screenshots
├─ ai-book-generator/
│  ├─ README.md
│  └─ workflow.json
|  ├─ screenshots
├─ seo-audit-engine/
│  ├─ README.md
│  └─ workflow.json
|  ├─ screenshots
├─ career-orchestrator/
│  ├─ README-hardcodedwithNotion.md
│  ├─ README-interactive.md
│  ├─ workflow-hardcodedwithNotion.json   # scheduled, hardcoded role/location + cover letters + Notion
│  └─ workflow-interactive.json           # form-driven role/location input
|  ├─ screenshots         
├─ jarvis-telegram/
│  ├─ README.md
│  └─ workflow.json
|  ├─ screenshots
├─ LICENSE
└─ README.md  ← you are here

- Projects:

1) AI Email Triage & Notion Dashboard (ai-email-triage/)
Classifies incoming Gmail via LLM (invoice / meeting / urgent / newsletter / personal / spam / other), assigns urgency & confidence, extracts summaries and action items, and logs everything into a single Notion “Email Triage – All” database.

2) AI Book Generator & Interactive Reader (ai-book-generator/)
Turns a user prompt into a structured, multi‑chapter story via LLM and renders it in a custom book‑style HTML/Tailwind reader (“The Magic Quill”) served directly by n8n.

3) AI‑Driven Technical SEO Audit Engine (seo-audit-engine/)
Scrapes and analyzes websites using a map‑reduce‑style LLM pipeline: fetches HTML, runs chunked content analysis, checks robots.txt/sitemap.xml, and generates a consolidated Markdown SEO report.

4) Autonomous & Interactive AI Career Orchestrator (career-orchestrator/)
Automates job search and shortlisting for tech roles:

    a) Autonomous workflow (workflow-autonomous.json):
    Scheduled agent that scrapes LinkedIn for a hardcoded role/location, deep‑scrapes full descriptions, uses an LLM to           rank/filter top paid roles, writes them to Notion, and emails you both a summary and tailored cover letters.
    
    b) Interactive workflow (workflow-interactive.json):
    Form‑driven version where you enter Target Role + Location; the agent runs the same scraping + ranking pipeline on            demand, sends you a summary email, and can also sync results into Notion.

5) Jarvis 2.0 – AI Telegram Chatbot (jarvis-telegram/)
Telegram bot built with n8n that routes messages between:

A chat branch (LLM answers like a concise, well‑formatted assistant), and an image branch (/generate image …), which calls an image generation node and replies with a photo. The workflow JSON is sanitized (no tokens, webhook IDs redacted, chat IDs placeholdered) and ready to import into n8n.
