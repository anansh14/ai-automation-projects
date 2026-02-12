# n8n + LLM Automation Lab

This repository is a small collection of “real” automation systems I’ve built with **n8n**, **LLMs**, and a few common SaaS tools (Gmail, Notion, etc.).

The goal wasn’t to make toy demos, but to explore what a solo engineer can automate end‑to‑end:

- turning messy HTML into structured datasets
- triaging real emails with AI
- generating full books from a single prompt
- running technical SEO and career‑intelligence pipelines

Each project lives in its own folder with:

- a short `README.md` that explains what it does
- an exported `workflow.json` from n8n (with secrets removed)

If you use n8n yourself, you can import the workflows and adapt them to your own stack.

---

## Repository structure

```text
.
├─ ai-email-triage/
│  ├─ README.md
│  └─ workflow.json
├─ ai-book-generator/
│  ├─ README.md
│  └─ workflow.json
├─ seo-audit-engine/
│  ├─ README.md
│  └─ workflow.json
├─ career-orchestrator/
│  ├─ README.md
│  └─ workflow.json
├─ LICENSE
└─ README.md  ← you are here


Projects:

1) AI Email Triage & Notion Dashboard (ai-email-triage/)
Classifies incoming Gmail with an LLM (invoice / meeting / urgent / etc.), extracts structured metadata, and logs everything to a single Notion “Email Triage – All” database.

2) AI Book Generator & Interactive Reader (ai-book-generator/)
Turns a user prompt into a multi‑chapter story via LLM, then renders it in a custom book‑style HTML/Tailwind reader.

3) AI‑Driven Technical SEO Audit Engine (seo-audit-engine/)
Scrapes and analyzes websites using a map‑reduce‑style LLM pipeline, combining technical checks with content/SEO insights.

4) Autonomous AI Career Orchestrator (career-orchestrator/)
Continuously scrapes job listings, cleans them into structured data, and applies semantic ranking (Gemma 2 9B) for career‑intelligence use cases.
