Sync weekly Famulor assistant inventory with Google Sheets

https://n8nworkflows.xyz/workflows/sync-weekly-famulor-assistant-inventory-with-google-sheets-20931


# Sync weekly Famulor assistant inventory with Google Sheets

### 1. Workflow Overview

The **Sync weekly Famulor assistant inventory with Google Sheets** workflow automates the retrieval, validation, and synchronization of AI assistant data from the Famulor API to a Google Sheets document. Designed for operations teams and Famulor workspace administrators, it ensures that an up-to-date inventory of active and inactive assistants is maintained centrally.

The logical flow is structured into the following blocks:
- **1.1 Input Reception & Configuration:** Triggers the workflow either manually for testing or automatically on a weekly schedule, initializing runtime configuration flags (such as `previewOnly`).
- **1.2 Data Extraction & Validation:** Interacts with the Famulor REST API using robust pagination parameters, retrieves assistant records, and executes deep programmatic validation checks.
- **1.3 Decision & Routing:** Determines whether the workflow operates in dry-run mode (previewing the prepared data) or live-execution mode (writing to the destination spreadsheet).
- **1.4 Spreadsheet Synchronization:** Upserts validated records into Google Sheets using the unique `Assistant ID` as the matching key.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
This block serves as the entry point for the workflow, handling schedule-based or manual triggers and setting foundational configuration flags.

- **Nodes Involved:** 
  - `Run an inventory preview`
  - `Every Monday at 09:00`
  - `Configure inventory`

- **Node Details:**
  - **Run an inventory preview**
    - *Type & Role:* Manual Trigger (n8n-nodes-base.manualTrigger). Initiates executions on demand.
    - *Configuration:* Default parameters.
    - *Connections:* Output connects to `Configure inventory`.
  - **Every Monday at 09:00**
    - *Type & Role:* Schedule Trigger (n8n-nodes-base.scheduleTrigger). Automated weekly execution engine.
    - *Configuration:* Runs every Monday at 09:00 AM (configured for timezone `Europe/Berlin`).
    - *Connections:* Output connects to `Configure inventory`.
  - **Configure inventory**
    - *Type & Role:* Set Node (n8n-nodes-base.set). Establishes configuration variables for downstream logic.
    - *Configuration:* Assigns a boolean variable `previewOnly` set to `true`.
    - *Connections:* Input from triggers; output connects to `Read Famulor assistant pages`.
    - *Potential Failures:* Expression resolution failures if downstream nodes improperly reference variable scopes.

---

#### 2.2 Data Extraction & Validation
This block fetches paginated records from the Famulor platform, enforces hard limits, and executes strict data integrity checks before transforming the payload.

- **Nodes Involved:**
  - `Read Famulor assistant pages`
  - `Validate and prepare assistant rows`

- **Node Details:**
  - **Read Famulor assistant pages**
    - *Type & Role:* HTTP Request (n8n-nodes-base.httpRequest). Pulls paginated assistant data from the Famulor REST API endpoint (`https://app.famulor.io/api/v1/assistants`).
    - *Configuration:* Uses GET method with query parameters (`limit=200`, `offset=0`). Configured with built-in pagination handling using parameter updates (`offset = $pageCount * 200`), requesting a maximum of 10 pages with a 300ms interval. Authentication uses generic HTTP Header Auth.
    - *Key Expressions:* Complete expression evaluates pagination stop conditions based on array structure, total metadata, and record count thresholds.
    - *Connections:* Input from `Configure inventory`; output connects to `Validate and prepare assistant rows`.
    - *Potential Failures:* Authentication errors (`401 Unauthorized`), API rate-limiting or timeouts, or changes in API response structure causing the completion expression to evaluate incorrectly.
  - **Validate and prepare assistant rows**
    - *Type & Role:* Code Node (n8n-nodes-base.code). Executes custom JavaScript (`runOnceForAllItems`) to validate pagination consistency, guard against unexpected data sizes (>2,000 records), and sanitize properties.
    - *Configuration:* Maps raw API entities into a clean nine-column inventory format (`Assistant ID`, `Name`, `Active`, `Mode`, `Speech language`, `Tags`, `Created at`, `Updated at`, `Last seen`).
    - *Key Expressions/Variables:* Generates an ISO timestamp for `Last seen` and injects the `previewOnly` flag fetched from the `Configure inventory` node.
    - *Connections:* Input from `Read Famulor assistant pages`; output connects to `Preview only inventory?`.
    - *Potential Failures:* Throws explicit runtime errors if pagination totals are invalid, record IDs are duplicated, or response limits are breached.

