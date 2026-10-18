Compare venue Instagram posting rates with RecentReborn, Google Sheets and Telegram

https://n8nworkflows.xyz/workflows/compare-venue-instagram-posting-rates-with-recentreborn--google-sheets-and-telegram-20901


# Compare venue Instagram posting rates with RecentReborn, Google Sheets and Telegram

### 1. Workflow Overview

This workflow automates the tracking, benchmarking, and reporting of Instagram posting rates for user-defined venues versus competitors using data from the RecentReborn API, Google Sheets, and Telegram. It provides two primary execution vectors: an interactive setup path via an n8n form to discover and save venue locations, and a routine operational path (triggered manually or on a 4-hour schedule) that samples recent posts, calculates posting frequencies, logs metrics, and broadcasts weekly analytical summaries or urgent alerts.

The logic is segmented into the following functional blocks:
- **1.1 Input Reception & Configuration:** Ingests triggers from webhooks (venue setup form), schedules, or manual runs, and initializes all shared operational parameters.
- **1.2 Venue Location Lookup & Setup:** Searches the RecentReborn API for matching Instagram locations based on user input, validates results, presents an interactive form to select specific location IDs, and saves them to Google Sheets.
- **1.3 Venue Loading & Job Scheduling:** Reads active venues from Google Sheets, enforces limits, guards against empty configurations, and initiates the looping process.
- **1.4 Sampling Loop & API Request Handling:** Iterates through venues sequentially, evaluates quota and safety abort flags, throttles requests using wait nodes, queries RecentReborn for location posts, and handles errors robustly.
- **1.5 Data Persistance & State Update:** Summarizes sample outcomes, and sequentially writes sample logs, newly discovered posts, and updated venue watermarks back to Google Sheets.
- **1.6 Report Generation, AI Processing & Dispatch:** Evaluates reporting schedules, aggregates historical log data from Google Sheets, optionally summarizes captions using an OpenAI LLM chain, builds structured messages, and dispatches them via Telegram.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Configuration
**Overview:** This block handles multiple entry points into the workflow—form submissions for adding new venues, a 4-hour interval schedule, and manual test executions—unifying them by passing execution data into a centralized configuration node.

**Nodes Involved:**
- `When Venue Setup Form Submitted` (n8n-nodes-base.formTrigger)
- `Every 4 Hours` (n8n-nodes-base.scheduleTrigger)
- `When Manually Triggered` (n8n-nodes-base.manualTrigger)
- `Set Venue Configuration` (n8n-nodes-base.set)
- `If Venue Form Submitted` (n8n-nodes-base.if)

**Node Details:**
1. **When Venue Setup Form Submitted**
   - **Type & Role:** Form Trigger (`formTrigger`). Webhook entry point capturing venue name and group designation.
   - **Configuration:** Path set to `venue-benchmark-setup`, requests fields for "Venue name" (text) and "Group" (dropdown: Own venue / Competitor).
   - **Inputs/Outputs:** Input: None (Trigger). Output: Connected to `Set Venue Configuration`.
   - **Edge Cases:** Invalid form submissions will fail to map parameters correctly; mitigated by downstream validation.

2. **Every 4 Hours**
   - **Type & Role:** Schedule Trigger (`scheduleTrigger`). Routine automation trigger.
   - **Configuration:** Interval set to fire every 4 hours.
   - **Inputs/Outputs:** Input: None. Output: Connected to `Set Venue Configuration`.

3. **When Manually Triggered**
   - **Type & Role:** Manual Trigger (`manualTrigger`). Developer testing entry point.
   - **Configuration:** Default parameters.
   - **Inputs/Outputs:** Input: None. Output: Connected to `Set Venue Configuration`.

4. **Set Venue Configuration**
   - **Type & Role:** Set Node (`set`). Global configuration initialization block.
   - **Configuration:** Defines variables including `sheetUrl`, `telegramChatId`, `setupFormUrl`, `timezone`, `postsPerSample`, `minPostsForRate`, `lookbackCapHours`, `maxVenues`, `secondsBetweenRequests`, `minQuotaRemaining`, `reportWeekday`, `reportHour`, `reportWindowDays`, `sendReportOnManualRun`, `topPostsPerGroup`, `useAi`, `aiCaptionsPerVenue`, and `aiInstructions`.
   - **Inputs/Outputs:** Input: From any trigger node. Output: Connected to `If Venue Form Submitted`.

5. **If Venue Form Submitted**
   - **Type & Role:** If Node (`if`). Conditional router.
   - **Configuration:** Checks if incoming payload contains a defined `Venue name` property to branch between setup form processing or routine sampling runs.
   - **Inputs/Outputs:** Input: `Set Venue Configuration`. True Output: `Fetch Venue Locations`. False Output: `Read Venues Data from Sheets`.

---

#### 2.2 Venue Location Lookup & Setup
**Overview:** Queries the RecentReborn API for location matching, transforms the API output into user-selectable multi-select dropdown options, and handles any lookup errors or empty datasets gracefully.

**Nodes Involved:**
- `Fetch Venue Locations` (n8n-nodes-base.httpRequest)
- `Build Venue Selection Form` (n8n-nodes-base.code)
- `If Venues Available` (n8n-nodes-base.if)
- `Venue Selection Form` (n8n-nodes-base.form)
- `Display Lookup Error` (n8n-nodes-base.form)
- `Map Venue Selection to IDs` (n8n-nodes-base.code)
- `Save Venues Data to Sheets` (n8n-nodes-base.googleSheets)
- `Notify Venues Saved` (n8n-nodes-base.form)

