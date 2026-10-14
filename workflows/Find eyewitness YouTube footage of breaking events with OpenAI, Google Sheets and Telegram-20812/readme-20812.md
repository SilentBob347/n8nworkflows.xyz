Find eyewitness YouTube footage of breaking events with OpenAI, Google Sheets and Telegram

https://n8nworkflows.xyz/workflows/find-eyewitness-youtube-footage-of-breaking-events-with-openai--google-sheets-and-telegram-20812


# Find eyewitness YouTube footage of breaking events with OpenAI, Google Sheets and Telegram

### 1. Workflow Overview

This workflow is engineered for newsrooms and investigative desks to autonomously discover, filter, and score credible eyewitness YouTube footage during breaking news events. It combines multi-criteria search execution, rule-based text filtering, optional LLM-based categorization via OpenAI, centralized Google Sheets state management, and real-time Telegram alerts and digests. 

The workflow is divided into the following functional blocks:
- **1.1 Input Reception & Configuration:** Ingests events via a web form, scheduled ticker, or manual execution, merging inputs into a central configuration node.
- **1.2 Event Creation & Announcement:** Handles new event form submissions by formatting records, writing them to a Google Sheets “Events” tab, posting a confirmation to Telegram, and returning a completion response.
- **1.3 Sheet State Loading:** Reads active monitoring windows and historical footage logs from Google Sheets for routine checks, routing read failures to an alerting node.
- **1.4 Search Planning & Rate-Limiting Loop:** Plans search jobs per event and query format, processes search pagination through an external API with intentional delays, and handles API errors gracefully.
- **1.5 Video Scoring & AI Classification:** Deduplicates harvested videos against historical logs, applies heuristic relevance scoring, and optionally evaluates candidates using an OpenAI language model.
- **1.6 Logging, Updates & Telegram Digest:** Writes verified footage records back to Google Sheets, closes expired monitoring windows, and dispatches compiled Telegram summaries or operational warnings.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Configuration
- **Overview:** Acts as the primary entry point for the workflow, consolidating triggers from web forms, scheduled intervals, and manual executions into a unified configuration state.
- **Nodes Involved:** 
  - `When Event Form Submitted`
  - `When Triggered Every 15 Min`
  - `When Manually Triggered`
  - `Set Desk Configuration`
  - `If Form Submitted`
- **Node Details:**
  - **When Event Form Submitted** (`n8n-nodes-base.formTrigger`)
    - *Role:* Captures query name, multi-line search queries, and monitoring duration via a web form.
    - *Configuration:* Path set to `/eyewitness-event-window`, captures custom form fields.
    - *Inputs/Outputs:* Input: Webhook; Output: `Set Desk Configuration`.
    - *Edge Cases:* Missing or empty form submissions; handled downstream by validation checks.
  - **When Triggered Every 15 Min** (`n8n-nodes-base.scheduleTrigger`)
    - *Role:* Periodically triggers automated monitoring cycles every 15 minutes.
    - *Configuration:* Interval set to 15 minutes.
    - *Inputs/Outputs:* Input: Cron timer; Output: `Set Desk Configuration`.
  - **When Manually Triggered** (`n8n-nodes-base.manualTrigger`)
    - *Role:* Allows manual execution for testing or forced checking.
    - *Configuration:* Default execution settings.
    - *Inputs/Outputs:* Input: Manual action; Output: `Set Desk Configuration`.
  - **Set Desk Configuration** (`n8n-nodes-base.set`)
    - *Role:* Centralizes global operating parameters (sheet URLs, chat IDs, scoring thresholds, query limits, and AI rubrics).
    - *Configuration:* Assigns variables such as `sheetUrl`, `telegramChatId`, `formats`, `defaultWindowHours`, `useAi`, and keyword lists.
    - *Inputs/Outputs:* Input: Any trigger node; Output: `If Form Submitted`.
  - **If Form Submitted** (`n8n-nodes-base.if`)
    - *Role:* Branches execution based on whether the run originated from a form submission or a scheduled/manual poll.
    - *Configuration:* Evaluates if `Search query` exists.
    - *Inputs/Outputs:* Input: `Set Desk Configuration`; Outputs: True branch to `Create Event Sheet Row`, False branch to `Fetch Events From Sheet`.

