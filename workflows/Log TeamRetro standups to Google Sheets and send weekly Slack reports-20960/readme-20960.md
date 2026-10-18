Log TeamRetro standups to Google Sheets and send weekly Slack reports

https://n8nworkflows.xyz/workflows/log-teamretro-standups-to-google-sheets-and-send-weekly-slack-reports-20960


# Log TeamRetro standups to Google Sheets and send weekly Slack reports

### 1. Workflow Overview

This workflow automates the processing and reporting of TeamRetro standups by capturing completed standup submissions, transforming them into individual records, and logging them into Google Sheets. Additionally, it runs on a weekly schedule to compile participation metrics, track blockers, post formatted progress reports to Slack, and log historical trend data.

The architecture is divided into two distinct operational flows:
- **1.1 Standup Ingestion & Logging:** Receives real-time webhook events when a TeamRetro standup completes, filters out duplicate webhook deliveries, sets processing options, flattens participant updates into individual rows, and upserts them into a Google Sheets "Log" tab.
- **1.2 Weekly Reporting & Analytics:** Triggers every Monday morning, retrieves historical standup records from Google Sheets, calculates metrics (such as participation rates, new blockers, and long-running blockers) per team, sends an executive summary to a Slack channel, and appends the aggregated weekly results to a "Weekly" trend tracking tab.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Standup Ingestion & Logging
- **Overview:** Captures completed standups via webhook, deduplicates repeated transmission attempts, configures log generation rules, transforms nested participant updates into structured row items, and synchronizes them with Google Sheets.
- **Nodes Involved:** 
  - `When Standup Closes`
  - `Skip Repeat Deliveries`
  - `Settings`
  - `One Row per Update`
  - `Upsert into the Log`

- **Node Details:**
  - **When Standup Closes**
    - *Type & Technical Role:* `n8n-nodes-teamretro.teamRetroTrigger` (Webhook Trigger). Listens for completed TeamRetro standups.
    - *Configuration:* Subscribed to the `standup.completed` event. Requires a webhook signing secret.
    - *Expressions/Variables:* None.
    - *Connections:* Output connects to `Skip Repeat Deliveries`.
    - *Version-specific / Failure modes:* Requires valid webhook configuration in TeamRetro. Webhook delivery failures can occur if the n8n instance is unreachable.
  - **Skip Repeat Deliveries**
    - *Type & Technical Role:* `n8n-nodes-base.removeDuplicates` (Data Filter). Prevents duplicate processing caused by webhook retries.
    - *Configuration:* Uses operation `removeItemsSeenInPreviousExecutions`.
    - *Expressions/Variables:* Deduplication value: `={{ $json.deliveryId }}`
    - *Connections:* Input from `When Standup Closes`; output to `Settings`.
  - **Settings**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformer). Defines global configuration parameters for the transformation step.
    - *Configuration:* Assigns boolean `includeCarriedOver` (`true`) and string `blockerTopic` (`"block"`).
    - *Expressions/Variables:* Static assignments.
    - *Connections:* Input from `Skip Repeat Deliveries`; output to `One Row per Update`.
  - **One Row per Update**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Custom JavaScript). Parses nested JSON structures from the TeamRetro payload and flattens every individual update into a distinct row object.
    - *Configuration:* Custom JavaScript iterating through participants, topics, and individual updates. Evaluates whether topic titles contain the blocker keyword.
    - *Expressions/Variables:* References upstream node data using `$('Settings').first().json` and `$('When Standup Closes').first().json`.
    - *Connections:* Input from `Settings`; output to `Upsert into the Log`.
    - *Edge cases:* Throws errors if expected payload paths are missing or null.
  - **Upsert into the Log**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Cloud Integration). Writes or updates flat standup update records in Google Sheets.
    - *Configuration:* Operation `appendOrUpdate`. Matching column set to `Update ID`. Cell format set to `RAW`.
    - *Expressions/Variables:* Dynamic mapping via auto-map input data.
    - *Connections:* Input from `One Row per Update`; terminal node for this branch.
    - *Edge cases:* Google API rate limits, invalid document/sheet IDs, or permission errors on the target spreadsheet.

---