**Node Details:**
1. **Fetch Venue Locations**
   - **Type & Role:** HTTP Request (`httpRequest`). External API connector.
   - **Configuration:** GET request to `https://app.recentreborn.com/api/v1/locations` with query parameter `query` mapped to input venue name. Uses Generic Credential Type (HTTP Header Auth). Timeout: 30,000ms. Configured with `onError: continueErrorOutput`.
   - **Inputs/Outputs:** Input: `If Venue Form Submitted`. Outputs: Main and Error outputs connected to `Build Venue Selection Form`.
   - **Edge Cases:** Network failure, invalid API key (401), rate limits (429). Captured via error branch.

2. **Build Venue Selection Form**
   - **Type & Role:** Code Node (`code`). Data transformer.
   - **Configuration:** JavaScript parsing HTTP responses, extracting location lists, removing duplicates, and constructing form choice payloads.
   - **Inputs/Outputs:** Input: `Fetch Venue Locations`. Output: Connected to `If Venues Available`.

3. **If Venues Available**
   - **Type & Role:** If Node (`if`). Conditional router.
   - **Configuration:** Validates if `status === 'found'`.
   - **Inputs/Outputs:** Input: `Build Venue Selection Form`. True Output: `Venue Selection Form`. False Output: `Display Lookup Error`.

4. **Venue Selection Form**
   - **Type & Role:** Form Node (`form`). Interactive user prompt.
   - **Configuration:** Dynamically renders form fields via JSON output from the previous code node, prompting the user to select correct location matches.
   - **Inputs/Outputs:** Input: `If Venues Available`. Output: Connected to `Map Venue Selection to IDs`.

5. **Display Lookup Error**
   - **Type & Role:** Form Node (`form`). Completion/Error message display.
   - **Configuration:** Operation set to `completion`, displaying failure reasons or empty search guidance.
   - **Inputs/Outputs:** Input: `If Venues Available`. Output: Terminal node for failed lookups.

6. **Map Venue Selection to IDs**
   - **Type & Role:** Code Node (`code`). Data mapper.
   - **Configuration:** Maps selected location labels back to structured rows (`Venue ID`, `Venue`, `Group`, `Search Query`, `Active`, `Added At`) destined for Google Sheets.
   - **Inputs/Outputs:** Input: `Venue Selection Form`. Output: Connected to `Save Venues Data to Sheets`.

7. **Save Venues Data to Sheets**
   - **Type & Role:** Google Sheets Node (`googleSheets`). Database writer.
   - **Configuration:** Operation set to `appendOrUpdate`, target sheet `Venues`, matching columns `Venue ID`. Uses Google Sheets OAuth2 API.
   - **Inputs/Outputs:** Input: `Map Venue Selection to IDs`. Output: Connected to `Notify Venues Saved`.

8. **Notify Venues Saved**
   - **Type & Role:** Form Node (`form`). Completion notifier.
   - **Configuration:** Operation set to `completion` confirming successful venue storage.
   - **Inputs/Outputs:** Input: `Save Venues Data to Sheets`. Output: Terminal node.

---

#### 2.3 Venue Loading & Job Scheduling
**Overview:** Reads active venues from Google Sheets, handles read failures, and transforms sheet rows into structured sampling jobs while implementing safeguards against missing configurations.

**Nodes Involved:**
- `Read Venues Data from Sheets` (n8n-nodes-base.googleSheets)
- `Notify Venue Read Failure` (n8n-nodes-base.telegram)
- `Schedule Sampling Jobs` (n8n-nodes-base.code)
- `If Venue Setup Needed` (n8n-nodes-base.if)
- `Notify Setup Required` (n8n-nodes-base.telegram)

**Node Details:**
1. **Read Venues Data from Sheets**
   - **Type & Role:** Google Sheets Node (`googleSheets`). Database reader.
   - **Configuration:** Operation set to read, sheet name `Venues`. Configured with `alwaysOutputData: true` and `onError: continueErrorOutput`.
   - **Inputs/Outputs:** Input: `If Venue Form Submitted`. Outputs: Main and Error outputs connected to `Schedule Sampling Jobs` and `Notify Venue Read Failure`.

2. **Notify Venue Read Failure**
   - **Type & Role:** Telegram Node (`telegram`). Alert dispatcher.
   - **Configuration:** Sends HTML-formatted alert to configured chat ID regarding sheet read failures.
   - **Inputs/Outputs:** Input: `Read Venues Data from Sheets` (Error branch). Output: Terminal node.

3. **Schedule Sampling Jobs**
   - **Type & Role:** Code Node (`code`). Job builder & rate limiter.
   - **Configuration:** Filters active rows from the sheet, caps total venues processed based on `maxVenues`, and flags if setup is required when no active venues are found.
   - **Inputs/Outputs:** Input: `Read Venues Data from Sheets`. Output: Connected to `If Venue Setup Needed`.

4. **If Venue Setup Needed**
   - **Type & Role:** If Node (`if`). Conditional router.
   - **Configuration:** Evaluates `setupNeeded` boolean flag.
   - **Inputs/Outputs:** Input: `Schedule Sampling Jobs`. True Output: `Notify Setup Required`. False Output: `Batch Process Venues`.

5. **Notify Setup Required**
   - **Type & Role:** Telegram Node (`telegram`). Notification dispatcher.
   - **Configuration:** Sends an HTML message prompting the user to configure initial venues.
   - **Inputs/Outputs:** Input: `If Venue Setup Needed`. Output: Terminal node.

---

#### 2.4 Sampling Loop & API Request Handling
**Overview:** Iterates through venues in batches, checks daily quotas and fatal error flags, throttles execution to avoid rate limits, fetches location posts from RecentReborn, and calculates posting rates or logs failure markers.

**Nodes Involved:**
- `Batch Process Venues` (n8n-nodes-base.splitInBatches)
- `Prepare Venue Sample Request` (n8n-nodes-base.code)
- `If Sample Request Skipped` (n8n-nodes-base.if)
- `Wait Between API Requests` (n8n-nodes-base.wait)
- `Get Venue Posts Data` (n8n-nodes-base.httpRequest)
- `Calculate Posting Rate` (n8n-nodes-base.code)
- `Log Search Failure` (n8n-nodes-base.code)