#### 1.2 Event Creation & Announcement
- **Overview:** Converts incoming form details into an active monitoring window, persists it in Google Sheets, announces it via Telegram, and displays a browser completion notice.
- **Nodes Involved:**
  - `Create Event Sheet Row`
  - `Append Event to Events Tab`
  - `Announce Event via Telegram`
  - `Display Monitoring Started Form`
  - `Display Save Failure Notice`
- **Node Details:**
  - **Create Event Sheet Row** (`n8n-nodes-base.code`)
    - *Role:* Transforms form variables into a structured database row schema with computed expiration windows.
    - *Configuration:* JavaScript execution sanitizing queries and enforcing `maxWindowHours`.
    - *Inputs/Outputs:* Input: `If Form Submitted` (True); Output: `Append Event to Events Tab`.
  - **Append Event to Events Tab** (`n8n-nodes-base.googleSheets`)
    - *Role:* Appends the new event record into the “Events” worksheet.
    - *Configuration:* Operation set to `append`, raw cell formatting, dynamic document ID from config.
    - *Inputs/Outputs:* Input: `Create Event Sheet Row`; Outputs: Success to `Announce Event via Telegram`, Error to `Display Save Failure Notice`.
    - *Edge Cases:* Google Sheets authentication failure or missing tab name; routed to error notification.
  - **Announce Event via Telegram** (`n8n-nodes-base.telegram`)
    - *Role:* Sends a notification to the desk chat indicating that a new event window has opened.
    - *Configuration:* HTML parse mode, target chat ID from config.
    - *Inputs/Outputs:* Input: `Append Event to Events Tab`; Output: `Display Monitoring Started Form`.
  - **Display Monitoring Started Form** (`n8n-nodes-base.form`)
    - *Role:* Renders a success completion screen to the form submitter.
    - *Configuration:* Completion operation with dynamic summary text.
    - *Inputs/Outputs:* Input: `Announce Event via Telegram`; Output: None (Terminal node).
  - **Display Save Failure Notice** (`n8n-nodes-base.form`)
    - *Role:* Renders an error completion screen if the sheet append operation fails.
    - *Configuration:* Completion operation with troubleshooting instructions.
    - *Inputs/Outputs:* Input: `Append Event to Events Tab` (Error); Output: None (Terminal node).

#### 1.3 Sheet State Loading
- **Overview:** Reads active event records and historical footage logs from Google Sheets during polling runs, capturing read errors to alert the desk.
- **Nodes Involved:**
  - `Fetch Events From Sheet`
  - `Fetch Footage Log From Sheet`
  - `Notify Read Failure via Telegram`
- **Node Details:**
  - **Fetch Events From Sheet** (`n8n-nodes-base.googleSheets`)
    - *Role:* Retrieves all rows from the “Events” tab.
    - *Configuration:* Operation set to `get`, always output data enabled.
    - *Inputs/Outputs:* Input: `If Form Submitted` (False); Outputs: Success to `Fetch Footage Log From Sheet`, Error to `Notify Read Failure via Telegram`.
  - **Fetch Footage Log From Sheet** (`n8n-nodes-base.googleSheets`)
    - *Role:* Retrieves existing video records from the “Footage Log” tab for cross-run deduplication.
    - *Configuration:* Operation set to `get`, executed once, always output data enabled.
    - *Inputs/Outputs:* Input: `Fetch Events From Sheet`; Outputs: Success to `Plan Search and Closing Tasks`, Error to `Notify Read Failure via Telegram`.
  - **Notify Read Failure via Telegram** (`n8n-nodes-base.telegram`)
    - *Role:* Alerts the Telegram desk channel if reading Google Sheets fails.
    - *Configuration:* HTML parse mode, structured error message.
    - *Inputs/Outputs:* Input: Error outputs from sheet fetch nodes; Output: None (Terminal node).

