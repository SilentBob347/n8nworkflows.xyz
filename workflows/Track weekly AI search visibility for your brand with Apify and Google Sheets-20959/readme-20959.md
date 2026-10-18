Track weekly AI search visibility for your brand with Apify and Google Sheets

https://n8nworkflows.xyz/workflows/track-weekly-ai-search-visibility-for-your-brand-with-apify-and-google-sheets-20959


# Track weekly AI search visibility for your brand with Apify and Google Sheets

### 1. Workflow Overview

This workflow measures how frequently a specific brand and its competitors are mentioned across selected AI search engines (such as ChatGPT, Perplexity, and Google AI Overviews) using Apify’s AI Share of Voice Actor. It automates weekly tracking, processes the resulting metrics, and logs them directly into a Google Sheets document.

The workflow logic is grouped into three distinct functional blocks:

- **1.1 Input Reception & Configuration:** Handles scheduled or manual triggers and defines the target brand, competitors, prompts, engines, and localization parameters.
- **1.2 Execution & Polling:** Initiates an asynchronous Apify Actor run, polls its status periodically, evaluates completion states, and handles errors or retrieves the final JSON report.
- **1.3 Data Transformation & Persistence:** Parses the hierarchical report into flattened individual records per brand and engine, then appends the rows to a designated Google Sheets worksheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Initializes the execution either on a weekly schedule (Mondays at 08:00) or via manual intervention, passing a structured JSON payload containing search queries and parameters.
- **Nodes Involved:** 
  - `Every Monday at 8:00`
  - `When clicking Execute workflow`
  - `Your brand and prompts`

- **Node Details:**
  - **Every Monday at 8:00**
    - *Type and Technical Role:* Schedule Trigger node. Automatically starts the workflow on a cron-based schedule.
    - *Configuration Choices:* Cron expression set to `0 8 * * 1`.
    - *Input/Output Connections:* Output connects to `Your brand and prompts`.
    - *Edge Cases/Failure Types:* Execution skipped if n8n instance is offline during the trigger window.
  - **When clicking Execute workflow**
    - *Type and Technical Role:* Manual Trigger node. Allows on-demand execution for testing or ad-hoc runs.
    - *Configuration Choices:* Default parameters.
    - *Input/Output Connections:* Output connects to `Your brand and prompts`.
  - **Your brand and prompts**
    - *Type and Technical Role:* Set (Edit Fields) node. Defines the raw JSON payload containing configuration data (brand name, domain, competitor list, target prompts, AI engines, country, language, and runs per prompt).
    - *Configuration Choices:* Raw mode enabled with a custom JSON payload representing target search tracking criteria.
    - *Input/Output Connections:* Inputs from `Every Monday at 8:00` and `When clicking Execute workflow`; output connects to `Start AI visibility check`.
    - *Edge Cases/Failure Types:* Invalid JSON syntax or unsupported engine identifiers will cause downstream API rejections.

#### 2.2 Execution & Polling
- **Overview:** Sends the configuration payload to the Apify Actor API, enforces spending caps, and manages a polling loop until the task succeeds, fails, or times out.
- **Nodes Involved:**
  - `Start AI visibility check`
  - `Wait 2 minutes`
  - `Check run status`
  - `Run finished?`
  - `Run failed`
  - `Get report`