**Node Details:**
1. **Batch Process Venues**
   - **Type & Role:** Split In Batches (`splitInBatches`). Loop controller.
   - **Configuration:** Processes venues sequentially (default batch size 1). Loops back from calculation/failure nodes until complete, then routes to summarization.
   - **Inputs/Outputs:** Inputs: `If Venue Setup Needed`, `Calculate Posting Rate`, `Log Search Failure`. Outputs: Loop continuation output connected to `Prepare Venue Sample Request`; completion output connected to `Summarize Sampling Outcome`.

2. **Prepare Venue Sample Request**
   - **Type & Role:** Code Node (`code`). Pre-flight validator.
   - **Configuration:** Checks global state for run abortion flags or exhausted quota limits, setting skip statuses if thresholds are breached.
   - **Inputs/Outputs:** Input: `Batch Process Venues`. Output: Connected to `If Sample Request Skipped`.

3. **If Sample Request Skipped**
   - **Type & Role:** If Node (`if`). Conditional router.
   - **Configuration:** Checks if `skipped === true`.
   - **Inputs/Outputs:** Input: `Prepare Venue Sample Request`. True Output: Loops back to `Batch Process Venues`. False Output: `Wait Between API Requests`.

4. **Wait Between API Requests**
   - **Type & Role:** Wait Node (`wait`). Rate limiter.
   - **Configuration:** Pauses execution for a configurable number of seconds (`secondsBetweenRequests`, default 4).
   - **Inputs/Outputs:** Input: `If Sample Request Skipped`. Output: Connected to `Get Venue Posts Data`.

5. **Get Venue Posts Data**
   - **Type & Role:** HTTP Request (`httpRequest`). External API connector.
   - **Configuration:** GET request to `https://app.recentreborn.com/api/v1/search/location` passing `location_id`. Full response captured to parse rate-limit headers (`x-ratelimit-remaining`, `x-ratelimit-reset`). Retries enabled (`maxTries: 2`). `onError: continueErrorOutput`.
   - **Inputs/Outputs:** Input: `Wait Between API Requests`. Outputs: Main and Error outputs connected to `Calculate Posting Rate` and `Log Search Failure`.

6. **Calculate Posting Rate**
   - **Type & Role:** Code Node (`code`). Mathematical processor.
   - **Configuration:** Parses post timestamps, filters out future/invalid dates, computes decay windows, calculates posting rates per hour, extracts fresh post metrics (Reels, likes, hashtags, captions), and stores rate limit headers in global workflow state.
   - **Inputs/Outputs:** Input: `Get Venue Posts Data`. Output: Loops back to `Batch Process Venues`.

7. **Log Search Failure**
   - **Type & Role:** Code Node (`code`). Error handler.
   - **Configuration:** Evaluates HTTP status codes (e.g., 401, 403, 429), flags fatal errors to abort subsequent venue queries in the run, and structures diagnostic error messages.
   - **Inputs/Outputs:** Input: `Get Venue Posts Data` (Error branch). Output: Loops back to `Batch Process Venues`.

---

#### 2.5 Data Persistence & State Update
**Overview:** Consolidates batch results, formats rows for Google Sheets, appends sample records and newly discovered posts, updates venue states/watermarks, and handles sheet write errors.

**Nodes Involved:**
- `Summarize Sampling Outcome` (n8n-nodes-base.code)
- `Divide Sample Rows` (n8n-nodes-base.splitOut)
- `Add Samples Data to Sheets` (n8n-nodes-base.googleSheets)
- `Proceed with Sample Results` (n8n-nodes-base.code)
- `If New Posts Available` (n8n-nodes-base.if)
- `Divide Post Rows` (n8n-nodes-base.splitOut)
- `Add Posts Data to Sheets` (n8n-nodes-base.googleSheets)
- `Proceed with Post Results` (n8n-nodes-base.code)
- `Divide Venue Rows` (n8n-nodes-base.splitOut)
- `Update Venues Data in Sheets` (n8n-nodes-base.googleSheets)
- `Proceed with Venue Updates` (n8n-nodes-base.code)
- `Notify Sheet Save Failure` (n8n-nodes-base.telegram)

**Node Details:**
1. **Summarize Sampling Outcome**
   - **Type & Role:** Code Node (`code`). Aggregator.
   - **Configuration:** Runs once post-loop; compiles sample rows, post rows, venue state updates, quota counters, and calculates whether the weekly report is due based on timezone, weekday, and hour settings.
   - **Inputs/Outputs:** Input: `Batch Process Venues` (Completion branch). Output: Connected to `Divide Sample Rows`.

2. **Divide Sample Rows**
   - **Type & Role:** Split Out (`splitOut`). Array flattener.
   - **Configuration:** Splits `sampleRows` array into individual items for database insertion.
   - **Inputs/Outputs:** Input: `Summarize Sampling Outcome`. Output: Connected to `Add Samples Data to Sheets`.

3. **Add Samples Data to Sheets**
   - **Type & Role:** Google Sheets Node (`googleSheets`). Database writer.
   - **Configuration:** Appends rows to sheet `Samples`. `onError: continueErrorOutput`.
   - **Inputs/Outputs:** Input: `Divide Sample Rows`. Outputs: Main and Error outputs connected to `Proceed with Sample Results` and `Notify Sheet Save Failure`.

4. **Proceed with Sample Results**
   - **Type & Role:** Code Node (`code`). State restorer.
   - **Configuration:** Restores the unified run summary object after sheet row processing.
   - **Inputs/Outputs:** Input: `Add Samples Data to Sheets`. Output: Connected to `If New Posts Available`.

