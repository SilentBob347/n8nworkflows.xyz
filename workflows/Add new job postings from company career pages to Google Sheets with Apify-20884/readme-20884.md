Add new job postings from company career pages to Google Sheets with Apify

https://n8nworkflows.xyz/workflows/add-new-job-postings-from-company-career-pages-to-google-sheets-with-apify-20884


# Add new job postings from company career pages to Google Sheets with Apify

### 1. Workflow Overview

This workflow automates the collection and tracking of new job postings from specified company career pages, saving them directly into a Google Sheets spreadsheet. It is designed for recruiters, job seekers, sales teams monitoring hiring activity, and market analysts.

The workflow logic is categorized into two main functional blocks:
- **1.1 Trigger and Data Extraction:** Initiates the process on a daily schedule and executes an Apify actor to fetch new job postings based on predefined target companies and filters.
- **1.2 Data Transformation and Storage:** Formats the raw job data into structured spreadsheet rows and appends or updates the records in a Google Sheets document using a unique `jobId` key.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Trigger and Data Extraction
- **Overview:** This block handles the scheduled initiation of the workflow and interacts with the Apify platform to scrape targeted career pages for newly published job listings.
- **Nodes Involved:** 
  - `Every day at 08:00`
  - `Get new job postings`
- **Node Details:**
  - **Every day at 08:00**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger node). Executes the workflow periodically.
    - *Configuration:* Interval set to 1 day, triggering daily at 08:00.
    - *Input/Output:* No inputs; outputs to the Apify node.
    - *Edge Cases/Failures:* Missed executions if the n8n instance is offline at the scheduled hour.
  - **Get new job postings**
    - *Type and Technical Role:* `@apify/n8n-nodes-apify.apify` (Action node). Runs an Apify actor to retrieve datasets.
    - *Configuration:* Uses Actor ID `ooXv5feBjdefzRfdC` (careers-jobs Actor). Custom JSON payload specifies target companies (`airbnb`, `figma`, `notion`, Lever URL, Ashby prefix), keyword filters (`engineer`, `data`, `product`), `onlyNew: true`, and a maximum result cap of 500. Sets a maximum total charge constraint of $1 USD.
    - *Input/Output:* Input connected from the Schedule Trigger; output connected to the Code node.
    - *Credentials:* Requires an Apify API token (`Apify account`).
    - *Edge Cases/Failures:* Authentication failures, API rate limits, exceeding the maximum charge limit, or malformed JSON input payloads.

#### Block 1.2: Data Transformation and Storage
- **Overview:** This block processes the raw items returned by the scraper, normalizes fields (such as salary ranges and locations), creates a unique identifier, and writes the results to Google Sheets.
- **Nodes Involved:**
  - `Shape sheet rows`
  - `Add rows to the Jobs sheet`
- **Node Details:**
  - **Shape sheet rows**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data transformation node). Executes custom JavaScript to filter valid jobs and construct normalized objects.
    - *Configuration:* Filters items requiring `title` and `url`. Maps fields to a standardized schema (`company`, `title`, `department`, `location`, `country`, `workplace`, `employmentType`, `salary`, `postedAt`, `url`, `jobId`, `foundAt`). Generates `jobId` by combining source and unique job identifier/URL.
    - *Input/Output:* Input connected from the Apify node; output connected to the Google Sheets node.
    - *Edge Cases/Failures:* JavaScript runtime errors if expected properties are null/undefined (mitigated by optional chaining), or empty arrays if no new jobs match the criteria.
  - **Add rows to the Jobs sheet**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Action node). Appends or updates rows within a Google Sheets document.
    - *Configuration:* Operation set to `appendOrUpdate`. Matches existing records using the `jobId` column. Targets sheet name `Jobs` within the specified document ID. Uses automatic data mapping.
    - *Input/Output:* Input connected from the Code node; terminal node with no subsequent outputs.
    - *Credentials:* Requires Google Sheets OAuth2 authentication (`Google Sheets account`).
    - *Edge Cases/Failures:* OAuth token expiration, missing sheet or tab names, permission errors on the target Google Sheet, or API quota limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every day at 08:00 | `scheduleTrigger` | Triggers the workflow daily at 08:00. | None | Get new job postings | Add new job postings from company career pages to Google Sheets... |
| Get new job postings | `@apify/n8n-nodes-apify` | Fetches new job postings using the Apify careers-jobs Actor. | Every day at 08:00 | Shape sheet rows | ## Get new postings<br>Your company list and filters live in the Input JSON. |
| Get new job postings | `@apify/n8n-nodes-apify` | Fetches new job postings using the Apify careers-jobs Actor. | Every day at 08:00 | Shape sheet rows | Add new job postings from company career pages to Google Sheets... |
| Shape sheet rows | `code` | Transforms and normalizes scraped job data into sheet rows. | Get new job postings | Add rows to the Jobs sheet | Add new job postings from company career pages to Google Sheets... |
| Add rows to the Jobs sheet | `googleSheets` | Appends or updates job records in Google Sheets based on `jobId`. | Shape sheet rows | None | ## Save to the sheet<br>One row per job, matched on `jobId`. |
| Add rows to the Jobs sheet | `googleSheets` | Appends or updates job records in Google Sheets based on `jobId`. | Shape sheet rows | None | Add new job postings from company career pages to Google Sheets... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Configure the interval trigger to run every `1` day at hour `8`.
   - Name the node `Every day at 08:00`.

2. **Create the Apify Integration Node:**
   - Add an **Apify** node (`@apify/n8n-nodes-apify.apify`). *Note: Requires the `@apify/n8n-nodes-apify` community node to be installed.*
   - Set **Resource** to `Actors` and **Operation** to `Run actor and get dataset`.
   - Set **Actor ID** (mode: ID) to `ooXv5feBjdefzRfdC`.
   - Populate the **Custom Body** JSON parameter with the company list, keywords, and filter parameters (e.g., `companies`, `titleKeywords`, `maxResults: 500`, `onlyNew: true`).
   - Set **Max Total Charge (USD)** to `1`.
   - Configure and link your **Apify API** credentials.
   - Name the node `Get new job postings`.
   - Connect `Every day at 08:00` to `Get new job postings`.

3. **Create the Data Transformation Node:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Set the execution mode to JavaScript and input the transformation logic to parse salaries, filter valid entries with a title and URL, and construct a unique `jobId`.
   - Name the node `Shape sheet rows`.
   - Connect `Get new job postings` to `Shape sheet rows`.

4. **Create the Google Sheets Output Node:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`).
   - Set **Operation** to `appendOrUpdate`.
   - Set **Document ID** (mode: ID) to your target spreadsheet identifier.
   - Set **Sheet Name** (mode: Name) to `Jobs`.
   - Configure **Mapping Mode** to `autoMapInputData` and specify `jobId` under **Matching Columns**.
   - Configure and link your **Google Sheets OAuth2** credentials.
   - Name the node `Add rows to the Jobs sheet`.
   - Connect `Shape sheet rows` to `Add rows to the Jobs sheet`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Made by locaihost, a small studio that builds its tools with AI agents. | Project credits and creator branding |