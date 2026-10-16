Sync job applicants between Google Sheets and Notion and email stage updates with Gmail

https://n8nworkflows.xyz/workflows/sync-job-applicants-between-google-sheets-and-notion-and-email-stage-updates-with-gmail-20943


# Sync job applicants between Google Sheets and Notion and email stage updates with Gmail

### 1. Workflow Overview

This workflow synchronizes job applications from a Google Sheets form response log into a Notion candidates database, prevents duplicates, marks processed rows, and automates candidate communication by sending stage-based email templates via Gmail whenever a candidate's status changes in Notion.

The system is divided into two primary logical blocks:

- **1.1 Candidate Intake and Synchronization:** Runs hourly or manually to read new rows from Google Sheets, cross-references them against existing Notion candidate records to eliminate duplicates, creates new records for valid candidates, and updates the source sheet to flag processed rows.
- **1.2 Stage Change Automation and Communication:** Listens for updates in the Notion Candidates database, tracks state transitions using an n8n Data Table to avoid false triggers, fetches corresponding email templates from Notion, parses dynamic variables, and sends status update emails to candidates via Gmail.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Candidate Intake and Synchronization

- **Overview:** Periodically pulls new job submissions from Google Sheets, validates against existing Notion records to prevent duplicate entries, writes valid candidates to Notion, and marks the source rows as processed.
- **Nodes Involved:**
  - `Every hour`
  - `Run it now`
  - `[cred] Sheets - read new applicants`
  - `[cred] Notion - read existing candidates`
  - `drop candidates already in Notion`
  - `new candidate?`
  - `[cred] Notion - add the candidate`
  - `mark the row processed`
  - `[cred] Sheets - write processed status`

- **Node Details:**
  - **`Every hour`**
    - **Type and Role:** Schedule Trigger node (`n8n-nodes-base.scheduleTrigger`, v1.3). Initiates the intake workflow automatically on a timed interval.
    - **Configuration:** Configured to run based on interval rules.
    - **Inputs / Outputs:** Input: None | Output: Triggers `[cred] Sheets - read new applicants`.
    - **Edge Cases:** Execution depends on n8n server uptime and time zones.
  - **`Run it now`**
    - **Type and Role:** Manual Trigger node (`n8n-nodes-base.manualTrigger`, v1). Provides an ad-hoc execution entry point.
    - **Configuration:** Default manual execution parameters.
    - **Inputs / Outputs:** Input: None | Output: Triggers `[cred] Sheets - read new applicants`.
  - **`[cred] Sheets - read new applicants`**
    - **Type and Role:** Google Sheets node (`n8n-nodes-base.googleSheets`, v4.7). Reads unprocessed response rows from the recruitment Google Sheet.
    - **Configuration:** Uses Google OAuth2 (`k4FQMmaNPKWmcG5f`), targets document ID `1mjlkvPr2qX3GMaMLSAFuhJ63ptYhXk_Qtprk7y2YfPg`, sheet name `Form Responses 1`, filtering by lookup column `Processed`. Configured with `executeOnce: true`.
    - **Inputs / Outputs:** Input: `Every hour` or `Run it now` | Output: Connects to `[cred] Notion - read existing candidates`.
    - **Edge Cases:** API rate limits, revoked Google OAuth credentials, or missing sheet headers.
  - **`[cred] Notion - read existing candidates`**
    - **Type and Role:** Notion node (`n8n-nodes-base.notion`, v2.2). Fetches existing candidate entries to map current database state.
    - **Configuration:** Uses Notion OAuth2 (`MfmYL7xaF5LI0gOl`), resource `databasePage`, operation `getAll`, targeting database ID `3d125666-79c4-8009-8133-def325c4219c` ("Candidates Tracker"). Configured with `alwaysOutputData: true`.
    - **Inputs / Outputs:** Input: `[cred] Sheets - read new applicants` | Output: Connects to `drop candidates already in Notion`.
    - **Edge Cases:** Notion API timeouts or permission errors on the connected database.
  - **`drop candidates already in Notion`**
    - **Type and Role:** Code node (`n8n-nodes-base.code`, v2). Filters out Google Sheet rows whose email addresses already exist in Notion.
    - **Configuration:** JavaScript block mapping email properties to drop existing items. Configured with `alwaysOutputData: true`.
    - **Key Expressions:** `const alreadyInNotionEmails = $input.all().map(item => item.json.property_email); return $("[cred] Sheets - read new applicants").all().filter(item => !item.json["Email Address"].includes(alreadyInNotionEmails));`
    - **Inputs / Outputs:** Input: `[cred] Notion - read existing candidates` | Output: Connects to `new candidate?`.
  - **`new candidate?`**
    - **Type and Role:** If node (`n8n-nodes-base.if`, v2.3). Evaluates whether filtered rows contain valid new applicants to proceed.
    - **Configuration:** Validates existence of `Email Address` on incoming items.
    - **Key Expressions:** `={{ $input.item.json["Email Address"] }}` (Operator: `exists`).
    - **Inputs / Outputs:** Input: `drop candidates already in Notion` | Output: True branch connects to `[cred] Notion - add the candidate`; False branch connects to `mark the row processed`.
  - **`[cred] Notion - add the candidate`**
    - **Type and Role:** Notion node (`n8n-nodes-base.notion`, v2.2). Creates a new candidate entry page in Notion.
    - **Configuration:** Target database ID `3d125666-79c4-8009-8133-def325c4219c`. Maps fields: Name, Email, Whatsapp, Role, CV / Resume (URL), and sets default status to `INTAKE`.
    - **Key Expressions:** Property values mapped using expressions like `={{ $json.Name }}` and `={{ $json['Email Address'] }}`.
    - **Inputs / Outputs:** Input: `new candidate?` (True) | Output: Connects to `mark the row processed`.
  - **`mark the row processed`**
    - **Type and Role:** Code node (`n8n-nodes-base.code`, v2). Consolidates applicant references to prepare row updates. Configured with `executeOnce: true`.
    - **Key Expressions:** `return $('[cred] Sheets - read new applicants').all();`
    - **Inputs / Outputs:** Input: `new candidate?` (False) or `[cred] Notion - add the candidate` | Output: Connects to `[cred] Sheets - write processed status`.
  - **`[cred] Sheets - write processed status`**
    - **Type and Role:** Google Sheets node (`n8n-nodes-base.googleSheets`, v4.7). Updates the source sheet row to flag the applicant as processed.
    - **Configuration:** Operation `update`, target document ID `1mjlkvPr2qX3GMaMLSAFuhJ63ptYhXk_Qtprk7y2YfPg`, sheet name `Form Responses 1`. Matches rows using `row_number` and sets `Processed` to `Yes`.
    - **Key Expressions:** `row_number` set via `={{ $json.row_number }}`.
    - **Inputs / Outputs:** Input: `mark the row processed` | Output: None.