---

#### 2.3 Decision & Routing
This block acts as a conditional gatekeeper, separating dry-run preview flows from live execution paths.

- **Nodes Involved:**
  - `Preview only inventory?`
  - `Preview assistant rows`
  - `Has assistant rows?`

- **Node Details:**
  - **Preview only inventory?**
    - *Type & Role:* If Node (n8n-nodes-base.if). Evaluates the state of the configuration flag.
    - *Configuration:* Condition checks whether `{{ $json.previewOnly }}` evaluates to `true`.
    - *Connections:* Input from `Validate and prepare assistant rows`; True branch connects to `Preview assistant rows`, False branch connects to `Has assistant rows?`.
  - **Preview assistant rows**
    - *Type & Role:* Code Node (n8n-nodes-base.code). Passes through items when preview mode is active.
    - *Configuration:* Returns all input items directly without writing to external systems.
    - *Connections:* Input from the True branch of `Preview only inventory?`.
  - **Has assistant rows?**
    - *Type & Role:* If Node (n8n-nodes-base.if). Verifies if valid records exist before attempting database writes.
    - *Configuration:* Condition checks whether `{{ $json.hasRows }}` is `true`.
    - *Connections:* Input from the False branch of `Preview only inventory?`; True branch connects to `Upsert assistant rows`.

---

#### 2.4 Spreadsheet Synchronization
This block synchronizes the validated assistant records with a target Google Sheets spreadsheet.

- **Nodes Involved:**
  - `Upsert assistant rows`

