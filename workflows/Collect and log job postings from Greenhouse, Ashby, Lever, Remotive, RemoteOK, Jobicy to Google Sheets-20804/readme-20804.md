Collect and log job postings from Greenhouse, Ashby, Lever, Remotive, RemoteOK, Jobicy to Google Sheets

https://n8nworkflows.xyz/workflows/collect-and-log-job-postings-from-greenhouse--ashby--lever--remotive--remoteok--jobicy-to-google-sheets-20804


# Collect and log job postings from Greenhouse, Ashby, Lever, Remotive, RemoteOK, Jobicy to Google Sheets

### 1. Workflow Overview

This workflow automates the collection, filtering, deduplication, and logging of job postings from multiple company-specific Applicant Tracking Systems (ATS) and remote job aggregator platforms. Operating on a scheduled interval (every three days), it pulls listings from Greenhouse, Ashby, Lever, Remotive, RemoteOK, and Jobicy, transforms them into a unified schema, filters them based on strict criteria (role keywords, location rules, and description thresholds), cross-references them against existing records in a Google Sheets document, and appends newly discovered matches. Finally, it records execution metrics and source health diagnostics in a dedicated logging tab.

The automation logic is divided into the following sequential and parallel functional blocks:

- **1.1 Schedule & Initialization:** Triggers the workflow execution on a regular cadence, establishes global search configuration parameters, and indexes existing records from the target Google Sheet to prevent duplicate entries.
- **1.2 Source Fan-Out & Data Ingestion (Company ATS):** Iterates dynamically through user-defined lists of target companies across Greenhouse, Ashby, and Lever, executing parallel HTTP requests to fetch raw job boards.
- **1.3 Source Fan-Out & Data Ingestion (Aggregator Feeds):** Fetches raw job listings concurrently from open remote job aggregators: Remotive, Jobicy, and RemoteOK.
- **1.4 Normalization & Merging:** Consolidates all disparate source streams into a single unified data pipeline and normalizes text properties, HTML markup, and date formats.
- **1.5 Filtering, Ranking, & Decision Logic:** Applies multi-stage rejection filters (missing fields, description length, age constraints, role regular expressions, location rules, and deduplication), ranks and caps the results, and evaluates whether any eligible new positions remain.
- **1.6 Output & Execution Logging:** Splits filtered items for individual database insertion, appends new records to the target Google Sheets document, and compiles a comprehensive execution summary written to the run log.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Schedule & Initialization

##### Overview
This block initiates the execution cycle on a scheduled basis, declares all filtering configurations, board tokens, and parameters, reads current rows from the destination spreadsheet, and calculates existing record indexes.

##### Nodes Involved
- `When Every 3 Days`
- `Set Job Search Criteria`
- `Read from Google Sheets`
- `Index Saved Job Data`

##### Node Details

