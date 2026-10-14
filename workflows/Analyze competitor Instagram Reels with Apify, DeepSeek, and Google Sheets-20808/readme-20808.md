Analyze competitor Instagram Reels with Apify, DeepSeek, and Google Sheets

https://n8nworkflows.xyz/workflows/analyze-competitor-instagram-reels-with-apify--deepseek--and-google-sheets-20808


# Analyze competitor Instagram Reels with Apify, DeepSeek, and Google Sheets

### 1. Workflow Overview

This workflow is a daily automated intelligence pipeline designed to monitor competitors' Instagram Reels, transcribe their audio, extract structured content-strategy insights using an OpenAI-compatible LLM (DeepSeek), and store the final output in Google Sheets. It includes built-in deduplication, retry mechanisms, and a centralized registry managed via an n8n Data Table.

The system logic is organized into the following functional blocks:
- **1.1 Initialization and Input Reception:** Handles the scheduled execution trigger and ensures the n8n Data Table registry exists before pulling competitor profiles from Google Sheets.
- **1.2 Competitor Filtering and Batching:** Cleans and filters the competitor list for active accounts, then loops through them sequentially.
- **1.3 Reel Discovery and Normalization:** Fetches recent Reels via Apify, normalizes the data structure, filters out non-video content, and constructs unique canonical URLs and keys.
- **1.4 Deduplication and Queue Management:** Evaluates the processed Reel queue against the n8n Data Table registry to filter out already-processed items and manage retry attempts.
- **1.5 Transcription and AI Analysis:** Extracts audio transcripts using an Apify actor, validates transcript usability, and routes content through a LangChain LLM chain powered by DeepSeek to generate structured JSON briefs.
- **1.6 Data Persistence and Reporting:** Saves processed insights to Google Sheets, updates the n8n registry status (`success` or `failed`), compiles a run digest, and optionally dispatches a Telegram summary.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Input Reception
This block triggers the daily pipeline and provisions the deduplication data table before loading competitor credentials from Google Sheets.

- **01 — When Competitors Should Be Checked**
  - **Type & Role:** `n8n-nodes-base.scheduleTrigger` (Trigger). Initiates the workflow daily at 09:00.
  - **Configuration:** Rule set to trigger every 1 day at 09:00 UTC.
  - **Expressions:** None.
  - **Connections:** Output connects to Node `01b`.
  - **Potential Failures:** None.

- **01b — Create Registry Table**
  - **Type & Role:** `n8n-nodes-base.dataTable` (Database Operation). Creates the deduplication registry table if it does not already exist.
  - **Configuration:** Resource: `table`, Operation: `create`, Table Name: `competitor_reel_registry`, Option `createIfNotExists` enabled. Schema defines columns: `reel_key`, `instagram_username`, `reel_url`, `status`, `attempts`, `error_code`, `processed_at`, `last_attempt_at`.
  - **Expressions:** None.
  - **Connections:** Input from Node `01`, output connects to Node `02`.
  - **Potential Failures:** Database permission errors.

- **02 — Load Competitors**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Data Input). Fetches the entire list of competitors from the designated Google Sheets document.
  - **Configuration:** Resource: `sheet`, Operation: `get`, Sheet Name: `Competitors`. Document ID is left unconfigured for template usage.
  - **Expressions:** None.
  - **Connections:** Input from Node `01b`, output connects to Node `02b — Filter Active Competitors`.
  - **Potential Failures:** Google Sheets API authentication failure, missing sheet tabs, or invalid document ID.

---

#### 2.2 Competitor Filtering and Batching
This block parses raw spreadsheet rows, filters for active competitor handles, and iterates through them one by one.