#### 1.4 Search Planning & Rate-Limiting Loop
- **Overview:** Plans granular search jobs based on active queries and formats, pages through the external YouTube search API with pauses, and handles request failures.
- **Nodes Involved:**
  - `Plan Search and Closing Tasks`
  - `Process Search Jobs`
  - `Prepare Search Page Request`
  - `If Search Skipped`
  - `Wait 4 Seconds`
  - `Fetch YouTube Videos`
  - `Analyze Page Results`
  - `Handle Search Failure`
  - `If Fetch Next Page`
- **Node Details:**
  - **Plan Search and Closing Tasks** (`n8n-nodes-base.code`)
    - *Role:* Identifies open events, marks expired windows for closure, and generates an array of search jobs bounded by credit limits.
    - *Configuration:* Complex JavaScript evaluation storing state in workflow static data.
    - *Inputs/Outputs:* Input: `Fetch Footage Log From Sheet`; Output: `Process Search Jobs`.
  - **Process Search Jobs** (`n8n-nodes-base.splitInBatches`)
    - *Role:* Iterates through the generated search jobs array.
    - *Configuration:* Standard batch processing loop.
    - *Inputs/Outputs:* Inputs: `Plan Search and Closing Tasks`, `If Search Skipped` (False), `Handle Search Failure`; Outputs: Loop items to `Prepare Search Page Request`, Finished batch to `Assess Video Relevance`.
  - **Prepare Search Page Request** (`n8n-nodes-base.code`)
    - *Role:* Prepares pagination payloads and evaluates quota/abort flags before executing external API calls.
    - *Configuration:* JavaScript evaluation of daily remaining quota.
    - *Inputs/Outputs:* Input: `Process Search Jobs`; Output: `If Search Skipped`.
  - **If Search Skipped** (`n8n-nodes-base.if`)
    - *Role:* Bypasses API execution if quotas are exhausted or errors occurred.
    - *Configuration:* Checks `skipped === true`.
    - *Inputs/Outputs:* Input: `Prepare Search Page Request`; Outputs: True branch loops back to `Process Search Jobs`, False branch to `Wait 4 Seconds`.
  - **Wait 4 Seconds** (`n8n-nodes-base.wait`)
    - *Role:* Introduces a rate-limiting pause between API requests.
    - *Configuration:* Unit set to seconds, amount set to 4.
    - *Inputs/Outputs:* Input: `If Search Skipped` (False); Output: `Fetch YouTube Videos`.
  - **Fetch YouTube Videos** (`n8n-nodes-base.httpRequest`)
    - *Role:* Queries the RecentReborn API for recent YouTube video listings matching search criteria.
    - *Configuration:* HTTP POST to `https://app.recentreborn.com/api/v1/search/query`, Generic Header Auth, 30s timeout, retry on fail.
    - *Inputs/Outputs:* Input: `Wait 4 Seconds`; Outputs: Success to `Analyze Page Results`, Error to `Handle Search Failure`.
    - *Edge Cases:* HTTP 400, 401, 403, 429, or 500 errors; caught by error output.
  - **Analyze Page Results** (`n8n-nodes-base.code`)
    - *Role:* Parses search API responses, extracts video metadata, tracks rate-limit headers, and determines if further pagination is needed.
    - *Configuration:* JavaScript handling pagination tokens and overlap detection.
    - *Inputs/Outputs:* Input: `Fetch YouTube Videos` (Success); Output: `If Fetch Next Page`.
  - **Handle Search Failure** (`n8n-nodes-base.code`)
    - *Role:* Converts failed HTTP requests into loop-safe markers and flags fatal status codes to abort subsequent searches.
    - *Configuration:* Error parsing and HTTP status categorization.
    - *Inputs/Outputs:* Input: `Fetch YouTube Videos` (Error); Output: `Process Search Jobs`.
  - **If Fetch Next Page** (`n8n-nodes-base.if`)
    - *Role:* Decides whether to loop back for another page of results.
    - *Configuration:* Checks `more === true`.
    - *Inputs/Outputs:* Input: `Analyze Page Results`; Outputs: True loops to `Prepare Search Page Request`, False proceeds to `Process Search Jobs`.