#### Block 1.2: Weekly Reporting & Analytics
- **Overview:** Executes a weekly schedule to analyze logged standup data, aggregates participation and blocker metrics per team, posts a digest to Slack, and records trends into a separate spreadsheet tab.
- **Nodes Involved:**
  - `Every Monday at 9 AM`
  - `Report Settings`
  - `Read the Log`
  - `Build the Weekly Report`
  - `Post the Report`
  - `Append to the Weekly Sheet`

- **Node Details:**
  - **Every Monday at 9 AM**
    - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Time Trigger). Initiates the weekly summary workflow.
    - *Configuration:* Interval set to trigger every week on Monday at 09:00.
    - *Expressions/Variables:* None.
    - *Connections:* Output connects to `Report Settings`.
  - **Report Settings**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformer). Defines reporting window parameters and target communication channels.
    - *Configuration:* Assigns numeric `days` (`7`) and string `slackChannel` (`"#scrum-masters"`).
    - *Expressions/Variables:* Static assignments.
    - *Connections:* Input from `Every Monday at 9 AM`; output to `Read the Log`.
  - **Read the Log**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Cloud Integration). Retrieves all rows from the primary log sheet for analysis.
    - *Configuration:* Operation `append` (default read behavior depending on sheet selection context).
    - *Connections:* Input from `Report Settings`; output to `Build the Weekly Report`.
  - **Build the Weekly Report**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Custom JavaScript). Filters log entries by date window, calculates team participation frequency, identifies fresh blockers, tracks longest-running blockers, and builds a Markdown-formatted Slack message.
    - *Configuration:* JavaScript code using Luxon DateTime utilities and native array manipulations.
    - *Expressions/Variables:* References `$('Report Settings').first().json` and input items (`$input.all()`).
    - *Connections:* Input from `Read the Log`; outputs to `Post the Report` and `Append to the Weekly Sheet`.
  - **Post the Report**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Cloud Integration). Sends the generated weekly metrics report to a designated Slack channel.
    - *Configuration:* Operation `select` (Post message). Target channel resolved dynamically.
    - *Expressions/Variables:* Text expression: `={{ $json.message }}`. Channel ID expression: `={{ $('Report Settings').first().json.slackChannel }}`.
    - *Connections:* Input from `Build the Weekly Report`; terminal node.
    - *Edge cases:* Slack token permission issues, channel not found, or missing bot scopes (`chat:write`).
  - **Append to the Weekly Sheet**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Cloud Integration). Appends aggregated weekly metrics to a trends tab.
    - *Configuration:* Operation `append`. Handling extra data set to ignore. Cell format set to `RAW`.
    - *Expressions/Variables:* Auto-maps input data containing summary fields (`Week ending`, `Team`, `Standups`, etc.).
    - *Connections:* Input from `Build the Weekly Report`; terminal node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Workflow overview and setup documentation. | None | None | ## TeamRetro: Log standups to Google Sheets and post a weekly participation report... |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Block 1 description. | None | None | ## 1. Catch the closed standup... |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Block 2 description. | None | None | ## 2. Upsert one row per update... |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Block 3 description. | None | None | ## 3. Read last week... |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Block 4 description. | None | None | ## 4. Report and record the trend... |
| `When Standup Closes` | `n8n-nodes-teamretro.teamRetroTrigger` | Triggers when a TeamRetro standup completes. | None | `Skip Repeat Deliveries` | |
| `Skip Repeat Deliveries` | `n8n-nodes-base.removeDuplicates` | Filters out duplicate webhook deliveries using the delivery ID. | `When Standup Closes` | `Settings` | |
| `Settings` | `n8n-nodes-base.set` | Defines configuration parameters for log generation (`includeCarriedOver`, `blockerTopic`). | `Skip Repeat Deliveries` | `One Row per Update` | |
| `One Row per Update` | `n8n-nodes-base.code` | Transforms participant updates into individual plain-text row objects. | `Settings` | `Upsert into the Log` | |
| `Upsert into the Log` | `n8n-nodes-base.googleSheets` | Upserts standup update rows into the Google Sheets Log tab based on Update ID. | `One Row per Update` | None | |
| `Every Monday at 9 AM` | `n8n-nodes-base.scheduleTrigger` | Triggers the weekly summary generation every Monday at 9:00 AM. | None | `Report Settings` | |
| `Report Settings` | `n8n-nodes-base.set` | Sets reporting scope parameters (`days`, `slackChannel`). | `Every Monday at 9 AM` | `Read the Log` | |
| `Read the Log` | `n8n-nodes-base.googleSheets` | Reads historical standup logs from Google Sheets. | `Report Settings` | `Build the Weekly Report` | |
| `Build the Weekly Report` | `n8n-nodes-base.code` | Aggregates team metrics, attendance, and blocker data, and builds a Slack message. | `Read the Log` | `Post the Report`, `Append to the Weekly Sheet` | |
| `Post the Report` | `n8n-nodes-base.slack` | Posts the formatted weekly participation and blocker report to Slack. | `Build the Weekly Report` | None | |
| `Append to the Weekly Sheet` | `n8n-nodes-base.googleSheets` | Appends weekly metrics summary data to the trends sheet. | `Build the Weekly Report` | None | |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, perform the following steps:

