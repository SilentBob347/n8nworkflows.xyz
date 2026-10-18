Log daily Indeed job searches to Google Sheets with Apify

https://n8nworkflows.xyz/workflows/log-daily-indeed-job-searches-to-google-sheets-with-apify-20851


# Log daily Indeed job searches to Google Sheets with Apify

### 1. Workflow Overview

This workflow automates the daily discovery and logging of job listings from Indeed into a Google Sheets document. Its primary purpose is to help job seekers, recruiters, and market researchers track new job postings within a specific niche without manual searching. 

The execution is structured into a single linear pipeline:
- **1.1 Trigger & Scraper Execution:** Initiates daily at 08:00 and triggers an external web scraping actor on Apify configured to search for specific job criteria.
- **1.2 Data Transformation:** Maps and normalizes the raw job scraper output into clear, structured fields suitable for spreadsheet insertion.
- **1.3 Data Persistence:** Appends the formatted job rows directly to a designated Google Sheets worksheet.

---

### 2. Block-by-Block Analysis

#### Block 1: Trigger & Scraper Execution
- **Overview:** This block handles the scheduled start of the automation and interacts with the Apify platform to retrieve fresh, unique job listings based on predefined search parameters.
- **Nodes Involved:** `Every morning`, `Search Indeed`

##### Node Details:
- **Every morning**
  - **Type and Technical Role:** `n8n-nodes-base.scheduleTrigger` (v1.2). Acts as the workflow's entry point, executing automatically on a fixed time interval.
  - **Configuration Choices:** Configured to trigger daily at hour 08:00.
  - **Key Expressions or Variables:** None.
  - **Input and Output Connections:** No input nodes; outputs to `Search Indeed`.
  - **Version-specific Requirements:** None.
  - **Edge Cases or Potential Failure Types:** Failure of n8n instance uptime during the scheduled execution window.
  - **Sub-workflow Reference:** None.

- **Search Indeed**
  - **Type and Technical Role:** `@apify/n8n-nodes-apify.apify` (v1). Executes an external Apify actor and waits for the dataset results.
  - **Configuration Choices:** Uses resource type `Actors`, operation `Run actor and get dataset`, targeting actor ID `robertosan16~indeed-jobs-scraper` with 512MB allocated memory and no timeout.
  - **Custom Body Payload:** 
    ```json
    {
      "searchQueries": ["data analyst"],
      "country": "US",
      "location": "New York, NY",
      "postedWithinDays": 1,
      "onlyNew": true,
      "includeDescription": false,
      "maxItems": 200
    }
    ```
  - **Key Expressions or Variables:** None (static JSON payload).
  - **Input and Output Connections:** Input from `Every morning`; outputs to `Pick columns`.
  - **Version-specific Requirements:** Requires the verified community node `@apify/n8n-nodes-apify` to be installed if missing.
  - **Edge Cases or Potential Failure Types:** Apify API authentication errors, insufficient account balance/credits, timeout limits reached during heavy scraping jobs, or invalid search filter parameters.
  - **Sub-workflow Reference:** None.

---

#### Block 2: Data Transformation
- **Overview:** This block extracts and reshapes the raw JSON attributes returned by the Apify scraper into a standardized schema for spreadsheet entry.
- **Nodes Involved:** `Pick columns`

##### Node Details:
- **Pick columns**
  - **Type and Technical Role:** `n8n-nodes-base.set` (v3.4). Transforms incoming item data by mapping specific properties to new named fields.
  - **Configuration Choices:** Operates using field assignments (`assignments` mode) to explicitly name columns.
  - **Key Expressions or Variables:**
    - `Title`: `={{ $json.title }}`
    - `Company`: `={{ $json.companyName }}`
    - `Location`: `={{ $json.location }}`
    - `Salary`: `={{ $json.salary ? $json.salary.text : '' }}`
    - `Job type`: `={{ $json.employmentType || '' }}`
    - `Remote`: `={{ $json.remote ? 'yes' : '' }}`
    - `Posted`: `={{ $json.postedAt ? $json.postedAt.slice(0, 10) : '' }}`
    - `Job link`: `={{ $json.jobUrl }}`
    - `Apply link`: `={{ $json.applyUrl }}`
  - **Input and Output Connections:** Input from `Search Indeed`; outputs to `Add to sheet`.
  - **Version-specific Requirements:** None.
  - **Edge Cases or Potential Failure Types:** Missing data fields in the scraped payload causing ternary expression evaluation errors if improperly chained.
  - **Sub-workflow Reference:** None.

---

#### Block 3: Data Persistence
- **Overview:** This block writes the processed job records into the target cloud spreadsheet.
- **Nodes Involved:** `Add to sheet`

