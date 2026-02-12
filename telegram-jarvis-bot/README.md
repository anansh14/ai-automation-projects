# Jarvis 2.0 – AI Telegram Chatbot (n8n + OpenRouter + Image Gen)

Jarvis 2.0 is a Telegram assistant built on **n8n** that:

- answers messages using an LLM, and  
- can generate images on command.

It routes each incoming Telegram message through either a **chat branch** (default) or an **image branch** based on a simple command.

---

## Stack

- **n8n** – workflow orchestration and credentials
- **Telegram Bot API** – trigger and replies
- **OpenRouter LLM** – `allenai/molmo-2-8b:free` (or any compatible model)
- **OpenAI / image node** – for image generation (can be swapped for another provider)
- **JavaScript Code / Set nodes** – minimal glue logic

---

## Features

- **Real‑time Telegram trigger** – `telegramTrigger` node listens for new messages.
- **Chat branch** – normal messages are sent to an LLM that replies like a concise, well‑formatted assistant.
- **Image branch** – messages starting with `/generate image ...` are routed to an image generation node and sent back as a photo.
- **Simple routing** – an `If` node uses a regex to decide whether a message is a chat or image request.
- **Sanitized workflow JSON** – no bot tokens or API keys in `workflow.json`; webhook IDs and chat IDs are placeholders.

---

## Architecture

High level:

```text
[ Telegram Trigger ]
        |
        v
[ If  — message matches /^\/?generate image\b/ ? ]
       /                      \
   (true)                      (false)
     |                            |
[ Generate an image ]        [ Jarvis 2.0 (LLM) ]
     |                            |
[ Send photo message ]       [ Output → Send text message ]
Nodes
Telegram Trigger

Listens to incoming updates (message) from your Telegram bot.
If

Checks message.text against the regex:

text:

^\/?generate image\b
If it matches (e.g. /generate image a cat, generate image a logo), it sends the item to the image path; otherwise to the chat path.

Jarvis 2.0 (AI Agent)

LLM agent node that receives:

text:

{{ $json.message.text }}

Answer this like a professional AI and 
keep the answers crisps and well formatted.
Connected to an OpenRouter chat model (allenai/molmo-2-8b:free by default).

Produces a text reply in json.output.

Output (Set)

Copies json.output into json.Output for clarity and to decouple from the LLM node.
Send a text message

Telegram node that sends {{ $json.Output }} back to a chat.
In this template, chatId is set to REPLACE_WITH_YOUR_CHAT_ID (you must change this locally).
Generate an image

OpenAI image node that uses message.text as the prompt.
You can adjust the model/provider in this node.
Send a photo message

Telegram node that sends the generated image as a photo.
Also uses chatId = REPLACE_WITH_YOUR_CHAT_ID in the template.
Files
workflow.json – sanitized n8n workflow:

credentials removed
webhook IDs redacted
Telegram chatId set to REPLACE_WITH_YOUR_CHAT_ID placeholder
You can import this file directly into n8n and then map your own credentials.

(Optional)

screenshots/ – you can add:
a screenshot of the workflow canvas
a screenshot or GIF of Jarvis replying in Telegram
Quick Start
1. Import the workflow
In n8n, go to Workflows → Import from File.
Select workflow.json from the jarvis-telegram/ folder.
2. Configure credentials
Create credentials in n8n and assign them to the nodes:

Telegram API
For:
Telegram Trigger
Send a text message
Send a photo message
OpenRouter
For OpenRouter Chat Model (connected to Jarvis 2.0).
OpenAI / image provider (optional)
For Generate an image node, if you keep the default provider.
The JSON intentionally ships with no credentials attached.

3. Update placeholders
In both Telegram send nodes:

Replace:

JSON:

"chatId": "REPLACE_WITH_YOUR_CHAT_ID"
With either:

your own chat ID (for a private bot), or

dynamic chat id from the trigger (recommended):

JSON:

"chatId": "={{ $json.message.chat.id }}"
If you change the image provider, update the model/endpoint in Generate an image accordingly.

4. Activate and test
Activate the workflow in n8n.

In Telegram:

Send a normal message to your bot
→ you should get a text reply from Jarvis (chat branch).

Send:

text:

/generate image a cat in a spacesuit
→ you should receive a photo (image branch).

Configuration notes
Image regex:

Default is ^\/?generate image\b.

It matches commands that start with /generate image or generate image.
You can extend it to something like:

text:

^\/?(draw|image|imagine|generate image)\b
if you want multiple triggers.

Prompt style:
The Jarvis 2.0 node currently adds a short instruction to keep responses “professional, crisp, and well formatted”. You can reinforce markdown formatting, length limits, or persona by editing that text.

Output node:
The Output Set node is a simple pass‑through today. You can add post‑processing here (e.g. trimming replies, censoring certain URLs, etc.) before they’re sent to Telegram.

Security:
This workflow JSON does not contain secrets:

Bot tokens, API keys, webhooks, and chat IDs are not embedded.
instanceId and webhookId are redacted or placeholders.
Always:

Keep your n8n instance behind authentication and HTTPS.
Rotate:
Telegram bot token
OpenRouter API key
OpenAI token (if used)
If you ever accidentally commit a non‑sanitized workflow.
Troubleshooting
Bot receives nothing

Ensure Telegram credentials are correctly mapped on:
Telegram Trigger
both send nodes.
Confirm n8n is reachable from Telegram (webhook registration or polling is working).
Image command returns text instead of photo

Check that your message starts with /generate image ... and the regex in If matches.
Verify the image node has valid credentials and a working model.
LLM replies are too verbose or off‑tone

Tighten the Jarvis 2.0 prompt used in the Agent node (e.g., fewer sentences, bullet list only, no disclaimers).
Possible extensions:
- Add /help and /about commands using extra branches.
- Add a small memory/database layer (e.g., Notion or Postgres) to persist notes.
- Add “tools” / function‑calling:
- web search
- calculator
- small scrapers
- Add rate‑limits or per‑chat moderation for shared bots.