###### `When Every 3 Days`
- **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` (Trigger Node). Initiates workflow execution based on a time interval.
- **Configuration Choices:** Configured to execute every 3 days at 09:00 AM.
- **Key Expressions / Variables:** None.
- **Input / Output Connections:** Input: None (Trigger). Output: `Set Job Search Criteria`.
- **Version-Specific Requirements:** Version 1.2.
- **Edge Cases / Potential Failure Types:** Missed executions if the n8n instance is offline at the exact trigger hour.

###### `Set Job Search Criteria`
- **Type & Technical Role:** `n8n-nodes-base.set` (Data Transformation / Configuration Node). Defines all global runtime settings, target company IDs, and filtering rules.
- **Configuration Choices:** Assigns JavaScript arrays and primitive values to runtime properties including `filters`, `greenhouse_boards`, `ashby_boards`, `lever_boards`, location rules, caps, and timestamps.
- **Key Expressions / Variables:** 
  - `run_at`: `={{ $now.toFormat('yyyy-MM-dd HH:mm ZZZZ') }}`
  - `run_date`: `={{ $now.toFormat('yyyy-MM-dd') }}`
- **Input / Output Connections:** Input: `When Every 3 Days`. Output: `Read from Google Sheets`.
- **Version-Specific Requirements:** Version 3.4.
- **Edge Cases / Potential Failure Types:** Syntax errors in array definitions or invalid date-formatting tokens.

###### `Read from Google Sheets`
- **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Integration Node). Reads existing rows from the specified target worksheet tab.
- **Configuration Choices:** Resource: `sheet`, Operation: `read`, Sheet Name: `Applications`. `Always Output Data` is enabled to prevent workflow interruption if the sheet is empty.
- **Key Expressions / Variables:** Relies on dynamic spreadsheet selection.
- **Input / Output Connections:** Input: `Set Job Search Criteria`. Output: `Index Saved Job Data`.
- **Version-Specific Requirements:** Version 4.5.
- **Edge Cases / Potential Failure Types:** OAuth2 authentication expiry, API rate limits, or renamed/missing worksheet tabs.

###### `Index Saved Job Data`
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node). Aggregates raw Google Sheet rows into a lookup structure containing saved job links, total row count, and the maximum serial number.
- **Configuration Choices:** Runs once for all items (`runOnceForAllItems`) to prevent downstream fan-out across every existing row.
- **Key Expressions / Variables:** Iterates over `$input.all()`, safely matching case-insensitive headers like `job posting link` and `s.no`.
- **Input / Output Connections:** Input: `Read from Google Sheets`. Output: Fan-out to all data ingestion nodes (`Distribute Greenhouse Boards`, `Distribute Ashby Boards`, `Distribute Lever Boards`, `Fetch Remotive Jobs`, `Fetch RemoteOK Jobs`, `Fetch Jobicy Jobs`).
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Unhandled exceptions if column headers are deleted or altered in Google Sheets.

---

#### 1.2 Source Fan-Out & Data Ingestion (Company ATS)

##### Overview
This block fans out runtime configurations into individual per-company requests for Greenhouse, Ashby, and Lever, executes HTTP API calls with retry logic, and normalizes raw responses into source-specific schema items.

##### Nodes Involved
- `Distribute Greenhouse Boards`
- `Fetch Greenhouse Jobs`
- `Transform Greenhouse Data`
- `Distribute Ashby Boards`
- `Fetch Ashby Jobs`
- `Transform Ashby Data`
- `Distribute Lever Boards`
- `Fetch Lever Jobs`
- `Transform Lever Data`

##### Node Details

###### `Distribute Greenhouse Boards`
- **Type & Technical Role:** `n8n-nodes-base.code` (Utility Node). Transforms the array of Greenhouse companies into individual items.
- **Configuration Choices:** Mode: `runOnceForAllItems`. `Always Output Data` is enabled.
- **Key Expressions / Variables:** Reads `greenhouse_boards` from `Set Job Search Criteria`.
- **Input / Output Connections:** Input: `Index Saved Job Data`. Output: `Fetch Greenhouse Jobs`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Empty board arrays emit a safe fallback placeholder item.

###### `Fetch Greenhouse Jobs`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Network Request Node). Queries the Greenhouse Board API for job listings.
- **Configuration Choices:** Method: `GET`, URL: `https://boards-api.greenhouse.io/v1/boards/{{ $json.token }}/jobs?content=true`, Timeout: 30,000ms. Error handling is set to `continueRegularOutput` with 3 retries.
- **Key Expressions / Variables:** `={{ $json.token }}`
- **Input / Output Connections:** Input: `Distribute Greenhouse Boards`. Output: `Transform Greenhouse Data`.
- **Version-Specific Requirements:** Version 4.2.
- **Edge Cases / Potential Failure Types:** HTTP 404/500 errors if a company token becomes invalid; mitigated by error routing.