1. **Create Node:** Add a **TeamRetro Trigger** (`When Standup Closes`).
   - Set events to `standup.completed`.
   - Provide your TeamRetro signing secret.
2. **Create Node:** Add a **Remove Duplicates** (`Skip Repeat Deliveries`) node.
   - Set operation to `Remove Items Seen in Previous Executions`.
   - Set the deduplication value expression to `={{ $json.deliveryId }}`.
   - Connect `When Standup Closes` to this node.
3. **Create Node:** Add an **Edit Fields (Set)** (`Settings`) node.
   - Add a boolean assignment: Name `includeCarriedOver`, Value `true`.
   - Add a string assignment: Name `blockerTopic`, Value `block`.
   - Connect `Skip Repeat Deliveries` to this node.
4. **Create Node:** Add a **Code** (`One Row per Update`) node.
   - Set mode to JavaScript.
   - Paste the logic to iterate through participants, topics, and updates, checking against `includeCarriedOver` and filtering blocker topics using `blockerTopic`.
   - Connect `Settings` to this node.
5. **Create Node:** Add a **Google Sheets** (`Upsert into the Log`) node.
   - Configure Google Sheets credentials.
   - Set operation to `Append or Update`.
   - Set matching columns to `Update ID`.
   - Select your target Document and the "Log" sheet.
   - Connect `One Row per Update` to this node.
6. **Create Node:** Add a **Schedule Trigger** (`Every Monday at 9 AM`).
   - Configure trigger rule to fire weekly on Mondays at 09:00.
7. **Create Node:** Add an **Edit Fields (Set)** (`Report Settings`) node.
   - Add a number assignment: Name `days`, Value `7`.
   - Add a string assignment: Name `slackChannel`, Value `#scrum-masters` (or your target channel).
   - Connect `Every Monday at 9 AM` to this node.
8. **Create Node:** Add a **Google Sheets** (`Read the Log`) node.
   - Configure Google Sheets credentials.
   - Select the same document and the "Log" sheet to read data.
   - Connect `Report Settings` to this node.
9. **Create Node:** Add a **Code** (`Build the Weekly Report`) node.
   - Set mode to JavaScript.
   - Paste the aggregation logic to compute standup counts, participation frequency, fresh blockers, and longest-running blockers, compiling them into a Markdown `message` property per team.
   - Connect `Read the Log` to this node.
10. **Create Node:** Add a **Slack** (`Post the Report`) node.
    - Configure Slack OAuth2 credentials.
    - Set resource to `Message` and operation to `Post`.
    - Set Message to expression `={{ $json.message }}`.
    - Set Channel to expression `={{ $('Report Settings').first().json.slackChannel }}`.
    - Connect `Build the Weekly Report` to this node.
11. **Create Node:** Add a **Google Sheets** (`Append to the Weekly Sheet`) node.
    - Configure Google Sheets credentials.
    - Set operation to `Append`.
    - Select your target Document and the "Weekly" sheet.
    - Set Extra Data handling to `Ignore It`.
    - Connect `Build the Weekly Report` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheets "Log" tab schema setup | Columns required: `Date`, `Team`, `Standup`, `Person`, `Question`, `Blocker`, `Update`, `Carried over`, `Update ID` |
| Google Sheets "Weekly" tab schema setup | Columns required: `Week ending`, `Team`, `Standups`, `People posting`, `Updates`, `New blockers`, `Longest blocker (standups)` |
| TeamRetro Webhook Configuration | Paste the n8n production webhook URL into TeamRetro under Settings → Integrations → Webhooks and enable the **Standup Completed** event. |