5. **If New Posts Available**
   - **Type & Role:** If Node (`if`). Conditional router.
   - **Configuration:** Checks if `postRows.length > 0`.
   - **Inputs/Outputs:** Input: `Proceed with Sample Results`. True Output: `Divide Post Rows`. False Output: `Divide Venue Rows`.

6. **Divide Post Rows**
   - **Type & Role:** Split Out (`splitOut`). Array flattener.
   - **Configuration:** Splits `postRows` array into individual items.
   - **Inputs/Outputs:** Input: `If New Posts Available`. Output: Connected to `Add Posts Data to Sheets`.

7. **Add Posts Data to Sheets**
   - **Type & Role:** Google Sheets Node (`googleSheets`). Database writer.
   - **Configuration:** Appends rows to sheet `Posts`. `onError: continueErrorOutput`.
   - **Inputs/Outputs:** Input: `Divide Post Rows`. Outputs: Main and Error outputs connected to `Proceed with Post Results` and `Notify Sheet Save Failure`.

8. **Proceed with Post Results**
   - **Type & Role:** Code Node (`code`). State restorer.
   - **Configuration:** Restores unified run summary object.
   - **Inputs/Outputs:** Input: `Add Posts Data to Sheets`. Output: Connected to `Divide Venue Rows`.

9. **Divide Venue Rows**
   - **Type & Role:** Split Out (`splitOut`). Array flattener.
   - **Configuration:** Splits `venueRows` array into individual items.
   - **Inputs/Outputs:** Inputs: `If New Posts Available`, `Proceed with Post Results`. Output: Connected to `Update Venues Data in Sheets`.

10. **Update Venues Data in Sheets**
    - **Type & Role:** Google Sheets Node (`googleSheets`). Database writer.
    - **Configuration:** Operation set to `appendOrUpdate`, sheet `Venues`, matching columns `Venue ID`. `onError: continueErrorOutput`.
    - **Inputs/Outputs:** Input: `Divide Venue Rows`. Outputs: Main and Error outputs connected to `Proceed with Venue Updates` and `Notify Sheet Save Failure`.

11. **Proceed with Venue Updates**
    - **Type & Role:** Code Node (`code`). State restorer.
    - **Configuration:** Restores unified run summary object.
    - **Inputs/Outputs:** Input: `Update Venues Data in Sheets`. Output: Connected to `Check Report Due`.

12. **Notify Sheet Save Failure**
    - **Type & Role:** Telegram Node (`telegram`). Alert dispatcher.
    - **Configuration:** Sends an HTML-formatted error message indicating Google Sheets write failures.
    - **Inputs/Outputs:** Input: Error outputs of Sheets nodes. Output: Terminal node.

---

#### 2.6 Report Generation, AI Processing & Dispatch
**Overview:** Evaluates reporting schedules, queries historical logs from Google Sheets, computes comparative weekly statistics, optionally generates AI-based caption summaries via OpenAI, formats outbound messages, dispatches them via Telegram, and logs delivery success.

**Nodes Involved:**
- `Check Report Due` (n8n-nodes-base.if)
- `Read Sample Log from Sheets` (n8n-nodes-base.googleSheets)
- `Read Post Log from Sheets` (n8n-nodes-base.googleSheets)
- `Notify Report Read Failure` (n8n-nodes-base.telegram)
- `Compute Weekly Metrics` (n8n-nodes-base.code)
- `Check AI Summary Enabled` (n8n-nodes-base.if)
- `AI-Caption Summary` (`@n8n/n8n-nodes-langchain.chainLlm`)
- `OpenAI Chatbot Model` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
- `Create Weekly Report Messages` (n8n-nodes-base.code)
- `Create Alert Messages` (n8n-nodes-base.code)
- `Dispatch Telegram Messages` (n8n-nodes-base.telegram)
- `Log Report Sent Status` (n8n-nodes-base.code)

**Node Details:**
1. **Check Report Due**
   - **Type & Role:** If Node (`if`). Conditional router.
   - **Configuration:** Checks if `reportDue === true`.
   - **Inputs/Outputs:** Input: `Proceed with Venue Updates`. True Output: `Read Sample Log from Sheets`. False Output: `Create Alert Messages`.

2. **Read Sample Log from Sheets**
   - **Type & Role:** Google Sheets Node (`googleSheets`). Database reader.
   - **Configuration:** Reads sheet `Samples`. `alwaysOutputData: true`, `onError: continueErrorOutput`.
   - **Inputs/Outputs:** Input: `Check Report Due`. Outputs: Main and Error outputs connected to `Read Post Log from Sheets` and `Notify Report Read Failure`.

3. **Read Post Log from Sheets**
   - **Type & Role:** Google Sheets Node (`googleSheets`). Database reader.
   - **Configuration:** Reads sheet `Posts`. `alwaysOutputData: true`, `onError: continueErrorOutput`.
   - **Inputs/Outputs:** Input: `Read Sample Log from Sheets`. Outputs: Main and Error outputs connected to `Compute Weekly Metrics` and `Notify Report Read Failure`.

4. **Notify Report Read Failure**
   - **Type & Role:** Telegram Node (`telegram`). Alert dispatcher.
   - **Configuration:** Sends error alert when historical logs cannot be read.
   - **Inputs/Outputs:** Input: Error outputs of log reading nodes. Output: Terminal node.

5. **Compute Weekly Metrics**
   - **Type & Role:** Code Node (`code`). Analytics processor.
   - **Configuration:** Aggregates sample and post logs over the reporting window, calculates median rates, volume shares, busiest dayparts, hashtag frequencies, and compiles the batch prompt payload for OpenAI.
   - **Inputs/Outputs:** Input: `Read Post Log from Sheets`. Output: Connected to `Check AI Summary Enabled`.

6. **Check AI Summary Enabled**
   - **Type & Role:** If Node (`if`). Conditional router.
   - **Configuration:** Checks `sendToAi === 'yes'`.
   - **Inputs/Outputs:** Input: `Compute Weekly Metrics`. True Output: `AI-Caption Summary`. False Output: `Create Weekly Report Messages`.