- **02b — Filter Active Competitors**
  - **Type & Role:** `n8n-nodes-base.code` (Data Transformation). Cleans input rows, drops empty usernames, handles diverse truthy variations for the `active` column, and normalizes check limits.
  - **Configuration:** Mode: `runOnceForAllItems`.
  - **Expressions/JS Code:** Custom JavaScript evaluating truthy values (`true`, `TRUE`, `yes`, `y`, `1`, `active`) and stripping leading `@` characters from usernames.
  - **Connections:** Input from Node `02`, output connects to Node `03`.
  - **Potential Failures:** Syntax issues if unexpected data types populate spreadsheet cells.

- **03 — Loop Over Competitors**
  - **Type & Role:** `n8n-nodes-base.splitInBatches` (Flow Control). Iterates through the active competitor array sequentially.
  - **Configuration:** Batch Size: `1`, Reset: `false`.
  - **Expressions:** None.
  - **Connections:** Input from Node `02b`. Output loops to Node `04` (when items remain) or Node `06` (when complete).
  - **Potential Failures:** Infinite loop scenarios if batch pointer logic is manually altered.

---

#### 2.3 Reel Discovery and Normalization
This block interacts with Apify to fetch recent Reels for the active competitor and converts them into a clean, normalized queue.

- **04 — Fetch Recent Reels**
  - **Type & Role:** `@apify/n8n-nodes-apify.apify` (External API Integration). Runs an Apify actor to retrieve Instagram Reels.
  - **Configuration:** Resource: `Actors`, Operation: `Run actor and get dataset`, Actor ID: `y3uzYAhSuIK2ad8hE`, Memory: 1024MB, Timeout: 300s, Max Cost: $0.50.
  - **Expressions:** Custom JSON body constructing profile URLs and setting content type to `reels` restricted by `max_reels_per_check`.
  - **Connections:** Input from Node `03`, output connects to Node `05 — Normalize Reel Data`.
  - **Potential Failures:** Apify API rate limits, invalid actor IDs, insufficient account balance, or target profile blocks.

- **05 — Normalize Reel Data**
  - **Type & Role:** `n8n-nodes-base.code` (Data Transformation). Filters out non-video items, constructs canonical URLs, and sorts Reels by publication date.
  - **Configuration:** Mode: `runOnceForAllItems`.
  - **Expressions/JS Code:** JavaScript filtering for `type === 'video'`, assembling `reel_key` as `username:shortcode`, and slicing output to match `max_reels_per_check`.
  - **Connections:** Input from Node `04`, output connects to Node `06 — Prepare Reel Queue`.
  - **Potential Failures:** Missing expected schema fields from the Apify dataset.

- **06 — Prepare Reel Queue**
  - **Type & Role:** `n8n-nodes-base.code` (Data Transformation). Performs cross-competitor deduplication to prevent duplicate processing of collaborative posts.
  - **Configuration:** Mode: `runOnceForAllItems`.
  - **Expressions/JS Code:** JavaScript utilizing a `Set` to filter out duplicate `reel_key` values across competing accounts.
  - **Connections:** Input from Node `05` (and loop completion from Node `03`), output connects to Node `07`.
  - **Potential Failures:** Memory overflow if queue sizes exceed runtime limits (mitigated by batch caps).

---

#### 2.4 Deduplication and Queue Management
This block loops through the prepared Reel queue and evaluates whether each item is new, requires a retry, or should be skipped entirely.

- **07 — Loop Over Reels**
  - **Type & Role:** `n8n-nodes-base.splitInBatches` (Flow Control). Iterates through the normalized Reel queue one item at a time.
  - **Configuration:** Batch Size: `1`, Reset: `false`.
  - **Expressions:** None.
  - **Connections:** Input from Node `06` (or downstream error/success loops). Outputs to Node `08` (to check status) and Node `17` (when queue empties).
  - **Potential Failures:** Premature loop termination if batch pointers are disrupted.