###### `Transform Greenhouse Data`
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node). Maps raw Greenhouse responses into the unified job format and reports source health.
- **Configuration Choices:** Runs once for all items (`runOnceForAllItems`). Uses paired-item mapping (`itemMatching`) to tie responses safely back to target companies.
- **Key Expressions / Variables:** Evaluates `item.json.error` and maps properties (`title`, `absolute_url`, `content`, `first_published`, `location`).
- **Input / Output Connections:** Input: `Fetch Greenhouse Jobs`. Output: `Merge All Job Sources` (Input index 0).
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Malformed JSON payloads or missing array structures.

###### `Distribute Ashby Boards`
- **Type & Technical Role:** `n8n-nodes-base.code` (Utility Node). Splits configured Ashby boards into separate items.
- **Configuration Choices:** Mode: `runOnceForAllItems`. `Always Output Data` is enabled.
- **Key Expressions / Variables:** Reads `ashby_boards` from `Set Job Search Criteria`.
- **Input / Output Connections:** Input: `Index Saved Job Data`. Output: `Fetch Ashby Jobs`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Empty configurations output a safe fallback item.

###### `Fetch Ashby Jobs`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Network Request Node). Queries the Ashby posting API.
- **Configuration Choices:** Method: `GET`, URL: `https://api.ashbyhq.com/posting-api/job-board/{{ $json.board }}?includeCompensation=false`, Timeout: 30,000ms. Error handling: `continueRegularOutput` with 3 retries.
- **Key Expressions / Variables:** `={{ $json.board }}`
- **Input / Output Connections:** Input: `Distribute Ashby Boards`. Output: `Transform Ashby Data`.
- **Version-Specific Requirements:** Version 4.2.
- **Edge Cases / Potential Failure Types:** Rate limiting or invalid board slugs.

###### `Transform Ashby Data`
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node). Maps Ashby job postings into the unified job shape.
- **Configuration Choices:** Runs once for all items using paired-item tracking (`itemMatching`). Filters out unlisted postings (`isListed !== false`).
- **Key Expressions / Variables:** Combines primary and secondary locations (`secondaryLocations`), checking remote flags.
- **Input / Output Connections:** Input: `Fetch Ashby Jobs`. Output: `Merge All Job Sources` (Input index 1).
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Missing location or description fields.

###### `Distribute Lever Boards`
- **Type & Technical Role:** `n8n-nodes-base.code` (Utility Node). Splits configured Lever slugs into separate items.
- **Configuration Choices:** Mode: `runOnceForAllItems`. `Always Output Data` enabled.
- **Key Expressions / Variables:** Reads `lever_boards` from `Set Job Search Criteria`.
- **Input / Output Connections:** Input: `Index Saved Job Data`. Output: `Fetch Lever Jobs`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Empty board list outputs a safe fallback item.

###### `Fetch Lever Jobs`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Network Request Node). Queries the Lever posting API.
- **Configuration Choices:** Method: `GET`, URL: `https://api.lever.co/v0/postings/{{ $json.slug }}?mode=json`, Timeout: 30,000ms. Error handling: `continueRegularOutput` with 3 retries.
- **Key Expressions / Variables:** `={{ $json.slug }}`
- **Input / Output Connections:** Input: `Distribute Lever Boards`. Output: `Transform Lever Data`.
- **Version-Specific Requirements:** Version 4.2.
- **Edge Cases / Potential Failure Types:** Invalid slugs returning empty arrays or error codes.

###### `Transform Lever Data`
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node). Transforms Lever posting objects into the unified schema.
- **Configuration Choices:** Runs once for all items using paired-item tracking. Appends requirement lists to descriptions to ensure comprehensive keyword matching.
- **Key Expressions / Variables:** Maps `text`, `hostedUrl`, `descriptionPlain`, `createdAt`, and categorization arrays.
- **Input / Output Connections:** Input: `Fetch Lever Jobs`. Output: `Merge All Job Sources` (Input index 2).
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Schema changes in Lever posting categories.

---

#### 1.3 Source Fan-Out & Data Ingestion (Aggregator Feeds)

##### Overview
This block executes parallel API calls to remote job aggregator platforms (Remotive, RemoteOK, and Jobicy) and normalizes their respective JSON responses.

