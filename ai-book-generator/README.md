# AI Book Generator & Interactive Reader (The Magic Quill)

This workflow turns a simple text prompt into a **multi‑chapter story** and renders it in a custom, book‑style UI.

The system has two parts:

1. A **frontend** served directly by n8n (The Magic Quill UI).
2. A **backend** webhook that calls an LLM to write a 4‑chapter mini‑book and returns structured JSON.

---

## Stack

- **n8n** – workflow engine
- **OpenRouter LLM** – `openrouter/pony-alpha` (or compatible model)
- **TailwindCSS + HTML + Vanilla JS** – front‑end UI
- **JavaScript Code nodes** – HTML template + JSON handling

---

## What the workflow does

### 1. Serve the “Magic Quill” UI

**Nodes:** `UI Webhook` → `Code in JavaScript` → `Respond to Webhook`

- `UI Webhook` exposes a **GET** endpoint at `/webhook/book-ui` (Production URL).
- `Code in JavaScript` generates a full HTML document:
  - Title: “The Magic Quill”
  - Textarea for the topic or source text.
  - “Bind into Book” button.
  - Hidden **Reader** view styled like a book:
    - paper texture
    - chapter headers
    - nicely spaced paragraphs
- `Respond to Webhook` returns this HTML with:
  - `respondWith: "text"`
  - `Content-Type: text/html`

When you open the UI Webhook’s Production URL in a browser, you see the book generator page.

Inside the HTML:

```js
const AI_URL = 'REPLACE_WITH_YOUR_AI_WEBHOOK_URL';
This must be replaced with the Production URL of the backend AI webhook (see below).

2. Generate the book via AI
Nodes: AI Webhook → AI Agent + OpenRouter Chat Model → Respond to Webhook1

AI Webhook is a POST endpoint (also at path book-ui, but POST instead of GET).

The frontend calls this endpoint with:
JSON

{ "rawText": "user's topic or pasted text" }
AI Agent:

User prompt: {{ $json.body.rawText }}

System message (key parts):

“You are a professional fantasy/historical novelist acting as a strict JSON API.”

Must output only valid JSON, no markdown, no extra text.

Structure:

JSON

{
  "bookTitle": "A creative, story-like title based on the topic",
  "chapters": [
    { "title": "Chapter 1: ...", "content": "..." },
    { "title": "Chapter 2: ...", "content": "..." },
    { "title": "Chapter 3: ...", "content": "..." },
    { "title": "Chapter 4: ...", "content": "..." }
  ]
}
4 chapters, each ~250–350 words.

Story style: scenes, characters, emotions, dialogue.

Paragraphs separated by <br><br> inside content.

OpenRouter Chat Model (language model):

Model: openrouter/pony-alpha (you can swap to any other chat model with good JSON obedience).
Connected to the AI Agent as its language model.
Respond to Webhook1:

respondWith: "text"
responseBody: {{ $json.output }}
Returns the raw JSON string from the AI Agent.
On the frontend, generateBook() does:

JavaScript

const response = await fetch(AI_URL, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ rawText: text })
});

const data = await response.json();
response.json() parses the raw JSON string into a JS object.

3. Render the book in the browser
Inside the HTML generateBook() function:

Reads data.bookTitle and sets it in the reader header.
Iterates over data.chapters:
Writes each chap.title as an <h2>.
Splits chap.content on <br> to reconstruct paragraphs.
Renders each paragraph in a styled <p> tag.
Adds a decorative divider ❦ between chapters.
Hides the generator screen and shows the reader screen.
If no chapters are returned, it shows:

HTML

<p class="text-red-500">The AI returned no chapters. Try again.</p>
The “New Book” button calls resetReader() to toggle back to the generator screen and clear the textarea.

Files
workflow.json – exported n8n workflow for the Book AI agent (with webhook IDs redacted; you must add your own credentials and URLs).
README.md – this file.

How to run this workflow:

Import into n8n
Create a new workflow.
Import workflow.json.
Configure the AI backend

Add an OpenRouter credential.
In OpenRouter Chat Model:
confirm the model name you want to use.
Ensure AI Webhook is set to:
method: POST
path: book-ui
response mode: responseNode (using Respond to Webhook1).
Get the AI Webhook Production URL

Open the AI Webhook node.
Switch to the Production URL tab.
Copy the full URL (e.g. https://your-n8n-instance/webhook/book-ui).
Paste AI URL into the HTML

In the Code in JavaScript node (UI HTML), find:

JavaScript:

const AI_URL = 'REPLACE_WITH_YOUR_AI_WEBHOOK_URL';
Replace the string with the Production URL you copied in step 3.

Serve the UI

Open the UI Webhook node.
Copy its Production URL (GET).
Open it in your browser → you should see “The Magic Quill”.
Generate a book

Type a topic like “Kings and Poverty” or “A sci‑fi story about time dilation”.
Click Bind into Book.
The frontend calls the AI webhook, then renders your 4‑chapter mini‑book.
Notes & extensions

If you want longer stories, you can:
Increase maxTokens / tokens per response.
Adjust the prompt (e.g. more chapters or higher word count per chapter).
Be mindful of n8n.cloud timeouts for very long outputs.

You could also:

- Save the generated book into a database (Notion, DB node).
- Add bookmarking / table of contents to the UI.
- Add PDF export via an HTML‑to‑PDF service.
- As provided, this workflow is a compact example of:
Serving a custom UI from n8n,
Using an LLM as a “book engine” with strict JSON output,
And rendering the result in a visually pleasant reading experience.