- **Node Details:**
  - **Start AI visibility check**
    - *Type and Technical Role:* HTTP Request node. Submits a POST request to start the Apify Actor run (`touchrock~ai-share-of-voice`).
    - *Configuration Choices:* Method `POST`, URL `https://api.apify.com/v2/acts/touchrock~ai-share-of-voice/runs`, query parameter `maxTotalChargeUsd` set to `5`. Body parameter sends stringified JSON from the preceding Set node. Uses generic Header Auth.
    - *Input/Output Connections:* Input from `Your brand and prompts`; output connects to `Wait 2 minutes`.
    - *Edge Cases/Failure Types:* Authentication failure if API tokens are misconfigured; insufficient funds or budget cap rejections from Apify.
  - **Wait 2 minutes**
    - *Type and Technical Role:* Wait node. Pauses workflow execution for a fixed duration to allow the Apify Actor time to process prompts.
    - *Configuration Choices:* Unit set to `minutes`, amount set to `2`.
    - *Input/Output Connections:* Input from `Start AI visibility check` (and `Run finished?` on active status); output connects to `Check run status`.
  - **Check run status**
    - *Type and Technical Role:* HTTP Request node. Queries the current execution status of the Apify run ID.
    - *Configuration Choices:* Method `GET`, URL constructed dynamically using the run ID from the initial trigger node (`https://api.apify.com/v2/actor-runs/{{ $('Start AI visibility check').first().json.data.id }}`). Uses generic Header Auth.
    - *Input/Output Connections:* Input from `Wait 2 minutes`; output connects to `Run finished?`.
    - *Edge Cases/Failure Types:* Network timeouts or transient API unavailability from Apify.
  - **Run finished?**
    - *Type and Technical Role:* Switch node. Evaluates the run status returned by the Apify API.
    - *Configuration Choices:* Evaluates `{{ $json.data.status }}` against three rules: `SUCCEEDED` (Output 0), `READY` or `RUNNING` (Output 1), and Fallback for any other status (`Failed`).
    - *Input/Output Connections:* Input from `Check run status`; outputs connect to `Get report`, `Wait 2 minutes`, and `Run failed` respectively.
  - **Run failed**
    - *Type and Technical Role:* Stop and Error node. Terminates the workflow execution when an Actor run fails or aborts.
    - *Configuration Choices:* Custom error message evaluating run ID and status, generating a direct URL link to the Apify console run view.
    - *Input/Output Connections:* Input from `Run finished?` (Failed path); no outputs.
  - **Get report**
    - *Type and Technical Role:* HTTP Request node. Fetches the final output dataset from the Apify key-value store once the run succeeds.
    - *Configuration Choices:* Method `GET`, URL dynamically uses the default key-value store ID (`https://api.apify.com/v2/key-value-stores/{{ $json.data.defaultKeyValueStoreId }}/records/OUTPUT`). Uses generic Header Auth.
    - *Input/Output Connections:* Input from `Run finished?` (Succeeded path); output connects to `One row per brand and engine`.
    - *Edge Cases/Failure Types:* Missing records or storage retrieval errors if the output store is empty.

#### 2.3 Data Transformation & Persistence
- **Overview:** Normalizes the nested JSON report structure into distinct tabular rows and appends them to a target Google Sheets document.
- **Nodes Involved:**
  - `One row per brand and engine`
  - `Append to Google Sheets`