##### Nodes Involved
- `Fetch Remotive Jobs`
- `Transform Remotive Data`
- `Fetch RemoteOK Jobs`
- `Transform RemoteOK Data`
- `Fetch Jobicy Jobs`
- `Transform Jobicy Data`

##### Node Details

###### `Fetch Remotive Jobs`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Network Request Node). Requests remote job postings from the Remotive API.
- **Configuration Choices:** Method: `GET`, URL: `https://remotive.com/api/remote-jobs`, Query parameter `limit=100`, Timeout: 30,000ms. Error handling: `continueRegularOutput` with 3 retries.
- **Key Expressions / Variables:** None.
- **Input / Output Connections:** Input: `Index Saved Job Data`. Output: `Transform Remotive Data`.
- **Version-Specific Requirements:** Version 4.2.
- **Edge Cases / Potential Failure Types:** API downtime or rate throttling.

###### `Transform Remotive Data`
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node). Parses Remotive payloads into unified job items and records source status.
- **Configuration Choices:** Mode: `runOnceForAllItems`.
- **Key Expressions / Variables:** Maps `company_name`, `title`, `url`, `description`, `publication_date`, and `candidate_required_location`.
- **Input / Output Connections:** Input: `Fetch Remotive Jobs`. Output: `Merge All Job Sources` (Input index 3).
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Empty job feeds or structural API changes.

###### `Fetch RemoteOK Jobs`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Network Request Node). Requests remote job listings from RemoteOK.
- **Configuration Choices:** Method: `GET`, URL: `https://remoteok.com/api`, Custom User-Agent header `n8n-job-search-template/1.0`, Timeout: 30,000ms. Error handling: `continueRegularOutput` with 3 retries.
- **Key Expressions / Variables:** None.
- **Input / Output Connections:** Input: `Index Saved Job Data`. Output: `Transform RemoteOK Data`.
- **Version-Specific Requirements:** Version 4.2.
- **Edge Cases / Potential Failure Types:** Anti-bot blocks or missing User-Agent headers.

###### `Transform RemoteOK Data`
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node). Filters out legal disclaimers from the RemoteOK response array and standardizes fields.
- **Configuration Choices:** Mode: `runOnceForAllItems`. Filters items containing `legal` properties.
- **Key Expressions / Variables:** Maps `company`, `position`, `url`, `description`, `date`, and `location`.
- **Input / Output Connections:** Input: `Fetch RemoteOK Jobs`. Output: `Merge All Job Sources` (Input index 4).
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Array structure changes.

###### `Fetch Jobicy Jobs`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Network Request Node). Fetches remote job postings from Jobicy.
- **Configuration Choices:** Method: `GET`, URL: `https://jobicy.com/api/v2/remote-jobs`, Query parameters `count=100`, `geo=usa`, Timeout: 30,000ms. Error handling: `continueRegularOutput` with 3 retries.
- **Key Expressions / Variables:** None.
- **Input / Output Connections:** Input: `Index Saved Job Data`. Output: `Transform Jobicy Data`.
- **Version-Specific Requirements:** Version 4.2.
- **Edge Cases / Potential Failure Types:** Geolocation parameter rejection or network timeout.

###### `Transform Jobicy Data`
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node). Maps Jobicy API properties into the unified schema.
- **Configuration Choices:** Mode: `runOnceForAllItems`.
- **Key Expressions / Variables:** Maps `companyName`, `jobTitle`, `url`, `jobDescription`, `pubDate`, and `jobGeo`.
- **Input / Output Connections:** Input: `Fetch Jobicy Jobs`. Output: `Merge All Job Sources` (Input index 5).
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Missing object properties.

---

#### 1.4 Normalization & Merging

##### Overview
This block consolidates all six ingestion streams into a single flow, decodes HTML entities, sanitizes markup tags, normalizes diverse date formats into standard ISO strings (`YYYY-MM-DD`), and prepares structured datasets for filtering.

