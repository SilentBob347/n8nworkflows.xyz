Log daily Google Jobs listings to Google Sheets using Apify

https://n8nworkflows.xyz/workflows/log-daily-google-jobs-listings-to-google-sheets-using-apify-20970


# Log daily Google Jobs listings to Google Sheets using Apify

### 1. Workflow Overview

The purpose of the **Daily Google Jobs digest in Google Sheets with Apify** workflow is to automate daily job market monitoring. It targets job seekers, recruiters, and sales teams by periodically scraping targeted job listings from Google Jobs and recording them into a centralized Google Sheets document.

The workflow operates in a single linear sequence broken down into the following logical blocks:
- **1.1 Trigger & Configuration:** Initiates the workflow on a daily schedule and defines the target job queries, geographic region, and country parameters.
- **1.2 Data Scraping:** Executes an external scraping actor via Apify to retrieve raw job listings based on the defined search parameters.
- **1.3 Data Filtering & Transformation:** Cleans the scraped dataset to ensure only valid records pass through, then formats and timestamps each individual job entry into a structured row.
- **1.4 Persistence:** Appends the formatted daily job records into the designated Google Sheets worksheet.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Trigger & Configuration
- **Overview:** This block establishes the operational cadence of the workflow and injects the necessary search criteria (job titles, country, and location) into the data pipeline.
- **Nodes Involved:** 
  - `Every morning`
  - `Set the jobs to watch`
- **Node Details:**
  - **Every morning**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.2) - Time-based trigger node.
    - *Configuration Choices:* Configured to trigger daily at hour 07:00.
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Output connects to `Set the jobs to watch`.
    - *Edge Cases/Failures:* Missed executions if n8n instance is offline at 07:00 (depends on n8n missed-execution settings).
  - **Set the jobs to watch**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4) - Data transformation/assignment node.
    - *Configuration Choices:* Assigns static variables for downstream processing.
    - *Key Expressions/Variables:* 
      - `queries`: Array `["data engineer", "product manager"]`
      - `country`: String `"US"`
      - `location`: String `"Austin,Texas"`
    - *Input/Output Connections:* Input from `Every morning`; output connects to `Search Google Jobs`.
    - *Edge Cases/Failures:* Malformed JSON arrays or syntax errors in string assignments.

#### Block 1.2: Data Scraping
- **Overview:** This block interfaces with the Apify platform to execute the Google Jobs scraper actor using the configuration variables defined in the previous block.
- **Nodes Involved:**
  - `Search Google Jobs`
- **Node Details:**
  - **Search Google Jobs**
    - *Type and Technical Role:* `@apify/n8n-nodes-apify.apify` (v1) - Cloud service integration node.
    - *Configuration Choices:* Executes the `hirepulse~google-jobs-scraper` actor, requesting dataset results with memory allocation set to 256MB and a maximum total charge cap of $1 USD.
    - *Key Expressions/Variables:* 
      - Custom body expression: `={{ JSON.stringify({ queries: $json.queries, country: $json.country, location: $json.location, max_jobs_per_query: 50 }) }}`
    - *Input/Output Connections:* Input from `Set the jobs to watch`; output connects to `Keep real job records`.
    - *Edge Cases/Failures:* API authentication errors, insufficient Apify account balance/plan limits (free plans return limited previews), actor execution timeouts, or rate limiting. Requires valid Apify credentials.

#### Block 1.3: Data Filtering & Transformation
- **Overview:** This block validates the incoming raw scraper results to discard empty or invalid records, then maps and formats the remaining valid entries alongside a current date timestamp.
- **Nodes Involved:**
  - `Keep real job records`
  - `Shape today's rows`