- **Node Details:**
  - **One row per brand and engine**
    - *Type and Technical Role:* Code node (JavaScript). Flattens the hierarchical `byEngine` report object into individual row objects containing key visibility metrics.
    - *Configuration Choices:* Iterates through engines and brands, calculates rounded percentages, assigns binary target brand flags (`yes`/`no`), formats dates to ISO strings (`YYYY-MM-DD`), and appends the Apify run URL.
    - *Input/Output Connections:* Input from `Get report`; output connects to `Append to Google Sheets`.
    - *Edge Cases/Failure Types:* Null reference errors if the report schema differs from expected data structures.
  - **Append to Google Sheets**
    - *Type and Technical Role:* Google Sheets node. Appends the mapped array of rows into a specific worksheet.
    - *Configuration Choices:* Operation set to `append`, mapping mode set to `autoMapInputData`. Requires configured OAuth2 credentials, target document ID, and sheet name.
    - *Input/Output Connections:* Input from `One row per brand and engine`; no downstream outputs.
    - *Edge Cases/Failure Types:* Authentication token expiration, missing target spreadsheet/worksheet names, or column header mismatches.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Documentation and setup instructions | None | None | ## Track your brand's AI visibility every week<br><br>**For:** marketers, SEO/GEO agencies and founders who want to know whether ChatGPT, Gemini, Perplexity, Claude and Google's AI answers recommend their brand or their competitors.<br><br>### How it works<br>1. Every Monday at 8:00 (or when you click **Execute workflow**) it sends your buyer questions to the AI engines you pick, through the [AI Visibility Tracker](https://apify.com/touchrock/ai-share-of-voice) Actor on Apify.<br>2. It checks every 2 minutes until the run ends. Small runs take a few minutes; many prompts and engines can take about 20.<br>3. It reads the run summary and appends one row per brand and engine to Google Sheets: mention rate, share of voice, average list rank and citation rate.<br><br>### Setup (about 10 minutes)<br>1. Create an Apify account and copy your API token from [API & Integrations](https://console.apify.com/settings/integrations).<br>2. Create one **Header Auth** credential: name `Authorization`, value `Bearer YOUR_APIFY_TOKEN`. Use it in **Start AI visibility check**, **Check run status** and **Get report**.<br>3. Edit **Your brand and prompts**: your brand, domain, competitors, the questions your buyers ask, engines, country and language.<br>4. Create a Google Sheet with a tab named `AI visibility` and these headers in row 1: `date, engine, brand, yourBrand, answers, mentionedIn, mentionRatePct, shareOfVoicePct, avgListRank, citationRatePct, runUrl`. Connect **Append to Google Sheets** and pick the sheet.<br>5. Click **Execute workflow** once to test, then activate the workflow.<br><br>### Cost<br>The Actor charges per AI answer checked (see its Apify page); failed checks are free. **Start AI visibility check** caps each run at 5 USD (`maxTotalChargeUsd`). On Apify's free plan the Actor checks up to 10 prompts per run. |
| Step 1 | n8n-nodes-base.stickyNote | Visual grouping for step 1 | None | None | ## 1. When and what to check<br>Runs weekly or by hand. Edit the JSON in **Your brand and prompts**. Engines: `chatgpt`, `gemini`, `perplexity`, `claude`, `google_ai_overview`, `google_ai_mode`. |
| Step 2 | n8n-nodes-base.stickyNote | Visual grouping for step 2 | None | None | ## 2. Run the Actor and wait<br>Starts the run, then checks its status every 2 minutes. A run that fails or times out stops the workflow with a link to the run. |
| Step 3 | n8n-nodes-base.stickyNote | Visual grouping for step 3 | None | None | ## 3. Save the results<br>Turns the run summary into one row per brand and engine (`all` = every engine together) and appends them to Google Sheets. |
| Every Monday at 8:00 | n8n-nodes-base.scheduleTrigger | Triggers workflow weekly | None | Your brand and prompts | |
| When clicking Execute workflow | n8n-nodes-base.manualTrigger | Allows manual execution | None | Your brand and prompts | |
| Your brand and prompts | n8n-nodes-base.set | Defines configuration payload | Every Monday at 8:00, When clicking Execute workflow | Start AI visibility check | |
| Start AI visibility check | n8n-nodes-base.httpRequest | Initiates Apify Actor run | Your brand and prompts | Wait 2 minutes | |
| Wait 2 minutes | n8n-nodes-base.wait | Pauses between polling checks | Start AI visibility check, Run finished? | Check run status | |
| Check run status | n8n-nodes-base.httpRequest | Polls execution status | Wait 2 minutes | Run finished? | |
| Run finished? | n8n-nodes-base.switch | Branches logic based on run state | Check run status | Get report, Wait 2 minutes, Run failed | |
| Run failed | n8n-nodes-base.stopAndError | Stops execution on failure | Run finished? | None | |
| Get report | n8n-nodes-base.httpRequest | Fetches output data | Run finished? | One row per brand and engine | |
| One row per brand and engine | n8n-nodes-base.code | Normalizes report into rows | Get report | Append to Google Sheets | |
| Append to Google Sheets | n8n-nodes-base.googleSheets | Saves metrics to spreadsheet | One row per brand and engine | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credentials:**
   - Generate an Apify API token from your Apify account settings.
   - Set up an n8n **Header Auth** credential with Name: `Authorization` and Value: `Bearer <YOUR_APIFY_TOKEN>`.
   - Set up n8n **Google Sheets OAuth2** credentials.

