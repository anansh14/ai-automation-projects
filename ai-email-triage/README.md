# AI Email Triage & Notion Dashboard

This workflow turns an ordinary Gmail/IMAP inbox into a structured triage dashboard.

For every incoming email it:

1. Reads the email via IMAP.
2. Cleans the body (removes quoted history).
3. Sends the email to an LLM that returns a strict JSON schema.
4. Normalizes / parses the JSON safely in a Code node.
5. Creates a row in a single **Notion** database with all triage details.

It’s designed as a small “email ops layer” you can build further automation on top of.

---

## Stack

- **n8n** – workflow engine  
- **IMAP (Gmail or other)** – email ingestion  
- **OpenRouter LLM** – `openrouter/pony-alpha` for triage  
- **Notion** – central dashboard for triage results  
- **JavaScript Code nodes** – cleaning, parsing, normalization

---

## What the workflow does

### 1. Email Trigger (IMAP)

**Node:** `Email Trigger (IMAP)` (`emailReadImap`)

- Connects to your mailbox via IMAP.
- `trackLastMessageId: true` so it only processes *new* emails.
- You configure IMAP credentials in n8n (Gmail app password or other provider).

### 2. Clean Email Body

**Node:** `Code in JavaScript`

Takes the raw IMAP item and normalizes it to:

```json
{
  "subject": "...",
  "from": "...",
  "to": "...",
  "date": "...",
  "body": "cleaned plain-text body"
}
Key points:

Uses textPlain from IMAP output (or you can switch to text if your node uses that field).
Removes quoted history lines using markers like:
On ... wrote:
From:
>
-----Original Message-----
So the LLM only sees the latest message content.

3. AI Triage (LLM)
Nodes: AI Agent + OpenRouter Chat Model

The User message is a formatted string:

text

From: ...
To: ...
Date: ...
Subject: ...

Body:
...
The System message instructs the model to act as an email triage assistant and to output only valid JSON in this schema:

JSON

{
  "category": "urgent | invoice | meeting | newsletter | personal | spam | other",
  "urgency": 1,
  "confidence": 0.0,
  "summary": "short summary in 1–3 sentences",
  "action_items": ["...", "..."],
  "suggested_reply": "short, polite reply or empty string",
  "metadata": {
    "invoice_amount": null,
    "invoice_due_date": null,
    "meeting_datetime": null,
    "meeting_location": null,
    "related_project": null
  }
}
urgency is an integer 1–5 (1 = low, 5 = critical).

confidence is a float 0.0–1.0 (how sure the model is).

category must be one of the fixed options.

The underlying language model is configured in OpenRouter Chat Model as openrouter/pony-alpha (you can change to any supported model).

4. Parse & normalize JSON
Node: Code in JavaScript1

This Code node:

Reads the raw LLM text output ($json.output / other fallbacks).
Strips any json backtick fences.
Attempts JSON.parse(cleaned).
If parsing fails, uses a fallback object with:
category: "other"
summary: "Failed to parse JSON from LLM."
_raw and _error for debugging.
Then it normalizes the triage object:

Ensures metadata exists.
Converts potential null / undefined to "" for:
metadata.meeting_location
metadata.meeting_datetime
metadata.related_project
Ensures suggested_reply is always a string (fallback "").
Finally, it attaches the original email metadata from the clean node:

JavaScript:

const cleanNode = $item(0).$node["Code in JavaScript"].json;

triage._subject = cleanNode.subject;
triage._from = cleanNode.from;
triage._to = cleanNode.to || '';
So the final object looks like:

JSON

{
  "category": "invoice",
  "urgency": 2,
  "confidence": 0.95,
  "summary": "...",
  "action_items": ["...", "..."],
  "suggested_reply": "...",
  "metadata": {
    "invoice_amount": 99.5,
    "invoice_due_date": null,
    "meeting_datetime": "",
    "meeting_location": "",
    "related_project": "n8n IMAP connection"
  },
  "_subject": "Hello, ...",
  "_from": "sender@example.com",
  "_to": "you@example.com"
}

5. Write to Notion
Node: Create a database page (Notion)

Resource: databasePage
Operation: Create Database Item
Database: REPLACE_WITH_YOUR_NOTION_DATABASE_ID (you must set this locally)
It maps the triage result into Notion properties:

Category (Multi-select)
={{ $json.category }}

Related Project (Rich text)
={{ $json.metadata.related_project }}

Invoice Amount (Number)
={{ $json.metadata.invoice_amount }}

Email Subject (Title)
={{ $json._subject }}

From (Rich text)
={{ $json._from }}

Suggested reply (Rich text)
={{ $json.suggested_reply }}

Summary (Rich text)
={{ $json.summary }}

Urgency (Number)
={{ $json.urgency }}

Location (Rich text)
={{ $json.metadata.meeting_location }}

Meeting Time (Rich text)
={{ $json.metadata.meeting_datetime }}

confidence (Number)
={{ $json.confidence }}

This creates a single unified “Email Triage – All” table in Notion where you can filter by Category, Urgency, Confidence, etc.

Notion schema (recommended)
Create a database in Notion with at least these properties:

Email Subject – Title
From – Rich text
To (optional) – Text / Rich text
Category – Multi-select
Options: invoice, meeting, urgent, newsletter, personal, spam, other
Urgency – Number
Confidence – Number
Summary – Rich text
Action Items (optional, if you extend the workflow) – Rich text
Suggested reply – Rich text
Invoice Amount – Number
Meeting DateTime – Text
Location – Text
Related Project – Text
Raw JSON (optional) – Text
Make sure the Category property has options that match the schema; otherwise Notion will reject values like "invoice".

Files
workflow.json – exported n8n workflow for AI Email Triage
(secrets and real database IDs are not included; you must configure credentials in your own n8n instance).
README.md – this file

How to run this workflow

Import into n8n
Create a new workflow.
Import workflow.json.
Configure IMAP

Create an IMAP credential (e.g. Gmail with app password).
Attach it to Email Trigger (IMAP).
Ensure IMAP returns textPlain (or adapt the code if the field name is text).
Configure OpenRouter

Add an OpenRouter credential.
Use it in OpenRouter Chat Model (model: openrouter/pony-alpha or your choice).
Configure Notion

Create the “Email Triage – All” DB with the schema above.
Create a Notion credential.
Replace REPLACE_WITH_YOUR_NOTION_DATABASE_ID with your actual database ID.
Test the Notion node once from n8n to ensure property names are correct.
Activate and test

Set the workflow to Active.
Send yourself a few test emails:
- an invoice‑style email
- a meeting invite
- an urgent incident message
- Check Executions in n8n and verify new rows appear in Notion with correct category, urgency, confidence, and details.