- **Node Details:**
  - **Keep real job records**
    - *Type and Technical Role:* `n8n-nodes-base.filter` (v2.2) - Data conditioning node.
    - *Configuration Choices:* Uses version 2 evaluation logic with case-sensitive loose type validation using an `AND` combinator.
    - *Key Expressions/Variables:* 
      - Left Value: `={{ $json.title }}`
      - Operator: `exists` (single value check).
    - *Input/Output Connections:* Input from `Search Google Jobs`; output connects to `Shape today's rows`.
    - *Edge Cases/Failures:* If the scraper response schema changes and `title` is moved or renamed, all records may be filtered out.
  - **Shape today's rows**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) - Custom JavaScript execution node.
    - *Configuration Choices:* Iterates over all incoming items using JavaScript to flatten, normalize, and timestamp the payload.
    - *Key Expressions/Variables:* 
      - Uses `$now.toFormat('yyyy-MM-dd')` for date generation.
      - Accesses properties such as `j.query`, `j.title`, `j.company`, `j.location`, `j.salary`, `j.salary_min`, `j.salary_max`, `j.contract_type`, `j.posted_ago`, `j.source`, and `j.apply_url` (with nullish coalescing fallbacks).
    - *Input/Output Connections:* Input from `Keep real job records`; output connects to `Append today's jobs`.
    - *Edge Cases/Failures:* JavaScript runtime errors if input data structures do not match expected object keys.

#### Block 1.4: Persistence
- **Overview:** This block takes the formatted job entries and commits them to an external spreadsheet document.
- **Nodes Involved:**
  - `Append today's jobs`
- **Node Details:**
  - **Append today's jobs**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.5) - Cloud storage integration node.
    - *Configuration Choices:* Operation set to `append` with automated column mapping (`autoMapInputData`). Document ID and Sheet Name must be bound via resource locator or list selection.
    - *Key Expressions/Variables:* None (auto-maps incoming item keys to sheet columns).
    - *Input/Output Connections:* Input from `Shape today's rows`; terminal node in the workflow.
    - *Edge Cases/Failures:* Google OAuth2 authentication expiration, quota limits on the Google Sheets API, incorrect document/sheet IDs, or column header mismatches causing misplaced data. Requires valid Google Sheets credentials.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every morning | n8n-nodes-base.scheduleTrigger | Daily schedule trigger | None | Set the jobs to watch | Daily Google Jobs digest in Google Sheets<br><br>Every morning this workflow searches Google Jobs for the roles you list, in the city or country you choose, and appends one dated row per job to Google Sheets: title, company, location, salary, job type, posting age, the site it is listed on and the apply link.<br><br>Useful for job seekers, recruiters and sales teams who sell to companies that are hiring.<br><br>### How it works<br>1. **Schedule** starts the run every day at 07:00.<br>2. **Set the jobs to watch** holds your searches, country and city.<br>3. **Apify** runs the [Google Jobs Scraper](https://apify.com/hirepulse/google-jobs-scraper) Actor and waits for the results.<br>4. **Filter** keeps real job records.<br>5. **Code** shapes one row per job with today's date.<br>6. **Google Sheets** appends the rows.<br><br>### Setup<br>- Connect your Apify account in the Apify node.<br>- Connect Google Sheets and pick a spreadsheet and sheet.<br><br>### Cost<br>$0.005 per search that finds jobs plus $0.001 per job. Two searches with 50 jobs each cost about $0.11 a day. Searches with no jobs are free. The Apify node caps each run at $1.<br><br>On the free Apify plan you get a preview: the first 10 jobs of the first search. A daily digest of several roles needs a **paid Apify plan**. |