2. **Set Up the Triggers:**
   - Add a **Schedule Trigger** node (`Every Monday at 8:00`) configured with cron expression `0 8 * * 1`.
   - Add a **Manual Trigger** node (`When clicking Execute workflow`).

3. **Configure the Input Payload:**
   - Add a **Set** node (`Your brand and prompts`).
   - Set mode to **Raw** and populate the JSON payload with your target brand, domain, competitor list, search prompts, chosen engines (`chatgpt`, `perplexity`, `google_ai_overview`), country, language, and runs-per-prompt settings.
   - Connect both triggers to this node.

4. **Initiate the Apify Run:**
   - Add an **HTTP Request** node (`Start AI visibility check`).
   - Set Method to `POST`, URL to `https://api.apify.com/v2/acts/touchrock~ai-share-of-voice/runs`, and authentication to the Apify Header Auth credential.
   - Add a query parameter: Name `maxTotalChargeUsd`, Value `5`.
   - Set body specification to JSON, sending `={{ JSON.stringify($json) }}`.
   - Connect `Your brand and prompts` to this node.

5. **Configure the Polling Loop:**
   - Add a **Wait** node (`Wait 2 minutes`) set to wait for `2` minutes. Connect `Start AI visibility check` to it.
   - Add an **HTTP Request** node (`Check run status`). Set Method to `GET`, URL to `https://api.apify.com/v2/actor-runs/{{ $('Start AI visibility check').first().json.data.id }}`, and use Header Auth. Connect `Wait 2 minutes` to it.
   - Add a **Switch** node (`Run finished?`). Connect `Check run status` to it. Configure rules to branch on `{{ $json.data.status }}`:
     - Output 0 (Succeeded): Equals `SUCCEEDED`
     - Output 1 (Still running): Equals `READY` or `RUNNING`
     - Fallback (Failed): Any other status.
   - Connect the "Still running" branch back to the `Wait 2 minutes` node.

6. **Handle Errors and Retrieve Results:**
   - Add a **Stop and Error** node (`Run failed`). Connect the fallback branch of `Run finished?` to it, with error message: `=Apify run {{ $json.data.id }} ended with status {{ $json.data.status }}: https://console.apify.com/view/runs/{{ $json.data.id }}`.
   - Add an **HTTP Request** node (`Get report`). Set Method to `GET`, URL to `https://api.apify.com/v2/key-value-stores/{{ $json.data.defaultKeyValueStoreId }}/records/OUTPUT`, and use Header Auth. Connect the "Succeeded" output of `Run finished?` to this node.

7. **Transform and Store Data:**
   - Add a **Code** node (`One row per brand and engine`) and paste the JavaScript normalization snippet to flatten the report into rows containing `date`, `engine`, `brand`, `yourBrand`, `answers`, `mentionedIn`, `mentionRatePct`, `shareOfVoicePct`, `avgListRank`, `citationRatePct`, and `runUrl`. Connect `Get report` to it.
   - Add a **Google Sheets** node (`Append to Google Sheets`). Connect `One row per brand and engine` to it.
   - Configure the node operation to `append`, select your target spreadsheet and worksheet (`AI visibility`), and map fields using **Auto Map Input Data**. Ensure your Google Sheet headers match: `date, engine, brand, yourBrand, answers, mentionedIn, mentionRatePct, shareOfVoicePct, avgListRank, citationRatePct, runUrl`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| AI Visibility Tracker Actor on Apify | [Apify Actor Documentation & Page](https://apify.com/touchrock/ai-share-of-voice) |
| Apify API & Integrations Token Settings | [Apify Console Settings](https://console.apify.com/settings/integrations) |