##### Node Details:
- **Add to sheet**
  - **Type and Technical Role:** `n8n-nodes-base.googleSheets` (v4.6). Connects to Google Sheets to append rows of data.
  - **Configuration Choices:** Operation set to `append`. Document ID and Sheet Name use dynamic list selectors requiring manual mapping. Auto-maps incoming input data to matching sheet columns (`autoMapInputData`).
  - **Key Expressions or Variables:** Relies on incoming node properties mapped via automatic schema matching.
  - **Input and Output Connections:** Input from `Pick columns`; no downstream output nodes.
  - **Version-specific Requirements:** Requires Google Sheets OAuth2 authentication.
  - **Edge Cases or Potential Failure Types:** OAuth token expiration, Google Sheets API rate limits, or missing/renamed column headers in the target worksheet causing insertion mismatches.
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Documentation and setup guide | None | None | ## Daily Indeed job alerts in Google Sheets<br><br>Every morning this workflow searches Indeed for jobs that match your keywords and adds the new ones to a Google Sheet: one row per job, with salary, job type, remote and the links. Good for job seekers, recruiters and anyone tracking hiring in a niche.<br><br>### How it works<br>1. **Every morning** starts the workflow at 8:00.<br>2. **Search Indeed** runs the [Indeed Jobs Scraper](https://apify.com/robertosan16/indeed-jobs-scraper) on Apify. It searches any of 62 countries, and with `onlyNew` it returns only jobs it has not returned before for the same search, so the sheet gets no duplicates.<br>3. **Pick columns** keeps the fields for the sheet.<br>4. **Add to sheet** appends one row per job.<br><br>### Set up<br>1. Create a free [Apify account](https://console.apify.com/sign-up) and add an Apify credential to the **Search Indeed** node: the API token from Apify Console → Settings → API & Integrations, or OAuth on n8n Cloud. If the Apify node is missing, install the verified community node `@apify/n8n-nodes-apify`.<br>2. In **Search Indeed**, change the search: `searchQueries`, `country` (a two-letter code such as `US`, `GB` or `DE`) and `location`.<br>3. Create a Google Sheet whose first row holds the column names: Title, Company, Location, Salary, Job type, Remote, Posted, Job link, Apply link. Connect Google Sheets and select the sheet in **Add to sheet**.<br>4. Activate the workflow.<br><br>### Cost<br>The scraper charges $1 per 1,000 jobs. The Apify free plan includes $5 of usage every month. |
| Search note | n8n-nodes-base.stickyNote | Search parameter guidelines | None | None | ### Your search<br>Edit the JSON: keywords, country and city. `postedWithinDays: 1` and `onlyNew: true` keep each run to the jobs that are new since yesterday. More filters: job type, remote, experience level, distance. |
| Every morning | n8n-nodes-base.scheduleTrigger | Starts the workflow daily at 8:00 AM | None | Search Indeed | ### Your search<br>Edit the JSON: keywords, country and city. `postedWithinDays: 1` and `onlyNew: true` keep each run to the jobs that are new since yesterday. More filters: job type, remote, experience level, distance. |
| Search Indeed | @apify/n8n-nodes-apify.apify | Runs the Apify Indeed Jobs Scraper actor | Every morning | Pick columns | ### Your search<br>Edit the JSON: keywords, country and city. `postedWithinDays: 1` and `onlyNew: true` keep each run to the jobs that are new since yesterday. More filters: job type, remote, experience level, distance. |
| Pick columns | n8n-nodes-base.set | Maps and formats raw scraper fields | Search Indeed | Add to sheet | |
| Add to sheet | n8n-nodes-base.googleSheets | Appends formatted job data to a Google Sheet | Pick columns | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create a Schedule Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Name it `Every morning`.
   - Set the interval rule trigger time to run at hour `8`.

2. **Create and Configure the Apify Node:**
   - Add an Apify node (`@apify/n8n-nodes-apify.apify`). *Note: Install the community node `@apify/n8n-nodes-apify` if it is unavailable.*
   - Name it `Search Indeed`.
   - Set **Resource** to `Actors` and **Operation** to `Run actor and get dataset`.
   - Set **Actor ID** to `robertosan16~indeed-jobs-scraper`.
   - Configure the **Custom Body** JSON parameter:
     ```json
     {
       "searchQueries": ["data analyst"],
       "country": "US",
       "location": "New York, NY",
       "postedWithinDays": 1,
       "onlyNew": true,
       "includeDescription": false,
       "maxItems": 200
     }
     ```
   - Connect your Apify API Token credential (generated from Apify Console → Settings → API & Integrations).
   - Connect the output of `Every morning` to `Search Indeed`.

3. **Create and Configure the Set Node (Data Mapping):**
   - Add a **Set** node (`n8n-nodes-base.set`).
   - Name it `Pick columns`.
   - Add the following string-type field assignments:
     - `Title`: `={{ $json.title }}`
     - `Company`: `={{ $json.companyName }}`
     - `Location`: `={{ $json.location }}`
     - `Salary`: `={{ $json.salary ? $json.salary.text : '' }}`
     - `Job type`: `={{ $json.employmentType || '' }}`
     - `Remote`: `={{ $json.remote ? 'yes' : '' }}`
     - `Posted`: `={{ $json.postedAt ? $json.postedAt.slice(0, 10) : '' }}`
     - `Job link`: `={{ $json.jobUrl }}`
     - `Apply link`: `={{ $json.applyUrl }}`
   - Connect the output of `Search Indeed` to `Pick columns`.

4. **Create and Configure the Google Sheets Node:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`).
   - Name it `Add to sheet`.
   - Set **Operation** to `Append`.
   - Connect your Google Sheets OAuth2 credential.
   - Select your target **Document ID** and **Sheet Name** from your Google Drive/Sheets account. Ensure your sheet contains the header columns: `Title`, `Company`, `Location`, `Salary`, `Job type`, `Remote`, `Posted`, `Job link`, `Apply link`.
   - Set **Mapping Mode** to `Auto Map Input Data`.
   - Connect the output of `Pick columns` to `Add to sheet`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apify Signup | Create a free account at [Apify Sign-up](https://console.apify.com/sign-up) |
| Indeed Jobs Scraper Actor | Actor reference and documentation on [Apify](https://apify.com/robertosan16/indeed-jobs-scraper) |