| Set the jobs to watch | n8n-nodes-base.set | Parameter definition | Every morning | Search Google Jobs | Daily Google Jobs digest in Google Sheets... *(truncated duplicate)* |
| Set the jobs to watch | n8n-nodes-base.set | Parameter definition | Every morning | Search Google Jobs | Your searches<br>Edit the roles, the two-letter country code and the city (`City,Region`, or empty for the whole country). |
| Search Google Jobs | @apify/n8n-nodes-apify.apify | External scraping execution | Set the jobs to watch | Keep real job records | Daily Google Jobs digest in Google Sheets... *(truncated duplicate)* |
| Keep real job records | n8n-nodes-base.filter | Validation and filtering | Search Google Jobs | Shape today's rows | Daily Google Jobs digest in Google Sheets... *(truncated duplicate)* |
| Shape today's rows | n8n-nodes-base.code | Data shaping and timestamping | Keep real job records | Append today's jobs | Daily Google Jobs digest in Google Sheets... *(truncated duplicate)* |
| Append today's jobs | n8n-nodes-base.googleSheets | Spreadsheet row persistence | Shape today's rows | None | Daily Google Jobs digest in Google Sheets... *(truncated duplicate)* |
| Append today's jobs | n8n-nodes-base.googleSheets | Spreadsheet row persistence | Shape today's rows | None | Ideas<br>- Send the rows to Slack or email instead of Sheets.<br>- Add `employment_types: ['fulltime']` to the Apify input.<br>- Combine with the [ATS Job Scraper](https://apify.com/hirepulse/ats-job-pulse) to track specific companies. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`, v1.2).
   - Set the interval rule to trigger every day at hour `07:00`.
   - Name the node `Every morning`.

2. **Create the Configuration Node:**
   - Add an **Edit Fields (Set)** node (`n8n-nodes-base.set`, v3.4).
   - Connect the output of `Every morning` to this node.
   - Configure three assignments:
     - `queries` (Array): `["data engineer", "product manager"]`
     - `country` (String): `"US"`
     - `location` (String): `"Austin,Texas"`
   - Name the node `Set the jobs to watch`.

3. **Create the Scraping Integration Node:**
   - Add an **Apify** node (`@apify/n8n-nodes-apify.apify`, v1).
   - Connect the output of `Set the jobs to watch` to this node.
   - Configure parameters:
     - **Resource:** `Actors`
     - **Operation:** `Run actor and get dataset`
     - **Actor ID:** `hirepulse~google-jobs-scraper`
     - **Memory:** `256`
     - **Max Total Charge (USD):** `1`
     - **Custom Body:** Set expression to `={{ JSON.stringify({ queries: $json.queries, country: $json.country, location: $json.location, max_jobs_per_query: 50 }) }}`
   - Configure required **Apify credentials**.
   - Name the node `Search Google Jobs`.

4. **Create the Filtering Node:**
   - Add a **Filter** node (`n8n-nodes-base.filter`, v2.2).
   - Connect the output of `Search Google Jobs` to this node.
   - Configure conditions:
     - **Version:** `2`
     - **Combinator:** `and`
     - **Condition 1:** Left Value = `={{ $json.title }}`, Operator = `Exists` (single value).
   - Name the node `Keep real job records`.

5. **Create the Code Transformation Node:**
   - Add a **Code** node (`n8n-nodes-base.code`, v2).
   - Connect the output of `Keep real job records` to this node.
   - Insert the following JavaScript code:
     ```javascript
     const day = $now.toFormat('yyyy-MM-dd');

     return $input.all().map((item) => {
       const j = item.json;
       return {
         json: {
           Date: day,
           Search: j.query,
           Title: j.title,
           Company: j.company,
           Location: j.location ?? '',
           Salary: j.salary ?? '',
           'Salary min': j.salary_min ?? '',
           'Salary max': j.salary_max ?? '',
           Type: j.contract_type ?? '',
           Posted: j.posted_ago ?? '',
           'Listed on': j.source ?? '',
           Apply: j.apply_url ?? '',
         },
       };
     });
     ```
   - Name the node `Shape today's rows`.

6. **Create the Persistence Node:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`, v4.5).
   - Connect the output of `Shape today's rows` to this node.
   - Configure parameters:
     - **Operation:** `Append`
     - **Document ID:** Select target spreadsheet using the resource locator.
     - **Sheet Name:** Select target worksheet using the resource locator.
     - **Columns:** Set mapping mode to `Auto Map Input Data`.
   - Configure required **Google Sheets OAuth2 credentials**.
   - Name the node `Append today's jobs`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Jobs Scraper Actor | [Apify Google Jobs Scraper](https://apify.com/hirepulse/google-jobs-scraper) |
| ATS Job Scraper Integration Idea | [Apify ATS Job Scraper](https://apify.com/hirepulse/ats-job-pulse) |