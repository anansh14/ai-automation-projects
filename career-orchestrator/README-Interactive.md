# Interactive Career Agent – User‑Configured Job Finder

This workflow is an interactive version of the **Autonomous Career Orchestrator**.

Instead of using hardcoded role/location and a schedule, it exposes a **form UI** where you enter:

- your **Target Role** (e.g. “Data Scientist”)
- your **Location** (e.g. “Remote, India”, “New York”)

The agent then:

1. Scrapes LinkedIn jobs for that role/location.
2. Deep‑scrapes each job’s full description.
3. Uses an LLM to rank/filter the best opportunities for that specific role.
4. Sends you a **summary email** of the top jobs.
5. Optionally writes them into a **Notion** database.

> ⚠️ This workflow scrapes LinkedIn pages & jobPosting APIs as a learning project.  
> Be mindful of LinkedIn’s Terms of Service and don’t use it aggressively or commercially.

---

## Stack

- **n8n** – workflow engine  
- **LinkedIn job search + jobPosting API** – data source  
- **OpenRouter LLMs** – `google/gemma-3-12b-it:free` for ranking/filtering  
- **Gmail** – sends the summary email  
- **Notion** – optional logging of top jobs  
- **JavaScript Code nodes** – scraping, deduplication, and JSON parsing  
- **Form Trigger + HTML** – simple UI for user input / demo console

---

## How the workflow works

### 1. User Input (form trigger)

**Node:** `User Input` (`formTrigger`)

- Exposes a form with:
  - `Target Role` (field `role`)
  - `Location` (field `location`)
- When the user submits the form, it triggers the workflow with:

```json
{
  "role": "Data Science Intern",
  "location": "India"
}
2. Build the LinkedIn search URL
Node: Code in JavaScript

Reads role and location from User Input:

JavaScript:

const query = $('User Input').first().json.role;
const loc = $('User Input').first().json.location;
Builds a LinkedIn search URL like:

text:

https://www.linkedin.com/jobs/search?keywords=...&location=...&f_TPR=r86400
f_TPR=r86400 → last 24 hours
Optional f_WT=2 if you set wantRemote = true.
Outputs { url } for the next node.

3. Scrape the LinkedIn search results
Nodes: HTTP Request → HTML → Code in JavaScript1

HTTP Request:

Fetches the LinkedIn search page with a realistic User-Agent, Accept-Language, and Referer.
HTML node:

Extracts arrays for:
title[] – .base-search-card__title
company[] – .base-search-card__subtitle
location[] – .job-search-card__location
link[] – .base-card__full-link[href]
Code in JavaScript1:

Cleans company names (remove newlines, trim).

Deduplicates by company so you don’t get duplicates.

Extracts jobId from each URL (e.g. /view/4369367348/).

Limits to 8 jobs to reduce ban risk.

Outputs items like:

JSON:

{
  "title": "...",
  "company": "...",
  "location": "...",
  "link": "https://www.linkedin.com/jobs/view/...",
  "jobId": "4369367348"
}
4. Deep‑scrape each job posting
Nodes: Loop Over Items (SplitInBatches) → HTTP Request1 → HTML1 → Code in JavaScript3 → back to Loop Over Items

For each job:

HTTP Request1:

Calls https://www.linkedin.com/jobs-guest/jobs/api/jobPosting/{{ jobId }}
HTML1:

Extracts .description__text → full job description.
Code in JavaScript3:

Merges the original job info with full_description (truncated to 3,000 chars).
Adds a small 1.5s delay as a naive anti‑ban measure.
Sends updated item back into Loop Over Items.
After this loop, each item has:

JSON:

{
  "title": "...",
  "company": "...",
  "location": "...",
  "link": "...",
  "full_description": "long job description text..."
}
}
5. Aggregate jobs into a single list
Node: Code in JavaScript4

Collects all looped items into:
JSON

{
  "job_list": [
    {
      "title": "...",
      "company": "...",
      "location": "...",
      "link": "...",
      "description": "full_description..."
    },
    ...
  ]
}
This is what you send to the ranking LLM.

6. Rank and filter jobs for the chosen role
Nodes: AI Agent + OpenRouter Chat Model

System prompt: “Elite Recruiter specializing in {{ role }} roles.”

Input: job_list JSON.
Instructions:
Select the Top 5 best opportunities matching the specific role.
Prioritize paid or stipend‑based roles.
Reject explicit “Unpaid” roles, unless extremely prestigious.
Boost roles whose descriptions clearly match the tools/skills for that role.
Output: a JSON array like:

JSON:

[
  {
    "company": "Inficore Soft",
    "title": "Machine Learning Intern",
    "location": "India",
    "pitch": "Paid internship (₹14,500) with a focus on Python, Scikit-learn...",
    "link": "https://...",
    "description": "full job description..."
  },
  ...
]
7. Build summary email + jobs_raw
Node: Code in JavaScript2

Cleans AI output (removes ``` fences, isolates [ ... ]).

Builds a human‑readable email_body listing the top jobs:

company
title
location
pitch
link
Builds jobs_raw[] for potential downstream steps (e.g., cover letters in the autonomous version).

Output:

JSON:

{
  "email_body": "...",
  "jobs_raw": [ { company, title, location, pitch, link, description }, ... ],
  "debug_raw_ai": "..."  // full LLM output for debugging
}
Node: Send a message (Gmail)

Sends email_body to your_email@example.com (placeholder – replace locally).
8. Save to Notion (optional)
Nodes: Code in JavaScript6 → Create a database page (Notion)

Code in JavaScript6:

Parses the JSON array from the AI.

Outputs one item per job:

JSON:

{ company, title, location, pitch, link }
Create a database page:

Writes each item into your Notion database:
Company (Title)
Role (Text)
Location (Text)
Pitch (Text)
Link (URL)
HTML demo console (optional UI)
Node: HTML2

Serves a static landing page with:

A nice “Configure Your Career Agent” hero section.
An input form for role + location.
A fake “terminal” that visually shows the agent “running”.
The JS in this page currently simulates the process and (optionally) can be wired to:

JavaScript:

// fetch('YOUR_N8N_WEBHOOK_URL', {
//   method: 'POST',
//   body: JSON.stringify({ role: role, location: location })
// });
So you can either:

Use the formTrigger (User Input) as the primary entrypoint, or
Adapt the HTML console to call an n8n webhook instead, if you want a more polished front‑end.
Files
workflow-interactive.json – this workflow (user‑input driven agent)
workflow-hardcoded.json – the autonomous version (scheduled, fixed role/location + cover letters) (in a separate file)
README.md – main project description (see parent folder)

How to run the interactive workflow

Import the workflow
Create a new workflow in n8n.
Import workflow-interactive.json.
Configure credentials

Gmail credential for the Send a message node.
OpenRouter for the OpenRouter Chat Model node.
Notion (optional) and set your databaseId in Create a database page.
Open the form URL

Open the User Input node.
Copy the Production URL.
Open it in your browser: you’ll see the “Autonomous Career Agent” form.
Submit role + location

Enter something like:
- Role: Machine Learning Intern
- Location: India or Remote
- Submit the form → the workflow runs:
- Scrapes LinkedIn
- Filters/ranks jobs
- Sends an email with the top roles
- Writes jobs to Notion (if configured)
- Check Executions

In n8n, open Executions to see each node’s output when debugging.
This interactive version shares the core logic of the autonomous agent but lets you control the role and location dynamically.






