# Autonomous Job Finder + Cover Letter Generator

This workflow is an **AI‑assisted career agent** built in **n8n**.

On a schedule, it:

1. Scrapes LinkedIn job listings for a specific role/location.
2. Deep‑scrapes each job’s full description.
3. Uses an LLM to rank and filter the best opportunities.
4. Sends you:
   - a summarized email of the top jobs, and  
   - a second email with **tailored cover letters** for each job.
5. Logs the top jobs into a **Notion** database.

It’s designed as a “hands‑off” helper that does the boring part of job hunting for you.

> ⚠️ **Note:** This workflow scrapes LinkedIn HTML and their public jobPosting API as a personal experiment.  
> Be mindful of LinkedIn’s Terms of Service and use it responsibly.

---

## Stack

- **n8n** – workflow engine
- **LinkedIn jobs pages** (guest search + jobPosting API)
- **OpenRouter LLMs**  
  - `google/gemma-3-12b-it:free` – ranks/filter jobs  
  - `tngtech/deepseek-r1t-chimera:free` – writes cover letters
- **Gmail** – sends summary + cover‑letter emails
- **Notion** – optional database for saving top jobs
- **JavaScript Code nodes** – scraping, deduplication, parsing, email/JSON formatting

---

## What the workflow does (step by step)

### 1. Schedule + profile configuration

**Nodes:** `Schedule Trigger` → `Profile Config`

- The workflow is scheduled to run daily at a specific hour.
- `Profile Config` sets a **hardcoded mini‑resume** (`my_resume`) used to condition the cover‑letter model.
  - This is sample text in the public version; replace it with your real resume locally.

### 2. Build the LinkedIn search URL

**Nodes:** `Code in JavaScript` → `HTTP Request` → `HTML`

- `Code in JavaScript` builds a LinkedIn **job search URL** using:
  - `query` = `"Data Science Intern"` (edit this)
  - `loc` = `"India"` (edit this)
  - `wantRemote` = `true/false`
- `HTTP Request` fetches that URL with a realistic `User-Agent` and language headers.
- `HTML` node extracts arrays from the search page:
  - `.base-search-card__title` → job titles
  - `.base-search-card__subtitle` → companies
  - `.job-search-card__location` → locations
  - `.base-card__full-link[href]` → job links

### 3. Clean & deduplicate jobs

**Node:** `Code in JavaScript1`

- Cleans company names (strips newlines, trims text).
- Extracts **jobId** from each LinkedIn job URL.
- Deduplicates by company (only first job per company).
- Limits to **8 jobs** to avoid hitting LinkedIn too hard.
- Outputs items like:

```json
{
  "title": "...",
  "company": "...",
  "location": "...",
  "link": "https://www.linkedin.com/jobs/...",
  "jobId": "4369367348"
}
4. Deep‑scrape each job description
Nodes: Loop Over Items (SplitInBatches) → HTTP Request1 → HTML1 → Code in JavaScript3 → back to Loop Over Items

For each job from the previous step:

HTTP Request1 calls https://www.linkedin.com/jobs-guest/jobs/api/jobPosting/{{ jobId }}.
HTML1 extracts the .description__text element (full job description).
Code in JavaScript3:
merges the original job info with full_description
truncates very long descriptions
adds a small delay (sleep) per job as a basic anti‑ban measure
sends it back to Loop Over Items for the next job
After the loop, each item looks like:

JSON

{
  "title": "...",
  "company": "...",
  "location": "...",
  "link": "...",
  "full_description": "long job description text..."
}
5. Aggregate jobs into one list
Node: Code in JavaScript4

Collects all items and returns a single object:
JSON

{
  "job_list": [
    { "title": "...", "company": "...", "location": "...", "link": "...", "description": "..." },
    ...
  ]
}
This job_list is what we send to the recruiter LLM.

6. Rank and filter jobs with an LLM
Nodes: AI Agent + OpenRouter Chat Model

System prompt: “Elite Tech Recruiter”.
Input: job_list JSON.
The model is asked to:
reject unpaid or very vague roles,
prioritize jobs with clear stipend/salary,
prefer roles mentioning tools like Python, SQL, Pandas, scikit‑learn, AWS, etc.
Output: a JSON array of the top opportunities, each with:
company, title, location, pitch, link, description
7. Build a summary email + raw job list
Node: Code in JavaScript2

Cleans the AI output (removes ```json fences, isolates [ ... ]).