- **Node Details:**
  - **Upsert assistant rows**
    - *Type & Role:* Google Sheets Node (n8n-nodes-base.googleSheets). Performs an upsert operation against a target Google Sheet.
    - *Configuration:* Operation set to `appendOrUpdate`. Matching column configured as `Assistant ID`. Mappings explicitly link output properties to spreadsheet columns (`Assistant ID`, `Name`, `Active`, `Mode`, `Speech language`, `Tags`, `Created at`, `Updated at`, `Last seen`). Cell formatting set to `RAW`.
    - *Connections:* Input from the True branch of `Has assistant rows?`.
    - *Potential Failures:* Google API authentication expiration, missing sheet headers, exceeding API write quotas, or concurrent write conflicts.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run an inventory preview | n8n-nodes-base.manualTrigger | Manual trigger entry point | None | Configure inventory | Who is this for? Operations teams and Famulor workspace administrators who want a weekly overview of their AI assistants in Google Sheets. <br><br>How it works Every Monday at 09:00, or on a manual run, the workflow reads assistants through the Famulor API. It checks pagination and prepares nine inventory fields. Google Sheets appends or updates rows by Assistant ID, preserving unrelated columns such as Owner and Review notes. Preview only starts enabled, so you can inspect the rows before writing. Prompts, tools, recordings and contact details are not copied to the sheet. Deleted assistants remain in the sheet; Last seen helps identify older entries. <br><br>Setup 1. Select an n8n HTTP Header Auth credential in Read Famulor assistant pages. Use Authorization as the header name and Bearer followed by your Famulor workspace API key as its value. The key needs assistants:read. Check the selected credential after import. <br>2. Create these sheet headers: Assistant ID, Name, Active, Mode, Speech language, Tags, Created at, Updated at, Last seen. Select your Google Sheets credential, spreadsheet and tab. <br>3. Set your workflow timezone; the default is Europe/Berlin. Run a manual preview, then turn Preview only off and activate the weekly schedule. <br><br>Requirements and customization Uses built-in n8n nodes and Google Sheets edit access. Incomplete, capped or inconsistent scans stop before writes; the default limit is 2,000 assistants. Customize the schedule and mapping. Restrict sheet access and execution-history access: API responses can include configuration fields even though the sheet excludes them. Concurrent sheet writes are not transactional. [Famulor setup guide](https://docs.famulor.io/automations/n8n). |
| Every Monday at 09:00 | n8n-nodes-base.scheduleTrigger | Scheduled weekly trigger | None | Configure inventory | Who is this for? Operations teams and Famulor workspace administrators who want a weekly overview of their AI assistants in Google Sheets. <br><br>How it works Every Monday at 09:00, or on a manual run, the workflow reads assistants through the Famulor API. It checks pagination and prepares nine inventory fields. Google Sheets appends or updates rows by Assistant ID, preserving unrelated columns such as Owner and Review notes. Preview only starts enabled, so you can inspect the rows before writing. Prompts, tools, recordings and contact details are not copied to the sheet. Deleted assistants remain in the sheet; Last seen helps identify older entries. <br><br>Setup 1. Select an n8n HTTP Header Auth credential in Read Famulor assistant pages. Use Authorization as the header name and Bearer followed by your Famulor workspace API key as its value. The key needs assistants:read. Check the selected credential after import. <br>2. Create these sheet headers: Assistant ID, Name, Active, Mode, Speech language, Tags, Created at, Updated at, Last seen. Select your Google Sheets credential, spreadsheet and tab. <br>3. Set your workflow timezone; the default is Europe/Berlin. Run a manual preview, then turn Preview only off and activate the weekly schedule. <br><br>Requirements and customization Uses built-in n8n nodes and Google Sheets edit access. Incomplete, capped or inconsistent scans stop before writes; the default limit is 2,000 assistants. Customize the schedule and mapping. Restrict sheet access and execution-history access: API responses can include configuration fields even though the sheet excludes them. Concurrent sheet writes are not transactional. [Famulor setup guide](https://docs.famulor.io/automations/n8n). |
| Configure inventory | n8n-nodes-base.set | Assigns configuration variables | Run an inventory preview, Every Monday at 09:00 | Read Famulor assistant pages | Who is this for? Operations teams and Famulor workspace administrators who want a weekly overview of their AI assistants in Google Sheets. <br><br>How it works Every Monday at 09:00, or on a manual run, the workflow reads assistants through the Famulor API. It checks pagination and prepares nine inventory fields. Google Sheets appends or updates rows by Assistant ID, preserving unrelated columns such as Owner and Review notes. Preview only starts enabled, so you can inspect the rows before writing. Prompts, tools, recordings and contact details are not copied to the sheet. Deleted assistants remain in the sheet; Last seen helps identify older entries. <br><br>Setup 1. Select an n8n HTTP Header Auth credential in Read Famulor assistant pages. Use Authorization as the header name and Bearer followed by your Famulor workspace API key as its value. The key needs assistants:read. Check the selected credential after import. <br>2. Create these sheet headers: Assistant ID, Name, Active, Mode, Speech language, Tags, Created at, Updated at, Last seen. Select your Google Sheets credential, spreadsheet and tab. <br>3. Set your workflow timezone; the default is Europe/Berlin. Run a manual preview, then turn Preview only off and activate the weekly schedule. <br><br>Requirements and customization Uses built-in n8n nodes and Google Sheets edit access. Incomplete, capped or inconsistent scans stop before writes; the default limit is 2,000 assistants. Customize the schedule and mapping. Restrict sheet access and execution-history access: API responses can include configuration fields even though the sheet excludes them. Concurrent sheet writes are not transactional. [Famulor setup guide](https://docs.famulor.io/automations/n8n). |
| Read Famulor assistant pages | n8n-nodes-base.httpRequest | Fetches paginated assistant records via API | Configure inventory | Validate and prepare assistant rows | Create an assistants:read workspace key. Use an n8n Header Auth credential, not a key pasted into the HTTP node. Verify the credential selected after import. |
| Validate and prepare assistant rows | n8n-nodes-base.code | Validates and formats assistant records | Read Famulor assistant pages | Preview only inventory? | Only the nine inventory fields go to Google Sheets. Prompt and tool details stay out of the destination. Invalid or incomplete pagination stops before writes. Execution-data storage is disabled in this template; choose your retention policy before using real data. |
| Preview only inventory? | n8n-nodes-base.if | Routes execution based on preview flag | Validate and prepare assistant rows | Preview assistant rows, Has assistant rows? | Only the nine inventory fields go to Google Sheets. Prompt and tool details stay out of the destination. Invalid or incomplete pagination stops before writes. Execution-data storage is disabled in this template; choose your retention policy before using real data. |
| Preview assistant rows | n8n-nodes-base.code | Outputs preview records | Preview only inventory? (True) | None | Create the exact headers in the overview. Choose your Google Sheets credential, spreadsheet and tab. Match on Assistant ID. Values use RAW input to keep names and tags as literal text. Extra Owner or Review notes columns are not mapped. Preview only starts enabled. |
| Has assistant rows? | n8n-nodes-base.if | Checks if valid records exist | Preview only inventory? (False) | Upsert assistant rows | Create the exact headers in the overview. Choose your Google Sheets credential, spreadsheet and tab. Match on Assistant ID. Values use RAW input to keep names and tags as literal text. Extra Owner or Review notes columns are not mapped. Preview only starts enabled. |
| Upsert assistant rows | n8n-nodes-base.googleSheets | Upserts inventory data into Google Sheets | Has assistant rows? (True) | None | Create the exact headers in the overview. Choose your Google Sheets credential, spreadsheet and tab. Match on Assistant ID. Values use RAW input to keep names and tags as literal text. Extra Owner or Review notes columns are not mapped. Preview only starts enabled. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Triggers:**
   - Add a **Manual Trigger** node (`Run an inventory preview`).
   - Add a **Schedule Trigger** node (`Every Monday at 09:00`). Set the trigger interval to weekly on Monday at 09:00 AM.
2. **Configure Variables:**
   - Add a **Set** node named `Configure inventory`. Connect both triggers to this node.
   - Define a boolean assignment: parameter name `previewOnly`, type `boolean`, value `true`.
3. **Set Up API Data Extraction:**
   - Add an **HTTP Request** node named `Read Famulor assistant pages`. Connect `Configure inventory` to it.
   - Set Method to `GET` and URL to `https://app.famulor.io/api/v1/assistants`.
   - Configure Authentication to `Generic Credential Type` -> `HTTP Header Auth`.
   - Add query parameters: `limit` = `200`, `offset` = `0`.
   - Enable pagination in node options: update parameter in each request (`offset` = `={{ $pageCount * 200 }}`), set max requests to 10, interval to 300ms, and configure the completion expression based on pagination metadata.
4. **Add Validation and Transformation Logic:**
   - Add a **Code** node named `Validate and prepare assistant rows`. Connect `Read Famulor assistant pages` to it.
   - Insert the JavaScript code to parse API responses, validate pagination totals, enforce the 2,000-record safety ceiling, map fields to the nine schema headers (`Assistant ID`, `Name`, `Active`, `Mode`, `Speech language`, `Tags`, `Created at`, `Updated at`, `Last seen`), and attach metadata (`hasRows`, `previewOnly`).
5. **Implement Routing and Preview Logic:**
   - Add an **If** node named `Preview only inventory?`. Connect `Validate and prepare assistant rows` to it.
   - Configure the condition to check if `{{ $json.previewOnly }}` equals `true`.
   - Add a **Code** node named `Preview assistant rows`. Connect the True output of `Preview only inventory?` to it, returning `$input.all()`.
   - Add an **If** node named `Has assistant rows?`. Connect the False output of `Preview only inventory?` to it. Configure condition to check if `{{ $json.hasRows }}` equals `true`.
6. **Configure Spreadsheet Synchronization:**
   - Add a **Google Sheets** node named `Upsert assistant rows`. Connect the True output of `Has assistant rows?` to it.
   - Select your Google Sheets credential, target Spreadsheet, and Sheet tab.
   - Set operation to `Append or Update`.
   - Define matching column as `Assistant ID`.
   - Map parameters explicitly:
     - `Assistant ID` $\rightarrow$ `={{ $json['Assistant ID'] }}`
     - `Name` $\rightarrow$ `={{ $json['Name'] }}`
     - `Active` $\rightarrow$ `={{ $json['Active'] }}`
     - `Mode` $\rightarrow$ `={{ $json['Mode'] }}`
     - `Speech language` $\rightarrow$ `={{ $json['Speech language'] }}`
     - `Tags` $\rightarrow$ `={{ $json['Tags'] }}`
     - `Created at` $\rightarrow$ `={{ $json['Created at'] }}`
     - `Updated at` $\rightarrow$ `={{ $json['Updated at'] }}`
     - `Last seen` $\rightarrow$ `={{ $json['Last seen'] }}`
   - Set cell format options to `RAW`.
7. **Credentials & Global Settings:**
   - Configure the Famulor HTTP Header Auth credential with header name `Authorization` and value `Bearer <your-workspace-api-key>`.
   - Configure the Google Sheets OAuth2 credential with access to your target document.
   - Set workflow timezone to `Europe/Berlin` in workflow settings.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Famulor official integration guide for automation | [Famulor setup guide](https://docs.famulor.io/automations/n8n) |