- **08 — Already Processed?**
  - **Type & Role:** `n8n-nodes-base.dataTable` (Database Operation). Queries the deduplication table for existing records matching the current `reel_key`.
  - **Configuration:** Resource: `row`, Operation: `get`, Table: `competitor_reel_registry`, Filter condition: `reel_key` equals current item `reel_key`. `alwaysOutputData` enabled.
  - **Expressions:** `={{ $json.reel_key }}`.
  - **Connections:** Input from Node `07`, output connects to Node `09 — Is New Or Retryable?`.
  - **Potential Failures:** Data table query timeouts or table corruption.

- **09 — Is New Or Retryable?**
  - **Type & Role:** `n8n-nodes-base.code` (Logic Evaluation). Determines whether a Reel is `new`, eligible for a `retry` (attempts < 3), `skip` (already successful), or `dead` (exceeded retry limits).
  - **Configuration:** Mode: `runOnceForAllItems`.
  - **Expressions/JS Code:** JavaScript evaluating registry record status and attempt counts against a maximum threshold of 3.
  - **Connections:** Input from Node `08`, output connects to Node `09b`.
  - **Potential Failures:** Null pointer exceptions if data table schema returns unexpected structures.

- **09b — Should Process?**
  - **Type & Role:** `n8n-nodes-base.if` (Flow Control). Routes execution based on the decision flag.
  - **Configuration:** Condition: `_decision` does not equal `skip`.
  - **Expressions:** `={{ $json._decision }}`.
  - **Connections:** Input from Node `09`. True branch connects to Node `10`; False branch loops back to Node `07`.
  - **Potential Failures:** Routing misconfigurations leading to infinite loops.

---

#### 2.5 Transcription and AI Analysis
This block fetches audio transcripts via Apify, validates their content, and sends them to DeepSeek via LangChain for structured content analysis.

- **10 — Run Instagram Transcript Actor**
  - **Type & Role:** `@apify/n8n-nodes-apify.apify` (External API Integration). Executes an Apify actor to extract text transcripts from specific Reel URLs.
  - **Configuration:** Resource: `Actors`, Operation: `Run actor and get dataset`, Actor ID: `ZuAmrsZhEP0VQ7E9o`, Memory: 512MB, Timeout: 600s, Max Cost: $0.50. Error handling configured to continue on error output.
  - **Expressions:** Custom JSON body containing target `urls`, `formats` set to `text`, and `enhancedMode` enabled.
  - **Connections:** Input from Node `09b`. Success output connects to Node `11`; Error output connects to Node `16`.
  - **Potential Failures:** Actor execution timeouts, unavailable media, or insufficient API credits.

- **11 — Validate Transcript**
  - **Type & Role:** `n8n-nodes-base.code` (Data Validation). Verifies the presence of usable transcript text and categorizes errors.
  - **Configuration:** Mode: `runOnceForAllItems`.
  - **Expressions/JS Code:** JavaScript checking for empty transcripts, mapping error codes, and flagging non-retryable speech errors (`no_speech`).
  - **Connections:** Input from Node `10`, output connects to Node `12`.
  - **Potential Failures:** Malformed actor responses.