Builds a human‑readable email_body listing the top jobs:

company
title
location
short pitch
link
Builds jobs_raw array for use in the cover‑letter generator.

Output:

JSON

{
  "email_body": "...",
  "jobs_raw": [ { company, title, location, pitch, link, description }, ... ],
  "debug_raw_ai": "..."  // raw LLM output for debugging
}
Node: Send a message (Gmail)

Sends email_body to your email as a “Top Jobs Found” summary.
8. Save jobs into Notion
Nodes: Code in JavaScript6 → Create a database page (Notion)

Code in JavaScript6 extracts the ranked jobs JSON and outputs separate items:
JSON

{ "company": "...", "title": "...", "location": "...", "pitch": "...", "link": "..." }
Create a database page creates a row per job in your Notion database (you must replace REPLACE_WITH_YOUR_NOTION_DATABASE_ID with your own DB ID locally).
9. Generate tailored cover letters
Nodes: Split Out → AI Agent1 + OpenRouter Chat Model1 → Code in JavaScript5 → Send a message1 (Gmail)

Split Out takes jobs_raw and emits one item per job.

AI Agent1:

gets the job JSON (company, title, location, description, etc.)

reads your resume from Profile Config.my_resume

writes one JSON object per job:

JSON

{
  "company": "...",
  "location": "...",
  "cover_letter": "full cover letter text...",
  "resume_bullets": ["...", "...", "..."]
}
Code in JavaScript5:

cleans any extra text around the JSON (finds first { and last }).
builds a long email_body with all cover letters, nicely formatted.
Send a message1 (Gmail) sends this as a second email:

a list of jobs, each with its tailored cover letter.
Files
workflow.json – complete n8n workflow (with webhookId and instanceId redacted; you must add your own credentials).
README.md – this file.
You may also want to keep a Notion template for the jobs table in a separate doc.

How to use this workflow:

Import into n8n

Create a new workflow in n8n.
Import workflow.json.
Set up credentials (locally, not in GitHub)

Gmail credential for the two “Send a message” nodes.
OpenRouter credentials for the two lmChatOpenRouter nodes (Gemma & DeepSeek models).
Notion credential + replace REPLACE_WITH_YOUR_NOTION_DATABASE_ID with your own DB ID.
Customize your profile

Edit the Profile Config node:
Replace the sample my_resume text with your real profile (or keep the sample in the public repo and change it only on your private clone).
Configure the search

In the first Code in JavaScript node:
Change query = "Data Science Intern" → your target role.
Change loc = "India" → your preferred location.
Set want Remote = true if you only want remote roles.
Run it

You can:
Trigger manually, or
Enable the Schedule Trigger to run once a day.
Watch the Executions tab:
See which jobs were scraped and filtered.
Confirm that:
summary email arrives,
cover‑letter email arrives,
Notion table updates.


Notes & disclaimers
This workflow scrapes LinkedIn job pages and their guest jobPosting API, which may violate their terms of service if used aggressively or commercially. Treat this as a personal learning project.
The anti‑ban logic here is very light (small sleep and job cap). If you run it frequently, be considerate.
Do not commit your real email, Notion IDs, or API keys to GitHub – keep those only in your private n8n instance.
This project is a good example of combining:

- structured scraping,
- LLM‑based ranking and filtering,
- personalized content generation (cover letters),
- and practical integrations (Email + Notion)
- into one coherent “career automation” workflow.