##### Nodes Involved
- `Merge All Job Sources`
- `Normalize Job Data`

##### Node Details

###### `Merge All Job Sources`
- **Type & Technical Role:** `n8n-nodes-base.merge` (Flow Control Node). Combines multiple input streams into a single output queue.
- **Configuration Choices:** Mode: `append`, Number of Inputs: `6`.
- **Key Expressions / Variables:** None.
- **Input / Output Connections:** Inputs: `Transform Greenhouse Data`, `Transform Ashby Data`, `Transform Lever Data`, `Transform Remotive Data`, `Transform RemoteOK Data`, `Transform Jobicy Data`. Output: `Normalize Job Data`.
- **Version-Specific Requirements:** Version 3.2.
- **Edge Cases / Potential Failure Types:** Unsynchronized input completion causing stalled execution branches.

###### `Normalize Job Data`
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node). Cleans description strings, handles double-encoded HTML entities, and standardizes date formats.
- **Configuration Choices:** Mode: `runOnceForAllItems`.
- **Key Expressions / Variables:** Custom regex cleaning functions and date conversion routines mapping epoch timestamps and ISO strings to `YYYY-MM-DD`.
- **Input / Output Connections:** Input: `Merge All Job Sources`. Output: `Filter and Rank Job Listings`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Unparseable date strings resulting in null dates, which are subsequently filtered out.

---

#### 1.5 Filtering, Ranking, & Decision Logic

##### Overview
This block enforces filtering rules (required fields, description length, maximum age, role keywords, and location rules), performs intra-run and historical deduplication against Google Sheets data, ranks matches, applies caps, and branches execution based on whether new jobs are found.

##### Nodes Involved
- `Filter and Rank Job Listings`
- `Check for New Jobs`

##### Node Details

###### `Filter and Rank Job Listings`
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node). Evaluates multi-stage rejection filters, performs deduplication, ranks opportunities, applies per-company and run caps, and shapes the output payload.
- **Configuration Choices:** Mode: `runOnceForAllItems`. Outputs a single consolidated object containing `rows` and `stats`.
- **Key Expressions / Variables:** Uses word-boundary regular expressions (`wordRx`), URL normalization utilities (`urlKey`), and configuration criteria from `Set Job Search Criteria`.
- **Input / Output Connections:** Input: `Normalize Job Data`. Output: `Check for New Jobs`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Overly restrictive keyword filters resulting in zero matches.

###### `Check for New Jobs`
- **Type & Technical Role:** `n8n-nodes-base.if` (Flow Control Node). Evaluates whether eligible new job listings were found.
- **Configuration Choices:** Condition type: Strict number comparison where `{{ $json.rows.length }}` > 0.
- **Key Expressions / Variables:** `={{ $json.rows.length }}`
- **Input / Output Connections:** Input: `Filter and Rank Job Listings`. Outputs: True branch -> `Split Job Listings`; False branch -> `Generate Run Log Entry`.
- **Version-Specific Requirements:** Version 2.2.
- **Edge Cases / Potential Failure Types:** Expression evaluation errors if upstream JSON structure changes.

---

#### 1.6 Output & Execution Logging

##### Overview
This block processes approved job matches, splits them into individual records, appends them to the Applications tab in Google Sheets, compiles execution metrics, and writes a diagnostic entry to the Run Log tab.

##### Nodes Involved
- `Split Job Listings`
- `Append Jobs to Google Sheets`
- `Generate Run Log Entry`
- `Append to Run Log in Sheets`

##### Node Details

###### `Split Job Listings`
- **Type & Technical Role:** `n8n-nodes-base.splitOut` (Data Transformation Node). Expands the array of row objects into individual items for database insertion.
- **Configuration Choices:** Field to split out: `rows`. Option `include` set to `noOtherFields`.
- **Key Expressions / Variables:** None.
- **Input / Output Connections:** Input: `Check for New Jobs` (True branch). Output: `Append Jobs to Google Sheets`.
- **Version-Specific Requirements:** Version 1.
- **Edge Cases / Potential Failure Types:** Empty arrays causing processing halts if upstream checks fail.