#### 1.5 Video Scoring & AI Classification
- **Overview:** Merges search results, removes duplicates against historical logs, scores videos using keyword heuristics, and optionally submits top candidates to OpenAI for semantic classification.
- **Nodes Involved:**
  - `Assess Video Relevance`
  - `If AI Classification Needed`
  - `AI Classify Video Titles`
  - `OpenAI GPT-6 Model`
  - `Generate AI Labels and Rows`
- **Node Details:**
  - **Assess Video Relevance** (`n8n-nodes-base.code`)
    - *Role:* Evaluates harvested videos against footage, news, and commentary lexicons, performing duplicate and copy detection.
    - *Configuration:* Extensive heuristic text-matching and Jaccard similarity scoring.
    - *Inputs/Outputs:* Input: `Process Search Jobs` (Finished); Output: `If AI Classification Needed`.
  - **If AI Classification Needed** (`n8n-nodes-base.if`)
    - *Role:* Checks whether AI enrichment is enabled and if candidate videos require LLM review.
    - *Configuration:* Checks `sendToAi === 'yes'`.
    - *Inputs/Outputs:* Input: `Assess Video Relevance`; Outputs: True branch to `AI Classify Video Titles`, False branch to `Generate AI Labels and Rows`.
  - **AI Classify Video Titles** (`@n8n/n8n-nodes-langchain.chainLlm`)
    - *Role:* Submits batched candidate video metadata to OpenAI for categorical classification.
    - *Configuration:* Text prompt type, linked to chat model.
    - *Inputs/Outputs:* Input: `If AI Classification Needed` (True); Output: `Generate AI Labels and Rows`.
  - **OpenAI GPT-6 Model** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
    - *Role:* Provides the underlying language model configuration for title classification.
    - *Configuration:* Model set to `gpt-6-luna`, JSON object response format.
    - *Inputs/Outputs:* Input/Output: Connected via AI model hook to `AI Classify Video Titles`.
    - *Credentials:* OpenAI API credential.
  - **Generate AI Labels and Rows** (`n8n-nodes-base.code`)
    - *Role:* Integrates AI classification results (or falls back to rules), builds worksheet insertion rows, and compiles run summaries.
    - *Configuration:* Validates JSON output structure from LLM and structures object rows.
    - *Inputs/Outputs:* Inputs: `AI Classify Video Titles`, `If AI Classification Needed` (False); Output: `If New Videos Exist`.

#### 1.6 Logging, Updates & Telegram Digest
- **Overview:** Persists new footage records and event updates to Google Sheets, handles write failures, and compiles and transmits Telegram digests and wrap-ups.
- **Nodes Involved:**
  - `If New Videos Exist`
  - `Distribute Footage Rows`
  - `Add to Footage Log Sheet`
  - `Resume with Run Summary`
  - `If Updates Needed for Events`
  - `Divide Event Updates`
  - `Update Events Sheet`
  - `Notify Write Failure via Telegram`
  - `Compile Telegram Digest`
  - `Transmit Telegram Digest`