7. **AI-Caption Summary**
   - **Type & Role:** LangChain LLM Chain (`@n8n/n8n-nodes-langchain.chainLlm`). AI processor.
   - **Configuration:** Executes text prompt utilizing configured instructions to analyze captions, themes, tones, and return JSON. `onError: continueRegularOutput`.
   - **Inputs/Outputs:** Input: `Check AI Summary Enabled`. AI Model Input: Connected from `OpenAI Chatbot Model`. Output: Connected to `Create Weekly Report Messages`.

8. **OpenAI Chatbot Model**
   - **Type & Role:** OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`). Model provider config.
   - **Configuration:** Model model name set (`gpt-6-luna`), response format configured as `json_object`. Uses OpenAI API credentials.
   - **Inputs/Outputs:** Output: Connected to `AI-Caption Summary` via `ai_languageModel` connection.

9. **Create Weekly Report Messages**
   - **Type & Role:** Code Node (`code`). Message formatter.
   - **Configuration:** Validates AI outputs, constructs HTML-formatted Telegram message chunks (respecting 4,096 character limits), and includes rankings, metrics, watchlists, and alerts.
   - **Inputs/Outputs:** Inputs: `AI-Caption Summary`, `Check AI Summary Enabled` (False branch). Output: Connected to `Dispatch Telegram Messages`.

10. **Create Alert Messages**
    - **Type & Role:** Code Node (`code`). Message formatter.
    - **Configuration:** Formats urgent operational alerts or manual test run health summaries when full weekly reports are not due.
    - **Inputs/Outputs:** Input: `Check Report Due` (False branch). Output: Connected to `Dispatch Telegram Messages`.

11. **Dispatch Telegram Messages**
    - **Type & Role:** Telegram Node (`telegram`). Message broadcaster.
    - **Configuration:** Sends messages to configured chat ID using HTML parse mode.
    - **Inputs/Outputs:** Inputs: `Create Weekly Report Messages`, `Create Alert Messages`. Output: Connected to `Log Report Sent Status`.

12. **Log Report Sent Status**
    - **Type & Role:** Code Node (`code`). State updater.
    - **Configuration:** Updates workflow static data to record successful weekly report delivery timestamps and local dates.
    - **Inputs/Outputs:** Input: `Dispatch Telegram Messages`. Output: Terminal node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation overview | None | None | ## Compare Instagram posting rates at your venues and competitors with RecentReborn, Google Sheets and Telegram... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Start and configure block note | None | None | ## Start and configure<br><br>Entry cluster for setup form submissions, scheduled runs, and manual tests... |
| Sticky Note2 | n8n-nodes-base.stickyNote | Venue location lookup block note | None | None | ## Venue location lookup<br><br>Looks up available venue locations in RecentReborn... |
| Sticky Note3 | n8n-nodes-base.stickyNote | Venue selection forms block note | None | None | ## Venue selection forms<br><br>Presents either the venue selection form when locations are available... |
| Sticky Note4 | n8n-nodes-base.stickyNote | Save selected venues block note | None | None | ## Save selected venues<br><br>Converts selected venue labels back into RecentReborn location IDs... |
| Sticky Note5 | n8n-nodes-base.stickyNote | Load venues and setup block note | None | None | ## Load venues and setup<br><br>Reads active venues from the spreadsheet... |
| Sticky Note6 | n8n-nodes-base.stickyNote | Sample loop gate block note | None | None | ## Sample loop gate<br><br>Iterates over configured venues, prepares each RecentReborn sample request... |
| Sticky Note7 | n8n-nodes-base.stickyNote | Fetch and classify posts block note | None | None | ## Fetch and classify posts<br><br>Spaces out API calls, fetches posts for the current venue... |
| Sticky Note8 | n8n-nodes-base.stickyNote | Summarize sample results block note | None | None | ## Summarize sample results<br><br>Runs after the venue loop, builds the single run summary... |
| Sticky Note9 | n8n-nodes-base.stickyNote | Restore summary branch block note | None | None | ## Restore summary branch<br><br>Collapses the per-row Google Sheets output back to the run summary... |
| Sticky Note10 | n8n-nodes-base.stickyNote | Append new posts block note | None | None | ## Append new posts<br><br>Splits new post records, appends them to the Posts tab... |
| Sticky Note11 | n8n-nodes-base.stickyNote | Update venue state block note | None | None | ## Update venue state<br><br>Splits updated venue rows, writes current venue state back to Google Sheets... |
| Sticky Note12 | n8n-nodes-base.stickyNote | Report routing and reads block note | None | None | ## Report routing and reads<br><br>Checks whether a full report is due, reads historical Samples and Posts logs... |
| Sticky Note13 | n8n-nodes-base.stickyNote | Compute report stats block note | None | None | ## Compute report stats<br><br>Builds weekly comparison statistics from the logs... |
| Sticky Note14 | n8n-nodes-base.stickyNote | Optional AI summary block note | None | None | ## Optional AI summary<br><br>Uses an OpenAI chat model through the LLM Chain to summarize caption themes... |
| Sticky Note15 | n8n-nodes-base.stickyNote | Build outbound messages block note | None | None | ## Build outbound messages<br><br>Creates either full weekly report messages or urgent alert-only messages... |
| Sticky Note16 | n8n-nodes-base.stickyNote | Send and record block note | None | None | ## Send and record<br><br>Sends the prepared Telegram messages and records delivery... |
| When Venue Setup Form Submitted | n8n-nodes-base.formTrigger | Trigger workflow via form submission | None | Set Venue Configuration | Start and configure |
| Every 4 Hours | n8n-nodes-base.scheduleTrigger | Routine schedule trigger | None | Set Venue Configuration | Start and configure |
| When Manually Triggered | n8n-nodes-base.manualTrigger | Manual execution trigger | None | Set Venue Configuration | Start and configure |
| Set Venue Configuration | n8n-nodes-base.set | Initialize global variables | When Venue Setup Form Submitted, Every 4 Hours, When Manually Triggered | If Venue Form Submitted | Start and configure |
| If Venue Form Submitted | n8n-nodes-base.if | Route based on entry type | Set Venue Configuration | Fetch Venue Locations, Read Venues Data from Sheets | Start and configure |
| Fetch Venue Locations | n8n-nodes-base.httpRequest | Query RecentReborn API for locations | If Venue Form Submitted | Build Venue Selection Form | Venue location lookup |
| Build Venue Selection Form | n8n-nodes-base.code | Transform location data into choices | Fetch Venue Locations | If Venues Available | Venue location lookup |
| If Venues Available | n8n-nodes-base.if | Validate location query results | Build Venue Selection Form | Venue Selection Form, Display Lookup Error | Venue location lookup |
| Venue Selection Form | n8n-nodes-base.form | Interactive location picker | If Venues Available | Map Venue Selection to IDs | Venue selection forms |
| Display Lookup Error | n8n-nodes-base.form | Display search failure completion | If Venues Available | None | Venue selection forms |
| Map Venue Selection to IDs | n8n-nodes-base.code | Map selections to sheet rows | Venue Selection Form | Save Venues Data to Sheets | Save selected venues |
| Save Venues Data to Sheets | n8n-nodes-base.googleSheets | Save venues to Google Sheets | Map Venue Selection to IDs | Notify Venues Saved | Save selected venues |
| Notify Venues Saved | n8n-nodes-base.form | Display success completion | Save Venues Data to Sheets | None | Save selected venues |
| Read Venues Data from Sheets | n8n-nodes-base.googleSheets | Read active venues from sheet | If Venue Form Submitted | Schedule Sampling Jobs, Notify Venue Read Failure | Load venues and setup |
| Notify Venue Read Failure | n8n-nodes-base.telegram | Send sheet read error alert | Read Venues Data from Sheets | None | Load venues and setup |
| Schedule Sampling Jobs | n8n-nodes-base.code | Prepare venue sampling jobs | Read Venues Data from Sheets | If Venue Setup Needed | Load venues and setup |
| If Venue Setup Needed | n8n-nodes-base.if | Check if venues exist | Schedule Sampling Jobs | Notify Setup Required, Batch Process Venues | Load venues and setup |
| Notify Setup Required | n8n-nodes-base.telegram | Send setup reminder alert | If Venue Setup Needed | None | Load venues and setup |
| Batch Process Venues | n8n-nodes-base.splitInBatches | Iterate venues sequentially | If Venue Setup Needed, Calculate Posting Rate, Log Search Failure | Prepare Venue Sample Request, Summarize Sampling Outcome | Sample loop gate |
| Prepare Venue Sample Request | n8n-nodes-base.code | Pre-flight request validation | Batch Process Venues | If Sample Request Skipped | Sample loop gate |
| If Sample Request Skipped | n8n-nodes-base.if | Check skip conditions | Prepare Venue Sample Request | Batch Process Venues (loop), Wait Between API Requests | Sample loop gate |
| Wait Between API Requests | n8n-nodes-base.wait | Throttle request execution | If Sample Request Skipped | Get Venue Posts Data | Fetch and classify posts |
| Get Venue Posts Data | n8n-nodes-base.httpRequest | Fetch posts for location | Wait Between API Requests | Calculate Posting Rate, Log Search Failure | Fetch and classify posts |
| Calculate Posting Rate | n8n-nodes-base.code | Compute posting rate metrics | Get Venue Posts Data | Batch Process Venues | Fetch and classify posts |
| Log Search Failure | n8n-nodes-base.code | Handle search API errors | Get Venue Posts Data | Batch Process Venues | Fetch and classify posts |
| Summarize Sampling Outcome | n8n-nodes-base.code | Aggregate run results and metrics | Batch Process Venues | Divide Sample Rows | Summarize sample results |
| Divide Sample Rows | n8n-nodes-base.splitOut | Flatten sample rows array | Summarize Sampling Outcome | Add Samples Data to Sheets | Summarize sample results |
| Add Samples Data to Sheets | n8n-nodes-base.googleSheets | Append sample logs to sheet | Divide Sample Rows | Proceed with Sample Results, Notify Sheet Save Failure | Summarize sample results |
| Proceed with Sample Results | n8n-nodes-base.code | Restore run summary context | Add Samples Data to Sheets | If New Posts Available | Summarize sample results |
| If New Posts Available | n8n-nodes-base.if | Check for newly found posts | Proceed with Sample Results | Divide Post Rows, Divide Venue Rows | Restore summary branch |
| Divide Post Rows | n8n-nodes-base.splitOut | Flatten post rows array | If New Posts Available | Add Posts Data to Sheets | Append new posts |
| Add Posts Data to Sheets | n8n-nodes-base.googleSheets | Append new posts to sheet | Divide Post Rows | Proceed with Post Results, Notify Sheet Save Failure | Append new posts |
| Proceed with Post Results | n8n-nodes-base.code | Restore run summary context | Add Posts Data to Sheets | Divide Venue Rows | Append new posts |
| Divide Venue Rows | n8n-nodes-base.splitOut | Flatten venue rows array | If New Posts Available, Proceed with Post Results | Update Venues Data in Sheets | Update venue state |
| Update Venues Data in Sheets | n8n-nodes-base.googleSheets | Update venue state and watermarks | Divide Venue Rows | Proceed with Venue Updates, Notify Sheet Save Failure | Update venue state |
| Proceed with Venue Updates | n8n-nodes-base.code | Restore run summary context | Update Venues Data in Sheets | Check Report Due | Update venue state |
| Notify Sheet Save Failure | n8n-nodes-base.telegram | Send sheet write error alert | Add Samples Data to Sheets, Add Posts Data to Sheets, Update Venues Data in Sheets | None | Update venue state |
| Check Report Due | n8n-nodes-base.if | Check if weekly report is due | Proceed with Venue Updates | Read Sample Log from Sheets, Create Alert Messages | Report routing and reads |
| Read Sample Log from Sheets | n8n-nodes-base.googleSheets | Read historical samples | Check Report Due | Read Post Log from Sheets, Notify Report Read Failure | Report routing and reads |
| Read Post Log from Sheets | n8n-nodes-base.googleSheets | Read historical posts | Read Sample Log from Sheets | Compute Weekly Metrics, Notify Report Read Failure | Report routing and reads |
| Notify Report Read Failure | n8n-nodes-base.telegram | Send report read error alert | Read Sample Log from Sheets, Read Post Log from Sheets | None | Report routing and reads |
| Compute Weekly Metrics | n8n-nodes-base.code | Compute weekly statistics | Read Post Log from Sheets | Check AI Summary Enabled | Compute report stats |
| Check AI Summary Enabled | n8n-nodes-base.if | Check if AI summaries are enabled | Compute Weekly Metrics | AI-Caption Summary, Create Weekly Report Messages | Compute report stats |
| AI-Caption Summary | @n8n/n8n-nodes-langchain.chainLlm | Generate AI caption summaries | Check AI Summary Enabled | Create Weekly Report Messages | Optional AI summary |
| OpenAI Chatbot Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide OpenAI model config | None | AI-Caption Summary | Optional AI summary |
| Create Weekly Report Messages | n8n-nodes-base.code | Format weekly report messages | AI-Caption Summary, Check AI Summary Enabled | Dispatch Telegram Messages | Build outbound messages |
| Create Alert Messages | n8n-nodes-base.code | Format alert messages | Check Report Due | Dispatch Telegram Messages | Build outbound messages |
| Dispatch Telegram Messages | n8n-nodes-base.telegram | Send messages to Telegram | Create Weekly Report Messages, Create Alert Messages | Log Report Sent Status | Send and record |
| Log Report Sent Status | n8n-nodes-base.code | Record report delivery status | Dispatch Telegram Messages | None | Send and record |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Credentials:**
   - **RecentReborn API:** Set up a Generic Credential Type (HTTP Header Auth) with header name `Authorization` and value `Bearer <YOUR_RECENTREBORN_API_KEY>`.
   - **Google Sheets:** Create a Google Sheets OAuth2 API credential linked to your Google account.
   - **Telegram Bot:** Create a Telegram Bot API credential using your BotFather token.
   - **OpenAI:** Create an OpenAI API credential (if utilizing AI reporting).

2. **Prepare Google Sheets:**
   - Create a Google Sheet containing three tabs: `Venues`, `Samples`, and `Posts`.
   - Add required headers in `Venues`: `Venue ID`, `Venue`, `Group`, `Search Query`, `Active`, `Added At`, `Last Post At`, `Last Sampled At`, `Last Rate Per Hour`, `Last Status`.
   - Add required headers in `Samples`: `Sampled At`, `Local Time`, `Daypart`, `Venue ID`, `Venue`, `Group`, `Status`, `Posts Used`, `Window Hours`, `Rate Per Hour`, `Distinct Authors`, `Note`.
   - Add required headers in `Posts`: `Post URL`, `Posted At`, `Venue ID`, `Venue`, `Group`, `Format`, `Likes`, `Hashtags`, `Caption`, `Sampled At`.

3. **Build Entry Points & Configuration:**
   - **Node 1 (`When Venue Setup Form Submitted`):** Add a Form Trigger node with path `venue-benchmark-setup`, title "Add venues to compare", and fields for `Venue name` (string) and `Group` (dropdown: Own venue / Competitor).
   - **Node 2 (`Every 4 Hours`):** Add a Schedule Trigger node set to every 4 hours.
   - **Node 3 (`When Manually Triggered`):** Add a Manual Trigger node.
   - **Node 4 (`Set Venue Configuration`):** Add a Set node (`set`). Assign `sheetUrl` (string, your sheet URL), `telegramChatId` (string, your Telegram chat ID), `setupFormUrl` (string), `timezone` (`Europe/Berlin`), and operational thresholds (`postsPerSample`: 20, `minPostsForRate`: 3, `lookbackCapHours`: 72, `maxVenues`: 15, `secondsBetweenRequests`: 4, `minQuotaRemaining`: 100, `reportWeekday`: 1, `reportHour`: 8, `reportWindowDays`: 7, `sendReportOnManualRun`: false, `topPostsPerGroup`: 3, `useAi`: true, `aiCaptionsPerVenue`: 15, `aiInstructions`: [Provided prompt text]).
   - **Node 5 (`If Venue Form Submitted`):** Add an If node checking `={{ $json['Venue name'] !== undefined ? 'yes' : 'no' }}` equals `yes`. Connect `Set Venue Configuration` output here.

4. **Build Location Lookup & Setup Branch:**
   - **Node 6 (`Fetch Venue Locations`):** Add an HTTP Request node (GET to `https://app.recentreborn.com/api/v1/locations`, query parameter `query` = `={{ $json['Venue name'] }}`). Set authentication to HTTP Header Auth. Set `onError` to continue error output.
   - **Node 7 (`Build Venue Selection Form`):** Add a Code node transforming the location array into multi-select form choices.
   - **Node 8 (`If Venues Available`):** Add an If node checking `={{ $json.status }}` equals `found`.
   - **Node 9 (`Venue Selection Form`):** Add a Form node (`form`) defining form fields via JSON output `={{ JSON.stringify($json.formFields) }}`.
   - **Node 10 (`Display Lookup Error`):** Add a Form node configured as completion to show lookup failure messages.
   - **Node 11 (`Map Venue Selection to IDs`):** Add a Code node mapping selected labels back to venue row objects.
   - **Node 12 (`Save Venues Data to Sheets`):** Add a Google Sheets node (`appendOrUpdate`, sheet `Venues`, matching columns `Venue ID`).
   - **Node 13 (`Notify Venues Saved`):** Add a Form completion node confirming save success.