###### `Append Jobs to Google Sheets`
- **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Integration Node). Appends individual job records to the Applications worksheet.
- **Configuration Choices:** Resource: `sheet`, Operation: `append`, Sheet Name: `Applications`. Error handling: `continueRegularOutput`.
- **Key Expressions / Variables:** Automatic mapping of input data properties to sheet columns.
- **Input / Output Connections:** Input: `Split Job Listings`. Output: `Generate Run Log Entry`.
- **Version-Specific Requirements:** Version 4.5.
- **Edge Cases / Potential Failure Types:** API rate limits or schema mismatches; handled by error continuation.

###### `Generate Run Log Entry`
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node). Compiles source health diagnostics, collection counts, and execution metrics into a summary log object.
- **Configuration Choices:** Mode: `runOnceForAllItems`. Evaluates execution status from both active and bypassed upstream nodes.
- **Key Expressions / Variables:** Reads execution state via `.isExecuted`, calculates error summaries, and tallies locations and roles.
- **Input / Output Connections:** Inputs: `Check for New Jobs` (False branch) and `Append Jobs to Google Sheets`. Output: `Append to Run Log in Sheets`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Missing metrics if upstream filter nodes output unexpected structures.

###### `Append to Run Log in Sheets`
- **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Integration Node). Writes the finalized execution log entry to the Run Log tab.
- **Configuration Choices:** Resource: `sheet`, Operation: `append`, Sheet Name: `Run Log`.
- **Key Expressions / Variables:** Automatic column mapping based on object keys.
- **Input / Output Connections:** Input: `Generate Run Log Entry`. Output: None (Terminal Node).
- **Version-Specific Requirements:** Version 4.5.
- **Edge Cases / Potential Failure Types:** Authentication failures or worksheet unavailability.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `When Every 3 Days` | `n8n-nodes-base.scheduleTrigger` | Triggers workflow execution | None | `Set Job Search Criteria` | Schedule and initialize |
| `Set Job Search Criteria` | `n8n-nodes-base.set` | Defines global search criteria & tokens | `When Every 3 Days` | `Read from Google Sheets` | Schedule and initialize |
| `Read from Google Sheets` | `n8n-nodes-base.googleSheets` | Reads existing saved applications | `Set Job Search Criteria` | `Index Saved Job Data` | Schedule and initialize |
| `Index Saved Job Data` | `n8n-nodes-base.code` | Builds deduplication index & serial numbers | `Read from Google Sheets` | *Distribute & Fetch nodes* | Schedule and initialize |
| `Distribute Greenhouse Boards` | `n8n-nodes-base.code` | Fans out Greenhouse target companies | `Index Saved Job Data` | `Fetch Greenhouse Jobs` | Fetch Greenhouse jobs |
| `Fetch Greenhouse Jobs` | `n8n-nodes-base.httpRequest` | Requests raw Greenhouse board data | `Distribute Greenhouse Boards` | `Transform Greenhouse Data` | Fetch Greenhouse jobs |
| `Transform Greenhouse Data` | `n8n-nodes-base.code` | Maps Greenhouse response to unified schema | `Fetch Greenhouse Jobs` | `Merge All Job Sources` | Fetch Greenhouse jobs |
| `Distribute Ashby Boards` | `n8n-nodes-base.code` | Fans out Ashby target boards | `Index Saved Job Data` | `Fetch Ashby Jobs` | Fetch Ashby jobs |
| `Fetch Ashby Jobs` | `n8n-nodes-base.httpRequest` | Requests raw Ashby board data | `Distribute Ashby Boards` | `Transform Ashby Data` | Fetch Ashby jobs |
| `Transform Ashby Data` | `n8n-nodes-base.code` | Maps Ashby response to unified schema | `Fetch Ashby Jobs` | `Merge All Job Sources` | Fetch Ashby jobs |
| `Distribute Lever Boards` | `n8n-nodes-base.code` | Fans out Lever target companies | `Index Saved Job Data` | `Fetch Lever Jobs` | Fetch Lever jobs |
| `Fetch Lever Jobs` | `n8n-nodes-base.httpRequest` | Requests raw Lever board data | `Distribute Lever Boards` | `Transform Lever Data` | Fetch Lever jobs |
| `Transform Lever Data` | `n8n-nodes-base.code` | Maps Lever response to unified schema | `Fetch Lever Jobs` | `Merge All Job Sources` | Fetch Lever jobs |
| `Fetch Remotive Jobs` | `n8n-nodes-base.httpRequest` | Fetches remote jobs from Remotive API | `Index Saved Job Data` | `Transform Remotive Data` | Fetch Remotive jobs |
| `Transform Remotive Data` | `n8n-nodes-base.code` | Maps Remotive feed to unified schema | `Fetch Remotive Jobs` | `Merge All Job Sources` | Fetch Remotive jobs |
| `Fetch RemoteOK Jobs` | `n8n-nodes-base.httpRequest` | Fetches remote jobs from RemoteOK API | `Index Saved Job Data` | `Transform RemoteOK Data` | Fetch RemoteOK jobs |
| `Transform RemoteOK Data` | `n8n-nodes-base.code` | Maps RemoteOK feed to unified schema | `Fetch RemoteOK Jobs` | `Merge All Job Sources` | Fetch RemoteOK jobs |
| `Fetch Jobicy Jobs` | `n8n-nodes-base.httpRequest` | Fetches remote jobs from Jobicy API | `Index Saved Job Data` | `Transform Jobicy Data` | Fetch Jobicy jobs |
| `Transform Jobicy Data` | `n8n-nodes-base.code` | Maps Jobicy feed to unified schema | `Fetch Jobicy Jobs` | `Merge All Job Sources` | Fetch Jobicy jobs |
| `Merge All Job Sources` | `n8n-nodes-base.merge` | Merges all six ingestion streams | *Transform & Aggregator nodes* | `Normalize Job Data` | Merge and normalize |
| `Normalize Job Data` | `n8n-nodes-base.code` | Sanitizes HTML and normalizes dates | `Merge All Job Sources` | `Filter and Rank Job Listings` | Merge and normalize |
| `Filter and Rank Job Listings` | `n8n-nodes-base.code` | Filters, deduplicates, ranks, and caps listings | `Normalize Job Data` | `Check for New Jobs` | Filter and decide |
| `Check for New Jobs` | `n8n-nodes-base.if` | Branches execution based on new matches | `Filter and Rank Job Listings` | `Split Job Listings`, `Generate Run Log Entry` | Filter and decide |
| `Split Job Listings` | `n8n-nodes-base.splitOut` | Splits job items for sheet insertion | `Check for New Jobs` | `Append Jobs to Google Sheets` | Append new applications |
| `Append Jobs to Google Sheets` | `n8n-nodes-base.googleSheets` | Appends new job records to spreadsheet | `Split Job Listings` | `Generate Run Log Entry` | Append new applications |
| `Generate Run Log Entry` | `n8n-nodes-base.code` | Compiles run execution metrics and health | `Check for New Jobs`, `Append Jobs to Google Sheets` | `Append to Run Log in Sheets` | Write run log |
| `Append to Run Log in Sheets` | `n8n-nodes-base.googleSheets` | Appends execution log to Run Log tab | `Generate Run Log Entry` | None | Write run log |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually recreate the workflow in n8n:

1. **Create Schedule Trigger:**
   - Add a **Schedule Trigger** node (`When Every 3 Days`).
   - Set interval type to `Days`, frequency to `3`, and configure execution hour to `09:00`.

2. **Configure Search Parameters:**
   - Add a **Set** node (`Set Job Search Criteria`).
   - Define custom assignments for `filters`, `greenhouse_boards`, `ashby_boards`, `lever_boards`, `location_us_explicit`, `location_block`, `location_allow`, `us_state_codes`, `max_results`, `max_per_company`, and date expressions (`run_at`, `run_date`).

3. **Read Existing Data:**
   - Add a **Google Sheets** node (`Read from Google Sheets`).
   - Connect credentials (OAuth2), select resource `sheet`, operation `read`, sheet name `Applications`. Enable `Always Output Data`.
   - Add a **Code** node (`Index Saved Job Data`) running once for all items to parse existing links and calculate the next serial number.

4. **Build Greenhouse Ingestion Branch:**
   - Add a **Code** node (`Distribute Greenhouse Boards`) with `Always Output Data` enabled to fan out board tokens.
   - Add an **HTTP Request** node (`Fetch Greenhouse Jobs`) requesting `https://boards-api.greenhouse.io/v1/boards/{{ $json.token }}/jobs?content=true`. Set error handling to continue execution and configure 3 retries.
   - Add a **Code** node (`Transform Greenhouse Data`) to map fields using paired-item tracking (`itemMatching`).