- **12 — Analyze Reel With AI**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.chainLlm` (AI Processing). Sends Reel metadata, caption, and transcript to the LLM using a rigid system prompt enforcing a strict JSON output structure.
  - **Configuration:** Prompt Type: `define`. Error handling configured to continue on error output.
  - **Expressions:** Text prompt string injecting JSON-stringify payloads of reel metadata and transcripts.
  - **Connections:** Inputs from Node `11` (main) and Node `12b` (AI model). Success output connects to Node `13`; Error output connects to Node `16`.
  - **Potential Failures:** LLM timeouts, API quota exhaustion, or malformed JSON responses failing schema compliance.

- **12b — DeepSeek (OpenAI-compatible)**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model Provider). Configures the OpenAI-compatible chat model endpoint pointing to DeepSeek.
  - **Configuration:** Model: `deepseek-flash`, Credential type: OpenAI API (`openAiApi`), Response Format: `json_object`, Max Tokens: 4000, Responses API disabled, Extra Body: `{"thinking": {"type": "disabled"}}`.
  - **Expressions:** None.
  - **Connections:** Output connects to Node `12` via `ai_languageModel`.
  - **Potential Failures:** Invalid API keys, base URL misconfigurations, or provider downtime.

- **13 — Validate AI Output**
  - **Type & Role:** `n8n-nodes-base.code` (Data Validation). Validates that all required fields are present in the AI-generated JSON response.
  - **Configuration:** Mode: `runOnceForAllItems`.
  - **Expressions/JS Code:** JavaScript verifying 16 mandatory schema keys, parsing array fields into formatted text strings, and adding an ISO timestamp (`processed_at`).
  - **Connections:** Input from Node `12`, output connects to Node `14`.
  - **Potential Failures:** Missing schema keys causing degradation flags (`ai_ok = false`).

---

#### 2.6 Data Persistence and Reporting
This block saves structured insights to Google Sheets, updates the n8n registry table, compiles run diagnostics, and optionally sends a Telegram notification.

- **14 — Save Reel Intelligence**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Data Output). Appends or updates processed Reel intelligence rows in Google Sheets.
  - **Configuration:** Resource: `sheet`, Operation: `appendOrUpdate`, Sheet Name: `Reel Intelligence`, Matching Column: `reel_key`, Extra Data Handling: `ignoreIt`. Error handling configured to continue on error output.
  - **Expressions:** Auto-mapped input data matching predefined column schemas.
  - **Connections:** Input from Node `13`. Success output connects to Node `15`; Error output connects to Node `16`.
  - **Potential Failures:** Sheet permission errors, missing columns in the target sheet, or network timeouts.

- **15 — Mark As Processed**
  - **Type & Role:** `n8n-nodes-base.dataTable` (Database Operation). Updates the registry table to mark a Reel as successfully processed.
  - **Configuration:** Resource: `row`, Operation: `upsert`, Table: `competitor_reel_registry`, Filter condition: `reel_key` equals item `reel_key`.
  - **Expressions:** Dynamic column mapping updating `status` to `success`, incrementing `attempts`, clearing error codes, and setting timestamps.
  - **Connections:** Input from Node `14`, output loops back to Node `07`.
  - **Potential Failures:** Database write failures.

- **16 — Record Failure**
  - **Type & Role:** `n8n-nodes-base.dataTable` (Database Operation). Logs failed processing attempts in the registry table with incremented attempt counts.
  - **Configuration:** Resource: `row`, Operation: `upsert`, Table: `competitor_reel_registry`, Filter condition: `reel_key` equals item `reel_key`.
  - **Expressions:** Dynamic column mapping setting `status` to `failed`, incrementing `attempts`, recording error codes, and updating `last_attempt_at`.
  - **Connections:** Inputs from Nodes `10`, `12`, and `14` (error paths). Output loops back to Node `07`.
  - **Potential Failures:** Database write failures.

- **17 — Build Run Digest**
  - **Type & Role:** `n8n-nodes-base.code` (Data Aggregation). Summarizes workflow execution metrics across all iteration runs.
  - **Configuration:** Mode: `runOnceForAllItems`.
  - **Expressions/JS Code:** Custom helper function (`collectAllRuns`) enumerating execution runs (`runIndex 0..1000`) to aggregate output items from Nodes `13` and `16` without throwing index errors.
  - **Connections:** Input from Node `07` (loop completion), output connects to Node `18`.
  - **Potential Failures:** None.

- **18 — IF Telegram Enabled**
  - **Type & Role:** `n8n-nodes-base.if` (Flow Control). Determines whether to send a Telegram digest based on environment configurations.
  - **Configuration:** Condition: Left value string `false` equals `true` (disabled by default).
  - **Expressions:** None.
  - **Connections:** Input from Node `17`. True branch connects to Node `19`; False branch terminates workflow.
  - **Potential Failures:** None.

- **19 — Send Telegram Digest**
  - **Type & Role:** `n8n-nodes-base.telegram` (External Notification). Dispatches a run summary message to a Telegram chat. Node is disabled by default.
  - **Configuration:** Resource: `message`, Operation: `sendMessage`. Node status: Disabled.
  - **Expressions:** Markdown text string compiling run timestamp, analyzed count, failure count, and competitor distribution breakdown.
  - **Connections:** Input from Node `18`.
  - **Potential Failures:** Invalid Telegram bot tokens, incorrect chat IDs, or network timeouts.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **01 — When Competitors Should Be Checked** | `scheduleTrigger` | Daily schedule trigger | None | 01b | Turn competitor Instagram Reels into an AI content brief in Google Sheets... |
| **01b — Create Registry Table** | `dataTable` | Provisions deduplication table | 01 | 02 | Turn competitor Instagram Reels into an AI content brief in Google Sheets... |
| **02 — Load Competitors** | `googleSheets` | Loads competitor accounts | 01b | 02b — Filter Active Competitors | Turn competitor Instagram Reels into an AI content brief in Google Sheets... |
| **02b — Filter Active Competitors** | `code` | Cleans and filters active rows | 02 | 03 — Loop Over Competitors | Turn competitor Instagram Reels into an AI content brief in Google Sheets... |
| **03 — Loop Over Competitors** | `splitInBatches` | Iterates over competitors | 02b | 04, 06 — Prepare Reel Queue | Turn competitor Instagram Reels into an AI content brief in Google Sheets... |
| **04 — Fetch Recent Reels** | `apify` | Fetches recent Reels via Apify | 03 | 05 — Normalize Reel Data | REELS DISCOVERY: The only swappable step (node 04). Replacing it with another source does not affect downstream nodes. |
| **05 — Normalize Reel Data** | `code` | Normalizes and filters video items | 04 | 06 — Prepare Reel Queue | Turn competitor Instagram Reels into an AI content brief in Google Sheets... |
| **06 — Prepare Reel Queue** | `code` | Cross-competitor deduplication | 05 | 07 — Loop Over Reels | Turn competitor Instagram Reels into an AI content brief in Google Sheets... |
| **07 — Loop Over Reels** | `splitInBatches` | Iterates over individual Reels | 06, 15, 16 | 08 — Already Processed?, 17 | Turn competitor Instagram Reels into an AI content brief in Google Sheets... |
| **08 — Already Processed?** | `dataTable` | Queries deduplication registry | 07 | 09 — Is New Or Retryable? | DEDUPLICATION: Data Table registry, created by the workflow itself. A Reel is only marked done after it is actually saved. |
| **09 — Is New Or Retryable?** | `code` | Evaluates processing status | 08 | 09b — Should Process? | DEDUPLICATION: Data Table registry, created by the workflow itself. A Reel is only marked done after it is actually saved. |
| **09b — Should Process?** | `if` | Conditional routing gate | 09 | 07 — Loop Over Reels, 10 | DEDUPLICATION: Data Table registry, created by the workflow itself. A Reel is only marked done after it is actually saved. |
| **10 — Run Instagram Transcript Actor** | `apify` | Extracts audio transcript | 09b | 11 — Validate Transcript, 16 | INSTAGRAM TRANSCRIPT: One Actor run per Reel. |
| **11 — Validate Transcript** | `code` | Validates transcript text | 10 | 12 — Analyze Reel With AI | INSTAGRAM TRANSCRIPT: One Actor run per Reel. |
| **12 — Analyze Reel With AI** | `chainLlm` | Generates AI content brief | 11, 12b | 13 — Validate AI Output, 16 | AI ANALYSIS: Returns a structured 16-field JSON brief. |
| **12b — DeepSeek (OpenAI-compatible)** | `lmChatOpenAi` | LLM Provider configuration | None | 12 — Analyze Reel With AI | AI ANALYSIS: Returns a structured 16-field JSON brief. |
| **13 — Validate AI Output** | `code` | Validates mandatory AI fields | 12 | 14 — Save Reel Intelligence | AI ANALYSIS: Returns a structured 16-field JSON brief. |
| **14 — Save Reel Intelligence** | `googleSheets` | Saves insights to Google Sheets | 13 | 15 — Mark As Processed, 16 | OUTPUT: Appends one row per Reel to the Reel Intelligence sheet. |
| **15 — Mark As Processed** | `dataTable` | Updates registry to success | 14 | 07 — Loop Over Reels | OUTPUT: Appends one row per Reel to the Reel Intelligence sheet. |
| **16 — Record Failure** | `dataTable` | Logs processing failures | 10, 12, 14 | 07 — Loop Over Reels | ERROR PATH: A failure bumps the attempt counter and is retried next run. One bad Reel never stops the batch. |
| **17 — Build Run Digest** | `code` | Aggregates run metrics | 07 | 18 — IF Telegram Enabled | Turn competitor Instagram Reels into an AI content brief in Google Sheets... |
| **18 — IF Telegram Enabled** | `if` | Conditional Telegram gate | 17 | 19 — Send Telegram Digest | OPTIONAL TELEGRAM: Disabled by default. |
| **19 — Send Telegram Digest** | `telegram` | Sends Telegram summary | 18 | None | OPTIONAL TELEGRAM: Disabled by default. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Create the Schedule Trigger and Registry Table
1. Add a **Schedule Trigger** node named `01 — When Competitors Should Be Checked`. Configure it to run every 1 day at `09:00`.
2. Add a **Data Table** node named `01b — Create Registry Table`. Set resource to `Table`, operation to `create`, and table name to `competitor_reel_registry`. Enable `createIfNotExists`. Define columns: `reel_key` (string), `instagram_username` (string), `reel_url` (string), `status` (string), `attempts` (number), `error_code` (string), `processed_at` (string), `last_attempt_at` (string). Connect `01` to `01b`.

#### Step 2: Configure Competitor Input and Filtering
3. Add a **Google Sheets** node named `02 — Load Competitors`. Set resource to `Sheet`, operation to `Get`, and select your target spreadsheet with a sheet name of `Competitors`. Connect `01b` to `02`.
4. Add a **Code** node named `02b — Filter Active Competitors`. Set mode to `Run Once for All Items` and paste JavaScript logic to sanitize usernames, filter truthy `active` flags, and enforce default check limits. Connect `02` to `02b`.
5. Add a **Split In Batches** node named `03 — Loop Over Competitors`. Set batch size to `1`. Connect `02b` to `03`.

#### Step 3: Set Up Reel Discovery and Normalization
6. Add an **Apify** community node named `04 — Fetch Recent Reels`. Set resource to `Actors`, operation to `Run actor and get dataset`, and Actor ID to `y3uzYAhSuIK2ad8hE`. Set memory to `1024MB`, timeout to `300s`, and max charge to `$0.50`. Provide a custom JSON body fetching profile URLs in `reels` mode. Connect the first output of `03` to `04`.
7. Add a **Code** node named `05 — Normalize Reel Data`. Paste JavaScript code to filter for video content, construct canonical URLs (`https://www.instagram.com/reel/{shortcode}/`), and sort items by publication date. Connect `04` to `05`.
8. Add a **Code** node named `06 — Prepare Reel Queue`. Paste JavaScript deduplication logic utilizing a `Set` to prevent duplicate processing of collaborative posts. Connect `05` to `06`. Also connect the second output (loop completion) of `03` to `06`.

