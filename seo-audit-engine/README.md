# AI‑Driven Technical SEO Audit (n8n Workflow)

This project is an automated **SEO audit engine** built with **n8n** and **LLMs**.  
You give it a **URL** and a **target keyword**; it:

- fetches and parses the page
- checks `robots.txt` and `sitemap.xml`
- runs detailed, section‑by‑section SEO analysis with an LLM
- generates a final **Markdown SEO report**
- exports that report as a `.txt` file

The idea is to simulate what a senior SEO consultant would do, but in an automated, repeatable workflow.

---

## Stack

- **n8n** – orchestration and workflow logic  
- **HTTP Request + HTML Extract nodes** – to fetch and parse web pages  
- **LangChain / LLM nodes**:
  - `gpt-4o-mini` – per‑section content analyzer
  - `arcee-ai/trinity-large-preview:free` (OpenRouter) – final report generator
- **JavaScript Code nodes** – URL parsing, content chunking, and data shaping

---

## What the workflow does (high level)

1. **Collects input via a form**

   Node: **SEO Audit Form**

   - `Website URL` (required) – page you want to audit  
   - `Target Keyword` (required) – the main keyword you care about

2. **Parses the URL**

   Node: **URL Parser**

   - Normalizes the URL
   - Extracts:
     - `original_url`
     - `root_domain` (e.g. `https://example.com`)
     - `target_keyword`

3. **Sets up workflow configuration**

   Node: **Workflow Configuration**

   - Sets helper fields:
     - `url`
     - `target_keyword`
     - `robots_url` → `root_domain/robots.txt`
     - `sitemap_url` → `root_domain/sitemap.xml`

4. **Fetches and parses the page HTML**

   - **Fetch HTML Content** – HTTP GET on the target URL with a realistic `User-Agent`
   - **Extract HTML Content** – HTML node that pulls out:
     - `<title>`
     - `<h1>`
     - `<meta name="description">`
     - main content (`article`, `main`, `.content`, `.post`, etc.), while skipping:
       - scripts, styles, nav, footer, menus, ads

5. **Chunks content into “sections”**

   Node: **Format Data for Loop**

   - Builds an array of `sections`:
     - Section 1: `"Page Metadata & Identity"`  
       - contains title, H1, meta description
     - Section 2+: `"Main Content Part N"`  
       - chunks of main content (`≈ 3000` characters per chunk)
       - hard safety cap at ~4 chunks (≈12k characters) to save tokens/cost
   - Each section has:
     - `section_name`
     - `content`
     - `target_keyword`
     - `is_metadata` flag

6. **Loops over sections & runs AI analysis (Map step)**

   - **Loop Over Sections** – `SplitInBatches` processes each section sequentially
   - **Section Analysis Schema** – defines strict JSON schema for the LLM:
     - `section_name`
     - `relevance_score` (0–100)
     - `page_type_detected`
     - `critical_issues[]`
     - `optimization_recommendations[]`
     - `keyword_analysis`
   - **OpenAI Chat Model (gpt-4o-mini)** – the underlying LLM
   - **AI Section Analyzer** – LangChain Agent node that:
     - receives one `section_name + content + target_keyword`
     - returns JSON matching the schema (one object per section)

7. **Aggregates section‑level results (Reduce step, part 1)**

   Node: **Aggregate AI Responses**

   - Collects all section‑level JSON objects into `content_analysis_report`
   - This is the `aiData` for the final report step

8. **Checks robots.txt and sitemap.xml**

   - **Check Robots.txt** → HTTP request to `root_domain/robots.txt`
   - **Robots Status Check** → `If` node to see if a response exists
     - **Robots Found** – sets:
       - `robots_status = "Found"`
       - `robots_details` (first 500 chars of body)
       - `robots_status_code`
     - **Robots Missing** – sets:
       - `robots_status = "Missing"`
       - `robots_details = "No robots.txt file found"`
       - `robots_status_code`
   - **Check Sitemap.xml** → HTTP request to `root_domain/sitemap.xml`
   - **Sitemap Status Check** → `If` node similar to robots
     - **Sitemap Found** – sets `sitemap_status = "Found"` + snippet
     - **Sitemap Missing** – sets `sitemap_status = "Missing"` + status code

9. **Merges everything together**

   - **Merge All Paths** – combines AI section analysis + robots info
   - **Merge All Paths1** – further combines with sitemap info
   - Resulting object (simplified conceptual structure):

     ```json
     {
       "robotsData": { ... },
       "sitemapData": { ... },
       "aiData": [ /* array of section analysis objects */ ],
       "target_keyword": "..."
     }
     ```

10. **Generates the final SEO report (Reduce step, part 2)**

    - **OpenRouter Chat Model** – `arcee-ai/trinity-large-preview:free`
    - **Final SEO Report Generator** – Agent node with a long system prompt telling the model to:
      - interpret `robotsData`, `sitemapData`, and `aiData`
      - determine page type
      - compute a global health score
      - deduplicate issues across chunks
      - output a **Markdown report** with:
        - Executive Summary
        - Technical Health table (robots.txt / sitemap)
        - Content & Structure Deep Dive
        - 3 prioritized recommendations

11. **Exports the report as a file**

    Node: **Convert to File**

    - Takes the LLM output (Markdown text)
    - Writes it into a `.txt` file:
      - `SEO_Audit_Report.txt`
    - The file is available as binary output from the workflow (you can email it, download it, store it, etc.)

---

## Files

- `workflow.json` – the complete n8n workflow for this SEO audit engine  
  *(API keys, tokens, and instance IDs have been removed or redacted – you need to plug in your own credentials.)*

- `README.md` – this file

---

## How to run this workflow

1. **Import the workflow**

   - In n8n, create a new workflow.
   - Use “Import from file” and select `workflow.json`.

2. **Set up credentials**

   - **LLM provider**:
     - For `gpt-4o-mini` node – configure OpenAI credentials (or swap to your own model).
     - For `arcee-ai/trinity-large-preview:free` – add OpenRouter credentials.
   - **No special HTTP credentials** are required for public pages; if your target site needs auth, you’ll need to adapt the HTTP nodes.

3. **Open the SEO Audit Form URL**

   - After activating the workflow, n8n will show you a **Form URL** for the `SEO Audit Form` node.
   - Open it in a browser, enter:
     - Website URL (e.g. `https://example.com/article`)
     - Target Keyword (e.g. `technical seo audit`)
   - Submit the form.

4. **Check Executions**

   - In n8n, go to **Executions**.
   - Open the latest run; you should see the nodes:
     - Fetch HTML, Extract HTML Content
     - AI Section Analyzer / Aggregate AI Responses
     - Check Robots.txt / Check Sitemap.xml
     - Final SEO Report Generator
   - The **Convert to File** node will have a binary file: `SEO_Audit_Report.txt`.

5. **Download the report**

   - From the execution, click the **binary data** of the `Convert to File` node.
   - Download the `.txt` file and open it – you should see the full Markdown‑style SEO report.

---

## Notes & Extensions

- The workflow is designed to be **safe on tokens**:
  - main content is chunked to ~3000 characters
  - hard limit at ~4 chunks per page
- You can easily adapt it to:
  - write reports into Notion or a database instead of / in addition to `.txt`
  - schedule audits for a list of URLs instead of using the form
  - change the models to any OpenRouter / OpenAI model you prefer

This project is mainly a reference for how to combine:

- scraping + HTML parsing  
- map‑reduce style LLM analysis  
- structured JSON parsing  
- and final human‑readable reporting

inside a single n8n workflow.