5. **Build Venue Loading & Job Scheduling Branch:**
   - **Node 14 (`Read Venues Data from Sheets`):** Add a Google Sheets node reading sheet `Venues` (`alwaysOutputData: true`, `onError: continueErrorOutput`). Connect to `If Venue Form Submitted` false branch.
   - **Node 15 (`Notify Venue Read Failure`):** Add a Telegram node sending sheet read failure alerts.
   - **Node 16 (`Schedule Sampling Jobs`):** Add a Code node filtering active rows and creating sampling job items.
   - **Node 17 (`If Venue Setup Needed`):** Add an If node checking `={{ $json.setupNeeded ? 'yes' : 'no' }}` equals `yes`.
   - **Node 18 (`Notify Setup Required`):** Add a Telegram node sending setup reminder notifications.

6. **Build Sampling Loop & Execution Engine:**
   - **Node 19 (`Batch Process Venues`):** Add a Split In Batches node processing items sequentially.
   - **Node 20 (`Prepare Venue Sample Request`):** Add a Code node evaluating run abort flags and low quotas.
   - **Node 21 (`If Sample Request Skipped`):** Add an If node checking `={{ String($json.skipped === true) }}` equals `true`.
   - **Node 22 (`Wait Between API Requests`):** Add a Wait node set to seconds (`={{ Number($('Set Venue Configuration').first().json.secondsBetweenRequests) || 4 }}`).
   - **Node 23 (`Get Venue Posts Data`):** Add an HTTP Request node (GET to `https://app.recentreborn.com/api/v1/search/location`, query parameter `location_id` = `={{ $json.venueId }}`). Enable full response options and retries. Set `onError` to continue error output.
   - **Node 24 (`Calculate Posting Rate`):** Add a Code node parsing post timestamps, computing decay windows, and capturing rate-limit headers. Connect back to `Batch Process Venues`.
   - **Node 25 (`Log Search Failure`):** Add a Code node parsing API errors and setting fatal abort flags. Connect back to `Batch Process Venues`.