#### Step 4: Configure Deduplication and Queue Loop
9. Add a **Split In Batches** node named `07 — Loop Over Reels`. Set batch size to `1`. Connect `06` (and downstream loop return nodes `15` and `16`) to `07`.
10. Add a **Data Table** node named `08 — Already Processed?`. Set resource to `Row`, operation to `Get`, table name to `competitor_reel_registry`, and filter condition where `reel_key` equals `={{ $json.reel_key }}`. Enable `Always Output Data`. Connect `07` to `08`.
11. Add a **Code** node named `09 — Is New Or Retryable?`. Paste JavaScript logic evaluating registration status and retry attempts (limit: 3). Connect `08` to `09`.
12. Add an **If** node named `09b — Should Process?`. Set condition where `_decision` does not equal `skip`. Connect `09` to `09b`. Connect the `false` output back to `07`.

#### Step 5: Implement Transcription and AI Analysis
13. Add an **Apify** community node named `10 — Run Instagram Transcript Actor`. Set resource to `Actors`, operation to `Run actor and get dataset`, and Actor ID to `ZuAmrsZhEP0VQ7E9o`. Set memory to `512MB`, timeout to `600s`, and enable `Continue On Fail`. Provide a custom JSON body requesting text format and `enhancedMode: true`. Connect the `true` output of `09b` to `10`.
14. Add a **Code** node named `11 — Validate Transcript`. Paste JavaScript to check for valid transcript text and categorize errors. Connect `10` to `11`.
15. Add a **Advanced AI -> Chain LLM** node named `12 — Analyze Reel With AI`. Set prompt type to `Define`, add a System Message prompt specifying the 16-field JSON structure requirement, and enable `Continue On Fail`. Connect `11` to `12`.
16. Add a **Model -> Open UserModel** node named `12b — DeepSeek (OpenAI-compatible)`. Configure model name as `deepseek-flash`, select OpenAI API credentials, set response format to `json_object`, max tokens to `4000`, disable responses API, and add extra body: `{"thinking": {"type": "disabled"}}`. Connect `12b` to `12` via the `ai_languageModel` input.
17. Add a **Code** node named `13 — Validate AI Output`. Paste JavaScript to validate the 16 required keys and format list outputs. Connect `12` to `13`.