- **Node Details:**
  - **If New Videos Exist** (`n8n-nodes-base.if`)
    - *Role:* Determines whether new video rows need to be appended to the sheet.
    - *Configuration:* Checks `footageRows.length > 0`.
    - *Inputs/Outputs:* Input: `Generate AI Labels and Rows`; Outputs: True to `Distribute Footage Rows`, False to `Resume with Run Summary`.
  - **Distribute Footage Rows** (`n8n-nodes-base.splitOut`)
    - *Role:* Splits the array of new video rows into individual items for batch sheet insertion.
    - *Configuration:* Field to split out set to `footageRows`.
    - *Inputs/Outputs:* Input: `If New Videos Exist` (True); Output: `Add to Footage Log Sheet`.
  - **Add to Footage Log Sheet** (`n8n-nodes-base.googleSheets`)
    - *Role:* Appends new footage records to the “Footage Log” sheet tab.
    - *Configuration:* Operation set to `append`, raw cell formatting.
    - *Inputs/Outputs:* Input: `Distribute Footage Rows`; Outputs: Success to `Resume with Run Summary`, Error to `Notify Write Failure via Telegram`.
  - **Resume with Run Summary** (`n8n-nodes-base.code`)
    - *Role:* Re-aggregates individual sheet output items back into a single run summary object.
    - *Configuration:* JavaScript returning initial summary payload.
    - *Inputs/Outputs:* Inputs: `Add to Footage Log Sheet`, `If New Videos Exist` (False); Output: `If Updates Needed for Events`.
  - **If Updates Needed for Events** (`n8n-nodes-base.if`)
    - *Role:* Checks if event records require status, counter, or window updates.
    - *Configuration:* Checks `eventRows.length > 0`.
    - *Inputs/Outputs:* Input: `Resume with Run Summary`; Outputs: True to `Divide Event Updates`, False to `Compile Telegram Digest`.
  - **Divide Event Updates** (`n8n-nodes-base.splitOut`)
    - *Role:* Splits event update rows for individual sheet processing.
    - *Configuration:* Field to split out set to `eventRows`.
    - *Inputs/Outputs:* Input: `If Updates Needed for Events` (True); Output: `Update Events Sheet`.
  - **Update Events Sheet** (`n8n-nodes-base.googleSheets`)
    - *Role:* Updates event statuses and counters in the “Events” tab.
    - *Configuration:* Operation set to `appendOrUpdate`, matching on column `Event ID`.
    - *Inputs/Outputs:* Input: `Divide Event Updates`; Outputs: Success to `Compile Telegram Digest`, Error to `Notify Write Failure via Telegram`.
  - **Notify Write Failure via Telegram** (`n8n-nodes-base.telegram`)
    - *Role:* Alerts the Telegram desk channel if writing to Google Sheets fails.
    - *Configuration:* HTML parse mode, contextual error message.
    - *Inputs/Outputs:* Input: Error outputs from sheet write nodes; Output: None (Terminal node).
  - **Compile Telegram Digest** (`n8n-nodes-base.code`)
    - *Role:* Formats candidates, event wrap-ups, and operational problems into an HTML-formatted Telegram digest, splitting long messages if necessary.
    - *Configuration:* JavaScript formatting with character length enforcement (max 3500 chars per message).
    - *Inputs/Outputs:* Inputs: `Update Events Sheet`, `If Updates Needed for Events` (False); Output: `Transmit Telegram Digest`.
  - **Transmit Telegram Digest** (`n8n-nodes-base.telegram`)
    - *Role:* Sends the compiled digest messages to the target Telegram chat.
    - *Configuration:* HTML parse mode, retry on fail, 3s wait between retries.
    - *Inputs/Outputs:* Input: `Compile Telegram Digest`; Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation & Setup Guide | None | None | ## Find eyewitness YouTube footage of breaking events with OpenAI, Google Sheets and Telegram... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Section divider | None | None | ## Start and route run... |