---

#### Block 1.2: Stage Change Automation and Communication

- **Overview:** Listens for status updates in the Notion Candidates database, evaluates state history against an n8n Data Table to confirm genuine status changes, pulls email templates from a Notion database, formats variables, and sends emails via Gmail.
- **Nodes Involved:**
  - `When a candidate stage changes`
  - `read the previous stage`
  - `store the new stage`
  - `route by candidate stage`
  - `[cred] Notion - get the stage template`
  - `build the email from the template`
  - `[cred] Gmail - email the candidate`

- **Node Details:**
  - **`When a candidate stage changes`**
    - **Type and Role:** Notion Trigger node (`n8n-nodes-base.notionTrigger`, v1). Triggers whenever a page is updated in the specified Notion database.
    - **Configuration:** Watches event `pagedUpdatedInDatabase` polled every minute. Database ID: `3d125666-79c4-8009-8133-def325c4219c`. Authenticated via OAuth2 (`MfmYL7xaF5LI0gOl`).
    - **Inputs / Outputs:** Input: None | Output: Connects to `read the previous stage`.
    - **Edge Cases:** Polling interval delay (up to 1 minute).
  - **`read the previous stage`**
    - **Type and Role:** Data Table node (`n8n-nodes-base.dataTable`, v1.1). Retrieves the last stored candidate snapshot from local storage.
    - **Configuration:** Operation `get`, data table ID `OOto8f6Bin6gLrv8` (`candidates_trackers_histories`). Filters condition where `notion_id` equals current item ID. Configured with `alwaysOutputData: true`.
    - **Key Expressions:** `={{ $json.id }}`
    - **Inputs / Outputs:** Input: `When a candidate stage changes` | Output: Connects to `store the new stage`.
  - **`store the new stage`**
    - **Type and Role:** Data Table node (`n8n-nodes-base.dataTable`, v1.1). Upserts the current candidate state into the history data table.
    - **Configuration:** Operation `upsert`, data table ID `OOto8f6Bin6gLrv8`. Matches by `notion_id`.
    - **Key Expressions:** `notion_id`: `={{ $('When a candidate stage changes').item.json.id }}`, `values`: `={{ JSON.stringify($('When a candidate stage changes').item.json) }}`.
    - **Inputs / Outputs:** Input: `read the previous stage` | Output: Connects to `route by candidate stage`.
  - **`route by candidate stage`**
    - **Type and Role:** Switch node (`n8n-nodes-base.switch`, v3.4). Evaluates if the candidate status has genuinely changed.
    - **Configuration:** Expression mode with 2 outputs. Compares previous and new status states.
    - **Key Expressions:** JavaScript expression checking equality of `newState?.Status === oldState?.Status`.
    - **Inputs / Outputs:** Input: `store the new stage` | Output: Connects to `[cred] Notion - get the stage template`.
  - **`[cred] Notion - get the stage template`**
    - **Type and Role:** Notion node (`n8n-nodes-base.notion`, v2.2). Fetches the matching email template from a Notion database based on the candidate's current status.
    - **Configuration:** Resource `databasePage`, operation `getAll`, database ID `3d125666-79c4-8090-8f93-d005f0832688` ("Email Templates"). Filter condition: `For Status` select equals candidate status.
    - **KeyExpressions:** `={{ $('When a candidate stage changes').item.json.Status }}`.
    - **Inputs / Outputs:** Input: `route by candidate stage` | Output: Connects to `build the email from the template`.
  - **`build the email from the template`**
    - **Type and Role:** Code node (`n8n-nodes-base.code`, v2). Parses template placeholders and replaces them with candidate properties.
    - **Configuration:** JavaScript string replacement engine matching bracketed keys against candidate payload fields.
    - **Key Expressions:** Regex evaluation mapping template properties (`property_subject`, `property_body`) to candidate data attributes.
    - **Inputs / Outputs:** Input: `[cred] Notion - get the stage template` | Output: Connects to `[cred] Gmail - email the candidate`.
  - **`[cred] Gmail - email the candidate`**
    - **Type and Role:** Gmail node (`n8n-nodes-base.gmail`, v2.2). Sends the compiled email update to the applicant.
    - **Configuration:** Uses Gmail OAuth2 (`YBQKv4RPXJuZdCao`). Email type set to text, appending no attribution.
    - **Key Expressions:** Recipient: `={{ $('When a candidate stage changes').item.json.Email }}`, Subject: `={{ $json.email_subject }}`, Message: `={{ $json.email_body }}`.
    - **Inputs / Outputs:** Input: `build the email from the template` | Output: None.
    - **Edge Cases:** Gmail sending limits, invalid email strings, or missing OAuth scope permissions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Every hour` | `scheduleTrigger` | Triggers intake flow on interval | None | `[cred] Sheets - read new applicants` | Template overview |
| `Run it now` | `manualTrigger` | Manual intake flow trigger | None | `[cred] Sheets - read new applicants` | Template overview |
| `[cred] Sheets - read new applicants` | `googleSheets` | Reads unprocessed Google Form responses | `Every hour`, `Run it now` | `[cred] Notion - read existing candidates` | Template overview<br>1. Collect the applications |
| `[cred] Notion - read existing candidates` | `notion` | Reads existing database records for matching | `[cred] Sheets - read new applicants` | `drop candidates already in Notion` | 1. Collect the applications |
| `drop candidates already in Notion` | `code` | Filters out applicants already present in Notion | `[cred] Notion - read existing candidates` | `new candidate?` | Template overview<br>2. The step that stops duplicates |
| `new candidate?` | `if` | Validates if candidate is genuinely new | `drop candidates already in Notion` | `[cred] Notion - add the candidate`, `mark the row processed` | 2. The step that stops duplicates |
| `[cred] Notion - add the candidate` | `notion` | Creates new candidate entry in Notion | `new candidate?` (True) | `mark the row processed` | 3. Add, then mark the row processed |
| `mark the row processed` | `code` | Prepares rows for status update | `new candidate?` (False), `[cred] Notion - add the candidate` | `[cred] Sheets - write processed status` | 3. Add, then mark the row processed |
| `[cred] Sheets - write processed status` | `googleSheets` | Marks sheet row as processed (`Yes`) | `mark the row processed` | None | Template overview<br>3. Add, then mark the row processed |
| `When a candidate stage changes` | `notionTrigger` | Listens for updates in Notion Candidates database | None | `read the previous stage` | Template overview<br>4. Why the previous stage is stored |
| `read the previous stage` | `dataTable` | Retrieves last recorded candidate stage snapshot | `When a candidate stage changes` | `store the new stage` | Template overview<br>4. Why the previous stage is stored |
| `store the new stage` | `dataTable` | Upserts current candidate state snapshot | `read the previous stage` | `route by candidate stage` | Template overview<br>4. Why the previous stage is stored |
| `route by candidate stage` | `switch` | Evaluates if candidate stage has actually changed | `store the new stage` | `[cred] Notion - get the stage template` | Template overview<br>5. One template per stage |
| `[cred] Notion - get the stage template` | `notion` | Fetches email template matching current status | `route by candidate stage` | `build the email from the template` | Template overview<br>5. One template per stage |
| `build the email from the template` | `code` | Builds email subject and body from template | `[cred] Notion - get the stage template` | `[cred] Gmail - email the candidate` | Template overview<br>5. One template per stage |
| `[cred] Gmail - email the candidate` | `gmail` | Sends status email update via Gmail | `build the email from the template` | None | Template overview |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Triggers:**
   - Add a **Schedule Trigger** node (`Every hour`) configured for interval execution.
   - Add a **Manual Trigger** node (`Run it now`) for ad-hoc testing.
   - Add a **Notion Trigger** node (`When a candidate stage changes`) set to watch `pagedUpdatedInDatabase` every minute, linked to your Candidates Tracker database ID.
2. **Setup Google Sheets Intake Integration:**
   - Add a **Google Sheets** node (`[cred] Sheets - read new applicants`). Authenticate with Google OAuth2. Set operation to read rows filtered by lookup column `Processed`.
   - Connect both triggers to this Google Sheets node.
3. **Setup Notion Reading and De-duplication:**
   - Add a **Notion** node (`[cred] Notion - read existing candidates`) targeting your Candidates Tracker database using Notion OAuth2 (`getAll` operation).
   - Add a **Code** node (`drop candidates already in Notion`) containing JavaScript logic to filter out email addresses already present in Notion.
   - Add an **If** node (`new candidate?`) checking if the filtered email address exists.
4. **Setup Candidate Creation and Sheet Update:**
   - Connect the True branch of the If node to a **Notion** node (`[cred] Notion - add the candidate`) mapping Name, Email, Whatsapp, Role, CV URL, and initial status (`INTAKE`).
   - Connect the False branch (and the output of the add candidate node) to a **Code** node (`mark the row processed`) to gather target row references.
   - Attach a **Google Sheets** node (`[cred] Sheets - write processed status`) updating rows where `row_number` matches, setting column `Processed` to `Yes`.
5. **Setup State Tracking & History:**
   - Connect the Notion Trigger to a **Data Table** node (`read the previous stage`) targeting your history table (`candidates_trackers_histories`) filtering by `notion_id`.
   - Connect this to another **Data Table** node (`store the new stage`) using an `upsert` operation on `notion_id` with stringified JSON values.
6. **Setup Routing and Email Communication:**
   - Attach a **Switch** node (`route by candidate stage`) evaluating whether previous and new status states match.
   - Connect the matching branch to a **Notion** node (`[cred] Notion - get the stage template`) querying your Email Templates database where `For Status` matches the candidate's status.
   - Add a **Code** node (`build the email from the template`) to parse bracketed template placeholders (`[Name]`, etc.) using candidate property fields.
   - Add a **Gmail** node (`[cred] Gmail - email the candidate`) authenticated via Gmail OAuth2, setting recipient, dynamic subject, and text body.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Designed for small recruiting teams and hiring managers collecting applications via Google Forms and tracking candidates in Notion. | Target Use Case |
| Requires three active OAuth2 connections: Notion, Google Sheets, and Gmail. | Credential Prerequisites |
| Ensure the n8n Data Table used for tracking state transitions exists with columns for `notion_id` and `values`. | Data Table Requirement |