7. **Build Persistence & State Update Cluster:**
   - **Node 26 (`Summarize Sampling Outcome`):** Add a Code node compiling run summaries, sample rows, post rows, and venue state updates. Connect from `Batch Process Venues` completion output.
   - **Node 27 (`Divide Sample Rows`):** Add a Split Out node splitting `sampleRows`.
   - **Node 28 (`Add Samples Data to Sheets`):** Add a Google Sheets node appending to sheet `Samples`.
   - **Node 29 (`Proceed with Sample Results`):** Add a Code node restoring run summary context.
   - **Node 30 (`If New Posts Available`):** Add an If node checking `={{ $json.postRows.length > 0 ? 'yes' : 'no' }}` equals `yes`.
   - **Node 31 (`Divide Post Rows`):** Add a Split Out node splitting `postRows`.
   - **Node 32 (`Add Posts Data to Sheets`):** Add a Google Sheets node appending to sheet `Posts`.
   - **Node 33 (`Proceed with Post Results`):** Add a Code node restoring run summary context.
   - **Node 34 (`Divide Venue Rows`):** Add a Split Out node splitting `venueRows`.
   - **Node 35 (`Update Venues Data in Sheets`):** Add a Google Sheets node (`appendOrUpdate`, sheet `Venues`, matching columns `Venue ID`).
   - **Node 36 (`Proceed with Venue Updates`):** Add a Code node restoring run summary context.
   - **Node 37 (`Notify Sheet Save Failure`):** Add a Telegram node handling sheet write errors.