| Sticky Note2 | n8n-nodes-base.stickyNote | Section divider | None | None | ## Create event record... |
| Sticky Note3 | n8n-nodes-base.stickyNote | Section divider | None | None | ## Load sheet state... |
| Sticky Note4 | n8n-nodes-base.stickyNote | Section divider | None | None | ## Plan search jobs... |
| Sticky Note5 | n8n-nodes-base.stickyNote | Section divider | None | None | ## Prepare page request... |
| Sticky Note6 | n8n-nodes-base.stickyNote | Section divider | None | None | ## Call YouTube search... |
| Sticky Note7 | n8n-nodes-base.stickyNote | Section divider | None | None | ## Handle search pages... |
| Sticky Note8 | n8n-nodes-base.stickyNote | Section divider | None | None | ## Score candidate videos... |
| Sticky Note9 | n8n-nodes-base.stickyNote | Section divider | None | None | ## Classify titles with AI... |
| Sticky Note10 | n8n-nodes-base.stickyNote | Section divider | None | None | ## Build footage rows... |
| Sticky Note11 | n8n-nodes-base.stickyNote | Section divider | None | None | ## Write footage log... |
| Sticky Note12 | n8n-nodes-base.stickyNote | Section divider | None | None | ## Update event rows... |
| Sticky Note13 | n8n-nodes-base.stickyNote | Section divider | None | None | ## Send final notifications... |
| When Event Form Submitted | n8n-nodes-base.formTrigger | Ingests event search queries | Webhook | Set Desk Configuration | |
| When Triggered Every 15 Min | n8n-nodes-base.scheduleTrigger | Triggers automated polling | Cron | Set Desk Configuration | |
| When Manually Triggered | n8n-nodes-base.manualTrigger | Triggers manual test runs | Manual | Set Desk Configuration | |
| Set Desk Configuration | n8n-nodes-base.set | Sets global operational variables | Triggers | If Form Submitted | Every value you may want to change. See the "Config values" sticky note. |
| If Form Submitted | n8n-nodes-base.if | Routes form vs polling runs | Set Desk Configuration | Create Event Sheet Row, Fetch Events From Sheet | |
| Create Event Sheet Row | n8n-nodes-base.code | Formats form data into event row | If Form Submitted | Append Event to Events Tab | |
| Append Event to Events Tab | n8n-nodes-base.googleSheets | Writes new event to sheet | Create Event Sheet Row | Announce Event via Telegram, Display Save Failure Notice | |
| Announce Event via Telegram | n8n-nodes-base.telegram | Announces new event in chat | Append Event to Events Tab | Display Monitoring Started Form | |
| Display Monitoring Started Form | n8n-nodes-base.form | Shows success screen | Announce Event via Telegram | None | |
| Display Save Failure Notice | n8n-nodes-base.form | Shows error screen | Append Event to Events Tab | None | |
| Fetch Events From Sheet | n8n-nodes-base.googleSheets | Reads events database | If Form Submitted | Fetch Footage Log From Sheet, Notify Read Failure via Telegram | |
| Fetch Footage Log From Sheet | n8n-nodes-base.googleSheets | Reads historical footage log | Fetch Events From Sheet | Plan Search and Closing Tasks, Notify Read Failure via Telegram | |
| Notify Read Failure via Telegram | n8n-nodes-base.telegram | Alerts read errors | Sheet fetch nodes | None | |
| Plan Search and Closing Tasks | n8n-nodes-base.code | Plans search queue & event windows | Fetch Footage Log From Sheet | Process Search Jobs | |
| Process Search Jobs | n8n-nodes-base.splitInBatches | Iterates through search jobs | Plan Search and Closing Tasks, If Search Skipped, Handle Search Failure | Prepare Search Page Request, Assess Video Relevance | |
| Prepare Search Page Request | n8n-nodes-base.code | Prepares pagination & quota checks | Process Search Jobs | If Search Skipped | |
| If Search Skipped | n8n-nodes-base.if | Bypasses skipped searches | Prepare Search Page Request | Process Search Jobs, Wait 4 Seconds | |
| Wait 4 Seconds | n8n-nodes-base.wait | Rate-limiting delay | If Search Skipped | Fetch YouTube Videos | |
| Fetch YouTube Videos | n8n-nodes-base.httpRequest | Queries YouTube search API | Wait 4 Seconds | Analyze Page Results, Handle Search Failure | upload_date is "hour" on normal runs. Pages are capped by maxPagesPerSearch and maxCreditsPerRun. |
| Analyze Page Results | n8n-nodes-base.code | Parses search results & pagination | Fetch YouTube Videos | If Fetch Next Page | |
| Handle Search Failure | n8n-nodes-base.code | Handles API error responses | Fetch YouTube Videos | Process Search Jobs | |
| If Fetch Next Page | n8n-nodes-base.if | Evaluates pagination continuity | Analyze Page Results | Prepare Search Page Request, Process Search Jobs | |
| Assess Video Relevance | n8n-nodes-base.code | Heuristic scoring & deduplication | Process Search Jobs | If AI Classification Needed | |
| If AI Classification Needed | n8n-nodes-base.if | Checks if LLM review is required | Assess Video Relevance | AI Classify Video Titles, Generate AI Labels and Rows | |
| AI Classify Video Titles | @n8n/n8n-nodes-langchain.chainLlm | Batched title classification via LLM | If AI Classification Needed | Generate AI Labels and Rows | One batched prompt per run. The rubric lives in "aiRubric" in the config node. |
| OpenAI GPT-6 Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides LLM backend | None | AI Classify Video Titles | |
| Generate AI Labels and Rows | n8n-nodes-base.code | Finalizes labels & builds sheet rows | AI Classify Video Titles, If AI Classification Needed | If New Videos Exist | |
| If New Videos Exist | n8n-nodes-base.if | Checks for new footage to log | Generate AI Labels and Rows | Distribute Footage Rows, Resume with Run Summary | |
| Distribute Footage Rows | n8n-nodes-base.splitOut | Splits footage rows for batch write | If New Videos Exist | Add to Footage Log Sheet | |
| Add to Footage Log Sheet | n8n-nodes-base.googleSheets | Appends footage to sheet log | Distribute Footage Rows | Resume with Run Summary, Notify Write Failure via Telegram | |
| Resume with Run Summary | n8n-nodes-base.code | Re-aggregates run summary object | Add to Footage Log Sheet, If New Videos Exist | If Updates Needed for Events | |
| If Updates Needed for Events | n8n-nodes-base.if | Checks for event status updates | Resume with Run Summary | Divide Event Updates, Compile Telegram Digest | |
| Divide Event Updates | n8n-nodes-base.splitOut | Splits event updates for batch write | If Updates Needed for Events | Update Events Sheet | |
| Update Events Sheet | n8n-nodes-base.googleSheets | Updates event rows in sheet | Divide Event Updates | Compile Telegram Digest, Notify Write Failure via Telegram | |
| Notify Write Failure via Telegram | n8n-nodes-base.telegram | Alerts sheet write errors | Sheet write nodes | None | |
| Compile Telegram Digest | n8n-nodes-base.code | Formats Telegram digest message | Update Events Sheet, If Updates Needed for Events | Transmit Telegram Digest | |
| Transmit Telegram Digest | n8n-nodes-base.telegram | Sends digest messages to chat | Compile Telegram Digest | None | |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create Entry Triggers:**
   - Add a **Form Trigger** (`n8n-nodes-base.formTrigger`) named `When Event Form Submitted`. Set path to `eyewitness-event-window`, configure form fields for `Query name` (string), `Search query` (textarea), and `Hours to monitor` (number).
   - Add a **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`) named `When Triggered Every 15 Min` set to run every 15 minutes.
   - Add a **Manual Trigger** (`n8n-nodes-base.manualTrigger`) named `When Manually Triggered`.

2. **Configure Central Settings:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Set Desk Configuration`. Connect all three triggers to it.
   - Assign assignments for `sheetUrl`, `telegramChatId`, `formats` (`video, shorts`), `defaultWindowHours` (6), `maxWindowHours` (24), `maxQueriesPerEvent` (4), `maxPagesPerSearch` (2), `maxCreditsPerRun` (12), `minQuotaRemaining` (50), `firstRunUploadDate` (`hour`), lexicons (`footageTerms`, `newsTerms`, `commentaryTerms`, `newsChannels`), `useAi` (true), `maxVideosForAi` (40), `aiRubric`, `minQueueScore` (60), `copyTitleSimilarity` (0.8), and `maxCandidatesPerEvent` (8).