5. **Build Ashby Ingestion Branch:**
   - Add a **Code** node (`Distribute Ashby Boards`) with `Always Output Data` enabled.
   - Add an **HTTP Request** node (`Fetch Ashby Jobs`) requesting `https://api.ashbyhq.com/posting-api/job-board/{{ $json.board }}?includeCompensation=false`.
   - Add a **Code** node (`Transform Ashby Data`) to normalize Ashby fields.

6. **Build Lever Ingestion Branch:**
   - Add a **Code** node (`Distribute Lever Boards`) with `Always Output Data` enabled.
   - Add an **HTTP Request** node (`Fetch Lever Jobs`) requesting `https://api.lever.co/v0/postings/{{ $json.slug }}?mode=json`.
   - Add a **Code** node (`Transform Lever Data`) to format Lever records.

7. **Build Aggregator Ingestion Branches:**
   - Add **HTTP Request** nodes for **Remotive** (`https://remotive.com/api/remote-jobs`), **RemoteOK** (`https://remoteok.com/api`), and **Jobicy** (`https://jobicy.com/api/v2/remote-jobs` with query parameters `count=100` and `geo=usa`).
   - Attach respective **Code** nodes (`Transform Remotive Data`, `Transform RemoteOK Data`, `Transform Jobicy Data`) to parse each feed.

8. **Merge and Normalize Streams:**
   - Add a **Merge** node (`Merge All Job Sources`) configured in `append` mode with `6` inputs, connecting all six transform outputs.
   - Add a **Code** node (`Normalize Job Data`) to decode HTML entities and standardize dates into `YYYY-MM-DD`.

9. **Filter, Rank, and Evaluate:**
   - Add a **Code** node (`Filter and Rank Job Listings`) to execute multi-stage rejection filters, deduplication, ranking, and capping.
   - Add an **If** node (`Check for New Jobs`) verifying if `{{ $json.rows.length }}` > 0.

10. **Handle Output and Logging:**
    - On the True branch of the If node, add a **SplitOut** node (`Split Job Listings`) splitting on `rows`.
    - Add a **Google Sheets** node (`Append Jobs to Google Sheets`) targeting the `Applications` tab with error continuation enabled.
    - Add a **Code** node (`Generate Run Log Entry`) collecting execution stats from both active and bypassed branches.
    - Add a final **Google Sheets** node (`Append to Run Log in Sheets`) targeting the `Run Log` tab.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Workflow Purpose & Architecture Summary** | Complete automated job collection pipeline spanning company ATS boards and remote aggregators with built-in deduplication and logging. |
| **Google Sheets Setup Requirements** | Requires a spreadsheet containing two exact tabs: `Applications` and `Run Log` with pre-configured header rows matching the script mappings. |
| **API Compliance & Rate Limiting** | Uses appropriate request intervals (every 3 days) and includes custom User-Agent headers and attribution link retention to comply with aggregator terms of service. |