8. **Build Reporting, AI & Dispatch Cluster:**
   - **Node 38 (`Check Report Due`):** Add an If node checking `={{ $json.reportDue ? 'yes' : 'no' }}` equals `yes`.
   - **Node 39 (`Read Sample Log from Sheets`):** Add a Google Sheets node reading sheet `Samples`.
   - **Node 40 (`Read Post Log from Sheets`):** Add a Google Sheets node reading sheet `Posts`.
   - **Node 41 (`Notify Report Read Failure`):** Add a Telegram node handling report log read errors.
   - **Node 42 (`Compute Weekly Metrics`):** Add a Code node computing weekly comparison statistics and AI payloads.
   - **Node 43 (`Check AI Summary Enabled`):** Add an If node checking `={{ $json.sendToAi }}` equals `yes`.
   - **Node 44 (`AI-Caption Summary`):** Add a LangChain LLM Chain node (`chainLlm`) with text prompt `={{ $json.prompt }}`.
   - **Node 45 (`OpenAI Chatbot Model`):** Add an OpenAI Chat Model node (`lmChatOpenAi`, model `gpt-6-luna`, response format `json_object`), connected to `AI-Caption Summary` via its AI language model input.
   - **Node 46 (`Create Weekly Report Messages`):** Add a Code node formatting weekly report message chunks.
   - **Node 47 (`Create Alert Messages`):** Add a Code node formatting urgent alerts or test run summaries.
   - **Node 48 (`Dispatch Telegram Messages`):** Add a Telegram node broadcasting messages using HTML formatting.
   - **Node 49 (`Log Report Sent Status`):** Add a Code node updating workflow static data delivery timestamps.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Official developer templates and tutorials for the RecentReborn API | [RecentReborn Developers Portal](https://recentreborn.com/developers) |
| RecentReborn platform web application interface | [RecentReborn App](https://app.recentreborn.com) |