3. **Build Form Submission Branch:**
   - Add an **If** node (`n8n-nodes-base.if`) named `If Form Submitted` to check if `Search query` exists.
   - **True branch:**
     - Add a **Code** node (`n8n-nodes-base.code`) named `Create Event Sheet Row` to parse queries and calculate expiration timestamps.
     - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Append Event to Events Tab` (operation: `append`, sheet name: `Events`). Configure `continueErrorOutput` error handling.
     - Add a **Telegram** node (`n8n-nodes-base.telegram`) named `Announce Event via Telegram` (HTML parse mode).
     - Add a **Form** node (`n8n-nodes-base.form`) named `Display Monitoring Started Form` (operation: `completion`).
     - On sheet write error, route to a **Form** node named `Display Save Failure Notice`.

4. **Build Polling & State Loading Branch:**
   - **False branch of `If Form Submitted`:**
     - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Fetch Events From Sheet` (operation: `get`, sheet name: `Events`). Enable `continueErrorOutput`.
     - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Fetch Footage Log From Sheet` (operation: `get`, sheet name: `Footage Log`, executed once). Enable `continueErrorOutput`.
     - Route sheet read errors to a **Telegram** node named `Notify Read Failure via Telegram`.

5. **Build Search Planning & Batch Loop:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Plan Search and Closing Tasks` after the footage log fetch.
   - Add a **Split In Batches** node (`n8n-nodes-base.splitInBatches`) named `Process Search Jobs`.
   - Add a **Code** node (`n8n-nodes-base.code`) named `Prepare Search Page Request`.
   - Add an **If** node (`n8n-nodes-base.if`) named `If Search Skipped`.
   - Add a **Wait** node (`n8n-nodes-base.wait`) named `Wait 4 Seconds` (amount: 4 seconds).
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `Fetch YouTube Videos` (POST to `https://app.recentreborn.com/api/v1/search/query`, Header Auth, timeout 30000ms, retry on fail). Enable `continueErrorOutput`.
   - Add a **Code** node (`n8n-nodes-base.code`) named `Analyze Page Results` for successful responses.
   - Add a **Code** node (`n8n-nodes-base.code`) named `Handle Search Failure` for failed requests, looping back to `Process Search Jobs`.
   - Add an **If** node (`n8n-nodes-base.if`) named `If Fetch Next Page` to loop pagination back to `Prepare Search Page Request`.