#### Step 6: Configure Output Persistence and Reporting
18. Add a **Google Sheets** node named `14 — Save Reel Intelligence`. Set resource to `Sheet`, operation to `Append or Update`, sheet name to `Reel Intelligence`, matching column to `reel_key`, and extra data handling to `Ignore It`. Enable `Continue On Fail`. Connect `13` to `14`.
19. Add a **Data Table** node named `15 — Mark As Processed`. Set resource to `Row`, operation to `Upsert`, table name to `competitor_reel_registry`, and matching column to `reel_key`. Map values to set `status` to `success` and increment `attempts`. Connect `14` (success output) to `15`. Connect `15` back to `07`.
20. Add a **Data Table** node named `16 — Record Failure`. Set resource to `Row`, operation to `Upsert`, table name to `competitor_reel_registry`, matching column to `reel_key`. Map values to set `status` to `failed`, increment `attempts`, and capture error codes. Connect error outputs from nodes `10`, `12`, and `14` to `16`. Connect `16` back to `07`.
21. Add a **Code** node named `17 — Build Run Digest`. Paste JavaScript utilizing run-index enumeration (`collectAllRuns`) to aggregate execution metrics from Nodes `13` and `16`. Connect the completion output of Node `07` to `17`.
22. Add an **If** node named `18 — IF Telegram Enabled`. Set condition where string `false` equals `true`. Connect `17` to `18`.
23. (Optional) Add a **Telegram** node named `19 — Send Telegram Digest`. Set resource to `Message`, operation to `Send Message`, provide bot credentials and chat ID, and format markdown text. Disable the node by default. Connect `18` (true output) to `19`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apify Instagram Reels Downloader Actor | [Apify Actor Documentation](https://apify.com/lance_api/instagram-reels-downloader-api) |
| Apify Instagram Video AI Transcript Actor | [Apify Actor Documentation](https://apify.com/lance_api/instagram-video-ai-transcript-scraper-api) |
| Apify Community Node Requirement | Requires `@apify/n8n-nodes-apify` installed on self-hosted instances. |