6. **Build Scoring & AI Classification Block:**
   - After the search batch loop completes, add a **Code** node (`n8n-nodes-base.code`) named `Assess Video Relevance` for heuristic scoring and deduplication.
   - Add an **If** node (`n8n-nodes-base.if`) named `If AI Classification Needed`.
   - **True branch:**
     - Add a **LangChain LLM Chain** node (`@n8n/n8n-nodes-langchain.chainLlm`) named `AI Classify Video Titles`.
     - Connect an **OpenAI Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) configured with model `gpt-6-luna` (JSON object response format) and OpenAI API credentials.
   - Add a **Code** node (`n8n-nodes-base.code`) named `Generate AI Labels and Rows` to process classifications and construct output rows.

7. **Build Logging, Updates & Digest Block:**
   - Add an **If** node (`n8n-nodes-base.if`) named `If New Videos Exist`.
   - **True branch:** Add a **Split Out** node (`n8n-nodes-base.splitOut`) named `Distribute Footage Rows`, followed by a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Add to Footage Log Sheet` (operation: `append`, sheet name: `Footage Log`). Enable `continueErrorOutput` and route errors to `Notify Write Failure via Telegram`.
   - Add a **Code** node (`n8n-nodes-base.code`) named `Resume with Run Summary`.
   - Add an **If** node (`n8n-nodes-base.if`) named `If Updates Needed for Events`.
   - **True branch:** Add a **Split Out** node named `Divide Event Updates`, followed by a **Google Sheets** node named `Update Events Sheet` (operation: `appendOrUpdate`, matching columns: `Event ID`, sheet name: `Events`). Enable `continueErrorOutput` and route errors to `Notify Write Failure via Telegram`.
   - Add a **Code** node (`n8n-nodes-base.code`) named `Compile Telegram Digest`.
   - Add a **Telegram** node (`n8n-nodes-base.telegram`) named `Transmit Telegram Digest` (HTML parse mode, retry on fail).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Find eyewitness YouTube footage of breaking events with OpenAI, Google Sheets and Telegram | Workflow title and functional scope |
| Visit https://recentreborn.com/developers for more tutorials and templates | External developer resources and API documentation |